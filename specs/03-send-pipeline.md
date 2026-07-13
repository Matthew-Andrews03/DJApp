# Spec 03 — Shared Send-Pipeline (the single compliance path for ALL outbound SMS)

**Repo:** all file paths below are in `seekly-client-insights` (Next.js 16 + Drizzle + Postgres + pg-boss).
**Source requirements:** doc 02 §"Compliance & messaging guardrails", doc 03 §"Shared send-pipeline", doc 06 §Twilio, doc 07 §"Definition of rock solid". Doc 10 supersedes stack assumptions (no Supabase — Drizzle + route handlers + pg-boss).

## Purpose

Every outbound SMS Seekly ever sends — in-app review requests (existing `src/server/reviews/engine.ts`), and the n8n engines (WF-1 POS-triggered requests, WF-2 speed-to-lead, WF-3 reactivation) — goes through **one** function: `sendMessage()` in `src/server/messaging/send-pipeline.ts`. n8n reaches it via `POST /api/internal/send`. No engine ever calls Twilio directly.

The pipeline enforces, in order:

1. **Opt-out ledger check** — cross-engine, client-scoped `opt_outs` (plus the legacy hashed `reviewOptOuts` list) blocks the number for ALL kinds.
2. **A2P approval hard gate** — no send of any kind before `comms_provisioning.a2p_campaign_status = 'approved'`.
3. **Frequency cap** — max 1 `marketing` message per customer per 30 days (default), SQL over the `messages` ledger, across all engines.
4. **Quiet hours** — 8am–9pm client-local, `marketing` only; out-of-window sends are **queued via pg-boss `startAfter`, never dropped**.
5. **Idempotent ledger insert** — unique `idempotency_key`; replaying any trigger produces zero duplicate sends.
6. **Twilio send via the client's subaccount** → status callback updates the ledger.
7. **Global inbound handling** — STOP/UNSUBSCRIBE/HELP with carrier-required responses; non-keyword inbound becomes a canonical `message.received` event; STOP spikes auto-pause campaigns.

This generalizes the existing review-request code. Reuse, don't reinvent:

| Existing code | Role in this spec |
|---|---|
| `src/server/reviews/compliance.ts` | Pattern for the new pure, fail-closed `src/server/messaging/compliance.ts` (keep `hashContact`/`normalizeContact` — imported for the legacy opt-out check) |
| `src/server/reviews/messaging.ts` → `sendReviewSms` | Replaced by the pipeline (SP-8). The raw-REST-no-SDK Twilio style is kept |
| `src/app/api/reviews/sms/inbound/route.ts` | Twilio signature validation is extracted and reused; route stays alive for the legacy shared number (SP-6) |
| `src/server/pipeline/queue.ts` + `src/worker.ts` | New `message-send` queue + handler follow the exact same `QUEUES`/`RETRY_QUEUES`/dlq/`boss.work` pattern |
| `src/server/spine/auth.ts` → `requireServiceToken` | Auth for `POST /api/internal/send` (same `SPINE_SERVICE_TOKEN` n8n already uses for spine routes) |

## `kind` semantics — which checks apply

| Check | `transactional` | `conversational` | `marketing` |
|---|---|---|---|
| Opt-out ledger (cross-engine) | ✅ blocks | ✅ blocks | ✅ blocks |
| A2P campaign approved | ✅ blocks | ✅ blocks | ✅ blocks |
| Frequency cap (30d default) | — exempt | — exempt | ✅ blocks |
| Quiet hours (8–21 client-local) | — exempt | — exempt | ✅ queues (never drops) |
| Idempotent ledger insert | ✅ | ✅ | ✅ |

**Kind assignment rules (engines must follow these):**

- `transactional` — direct service messages the customer explicitly triggered: booking confirmations, feedback-link re-sends the customer asked for. Rare in v1.
- `conversational` — any message in an active two-way thread: WF-2 first-touch reply to an inbound lead (doc 03: an overnight lead gets an instant reply — it is a direct response to their inquiry, so it is exempt from quiet-hours *initiation*), all AI conversation turns, WF-3 replies after a customer responds.
- `marketing` — anything Seekly initiates that isn't a reply: **review requests (WF-1)**, reactivation campaign sends (WF-3), follow-up sequence touches after a lead goes silent (WF-2 day-1/3/5/7 touches). When in doubt, use `marketing` — it is the strictest.

