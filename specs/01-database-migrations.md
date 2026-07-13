# Spec 01 — Database Migrations: Growth-Platform Schema

## Purpose

Add every control-plane table the Seekly growth platform needs (docs 04 + 09, as revised by
doc 10) to the **existing** `seekly-client-insights` app (Next.js 16 + Drizzle ORM + Postgres).
There is no new database and no Supabase: all tables below are appended to
`src/server/db/schema.ts` in that repo and shipped as ordinary drizzle-kit migrations.

Everything in this spec is implementable without touching n8n. Spec 02 (internal API) and
spec 03 (send pipeline) build on these tables.

**Repo:** `seekly-client-insights` (the app at `/workspace/seekly-client-insights` — paths
below are relative to that repo root unless absolute).

## Ticket breakdown

Each ticket = **one schema edit + one generated migration**, applied and verified before the
next ticket starts. Migrations are strictly ordered; the dependency column is a hard ordering
constraint (foreign keys reference tables created by earlier tickets).

| ID | Title | Depends on |
|-------|-------|------------|
| DB-1  | `clients` ALTERs: tier, account status, HubSpot id, industry, timezone, billing | — |
| DB-2  | `client_profile` + `workflow_config` (the switchboard) | DB-1 |
| DB-3  | `connections` (OAuth token vault) + `comms_provisioning` | DB-1 |
| DB-4  | `events` (canonical event store) + `customers` | DB-1 |
| DB-5  | `campaigns` + `campaign_members` | DB-4 |
| DB-6  | `opt_outs` + `messages` ledger + `conversations` (+ opt-out backfill & repoint) | DB-4, DB-5 |
| DB-7  | `content_items` + `syndicated_posts` | DB-1 |
| DB-8  | `competitor_snapshots` + `intel_digests` | DB-1 |
| DB-9  | `activity_log` + `metrics_daily` | DB-1 |
| DB-10 | Intelligence tables: `insights`, `client_briefs`, `experiments`, `playbooks`, `anomalies` | DB-9 |
| DB-11 | Extend `reviews` (AI response fields), `review_requests` (source event), `review_contacts` (customer link) | DB-4 |
| DB-12 | Seed script: demo growth tenant (`scripts/seed-growth.ts`) | DB-1..DB-11 |

## Migration workflow (identical for every ticket)

The repo's drizzle-kit workflow (from its README — do not invent a different one):

```bash
# 1. Edit src/server/db/schema.ts (append the ticket's code below).
npm run db:generate           # drizzle-kit generate → new SQL file in src/server/db/migrations/
# 2. READ the generated SQL. Confirm it only contains this ticket's DDL.
npm run db:migrate            # apply to the LOCAL dev DB (.env)
npx tsc --noEmit && npm test  # must stay green
# 3. Prod is applied later via `npm run db:migrate:prod` (worker also self-migrates on boot).
```

Rules:

1. **Never** edit an already-applied migration file. If a ticket's SQL is wrong, write a new
   migration.
2. Hand-written SQL (the DB-6 backfill) is **appended to that ticket's freshly generated,
   not-yet-applied migration file** using drizzle's statement separator
   (`--> statement-breakpoint`), before running `db:migrate`.
3. All new code goes at the **bottom** of `src/server/db/schema.ts`, in ticket order, under a
   banner comment per ticket (e.g. `// --- Growth platform (DB-2): profile + switchboard ---`).
4. Enum consts go immediately above the first table that uses them (existing file convention).

## Conventions (match `src/server/db/schema.ts` exactly)

- TS field names camelCase; column names snake_case: `clientId: uuid("client_id")`.
- Every tenant-scoped table: `clientId ... .references(() => clients.id, { onDelete: "cascade" })`.
- `uuid("id").primaryKey().defaultRandom()` for PKs (exception: `events.id`, see DB-4).
- Timestamps: `timestamp("x", { withTimezone: true })`; `createdAt ... .notNull().defaultNow()`.
- Closed vocabularies → `pgEnum` exported as `xxxEnum`; open/growing vocabularies → `text`
  with a documented registry (the file does this for `adapter`, `botId`, `sourceId`).
- Index array form: `(t) => [index("table_cols_idx").on(...), uniqueIndex(...)]`.
- Typed jsonb via `.$type<...>()` with a default.
- **This codebase does not use drizzle `relations()`** — relations are expressed as
  `.references()` FKs only. Do not add a `relations.ts`.
- JSDoc comments on every table and non-obvious column (the file is the documentation).

### Design decisions baked into this spec (do not re-litigate in code review)

1. **No `locations` table in v1.** Doc 04 sketched one; doc 10's authoritative new-table list
   omits it. Pilots are single-location; NAP already lives on `clients`
   (`napAddress`/`napPhone`/`gbpUrl`), and the remaining per-location fields (hours, GBP
   location id, BrightLocal id, listings health) go on `client_profile`. Multi-location is a
   v1.1 extraction.
2. **`opt_outs` stores `contactHash`, not raw phone.** Doc 04 wrote `phone`; the existing
   `reviewOptOuts` table deliberately stores only `sha256(normalized contact)` (no raw PII in
   the suppression list — see `src/server/reviews/compliance.ts` `hashContact`). We keep that
   idiom and generalize it cross-engine.
3. **`clients.status` from doc 04 becomes `clients.accountStatus`** — `clients` already has
   `kind` (client|prospect), `isActive`, and `deletedAt`; a bare `status` column would be
   ambiguous.
4. **`insights.embedding` is `jsonb` (number[]), not pgvector.** At pilot scale (≤ a few
   hundred ~200-token notes per client) in-process cosine similarity is fine and needs no
   Postgres extension. Upgrading to pgvector later is one migration; flagged as an open item.
5. **`engine` columns:** where the value set is exactly the 8 switchboard modules
   (`workflow_config.module`, `messages.engine`, `conversations.engine`) use the
   `workflowModuleEnum`; where the vocabulary is open (`activity_log.engine`,
   `metrics_daily.module` — provisioning, send_pipeline, intelligence jobs also log) use
   `text` with the registry documented in the column comment.
6. **Event type enum values contain dots** (`sale.completed`). Postgres enum labels may
   contain any characters; this matches the canonical wire vocabulary in doc 02 exactly so
   nothing ever translates event names.

---

## DB-1 — `clients` ALTERs

Add the growth-platform tenancy fields from docs 01/04 to the existing `clients` table.

### Schema changes

