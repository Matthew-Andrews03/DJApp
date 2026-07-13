# 05 — Client Portal Specification

The portal owns the **service relationship** (never Seekly's sales process). It must feel like
software from the first login: connect accounts, see live results, approve what needs approving.
Two audiences, one app: clients (scoped to their tenant) and Seekly admins (all tenants +
switchboard).

**Stack:** Next.js (App Router) on Vercel · Supabase (Postgres, Auth magic links, RLS,
Storage for photos) · shadcn/ui. Domain: `app.<seekly-domain>`.

## Information architecture (client view)

```
CLIENT PORTAL
├── Overview
│   ├── Growth score (composite from metrics_daily)
│   ├── Revenue recovered/generated (attribution rollup)
│   ├── Opportunities (found-money feed: unanswered leads, lapsed count, review gaps)
│   └── Recent activity (from activity_log, human-readable)
├── Visibility
│   ├── SEO (GSC clicks/impressions on tracked queries)
│   ├── GEO / AI Share of Voice (brand-mention spot checks over time)
│   ├── GBP (posts published, profile health)
│   ├── Listings (directory sync health score, per-directory status)
│   └── Rankings (target keyword positions)
├── Reputation
│   ├── Reviews feed (rating, text, AI response + status)
│   ├── Sentiment summary
│   └── Review request stats (sent → clicked → reviewed funnel)
├── Growth
│   ├── Leads (conversations list: source, response time, status, transcript)
│   ├── Speed-to-lead stats (median response seconds — the hero number)
│   ├── Reactivation (campaigns, replies, bookings, revenue estimate)
│   └── Revenue leakage report (monthly)
├── Content
│   ├── Published posts + syndicated GBP/FB pieces with clicks
│   └── Upcoming calendar
├── Competitors
│   └── Monthly espionage digests (archive)
├── Integrations
│   ├── Connect Google (GBP + GA4 + GSC in one consent)
│   ├── Connect Facebook/Instagram
│   ├── Connect Website (platform-specific: WordPress creds / instructions)
│   ├── POS / Booking (status of adapter: email-parse active, native connected, CSV)
│   └── Phone (v1.1 tracking number status)
├── Approvals (only modules configured to require them)
│   ├── Review responses · GBP posts · Content · Campaigns
│   └── One-tap approve/edit/reject; SMS/email notification with deep link
└── Settings
    ├── Business profile (NAP, hours, holiday hours → emits profile.updated)
    ├── Brand voice review
    └── Team users
```

## Onboarding flow (first login)

1. Magic-link from WF-0 welcome email → set up account.
2. **Guided connect sequence** (checklist generated from tier + intake): Connect Google →
   pick GBP location → Connect Facebook → pick Page → website access → POS path
   (forward-your-receipts email instructions, or OAuth where native adapter exists) →
   customer CSV upload (reactivation).
3. Confirm the 10–15 strategic answers captured at intake (pre-filled — client verifies, not
   re-types).
4. Each completed connection flips its integration to `active`; when a module's dependencies
   are all green, Seekly admin flips the module switch — client sees modules light up.
5. Completion syncs `Onboarding Status: Complete` back to HubSpot (WF-0 step 8).

## Admin area (Seekly staff only)

- **Tenant list:** health at a glance (connections status, module switches, error counts,
  A2P status, last activity).
- **THE SWITCHBOARD:** per client, every module toggle + settings editor (JSON-schema-driven
  forms over `workflow_config.settings`). Flip = instant, logged.
- **Ops queues:** dead-letter events, failed sends, low-confidence email-parses, content
  quality-gate failures.
- **Intake runner:** create client manually (pilots bypass HubSpot), run WF-0 provisioning.
- **Prospect mode:** run WF-6 espionage for a prospect (lead-magnet generation).

## Portal MVP cut (2-week milestone)

Ship ONLY:
1. Auth (magic link) + tenant scoping
2. Integrations page: Connect Google OAuth (full flow → token vault → location select),
   POS email-forwarding instructions, CSV upload
3. Overview: activity feed + 3 stat cards (reviews sent/received, leads answered, median
   response time) straight from `activity_log`/`metrics_daily`
4. Reputation: reviews feed with AI responses
5. Admin switchboard (toggles + raw JSON settings editor)

Everything else lands in weeks 3–4 in this order: Growth (conversations + transcripts) →
Approvals → Content + Competitors → Visibility rollups → Settings/profile editing
(until then, profile edits go through Seekly admin).

## Design notes

- Every number on Overview must be **defensible from the activity log** — this dashboard is
  the case study and the renewal pitch. No vanity metrics.
- Client-facing copy frames everything as money: "3 leads answered in under a minute this
  week," "12 lapsed customers came back — est. $840."
- Mobile-first: owners live on their phones; approvals especially must be one thumb-tap.
