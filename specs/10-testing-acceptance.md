# 10 — Testing & Acceptance Spec

**Audience:** a junior developer who must verify the platform without asking questions.
**Scope:** everything in docs 01–10, centered on doc 07's "Definition of rock solid".
**Where code lives:** all automated tests and drill scripts live in the
`seekly-client-insights` repo (it IS the control plane per doc 10). This document lives in
the planning repo. n8n workflow JSON + drill runbooks that touch n8n live in
`seekly-platform/workflows/`.

Nothing in this spec requires a real Twilio number, a real Claude call, or a real client —
everything below runs in **mock mode** except the explicitly marked "LIVE" steps in the
go-live checklist (section 6).

---

## 0. Acceptance criteria — doc 07 "rock solid", operationalized

Every bullet of doc 07's definition maps to concrete test IDs in this spec. A module is
"rock solid" only when every mapped test passes. This table is the traceability matrix;
keep it updated when tests are added.

| Doc 07 bullet | Meaning, operationalized | Test IDs (this doc) |
|---|---|---|
| **Idempotent** — replaying any event/webhook produces zero duplicate messages/posts (proven by test) | Same `event_id` POSTed N times → exactly 1 ledger row / post / DB row per engine; n8n re-run after kill → no dupes; campaign crash-resume → no re-texts | R-1 … R-8 (§3), U-5 (§1.2) |
| **Safe-off** — flipping the switch mid-flight strands nothing | Each of the 7 switches flipped mid-flight behaves exactly per doc 03's safe-off table; re-enable resumes without duplication or retroactive sends | S-1 … S-8 (§4) |
| **Compliant** — STOP honored instantly across all engines; zero sends outside quiet hours or before A2P approval; pipeline-enforced | The MUST-PASS compliance suite; blocks proven at the send-pipeline layer (an engine that bypasses the pipeline cannot pass these tests) | C-1 … C-10 (§2), U-1 … U-4 (§1.2) |
| **Observable** — every action in `activity_log`; failures alert ops within 5 minutes; no silent drops | Every test in §2–§5 also asserts its activity_log row; D-4 measures alert latency with timestamps; "no silent drops" = every blocked/failed send has a ledger row with `blocked_reason`/`status=failed` | A-1 (§1.4), D-4 (§5), all §2 "ledger" assertions |
| **Fallback-first** — AI failure → template; API failure → retry → parked + alert; connection broken → module auto-pause with visible reason | Failure-injection drills with mock faults | D-1 … D-6 (§5) |

**Global invariant asserted by the whole suite (memorize it):**

> A message either (a) exists exactly once in the `messages` ledger with a terminal status,
> or (b) exists exactly once with `status=blocked` + a `blocked_reason`, or (c) was queued
> for quiet hours and exists with `status=queued`. There is no fourth outcome. No send
> without a ledger row; no ledger row without an activity_log entry.

---

## 1. Test strategy

Three layers. Each layer has one command, one config, and one convention.

| Layer | Runner | Command | Needs | What it proves |
|---|---|---|---|---|
| Unit | vitest (existing) | `npm test` | nothing (no DB, no network) | pure logic: gates, math, schemas, key derivation |
| Integration | vitest, separate config | `npm run test:int` | dev Postgres (Docker) + `MESSAGING_MODE=mock` + `ENGINE_MODE=mock` | SQL, ledger writes, pipeline end-to-end inside the app |
| E2E drills | bash/tsx scripts | `scripts/drills/dNN-*.sh` | full dev stack: app + worker + n8n, all mocks | cross-process behavior: n8n kills, alert latency, switch flips |

### 1.1 Conventions (extend what exists — do not invent parallel patterns)

The repo already has the pattern to copy. Follow it exactly:

1. **Unit tests are colocated** as `*.test.ts` next to the source file
   (`src/server/reviews/compliance.test.ts` is the model). `npm test` runs `vitest run`.
