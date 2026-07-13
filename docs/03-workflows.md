# 03 — Master Workflow Specifications

Eight multi-tenant n8n workflows. Every one follows the standard shape:

```
Trigger → resolve client_id → load client context (profile, module config,
switches, tokens) → GATE: module switch on? → execute → log activity
→ on error: retry ×3 (exp. backoff) → dead-letter + ops alert
```

Conventions used below:
- **Switch** — the per-client boolean in `workflow_config` checked at runtime.
- **Config** — per-client settings (with defaults) read from the control plane.
- **Guardrails** — all SMS passes through the shared send-pipeline: opt-out ledger check →
  quiet hours → frequency cap → message ledger write → Twilio send → delivery status update.
  No engine ever calls Twilio directly.

---

## WF-0 · Client Provisioning ("the flick of the switch")

**Purpose:** turn a closed deal into a running client with zero manual setup.
**Trigger:** HubSpot webhook — deal moves to `Closed Won`. (Also manually invocable for pilots
that never went through HubSpot.)

**Flow:**
1. Receive deal payload → extract company, contact, package/tier, retainer, locations.
2. Generate `client_id` (UUID) → create tenant row + business profile skeleton.
3. Create portal user account → send magic-link invite ("Set up your Seekly account").
4. Create onboarding record with integration checklist derived from tier + intake answers
   (which OAuth connections and adapters this client needs).
5. Set `workflow_config` defaults for the purchased tier (all switches OFF until onboarding
   completes — modules flip on individually as their dependencies connect).
6. Provision comms: Twilio subaccount + phone number purchase + A2P brand/campaign submission.
7. Notify Seekly ops (email/Slack) with the checklist.
8. Sync back to HubSpot: `Onboarding Status`, `Account Status`, later `Health Score`, `MRR`,
   `Renewal Date` — summary fields only, never operational data.

**Failure handling:** provisioning is a saga — each step records completion; re-runs skip
completed steps (idempotent on deal ID). Any step failing alerts ops with a resume link.

---

## WF-1 · Review Velocity Engine

**Purpose:** convert completed sales/visits into a steady stream of Google reviews, and answer
every review instantly with local-keyword-rich, on-brand responses.
**Switch:** `review_automation` (sub-switches: `review_requests`, `review_responses`).

**Config (defaults):** send delay after sale (`2h`), re-ask cooldown per customer (`90d`),
daily send cap (`25`), review link (GBP place ID short link), request template/tone, response
approval mode (`auto` — pilots run full-auto; per-client override to `approve_all` or
`approve_negative_only`), negative threshold (`≤3★`), target local keywords + entity references.

### Part A — Review requests
**Trigger:** canonical `sale.completed` (from POS via email-parse/webhook/native adapter, or
nightly batch import).
1. Gate: switch on, customer has phone, consent basis valid.
2. Dedupe: skip if customer messaged for a review within cooldown, or already reviewed, or
   opted out.
3. Wait configured delay (n8n Wait node, persisted).
4. Compose SMS (brand voice, first name, direct Google review link) → shared send-pipeline.
5. Log activity `review_request.sent`; link click tracked via short-link redirect.

### Part B — Review detection & response
**Trigger:** GBP poll every 15 min per connected location (API) → emits `review.received`.
1. Persist review (rating, text, author, review ID — idempotent on review ID).
2. AI drafts response: loads brand voice, location, services, target local keywords, semantic
   entity references, response rules; 1–3★ drafts are empathetic + take-it-offline, never
   defensive; no incentives language (Google policy).
3. Approval gate per config: `auto` → publish immediately via GBP API. `approve_*` → queue in
   portal/SMS approval; publish on approve.
4. **Always**, for ≤3★: fire complaint-escalation alert to owner (SMS + email) with suggested
   service-recovery step — regardless of approval mode.
5. Log `review_response.published`; update review-velocity metrics.

**Failure handling:** GBP publish failures retry ×3; if the GBP connection is broken, module
auto-pauses with portal banner ("Reconnect Google"). Poll cursor persisted so no review is
missed across downtime; duplicate polls are no-ops.

