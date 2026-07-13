# 02 — Architecture

## Design principles

1. **n8n is the worker, never the product.** Seekly owns the tenant, authorization, and
   configuration layers in its own control plane. n8n executes workflows; it is never
   client-facing and its credential store is NOT the multi-tenant integration architecture
   (n8n workflow/credential sharing has its own semantics — client isolation is designed
   deliberately in our own token vault).
2. **One master workflow per engine, multi-tenant by `client_id`.** Never N copies per client.
   Every trigger resolves the client, loads config + connections, executes standardized steps,
   and logs the result.
3. **Workflows consume canonical events, never vendor payloads.** Client software is adapted
   *into* Seekly's event model at the edge. Engines are written once and stay frozen while
   adapters proliferate.
4. **Per-client switchboard.** Every engine checks its module toggle at runtime. Flipping a
   switch in the portal enables/disables an engine instantly, with no deploys.
5. **Generic before specific.** No vendor adapters are built speculatively. The generic paths
   (webhook, email-parse, CSV) work with any software on day one; native OAuth/API adapters are
   added per client based on the intake form, when they buy seamlessness.
6. **One clear responsibility per system.** HubSpot = Seekly's sales. Control plane = tenancy,
   config, client data. n8n = execution. Portal = the client's window. Bidirectional event
   sync between them; never database mirroring; never mixed datasets.

## System diagram

```
                     SEEKLY INTERNAL
              HubSpot (Seekly's own sales)
           Prospect → Deal → Closed Won
                        │ webhook
                        ▼
              ┌───────────────────────┐
              │   n8n Provisioning    │  create tenant, user, onboarding
              └──────────┬────────────┘  record, integration checklist
                         ▼
┌─────────────────────────────────────────────────────────┐
│                 SEEKLY CLIENT PORTAL                    │
│  Onboarding · Integration Hub · Dashboards · Approvals  │
│  "Connect Google"  "Connect Facebook"  "Connect Website"│
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│              SEEKLY CONTROL PLANE (API + DB)            │
│   Tenants · Users · Client Config · Workflow Switches   │
│   Canonical Events · Customers · Messages · Activity    │
│         Integration/Auth Service + Token Vault          │
│              (encrypted OAuth tokens at rest)           │
└───────┬───────────────────────────────────────┬─────────┘
        │  canonical events (webhooks/queue)    │ config + tokens (API)
        ▼                                       ▼
┌──────────────────┐                 ┌─────────────────────────┐
│  INGESTION EDGE  │                 │    n8n ORCHESTRATOR     │
│ per-client hooks │                 │  7 master workflows +   │
│ email-parse inbox│ ──────────────▶ │  provisioning, all      │
│ CSV import, polls│                 │  multi-tenant           │
└──────────────────┘                 └───────────┬─────────────┘
   ▲ POS, forms, Meta,                           ▼
   │ GBP, phone, etc.                    External APIs
   └── client's existing software        (GBP, Meta, Twilio,
                                          BrightLocal, WordPress)
```

## Layer 1 — Control plane (source of truth)

Owns per client: identity/tenancy, business profile (NAP, hours, services, service areas,
target keywords, competitors, brand voice), connections, workflow config (switches + per-module
settings), canonical customer records, message ledger, opt-outs, activity log, content items,
campaigns. Full schema in [04-data-model.md](04-data-model.md).

Exposed as a small internal API that both the portal and n8n consume:

- `GET /internal/clients/:id/context?module=review_automation` → one call returns the client's
  profile, module config, toggles, and decrypted tokens needed by a workflow run (n8n
  authenticates with a service token; responses are never cached in n8n).
- `POST /internal/events` → adapters emit canonical events here; the control plane validates,
  dedupes (idempotency key), persists, and forwards to the right n8n webhook.
- `POST /internal/activity` → workflows write every action taken (powers dashboards + reports).

## Layer 2 — Integration/Auth service + token vault

- **One Seekly OAuth app per provider** (Google, Meta, HubSpot, etc.) — clients NEVER create
  their own cloud projects. Client clicks "Connect Google" → standard consent → callback →
  encrypted refresh token stored → available locations/accounts fetched → client selects →
  connection saved to `client_id` → dependent workflows can be activated.
- Token records: `client_id, provider, provider_account_id, encrypted_access_token,
  encrypted_refresh_token, scopes, expires_at, connection_status`.
- Encryption at application level (libsodium sealed box / AES-256-GCM with a key held only by
  the control plane service), TLS in transit, tokens revoked + deleted on disconnect.
