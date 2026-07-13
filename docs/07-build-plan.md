# 07 — Build Plan: Stack, Sequence, Costs

Builder: Matthew + Claude Code. Timeline: **first engine live for the golf pilot in ~2 weeks;
all seven engines in ~30 days.** The sequence below is honest about external approval
latencies (A2P, GBP API) — those start on day 1 because they gate everything customer-visible.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Portal + control-plane API | Next.js (App Router) on Vercel | One codebase for client portal, admin, OAuth callbacks, internal API routes |
| DB / Auth / Storage | Supabase (Postgres + RLS + magic-link auth) | Fast multi-tenant foundation; RLS = client isolation |
| Token encryption | libsodium sealed box, key in env (control plane only) | Tokens unreadable even with DB access; n8n never sees refresh tokens |
| Automation | n8n self-hosted, Docker + Postgres, on Hetzner/Railway VPS | Unlimited executions ~$15/mo; full control |
| SMS | Twilio (subaccount/client) | A2P 10DLC, programmable inbound |
| AI | Claude API (Sonnet default; Haiku for extraction/classification) | Conversations, review replies, content, espionage analysis, email-parse |
| Inbound email | SendGrid Inbound Parse (or Postmark) | Email-parse adapter rung |
| Listings | BrightLocal API | Directory sync + citation audits |
| Seekly CRM | HubSpot free | Sales pipeline + Closed Won webhook |
| Repo layout | Monorepo: `apps/portal`, `packages/db` (migrations), `workflows/` (n8n JSON exports), `docs/` | Everything versioned, n8n workflows exported to git |

## Day 1 — external approvals (all in parallel, before writing code)

1. **Twilio:** master account → pilot subaccount → buy number → submit A2P brand + campaign
   for the golf venue (need their EIN — ask today).
2. **Google:** GCP project → OAuth consent screen → request GBP API access → begin app
   verification for `business.manage`.
3. **Meta:** developer app → add pilot's FB admin as tester (dev mode works immediately).
4. **BrightLocal:** account + API key.
5. **Pilot homework:** send the golf venue the intake form (08-client-intake.md) — need POS
   name + notification-email sample, customer CSV export, GBP manager invite to Seekly's
   Google account, FB page admin, review link, competitor list, EIN.

## Week 1 — foundation + first value (no client OAuth needed yet)

- Supabase schema (04) + Next.js skeleton + auth + internal API (`/internal/events`,
  `/internal/clients/:id/context`, `/internal/activity`).
- n8n deployed; standard workflow shell (context loading, gating, logging, retries,
  dead-letter) proven with a dummy client.
- Shared **send-pipeline** sub-workflow complete (opt-outs, quiet hours, caps, ledger) —
  tested against a Twilio test number.
- Email-parse infra live (`{client}@in.seekly.app` → extraction → canonical events) tuned on
  the pilot's real POS notification sample.
- **WF-6 Competitor Espionage live** (zero client dependencies) → generate the pilot's first
  digest AND 2–3 prospect digests for outbound.
- Manual provisioning path (WF-0 lite via admin script) — pilot tenant created, config seeded
  from intake.

## Week 2 — first engines live for the pilot  ← THE 2-WEEK MILESTONE

- Portal MVP cut (05): magic-link auth, Integrations page w/ Google OAuth flow, Overview stat
  cards, Reviews feed, admin switchboard.
- **WF-1 Review Velocity live:** POS emails → `sale.completed` → review request SMS
  *(gated on A2P approval — if still pending, run in shadow mode: everything executes except
  the actual send, so the moment approval lands you flip one switch)*; review detection via
  Places API; AI replies published via GBP manager access (ops-assisted until API approval).
- **WF-2 Speed-to-Lead live for website forms:** form notification CC → instant AI SMS +
  qualification loop + owner hot-lead alerts.
- Definition of "live": real customers receiving real messages, activity log capturing
  case-study baselines.

## Weeks 3–4 — full platform

- **WF-3 Reactivation:** CSV import → segments → first win-back campaign (client approves copy
  in portal).
- **WF-4 Content Engine:** topic queue from intake → first 2 GEO posts → publish (WordPress or
  email_draft per intake).
- **WF-7 Syndication:** `content.published` → GBP + FB posts (Meta app still dev mode = fine
  for pilots).
- **WF-5 Directory Sync:** BrightLocal location + baseline audit (the before-number) → sync.
- **WF-0 full provisioning:** HubSpot Closed Won webhook end-to-end + status sync back.
- Portal: conversations view, approvals, content/competitor tabs, profile editing
  (`profile.updated` → WF-5).
- Meta App Review submission; GBP API swap-in when approved.
- Hardening pass: dead-letter drills, duplicate-event replay tests, STOP-rate auto-pause test,
  restore-from-backup drill.

## Post-30-day roadmap (v1.1+)

1. Missed-call text-back (tracking number overlay) — designed in WF-2, ships when pilots stable
2. Native POS adapters as intakes demand (Square first for golf/rec)
3. Growth Action Queue (weekly AI-prioritized actions from all engine data) + auto-execution
4. Revenue Leakage Report as automated monthly artifact (pilot conversion + sales tool)
5. Review-response approval graduation flows, referral generation, churn prediction
6. Seekly-embedded lead forms/widgets (Conversion Engine)

## Definition of "rock solid" (acceptance criteria per engine)

- Idempotent: replaying any event/webhook produces zero duplicate messages/posts (proven by test)
- Safe-off: flipping the switch mid-flight strands nothing (campaigns resumable, waits cancel)
- Compliant: STOP honored across all engines instantly; zero sends outside quiet hours or
  before A2P approval (pipeline-enforced, not engine-enforced)
- Observable: every action in `activity_log`; failures alert ops within 5 minutes; no silent drops
- Fallback-first: AI failure → template; API failure → retry → parked + alert; connection
  broken → module auto-pause with visible reason

## Monthly cost estimate (pilot phase)

| Item | Est./mo |
|---|---|
| n8n VPS | $10–20 |
| Supabase + Vercel | $0–45 |
| Twilio (number, A2P fees, ~1–2k msgs) | $30–80 |
| Claude API | $30–100 |
| BrightLocal (1–3 locations) | $30–80 |
| Inbound/outbound email | $0–20 |
| HubSpot | $0 |
| **Total** | **~$100–345/mo** |

Covered by roughly one-third of a single Core-tier retainer once the first pilot converts.