**Connections:** POS (canonical event), Twilio (via send-pipeline), GBP API
(`business.manage`). *Interim before GBP API approval:* reviews fetched via Places API
(read-only) + responses posted manually from an approval queue by Seekly ops.

---

## WF-2 · Speed-to-Lead Engine

**Purpose:** answer every new lead by SMS within seconds, qualify conversationally, and drive
to a booking — 24/7.
**Switch:** `speed_to_lead`.

**Config (defaults):** qualification questions per client (from intake — e.g. golf venue:
party size, date, occasion, bay/sim preference), booking link, hot-lead alert recipients,
follow-up cadence (`5 touches / 7 days`), AI persona name, business hours + after-hours
messaging, handoff keywords (price negotiation, complaints → human).

**Sources → `lead.created`:**
- **Existing website form** (v1): per-client email-parse inbox receives the form notification
  email → AI extraction → normalized lead. Where the form supports webhooks, rung-2 webhook
  instead.
- **Meta Lead Ads** (v1): leadgen webhook on Seekly's Meta app → fetch full lead via Graph API.
- **Missed/after-hours calls** (v1.1): Twilio tracking number overlay → `call.missed` →
  text-back. Designed now, shipped after pilots stabilize.

**Flow:**
1. Gate: switch on; dedupe on `event_id` + (phone, 24h) window.
2. Instant first-touch SMS (< 60s from event): acknowledges the specific request
   ("Got your request about a birthday party for 8 on Friday…"), asks first qualification
   question. Send-pipeline enforces compliance (conversational messages exempt from quiet-hours
   *initiation* rule only for immediate replies to an inbound lead; overnight leads get instant
   reply — this is a direct response to their inquiry).
3. Conversation loop: inbound `message.received` routes to the AI agent with full thread
   context; agent works through qualification config; every exchange logged.
4. Outcomes:
   - **Qualified/hot** → send booking link; fire hot-lead alert (SMS+email to owner with
     transcript summary); mark lead `qualified`.
   - **Needs human** (handoff keyword, confusion, or 2 consecutive low-confidence turns) →
     alert owner, agent sends "the owner will text you directly," conversation flagged.
   - **No response** → follow-up sequence (default day 0 +2h, day 1, day 3, day 5, day 7,
     varied angles) until reply or sequence end → mark `cold`.
5. Attribution: lead source, response time, outcome, booked flag → activity log (case-study
   metrics).

