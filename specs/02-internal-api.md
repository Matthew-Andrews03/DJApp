# Spec 02 — Internal API for n8n

## Purpose

The control-plane HTTP surface that n8n (and the ingestion edge: email-parse, Twilio inbound,
CSV importers) consumes — doc 02 layer 1 as revised by doc 10: **Next.js route handlers inside
`seekly-client-insights`**, authenticated with a shared service token. Four endpoints:

| Endpoint | Role |
|---|---|
| `POST /api/internal/events` | Canonical event ingestion: validate → dedupe on `event_id` → persist → forward (HMAC-signed) to the right n8n webhook |
| `GET /api/internal/clients/:id/context?module=` | One call per workflow run: profile + module config + all switches + short-lived provider access tokens |
| `POST /api/internal/activity` | Activity-log writes (single or batch) — every action every engine takes |
| `POST /api/internal/send` | The shared send-pipeline as a service (contract here; internals in spec 03) |

Depends on spec 01 tables (`events`, `customers`, `clientProfile`, `workflowConfig`,
`connections`, `commsProvisioning`, `activityLog`, `messages`). All file paths are inside the
`seekly-client-insights` repo.

## Ticket breakdown

| ID | Title | Depends on |
|-------|-------|------------|
| API-1 | Env additions + service-token auth middleware + rate-limit helper | DB-1 |
| API-2 | Canonical event zod schemas (`src/server/internal/events-schema.ts`) | — |
| API-3 | `POST /api/internal/events` route + n8n forwarder (HMAC) | API-1, API-2, DB-4 |
| API-4 | Token vault crypto (`connections/crypto.ts`) + `getFreshAccessToken` refresh | API-1, DB-3 |
| API-5 | `GET /api/internal/clients/[id]/context` route | API-1, API-4, DB-2, DB-3 |
| API-6 | `POST /api/internal/activity` route (batch) | API-1, DB-9 |
| API-7 | `POST /api/internal/send` route + send-pipeline interface stub | API-1, DB-6 |
| API-8 | Vitest unit tests + curl smoke pack | API-1..API-7 |

## Global contract rules

1. **Wire format is snake_case** on every `/api/internal/*` request and response
   (`event_id`, `client_id`, `blocked_reason`, …). Rationale: docs 02/03/04 define the
   canonical envelope and config vocabulary in snake_case, and n8n workflow JSON will be
   written against those docs verbatim. TS code stays camelCase; zod schemas do the mapping.