Add these enum consts **above** `export const clients = pgTable(...)` (they're used inside it):

```ts
/** Purchased package (docs/01). Null = legacy SoV-only client, not on the growth platform. */
export const clientTierEnum = pgEnum("client_tier", [
  "pilot",
  "core",
  "growth",
  "market_leader",
]);

/**
 * Growth-platform lifecycle (doc 04 `clients.status`). Distinct from `kind`
 * (client vs prospecting snapshot), `isActive` (SoV schedule participation) and
 * `deletedAt` (soft delete): this tracks the SERVICE relationship. WF-0
 * provisioning creates clients as 'onboarding'; existing rows default 'active'.
 */
export const accountStatusEnum = pgEnum("account_status", [
  "prospect",
  "onboarding",
  "active",
  "paused",
  "churned",
]);
```

Add these columns **inside** the `clients` table definition, after `reviewEngineConfig` and
before `createdAt`:

```ts
  // --- Growth platform (DB-1) ---
  /** Purchased package; null = not on the growth platform (legacy SoV-only). */
  tier: clientTierEnum("tier"),
  /** Service-relationship lifecycle; see accountStatusEnum doc. */
  accountStatus: accountStatusEnum("account_status").notNull().default("active"),
  /**
   * Bridge to Seekly's OWN sales CRM. The ONLY HubSpot linkage stored here —
   * summary fields sync back keyed on this (WF-0 step 8); operational data
   * never mirrors either way (doc 02 §design principle 6).
   */
  hubspotCompanyId: text("hubspot_company_id").unique(),
  /** Free-text vertical, e.g. "golf simulator venue", "hvac". Feeds niche playbooks (L4). */
  industry: text("industry"),
  /**
   * IANA timezone for quiet-hours enforcement and "client-local" scheduling
   * (doc 02 compliance). Replaces the coarse country→tz mapping for all
   * growth-engine sends; the legacy review engine keeps its own path until
   * spec 03 unifies them.
   */
  timezone: text("timezone").notNull().default("America/Toronto"),
  /** Free-text billing note: "pilot_free" | "paid" | "past_due" | ... (no enum — Stripe integration TBD). */
  billingStatus: text("billing_status"),
```

### Acceptance criteria

1. `npm run db:generate` produces exactly one new migration containing two `CREATE TYPE`
   statements and six `ALTER TABLE "clients" ADD COLUMN` statements (plus the unique
   constraint/index for `hubspot_company_id`); nothing else.
2. `npm run db:migrate` applies cleanly to a dev DB that already has all prior migrations.
3. Existing rows read back with `tier = NULL`, `accountStatus = 'active'`,
   `timezone = 'America/Toronto'`.
4. `npx tsc --noEmit` and `npm test` pass.
5. Inserting two clients with the same non-null `hubspotCompanyId` fails with a unique
   violation; two clients with NULL `hubspotCompanyId` both insert.

---

## DB-2 — `client_profile` + `workflow_config`

The intake output (doc 08 → doc 04 `client_profile`) and THE SWITCHBOARD (doc 04
`workflow_config`).

### Schema additions (append to bottom of `schema.ts`)

```ts
// --- Growth platform (DB-2): client profile + workflow switchboard ---

/**
 * The 8 switchboard modules — one master n8n workflow each (docs/03). This
 * enum is the closed vocabulary for workflow_config.module and for
 * messages/conversations.engine. Adding an engine = a migration, on purpose.
 */
export const workflowModuleEnum = pgEnum("workflow_module", [
  "review_automation",
  "speed_to_lead",
  "reactivation",
  "content_engine",
  "directory_sync",
  "competitor_intel",
  "social_syndication",
  "intelligence",
]);

/**
 * The "10–15 strategic questions" + intake output (docs/08) — everything only
 * the owner can tell us. One row per client, created empty at provisioning and
 * filled by intake. Single-location v1: the per-location fields (hours, GBP
 * location, BrightLocal) live here; a `locations` table is a v1.1 extraction.
 */
export const clientProfile = pgTable("client_profile", {
  clientId: uuid("client_id")
    .primaryKey()
    .references(() => clients.id, { onDelete: "cascade" }),
  /** Services as a customer would name them (intake Q7). */
  services: text("services").array().notNull().default([]),
  /** Neighborhoods/areas to own (intake Q8) — feeds keywords + content topics. */
  serviceAreas: text("service_areas").array().notNull().default([]),
  targetKeywords: text("target_keywords").array().notNull().default([]),
  /** 3–5 competitors (intake Q9). Resolved by ops: name + website + GBP place id + FB page. */
  competitors: jsonb("competitors")
    .$type<{ name: string; website?: string; placeId?: string; fbPage?: string }[]>()
    .notNull()
    .default([]),
  /** Tone, phrases to use/avoid, AI persona name, 3 example sentences (intake Q10). */
  brandVoice: jsonb("brand_voice")
    .$type<{
      tone?: string;
      personaName?: string;
      phrasesToUse?: string[];
      phrasesToAvoid?: string[];
      exampleSentences?: string[];
    }>()
    .notNull()
    .default({}),
  /**
   * Things the AI must NEVER promise — pricing, availability, guarantees
   * (intake Q11). Hard guardrails injected into every WF-2/WF-3 conversation
   * prompt.
   */
  aiGuardrails: text("ai_guardrails").array().notNull().default([]),
  /** Speed-to-lead qualification flow (intake Q12): ordered questions + hot-lead definition. */
  qualificationQuestions: jsonb("qualification_questions")
    .$type<{ question: string; key: string; hotSignal?: string }[]>()
    .notNull()
    .default([]),
  /** Client-approved win-back offers (intake Q13) — the only angles WF-3 may use. */
  campaignAngles: jsonb("campaign_angles")
    .$type<{ key: string; label: string; description: string; approved: boolean }[]>()
    .notNull()
    .default([]),
  bookingLink: text("booking_link"),
  websiteUrl: text("website_url"),
  /** "wordpress" | "wix" | "webflow" | "shopify" | "squarespace" | "custom" | "unknown" (intake Q4). */
  websitePlatform: text("website_platform"),
  /** Direct Google review link (GBP place-id short link) used in review-request SMS. */
  reviewLink: text("review_link"),
  /** Per-module approval modes (intake Q21), e.g. {"review_responses":"auto","campaigns":"approve_first","content":"approve_first_3"}. */
  approvalPreferences: jsonb("approval_preferences")
    .$type<Record<string, string>>()
    .notNull()
    .default({}),
  /** Hot-lead / negative-review alert recipients (intake Q23). */
  escalationContacts: jsonb("escalation_contacts")
    .$type<{ name: string; mobile?: string; email?: string; alerts: string[] }[]>()
    .notNull()
    .default([]),
  /** Weekly hours, {"mon":{"open":"09:00","close":"22:00"},...}; null = unknown. Feeds WF-5. */
  hoursJson: jsonb("hours_json").$type<Record<
    string,
    { open: string; close: string } | null
  > | null>(),
  holidayHoursJson: jsonb("holiday_hours_json"),
  /** GBP API location resource id ("locations/123..."), set when Google connects. */
  gbpLocationId: text("gbp_location_id"),
  /** Google Places place_id — public-data lookups (reviews interim path, WF-6). */
  placeId: text("place_id"),
  brightlocalLocationId: text("brightlocal_location_id"),
  /** 0–100 listings-consistency score from the latest BrightLocal audit. */
  listingsHealthScore: smallint("listings_health_score"),
  /** What a new customer is worth per year (intake Q15) — anchors revenue-recovered math. */
  avgCustomerValueUsd: numeric("avg_customer_value_usd", { precision: 10, scale: 2 }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

/**
 * THE SWITCHBOARD (docs/03, /04). One row per (client, module). `enabled` is
 * checked by every n8n master workflow at run start — flipping it is instant
 * and requires no deploy. `settings` carries every per-module default from
 * docs/03 as DATA (send delays, caps, cadences, thresholds…); code-side
 * defaults live in src/server/growth/module-defaults.ts (spec 03), which is
 * merged UNDER this jsonb at read time — a missing key never crashes a
 * workflow.
 */
export const workflowConfig = pgTable(
  "workflow_config",
  {
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    module: workflowModuleEnum("module").notNull(),
    enabled: boolean("enabled").notNull().default(false),
    settings: jsonb("settings")
      .$type<Record<string, unknown>>()
      .notNull()
      .default({}),
    /** Who flipped/edited last — admin email or "system:wf-0". Every flip is also activity-logged. */
    updatedBy: text("updated_by"),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [primaryKey({ columns: [t.clientId, t.module] })],
);
```

### Acceptance criteria

1. Migration creates `workflow_module` enum, `client_profile`, `workflow_config`; applies clean.
2. `client_profile.clientId` is both PK and FK; inserting a second row for the same client fails.
3. `workflow_config` PK is `(client_id, module)`; upserting the same pair twice with
   `onConflictDoUpdate` works (write a scratch script or use Drizzle Studio to verify).
4. Deleting a client cascades both tables' rows.
5. `npx tsc --noEmit` passes; all jsonb defaults type-check with the declared `$type`s.

---

## DB-3 — `connections` (OAuth token vault) + `comms_provisioning`

The Integration-Hub storage (doc 02 layer 2, doc 10 §5). Tokens are encrypted at the
**application layer** (AES-256-GCM, key only in the app's env — see spec 02 API-4 for
`src/server/connections/crypto.ts`); this ticket only creates the columns. The legacy
`clientCredentials` table stays for non-OAuth secrets (Bitwarden-backed) — do not touch it.

### Schema additions

```ts
// --- Growth platform (DB-3): OAuth token vault + comms provisioning ---

export const connectionStatusEnum = pgEnum("connection_status", [
  "pending",   // OAuth started / connection created, not yet completed+verified
  "active",    // tokens valid as of lastVerifiedAt
  "broken",    // refresh failed — dependent modules auto-pause (doc 02 health monitor)
  "revoked",   // client disconnected; tokens wiped
]);

/**
 * One authorized provider account per row — Seekly's OAuth app, client's
 * consent (doc 02 layer 2). `provider` is TEXT, not an enum: adapters
 * proliferate per the intake ladder (square, skedda, jobber…). Registry of
 * known values lives in src/server/connections/providers.ts (spec 02):
 *   "google" (one consent: GBP + GSC + GA4) | "meta" | "wordpress" |
 *   "brightlocal" | "hubspot" | future POS/booking providers.
 * Token columns hold AES-256-GCM sealed strings (crypto.ts), NEVER plaintext.
 * n8n receives only short-lived access tokens via the context endpoint —
 * refresh tokens never leave this table.
 */
export const connections = pgTable(
  "connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    provider: text("provider").notNull(),
    /** Provider-side account id (Google sub, Meta page id, WP user). Null until account selection. */
    providerAccountId: text("provider_account_id"),
    scopes: text("scopes").array().notNull().default([]),
    /** AES-256-GCM sealed access token (base64 iv‖tag‖ciphertext); null when expired-and-unrefreshed. */
    encryptedAccessToken: text("encrypted_access_token"),
    /** AES-256-GCM sealed refresh token. Null for providers without refresh (WP app-password lives in encryptedAccessToken). */
    encryptedRefreshToken: text("encrypted_refresh_token"),
    /** Access-token expiry; the vault refreshes when < 5 min away (spec 02 API-4). */
    expiresAt: timestamp("expires_at", { withTimezone: true }),
    status: connectionStatusEnum("status").notNull().default("pending"),
    lastVerifiedAt: timestamp("last_verified_at", { withTimezone: true }),
    /** Last refresh/health-check error, for the portal "Reconnect Google" banner. */
    lastError: text("last_error"),
    /** Provider-specific selections: {"gbpLocationId":...}, {"pageId":...,"pageName":...}, {"siteUrl":...}. */
    metadata: jsonb("metadata").$type<Record<string, unknown>>().notNull().default({}),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    // NULLS NOT DISTINCT: at most ONE pending (null-account) connection per
    // (client, provider) — an abandoned OAuth restart upserts instead of piling up.
    unique("connections_client_provider_account_unique")
      .on(t.clientId, t.provider, t.providerAccountId)
      .nullsNotDistinct(),
    index("connections_client_idx").on(t.clientId),
    index("connections_status_idx").on(t.status),
  ],
);

/** A2P registration lifecycle — both brand and campaign use it (docs/06 Twilio). */
export const a2pStatusEnum = pgEnum("a2p_status", [
  "none",
  "submitted",
  "approved",
  "rejected",
]);

/**
 * Per-client comms infrastructure created by WF-0 step 6 (doc 03): Twilio
 * subaccount + number + A2P 10DLC registration + the email-parse inbound
 * address. The send pipeline HARD-GATES on a2pCampaignStatus='approved'
 * (doc 06 — no sends before campaign approval, ever). The Twilio subaccount
 * AUTH TOKEN is NOT stored here or anywhere in this DB — it stays in the
 * platform Twilio account, addressed by SID from the master credentials.
 */
export const commsProvisioning = pgTable("comms_provisioning", {
  clientId: uuid("client_id")
    .primaryKey()
    .references(() => clients.id, { onDelete: "cascade" }),
  twilioSubaccountSid: text("twilio_subaccount_sid"),
  /** E.164, e.g. "+16135550142" — the client's one local number (docs/06). */
  phoneNumber: text("phone_number"),
  messagingServiceSid: text("messaging_service_sid"),
  a2pBrandStatus: a2pStatusEnum("a2p_brand_status").notNull().default("none"),
  a2pCampaignStatus: a2pStatusEnum("a2p_campaign_status").notNull().default("none"),
  /** Twilio use-case, e.g. "MIXED" (marketing + customer care). */
  a2pCampaignType: text("a2p_campaign_type"),
  /** Per-client email-parse inbox, "{slug}@in.seekly.app" (doc 06). */
  inboundEmailAddress: text("inbound_email_address").unique(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

### Acceptance criteria

1. Migration creates both tables + both enums; applies clean.
2. Two `connections` rows with the same `(clientId, provider)` and NULL `providerAccountId`
   violate the unique constraint (verify NULLS NOT DISTINCT made it into the SQL: the
   generated constraint must say `NULLS NOT DISTINCT`).
3. Same `(clientId, provider)` with two **different** non-null `providerAccountId`s both
   insert (multi-page Meta support).
4. `comms_provisioning` allows exactly one row per client (PK) and enforces global uniqueness
   of `inbound_email_address`.
5. No plaintext-token column exists (`\d connections` shows only `encrypted_*` token columns).

---

## DB-4 — `events` (canonical event store) + `customers`

The canonical event keystone (doc 02 layer 3) and the client's-customers table (reactivation
audience, doc 04).

### Schema additions

```ts
// --- Growth platform (DB-4): canonical events + customers ---

/**
 * The frozen canonical vocabulary (doc 02 layer 3). Values intentionally
 * contain dots — they ARE the wire `type` strings; nothing ever translates
 * event names. Adding an event type = a migration + explicit engine work.
 */
export const eventTypeEnum = pgEnum("event_type", [
  "sale.completed",
  "lead.created",
  "message.received",
  "call.missed",
  "review.received",
  "customer.imported",
  "content.published",
  "profile.updated",
  "client.provisioned",
]);

export const eventStatusEnum = pgEnum("event_status", [
  "pending",    // persisted; not yet handed to n8n
  "processed",  // forwarded to (and 2xx-acked by) the n8n webhook
  "failed",     // forwarding failed after the attempt — replayable
  "ignored",    // consciously dropped (e.g. module off and event type is skippable)
]);

/**
 * Canonical event store — every adapter POSTs here via /api/internal/events
 * (spec 02) and every engine consumes from n8n, never vendor payloads.
 *
 * `id` IS the adapter-supplied `event_id` idempotency key (doc 02 envelope):
 * NO defaultRandom — the caller must supply it, and insert-on-conflict on this
 * PK makes redelivery a no-op. `payload` stores the envelope payload verbatim
 * (unknown fields preserved for later adapter improvements — doc 02 rules).
 */
export const events = pgTable(
  "events",
  {
    /** Adapter-supplied idempotency key (uuid). Deliberately no default. */
    id: uuid("id").primaryKey(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    type: eventTypeEnum("type").notNull(),
    /**
     * Adapter registry id: "square" | "email-parse" | "webhook" | "csv" |
     * "manual" | "gbp-poll" | "meta" | "twilio" | "portal" | "hubspot".
     * Open vocabulary (adapter ladder) — text, not enum.
     */
    source: text("source").notNull(),
    /** When the real-world thing happened (adapter clock). */
    occurredAt: timestamp("occurred_at", { withTimezone: true }).notNull(),
    receivedAt: timestamp("received_at", { withTimezone: true }).notNull().defaultNow(),
    payload: jsonb("payload").$type<Record<string, unknown>>().notNull(),
    status: eventStatusEnum("status").notNull().default("pending"),
    processedAt: timestamp("processed_at", { withTimezone: true }),
    error: text("error"),
  },
  (t) => [
    index("events_client_received_idx").on(t.clientId, t.receivedAt),
    index("events_client_type_occurred_idx").on(t.clientId, t.type, t.occurredAt),
    // The ops replay sweep scans only stuck rows — partial index keeps it off the table.
    index("events_replayable_idx")
      .on(t.status, t.receivedAt)
      .where(sql`${t.status} in ('pending', 'failed')`),
  ],
);

/** Recorded consent basis per customer (doc 02 compliance): reactivation sends
 *  only to customers with a prior relationship. */
export const consentBasisEnum = pgEnum("consent_basis", [
  "existing_customer",
  "lead_inbound",
  "imported_list",
]);

/**
 * The CLIENT'S customers (doc 04) — the reactivation audience. Populated by
 * `sale.completed` / `customer.imported` events and CSV upload; NEVER mixed
 * with Seekly's own CRM data (that lives in HubSpot). Distinct from
 * reviewContacts (a send-funnel working table): this is the canonical person
 * record; DB-11 links reviewContacts here.
 */
export const customers = pgTable(
  "customers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    name: text("name"),
    /** E.164-normalized at write time (spec 02/03 normalize before insert). */
    phone: text("phone"),
    email: text("email"),
    firstSeenAt: timestamp("first_seen_at", { withTimezone: true }).notNull().defaultNow(),
    /** Drives reactivation segmentation ("lapsed > 90d"). Updated on every sale.completed. */
    lastVisitAt: timestamp("last_visit_at", { withTimezone: true }),
    visitCount: integer("visit_count").notNull().default(0),
    lifetimeValueEstimate: numeric("lifetime_value_estimate", { precision: 10, scale: 2 }),
    consentBasis: consentBasisEnum("consent_basis"),
    /** Which adapter/path created the record: "csv" | "square" | "email-parse" | "manual" | ... */
    source: text("source"),
    tags: text("tags").array().notNull().default([]),
    /** POS/booking-system ids, e.g. {"square":"CUST_123"} — dedupe key for native adapters. */
    externalIds: jsonb("external_ids").$type<Record<string, string>>().notNull().default({}),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    // Doc 04: unique (client_id, phone). Nulls are distinct — phone-less
    // records (email-only imports) never collide.
    uniqueIndex("customers_client_phone_unique").on(t.clientId, t.phone),
    index("customers_client_email_idx").on(t.clientId, t.email),
    index("customers_client_last_visit_idx").on(t.clientId, t.lastVisitAt),
  ],
);
```

### Acceptance criteria

1. Migration creates 3 enums + 2 tables; applies clean.
2. `events.id` has **no default**: an insert without an explicit id fails (verify in a scratch
   script), and inserting the same id twice with `.onConflictDoNothing({ target: events.id })`
   leaves exactly one row.
3. Generated SQL contains the partial index `events_replayable_idx ... WHERE ... in ('pending', 'failed')`.
4. Two customers with the same `(client_id, phone)` fail; two with NULL phone both insert.
5. `payload` round-trips unknown keys verbatim (insert `{"weird_field":1}`, read it back).

---

## DB-5 — `campaigns` + `campaign_members`

Reactivation campaigns (WF-3) with the resumable per-member send ledger (doc 04) — a crash
mid-batch never re-texts anyone.

### Schema additions

```ts
// --- Growth platform (DB-5): reactivation campaigns ---

export const campaignStatusEnum = pgEnum("campaign_status", [
  "draft",
  "approved",   // copy approved (portal) — first-ever campaign for a client always requires this
  "sending",
  "paused",     // switch flipped off / STOP-rate auto-pause (>3% of a batch, doc 03)
  "done",
]);

export const campaignMemberStatusEnum = pgEnum("campaign_member_status", [
  "queued",
  "sent",
  "replied",
  "booked",
  "excluded",   // opt-out / freq cap / no consent at selection time
]);

/** One win-back campaign (doc 04). `stats` is a denormalized rollup the worker
 *  refreshes; the message ledger stays the source of truth. */
export const campaigns = pgTable(
  "campaigns",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    /** The clientProfile.campaignAngles key this campaign uses (client-approved only). */
    angle: text("angle").notNull(),
    /** AI-generated message variants, [{key, body}] — personalization happens per member at send. */
    copyVariants: jsonb("copy_variants")
      .$type<{ key: string; body: string }[]>()
      .notNull()
      .default([]),
    /** Segment definition, e.g. {"lapsedDays":90,"minVisits":1,"tags":[]}. */
    audienceFilter: jsonb("audience_filter")
      .$type<Record<string, unknown>>()
      .notNull()
      .default({}),
    status: campaignStatusEnum("status").notNull().default("draft"),
    stats: jsonb("stats")
      .$type<{ sent: number; replies: number; stops: number; bookings: number; revenueEst: number }>()
      .notNull()
      .default({ sent: 0, replies: 0, stops: 0, bookings: 0, revenueEst: 0 }),
    approvedBy: text("approved_by"),
    approvedAt: timestamp("approved_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("campaigns_client_idx").on(t.clientId, t.createdAt)],
);

/**
 * The RESUME LEDGER (doc 04): composite PK means a customer can be in a
 * campaign exactly once; WF-3 batches select `queued` members and mark `sent`
 * transactionally with the ledger write, so re-runs skip everyone already
 * touched.
 */
export const campaignMembers = pgTable(
  "campaign_members",
  {
    campaignId: uuid("campaign_id")
      .notNull()
      .references(() => campaigns.id, { onDelete: "cascade" }),
    customerId: uuid("customer_id")
      .notNull()
      .references(() => customers.id, { onDelete: "cascade" }),
    status: campaignMemberStatusEnum("status").notNull().default("queued"),
    /** Why excluded, when status='excluded': "opt_out" | "freq_cap" | "no_consent" | "no_phone". */
    excludedReason: text("excluded_reason"),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    repliedAt: timestamp("replied_at", { withTimezone: true }),
  },
  (t) => [
    primaryKey({ columns: [t.campaignId, t.customerId] }),
    index("campaign_members_status_idx").on(t.campaignId, t.status),
  ],
);
```

### Acceptance criteria

1. Migration creates 2 enums + 2 tables; applies clean.
2. Inserting the same `(campaignId, customerId)` twice fails (PK).
3. `stats` default reads back as the zeroed object.
4. Deleting a campaign cascades its members; deleting a customer cascades their memberships.

---

## DB-6 — `opt_outs` + `messages` ledger + `conversations` (+ backfill)

The cross-engine compliance core (doc 02): ONE opt-out ledger spanning all engines, the
idempotent message ledger, and speed-to-lead/reactivation conversation threads. This ticket
also **generalizes `reviewOptOuts`**: backfills its rows and repoints the review engine's
suppression reads/writes.

### Schema additions

```ts
// --- Growth platform (DB-6): opt-outs, message ledger, conversations ---

/** Communication channel for opt-outs and (later) multichannel sends. */
export const commChannelEnum = pgEnum("comm_channel", ["email", "sms"]);

/**
 * THE cross-engine suppression ledger (doc 02): a STOP instantly blocks the
 * customer across review requests, lead follow-ups, and reactivation.
 * Generalizes reviewOptOuts (backfilled by this ticket's migration; the old
 * table is frozen read-only and dropped in a later cleanup).
 *
 * Same privacy idiom as reviewOptOuts: `contactHash` = sha256 of the
 * normalized contact (hashContact() in src/server/reviews/compliance.ts) — no
 * raw PII in the suppression list. `clientId` NULL = GLOBAL suppression
 * (legacy shared-sender STOPs); the send-pipeline check matches
 * (channel, contactHash) where clientId IS NULL OR clientId = current.
 */
export const optOuts = pgTable(
  "opt_outs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").references(() => clients.id, { onDelete: "cascade" }),
    channel: commChannelEnum("channel").notNull(),
    contactHash: text("contact_hash").notNull(),
    /** "sms_stop" | "email_unsubscribe" | "manual" | "review_opt_outs_migration". */
    source: text("source").notNull().default("manual"),
    /** The inbound STOP message that created this row, when known. */
    sourceMessageId: uuid("source_message_id"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    // NULLS NOT DISTINCT: one global row + at most one per client per contact.
    unique("opt_outs_scope_unique")
      .on(t.clientId, t.channel, t.contactHash)
      .nullsNotDistinct(),
    index("opt_outs_lookup_idx").on(t.channel, t.contactHash),
  ],
);

export const messageDirectionEnum = pgEnum("message_direction", ["in", "out"]);
/** Doc 02: transactional/conversational are exempt from quiet hours + freq caps; marketing is not. */
export const messageKindEnum = pgEnum("message_kind", [
  "transactional",
  "conversational",
  "marketing",
]);
export const messageStatusEnum = pgEnum("message_status", [
  "queued",     // accepted; waiting (quiet hours window / batch scheduler)
  "sent",       // handed to Twilio
  "delivered",  // Twilio delivery callback
  "failed",     // Twilio error after send attempt
  "blocked",    // compliance gate refused — see blockedReason
]);
export const blockedReasonEnum = pgEnum("blocked_reason", [
  "opt_out",
  "quiet_hours",     // recorded when a queued send is CANCELLED at window edge, not for normal deferral
  "freq_cap",
  "a2p_pending",
  "no_consent",
  "no_destination",
  "module_disabled",
]);

/**
 * The message ledger (doc 04) — every SMS in or out, all engines. The
 * idempotency contract that makes "a crash never double-texts" true:
 * (clientId, idempotencyKey) is unique, and the send pipeline WRITES THE
 * LEDGER ROW BEFORE calling Twilio (doc 03 send-pipeline step 5).
 */
export const messages = pgTable(
  "messages",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    /** Null for inbound from unknown numbers (matched lazily by phone). */
    customerId: uuid("customer_id").references(() => customers.id, {
      onDelete: "set null",
    }),
    direction: messageDirectionEnum("direction").notNull(),
    kind: messageKindEnum("kind").notNull(),
    /** Which engine sent/owns it (closed set — the switchboard modules). */
    engine: workflowModuleEnum("engine").notNull(),
    /** Counterparty number, E.164 (customer side, both directions). */
    phone: text("phone").notNull(),
    body: text("body").notNull(),
    mediaUrls: text("media_urls").array().notNull().default([]),
    twilioSid: text("twilio_sid"),
    status: messageStatusEnum("status").notNull(),
    blockedReason: blockedReasonEnum("blocked_reason"),
    /**
     * Caller-supplied, e.g. "{event_id}:review_request" or
     * "{campaign_id}:{customer_id}:touch1". Unique per client — the
     * duplicate-send backstop. Required for direction='out'; null for inbound.
     */
    idempotencyKey: text("idempotency_key"),
    campaignId: uuid("campaign_id").references(() => campaigns.id, {
      onDelete: "set null",
    }),
    conversationId: uuid("conversation_id"),
    /** When status='queued' for quiet hours: the earliest allowed send time. */
    queuedUntil: timestamp("queued_until", { withTimezone: true }),
    error: text("error"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    deliveredAt: timestamp("delivered_at", { withTimezone: true }),
  },
  (t) => [
    uniqueIndex("messages_client_idempotency_unique").on(t.clientId, t.idempotencyKey),
    index("messages_client_ts_idx").on(t.clientId, t.createdAt),
    index("messages_conversation_idx").on(t.conversationId),
    index("messages_campaign_idx").on(t.campaignId),
    index("messages_twilio_sid_idx").on(t.twilioSid),
    // Frequency-cap check: "marketing sends to this customer in the last N days".
    index("messages_freq_cap_idx")
      .on(t.clientId, t.customerId, t.createdAt)
      .where(sql`${t.kind} = 'marketing' and ${t.direction} = 'out'`),
    // Quiet-hours release sweep.
    index("messages_queued_idx")
      .on(t.queuedUntil)
      .where(sql`${t.status} = 'queued'`),
  ],
);

export const conversationStatusEnum = pgEnum("conversation_status", [
  "active",
  "qualified",
  "booked",
  "handed_off",
  "cold",
  "closed",
]);

/**
 * A speed-to-lead / reactivation thread (doc 04). The conversation router
 * (message.received → WF-2/WF-3) attaches inbound messages to the single
 * active thread per (client, customer) — enforced by the partial unique below,
 * same pattern as runs_one_active_per_client_idx.
 */
export const conversations = pgTable(
  "conversations",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    customerId: uuid("customer_id")
      .notNull()
      .references(() => customers.id, { onDelete: "cascade" }),
    engine: workflowModuleEnum("engine").notNull(),
    status: conversationStatusEnum("status").notNull().default("active"),
    /** "website_form" | "meta_lead_ad" | "missed_call" | "reactivation_reply" | ... */
    leadSource: text("lead_source"),
    /** The lead.created / message.received event that opened the thread. */
    leadEventId: uuid("lead_event_id").references(() => events.id, {
      onDelete: "set null",
    }),
    /** Extracted intent + qualification answers, keyed by clientProfile.qualificationQuestions[].key. */
    context: jsonb("context").$type<Record<string, unknown>>().notNull().default({}),
    /** Lead-event → first outbound SMS latency — THE hero metric (target < 60). */
    firstResponseSeconds: integer("first_response_seconds"),
    outcome: text("outcome"),
    booked: boolean("booked").notNull().default(false),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    closedAt: timestamp("closed_at", { withTimezone: true }),
  },
  (t) => [
    index("conversations_client_ts_idx").on(t.clientId, t.createdAt),
    index("conversations_client_status_idx").on(t.clientId, t.status),
    uniqueIndex("conversations_one_active_per_customer_idx")
      .on(t.clientId, t.customerId)
      .where(sql`${t.status} = 'active'`),
  ],
);
```

> **Note — `messages.conversationId` has no `.references()`:** `conversations` also has no FK
> to `messages`; the conversation FK is left as a plain uuid to avoid a circular-creation
> headache inside one migration and because inbound messages can arrive before their thread
> exists. This mirrors doc 04 ("conversation_id fk null") loosely; integrity is enforced by
> the router code (spec 03). Same for `optOuts.sourceMessageId`.

### Hand-written backfill (append to THIS ticket's generated migration file)

After `npm run db:generate`, append to the new SQL file (keep drizzle's breakpoint comment):

```sql
--> statement-breakpoint
INSERT INTO "opt_outs" ("id", "client_id", "channel", "contact_hash", "source", "created_at")
SELECT "id", "client_id", "channel"::text::"comm_channel", "contact_hash",
       'review_opt_outs_migration', "created_at"
