# 06 — Integrations & External Services

Principle: **one Seekly OAuth app per provider**, clients authorize through it. Generic paths
(webhook / email-parse / CSV) make any client connectable on day one; native adapters are added
per client, per the intake form, when available. Target: the OAuth + generic stack covers ~80%
of local businesses regardless of niche.

## Google (one consent, three products)

**Seekly GCP project** with OAuth consent screen (external, verified) requesting:
- `https://www.googleapis.com/auth/business.manage` — GBP: reviews, replies, posts, hours
- Search Console readonly — GSC metrics for content engine
- Analytics readonly (GA4) — visibility dashboards

**Flow:** Client → "Connect Google" → consent ("Seekly wants permission to manage your
Business Profile") → callback → encrypted refresh token → fetch GBP accounts/locations →
client selects location → save to `connections` + `locations.gbp_location_id` → dependent
modules activatable.

**⚠️ GBP API requires a formal access application** (request form per project) and OAuth app
verification for sensitive scopes — timelines are days-to-weeks and outside our control.
**Submit on day 1.** Interim fallbacks that keep pilots moving:
- Reviews *read*: Places API (top reviews) — enough to detect and draft.
- Review *replies* + posts: Seekly ops publishes from the approval queue by hand (client adds
  Seekly's Google account as a **GBP Manager** — standard agency practice, no API needed).
- Hours updates: manual via manager access.
The workflows are written against our internal interface, so swapping fallback → API is a
config change, not a rebuild.

## Meta (Facebook/Instagram)

**Seekly Meta developer app** with `leads_retrieval`, `pages_manage_posts`, `pages_show_list`,
`pages_read_engagement`.
- Lead Ads: page subscribed to leadgen webhook → `lead.created` within seconds.
- Pages posting: WF-7 Facebook updates.
- Pilots can run while the app is in dev mode (pilot admins added as testers); **submit App
  Review early** for production use by arbitrary clients.

## Twilio (all SMS)

- Master account → **subaccount per client** (isolation, per-client cost tracking) → one local
  number per client (used for review requests, lead conversations, campaigns; becomes the
  tracking number in v1.1).
- **A2P 10DLC is a hard gate for every client:** register Seekly-managed Brand (client's EIN;
  sole-prop path if none) + Campaign (mixed marketing/customer-care use case). Vetting takes
  days→weeks; fees are small but real (one-time brand/campaign vetting + monthly campaign fee).
  WF-0 submits registration at provisioning; the send-pipeline blocks sends until
  `a2p_campaign_status = approved`. **This is the #1 schedule risk for the 2-week pilot —
  submit the pilot's registration immediately, before the portal exists if necessary.**
- Inbound: all replies webhook → control plane → `message.received` → conversation router.
  STOP/HELP handled at the pipeline level.

## BrightLocal

- Seekly account + API key; per-location billing (~$20–40/mo each, absorbed for pilots).
- Used for: location creation, citation audit (baseline + quarterly), listings sync across
  its directory/aggregator network, NAP consistency reporting.
- GBP-direct changes (hours/holidays) go through our own Google OAuth for speed; BrightLocal
  handles the long tail of directories.

## Client website (content publishing)

No universal "Connect Website" OAuth exists — platform decided per intake:

| Platform | Method |
|---|---|
| **Wix** | **Native OAuth (Day-1/Wave-1) — one "Connect Wix" also covers bookings, forms, and contacts. See [specs/11-native-adapters.md](../specs/11-native-adapters.md).** |
| WordPress | REST API + Application Password (client creates a Seekly editor user) |
| Webflow / Shopify | Native OAuth adapters (Wave 2, spec 11) |
| Squarespace | No content-publish API → `email_draft` (20% bucket) |
| Custom / unknown | `email_draft` fallback: HTML + instructions to their webmaster |

`email_draft` keeps the content engine universal on day one; native publishing is an upgrade
per client. **The first pilot runs on Wix, so the Wix adapter is built Day-1** — for that client
Wix is not just a publishing target but the primary source of `sale.completed` (Wix
Bookings/eCom), `lead.created` (Wix Forms), and the reactivation audience (Wix Contacts),
covering engines WF-1/WF-2/WF-3/WF-4/WF-7 from a single connection.

## POS / booking systems (the adapter ladder in practice)

v1 for the golf pilot: **email-parse** — venue forwards/CCs its POS "sale/booking completed"
notification emails to `{client}@in.seekly.app`; AI extraction normalizes to `sale.completed`
(confidence threshold; low-confidence → ops queue, never a wrong text). Plus **CSV upload** of
customer history for reactivation.

Native adapters (built when an intake names them, in likely order of demand):

| System | Niche | Hooks |
|---|---|---|
| **Wix (Day-1)** | golf/rec, any Wix site | Bookings *Booking Confirmed* + eCom *Order Paid* → `sale.completed`; Forms → `lead.created`; Contacts → `customer.imported` |
| Square | rec/retail/golf | Payments/Orders webhooks → `sale.completed`; Customers API → `customer.imported` |
| Skedda / booking tools | golf sims, venues | Booking webhooks/Zapier bridge → `sale.completed` |
| Mindbody | fitness/rec | Webhooks API |
| Jobber / Housecall Pro / ServiceTitan | home services (niche #2) | Job-completed webhooks, customer sync |
| Calendly / Acuity | services | Booking created/completed |

Each adapter is only: auth + payload mapping → canonical event POST. Engines never change.

## Email-parse infrastructure

- Inbound domain `in.seekly.app` on an inbound-email service (SendGrid Inbound Parse /
  Postmark / Mailgun) → webhook → control plane → AI extraction (per-template few-shot,
  improves per client) → canonical event.
- Also powers "their existing website form" lead capture: client sets their form
  notifications to CC the Seekly address → `lead.created`. Works on any form stack with zero
  site changes.

## HubSpot (Seekly's own)

- Free tier. Private app token for API; workflow webhook on deal-stage → `Closed Won` fires
  WF-0.
- Receives back only: onboarding status, account status, health score, MRR, renewal date.

## Seekly-owned service accounts summary

| Service | Purpose | When |
|---|---|---|
| GCP project (OAuth + GBP/GSC/GA4 APIs) | client Google connections | Day 1 (application!) |
| Meta developer app | lead ads + pages | Day 1 (dev mode), review by week 3 |
| Twilio master + subaccounts | all SMS | Day 1 (A2P!) |
| BrightLocal | listings | Week 2–3 |
| SendGrid/Postmark inbound + outbound | email-parse, digests, alerts | Week 1 |
| Anthropic API (Claude) | all AI: conversations, reviews, content, extraction, intel | Week 1 |
| Supabase / Vercel / VPS (n8n) | platform | Week 1 |
| HubSpot | Seekly sales | Week 1 (free) |
