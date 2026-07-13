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

---

### 3.4 WF-3 Reactivation — Campaign Runner

**Module:** `reactivation`. Settings (defaults): `lapse_days: 90`, `batch_size_per_day: 50`, `stop_rate_pause_pct: 3`, `send_hour_local: 11` (advisory — the pipeline still enforces quiet hours), approval rule: first-ever campaign for a client is always `approve_first` (doc 03 flow 3).
**Audience building** (CSV import, `sale.completed` upserts into `customers`) is control-plane work, not n8n. Campaign creation + audience filter selection happens in the portal/admin (SPEC-05). WF-3 does two jobs: **(a) compose** copy variants for a draft campaign, **(b) run** approved campaigns in daily batches.

**Triggers (two, same workflow):**
- Webhook path `campaign-launch`, event `campaign.launch` (`payload: {campaign_id, stage: "compose" | "send"}`) — emitted by the control plane when a campaign is created (compose) or approved/launched (send).
- Schedule cron `0 15 * * *` UTC (daily continuation) → Shape-B head over `GET /api/internal/clients?module=reactivation&enabled=true`, then per client `GET /api/internal/campaigns?client_id=…&status=sending` → for each, run the SEND lane below with `campaign_id`. Use a `Merge` node (`n8n-nodes-base.merge`, mode Append) to join both trigger paths into the common lane; a `Code` node normalizes to `{client_id, campaign_id, stage}`.

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head A for the webhook path / Head B for the cron path, module `reactivation`)* | | |
| 7 | `Get Campaign` | HTTP Request | GET `…/api/internal/campaigns/{{ $json.campaign_id }}` → `{status, angle, copy_variants, audience_filter, stats}`. |
| 8 | `Stage?` | Switch | `compose` → node 9; `send` (or cron continuation with `status==='sending'||'approved'`) → node 12; anything else → `Log Skipped`. |
| — | **COMPOSE lane** | | |
| 9 | `Compose Variants (Claude)` | HTTP Request | Anthropic call (§2.8): model `claude-sonnet-5`, `max_tokens: 1200`, system **Prompt 4.3**, user content: JSON `{angle: <campaign.angle resolved from profile.campaign_angles>, business, services, lapse_days, booking_link}`; `output_config` schema §4.3 (3 variants + merge-field slots). Retry 2×; on error → ops email "composer failed" + PATCH campaign `status:"draft"` unchanged. |
| 10 | `Guard Variants` | Code | Deterministic checks per variant: contains `{first}` slot; contains opt-out-safe copy (pipeline appends STOP footer — variant must NOT contain its own); length ≤ 320 chars; no `$`-price unless the angle's `offer` text contains it verbatim; no banned phrases (`profile.brand_voice.phrases_avoid`). Fail → drop variant; if <2 survive → ops email + stop. |
| 11 | `Save Variants` | HTTP Request | PATCH campaign `{"copy_variants": [...], "status": "pending_approval"}` + activity `campaign.composed`. Portal approval (SPEC-05) flips to `approved` and emits `campaign.launch{stage:"send"}`. Lane ends. |
| — | **SEND lane** | | |
| 12 | `Send Gates` | IF | ALL of: `{{ $('Get Context').item.json.module.enabled }}` · `{{ $('Get Context').item.json.comms.a2p_campaign_status === 'approved' }}` · `{{ ['approved','sending'].includes($('Get Campaign').item.json.status) }}`. False → activity `campaign.gated` detail reason → NoOp. (A2P is a hard gate — doc 06.) |
| 13 | `Mark Sending` | HTTP Request | PATCH campaign `{"status":"sending"}` (idempotent). |
| 14 | `Fetch Today's Batch` | HTTP Request | GET `…/api/internal/campaigns/{{ campaign_id }}/members?status=queued&limit={{ $('Get Context').item.json.module.settings.batch_size_per_day ?? 50 }}` → `[{customer_id, name, phone, last_visit_at, visit_count}]` (control plane already excluded: opt-outs, active conversations, freq-capped, no-consent — doc 03 gate 1; ranked by historical value, gate 2). |
| 15 | `Empty?` | IF | `{{ $json.members.length === 0 }}` → true: `Finalize Campaign` — GET members count `status=queued`; if 0 → PATCH campaign `{"status":"done"}` + activity `campaign.done` → NoOp. |
| 16 | `Loop Members` | Split In Batches | Batch Size `1`. |
| 17 | `Merge Fields` | Code | Rotate variants: `const v = campaign.copy_variants[ index % campaign.copy_variants.length ]`; replace `{first}` (fallback "there"), `{business}`, `{last_visit_note}` (e.g. "It's been a few months since your last visit" derived from `last_visit_at` — template phrasing, not LLM), `{offer}`, `{booking_link}`. Output `{body, variant_id}`. |
| 18 | `Send Campaign SMS` | HTTP Request | POST `…/api/internal/send` `{"kind":"marketing","engine":"wf-3","campaign_id":"…","customer":{"phone":"…","name":"…"},"body":…,"idempotency_key":"{{ campaign_id }}:{{ $json.customer_id }}"}`. Marketing kind → pipeline enforces quiet hours (queues), 30-day cross-engine frequency cap, opt-out, A2P. |
| 19 | `Update Member` | HTTP Request | PATCH `…/api/internal/campaigns/{{ campaign_id }}/members/{{ customer_id }}` body `{"status": "{{ ['sent','queued','duplicate'].includes($json.status) ? 'sent' : 'excluded' }}", "detail":{"send_status":"{{ $json.status }}","blocked_reason":"{{ $json.blocked_reason ?? '' }}"}}` → **the resume ledger**: a crash between 18 and 19 is healed by the `duplicate` response on re-run (member re-selected as `queued`, send dedupes, member then marked `sent`). Loop back to node 16. |
| 20 | `STOP-rate Check` | HTTP Request | After loop completes: GET `…/api/internal/campaigns/{{ campaign_id }}/stats?window=24h` → `{sent, stops, stop_rate}`. |
| 21 | `Too Many STOPs?` | IF | `{{ $json.stop_rate > (($('Get Context').item.json.module.settings.stop_rate_pause_pct ?? 3) / 100) && $json.sent >= 20 }}` (min-sample guard). True → `Auto-Pause`: PATCH campaign `{"status":"paused"}` + activity `campaign.autopaused` `status:"failed"` detail `{stop_rate}` + ops email "STOP-rate breach — campaign paused" (doc 03 failure handling). |
| 22 | `Log Batch` | HTTP Request | POST activity `action:"campaign.batch_sent"` detail `{sent, queued, excluded, campaign_id}`. |