FROM "review_opt_outs"
ON CONFLICT DO NOTHING;
```

(`review_request_channel` and `comm_channel` have identical labels `email`/`sms`; the
`::text::"comm_channel"` double-cast converts between the enum types.)

### Code repoint (same ticket, same PR)

In `src/server/reviews/engine.ts`, repoint the three suppression touchpoints from
`reviewOptOuts` to `optOuts` (imports change from `reviewOptOuts` to `optOuts`):

1. The suppression **check** (the `select` at ~line 100): query `optOuts` on
   `(channel, contactHash)` — clientId-agnostic match preserves today's global-STOP behavior.
2. `recordOptOut` (~line 203): insert into `optOuts` with `clientId: null`,
   `source: "sms_stop"` (SMS path) / `"email_unsubscribe"` (unsubscribe path),
   `onConflictDoNothing({ target: [optOuts.clientId, optOuts.channel, optOuts.contactHash] })`.
3. `removeOptOut` (~line 213): delete from `optOuts` by `(channel, contactHash)`.

Do **not** drop `review_opt_outs` in this ticket — it stays frozen (no more writes) as a
rollback safety net; a later cleanup migration removes it.

### Acceptance criteria

1. Migration creates 6 enums + 3 tables, then backfills: after `db:migrate`,
   `select count(*) from opt_outs where source='review_opt_outs_migration'` equals
   `select count(*) from review_opt_outs`.
2. Re-running the backfill INSERT manually inserts 0 rows (ON CONFLICT holds).
3. Texting STOP to the inbound webhook (`/api/reviews/sms/inbound`) creates an `opt_outs` row
   (clientId NULL, channel sms) and NO new `review_opt_outs` row; the review scheduler then
   suppresses that contact (existing `compliance.test.ts` + a new engine test cover this).
4. Two outbound messages with the same `(clientId, idempotencyKey)` — second insert conflicts.
5. Two `active` conversations for the same `(clientId, customerId)` — second insert fails;
   closing the first (status→`closed`) lets a new active one insert.
6. Partial indexes `messages_freq_cap_idx`, `messages_queued_idx`,
   `conversations_one_active_per_customer_idx` appear in the generated SQL with their WHERE
   clauses.
7. `npm test` green (update any suppression tests that referenced `reviewOptOuts`).

---

## DB-7 — `content_items` + `syndicated_posts`

WF-4 content engine output and WF-7 syndication ledger (doc 04).

### Schema additions

```ts
// --- Growth platform (DB-7): content + syndication ---