2. **Auth:** every route requires `Authorization: Bearer $N8N_SERVICE_TOKEN`. No cookie/session
   fallback — these routes are never browser-facing. 401 body is always
   `{"error":"unauthorized"}` (no detail; don't leak whether the token or the route is wrong).
3. **No caching anywhere:** every response carries `Cache-Control: no-store`. n8n must fetch
   context per run (doc 02: "responses are never cached in n8n") — toggles and tokens go
   stale in seconds.
4. **`client_id` always in the body/path and always validated** against a live
   (`deletedAt IS NULL`) client. Unknown → 404 `unknown_client`. The token is platform-wide;
   per-client scoping is by explicit id, mirroring how n8n runs are keyed.
5. **Timestamps** are ISO-8601 with offset (`2026-07-13T14:03:00-04:00` or `...Z`), validated
   with `z.string().datetime({ offset: true })` (the codebase's existing idiom).
6. **Every route is idempotent** where a retry is possible: events on `event_id`, activity on
   `dedupe_key`, send on `idempotency_key`. Retrying any request is always safe.

### Error-code table (all endpoints)

| HTTP | `error` code | When | Caller action |
|---|---|---|---|
| 200 | — | Success (including deduped replays) | proceed |
| 201 | — | `/events` only: new event persisted + forwarded | proceed |
| 202 | — | `/events` only: persisted but NOT forwarded (`forwarded: false`) | do **not** retry (it's stored); ops replays |
| 400 | `invalid_json` | Body isn't parseable JSON | fix caller |
| 400 | `invalid_client_id` | Path param isn't a uuid | fix caller |
| 401 | `unauthorized` | Missing/bad service token (or `N8N_SERVICE_TOKEN` unset) | fix config |
| 404 | `unknown_client` | `client_id` not found or soft-deleted | stop; alert ops |
| 404 | `unknown_customer` | `/send` `customer_id` not found for that client | stop; alert ops |
| 409 | `event_id_conflict` | Same `event_id` previously stored for a **different** client/type | bug in adapter — alert ops |
| 413 | `payload_too_large` | Body over the route's size limit | fix caller |
| 422 | `validation_error` | zod parse failed (`issues` array included) | fix caller |
| 429 | `rate_limited` | Route bucket exceeded (`Retry-After` header set) | back off |
| 500 | `internal_error` | Unhandled exception (logged server-side) | retry with backoff |

### Rate & size limits (per route, in-memory per app instance)

| Route | Max req/min | Max body |
|---|---|---|
| `POST /api/internal/events` | 120 | 64 KB |
| `GET /api/internal/clients/:id/context` | 120 | — |
| `POST /api/internal/activity` | 300 | 256 KB |
| `POST /api/internal/send` | 60 | 64 KB |

Same fixed-window bucket pattern as `/api/ingest/conversion` — good enough for a
single-instance deploy; revisit if the app ever scales horizontally.

---

## API-1 — Env additions, auth middleware, rate-limit helper

### 1a. `src/env.ts` — add to the zod `schema` object (after the vault block)

```ts
  // --- Optional: internal API for n8n (spec 02). Absent = every /api/internal
  // route answers 401/skips forwarding — the app boots fine without n8n. ---
  /** Shared bearer token n8n sends on every internal-API call. Generate: openssl rand -hex 32 */
  N8N_SERVICE_TOKEN: z.string().min(32).optional(),
  /** Base URL of the n8n instance, e.g. https://n8n.seekly.app (no trailing slash needed). */
  N8N_WEBHOOK_BASE_URL: z.string().url().optional(),
  /** HMAC key for signing events forwarded to n8n webhooks. Generate: openssl rand -hex 32 */
  N8N_WEBHOOK_SECRET: z.string().min(32).optional(),
  /** 32-byte base64 key for AES-256-GCM token sealing. Generate: openssl rand -base64 32 */
  CONNECTIONS_ENCRYPTION_KEY: z.string().optional(),
  /** Seekly's Google OAuth app (one app, all clients — doc 06). Needed for token refresh. */
  GOOGLE_OAUTH_CLIENT_ID: z.string().optional(),
  GOOGLE_OAUTH_CLIENT_SECRET: z.string().optional(),
```

Also add all six names (values blank) to `.env.example` with the generate commands as comments.

### 1b. New file `src/server/internal/auth.ts` (complete)

Mirror of `src/server/spine/auth.ts` — same timing-safe compare, different token, so the two
integration surfaces can be rotated/revoked independently:

```ts
import { timingSafeEqual } from "node:crypto";
import { env } from "@/env";

/** Constant-time string compare — avoids leaking token length/prefix via response timing. */
function tokensMatch(a: string, b: string): boolean {
  const bufA = Buffer.from(a);
  const bufB = Buffer.from(b);
  return bufA.length === bufB.length && timingSafeEqual(bufA, bufB);
}

/**
 * Guards an /api/internal route with the n8n service token
 * (`Authorization: Bearer $N8N_SERVICE_TOKEN`). Returns a 401 Response, or
 * null when authorized. Fails CLOSED when the env var is unset — an
 * undeployed token can never mean an open endpoint.
 */
export function requireInternalToken(req: Request): Response | null {
  const expected = env.N8N_SERVICE_TOKEN;
  const header = req.headers.get("authorization");
  const token = header?.startsWith("Bearer ") ? header.slice(7) : null;
  if (!expected || !token || !tokensMatch(token, expected)) {
    return Response.json(
      { error: "unauthorized" },
      { status: 401, headers: { "cache-control": "no-store" } },
    );
  }
  return null;
}
```

### 1c. New file `src/server/internal/rate-limit.ts` (complete)

```ts
/**
 * Fixed-window in-memory rate limiter for the internal API — same pattern as
 * /api/ingest/conversion. Keyed per ROUTE (the caller population is one n8n
 * instance plus a handful of edge adapters, all sharing one token, so
 * per-token keying adds nothing). Single-instance semantics; acceptable for
 * the current one-box deploy.
 */
const WINDOW_MS = 60_000;
const buckets = new Map<string, { count: number; resetAt: number }>();

export function allowInternalRequest(routeKey: string, maxPerMinute: number): boolean {
  const now = Date.now();
  const bucket = buckets.get(routeKey);
  if (!bucket || bucket.resetAt < now) {
    buckets.set(routeKey, { count: 1, resetAt: now + WINDOW_MS });
    return true;
  }
  bucket.count += 1;
  return bucket.count <= maxPerMinute;
}

/** 429 helper so every route returns the same shape + Retry-After. */
export function rateLimited(): Response {
  return Response.json(
    { error: "rate_limited" },
    { status: 429, headers: { "retry-after": "60", "cache-control": "no-store" } },
  );
}
```

### Acceptance criteria

1. With `N8N_SERVICE_TOKEN` unset, every `/api/internal/*` route returns 401.
2. Wrong token → 401; right token → route logic runs. Token match is `timingSafeEqual`.
3. `npx tsc --noEmit` passes; `.env.example` documents all six new vars.
4. Boot without any of the new vars succeeds (all optional).

---

## API-2 — Canonical event zod schemas

New file `src/server/internal/events-schema.ts` (complete). One envelope, nine per-type
payload schemas (docs 02/03). Payloads use `.loose()` objects so **unknown fields are
preserved verbatim** into `events.payload` (doc 02: "unknown fields are preserved in
`payload.raw` for later adapter improvements" — we go one better and store the whole payload
as received).

```ts
import { z } from "zod";

/**
 * Canonical event envelope + per-type payload schemas (docs/02 layer 3).
 * Wire format is snake_case — this file IS the contract adapters and n8n are
 * written against; keep it in lockstep with eventTypeEnum in db/schema.ts.
 */

/** Known adapter sources (doc 02 envelope). Open set — z.string() with a documented registry,
 *  not z.enum, so a new adapter never needs an app deploy to start emitting. */
export const KNOWN_EVENT_SOURCES = [
  "square", "email-parse", "webhook", "csv", "manual", "gbp-poll", "meta", "twilio",
  "portal", "hubspot",
] as const;

const phone = z.string().min(7).max(32);       // E.164-ish; strictly normalized at consumption
const email = z.string().email().max(320);
const isoTs = z.string().datetime({ offset: true });

/**
 * The normalized customer block (doc 02: sale.completed always carries
 * customer {name, phone, email?}). Name required; at least one reachable
 * handle (phone/email) required — an unreachable customer record is useless
 * to every consumer. Loose: adapters may attach extra keys.
 */
const customerBlock = z
  .looseObject({
    name: z.string().min(1).max(200),
    phone: phone.optional(),
    email: email.optional(),
  })
  .refine((c) => c.phone || c.email, { message: "customer needs phone or email" });

// --- Per-type payloads (all .looseObject: unknown keys stored verbatim) ---

export const saleCompletedPayload = z.looseObject({
  customer: customerBlock,
  /** Sale total in the client's currency; optional (some POS emails omit it). */
  amount: z.number().nonnegative().optional(),
  currency: z.string().length(3).optional(),
  items: z.array(z.looseObject({ name: z.string(), quantity: z.number().optional() })).optional(),
  /** POS-side id of the sale/booking, when the adapter has one. */
  external_id: z.string().max(200).optional(),
  raw: z.unknown().optional(),
});

export const leadCreatedPayload = z.looseObject({
  customer: customerBlock,
  /** Where the lead came from within the source: "contact-form", "party-inquiry", ad id… */
  form: z.string().max(200).optional(),
  /** The lead's own words — drives the specific first-touch SMS (doc 03 WF-2). */
  message: z.string().max(4000).optional(),
  fields: z.record(z.string(), z.unknown()).optional(),
  raw: z.unknown().optional(),
});

export const messageReceivedPayload = z.looseObject({
  from: phone,
  to: phone,
  body: z.string().max(4000),
  twilio_sid: z.string().max(64).optional(),
  media_urls: z.array(z.string().url()).optional(),
  raw: z.unknown().optional(),
});

export const callMissedPayload = z.looseObject({
  from: phone,
  to: phone,
  /** "no-answer" | "busy" | "after-hours" (Twilio DialCallStatus-derived). */
  reason: z.string().max(50).optional(),
  raw: z.unknown().optional(),
});

export const reviewReceivedPayload = z.looseObject({
  provider: z.literal("google"),
  /** Provider's stable review id — reviews-table idempotency key (DB spec). */
  external_id: z.string().min(1).max(200),
  rating: z.number().int().min(1).max(5),
  author: z.string().max(200).optional(),
  text: z.string().max(10_000).optional(),
  url: z.string().url().optional(),
  reviewed_at: isoTs,
  raw: z.unknown().optional(),
});

export const customerImportedPayload = z.looseObject({
  customer: customerBlock,
  last_visit_at: isoTs.optional(),
  visit_count: z.number().int().nonnegative().optional(),
  lifetime_value: z.number().nonnegative().optional(),
  consent_basis: z.enum(["existing_customer", "lead_inbound", "imported_list"]),
  external_ids: z.record(z.string(), z.string()).optional(),
  raw: z.unknown().optional(),
});

export const contentPublishedPayload = z.looseObject({
  url: z.string().url(),
  title: z.string().max(300).optional(),
  /** Set when WF-4 published it; absent for RSS-discovered client-authored posts. */
  content_item_id: z.string().uuid().optional(),
  published_at: isoTs.optional(),
  raw: z.unknown().optional(),
});

export const profileUpdatedPayload = z.looseObject({
  /** client_profile field names that changed, e.g. ["hours_json","review_link"]. */
  changed_fields: z.array(z.string()).min(1),
  raw: z.unknown().optional(),
});

export const clientProvisionedPayload = z.looseObject({
  hubspot_deal_id: z.string().max(100).optional(),
  hubspot_company_id: z.string().max(100).optional(),
  company_name: z.string().min(1).max(300),
  tier: z.enum(["pilot", "core", "growth", "market_leader"]),
  contact: z.looseObject({
    name: z.string().max(200).optional(),
    email,
    phone: phone.optional(),
  }),
  raw: z.unknown().optional(),
});

// --- Envelope (doc 02): discriminated on `type` so each event type gets its payload schema ---

const base = {
  /** THE idempotency key — becomes events.id. Adapters must generate it deterministically per real-world occurrence. */
  event_id: z.string().uuid(),
  client_id: z.string().uuid(),
  occurred_at: isoTs,
  source: z.string().min(1).max(50),
};

export const eventEnvelopeSchema = z.discriminatedUnion("type", [
  z.object({ ...base, type: z.literal("sale.completed"), payload: saleCompletedPayload }),
  z.object({ ...base, type: z.literal("lead.created"), payload: leadCreatedPayload }),
  z.object({ ...base, type: z.literal("message.received"), payload: messageReceivedPayload }),
  z.object({ ...base, type: z.literal("call.missed"), payload: callMissedPayload }),
  z.object({ ...base, type: z.literal("review.received"), payload: reviewReceivedPayload }),
  z.object({ ...base, type: z.literal("customer.imported"), payload: customerImportedPayload }),
  z.object({ ...base, type: z.literal("content.published"), payload: contentPublishedPayload }),
  z.object({ ...base, type: z.literal("profile.updated"), payload: profileUpdatedPayload }),
  z.object({ ...base, type: z.literal("client.provisioned"), payload: clientProvisionedPayload }),
]);

export type EventEnvelope = z.infer<typeof eventEnvelopeSchema>;
export type CanonicalEventType = EventEnvelope["type"];
```

### Acceptance criteria

1. A valid `sale.completed` envelope parses; the same envelope with `type` swapped to
   `lead.created` still parses (customer block is shared) but `review.received` without
   `external_id` fails with a 1-issue error.
2. Unknown payload keys survive `parse()` (assert `parsed.payload.weird === 1`).
3. A customer block with neither phone nor email fails.
4. `event_id`/`client_id` reject non-uuids; `occurred_at` rejects `"2026-07-13"` (no time).
5. Unit tests in `src/server/internal/events-schema.test.ts` cover 1–4.

---

## API-3 — `POST /api/internal/events` + n8n forwarder

### 3a. New file `src/server/internal/forward.ts` (complete)

```ts
import { createHmac } from "node:crypto";
import { env } from "@/env";
import type { CanonicalEventType, EventEnvelope } from "./events-schema";

/**
 * Forwards a persisted canonical event to its n8n master-workflow webhook,
 * signed so n8n can verify the control plane sent it (doc 02: "webhook
 * endpoints receive only from the control plane (signed)").
 *
 * Signature scheme (document this in the n8n workflow README too):
 *   X-Seekly-Timestamp: unix seconds
 *   X-Seekly-Signature: sha256=hex( HMAC-SHA256(N8N_WEBHOOK_SECRET, "{ts}.{rawBody}") )
 * n8n verifies with a Code node: recompute, reject if mismatch or |now-ts| > 300s.
 */
const EVENT_WEBHOOK_PATHS: Record<CanonicalEventType, string> = {
  "sale.completed": "seekly-sale-completed",
  "lead.created": "seekly-lead-created",
  "message.received": "seekly-message-received",
  "call.missed": "seekly-call-missed",
  "review.received": "seekly-review-received",
  "customer.imported": "seekly-customer-imported",
  "content.published": "seekly-content-published",
  "profile.updated": "seekly-profile-updated",
  "client.provisioned": "seekly-client-provisioned",
};

export type ForwardResult = { ok: true } | { ok: false; error: string };

export async function forwardEventToN8n(evt: EventEnvelope): Promise<ForwardResult> {
  if (!env.N8N_WEBHOOK_BASE_URL || !env.N8N_WEBHOOK_SECRET) {
    return { ok: false, error: "n8n forwarding not configured" };
  }
  const url = `${env.N8N_WEBHOOK_BASE_URL.replace(/\/+$/, "")}/webhook/${EVENT_WEBHOOK_PATHS[evt.type]}`;
  const body = JSON.stringify(evt);
  const ts = Math.floor(Date.now() / 1000).toString();
  const signature = createHmac("sha256", env.N8N_WEBHOOK_SECRET)
    .update(`${ts}.${body}`)
    .digest("hex");
  try {
    const res = await fetch(url, {
      method: "POST",
      headers: {
        "content-type": "application/json",
        "x-seekly-timestamp": ts,
        "x-seekly-signature": `sha256=${signature}`,
      },
      body,
      signal: AbortSignal.timeout(10_000),
    });
    if (!res.ok) return { ok: false, error: `n8n responded ${res.status}` };
    return { ok: true };
  } catch (err) {
    return { ok: false, error: err instanceof Error ? err.message : "fetch failed" };
  }
}
```

### 3b. New file `src/app/api/internal/events/route.ts` (complete — the reference handler)

```ts
import { and, eq, isNull } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, events } from "@/server/db/schema";
import { requireInternalToken } from "@/server/internal/auth";
import { allowInternalRequest, rateLimited } from "@/server/internal/rate-limit";
import { eventEnvelopeSchema } from "@/server/internal/events-schema";
import { forwardEventToN8n } from "@/server/internal/forward";
import { logError } from "@/lib/logger";

/**
 * Canonical event ingestion (doc 02 layer 3): validate → dedupe on event_id →
 * persist → forward (signed) to the event type's n8n master workflow.
 *
 * Idempotency contract: events.id IS the adapter's event_id. Redelivering an
 * already-PROCESSED event is a pure no-op (200, duplicate:true). Redelivering
 * a PENDING/FAILED one retries the forward — so "replay a stuck event" is
 * just "POST it again".
 */
const MAX_BODY_BYTES = 64 * 1024;
const NO_STORE = { "cache-control": "no-store" } as const;

export async function POST(request: Request) {
  const unauth = requireInternalToken(request);
  if (unauth) return unauth;
  if (!allowInternalRequest("internal:events", 120)) return rateLimited();

  const rawBody = await request.text();
  if (Buffer.byteLength(rawBody) > MAX_BODY_BYTES) {
    return Response.json({ error: "payload_too_large" }, { status: 413, headers: NO_STORE });
  }
  let body: unknown;
  try {
    body = JSON.parse(rawBody);
  } catch {
    return Response.json({ error: "invalid_json" }, { status: 400, headers: NO_STORE });
  }
  const parsed = eventEnvelopeSchema.safeParse(body);
  if (!parsed.success) {
    return Response.json(
      { error: "validation_error", issues: parsed.error.issues },
      { status: 422, headers: NO_STORE },
    );
  }
  const evt = parsed.data;

  const [client] = await db
    .select({ id: clients.id })
    .from(clients)
    .where(and(eq(clients.id, evt.client_id), isNull(clients.deletedAt)));
  if (!client) {
    return Response.json({ error: "unknown_client" }, { status: 404, headers: NO_STORE });
  }

  // --- Idempotent persist: insert-on-conflict on the event_id PK ---
  const inserted = await db
    .insert(events)
    .values({
      id: evt.event_id,
      clientId: evt.client_id,
      type: evt.type,
      source: evt.source,
      occurredAt: new Date(evt.occurred_at),
      payload: evt.payload,
    })
    .onConflictDoNothing({ target: events.id })
    .returning({ id: events.id });

  const duplicate = inserted.length === 0;
  if (duplicate) {
    const [existing] = await db
      .select({ clientId: events.clientId, type: events.type, status: events.status })
      .from(events)
      .where(eq(events.id, evt.event_id));
    // Same id, different client/type = adapter bug (colliding ids), never a replay.
    if (!existing || existing.clientId !== evt.client_id || existing.type !== evt.type) {
      return Response.json({ error: "event_id_conflict" }, { status: 409, headers: NO_STORE });
    }
    if (existing.status === "processed" || existing.status === "ignored") {
      return Response.json(
        { accepted: true, duplicate: true, forwarded: false },
        { headers: NO_STORE },
      );
    }
    // pending/failed duplicate: fall through — this redelivery retries the forward.
  }

  const forward = await forwardEventToN8n(evt);
  if (forward.ok) {
    await db
      .update(events)
      .set({ status: "processed", processedAt: new Date(), error: null })
      .where(eq(events.id, evt.event_id));
    return Response.json(
      { accepted: true, duplicate, forwarded: true },
      { status: duplicate ? 200 : 201, headers: NO_STORE },
    );
  }

  await db
    .update(events)
    .set({ status: "failed", error: forward.error })
    .where(eq(events.id, evt.event_id));
  logError("internal/events forward failed", evt.event_id, evt.type, forward.error);
  // Persisted but not handed off: 202 tells the adapter NOT to retry (it's stored);
  // ops replays by re-POSTing the same envelope (or a future sweep does).
  return Response.json(
    { accepted: true, duplicate, forwarded: false, forward_error: forward.error },
    { status: 202, headers: NO_STORE },
  );
}
```

### curl example

```bash
curl -sS -X POST "$APP_URL/api/internal/events" \
  -H "Authorization: Bearer $N8N_SERVICE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "event_id": "5f0c9d5e-8a3f-4d1c-9b6e-2f7a1c4d8e90",
    "client_id": "11111111-2222-4333-a444-555555555555",
    "type": "sale.completed",
    "occurred_at": "2026-07-13T14:03:00-04:00",
    "source": "email-parse",
    "payload": {
      "customer": { "name": "Jordan Lee", "phone": "+16135551234" },
      "amount": 120,
      "items": [{ "name": "2hr bay rental" }],
      "raw": { "pos_email_subject": "Booking confirmed #4412" }
    }
  }'
# → 201 {"accepted":true,"duplicate":false,"forwarded":true}
# repeat the same command → 200 {"accepted":true,"duplicate":true,"forwarded":false}
```

### Acceptance criteria

1. New event → 201, row in `events` with `status='processed'`, and the n8n webhook received
   the exact envelope with valid `X-Seekly-Signature` (verify by recomputing in a test
   listener).
2. Exact re-POST of a processed event → 200 `duplicate:true`; **zero** additional forwards
   (assert one delivery total on the test listener); still exactly one `events` row.
3. With `N8N_WEBHOOK_BASE_URL` unset → 202, row persists with `status='failed'`,
   `error='n8n forwarding not configured'`; re-POST after setting the var → 200 + forwarded,
   row flips to `processed`.
4. Same `event_id` with a different `client_id` → 409, no row modified.
5. 400/401/404/413/422/429 paths all return the documented codes (tests hit each).
6. Handler never throws on forward failure — the event is never lost after passing validation.

---

## API-4 — Token vault crypto + access-token refresh

### 4a. New file `src/server/connections/crypto.ts` (complete)

```ts
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";
import { env } from "@/env";

/**
 * Application-layer sealing for OAuth tokens at rest (doc 02 layer 2):
 * AES-256-GCM, key ONLY in env (CONNECTIONS_ENCRYPTION_KEY) — tokens are
 * unreadable even with full DB access. Wire format: base64( iv(12) ‖ tag(16) ‖
 * ciphertext ). Distinct from src/server/vault (Bitwarden, human credentials):
 * this vault is hot-path, per-request decryption for machine tokens.
 */
const ALG = "aes-256-gcm";

export class TokenCryptoNotConfiguredError extends Error {}

function key(): Buffer {
  const raw = env.CONNECTIONS_ENCRYPTION_KEY;
  if (!raw) throw new TokenCryptoNotConfiguredError("CONNECTIONS_ENCRYPTION_KEY is not set");
  const buf = Buffer.from(raw, "base64");
  if (buf.length !== 32) {
    throw new TokenCryptoNotConfiguredError(
      "CONNECTIONS_ENCRYPTION_KEY must be 32 bytes base64 (openssl rand -base64 32)",
    );
  }
  return buf;
}

export function tokenCryptoConfigured(): boolean {
  try {
    key();
    return true;
  } catch {
    return false;
  }
}

export function encryptToken(plaintext: string): string {
  const iv = randomBytes(12);
  const cipher = createCipheriv(ALG, key(), iv);
  const enc = Buffer.concat([cipher.update(plaintext, "utf8"), cipher.final()]);
  return Buffer.concat([iv, cipher.getAuthTag(), enc]).toString("base64");
}

export function decryptToken(sealed: string): string {
  const buf = Buffer.from(sealed, "base64");
  const decipher = createDecipheriv(ALG, key(), buf.subarray(0, 12));
  decipher.setAuthTag(buf.subarray(12, 28));
  return Buffer.concat([decipher.update(buf.subarray(28)), decipher.final()]).toString("utf8");
}
```

### 4b. New file `src/server/connections/tokens.ts` (complete)

```ts
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { connections } from "@/server/db/schema";
import { decryptToken, encryptToken } from "./crypto";
import { env } from "@/env";
import { logError } from "@/lib/logger";

/**
 * Short-lived access tokens for n8n (doc 02: "n8n receives short-lived access
 * tokens per run, never refresh tokens"). Refreshes when < 5 minutes of life
 * remain; a refresh failure marks the connection BROKEN so dependent modules
 * auto-pause with a visible reason instead of failing silently.
 */
type ConnectionRow = typeof connections.$inferSelect;
const REFRESH_SKEW_MS = 5 * 60 * 1000;

export interface FreshToken {
  accessToken: string;
  expiresAt: Date | null; // null = non-expiring (WP application password)
}

export async function getFreshAccessToken(conn: ConnectionRow): Promise<FreshToken | null> {
  if (conn.status !== "active") return null;

  // Still-valid stored token → decrypt and return.
  if (
    conn.encryptedAccessToken &&
    (conn.expiresAt === null || conn.expiresAt.getTime() - Date.now() > REFRESH_SKEW_MS)
  ) {
    return { accessToken: decryptToken(conn.encryptedAccessToken), expiresAt: conn.expiresAt };
  }

  if (!conn.encryptedRefreshToken) {
    await markBroken(conn.id, "access token expired and no refresh token stored");
    return null;
  }

  const refreshed = await refreshWithProvider(conn.provider, decryptToken(conn.encryptedRefreshToken));
  if (!refreshed) {
    await markBroken(conn.id, `token refresh failed for provider ${conn.provider}`);
    return null;
  }

  await db
    .update(connections)
    .set({
      encryptedAccessToken: encryptToken(refreshed.accessToken),
      expiresAt: refreshed.expiresAt,
      status: "active",
      lastVerifiedAt: new Date(),
      lastError: null,
      updatedAt: new Date(),
    })
    .where(eq(connections.id, conn.id));
  return refreshed;
}

async function markBroken(connectionId: string, reason: string): Promise<void> {
  logError("connections: marking broken", connectionId, reason);
  await db
    .update(connections)
    .set({ status: "broken", lastError: reason, updatedAt: new Date() })
    .where(eq(connections.id, connectionId));
}

/**
 * Provider refresh registry. google = standard OAuth refresh grant.
 * wordpress = application password (stored in encryptedAccessToken,
 * non-expiring — never reaches here). meta = long-lived page token
 * (60d, re-acquired by the connection health cron, not per-request) —
 * returning null here surfaces as BROKEN, which is the correct behavior
 * when a page token has aged out.
 */
async function refreshWithProvider(
  provider: string,
  refreshToken: string,
): Promise<FreshToken | null> {
  if (provider === "google") return refreshGoogle(refreshToken);
  return null;
}

async function refreshGoogle(refreshToken: string): Promise<FreshToken | null> {
  if (!env.GOOGLE_OAUTH_CLIENT_ID || !env.GOOGLE_OAUTH_CLIENT_SECRET) return null;
  try {
    const res = await fetch("https://oauth2.googleapis.com/token", {
      method: "POST",
      headers: { "content-type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        client_id: env.GOOGLE_OAUTH_CLIENT_ID,
        client_secret: env.GOOGLE_OAUTH_CLIENT_SECRET,
        refresh_token: refreshToken,
        grant_type: "refresh_token",
      }),
      signal: AbortSignal.timeout(10_000),
    });
    if (!res.ok) return null;
    const data = (await res.json()) as { access_token: string; expires_in: number };
    return {
      accessToken: data.access_token,
      expiresAt: new Date(Date.now() + data.expires_in * 1000),
    };
  } catch {
    return null;
  }
}
```

### Acceptance criteria

1. `encryptToken`/`decryptToken` round-trip; flipping one ciphertext byte makes decrypt throw
   (GCM auth), never return garbage.
2. Missing/short `CONNECTIONS_ENCRYPTION_KEY` → `TokenCryptoNotConfiguredError`, not silent.
3. `getFreshAccessToken` on a fresh token performs **zero** network calls (unit test with a
   fetch spy).
4. A failed Google refresh flips the row to `status='broken'` with `lastError` set.
5. No code path ever returns or logs a refresh token.

---

## API-5 — `GET /api/internal/clients/[id]/context?module=`

One call at the top of every n8n run (doc 02). Returns profile + requested module's config +
ALL switches + comms + connections with **short-lived access tokens** (refresh tokens never
leave the DB). `?module=` is optional; when present it must be one of the 8 module slugs and
gates which providers get tokens decrypted (least exposure per run).

### Response schema — new file `src/server/internal/context-schema.ts` (complete)

```ts
import { z } from "zod";

/** Which providers each module's workflow needs tokens for (least-exposure map). */
export const MODULE_PROVIDERS: Record<string, string[]> = {
  review_automation: ["google"],
  speed_to_lead: ["meta"],
  reactivation: [],
  content_engine: ["google", "wordpress"],
  directory_sync: ["google", "brightlocal"],
  competitor_intel: [],
  social_syndication: ["google", "meta"],
  intelligence: [],
};

export const moduleSlugSchema = z.enum([
  "review_automation", "speed_to_lead", "reactivation", "content_engine",
  "directory_sync", "competitor_intel", "social_syndication", "intelligence",
]);

/** The contract n8n workflows are written against — snake_case, additive-only changes. */
export const contextResponseSchema = z.object({
  client: z.object({
    id: z.string().uuid(),
    slug: z.string(),
    name: z.string(),
    timezone: z.string(),
    industry: z.string().nullable(),
    tier: z.enum(["pilot", "core", "growth", "market_leader"]).nullable(),
    account_status: z.enum(["prospect", "onboarding", "active", "paused", "churned"]),
  }),
  profile: z.object({
    services: z.array(z.string()),
    service_areas: z.array(z.string()),
    target_keywords: z.array(z.string()),
    competitors: z.array(z.record(z.string(), z.unknown())),
    brand_voice: z.record(z.string(), z.unknown()),
    ai_guardrails: z.array(z.string()),
    qualification_questions: z.array(z.record(z.string(), z.unknown())),
    campaign_angles: z.array(z.record(z.string(), z.unknown())),
    booking_link: z.string().nullable(),
    website_url: z.string().nullable(),
    website_platform: z.string().nullable(),
    review_link: z.string().nullable(),
    approval_preferences: z.record(z.string(), z.string()),
    escalation_contacts: z.array(z.record(z.string(), z.unknown())),
    hours_json: z.record(z.string(), z.unknown()).nullable(),
    gbp_location_id: z.string().nullable(),
    place_id: z.string().nullable(),
    brightlocal_location_id: z.string().nullable(),
  }).nullable(), // null until intake creates the row
  /** The requested module (echo of ?module=), with merged settings; null when no ?module=. */
  module: z.object({
    name: moduleSlugSchema,
    enabled: z.boolean(),
    settings: z.record(z.string(), z.unknown()),
  }).nullable(),
  /** ALL switches — engines gate on their own, WF-8 reads the whole board. */
  switches: z.record(z.string(), z.boolean()),
  connections: z.array(z.object({
    provider: z.string(),
    provider_account_id: z.string().nullable(),
    status: z.enum(["pending", "active", "broken", "revoked"]),
    scopes: z.array(z.string()),
    metadata: z.record(z.string(), z.unknown()),
    /** Present ONLY for active connections whose provider is in MODULE_PROVIDERS[module]. */
    access_token: z.string().nullable(),
    access_token_expires_at: z.string().nullable(),
  })),
  comms: z.object({
    phone_number: z.string().nullable(),
    inbound_email_address: z.string().nullable(),
    twilio_subaccount_sid: z.string().nullable(),
    a2p_brand_status: z.enum(["none", "submitted", "approved", "rejected"]),
    a2p_campaign_status: z.enum(["none", "submitted", "approved", "rejected"]),
  }).nullable(),
});
export type ContextResponse = z.infer<typeof contextResponseSchema>;
```

### Route — new file `src/app/api/internal/clients/[id]/context/route.ts` (complete)

```ts
import { and, eq, isNull } from "drizzle-orm";
import { z } from "zod";
import { db } from "@/server/db";
import {
  clientProfile, clients, commsProvisioning, connections, workflowConfig,
} from "@/server/db/schema";
import { requireInternalToken } from "@/server/internal/auth";
import { allowInternalRequest, rateLimited } from "@/server/internal/rate-limit";
import {
  contextResponseSchema, MODULE_PROVIDERS, moduleSlugSchema, type ContextResponse,
} from "@/server/internal/context-schema";
import { getFreshAccessToken } from "@/server/connections/tokens";

const NO_STORE = { "cache-control": "no-store" } as const;

export async function GET(request: Request, ctx: { params: Promise<{ id: string }> }) {
  const unauth = requireInternalToken(request);
  if (unauth) return unauth;
  if (!allowInternalRequest("internal:context", 120)) return rateLimited();

  const { id } = await ctx.params;
  if (!z.string().uuid().safeParse(id).success) {
    return Response.json({ error: "invalid_client_id" }, { status: 400, headers: NO_STORE });
  }
  const moduleParam = new URL(request.url).searchParams.get("module");
  const parsedModule = moduleParam === null ? null : moduleSlugSchema.safeParse(moduleParam);
  if (parsedModule && !parsedModule.success) {
    return Response.json(
      { error: "validation_error", issues: parsedModule.error.issues },
      { status: 422, headers: NO_STORE },
    );
  }
  const module = parsedModule?.data ?? null;

  const [client] = await db
    .select()
    .from(clients)
    .where(and(eq(clients.id, id), isNull(clients.deletedAt)));
  if (!client) {
    return Response.json({ error: "unknown_client" }, { status: 404, headers: NO_STORE });
  }

  const [profile] = await db.select().from(clientProfile).where(eq(clientProfile.clientId, id));
  const configs = await db.select().from(workflowConfig).where(eq(workflowConfig.clientId, id));
  const [comms] = await db
    .select()
    .from(commsProvisioning)
    .where(eq(commsProvisioning.clientId, id));
  const conns = await db.select().from(connections).where(eq(connections.clientId, id));

  const moduleRow = module ? (configs.find((c) => c.module === module) ?? null) : null;
  const tokenProviders = module ? new Set(MODULE_PROVIDERS[module]) : new Set<string>();

  const connsOut: ContextResponse["connections"] = [];
  for (const conn of conns) {
    let token: { accessToken: string; expiresAt: Date | null } | null = null;
    if (conn.status === "active" && tokenProviders.has(conn.provider)) {
      token = await getFreshAccessToken(conn); // marks the row broken on refresh failure
    }
    connsOut.push({
      provider: conn.provider,
      provider_account_id: conn.providerAccountId,
      status: token === null && tokenProviders.has(conn.provider) && conn.status === "active"
        ? "broken" // just flipped by getFreshAccessToken; report reality, not the stale read
        : conn.status,
      scopes: conn.scopes,
      metadata: conn.metadata,
      access_token: token?.accessToken ?? null,
      access_token_expires_at: token?.expiresAt?.toISOString() ?? null,
    });
  }

  const response: ContextResponse = {
    client: {
      id: client.id,
      slug: client.slug,
      name: client.name,
      timezone: client.timezone,
      industry: client.industry,
      tier: client.tier,
      account_status: client.accountStatus,
    },
    profile: profile
      ? {
          services: profile.services,
          service_areas: profile.serviceAreas,
          target_keywords: profile.targetKeywords,
          competitors: profile.competitors,
          brand_voice: profile.brandVoice,
          ai_guardrails: profile.aiGuardrails,
          qualification_questions: profile.qualificationQuestions,
          campaign_angles: profile.campaignAngles,
          booking_link: profile.bookingLink,
          website_url: profile.websiteUrl,
          website_platform: profile.websitePlatform,
          review_link: profile.reviewLink,
          approval_preferences: profile.approvalPreferences,
          escalation_contacts: profile.escalationContacts,
          hours_json: profile.hoursJson,
          gbp_location_id: profile.gbpLocationId,
          place_id: profile.placeId,
          brightlocal_location_id: profile.brightlocalLocationId,
        }
      : null,
    module: moduleRow
      ? { name: moduleRow.module, enabled: moduleRow.enabled, settings: moduleRow.settings }
      : module
        ? { name: module, enabled: false, settings: {} } // no row yet = off, empty settings
        : null,
    switches: Object.fromEntries(configs.map((c) => [c.module, c.enabled])),
    connections: connsOut,
    comms: comms
      ? {
          phone_number: comms.phoneNumber,
          inbound_email_address: comms.inboundEmailAddress,
          twilio_subaccount_sid: comms.twilioSubaccountSid,
          a2p_brand_status: comms.a2pBrandStatus,
          a2p_campaign_status: comms.a2pCampaignStatus,
        }
      : null,
  };

  // Contract self-check in dev only (cheap regression net; never 500s prod on a drift).
  if (process.env.NODE_ENV !== "production") contextResponseSchema.parse(response);
  return Response.json(response, { headers: NO_STORE });
}
```

### curl example

```bash
curl -sS "$APP_URL/api/internal/clients/11111111-2222-4333-a444-555555555555/context?module=review_automation" \
  -H "Authorization: Bearer $N8N_SERVICE_TOKEN"
# → 200 { "client": {...}, "profile": {...},
#         "module": {"name":"review_automation","enabled":true,"settings":{"sendDelayHours":2,...}},
#         "switches": {"review_automation":true,"speed_to_lead":true,...},
#         "connections": [{"provider":"google","status":"active","access_token":"ya29....","access_token_expires_at":"..."}],
#         "comms": {"phone_number":"+16135550142","a2p_campaign_status":"approved",...} }
```

### Acceptance criteria

1. Response validates against `contextResponseSchema`; header `Cache-Control: no-store` set on
   every status.
2. `?module=review_automation` returns a `google` access token (when active) and **never** a
   token for providers outside `MODULE_PROVIDERS[module]`; no `?module=` → all
   `access_token: null`.
3. No response, log line, or error ever contains a refresh token or an `encrypted_*` value.
4. Client with no profile/comms rows → those keys are `null`, not 500; unknown module slug →
   422; module with no config row → `enabled:false, settings:{}`.
5. When Google refresh fails mid-request, the response shows that connection as `broken` with
   `access_token: null`, and the DB row is `broken` (portal banner path).

---

## API-6 — `POST /api/internal/activity`

### Route — new file `src/app/api/internal/activity/route.ts` (complete)

```ts
import { and, eq, isNull, inArray } from "drizzle-orm";
import { z } from "zod";
import { db } from "@/server/db";
import { activityLog, clients } from "@/server/db/schema";
import { requireInternalToken } from "@/server/internal/auth";
import { allowInternalRequest, rateLimited } from "@/server/internal/rate-limit";

/**
 * Activity-log ingestion (doc 02): every n8n workflow POSTs what it did.
 * Accepts one entry or {entries:[...]} (max 100). Idempotent on
 * (client_id, dedupe_key) — n8n retry policies can re-POST freely.
 */
const NO_STORE = { "cache-control": "no-store" } as const;
const MAX_BODY_BYTES = 256 * 1024;

const entrySchema = z.object({
  client_id: z.string().uuid(),
  /** Open registry: the 8 module slugs + "provisioning" | "send_pipeline" | "intelligence" | "system". */
  engine: z.string().min(1).max(64),
  /** Dotted verb, e.g. "review_request.sent". */
  action: z.string().min(1).max(128),
  status: z.enum(["ok", "failed", "skipped"]).default("ok"),
  entity_type: z.string().max(64).optional(),
  entity_id: z.string().max(200).optional(),
  detail: z.record(z.string(), z.unknown()).optional(),
  dedupe_key: z.string().min(1).max(200).optional(),
  /** When the action happened, if not "now" (backfills, batch flushes). */
  created_at: z.string().datetime({ offset: true }).optional(),
});
const bodySchema = z.union([
  entrySchema.transform((e) => ({ entries: [e] })),
  z.object({ entries: z.array(entrySchema).min(1).max(100) }),
]);

export async function POST(request: Request) {
  const unauth = requireInternalToken(request);
  if (unauth) return unauth;
  if (!allowInternalRequest("internal:activity", 300)) return rateLimited();

  const rawBody = await request.text();
  if (Buffer.byteLength(rawBody) > MAX_BODY_BYTES) {
    return Response.json({ error: "payload_too_large" }, { status: 413, headers: NO_STORE });
  }
  let body: unknown;
  try {
    body = JSON.parse(rawBody);
  } catch {
    return Response.json({ error: "invalid_json" }, { status: 400, headers: NO_STORE });
  }
  const parsed = bodySchema.safeParse(body);
  if (!parsed.success) {
    return Response.json(
      { error: "validation_error", issues: parsed.error.issues },
      { status: 422, headers: NO_STORE },
    );
  }
  const { entries } = parsed.data;

  // All referenced clients must exist and be live — reject the whole batch
  // otherwise (a partial write would make retries ambiguous).
  const clientIds = [...new Set(entries.map((e) => e.client_id))];
  const liveRows = await db
    .select({ id: clients.id })
    .from(clients)
    .where(and(inArray(clients.id, clientIds), isNull(clients.deletedAt)));
  if (liveRows.length !== clientIds.length) {
    const live = new Set(liveRows.map((r) => r.id));
    return Response.json(
      { error: "unknown_client", client_ids: clientIds.filter((id) => !live.has(id)) },
      { status: 404, headers: NO_STORE },
    );
  }

  const inserted = await db
    .insert(activityLog)
    .values(
      entries.map((e) => ({
        clientId: e.client_id,
        engine: e.engine,
        action: e.action,
        status: e.status,
        entityType: e.entity_type ?? null,
        entityId: e.entity_id ?? null,
        detail: e.detail ?? {},
        dedupeKey: e.dedupe_key ?? null,
        ...(e.created_at ? { createdAt: new Date(e.created_at) } : {}),
      })),
    )
    .onConflictDoNothing({ target: [activityLog.clientId, activityLog.dedupeKey] })
    .returning({ id: activityLog.id });

  return Response.json(
    { inserted: inserted.length, deduped: entries.length - inserted.length },
    { headers: NO_STORE },
  );
}
```

### curl example

```bash
curl -sS -X POST "$APP_URL/api/internal/activity" \
  -H "Authorization: Bearer $N8N_SERVICE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entries":[
    {"client_id":"11111111-2222-4333-a444-555555555555","engine":"review_automation",
     "action":"review_request.sent","status":"ok","entity_type":"message",
     "entity_id":"9c1e...","detail":{"channel":"sms"},
     "dedupe_key":"5f0c9d5e-8a3f-4d1c-9b6e-2f7a1c4d8e90:wf1:request_sent"},
    {"client_id":"11111111-2222-4333-a444-555555555555","engine":"speed_to_lead",
     "action":"lead.first_touch","detail":{"response_seconds":24}}
  ]}'
# → 200 {"inserted":2,"deduped":0}   (re-run → {"inserted":1,"deduped":1} — only the keyless row re-inserts)
```

### Acceptance criteria

1. Single-entry body (no `entries` wrapper) and batch body both accepted; 101 entries → 422.
2. Replaying a batch with `dedupe_key`s inserts 0 duplicates and reports `deduped` correctly.
3. Batch containing one unknown `client_id` → 404 listing the offending id(s), **zero rows
   written** (verify count unchanged).
4. `created_at` omitted → DB default now; provided → stored as given.

---

## API-7 — `POST /api/internal/send`

Thin HTTP wrapper over the shared send-pipeline. **Spec 03 owns the internals** (opt-out
check → quiet hours → freq cap → A2P gate → ledger write → Twilio → callbacks); this ticket
defines the HTTP contract, the pipeline's TS interface, and the route that binds them. n8n
engines call this and NEVER Twilio directly (doc 03).

### 7a. Pipeline interface — new file `src/server/send/pipeline.ts` (interface + temporary stub)

```ts
import type { blockedReasonEnum, messageKindEnum, workflowModuleEnum } from "@/server/db/schema";

/**
 * The shared send-pipeline contract (doc 03 "shared send-pipeline"). Spec 03
 * replaces the stub with the real implementation; the HTTP contract in
 * /api/internal/send is written against THIS interface and must not change.
 */
export type WorkflowModule = (typeof workflowModuleEnum.enumValues)[number];
export type MessageKind = (typeof messageKindEnum.enumValues)[number];
export type BlockedReason = (typeof blockedReasonEnum.enumValues)[number];

export interface SendRequest {
  clientId: string;
  engine: WorkflowModule;
  kind: MessageKind;
  /** Resolve the destination: customerId preferred; raw phone allowed for lead first-touch. */
  customerId?: string;
  phone?: string; // E.164
  body: string;
  mediaUrls?: string[];
  /** REQUIRED — the duplicate-send backstop, unique per client (messages ledger). */
  idempotencyKey: string;
  campaignId?: string;
  conversationId?: string;
}

export type SendResult =
  | { outcome: "sent"; messageId: string; deduped: boolean }
  | { outcome: "queued"; messageId: string; deduped: boolean; queuedUntil: string }
  | { outcome: "blocked"; messageId: string; blockedReason: BlockedReason };

export class UnknownCustomerError extends Error {}

/** Stub until spec 03 lands: fail loudly, never pretend to send. */
export async function sendMessage(_req: SendRequest): Promise<SendResult> {
  throw new Error("send pipeline not implemented yet (spec 03)");
}
```

### 7b. Route — new file `src/app/api/internal/send/route.ts` (complete)

```ts
import { and, eq, isNull } from "drizzle-orm";
import { z } from "zod";
import { db } from "@/server/db";
import { clients } from "@/server/db/schema";
import { requireInternalToken } from "@/server/internal/auth";
import { allowInternalRequest, rateLimited } from "@/server/internal/rate-limit";
import { sendMessage, UnknownCustomerError } from "@/server/send/pipeline";
import { logError } from "@/lib/logger";

const NO_STORE = { "cache-control": "no-store" } as const;
const MAX_BODY_BYTES = 64 * 1024;

const sendSchema = z
  .object({
    client_id: z.string().uuid(),
    engine: z.enum([
      "review_automation", "speed_to_lead", "reactivation", "content_engine",
      "directory_sync", "competitor_intel", "social_syndication", "intelligence",
    ]),
    kind: z.enum(["transactional", "conversational", "marketing"]),
    customer_id: z.string().uuid().optional(),
    phone: z.string().min(7).max(32).optional(),
    body: z.string().min(1).max(1600), // Twilio hard limit
    media_urls: z.array(z.string().url()).max(10).optional(),
    idempotency_key: z.string().min(1).max(200),
    campaign_id: z.string().uuid().optional(),
    conversation_id: z.string().uuid().optional(),
  })
  .refine((v) => v.customer_id || v.phone, { message: "customer_id or phone required" });

export async function POST(request: Request) {
  const unauth = requireInternalToken(request);
  if (unauth) return unauth;
  if (!allowInternalRequest("internal:send", 60)) return rateLimited();

  const rawBody = await request.text();
  if (Buffer.byteLength(rawBody) > MAX_BODY_BYTES) {
    return Response.json({ error: "payload_too_large" }, { status: 413, headers: NO_STORE });
  }
  let body: unknown;
  try {
    body = JSON.parse(rawBody);
  } catch {
    return Response.json({ error: "invalid_json" }, { status: 400, headers: NO_STORE });
  }
  const parsed = sendSchema.safeParse(body);
  if (!parsed.success) {
    return Response.json(
      { error: "validation_error", issues: parsed.error.issues },
      { status: 422, headers: NO_STORE },
    );
  }
  const req = parsed.data;

  const [client] = await db
    .select({ id: clients.id })
    .from(clients)
    .where(and(eq(clients.id, req.client_id), isNull(clients.deletedAt)));
  if (!client) {
    return Response.json({ error: "unknown_client" }, { status: 404, headers: NO_STORE });
  }

  try {
    const result = await sendMessage({
      clientId: req.client_id,
      engine: req.engine,
      kind: req.kind,
      customerId: req.customer_id,
      phone: req.phone,
      body: req.body,
      mediaUrls: req.media_urls,
      idempotencyKey: req.idempotency_key,
      campaignId: req.campaign_id,
      conversationId: req.conversation_id,
    });
    // Compliance refusals are SUCCESSFUL handling (200) — the pipeline did its
    // job; n8n branches on `outcome`, it does not retry.
    return Response.json(
      result.outcome === "blocked"
        ? { outcome: "blocked", message_id: result.messageId, blocked_reason: result.blockedReason }
        : {
            outcome: result.outcome,
            message_id: result.messageId,
            deduped: result.deduped,
            ...(result.outcome === "queued" ? { queued_until: result.queuedUntil } : {}),
          },
      { headers: NO_STORE },
    );
  } catch (err) {
    if (err instanceof UnknownCustomerError) {
      return Response.json({ error: "unknown_customer" }, { status: 404, headers: NO_STORE });
    }
    logError("internal/send failed", req.client_id, req.idempotency_key, err);
    return Response.json({ error: "internal_error" }, { status: 500, headers: NO_STORE });
  }
}
```

### `blocked_reason` codes (mirror of `blockedReasonEnum`, DB spec DB-6)

| Code | Meaning | Doc 03 pipeline step |
|---|---|---|
| `opt_out` | Contact is on the cross-engine suppression ledger | 1 |
| `quiet_hours` | Send cancelled at the window edge (normal deferral returns `outcome:"queued"` instead) | 2 |
| `freq_cap` | Marketing message inside the per-customer cap window (default 30d) | 3 |
| `a2p_pending` | Client's A2P campaign not `approved` — hard gate | 4 |
| `no_consent` | Marketing send without a recorded consent basis | 1/3 |
| `no_destination` | No usable phone for the customer | — |
| `module_disabled` | Engine's switch is off at send time | — |

### curl example

```bash
curl -sS -X POST "$APP_URL/api/internal/send" \
  -H "Authorization: Bearer $N8N_SERVICE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "11111111-2222-4333-a444-555555555555",
    "engine": "review_automation",
    "kind": "transactional",
    "customer_id": "22222222-3333-4444-a555-666666666666",
    "body": "Thanks for coming in, Jordan! Mind leaving us a quick review? https://g.page/r/x/review",
    "idempotency_key": "5f0c9d5e-8a3f-4d1c-9b6e-2f7a1c4d8e90:review_request"
  }'
# → 200 {"outcome":"sent","message_id":"...","deduped":false}
# same idempotency_key again → 200 {"outcome":"sent","message_id":"...","deduped":true}  (no second SMS)
# opted-out customer → 200 {"outcome":"blocked","message_id":"...","blocked_reason":"opt_out"}
```

### Acceptance criteria

1. Contract tests (with `sendMessage` mocked): all three outcome shapes serialize exactly as
   documented (snake_case, `queued_until` only on queued, `blocked_reason` only on blocked).
2. Neither `customer_id` nor `phone` → 422; unknown client → 404; `UnknownCustomerError` → 404
   `unknown_customer`.
3. Blocked outcomes return HTTP **200** (never 4xx) — asserted in a test with a comment
   explaining why (n8n must branch, not retry).
4. With the spec-03 stub in place, a real call returns 500 `internal_error` and logs — it can
   never fake a send.

---

## API-8 — Tests + curl smoke pack

1. **Vitest** (colocated `route.test.ts` / `*.test.ts`, matching `src/app/api/spine/v1/*`
   test style): auth guard (both tokens), events idempotency + 409 + forward-failure 202
   (mock `fetch`), schema suites from API-2, activity dedupe/batch-atomicity, send contract
   shapes, crypto round-trip. Use the seeded growth tenant (`npm run db:seed:growth`) where a
   DB is needed.
2. **Smoke pack** — new file `scripts/smoke-internal-api.sh`: the four curl examples above
   verbatim, parameterized by `APP_URL` + `N8N_SERVICE_TOKEN` + `CLIENT_ID`, exiting non-zero
   on any unexpected HTTP code (use `curl -sS -o /dev/null -w "%{http_code}"` per call).
   Referenced from the repo README's command table.

### Acceptance criteria

1. `npm test` green with all new suites; no test requires n8n to be running (fetch mocked).
2. `bash scripts/smoke-internal-api.sh` against a local dev server + seeded DB exits 0.
3. A deliberate schema drift (rename one response field) fails at least one test — proving the
   contract is pinned.

---

## Open questions (for the tech lead, non-blocking)

1. **Event replay sweep:** 202-persisted-but-unforwarded events currently rely on manual
   re-POST. Should a pg-boss cron re-forward `status IN ('pending','failed')` events
   automatically (bounded retries)? Recommended: yes, ~5-minute cadence — small follow-up
   ticket.
2. **Meta long-lived page tokens:** health-cron re-acquisition (60-day expiry) is asserted in
   API-4's comment but not yet ticketed — belongs to the Integration Hub spec.
3. **Per-client rate limits:** current limits are global per route. Fine for 1–3 pilots;
   revisit when one noisy adapter can starve others.
