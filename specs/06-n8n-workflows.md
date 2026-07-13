# SPEC-06 — n8n Workflow Implementation Spec

**Audience:** the developer building the workflows in the n8n editor. Every node is named, typed, and configured — build top to bottom, no interpretation needed.
**Sources:** `docs/03-workflows.md` (behavioral spec — this document translates it to nodes), `docs/02-architecture.md`, `docs/04-data-model.md`, `docs/06-integrations.md`, `docs/09-intelligence-platform.md`, and `docs/10-reusing-seekly-client-insights.md` (**supersedes 02/04/05/07 where they conflict**).

**Non-negotiable architecture rules (from doc 10):**

1. The control plane is the existing **seekly-client-insights** app. n8n talks to it **only** through the internal API. n8n never connects to the platform Postgres.
2. Base URL comes from env `SEEKLY_API_URL`; every internal call sends header `Authorization: Bearer {{ $env.N8N_SERVICE_TOKEN }}`.
3. Core internal endpoints: `POST /api/internal/events`, `GET /api/internal/clients/:id/context?module=`, `POST /api/internal/activity`, `POST /api/internal/send`. Additional endpoints used below are listed in §2.7 and must land in SPEC-02 before the dependent ticket starts.
4. **n8n never calls Twilio.** All SMS goes through `POST /api/internal/send` (the shared send-pipeline: opt-outs, quiet hours, frequency caps, A2P gate, message ledger, Twilio).
5. **n8n never holds client OAuth refresh tokens.** The context endpoint returns short-lived access tokens per run. Never cache them in static data, workflow static data, or credentials.

---

## 1. Deployment

### 1.1 docker-compose.yml

Runs on the ops VPS (Hetzner CX22 or equivalent), directory `/opt/seekly-n8n/`. Pin exact image versions — never `:latest`. The pins below are the versions verified when this spec was authored; bump them deliberately (change the pin, redeploy, run the WF-E smoke test) and record the bump in the seekly-platform repo.

```yaml
# /opt/seekly-n8n/docker-compose.yml
services:
  postgres:
    image: postgres:16.9-alpine          # pinned
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n -d n8n"]
      interval: 10s
      timeout: 5s
      retries: 10

  n8n:
    image: n8nio/n8n:1.94.1              # pinned — verify current stable 1.x at deploy time
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "127.0.0.1:5678:5678"            # only the reverse proxy reaches it
    environment:
      # --- database ---
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      # --- identity / URLs (HTTPS terminated at the reverse proxy) ---
      N8N_HOST: n8n.seekly.app
      N8N_PROTOCOL: https
      N8N_PORT: 5678
      WEBHOOK_URL: https://n8n.seekly.app/     # REQUIRED — webhook URLs are built from this
      N8N_EDITOR_BASE_URL: https://n8n.seekly.app/
      # --- security ---
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}   # 32+ random chars; see §1.4
      N8N_DIAGNOSTICS_ENABLED: "false"
      N8N_BLOCK_ENV_ACCESS_IN_NODE: "false"       # expressions must read $env.* (service token, API URL)
      NODE_FUNCTION_ALLOW_BUILTIN: "crypto"       # Code nodes need HMAC verification
      # --- execution data pruning ---
      EXECUTIONS_DATA_PRUNE: "true"
      EXECUTIONS_DATA_MAX_AGE: 336                # hours = 14 days
      EXECUTIONS_DATA_PRUNE_MAX_COUNT: 50000
      EXECUTIONS_DATA_SAVE_ON_SUCCESS: "all"      # observability: doc 02 requires runs visible
      EXECUTIONS_DATA_SAVE_ON_ERROR: "all"
      # --- timezone (Schedule Trigger default; per-client tz enforced by send-pipeline) ---
      GENERIC_TIMEZONE: America/New_York
      TZ: America/New_York
      # --- Seekly control plane ---
      SEEKLY_API_URL: https://app.seekly.app
      N8N_SERVICE_TOKEN: ${N8N_SERVICE_TOKEN}          # bearer token minted by control plane
      SEEKLY_WEBHOOK_HMAC_SECRET: ${SEEKLY_WEBHOOK_HMAC_SECRET}  # verifies inbound event webhooks
      # --- vendor keys held by n8n (Seekly-owned, never client-owned) ---
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      GOOGLE_MAPS_API_KEY: ${GOOGLE_MAPS_API_KEY}      # Places API (public data; WF-1B interim, WF-6)
      BRIGHTLOCAL_API_KEY: ${BRIGHTLOCAL_API_KEY}
      BRIGHTLOCAL_API_SECRET: ${BRIGHTLOCAL_API_SECRET}
      OPS_ALERT_EMAIL: ops@seekly.app
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  pg_data:
  n8n_data:
```

Secrets live in `/opt/seekly-n8n/.env` (chmod 600, root-owned), never in the compose file, never in git.

### 1.2 Reverse proxy — HTTPS is required

Webhook producers (control plane, and nothing else) must reach n8n over HTTPS; Google/Meta OAuth-adjacent calls and the editor also require TLS. Use Caddy (simplest) or nginx + certbot:

```
# /etc/caddy/Caddyfile
n8n.seekly.app {
    # Editor + API: allow only Seekly office/VPN IPs (doc 02: IP-restricted editor)
    @editor not path /webhook/* /webhook-test/*
    handle @editor {
        @blocked not remote_ip 203.0.113.0/24 198.51.100.7   # replace with real allowlist
        respond @blocked 403
        reverse_proxy 127.0.0.1:5678
    }
    # Webhooks: open (authenticated by HMAC signature inside the workflow, §2.4)
    handle {
        reverse_proxy 127.0.0.1:5678
    }
}
```

Notes:

- n8n 1.x uses its own user-management login (owner account) — the legacy `N8N_BASIC_AUTH_*` vars are gone. The doc-02 "basic auth" requirement is satisfied by: owner login + proxy IP allowlist on editor paths. Create exactly one owner account; invite no client users, ever.
- `/webhook/*` stays publicly reachable but every workflow verifies `X-Seekly-Signature` (HMAC-SHA256, §2.4) before doing anything. Unverified requests are dropped with 401.

### 1.3 Backup strategy

| What | How | Cadence | Where |
|---|---|---|---|
| n8n Postgres (workflows, credentials, executions) | `docker compose exec -T postgres pg_dump -U n8n n8n \| gzip > /backups/n8n-$(date +%F).sql.gz` via root cron | Nightly 02:30 | Local `/backups` (7 days) + rclone to object storage (30 days) |
| `N8N_ENCRYPTION_KEY` | Stored in the team password manager the day it is generated | Once | Password manager. **Without this key every credential in the DB is unrecoverable** — a DB restore with the wrong key is useless. |
| Workflow definitions | Git export (§1.4) | After every change | `seekly-platform` repo `workflows/` |
| Restore drill | Restore latest dump + key into a scratch compose stack, open editor, run WF-E test | Monthly (doc 07 hardening: "restore-from-backup drill") | — |

### 1.4 Git export of workflow JSON

Workflows are versioned in the **seekly-platform** repo under `workflows/`, one JSON file per workflow. Credentials are **never** exported (`n8n export:credentials` is forbidden — the repo must contain zero secrets, encrypted or not).