export const contentTypeEnum = pgEnum("content_item_type", ["blog"]);
export const contentStatusEnum = pgEnum("content_status", [
  "queued",            // topic picked, not yet drafted
  "drafted",
  "pending_approval",
  "approved",
  "published",
  "failed",            // publish failed after retries — parked with ops alert, never silently lost
]);

/** A GEO/SEO content piece (doc 03 WF-4): TL;DR + FAQ + JSON-LD pattern. */
export const contentItems = pgTable(
  "content_items",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    type: contentTypeEnum("type").notNull().default("blog"),
    topic: text("topic").notNull(),
    title: text("title"),
    bodyHtml: text("body_html"),
    /** LocalBusiness/Service/FAQPage JSON-LD blocks; validated before publish (quality gate). */
    schemaJsonld: jsonb("schema_jsonld"),
    status: contentStatusEnum("status").notNull().default("queued"),
    /** "wordpress" | "email_draft" | per-intake platform — copied from settings at draft time. */
    publishTarget: text("publish_target"),
    publishedUrl: text("published_url"),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    /** GSC clicks/impressions, ranking spot-checks — refreshed by the tracking job. */
    metrics: jsonb("metrics").$type<Record<string, unknown>>().notNull().default({}),
    error: text("error"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    index("content_items_client_status_idx").on(t.clientId, t.status),
    index("content_items_client_ts_idx").on(t.clientId, t.createdAt),
  ],
);

