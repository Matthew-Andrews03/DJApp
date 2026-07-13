# 11 — Native Adapters (Wix is Day-1)

Native adapters translate a client's existing software into Seekly's canonical events
(`sale.completed`, `lead.created`, `customer.imported`, `content.published`, `profile.updated`)
and act as publishing targets. They plug into the OAuth Integration Hub (spec 04) and the
internal event API (spec 02). Engines never change when an adapter is added.

**This revision promotes Wix to a Wave-1, Day-1 onboarding OAuth integration** because the first
pilot (golf/recreation venue) runs on Wix. Wix is unusual and valuable: it can be the pilot's
*single primary integration*, covering four engines at once —

| Wix surface | Canonical event | Engine served |
|---|---|---|
| Wix Bookings (booking confirmed) | `sale.completed` | Review Velocity (WF-1) |
| Wix Stores/eCom (order paid) | `sale.completed` | Review Velocity (WF-1) |
| Wix Forms (submission) | `lead.created` | Speed-to-Lead (WF-2) |
| Wix Contacts (query + created) | `customer.imported` | Reactivation (WF-3) |
| Wix Blog (draft → publish) | `content.published` target | Content (WF-4) → Syndication (WF-7) |

So for the pilot, "Connect Wix" alone can light up WF-1, WF-2, WF-3, WF-4, WF-7 — no POS
email-parse, no CSV, no WordPress. That is the fastest possible path to a live pilot.

---

## Coverage matrix (all planned adapters)

| Platform | Emits | Connect method | Approval gate | Wave |
|---|---|---|---|---|
| **Wix** | sale.completed, lead.created, customer.imported, content.published(target), profile seed | **OAuth (App Market app)** | App Market app creation (self-serve, no review to run privately) | **1 (Day-1)** |
| Square | sale.completed, customer.imported, (bookings) | OAuth 2.0 app | Self-serve dev app | 1 |
| Calendly | lead.created/booking → sale.completed | OAuth 2.0 app | Self-serve | 1 |
| Google (GBP/GA4/GSC) | review.received, profile push, publish target | OAuth (spec 04) | GBP API application | 1 (in spec 04) |
| Meta (FB/IG/Lead Ads) | lead.created, publish target | OAuth (spec 04) | App Review for prod | 1 (in spec 04) |
| Shopify | sale.completed, customer.imported, content.published(target) | OAuth app | Protected customer-data approval | 2 |
| Webflow | content.published(target), form → lead.created | OAuth app | Self-serve | 2 |
| Jobber | sale.completed (job complete), customer.imported | OAuth (GraphQL) | Developer app | 2 |
| Housecall Pro | sale.completed, customer.imported | OAuth (verify) | Developer program | 2 |
| Acuity Scheduling | sale.completed (appt), lead.created | OAuth 2.0 | Self-serve | 2 |
| Clover | sale.completed, customer.imported | OAuth | Dev app + device nuances | 2 |
| ServiceTitan | sale.completed, customer.imported | OAuth | Gated dev program | 3 |
| Mindbody | sale.completed, customer.imported | OAuth | Gated partner program | 3 |
| GoHighLevel | lead.created, customer.imported | Marketplace OAuth | Marketplace app | 3 |
| Squarespace | (content limited) | API key (no content publish API) | — | **20% bucket** |
| Toast / OpenTable / Resy | sale.completed / guest list | Partner-gated / CSV | Partner program | **20% bucket** |

Wave 1 = with the pilot. Wave 2 = weeks 4–6. Wave 3 = as gated approvals clear. The 20% bucket
uses the generic rungs (email-parse / webhook / CSV) from specs 02/05 — explicitly out of scope
for native OAuth per founder direction.

> Full per-platform build detail beyond Wix is captured in the research notes and will be
> expanded into this file platform-by-platform as each wave opens. Wix (below) is the complete,
> build-ready reference; Square is the second reference (framework §Reference) since it is the
> most likely POS across verticals.

---

## Shared adapter framework

Every adapter is a module at `src/server/adapters/<platform>.ts` implementing one interface, so
the Integration Hub and engines treat all platforms uniformly.