`seekly-platform/scripts/export-workflows.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /opt/seekly-n8n
docker compose exec -T n8n sh -c \
  'rm -rf /tmp/wf && mkdir -p /tmp/wf && n8n export:workflow --all --separate --pretty --output=/tmp/wf'
rm -rf "$REPO/workflows" && mkdir -p "$REPO/workflows"
docker compose cp n8n:/tmp/wf/. "$REPO/workflows/"
# Normalize file names to workflow names (export uses IDs)
cd "$REPO" && node scripts/rename-workflow-exports.js   # reads .name from each JSON, kebab-cases it
git -C "$REPO" add workflows && git -C "$REPO" commit -m "n8n export $(date -u +%FT%TZ)"
```

Process rules:

1. **Export after every change.** Editing a workflow without a same-day export commit is a process violation; the Friday ops checklist runs the script regardless.
2. One JSON per workflow (`--separate`); file name = kebab-cased workflow name, e.g. `wf-1a-review-velocity-requests.json`.
3. Import to a fresh instance: `n8n import:workflow --separate --input=/tmp/wf` then recreate credentials by hand (they are not in git by design), then re-select credentials on nodes that need them.
4. PR review applies to workflow JSON like code: diffs of the JSON are the review artifact.

---

## 2. Conventions

### 2.1 Workflow naming, tags, IDs

One master workflow per engine (multi-tenant by `client_id`) — never a copy per client. Names in n8n exactly as follows:

| n8n workflow name | Module switch checked | Trigger type |
|---|---|---|
| `WF-0 Provisioning` | — (runs pre-switch) | Webhook `client.provisioned` |
| `WF-1A Review Velocity — Requests` | `review_automation` + sub `review_requests` | Webhook `sale.completed` |
| `WF-1B Review Velocity — Review Poller` | `review_automation` | Schedule (15 min) |
| `WF-1C Review Velocity — Responder` | `review_automation` + sub `review_responses` | Webhook `review.received` |
| `WF-2A Speed-to-Lead — First Touch` | `speed_to_lead` | Webhook `lead.created` |
| `WF-2B Speed-to-Lead — Conversation Loop` | `speed_to_lead` (or `reactivation` for campaign replies) | Webhook `message.received` |
| `WF-3 Reactivation — Campaign Runner` | `reactivation` | Webhook `campaign.launch` + Schedule (daily) |
| `WF-4 Content Engine` | `content_engine` | Schedule (cron) |
| `WF-5 Directory Sync` | `directory_sync` | Webhook `profile.updated` + Schedule (quarterly + daily status poll) |
| `WF-6A Espionage — Weekly Snapshot` | `competitor_intel` | Schedule (weekly) |
| `WF-6B Espionage — Monthly Digest` | `competitor_intel` | Schedule (monthly) |
| `WF-7 Syndication` | `social_syndication` + subs `gbp_posts`, `facebook_posts` | Webhook `content.published` |
| `WF-E Error Handler` | — | Error Trigger |

Tags (n8n tag feature) on every workflow: `seekly`, `prod`, and `engine:<module>` (e.g. `engine:review_automation`). WF-E gets `infra`.

### 2.2 Error Workflow (WF-E) — required setting on every workflow