export const syndicationChannelEnum = pgEnum("syndication_channel", [
  "gbp",
  "facebook",
]);
export const syndicationStatusEnum = pgEnum("syndication_status", [
  "queued",
  "scheduled",
  "posted",
  "failed",     // per-channel isolation: a facebook failure never blocks the gbp post (doc 03 WF-7)
]);

/** One channel-native cut of a content item (doc 04). */
export const syndicatedPosts = pgTable(
  "syndicated_posts",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    /** Null when syndicating a client's own blog post discovered via RSS (no contentItems row). */
    contentItemId: uuid("content_item_id").references(() => contentItems.id, {
      onDelete: "cascade",
    }),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    channel: syndicationChannelEnum("channel").notNull(),
    /** The source URL for RSS-discovered posts (idempotency key component when contentItemId is null). */
    sourceUrl: text("source_url"),
    body: text("body").notNull(),
    /** CTA link with UTM params — attribution into the Visibility dashboard. */
    ctaUrlUtm: text("cta_url_utm"),
    status: syndicationStatusEnum("status").notNull().default("queued"),
    scheduledAt: timestamp("scheduled_at", { withTimezone: true }),
    postedAt: timestamp("posted_at", { withTimezone: true }),
    externalPostId: text("external_post_id"),
    clicks: integer("clicks").notNull().default(0),
    error: text("error"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    // One post per channel per content item — replaying content.published is a no-op.
    unique("syndicated_posts_item_channel_unique")
      .on(t.contentItemId, t.channel)
      .nullsNotDistinct(),
    // RSS path idempotency (contentItemId null): one post per channel per source URL.
    uniqueIndex("syndicated_posts_source_channel_unique").on(
      t.clientId,
      t.sourceUrl,
      t.channel,
    ),
    index("syndicated_posts_client_idx").on(t.clientId, t.createdAt),
  ],
);
```

### Acceptance criteria

1. Migration creates 4 enums + 2 tables; applies clean.
2. Two `syndicated_posts` for the same `(contentItemId, channel)` conflict (including the
   NULL-item RSS path via the `(clientId, sourceUrl, channel)` unique).
3. Deleting a content item cascades its syndicated posts.
4. `content_items.status` transitions are unconstrained at the DB layer (state machine lives
   in the workflow) — confirm no CHECK constraints were generated.

---

## DB-8 — `competitor_snapshots` + `intel_digests`

WF-6 espionage storage (doc 04). Zero client OAuth required — this pair unblocks the first
live engine.

### Schema additions

```ts
// --- Growth platform (DB-8): competitor espionage ---

/**
 * One weekly snapshot per competitor (doc 03 WF-6 step 1): normalized website
 * extract + public GBP stats. `competitorKey` = the slugified competitor name
 * from clientProfile.competitors (stable across snapshots; place ids can
 * change hosts). (clientId, competitorKey, capturedOn) unique = the weekly
 * cron is idempotent per day.
 */
export const competitorSnapshots = pgTable(
  "competitor_snapshots",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    competitorKey: text("competitor_key").notNull(),
    capturedAt: timestamp("captured_at", { withTimezone: true }).notNull().defaultNow(),
    /** Calendar day (UTC) of capture — the idempotency component. */
    capturedOn: date("captured_on").notNull(),
    /** sha256 of the normalized page text — cheap "anything changed?" check before diffing. */
    websiteTextHash: text("website_text_hash"),
    /** AI-extracted offers/prices/services; {"unreachable":true} when the site was down (never fabricated). */
    websiteExtract: jsonb("website_extract").$type<Record<string, unknown>>().notNull().default({}),
    /** Places API public data: {"rating":4.6,"reviewCount":312,"photoCount":41,...}. */
    gbpStats: jsonb("gbp_stats").$type<Record<string, unknown>>().notNull().default({}),
  },
  (t) => [
    uniqueIndex("competitor_snapshots_daily_unique").on(
      t.clientId,
      t.competitorKey,
      t.capturedOn,
    ),
    index("competitor_snapshots_client_key_idx").on(t.clientId, t.competitorKey, t.capturedAt),
  ],
);

/**
 * Monthly espionage digest (doc 03 WF-6 step 2-3): the human-readable artifact
 * (emailed + archived in portal). Machine-readable findings also land as
 * competitive-domain L2 insights (DB-10) — the digest is for humans, the notes
 * are for the system (doc 09).
 */
export const intelDigests = pgTable(
  "intel_digests",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    /** "YYYY-MM" — one digest per client per month (unique below = cron idempotency). */
    period: text("period").notNull(),
    bodyHtml: text("body_html").notNull(),
    /** Structured findings: [{competitorKey, kind, summary, recommendation}]. */
    findings: jsonb("findings").$type<Record<string, unknown>[]>().notNull().default([]),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("intel_digests_period_unique").on(t.clientId, t.period)],
);
```

### Acceptance criteria

1. Migration creates both tables; applies clean.
2. Same `(clientId, competitorKey, capturedOn)` twice → conflict; same key on a different day
   inserts.
3. Same `(clientId, period)` digest twice → conflict.

---

## DB-9 — `activity_log` + `metrics_daily`

Observability + value proof (doc 04): every action every engine takes, and the nightly
rollups (L1 of the doc 09 memory hierarchy). These exist from day one because the case study
and the intelligence layer are built on them.

### Schema additions

```ts
// --- Growth platform (DB-9): activity log + daily metric rollups ---

export const activityStatusEnum = pgEnum("activity_status", ["ok", "failed", "skipped"]);

/**
 * Every action every engine takes (doc 04) — powers dashboards, the case
 * study, and ops alerting. Written via POST /api/internal/activity (n8n) and
 * directly by in-app jobs. `engine` is OPEN vocabulary (text): the 8 module
 * slugs PLUS "provisioning", "send_pipeline", "intelligence", "system".
 * `dedupeKey` makes n8n retry-writes idempotent (same pattern as
 * conversionEvents.dedupeKey — NULLs never collide).
 */