**Replies:** inbound answers route through WF-2B (node 7 there selects `module=reactivation` for `engine:"wf-3"` conversations); the AI books them or hands off — same loop, same prompt with reactivation context. The send-pipeline creates/links the `conversation_id` on first inbound (SPEC-03).
**Idempotency:** `campaign_members` ledger + `campaign_id:customer_id` send keys — a mid-batch crash never re-texts anyone (doc 03).

**Acceptance tests (WF-3)**

1. Compose stage on a draft campaign → 3 guarded variants saved, status `pending_approval`, nothing sent.
2. Launch with A2P `submitted` (not approved) → gated, zero sends, activity `campaign.gated` reason `a2p_pending`.
3. Approved campaign, 120 queued members, `batch_size_per_day: 50` → day 1: 50 sent; day 2 cron: 50; day 3: 20 + campaign flips `done`.
4. Kill n8n mid-batch after 23 sends; re-run → ledger shows exactly 120 sends total, zero duplicates (spot-check the 23: `duplicate` responses, members `sent`).
5. Member opted out between queueing and send → `blocked/opt_out`, member `excluded`, not retried.
6. Simulate 2 STOP replies in first 40 sends (5%) → campaign auto-paused, ops email, remaining members still `queued`.
7. Flip `reactivation` switch off mid-campaign → next daily cron gates out; members remain `queued`; flipping back on resumes exactly where it left off.
8. Variant containing "only $99 today!" when the approved angle has no price → guard drops it; if <2 variants remain, composer halts with ops alert.
9. Reply "YES sounds fun" to a campaign SMS → WF-2B routes with `module=reactivation`, AI answers using the campaign angle context, conversation linked to campaign for attribution.
10. Campaign stats after completion → `stats.sent/replies/stops/bookings/revenue_est` populated (booking/revenue via conversation outcomes) — feeds the portal case-study numbers.

---

### 3.5 WF-4 Content Engine