Every workflow's **Settings → Error Workflow** is set to `WF-E Error Handler`. WF-E fires whenever an execution ends in an unhandled error.

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1 | `Error Trigger` | Error Trigger (`n8n-nodes-base.errorTrigger`) | No parameters. Input `$json` contains `workflow.name`, `execution.id`, `execution.url`, `execution.error.message`, `execution.lastNodeExecuted`. |
| 2 | `Build Report` | Code (`n8n-nodes-base.code`) | Mode: Run Once for All Items. JS: pull `client_id` and `event_id` out of `$json.execution` trigger data if present (`const d=$json; const run=d.execution??{}; let client_id=null,event_id=null; try{const td=run.data?.resultData?.runData; /* best effort */}catch(e){}; return [{json:{ workflow: d.workflow?.name, execution_id: run.id, execution_url: run.url, error: run.error?.message, failed_node: run.lastNodeExecuted, client_id, event_id, at: new Date().toISOString() }}];` |
| 3 | `Log Failed Activity` | HTTP Request (`n8n-nodes-base.httpRequest`) | POST `{{ $env.SEEKLY_API_URL }}/api/internal/activity` · Header `Authorization: Bearer {{ $env.N8N_SERVICE_TOKEN }}` · JSON body: `{"client_id": {{ JSON.stringify($json.client_id) }}, "engine": {{ JSON.stringify($json.workflow) }}, "action": "workflow.error", "status": "failed", "entity_type": "n8n_execution", "entity_id": {{ JSON.stringify($json.execution_id) }}, "detail": {{ JSON.stringify($json) }} }` · **On Error: Continue** (an alert must still go out if the control plane itself is down). `client_id: null` is accepted by the activity endpoint and lands in the ops dead-letter queue (SPEC-02). |
| 4 | `Ops Alert Email` | Send Email (`n8n-nodes-base.emailSend`) | Credential: `SMTP Seekly Ops` (SendGrid/Postmark SMTP). To: `{{ $env.OPS_ALERT_EMAIL }}` · Subject: `[n8n FAIL] {{ $json.workflow }} — {{ $json.error }}` · Text: workflow, failed node, error, execution URL. |

This satisfies doc 02's failure policy ("on final failure write activity_log status=failed, push alert to Seekly ops") and doc 07's "failures alert ops within 5 minutes".

### 2.3 Retry settings per node type

Configured per node under **Settings → Retry On Fail**. n8n retries are fixed-interval; treat them as the transient-blip layer. The durable layer is idempotent re-runs (every mutation carries an idempotency key, so replaying a whole execution is safe).

| Node class | Retry On Fail | Max Tries | Wait Between Tries | On Error |
|---|---|---|---|---|
| HTTP → internal API `GET` (context, lookups) | Yes | 3 | 2000 ms | Stop (bubbles to WF-E) |
| HTTP → internal API mutations (`/send`, `/events`, `/activity`, PATCH/POST) | Yes | 3 | 3000 ms | Stop. Safe to retry: **all** mutations are idempotent (idempotency key / upsert semantics). |
| HTTP → external vendor APIs (GBP, Meta Graph, BrightLocal, WordPress, Places) | Yes | 3 | 5000 ms | **Continue (using error output)** → route to the workflow's explicit error branch (park + ops alert). Never let a vendor 500 kill a batch loop. |
| HTTP → Anthropic API | Yes | 2 | 5000 ms | **Continue (using error output)** → deterministic fallback path (each workflow defines one). AI failure must never break a customer-facing promise (doc 03 WF-2). |
| HTTP → competitor websites (WF-6) | No | 1 | — | Continue — unreachable is a *finding*, not an error. |
| Code, IF, Switch, Wait, SplitInBatches | No | — | — | Stop (logic errors must surface in WF-E). |
| Send Email | Yes | 2 | 5000 ms | Continue + activity log `status=failed`. |

Additional per-workflow settings (Workflow → Settings): `Save manual executions: yes`, `Save execution progress: yes` for WF-2A and WF-3 (long-lived Waits / resumable batches), `Timeout: 3600s` default, `Error workflow: WF-E`.

### 2.4 How every workflow starts — the standard head

Two trigger shapes exist. **Every** workflow begins with one of them followed by the same three-node context/gate sequence. Build this once in WF-1A, then copy-paste the head into each new workflow.

**Shape A — Webhook trigger (canonical event pushed by the control plane).** The control plane's event router (SPEC-02) receives adapter events on `POST /api/internal/events`, validates + dedupes on `event_id`, persists, then forwards the canonical envelope to the workflow's production webhook URL with header `X-Seekly-Signature: hex(hmac_sha256(raw_body, SEEKLY_WEBHOOK_HMAC_SECRET))`. Delivery is retried by the control plane on non-2xx (backoff 1m/5m/30m, then dead-letter queue).

Canonical envelope (doc 02) — the webhook body, always:

```json
{
  "event_id": "9f8f6c1e-…",          // idempotency key
  "client_id": "c0ffee-…",
  "type": "sale.completed",
  "occurred_at": "2026-07-13T14:03:00Z",
  "source": "email-parse",
  "payload": { }
}
```

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1 | `Webhook` | Webhook (`n8n-nodes-base.webhook`) | HTTP Method: `POST` · Path: per workflow, e.g. `sale-completed` (URL becomes `https://n8n.seekly.app/webhook/sale-completed`) · Respond: `Using 'Respond to Webhook' Node` · Options → Raw Body: **on** (needed for HMAC). |
| 2 | `Verify Signature` | Code | Run Once for All Items. ```const crypto = require('crypto'); const raw = $input.first().binary?.data ? Buffer.from($input.first().binary.data.data,'base64').toString() : JSON.stringify($input.first().json.body ?? $input.first().json); const sig = $input.first().json.headers['x-seekly-signature']; const expect = crypto.createHmac('sha256', $env.SEEKLY_WEBHOOK_HMAC_SECRET).update(raw).digest('hex'); const ok = !!sig && crypto.timingSafeEqual(Buffer.from(sig,'hex'), Buffer.from(expect,'hex')); const body = typeof $input.first().json.body === 'object' ? $input.first().json.body : JSON.parse(raw); return [{ json: { ok, ...body } }];``` |
| 3 | `Signature OK?` | IF | Condition: Boolean → `{{ $json.ok }}` **is true**. False → node 3f. |
| 3f | `Respond 401` | Respond to Webhook | Response Code `401`, body `{"error":"bad signature"}`. Branch ends. |
| 4 | `Respond 200` | Respond to Webhook (`n8n-nodes-base.respondToWebhook`) | Response Code `200`, body `{"accepted": true, "event_id": "{{ $json.event_id }}"}`. Acks fast; processing continues after this node. |
| 5 | `Get Context` | HTTP Request | GET `{{ $env.SEEKLY_API_URL }}/api/internal/clients/{{ $json.client_id }}/context?module=<MODULE>` · Header `Authorization: Bearer {{ $env.N8N_SERVICE_TOKEN }}` · Retry per §2.3. Response shape in §2.6. |
| 6 | `Module Enabled?` | IF | Boolean → `{{ $('Get Context').item.json.module.enabled }}` **is true**. (Workflows with sub-switches AND them here, e.g. `{{ $('Get Context').item.json.module.enabled && $('Get Context').item.json.module.settings.review_requests !== false }}`.) |
| 6f | `Log Skipped` | HTTP Request | POST `{{ $env.SEEKLY_API_URL }}/api/internal/activity` body `{"client_id":"{{ $('Verify Signature').item.json.client_id }}","engine":"<MODULE>","action":"<event type>.skipped","status":"skipped","entity_type":"event","entity_id":"{{ $('Verify Signature').item.json.event_id }}","detail":{"reason":"module_disabled"}}` → `NoOp` (No Operation, `n8n-nodes-base.noOp`). Branch ends. Instant safe-off (doc 03 switchboard). |

**Shape B — Schedule trigger (polling/cron workflows).** No single client in scope, so the head fans out:

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1 | `Schedule` | Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) | Cron expression per workflow (each workflow section states it). |
| 2 | `List Enabled Clients` | HTTP Request | GET `{{ $env.SEEKLY_API_URL }}/api/internal/clients?module=<MODULE>&enabled=true` · auth header as always. Returns `[{ "client_id": "...", "business_name": "...", "status": "active" }, …]`. (SPEC-02 addition, §2.7.) |
| 3 | `Split Clients` | Split Out (`n8n-nodes-base.splitOut`) | Field to split out: `clients` (or root array) → one item per client. |
| 4 | `Loop Clients` | Split In Batches (`n8n-nodes-base.splitInBatches`) | Batch Size `1` — serialize per client so one client's vendor failure can't poison another's run. |
| 5 | `Get Context` | HTTP Request | Same as Shape A node 5, `client_id` from `{{ $json.client_id }}`. |
| 6 | `Module Enabled?` | IF | Same as Shape A node 6; false → `Log Skipped` → back to `Loop Clients`. |

Expression conventions used throughout this spec:

- Current item field: `{{ $json.client_id }}`
- Named node output: `{{ $('Get Context').item.json.profile.brand_voice.tone }}`
- Env: `{{ $env.SEEKLY_API_URL }}`
- Guarded JSON embedding in HTTP bodies: `{{ JSON.stringify($json.payload) }}`

### 2.5 Idempotency rules (doc 03/04, restated as build rules)

1. **Events:** the control plane dedupes on `event_id` before forwarding; a replayed event never reaches a workflow twice *unless* an operator manually re-delivers. Workflows must still be no-ops on replay because of rules 2–5.
2. **Sends:** every `POST /api/internal/send` carries `idempotency_key` = `<event_id>:<step>` (e.g. `9f8f…:review_request`, `9f8f…:first_touch`, `<campaign_id>:<customer_id>`). The ledger rejects duplicates → response `{"status":"duplicate"}`; treat as success.
3. **Reviews:** `review.received` events use `event_id = "gbrev:" + provider_review_id`; the reviews table PK is the provider review id → upserts.
4. **Campaigns:** `campaign_members` is the resume ledger. A crashed batch re-run re-reads `status=queued` members only; already-`sent` members are never re-selected.
5. **Provisioning:** saga keyed on `hubspot_deal_id`; every `/api/internal/provision/*` endpoint is an upsert that returns `{"created": false}` when the step already ran.
6. **Content:** one `content_item` per client per calendar slot — `POST /api/internal/content/claim-topic` is idempotent on `(client_id, period)`.

### 2.6 Context endpoint response (single source of truth for expressions)

`GET /api/internal/clients/:id/context?module=<module>` returns (SPEC-02 owns the implementation):

```json
{
  "client":   { "id": "…", "business_name": "…", "timezone": "America/Chicago", "status": "active" },
  "profile":  {
    "services": ["…"], "service_areas": ["…"], "target_keywords": ["…"],
    "competitors": [{ "name": "…", "website": "…", "place_id": "…", "fb_page": "…" }],
    "brand_voice": { "tone": "…", "persona_name": "…", "phrases_use": ["…"], "phrases_avoid": ["…"], "example_sentences": ["…"] },
    "qualification_questions": [{ "id": "party_size", "question": "…", "hot_signal": "…" }],
    "campaign_angles": [{ "id": "league_night", "name": "…", "offer": "…", "constraints": "…" }],
    "booking_link": "https://…", "website_url": "…", "website_platform": "wordpress",
    "review_link": "https://g.page/r/…", "approval_preferences": { "review_responses": "auto", "campaigns": "approve_first", "content": "approve_first_3" },
    "ai_never_promise": ["specific prices", "availability", "refunds"]
  },
  "module":   { "name": "review_automation", "enabled": true, "settings": { /* per-module, defaults from doc 03 */ } },
  "locations": [{ "id": "…", "gbp_location_id": "locations/123", "gbp_account_id": "accounts/456", "place_id": "ChIJ…", "brightlocal_location_id": null, "hours_json": {}, "address_json": {}, "phone": "…" }],
  "connections": {
    "google_business": { "status": "active", "access_token": "ya29.SHORT-LIVED", "expires_in": 3500 },
    "meta":            { "status": "active", "page_id": "1234", "page_access_token": "EAAG.SHORT-LIVED" },
    "wordpress":       { "status": "active", "base_url": "https://client.com", "username": "seekly", "app_password": "xxxx xxxx" }
  },
  "comms":    { "phone_number": "+1555…", "a2p_campaign_status": "approved", "inbound_email_address": "clubx@in.seekly.app" }
}
```