STOP always blocks everything: after an opt-out, even a conversational reply must not send (TCPA/carrier rule — the pipeline enforces this, engines don't get a vote).

---

## Ticket table

| # | Ticket | Depends on | Size |
|---|---|---|---|
| SP-1 | Schema: `messages` ledger, `opt_outs`, `comms_provisioning`, `clients.timezone` + migration & backfill | — | M |
| SP-2 | Pure compliance module (`decideSend`, quiet-hours math, E.164) + unit tests | — | S |
| SP-3 | Pipeline core: `sendMessage()` / `performSend()`, pg-boss `message-send` queue, worker wiring | SP-1, SP-2 | L |
| SP-4 | Twilio REST client (subaccount send) + status-callback route | SP-1 | M |
| SP-5 | `POST /api/internal/send` (n8n entry point) | SP-3 | S |
| SP-6 | Global inbound handler: STOP/HELP/START, `message.received`, client resolution by number | SP-1, SP-3 | M |
| SP-7 | Twilio provisioning helper: subaccount, number purchase, messaging service, A2P brand/campaign submit + status poll | SP-1 | L |
| SP-8 | Migrate review-request sends onto the pipeline (changes to `engine.ts` / `messaging.ts`) | SP-3, SP-4 | M |
| SP-9 | Failure handling: 429 retry, undelivered statuses, error-code opt-outs, STOP-rate campaign auto-pause, ops alerts | SP-3, SP-6 | M |
| SP-10 | Compliance test pack (doc 07 "rock solid" proof) | SP-3–SP-9 | M |

New env vars (add to the zod schema in `src/env.ts`, all in the "Optional" style used there):

```ts
  // --- Optional: messaging ops alerts (Resend; falls back to log-only when unset) ---
  OPS_ALERT_EMAIL: z.string().email().optional(),
```

Existing env vars used (already in `src/env.ts`): `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` (master account credentials), `TWILIO_FROM_NUMBER` (legacy shared number only), `APP_URL` (webhook/callback URLs), `RESEND_API_KEY` + `REPORT_EMAIL_FROM` (ops alerts), `SPINE_SERVICE_TOKEN` (internal API auth).

---

## SP-1 — Schema & migration

Add to `src/server/db/schema.ts`. If spec 01 (platform schema) has already landed some of these tables, reconcile column-by-column — the pipeline requires exactly the columns below. Run `npm run db:generate` then `npm run db:migrate`.

```ts
// ---------------------------------------------------------------------------
// Messaging (spec 03): the cross-engine message ledger + opt-outs + per-client
// Twilio provisioning. ALL outbound SMS flows through these tables.
// ---------------------------------------------------------------------------

export const messageDirectionEnum = pgEnum("message_direction", ["in", "out"]);
export const messageKindEnum = pgEnum("message_kind", [
  "transactional",
  "conversational",
  "marketing",
]);
export const messageStatusEnum = pgEnum("message_status", [
  "queued",      // accepted by the pipeline; waiting (quiet hours) or about to send
  "sending",     // a worker holds it; transition guard against double-send
  "sent",        // accepted by Twilio (sid assigned)
  "delivered",   // Twilio status callback: delivered
  "undelivered", // Twilio status callback: undelivered (carrier rejected)
  "failed",      // Twilio rejected the API call, or callback: failed
  "blocked",     // compliance gate refused it — see blockedReason
]);
export const messageBlockedReasonEnum = pgEnum("message_blocked_reason", [
  "opt_out",
  "freq_cap",
  "a2p_pending",
  "no_destination",
]);

/**
 * The message ledger — one row per SMS in or out, every engine. Idempotency:
 * `idempotencyKey` is unique; re-sending the same trigger is a no-op that
 * returns the existing row. Frequency caps and conversation threading are
 * SQL over this table.
 */
export const messages = pgTable(
  "messages",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    /** Optional link to the canonical customer record (spec 01 `customers`). */
    customerId: uuid("customer_id"),
    direction: messageDirectionEnum("direction").notNull(),
    kind: messageKindEnum("kind").notNull(),
    /** Which engine produced it: "review_velocity" | "speed_to_lead" | "reactivation" | ... */
    engine: text("engine").notNull(),
    /** E.164. For direction=out this is the customer; for in, the sender. */
    toPhone: text("to_phone").notNull(),
    fromPhone: text("from_phone").notNull(),
    body: text("body").notNull(),
    mediaUrls: jsonb("media_urls").$type<string[]>(),
    twilioSid: text("twilio_sid"),
    status: messageStatusEnum("status").notNull().default("queued"),
    blockedReason: messageBlockedReasonEnum("blocked_reason"),
    /** Twilio error code (e.g. "30003") from the send call or status callback. */
    errorCode: text("error_code"),
    /** e.g. "review-request:<requestId>", "n8n:<event_id>:step2". Unique = idempotent. */
    idempotencyKey: text("idempotency_key").notNull(),
    campaignId: uuid("campaign_id"),
    conversationId: uuid("conversation_id"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    deliveredAt: timestamp("delivered_at", { withTimezone: true }),
  },
  (t) => [
    uniqueIndex("messages_idempotency_unique").on(t.idempotencyKey),
    // Frequency-cap lookup: latest marketing sends per (client, phone).
    index("messages_freq_cap_idx").on(t.clientId, t.toPhone, t.kind, t.createdAt),
    index("messages_twilio_sid_idx").on(t.twilioSid),
    index("messages_campaign_idx").on(t.campaignId),
    index("messages_client_created_idx").on(t.clientId, t.createdAt),
  ],
);

/**
 * Cross-engine opt-out ledger (doc 04). One row per (client, phone). Raw E.164
 * is stored (not a hash) because campaign exclusion joins against
 * `customers.phone`. The legacy hashed `reviewOptOuts` table remains and is
 * ALSO consulted at send time (see compliance module) so no pre-migration
 * STOP is ever forgotten.
 */
export const optOuts = pgTable(
  "opt_outs",
  {
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    phone: text("phone").notNull(), // E.164
    optedOutAt: timestamp("opted_out_at", { withTimezone: true }).notNull().defaultNow(),
    /** "sms_stop" | "manual" | "carrier_error" (Twilio 21610) */
    source: text("source").notNull().default("sms_stop"),
    sourceMessageId: uuid("source_message_id"),
  },
  (t) => [primaryKey({ columns: [t.clientId, t.phone] })],
);

export const a2pCampaignStatusEnum = pgEnum("a2p_campaign_status", [
  "none",
  "submitted",
  "approved",
  "rejected",
]);

/** Per-client Twilio + inbound-email provisioning (doc 04 `comms_provisioning`). */
export const commsProvisioning = pgTable("comms_provisioning", {
  clientId: uuid("client_id")
    .primaryKey()
    .references(() => clients.id, { onDelete: "cascade" }),
  twilioSubaccountSid: text("twilio_subaccount_sid"),
  /**
   * Subaccount auth token, needed for messaging.twilio.com / trusthub.twilio.com
   * calls that must authenticate AS the subaccount. SECURITY NOTE: plaintext for
   * v1 (DB access is already the trust boundary for `clients.ingestKey` etc.);
   * migrate into the Bitwarden vault (src/server/vault) in a follow-up ticket.
   */
  twilioSubaccountAuthToken: text("twilio_subaccount_auth_token"),
  /** The client's dedicated E.164 number. */
  phoneNumber: text("phone_number"),
  messagingServiceSid: text("messaging_service_sid"),
  a2pBrandSid: text("a2p_brand_sid"),
  a2pBrandStatus: text("a2p_brand_status").notNull().default("none"),
  a2pCampaignSid: text("a2p_campaign_sid"),
  a2pCampaignStatus: a2pCampaignStatusEnum("a2p_campaign_status").notNull().default("none"),
  a2pCampaignType: text("a2p_campaign_type"),
  /** {slug}@in.seekly.app — assigned here, consumed by spec 05. */
  inboundEmailAddress: text("inbound_email_address"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Also add one column to the existing `clients` table (quiet hours need a real IANA zone; the existing `country`→zone mapping in `reviews/compliance.ts` stays only for the legacy email path):

```ts
  /** IANA timezone for quiet-hours evaluation (doc 04). Set at intake. */
  timezone: text("timezone").notNull().default("America/Toronto"),
```

**Backfill (one-off SQL, run after migration)** — every existing active client gets a provisioning row pointing at the legacy shared number so the pipeline can serve them before per-client subaccounts exist. `a2p_campaign_status` stays `none` until ops confirms the shared number's campaign is approved (the hard gate is real — shadow mode until then, per doc 07):

```sql
INSERT INTO comms_provisioning (client_id, phone_number, inbound_email_address)
SELECT id, '<value of TWILIO_FROM_NUMBER>', slug || '@in.seekly.app'
FROM clients WHERE deleted_at IS NULL
ON CONFLICT (client_id) DO NOTHING;
-- When ops confirms A2P approval of the shared number's campaign:
-- UPDATE comms_provisioning SET a2p_campaign_status = 'approved' WHERE client_id = '<pilot id>';
```

**Acceptance criteria**

1. `npm run db:generate` produces a migration; `npm run db:migrate` applies cleanly to a fresh DB and to a copy of the current schema.
2. Inserting two `messages` rows with the same `idempotencyKey` violates the unique index (proven in a test).
3. `opt_outs` PK prevents duplicate (client, phone) rows; `ON CONFLICT DO NOTHING` insert is a no-op.
4. Backfill SQL run twice is idempotent.

---

## SP-2 — Pure compliance module

`src/server/messaging/compliance.ts` — like `src/server/reviews/compliance.ts`: **pure, fail-closed, no DB, no clock**, exhaustively unit-testable.

```ts
/**
 * Send-pipeline compliance gate — the money path for ALL outbound SMS.
 * Pure and FAIL-CLOSED: a send happens only when every rule explicitly
 * permits it. Mirrors src/server/reviews/compliance.ts, generalized to
 * three message kinds (doc 02 guardrails, doc 03 send-pipeline).
 */

export type MessageKind = "transactional" | "conversational" | "marketing";

export type BlockedReason = "opt_out" | "freq_cap" | "a2p_pending" | "no_destination";

export interface SendDecisionInput {
  kind: MessageKind;
  hasDestination: boolean;
  /** True if the number appears in opt_outs OR the legacy reviewOptOuts list. */
  optedOut: boolean;
  /** comms_provisioning.a2pCampaignStatus === "approved". */
  a2pApproved: boolean;
  /** True if 8:00–20:59 in the client's timezone (only consulted for marketing). */
  withinSendWindow: boolean;
  /** True if no marketing message went to this number within the cap window. */
  underFrequencyCap: boolean;
}

export type SendDecision =
  | { action: "send" }
  | { action: "queue"; reason: "quiet_hours" }
  | { action: "block"; reason: BlockedReason };

/**
 * Hard blocks are evaluated before the quiet-hours queue decision so we never
 * queue a message that could never legally send. Opt-out and A2P apply to
 * every kind; frequency cap and quiet hours are marketing-only (doc 02).
 */
export function decideSend(i: SendDecisionInput): SendDecision {
  if (!i.hasDestination) return { action: "block", reason: "no_destination" };
  if (i.optedOut) return { action: "block", reason: "opt_out" };
  if (!i.a2pApproved) return { action: "block", reason: "a2p_pending" };
  if (i.kind === "marketing") {
    if (!i.underFrequencyCap) return { action: "block", reason: "freq_cap" };
    if (!i.withinSendWindow) return { action: "queue", reason: "quiet_hours" };
  }
  return { action: "send" };
}

// --- Quiet hours (TCPA-safe: 8:00 inclusive – 21:00 exclusive, client-local) ---

export const QUIET_START = 8;
export const QUIET_END = 21;
export const DEFAULT_FREQ_CAP_DAYS = 30;

function localParts(timezone: string, now: Date): { hour: number; minute: number } {
  const parts = new Intl.DateTimeFormat("en-US", {
    timeZone: timezone,
    hour: "numeric",
    minute: "numeric",
    hour12: false,
  }).formatToParts(now);
  const get = (t: string) => parseInt(parts.find((p) => p.type === t)?.value ?? "0", 10);
  // Intl can render "24" for midnight in some environments; normalize.
  return { hour: get("hour") % 24, minute: get("minute") };
}

export function withinSendWindow(timezone: string, now: Date): boolean {
  const { hour } = localParts(timezone, now);
  return hour >= QUIET_START && hour < QUIET_END;
}

/**
 * When the send window next opens (8:00 client-local). Used as pg-boss
 * `startAfter`. This is scheduling, not enforcement — performSend() re-checks
 * withinSendWindow() before actually sending, so a DST-shifted early wake-up
 * simply re-queues. +5 min margin keeps a boundary wake from landing at 07:59.
 */
export function nextSendWindow(timezone: string, now: Date): Date {
  if (withinSendWindow(timezone, now)) return now;
  const { hour, minute } = localParts(timezone, now);
  const minutesPastMidnight = hour * 60 + minute;
  const target = QUIET_START * 60;
  const minutesUntil =
    minutesPastMidnight < target
      ? target - minutesPastMidnight
      : 24 * 60 - minutesPastMidnight + target;
  return new Date(now.getTime() + (minutesUntil + 5) * 60_000);
}

// --- Phone normalization (NANP-first; the pilot market is US/CA) ---

/** Normalize to E.164 or return null (null → blocked no_destination). */
export function toE164(raw: string): string | null {
  const trimmed = raw.trim();
  if (trimmed.startsWith("+")) {
    const rest = trimmed.slice(1).replace(/\D/g, "");
    return /^\d{8,15}$/.test(rest) ? `+${rest}` : null;
  }
  const bare = trimmed.replace(/\D/g, "");
  if (bare.length === 10) return `+1${bare}`;
  if (bare.length === 11 && bare.startsWith("1")) return `+${bare}`;
  return null;
}
```

`src/server/messaging/compliance.test.ts` (vitest, mirrors `reviews/compliance.test.ts`): table-driven tests over `decideSend` covering every branch × every kind, plus `withinSendWindow`/`nextSendWindow` at 07:59 / 08:00 / 20:59 / 21:00 in `America/Toronto` and `America/Los_Angeles` with fixed `Date` inputs, plus `toE164` cases (`"(613) 555-0142"` → `"+16135550142"`, `"+447911123456"` passthrough, `"555-0142"` → null).

**Acceptance criteria**

1. `decideSend` returns `block/opt_out` for all three kinds when `optedOut` — including `conversational` and `transactional`.
2. `decideSend` returns `block/a2p_pending` for all three kinds when `!a2pApproved`, even when everything else passes.
3. `marketing` outside the window → `queue/quiet_hours`; `transactional`/`conversational` outside the window → `send`.
4. `marketing` over cap → `block/freq_cap` even inside the window; over-cap + outside-window → `block/freq_cap` (block wins over queue).
5. 100% branch coverage of `decideSend` in `npm test`.

---

## SP-3 — Pipeline core + pg-boss queue

### Queue registration — `src/server/pipeline/queue.ts`

Add to the existing constants (same file, same patterns):

```ts
export const QUEUES = {
  // ...existing...
  messageSend: "message-send",
} as const;

const RETRY_QUEUES = {
  // ...existing...
  // Twilio 429/5xx retries: 3 tries, 60s base, exponential (doc 02 failure policy).
  messageSend: { retryLimit: 3, retryDelay: 60, retryBackoff: true },
};

export interface MessageSendJob {
  messageId: string;
}
```

(The existing `getBoss()` loop auto-creates `message-send` and `message-send-dlq` — no other change.)

### Worker — `src/worker.ts`

```ts
import { performSend } from "./server/messaging/send-pipeline";
// inside main(), with the other handlers:
await boss.work<MessageSendJob>(QUEUES.messageSend, async ([job]) => {
  await performSend(job.data.messageId);
});
```

### `src/server/messaging/send-pipeline.ts` — complete implementation

```ts
/**
 * The shared send-pipeline: THE single compliance path for all outbound SMS
 * (doc 03 §shared send-pipeline). Callers: review engine (in-process), n8n via
 * POST /api/internal/send. Flow:
 *
 *   sendMessage(): gates (read-only) → idempotent ledger insert → either
 *   blocked (audited), queued via pg-boss startAfter (quiet hours), or
 *   performSend() inline.
 *
 *   performSend(): claims the row (queued → sending, the double-send guard),
 *   RE-CHECKS opt-out / A2P / quiet hours (a STOP may have arrived while
 *   queued — fail closed at the last moment), then Twilio send via the
 *   client's subaccount.
 */
import { and, eq, gte, inArray, sql } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, commsProvisioning, messages, optOuts, reviewOptOuts } from "@/server/db/schema";
import { getBoss, QUEUES } from "@/server/pipeline/queue";
import { hashContact } from "@/server/reviews/compliance"; // legacy suppression list
import { log, logError } from "@/lib/logger";
import {
  decideSend,
  DEFAULT_FREQ_CAP_DAYS,
  nextSendWindow,
  toE164,
  withinSendWindow,
  type MessageKind,
} from "./compliance";
import { twilioSendSms, TwilioRateLimitedError } from "./twilio";