**Module:** `content_engine`. Settings (defaults): `cadence: "1,15"` (days-of-month), `publish_hour_utc: 7`, `min_words: 900`, `approval_mode: "approve_first_3"`, `publishing_target: "wordpress" | "email_draft"`, `webmaster_email` (for email_draft), `schema_profile: ["LocalBusiness","Service","FAQPage"]`.
**Trigger:** Schedule cron `0 7 1,15 * *` (UTC). Per-client cadence honored by the claim-topic endpoint (it refuses if the client isn't due).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head B, module `content_engine`)* | | |
| 7 | `Claim Topic` | HTTP Request | POST `…/api/internal/content/claim-topic` body `{"client_id":"{{ $json.client_id }}","period":"{{ $now.toFormat('yyyy-LL') }}-{{ $now.day <= 7 ? 'a' : 'b' }}"}` → `{claimed, content_item_id, topic, target_keywords, competitor_gap_notes}` or `{claimed:false, reason:"queue_empty"|"not_due"|"already_claimed"}`. Idempotent per (client, period) — a re-run of the cron cannot double-draft. |
| 8 | `Claimed?` | IF | `{{ $json.claimed }}`. False + reason `queue_empty` → `Ops Alert (replenish queue)` Send Email + activity `content.queue_empty` `status:"failed"` (doc 03: alert ops to replenish). False otherwise → skip-log → next client. |
| 9 | `Research Snapshot` | HTTP Request | GET `…/api/internal/clients/{{ client_id }}/competitor-snapshots?since={{ $now.minus({days:35}).toISO() }}` (WF-6 data feeds outlines — doc 09). **On Error: Continue** with empty array. |
| 10 | `Draft Post (Claude)` | HTTP Request | Anthropic call: model `claude-sonnet-5`, `max_tokens: 8000`, system **Prompt 4.6** (WF-4 content drafter, §4.6), user content = JSON `{topic, target_keywords, services, service_areas, competitors_recent: <node 9 extract>, brand_voice, business facts: NAP/hours/booking_link}`; `output_config` json_schema: `{title, slug, meta_description, tldr_html, body_html, faq: [{q, a}], jsonld: object, internal_link_suggestions: []}`. Retry 2×; on error → PATCH content item `{"status":"failed"}` + ops email + next client. |
| 11 | `Quality Gate` | Code | Deterministic checks (doc 03 flow 4): (1) word count of `body_html` stripped ≥ `min_words`; (2) `JSON.parse`-able `jsonld` with `@type` in schema_profile and `FAQPage` entries matching `faq[]`; (3) every `target_keywords[0..2]` appears ≥1× in body; (4) **no hallucinated facts**: regex `/\$\s?\d|\bfrom \d+ per\b/i` fails the draft unless the matched string exists in `profile` facts; phone/address in body must equal `locations[0]` values; (5) `tldr_html` non-empty and first in body; (6) no placeholder text (`lorem|TODO|\[insert`). Output `{pass, failures[]}`. |
| 12 | `QA Pass?` | IF | Fail → PATCH content item `{"status":"failed","detail":{failures}}` + activity `content.qa_failed` `status:"failed"` + ops email (schema validation failure blocks publish — doc 03). |
| 13 | `Approval Needed?` | IF | `{{ $('Get Context').item.json.module.settings.approval_mode === 'approve_first_3' && ($('Get Context').item.json.module.settings.published_count ?? 0) < 3 }}` (published_count maintained by control plane in module settings). True → PATCH content item `{"status":"pending_approval","title":…,"body_html":…,"schema_jsonld":…}` + activity `content.queued_approval` → next client. Approval in portal re-emits `campaign`-style event? No — portal publishes via this same lane by calling the control plane, which emits `content.approved` → **routes to this workflow's second webhook** `Webhook (content-approved)` (path `content-approved`) that jumps straight to node 14 (add this small entry: Head-A nodes 1–4 + `Get Context` + IF enabled → node 14). |
| 14 | `Save Draft` | HTTP Request | PATCH content item `{"status":"approved","title":…,"body_html": <tldr_html + body_html + faq rendered>,"schema_jsonld": jsonld}`. A Code node `Assemble HTML` right before renders: TL;DR block first, body, FAQ section, `<script type="application/ld+json">{{ jsonld }}</script>` appended (GEO pattern — doc 03). |
| 15 | `Target?` | Switch | `{{ $('Get Context').item.json.module.settings.publishing_target }}`: `wordpress` → 16a; `email_draft` → 16b; else → park + ops. |
| 16a | `Publish to WordPress` | HTTP Request | POST `{{ connections.wordpress.base_url }}/wp-json/wp/v2/posts` · Auth: Generic → Basic Auth with `{{ connections.wordpress.username }}` / `{{ connections.wordpress.app_password }}` (values from context — do NOT store as n8n credentials; per-client) · body `{"title":…,"slug":…,"content": <assembled html>,"status":"publish","excerpt": meta_description}` → response `link` is the URL. Retry 3×; **On Error: Continue** → `Park Ready-to-Publish`: PATCH `{"status":"ready_to_publish"}` + ops email (never silently lost — doc 03). |
| 16b | `Email Draft to Webmaster` | Send Email | To `{{ module.settings.webmaster_email }}` cc ops · Subject `New post for {{ client.business_name }}: {{ title }}` · HTML body = assembled post + publishing instructions + the JSON-LD block. Then PATCH `{"status":"published","published_url":null}` with `detail.channel:"email_draft"`. |
| 17 | `Mark Published` | HTTP Request | (wordpress lane) PATCH content item `{"status":"published","published_url":"{{ $json.link }}","published_at":"{{ $now.toISO() }}"}`. |
| 18 | `Emit content.published` | HTTP Request | POST `…/api/internal/events` `{"event_id":"content:{{ content_item_id }}","client_id":…,"type":"content.published","occurred_at":"{{ $now.toISO() }}","source":"wf-4","payload":{"content_item_id":…,"url":"{{ $json.link }}","title":…}}` → fans out to WF-7. Idempotent on the content-item-scoped event_id. |
| 19 | `Log` | HTTP Request | POST activity `action:"content.published"` `entity_type:"content_item"`. GSC/rankings/SoV tracking is in-app (pg-boss jobs, doc 10) — not n8n. |

**Idempotency:** claim-topic per period; `content:` event id; WP re-publish guarded by content status (`published` items are never re-entered — node 7 refuses with `already_claimed`).

**Acceptance tests (WF-4)**

1. Cron with due client + stocked queue → one content item drafted, QA-passed, published (or queued per approval mode), `content.published` event emitted once.
2. Re-run the cron the same day → `already_claimed`, zero new drafts.
3. Empty topic queue → ops email, activity `content.queue_empty`, no draft, other clients proceed.
4. Draft invents "$25 league night" not present in profile facts → QA fail, status `failed`, ops email, nothing published.
5. Invalid JSON-LD from the model → QA fail (schema validation blocks publish).
6. First 3 posts for a new client → all `pending_approval`; 4th auto-publishes. Portal approval of post 2 → published via `content-approved` webhook path, URL captured, event emitted.
7. WordPress returns 500 ×3 → item `ready_to_publish`, ops email, no `content.published` event.
8. `email_draft` client → webmaster email contains TL;DR, FAQ, JSON-LD; item marked published with `channel:"email_draft"`; event still emitted (syndication may still run — GBP/FB don't need the blog URL? They do for CTA → WF-7 handles null URL by using booking link CTA). 
9. Switch off `content_engine` → cron skips client, drafts in flight (pending_approval) stay parked untouched.

---

### 3.6 WF-5 Directory Sync

**Module:** `directory_sync`. Settings: `auto_push: true`, `audit_frequency: "quarterly"`, `directory_set: "brightlocal_default"`.
**BrightLocal auth:** all calls send `api-key: {{ $env.BRIGHTLOCAL_API_KEY }}` (plus sig where the endpoint family requires it). Base URL `https://tools.brightlocal.com/seo-tools/api`. **The exact endpoint paths below must be confirmed against the current BrightLocal API reference during N8N-12; the node structure and payloads are fixed.**

**Triggers (three lanes in one workflow):**
- Webhook path `profile-updated`, event `profile.updated` (`payload: {location_id, changed: ["hours","phone",…], profile: {name, address_json, phone, hours_json, holiday_hours_json, website}}`) → PUSH lane.
- Schedule `0 4 * * *` (daily) → STATUS-POLL lane.
- Schedule `0 5 1 1,4,7,10 *` (quarterly) → AUDIT lane.

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head A for webhook / Head B for schedules, module `directory_sync`)* | | |
| — | **PUSH lane** (`profile.updated`) | | |
| 7 | `Validate NAP` | Code | Require non-empty name, street, city, state, zip, E.164 phone; hours_json parses; reject with activity `profile.push_invalid` `status:"failed"` + ops email if not. Never push garbage to 15 directories. |
| 8 | `Has BL Location?` | IF | `{{ $('Get Context').item.json.locations.find(l => l.id === $json.payload.location_id)?.brightlocal_location_id }}` empty → node 9; else → node 10. |
| 9 | `BL Create Location` | HTTP Request | POST `{{BL}}/v1/clients-and-locations/locations` form/JSON body mapped from profile (business name, address, phone, urls, business category). → `{location_id}` → `Persist BL ID`: PATCH `…/api/internal/locations/{{ location_id }}` `{"brightlocal_location_id": …}`. Then → `BL Baseline Audit` (same call as AUDIT lane node 15) — the before-number for the case study (doc 03 onboarding). |
| 10 | `BL Update Location` | HTTP Request | PUT `{{BL}}/v1/clients-and-locations/locations/{{ brightlocal_location_id }}` with changed fields. Retry 3×; On Error: Continue → park + ops email. |
| 11 | `BL Start Sync` | HTTP Request | POST Citation Builder submission for the location (BrightLocal "CBT" campaign / listings sync per directory_set). Response: submission id → PATCH internal location `detail.bl_submission_id`. |
| 12 | `GBP Direct Push?` | IF | `{{ ['hours','holiday_hours','phone'].some(f => $json.payload.changed.includes(f)) && $('Get Context').item.json.connections.google_business?.status === 'active' && $('Get Context').item.json.module.settings.gbp_mode !== 'places_interim' }}` — hours pushed straight to GBP because aggregator propagation is slow (doc 03). False → activity `gbp.push_skipped` detail reason (interim mode: ops updates via manager access). |
| 13 | `GBP Update Hours` | HTTP Request | PATCH `https://mybusinessbusinessinformation.googleapis.com/v1/{{ gbp_location_id }}?updateMask=regularHours,specialHours,phoneNumbers` · Bearer context token · body built by Code node `Map Hours → GBP` (hours_json → `regularHours.periods[]`, holiday_hours_json → `specialHours`). Error → connection-broken lane (as WF-1B). |
| 14 | `Log Push` | HTTP Request | POST activity `action:"profile.pushed"` detail `{changed, bl_submission_id, gbp_pushed: bool}`. |
| — | **STATUS-POLL lane** (daily) | | |
| 15 | `BL Submission Status` | HTTP Request | Per client-location with an open submission: GET CBT campaign/submission status → `[{directory, status: live|pending|failed, url}]`. |
| 16 | `Update Listings Health` | HTTP Request | PATCH internal location `{"listings_health_score": <live/total*100>, "detail":{"per_directory": [...]}}` + activity `listings.status` — honest per-directory status in portal ("12/15 synced, 3 pending" — doc 03; no silent claims of consistency). |
| — | **AUDIT lane** (quarterly) | | |
| 17 | `BL Run Citation Audit` | HTTP Request | POST citation-tracker report run for the location → poll (Wait 10 min → GET report ×6 max) → results. |
| 18 | `Diff & Fix` | Code + HTTP | Diff new inconsistencies vs stored per-directory state → for each fixable diff POST the sync (node 11 call) → PATCH location health + activity `listings.audit` detail `{new_inconsistencies, fixes_submitted}` → portal report (SPEC-05 renders). |

**Idempotency:** BL location create guarded by the `brightlocal_location_id` IF; submissions idempotent per (location, day) — Code node skips if `bl_submission_id` open; `profile.updated` replays are harmless (same PUT payload).

**Acceptance tests (WF-5)**

1. First `profile.updated` for a location with no BL id → location created in BrightLocal, id persisted, baseline audit kicked off, sync submitted.
2. Replay the same event → no second BL location (IF routes to update), no duplicate submission same-day.
3. Hours change with active Google connection (`gbp_mode: api`) → GBP `regularHours` PATCHed within the run; BL sync also submitted.
4. Hours change in `places_interim` mode → BL push happens, GBP push skipped with logged reason.
5. BL API 500 ×3 on update → parked + ops email; GBP push still attempted (lanes independent).
6. Daily poll after 3 directories go live → `listings_health_score` recalculated, per-directory statuses visible via internal API.
7. Invalid payload (missing zip) → rejected before any vendor call, activity `failed`, ops email.
8. Quarterly audit finds 2 drifted listings → 2 fix submissions, activity `listings.audit` with counts.
9. Module off → pushes skipped with reason, last-known health untouched (doc 03 safe-off).

---

### 3.7 WF-6 Competitor Espionage

**Module:** `competitor_intel`. Settings: `digest_recipients: ["owner@client.com"]` (+ ops always), `mode: "client" | "prospect"`. Competitors from `profile.competitors[]` (3–5 per client, from intake Q9). **Zero client OAuth needed — first engine live (doc 07 week 1).**
**Prospect mode:** prospects are ordinary `clients` rows with `status: "prospect"` and `competitor_intel` enabled; the admin "prospect runner" (doc 05) creates them. `GET /api/internal/clients?module=competitor_intel&enabled=true&include_prospects=true` returns them. `mode:"prospect"` in module settings switches the digest template + recipients (Seekly only — never email a prospect's "client").

#### WF-6A Espionage — Weekly Snapshot

**Trigger:** Schedule cron `0 3 * * 1` (Mondays 03:00).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head B, module `competitor_intel`, list call includes `include_prospects=true`)* | | |
| 7 | `Split Competitors` | Split Out | Field `profile.competitors` from context; Code node first merges `{client_id, competitor}` per item and computes `competitor_key = slugify(competitor.name)`. |
| 8 | `Fetch Website` | HTTP Request | GET `{{ $json.competitor.website }}` · Response: string · Follow redirects · Timeout 15000 ms · **No retry, On Error: Continue** (unreachable = finding, §2.3). |
| 9 | `Extract Text` | Code | If fetch errored → `{unreachable: true}`. Else strip `<script>/<style>/tags`, collapse whitespace, keep first 30k chars, `website_text_hash = sha256(text)`; regex-harvest price mentions (`/\$\s?\d[\d,.]*/g` with 60-char context windows) into `price_mentions[]`. **No LLM here** — weekly lane stays cheap (doc 03: "internal, cheap"); Claude only runs monthly. |
| 10 | `Fetch Places Stats` | HTTP Request | GET `https://maps.googleapis.com/maps/api/place/details/json?place_id={{ $json.competitor.place_id }}&fields=rating,user_ratings_total,business_status,price_level&key={{ $env.GOOGLE_MAPS_API_KEY }}` · On Error: Continue (missing place_id → nulls). Public data only. |
| 11 | `Store Snapshot` | HTTP Request | POST `…/api/internal/competitor-snapshots` body `{"client_id":…,"competitor_key":…,"captured_at":"{{ $now.toISO() }}","website_text_hash":…,"website_extract":{"text_head": <first 5k>, "price_mentions": [...], "unreachable": bool},"gbp_stats":{"rating":…,"review_count": user_ratings_total}}` + activity `competitor.snapshot`. |

#### WF-6B Espionage — Monthly Digest

**Trigger:** Schedule cron `0 6 1 * *` (1st of month 06:00).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head B, module `competitor_intel`, `include_prospects=true`)* | | |
| 7 | `Load Month Snapshots` | HTTP Request | GET `…/api/internal/clients/{{ client_id }}/competitor-snapshots?since={{ $now.minus({days:35}).toISO() }}`. |
| 8 | `Compute Diffs` | Code | Pure-deterministic per competitor (doc 09: "numbers stay numeric — the LLM is never the calculator"): review_count delta first→last snapshot + weekly velocity; rating delta; `text_changed` = first/last hash differ; changed price_mentions (set diff); `unreachable_weeks` count. Output one item per client: `{competitors: [ {name, review_delta, review_velocity_anomaly: delta > 2*trailing_avg, rating_delta, price_changes:[], text_changed, extract_last, unreachable_weeks} ], month}`. |
| 9 | `All Quiet?` | Code | Flag `all_quiet = every competitor has no deltas/changes`. Digest is still generated + sent — "proof of monitoring" (doc 03) — the prompt handles brevity. |
| 10 | `Analyze (Claude)` | HTTP Request | Anthropic call: model `claude-sonnet-5`, `max_tokens: 3000`, system **Prompt 4.4**, user content = JSON from node 8 plus `{business, industry, services, mode: client|prospect, all_quiet}`; `output_config` schema §4.4. Retry 2×; on error → ops email "digest failed for <client>" + activity failed; **do not** send a half-digest. |
| 11 | `Render Email` | Code | Wrap `digest_html` in the branded email shell (logo header, footer). Prospect mode uses the prospect shell ("Prepared by Seekly — competitor intelligence sample for <prospect business>"). |
| 12 | `Store Digest` | HTTP Request | POST `…/api/internal/intel-digests` body `{"client_id":…,"period":"{{ $now.minus({months:1}).toFormat('yyyy-LL') }}","body_html":…,"findings": {{ JSON.stringify($('Analyze (Claude)').item.json.findings) }} ,"sent_at":null}` → portal archive; `findings[]` later promoted to `competitive`-domain L2 insight notes by the in-app intelligence jobs (doc 09 — the digest is for humans, the notes are for the system). |
| 13 | `Recipients` | Code | `mode==='prospect'` → `[OPS]` only; else `module.settings.digest_recipients + [OPS]`. |
| 14 | `Send Digest` | Send Email | To = node 13 list · Subject `{{ business }} — Competitor Intelligence, {{ month }}` (prospect: `What {{ prospect }}'s competitors did last month`) · HTML = node 11. Retry 2×; then PATCH digest `sent_at` via `POST …/api/internal/intel-digests` upsert / activity `intel.digest_sent`. |