Tokens are minted per request and expire in ≤ 60 minutes. Long-running executions (WF-2A waits) must re-fetch context after every Wait node rather than reusing stale tokens or stale switch states.

### 2.7 Internal API surface used by n8n

Core four (already contracted in docs 02/03/10) plus additions this spec requires. **Every addition below is a SPEC-02 deliverable; the ticket table (§5) sequences them.**

| Endpoint | Used by | Purpose |
|---|---|---|
| `POST /api/internal/events` | WF-1B, WF-4 | Emit canonical events (`review.received`, `content.published`); control plane dedupes + routes back to webhooks. |
| `GET /api/internal/clients/:id/context?module=` | all | Per-run config, toggles, short-lived tokens. |
| `POST /api/internal/activity` | all | Every action logged (`ok`/`failed`/`skipped`). |
| `POST /api/internal/send` | WF-1A/1C, WF-2A/2B, WF-3 | The shared send-pipeline. Body: `{client_id, customer:{name?,phone}, kind: "transactional"|"conversational"|"marketing", engine, body, idempotency_key, conversation_id?, campaign_id?}` → `{status:"sent"|"queued"|"blocked"|"duplicate", blocked_reason?, message_id}`. |
| `GET /api/internal/clients?module=&enabled=true` *(addition)* | Shape-B heads | Enumerate clients with a module on (includes `status=prospect` rows for WF-6 prospect mode). |
| `POST /api/internal/eligibility` *(addition)* | WF-1A, WF-2A | `{client_id, phone, check: "review_request"|"lead_dedupe", params:{cooldown_days}}` → `{eligible, reasons[], customer_id}`. Centralizes engine-level dedupe (re-ask cooldown, already-reviewed, 24h lead window). |
| `GET/POST/PATCH /api/internal/conversations…` *(addition)* | WF-2A/2B | `POST /conversations` (upsert on `lead_event_id`), `GET /conversations/lookup?client_id=&phone=`, `GET /conversations/:id` (+`/messages` transcript), `PATCH /conversations/:id` (status, outcome, context). |
| `GET/PATCH /api/internal/campaigns…` *(addition)* | WF-3 | `GET /campaigns/:id`, `GET /campaigns/:id/members?status=queued&limit=`, `PATCH /campaigns/:id` (status, copy_variants, stats), `PATCH /campaigns/:id/members/:customer_id` (status). `GET /campaigns/:id/stats?window=24h` returns `{sent, stops, stop_rate}`. |
| `PATCH /api/internal/reviews/:id` *(addition)* | WF-1C | Store draft/published response, response_status transitions. |
| `POST /api/internal/content/claim-topic`, `PATCH /api/internal/content/:id`, `GET /api/internal/content/:id` *(additions)* | WF-4, WF-7 | Topic queue pop (idempotent per period), content item lifecycle. |
| `POST /api/internal/competitor-snapshots`, `GET /api/internal/clients/:id/competitor-snapshots?since=` *(additions)* | WF-6A/6B | Snapshot store/read. |
| `POST /api/internal/intel-digests` *(addition)* | WF-6B | Digest storage (`body_html`, `findings` jsonb → later promoted to L2 notes by the in-app intelligence jobs, doc 09). |
| `PATCH /api/internal/locations/:id` *(addition)* | WF-5 | Persist `brightlocal_location_id`, `listings_health_score`, per-directory statuses. |
| `POST /api/internal/provision/client|portal-user|onboarding|comms|hubspot-sync` *(additions)* | WF-0 | Saga steps executed by the control plane (it owns DB writes, Twilio subaccount creation, HubSpot API). Each is idempotent on `hubspot_deal_id`; each returns `{done: true, created: bool, …ids}`. |

### 2.8 Anthropic API usage from n8n

- **Interactive agent (WF-2B only):** n8n `AI Agent` node (`@n8n/n8n-nodes-langchain.agent`) with an **Anthropic Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`), credential `Anthropic Seekly` (API key). Model: `claude-sonnet-5`.
- **One-shot generations (WF-1C, WF-3 composer, WF-4 draft, WF-6B analyst, WF-7 cutter):** plain HTTP Request node to `https://api.anthropic.com/v1/messages`, headers `x-api-key: {{ $env.ANTHROPIC_API_KEY }}`, `anthropic-version: 2023-06-01`, `content-type: application/json`. Use **structured outputs** so downstream nodes never parse prose: body includes `"output_config": {"format": {"type": "json_schema", "schema": { …per-workflow schema in §4… }}}`. Models: `claude-sonnet-5` for generation, `claude-haiku-4-5` for extraction/classification (doc 07 stack choice). Set `max_tokens` per call site (stated in each workflow).
- Every AI node has a deterministic fallback branch (§2.3). Every AI output that leaves Seekly (SMS, review reply, post, blog) passes a **deterministic guard Code node** (banned-content regexes) before send/publish — belt and braces on top of prompt guardrails (§4).

---

## 3. Workflow-by-workflow build instructions

Every table lists nodes in wiring order; branch nodes state where each output goes. "Head A/B nodes 1–6" refers to §2.4 — build those first, verbatim, with the module named in the row.

---

### 3.1 WF-0 Provisioning — "the flick of the switch"