- Never store or pass client passwords. n8n receives short-lived access tokens per run, never
  refresh tokens.
- Connection health monitor (cron): refresh failures → `connection_status = broken` → client
  + Seekly alerted → affected modules auto-pause with a visible reason instead of failing
  silently.

## Layer 3 — Canonical event model (the integration keystone)

Engines subscribe to a small, frozen vocabulary:

| Event | Emitted when | Consumed by |
|---|---|---|
| `sale.completed` | POS sale / booking finishes | Review Velocity |
| `lead.created` | Any lead source fires | Speed-to-Lead |
| `message.received` | Inbound SMS from a customer | Conversation router (Speed-to-Lead / Reactivation) |
| `call.missed` | Phone rings out / after hours (v1.1) | Speed-to-Lead |
| `review.received` | New review detected on GBP | Review Velocity |
| `customer.imported` | Customer record synced/uploaded | Reactivation (audience building) |
| `content.published` | Blog post goes live | Social Syndication |
| `profile.updated` | NAP/hours change in portal | Directory Sync |
| `client.provisioned` | HubSpot Closed Won | Provisioning |

Event envelope (all events):

```json
{
  "event_id": "uuid (idempotency key)",
  "client_id": "uuid",
  "type": "sale.completed",
  "occurred_at": "ISO-8601",
  "source": "square|email-parse|webhook|csv|manual|gbp-poll|meta",
  "payload": { }
}
```

Rules: consumers must be idempotent on `event_id`; payloads are normalized by adapters (e.g.
`sale.completed` always has `customer: {name, phone, email?}`, `amount?`, `items?`); unknown
fields are preserved in `payload.raw` for later adapter improvements.

## The adapter ladder (how any client software connects)

Built generic-first; native adapters added per client per the intake form:

| Rung | Mechanism | Works with | Effort |
|---|---|---|---|
| 1 | **Native OAuth/API adapter** (Square, Skedda, Mindbody, Jobber, …) | Platforms with APIs | Per-platform, built when a client needs it |
| 2 | **Generic inbound webhook** — unique signed URL per client per event type | Anything that can POST or be bridged (Zapier/Make as client-side glue if needed) | Zero code per client |
| 3 | **Email-parse** — per-client inbound address (`{client}@in.seekly.app`); client forwards POS receipts / form notifications; AI extraction normalizes to events | Literally any software that sends email | Zero code; prompt-tuned per template |
| 4 | **CSV import / nightly batch** — portal upload or scheduled pull | Legacy systems, initial customer-list loads | Zero code |

The pilot golf venue's POS goes through rung 3 (its "sale/booking completed" notification
emails) on day one, upgraded to a native rung-1 adapter once the intake form confirms the
platform and volume justifies it.

## n8n execution layer

- Self-hosted (Docker on a small VPS), Postgres-backed, HTTPS, basic-auth + IP-restricted
  editor. Webhook endpoints receive only from the control plane (signed).
- Standard shape of every master workflow:
  `Trigger → resolve client_id → GET client context (config+toggles+tokens) → gate: module
  enabled? → execute standardized steps → POST activity log → error branch: retry w/ backoff →
  dead-letter + alert`.
- Idempotency: every run keyed by `event_id`; re-delivery is a no-op.
- Failure policy: 3 retries with exponential backoff on transient API errors; on final failure
  write `activity_log` entry with status `failed`, push alert to Seekly ops channel, and (for
  messaging engines) never risk duplicate sends — check the message ledger before send.
- Observability: n8n execution data ships to the activity log; a daily digest cron reports
  runs/failures per client per module.

## Compliance & messaging guardrails (shared by all SMS engines)

- **One opt-out ledger per client** spanning ALL engines: STOP/UNSUBSCRIBE instantly blocks the
  customer across review requests, lead follow-ups, and reactivation. HELP responses answered.
- Quiet hours enforced centrally (default 8am–9pm client-local; TCPA-safe), sends outside the
  window are queued, not dropped.
- Frequency caps: max 1 marketing message per customer per N days (default 30) across engines;
  transactional/conversational replies exempt.
- A2P 10DLC: every client gets a registered brand + campaign under Seekly's Twilio account
  (subaccount per client). No sends before campaign approval. See
  [06-integrations.md](06-integrations.md).
- Consent basis recorded per customer (`existing_customer`, `lead_inbound`, `imported_list`);
  reactivation sends only to customers with a recorded prior relationship.
