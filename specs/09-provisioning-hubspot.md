# Spec 09 — Sales → Tenant Provisioning (HubSpot, WF-0 app-side, status sync-back)

**Repo:** `seekly-client-insights` (the platform app — see doc 10) + HubSpot ops runbook.
**Depends on:** Spec 01 (schema: `clients` extensions, `client_profile`, `workflow_config`,
`comms_provisioning`, `activity_log`), Spec 06 (n8n owns the WF-0 saga orchestration — this
spec defines the app-side endpoints n8n calls and the HubSpot configuration that triggers it).
**Consistent with:** Spec 04 conventions — env vars declared in `src/env.ts`, secrets via
`src/server/vault/bitwarden.ts` where applicable, pg-boss queue pattern from
`src/server/pipeline/queue.ts`, service-token route guard modeled on
`src/server/spine/auth.ts`.

## Purpose

"Selling a client and turning them on is one motion" (doc 01). A HubSpot deal hitting
**Closed Won** must automatically produce a live tenant: client row, profile skeleton,
module switchboard (all OFF), portal login invite, comms record, and an onboarding checklist —
idempotently, so any replay/re-run is safe. Pilots that never touch HubSpot use the same
endpoint via an admin script. A daily job pushes summary fields back to the HubSpot company
record (summary fields ONLY — never operational data, doc 01/02 separation of concerns).

```
HubSpot deal → Closed Won
      │ private-app webhook (deal.propertyChange, dealstage)
      ▼
n8n WF-0 (Spec 06): filter closedwon → fetch deal+company+contact → build payload
      │ POST /api/internal/provision   (Bearer INTERNAL_SERVICE_TOKEN)
      ▼
control plane (THIS SPEC): saga steps 1–6, progress-recorded, idempotent on deal id
      │
      └─ n8n continues: Twilio subaccount + A2P submission, ops notify (Spec 06)
Daily: pg-boss `hubspot-sync` → PATCH company summary properties (THIS SPEC)
```

### Existing code you MUST reuse (do not reinvent)

| What | Where | Use |
|---|---|---|
| Service-token guard pattern | `src/server/spine/auth.ts` (`requireServiceToken`) | Copy the constant-time-compare pattern into `src/server/internal/auth.ts` with its own env var |
| Portal invite mechanics | `src/server/actions/portal-auth.ts` (`invitePortalUser`) | Extract shared helper (PR-4 §invite); token = `crypto.randomBytes(32).toString("base64url")`, bcrypt hash, 7-day expiry, `sendInviteEmail` |
| Invite email | `src/server/reports/email.ts` (`sendInviteEmail`) | called by the helper |
| pg-boss | `src/server/pipeline/queue.ts` (`getBoss`, `QUEUES`, `RETRY_QUEUES`) | add `hubspotSync` queue |
| Env | `src/env.ts` (`env`, `appUrl`) | add the two vars below |
| Logging | `src/lib/logger.ts` (`log`, `logError`) | everywhere |
| DB | `src/server/db` (`db`), `src/server/db/schema.ts` | tables below |

### New environment variables (add to the zod schema in `src/env.ts`)

```ts
  // --- Provisioning & HubSpot (Spec 09) ---
  // Private-app access token from the Seekly HubSpot portal (PR-1). Server-side only.
  HUBSPOT_PRIVATE_APP_TOKEN: z.string().optional(),
  // Bearer token n8n presents to /api/internal/* routes (generate: openssl rand -base64 32).
  // Distinct from SPINE_SERVICE_TOKEN — different consumer, independently rotatable.
  INTERNAL_SERVICE_TOKEN: z.string().min(32).optional(),
```