export const activityLog = pgTable(
  "activity_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    engine: text("engine").notNull(),
    /** Dotted verb, e.g. "review_request.sent", "lead.first_touch", "campaign.batch_sent". */
    action: text("action").notNull(),
    status: activityStatusEnum("status").notNull().default("ok"),
    /** What the action touched: "message" | "review" | "campaign" | "content_item" | "event" | ... */
    entityType: text("entity_type"),
    /** Id of the touched entity (text — provider review ids aren't uuids). */
    entityId: text("entity_id"),
    detail: jsonb("detail").$type<Record<string, unknown>>().notNull().default({}),
    /** Caller-supplied idempotency key, e.g. "{event_id}:wf1:request_sent". */
    dedupeKey: text("dedupe_key"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    uniqueIndex("activity_log_dedupe_unique").on(t.clientId, t.dedupeKey),
    index("activity_log_client_ts_idx").on(t.clientId, t.createdAt),
    index("activity_log_client_engine_ts_idx").on(t.clientId, t.engine, t.createdAt),
    // Ops queues scan failures across clients.
    index("activity_log_failed_idx")
      .on(t.createdAt)
      .where(sql`${t.status} = 'failed'`),
  ],
);

/**
 * Growth-engine daily rollups (doc 04) — L1 of the intelligence hierarchy
 * (doc 09): exact numbers, zero AI, computed nightly by a pg-boss job from
 * activity_log/messages/conversations/reviews. Lives ALONGSIDE the existing
 * dailyAggregates (SoV product) — different product surfaces, deliberately
 * not merged. `module` is text: the 8 module slugs plus derived sheets like
 * "leakage". Example metrics payloads (doc 04): reviews →
 * {requestsSent, reviewsReceived, avgRating, responsesPublished}; leads →
 * {leads, medianResponseS, qualified, booked}.
 */
export const metricsDaily = pgTable(
  "metrics_daily",
  {
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    date: date("date").notNull(),
    module: text("module").notNull(),
    metrics: jsonb("metrics").$type<Record<string, number>>().notNull().default({}),
    computedAt: timestamp("computed_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    primaryKey({ columns: [t.clientId, t.date, t.module] }),
    index("metrics_daily_client_module_idx").on(t.clientId, t.module, t.date),
  ],
);
```

### Acceptance criteria

1. Migration creates the enum + 2 tables; `metrics_daily` PK is the 3-column composite.
2. Two activity rows with the same `(clientId, dedupeKey)` conflict; unlimited NULL-dedupeKey
   rows insert.
3. `onConflictDoUpdate` upsert on `metrics_daily` (target: the 3 PK columns) works — the
   nightly job recomputes idempotently.
4. Partial index `activity_log_failed_idx` present in generated SQL.

---

## DB-10 — Intelligence tables (`insights`, `client_briefs`, `experiments`, `playbooks`, `anomalies`)

Doc 09's storage additions: L2 insight notes, L3 versioned briefs, the experiment ledger, L4
niche playbooks, and the anomaly detector queue.

### Schema additions

```ts
// --- Growth platform (DB-10): intelligence layer (doc 09) ---

export const insightDomainEnum = pgEnum("insight_domain", [
  "reputation",
  "leads",
  "reactivation",
  "content",
  "visibility",
  "competitive",
]);
export const insightStatusEnum = pgEnum("insight_status", [
  "active",
  "superseded",
  "refuted",
]);

/**
 * L2 — AI-written, structured, ~200-token insight notes with confidence +
 * evidence pointers (doc 09). Agents retrieve these by domain; they NEVER read
 * raw logs. `embedding` is a plain jsonb float array for now (in-process
 * cosine at pilot scale); pgvector is a later one-migration upgrade — do NOT
 * add the extension in this ticket.
 */
export const insights = pgTable(
  "insights",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    domain: insightDomainEnum("domain").notNull(),
    /** The note body — hard budget ~200 tokens, enforced by the writing job, not the DB. */
    finding: text("finding").notNull(),
    /** Metric refs + event samples backing the finding: {"metricRefs":[...],"samples":[...]}. */
    evidence: jsonb("evidence").$type<Record<string, unknown>>().notNull().default({}),
    /** 0–1; experiment verdicts move it (doc 09 learning loop). */
    confidence: real("confidence").notNull().default(0.5),
    status: insightStatusEnum("status").notNull().default("active"),
    /** Embedding vector (jsonb number[]); null until embedded. */
    embedding: jsonb("embedding").$type<number[] | null>(),
    /** The newer note that replaced this one. Plain uuid (self-FK omitted deliberately). */
    supersededBy: uuid("superseded_by"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    index("insights_client_domain_idx").on(t.clientId, t.domain, t.status),
    index("insights_client_ts_idx").on(t.clientId, t.createdAt),
  ],
);

/**
 * L3 — the versioned living brief, THE base context every agent loads
 * (doc 09; hard cap ~1.5k tokens, enforced by the regenerating job).
 * Append-only: a refresh inserts version n+1; readers take max(version).
 */
export const clientBriefs = pgTable(
  "client_briefs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    version: integer("version").notNull(),
    bodyMd: text("body_md").notNull(),
    /** Measured token count at generation — regression guard on the 1.5k cap. */
    tokenCount: integer("token_count"),
    /** insight ids (and other inputs) the brief was generated from — auditability. */
    inputs: jsonb("inputs").$type<string[]>().notNull().default([]),
    generatedAt: timestamp("generated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("client_briefs_version_unique").on(t.clientId, t.version)],
);

export const experimentSourceEnum = pgEnum("experiment_source", [
  "action_queue",
  "campaign",
  "content",
  "config_change",
  "manual",
]);
export const experimentStatusEnum = pgEnum("experiment_status", ["running", "closed"]);

/**
 * The learning ledger (doc 09): every meaningful action framed as an
 * experiment with hypothesis → expected impact → measured outcome. Losses are
 * retained as anti-patterns. `status` is not in doc 09 but makes "open
 * experiments" a query instead of an outcome-is-null convention.
 */
export const experiments = pgTable(
  "experiments",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    source: experimentSourceEnum("source").notNull(),
    hypothesis: text("hypothesis").notNull(),
    /** {module, configDelta} or {campaignId} — what was actually changed/launched. */
    action: jsonb("action").$type<Record<string, unknown>>().notNull(),
    expectedImpact: jsonb("expected_impact")
      .$type<{ metric: string; direction: "up" | "down"; magnitudeGuess?: string }>()
      .notNull(),
    status: experimentStatusEnum("status").notNull().default("running"),
    startedAt: timestamp("started_at", { withTimezone: true }).notNull().defaultNow(),
    measureUntil: timestamp("measure_until", { withTimezone: true }).notNull(),
    /** Null while running. {metricBefore, metricAfter, revenueEst, verdict: "win"|"loss"|"flat"}. */
    outcome: jsonb("outcome").$type<{
      metricBefore: number;
      metricAfter: number;
      revenueEst?: number;
      verdict: "win" | "loss" | "flat";
    } | null>(),
    /** The L2 insight written when the experiment closed. */
    learnedNoteId: uuid("learned_note_id").references(() => insights.id, {
      onDelete: "set null",
    }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    index("experiments_client_status_idx").on(t.clientId, t.status),
    // The measurement sweep closes experiments whose window has passed.
    index("experiments_due_idx")
      .on(t.measureUntil)
      .where(sql`${t.status} = 'running'`),
  ],
);

/**
 * L4 — cross-client niche playbooks (doc 09). GLOBAL (no clientId — the one
 * intentionally tenant-free growth table). PRIVACY RULE enforced by the
 * generating job: aggregate patterns only, no client-identifiable data.
 * Append-only versions like clientBriefs.
 */
export const playbooks = pgTable(
  "playbooks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    /** Vertical key, e.g. "golf_recreation", "home_services". */
    niche: text("niche").notNull(),
    version: integer("version").notNull(),
    bodyMd: text("body_md").notNull(),
    /** How many closed experiments back this version — confidence signal. */
    supportingExperiments: integer("supporting_experiments").notNull().default(0),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("playbooks_niche_version_unique").on(t.niche, t.version)],
);

export const anomalyDirectionEnum = pgEnum("anomaly_direction", ["up", "down"]);
export const anomalyStatusEnum = pgEnum("anomaly_status", [
  "new",
  "reviewed",
  "actioned",   // an Action Queue item / experiment was opened for it
  "dismissed",
]);

/**
 * Detector output queue (doc 09 step 1 "Observe") — pure SQL/stats over
 * metrics_daily, zero AI. Weekly insight jobs consume `new` rows.
 * `windowDays` (smallint) instead of doc 09's free-text `window` — matches
 * monitors.windowDays.
 */
export const anomalies = pgTable(
  "anomalies",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    /** "{module}.{metric}" from metrics_daily, e.g. "review_automation.reviewsReceived". */
    metric: text("metric").notNull(),
    direction: anomalyDirectionEnum("direction").notNull(),
    /** Effect size in the metric's own units (or σ for z-score detectors — detail in detector). */
    magnitude: numeric("magnitude", { precision: 12, scale: 4 }).notNull(),
    windowDays: smallint("window_days").notNull().default(7),
    detectedAt: timestamp("detected_at", { withTimezone: true }).notNull().defaultNow(),
    status: anomalyStatusEnum("status").notNull().default("new"),
  },
  (t) => [index("anomalies_client_status_idx").on(t.clientId, t.status, t.detectedAt)],
);
```

### Acceptance criteria

1. Migration creates 6 enums + 5 tables; applies clean; no `CREATE EXTENSION` anywhere.
2. `client_briefs`: same `(clientId, version)` twice conflicts; version 1 and 2 both insert.
3. `experiments.learnedNoteId` FK: deleting the referenced insight sets it NULL.
4. `playbooks` has no `client_id` column (verify in SQL) — the privacy boundary is structural.
5. Partial index `experiments_due_idx` present with `WHERE ... = 'running'`.

---

## DB-11 — Extend `reviews`, `review_requests`, `review_contacts`

Doc 10 §4: the existing review-request funnel becomes WF-1's substrate. Three additive
changes, no data migration.

### Schema changes (edit the EXISTING table definitions in place)

1. Add above `reviews`:

```ts
/** Lifecycle of the AI-drafted owner response to a review (doc 04). */
export const reviewResponseStatusEnum = pgEnum("review_response_status", [
  "none",
  "drafted",
  "pending_approval",
  "published",
  "failed",
]);
```

2. Append to the `reviews` table columns (after `raw`, before `ingestedAt`):

```ts
    // --- Growth platform (DB-11): WF-1 Part B — AI review responses ---
    /** AI-drafted (and possibly edited) owner response text. */
    responseBody: text("response_body"),
    responseStatus: reviewResponseStatusEnum("response_status").notNull().default("none"),
    /** When the response went live on the provider (GBP API or ops manual publish). */
    respondedAt: timestamp("responded_at", { withTimezone: true }),
    responseError: text("response_error"),