2. **Pure, fail-closed decision functions.** `src/server/reviews/compliance.ts` →
   `decideSend()` is the house style: no DB, no clock, every rule falls through to
   "suppress"/"blocked", and the test file flips one field at a time from a known-good
   baseline. The generalized send-pipeline (doc 10's `POST /api/internal/send`) MUST keep
   its decision core as a pure function exactly like this so the compliance suite can be
   exhaustive without a database.
3. **Mock-by-default adapters.** `src/server/engines/registry.ts` resolves adapters via
   `env.ENGINE_MODE` (`mock` default, `live` opt-in), with deterministic mocks
   (`src/server/engines/mock.ts` hashes the input for stable-but-varied output).
   Messaging copies this pattern (§1.3).
4. **Integration tests** are named `*.int.test.ts`, colocated the same way. Add to
   `vitest.config.ts` `exclude`: `"**/*.int.test.ts"` (so `npm test` stays hermetic) and
   create `vitest.integration.config.ts` that *includes only* `**/*.int.test.ts`. Add:
   ```jsonc
   // package.json scripts
   "test:int": "vitest run --config vitest.integration.config.ts",
   "test:compliance": "vitest run --config vitest.integration.config.ts src/server/pipeline/compliance-suite.int.test.ts"
   ```
5. **Integration DB** = a second database on the existing dev container so tests can
   truncate freely without touching your seeded dev data:
   ```bash
   docker exec seekly-insights-pg createdb -U seekly seekly_insights_test || true
   TEST_DATABASE_URL=postgres://seekly:seekly_dev@localhost:5433/seekly_insights_test \
     npm run db:migrate:test   # add script: ENV-overridden drizzle-kit migrate
   ```
   Integration setup (`src/test/int-setup.ts`, wired via the integration config's
   `setupFiles`): connect to `TEST_DATABASE_URL`, refuse to run if the URL does not end in
   `_test`, truncate all growth-plane tables between test files, and load fixtures from
   `src/test/fixtures/` (one canonical pilot client: `client_id` fixed UUID
   `00000000-0000-4000-8000-000000000001`, timezone `America/Denver`, quiet hours default,
   A2P `approved` unless a test says otherwise).
6. **Time is always faked in unit tests** (`vi.useFakeTimers()` + `vi.setSystemTime()`), so
   DST tests run identically any day of the year. Integration tests that need "now" pass an
   explicit `now: Date` parameter into the pipeline (the pipeline must accept one; default
   `new Date()`).
7. **Server TZ is pinned to UTC** in every test entrypoint (`process.env.TZ = "UTC"` in
   setup files) — this is what makes the "client TZ ≠ server TZ" tests honest.

### 1.2 Unit coverage map (what to cover, per spec)

| ID | Module under test | File (new unless noted) | Required cases |
|---|---|---|---|
| U-1 | **Send-pipeline gates** — the pure decision core of `/api/internal/send` (doc 03 pipeline steps 1–4) | `src/server/pipeline/decide.ts` + `decide.test.ts` | Baseline-that-sends, then flip one input per test (the `compliance.test.ts` pattern): opted-out → `blocked/opt_out`; quiet hours + `kind=marketing` → `queued/quiet_hours`; quiet hours + `kind=conversational` (reply to inbound lead) → send (doc 03 WF-2 exemption); quiet hours + `kind=transactional` → send; freq-cap exceeded + marketing → `blocked/freq_cap`; freq-cap exceeded + conversational → send; `a2p_campaign_status != 'approved'` → `blocked/a2p_pending` for ALL kinds; no consent basis + marketing → blocked; unknown/missing field → blocked (fail-closed) |
| U-2 | **Quiet-hours / timezone math** | `src/server/pipeline/quiet-hours.ts` + test | `localHourInTz(tz, date)` for: client `America/Denver`, server UTC (see C-4 vectors); DST fall-back + spring-forward vectors (C-5); midnight normalization (Intl "24" quirk — existing `localHourFor` already guards this; keep the `% 24`); window boundaries: 07:59 queue, 08:00 send, 20:59 send, 21:00 queue; invalid/unknown tz → treat as quiet (fail closed) + flag for ops. Also `nextWindowOpen(tz, date)` returns the next 08:00 client-local instant (used to schedule queued sends) — assert across the DST boundary it is 08:00 *new-offset* local |
| U-3 | **Frequency-cap SQL** | `src/server/pipeline/freq-cap.ts` + `freq-cap.int.test.ts` (SQL runs in the int layer; the unit layer tests the window arithmetic) | Cap counts only `kind='marketing'`, `direction='out'`, `status IN ('queued','sent','delivered')` (a blocked message must NOT consume the cap), window `created_at > now() - interval '30 days'` (per-client override from `workflow_config.settings.freq_cap_days`), **across engines** (`engine` column deliberately absent from the WHERE clause — test asserts a wf-1 row blocks a wf-3 send), scoped `(client_id, customer_id)` — same phone under two different clients does NOT cross-block |
| U-4 | **Email-parse extraction schemas** | `src/server/ingest/schemas.ts` + test; fixtures in `src/test/fixtures/emails/` | One zod schema per canonical event payload (doc 02 envelope rules): `sale.completed` requires `customer.{name,phone}` with `amount?`, `items?`; `lead.created` requires contact + source. Golden-sample tests: pilot POS notification email fixture → parsed payload deep-equals a checked-in golden JSON; **confidence threshold**: extraction result below `MIN_EXTRACTION_CONFIDENCE` (default 0.8) → routed to ops review queue, NO event emitted (assert zero events, one ops-queue row); phone normalization to E.164; unknown extra fields preserved under `payload.raw` |
| U-5 | **Idempotency key derivation** | `src/server/pipeline/idempotency.ts` + test | `messageKey(event_id, engine, step)` = `${event_id}:${engine}:${step}` — deterministic, stable, distinct per step (WF-2 first-touch vs follow-up #1 differ); campaign sends use `${campaign_id}:${customer_id}` (matches the `campaign_members` PK); replaying the same inputs yields the identical key (this is what the DB unique constraint on `messages.idempotency_key` catches) |
| U-6 | **STOP/HELP parsing** | `src/server/pipeline/inbound.ts` + test | Case/whitespace-insensitive STOP, STOPALL, UNSUBSCRIBE, CANCEL, END, QUIT → opt-out; HELP/INFO → help reply; "stop by tomorrow?" (keyword inside a sentence) → per carrier rules, single-word-message match only — document and test the exact rule; anything else → `message.received` conversation event |

### 1.3 Integration layer: `MESSAGING_MODE=mock` (mirrors `ENGINE_MODE`)

Define in `src/env.ts`, exactly parallel to `ENGINE_MODE`:

```ts
MESSAGING_MODE: z.enum(["mock", "live"]).default("mock"),
```

New module `src/server/messaging/registry.ts` mirroring `src/server/engines/registry.ts`:
`getMessagingAdapter()` returns the mock unless `MESSAGING_MODE === "live"` **and** Twilio
env vars are present (fail closed to mock, same as `getAdapter`'s `hasLiveCredentials`).
The existing `src/server/reviews/messaging.ts` `sendReviewSms` moves behind this adapter —
after this refactor there must be exactly **one** call site in the codebase that can reach
Twilio (grep-able acceptance check:
`grep -rn "api.twilio.com" src/ --include="*.ts"` returns only the live adapter file).

**Mock Twilio adapter contract** (`src/server/messaging/mock.ts`):

1. Deterministic: returns `twilio_sid = "MOCK" + sha1(client_id + to + idempotency_key).slice(0,30)`.
2. Writes are real: the ledger row, status transitions `queued → sent → delivered`
   (delivered simulated after `MOCK_TWILIO_DELIVERY_MS`, default 50ms, via the normal
   status-callback code path — the mock POSTs to our own status-callback route so that code
   is exercised too).
3. **Magic numbers** (superset of Twilio's own test numbers, so drills read naturally):

   | To-number | Mock behavior |
   |---|---|
   | `+15005550006` | success (valid, delivers) |
   | `+15005550001` | Twilio 400 invalid number (non-retryable → fail immediately, no retries) |
   | `+15005550500` | Twilio 500 on every attempt (retryable → drives D-1) |
   | `+15005550503` | Twilio 500 twice, then success (proves retry actually retries) |
   | `+15005550429` | Twilio 429 rate-limit (retryable with backoff) |

4. Env fault knobs (for drills that can't choose the number):
   `MOCK_TWILIO_FORCE_STATUS=500`, `MOCK_TWILIO_LATENCY_MS=2000`.

**Mock inbound SMS** — needed to simulate STOP/replies without a phone. Ship a script (not
an HTTP backdoor):

```bash
npm run sim:sms -- --client golf-pilot --from "+16135550199" --body "STOP"
```

`scripts/sim-sms.ts` builds the exact Twilio webhook form payload and invokes the inbound
route handler in-process (same code path as production, including signature validation
bypassed only when `MESSAGING_MODE=mock` — the bypass must throw if `APP_ENV=production`).

**Mock Claude** — growth-engine AI calls (review replies, WF-2 conversation, WF-3 copy,
extraction) go through a single `src/server/ai/` client that respects the existing
`ENGINE_MODE=mock` and adds fault injection:

```
AI_MOCK_FAULT= (unset) | timeout | error | garbage
```

`timeout` = hang past the pipeline's AI deadline (drives D-2 fallback-template test);
`garbage` = syntactically valid but schema-invalid output (drives quality-gate tests).
Mock generations are deterministic (hash of prompt, same trick as `engines/mock.ts`).

### 1.4 Integration suite contents

Beyond the compliance/replay suites (§2–§3), the int layer must include:

- **A-1 Ledger/activity invariant test** (`src/server/pipeline/invariant.int.test.ts`):
  run a mixed scenario (10 sends: 2 opted out, 2 in quiet hours, 1 over cap, 5 clean), then
  assert: `messages` row count == 10; every row has terminal or queued status; every row has
  a same-transaction `activity_log` entry; zero rows in neither table ("no silent drops").
- **Internal API auth**: `/api/internal/*` without `Authorization: Bearer $SEEKLY_INTERNAL_TOKEN`
  → 401; with it → 2xx; token absent from all client-facing responses.
- **Context endpoint**: `GET /api/internal/clients/:id/context?module=review_automation`
  returns profile + module config + switch state; returns short-lived access token, NEVER a
  refresh token (assert response body does not contain the refresh token string).

### 1.5 E2E drill scripts

Live in `scripts/drills/`, numbered to match this spec (`d01-twilio-500.sh`, …,
`s01-switch-review.sh`, `r01-replay-sale.sh`). Each script: sets up via `psql` + curl,
acts, polls, prints `PASS`/`FAIL <reason>` and exits 0/1 accordingly. Shared header
sourced by all drills:

```bash
# scripts/drills/env.sh
export BASE_URL="${BASE_URL:-http://localhost:3000}"
export SEEKLY_INTERNAL_TOKEN="${SEEKLY_INTERNAL_TOKEN:?set me}"
export CLIENT_ID="${CLIENT_ID:-00000000-0000-4000-8000-000000000001}"
PSQL() { docker exec -i seekly-insights-pg psql -U seekly -d seekly_insights -qtAX -c "$1"; }
api() { curl -sS -H "Authorization: Bearer $SEEKLY_INTERNAL_TOKEN" -H "Content-Type: application/json" "$@"; }
```

Full dev stack for drills (4 terminals or a compose file):
`npm run dev` (app) · `npm run worker` (pg-boss) · n8n container · nothing else. All env:
`ENGINE_MODE=mock MESSAGING_MODE=mock APP_ENV=development`.

---

## 2. Compliance test suite — MUST-PASS before any client sends

**Rule: `npm run test:compliance` green is a hard release gate for every client go-live and
every deploy that touches `src/server/pipeline/**`.** These are integration tests against
the test DB with `MESSAGING_MODE=mock`; each is also mirrored in miniature as a U-1 unit
case. "Expected ledger" below means a row in `messages` — remember the global invariant:
blocked sends still write a row.

Common fixture (loaded by `int-setup.ts`): client `golf-pilot`
(`00000000-…-0001`), `timezone='America/Denver'`, `comms_provisioning.a2p_campaign_status='approved'`,
switches ON for `review_automation`, `speed_to_lead`, `reactivation`; customer Casey
`+16135550199`, `consent_basis='existing_customer'`, no opt-out. Tests state only their
deltas from this baseline.

### C-1 · STOP honored across all engines within 1 message

**Setup:** Casey has an active WF-2 conversation AND is a queued member of reactivation
campaign K (status `sending`, Casey not yet sent) AND a `sale.completed` for Casey is
mid-delay in WF-1 (send scheduled in the future).
**Act:** `npm run sim:sms -- --client golf-pilot --from "+16135550199" --body "STOP"`.
Then release all three pending sends (advance the WF-1 delay, tick the campaign batch,
have WF-2 attempt a follow-up).
**Expected:**
1. `opt_outs` row `(client_id, '16135550199')` exists with `source_message_id` set, written
   before the sim:sms call returns (synchronous — this is what "within 1 message" means:
   after the STOP is processed, zero further messages may go out; nothing waits for a cron).
2. Exactly one outbound after STOP: the carrier-required STOP confirmation
   (`kind='transactional'`, body per carrier template). Nothing else.
3. All three pending sends produce ledger rows `status='blocked', blocked_reason='opt_out'`
   — one per engine (`wf-1`, `wf-2`, `wf-3`), proving the block is pipeline-level, not
   engine-level.
4. `campaign_members` row for Casey → `status='excluded'`.
5. Three `activity_log` rows `action='send.blocked'`, `status='skipped'`.

```sql
-- verification (expect: 1 / 1 / 3 / 0)
select count(*) from opt_outs where client_id=:cid and phone='16135550199';
select count(*) from messages where client_id=:cid and direction='out'
  and created_at > :stop_at and kind='transactional';           -- the confirmation
select count(*) from messages where client_id=:cid and status='blocked'
  and blocked_reason='opt_out' and created_at > :stop_at;
select count(*) from messages where client_id=:cid and direction='out'
  and status in ('sent','delivered') and created_at > :stop_at
  and kind <> 'transactional';                                   -- MUST be 0
```

### C-2 · HELP answered, does not opt out

**Act:** sim:sms body `HELP`. **Expected:** one transactional reply (help template with
business name + opt-out instructions), zero `opt_outs` rows, subsequent sends unaffected.

### C-3 · Quiet hours: queued, never dropped, never sent

**Setup:** `now = 2026-07-13T04:30:00Z` (= 22:30 July 12, America/Denver — inside quiet
hours). WF-3 attempts a marketing send to Casey.
**Expected:** ledger row `status='queued'`, `blocked_reason` NULL (queued ≠ blocked), with
scheduled release at the next 08:00 client-local = `2026-07-13T14:00:00Z`. Advance the
worker clock past 14:00Z → message sends exactly once. Assert zero `sent` rows with
`created_at` in the quiet window, and total sends for the trigger == 1 (queue release must
not double-send: same idempotency key).

### C-4 · Timezone edge: client TZ ≠ server TZ

Server runs UTC (`TZ=UTC` asserted in setup). Client stays `America/Denver` (MDT, UTC-6 in
July). Table-driven through the full pipeline (not just the unit fn):

| `now` (UTC) | Denver local | Marketing send → expected |
|---|---|---|
| `2026-07-13T13:59:00Z` | 07:59 | queued |
| `2026-07-13T14:00:00Z` | 08:00 | sent |
| `2026-07-14T02:59:00Z` | 20:59 | sent |
| `2026-07-14T03:00:00Z` | 21:00 | queued |

A pipeline that evaluates quiet hours in server-local or UTC time fails at least two rows.
Repeat one row with a second client fixture in `Australia/Sydney` (UTC+10 — date rolls
over) to catch date-boundary bugs: `2026-07-13T22:30:00Z` = 08:30 **July 14** Sydney → sent.

### C-5 · DST boundaries

Client fixture in `America/New_York`. Fake time (`vi.setSystemTime` in the int test /
explicit `now` param):

| `now` (UTC) | NY local (correct) | Naive fixed-offset answer | Expected |
|---|---|---|---|
| `2026-11-01T12:30:00Z` (fall-back day; EST from 06:00Z) | 07:30 EST | 08:30 (stale UTC-4) → would wrongly send | **queued** |
| `2026-11-01T13:00:00Z` | 08:00 EST | 09:00 | **sent** |
| `2027-03-14T12:30:00Z` (spring-forward day; EDT from 07:00Z) | 08:30 EDT | 07:30 (stale UTC-5) → would wrongly queue | **sent** |

Also assert C-3's `nextWindowOpen` across fall-back: a message queued at
`2026-11-01T04:00:00Z` (00:00 EDT) releases at `2026-11-01T13:00:00Z` (08:00 **EST**), not
12:00Z.

### C-6 · Zero sends before A2P approval

**Setup:** `update comms_provisioning set a2p_campaign_status='submitted' where client_id=:cid;`
**Act:** trigger one send from each engine kind: WF-1 review request (marketing), WF-2
first touch (conversational), WF-3 campaign batch (marketing), plus the STOP confirmation
path.
**Expected:** ALL of them — including transactional/conversational — produce
`status='blocked', blocked_reason='a2p_pending'` (doc 03 step 4 is a hard gate above the
kind-based exemptions; doc 06: "the send-pipeline blocks sends until approved"). Zero rows
with status `sent`. Shadow mode (doc 07) is exactly this state: engines run, activity is
logged (`review_request.blocked` etc.), nothing egresses.
**Then:** set `approved`, re-fire the same triggers → sends succeed, and the idempotency
keys are NEW keys (a blocked attempt does not poison the key space — blocked rows carry a
`:blocked:<n>` suffix or equivalent; the test pins whichever convention the pipeline
implements).

```sql
select status, blocked_reason, count(*) from messages
 where client_id=:cid group by 1,2 order by 1;
```

### C-7 · Frequency cap across engines (review request + campaign, same customer)

**Policy under test (pin it):** review requests are written with `kind='marketing'`; the
30-day cap counts marketing messages regardless of `engine`.
**Setup:** day 0 — WF-1 sends Casey a review request (delivered). Day 10 — Casey is in
campaign K's audience.
**Expected:** campaign send for Casey → `blocked/freq_cap`; `campaign_members.status='excluded'`;
the OTHER campaign members send normally. Day 31 (fake time) — re-run audience build →
Casey eligible again.
**Counter-cases:** (a) Casey replies and WF-2 answers — conversational reply is NOT blocked
by the cap; (b) a *blocked* marketing row from day 0 would NOT have consumed the cap (U-3).

### C-8 · Opt-out survives customer CSV re-import

**Setup:** Casey opted out (C-1 state). Ops re-imports the client's full customer CSV
(portal upload path), which includes Casey with fresh `last_visit_at` and a different name
casing.
**Expected:** `customers` row updated (upsert on `(client_id, phone)` per doc 04 — assert
NO duplicate customer row); `opt_outs` row untouched (same `opted_out_at`); a new campaign
audience build excludes Casey; a forced send attempt → `blocked/opt_out`. Also assert the
import cannot write any consent field that re-enables sending (imported rows get
`consent_basis='imported_list'` only where previously NULL — an import never upgrades
consent or clears an opt-out).

### C-9 · Consent basis required for reactivation

**Setup:** customer Riley `+16135550188` with `consent_basis=NULL` (e.g. created from a
scraped source).
**Expected:** WF-3 audience build excludes Riley (doc 02: reactivation only with recorded
prior relationship); forced send → `blocked` with reason `no_consent`; WF-2 conversational
reply to an *inbound* Riley lead still allowed (`lead_inbound` basis is set by the inbound
event itself — assert the event sets it).

### C-10 · Cross-client isolation of compliance state

**Setup:** second client fixture `hvac-pilot`; the same phone number `+16135550199` exists
as a customer of both. Casey opted out of `golf-pilot` only.
**Expected:** `hvac-pilot` sends to that phone still deliver (opt-outs are per-client per
doc 04 `pk (client_id, phone)`); `golf-pilot` remains blocked. Frequency cap likewise does
not leak across clients (U-3 counter-case at the SQL layer).

---

## 3. Idempotency / replay suite (per engine)

### 3.1 The canonical replay procedure (use everywhere)

Doc 02: `POST /internal/events` dedupes on `event_id`; consumers must be idempotent on it.
The generic drill, copy-runnable (`scripts/drills/r00-replay.sh` parameterizes this):

```bash
source scripts/drills/env.sh
EVENT_ID=$(uuidgen)
BODY=$(cat <<JSON
{
  "event_id": "$EVENT_ID",
  "client_id": "$CLIENT_ID",
  "type": "sale.completed",
  "occurred_at": "2026-07-13T15:00:00Z",
  "source": "manual",
  "payload": { "customer": { "name": "Replay Test", "phone": "+15005550006" }, "amount": 42.50 }
}
JSON
)

# 1st delivery
api -X POST "$BASE_URL/api/internal/events" -d "$BODY" -w "\n%{http_code}\n"
# expect: {"status":"accepted","event_id":"..."}  201

# replays: 5× the identical body (same event_id)
for i in 1 2 3 4 5; do
  api -X POST "$BASE_URL/api/internal/events" -d "$BODY" -w "\n%{http_code}\n"
done
# expect each: {"status":"duplicate","event_id":"..."}  200  — never 5xx, never 201

sleep 15   # let the worker/n8n consume

PSQL "select count(*) from events   where id='$EVENT_ID';"                     # = 1
PSQL "select count(*) from messages where idempotency_key like '$EVENT_ID:%';" # = 1
PSQL "select count(*) from activity_log where detail->>'event_id'='$EVENT_ID'
        and action like '%.sent';"                                             # = 1
```

Additionally replay at the **n8n layer**: re-forward the already-processed event to the
engine webhook directly (simulating control-plane re-delivery) — same assertions. Both
layers must independently dedupe (defense in depth: `events.id` PK at the edge,
`messages.idempotency_key` unique at the ledger).

### 3.2 Per-engine replay matrix

For each row: run §3.1 with the stated event, then run the stated assertion. Exactly-one
means the count query returns 1 after 6 total deliveries of the same `event_id`.

| ID | Engine | Event replayed | Exactly-one assertion (SQL against test/dev DB) |
|---|---|---|---|
| R-1 | WF-1A review request | `sale.completed` | `messages` where key `like '<eid>:wf-1:%'` = 1; second `sale.completed` with a NEW event_id for the same customer inside the 90d re-ask cooldown → 0 additional sends (`activity_log action='review_request.skipped'` = 1) |
| R-2 | WF-1B review response | `review.received` (same provider review id twice via double poll) | `reviews` rows for that provider id = 1 (it's the PK); `response_status` transitions once; GBP publish activity = 1. Simulate the double poll by resetting the poll cursor and re-polling — doc 03: "duplicate polls are no-ops" |
| R-3 | WF-2 first touch | `lead.created` | `conversations` for `lead_event_id='<eid>'` = 1; first-touch `messages` = 1; also the secondary dedupe: NEW event_id, same phone, within 24h (doc 03 WF-2 gate) → still 1 conversation |
| R-4 | WF-2 inbound turn | `message.received` (Twilio webhook re-delivery — replay via `sim:sms` twice with same MessageSid) | inbound `messages` rows for that sid = 1; AI replies = 1 |
| R-5 | WF-3 campaign member | re-fire campaign batch job for campaign K | per (campaign, customer): outbound `messages` = 1 (key `K:customer`); `campaign_members.status` monotonic (`queued→sent`, never back) |
| R-6 | WF-5 directory sync | `profile.updated` ×6 (same event_id) | BrightLocal push activity rows = 1; GBP hours push = 1 |
| R-7 | WF-7 syndication | `content.published` | `syndicated_posts` rows for the content item = 2 total (1 gbp + 1 facebook), not 12 |
| R-8 | WF-0 provisioning saga | `client.provisioned` (same HubSpot deal id) re-run after killing it between steps 3 and 4 | one tenant, one portal user, one Twilio subaccount, one A2P submission; completed steps skipped on resume (saga table shows each step exactly once); doc 03: "idempotent on deal ID" |

### 3.3 n8n workflow re-run after mid-flight kill → no duplicates

Drill `scripts/drills/r09-kill-resume.sh`:

1. Slow the mock: `MOCK_TWILIO_LATENCY_MS=2000` on the app.
2. Launch campaign K with 10 members (`api -X POST $BASE_URL/api/internal/... campaigns/K/start`
   or the admin UI).
3. After ~5 sends: `docker restart seekly-n8n` (hard kill of the executing workflow).
4. When n8n is back, re-trigger the batch (re-fire the campaign job — the normal recovery
   action an operator would take, no special flags).
5. Assert:

```sql
-- 0 rows = PASS: nobody got two texts
select customer_id, count(*) from messages
 where campaign_id = :K and direction='out' and status in ('sent','delivered')
 group by 1 having count(*) > 1;
-- all 10 accounted for (sent or a terminal blocked reason):
select status, count(*) from campaign_members where campaign_id=:K group by 1;
```

Run the same kill drill against WF-1 (kill during the 2h Wait) and WF-2 (kill between AI
draft and send): after resume, `messages` per idempotency key = 1.

### 3.4 Campaign resume after crash → no re-texts

Covered structurally by R-5 + §3.3, plus the crash-*between-ledger-and-Twilio* case, which
is the only dangerous window: force it with `MOCK_TWILIO_FORCE_STATUS=500` so the ledger
row exists with `status='queued'` but no send happened, then crash, then resume. Expected:
resume finds the queued row (its key exists) and follows the RETRY path of that same row —
it must NOT insert a second row, and must NOT skip the customer silently (that would be a
drop). Terminal state: exactly one row, `sent` (after mock recovers) with same key.

---

## 4. Switch tests — doc 03's safe-off table, case by case

Procedure for all: put the engine mid-flight, flip the switch OFF via the admin
switchboard (or `update workflow_config set enabled=false where client_id=:cid and module='<m>';`
— but prefer the UI so `updated_by` audit is exercised), observe, then flip back ON and
observe resume semantics. Every flip must appear in `workflow_config.updated_by/updated_at`
AND `activity_log (action='module.toggled')` — assert in each test. "Instant" = the next
run/step re-reads the switch; no deploy, no n8n restart.

| ID | Switch | Mid-flight state to set up | Expected while OFF (doc 03) | Expected on re-enable |
|---|---|---|---|---|
| S-1 | `review_automation` | `sale.completed` accepted, WF-1 in its 2h Wait; GBP poll cron active | The waiting request does NOT send when the Wait fires (switch re-checked after every persisted Wait — this is the load-bearing assertion); ledger row `blocked_reason` = module_off or activity `skipped`; **detection continues**: new review via poll still lands in `reviews` (sub-switch semantics: `review_requests` off ≠ polling off) | New sales trigger requests again; the stranded pre-flip sale does NOT retroactively send (its window passed; log shows skipped, not lost) |
| S-2 | `speed_to_lead` | Live conversation in progress + fresh `lead.created` arriving after flip | New lead: logged (`conversations` row or activity `lead.skipped`) + **owner alert sent instead** (assert the alert email/SMS to escalation contact); NO AI SMS to the lead; in-flight conversation sends no further AI turns | New leads answered < 60s again; skipped-lead backlog is NOT auto-texted (stale leads need explicit ops action) |
| S-3 | `reactivation` | Campaign K `sending`, 4/10 members sent | Batch halts at the member boundary — members 5–10 stay `queued`; campaign status `paused`; zero sends after flip timestamp | Resume sends EXACTLY members 5–10 (R-5 keys prove no re-text of 1–4) |
| S-4 | `content_engine` | A draft in `pending_approval`; cron due tonight | Cron no-ops (activity `skipped`); the in-flight draft stays parked in its status, is NOT published, NOT deleted | Calendar resumes; parked draft still there, publishable |
| S-5 | `directory_sync` | Pending `profile.updated` just emitted | No BrightLocal push, no GBP hours push; portal still shows **last-known** listings health (not blank, not "synced") | Next `profile.updated` pushes; the missed one is NOT auto-replayed (portal shows profile newer than listings — visible honesty per doc 03 WF-5) |
| S-6 | `competitor_intel` | Weekly snapshot cron due | Snapshot cron skips; `competitor_snapshots` history retained (row count unchanged, nothing deleted) | Next cron runs; month-diff handles the gap ("site unreachable/no data this period" style honesty, never fabricated diffs) |
| S-7 | `social_syndication` | Flip off, THEN emit `content.published` | Event **ignored, not queued** (`events.status='ignored'`); zero `syndicated_posts` | Re-enable, emit a NEW `content.published` → posts; the ignored old event is never retro-posted |
| S-8 | Sub-switches | `review_responses` off, `review_requests` on | New review → persisted + escalation alert for ≤3★ still fires (doc 03 step B4 is "always") but NO AI draft published; requests unaffected | Drafting resumes for NEW reviews only |

Automate S-1, S-3, S-7 as integration tests (they're pure control-plane); the rest are
drill scripts `scripts/drills/s0N-*.sh`.

---

## 5. Failure-injection drills

Each drill asserts three things: **behavior** (fallback/park/pause), **ledger truth** (no
silent drop, no duplicate), **alert** (ops notified within SLA). SLA per doc 07: failures
alert ops within **5 minutes**.

### D-1 · Twilio 500s → retry ×3 → dead-letter + alert, no duplicates

```bash
source scripts/drills/env.sh                       # scripts/drills/d01-twilio-500.sh
T0=$(date -u +%FT%TZ)
# magic number forces 500 on every attempt (§1.3)
api -X POST "$BASE_URL/api/internal/events" -d '{
  "event_id":"'$(uuidgen)'","client_id":"'$CLIENT_ID'","type":"lead.created",
  "occurred_at":"'$T0'","source":"webhook",
  "payload":{"contact":{"name":"Fail Test","phone":"+15005550500"},"form":"contact"}}'
```

**Expected:** worker/n8n logs show exactly 4 attempts (1 + 3 retries) with exponential
backoff spacing; final `messages.status='failed'`; ONE ledger row for all 4 attempts (same
idempotency key — retries update, never insert); `activity_log` row `status='failed'`;
dead-letter entry; ops alert received at `T_alert` with `T_alert - T0 < 5 min`. Then rerun
the event (replay) → still no customer message, no second ledger row.
Also run the `+15005550503` variant: 2 failures then success → final status `sent`, ONE
row, NO alert (recovered failures don't page).

### D-2 · Claude timeout → fallback template fires (< 60s promise holds)

```bash
# on the app/worker: AI_MOCK_FAULT=timeout (ENGINE_MODE=mock)
T0=$(date -u +%s)
api -X POST "$BASE_URL/api/internal/events" -d '<lead.created for +15005550006>'
# poll for the first-touch message
PSQL "select body, extract(epoch from created_at) - $T0 from messages
      where client_id='$CLIENT_ID' and direction='out' order by created_at desc limit 1;"
```

**Expected:** first-touch body == the configured fallback template ("Thanks for reaching
out — what date were you thinking?" per doc 03 WF-2), sent < 60s from event; `activity_log`
notes `ai_fallback=true`; then unset `AI_MOCK_FAULT`, sim an inbound reply → the AI answers
normally (conversation resumed with AI, not stuck on template). Repeat for WF-1B: review
reply drafting with `AI_MOCK_FAULT=timeout` → response parked as `drafted`-failed state +
ops alert, review NEVER auto-published with garbage (with `AI_MOCK_FAULT=garbage`, the
schema/quality gate rejects and parks — no publish).

### D-3 · GBP token revoked → module auto-pause + banner

**Inject:** in mock: `update connections set connection_status='active'` then set the mock
Google refresh to fail (`MOCK_GOOGLE_REFRESH=invalid_grant`) and run the connection-health
cron; in live staging: revoke the grant at <https://myaccount.google.com/permissions>.
**Expected within one health-cron cycle:**
1. `connections.connection_status='broken'` (`last_verified_at` updated).
2. `workflow_config` for dependent modules (`review_automation`, `social_syndication`,
   `directory_sync`) auto-paused with a machine-readable reason
   (`settings.paused_reason='connection:google_business'`) — doc 02 layer-2 promise.
3. Portal shows the "Reconnect Google" banner (Playwright/manual: banner visible on client
   dashboard; switchboard shows *why* the module is off, per doc 08).
4. Client + ops both alerted; NO retry-storm in logs (health cron backs off, engines gate
   on connection status instead of hammering the API).
5. Reconnect via OAuth → status `active`, modules resume (switch state restored, not left
   off silently).

### D-4 · Dead-letter alert timing (< 5 min, measured)

**Inject a poison event** (fails all retries deterministically): dev-only payload flag
honored by the workflow shell when `APP_ENV != production`:

```bash
T0=$(date -u +%FT%TZ)
api -X POST "$BASE_URL/api/internal/events" -d '{
  "event_id":"'$(uuidgen)'","client_id":"'$CLIENT_ID'","type":"sale.completed",
  "occurred_at":"'$T0'","source":"manual",
  "payload":{"__force_fail":true,"customer":{"name":"Poison","phone":"+15005550006"}}}'
# then measure:
PSQL "select received_at from events where payload->>'__force_fail'='true' order by received_at desc limit 1;"
PSQL "select created_at, detail from activity_log where action='ops.alert.sent'
      order by created_at desc limit 1;"
```

**PASS:** `alert.created_at - events.received_at < interval '5 minutes'`, and the alert
payload names client, engine, event_id, and a dead-letter/resume link. Also verify the
alert actually egressed (check the ops inbox / Slack channel — see §7 channels), not just
the log row. Run this drill monthly in prod with a synthetic client (see QA-14).

### D-5 · Email-parse low confidence → ops queue, never a wrong text

**Inject:** send a garbled/ambiguous email to the client's `{slug}@in.seekly.app` dev
inbox (fixture `src/test/fixtures/emails/garbled.eml`, replayed through the inbound-parse
webhook route).
**Expected:** zero `events` emitted; one ops-review-queue row with the raw email +
extraction attempt + confidence; ops alert if the queue is nonempty for > 1h; approving the
queued item from the admin UI emits the corrected event exactly once (then R-1 applies).

### D-6 · STOP-rate spike auto-pauses a campaign

**Setup:** campaign K sending to 40 mock members. After 10 sends, `sim:sms` STOP from 2 of
them (>3% of batch, doc 03 WF-3).
**Expected:** campaign status `paused` before the batch completes; zero sends after the
pause timestamp; ops alert with the STOP count/rate; resuming requires explicit ops action
(no auto-resume).

---

## 6. Go-live checklist per client (operationalizes doc 08 §E)

Run top to bottom for every client, no skips. Each box has a verification command; a box
is checked only when the command returns the stated PASS value. Wrap the queryable subset
as `npm run golive-check -- --client <slug>` (script iterates these same queries and
prints a table) — but the two LIVE boxes are always manual.

Set once: `CID=$(PSQL "select id from clients where business_name ilike '%<name>%';")`

| # | Box (doc 08 §E) | Verification | PASS |
|---|---|---|---|
| G-1 | EIN received → A2P submitted | `PSQL "select a2p_brand_status, a2p_campaign_status from comms_provisioning where client_id='$CID';"` | brand + campaign at least `submitted` on day 1; **`approved` required before G-12** |
| G-2 | GBP manager invite accepted / Google connected | `PSQL "select provider, connection_status, last_verified_at from connections where client_id='$CID' and provider='google_business';"` | `active`, verified < 24h ago (interim manager-access path: metadata notes `manager_access=true`) |
| G-3 | POS notification sample → parser tuned → `sale.completed` verified | `PSQL "select count(*) from events where client_id='$CID' and type='sale.completed' and source='email-parse' and status='processed';"` and zero pending low-confidence parses: `…from ops_parse_queue where client_id='$CID' and status='pending'` | ≥ 3 processed real samples, 0 pending queue items |
| G-4 | Customer CSV imported → audience segmented | `PSQL "select count(*), count(*) filter (where phone is not null), count(*) filter (where consent_basis is not null) from customers where client_id='$CID';"` | counts match the client's export ±0; every row has consent_basis; audience preview for the default lapse threshold returns > 0 |
| G-5 | Form notifications CC'd → `lead.created` verified | Submit a test lead on the client's real form → `PSQL "select count(*) from events where client_id='$CID' and type='lead.created' and received_at > now()-interval '1 hour';"` | = 1, and (in shadow mode) a `blocked/a2p_pending` or (post-A2P) a sent first touch exists for it |
| G-6 | FB admin added as Meta app tester | Graph API page fetch through the stored connection succeeds; `connections.provider='meta'` → `active` | active |
| G-7 | Website publishing path chosen | `PSQL "select website_platform from client_profile where client_id='$CID';"` + `workflow_config.settings->>'publish_target'` for `content_engine` | non-null; if `wordpress`, a draft-mode test post succeeded and was deleted |
| G-8 | Competitor list resolved → first espionage snapshot | `PSQL "select competitor_key, max(captured_at) from competitor_snapshots where client_id='$CID' group by 1;"` | every intake competitor has ≥1 snapshot < 8 days old |
| G-9 | Review link + brand voice loaded → **WF-1 shadow-mode test passed** | `PSQL "select review_link is not null, brand_voice is not null from client_profile where client_id='$CID';"`; then run §3.1 with a test `sale.completed` while A2P pending | envelope accepted, message row `blocked/a2p_pending`, body renders with real first name + real review link (read `messages.body`) — i.e., C-6 shadow behavior on THIS client's data |
| G-10 | **Compliance suite green on this client's config** | `npm run test:compliance` (fixtures parameterized by the client's timezone + quiet-hour overrides), plus one LIVE-mock C-1 STOP drill against the client tenant in dev | all green, C-1 drill PASS |
| G-11 | Monitoring live for this client | Force D-4 poison event for this tenant in staging; check digest includes the client | alert < 5 min; client appears in today's ops digest |
| G-12 | A2P approved → switches ON | `PSQL "select module, enabled from workflow_config where client_id='$CID' order by 1;"` after flipping per tier | `a2p_campaign_status='approved'` AND target modules `enabled=true` AND `activity_log` has the `module.toggled` entries with `updated_by` = a real ops user |
| G-13 | **LIVE smoke: one real SMS** | `MESSAGING_MODE=live`, send a review request to a **Seekly-owned phone** enrolled as a test customer; reply STOP; reply START | message received on the phone; STOP confirmation received; ledger shows opt-out; START re-enables |
| G-14 | **LIVE smoke: baseline captured** (feeds §8) | `PSQL "select action, created_at from activity_log where client_id='$CID' and action='baseline.captured';"` | exists, dated BEFORE the first non-test send (G-13 excluded via the test-customer tag) |

Doc 08's rule holds in the tooling: every unchecked box maps to exactly one blocked
module, and the admin switchboard must display which box blocks which module.

---

## 7. Monitoring & alerting spec

### 7.1 Channels & env vars

Add to `src/env.ts` (all optional in dev, required-by-check in prod boot when
`APP_ENV=production`):

```
OPS_ALERT_EMAIL          # e.g. ops@seekly.app  — critical alerts + daily digest (via existing RESEND_API_KEY)
OPS_SLACK_WEBHOOK_URL    # #seekly-ops channel — all alerts, including warnings
OPS_DIGEST_HOUR_UTC=12   # daily digest send hour (12Z = 7am ET)
ALERTS_ENABLED=true      # kill-switch for staging noise; MUST be true in prod (asserted at boot)
```

Severity: **CRITICAL** → Slack + email (dead-letter, failed sends, broken connections,
STOP spike, alert-pipeline self-test failure). **WARN** → Slack only (queue depth, parse
queue aging, A2P status change). Every alert body includes: client, module, count, first
occurrence, and a deep link to the admin ops queue.

### 7.2 Alert rules — the exact queries (run by a worker cron every 60s)

```sql
-- 1. FAILED SENDS (CRITICAL): any terminal send failure in the last 5 min
select client_id, engine, count(*)
from messages
where status = 'failed' and created_at > now() - interval '5 minutes'
group by 1,2;

-- 2. DEAD-LETTER DEPTH (CRITICAL when >0 for two consecutive ticks)
select client_id, type, count(*), min(received_at) as oldest
from events
where status = 'failed'
group by 1,2;

-- 3. STOP-RATE (CRITICAL): >3% per campaign (doc 03) or >1% per client per day
select m.campaign_id,
       count(*) filter (where o.phone is not null)::float / greatest(count(*),1) as stop_rate
from messages m
left join opt_outs o
  on o.client_id = m.client_id
 and o.source_message_id in (select id from messages i where i.customer_id = m.customer_id and i.direction='in')
where m.campaign_id is not null and m.direction = 'out'
group by 1 having count(*) >= 20;         -- min denominator to avoid 1-STOP false pages

-- 4. CONNECTION HEALTH (CRITICAL on transition to broken; daily re-page while broken)
select client_id, provider, connection_status, last_verified_at
from connections
where connection_status = 'broken';

-- 5. QUIET-HOURS QUEUE AGING (WARN): queued marketing older than 24h = release bug
select client_id, count(*) from messages
where status = 'queued' and created_at < now() - interval '24 hours'
group by 1;

-- 6. PARSE QUEUE AGING (WARN at 1h, CRITICAL at 4h)
select client_id, count(*), min(created_at)
from ops_parse_queue where status = 'pending' group by 1;

-- 7. A2P STATUS CHANGE (WARN; CRITICAL if 'rejected')
select client_id, a2p_campaign_status from comms_provisioning
where a2p_campaign_status in ('rejected');   -- plus change-detection on the row's updated_at
```

Alert de-dup: each rule keys an `ops_alerts` row `(rule, client_id, fingerprint)`; re-page
only on state change or every 24h while unresolved (prevents a 500-storm from paging 500
times). D-4 is the acceptance test for rule 2's latency; add a **self-test**: a daily
synthetic CRITICAL that must arrive in Slack — if it doesn't, that's its own page path via
email (watches the watcher).

### 7.3 Ops daily digest (existing daily-digest cron per doc 02, extended)

Sent at `OPS_DIGEST_HOUR_UTC` to `OPS_ALERT_EMAIL` + Slack. One section per client, one
line per module, sourced ONLY from these queries (no hand-maintained numbers):

```sql
-- per client per module, last 24h
select client_id, engine,
  count(*) filter (where direction='out' and status in ('sent','delivered'))     as sent,
  count(*) filter (where status='blocked' and blocked_reason='opt_out')          as blocked_opt_out,
  count(*) filter (where status='blocked' and blocked_reason='freq_cap')         as blocked_cap,
  count(*) filter (where status='blocked' and blocked_reason='a2p_pending')      as blocked_a2p,
  count(*) filter (where status='queued')                                        as queued_quiet,
  count(*) filter (where status='failed')                                        as failed
from messages where created_at > now() - interval '24 hours' group by 1,2;

select client_id, engine, status, count(*)                     -- run outcomes
from activity_log where created_at > now() - interval '24 hours' group by 1,2,3;
```

Plus: dead-letter depth (rule 2), new opt-outs count, connections not `active`,
A2P statuses not `approved`, parse-queue depth, campaigns by status, and yesterday's
`metrics_daily` rollup confirmation (rows present for every active client — a missing
rollup row is itself a WARN).
**Acceptance:** with a seeded day of mixed traffic in staging, the digest's numbers
exactly equal the queries run by hand (diff = 0), and a day with zero activity still sends
("all quiet" — proof of monitoring, same principle as doc 03 WF-6).

---

## 8. Case-study data validation (pilot metrics, doc 01)

The case study is only as good as its provenance (doc 05: "every number defensible from
the activity log"). Two mechanisms, both tested:

### 8.1 Baseline capture on day 0

When a client's FIRST module flips on, a `baseline.captured` job writes an
`activity_log` row + a `metrics_daily` row `module='baseline'` containing: current Google
review count + avg rating (from the first poll), listings-health score (BrightLocal first
audit, doc 03 WF-5), current AI share-of-voice (existing SoV product), lead-response
baseline `null` (they had none — that IS the baseline), customer count + lapsed count from
the CSV.

**Validation queries (must pass at G-14 and again before the case study is written):**

```sql
-- baseline exists and predates first real send
select (select min(created_at) from activity_log
         where client_id=:cid and action='baseline.captured')
     < (select min(created_at) from messages
         where client_id=:cid and direction='out' and status in ('sent','delivered')
           and customer_id not in (select id from customers where client_id=:cid and 'seekly_test' = any(tags)))
  as baseline_before_first_send;   -- must be TRUE

select metrics from metrics_daily
 where client_id=:cid and module='baseline';  -- non-null review_count, rating, listings_health
```

### 8.2 Weekly snapshots + metric provenance

Nightly `metrics_daily` rollups are the L1 layer (doc 09); the case study reads weekly
aggregates of them. Tests:

```sql
-- (a) NO GAPS: one row per active client per active module per day since activation
select d::date
from generate_series((select min(date) from metrics_daily where client_id=:cid),
                     current_date - 1, interval '1 day') d
left join metrics_daily m on m.client_id=:cid and m.module='reviews' and m.date=d::date
where m.client_id is null;                       -- 0 rows = PASS

-- (b) ROLLUPS RECONCILE WITH RAW (provenance — run for each headline metric):
-- review velocity
select (metrics->>'requests_sent')::int = (
  select count(*) from messages
   where client_id=:cid and engine='wf-1' and kind='marketing'
     and status in ('sent','delivered') and created_at::date = m.date)
from metrics_daily m where client_id=:cid and module='reviews' and date=:d;

-- median lead response time (the hero number)
select percentile_cont(0.5) within group (order by first_response_seconds)
from conversations where client_id=:cid and created_at between :t0 and :t1;
-- must equal metrics_daily leads.median_response_s for the same window (±1s rounding)

-- reactivation funnel: campaign stats jsonb must equal recomputed raw counts
select c.stats->>'sent',    (select count(*) from messages where campaign_id=c.id and status in ('sent','delivered')),
       c.stats->>'replies', (select count(distinct m.customer_id) from messages m where m.campaign_id is not null and m.direction='in' and m.client_id=c.client_id),
       c.stats->>'bookings',(select count(*) from conversations v where v.client_id=c.client_id and v.booked and v.engine='wf-3')
from campaigns c where c.id = :K;                -- each pair equal = PASS

-- before/after review velocity (the case-study chart)
select count(*) filter (where received_at <  :activation) * 1.0
         / greatest(extract(days from :activation - min(received_at))/30, 1)  as per30_before,
       count(*) filter (where received_at >= :activation) * 1.0
         / greatest(extract(days from now() - :activation)/30, 1)             as per30_after
from reviews where client_id=:cid;
```

Automate (a)+(b) as `npm run validate:case-study -- --client <slug>` (tsx script printing
a reconciliation table); it must exit 0 weekly in the pilot phase (add to the ops digest:
"case-study provenance: OK/DRIFT"). Test customers are excluded everywhere via the
`'seekly_test'` tag (G-13's phone MUST carry it — assert in the script).

---

## 9. Ticket table

Build-plan dependencies refer to doc 10's revised sequence (send-pipeline service =
doc 10 §Internal API; schema migrations = doc 10 §Schema additions).

| ID | Ticket | Deliverable | Depends on |
|---|---|---|---|
| QA-1 | Integration test harness | `vitest.integration.config.ts`, `_test` DB guard + truncation, fixtures (golf-pilot, hvac-pilot, Casey/Riley, email .eml samples), `npm run test:int` | schema migrations (build) |
| QA-2 | `MESSAGING_MODE` + mock Twilio | env enum, `messaging/registry.ts` + mock adapter w/ magic numbers + fault knobs, status-callback loop, `sim:sms` script, refactor `reviews/messaging.ts` behind it, grep-check for single Twilio call site | send-pipeline service (build) |
| QA-3 | AI mock faults | `AI_MOCK_FAULT` in the shared AI client (timeout/error/garbage), deterministic mock generations | build's shared AI client |
| QA-4 | Unit suites U-1…U-6 | pipeline decision core extracted pure + tests; quiet-hours/TZ vectors incl. DST; idempotency keys; extraction schemas + golden samples; STOP/HELP parsing | QA-1 (fixtures), pipeline code (build) |
| QA-5 | Freq-cap + opt-out int tests | U-3 SQL cases, C-8 import-survival, C-10 isolation | QA-1, QA-2 |
| QA-6 | Compliance suite C-1…C-10 | `compliance-suite.int.test.ts` + `npm run test:compliance`; wired as CI gate on `src/server/pipeline/**` | QA-2, QA-4, QA-5 |
| QA-7 | Replay suite R-1…R-8 | `r00-replay.sh` generic driver + per-engine int tests + n8n-layer replay | QA-1; each engine as it ships |
| QA-8 | Kill/resume drills | `r09-kill-resume.sh` (§3.3) + WF-1/WF-2 kill variants + §3.4 crash-window test | n8n deployed, QA-2 |
| QA-9 | Switch tests S-1…S-8 | int tests (S-1, S-3, S-7) + drill scripts, incl. audit-trail assertions | switchboard (build), QA-2 |
| QA-10 | Failure drills D-1…D-6 | `d01…d06` scripts + parse-queue fixture + poison-event dev flag in workflow shell | QA-2, QA-3, QA-11 (alerts to observe) |
| QA-11 | Alerting + digest | env vars, 7 alert rules cron, de-dup table, digest cron + reconciliation test, self-test synthetic | activity_log/messages live (build) |
| QA-12 | Go-live checker | `npm run golive-check -- --client <slug>` printing G-1…G-14 table (LIVE rows marked manual) | QA-6, QA-7, QA-11 |
| QA-13 | Case-study validation | baseline job + `validate:case-study` reconciliation script + weekly digest hook, `seekly_test` tag exclusion | metrics_daily rollups (build) |
| QA-14 | Prod drill cadence | Monthly calendar: D-4 poison in prod (synthetic tenant), D-3 in staging, restore-from-backup drill (doc 07 week-4 hardening) — runbook doc in `seekly-platform` | QA-10, QA-12 |

**Suggested order:** QA-1 → QA-2/QA-3 (parallel) → QA-4/QA-5 → QA-6 (first hard gate) →
QA-7/QA-9 as engines land → QA-11 → QA-10 → QA-8 → QA-12/QA-13 before the pilot's A2P
approval day → QA-14 ongoing.

**Definition of done for this spec:** `npm test`, `npm run test:int`,
`npm run test:compliance` all green in CI; all drill scripts exist and PASS against the
dev stack; `golive-check` passes for the golf pilot with G-13/G-14 signed off by a human;
the §0 traceability table has no empty cells.