```ts
// src/server/adapters/types.ts
export interface Adapter {
  readonly provider: string;                 // matches connections.provider enum value

  /** OAuth: build the provider authorize/install URL for "Connect X". */
  buildConnectUrl(args: { clientId: string; state: string; redirectUri: string }): string;

  /** OAuth: exchange the callback (code/instanceId/etc.) → a stored connection row. */
  handleCallback(args: {
    clientId: string;
    query: Record<string, string>;
  }): Promise<{ providerAccountId: string; scopes: string[]; metadata: Record<string, unknown> }>;

  /** Mint a short-lived access token for API calls (cached until expiry). */
  getFreshAccessToken(connectionId: string): Promise<string>;

  /** Verify + parse an inbound webhook into canonical events (0..n). */
  parseWebhook(args: { rawBody: string; headers: Record<string, string>; connection: Connection })
    : Promise<CanonicalEvent[]>;

  /** One-time + incremental customer backfill for reactivation. */
  backfillCustomers(args: { connection: Connection; since?: string })
    : AsyncIterable<CanonicalCustomer>;

  /** Publish a content item (website adapters only; others throw NotSupported). */
  publishContent?(args: { connection: Connection; item: ContentItem }): Promise<{ url: string }>;

  /** Health check used by the connection-health cron (spec 04). */
  healthCheck(connection: Connection): Promise<{ ok: boolean; reason?: string }>;
}
```

Registration: `src/server/adapters/registry.ts` maps `provider → Adapter`. The Hub's generic
`/api/oauth/[provider]/start` and `/callback` routes (spec 04) look the adapter up by name, so
adding a platform is: write the module, add its `provider` enum value (R-item below), register
it. No route or engine changes.

Canonical helpers reused: `emitCanonicalEvent()` (spec 02) for events, `upsertCustomer()`
(spec 05) for backfill, `getFreshAccessToken` stores/reads the cached token via the vault
(spec 04).

---

## Wix adapter — full spec (Wave 1, Day-1)