```

3. Append to `reviewRequests` columns (after `error`, before `createdAt`):

```ts
    /** The canonical sale.completed event that triggered this ask (POS-triggered
     *  WF-1 path); null for the legacy webhook/manual funnel. */
    sourceEventId: uuid("source_event_id").references(() => events.id, {
      onDelete: "set null",
    }),
```

4. Append to `reviewContacts` columns (after `adapter`, before `createdAt`):

```ts
    /** Link to the canonical customers row (DB-4); backfilled lazily by phone/email match. */
    customerId: uuid("customer_id").references(() => customers.id, {
      onDelete: "set null",
    }),
```

> `reviews` needs no new location/provider work: its `source` enum already contains `"gbp"`,
> and `(clientId, source, externalId)` uniqueness already gives the "idempotent on provider
> review id" guarantee doc 03 WF-1 requires.

### Acceptance criteria

1. Migration is purely additive: 1 `CREATE TYPE` + 6 `ADD COLUMN` + 2 FK constraints; no
   `DROP`/`ALTER COLUMN` of existing columns (read the SQL).
2. Existing reviews read back with `responseStatus = 'none'`.
3. Deleting an event / customer nulls the new FK columns rather than deleting requests/contacts.
4. `npm test` stays green (review engine tests unaffected).

---

## DB-12 — Seed script: demo growth tenant

Extend seeding so every growth-platform page and API can be exercised locally. New file, new
npm script — the existing `scripts/seed.ts` (SoV demo) is untouched and remains a
prerequisite (it creates the `seekly` demo client).

### `package.json` — add to `"scripts"`

```json
    "db:seed:growth": "tsx --env-file=.env scripts/seed-growth.ts",
```

### New file `scripts/seed-growth.ts` (complete)

```ts
/**
 * Seeds the GROWTH-PLATFORM tables for the demo client created by
 * `npm run db:seed` (slug "seekly"). Deterministic and idempotent: wipes and
 * recreates only growth rows for that client. Run AFTER db:seed.
 */
import { eq } from "drizzle-orm";
import { db } from "../src/server/db";
import {
  activityLog,
  campaignMembers,
  campaigns,
  clientProfile,
  clients,
  commsProvisioning,
  competitorSnapshots,
  contentItems,
  conversations,
  customers,
  events,
  intelDigests,
  messages,
  metricsDaily,
  syndicatedPosts,
  workflowConfig,
} from "../src/server/db/schema";
import { assertSafeToWipe } from "./_guard";