export interface SendRequest {
  clientId: string;
  /** Raw phone; normalized to E.164 internally. */
  to: string;
  body: string;
  kind: MessageKind;
  /** "review_velocity" | "speed_to_lead" | "reactivation" | ... */
  engine: string;
  /** Caller-supplied, globally unique per logical send. THE idempotency contract. */
  idempotencyKey: string;
  customerId?: string;
  campaignId?: string;
  conversationId?: string;
  mediaUrls?: string[];
}

export interface SendResult {
  messageId: string;
  /** "sent" | "queued" | "blocked" | "failed" | "duplicate" */
  outcome: "sent" | "queued" | "blocked" | "failed" | "duplicate";
  status: string;
  blockedReason?: string;
}

// Per-client overrides land in workflow_config.settings (spec 01) later; until
// then the doc-02 defaults are compiled in so behavior is deterministic.
const FREQ_CAP_DAYS = DEFAULT_FREQ_CAP_DAYS;

async function isOptedOut(clientId: string, phone: string): Promise<boolean> {
  const [modern] = await db
    .select({ phone: optOuts.phone })
    .from(optOuts)
    .where(and(eq(optOuts.clientId, clientId), eq(optOuts.phone, phone)))
    .limit(1);
  if (modern) return true;
  // Legacy hashed suppression list (pre-pipeline STOPs). Global by design —
  // one shared sender served all clients then.
  const [legacy] = await db
    .select({ id: reviewOptOuts.id })
    .from(reviewOptOuts)
    .where(
      and(eq(reviewOptOuts.channel, "sms"), eq(reviewOptOuts.contactHash, hashContact("sms", phone))),
    )
    .limit(1);
  return Boolean(legacy);
}

async function underFrequencyCap(clientId: string, phone: string): Promise<boolean> {
  const cutoff = new Date(Date.now() - FREQ_CAP_DAYS * 24 * 60 * 60 * 1000);
  const [row] = await db
    .select({ id: messages.id })
    .from(messages)
    .where(
      and(
        eq(messages.clientId, clientId),
        eq(messages.toPhone, phone),
        eq(messages.direction, "out"),
        eq(messages.kind, "marketing"),
        // Blocked/failed sends never reached the customer; they don't consume cap.
        inArray(messages.status, ["queued", "sending", "sent", "delivered", "undelivered"]),
        gte(messages.createdAt, cutoff),
      ),
    )
    .limit(1);
  return !row;
}

async function loadClientContext(clientId: string) {
  const [row] = await db
    .select({
      timezone: clients.timezone,
      provisioning: commsProvisioning,
    })
    .from(clients)
    .leftJoin(commsProvisioning, eq(commsProvisioning.clientId, clients.id))
    .where(eq(clients.id, clientId));
  if (!row) throw new Error(`sendMessage: unknown client ${clientId}`);
  return row;
}

/** Entry point. Never throws for compliance outcomes — only for infra errors. */
export async function sendMessage(req: SendRequest, now: Date = new Date()): Promise<SendResult> {
  const ctx = await loadClientContext(req.clientId);
  const phone = toE164(req.to);
  const fromPhone = ctx.provisioning?.phoneNumber ?? "";

  const decision = decideSend({
    kind: req.kind,
    hasDestination: Boolean(phone) && Boolean(fromPhone),
    optedOut: phone ? await isOptedOut(req.clientId, phone) : false,
    a2pApproved: ctx.provisioning?.a2pCampaignStatus === "approved",
    withinSendWindow: withinSendWindow(ctx.timezone, now),
    underFrequencyCap: phone ? await underFrequencyCap(req.clientId, phone) : true,
  });

  // Idempotent insert: ON CONFLICT DO NOTHING + RETURNING. No row returned
  // means the key already exists → return the existing row untouched (never
  // re-send, never re-evaluate: the first decision stands).
  const initialStatus = decision.action === "block" ? "blocked" : "queued";
  const [inserted] = await db
    .insert(messages)
    .values({
      clientId: req.clientId,
      customerId: req.customerId,
      direction: "out",
      kind: req.kind,
      engine: req.engine,
      toPhone: phone ?? req.to,
      fromPhone,
      body: req.body,
      mediaUrls: req.mediaUrls,
      status: initialStatus,
      blockedReason: decision.action === "block" ? decision.reason : null,
      idempotencyKey: req.idempotencyKey,
      campaignId: req.campaignId,
      conversationId: req.conversationId,
    })
    .onConflictDoNothing({ target: messages.idempotencyKey })
    .returning({ id: messages.id });

  if (!inserted) {
    const [existing] = await db
      .select({ id: messages.id, status: messages.status, blockedReason: messages.blockedReason })
      .from(messages)
      .where(eq(messages.idempotencyKey, req.idempotencyKey));
    log(`[send-pipeline] duplicate idempotencyKey ${req.idempotencyKey} → ${existing.id}`);
    return {
      messageId: existing.id,
      outcome: "duplicate",
      status: existing.status,
      blockedReason: existing.blockedReason ?? undefined,
    };
  }

  if (decision.action === "block") {
    log(`[send-pipeline] blocked (${decision.reason}) client=${req.clientId} key=${req.idempotencyKey}`);
    return { messageId: inserted.id, outcome: "blocked", status: "blocked", blockedReason: decision.reason };
  }

  if (decision.action === "queue") {
    // Quiet hours: queue-not-drop (doc 02). pg-boss startAfter wakes the worker
    // at the next window open; singletonKey = messageId prevents double jobs.
    const startAfter = nextSendWindow(ctx.timezone, now);
    const boss = await getBoss();
    await boss.send(QUEUES.messageSend, { messageId: inserted.id }, {
      startAfter,
      singletonKey: inserted.id,
    });
    log(`[send-pipeline] quiet hours → queued ${inserted.id} until ${startAfter.toISOString()}`);
    return { messageId: inserted.id, outcome: "queued", status: "queued" };
  }

  return performSend(inserted.id, now);
}

/**
 * Actually transmit one queued ledger row. Called inline (in-window sends) and
 * by the message-send worker (quiet-hours deferrals + 429 retries). Safe to
 * call repeatedly: the queued→sending conditional UPDATE is the double-send guard.
 */
