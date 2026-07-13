# 10 — Reusing seekly-client-insights as the Platform & Portal

**Status: this doc supersedes docs 02/04/05/07 wherever they conflict.** Those docs were
written assuming a greenfield build (Next.js + Supabase + new portal). Reality:
`seekly-client-insights` is a mature Next.js 16 + Drizzle + Postgres + pg-boss application
that already implements large parts of the platform. The plan changes from "build a portal"
to "extend the product that exists."

## What already exists (verified in the codebase)

| Planned component | Status in seekly-client-insights |
|---|---|
| **Visibility Engine — AI Share of Voice** | ✅ **Fully built, beyond plan**: engine adapters (ChatGPT, AI Overview/AI Mode, Gemini, Perplexity, Claude), tracked prompts, scheduled runs, mention detection, sentiment, citations, daily aggregates, 360 Insights (crawled→cited→clicked), crawler logs, visitor analytics, conversion events, agent-readiness checks, monitors |
| **Client portal** | ✅ Built: iron-session auth (login/set-password), dashboard, prompts, sentiment, citations, competitors, traffic, review, messages, documents, credentials pages |
| **Review Velocity — Part A (requests)** | 🟡 Partially built: `reviewContacts`, `reviewRequests` (SMS/email channels, feedback tokens, public funnel page), `reviewOptOuts`, Twilio send + inbound SMS route, `reviewInsights` |
| **Prospect mode** | ✅ Built: `(app)/prospecting` |
| **Reporting** | ✅ Built: PDF/DOCX generation, report schedules |
| **Job infrastructure** | ✅ Built: pg-boss worker, hourly scheduler, per-client cost guards |
| **Admin area** | ✅ Built: `(app)` per-client admin views |
| **Content Agent / Actions / Monitors** | ✅ Per roadmap: content gap/create/optimize, actions, monitor events |
| **Intelligence direction** | 🟡 Roadmap already targets revenue attribution, volatility radar, causality — aligns with doc 09 |

Not present (the growth-engine gap n8n fills): canonical event ingestion, OAuth integration
hub (GBP/Meta connections + token vault for OAuth), speed-to-lead conversations, reactivation
campaigns, directory sync, competitor *espionage* (offers/pricing diffs — SoV competitor
tracking exists), HubSpot provisioning, generalized send-pipeline, workflow switchboard.

## Revised architecture decisions

1. **seekly-client-insights IS the control plane + portal.** No Supabase, no second app. The
   control plane responsibilities from doc 02 (tenancy, client profile, workflow config,
   connections/token vault, canonical events, activity log) become schema + API routes here.
   It already owns `clients`, `clientUsers`, credentials, audit logs, and a worker.
2. **Stack corrections to earlier docs:** Drizzle + Postgres (existing) replaces Supabase;
   iron-session (existing) replaces Supabase Auth; the internal API for n8n is Next.js route
   handlers with a service token; RLS-by-`client_id` becomes what it already is here —
   query-layer scoping.
3. **Two execution planes, clear ownership:**
   - **pg-boss (in-app):** everything already built — SoV pipeline, crawler/visitor ingestion,
     reports, monitors — plus future intelligence jobs (doc 09 rollups/insights/briefs) since
     they're pure DB+LLM work living next to the data.
   - **n8n (growth engines):** WF-0 provisioning, WF-1 (extend the existing review-request
     module per below), WF-2 speed-to-lead, WF-3 reactivation, WF-5 directory sync, WF-6
     espionage, WF-7 syndication — the integration-heavy, per-client-configured automations.
   - Rule of thumb: **data products in pg-boss, outreach/integration automations in n8n.**
4. **Review Velocity split:** keep and extend the in-app request funnel (contacts, tokens,
   opt-outs, Twilio inbound) — it becomes the shared send-pipeline's first citizen. n8n adds
   the missing halves: POS-triggered requests via canonical `sale.completed`, GBP review
   polling → `review.received`, AI response generation + publishing.
