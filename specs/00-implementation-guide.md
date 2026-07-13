# 00 — Implementation Guide (read this first)

This directory turns the planning docs (`/docs`) into junior-dev-implementable work. Every
spec has a ticket table; this guide gives the cross-spec build order, precedence rules, and
the consolidated open-decisions list.

## Spec index & ticket prefixes

| Spec | Prefix | Scope |
|---|---|---|
| [01-database-migrations.md](01-database-migrations.md) | `DB-` | All Drizzle schema additions/alterations, migration order, seeds |
| [02-internal-api.md](02-internal-api.md) | `API-` | Service auth, `/api/internal/events` + context + activity + send contracts |
| [03-send-pipeline.md](03-send-pipeline.md) | `SP-` | The single SMS compliance path, Twilio/A2P, inbound STOP/HELP |
| [04-oauth-integration-hub.md](04-oauth-integration-hub.md) | `OA-` | Google/Meta OAuth flows, token vault, health cron, WordPress connect |
| [05-email-parse-ingestion.md](05-email-parse-ingestion.md) | `EP-` | Inbound email → AI extraction → canonical events; CSV import |
| [06-n8n-workflows.md](06-n8n-workflows.md) | `N8N-` | n8n deployment + node-by-node builds for WF-0…WF-7, prompt templates |
| [07-portal-pages.md](07-portal-pages.md) | `UI-` | Integrations, Growth, Approvals, Overview, Settings, admin switchboard |
| [08-intelligence-jobs.md](08-intelligence-jobs.md) | `INT-` | pg-boss rollups, anomalies, insights, briefs, WF-8 action queue |
| [09-provisioning-hubspot.md](09-provisioning-hubspot.md) | `PR-` | HubSpot setup, Closed Won → provision saga, status sync-back |
| [10-testing-acceptance.md](10-testing-acceptance.md) | `QA-` | Compliance/replay/switch/drill suites, go-live checklist, alerting |
| [11-native-adapters.md](11-native-adapters.md) | `AD-` | Native OAuth adapters; **Wix is Day-1/Wave-1** (pilot runs on Wix), shared adapter framework, coverage/wave matrix, vertical map |
| [12-connector-resolution-onboarding.md](12-connector-resolution-onboarding.md) | `CN-` | Per-client connector resolution + tailored onboarding — each client sees ONLY the 2–4 connectors they need, not the full catalog |

## Precedence rules (when documents disagree)

1. `docs/10` supersedes `docs/02/04/05/07` (the platform extends seekly-client-insights;
   no Supabase).
2. **Spec 01 wins all schema questions** — table/column/enum names in other specs defer to
   spec 01; reconcile before running `drizzle-kit generate` (see R-1 below).
3. **Spec 02 wins all internal-API wire questions** — snake_case wire format, envelope shape,
   error codes.
4. **Spec 03 owns send semantics** — kind table (review requests = `marketing`), gate order.
5. QA spec's pinned policies (§ below) are binding unless the founder overrules them.

## Build order (maps to docs/07 + docs/10 sequencing)

**Phase 0 — Day 1, mostly human (do these before/while coding):**
external approvals — Twilio A2P (SP-2 console path), GBP API application + OAuth consent
(OA GCP setup steps), Meta dev app + pilot tester, BrightLocal account, HubSpot portal setup
(PR-1, PR-2), SendGrid inbound domain (EP-1 DNS), **Seekly Wix App creation (AD-W1) — the pilot
runs on Wix, so file this Day-1**, pilot intake form sent (docs/08).

**Phase 1 — Week 1 (foundation, no client OAuth needed):**
`DB-*` → `API-1..4` (auth, events, context, activity) → `SP-*` (send pipeline + inbound) →
`EP-*` (email parse + CSV) → `N8N-1..3` (deploy, conventions, error workflow) → WF-6
espionage build (needs only internal API) → `PR-5` manual pilot provisioning.
Gate: QA idempotency/replay suite green on events + sends.