export async function performSend(messageId: string, now: Date = new Date()): Promise<SendResult> {
  const [msg] = await db.select().from(messages).where(eq(messages.id, messageId));
  if (!msg) throw new Error(`performSend: no message ${messageId}`);
  if (msg.status !== "queued") {
    return { messageId, outcome: msg.status === "blocked" ? "blocked" : "sent", status: msg.status };
  }

  const ctx = await loadClientContext(msg.clientId);

  // Re-check the world at send time — fail closed (doc 07: STOP honored
  // instantly, zero sends before A2P approval; both pipeline-enforced).
  if (await isOptedOut(msg.clientId, msg.toPhone)) {
    await db.update(messages)
      .set({ status: "blocked", blockedReason: "opt_out" })
      .where(eq(messages.id, messageId));
    return { messageId, outcome: "blocked", status: "blocked", blockedReason: "opt_out" };
  }
  if (ctx.provisioning?.a2pCampaignStatus !== "approved") {
    await db.update(messages)
      .set({ status: "blocked", blockedReason: "a2p_pending" })
      .where(eq(messages.id, messageId));
    return { messageId, outcome: "blocked", status: "blocked", blockedReason: "a2p_pending" };
  }
  if (msg.kind === "marketing" && !withinSendWindow(ctx.timezone, now)) {
    // Woke up early (DST / retry drift): push back to the next window.
    const boss = await getBoss();
    await boss.send(QUEUES.messageSend, { messageId }, {
      startAfter: nextSendWindow(ctx.timezone, now),
      singletonKey: messageId,
    });
    return { messageId, outcome: "queued", status: "queued" };
  }

  // Claim: only one worker wins the queued → sending transition.
  const claimed = await db
    .update(messages)
    .set({ status: "sending" })
    .where(and(eq(messages.id, messageId), eq(messages.status, "queued")))
    .returning({ id: messages.id });
  if (claimed.length === 0) {
    return { messageId, outcome: "duplicate", status: "sending" };
  }

  try {
    const { sid } = await twilioSendSms({
      subaccountSid: ctx.provisioning.twilioSubaccountSid ?? undefined,
      from: ctx.provisioning.phoneNumber!,
      to: msg.toPhone,
      body: msg.body,
      mediaUrls: msg.mediaUrls ?? undefined,
    });
    await db.update(messages)
      .set({ status: "sent", twilioSid: sid, sentAt: now })
      .where(eq(messages.id, messageId));
    return { messageId, outcome: "sent", status: "sent" };
  } catch (err) {
    if (err instanceof TwilioRateLimitedError) {
      // Release the claim and let pg-boss retry with backoff (SP-9).
      await db.update(messages).set({ status: "queued" }).where(eq(messages.id, messageId));
      throw err;
    }
    logError(`[send-pipeline] Twilio send failed for ${messageId}:`, err);
    await db.update(messages)
      .set({
        status: "failed",
        errorCode: err instanceof TwilioApiError ? String(err.code) : null,
      })
      .where(eq(messages.id, messageId));
    return { messageId, outcome: "failed", status: "failed" };
  }
}
```

(`TwilioApiError` / `TwilioRateLimitedError` are defined in SP-4; import both.)

**Acceptance criteria**

1. Two concurrent `sendMessage()` calls with the same `idempotencyKey` produce exactly one `messages` row and at most one Twilio API call (test with mocked `fetch` + `Promise.all`).
2. A `marketing` send at 23:00 client-local creates a `queued` row and a pg-boss job whose `startAfter` is 08:00–08:06 client-local the next morning; **no** Twilio call is made; the row is never deleted.
3. `performSend` on a row whose number was opted out after queueing flips it to `blocked/opt_out` without calling Twilio.
4. `performSend` called twice for the same id makes one Twilio call (queued→sending claim).
5. A `conversational` send at 23:00 goes out immediately.
6. The `message-send` worker handler is registered in `src/worker.ts` and drains queued jobs (manual check: `npm run worker`, insert job, observe log line).

---

## SP-4 — Twilio REST client + status callback

### `src/server/messaging/twilio.ts`

Same no-SDK raw-REST style as `src/server/reviews/messaging.ts`. Master credentials (`TWILIO_ACCOUNT_SID`/`TWILIO_AUTH_TOKEN`) are authorized on subaccount resources under `api.twilio.com` when the subaccount SID is in the URL path.

```ts
import { appUrl } from "@/env";

export class TwilioApiError extends Error {
  constructor(public httpStatus: number, public code: number | null, message: string) {
    super(message);
  }
}
/** HTTP 429 or Twilio error 20429 — retryable via pg-boss backoff. */
export class TwilioRateLimitedError extends TwilioApiError {}

function authHeader(): string {
  const sid = process.env.TWILIO_ACCOUNT_SID!;
  const token = process.env.TWILIO_AUTH_TOKEN!;
  return "Basic " + Buffer.from(`${sid}:${token}`).toString("base64");
}

export async function twilioSendSms(input: {
  /** Client subaccount; falls back to the master account (legacy shared number). */
  subaccountSid?: string;
  from: string;
  to: string;
  body: string;
  mediaUrls?: string[];
}): Promise<{ sid: string }> {
  const account = input.subaccountSid ?? process.env.TWILIO_ACCOUNT_SID!;
  const form = new URLSearchParams({
    To: input.to,
    From: input.from,
    Body: input.body,
    StatusCallback: `${appUrl}/api/messaging/twilio/status`,
  });
  for (const url of input.mediaUrls ?? []) form.append("MediaUrl", url);

  const res = await fetch(`https://api.twilio.com/2010-04-01/Accounts/${account}/Messages.json`, {
    method: "POST",
    headers: { Authorization: authHeader(), "Content-Type": "application/x-www-form-urlencoded" },
    body: form.toString(),
    signal: AbortSignal.timeout(30_000),
  });

  const payload = (await res.json().catch(() => ({}))) as {
    sid?: string; code?: number; message?: string;
  };
  if (res.status === 429 || payload.code === 20429) {
    throw new TwilioRateLimitedError(res.status, payload.code ?? null, "Twilio rate limited");
  }
  if (!res.ok || !payload.sid) {
    throw new TwilioApiError(res.status, payload.code ?? null,
      `Twilio ${res.status}: ${(payload.message ?? "").slice(0, 200)}`);
  }
  return { sid: payload.sid };
}
```

### Shared signature helper — `src/server/messaging/twilio-signature.ts`

Extract the HMAC-SHA1 validator currently inlined in `src/app/api/reviews/sms/inbound/route.ts` (identical logic, one source of truth):

```ts
import { createHmac } from "node:crypto";

/** Twilio's scheme: HMAC-SHA1 over (url + sorted POST key+value pairs). */
export function validTwilioSignature(
  url: string,
  params: Record<string, string>,
  header: string | null,
): boolean {
  const token = process.env.TWILIO_AUTH_TOKEN;
  if (!token || !header) return false;
  const data = url + Object.keys(params).sort().map((k) => k + params[k]).join("");
  const expected = createHmac("sha1", token).update(data).digest("base64");
  return expected === header;
}
```

Update the legacy inbound route to import this instead of its local copy.

> Note: callbacks for messages sent via a **subaccount** are signed with the **subaccount's** auth token. `validTwilioSignature` must therefore also be tried with `commsProvisioning.twilioSubaccountAuthToken` when the master check fails — the status route below resolves the row by `MessageSid` first, then validates against the owning account's token, falling back to the master token.

### Status callback route — `src/app/api/messaging/twilio/status/route.ts`

```ts
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { commsProvisioning, messages, optOuts } from "@/server/db/schema";
import { validTwilioSignature } from "@/server/messaging/twilio-signature";
import { createHmac } from "node:crypto";
import { log, logError } from "@/lib/logger";

// Callbacks arrive out of order; never regress a terminal status.
const STATUS_RANK: Record<string, number> = {
  queued: 0, sending: 1, sent: 2, delivered: 3, undelivered: 3, failed: 3,
};
const TWILIO_TO_LEDGER: Record<string, string> = {
  queued: "sent",      // accepted by Twilio — we already recorded "sent"
  accepted: "sent",
  sending: "sent",
  sent: "sent",
  delivered: "delivered",
  undelivered: "undelivered",
  failed: "failed",
};

function signedWith(url: string, params: Record<string, string>, header: string | null, token: string): boolean {
  if (!header) return false;
  const data = url + Object.keys(params).sort().map((k) => k + params[k]).join("");
  return createHmac("sha1", token).update(data).digest("base64") === header;
}

export async function POST(request: Request) {
  const form = await request.formData();
  const params: Record<string, string> = {};
  for (const [k, v] of form.entries()) params[k] = String(v);

  const sid = params.MessageSid ?? "";
  const twStatus = params.MessageStatus ?? "";
  if (!sid || !twStatus) return new Response(null, { status: 204 });

  const [msg] = await db.select().from(messages).where(eq(messages.twilioSid, sid));
  if (!msg) return new Response(null, { status: 204 }); // not ours (or race) — ack anyway

  // Validate against master token, else the owning subaccount's token.
  const header = request.headers.get("x-twilio-signature");
  let verified = validTwilioSignature(request.url, params, header);
  if (!verified) {
    const [prov] = await db.select().from(commsProvisioning)
      .where(eq(commsProvisioning.clientId, msg.clientId));
    if (prov?.twilioSubaccountAuthToken) {
      verified = signedWith(request.url, params, header, prov.twilioSubaccountAuthToken);
    }
  }
  if (!verified) return new Response("bad signature", { status: 403 });

  const mapped = TWILIO_TO_LEDGER[twStatus];
  if (!mapped) return new Response(null, { status: 204 });
  if ((STATUS_RANK[mapped] ?? 0) <= (STATUS_RANK[msg.status] ?? 0)) {
    return new Response(null, { status: 204 }); // stale/out-of-order callback
  }

  const errorCode = params.ErrorCode || null;
  await db.update(messages)
    .set({
      status: mapped as typeof msg.status,
      errorCode,
      deliveredAt: mapped === "delivered" ? new Date() : msg.deliveredAt,
    })
    .where(eq(messages.id, msg.id));

  // 21610: recipient is on Twilio's own STOP list — mirror into our ledger so
  // the pipeline blocks future sends without needing another attempt.
  if (errorCode === "21610") {
    await db.insert(optOuts)
      .values({ clientId: msg.clientId, phone: msg.toPhone, source: "carrier_error", sourceMessageId: msg.id })
      .onConflictDoNothing();
  }
  if (mapped === "undelivered" || mapped === "failed") {
    logError(`[send-pipeline] ${mapped} sid=${sid} error=${errorCode ?? "none"} client=${msg.clientId}`);
  } else {
    log(`[send-pipeline] status ${twStatus} sid=${sid}`);
  }
  return new Response(null, { status: 204 });
}
```

**Acceptance criteria**

1. Send with a valid subaccount SID hits `https://api.twilio.com/2010-04-01/Accounts/{subSid}/Messages.json` with master Basic auth (assert via mocked `fetch`).
2. HTTP 429 and Twilio code 20429 raise `TwilioRateLimitedError`; other non-2xx raise `TwilioApiError` with the Twilio `code` captured.
3. Status callback with a bad signature → 403, ledger untouched.
4. `delivered` after `sent` updates the row + `deliveredAt`; a late `sent` after `delivered` is a no-op (rank guard).
5. Callback with `ErrorCode=21610` inserts an `opt_outs` row.

---

## SP-5 — `POST /api/internal/send` (n8n entry point)