Both optional so dev boots without them; the routes/jobs fail closed with clear errors
(same philosophy as Spec 04's provider vars).

### Interface with Spec 01 (assumed schema)

Spec 01 owns these migrations; **Spec 01 wins on naming** — reconcile mechanically if it
differs. This spec's code assumes:

```ts
// clients — existing table, extended by Spec 01 (doc 10 "add tier/status/hubspot fields"):
export const clientTierEnum = pgEnum("client_tier", ["pilot", "core", "growth", "market_leader"]);
export const accountStatusEnum = pgEnum("account_status", ["onboarding", "active", "paused", "churned"]);
// added columns on clients:
//   tier: clientTierEnum("tier"),
//   accountStatus: accountStatusEnum("account_status").notNull().default("onboarding"),
//   hubspotCompanyId: text("hubspot_company_id"),   // unique where not null
//   hubspotDealId: text("hubspot_deal_id"),
//   mrr: integer("mrr_cents"),                       // cents; 49700 = $497
//   renewalAt: date("renewal_at"),
//   timezone: text("timezone").notNull().default("America/Toronto"),
//   industry: text("industry"),

// client_profile (doc 04): clientId pk/fk + services/serviceAreas/targetKeywords/competitors/
//   brandVoice/qualificationQuestions/campaignAngles/bookingLink/websiteUrl/websitePlatform/
//   reviewLink/approvalPreferences — all nullable (skeleton insert works with clientId alone).

// workflow_config (doc 04): pk (clientId, module), enabled bool default false, settings jsonb,
//   updatedBy text, updatedAt.

// comms_provisioning (doc 04): clientId pk/fk, twilioSubaccountSid, phoneNumber,
//   a2pBrandStatus text default 'none', a2pCampaignStatus text default 'none',
//   a2pCampaignType text, inboundEmailAddress text.
```

**Owned by THIS spec** (add to `src/server/db/schema.ts` + `npm run db:generate` migration,
unless Spec 01 already defined them — reconcile before writing the migration):

```ts
export const provisioningStatusEnum = pgEnum("provisioning_status", ["running", "completed", "failed"]);

/** WF-0 saga ledger: one row per provisioning attempt-target, keyed for idempotency. */
export const provisioningRuns = pgTable(
  "provisioning_runs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    // "hubspot:<dealId>" | "manual:<externalRef>" — THE idempotency key.
    idempotencyKey: text("idempotency_key").notNull(),
    clientId: uuid("client_id").references(() => clients.id, { onDelete: "set null" }),
    source: text("source").notNull(),                 // hubspot | manual
    payload: jsonb("payload").notNull(),              // validated request, replayable
    // { [stepKey]: { status: "done", at: ISO, detail?: string } } — re-runs skip "done".
    steps: jsonb("steps").notNull().default({}),
    status: provisioningStatusEnum("status").notNull().default("running"),
    error: text("error"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("provisioning_runs_idem_unique").on(t.idempotencyKey)],
);

export const onboardingTaskStatusEnum = pgEnum("onboarding_task_status", ["todo", "done", "n_a"]);

/** Onboarding checklist (doc 05/08): every unchecked box maps to a blocked module. */
export const onboardingTasks = pgTable(
  "onboarding_tasks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull().references(() => clients.id, { onDelete: "cascade" }),
    key: text("key").notNull(),                       // e.g. connect_google
    label: text("label").notNull(),                   // human-readable, shown in portal/admin
    blockingModule: text("blocking_module"),          // which switch this gates (nullable)
    status: onboardingTaskStatusEnum("status").notNull().default("todo"),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("onboarding_tasks_client_key_unique").on(t.clientId, t.key)],
);
```

---

## Ticket table

| Ticket | Title | Depends on |
|---|---|---|
| PR-1 | HubSpot portal setup: pipeline stages, private app + token, custom deal properties | — (ops task) |
| PR-2 | HubSpot company summary properties (UI steps + creation script) | PR-1 |
| PR-3 | Webhook subscription `dealstage` → n8n (exact UI steps + n8n→app contract) | PR-1, Spec 06 |
| PR-4 | `POST /api/internal/provision` — zod contract, saga handler, progress record | Spec 01 migration, own migration above |
| PR-5 | Manual provisioning path for pilots (script + admin note) | PR-4 |
| PR-6 | Status sync-back pg-boss job → HubSpot company properties | PR-2, PR-4 |

---

## PR-1 — HubSpot portal setup (ops runbook)

### 1a. Create the portal

1. app.hubspot.com → Create account → **Free CRM** ("CRM Suite Free"). Company name `Seekly`.
2. Invite the founder account (matthewa0903@gmail.com) as Super Admin if not the creator.

### 1b. Deal pipeline stages (doc 01: prospects → outreach → conversations → meetings → proposals → deals → Closed Won)

Edit the **default** Sales Pipeline rather than creating a new one — the default pipeline's
Closed Won stage keeps the stable internal ID `closedwon`, which the webhook filter (PR-3)
relies on.

1. Settings (gear) → **Objects → Deals** → **Pipelines** tab → pipeline "Sales Pipeline".
2. Rename/add stages so the ladder reads exactly (win probability in parentheses):

| Order | Stage label | Probability | Internal ID |
|---|---|---|---|
| 1 | New Prospect | 10% | (auto-generated) |
| 2 | Outreach Active | 20% | (auto-generated) |
| 3 | In Conversation | 40% | (auto-generated) |
| 4 | Meeting Booked | 60% | `appointmentscheduled` (rename existing) |
| 5 | Proposal Sent | 80% | `contractsent` (rename existing) |
| 6 | Closed Won | 100% (Won) | `closedwon` — DO NOT delete/recreate; rename only |
| 7 | Closed Lost | 0% (Lost) | `closedlost` — same |

3. Delete leftover default stages not in the table. Click **Save**.
4. Verify the Closed Won internal ID survived: run
   `curl -s -H "Authorization: Bearer $HUBSPOT_PRIVATE_APP_TOKEN" https://api.hubapi.com/crm/v3/pipelines/deals`
   (after 1c) and confirm a stage with `"id": "closedwon"`. If your portal shows a numeric ID
   instead, record that ID — it replaces `closedwon` in the PR-3 n8n filter and nothing else
   changes.

> Reminder (doc 01): cold outbound stays OUT of HubSpot; only positive responses are created
> here as deals.

### 1c. Private app + token

1. Settings → **Integrations → Private Apps** → **Create a private app**.
2. Basic info: name `Seekly Control Plane`, description "Provisioning trigger + summary-field
   sync for the Seekly platform."
3. **Scopes** tab — check exactly:
   - `crm.objects.deals.read`
   - `crm.objects.companies.read`
   - `crm.objects.companies.write` (needed by PR-6 to PATCH summary properties)
   - `crm.objects.contacts.read` (n8n reads the deal's primary contact for the portal invite)
   - `crm.schemas.companies.write` and `crm.schemas.deals.write` (needed once, by the PR-2
     property-creation script; may be removed after running it)
   - Webhooks are configured on the app itself (PR-3), no extra scope checkbox in current UI;
     if your portal shows a `webhooks` scope entry, check it.
4. **Create app** → confirm → **Show token** → copy the `pat-...` token.
5. Store it: env `HUBSPOT_PRIVATE_APP_TOKEN` in the app's production env AND as an n8n
   credential (Header Auth, `Authorization: Bearer pat-...`) — n8n fetches deal/company/contact
   in WF-0. Do not commit it anywhere.

### 1d. Custom DEAL properties (what n8n reads at Closed Won)

Doc 01: deals carry proposed package, monthly retainer, setup fee. Settings → **Properties**
→ object **Deal** → **Create property** ×3:

| Label | Internal name | Group | Type |
|---|---|---|---|
| Seekly Package | `seekly_package` | Deal information | Dropdown select: `pilot`, `core`, `growth`, `market_leader` (values lowercase exactly) |
| Seekly Monthly Retainer | `seekly_monthly_retainer` | Deal information | Number (currency formatting optional) — dollars |
| Seekly Setup Fee | `seekly_setup_fee` | Deal information | Number — dollars |

Make all three **required on the Closed Won stage**: Pipelines tab → Closed Won → "Update
stage properties" → add the three properties as required. A deal cannot be won without the
data provisioning needs.

### PR-1 acceptance criteria

1. `GET https://api.hubapi.com/crm/v3/pipelines/deals` with the token returns the 7 stages,
   Closed Won with `"metadata": {"isClosed": "true", "probability": "1.0"}` and its internal
   ID recorded in the ops checklist.
2. Token works: `curl -s -H "Authorization: Bearer $TOKEN" "https://api.hubapi.com/crm/v3/objects/deals?limit=1"` returns 200.
3. Moving a test deal to Closed Won in the UI forces the three `seekly_*` deal properties to
   be filled before saving.
4. Token present in production env as `HUBSPOT_PRIVATE_APP_TOKEN` and as an n8n credential;
   nowhere in git.

---

## PR-2 — Company summary properties (sync-back targets)

Five company properties receive the summary fields (doc 01: "HubSpot receives only summary
fields back"). Create them via the one-time script (preferred — reproducible), or the UI steps
below if the script can't run yet.

### 2a. One-time creation script — `scripts/hubspot-create-properties.ts` (new file)

Run: `npx tsx --env-file=.env scripts/hubspot-create-properties.ts`

```ts
/**
 * One-time setup: creates the "seekly" company property group and the five
 * summary properties PR-6 syncs into. Idempotent — 409 "already exists" is OK.
 * Requires HUBSPOT_PRIVATE_APP_TOKEN with crm.schemas.companies.write.
 */
import { env } from "../src/env";

const BASE = "https://api.hubapi.com";

async function hs(path: string, body: unknown): Promise<void> {
  const res = await fetch(`${BASE}${path}`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${env.HUBSPOT_PRIVATE_APP_TOKEN}`,
      "content-type": "application/json",
    },
    body: JSON.stringify(body),
  });
  if (res.status === 409) {
    console.log(`  already exists — OK`);
    return;
  }
  if (!res.ok) throw new Error(`${path} → ${res.status}: ${(await res.text()).slice(0, 300)}`);
  console.log(`  created`);
}