**Failure handling:** if AI generation fails, fallback template ("Thanks for reaching out —
what date were you thinking?") so the < 60s promise never breaks; conversation resumes with AI
on next turn. Email-parse extraction below confidence threshold → ops review queue, never a
wrong text to a customer.

**Connections:** email-parse infra, Meta Graph API (leadgen), Twilio, booking link (no
integration needed v1).

---

## WF-3 · Database Reactivation Engine

**Purpose:** win back lapsed customers with personalized, offer-driven campaigns — the fastest
provable revenue for repeat-visit businesses.
**Switch:** `reactivation`.

**Config (defaults):** lapse threshold (`90d` for golf/rec; per client), audience filters,
campaign angle library (client-approved at intake: league nights, birthday party specials,
lesson packages, seasonal offers), batch size/day (`50`), campaign cadence (monthly, or
manual launch from portal), consent rule: `imported_list`/`existing_customer` only.

**Audience source:** canonical `customers` table, populated by POS API sync (rung 1),
CSV upload at onboarding (rung 4), or ongoing `sale.completed` events (every sale
creates/updates a customer + `last_visit_at`).

**Flow (per campaign):**
1. Gate: switch on; A2P campaign approved; audience > 0 after exclusions (opted out, active
   conversation, messaged within frequency cap, no consent basis).
2. Segment: `last_visit_at` older than threshold, has phone; rank by historical value.
3. AI generates the campaign message set in brand voice from the chosen angle — personalized
   per customer (name, last visit context where known); Seekly approves campaign copy in
   portal before first-ever send for a client (then per config).
4. Batched sends through the send-pipeline (respecting caps + quiet hours) over N days.
5. Replies → same conversation loop as WF-2 (the AI books them or hands off).
6. Attribution: reply rate, bookings, revenue estimate per campaign → portal + case study.

**Failure handling:** campaign is resumable (send ledger per campaign member); a crash mid-batch
never re-texts anyone. Any spike in STOP rate (>3% of a batch) auto-pauses the campaign and
alerts ops.

**Connections:** POS/CSV customer data, Twilio. No client-side software required in v1.

---

## WF-4 · SEO/GEO Content Engine

**Purpose:** publish structured, entity-rich content that ranks in Google AND gets cited by AI
engines (ChatGPT, Perplexity, AI Overviews) for high-intent local queries.
**Switch:** `content_engine`.

**Config (defaults):** cadence (`2 posts/mo` pilots), topic queue (seeded at intake from
services × service areas × target keywords × competitor gaps), brand voice, publishing target
(WordPress REST API w/ application password; other platforms per intake; `email_draft`
universal fallback = send HTML to client/webmaster), approval mode (`approve_first_3` then
auto), schema profile (LocalBusiness + Service + FAQPage).

**Flow (cron per client per calendar):**
1. Gate + pick next topic from queue (skip if queue empty → alert ops to replenish).
2. Research pass: competitor content on the topic, People-Also-Ask, current GBP categories —
   builds the outline and target entities.
3. AI drafts: H1/H2 structure, **TL;DR block up top, FAQ section with FAQPage schema, custom
   JSON-LD (LocalBusiness/Service), semantic entity references (place names, services,
   landmarks), internal links** — the GEO pattern from the brainstorm notes, tailored to the
   client's niche.
4. Quality gate: automated checks (length, schema validates, keywords present, no hallucinated
   claims like fake prices — facts only from client profile) → approval per config.
5. Publish via configured target → capture URL → emit **`content.published`**.
6. Track: GSC impressions/clicks for the URL, rankings on target queries, AI share-of-voice
   spot-checks (scheduled prompts to AI engines logging brand mentions) → Visibility dashboard.

**Failure handling:** publish failure → retry, then park as `ready_to_publish` with ops alert
(never silently lost). Schema validation failure blocks publish.

**Connections:** WordPress REST (or per-intake platform), GSC API (already in Google OAuth
scopes), GBP (categories), AI engines for SoV checks.

---

## WF-5 · Directory Sync Engine

**Purpose:** one source of truth for NAP + hours pushed everywhere; consistent entity data for
local rank and AI citation.
**Switch:** `directory_sync`.

**Config:** BrightLocal location ID, directory set (BrightLocal default network), audit
frequency (`quarterly`), auto-push on profile change (`on`).

**Flow:**
- **Onboarding:** create BrightLocal location from business profile → initial citation
  audit (baseline listings-health score for the case study) → submit/sync listings.
- **`profile.updated`** (hours, phone, address, holiday hours changed in portal): validate →
  push to BrightLocal API → *also* push hours/holiday hours directly to GBP via our own OAuth
  (faster than aggregator propagation) → poll submission statuses → update listings-health.
- **Quarterly cron:** re-audit → diff → new inconsistencies → fix via sync → report in portal.

**Failure handling:** BrightLocal submission failures surface per-directory status in portal
(honest "12/15 synced, 3 pending"); no silent claims of consistency.

**Connections:** BrightLocal API (Seekly account, per-location billing — pilot COGS absorbed),
GBP API.

---

## WF-6 · Competitor Espionage Engine

**Purpose:** monthly AI deep-dive on each client's competitors — offers, pricing changes, new
services, review spikes — delivered as an owner-readable digest with recommended counters.
**Switch:** `competitor_intel`.

**Config:** 3–5 competitors per client (GBP place IDs + website URLs + FB pages, from intake),
cadence (`monthly` deep-dive per decision; weekly snapshots retained internally), digest
recipients (client owner + Seekly).

**Flow:**
1. **Weekly snapshot cron (internal, cheap):** fetch competitor websites (pricing/services
   pages) → store normalized text snapshot; pull GBP public data via Places API (rating,
   review count, photos, attributes) → store. No client OAuth needed — fully public data.
2. **Monthly digest cron:** diff snapshots across the month → AI analysis: new/changed offers,
   price movements, new services, review-velocity anomalies (spike = they're running a review
   campaign), posting activity → generate digest: what changed, why it matters, recommended
   counter-moves (feeds future Growth Action Queue).
3. Deliver: email to client (branded), archived in portal, copy to Seekly ops for
   account-management context.
4. **Prospecting mode:** same workflow runnable against a *prospect's* competitors with
   `client_id = prospect` — output becomes the outbound lead magnet.

**Failure handling:** unreachable competitor site → noted in digest ("site unreachable this
month"), never fabricated. Diffs with no material changes → shorter "all quiet" digest (still
sent — proof of monitoring).

**Connections:** Places API (public), HTTP fetch + AI extraction. Zero client-side integration
— **this engine can go live before any client OAuth exists.**

---

## WF-7 · Social Syndication Engine

**Purpose:** every blog post automatically becomes high-intent, keyword-packed, GEO-optimized
GBP posts + Facebook updates — content works three channels, written once.
**Switch:** `social_syndication` (sub-switches per channel: `gbp_posts`, `facebook_posts`).

**Config:** channels (v1: GBP + Facebook), posting schedule preferences, CTA defaults
(booking link with UTM), image handling (client photo library in portal; fallback branded
template card).

**Flow:**
1. **Trigger:** canonical `content.published` — from WF-4, *or* from the RSS/sitemap watcher
   on a client's existing blog (syndication works even for clients who write their own
   content — independent toggle, own value).