`src/app/api/internal/send/route.ts`. Auth: existing `requireServiceToken` (`SPINE_SERVICE_TOKEN` — the same service token n8n uses for spine routes; if spec 02 introduces a dedicated `INTERNAL_API_TOKEN` helper, swap the import, nothing else changes).

```ts
import { z } from "zod";
import { requireServiceToken } from "@/server/spine/auth";
import { sendMessage } from "@/server/messaging/send-pipeline";
import { logError } from "@/lib/logger";

const BodySchema = z.object({
  client_id: z.string().uuid(),
  to: z.string().min(7),
  body: z.string().min(1).max(1600),
  kind: z.enum(["transactional", "conversational", "marketing"]),
  engine: z.string().min(1),
  idempotency_key: z.string().min(8).max(200),
  customer_id: z.string().uuid().optional(),
  campaign_id: z.string().uuid().optional(),
  conversation_id: z.string().uuid().optional(),
  media_urls: z.array(z.string().url()).max(10).optional(),
});

export async function POST(request: Request) {
  const unauth = requireServiceToken(request);
  if (unauth) return unauth;

  let raw: unknown;
  try {
    raw = await request.json();
  } catch {
    return Response.json({ error: "Invalid JSON body" }, { status: 400 });
  }
  const parsed = BodySchema.safeParse(raw);
  if (!parsed.success) {
    return Response.json({ error: parsed.error.issues }, { status: 400 });
  }
  const b = parsed.data;
  try {
    const result = await sendMessage({
      clientId: b.client_id,
      to: b.to,
      body: b.body,
      kind: b.kind,
      engine: b.engine,
      idempotencyKey: b.idempotency_key,
      customerId: b.customer_id,
      campaignId: b.campaign_id,
      conversationId: b.conversation_id,
      mediaUrls: b.media_urls,
    });
    // Blocked/queued/duplicate are SUCCESSFUL pipeline outcomes (200), not
    // errors — n8n must not retry them.
    return Response.json({
      message_id: result.messageId,
      outcome: result.outcome,
      status: result.status,
      blocked_reason: result.blockedReason ?? null,
    });
  } catch (err) {
    logError("[internal/send] pipeline error:", err);
    return Response.json({ error: "send pipeline failure" }, { status: 500 });
  }
}
```

**n8n contract** (documented in `seekly-platform/workflows/README` when workflows are exported): every engine node calls this route with `idempotency_key = "<event_id>:<step>"`; a 200 with `outcome: "blocked"` must be written to the activity log by the workflow, never retried; only 5xx are retried (3×, exponential).

**Acceptance criteria**

1. Missing/wrong bearer token → 401 with no DB writes.
2. Valid request returns 200 with `message_id` and one of `sent|queued|blocked|duplicate|failed`.
3. Replaying the same body twice returns `outcome: "duplicate"` and the same `message_id` the second time.
4. Invalid `kind` or non-UUID `client_id` → 400 with zod issues.

---

## SP-6 — Global inbound handler

New route `src/app/api/messaging/twilio/inbound/route.ts` + logic module `src/server/messaging/inbound.ts`. Every **new per-client number** gets this route as its `SmsUrl` (set at purchase, SP-7). The legacy route `/api/reviews/sms/inbound` stays wired to the legacy shared number until it is retired; as part of this ticket, additionally point the shared number's `SmsUrl` at the new route once SP-8 ships (the new route also writes the legacy hashed table, so behavior is a strict superset).

### `src/server/messaging/inbound.ts`

```ts
/**
 * Global inbound SMS handling: carrier-required keywords (STOP/HELP/START),
 * cross-engine opt-out writes, message.received canonical events, and the
 * STOP-rate campaign auto-pause (doc 03 WF-3 failure handling).
 */
import { and, desc, eq, gte, isNotNull, sql } from "drizzle-orm";
import { db } from "@/server/db";
import { campaigns, clients, commsProvisioning, messages, optOuts } from "@/server/db/schema";
import { recordOptOut, removeOptOut } from "@/server/reviews/engine"; // legacy hashed list stays in sync
import { emitCanonicalEvent } from "@/server/events/emit"; // spec 02; see note below
import { sendOpsAlert } from "./ops-alert";
import { log } from "@/lib/logger";

export const STOP_KEYWORDS = ["STOP", "STOPALL", "UNSUBSCRIBE", "CANCEL", "END", "QUIT"];
export const START_KEYWORDS = ["START", "UNSTOP", "YES"];
export const HELP_KEYWORDS = ["HELP", "INFO"];

const STOP_RATE_THRESHOLD = 0.03; // doc 03: >3% of a batch
const STOP_RATE_MIN_SENDS = 25;   // don't pause a 3-message test batch on one STOP

export async function resolveClientByNumber(toNumber: string) {
  const [row] = await db
    .select({
      clientId: commsProvisioning.clientId,
      businessName: clients.name,
    })
    .from(commsProvisioning)
    .innerJoin(clients, eq(clients.id, commsProvisioning.clientId))
    .where(eq(commsProvisioning.phoneNumber, toNumber));
  return row ?? null;
}

/** Carrier-required copy. Twilio appends nothing for us — we own the text. */
export function stopConfirmation(businessName: string): string {
  return `${businessName}: You have been unsubscribed and will receive no further messages. Reply START to resubscribe.`;
}
export function helpResponse(businessName: string): string {
  return `${businessName} via Seekly: for help visit your provider or reply to this number. Msg&Data rates may apply. Reply STOP to unsubscribe.`;
}
export function startConfirmation(businessName: string): string {
  return `${businessName}: You are resubscribed. Reply STOP to unsubscribe. Msg&Data rates may apply.`;
}

export async function recordInboundMessage(input: {
  clientId: string; from: string; to: string; body: string; twilioSid: string;
}): Promise<string | null> {
  const [row] = await db
    .insert(messages)
    .values({
      clientId: input.clientId,
      direction: "in",
      kind: "conversational",
      engine: "inbound",
      toPhone: input.from,   // ledger convention: toPhone = the customer's number
      fromPhone: input.to,
      body: input.body,
      status: "delivered",
      idempotencyKey: `twilio-inbound:${input.twilioSid}`,
    })
    .onConflictDoNothing({ target: messages.idempotencyKey })
    .returning({ id: messages.id });
  return row?.id ?? null; // null = webhook redelivery, already recorded
}

export async function handleStop(clientId: string, phone: string, messageId: string | null) {
  await db.insert(optOuts)
    .values({ clientId, phone, source: "sms_stop", sourceMessageId: messageId })
    .onConflictDoNothing();
  await recordOptOut({ channel: "sms", value: phone, clientId }); // keep legacy list in sync
  await maybePauseCampaignForStop(clientId, phone);
}

export async function handleStart(clientId: string, phone: string) {
  await db.delete(optOuts).where(and(eq(optOuts.clientId, clientId), eq(optOuts.phone, phone)));
  await removeOptOut("sms", phone);
}

/**
 * STOP-rate auto-pause: attribute the STOP to the most recent outbound
 * campaign message to this number (7-day window), bump the campaign's stop
 * count, and pause the campaign when stops/sent exceeds 3% with a meaningful
 * sample. Crash-safe: counts are recomputed from the messages ledger, not
 * incremented blindly.
 */
export async function maybePauseCampaignForStop(clientId: string, phone: string) {
  const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const [last] = await db
    .select({ campaignId: messages.campaignId })
    .from(messages)
    .where(and(
      eq(messages.clientId, clientId),
      eq(messages.toPhone, phone),
      eq(messages.direction, "out"),
      isNotNull(messages.campaignId),
      gte(messages.createdAt, weekAgo),
    ))
    .orderBy(desc(messages.createdAt))
    .limit(1);
  if (!last?.campaignId) return;

  const [counts] = await db
    .select({
      sent: sql<number>`count(*) filter (where ${messages.direction} = 'out'
        and ${messages.status} in ('sent','delivered','undelivered'))`,
      stops: sql<number>`(select count(*) from ${optOuts} o
        where o.client_id = ${clientId}
        and o.source_message_id in
          (select id from ${messages} m2 where m2.campaign_id = ${last.campaignId}))`,
    })
    .from(messages)
    .where(eq(messages.campaignId, last.campaignId));

  const sent = Number(counts?.sent ?? 0);
  const stops = Number(counts?.stops ?? 0) + 1; // +1: this STOP's opt_outs row may not reference a campaign message yet
  if (sent >= STOP_RATE_MIN_SENDS && stops / sent > STOP_RATE_THRESHOLD) {
    await db.update(campaigns)
      .set({ status: "paused" })
      .where(and(eq(campaigns.id, last.campaignId), eq(campaigns.status, "sending")));
    log(`[inbound] STOP-rate auto-pause campaign=${last.campaignId} stops=${stops} sent=${sent}`);
    await sendOpsAlert(
      `Campaign auto-paused (STOP rate ${(100 * stops / sent).toFixed(1)}%)`,
      `Campaign ${last.campaignId} for client ${clientId}: ${stops} STOPs over ${sent} sends. Paused automatically (doc 03 >3% rule).`,
    );
  }
}
```