**Trigger:** Webhook, path `client-provisioned`, event `client.provisioned`. The control plane's public HubSpot webhook route receives the `Closed Won` deal-stage webhook, normalizes it to a canonical `client.provisioned` event (`client_id` may be `null` — the tenant doesn't exist yet; `payload.hubspot_deal_id` is the idempotency anchor), and forwards it here. The admin "intake runner" (doc 05) posts the same event manually for pilots that skipped HubSpot.
**Head:** Shape A nodes 1–4 only (no context/gate — there is no client yet).
**Design:** n8n orchestrates a **saga**; the control plane executes each step (DB writes, Twilio subaccount, HubSpot writes stay server-side). Re-runs skip completed steps because every step endpoint is idempotent on `hubspot_deal_id`.

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–4 | *(Head A: `Webhook` path `client-provisioned`, `Verify Signature`, `Signature OK?`, `Respond 200`)* | | |
| 5 | `Extract Deal` | Code | Map envelope → `{hubspot_deal_id, company, contact_name, contact_email, tier, retainer, setup_fee, locations[], intake: payload.intake ?? null}` from `{{ $json.payload }}`. Throw if `hubspot_deal_id` and `company` are missing (manual events must fake a deal id like `manual:<slug>`). |
| 6 | `Step 1 — Create Tenant` | HTTP Request | POST `{{ $env.SEEKLY_API_URL }}/api/internal/provision/client` body `{"hubspot_deal_id":"{{ $json.hubspot_deal_id }}","business_name":"{{ $json.company }}","tier":"{{ $json.tier }}","retainer":{{ $json.retainer }},"locations":{{ JSON.stringify($json.locations) }},"intake":{{ JSON.stringify($json.intake) }}}` → returns `{client_id, created}`. Creates tenant row + `client_profile` skeleton + `workflow_config` defaults for the tier with **all switches OFF** (doc 03 step 5). |
| 7 | `Step 2 — Portal User` | HTTP Request | POST `…/api/internal/provision/portal-user` body `{"client_id":"{{ $json.client_id }}","email":"{{ $('Extract Deal').item.json.contact_email }}","name":"{{ $('Extract Deal').item.json.contact_name }}"}` → control plane creates the user + sends the magic-link welcome email. |
| 8 | `Step 3 — Onboarding Checklist` | HTTP Request | POST `…/api/internal/provision/onboarding` body `{"client_id":"{{ $json.client_id }}","tier":"{{ $('Extract Deal').item.json.tier }}"}` → checklist derived from tier + intake answers (doc 08 §E). Response echoes `checklist[]` for the ops email. |
| 9 | `Step 4 — Comms Provisioning` | HTTP Request | POST `…/api/internal/provision/comms` body `{"client_id":"{{ $json.client_id }}"}` → control plane creates Twilio subaccount, buys local number, submits A2P brand+campaign (its own Twilio creds — n8n has none), writes `comms_provisioning`. Returns `{phone_number, a2p_brand_status, a2p_campaign_status}`. **On Error: Continue** — A2P submission failure must not abort the saga; route error output to node 13 with `step:"comms"`. |
| 10 | `Step 5 — HubSpot Sync-back` | HTTP Request | POST `…/api/internal/provision/hubspot-sync` body `{"client_id":"{{ $json.client_id }}","fields":{"onboarding_status":"Provisioned","account_status":"Onboarding"}}` — summary fields only, never operational data (doc 01). Skips silently for `manual:` deals. |
| 11 | `Log Provisioned` | HTTP Request | POST `…/api/internal/activity` body `{"client_id":"{{ $json.client_id }}","engine":"provisioning","action":"client.provisioned","status":"ok","entity_type":"client","entity_id":"{{ $json.client_id }}","detail":{"deal":"{{ $('Extract Deal').item.json.hubspot_deal_id }}","tier":"{{ $('Extract Deal').item.json.tier }}"}}` |
| 12 | `Notify Ops` | Send Email | To `{{ $env.OPS_ALERT_EMAIL }}` · Subject `New client provisioned: {{ $('Extract Deal').item.json.company }}` · Body: client_id, tier, phone number, A2P status, checklist items from node 8, resume link `{{ $env.SEEKLY_API_URL }}/admin/clients/{{ $json.client_id }}`. |
| 13 | `Park Failed Step` | HTTP Request | (Error lane, fed by any step's error output.) POST `…/api/internal/activity` `status:"failed"`, `action:"provision.step_failed"`, `detail:{step, error}` → then `Ops Alert (resume link)` Send Email: "Provisioning halted at step X for <company> — fix and re-POST the client.provisioned event; completed steps will be skipped." |

**Error branch:** steps 6–8 and 10 stop on final failure → WF-E (plus node 13 lane where marked). Because every step is idempotent, ops recovery = re-deliver the event from the control-plane dead-letter UI.
**Idempotency:** whole saga keyed on `hubspot_deal_id` (§2.5.5); replaying the event N times yields exactly one tenant, one user, one subaccount, one A2P submission.

**Acceptance tests**

1. POST a valid `client.provisioned` event → tenant, portal user, onboarding record, `workflow_config` rows (all `enabled=false`), Twilio subaccount + number, A2P submission, ops email — all present exactly once.
2. Replay the same event (same `hubspot_deal_id`) 3× → zero new rows anywhere; each step returns `created:false`; activity shows one `client.provisioned` and N `…skipped`/idempotent entries, no duplicates of side effects.
3. Kill the control plane between steps 2 and 3, re-deliver → steps 1–2 skipped, 3–5 complete; no duplicate magic-link email.
4. `provision/comms` returns 500 (Twilio down) → saga continues, ops gets "halted at step comms" email with resume link, activity row `status=failed` written.
5. Manual event with `hubspot_deal_id: "manual:golfclub"` and no HubSpot fields → provisions fully, HubSpot sync-back step is a no-op, no error.
6. Bad signature on the webhook → 401 response, nothing executed, nothing logged as failed.
7. Event missing `company` → execution fails at `Extract Deal` → WF-E fires: activity `workflow.error` + ops email with execution URL.
8. Tier `core` vs `growth` → onboarding checklists differ per tier mapping; verify checklist content in ops email.

---

### 3.2 WF-1 Review Velocity

**Module:** `review_automation`. Settings (defaults, doc 03): `send_delay_hours: 2`, `reask_cooldown_days: 90`, `daily_send_cap: 25`, `request_template`, `negative_threshold: 3`, `response_mode: "auto" | "approve_all" | "approve_negative_only"`, `gbp_mode: "api" | "places_interim"`, sub-switches `review_requests: true`, `review_responses: true`.

#### WF-1A Review Velocity — Requests

**Trigger:** Webhook path `sale-completed`, event `sale.completed` (payload always has `customer: {name, phone?, email?}`, `amount?`, `items?` — normalized by adapters).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head A, module `review_automation`; node 6 condition: `{{ $('Get Context').item.json.module.enabled && $('Get Context').item.json.module.settings.review_requests !== false }}`)* | | |
| 7 | `Has Phone?` | IF | String → `{{ $('Verify Signature').item.json.payload.customer.phone }}` **is not empty**. False → `Log Skipped (no_phone)` (activity `status=skipped`, `detail.reason:"no_phone"`) → NoOp. |
| 8 | `Check Eligibility` | HTTP Request | POST `{{ $env.SEEKLY_API_URL }}/api/internal/eligibility` body `{"client_id":"{{ $('Verify Signature').item.json.client_id }}","phone":"{{ $('Verify Signature').item.json.payload.customer.phone }}","check":"review_request","params":{"cooldown_days":{{ $('Get Context').item.json.module.settings.reask_cooldown_days ?? 90 }}}}` → `{eligible, reasons[], customer_id}`. Control plane checks: opt-out, asked within cooldown, already reviewed, consent basis valid (doc 03 gates 1–2). |
| 9 | `Eligible?` | IF | `{{ $json.eligible }}` is true. False → `Log Skipped (ineligible)` activity with `detail.reasons` → NoOp. |
| 10 | `Wait Send Delay` | Wait (`n8n-nodes-base.wait`) | Resume: `After Time Interval` · Wait Amount: `{{ $('Get Context').item.json.module.settings.send_delay_hours ?? 2 }}` · Unit: Hours. n8n persists the execution (survives restarts). |
| 11 | `Re-fetch Context` | HTTP Request | Same as head node 5. **Mandatory after any Wait** — the switch may have been flipped mid-delay (safe-off, doc 03) and tokens have expired. |
| 12 | `Still Enabled?` | IF | `{{ $json.module.enabled && $json.module.settings.review_requests !== false }}`. False → `Log Skipped (disabled_mid_flight)` → NoOp. |
| 13 | `Compose SMS` | Code | Deterministic template merge (no LLM — approved copy): ```const ctx=$('Re-fetch Context').item.json; const ev=$('Verify Signature').item.json; const first=(ev.payload.customer.name||'').split(' ')[0]; const tpl=ctx.module.settings.request_template ?? "Hi {first}, thanks for visiting {business}! Would you mind sharing a quick Google review? It really helps us: {link}"; const body=tpl.replace('{first}', first||'there').replace('{business}', ctx.client.business_name).replace('{link}', ctx.profile.review_link); return [{json:{body}}];``` The `{link}` value is the short-link redirect so clicks are tracked (doc 03 A.5). |
| 14 | `Send Review Request` | HTTP Request | POST `…/api/internal/send` body `{"client_id":"{{ $('Verify Signature').item.json.client_id }}","customer":{"name":"{{ $('Verify Signature').item.json.payload.customer.name }}","phone":"{{ $('Verify Signature').item.json.payload.customer.phone }}"},"kind":"transactional","engine":"wf-1","body":{{ JSON.stringify($json.body) }},"idempotency_key":"{{ $('Verify Signature').item.json.event_id }}:review_request"}`. Pipeline enforces opt-out/quiet-hours/A2P/daily cap (`daily_send_cap` configured in module settings is enforced server-side per SPEC-03). |
| 15 | `Sent?` | Switch | Rules on `{{ $json.status }}`: `sent`/`queued` → node 16 · `blocked` → node 16b · `duplicate` → node 16c. |
| 16 | `Log Sent` | HTTP Request | POST activity `{"action":"review_request.sent","status":"ok","entity_type":"message","entity_id":"{{ $json.message_id }}", "detail":{"event_id":"…","queued": {{ $json.status === 'queued' }}}}`. |
| 16b | `Log Blocked` | HTTP Request | POST activity `action:"review_request.blocked"`, `status:"skipped"`, `detail:{blocked_reason: $json.blocked_reason}`. Not an error — pipeline did its job. |
| 16c | `Log Duplicate` | HTTP Request | POST activity `action:"review_request.duplicate"`, `status:"skipped"`. |

