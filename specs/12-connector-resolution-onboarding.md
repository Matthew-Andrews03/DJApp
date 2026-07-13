# 12 — Connector Resolution & Tailored Onboarding

**Problem this solves:** the adapter catalog grows to 15+ connectors (spec 11), but any given
client needs only 2–4. Onboarding must show a client *only* the connectors relevant to them —
never "5 CRMs and 10 website connectors they don't need." A golf venue on Wix should see
**Connect Wix, Connect Google, (optional) Connect Facebook** — nothing else.

This spec makes the Integrations page and onboarding **data-driven from a per-client resolved
connector set**, replacing the hardcoded 5-card list in spec 07 §1.

## Concept

```
intake answers ──► resolver ──► client_connectors (per-client required set)
(client_profile)                        │
                                        ├──► Onboarding wizard (only these, in order)
                                        └──► Integrations page (only these + "Add more" for the long tail)
```

Three parts: (1) a static **catalog** of every connector; (2) a **resolver** that picks the
subset a client needs from their intake; (3) **UI** driven by that subset.

---

## CN-1 · Connector catalog

`src/server/integrations/catalog.ts` — the master registry. Adding an adapter (spec 11) adds one
entry here; nothing else in onboarding changes.

```ts
export type ConnectorCategory = "website" | "pos_booking" | "crm" | "reviews" | "ads" | "phone";
export type ConnectMethod = "oauth" | "app_install" | "app_password" | "email_parse" | "csv" | "manual";

export interface ConnectorDef {
  provider: string;                 // matches connections.provider enum (spec 01 / R-11)
  label: string;                    // "Wix", "Google Business Profile"
  category: ConnectorCategory;
  method: ConnectMethod;
  unlocksModules: string[];         // workflow_config module keys this connector feeds
  emits: string[];                  // canonical events (docs 02)
  startPath?: string;               // e.g. "/api/oauth/wix/start" (oauth/app_install)
  wave: 1 | 2 | 3;                  // availability (spec 11); wave>current → renders "coming_soon"
  blurb: string;                    // one client-facing sentence
}

export const CONNECTOR_CATALOG: ConnectorDef[] = [
  { provider: "wix", label: "Wix", category: "website", method: "app_install",
    unlocksModules: ["review_automation","speed_to_lead","reactivation","content_engine","social_syndication"],
    emits: ["sale.completed","lead.created","customer.imported","content.published"],
    startPath: "/api/oauth/wix/start", wave: 1,
    blurb: "One connection powers reviews, lead replies, win-backs, and content." },
  { provider: "google_business", label: "Google Business Profile", category: "reviews",
    method: "oauth", unlocksModules: ["review_automation","directory_sync","social_syndication"],
    emits: ["review.received"], startPath: "/api/oauth/google/start", wave: 1,
    blurb: "Reviews, posts, hours, and search analytics." },
  { provider: "meta", label: "Facebook / Instagram", category: "ads", method: "oauth",
    unlocksModules: ["speed_to_lead","social_syndication"], emits: ["lead.created"],
    startPath: "/api/oauth/meta/start", wave: 1,
    blurb: "Answer lead ads in seconds and keep your page active." },
  { provider: "wordpress", label: "WordPress", category: "website", method: "app_password",
    unlocksModules: ["content_engine","social_syndication"], emits: ["content.published"],
    wave: 1, blurb: "Publish SEO content straight to your site." },
  { provider: "square", label: "Square", category: "pos_booking", method: "oauth",
    unlocksModules: ["review_automation","reactivation"], emits: ["sale.completed","customer.imported"],
    startPath: "/api/oauth/square/start", wave: 1, blurb: "Trigger reviews after every sale." },
  { provider: "calendly", label: "Calendly", category: "pos_booking", method: "oauth",
    unlocksModules: ["review_automation","speed_to_lead"], emits: ["sale.completed","lead.created"],
    startPath: "/api/oauth/calendly/start", wave: 1, blurb: "Reviews after appointments." },
  // Wave 2/3 entries (shopify, webflow, jobber, housecall_pro, acuity, clover, servicetitan,
  // mindbody, gohighlevel) added as their adapters land — render "coming_soon" until then.
  // Generic fallbacks (always available, chosen by the resolver when no native adapter fits):
  { provider: "email_parse", label: "Email forwarding", category: "pos_booking", method: "email_parse",
    unlocksModules: ["review_automation","speed_to_lead"], emits: ["sale.completed","lead.created"],
    wave: 1, blurb: "Forward your booking/sale emails — works with any system." },
  { provider: "csv_customers", label: "Customer list", category: "crm", method: "csv",
    unlocksModules: ["reactivation"], emits: ["customer.imported"], wave: 1,
    blurb: "Upload your customer list to win back lapsed customers." },
];
```