5. **Credentials page evolves into the Integration Hub:** today `clientCredentials` stores
   free-text/labelled credentials. Add proper OAuth connections (doc 02 token vault: encrypted
   tokens, provider, scopes, status, health monitor) alongside; keep the legacy table for
   non-OAuth secrets. The existing `src/server/vault` module is the natural home for token
   encryption.

## Portal changes to account for the design (doc 05 IA → this app)

Existing portal nav (dashboard, prompts, sentiment, citations, competitors, traffic, review,
messages, documents, credentials) maps to the Visibility + Reputation slices of the target IA.
Additions, in build order:

1. **Integrations** — replace/extend credentials page: Connect Google (GBP+GA4+GSC), Connect
   Facebook, POS adapter status (email-parse address, CSV upload), connection health banners
2. **Growth** — conversations list (speed-to-lead transcripts, response-time hero stat),
   reactivation campaigns + results, revenue-leakage report
3. **Approvals** — unified queue (review responses, GBP posts, content, campaigns) with
   one-tap mobile approve; SMS/email deep links
4. **Overview upgrade** — current dashboard is SoV-centric; becomes the doc 05 Overview
   (growth score, revenue recovered, opportunities feed, cross-engine activity) once
   `activity_log` + `metrics_daily` exist
5. **Settings/Profile** — NAP, hours, brand voice editing → emits `profile.updated` (feeds
   WF-5 directory sync)
6. **Admin switchboard** — in `(app)`: per-client module toggles + settings editor over
   `workflow_config`, ops queues (dead-letter, low-confidence parses, failed sends)

Design-language note: new pages follow the app's existing component system (shadcn/radix +
tailwind, recharts, existing card/table patterns) — the plan adapts to the app's design, not
the reverse.

## Schema additions (Drizzle migrations, from docs 04 + 09)

New tables: `client_profile`, `workflow_config`, `connections` (OAuth vault),
`comms_provisioning`, `events` (canonical), `customers`, `opt_outs` (generalize
`reviewOptOuts` to cross-engine), `messages` ledger (generalize review SMS sends),
`conversations`, `campaigns` + `campaign_members`, `content_items`, `syndicated_posts`,
`competitor_snapshots`, `intel_digests`, `activity_log`, `metrics_daily`, and doc 09's
`insights`, `client_briefs`, `experiments`, `playbooks`, `anomalies`.

Reuse instead of duplicating: `clients` (add tier/status/hubspot fields), `clientUsers`,
`reviews`/`reviewRequests`/`reviewContacts` (extend with POS-trigger source + response
fields), `portalAuditLog`, `dailyAggregates` (SoV) alongside new `metrics_daily` (growth).

## Internal API for n8n (new route handlers)

- `POST /api/internal/events` — canonical event ingestion (signed; idempotent on `event_id`)
- `GET /api/internal/clients/:id/context?module=` — profile + module config + short-lived
  provider access tokens
- `POST /api/internal/activity` — activity log writes
- `POST /api/internal/send` — the send-pipeline as a service (opt-outs, quiet hours, caps,
  ledger, Twilio) so n8n engines never talk to Twilio directly and in-app review requests and
  n8n campaigns share one compliance path

## Repo layout going forward

| Repo | Role |
|---|---|
| `seekly-client-insights` | Platform: portal + control plane + SoV product + intelligence jobs |
| `seekly-platform` | Planning docs (this repo), n8n workflow JSON exports, infra notes |
| `seekly-agents` | (to review) — likely AI agent experiments; reconcile with WF-2/WF-3 conversation agents |
| `Seekly-Website` | Marketing site |

## Revised near-term sequence (replaces doc 07 weeks 1–2 detail)

1. Day 1 unchanged: A2P + GBP API applications, Meta app, intake to pilot ← still the
   critical path
2. Schema migrations above + internal API routes + send-pipeline service (generalizing the
   existing review SMS code)
3. n8n deployed; WF-6 espionage live (needs only `competitor_snapshots` + digests)
4. Integration Hub: Google OAuth flow into the vault; email-parse inbound → `events`
5. WF-1 completed end-to-end on the pilot (POS email → request → review poll → AI response)
6. WF-2 speed-to-lead; portal Growth + Approvals pages