#### WF-1B Review Velocity — Review Poller

**Trigger:** Schedule, cron `*/15 * * * *` (doc 03: poll every 15 min per connected location). Emits `review.received` events; it never messages anyone.

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head B, module `review_automation`. Note: detection continues even when `review_requests`/`review_responses` are off — node 6 checks only `module.enabled`; per doc 03 switchboard, requests stop but detection keeps data fresh. If `module.enabled` is false → skip client.)* | | |
| 7 | `Split Locations` | Split Out | Field: `locations` from `{{ $('Get Context').item.json }}` (merge `client_id` into each item in a preceding `Code` node if needed: one item per location with `{client_id, location}`) |
| 8 | `GBP Mode?` | IF | `{{ $('Get Context').item.json.module.settings.gbp_mode === 'api' && $('Get Context').item.json.connections.google_business?.status === 'active' }}` → true: node 9a; false: node 9b (**interim Places mode** — pre-GBP-API-approval path, doc 06). |
| 9a | `Fetch GBP Reviews` | HTTP Request | GET `https://mybusiness.googleapis.com/v4/{{ $json.location.gbp_account_id }}/{{ $json.location.gbp_location_id }}/reviews?pageSize=50&orderBy=updateTime desc` · Header `Authorization: Bearer {{ $('Get Context').item.json.connections.google_business.access_token }}` · Retry §2.3 vendor row; **On Error: Continue** → node 13 (connection-broken lane). |
| 9b | `Fetch Places Reviews` | HTTP Request | GET `https://maps.googleapis.com/maps/api/place/details/json?place_id={{ $json.location.place_id }}&fields=rating,user_ratings_total,reviews&key={{ $env.GOOGLE_MAPS_API_KEY }}` — Places returns only ~5 most-recent reviews; good enough to *detect and draft* (doc 06 interim). No client OAuth involved. |
| 10 | `Normalize Reviews` | Code | Map both shapes to `[{review_id, rating (1–5 int), author, body, received_at}]`. GBP: `reviewId`, `starRating` enum→int, `comment`, `updateTime`. Places: synthesize `review_id = sha1(place_id + author_name + time)` (Places exposes no stable id — flag `source:"places"`), `rating`, `text`, ISO from `time`. |
| 11 | `Split Reviews` | Split Out | One item per review. |
| 12 | `Emit review.received` | HTTP Request | POST `…/api/internal/events` body `{"event_id":"gbrev:{{ $json.review_id }}","client_id":"{{ $('Loop Clients').item.json.client_id }}","type":"review.received","occurred_at":"{{ $json.received_at }}","source":"{{ $json.source ?? 'gbp-poll' }}","payload":{{ JSON.stringify($json) }}}`. Control plane dedupes on `event_id` → **duplicate polls are no-ops**; no poll cursor needed in n8n (cursor lives implicitly in the events table — doc 03 "poll cursor persisted so no review is missed across downtime"). Response `{status:"duplicate"}` is success. Persisting the review row (upsert on provider id) happens in the control plane's `review.received` event processor (SPEC-02). |
| 13 | `Connection Broken Lane` | HTTP Request | (Fed by 9a error output.) POST activity `action:"review_poll.failed"`, `status:"failed"`, `detail:{http: error}`. If HTTP 401/403: also POST `…/api/internal/activity` `action:"connection.broken", detail:{provider:"google_business"}` — the control plane's connection-health monitor flips `connection_status=broken`, auto-pauses the module with a portal banner ("Reconnect Google", doc 03). Then continue the client loop — never abort other clients. |

#### WF-1C Review Velocity — Responder

