# 01 — Business Plan

## Vision

Local businesses lose money every day in ways they never see: missed calls, forms answered too
late, customers who quietly stop coming back, happy customers who never leave a review, weak
visibility in the AI engines their future customers now ask for recommendations. Seekly is a
managed AI growth platform that finds that leaking revenue and automatically fixes it.

The product promise that anchors all marketing:

> **"Seekly finds where your business is losing money — and fixes it automatically."**

Seekly is *managed* software, not a self-serve SaaS and not a traditional agency:

- Like software: clients get a portal, connected integrations, live dashboards, and automation
  that runs 24/7. Onboarding feels like installing a product, not filling out an agency
  questionnaire (10–15 strategic questions; integrations provide everything else).
- Like an agency: Seekly configures, monitors, and tunes everything. The client never touches
  n8n, prompts, or settings. Modules are enabled per client by Seekly with the flick of a switch.

## Product structure (marketing frame → engines)

Publicly, Seekly is sold as six engines. Internally they map to seven master workflows plus
shared infrastructure:

| Marketing engine | Promise | Internal workflows |
|---|---|---|
| Visibility Engine | Get found (Google + AI engines) | SEO/GEO Content, Directory Sync, Social Syndication |
| Conversion Engine | Turn attention into leads | Seekly forms/widgets (roadmap), landing pages (roadmap) |
| Sales Engine | Turn leads into revenue | Speed-to-Lead (+ missed-call recovery in v1.1) |
| Reputation Engine | Build trust | Review Velocity |
| Customer Growth Engine | Increase lifetime value | Database Reactivation |
| Intelligence Engine | Run the growth system | Competitor Espionage, Growth Action Queue (roadmap), Revenue Leakage Report |

**Revenue Leakage Detection** is the commercial wedge: an automated audit that quantifies money
already being lost (missed calls, un-contacted forms, lapsed customers, unreviewed happy
customers, competitors outpacing them). It is both the sales tool (run it on prospects) and the
recurring proof of value (monthly report shows what Seekly recovered).

## Ideal customer profile

- Local service/experience businesses with repeat-visit economics and bookable inventory.
- Beachhead: **recreation venues** (golf sims, ranges, entertainment venues) — first pilots are
  in this niche. Home services (HVAC, plumbing, landscaping) is the designed second niche.
- The platform stays **niche-agnostic by architecture**: OAuth integrations and canonical events
  cover ~80% of local businesses regardless of vertical; niche shows up only in configuration
  (qualification questions, campaign angles, content topics), never in code.

## Pilot strategy (now → ~90 days)

- 1–3 pilot clients, starting with the golf/recreation venue. **Free for 60–90 days** in
  exchange for: written case-study rights, before/after metrics, a testimonial, and being
  reference customers.
- Seekly eats all pilot COGS including BrightLocal per-location fees (~$30–80/mo total) —
  the listings-health before/after strengthens the case study.
- Success metrics captured automatically by the activity log from day one:
  - Review count + rating before/after; review velocity per month
  - Median lead response time (target: < 60 seconds vs. industry hours)
  - Leads contacted / qualified / booked, by source
  - Reactivation campaign: messages sent → replies → bookings → estimated revenue
  - Rankings + AI share of voice on target queries; listings consistency score
- Exit criteria for a pilot → paid conversion: demonstrable recovered revenue > 3× the retainer.

## Offer & pricing (post-pilot)

Suggested tiers (validate against pilot results; all per location per month):

| Tier | Price | Modules |
|---|---|---|
| Core | $497 | Review Velocity + Speed-to-Lead + monthly leakage report |
| Growth | $997 | Core + Reactivation + Content Engine + Social Syndication |
| Market Leader | $1,997 | Growth + Directory Sync + Competitor Espionage + priority support |

- One-time setup fee ($250–750) covers onboarding, A2P registration, and integration work.
- Anchor pricing to recovered revenue ("one saved booking a week pays for Core"), not deliverables.
- Per-module COGS stays low (SMS, AI tokens, BrightLocal) — target 80%+ gross margin at Growth tier.

## Seekly's own sales operation

Separation of concerns (never mix these datasets):

- **Seekly's CRM (HubSpot, free tier to start):** prospects → outreach → conversations →
  meetings → proposals → deals → Closed Won. Company records carry business name, website,
  industry, locations, estimated opportunity, lead source, prospect score. Deals carry pipeline
  stage, proposed package, monthly retainer, setup fee, expected close.
- **Cold outbound stays out of HubSpot:** prospect database + enrichment + AI research +
  personalized outreach in a dedicated outbound tool; only positive responses sync into HubSpot.
- **Seekly control plane:** owns the *service* relationship (tenants, connections, workflow
  config, client customer data). HubSpot receives only summary fields back: onboarding status,
  account status, health score, MRR, renewal date. Bidirectional event sync, never database
  mirroring.
- **Closed Won → automatic provisioning:** HubSpot webhook → n8n creates tenant, portal user,
  onboarding record, subscription, integration checklist → welcome email → Seekly team notified.
  Selling a client and turning them on is one motion.

### Outbound lead magnet

The Competitor Espionage engine doubles as the outreach hook: run it against a prospect's
competitors, send the digest ("your competitor gained 14 reviews last month and launched a $25
league night — here's what that costs you"), book the meeting from evidence, not claims.

## Moat & compounding advantages

1. **Cross-client intelligence:** every client makes the prompts, qualification flows, and
   campaign angles better for the niche; new clients onboard with proven playbooks.
2. **Switchboard economics:** marginal cost of activating a module for a new client approaches
   zero; price is anchored to value, not effort.
3. **Data ownership:** activity log + attribution gives Seekly the receipts (recovered revenue)
   that generic agencies can't produce.
4. **GEO head start:** structured, entity-consistent content pipelines position clients in AI
   answers while competitors still optimize only for the ten blue links.