> `campaigns` is a spec-01 table (doc 04). If it hasn't landed yet, guard the auto-pause with a table-exists check is NOT acceptable — instead land the minimal `campaigns` table (id, clientId, status) in this ticket's migration and let spec 01 extend it.
>
> `emitCanonicalEvent` comes from spec 02 (`src/server/events/emit.ts`, signature `{ eventId, clientId, type, occurredAt, source, payload }` → idempotent insert into `events` + forward to the type's n8n webhook). If spec 02 hasn't merged, implement a stub in that path that only inserts the `events` row and logs — the call-site contract here must not change.

### `src/server/messaging/ops-alert.ts`

```ts
/** Best-effort ops email via Resend (same raw-REST style as reviews/messaging.ts). */
import { log, logError } from "@/lib/logger";

export async function sendOpsAlert(subject: string, text: string): Promise<void> {
  const to = process.env.OPS_ALERT_EMAIL;
  if (!to || !process.env.RESEND_API_KEY || !process.env.REPORT_EMAIL_FROM) {
    log(`[ops-alert] (email not configured) ${subject}: ${text}`);
    return;
  }
  try {
    const res = await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.RESEND_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        from: process.env.REPORT_EMAIL_FROM,
        to: [to],
        subject: `[Seekly ops] ${subject}`,
        text,
      }),
      signal: AbortSignal.timeout(15_000),
    });
    if (!res.ok) throw new Error(`Resend ${res.status}`);
  } catch (err) {
    logError("[ops-alert] failed to send:", err); // alerts must never break the pipeline
  }
}
```

### Route — `src/app/api/messaging/twilio/inbound/route.ts`

```ts
import { db } from "@/server/db";
import { commsProvisioning } from "@/server/db/schema";
import { eq } from "drizzle-orm";
import { validTwilioSignature } from "@/server/messaging/twilio-signature";
import {
  HELP_KEYWORDS, START_KEYWORDS, STOP_KEYWORDS,
  handleStart, handleStop, helpResponse, recordInboundMessage,
  resolveClientByNumber, startConfirmation, stopConfirmation,
} from "@/server/messaging/inbound";
import { emitCanonicalEvent } from "@/server/events/emit";
import { createHmac } from "node:crypto";

const twiml = (message?: string) =>
  new Response(
    `<?xml version="1.0" encoding="UTF-8"?><Response>${
      message ? `<Message>${message.replace(/&/g, "&amp;").replace(/</g, "&lt;")}</Message>` : ""
    }</Response>`,
    { headers: { "content-type": "text/xml" } },
  );

export async function POST(request: Request) {
  const form = await request.formData();
  const params: Record<string, string> = {};
  for (const [k, v] of form.entries()) params[k] = String(v);

  const from = params.From ?? "";
  const to = params.To ?? "";
  const body = (params.Body ?? "").trim();
  const keyword = body.toUpperCase();
  const sid = params.MessageSid ?? "";
  if (!from || !to || !sid) return twiml();

  // Inbound webhooks for a subaccount number are signed with the SUBACCOUNT
  // token. Resolve client first, then verify against sub token, falling back
  // to the master token (legacy shared number).
  const client = await resolveClientByNumber(to);
  if (!client) return twiml(); // unknown number — ack so Twilio stops retrying

  const header = request.headers.get("x-twilio-signature");
  let verified = validTwilioSignature(request.url, params, header);
  if (!verified) {
    const [prov] = await db.select().from(commsProvisioning)
      .where(eq(commsProvisioning.clientId, client.clientId));
    const token = prov?.twilioSubaccountAuthToken;
    if (token && header) {
      const data = request.url + Object.keys(params).sort().map((k) => k + params[k]).join("");
      verified = createHmac("sha1", token).update(data).digest("base64") === header;
    }
  }

  const messageId = await recordInboundMessage({
    clientId: client.clientId, from, to, body, twilioSid: sid,
  });

  if (STOP_KEYWORDS.includes(keyword)) {
    // Adding suppression is safe even unverified (fail closed on the risky
    // direction — same reasoning as the legacy route).
    await handleStop(client.clientId, from, messageId);
    return twiml(stopConfirmation(client.businessName));
  }
  if (START_KEYWORDS.includes(keyword)) {
    if (verified) await handleStart(client.clientId, from); // removal requires a valid signature
    return twiml(startConfirmation(client.businessName));
  }
  if (HELP_KEYWORDS.includes(keyword)) {
    return twiml(helpResponse(client.businessName));
  }

  // Non-keyword: canonical message.received → conversation router (WF-2/WF-3).
  if (verified && messageId) {
    await emitCanonicalEvent({
      eventId: messageId, // ledger row id doubles as the idempotency key
      clientId: client.clientId,
      type: "message.received",
      occurredAt: new Date().toISOString(),
      source: "webhook",
      payload: { phone: from, body, twilio_sid: sid },
    });
  }
  return twiml(); // no auto-reply; the AI conversation loop answers via the pipeline
}
```

**Twilio console note (document in the ticket):** on each Messaging Service, set Opt-Out Management to **Advanced Opt-Out disabled / default off** is *not* available via API for all account types — where Twilio's default STOP filtering is active it will additionally block sends at the carrier layer and may send its own STOP confirmation. That is a harmless second layer; our ledger remains the source of truth and our TwiML confirmation is suppressed by Twilio when it already replied. Verify actual behavior on the pilot number during SP-10 and record which layer answered.

**Acceptance criteria**

1. POST with `Body=STOP` writes an `opt_outs` row for the resolved client AND a `reviewOptOuts` hash row, replies with TwiML containing the confirmation copy, and a subsequent `sendMessage` of any kind to that number is `blocked/opt_out`.
2. `Body=HELP` returns TwiML with business name, "Msg&Data rates may apply", and "Reply STOP".
3. `Body=START` with an invalid signature does NOT remove the opt-out; with a valid signature it removes both ledger entries.
4. Non-keyword body inserts a `direction=in` ledger row and one `events` row of type `message.received` (idempotent on webhook redelivery — same `MessageSid` twice → one of each).
5. Unknown `To` number → empty TwiML 200, no writes.
6. `STOPALL`, `unsubscribe`, `Quit` (case-insensitive) all count as STOP.

---

## SP-7 — Twilio subaccount provisioning helper

`src/server/messaging/provisioning.ts`. Called by WF-0 provisioning (n8n → a thin internal route, or the admin script `scripts/provision-comms.ts`). All calls raw REST with `TWILIO_ACCOUNT_SID`/`TWILIO_AUTH_TOKEN`.

```ts
/**
 * Twilio comms provisioning: subaccount → local number → messaging service →
 * A2P brand + campaign submission → status polling. Doc 06: "subaccount per
 * client (isolation, per-client cost tracking) → one local number per client";
 * A2P is THE hard gate — the pipeline blocks all sends until
 * a2p_campaign_status = 'approved'.
 */
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, commsProvisioning } from "@/server/db/schema";
import { appUrl } from "@/env";
import { log } from "@/lib/logger";

function masterAuth(): string {
  return "Basic " + Buffer.from(
    `${process.env.TWILIO_ACCOUNT_SID}:${process.env.TWILIO_AUTH_TOKEN}`,
  ).toString("base64");
}
function subAuth(subSid: string, subToken: string): string {
  return "Basic " + Buffer.from(`${subSid}:${subToken}`).toString("base64");
}

async function twilioPost<T>(url: string, form: URLSearchParams, auth: string): Promise<T> {
  const res = await fetch(url, {
    method: "POST",
    headers: { Authorization: auth, "Content-Type": "application/x-www-form-urlencoded" },
    body: form.toString(),
    signal: AbortSignal.timeout(30_000),
  });
  if (!res.ok) throw new Error(`Twilio ${res.status} ${url}: ${(await res.text()).slice(0, 300)}`);
  return res.json() as Promise<T>;
}

/** Step 1: subaccount. Returns and PERSISTS sid + auth token. Idempotent per client. */
export async function createSubaccount(clientId: string): Promise<{ sid: string }> {
  const [existing] = await db.select().from(commsProvisioning)
    .where(eq(commsProvisioning.clientId, clientId));
  if (existing?.twilioSubaccountSid) return { sid: existing.twilioSubaccountSid };

  const [client] = await db.select({ slug: clients.slug }).from(clients)
    .where(eq(clients.id, clientId));
  const created = await twilioPost<{ sid: string; auth_token: string }>(
    "https://api.twilio.com/2010-04-01/Accounts.json",
    new URLSearchParams({ FriendlyName: `seekly-${client.slug}` }),
    masterAuth(),
  );
  await db.insert(commsProvisioning)
    .values({
      clientId,
      twilioSubaccountSid: created.sid,
      twilioSubaccountAuthToken: created.auth_token,
      inboundEmailAddress: `${client.slug}@in.seekly.app`,
    })
    .onConflictDoUpdate({
      target: commsProvisioning.clientId,
      set: {
        twilioSubaccountSid: created.sid,
        twilioSubaccountAuthToken: created.auth_token,
        updatedAt: new Date(),
      },
    });
  return { sid: created.sid };
}

/** Step 2: search + buy a local number under the subaccount; wire webhooks. */
export async function buyLocalNumber(clientId: string, areaCode: string): Promise<{ phoneNumber: string }> {
  const [prov] = await db.select().from(commsProvisioning)
    .where(eq(commsProvisioning.clientId, clientId));
  if (!prov?.twilioSubaccountSid) throw new Error("createSubaccount first");
  if (prov.phoneNumber) return { phoneNumber: prov.phoneNumber };

  const search = await fetch(
    `https://api.twilio.com/2010-04-01/Accounts/${prov.twilioSubaccountSid}` +
      `/AvailablePhoneNumbers/US/Local.json?AreaCode=${areaCode}&SmsEnabled=true&PageSize=1`,
    { headers: { Authorization: masterAuth() }, signal: AbortSignal.timeout(30_000) },
  );
  const found = (await search.json()) as { available_phone_numbers: { phone_number: string }[] };
  const candidate = found.available_phone_numbers[0]?.phone_number;
  if (!candidate) throw new Error(`No SMS-capable numbers in area code ${areaCode}`);

  const bought = await twilioPost<{ phone_number: string }>(
    `https://api.twilio.com/2010-04-01/Accounts/${prov.twilioSubaccountSid}/IncomingPhoneNumbers.json`,
    new URLSearchParams({
      PhoneNumber: candidate,
      SmsUrl: `${appUrl}/api/messaging/twilio/inbound`,
      SmsMethod: "POST",
    }),
    masterAuth(),
  );
  await db.update(commsProvisioning)
    .set({ phoneNumber: bought.phone_number, updatedAt: new Date() })
    .where(eq(commsProvisioning.clientId, clientId));
  return { phoneNumber: bought.phone_number };
}