**Phase 2 — Week 2 (pilot live milestone):**
`AD-1` adapter framework + **`AD-W2/W3` Wix connect + webhooks** (the pilot's primary
integration — one connect lights up WF-1/WF-2/WF-3/WF-4/WF-7) → `OA-*` Google flow + `UI-1`
Integrations page → WF-1 Review Velocity (fed by Wix Bookings/eCom `sale.completed`; shadow mode
until A2P approval; interim GBP manager-access mode until API approval) → WF-2 Speed-to-Lead
(Wix Forms `lead.created`) → portal MVP cut (Overview cards, Reviews feed, admin switchboard).
`AD-W4` (Wix contacts backfill → WF-3) and `AD-W5` (Wix Blog publish → WF-4/WF-7) follow in
weeks 3–4 or immediately if the pilot prioritizes reactivation/content.
Gate: QA compliance suite (C-1..C-10) green BEFORE first real send; go-live checklist for pilot.

**Phase 3 — Weeks 3–4 (full platform):**
WF-3 reactivation (CSV audience) → WF-4 content + WF-7 syndication → WF-5 directory sync →
`UI-*` Growth/Approvals/Settings → `PR-3/4/6` HubSpot end-to-end → Meta App Review submission
→ QA switch tests + failure drills.

**Phase 4 — Weeks 5–8 (intelligence):**
`INT-1..10` phase in per spec 08 (rollups first — they only need activity data the engines
are already writing).

## Consolidated reconciliation items (R-#) — resolve before the affected ticket

| # | Item | Resolve during |
|---|---|---|
| R-1 | Column-name drift between specs written in parallel (e.g. `mrr_cents` vs `mrr`, `provisioning_runs`/`onboarding_tasks`/`ops_review_queue` definitions, `client_profile` nullability). Spec 01 wins; patch specs 04/05/09 references while generating migrations | DB-1 |
| R-2 | **Service-token env var named differently across specs** (`N8N_SERVICE_TOKEN` in 02, `SPINE_SERVICE_TOKEN` reuse in 03, `INTERNAL_SERVICE_TOKEN` in 09) — unify to `N8N_SERVICE_TOKEN` + keep one server-side `requireServiceToken` | API-1 |
| R-3 | `emitCanonicalEvent` signature assumed by specs 03/05 — verify against spec 02's envelope | API-2 |
| R-4 | `review.received` payload must echo `response_status` for WF-1 replay guard | API-2 / N8N-6 |
| R-5 | Event auto-replay cron for `pending/failed` events (recommended) | API follow-up, Phase 3 |
| R-6 | Meta 60-day page-token re-acquisition cron — home is OA health cron | OA health ticket |
| R-7 | BrightLocal endpoint paths + Meta Graph API version pin — confirm against current docs at build time | N8N-11/12 |
| R-8 | HubSpot Closed Won stage ID if a custom pipeline is used instead of renamed default | PR-1/PR-3 |
| R-9 | `insights.embedding` ships as jsonb; pgvector upgrade deferred | INT phase |
| R-10 | Health Score syncs as null until INT rollups exist | PR-6 |
| R-11 | `connections.provider` enum must add `wix` (Wave 1) + the other adapter providers as waves open (list in spec 11) | DB-1 / AD-1 |
| R-12 | Extend `onboarding_tasks` with `requirement` + `unlocks_modules`; ensure `client_profile` has `website_platform`/`pos_system`/`uses_meta_ads` for the resolver (spec 12) | DB-1 / CN-3 |

## Pinned policy decisions (QA spec §0 enforces these — founder may overrule)

1. Review-request SMS counts as `marketing` → subject to the 30-day cross-engine frequency cap.
2. A2P gate blocks ALL message kinds until campaign approval; blocked sends are re-fired
   manually at go-live, not auto-released.
3. STOP matching = single-word-message match (Twilio Advanced Opt-Out compatible).
4. Quiet-hours timezone source = `clients.timezone` (supersedes any country→TZ heuristic).
5. Twilio subaccount auth token stored in `comms_provisioning` (not vault) for v1 — flagged
   follow-up to move it (SP ticket note).
6. CSV import does NOT emit per-row `customer.imported` events (bulk upsert only; rationale
   in spec 05).

## Conventions recap

- TS camelCase / DB snake_case / wire snake_case; zod everywhere at boundaries.
- Every mutation idempotent: events on `event_id`, sends on `idempotency_key`, provisioning
  on `hubspot_deal_id` step ledger, reviews on provider review id.
- n8n never holds refresh tokens, never calls Twilio, always starts runs with
  context-fetch + switch gate, always writes activity.
- All new UI reuses existing shadcn/radix components and query patterns from the app.
- Workflow JSON exports live in `workflows/` in this repo — export after every n8n change.