**Idempotency:** snapshots append-only keyed (client, competitor, captured_at); digest per (client, period) — the intel-digests endpoint upserts on that pair, so a re-run replaces rather than duplicates, and node 14 skips email if the stored digest for the period already has `sent_at` (IF before send).

**Acceptance tests (WF-6)**

1. Weekly cron over a client with 3 competitors → 3 snapshot rows with hashes + GBP stats.
2. Competitor site times out → snapshot stored with `unreachable: true`; digest later says "site unreachable this month" — never fabricated content.
3. Competitor gains 14 reviews in the month → diff computes delta + anomaly flag; digest calls out the review spike with the number 14 exactly (deterministic math, LLM narrates).
4. No changes anywhere → short "all quiet" digest still generated, stored, and emailed.
5. Re-run monthly cron same day → digest upserted (one row for the period), email not re-sent (sent_at guard).
6. Prospect-mode client → digest stored, email goes to ops ONLY, prospect shell used.
7. Claude 500s → no email, ops alerted, snapshots untouched; re-run after fix produces the digest.
8. Client with `competitor_intel` off → skipped in both lanes; historical snapshots retained (safe-off).
9. Digest numbers audit: every number in the email exists in node 8's deterministic output (spot-check 3 digests — no LLM-invented figures).

---