**Trigger:** Webhook path `review-received`, event `review.received` (forwarded by the control plane after persisting the review row).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head A, module `review_automation`; node 6 condition adds `&& $('Get Context').item.json.module.settings.review_responses !== false`)* | | |
| 7 | `Draft Response (Claude)` | HTTP Request | POST `https://api.anthropic.com/v1/messages` · headers §2.8 · body: `model: "claude-sonnet-5"`, `max_tokens: 1000`, `system`: **Prompt 4.2** (placeholders filled by expressions from context + event), `messages: [{role:"user", content: "REVIEW ({{ $('Verify Signature').item.json.payload.rating }} stars) by {{ $('Verify Signature').item.json.payload.author }}: {{ $('Verify Signature').item.json.payload.body }}" }]`, `output_config` json_schema from §4.2. Retry 2×; **On Error: Continue** → node 7f. |
| 7f | `AI Fallback` | Code | Emit `{draft: null, needs_human: true}` → merges into node 8 path; a null draft forces the approval queue (never auto-publish a non-AI template for reviews — a human writes it). |
| 8 | `Save Draft` | HTTP Request | PATCH `…/api/internal/reviews/{{ $('Verify Signature').item.json.payload.review_id }}` body `{"client_id":"…","response_body":{{ JSON.stringify($json.reply_text ?? null) }},"response_status":"drafted"}`. |
| 9 | `Negative?` | IF | `{{ $('Verify Signature').item.json.payload.rating <= ($('Get Context').item.json.module.settings.negative_threshold ?? 3) }}` → true: node 10 (escalation lane runs **in addition to** whatever the approval mode does — doc 03 B.4 "Always, for ≤3★"). Both outputs continue to node 11. |
| 10 | `Escalate Negative` | HTTP Request ×2 | (a) POST `…/api/internal/send` `{"kind":"transactional","engine":"wf-1","customer":{"phone":"<owner mobile from profile.escalation_contacts>"},"body":"⚠ {{ rating }}★ review from {{ author }} at {{ business }}: \"{{ first 120 chars }}\" — suggested next step: {{ $('Draft Response (Claude)').item.json.service_recovery_suggestion }}","idempotency_key":"{{ event_id }}:neg_alert"}`; (b) Send Email to owner + ops with full text + draft. |
| 11 | `Approval Route` | Switch | Value: `{{ $('Get Context').item.json.profile.approval_preferences.review_responses ?? $('Get Context').item.json.module.settings.response_mode ?? 'auto' }}` with the negative override: Code expression → output `queue` if mode is `approve_all`, or (`approve_negative_only` and rating ≤ threshold), or draft is null; output `auto` otherwise. |
| 12q | `Queue for Approval` | HTTP Request | PATCH review `response_status:"pending_approval"` → portal Approvals queue (SPEC-05); one-tap approve there calls the control plane which publishes (or hands ops the manual task in `places_interim` mode). POST activity `action:"review_response.queued"`. Branch ends. |
| 12a | `Publish Mode?` | IF | (auto lane) `{{ $('Get Context').item.json.module.settings.gbp_mode === 'api' }}` false → `Queue Manual Publish`: PATCH review `response_status:"pending_manual"` + activity `review_response.queued_manual` (ops publishes via GBP manager access — doc 06 interim). |
| 13 | `Publish Reply (GBP)` | HTTP Request | PUT `https://mybusiness.googleapis.com/v4/{{ gbp_account_id }}/{{ gbp_location_id }}/reviews/{{ $('Verify Signature').item.json.payload.review_id }}/reply` · Bearer context token · body `{"comment": {{ JSON.stringify($('Draft Response (Claude)').item.json.reply_text) }} }` · Retry 3×; On Error: Continue → `Park Publish Failure`: PATCH review `response_status:"failed"` + activity `status:"failed"` + ops email. 401/403 → connection-broken lane as WF-1B node 13. |
| 14 | `Mark Published + Log` | HTTP Request ×2 | PATCH review `{"response_status":"published","responded_at":"{{ $now.toISO() }}"}` · POST activity `action:"review_response.published","status":"ok","entity_type":"review","entity_id":"<review_id>"`. |

**Idempotency (WF-1):** sends keyed `event_id:review_request` (§2.5.2); reviews keyed on provider review id; replayed `review.received` events hit `response_status != 'none'` — add an IF after head: `{{ $('Verify Signature').item.json.payload.response_status ?? 'none' }} == 'none'`, else skip-log (the event payload carries current `response_status`, SPEC-02).

**Acceptance tests (WF-1)**

1. Replay the same `sale.completed` `event_id` twice → exactly one SMS row in the message ledger (second send returns `duplicate`).
2. Sale event for opted-out phone → `blocked` with `blocked_reason:"opt_out"`, activity `review_request.blocked`, zero Twilio traffic.
3. Sale at 22:30 client-local with `kind:"transactional"`? → verify against SPEC-03 policy: review requests are quiet-hours-gated (queued until 08:00), assert `status:"queued"` and morning delivery.
4. Same customer sale twice in a week → second run ineligible (`reasons:["cooldown"]`), no SMS.
5. Flip `review_requests` off while an execution sits in `Wait Send Delay` → after resume, `Still Enabled?` routes to skip; no SMS (doc 07 "safe-off strands nothing").
6. WF-1B double-runs in the same 15-min window → `POST /events` returns `duplicate` for every review; zero `review.received` webhooks re-fired.
7. `gbp_mode: places_interim` → reviews detected from Places, drafts created, `response_status:"pending_manual"`, nothing calls `mybusiness.googleapis.com`.
8. 1★ review in `auto` mode → response queued (never auto-published if `approve_negative_only`… in plain `auto` it publishes) **and** owner alert SMS + email fired regardless of mode.
9. GBP publish 500 ×3 → review `failed`, ops email, WF-E not fired (handled branch), other reviews in batch unaffected.
10. Revoke Google connection then poll → activity `connection.broken`, module auto-paused by control plane, poller logs and moves to next client.

---

### 3.3 WF-2 Speed-to-Lead

**Module:** `speed_to_lead`. Settings (defaults): `followup_schedule_hours: [2, 24, 72, 120, 168]` (day 0 +2h, d1, d3, d5, d7), `handoff_keywords: ["price match","discount","lawyer","complaint","refund","manager"]`, `low_confidence_handoff_after: 2`, `hot_alert_recipients` (from `profile.escalation_contacts`), `after_hours_note`, `persona_name` (falls back to `brand_voice.persona_name`).

#### WF-2A Speed-to-Lead — First Touch & Follow-ups

**Trigger:** Webhook path `lead-created`, event `lead.created` (`payload`: `{customer:{name?,phone,email?}, source, message?, form_fields?}` — normalized by email-parse/Meta adapters in the control plane).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head A, module `speed_to_lead`)* | | |
| 7 | `Lead Dedupe` | HTTP Request | POST `…/api/internal/eligibility` `{"check":"lead_dedupe","phone":"…","params":{"window_hours":24}}` → false when the same phone opened a lead in 24h (doc 03 gate 1). Ineligible → `Log Skipped (dup_lead)` → NoOp. |
| 8 | `Create Conversation` | HTTP Request | POST `…/api/internal/conversations` body `{"client_id":"…","lead_event_id":"{{ event_id }}","engine":"wf-2","lead_source":"{{ payload.source }}","customer":{"name":"…","phone":"…"},"context":{{ JSON.stringify($('Verify Signature').item.json.payload) }}}` → `{conversation_id, created}` (upsert on `lead_event_id`). |
| 9 | `First Touch (Claude)` | HTTP Request | Anthropic messages call (§2.8): model `claude-sonnet-5`, `max_tokens: 400`, system **Prompt 4.1** (with `MODE: first_touch`), user content = JSON of the lead payload, `output_config` schema §4.1. Retry 2×, **On Error: Continue** → 9f. Timeout (node Settings → Timeout): 15000 ms — the <60s promise beats eloquence. |
| 9f | `Template Fallback` | Code | `return [{json:{reply_text: \`Hi\${first? ' '+first:''}! Thanks for reaching out to \${business} — what date were you thinking?\`, status:"continue", confidence: 1, extracted:{}}}]` (doc 03: fallback so the <60s promise never breaks; AI resumes next turn). |
| 10 | `Send First Touch` | HTTP Request | POST `…/api/internal/send` `{"kind":"conversational","engine":"wf-2","conversation_id":"…","body":{{ JSON.stringify($json.reply_text) }},"idempotency_key":"{{ event_id }}:first_touch"}`. Conversational kind = exempt from quiet-hours *initiation* for immediate replies to an inbound lead (SPEC-03 rule; overnight leads still get instant reply — doc 03). |
| 11 | `Log First Response` | HTTP Request ×2 | PATCH conversation `{"first_response_seconds": {{ Math.round((Date.now() - new Date($('Verify Signature').item.json.occurred_at).getTime())/1000) }} }` · POST activity `action:"lead.first_touch","detail":{"seconds":…,"source":…}` — the case-study hero metric (doc 05). |
| 12 | `FOLLOW-UP LOOP` | — | Nodes 13–17 repeat 5×, one block per entry in `followup_schedule_hours`. Build as five sequential Wait blocks (explicit > clever; the array is fixed-length by config contract). Block *k*: |
| 13k | `Wait FU-k` | Wait | After Time Interval · Amount `{{ $('Get Context').item.json.module.settings.followup_schedule_hours[k] - ($('Get Context').item.json.module.settings.followup_schedule_hours[k-1] ?? 0) }}` hours (delta from previous block). Persisted execution. |
| 14k | `Refresh Context+Convo` | HTTP Request ×2 | Re-GET context; GET `…/api/internal/conversations/{{ conversation_id }}` → `{status, last_inbound_at, messages_out_count}`. |
| 15k | `Still Cold?` | IF | `{{ $json.status === 'active' && !$json.last_inbound_at && $('Refresh Context+Convo').item.json.module.enabled }}` — any inbound reply, status change, or switch-off ends the sequence → `NoOp (sequence ended)`. |
| 16k | `Compose FU-k` | Code | Pick angle *k* from a fixed varied-angle template table (`["quick nudge","value/social proof","question about their need","booking link direct","last check-in"]`) merged with business name, persona, booking link. Deterministic (approved copy, no per-touch LLM spend). |
| 17k | `Send FU-k` | HTTP Request | POST `/send` `kind:"conversational"`, `idempotency_key:"{{ event_id }}:fu{{ k }}"` → activity `lead.followup_sent` detail `{n: k}`. |
| 18 | `Mark Cold` | HTTP Request | After block 5: PATCH conversation `{"status":"cold","outcome":"no_response"}` + activity `lead.cold`. |