2. AI cuts the post into channel-native pieces:
   - **GBP "What's New" post:** ≤1,500 chars, local keywords + geo references, service
     entities, CTA button (Learn more/Book) with UTM link. Complies with GBP post policies.
   - **Facebook page update:** looser, community tone; link back to post.
3. Schedule at optimal times (config) → publish via GBP API + Meta Pages API.
4. Log posts + track UTM clicks → attribution into Visibility dashboard.

**Failure handling:** per-channel isolation — Facebook failure never blocks the GBP post;
failed channel retried, then queued with ops alert.

**Connections:** GBP + Meta Pages OAuth (same connections as WF-1/WF-2 — no new integrations).

---

## Shared send-pipeline (sub-workflow, used by WF-1/2/3)

```
send(client_id, customer, message, kind: transactional|conversational|marketing)
  1. opt-out ledger check (client-scoped, cross-engine)  → blocked? drop + log
  2. quiet hours (client timezone; marketing only; queue until window)
  3. frequency cap (marketing only, default 30d across ALL engines)
  4. A2P campaign approved? (hard gate)
  5. message ledger write (idempotency: no duplicate send for same trigger)
  6. Twilio send (client subaccount) → status callback updates ledger
  7. inbound STOP/HELP handled globally → ledger + confirmation per carrier rules
```

## Switchboard summary

| Switch | Engine | Safe-off behavior |
|---|---|---|
| `review_automation` | WF-1 | Requests stop; detection continues (data kept fresh) |
| `speed_to_lead` | WF-2 | New leads logged but not messaged; alert owner instead |
| `reactivation` | WF-3 | Campaigns pause mid-batch safely (resumable) |
| `content_engine` | WF-4 | Calendar pauses; drafts in flight parked |
| `directory_sync` | WF-5 | No pushes; audits stop; last-known health shown |
| `competitor_intel` | WF-6 | Snapshots stop; history retained |
| `social_syndication` | WF-7 | `content.published` events ignored (not queued) |

Every switch is instant (checked at run start), logged (who flipped it, when), and visible to
the client in the portal as their plan's feature list.