### 3.8 WF-7 Syndication

**Module:** `social_syndication`, sub-switches `gbp_posts`, `facebook_posts`. Settings: `utm_defaults: {source:"gbp"|"facebook", medium:"social", campaign:"syndication"}`, `cta_default: "LEARN_MORE"`.
**Trigger:** Webhook path `content-published`, event `content.published` (from WF-4 **or** the control plane's RSS/sitemap watcher on client blogs — same envelope, `source:"rss-watch"`; syndication works even for clients who write their own content, doc 03).

| # | Node name | Type | Key parameters |
|---|---|---|---|
| 1–6 | *(Head A, module `social_syndication`; node 6 checks `module.enabled` only — per-channel sub-switches gate their own branches below)* | | |
| 7 | `Get Content` | HTTP Request | GET `…/api/internal/content/{{ payload.content_item_id }}` (WF-4 posts) — for RSS-sourced events skip this and use `payload.{url,title,summary}`; Code node `Normalize Source` outputs `{title, url, summary_or_body}` either way. `url` may be null (email_draft posts) → CTA falls back to `profile.booking_link`. |
| 8 | `Cut Channels (Claude)` | HTTP Request | Anthropic call: model `claude-sonnet-5`, `max_tokens: 1500`, system **Prompt 4.5**, user = JSON `{title, url, body_or_summary (first 4k chars), business, services, service_areas, target_keywords, brand_voice, booking_link}`; `output_config` schema §4.5. Retry 2×; On Error: Continue → `Park (cut failed)`: activity failed + ops email; branch ends (do not post raw excerpts). |
| 9 | `Validate Cuts` | Code | GBP: length ≤ 1500 chars, strip phone numbers if GBP policy config says so, no banned phrases, no ALL-CAPS shouting (`/[A-Z]{6,}/` on words), CTA url present; FB: length ≤ 5000, link present. Build final CTA URLs: `url ?? booking_link` + `?utm_source=<ch>&utm_medium=social&utm_campaign=syndication&utm_content={{ content_item_id }}`. Failures → per-channel park (a bad GBP cut must not kill the FB post). |
| 10 | *(fan-out: two parallel branches from node 9 — per-channel isolation, doc 03)* | | |
| — | **GBP branch** | | |
| 11g | `GBP Enabled?` | IF | `{{ module.settings.gbp_posts !== false }}` AND cut valid. False → activity `syndication.gbp_skipped`. |
| 12g | `GBP Mode?` | IF | `{{ module.settings.gbp_mode === 'api' && connections.google_business?.status === 'active' }}`. False → `Queue Manual GBP`: POST `…/api/internal/syndicated-posts`? — store via `PATCH`-style POST `…/api/internal/content/{{id}}` sub-resource: POST `…/api/internal/syndicated-posts` `{status:"pending_manual", channel:"gbp", body, cta_url_utm}` + activity (ops posts by hand, interim mode). |
| 13g | `Post GBP` | HTTP Request | POST `https://mybusiness.googleapis.com/v4/{{ gbp_account_id }}/{{ gbp_location_id }}/localPosts` · Bearer context token · body `{"languageCode":"en-US","summary": {{ JSON.stringify(gbp_cut) }},"topicType":"STANDARD","callToAction":{"actionType":"{{ cta_default }}","url":"{{ cta_url_gbp }}"}}` · Retry 3×; On Error: Continue → `Park GBP`: store `status:"failed"` + ops email; **FB branch unaffected**. |
| 14g | `Store GBP Post` | HTTP Request | POST `…/api/internal/syndicated-posts` `{"content_item_id":…,"client_id":…,"channel":"gbp","body":…,"cta_url_utm":…,"status":"posted","external_post_id":"{{ $json.name }}","posted_at":"{{ $now.toISO() }}"}` + activity `syndication.gbp_posted`. |
| — | **Facebook branch** | | |
| 11f | `FB Enabled?` | IF | `{{ module.settings.facebook_posts !== false && connections.meta?.status === 'active' }}`. False → activity `syndication.fb_skipped` (reason: switch or no connection). |
| 13f | `Post Facebook` | HTTP Request | POST `https://graph.facebook.com/v21.0/{{ connections.meta.page_id }}/feed` · Query/body: `message={{ fb_cut }}`, `link={{ cta_url_fb }}`, `access_token={{ connections.meta.page_access_token }}` (short-lived, from context) · Retry 3×; On Error: Continue → `Park FB` + ops email. |
| 14f | `Store FB Post` | HTTP Request | POST `…/api/internal/syndicated-posts` `{"channel":"facebook","status":"posted","external_post_id":"{{ $json.id }}", …}` + activity `syndication.fb_posted`. |
| 15 | `Merge + Final Log` | Merge (Append) + HTTP | Join branches → POST activity `action:"syndication.done"` detail `{gbp: status, fb: status}`. UTM click tracking is read by in-app analytics (doc 10) — not n8n. |

**Idempotency:** the syndicated-posts store is unique on `(content_item_id, channel)` — replayed `content.published` events hit the unique constraint; n8n treats the 409 response as `duplicate` success and skips the vendor call (add IF: pre-check GET `…/api/internal/syndicated-posts?content_item_id=&channel=` before each post — cheaper than relying on the constraint alone).

**Acceptance tests (WF-7)**

1. `content.published` from WF-4 → one GBP post + one FB post, both stored with external ids and UTM CTAs.
2. Replay the event → zero new vendor posts (pre-check catches both), activity shows duplicates skipped.
3. FB token invalid → FB parked + ops email; GBP post still published (per-channel isolation).
4. `gbp_posts` off, `facebook_posts` on → only FB posts; GBP skip logged.
5. GBP cut comes back 1700 chars → validation fails GBP branch only; ops sees the parked cut; FB proceeds.
6. RSS-sourced event (client's own blog) → cuts generated from summary, CTA uses the blog URL, both channels post.
7. `content.published` with null URL (email_draft) → CTA falls back to booking link with UTMs.
8. `places_interim` GBP mode → GBP cut stored `pending_manual` for ops; nothing calls the GBP API.
9. Module off entirely → event ignored, not queued (doc 03 safe-off: "`content.published` events ignored").

---

## 4. AI prompt templates

Rules that apply to **every** template:

- Placeholders `«like_this»` are filled by n8n expressions from the **context endpoint** (§2.6); the mapping table under each prompt is exact.
- Every call sets `output_config: {format: {type: "json_schema", schema: …}}` (§2.8) — the model returns only the JSON object; no prose parsing anywhere.
- Shared guardrails (from docs 02/03/08, embedded verbatim in each system prompt):
  - **Never state or promise prices, discounts, availability, or guarantees** unless the exact fact appears in BUSINESS FACTS. The intake's "never promise" list («ai_never_promise») is absolute.
  - **Never offer incentives for reviews** or mention reviews in exchange for anything (Google policy).
  - **Never fabricate** facts, numbers, testimonials, or competitor claims. If you don't know, say a human will follow up.
  - Compliance (opt-out lines, quiet hours, frequency) is handled by the platform — **never** add "reply STOP" or similar footers yourself.
  - Write in the brand voice: tone «brand_voice.tone», use phrases like «brand_voice.phrases_use», never use «brand_voice.phrases_avoid». Match the register of «brand_voice.example_sentences».

### 4.1 WF-2 conversation agent (first touch + conversation loop)

**System prompt template:**

```
You are «persona_name», the friendly virtual assistant for «business_name», a «industry» business in «city». You are texting (SMS) with a potential customer who just reached out. MODE: «mode».   // "first_touch" | "conversation"

YOUR JOB
1. Reply fast, warm, and specific — reference exactly what they asked about.
2. Work through the qualification questions ONE at a time, weaving them naturally into conversation (never as a form):
«qualification_questions»   // rendered as "- id: question (hot when: hot_signal)" lines
3. When the lead is qualified (all questions answered, or a hot signal appears), move them to booking: share «booking_link» and encourage them to pick a time.
4. Keep every message under 300 characters. One question per message, max. Plain text only, no markdown.

BUSINESS FACTS (the ONLY facts you may state)
- Services: «services»
- Areas served: «service_areas»
- Booking: «booking_link»
- Hours: «hours_summary»

HARD RULES
- NEVER promise or estimate: «ai_never_promise». If asked, say: "Great question — the owner will text you the details directly."
- If the customer is upset, mentions a complaint, legal issues, or negotiates price → set status to "needs_human" and write a short, warm holding reply.
- Never fabricate availability, staff names, or policies. Never mention being an AI unless directly asked; if asked, answer honestly and continue helping.
- Never add opt-out/STOP language — the platform appends it.
- Brand voice: tone «brand_voice.tone»; use: «phrases_use»; avoid: «phrases_avoid». Sound like: «example_sentences».
- After-hours (customer wrote outside «business_hours»): acknowledge you're answering right away anyway; do not promise a human until opening hours. «after_hours_note»

OUTPUT: a single JSON object per the schema. "extracted" must contain any qualification answers you can infer from the whole transcript, keyed by question id. Set confidence 0–1 for how sure you are the reply is appropriate and on-policy.
```

**Output JSON schema:**

```json
{ "type": "object", "additionalProperties": false,
  "required": ["reply_text", "status", "confidence", "extracted"],
  "properties": {
    "reply_text": { "type": "string" },
    "status": { "type": "string", "enum": ["continue", "qualified", "needs_human"] },
    "confidence": { "type": "number" },
    "extracted": { "type": "object", "additionalProperties": { "type": "string" } },
    "summary_for_owner": { "type": "string" }
  } }
```

**Placeholder map:** `persona_name` ← `module.settings.persona_name ?? profile.brand_voice.persona_name` · `business_name` ← `client.business_name` · `industry` ← `client.industry` · `city` ← `locations[0].address_json.city` · `mode` ← literal per call site · `qualification_questions` ← `profile.qualification_questions` (Code-rendered lines) · `booking_link` ← `profile.booking_link` · `services` ← `profile.services.join(', ')` · `service_areas` ← `profile.service_areas.join(', ')` · `hours_summary` ← Code-rendered from `locations[0].hours_json` · `ai_never_promise` ← `profile.ai_never_promise.join('; ')` · `brand_voice.*` ← `profile.brand_voice` · `business_hours`/`after_hours_note` ← `module.settings`.

### 4.2 WF-1 review responses

**System prompt template:**

```
You write the public owner responses to Google reviews for «business_name» («industry», «city»). Every response is read by future customers AND by search/AI engines — it is marketing and local SEO in one.

STYLE
- Brand voice: tone «brand_voice.tone»; use: «phrases_use»; avoid: «phrases_avoid».
- 2–4 sentences. Address the reviewer by first name when given. Vary openings — never start two responses the same way.
- Weave in AT MOST ONE natural local/entity reference per response, drawn ONLY from: «target_keywords», «services», «service_areas», «landmarks». Forced keyword stuffing is worse than none.

RATING RULES
- 4–5 stars: thank them, mirror one specific detail they mentioned, invite them back (reference a real service).
- 1–3 stars: empathetic, accountable, zero defensiveness, no excuses, never argue facts. Apologize for the experience (not "if you felt"), state one concrete make-it-right step, and take it offline: invite them to contact «owner_contact_channel». Do NOT offer refunds, discounts, or compensation.

HARD RULES
- NEVER offer incentives, discounts, or anything in exchange for the review or for changing it (Google policy — instant policy violation).
- NEVER promise: «ai_never_promise».
- Never dispute what happened, never mention other customers, never share personal data, never mention internal staff issues.
- No hashtags, no emojis unless «brand_voice.tone» explicitly allows them, no URLs.

Also produce service_recovery_suggestion: one sentence the OWNER (not the reviewer) should do next for ≤3★ reviews (e.g. "Call them today and offer to personally host their next visit"). Empty string for 4–5★.

OUTPUT: single JSON object per schema.
```

**Output JSON schema:** `{"type":"object","additionalProperties":false,"required":["reply_text","service_recovery_suggestion"],"properties":{"reply_text":{"type":"string"},"service_recovery_suggestion":{"type":"string"}}}`

**Placeholder map:** `landmarks` ← `profile.service_areas` + intake landmark list if present (`profile.target_keywords` doubles as entity source) · `owner_contact_channel` ← `locations[0].phone` or configured email · rest as §4.1. Review itself arrives in the **user** message (rating, author, text) — never in the system prompt.

### 4.3 WF-3 campaign composer

**System prompt template:**

```
You write SMS win-back campaigns for «business_name» («industry», «city»). The audience: past customers who haven't visited in «lapse_days»+ days. Goal: one warm, personal-feeling text that gets a reply or a booking — not a blast ad.

THE APPROVED ANGLE (you may ONLY make this offer, worded faithfully):
- Name: «angle.name»
- Offer: «angle.offer»
- Constraints: «angle.constraints»

WRITE 3 VARIANTS. Each must:
- Feel like a text from a real person at the business, not a campaign. Brand voice: tone «brand_voice.tone»; use: «phrases_use»; avoid: «phrases_avoid».
- Use merge fields literally: {first} (customer first name), {business}, {last_visit_note}, {offer}, {booking_link}. Include {first} and {booking_link} in every variant. Do not invent other fields.
- Be ≤ 300 characters BEFORE merge expansion. One clear call to action. No ALL CAPS, max one exclamation mark, no emojis unless tone allows.
- Differ meaningfully from each other (angle of approach, not synonyms): e.g. friendly check-in / event or occasion hook / straight offer.

HARD RULES
- State ONLY the approved offer — no invented discounts, prices, dates, or urgency ("today only") unless present in «angle.offer» verbatim.
- NEVER promise: «ai_never_promise».
- No opt-out language ("reply STOP") — the platform appends it.
- Nothing that would embarrass the owner if screenshotted.

OUTPUT: single JSON object per schema.
```

**Output JSON schema:** `{"type":"object","additionalProperties":false,"required":["variants"],"properties":{"variants":{"type":"array","minItems":3,"maxItems":3,"items":{"type":"object","additionalProperties":false,"required":["id","body"],"properties":{"id":{"type":"string"},"body":{"type":"string"}}}}}}`

**Placeholder map:** `angle.*` ← the `profile.campaign_angles[]` entry whose `id` equals `campaign.angle` · `lapse_days` ← `module.settings.lapse_days` · rest as §4.1.

### 4.4 WF-6 digest analyst

**System prompt template:**

```
You are Seekly's competitive intelligence analyst. You write the monthly competitor digest for «business_name» («industry», «city»). Reader: the busy owner. MODE: «mode».   // "client" | "prospect"

INPUT: a JSON diff computed by our systems — review deltas, rating changes, detected price/offer text changes, website-change flags, unreachable flags. These numbers are ground truth.

WRITE
1. digest_html: a short, punchy HTML email body (no <html>/<head> wrapper — content only):
   - Opening line: the single most important thing that happened this month, framed as money/threat/opportunity.
   - One section per competitor with material changes: what changed, why it matters to «business_name», and ONE recommended counter-move the owner could take this month. Concrete, small, doable.
   - Review-velocity spikes: call them out explicitly ("gained N reviews — they are likely running a review campaign").
   - Competitors with no changes: one collective line. Unreachable sites: say "site unreachable this month" — never guess at their content.
   - If input has all_quiet=true: a 4–6 line "all quiet" digest confirming monitoring ran and what we watched. Still useful, still confident.
   - Close (client mode): one line on what Seekly is doing about it automatically. (prospect mode): one line: "This is a sample of Seekly's monthly monitoring."
2. findings: machine-readable findings for our intelligence system.

HARD RULES
- Every number in your output MUST appear in the input JSON. Never invent counts, prices, or dates. Never fabricate offers you can't quote from website_extract.
- No generic filler ("in today's competitive landscape"). Every sentence earns its place.
- Recommended counter-moves must be things a local business can actually do (offer, event, review push, GBP post, page update) — never "consider a comprehensive strategy".
- Prospect mode: do not reveal or imply access to the prospect's own private data — competitor data is public.
```

**Output JSON schema:**

```json
{ "type": "object", "additionalProperties": false,
  "required": ["digest_html", "findings"],
  "properties": {
    "digest_html": { "type": "string" },
    "findings": { "type": "array", "items": { "type": "object", "additionalProperties": false,
      "required": ["competitor", "domain", "finding", "evidence", "recommended_counter", "confidence"],
      "properties": {
        "competitor": { "type": "string" },
        "domain": { "type": "string", "enum": ["competitive"] },
        "finding": { "type": "string" },
        "evidence": { "type": "string" },
        "recommended_counter": { "type": "string" },
        "confidence": { "type": "number" } } } } } }
```

**Placeholder map:** `mode` ← `module.settings.mode ?? 'client'`; rest as §4.1. The diff JSON goes in the **user** message. `findings[].domain` is fixed `"competitive"` so the in-app job can file them as L2 notes (doc 09).

### 4.5 WF-7 channel cutter

**System prompt template:**

```
You repurpose one blog post into channel-native local social content for «business_name» («industry», «city»). Two outputs, written independently — never a truncated copy of each other.

1. gbp_post — Google Business Profile "What's New" post:
   - ≤ 1400 characters (hard platform limit 1500 — leave headroom).
   - Local-intent first: open with the concrete benefit/insight for people in «service_areas», not with "New blog post!".
   - Naturally include 1–2 of: «target_keywords», one service from «services», one geo reference from «service_areas». No stuffing.
   - End with a soft prompt to act; the CTA button ("Learn more") is added by the platform — do not write "click the link below".
   - GBP policy: no phone numbers, no ALL-CAPS words, no excessive punctuation/emoji, no offers or prices unless present verbatim in the source post.
2. facebook_post — Page update:
   - Looser, community tone; 2–5 short paragraphs or lines; a hook first line; it's fine to be warmer/more personal. Link preview does the selling — don't paste the URL in the text (the platform attaches it).
   - May use up to 2 emojis if brand tone allows.

BOTH: brand voice tone «brand_voice.tone»; use: «phrases_use»; avoid: «phrases_avoid». Facts only from the source post and BUSINESS FACTS. NEVER promise: «ai_never_promise». No review solicitation. No hashtag walls (max 2 on Facebook, 0 on GBP).

OUTPUT: single JSON object per schema.
```

**Output JSON schema:** `{"type":"object","additionalProperties":false,"required":["gbp_post","facebook_post"],"properties":{"gbp_post":{"type":"string"},"facebook_post":{"type":"string"}}}`

**Placeholder map:** as §4.1; the source post `{title, url, body_or_summary}` arrives in the user message.

### 4.6 WF-4 content drafter (referenced by §3.5 node 10)

**System prompt template (condensed — the GEO pattern from doc 03 flow 3):**

```
You write local-SEO/GEO blog posts for «business_name» («industry», «city») that rank in Google AND get cited by AI engines. Audience: local customers searching high-intent queries.

STRUCTURE (mandatory): H1 with the primary keyword; TL;DR block first (3–4 bullet answers, direct and quotable); H2 sections answering the topic and People-Also-Ask-style subquestions; FAQ section (4–6 Q&As, conversational questions); semantic entity references woven in (place names from «service_areas», services from «services», local landmarks); 2–3 internal link placeholders as <a href="{{internal:slug-suggestion}}">anchor</a>.

JSON-LD: produce valid schema.org markup — LocalBusiness (name/address/phone EXACTLY as given in BUSINESS FACTS), Service for the topic service, FAQPage mirroring the FAQ section verbatim.

HARD RULES: facts, prices, hours, offerings ONLY from BUSINESS FACTS — a single invented price fails the whole draft. NEVER promise: «ai_never_promise». ≥ «min_words» words. Brand voice per «brand_voice». No fluff intros ("In today's world…"). Write to be quoted: short declarative answer sentences at the top of each section.

OUTPUT: single JSON object per schema (title, slug, meta_description, tldr_html, body_html, faq[], jsonld, internal_link_suggestions[]).
```

Placeholder map as §4.1 plus `min_words` ← `module.settings.min_words`; topic/keywords/competitor-gap notes arrive in the user message.

---

## 5. Ticket table and build order

Assumed companion specs (referenced as dependencies): **SPEC-01** control-plane schema migrations (doc 10 table list) · **SPEC-02** internal API (core four + §2.7 additions + event router/HMAC forwarding + HubSpot inbound) · **SPEC-03** send-pipeline service (`/api/internal/send` + inbound SMS → `message.received`) · **SPEC-04** integration hub (Google/Meta OAuth vault, email-parse → events, short-lived token minting) · **SPEC-05** portal additions (approvals queue, switchboard, campaign creation, intake/prospect runners).

| # | Ticket | Builds | Depends on | Order rationale |
|---|---|---|---|---|
| N8N-1 | Deploy n8n stack | §1 compose, proxy, backups, export script, `Anthropic Seekly` + `SMTP Seekly Ops` credentials | VPS + DNS only | Day 1 of week 1. |
| N8N-2 | WF-E + standard head proven | §2.2 error workflow; head Shape A+B built in a `WF-TEST Dummy` workflow against a seeded dummy client | SPEC-01 (clients row), SPEC-02 (context, activity, events, HMAC forwarding) | Doc 07 week 1: "standard workflow shell proven with a dummy client". Blocks everything below. |
| N8N-3 | WF-6A + WF-6B live | §3.7 | N8N-2; SPEC-02 (`clients?module=`, snapshots, intel-digests endpoints); Places API key | First engine live — zero client OAuth (doc 07 week 1). Generates pilot digest + 2–3 prospect digests. |
| N8N-4 | WF-0 lite (manual provisioning) | §3.1 with manual `client.provisioned` events; `provision/*` endpoints stubbed to core steps (no HubSpot) | SPEC-01, SPEC-02 (provision endpoints), Twilio account | Pilot tenant creation week 1; full HubSpot path deferred to N8N-13. |
| N8N-5 | WF-1A requests | §3.2 A | N8N-2; SPEC-03 (`/send`); SPEC-02 (eligibility); email-parse live (SPEC-04) for real `sale.completed` | Week 2 milestone. Runs in shadow mode until A2P approved (pipeline blocks sends — flip = A2P status change, no workflow change). |
| N8N-6 | WF-1B poller + WF-1C responder | §3.2 B/C, interim `places_interim` mode | N8N-2; SPEC-02 (reviews PATCH, event routing); Places key; portal approvals (SPEC-05) for queue modes | Week 2: detection via Places, replies via approval queue + ops manual publish until GBP API approval lands. |
| N8N-7 | WF-2A first touch + follow-ups | §3.3 A | N8N-2; SPEC-03; SPEC-02 (conversations, eligibility); email-parse form CC live | Week 2 milestone (`<60s` promise). |
| N8N-8 | WF-2B conversation loop | §3.3 B | N8N-7; SPEC-03 inbound → `message.received` | Immediately after N8N-7 — first-touch without a reply loop is a broken promise. |
| N8N-9 | WF-3 campaign runner | §3.4 | N8N-8 (reply routing); SPEC-02 (campaigns endpoints); SPEC-05 (campaign creation + approval); customer CSV imported | Weeks 3–4. |
| N8N-10 | WF-4 content engine | §3.5 | N8N-2; SPEC-02 (content endpoints); WordPress creds or webmaster email per intake | Weeks 3–4; feeds WF-7. |
| N8N-11 | WF-7 syndication | §3.8 | N8N-10 (event source); SPEC-04 (Meta page token in context); GBP interim path from N8N-6 | Immediately after first WF-4 publish. |
| N8N-12 | WF-5 directory sync | §3.6 | N8N-2; SPEC-02 (locations PATCH); BrightLocal account/key; portal profile editing emits `profile.updated` (SPEC-05) | Weeks 3–4; baseline audit ASAP for the before-number. Confirm BL endpoint paths here. |
| N8N-13 | WF-0 full (HubSpot end-to-end) | HubSpot Closed-Won → control plane → WF-0; sync-back fields | N8N-4; SPEC-02 HubSpot inbound + sync-back | Weeks 3–4 (doc 07). |
| N8N-14 | Hardening pass | Replay drills for every acceptance test marked idempotency; STOP-rate auto-pause test; switch-off-mid-flight tests; restore-from-backup drill; GBP API swap-in (`gbp_mode: api`) when approval lands | All above | Doc 07 weeks 3–4 hardening list, verbatim. |

**Definition of done per ticket:** all numbered acceptance tests for the workflow pass against staging; workflow JSON exported and committed (§1.4); every action visible in `activity_log`; WF-E fires on an induced failure within 5 minutes.

---

## Appendix A — Open questions (tracked; do not block N8N-1..3)

1. **BrightLocal endpoint paths** (§3.6) — structure fixed, paths to be confirmed against current BL docs during N8N-12.
2. **`GET /api/internal/campaigns?client_id&status=sending`** list form is implied by the WF-3 cron lane — confirm inclusion in SPEC-02.
3. **Places review IDs are synthetic** (§3.2 WF-1B node 10) — acceptable for interim mode? Duplicate-risk window is small (author+time hash) but nonzero; GBP API mode eliminates it.
4. **`review.received` payload must echo `response_status`** for the WF-1C replay guard — confirm the event router enriches payloads from the reviews table.
5. **Meta Graph API version pin** (`v21.0` in §3.8) — bump to current stable at N8N-11 build time and record in the workflow JSON.