**Acceptance:** every provider in the `connections` enum has exactly one catalog entry; each
`unlocksModules` value is a real `workflow_config` module key.

---

## CN-2 · Resolver (intake → required connectors)

`src/server/integrations/resolver.ts`. Pure function over `client_profile` + tier +
`enabled_modules`. Returns the minimal set, each tagged required/recommended/optional.

```ts
export type Requirement = "required" | "recommended" | "optional";
export interface ResolvedConnector { provider: string; requirement: Requirement; reason: string; }

export function resolveConnectors(input: {
  websitePlatform?: string;      // client_profile.website_platform ("wix"|"wordpress"|"shopify"|"other"|…)
  posSystem?: string;            // free-text/normalized POS name from intake
  usesMetaAds?: boolean;         // intake Q
  hasCustomerList?: boolean;     // intake Q (can export CSV / has contacts)
  enabledModules: string[];      // purchased tier's modules
}): ResolvedConnector[];
```

Resolution rules (junior-dev implementable, in order):

1. **Website / primary platform.** Map `websitePlatform` to a catalog website connector:
   `wix→wix`, `wordpress→wordpress`, `shopify→shopify`(wave-gated), else none.
   - If it's **Wix**, mark `wix` **required** — and because Wix also emits `sale.completed`,
     `lead.created`, `customer.imported`, the resolver **suppresses** the POS and CSV connectors
     for this client (Wix already covers them). This is the "one connection" outcome.
2. **POS / booking** (only if website connector doesn't already emit `sale.completed` and a
   review/reactivation module is enabled): map `posSystem` to a native connector
   (`square`, `calendly`, …). If no native adapter matches → `email_parse` (**required**).
3. **Reviews:** if `review_automation` or `directory_sync` enabled → `google_business`
   **required**.
4. **Ads/leads:** if `speed_to_lead` enabled and `usesMetaAds` → `meta` **recommended**; else
   `meta` **optional**.
5. **Customer list:** if `reactivation` enabled and step 1 didn't already cover
   `customer.imported` → `csv_customers` **required** (or `recommended` if the POS backfills it).
6. De-dupe; drop any connector whose `unlocksModules` are all disabled for this client.

**Worked example — the golf pilot (Wix, runs FB lead ads, all modules):**

| Connector | Requirement | Why |
|---|---|---|
| `wix` | required | website=Wix → also covers sales, leads, contacts |
| `google_business` | required | review_automation + directory_sync |
| `meta` | recommended | speed_to_lead + usesMetaAds |

→ **3 cards. No Square, no Shopify, no Jobber, no CSV, no email-parse.** That's the whole point.

**Acceptance:** unit tests for the pilot case (3 connectors) and a "WordPress + Square + no Meta
ads" case (wordpress required, square required, google required, meta optional, csv suppressed
because Square backfills). Changing `websitePlatform` from wix→wordpress flips the POS/CSV
suppression.

---

## CN-3 · Persistence & re-resolution

- Resolved set is written to **`onboarding_tasks`** (spec 09) as `connect:{provider}` rows with
  `requirement` + `unlocks_modules` columns (extend that table; R-item below). Reuses the
  existing "unchecked box → blocked module" model, so provisioning, the wizard, the Integrations
  page, and the admin switchboard all read one source of truth.