// --- deterministic RNG (mulberry32), same pattern as scripts/seed.ts ---
function mulberry32(seed: number) {
  return function () {
    let t = (seed += 0x6d2b79f5);
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const rand = mulberry32(20260713);
const pick = <T>(arr: readonly T[]) => arr[Math.floor(rand() * arr.length)];

const DAY = 24 * 60 * 60 * 1000;
const daysAgo = (n: number) => new Date(Date.now() - n * DAY);
const dateStr = (d: Date) => d.toISOString().slice(0, 10);

// Deterministic uuid-shaped ids (seed data only — NOT a general uuid factory).
function seedUuid(): string {
  const hex = () => Math.floor(rand() * 16).toString(16);
  const s = (n: number) => Array.from({ length: n }, hex).join("");
  return `${s(8)}-${s(4)}-4${s(3)}-a${s(3)}-${s(12)}`;
}

const FIRST = ["Alex", "Jordan", "Sam", "Taylor", "Morgan", "Casey", "Riley", "Jamie", "Drew", "Quinn"];
const LAST = ["Smith", "Lee", "Patel", "Brown", "Wilson", "Nguyen", "Garcia", "Martin", "Clark", "Baker"];

const MODULES = [
  "review_automation",
  "speed_to_lead",
  "reactivation",
  "content_engine",
  "directory_sync",
  "competitor_intel",
  "social_syndication",
  "intelligence",
] as const;

async function main() {
  await assertSafeToWipe();

  const [client] = await db.select().from(clients).where(eq(clients.slug, "seekly"));
  if (!client) {
    throw new Error('Demo client "seekly" not found — run `npm run db:seed` first.');
  }
  const clientId = client.id;

  // --- wipe this client's growth rows (children first; FKs cascade the rest) ---
  await db.delete(activityLog).where(eq(activityLog.clientId, clientId));
  await db.delete(metricsDaily).where(eq(metricsDaily.clientId, clientId));
  await db.delete(intelDigests).where(eq(intelDigests.clientId, clientId));
  await db.delete(competitorSnapshots).where(eq(competitorSnapshots.clientId, clientId));
  await db.delete(syndicatedPosts).where(eq(syndicatedPosts.clientId, clientId));
  await db.delete(contentItems).where(eq(contentItems.clientId, clientId));
  await db.delete(messages).where(eq(messages.clientId, clientId));
  await db.delete(conversations).where(eq(conversations.clientId, clientId));
  await db.delete(campaigns).where(eq(campaigns.clientId, clientId)); // members cascade
  await db.delete(events).where(eq(events.clientId, clientId));
  await db.delete(customers).where(eq(customers.clientId, clientId));
  await db.delete(commsProvisioning).where(eq(commsProvisioning.clientId, clientId));
  await db.delete(workflowConfig).where(eq(workflowConfig.clientId, clientId));
  await db.delete(clientProfile).where(eq(clientProfile.clientId, clientId));

  // --- tenant fields ---
  await db
    .update(clients)
    .set({ tier: "pilot", accountStatus: "active", industry: "golf simulator venue", timezone: "America/Toronto" })
    .where(eq(clients.id, clientId));

  // --- profile (a golf-venue intake, doc 08) ---
  await db.insert(clientProfile).values({
    clientId,
    services: ["golf simulator rentals", "birthday parties", "league nights", "lessons"],
    serviceAreas: ["Kingston", "Napanee", "Gananoque"],
    targetKeywords: ["golf simulator kingston", "indoor golf kingston", "golf lessons kingston"],
    competitors: [
      { name: "FairwayZone", website: "https://fairwayzone.example.com" },
      { name: "TeeBox Indoor Golf", website: "https://teebox.example.com" },
      { name: "SwingCity", website: "https://swingcity.example.com" },
    ],
    brandVoice: {
      tone: "friendly, upbeat, never stuffy",
      personaName: "Sam from Seekly Golf",
      phrasesToAvoid: ["golf-course quality", "guaranteed improvement"],
      exampleSentences: ["Grab a bay, bring your crew — we'll handle the fun part."],
    },
    aiGuardrails: ["never quote prices", "never promise availability", "never offer unlisted discounts"],
    qualificationQuestions: [
      { question: "How many people?", key: "party_size", hotSignal: ">=6" },
      { question: "What date were you thinking?", key: "date" },
      { question: "Special occasion?", key: "occasion" },
    ],
    campaignAngles: [
      { key: "league_night", label: "League night invite", description: "Tuesday league open spots", approved: true },
      { key: "birthday", label: "Birthday package", description: "Party package pitch", approved: true },
    ],
    bookingLink: "https://book.example.com/seekly-golf",
    websiteUrl: client.website,
    websitePlatform: "wordpress",
    reviewLink: "https://g.page/r/seekly-golf/review",
    approvalPreferences: { review_responses: "auto", campaigns: "approve_first", content: "approve_first_3" },
    escalationContacts: [{ name: "Matt", mobile: "+16135550100", alerts: ["hot_lead", "negative_review"] }],
    avgCustomerValueUsd: "480.00",
  });

  // --- switchboard: review + speed-to-lead ON, rest OFF (2-week milestone state) ---
  await db.insert(workflowConfig).values(
    MODULES.map((module) => ({
      clientId,
      module,
      enabled: module === "review_automation" || module === "speed_to_lead",
      settings:
        module === "review_automation"
          ? { sendDelayHours: 2, reAskCooldownDays: 90, dailySendCap: 25, negativeThreshold: 3 }
          : module === "speed_to_lead"
            ? { followupTouches: 5, followupDays: 7 }
            : {},
      updatedBy: "seed",
    })),
  );

  // --- comms (A2P approved so the send pipeline is exercisable in dev) ---
  await db.insert(commsProvisioning).values({
    clientId,
    twilioSubaccountSid: "ACseeddemo00000000000000000000000",
    phoneNumber: "+16135550142",
    a2pBrandStatus: "approved",
    a2pCampaignStatus: "approved",
    a2pCampaignType: "MIXED",
    inboundEmailAddress: "seekly@in.seekly.app",
  });

  // --- customers: 40, lastVisitAt spread over 200 days (some lapsed > 90d) ---
  const customerRows = Array.from({ length: 40 }, (_, i) => ({
    id: seedUuid(),
    clientId,
    name: `${pick(FIRST)} ${pick(LAST)}`,
    phone: `+1613555${String(1000 + i)}`,
    email: `customer${i}@example.com`,
    lastVisitAt: daysAgo(Math.floor(rand() * 200)),
    visitCount: 1 + Math.floor(rand() * 12),
    consentBasis: "existing_customer" as const,
    source: "csv",
  }));
  await db.insert(customers).values(customerRows);

  // --- canonical events: sales + leads over the last 14 days ---
  const eventRows = Array.from({ length: 20 }, (_, i) => {
    const c = customerRows[i];
    const sale = i % 2 === 0;
    return {
      id: seedUuid(),
      clientId,
      type: (sale ? "sale.completed" : "lead.created") as "sale.completed" | "lead.created",
      source: sale ? "email-parse" : "webhook",
      occurredAt: daysAgo(Math.floor(rand() * 14)),
      payload: sale
        ? { customer: { name: c.name, phone: c.phone }, amount: 40 + Math.floor(rand() * 200) }
        : { customer: { name: c.name, phone: c.phone }, form: "party-inquiry", message: "Looking to book a bay" },
      status: "processed" as const,
      processedAt: daysAgo(Math.floor(rand() * 14)),
    };
  });
  await db.insert(events).values(eventRows);

  // --- conversations + messages for the lead events ---
  const leadEvents = eventRows.filter((e) => e.type === "lead.created");
  for (const evt of leadEvents) {
    const cust = customerRows.find((c) => c.phone === (evt.payload as { customer: { phone: string } }).customer.phone)!;
    const convId = seedUuid();
    const status = pick(["qualified", "booked", "cold", "closed"] as const);
    await db.insert(conversations).values({
      id: convId,
      clientId,
      customerId: cust.id,
      engine: "speed_to_lead",
      status,
      leadSource: "website_form",
      leadEventId: evt.id,
      context: { party_size: 2 + Math.floor(rand() * 8) },
      firstResponseSeconds: 20 + Math.floor(rand() * 40),
      booked: status === "booked",
      createdAt: evt.occurredAt,
    });
    await db.insert(messages).values([
      {
        clientId,
        customerId: cust.id,
        direction: "out",
        kind: "conversational",
        engine: "speed_to_lead",
        phone: cust.phone!,
        body: `Hi ${cust.name!.split(" ")[0]}! Got your request — what date were you thinking?`,
        status: "delivered",
        idempotencyKey: `${evt.id}:first_touch`,
        conversationId: convId,
        createdAt: new Date(evt.occurredAt.getTime() + 30 * 1000),
        sentAt: new Date(evt.occurredAt.getTime() + 30 * 1000),
      },
      {
        clientId,
        customerId: cust.id,
        direction: "in",
        kind: "conversational",
        engine: "speed_to_lead",
        phone: cust.phone!,
        body: "This Friday around 7 if you have space",
        status: "delivered",
        conversationId: convId,
        createdAt: new Date(evt.occurredAt.getTime() + 5 * 60 * 1000),
      },
    ]);
  }

  // --- one reactivation campaign with a resumable member ledger ---
  const campaignId = seedUuid();
  await db.insert(campaigns).values({
    id: campaignId,
    clientId,
    angle: "league_night",
    copyVariants: [{ key: "v1", body: "We miss you at the bays, {first_name} — Tuesday league has 3 open spots." }],
    audienceFilter: { lapsedDays: 90 },
    status: "sending",
    stats: { sent: 10, replies: 3, stops: 0, bookings: 2, revenueEst: 960 },
    approvedBy: "seed",
    approvedAt: daysAgo(5),
  });
  const lapsed = customerRows.filter((c) => c.lastVisitAt < daysAgo(90)).slice(0, 15);
  await db.insert(campaignMembers).values(
    lapsed.map((c, i) => ({
      campaignId,
      customerId: c.id,
      status: (i < 10 ? (i < 3 ? "replied" : "sent") : "queued") as "replied" | "sent" | "queued",
      sentAt: i < 10 ? daysAgo(4) : null,
    })),
  );

  // --- content + syndication ---
  const contentId = seedUuid();
  await db.insert(contentItems).values({
    id: contentId,
    clientId,
    topic: "best indoor golf simulators in Kingston",
    title: "Indoor Golf in Kingston: The Complete 2026 Guide",
    bodyHtml: "<h1>Indoor Golf in Kingston</h1><p>TL;DR — …</p>",
    schemaJsonld: { "@type": "LocalBusiness" },
    status: "published",
    publishTarget: "wordpress",
    publishedUrl: `${client.website}/blog/indoor-golf-kingston`,
    publishedAt: daysAgo(6),
  });
  await db.insert(syndicatedPosts).values([
    { clientId, contentItemId: contentId, channel: "gbp", body: "New guide: indoor golf in Kingston ⛳", ctaUrlUtm: `${client.website}/blog/indoor-golf-kingston?utm_source=gbp`, status: "posted", postedAt: daysAgo(5), clicks: 14 },
    { clientId, contentItemId: contentId, channel: "facebook", body: "Everything about indoor golf in Kingston →", ctaUrlUtm: `${client.website}/blog/indoor-golf-kingston?utm_source=fb`, status: "posted", postedAt: daysAgo(5), clicks: 9 },
  ]);

  // --- espionage: 4 weekly snapshots per competitor + one monthly digest ---
  for (const comp of ["fairwayzone", "teebox-indoor-golf", "swingcity"]) {
    for (let w = 0; w < 4; w++) {
      const at = daysAgo(7 * w + 1);
      await db.insert(competitorSnapshots).values({
        clientId,
        competitorKey: comp,
        capturedAt: at,
        capturedOn: dateStr(at),
        websiteTextHash: seedUuid().replace(/-/g, ""),
        websiteExtract: { offers: w === 0 && comp === "fairwayzone" ? ["$25 league night"] : [] },
        gbpStats: { rating: 4 + rand(), reviewCount: 80 + w * 4 + Math.floor(rand() * 10) },
      });
    }
  }
  await db.insert(intelDigests).values({
    clientId,
    period: dateStr(daysAgo(15)).slice(0, 7),
    bodyHtml: "<h2>Competitor digest</h2><p>FairwayZone launched a $25 league night…</p>",
    findings: [{ competitorKey: "fairwayzone", kind: "new_offer", summary: "$25 league night", recommendation: "Counter with Tuesday league promo" }],
    sentAt: daysAgo(14),
  });

  // --- activity log + metrics_daily (14 days) ---
  const activity: (typeof activityLog.$inferInsert)[] = [];
  const metrics: (typeof metricsDaily.$inferInsert)[] = [];
  for (let d = 0; d < 14; d++) {
    const day = daysAgo(d);
    activity.push({
      clientId,
      engine: "review_automation",
      action: "review_request.sent",
      status: "ok",
      entityType: "message",
      detail: { channel: "sms" },
      dedupeKey: `seed:wf1:${dateStr(day)}`,
      createdAt: day,
    });
    metrics.push(
      { clientId, date: dateStr(day), module: "review_automation", metrics: { requestsSent: 1 + Math.floor(rand() * 4), reviewsReceived: Math.floor(rand() * 2), responsesPublished: Math.floor(rand() * 2) } },
      { clientId, date: dateStr(day), module: "speed_to_lead", metrics: { leads: Math.floor(rand() * 3), medianResponseS: 25 + Math.floor(rand() * 30), booked: Math.floor(rand() * 2) } },
    );
  }
  await db.insert(activityLog).values(activity);
  await db.insert(metricsDaily).values(metrics);

  console.log("Growth-platform seed complete for client", client.slug);
  process.exit(0);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

### Acceptance criteria

1. Fresh dev DB: `npm run db:migrate && npm run db:seed && npm run db:seed:growth` completes
   with exit code 0.
2. Running `db:seed:growth` **twice in a row** succeeds (idempotent wipe+recreate) and row
   counts are identical after each run.
3. It refuses to run against a non-dev database (existing `assertSafeToWipe` guard fires).
4. Spot checks: 40 customers; 8 `workflow_config` rows of which exactly 2 enabled; 1 campaign
   with 15 members (3 replied / 7 sent / 5 queued); 12 competitor snapshots; 28 `metrics_daily`
   rows; every seeded outbound message has an `idempotencyKey`.
5. `npx tsc --noEmit` passes (the script type-checks against the new schema).

---

## Open items (tracked, not blockers)

- **pgvector upgrade** for `insights.embedding` once note volume makes in-process cosine slow
  (one migration: extension + column type + HNSW index).
- **Drop `review_opt_outs`** after one release cycle of the DB-6 repoint running clean.
- **`locations` table extraction** when the first multi-location client signs (move the
  location-ish `client_profile` fields).