Docs: [About OAuth](https://dev.wix.com/docs/build-apps/develop-your-app/access/authentication/about-oauth) ·
[Authenticate Using OAuth](https://dev.wix.com/docs/build-apps/develop-your-app/access/authentication/authenticate-using-oauth) ·
[Create Access Token](https://dev.wix.com/docs/api-reference/app-management/oauth-2/create-access-token) ·
[Blog Draft Posts](https://dev.wix.com/docs/api-reference/business-solutions/blog/draft-posts/create-draft-post) ·
[Query Contacts](https://dev.wix.com/docs/rest/crm/members-contacts/contacts/contacts/contact-v4/query-contacts) ·
[Booking Confirmed webhook](https://dev.wix.com/docs/api-reference/business-solutions/bookings/bookings/bookings-writer-v2/booking-confirmed) ·
[eCom Order Created](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/orders/orders/order-created) ·
[Verify webhook requests](https://dev.wix.com/docs/build-apps/develop-your-app/access/authentication/verify-requests-received-from-wix) ·
[Media import](https://dev.wix.com/docs/api-reference/assets/media/media-manager/files/import-file)

### AD-W1 · Create the Seekly Wix App (one-time, Day-1 application)

In the **Wix Developers Center** (dev.wix.com → Custom Apps / Build an App):
1. Create an app "Seekly". Record **App ID** and **App Secret Key** → env
   `WIX_APP_ID`, `WIX_APP_SECRET` (added to `src/env.ts`).
2. **OAuth → Redirect URLs:** add `https://app.<seekly-domain>/api/oauth/wix/callback`.
3. **Permissions** (App Dashboard → Permissions → Add Permissions) — request only these:
   - *Manage Blog* (create/publish draft posts, manage categories) → publishing
   - *Read Contacts* + *Manage Contacts* → reactivation backfill + incremental
   - *Read Bookings* (Bookings Reader) → booking-confirmed events
   - *Read Orders* (eCommerce) → order events (only relevant if client runs Wix Stores)
   - *Manage Media* → import blog images
   - *Read Site Properties* → seed NAP/hours for directory sync
   - Wix Forms read (submissions) → lead capture
4. **Webhooks** (App Dashboard → Webhooks → per event; copy the **Public Key** → env
   `WIX_WEBHOOK_PUBLIC_KEY`). Subscribe:
   - `wix.bookings.v2.booking` → *Booking Confirmed*
   - `wix.ecom.v1.order` → *Order Created* and *Order Updated* (payment status)
   - Wix Forms → *Form Submission*
   - `wix.contacts` → *Contact Created* (incremental customers)
   - App lifecycle → *App Instance Installed* / *Removed*
   All webhook targets point to `https://app.<seekly-domain>/api/webhooks/wix`.
5. The app can be used privately against pilot sites immediately (share the install link); App
   Market **listing/review is only required to appear in the public marketplace**, which is a
   later growth step, not a pilot blocker.

**Acceptance:** App ID/secret/public key in env; redirect + webhooks configured; the pilot's
Wix site can install the app via the link from AD-W2.

### AD-W2 · Connect flow ("Connect Wix" button)

Auth model: Wix App-Market OAuth. Install (authorization-code) captures the site's
`instanceId` and the owner's consent; API calls then use the **client-credentials** grant with
`instanceId` + app creds to mint 4-hour access tokens (no per-client refresh token to store).

```
Client clicks "Connect Wix"
  → GET /api/oauth/wix/start
      state = signed JWT { clientId, userId, nonce }   (spec 04 pattern)
      302 → https://www.wix.com/installer/install
              ?appId={WIX_APP_ID}
              &redirectUrl={ENC}https://app.<domain>/api/oauth/wix/callback
              &state={state}
  → Owner approves permissions on Wix
  → Wix 302 → /api/oauth/wix/callback?code=...&instanceId=...&state=...
      verify state JWT → clientId
      (code is optional to exchange; instanceId is the durable handle)
      store connection: provider='wix', provider_account_id=instanceId,
        scopes=[...granted], connection_status='active',
        metadata={ instanceId, siteName? }
      → redirect to portal Integrations page (wix card = active)
```

`buildConnectUrl` returns the install URL; `handleCallback` verifies state and persists the
connection. **No encrypted refresh token is stored** — `WIX_APP_SECRET` is a Seekly-level env
var, and per-client identity is the `instanceId`. (This is a deliberate, documented exception to
spec 04's refresh-token vault pattern; the vault still caches the short-lived access token.)

```ts
// getFreshAccessToken(connectionId)
POST https://www.wixapis.com/oauth2/token
  { grant_type: "client_credentials",
    client_id: WIX_APP_ID, client_secret: WIX_APP_SECRET,
    instance_id: connection.providerAccountId }
→ { access_token, token_type: "Bearer", expires_in: 14400 }
cache in vault keyed by connectionId with expiry = now + 14400 - 300s skew.
```

**Acceptance:** connecting the pilot site yields an `active` wix connection with a real
`instanceId`; `getFreshAccessToken` returns a working Bearer token and re-mints after 4h.

### AD-W3 · Webhook ingestion → canonical events

Route `POST /api/webhooks/wix` (public, no session):
1. Read the raw body — Wix sends the event as a **signed JWT**.
2. **Verify** the JWT signature against `WIX_WEBHOOK_PUBLIC_KEY` (RS256). Reject if invalid.
3. Decode → `{ instanceId, eventType, entityFqdn, slug, eventId, data }`.
4. Resolve `clientId` from the `instanceId` (connections lookup). Unknown instance → 202 + log.
5. Map to canonical event(s) and POST to `/api/internal/events` using **`eventId` as the
   canonical `event_id`** (Wix guarantees a unique event ID per event → free idempotency).

Mapping table:

| Wix event (`entityFqdn` / slug) | Canonical | Payload mapping |
|---|---|---|
| `wix.bookings.v2.booking` / confirmed | `sale.completed` | customer ← booking contact (name/phone/email via Contacts lookup on `contactId`); `amount` ← booking total if present; `occurred_at` ← booking start; `source:'wix'` |
| `wix.ecom.v1.order` / created·updated(paid) | `sale.completed` | customer ← `buyerInfo`/billing contact; `amount` ← order total; dedupe so created+paid on the same order emit once (guard on order id + a `paid` flag) |
| Wix Forms / submission | `lead.created` | contact ← submitted fields (name/phone/email); `intent` ← mapped form fields (config: which field = message/service/date) |
| `wix.contacts` / contact_created | `customer.imported` | name/phone/email; `consent_basis:'existing_customer'` only if the contact came from a booking/order, else omit (flag for review) |
| App Instance Removed | (internal) | mark connection `revoked`, auto-pause Wix-dependent modules (spec 04) |

**Gotcha (dedupe):** eCom emits both `created` and `updated`; only emit `sale.completed` when
payment status becomes `PAID`. Bookings "confirmed" is the single clean trigger for booking-led
venues — prefer it for the golf pilot.

**Acceptance:** a real test booking on the pilot site produces exactly one `sale.completed`
(verified by replaying the same webhook → still one, via `event_id`); an invalid-signature body
is rejected 401.

### AD-W4 · Customer backfill (reactivation audience)

`backfillCustomers` uses `POST https://www.wixapis.com/contacts/v4/contacts/query`:
- Cursor paging, up to 1,000 contacts/page; **set filter + sort on the first request only**,
  then pass only the returned cursor (Wix ignores filter/sort on cursor follow-ups).
- Incremental: filter `info.updatedDate > since`, sort `updatedDate asc`; store the max
  `updatedDate` as the watermark for the next run (nightly).
- Map each contact → `upsertCustomer` (name, phone, email, `external_ids.wixContactId`,
  `last_visit_at` from latest booking/order where resolvable).
- **Consent caveat (must enforce):** the full contact list may include non-customers (newsletter
  signups). Default `consent_basis` for bulk contacts is `imported_list` (reactivation already
  requires a prior-relationship basis, and the portal campaign approval gate stands). Contacts
  derived from bookings/orders get `existing_customer`. Document this in the onboarding UI so the
  client attests they have consent to be contacted.

**Acceptance:** initial backfill imports the pilot's contacts with phones; a subsequent run only
pulls contacts changed since the watermark; no duplicate customers (unique on `client_id`,phone).

### AD-W5 · Content publishing (WF-4 target → emits `content.published`)

`publishContent` for Wix Blog:
1. Ensure a `memberId` for authorship — **required for 3rd-party apps**; store the chosen
   author memberId in the connection metadata at connect time (pick a site member; document the
   step). 
2. Convert the engine's HTML draft → **Ricos rich-content JSON** (Wix does *not* accept raw
   HTML). Implement `htmlToRicos()` in `src/server/adapters/wix/ricos.ts`:
   - Every TEXT node must be wrapped in a PARAGRAPH node (never place TEXT directly in
     BLOCKQUOTE/LIST_ITEM). Support h1–h3, p, ul/ol, blockquote, links, images.
   - Enforce the **400 KB** per-draft limit; if exceeded, split or trim and warn.
3. Import images first via `POST /site-media/v1/files/import` (external URL → wixstatic URL),
   reference them in the Ricos `IMAGE` nodes.
4. `POST /blog/v3/draft-posts` (title, richContent, memberId, categoryIds that already exist —
   unknown category IDs are silently dropped, so create/lookup categories first).
5. `POST /blog/v3/draft-posts/{id}/publish` → returns the live post → capture URL → emit
   `content.published` (which WF-7 syndicates to GBP + Facebook).

**Gotchas & schema handling:** Ricos conversion is the real work here (biggest risk in the Wix
adapter); ship it with unit tests over representative posts. FAQ blocks render as headed Q/A
paragraphs in the Ricos body.

Structured-data / schema on Wix (corrected — schema IS supported, several ways):
- **Automatic:** every published Wix Blog post gets `BlogPosting`/`Article` preset markup, and
  Wix auto-generates additional structured data for eligible posts — so our published content is
  never schema-less. ([auto structured data](https://support.wix.com/en/article/using-ai-generated-structured-data-for-blog-posts))
- **Custom JSON-LD, manual:** per page via Editor → SEO Basics → Advanced SEO → Structured Data
  Markup (JSON-LD only, **< 7,000 chars/markup**). Use a one-time onboarding pass to add
  `FAQPage` / richer `LocalBusiness` / `Service` to evergreen pages.
  ([add markup](https://support.wix.com/en/article/adding-structured-data-markup-to-your-sites-pages-2546962))
- **Programmatic:** `wixSeoFrontend.setStructuredData()` (Velo, runs on the site) can set JSON-LD
  dynamically — a path to automate custom schema if we ship a small Velo snippet on the client's
  site. ([Velo setStructuredData](https://dev.wix.com/docs/velo/api-reference/wix-seo-frontend/set-structured-data))
- **AD-W5a (verify):** confirm whether the Blog `create-draft-post` REST object accepts a
  settable `seoData` (structured-data tags) field. **If yes, our backend adapter can inject
  custom JSON-LD (FAQPage etc.) end-to-end with no manual step** — this is the preferred path;
  fall back to the manual onboarding pass only if the REST field can't carry structured data.

Net: Wix content ships with automatic Article schema; custom schema is available manually or
(pending AD-W5a) programmatically. Wix's custom-schema *automation* is thinner than WordPress,
so if AD-W5a fails, directory-sync + GBP carry more of the GEO weight for Wix clients — but the
site is not schema-poor.

**Acceptance:** a WF-4 draft publishes to the pilot's Wix blog with correct headings/images and
emits `content.published`; the published URL is reachable.

### AD-W6 · Health check & disconnect

- `healthCheck`: mint a token + `GET` a cheap endpoint (e.g. Contacts query limit 1). Failure →
  `connection_status='broken'` → portal banner + auto-pause Wix-dependent modules (spec 04
  health cron).
- Disconnect (portal): on *App Instance Removed* webhook OR client clicking Disconnect, set
  `revoked`, stop token minting, pause modules. (No refresh token to revoke; app secret stays.)

**Acceptance:** revoking the app on the Wix side flips the connection to broken/revoked within a
health-cron cycle and pauses the affected modules with a visible reason.

---

## Vertical coverage map (brief)

| Vertical | Primary native adapters | Notes |
|---|---|---|
| Golf / recreation | **Wix** (pilot), Square, Calendly, Acuity | Wix Bookings or Square covers sale.completed; Wix covers all four engines for the pilot |
| Home services | Jobber, Housecall Pro, ServiceTitan(gated) | job-complete → review; strong reactivation |
| Restaurants | Square, Clover | Toast/OpenTable/Resy = 20% bucket (partner-gated; email-parse + guest CSV) |
| Salons / spas | Square Appointments, (Fresha/Vagaro/Booksy — verify per client) | booking-led review + reactivation |
| Dental / physio / chiro | **PHI-free config only** | HIPAA/PHI: no visit-triggered review or patient-list reactivation until a BAA-backed compliance track exists. Serve SEO/GEO content, GBP, directory sync, espionage, syndication, and speed-to-lead for *new-patient inquiries* (a form lead, not a patient record). PMS systems (Dentrix/Eaglesoft/Open Dental; Jane/Cliniko/WebPT) are partner-gated/legacy → 20% bucket. |

Healthcare is a **v2 vertical** — gated integrations plus regulatory overhead; the PHI-free
configuration is the wedge in the meantime.

---

## Ticket table

| ID | Title | Depends on |
|---|---|---|
| AD-1 | Adapter interface + registry (`src/server/adapters/`) | spec 01, 02, 04 |
| AD-W1 | Create Seekly Wix App (dev center, perms, webhooks, env) | — (Day-1 application) |
| AD-W2 | Wix connect flow (start/callback, instanceId, token minting) | AD-1, spec 04 |
| AD-W3 | Wix webhook route → canonical events (JWT verify, mapping, dedupe) | AD-W2, spec 02 |
| AD-W4 | Wix customer backfill (Contacts query, cursor, watermark, consent) | AD-W2, spec 01 |
| AD-W5 | Wix Blog publishing + `htmlToRicos()` + media import | AD-W2, WF-4 |
| AD-W6 | Wix health check + disconnect | AD-W2, spec 04 health cron |
| AD-2 | Square adapter (second reference; POS across verticals) | AD-1 |
| AD-3 | Calendly adapter | AD-1 |
| AD-4..N | Wave 2/3 adapters as waves open | AD-1 |

**Build order for the pilot:** AD-1 → AD-W1 (file the app today) → AD-W2 → AD-W3 (unblocks WF-1
review triggers + WF-2 leads) → AD-W4 (unblocks WF-3) → AD-W5 (unblocks WF-4/WF-7). AD-W2+W3
are the minimum to make the Wix pilot live.

## R-item for the implementation guide

`connections.provider` enum (spec 01) must include: `wix`, `square`, `calendly`, `shopify`,
`webflow`, `jobber`, `housecall_pro`, `acuity`, `clover`, `servicetitan`, `mindbody`,
`gohighlevel` (in addition to the spec-04 providers `google_business`, `google_analytics`,
`search_console`, `meta`, `wordpress`). Add `wix` now (Wave 1); add others as their waves open.
Flagged as **R-11** in `specs/00-implementation-guide.md`.