- **Run the resolver:** at provisioning (WF-0 / manual pilot script), and re-run on any
  `client_profile` change to `website_platform` / `pos_system` / `uses_meta_ads` /
  `enabled_modules` (portal Settings save or admin edit) — additively (never deletes a connector
  that's already `active`; new requirements appear, removed ones become `optional`).
- A connector task flips `done` when its `connections` row (or `commsProvisioning` email / CSV
  import) reaches `active`.

**Acceptance:** editing a client's website platform in admin re-resolves and the Integrations
page updates without a deploy; an already-connected provider is never dropped by re-resolution.

---

## CN-4 · Guided onboarding wizard (client)

Route `/portal/onboarding` (exists as a stub — `src/app/onboarding/page.tsx`; move/extend into
the authed portal). A stepper that shows **only the client's required + recommended connectors**,
in resolver order, with progress.

```
OnboardingWizard (server: reads resolved onboarding_tasks)
├─ ProgressHeader  "2 of 3 connected"   (linear progress bar)
├─ StepList (one ConnectorStep per required/recommended task, in order)
│   └─ ConnectorStep  — reuses ProviderCard (spec 07 §1.3) with catalog blurb + startPath
│        states: not_connected → (connect) → pending(pick location) → active ✓
├─ optional connectors collapsed under "Optional add-ons"
└─ FinishCard — enabled only when all `required` tasks are `done`
     → marks onboarding complete → syncs "Onboarding Status: Complete" to HubSpot (spec 09)
       → Seekly admin flips the now-eligible module switches
```

- Recommended (not required) steps are skippable ("Skip for now" → task stays `todo`, module
  stays off, reappears on the Integrations page).
- Mobile-first single-column; each step is one thumb-tap to start its OAuth/app-install flow.
- Copy is money-framed (docs 05): e.g. Wix step *"Connect Wix once — Seekly then handles reviews,
  new-lead replies, win-backs, and posts automatically."*

**Acceptance:** the pilot sees exactly 3 steps (Wix, Google required; Facebook recommended);
Finish is disabled until Wix + Google are active; completing it syncs status to HubSpot.

---

## CN-5 · Integrations page becomes data-driven (amends spec 07 §1)

Replace the hardcoded `ProviderCard provider="google" | "facebook" | "website" | "pos" | "phone"`
list with:

- **Your integrations:** render one `ProviderCard` per resolved `onboarding_tasks`/`connections`
  entry (required + recommended + any already-connected), using `CONNECTOR_CATALOG[provider]` for
  label/blurb/startPath. Card states unchanged (spec 07 §1.4: not_connected/pending/active/
  broken), plus `coming_soon` for wave-gated providers.
- **Add another integration:** a collapsed expander listing the rest of the catalog (filtered to
  categories that make sense) so a client *can* self-add one Seekly didn't auto-detect — but it's
  hidden by default, not thrown in their face. Selecting one adds an `optional` connector task.
- Admin mirror (`(app)/[clientSlug]/integrations`) additionally shows the raw resolved set and an
  **admin override** to add/remove a connector requirement (e.g., client reveals a second system
  mid-engagement).

**Acceptance:** a Wix client's Integrations page shows 3 cards, not the full catalog; the long
tail is one click away under "Add another integration"; admin can force-add `square` for a client
and it appears as a card.

---

## Ticket table

| ID | Title | Depends on |
|---|---|---|
| CN-1 | Connector catalog (`src/server/integrations/catalog.ts`) | spec 11, spec 01 enum |
| CN-2 | Resolver + unit tests (intake → required set) | CN-1, `client_profile` (spec 01) |
| CN-3 | Persist to `onboarding_tasks` (+ `requirement`/`unlocks_modules` cols); re-resolution triggers | CN-2, spec 09 |
| CN-4 | Guided onboarding wizard (`/portal/onboarding`) | CN-3, spec 07 §1.3 ProviderCard |
| CN-5 | Data-driven Integrations page + "Add another" + admin override | CN-3, spec 07 §1 |

**Build order:** CN-1 → CN-2 (pure logic, fully testable before any UI) → CN-3 → CN-5 →
CN-4. CN-1/CN-2 can land in the same PR as the Wix adapter (spec 11) so the pilot onboards
through the tailored flow from day one.

## R-item for the implementation guide

**R-12:** extend `onboarding_tasks` (spec 09) with `requirement` (`required|recommended|optional`)
and `unlocks_modules text[]`, and confirm `client_profile` carries `website_platform`,
`pos_system`, `uses_meta_ads` (intake fields, docs 08) so the resolver has its inputs. Reconcile
at DB-1.