async function main() {
  if (!env.HUBSPOT_PRIVATE_APP_TOKEN) throw new Error("HUBSPOT_PRIVATE_APP_TOKEN is not set");

  console.log("group: seekly");
  await hs("/crm/v3/properties/companies/groups", {
    name: "seekly",
    label: "Seekly Platform",
    displayOrder: -1,
  });

  const enumProp = (name: string, label: string, options: string[]) => ({
    name,
    label,
    groupName: "seekly",
    type: "enumeration",
    fieldType: "select",
    options: options.map((o, i) => ({ label: o, value: o.toLowerCase().replace(/ /g, "_"), displayOrder: i })),
  });

  const props = [
    enumProp("seekly_onboarding_status", "Onboarding Status", ["Not Started", "In Progress", "Complete"]),
    enumProp("seekly_account_status", "Account Status", ["Onboarding", "Active", "Paused", "Churned"]),
    { name: "seekly_health_score", label: "Health Score", groupName: "seekly", type: "number", fieldType: "number" },
    { name: "seekly_mrr", label: "MRR", groupName: "seekly", type: "number", fieldType: "number" },
    { name: "seekly_renewal_date", label: "Renewal Date", groupName: "seekly", type: "date", fieldType: "date" },
  ];
  for (const p of props) {
    console.log(`property: ${p.name}`);
    await hs("/crm/v3/properties/companies", p);
  }
  console.log("done");
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Add npm script to `package.json`:
`"hubspot:setup-properties": "tsx --env-file=.env scripts/hubspot-create-properties.ts"`.

### 2b. Equivalent UI steps (fallback)

Settings → **Properties** → object **Company** → "Groups" tab → Create group `Seekly Platform`
(internal name `seekly`). Then Create property ×5 in that group with internal names/types
exactly as the table:

| Label | Internal name | Type |
|---|---|---|
| Onboarding Status | `seekly_onboarding_status` | Dropdown select: `not_started`, `in_progress`, `complete` |
| Account Status | `seekly_account_status` | Dropdown select: `onboarding`, `active`, `paused`, `churned` |
| Health Score | `seekly_health_score` | Number |
| MRR | `seekly_mrr` | Number |
| Renewal Date | `seekly_renewal_date` | Date picker |

### PR-2 acceptance criteria

1. Script run exits 0; second run also exits 0 (idempotent, 409s tolerated).
2. `GET /crm/v3/properties/companies/seekly_onboarding_status` returns the enumeration with the
   three values `not_started|in_progress|complete`.
3. All five properties visible on a company record under the "Seekly Platform" section.

---

## PR-3 — Webhook subscription: dealstage → n8n

n8n owns WF-0 orchestration (Spec 06). This ticket wires HubSpot → n8n and freezes the
n8n → app contract.

### 3a. HubSpot UI steps (private-app webhooks)

1. Settings → **Integrations → Private Apps** → open `Seekly Control Plane` → **Webhooks** tab.
2. **Target URL:** `https://<n8n host>/webhook/hubspot-deals` (the production n8n Webhook node
   URL from Spec 06's WF-0 export; use the `/webhook-test/...` URL only while building).
3. Click **Create subscription**:
   - Object type: **Deal**
   - Listen for: **Property changed**
   - Property: **dealstage**
4. Save → toggle the subscription **Active**.
5. Throttling: leave default (HubSpot batches events; n8n receives an ARRAY of event objects
   per POST — the workflow must iterate).
6. Test: move a sandbox deal into Closed Won; confirm the n8n execution list shows a run.

HubSpot delivers (shape n8n receives — array!):

```json
[
  {
    "eventId": 100,
    "subscriptionType": "deal.propertyChange",
    "objectId": 987654321,
    "propertyName": "dealstage",
    "propertyValue": "closedwon",
    "occurredAt": 1760000000000,
    "attemptNumber": 0
  }
]
```

### 3b. n8n WF-0 front-half contract (implemented in Spec 06 — recorded here as the interface)

1. Webhook node (POST, path `hubspot-deals`) → **respond immediately 200** (HubSpot retries
   non-2xx with backoff; never make it wait on provisioning).
2. Filter items: `subscriptionType == "deal.propertyChange" && propertyName == "dealstage" &&
   propertyValue == "closedwon"` (or the recorded numeric stage ID from PR-1 §1b.4).
3. HTTP node: `GET https://api.hubapi.com/crm/v3/objects/deals/{objectId}?properties=dealname,seekly_package,seekly_monthly_retainer,seekly_setup_fee,closedate&associations=companies,contacts`
   (Header Auth credential from PR-1).
4. HTTP nodes: fetch first associated company
   (`GET /crm/v3/objects/companies/{id}?properties=name,domain,website,industry,city,state`)
   and first associated contact (`GET /crm/v3/objects/contacts/{id}?properties=email,firstname,lastname`).
5. Build the provision payload (schema in PR-4) and
   `POST {APP_URL}/api/internal/provision` with header
   `Authorization: Bearer {INTERNAL_SERVICE_TOKEN}`. Retry ×3 exponential on 5xx/network;
   4xx → dead-letter + ops alert (n8n standard error branch, doc 02).
6. Continue WF-0 tail (Twilio subaccount, A2P submission, ops notification) — Spec 06.

### PR-3 acceptance criteria

1. Moving a deal to any stage other than Closed Won produces an n8n execution that filters to
   zero items and calls nothing.
2. Moving a deal to Closed Won results in exactly one `POST /api/internal/provision` with a
   payload passing the PR-4 zod schema (verify via n8n execution data + app logs).
3. HubSpot redelivery (replay the event from the private app's webhook **Monitoring** tab)
   results in a second POST that returns 200 with the SAME `runId`/`clientId` and creates no
   duplicate rows (proves end-to-end idempotency).
4. n8n responds 200 to HubSpot in <2 s (webhook Monitoring tab shows no failures/timeouts).

---

## PR-4 — `POST /api/internal/provision`

The app-side saga: creates everything the control plane owns. Idempotent on
`hubspot:{dealId}` / `manual:{externalRef}`; each step records completion in
`provisioningRuns.steps` so re-runs (n8n retry, HubSpot redelivery, ops "resume" click)
skip completed steps and only execute what's missing.

### 4a. Internal service auth — `src/server/internal/auth.ts` (new file)

```ts
import { timingSafeEqual } from "node:crypto";
import { env } from "@/env";

function tokensMatch(a: string, b: string): boolean {
  const bufA = Buffer.from(a);
  const bufB = Buffer.from(b);
  return bufA.length === bufB.length && timingSafeEqual(bufA, bufB);
}

/**
 * Guards /api/internal/* with INTERNAL_SERVICE_TOKEN (n8n's bearer token).
 * Same fail-closed pattern as src/server/spine/auth.ts: no token configured → 401 everything.
 * Returns a 401 Response, or null when authorized.
 */
export function requireInternalServiceToken(req: Request): Response | null {
  const expected = env.INTERNAL_SERVICE_TOKEN;
  const header = req.headers.get("authorization");
  const token = header?.startsWith("Bearer ") ? header.slice(7) : null;
  if (!expected || !token || !tokensMatch(token, expected)) {
    return Response.json({ error: "unauthorized" }, { status: 401 });
  }
  return null;
}
```

### 4b. Zod contract — `src/server/provisioning/contract.ts` (new file)

```ts
import { z } from "zod";

export const TIERS = ["pilot", "core", "growth", "market_leader"] as const;

export const provisionRequestSchema = z
  .object({
    source: z.enum(["hubspot", "manual"]),
    // Idempotency identifiers — exactly one applies per source.
    hubspotDealId: z.string().min(1).optional(),
    externalRef: z.string().min(1).optional(),        // e.g. "pilot-golf-1"
    hubspotCompanyId: z.string().min(1).optional(),   // enables PR-6 sync-back
    business: z.object({
      name: z.string().min(1),
      website: z.string().url(),
      industry: z.string().optional(),
      timezone: z.string().default("America/Toronto"),
      country: z.string().length(2).default("CA"),
      location: z.string().optional(),                // "Kingston, ON"
    }),
    contact: z.object({
      email: z.string().email(),
      name: z.string().optional(),
    }),
    package: z.object({
      tier: z.enum(TIERS),
      mrrCents: z.number().int().nonnegative().default(0),
      setupFeeCents: z.number().int().nonnegative().default(0),
      renewalDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
    }),
  })
  .superRefine((d, ctx) => {
    if (d.source === "hubspot" && !d.hubspotDealId) {
      ctx.addIssue({ code: "custom", path: ["hubspotDealId"], message: "required when source=hubspot" });
    }
    if (d.source === "manual" && !d.externalRef) {
      ctx.addIssue({ code: "custom", path: ["externalRef"], message: "required when source=manual" });
    }
  });

export type ProvisionRequest = z.infer<typeof provisionRequestSchema>;

export function idempotencyKeyFor(req: ProvisionRequest): string {
  return req.source === "hubspot" ? `hubspot:${req.hubspotDealId}` : `manual:${req.externalRef}`;
}

/** Ordered saga steps — the handler executes them in this order, skipping "done". */
export const PROVISION_STEPS = [
  "create_client",
  "client_profile",
  "workflow_config",
  "comms_provisioning",
  "onboarding_checklist",
  "portal_invite",
] as const;
export type ProvisionStep = (typeof PROVISION_STEPS)[number];
```

### 4c. Tier → module defaults + checklist — `src/server/provisioning/defaults.ts` (new file)

```ts
import type { ProvisionRequest } from "./contract";

/** All 8 switches exist for every client; tier decides which rows are created
 *  (a module without a row can never be flipped on — the switchboard shows it
 *  as "not in plan"). ALL created rows start enabled=false (doc 03 WF-0 step 5). */
const MODULES_BY_TIER: Record<ProvisionRequest["package"]["tier"], string[]> = {
  core: ["review_automation", "speed_to_lead", "intelligence"],
  growth: ["review_automation", "speed_to_lead", "intelligence", "reactivation", "content_engine", "social_syndication"],
  market_leader: [
    "review_automation", "speed_to_lead", "intelligence", "reactivation",
    "content_engine", "social_syndication", "directory_sync", "competitor_intel",
  ],
  pilot: [
    "review_automation", "speed_to_lead", "intelligence", "reactivation",
    "content_engine", "social_syndication", "directory_sync", "competitor_intel",
  ],
};

/** Per-module settings defaults, verbatim from doc 03 "Config (defaults)". */
export const DEFAULT_MODULE_SETTINGS: Record<string, Record<string, unknown>> = {
  review_automation: {
    sendDelayHours: 2, reAskCooldownDays: 90, dailySendCap: 25,
    approvalMode: "auto", negativeThreshold: 3,
    subSwitches: { review_requests: true, review_responses: true },
  },
  speed_to_lead: { followUpTouches: 5, followUpDays: 7, handoffKeywords: ["price", "complaint", "manager"] },
  reactivation: { lapseThresholdDays: 90, batchSizePerDay: 50, cadence: "monthly" },
  content_engine: { postsPerMonth: 2, approvalMode: "approve_first_3", schemaProfile: ["LocalBusiness", "Service", "FAQPage"] },
  directory_sync: { auditFrequency: "quarterly", autoPushOnProfileChange: true },
  competitor_intel: { cadence: "monthly" },
  social_syndication: { subSwitches: { gbp_posts: true, facebook_posts: true } },
  intelligence: { digestEnabled: false },
};

export function modulesForTier(tier: ProvisionRequest["package"]["tier"]): string[] {
  return MODULES_BY_TIER[tier];
}

interface ChecklistItem { key: string; label: string; blockingModule: string | null }

/** Onboarding checklist derived from tier (doc 08 §E — every box maps to a blocked module). */
export function checklistForTier(tier: ProvisionRequest["package"]["tier"]): ChecklistItem[] {
  const modules = new Set(modulesForTier(tier));
  const items: ChecklistItem[] = [
    { key: "intake_form", label: "Intake form completed (10–15 strategic questions)", blockingModule: null },
    { key: "a2p_ein", label: "EIN received → A2P brand/campaign submitted", blockingModule: "review_automation" },
    { key: "connect_google", label: "Google connected + GBP location selected", blockingModule: "review_automation" },
    { key: "pos_email_forwarding", label: "POS notification emails forwarding to Seekly inbox", blockingModule: "review_automation" },
    { key: "form_notifications", label: "Website form notifications CC'd to Seekly inbox", blockingModule: "speed_to_lead" },
  ];
  if (modules.has("speed_to_lead")) {
    items.push({ key: "connect_meta", label: "Facebook connected + Page selected (Lead Ads)", blockingModule: "speed_to_lead" });
  }
  if (modules.has("reactivation")) {
    items.push({ key: "customer_csv", label: "Customer CSV imported (phone + last visit)", blockingModule: "reactivation" });
  }
  if (modules.has("content_engine")) {
    items.push({ key: "publishing_path", label: "Website publishing path chosen (WordPress / email draft)", blockingModule: "content_engine" });
  }
  if (modules.has("competitor_intel")) {
    items.push({ key: "competitor_list", label: "Competitor list resolved (3–5, with GBP + website)", blockingModule: "competitor_intel" });
  }
  if (modules.has("directory_sync")) {
    items.push({ key: "brightlocal_location", label: "BrightLocal location created + baseline audit", blockingModule: "directory_sync" });
  }
  return items;
}
```

### 4d. The saga — `src/server/provisioning/provision.ts` (new file)

```ts
import { randomBytes } from "node:crypto";
import bcrypt from "bcryptjs";
import { eq, sql } from "drizzle-orm";
import { appUrl } from "@/env";
import { db } from "@/server/db";
import {
  clients, clientProfile, clientUsers, commsProvisioning,
  onboardingTasks, provisioningRuns, workflowConfig,
} from "@/server/db/schema";
import { sendInviteEmail } from "@/server/reports/email";
import { log, logError } from "@/lib/logger";
import {
  idempotencyKeyFor, PROVISION_STEPS, type ProvisionRequest, type ProvisionStep,
} from "./contract";
import { checklistForTier, DEFAULT_MODULE_SETTINGS, modulesForTier } from "./defaults";

type StepRecord = Partial<Record<ProvisionStep, { status: "done"; at: string; detail?: string }>>;

export interface ProvisionResult {
  runId: string;
  clientId: string | null;
  status: "completed" | "failed";
  steps: StepRecord;
  error?: string;
}

function slugify(name: string): string {
  return name.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "").slice(0, 40) || "client";
}

async function uniqueSlug(base: string): Promise<string> {
  for (let i = 0; i < 50; i++) {
    const candidate = i === 0 ? base : `${base}-${i + 1}`;
    const [hit] = await db.select({ id: clients.id }).from(clients).where(eq(clients.slug, candidate)).limit(1);
    if (!hit) return candidate;
  }
  return `${base}-${randomBytes(3).toString("hex")}`;
}

/**
 * Runs (or resumes) provisioning for the payload. Safe to call any number of
 * times with the same idempotency key: an advisory lock serializes concurrent
 * calls, and completed steps are skipped via the run's `steps` ledger.
 */
export async function provision(req: ProvisionRequest): Promise<ProvisionResult> {
  const key = idempotencyKeyFor(req);

  return db.transaction(async (tx) => {
    // Serialize concurrent provisions of the same deal (n8n retry racing a redelivery).
    await tx.execute(sql`SELECT pg_advisory_xact_lock(hashtext('provision'), hashtext(${key}))`);

    // Load-or-create the saga row.
    let [run] = await tx.select().from(provisioningRuns).where(eq(provisioningRuns.idempotencyKey, key)).limit(1);
    if (run?.status === "completed") {
      return { runId: run.id, clientId: run.clientId, status: "completed", steps: run.steps as StepRecord };
    }
    if (!run) {
      [run] = await tx
        .insert(provisioningRuns)
        .values({ idempotencyKey: key, source: req.source, payload: req, steps: {}, status: "running" })
        .returning();
    }

    const steps = { ...(run.steps as StepRecord) };
    let clientId = run.clientId;
    const markDone = async (step: ProvisionStep, detail?: string) => {
      steps[step] = { status: "done", at: new Date().toISOString(), ...(detail ? { detail } : {}) };
      await tx
        .update(provisioningRuns)
        .set({ steps, clientId, status: "running", error: null, updatedAt: new Date() })
        .where(eq(provisioningRuns.id, run!.id));
    };

    try {
      for (const step of PROVISION_STEPS) {
        if (steps[step]?.status === "done") continue; // ← re-runs skip completed steps

        switch (step) {
          case "create_client": {
            const slug = await uniqueSlug(slugify(req.business.name));
            const [row] = await tx
              .insert(clients)
              .values({
                name: req.business.name,
                slug,
                kind: "client",
                website: req.business.website,
                country: req.business.country,
                location: req.business.location ?? null,
                isActive: true,
                // Spec 01 columns:
                tier: req.package.tier,
                accountStatus: "onboarding",
                hubspotCompanyId: req.hubspotCompanyId ?? null,
                hubspotDealId: req.hubspotDealId ?? null,
                mrr: req.package.mrrCents,
                renewalAt: req.package.renewalDate ?? null,
                timezone: req.business.timezone,
                industry: req.business.industry ?? null,
              })
              .returning({ id: clients.id, slug: clients.slug });
            clientId = row.id;
            await markDone(step, `slug=${row.slug}`);
            break;
          }
          case "client_profile": {
            await tx
              .insert(clientProfile)
              .values({ clientId: clientId!, websiteUrl: req.business.website })
              .onConflictDoNothing();
            await markDone(step);
            break;
          }
          case "workflow_config": {
            const modules = modulesForTier(req.package.tier);
            await tx
              .insert(workflowConfig)
              .values(
                modules.map((module) => ({
                  clientId: clientId!,
                  module,
                  enabled: false, // doc 03 WF-0 step 5: ALL switches OFF until onboarding
                  settings: DEFAULT_MODULE_SETTINGS[module] ?? {},
                  updatedBy: "system:provision",
                  updatedAt: new Date(),
                })),
              )
              .onConflictDoNothing();
            await markDone(step, `modules=${modules.length}`);
            break;
          }
          case "comms_provisioning": {
            const [c] = await tx.select({ slug: clients.slug }).from(clients).where(eq(clients.id, clientId!)).limit(1);
            await tx
              .insert(commsProvisioning)
              .values({
                clientId: clientId!,
                a2pBrandStatus: "none",
                a2pCampaignStatus: "none",
                inboundEmailAddress: `${c.slug}@in.seekly.app`,
              })
              .onConflictDoNothing();
            await markDone(step);
            break;
          }
          case "onboarding_checklist": {
            await tx
              .insert(onboardingTasks)
              .values(
                checklistForTier(req.package.tier).map((t) => ({
                  clientId: clientId!,
                  key: t.key,
                  label: t.label,
                  blockingModule: t.blockingModule,
                })),
              )
              .onConflictDoNothing();
            await markDone(step);
            break;
          }
          case "portal_invite": {
            const detail = await createPortalInvite(tx, clientId!, req.contact.email, req.business.name);
            await markDone(step, detail);
            break;
          }
        }
      }

      await tx
        .update(provisioningRuns)
        .set({ status: "completed", clientId, steps, error: null, updatedAt: new Date() })
        .where(eq(provisioningRuns.id, run.id));
      log(`[provision] ${key} completed (client ${clientId})`);
      return { runId: run.id, clientId, status: "completed", steps };
    } catch (err) {
      const message = err instanceof Error ? err.message : String(err);
      logError(`[provision] ${key} failed`, err);
      await tx
        .update(provisioningRuns)
        .set({ status: "failed", clientId, steps, error: message, updatedAt: new Date() })
        .where(eq(provisioningRuns.id, run.id));
      return { runId: run.id, clientId, status: "failed", steps, error: message };
    }
  });
}

/**
 * Same mechanics as invitePortalUser in src/server/actions/portal-auth.ts
 * (single-use token, bcrypt hash, 7-day expiry, set-password link) but callable
 * without an admin session. REFACTOR NOTE: after landing this, change
 * invitePortalUser to delegate its token/insert/email block here so the logic
 * exists once.
 */
async function createPortalInvite(
  tx: Pick<typeof db, "select" | "insert" | "update">,
  clientId: string,
  email: string,
  clientName: string,
): Promise<string> {
  const normalizedEmail = email.trim().toLowerCase();
  const rawToken = randomBytes(32).toString("base64url");
  const inviteTokenHash = await bcrypt.hash(rawToken, 10);
  const inviteExpiresAt = new Date(Date.now() + 7 * 24 * 3600 * 1000);

  const [existing] = await tx.select().from(clientUsers).where(eq(clientUsers.email, normalizedEmail)).limit(1);
  if (existing && existing.clientId !== clientId) {
    throw new Error(`Contact email ${normalizedEmail} already belongs to another client's portal login.`);
  }

  let userId: string;
  if (existing) {
    userId = existing.id;
    await tx.update(clientUsers).set({ inviteTokenHash, inviteExpiresAt }).where(eq(clientUsers.id, userId));
  } else {
    const [inserted] = await tx
      .insert(clientUsers)
      .values({ clientId, email: normalizedEmail, status: "pending", inviteTokenHash, inviteExpiresAt })
      .returning({ id: clientUsers.id });
    userId = inserted.id;
  }

  const setPasswordUrl = `${appUrl}/portal/set-password?token=${rawToken}&uid=${userId}`;
  const result = await sendInviteEmail({ to: normalizedEmail, clientName, setPasswordUrl });
  if (!result.sent) {
    // Same behavior as invitePortalUser: log the URL so ops can hand-deliver.
    log(`[provision] invite email not sent — set-password URL: ${setPasswordUrl}`);
  }
  return `user=${userId} emailSent=${result.sent}`;
}
```

> Transaction note: the whole saga runs in one transaction *per invocation*, but progress is
> written via `markDone` inside it — if a later step throws, Postgres rolls the transaction
> back, so a failed run re-executes from its last **committed** run state (the previous
> invocation's ledger). That is exactly the resume semantics we want: an invocation either
> advances the ledger to `completed`/`failed`-with-fresh-error, or leaves it untouched.
> The one non-transactional side effect is `sendInviteEmail`; it is last in `PROVISION_STEPS`
> so a rollback can at worst re-send an invite email (harmless — same semantics as re-invite).

### 4e. Route — `src/app/api/internal/provision/route.ts` (new file)

```ts
import { requireInternalServiceToken } from "@/server/internal/auth";
import { provisionRequestSchema } from "@/server/provisioning/contract";
import { provision } from "@/server/provisioning/provision";

export async function POST(request: Request) {
  const unauth = requireInternalServiceToken(request);
  if (unauth) return unauth;

  let json: unknown;
  try {
    json = await request.json();
  } catch {
    return Response.json({ error: "invalid JSON" }, { status: 400 });
  }
  const parsed = provisionRequestSchema.safeParse(json);
  if (!parsed.success) {
    return Response.json(
      { error: "validation failed", issues: parsed.error.issues.map((i) => `${i.path.join(".")}: ${i.message}`) },
      { status: 422 },
    );
  }

  const result = await provision(parsed.data);
  // failed = 500 so n8n's retry/error branch fires; completed (fresh OR replay) = 200.
  return Response.json(result, { status: result.status === "completed" ? 200 : 500 });
}
```

### PR-4 acceptance criteria

1. No/wrong bearer token → 401 with no body leakage; empty `INTERNAL_SERVICE_TOKEN` env → 401
   (fail closed).
2. Malformed payload → 422 listing each zod issue path; `source=hubspot` without
   `hubspotDealId` → 422.
3. Happy path creates, in one call: `clients` row (`accountStatus='onboarding'`, tier, hubspot
   ids, unique slug), `client_profile` row, N `workflow_config` rows **all `enabled=false`**
   (N = 3 core / 6 growth / 8 market_leader / 8 pilot), `comms_provisioning` row with
   `inboundEmailAddress='<slug>@in.seekly.app'`, checklist rows per tier, and a `pending`
   `clientUsers` row; the invite email (or logged set-password URL) works end-to-end into
   `/portal/set-password`.
4. Same payload POSTed twice (sequentially): second response is 200 with identical
   `runId`+`clientId`; row counts in all six tables unchanged (test asserts counts).
5. Same payload POSTed twice **concurrently** (`Promise.all`): exactly one client row exists
   afterward (advisory lock proves out).
6. Kill a mid-saga step (temporarily make `sendInviteEmail` throw): run ends `failed` with
   `error` set; re-POST resumes, skips `create_client`…`onboarding_checklist` (their `steps`
   timestamps unchanged) and completes only the invite.
7. Two different businesses named "The Golf Shack" get slugs `the-golf-shack` and
   `the-golf-shack-2`.
8. A contact email already used by ANOTHER client's portal login fails the run with a clear
   error (never reassigns the login), and the first five steps remain completed for resume.

---

## PR-5 — Manual provisioning path (pilots, no HubSpot)

Same endpoint, same contract, `source: "manual"` + `externalRef`. Doc 03: "Also manually
invocable for pilots that never went through HubSpot."

### Script — `scripts/provision-client.ts` (new file)

Run: `npm run provision:client -- pilots/golf.json`
(add to `package.json`: `"provision:client": "tsx --env-file=.env scripts/provision-client.ts"`)

```ts
/**
 * Manual provisioning for pilots: POSTs a payload file to /api/internal/provision
 * on the running app (local or prod, per APP_URL) using INTERNAL_SERVICE_TOKEN.
 * Usage: npm run provision:client -- path/to/payload.json
 */
import { readFileSync } from "node:fs";
import { appUrl, env } from "../src/env";
import { provisionRequestSchema } from "../src/server/provisioning/contract";

async function main() {
  const file = process.argv[2];
  if (!file) throw new Error("usage: provision-client.ts <payload.json>");
  if (!env.INTERNAL_SERVICE_TOKEN) throw new Error("INTERNAL_SERVICE_TOKEN is not set");

  const parsed = provisionRequestSchema.safeParse(JSON.parse(readFileSync(file, "utf8")));
  if (!parsed.success) {
    console.error("payload invalid:");
    for (const i of parsed.error.issues) console.error(`  - ${i.path.join(".")}: ${i.message}`);
    process.exit(1);
  }

  const res = await fetch(`${appUrl}/api/internal/provision`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${env.INTERNAL_SERVICE_TOKEN}`,
      "content-type": "application/json",
    },
    body: JSON.stringify(parsed.data),
  });
  const body = await res.json();
  console.log(`HTTP ${res.status}`);
  console.log(JSON.stringify(body, null, 2));
  process.exit(res.ok ? 0 : 1);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Example payload file (`pilots/golf.json` — do not commit real contact data; keep `pilots/` in
`.gitignore`):

```json
{
  "source": "manual",
  "externalRef": "pilot-golf-1",
  "business": {
    "name": "Example Golf Sims",
    "website": "https://examplegolfsims.ca",
    "industry": "recreation",
    "timezone": "America/Toronto",
    "country": "CA",
    "location": "Kingston, ON"
  },
  "contact": { "email": "owner@examplegolfsims.ca", "name": "Pat Owner" },
  "package": { "tier": "pilot", "mrrCents": 0, "setupFeeCents": 0 }
}
```

### Admin UI note

Doc 05's admin "Intake runner" button ("create client manually, run WF-0") is a later admin
ticket: a small form in `(app)` that builds this same payload and calls `provision()` directly
(server action behind `requireSession()`), not via HTTP. Out of scope here; the script is the
pilot path.

### PR-5 acceptance criteria

1. `npm run provision:client -- pilots/golf.json` against a dev server provisions the pilot
   end-to-end (all PR-4 AC 3 artifacts) and prints the JSON result with `status: "completed"`.
2. Re-running the script is a no-op replay (same `runId`, exit 0).
3. An invalid payload file exits 1 with per-field messages BEFORE any network call.
4. A manual client has `hubspotCompanyId=null` and is silently skipped by the PR-6 sync job.

---

## PR-6 — Status sync-back → HubSpot company properties

Daily pg-boss job pushing the five summary fields to HubSpot (doc 01: "bidirectional event
sync, never database mirroring" — this is the only data that ever flows back).

### Queue wiring (`src/server/pipeline/queue.ts` + `src/worker.ts`, same pattern as Spec 04's `connectionsHealth`)

```ts
// QUEUES:
  hubspotSync: "hubspot-sync",
// RETRY_QUEUES:
  hubspotSync: { retryLimit: 2, retryDelay: 300, retryBackoff: true },
```

```ts
// src/worker.ts:
import { syncHubspotCompanies } from "./server/hubspot/sync";
// ...
await boss.work(QUEUES.hubspotSync, async () => {
  const r = await syncHubspotCompanies();
  log(`[worker] hubspot-sync: ${r.synced} companies updated, ${r.skipped} skipped, ${r.failed} failed`);
});
await boss.schedule(QUEUES.hubspotSync, "0 7 * * *"); // daily 07:00 UTC, after 06:30 health sweep
```

### Job — `src/server/hubspot/sync.ts` (new file)

```ts
import { eq, isNotNull, sql } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, onboardingTasks } from "@/server/db/schema";
import { env } from "@/env";
import { log, logError } from "@/lib/logger";

const BASE = "https://api.hubapi.com";

interface CompanyUpdate {
  id: string; // hubspotCompanyId
  properties: Record<string, string>;
}

/** "not_started" | "in_progress" | "complete" from the checklist rows. */
function onboardingStatus(total: number, done: number): string {
  if (total === 0 || done === 0) return "not_started";
  return done >= total ? "complete" : "in_progress";
}

export async function syncHubspotCompanies(): Promise<{ synced: number; skipped: number; failed: number }> {
  if (!env.HUBSPOT_PRIVATE_APP_TOKEN) {
    log("[hubspot-sync] HUBSPOT_PRIVATE_APP_TOKEN not set — skipping");
    return { synced: 0, skipped: 0, failed: 0 };
  }

  // One row per client with a HubSpot company: status fields + checklist rollup.
  const rows = await db
    .select({
      clientId: clients.id,
      hubspotCompanyId: clients.hubspotCompanyId,
      accountStatus: clients.accountStatus,
      mrr: clients.mrr,
      renewalAt: clients.renewalAt,
      tasksTotal: sql<number>`(select count(*)::int from ${onboardingTasks} t
         where t.client_id = ${clients.id} and t.status <> 'n_a')`,
      tasksDone: sql<number>`(select count(*)::int from ${onboardingTasks} t
         where t.client_id = ${clients.id} and t.status = 'done')`,
      // Health score lands with doc 09 (intelligence platform); the expression
      // below returns null until a metrics-backed score exists. Column contract:
      // 0–100 integer. Replace with the doc-09 rollup when it ships.
      healthScore: sql<number | null>`null::int`,
    })
    .from(clients)
    .where(isNotNull(clients.hubspotCompanyId));

  const updates: CompanyUpdate[] = rows.map((r) => {
    const properties: Record<string, string> = {
      seekly_onboarding_status: onboardingStatus(r.tasksTotal, r.tasksDone),
      seekly_account_status: r.accountStatus ?? "onboarding",
      seekly_mrr: String(((r.mrr ?? 0) / 100).toFixed(2)),
    };
    if (r.renewalAt) properties.seekly_renewal_date = r.renewalAt; // date prop: "YYYY-MM-DD"
    if (r.healthScore != null) properties.seekly_health_score = String(r.healthScore);
    return { id: r.hubspotCompanyId!, properties };
  });

  let synced = 0;
  let failed = 0;
  // Batch API: max 100 inputs per call.
  for (let i = 0; i < updates.length; i += 100) {
    const batch = updates.slice(i, i + 100);
    const res = await fetch(`${BASE}/crm/v3/objects/companies/batch/update`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${env.HUBSPOT_PRIVATE_APP_TOKEN}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({ inputs: batch }),
    });
    if (res.ok) {
      synced += batch.length;
      continue;
    }
    // Batch failed (e.g. one deleted company poisons it) → fall back to per-company
    // PATCH so one bad record can't block the rest.
    logError(`[hubspot-sync] batch update ${res.status}: ${(await res.text()).slice(0, 300)} — retrying individually`);
    for (const u of batch) {
      const single = await fetch(`${BASE}/crm/v3/objects/companies/${u.id}`, {
        method: "PATCH",
        headers: {
          authorization: `Bearer ${env.HUBSPOT_PRIVATE_APP_TOKEN}`,
          "content-type": "application/json",
        },
        body: JSON.stringify({ properties: u.properties }),
      });
      if (single.ok) synced++;
      else {
        failed++;
        logError(`[hubspot-sync] company ${u.id} → ${single.status}: ${(await single.text()).slice(0, 200)}`);
      }
    }
  }

  return { synced, skipped: 0, failed };
}
```

Notes:
- `seekly_renewal_date` is a HubSpot **date** property: send `"YYYY-MM-DD"` (v3 accepts it);
  never a datetime.
- MRR is stored in cents in our DB (Spec 01) and synced in dollars — HubSpot number props are
  display-facing.
- Health Score stays absent from the payload until the doc-09 intelligence rollup exists;
  HubSpot keeps the property blank rather than showing a fake 0.
- Onboarding completion is DERIVED from `onboardingTasks` — the same rows the portal
  checklist and admin switchboard update — so "Onboarding Status: Complete" in HubSpot always
  matches what the client sees (doc 05 step 5).

### PR-6 acceptance criteria

1. `hubspot-sync` scheduled at `0 7 * * *`; visible in `pgboss.schedule`.
2. For a provisioned test client (PR-4) with its `hubspotCompanyId` pointing at a sandbox
   company: after one job run the company shows Onboarding Status `Not started`, Account
   Status `Onboarding`, MRR `0`, Renewal Date blank, Health Score blank.
3. Marking all checklist rows `done` + setting `accountStatus='active'`, `mrr=99700`,
   `renewalAt='2026-10-01'` in the DB, then re-running: company shows `Complete`, `Active`,
   `997`, `2026-10-01`.
4. Clients with `hubspotCompanyId=null` (pilots) trigger zero HubSpot API calls.
5. One deleted/invalid company id does not block the rest (per-company fallback path covered
   by a test with a mocked fetch: batch 400 → individual PATCHes → `failed=1`, others synced).
6. Missing `HUBSPOT_PRIVATE_APP_TOKEN` → job logs a skip and completes (never crashes the worker).

---

## Out of scope (owned elsewhere)

- n8n WF-0 workflow JSON (webhook receipt, HubSpot fetches, Twilio subaccount + A2P
  submission, ops Slack/email notify, resume-link alerting) — Spec 06.
- `clients`/`client_profile`/`workflow_config`/`comms_provisioning` migrations — Spec 01
  (this spec owns only `provisioning_runs` + `onboarding_tasks`).
- Portal onboarding checklist UI + admin switchboard rendering of `onboardingTasks` /
  blocked-module reasons — portal specs.
- Twilio/A2P registration mechanics — comms spec (doc 06).