/** Step 3: Messaging Service under the subaccount (A2P campaigns attach to one). */
export async function createMessagingService(clientId: string): Promise<{ sid: string }> {
  const [prov] = await db.select().from(commsProvisioning)
    .where(eq(commsProvisioning.clientId, clientId));
  if (!prov?.twilioSubaccountSid || !prov.twilioSubaccountAuthToken || !prov.phoneNumber) {
    throw new Error("subaccount + number first");
  }
  if (prov.messagingServiceSid) return { sid: prov.messagingServiceSid };

  // messaging.twilio.com scopes to the AUTHENTICATING account → auth as subaccount.
  const auth = subAuth(prov.twilioSubaccountSid, prov.twilioSubaccountAuthToken);
  const svc = await twilioPost<{ sid: string }>(
    "https://messaging.twilio.com/v1/Services",
    new URLSearchParams({
      FriendlyName: `seekly-${clientId.slice(0, 8)}`,
      InboundRequestUrl: `${appUrl}/api/messaging/twilio/inbound`,
    }),
    auth,
  );
  // Attach the number to the service.
  const [num] = ((await (await fetch(
    `https://api.twilio.com/2010-04-01/Accounts/${prov.twilioSubaccountSid}/IncomingPhoneNumbers.json?PhoneNumber=${encodeURIComponent(prov.phoneNumber)}`,
    { headers: { Authorization: masterAuth() } },
  )).json()) as { incoming_phone_numbers: { sid: string }[] }).incoming_phone_numbers;
  await twilioPost(
    `https://messaging.twilio.com/v1/Services/${svc.sid}/PhoneNumbers`,
    new URLSearchParams({ PhoneNumberSid: num.sid }),
    auth,
  );
  await db.update(commsProvisioning)
    .set({ messagingServiceSid: svc.sid, updatedAt: new Date() })
    .where(eq(commsProvisioning.clientId, clientId));
  return { sid: svc.sid };
}
```

### A2P brand + campaign submission

The full ISV flow requires a TrustHub **secondary customer profile** for the client (business info, EIN, authorized rep, address — ~8 chained TrustHub calls) *before* the brand can be registered. Two paths, both supported:

**Path A — pilot / low volume (do this now):** ops creates the secondary customer profile + brand + campaign in the **Twilio Console** (Console → Messaging → Regulatory Compliance → A2P Messaging → "Register a brand" under the client's subaccount; use the EIN from intake Q1; campaign use case `MIXED`, sample messages = one review request + one conversational reply + one reactivation offer with opt-out language). Then record the SIDs:

```bash
npx tsx --env-file=.env scripts/record-a2p.ts <clientId> <brandSid BNxxxx> <campaignSid QExxxx>
# scripts/record-a2p.ts: UPDATE comms_provisioning SET a2p_brand_sid=$2,
# a2p_brand_status='submitted', a2p_campaign_sid=$3, a2p_campaign_status='submitted' WHERE client_id=$1
```

**Path B — automated (this helper; ship it, but Path A unblocks the pilot):**

```ts
/**
 * Submit the A2P brand for a client whose secondary customer profile bundle
 * already exists (bundleSid BU...). Brand vetting is asynchronous (days).
 */
export async function submitA2pBrand(clientId: string, customerProfileBundleSid: string,
  a2pTrustBundleSid: string): Promise<{ brandSid: string }> {
  const [prov] = await db.select().from(commsProvisioning)
    .where(eq(commsProvisioning.clientId, clientId));
  const auth = subAuth(prov!.twilioSubaccountSid!, prov!.twilioSubaccountAuthToken!);
  const brand = await twilioPost<{ sid: string; status: string }>(
    "https://messaging.twilio.com/v1/a2p/BrandRegistrations",
    new URLSearchParams({
      CustomerProfileBundleSid: customerProfileBundleSid, // BU... (TrustHub secondary profile)
      A2PProfileBundleSid: a2pTrustBundleSid,             // BU... (A2P trust product)
    }),
    auth,
  );
  await db.update(commsProvisioning)
    .set({ a2pBrandSid: brand.sid, a2pBrandStatus: brand.status.toLowerCase(), updatedAt: new Date() })
    .where(eq(commsProvisioning.clientId, clientId));
  return { brandSid: brand.sid };
}

/** Submit the campaign once the brand is APPROVED. Use case MIXED per doc 06. */
export async function submitA2pCampaign(clientId: string): Promise<{ campaignSid: string }> {
  const [prov] = await db.select().from(commsProvisioning)
    .where(eq(commsProvisioning.clientId, clientId));
  if (!prov?.messagingServiceSid || !prov.a2pBrandSid) throw new Error("brand + messaging service first");
  const auth = subAuth(prov.twilioSubaccountSid!, prov.twilioSubaccountAuthToken!);
  const campaign = await twilioPost<{ sid: string; campaign_status: string }>(
    `https://messaging.twilio.com/v1/Services/${prov.messagingServiceSid}/Compliance/Usa2p`,
    new URLSearchParams({
      BrandRegistrationSid: prov.a2pBrandSid,
      UsAppToPersonUsecase: "MIXED",
      Description:
        "Local business customer messaging: review requests after visits, replies to customer inquiries, and win-back offers to existing customers.",
      MessageFlow:
        "Customers opt in at point of sale / via the business's website forms; consent is recorded per customer. Every message includes 'Reply STOP to opt out.'",
      "MessageSamples": // Twilio accepts repeated MessageSamples params
        "{{business}}: thanks for visiting! Mind leaving a quick review? {{link}} Reply STOP to opt out.",
      HasEmbeddedLinks: "true",
      HasEmbeddedPhone: "false",
      SubscriberOptIn: "true",
      OptInKeywords: "START",
      OptOutKeywords: "STOP,STOPALL,UNSUBSCRIBE,CANCEL,END,QUIT",
      HelpKeywords: "HELP,INFO",
      OptInMessage: "You are subscribed to messages from {{business}}. Reply STOP to unsubscribe.",
      OptOutMessage: "You have been unsubscribed and will receive no further messages. Reply START to resubscribe.",
      HelpMessage: "Reply STOP to unsubscribe. Msg&Data rates may apply.",
    }),
    auth,
  );
  await db.update(commsProvisioning)
    .set({
      a2pCampaignSid: campaign.sid,
      a2pCampaignStatus: "submitted",
      a2pCampaignType: "MIXED",
      updatedAt: new Date(),
    })
    .where(eq(commsProvisioning.clientId, clientId));
  return { campaignSid: campaign.sid };
}

/** Hourly status poll — wired into the existing checkSchedules cron in src/worker.ts. */
export async function pollA2pStatuses(): Promise<number> {
  const rows = await db.select().from(commsProvisioning)
    .where(eq(commsProvisioning.a2pCampaignStatus, "submitted"));
  let updated = 0;
  for (const prov of rows) {
    if (!prov.messagingServiceSid || !prov.a2pCampaignSid) continue;
    const auth = subAuth(prov.twilioSubaccountSid!, prov.twilioSubaccountAuthToken!);
    const res = await fetch(
      `https://messaging.twilio.com/v1/Services/${prov.messagingServiceSid}/Compliance/Usa2p/${prov.a2pCampaignSid}`,
      { headers: { Authorization: auth }, signal: AbortSignal.timeout(30_000) },
    );
    if (!res.ok) continue;
    const body = (await res.json()) as { campaign_status: string }; // IN_PROGRESS | VERIFIED | FAILED
    const mapped =
      body.campaign_status === "VERIFIED" ? "approved" :
      body.campaign_status === "FAILED" ? "rejected" : "submitted";
    if (mapped !== prov.a2pCampaignStatus) {
      await db.update(commsProvisioning)
        .set({ a2pCampaignStatus: mapped, updatedAt: new Date() })
        .where(eq(commsProvisioning.clientId, prov.clientId));
      log(`[a2p] client=${prov.clientId} campaign → ${mapped}`);
      updated++;
      if (mapped === "approved") {
        await sendOpsAlert("A2P campaign approved", `Client ${prov.clientId} can now send. Flip module switches.`);
      }
      if (mapped === "rejected") {
        await sendOpsAlert("A2P campaign REJECTED", `Client ${prov.clientId}: campaign ${prov.a2pCampaignSid} rejected — fix and resubmit.`);
      }
    }
  }
  return updated;
}
```

Wire `pollA2pStatuses()` into the existing hourly `checkSchedules` handler in `src/worker.ts` (next to `checkReviewIngest()`); also create `scripts/provision-comms.ts` (tsx, `--env-file=.env`) that runs steps 1–3 for a client id + area code and prints the SIDs — this is the "WF-0 lite" manual provisioning path from doc 07 week 1.

**Acceptance criteria**

1. `scripts/provision-comms.ts <clientId> <areaCode>` on a Twilio test account creates a subaccount, buys a number with `SmsUrl` pointing at `/api/messaging/twilio/inbound`, creates a messaging service with the number attached, and the `comms_provisioning` row holds all SIDs. Running it twice changes nothing (idempotent per step).
2. `submitA2pCampaign` posts `UsAppToPersonUsecase=MIXED` and all opt-in/out/help keyword fields.
3. `pollA2pStatuses` flips `submitted → approved` when Twilio returns `VERIFIED`, and sends an ops alert; the very next `sendMessage` for that client passes the A2P gate (no deploy, no cache).
4. `scripts/record-a2p.ts` (Path A) exists and updates the row.
5. No function ever writes `a2pCampaignStatus = 'approved'` except `pollA2pStatuses` and the explicit ops script — grep-provable.

---

## SP-8 — Migrate review-request sends onto the pipeline

**What changes and what doesn't:** the review engine keeps everything about *who/when/whether-to-ask* (scheduling, per-contact SMS consent, feedback tokens, follow-ups, email channel). It stops doing its own SMS *delivery compliance* — opt-out, quiet hours, ledger, Twilio — because the pipeline owns those now.

### `src/server/reviews/messaging.ts`

- **Delete** `sendReviewSms` and `smsConfigured` (grep for usages first; `engine.ts` is the only caller).
- Keep `sendReviewEmail` / `emailConfigured` untouched (email is out of the SMS pipeline's scope).

### `src/server/reviews/engine.ts` — `sendRequest()`

Replace the SMS branch. The email branch and `decideSend` flow stay exactly as-is for `channel === "email"`. For `channel === "sms"`:

```ts
import { sendMessage } from "@/server/messaging/send-pipeline";

