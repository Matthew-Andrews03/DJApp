# 08 — Client Intake & Discovery Form

The intake is the product's configuration file. It captures the "10–15 strategic questions"
only the owner can answer; integrations provide everything else. Output populates
`client_profile`, `workflow_config.settings`, and the onboarding integration checklist —
and tells us which adapter rung each of their systems needs.

Format: one guided form (portal or call-assisted), ~20 minutes. Sections marked ⚙ feed
adapters; marked ✅ are required before any module can switch on.

## A. Business identity ✅

1. Legal business name, DBA, EIN (required for A2P registration — day 1 blocker)
2. Locations: address, phone, timezone (per location)
3. Hours + typical holiday exceptions
4. Website URL ⚙ (platform? WordPress/Wix/Webflow/Shopify/custom/unknown)
5. Google Business Profile: confirm ownership access; send GBP manager invite to Seekly's
   Google account ✅ (interim path while API approval pends)
6. Facebook page (admin name for tester access during Meta dev mode)

## B. The strategic questions (owner-only knowledge)

7. Describe your ideal customer and your 3 highest-margin services/offers
8. Service areas / neighborhoods you want to own (feeds keywords + content topics)
9. Top 3–5 competitors (names; we resolve websites + GBP listings) — feeds WF-6
10. Brand voice: how formal? phrases you love / never say? (3 example sentences they like)
11. What should an AI assistant NEVER promise? (pricing, availability, guarantees — guardrails
    for WF-2 conversations)
12. Qualification questions that matter when a lead comes in (e.g., golf: party size, date,
    occasion, sim vs. bay) + what makes a lead "hot"
13. Approved win-back angles: what offers can we make to lapsed customers without asking you?
    (e.g., league night invite, birthday package, % off lesson pack) — feeds WF-3
14. Review situation: current count/rating, any past review-request attempts, sensitive topics
15. What does a new customer cost you / what's an average customer worth per year? (anchors
    the revenue-recovered math in reporting)

## C. Systems inventory ⚙ (decides which connectors the client sees — drives spec 12 resolver)

15b. Website platform: which builder is your site on? (Wix / WordPress / Shopify / Squarespace /
    other / not sure) → `client_profile.website_platform`
16. POS / booking system(s): name + plan tier → `client_profile.pos_system`
    - **Do your bookings/sales run THROUGH your website (e.g. Wix Bookings), or a separate tool?**
      → `client_profile.website_handles_bookings` (this decides whether the website connector
      alone covers sales, or a separate booking connector is also needed — spec 12 rule 1/2)
    - Does it send "sale completed" / "booking confirmed" notification emails? → forward a
      sample to `{client}@in.seekly.app` (rung 3 live immediately)
    - Does it support webhooks/Zapier? (rung 2) · Does it have an API/OAuth? (rung 1 candidate —
      if enough clients use it, promote to a native adapter per spec 11)
17. Customer list export: can you export customers with phone + last visit date? (CSV template
    provided) ✅ for reactivation
18. Website forms: which forms exist, where do their notification emails go? → CC/forward to
    Seekly inbound address
19. Phone: who answers, what happens after hours? (sizes the v1.1 missed-call opportunity)
20. Any CRM/email tool in use? (dedupe + future adapter)

## D. Permissions & guardrails ✅

21. Approval preferences per module (default: review replies full-auto, campaigns approve
    first send, content approve first 3)
22. Messaging quiet hours preference (default 8am–9pm local) + daily send cap comfort
23. Escalation contacts: who gets hot-lead alerts and negative-review alerts (name, mobile)
24. Case-study consent (pilots): metrics + name usable in Seekly marketing ✅ pilot contract

## E. Access checklist generated from answers

The form output auto-builds the onboarding checklist, e.g. for the golf pilot:

- [ ] EIN received → A2P submitted (day 1)
- [ ] GBP manager invite accepted
- [ ] POS notification sample email received → parser tuned → `sale.completed` verified
- [ ] Customer CSV imported → audience segmented
- [ ] Form notifications CC'd → `lead.created` verified
- [ ] FB admin added as Meta app tester
- [ ] Website publishing path chosen (WP app-password / email_draft)
- [ ] Competitor list resolved → first espionage snapshot
- [ ] Review link + brand voice loaded → WF-1 shadow-mode test passed
- [ ] A2P approved → switches ON (WF-1, WF-2), campaigns scheduled

Every unchecked box maps to exactly one blocked module — the switchboard shows *why* a module
can't turn on yet.

---

## Pilot profile (pre-filled) — Norm's Golf & Social

First pilot. Detected from public info (site is Wix; live fetch was egress-policy blocked, so
platform inferred from Wix Bookings URL patterns `/book-online` + `/service-page/…` — confirm in
onboarding). Business: golf-simulator social clubhouse (4 Trackman iO sims + bar + leagues +
events), 616 Gardiners Road, Kingston ON. Contact `booking.norms.kingston@gmail.com`,
647-594-6676. Social: facebook.com/norms.kingston, instagram.com/norms.kingston.

Resolver inputs (spec 12):

| Field | Value |
|---|---|
| `website_platform` | `wix` |
| `website_handles_bookings` | `true` (Wix Bookings — sim sessions sold as services) |
| `pos_system` | Wix Bookings (same) |
| `uses_meta_ads` | confirm at intake (FB + IG present) |
| `has_customer_list` | Wix Contacts (via adapter backfill) |
| `enabled_modules` | all (free pilot) |

→ **Resolved connectors: `wix` (required), `google_business` (required), `meta` (recommended).**
No POS, CSV, or email-parse needed — the clean "one connection" case.

Niche config seeds: reactivation angle = **league nights / lapsed players**; content + syndication
topics = leagues, events, live music, corporate/private bookings; qualification (speed-to-lead)
= party size, date, session length, occasion. Day-1 to-dos specific to this pilot: create the
Seekly Wix App (AD-W1) + share install link; GBP manager invite; A2P registration (need Norm's
EIN/business number); add their FB/IG as Meta app testers.
