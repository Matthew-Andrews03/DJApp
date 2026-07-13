# Seekly — Managed AI Growth Platform

Seekly is a productized, managed AI growth agency for local businesses. Clients connect their
Google Business Profile, website, social pages, and POS/booking system once; Seekly's automation
engines then run their local growth on autopilot — visibility, lead response, reputation,
reactivation, and competitive intelligence — with every module toggleable per client with the
flick of a switch.

**Positioning:** "Seekly finds where your business is losing money and automatically fixes it."

## The seven engines

| # | Engine | Switch | What it does |
|---|--------|--------|--------------|
| 1 | Review Velocity | `review_automation` | Sale completes → SMS review request → AI replies to reviews with local keywords |
| 2 | Speed-to-Lead | `speed_to_lead` | Form fill / lead ad → AI texts back in seconds, qualifies, books |
| 3 | Database Reactivation | `reactivation` | Lapsed customers → personalized AI win-back campaigns |
| 4 | SEO/GEO Content | `content_engine` | Schema-rich, AI-citable blog content tailored to target niches |
| 5 | Directory Sync | `directory_sync` | NAP + hours synced across directories via BrightLocal |
| 6 | Competitor Espionage | `competitor_intel` | Monthly AI deep-dive on competitor pricing, offers, review spikes |
| 7 | Social Syndication | `social_syndication` | Blogs auto-cut into GEO-optimized GBP posts + Facebook updates |
| 8 | Intelligence / Growth Action Queue | `intelligence` | Layered memory learns what drives each client's revenue; weekly ranked actions |

All engines are **multi-tenant master workflows** in self-hosted n8n. One copy of each workflow
serves every client, keyed by `client_id`, reading per-client config and toggles from the Seekly
control plane at runtime. Client systems connect through a **canonical event layer** — workflows
never integrate with client software directly.

## Documentation

| Doc | Contents |
|-----|----------|
| [docs/01-business-plan.md](docs/01-business-plan.md) | Vision, ICP, offer & pricing, pilot strategy, case-study plan, sales stack |
| [docs/02-architecture.md](docs/02-architecture.md) | Three-layer architecture, canonical events, adapter ladder, multi-tenancy, security |
| [docs/03-workflows.md](docs/03-workflows.md) | Full specs for all 8 master workflows (7 engines + provisioning) |
| [docs/04-data-model.md](docs/04-data-model.md) | Control-plane database schema |
| [docs/05-portal-spec.md](docs/05-portal-spec.md) | Client portal MVP spec + admin switchboard |
| [docs/06-integrations.md](docs/06-integrations.md) | OAuth apps, GBP, Meta, Twilio/A2P, BrightLocal, POS adapter ladder |
| [docs/07-build-plan.md](docs/07-build-plan.md) | Tech stack, 30-day build sequence, 2-week milestone, day-1 critical path, costs |
| [docs/08-client-intake.md](docs/08-client-intake.md) | Intake/discovery form that drives per-client adapters and config |
| [docs/09-intelligence-platform.md](docs/09-intelligence-platform.md) | Layered memory architecture, experiment ledger, Growth Action Queue, niche playbooks |

## Current status

- **Stage:** planning complete → build starting
- **Pilots:** golf/recreation venue(s), free in exchange for case-study rights
- **Target:** first engine live for pilot in ~2 weeks; all seven engines in ~30 days
- **Assets:** brand + domain secured; everything else built per [docs/07-build-plan.md](docs/07-build-plan.md)

## Day-1 critical path (external approvals — start immediately)

1. **Twilio A2P 10DLC** brand + campaign registration (days to weeks of vetting)
2. **Google Business Profile API** access application for the Seekly GCP project (up to 2+ weeks)
3. **Meta developer app** for Lead Ads + Pages (app review for public use; pilots can run in dev mode)
4. **BrightLocal** account + API key

See [docs/07-build-plan.md](docs/07-build-plan.md) for the full sequence.