// ...inside sendRequest(), after loading `row`:

if (channel === "sms") {
  // Engine-level gates the pipeline can't know about: module switch, per-contact
  // consent + client attestation (TCPA §consent is engine data, not pipeline data).
  if (!config.enabled) {
    await db.update(reviewRequests)
      .set({ status: "suppressed", error: "engine disabled" })
      .where(eq(reviewRequests.id, requestId));
    return;
  }
  if (!row.contact.consentSms || !config.smsConsentAttested) {
    await db.update(reviewRequests)
      .set({ status: "suppressed", error: "no sms consent" })
      .where(eq(reviewRequests.id, requestId));
    return;
  }

  const feedbackUrl = `${appUrl}/reviews/${row.req.feedbackToken}`;
  const result = await sendMessage({
    clientId: row.req.clientId,
    to: dest!,
    body: `${row.businessName}: thanks for visiting! Mind leaving a quick review? ${feedbackUrl} Reply STOP to opt out.`,
    kind: "marketing", // review requests are Seekly-initiated → strictest kind
    engine: "review_velocity",
    idempotencyKey: `review-request:${requestId}`, // one ledger row per request row
  });

  // The pipeline is now the delivery system of record; the request row records
  // the handoff outcome. "queued" (quiet hours) counts as sent-from-the-
  // engine's-perspective — the pipeline guarantees delivery or a blocked audit row.
  if (result.outcome === "blocked") {
    await db.update(reviewRequests)
      .set({ status: "suppressed", error: `pipeline: ${result.blockedReason}` })
      .where(eq(reviewRequests.id, requestId));
    return;
  }
  if (result.outcome === "failed") {
    await db.update(reviewRequests)
      .set({ status: "failed", error: "pipeline send failed" })
      .where(eq(reviewRequests.id, requestId));
    return;
  }
  await db.update(reviewRequests)
    .set({ status: "sent", sentAt: now })
    .where(eq(reviewRequests.id, requestId));
  // follow-up scheduling block: unchanged from current code
  ...
  return;
}
// email branch: existing decideSend() flow, unchanged
```

Consequences to verify in review:

- The engine's own quiet-hours `defer` no longer applies to SMS (the pipeline queues instead) — `DEFER_MS` logic remains only for email.
- The engine no longer reads `reviewOptOuts` for SMS (`isSuppressed` still used for email); the pipeline checks **both** opt-out tables, so pre-migration STOPs still block.
- Re-ask cooldown (90d, doc 03) is engine logic and additionally backstopped by the pipeline's 30d marketing frequency cap.
- The `From` number is now the client's provisioned number (or the backfilled legacy shared number from SP-1), not `TWILIO_FROM_NUMBER` read at send time.

**Acceptance criteria**

1. A due SMS review request produces exactly one `messages` ledger row with `engine='review_velocity'`, `kind='marketing'`, `idempotencyKey='review-request:<id>'`.
2. Re-running `sendDueRequests()` after a crash mid-batch never double-texts: the request row is `sent`, and even if it weren't, the pipeline idempotency key dedupes.
3. A contact on the legacy `reviewOptOuts` list gets `suppressed` with `pipeline: opt_out`.
4. A request due at 22:00 client-local: request row = `sent`, ledger row = `queued`, message actually transmits after 08:00 (assert the pg-boss job's `startAfter`).
5. Without consent (`consentSms=false` or attestation off) nothing reaches the pipeline (no ledger row).
6. Email requests behave byte-identically to before (existing tests still green).

---

## SP-9 — Failure & edge cases

Most implementation already appears in SP-3/SP-4/SP-6; this ticket completes and verifies the matrix:

| Case | Behavior | Where |
|---|---|---|
| Twilio HTTP 429 / error 20429 | Row reverts `sending→queued`, `TwilioRateLimitedError` thrown → pg-boss retries ×3 with exponential backoff; after final failure job lands in `message-send-dlq` and the row stays `queued` for manual replay | SP-3 `performSend`, SP-4 |
| Twilio 5xx | Same retry path as 429 (treat ≥500 as retryable: extend the `throw` branch in `twilioSendSms` — `if (res.status >= 500) throw new TwilioRateLimitedError(...)`) | SP-4 |
| Twilio 4xx (21211 invalid number, 21408 no international permission, …) | Row → `failed` + `errorCode`; NOT retried (permanent) | SP-3 |
| Status callback `undelivered` (30003 unreachable, 30005 unknown, 30006 landline) | Ledger → `undelivered` + `errorCode`, error log line. Follow-up: nightly rollup counts per client feed `metrics_daily` (spec 01) | SP-4 |
| Error 21610 (recipient blacklisted STOP at Twilio) | Auto-insert `opt_outs` row (`source='carrier_error'`) | SP-4 |
| STOP rate > 3% of a campaign batch (min 25 sends) | Campaign `status='paused'` + ops alert email | SP-6 `maybePauseCampaignForStop` |
| dlq accumulation | Add dlq depth check to the hourly `checkSchedules` handler: `SELECT count(*) FROM pgboss.job WHERE name='message-send-dlq' AND state='created'` via `boss` API → `sendOpsAlert` when > 0 (doc 07: failures alert ops within 5 minutes → the hourly cron plus immediate error logs satisfies v1; tighten later) | this ticket |
| Worker crash mid-send (row stuck `sending`) | Sweep in `checkSchedules`: rows `sending` older than 10 min → back to `queued` + re-enqueue with `singletonKey` (safe: Twilio was either called — sid recorded before any crash point after success — or not) | this ticket |

The `sending`-sweep code (add to a new `src/server/messaging/sweep.ts`, called from the `checkSchedules` handler):

```ts
export async function sweepStuckSends(): Promise<number> {
  const cutoff = new Date(Date.now() - 10 * 60_000);
  const stuck = await db.select({ id: messages.id }).from(messages)
    .where(and(eq(messages.status, "sending"), isNull(messages.twilioSid),
      lte(messages.createdAt, cutoff)));
  const boss = await getBoss();
  for (const row of stuck) {
    await db.update(messages).set({ status: "queued" }).where(eq(messages.id, row.id));
    await boss.send(QUEUES.messageSend, { messageId: row.id }, { singletonKey: row.id });
  }
  return stuck.length;
}
```

**Acceptance criteria**

1. Mocked 429 → row is `queued` again and the pg-boss job fails (retry scheduled); after 3 failures the job is in `message-send-dlq` and the row is still `queued` (not lost, not sent).
2. Mocked 21211 → row `failed`, `errorCode='21211'`, no retry job.
3. Simulated campaign with 100 `sent` rows and 4 STOP webhooks → `campaigns.status='paused'`, exactly one ops alert.
4. `sweepStuckSends` re-queues a 15-minute-old `sending` row and ignores a fresh one.

---

## SP-10 — Compliance test pack (doc 07 "rock solid", pipeline-scoped)

`src/server/messaging/pipeline.test.ts` (vitest; mock `fetch` globally, use the test-DB pattern the repo uses — or `vi.mock("@/server/db")` with an in-memory row store if no test DB exists; the PURE tests in SP-2 need no DB either way). These are the named tests that must exist and pass — each maps to a doc 07 bullet:

| # | Test name | Doc 07 bullet | Assertion |
|---|---|---|---|
| 1 | `replay of the same trigger produces zero duplicate sends` | Idempotent | 2× `sendMessage` same key → 1 ledger row, ≤1 `fetch` to Twilio |
| 2 | `STOP blocks every kind across engines instantly` | Compliant | after `handleStop`, `sendMessage` for `marketing`+`conversational`+`transactional`, engines `review_velocity`+`reactivation` → all `blocked/opt_out`, zero fetches |
| 3 | `zero sends outside quiet hours — queued, not dropped` | Compliant | marketing at 23:00 → row exists, status `queued`, boss job `startAfter` within [08:00, 08:06] local; no fetch |
| 4 | `zero sends before A2P approval` | Compliant | `a2pCampaignStatus='submitted'` → all kinds `blocked/a2p_pending`; flip to `approved` → next send passes |
| 5 | `frequency cap across engines` | Compliant | marketing sent 10 days ago by `review_velocity` → new marketing from `reactivation` `blocked/freq_cap`; conversational passes |
| 6 | `legacy reviewOptOuts still suppress` | Compliant | hash-only opt-out → `blocked/opt_out` |
| 7 | `STOP-rate auto-pause at >3%` | Safe-off | 100 sent + 4 STOPs → campaign paused; 100 sent + 2 STOPs → not paused; 10 sent + 1 STOP → not paused (min-sample) |
| 8 | `queued messages survive a switch-off` | Safe-off | `performSend` on a queued row after opt-out arrives → blocked, ledger auditable |
| 9 | `every outcome is observable` | Observable | each of sent/queued/blocked/failed leaves a ledger row with a non-null status + reason/errorCode where applicable |
| 10 | `Twilio failure falls back safely` | Fallback-first | 429 → retry path; 4xx → `failed` row; neither ever double-sends |

**Acceptance criteria**: all 10 tests exist with these behaviors, run in `npm test`, and are green in CI. This table is the release checklist for flipping the pilot's `review_automation` switch to live sends.