#### WF-2B Speed-to-Lead — Conversation Loop

**Trigger:** Webhook path `message-received`, event `message.received` (`payload: {phone, body, twilio_sid}`). This is THE conversation router — WF-3 replies land here too.

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–4 | *(Head A nodes 1–4; context fetch happens after routing — module depends on the conversation's engine)* | | |
| 5 | `Lookup Conversation` | HTTP Request | GET `…/api/internal/conversations/lookup?client_id={{ client_id }}&phone={{ payload.phone }}&status=active,qualified` → `{found, conversation:{id, engine, status, context}}`. (STOP/HELP never reach here — handled at pipeline level before the event is emitted, doc 03.) |
| 6 | `Found?` | IF | `{{ $json.found }}`. False → `Log Orphan Inbound` activity `action:"message.unrouted"` (portal shows it in the client inbox; owner handles manually) → NoOp. |
| 7 | `Get Context` | HTTP Request | GET context with `module={{ $json.conversation.engine === 'wf-3' ? 'reactivation' : 'speed_to_lead' }}`. |
| 8 | `Module Enabled?` | IF | Standard gate. Disabled → activity skipped; do **not** reply (safe-off: "leads logged but not messaged; alert owner instead" → also send owner email in this lane). |
| 9 | `Load Transcript` | HTTP Request | GET `…/api/internal/conversations/{{ conversation.id }}/messages?limit=40` → `[{direction, body, created_at}]` (the message ledger is the memory — no n8n memory node). |
| 10 | `Handoff Keyword?` | Code | Deterministic pre-AI guard: lowercase inbound body, test against `module.settings.handoff_keywords`; output `{forced_handoff: bool}`. |
| 11 | `AI Agent` | AI Agent (`@n8n/n8n-nodes-langchain.agent`) | **Chat Model:** Anthropic Chat Model sub-node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`), credential `Anthropic Seekly`, model `claude-sonnet-5`, Max Tokens 600. **System Message:** Prompt 4.1 (MODE: conversation) via expression. **Prompt (User Message):** ```TRANSCRIPT:\n{{ $('Load Transcript').item.json.messages.map(m => (m.direction==='in'?'CUSTOMER: ':'YOU: ')+m.body).join('\n') }}\n\nNEW CUSTOMER MESSAGE: {{ $('Verify Signature').item.json.payload.body }}``` **Output Parser:** Structured Output Parser sub-node (`@n8n/n8n-nodes-langchain.outputParserStructured`) with schema §4.1. No tools attached. **On Error: Continue** → 11f. |
| 11f | `Fallback Reply` | Code | `{reply_text:"Got it — let me check on that and get right back to you.", status:"needs_human", confidence:0}` → forces handoff lane so a human follows up (doc 03: AI failure → template; conversation resumes with AI next turn). |
| 12 | `Update Extracted` | HTTP Request | PATCH conversation `{"context": {{ JSON.stringify($json.extracted) }}, "low_confidence_streak": {{ $json.confidence < 0.5 ? ($('Lookup Conversation').item.json.conversation.low_confidence_streak ?? 0) + 1 : 0 }} }`. |
| 13 | `Route Outcome` | Switch | Value (Code-computed in node 12 output `route`): `handoff` if `forced_handoff` or AI `status==='needs_human'` or streak ≥ `low_confidence_handoff_after`; `qualified` if AI `status==='qualified'`; else `continue`. |
| 14a | `Reply (continue)` | HTTP Request | POST `/send` `kind:"conversational"`, body = `reply_text`, `idempotency_key:"{{ event_id }}:reply"` → activity `conversation.reply`. |
| 14b | `Qualified Lane` | HTTP Request ×3 | (1) POST `/send` reply including booking link (`{{ profile.booking_link }}` — the AI was instructed to include it; guard: Code appends if missing). (2) Hot-lead alert: POST `/send` to each `escalation_contacts.mobile` `kind:"transactional"` `"🔥 Hot lead: {name} {phone} — {ai summary}"` `idempotency_key:"{{ event_id }}:hot_alert:<i>"` + Send Email with transcript summary. (3) PATCH conversation `{"status":"qualified","outcome":"qualified"}` + activity `lead.qualified`. |
| 14c | `Handoff Lane` | HTTP Request ×3 | (1) POST `/send` `"Thanks! The owner will text you directly in just a bit."` `idempotency_key:"{{ event_id }}:handoff_msg"`. (2) Owner alert SMS+email with transcript. (3) PATCH conversation `{"status":"handed_off"}` + activity `lead.handed_off`. |

**Idempotency (WF-2):** conversation upsert on `lead_event_id`; every outbound keyed `event_id:<step>`; follow-up steps keyed `event_id:fuK` so a crashed/restored execution never re-texts; replayed `message.received` produces the same `event_id:reply` key → ledger rejects the duplicate send even though the AI would have produced new text.

**Acceptance tests (WF-2)**

1. `lead.created` → first SMS visible in ledger with `first_response_seconds < 60` in the conversation row (test with adapters' real latency budget).
2. Replay `lead.created` (same `event_id`) → one conversation, one first-touch SMS.
3. Same phone submits two different forms within 24h → second is `dup_lead`-skipped; first conversation untouched.
4. Anthropic returns 529 twice → template fallback sent within timeout; next inbound message gets a normal AI reply.
5. No reply for 7 days → exactly 5 follow-ups at +2h/d1/d3/d5/d7 (ledger timestamps), then conversation `cold`. Customer replies after FU-2 → FU-3..5 never sent.
6. Switch off mid-sequence → pending Waits resume, `Still Cold?` gate stops sends; zero messages after flip timestamp.
7. Inbound "what would the price be if I bring 10 people" (handoff keyword "price") → forced handoff: customer gets handoff message, owner alerted, status `handed_off`; AI reply (if any) discarded.
8. Qualification completes (all questions answered, hot signal) → booking link present in final SMS, hot-lead alert to every escalation contact, status `qualified`.
9. Two consecutive AI turns with confidence < 0.5 → auto-handoff on the second.
10. Inbound from unknown phone → `message.unrouted` activity, no SMS, no crash.

<!-- PART3 -->
