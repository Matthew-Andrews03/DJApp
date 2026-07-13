# Spec 07 — Portal Pages (client portal + admin switchboard)

**Repo:** `seekly-client-insights` (the portal IS this app — per `docs/10-reusing-seekly-client-insights.md`, which supersedes docs 02/04/05/07 where they conflict). We extend the existing app; we do not build a new one. Every new page follows the app's existing design system and code patterns exactly.

**Audience:** junior developer, implementing without questions. All paths below are real paths in `seekly-client-insights` unless marked **NEW**.

**Depends on sibling specs** (referenced throughout as spec 01/02/04):

| Ref | Spec | What this spec consumes from it |
|---|---|---|
| spec 01 | `specs/01-*.md` (schema migrations) | New Drizzle tables in `src/server/db/schema.ts`: `connections`, `commsProvisioning`, `locations`, `clientProfile`, `workflowConfig`, `events`, `customers`, `optOuts`, `messageLedger` (pg table `messages`), `conversations`, `campaigns`, `campaignMembers`, `contentItems`, `syndicatedPosts`, `competitorSnapshots`, `intelDigests`, `activityLog`, `metricsDaily`, `approvalRequests`, plus response columns added to the existing `reviews` table (`responseBody`, `responseStatus`, `respondedAt`) per doc 10 |
| spec 02 | `specs/02-*.md` (internal API / canonical events / send-pipeline) | `emitEvent()` helper (`src/server/events/emit.ts`), `notifyN8n(webhook, payload)` helper, `reprocessEvent(eventId)`, WF-0 provisioning trigger, WF-6 prospect-run trigger |
| spec 04 | `specs/04-*.md` (OAuth / integration contracts) | Route handlers `GET /api/oauth/google/start`, `GET /api/oauth/google/callback`, `GET /api/oauth/meta/start`, `GET /api/oauth/meta/callback`; connection health monitor that sets `connections.connectionStatus = 'broken'`; GBP location / FB page selection persistence |

If an export name here disagrees with the sibling spec, the sibling spec wins — update the import, not the design.

---

## 0. Conventions — read first, reuse everywhere

### 0.1 Page pattern (server component)

Every portal page copies the shape of `src/app/(portal)/portal/(authed)/review/page.tsx`:

```tsx
import { PageHeader } from "@/components/shell/page-header";
import { requireClientSession } from "@/server/auth/session";
import { getClientById } from "@/server/queries/clients";

export const dynamic = "force-dynamic";

export default async function Page() {
  const s = await requireClientSession();               // defense in depth; proxy.ts also guards
  const client = await getClientById(s.clientId!);      // react cache()'d, 404s if missing
  const [a, b] = await Promise.all([queryA(client.id), queryB(client.id)]);
  return (
    <div className="space-y-5">
      <PageHeader title="…" description="…" />
      {/* view components */}
    </div>
  );
}
```

- Data access ONLY via functions in `src/server/queries/*.ts` (plain async Drizzle selects, always scoped by `clientId`). Never query from components.
- Admin pages use `src/app/(app)/[clientSlug]/...` with `getClientBySlug(clientSlug)` (see `src/app/(app)/[clientSlug]/settings/page.tsx`).
- Filters go through URL `searchParams` (see `src/components/dashboard/filter-bar.tsx` — `useSearchParams` + `router.push`), never client state.

### 0.2 Server action pattern

Copy `src/server/actions/reviews.ts` (`updateReviewEngineConfig`):

```ts
"use server";
import { z } from "zod";
import { revalidatePath } from "next/cache";
import { requireClientSession, requireSession } from "@/server/auth/session";

const schema = z.object({ /* … */ });
export async function actionName(
  input: z.infer<typeof schema>,
): Promise<{ error: string } | { ok: true }> {
  const s = await requireClientSession();          // or requireSession() for admin
  const parsed = schema.safeParse(input);
  if (!parsed.success) return { error: "Human-readable message." };
  // …drizzle writes, always AND-ed with eq(table.clientId, s.clientId!)
  revalidatePath("/portal/…");
  return { ok: true };
}
```

- Client-visible portal mutations also call `writePortalAudit()` (`src/server/audit/portal-audit.ts`) — see `src/server/actions/portal-credentials.ts` for the pattern.
- Cross-engine actions additionally insert an `activityLog` row (spec 01) so the Overview feed and case-study numbers stay defensible.
- Client components call actions inside `useTransition` and surface results with `sonner` `toast.success` / `toast.error`, then `router.refresh()` — see `src/components/credentials/credentials-manager.tsx`.

### 0.3 Design system (from `AGENTS.md` — mandatory)

- **Semantic tokens only**: `bg-background`, `bg-card`, `text-foreground`, `text-muted-foreground`, `bg-primary`, `border-border`. Never `gray-*`/`indigo-*`/`white`.
- Fonts: `font-heading` for display numbers/headings, `font-mono` for eyebrows/labels/badges, default DM Sans body.
- Cards: `rounded-xl border border-border bg-card p-5` (see dashboard page).
- shadcn/radix primitives from `src/components/ui/`: `Button`, `Badge`, `Card`, `Input`, `Label`, `Textarea`, `Select`, `Switch`, `Checkbox`, `Tabs`, `Sheet`, `Dialog`, `Table`, `Tooltip`, `DropdownMenu`, `sonner`.
- Charts: recharts, brand-ink stroke `#1a202c` + gradient fill — copy `src/components/charts/review-volume-chart.tsx` verbatim as the template (including the single-point dot workaround and the empty-state div).
- Loading: the shared `src/app/(portal)/portal/(authed)/loading.tsx` shimmer skeleton already covers all new child routes; only add per-route `loading.tsx` where specified.
- Mobile: sidebar collapses via `src/components/shell/mobile-nav.tsx` (Sheet drawer) automatically; page content must work at 375 px — stack cards, tables get `overflow-x-auto` wrappers, primary tap targets ≥ `h-11`.

### 0.4 Shared plumbing shipped once (ticket UI-1)

1. **Tab registry** — edit `src/lib/portal-tabs.ts` `PORTAL_TAB_REGISTRY` to (icons from `lucide-react`):

```ts
{ key: "dashboard",    label: "Overview",     href: "/portal/dashboard",     icon: LayoutDashboard },
{ key: "approvals",    label: "Approvals",    href: "/portal/approvals",     icon: ClipboardCheck, group: "Growth" },
{ key: "growth",       label: "Growth",       href: "/portal/growth/leads",  icon: TrendingUp,     group: "Growth" },
{ key: "prompts",      … unchanged Insights group … },
{ key: "citations",    … }, { key: "competitors", … }, { key: "sentiment", … },
{ key: "traffic",      … unchanged … },
{ key: "review",       label: "Reputation",   href: "/portal/review",        icon: Star, group: "Reputation" },
{ key: "documents",    … }, { key: "messages", … },
{ key: "integrations", label: "Integrations", href: "/portal/integrations",  icon: Plug,     group: "Account" },
{ key: "settings",     label: "Settings",     href: "/portal/settings",      icon: Settings, group: "Account" },
```

   - `credentials` key is REMOVED from the registry. Keep the route: replace the body of `src/app/(portal)/portal/(authed)/credentials/page.tsx` with `redirect("/portal/integrations")`.
   - Update the `portalConfig` jsonb default in `src/server/db/schema.ts` (~line 172) to the new key list (the comment there says to keep the two in sync). Write a data migration mapping existing rows: replace `"credentials"` with `"integrations"`; new keys (`approvals`, `growth`, `settings`) are opt-in per client via the existing admin `PortalSettings` component (`src/components/settings/portal-settings.tsx`) so half-built modules never leak.
2. **Module labels** — **NEW** `src/lib/modules.ts`:

```ts
export const MODULE_LABELS: Record<string, string> = {
  review_automation: "Review Engine",
  speed_to_lead: "Lead Response",
  reactivation: "Win-Back Campaigns",
  content_engine: "Content Engine",
  directory_sync: "Listings Sync",
  competitor_intel: "Competitor Watch",
  social_syndication: "Social Posts",
  intelligence: "Growth Intelligence",
};
export const PROVIDER_MODULES: Record<string, string[]> = {
  google_business: ["review_automation", "social_syndication", "directory_sync"],
  meta: ["speed_to_lead", "social_syndication"],
  wordpress: ["content_engine"],
  pos: ["review_automation", "reactivation"],
};
```

   Client-facing surfaces use `MODULE_LABELS` values only — internal keys, `WF-*`, `n8n`, "canonical event" never appear in the portal.
3. **StatCard** — **NEW** `src/components/portal/growth-stat-card.tsx` (generic sibling of `src/components/dashboard/stat-cards.tsx`):

```tsx
export function GrowthStatCard({ label, value, sub, icon: Icon }: {
  label: string; value: string; sub?: string;
  icon?: React.ComponentType<{ className?: string }>;
}) {
  return (
    <div className="rounded-xl border border-border bg-card p-4">
      <div className="flex items-center gap-2 text-sm font-medium text-muted-foreground">
        {Icon && <Icon className="size-4" />} {label}
      </div>
      <p className="mt-3 font-heading text-2xl font-bold tracking-tight">{value}</p>
      {sub && <p className="text-xs text-muted-foreground">{sub}</p>}
    </div>
  );
}
```

4. **StatusBadge** — **NEW** `src/components/portal/status-badge.tsx`: maps a status string to `Badge` variants (`default` = good/active, `secondary` = pending/neutral, `destructive` = broken/failed, `outline` = off/none) + `font-mono text-[10px] uppercase tracking-wide` label, mirroring the tone classes in `src/app/(app)/prospecting/page.tsx` `StatusBadge`.
5. **GrowthTabs** — **NEW** `src/components/portal/growth-tabs.tsx`: copy `src/components/portal/portal-traffic-tabs.tsx` with `ITEMS = [{ label: "Leads", href: "/portal/growth/leads" }, { label: "Win-Back", href: "/portal/growth/reactivation" }, { label: "Revenue Report", href: "/portal/growth/revenue-report" }]` and eyebrow `Growth`.
6. **Error boundary** — **NEW** `src/app/(portal)/portal/(authed)/error.tsx` (client component): card with copy *"Something went wrong loading this page. Your data is safe — try refreshing."* + `Button` "Refresh" calling `reset()`. This is the error state for every portal page unless a page-specific one is specified.
7. **Connection banners** — **NEW** `src/components/portal/connection-banners.tsx` (see §1.6) rendered in `src/app/(portal)/portal/(authed)/layout.tsx` above `{children}` inside `PortalShell`.
8. **Approvals nav dot** — `PortalShell` (`src/components/portal/portal-shell.tsx`) gains prop `pendingApprovals: number`; `PortalNav` renders the same red dot it already renders for `messages` unread (`src/components/portal/portal-nav.tsx` ~line 75) on the `approvals` tab when `> 0`. Layout fetches it with `countPendingApprovals(client.id)` (§3).
9. **Money formatter** — add to `src/lib/formatting.ts`:

```ts
export function formatMoney(value: number | string | null | undefined, currency = "CAD"): string {
  if (value === null || value === undefined) return "–";
  const n = typeof value === "string" ? parseFloat(value) : value;
  if (Number.isNaN(n)) return "–";
  return n.toLocaleString("en-US", { style: "currency", currency, maximumFractionDigits: 0 });
}
```

### 0.5 Client-facing copy rules (doc 05 — apply to EVERY page below)

1. **Money first.** Any stat that can be framed as dollars found/saved is: *"12 lapsed customers came back — est. $840"*, never "12 reactivations".
2. **Defensible only.** Every number traces to `activityLog` / `metricsDaily` rows. No projections without "est." prefix; estimates always say what they're based on (average customer value from intake, `client_profile`).
3. **Speed is the hero.** Lead response time is always shown in seconds/minutes with the industry contrast: *"Median reply: 41 seconds (industry average: hours)."*
4. **Plain-owner language.** `MODULE_LABELS` everywhere; "paused" not "disabled"; "needs attention" not "broken"; "we" = Seekly doing work for them: *"We replied to 8 reviews this week."*
5. **Empty states sell the next step**, never apologize: *"No campaigns yet — your first win-back campaign will appear here for approval before anything is sent."*

---

## 1. Integrations page (replaces the credentials tab)

### 1.1 Routes & files

| Item | Path |
|---|---|
| Route | `/portal/integrations` |
| Page | **NEW** `src/app/(portal)/portal/(authed)/integrations/page.tsx` |
| Legacy redirect | `src/app/(portal)/portal/(authed)/credentials/page.tsx` → `redirect("/portal/integrations")` |
| View components | **NEW** `src/components/integrations/provider-card.tsx`, `integrations-view.tsx`, `csv-upload.tsx`, `location-picker-dialog.tsx` |
| CSV endpoint | **NEW** `src/app/api/portal/customers/upload/route.ts` |
| Admin mirror | **NEW** `src/app/(app)/[clientSlug]/integrations/page.tsx` renders the same `IntegrationsView` (shared-body pattern documented in `src/components/reviews/reviews-view.tsx`) plus per-connection raw status |

### 1.2 Data requirements

Tables (spec 01): `connections`, `commsProvisioning`, `clientProfile`, `workflowConfig`, `customers`, `events`. Legacy: `clientCredentials` (existing).

**NEW** `src/server/queries/connections.ts`:

```ts
export interface ConnectionRow {
  id: string; provider: string;                  // 'google_business' | 'meta' | 'wordpress' | …
  connectionStatus: "pending" | "active" | "broken" | "revoked";
  providerAccountId: string | null;
  metadata: Record<string, unknown> | null;      // { gbpLocationName?, pageName?, availableLocations?: […] }
  lastVerifiedAt: Date | null;
}
export async function listConnections(clientId: string): Promise<ConnectionRow[]>;
export async function getCommsProvisioning(clientId: string): Promise<{
  inboundEmailAddress: string | null; phoneNumber: string | null;
  a2pCampaignStatus: "none" | "submitted" | "approved" | "rejected";
} | null>;
/** Broken connections + the client-facing names of modules they pause. */
export async function getBrokenConnections(clientId: string): Promise<
  { provider: string; pausedModules: string[] }[]   // pausedModules = MODULE_LABELS values, filtered to modules with workflowConfig.enabled = true
>;
export async function getCustomerImportStats(clientId: string): Promise<{ count: number; lastImportedAt: Date | null }>;
```

`getBrokenConnections`: join `connections` (`connectionStatus = 'broken'`) × `PROVIDER_MODULES` × `workflowConfig` (`enabled = true`) so the banner only names modules that were actually running. **Never select token columns in any query here.**

### 1.3 Page layout / component tree

```
IntegrationsPage (server)
├─ PageHeader title="Integrations"
│    description="Connect your accounts once — Seekly runs your growth from them."
├─ IntegrationsView (server component, props: connections, comms, profile, importStats)
│   ├─ ProviderCard provider="google"    (client component)
│   ├─ ProviderCard provider="facebook"
│   ├─ ProviderCard provider="website"
│   ├─ ProviderCard provider="pos"
│   └─ ProviderCard provider="phone"
└─ section "Other logins"  — legacy vault, collapsed by default
    └─ CredentialsManager (existing src/components/credentials/credentials-manager.tsx, unchanged)
```

Grid: `grid gap-5 md:grid-cols-2` (cards stack on mobile). `ProviderCard` skeleton:

```tsx
"use client";
export function ProviderCard({ provider, status, detail, children }: {
  provider: "google" | "facebook" | "website" | "pos" | "phone";
  status: "not_connected" | "pending" | "active" | "broken" | "coming_soon";
  detail?: string;                       // e.g. selected GBP location name
  children?: React.ReactNode;            // provider-specific body
}) { /* Card + header row (icon, name, StatusBadge) + body + footer CTA */ }
```

Status → `StatusBadge`: `active` = default badge "Connected", `pending` = secondary "Action needed", `broken` = destructive "Needs attention", `not_connected` = outline "Not connected", `coming_soon` = outline "Coming soon".

### 1.4 Per-provider card content (exact copy)

**Google** (one consent for GBP + Analytics + Search Console — doc 06):
- `not_connected`: body *"Reviews, Business Profile posts, hours updates, and search analytics all flow through this one connection."* CTA `Button asChild` → `<a href="/api/oauth/google/start">Connect Google</a>` (route from spec 04; it derives the client from the portal session and redirects back to `/portal/integrations?connected=google`).
- `pending` (connected, no location chosen — `metadata.gbpLocationId` empty while `metadata.availableLocations` set): body *"Almost done — choose which business location Seekly should manage."* CTA "Choose location" opens `LocationPickerDialog` (radix `Dialog`, radio list of `metadata.availableLocations`, confirm calls `selectGbpLocation` action).
- `active`: detail line *"Managing: {metadata.gbpLocationName}"* + `text-xs text-muted-foreground` *"Last verified {lastVerifiedAt, date-fns format 'MMM d'}"*. Footer link "Disconnect" (ghost, confirm dialog: *"Disconnecting pauses {paused module labels}. Continue?"*) → `disconnectProvider` action.
- `broken`: body *"Google stopped accepting our access — this usually means the password changed or access was revoked. Reconnecting takes under a minute."* CTA "Reconnect Google" → same start URL.

**Facebook / Instagram**: same states; connect href `/api/oauth/meta/start`; pending state picks a Page (`PagePicker` = same `LocationPickerDialog` component, generic title prop). `not_connected` body: *"Connect your Facebook Page to answer lead ads in seconds and keep your page active automatically."*

**Website**: driven by `clientProfile.websitePlatform`:
- `wordpress`: form (Label/Input) — Site URL, Username, Application password (`type="password"`) + help text *"In WordPress: Users → Profile → Application Passwords → create one named 'Seekly'."* Submit → `connectWordpress` action (§1.5). Active state: *"Publishing to {url}"*.
- anything else / unknown: static info card — *"We publish your content by sending ready-to-post drafts to your webmaster — nothing to set up. Want direct publishing? Message us."* + link to `/portal/messages`. Status shows `active` (email_draft path is always live, doc 06).

**POS / Booking** (adapter ladder rungs 3+4, doc 02):
- Always-visible body, three stacked blocks:
  1. *"Forward your point-of-sale receipts"* — `CopyBlock` (existing `src/components/copy-block.tsx`) `label="Forwarding address"` `code={comms.inboundEmailAddress}`. Sub-copy: *"Set your POS or booking system to email (or auto-forward) sale confirmations to this address. Each sale becomes a review request automatically."* If `inboundEmailAddress` is null: muted *"Your private forwarding address is being set up — check back shortly."*
  2. `CsvUpload` (client component): file input (`accept=".csv"`) + Button "Upload customer list" + link `<a href="/customer-import-template.csv" download>` *"Download the CSV template"* (**NEW** static file in `public/`, columns `name,phone,email,last_visit_date`). Shows result line after upload: *"Imported {imported} customers ({skipped} skipped)."* and importStats line *"{count} customers on file · last import {date}"*.
  3. Native adapter status: if a `connections` row with `provider = 'pos'` exists show its status; else muted text *"Using email + CSV (works with every system). A direct connection can be added later."*
- Card status = `active` if any of: inbound address set AND ≥1 `sale.completed` event processed, or customers imported, or native adapter active; else `pending` with badge copy "Action needed".

**Phone** (v1.1 placeholder): status `coming_soon`. Body: *"Missed-call text-back is coming soon: when you can't answer, Seekly texts the caller back before they call your competitor."* If `comms.phoneNumber` set, show *"Your Seekly number: {phoneNumber}"* + A2P line: `a2pCampaignStatus`
- `approved` → *"Text messaging: approved and active."*
- `submitted` → *"Text messaging: carrier approval in progress (this takes days, not minutes — we're on it)."*
- `none`/`rejected` → *"Text messaging: registration pending — your Seekly team is handling it."*
No buttons.

### 1.5 Server actions — **NEW** `src/server/actions/integrations.ts`

```ts
"use server";
const selectSchema = z.object({ connectionId: z.string().uuid(), selectionId: z.string().min(1) });
export async function selectGbpLocation(input): Promise<{ error: string } | { ok: true }>
// requireClientSession → ownership check (connection.clientId === s.clientId) →
// set metadata.gbpLocationId/-Name from availableLocations, connectionStatus='active',
// writePortalAudit(action:'integration.location_selected'), insert activityLog
// {engine:'integrations', action:'connection.activated', detail:{provider}}, revalidatePath('/portal/integrations')

export async function selectMetaPage(input /* same schema */)

const wpSchema = z.object({
  url: z.string().url(), username: z.string().min(1), applicationPassword: z.string().min(8),
});
export async function connectWordpress(input): Promise<{ error: string } | { ok: true }>
// store secret via the existing vault path (storeSecret in src/server/vault/bitwarden.ts, exactly as
// src/server/actions/portal-credentials.ts does), insert clientCredentials row (platform:'wordpress'),
// upsert connections row { provider:'wordpress', connectionStatus:'active', metadata:{ url } }

const disconnectSchema = z.object({ connectionId: z.string().uuid() });
export async function disconnectProvider(input)
// sets connectionStatus='revoked'; token deletion + provider revoke happen in spec 04's helper
// revokeConnection(connectionId) — call it, don't reimplement.
```

Error copy: invalid WP input → `{ error: "Check the site URL and application password and try again." }`.

**CSV route** `src/app/api/portal/customers/upload/route.ts` (multipart → route handler, not action; pattern: existing `src/app/api/portal/documents/` handlers):
- `requireClientSession()`; 400 if file > 2 MB or not parseable.
- Parse with **NEW** `src/lib/csv.ts` (tiny RFC-4180 parser, no new dependency). Required header `phone`; optional `name,email,last_visit_date`.
- For each row: normalize phone (strip non-digits, `+1` default); upsert `customers` on `(clientId, phone)`; set `consentBasis = 'imported_list'`, `source = 'csv'`.
- Emit one `customer.imported` canonical event per batch via spec 02 `emitEvent` (payload `{count}`), insert `activityLog` row `{engine:'integrations', action:'customers.imported', detail:{imported, skipped}}`.
- Response JSON: `{ imported: number, skipped: number, errors: string[] }` (max 10 error strings, e.g. `"row 7: missing phone"`).

### 1.6 Connection-broken banners (rendered on every portal page)

`src/components/portal/connection-banners.tsx` (server component; layout passes `await getBrokenConnections(client.id)`):

```tsx
// one banner per broken provider
<div className="mb-4 flex flex-col gap-2 rounded-xl border border-destructive/40 bg-destructive/10 p-3 sm:flex-row sm:items-center">
  <AlertTriangle className="size-4 shrink-0 text-destructive" />
  <p className="flex-1 text-sm">
    <span className="font-medium">Google connection needs attention.</span>{" "}
    Review Engine and Social Posts are paused until it&apos;s reconnected.
  </p>
  <Button size="sm" asChild><Link href="/portal/integrations">Fix it</Link></Button>
</div>
```

Copy template: `"{Provider} connection needs attention. {module labels, comma+and} {is/are} paused until it's reconnected."` Providers display as Google / Facebook / Website / POS.

### 1.7 States

- **Loading:** covered by shared `(authed)/loading.tsx`.
- **Empty (no comms row, no connections):** all cards render in `not_connected` (POS shows the "being set up" line) — the page is never blank.
- **Error:** shared `error.tsx`. CSV upload failures render inline under the input in `text-destructive text-sm`, not a toast: *"That file couldn't be read — use the CSV template above."*
- **Callback feedback:** on mount (client wrapper in `IntegrationsView`), read `searchParams`: `?connected=google` → `toast.success("Google connected.")`; `?error=oauth_denied` → `toast.error("Connection was cancelled — nothing was changed.")` then strip params via `router.replace`.

### 1.8 Mobile

Single-column cards; `CopyBlock` already scrolls horizontally; dialogs become near-fullscreen by default shadcn behavior; CTA buttons `w-full sm:w-auto`.

### 1.9 Acceptance criteria

1. `/portal/credentials` 302s to `/portal/integrations`; old vault entries remain manageable under "Other logins".
2. Each provider card shows exactly one of the five states, derived only from `connections` + `commsProvisioning` rows (no hardcoded state).
3. "Connect Google" navigates to `/api/oauth/google/start`; after the spec 04 callback the card is `pending` until a location is chosen, then `active` with the location name.
4. Setting a connection's `connectionStatus` to `'broken'` in the DB makes (a) the card show "Needs attention" and (b) the banner appear on dashboard/review/growth pages, naming only modules whose `workflowConfig.enabled = true`.
5. Uploading the template CSV with 3 rows (1 duplicate phone) reports `imported: 2, skipped: 1`, creates `customers` rows, one `customer.imported` event, one `activityLog` row.
6. No query in `src/server/queries/connections.ts` selects `encryptedAccessToken`/`encryptedRefreshToken` (grep-verifiable).
7. All copy strings match §1.4/§1.6 exactly.

---

## 2. Growth section

Three sibling pages under one nav item, joined by `GrowthTabs` (§0.4.5) — identical UX shape to the Traffic cluster (`/portal/traffic*`).

## 2a. Leads (conversations)

### Routes & files

| Item | Path |
|---|---|
| List route | `/portal/growth/leads` — **NEW** `src/app/(portal)/portal/(authed)/growth/leads/page.tsx` |
| Detail route | `/portal/growth/leads/[id]` — **NEW** `…/growth/leads/[id]/page.tsx` |
| Components | **NEW** `src/components/growth/leads-view.tsx`, `lead-filters.tsx`, `transcript-view.tsx`, `lead-detail.tsx` |

### Data requirements

Tables (spec 01): `conversations`, `messageLedger`, `customers`, `metricsDaily`.

**NEW** `src/server/queries/conversations.ts`:

```ts
export type ConversationStatus = "active" | "qualified" | "booked" | "handed_off" | "cold" | "closed";
export interface LeadFilters { clientId: string; rangeDays: number; status: ConversationStatus | null; source: string | null; }

export interface LeadListRow {
  id: string; customerName: string | null; leadSource: string;      // 'website_form' | 'meta_lead_ad' | …
  status: ConversationStatus; firstResponseSeconds: number | null;
  booked: boolean; createdAt: Date; lastMessageAt: Date | null; lastMessagePreview: string | null;
}
export async function listLeads(f: LeadFilters): Promise<LeadListRow[]>;            // join customers, order by createdAt desc, limit 100

export interface LeadStats {
  leads: number; medianResponseSeconds: number | null;                              // percentile_cont(0.5) over first_response_seconds
  underOneMinute: number; qualified: number; booked: number;
}
export async function getLeadStats(clientId: string, rangeDays: number): Promise<LeadStats>;

export interface TranscriptMessage { id: string; direction: "in" | "out"; body: string; createdAt: Date;
  status: string; }                                                                 // from messageLedger where conversationId = …
export interface LeadDetail extends LeadListRow {
  customerPhoneMasked: string | null;      // "(613) •••-4821" — mask in SQL/TS, raw number never sent to the client bundle
  context: Record<string, unknown> | null; // extracted answers: { partySize: 8, date: "Friday", occasion: "birthday" }
  outcome: string | null;
}
export async function getLead(id: string, clientId: string): Promise<LeadDetail | null>;
export async function getTranscript(conversationId: string, clientId: string): Promise<TranscriptMessage[]>;
export async function listLeadSources(clientId: string): Promise<string[]>;         // distinct leadSource, for the filter
```

### List page component tree

```
LeadsPage (server; searchParams: { range?, status?, source? })
├─ GrowthTabs
├─ PageHeader title="Leads" description="Every enquiry, answered in seconds — day or night."
├─ hero row: grid grid-cols-2 gap-4 lg:grid-cols-4
│   ├─ GrowthStatCard label="Median reply time" value="41s" sub="industry average: hours"   ← THE hero
│   ├─ GrowthStatCard label="Leads" value="23" sub="last 30 days"
│   ├─ GrowthStatCard label="Answered < 1 min" value="21 of 23"
│   └─ GrowthStatCard label="Booked" value="9" sub="from these leads"
├─ LeadFilters (client) — two Selects, exactly the FilterBar pattern (src/components/dashboard/filter-bar.tsx):
│     range 7/30/90 (default 30), status All/Active/Qualified/Booked/Handed off/Cold, source All + listLeadSources
└─ LeadsView (server) — rounded-xl border card containing a divide-y list; each row is a <Link href={`/portal/growth/leads/${id}`}>
      row: name (or "New lead") · StatusBadge · source label · "replied in 38s" (font-mono text-xs) ·
           lastMessagePreview truncated (truncate util, 90 chars) · relative time
```

Response-time formatting helper in `src/lib/formatting.ts`: `formatResponseTime(s: number | null)` → `"38s"`, `"4m 12s"`, `"–"`; ≥ 1 h renders `"1h+"`.

### Detail page component tree

```
LeadDetailPage (server; params.id; notFound() if getLead returns null)
├─ back link: <Link href="/portal/growth/leads" className="text-sm text-muted-foreground">← All leads</Link>
├─ PageHeader title={customerName ?? "New lead"} description={`${sourceLabel} · ${format(createdAt, "MMM d, h:mm a")}`}
├─ grid lg:grid-cols-3 gap-5
│   ├─ lg:col-span-2: TranscriptView — read-only bubble thread; copy the bubble markup from
│   │    src/components/portal/message-thread.tsx lines 95–119 verbatim (own = direction "out" =
│   │    bg-primary bubble labeled "Seekly AI"; "in" = customer). No composer.
│   └─ side column cards:
│       ├─ "Response" card: firstResponseSeconds hero (font-heading text-3xl) + status + booked check line
│       ├─ "What they asked for" card: context entries as dl rows (skip if context null)
│       └─ "Need to step in?" card: "Reply to this customer yourself — just text them back from
│            your phone. Seekly hands off automatically when you do." (static copy, no action in v1)
```

### States / mobile / copy

- **Empty list:** *"No leads in this period yet. When someone fills out your website form or a Facebook lead ad, Seekly texts them back within a minute — and the conversation shows up here."*
- **Transcript empty (event logged, send blocked):** *"We logged this lead but messaging is paused ({reason})."* where reason maps `messageLedger.blockedReason`: `a2p_pending` → "text registration in progress", `opt_out` → "this number opted out", `quiet_hours` → "queued for morning".
- **Loading:** shared skeleton. **Error:** shared boundary.
- **Mobile:** stat grid `grid-cols-2`; detail grid stacks (transcript first); rows show name+badge on line 1, meta on line 2.

### Acceptance criteria

1. Median reply stat equals `percentile_cont(0.5)` over `first_response_seconds` of conversations in range (verified with seeded fixtures; null-safe when no leads).
2. Filters round-trip through the URL (`?range=90&status=booked&source=meta_lead_ad`) and are shareable/bookmarkable.
3. Detail page 404s for a conversation id belonging to another client (`getLead` scopes by clientId).
4. Raw customer phone numbers never appear in the HTML payload — masked format only.
5. Transcript renders in < 1 query per table (no N+1: one `getLead`, one `getTranscript`).

## 2b. Reactivation (campaigns)

### Routes & files

| Item | Path |
|---|---|
| List | `/portal/growth/reactivation` — **NEW** `…/growth/reactivation/page.tsx` |
| Detail | `/portal/growth/reactivation/[id]` — **NEW** `…/growth/reactivation/[id]/page.tsx` |
| Components | **NEW** `src/components/growth/campaign-list.tsx`, `campaign-detail.tsx`, `campaign-funnel.tsx`, `approve-campaign.tsx` |
| Actions | **NEW** `src/server/actions/campaigns.ts` |

### Data requirements

Tables (spec 01): `campaigns`, `campaignMembers`, `customers`, `clientProfile` (average customer value for est. revenue framing).

**NEW** `src/server/queries/campaigns.ts`:

```ts
export type CampaignStatus = "draft" | "approved" | "sending" | "paused" | "done";
export interface CampaignListRow { id: string; angle: string; status: CampaignStatus;
  audienceCount: number; stats: { sent: number; replies: number; stops: number; bookings: number; revenueEst: number } | null;
  createdAt: Date; }
export async function listCampaigns(clientId: string): Promise<CampaignListRow[]>;
export interface CampaignDetail extends CampaignListRow {
  copyVariants: { label: string; body: string }[];                     // from campaigns.copy_variants jsonb
  audienceFilter: Record<string, unknown> | null;
}
export async function getCampaign(id: string, clientId: string): Promise<CampaignDetail | null>;
export async function getCampaignFunnel(id: string, clientId: string): Promise<{
  queued: number; sent: number; replied: number; booked: number; excluded: number; }>; // group-by campaignMembers.status
```

### List page

```
ReactivationPage (server)
├─ GrowthTabs
├─ PageHeader title="Win-Back Campaigns"
│    description="Lapsed customers, invited back with offers you approved."
├─ if any campaign.status === 'draft': AwaitingApprovalCallout — rounded-xl border-primary/40 bg-primary/5 card:
│    "A campaign is waiting for your approval." + Button asChild → detail page "Review & approve"
└─ CampaignList — card per campaign (space-y-3):
     angle title · StatusBadge (draft="Awaiting your approval" secondary; approved="Scheduled";
     sending="Sending"; paused="Paused"; done="Complete") ·
     stats line when present: "142 sent · 19 replies · 6 bookings — est. $2,140 back in the till"
     (formatMoney(stats.revenueEst)) · <Link> wraps card → detail
```

### Detail page

```
CampaignDetailPage (server)
├─ back link "← All campaigns"
├─ PageHeader title={angle} description={statusDescription}
├─ status === 'draft':
│   └─ ApproveCampaign (client):
│       ├─ audience card: "This will go to {audienceCount} customers who haven't visited in {threshold}+ days.
│       │    Nobody who opted out or was messaged recently is included."
│       ├─ message preview cards: one per copyVariant — bubble-styled (reuse TranscriptView bubble classes),
│       │    heading = variant.label (font-mono text-xs uppercase)
│       └─ sticky action bar (sticky bottom-0 bg-background/95 border-t border-border p-3 flex gap-2):
│           Button size="lg" className="flex-1" → "Approve & schedule"
│           Button variant="outline" size="lg" → "Request changes" (opens Dialog with Textarea, sends note)
├─ status !== 'draft':
│   ├─ CampaignFunnel: 4 GrowthStatCards (Queued→Sent→Replied→Booked) + revenue line
│   │    "est. {formatMoney} recovered — based on your average customer value" (from clientProfile)
│   └─ copy variants shown read-only below
```

### Server actions (`src/server/actions/campaigns.ts`)

```ts
"use server";
const approveSchema = z.object({ campaignId: z.string().uuid() });
export async function approveCampaign(input): Promise<{ error: string } | { ok: true }> {
  // requireClientSession; load campaign scoped by clientId; must be status 'draft'
  //   else return { error: "This campaign was already handled." }
  // update campaigns.status = 'approved'
  // insert activityLog { engine:'reactivation', action:'campaign.approved', entityType:'campaign', entityId }
  // writePortalAudit({ action:'campaign.approve' })
  // fire-and-forget notifyN8n('reactivation/campaign-approved', { clientId, campaignId })  // spec 02; WF-3 also
  //   polls status so a missed webhook is not fatal
  // revalidatePath('/portal/growth/reactivation')
}
const changesSchema = z.object({ campaignId: z.string().uuid(), note: z.string().min(1).max(2000) });
export async function requestCampaignChanges(input)
// posts the note onto the existing portal message thread via the same insert used by
// sendClientMessage (src/server/actions/portal-messages.ts) prefixed
// "Campaign change request ({angle}): " — Seekly ops sees it in admin Messages; campaign stays 'draft'.
```

`ApproveCampaign` disables both buttons in `useTransition`, toasts *"Campaign approved — sends begin within the allowed texting window."* and `router.refresh()`.

### States / mobile / copy

- **Empty:** *"No campaigns yet — your first win-back campaign will appear here for approval before anything is sent."*
- **Paused (STOP-rate auto-pause, doc 03):** description under title: *"Paused by our safety checks — your Seekly team is reviewing it."*
- **Mobile:** sticky action bar keeps Approve reachable with one thumb; funnel cards `grid-cols-2`.

### Acceptance criteria

1. Approve is only possible from `draft`; double-submit returns the "already handled" error and the UI refreshes to the approved state.
2. Approving writes exactly one `campaigns` update + one `activityLog` row + one portal audit row, and calls `notifyN8n` once.
3. Funnel numbers come from `campaignMembers` group-by, not `campaigns.stats` (stats jsonb is display-fallback only when members are empty).
4. Revenue line always carries "est." and the basis phrase.
5. "Request changes" lands the note in the existing Messages thread (visible at `/portal/messages` and admin `/{clientSlug}/messages`).

## 2c. Revenue Leakage report

### Routes & files

- `/portal/growth/revenue-report` — **NEW** `…/growth/revenue-report/page.tsx`; components **NEW** `src/components/growth/leakage-report.tsx`.

### Data requirements

Table (spec 01): `intelDigests` with discriminator `kind: 'competitor' | 'revenue_leakage'` (**spec 01 delta — flagged in open questions**). Monthly artifact rows are written by the intelligence job (doc 09); this page only renders.

**NEW** in `src/server/queries/intel.ts`:

```ts
export interface LeakageReport { id: string; period: string;          // '2026-07'
  bodyHtml: string | null;
  findings: {
    recovered: { label: string; amountEst: number; detail: string }[];
    leaking:   { label: string; amountEst: number; detail: string }[];
    totalRecoveredEst: number; totalLeakingEst: number;
  } | null;
  sentAt: Date | null; }
export async function listLeakageReports(clientId: string): Promise<Pick<LeakageReport, "id" | "period">[]>;
export async function getLeakageReport(clientId: string, period: string | null): Promise<LeakageReport | null>; // null period = latest
```

### Component tree

```
RevenueReportPage (server; searchParams: { period? })
├─ GrowthTabs
├─ PageHeader title="Revenue Report"
│    description="What was leaking, what Seekly recovered, and what's still on the table."
│    actions={<PeriodSelect />}   ← Select of listLeakageReports periods, FilterBar URL pattern
└─ LeakageReport
    ├─ two hero GrowthStatCards side by side:
    │    "Recovered this month" value=formatMoney(totalRecoveredEst) (default badge tone)
    │    "Still recoverable" value=formatMoney(totalLeakingEst)
    ├─ "What Seekly recovered" card — list rows: label · detail · amount right-aligned font-mono
    ├─ "Where money is still leaking" card — same rows + per-row CTA link when detail maps to a page
    │    (unanswered leads → /portal/growth/leads, review gap → /portal/review, lapsed → /portal/growth/reactivation)
    └─ bodyHtml (when findings null): rendered via <div className="report-prose" dangerouslySetInnerHTML …>
         — trusted internal pipeline output only; add a .report-prose block to src/app/globals.css
         (h2/h3/p/ul spacing using semantic tokens; there is no typography plugin in this repo)
```

### States / copy / AC

- **Empty (no reports yet):** *"Your first revenue report lands after your first full month — it shows exactly what Seekly found and fixed, in dollars."*
- **Mobile:** hero cards stack; rows wrap amount below label.
- AC: 1. `?period=2026-06` renders that month; invalid period falls back to latest. 2. findings-based render preferred over bodyHtml when both exist. 3. Every amount uses `formatMoney` and an "est." qualifier appears in each hero card's sub-line (*"estimated from your average customer value"*).

---

## 3. Approvals — unified queue

### 3.1 Routes & files

| Item | Path |
|---|---|
| Queue | `/portal/approvals` — **NEW** `src/app/(portal)/portal/(authed)/approvals/page.tsx` |
| Deep link (session) | `/portal/approvals/[id]` — **NEW** `…/approvals/[id]/page.tsx` (single item, same card UI) |
| Deep link (token, no session — SMS/email) | `/a/[token]` — **NEW** `src/app/a/[token]/page.tsx` |
| Components | **NEW** `src/components/approvals/approvals-queue.tsx`, `approval-card.tsx`, `approval-edit-dialog.tsx` |
| Actions | **NEW** `src/server/actions/approvals.ts` |
| Route guard | add `"/a"` to `PUBLIC_PATHS` in `src/proxy.ts` (same reasoning as the existing public token page `src/app/reviews/[feedbackToken]/page.tsx`) |

### 3.2 Data model (spec 01)

`approvalRequests` — one row per thing awaiting approval, created by whichever engine set the underlying entity to pending:

```
approval_requests: id uuid pk, client_id fk, kind ('review_response'|'gbp_post'|'content_draft'|'campaign'),
entity_id text, title, body text, meta jsonb, token uuid unique default random,
status ('pending'|'approved'|'rejected'|'expired'), requested_at, resolved_at, resolved_by ('portal'|'token'|'admin')
```

The queue reads ONLY this table; resolving fans out to the entity (below). This keeps the page one query and gives SMS deep links a token home (mirrors `reviewRequests.feedbackToken`).

**NEW** `src/server/queries/approvals.ts`:

```ts
export type ApprovalKind = "review_response" | "gbp_post" | "content_draft" | "campaign";
export interface ApprovalItem { id: string; kind: ApprovalKind; title: string; body: string;
  meta: Record<string, unknown>;      // review_response: { rating, author, reviewText } · gbp_post: { imageUrl? } · campaign: { audienceCount }
  requestedAt: Date; entityId: string; }
export async function listPendingApprovals(clientId: string): Promise<ApprovalItem[]>;   // status='pending', order requested_at asc
export async function countPendingApprovals(clientId: string): Promise<number>;          // used by the layout nav dot (§0.4.8)
export async function getApproval(id: string, clientId: string): Promise<ApprovalItem | null>;
export async function getApprovalByToken(token: string): Promise<(ApprovalItem & { status: string; clientId: string; clientName: string }) | null>;
```

### 3.3 Queue page component tree (mobile-first)

```
ApprovalsPage (server)
├─ PageHeader title="Approvals" description="Everything here waits for your OK before it goes out."
└─ ApprovalsQueue (client; props: initialItems)
    └─ per item: ApprovalCard
        ├─ header: kind chip (font-mono text-[10px] uppercase — "Review reply" / "Business Profile post" /
        │          "Blog draft" / "Win-back campaign") + relative time
        ├─ context block (kind-specific):
        │    review_response → the customer review quoted: Stars row (copy the Stars component from
        │       src/components/reviews/reviews-view.tsx) + meta.author + meta.reviewText, in a muted inset
        │       (rounded-md bg-muted p-3 text-sm)
        │    campaign → "Goes to {meta.audienceCount} lapsed customers"
        ├─ body: the draft text (whitespace-pre-wrap text-sm; content_draft shows title + first 400 chars +
        │        "Read full draft" expanding <details>)
        └─ action row (flex gap-2, each button h-11, flex-1 on mobile):
             Button "Approve" · Button variant="outline" "Edit" · Button variant="ghost" className="text-destructive" "Reject"
```

**Optimistic updates:** `ApprovalsQueue` keeps `items` in `useState`; on any action it removes the card immediately, calls the server action in `startTransition`; on `{ error }` it re-inserts the card at its old index and `toast.error(error)`; on success `toast.success` (Approve: *"Approved — it's on its way."* Reject: *"Rejected — we won't send it."*).

**Edit:** `ApprovalEditDialog` — `Dialog` with `Textarea` prefilled with `body` (`maxLength` 4000; 1500 for `gbp_post` with live counter `"{n}/1500"`), buttons "Save & approve" / "Cancel".

### 3.4 Deep links

- SMS/email notifications (sent by engines, spec 02) link to `https://app…/a/{token}`.
- `/a/[token]/page.tsx`: no session required. `getApprovalByToken`; if `null` → generic 404 copy *"This link is no longer valid."*; if `status !== 'pending'` → resolved card *"Already handled — this {kind label} was {approved/rejected} on {date}."*; else render a single `ApprovalCard` (same component) + footer link *"See all approvals"* → `/portal/approvals` (goes through login). Actions from this page call the same server actions with `{ token }` instead of `{ id }` — the token IS the authorization (single item, unguessable uuid, expires with `status`), `resolved_by = 'token'`.
- `/portal/approvals/[id]`: session-scoped single item — used by in-portal notifications and as the canonical share URL.

### 3.5 Server actions (`src/server/actions/approvals.ts`)

```ts
"use server";
const resolveSchema = z.object({
  id: z.string().uuid().optional(), token: z.string().uuid().optional(),
  editedBody: z.string().min(1).max(4000).optional(),
}).refine((v) => !!v.id !== !!v.token, { message: "id xor token" });

export async function approveItem(input): Promise<{ error: string } | { ok: true }>
export async function rejectItem(input): Promise<{ error: string } | { ok: true }>
```

Shared internals (`resolveApproval(kind fan-out)`):
- Auth: `id` path → `requireClientSession()` + clientId ownership check; `token` path → row lookup by token only (no session), rate-limited per IP via the existing helper pattern in `src/server/auth/rate-limit.ts`.
- Guard: row must be `pending`, else `{ error: "Already handled — nothing was changed." }` (idempotent; safe on SMS double-taps).
- Fan-out on approve (single transaction):
  - `review_response` → update `reviews` row (`responseBody = editedBody ?? body`, `responseStatus = 'approved'`) — publishing to GBP is the engine's job (WF-1 picks up approved rows; interim ops-manual path per doc 06).
  - `gbp_post` → `syndicatedPosts.status = 'approved'` (+ body override).
  - `content_draft` → `contentItems.status = 'approved'`.
  - `campaign` → same effect as `approveCampaign` (§2b) — call the shared internal helper, don't duplicate.
- Always: `approvalRequests.status`, `resolvedAt`, `resolvedBy`; `activityLog` row `{engine:'approvals', action:'{kind}.{approved|rejected}'}`; `notifyN8n('approvals/resolved', { clientId, kind, entityId, status })` fire-and-forget; `revalidatePath('/portal/approvals')`.

### 3.6 States / mobile / AC

- **Empty queue:** centered card — CheckCircle icon (`text-muted-foreground`), *"You're all caught up. Anything that needs your OK — review replies, posts, campaigns — will wait here (and we'll text you a link)."*
- **Loading:** **NEW** `…/approvals/loading.tsx` — three `h-40 rounded-xl` shimmer Blocks (copy the `Block` helper inline per the ponytail note in `(authed)/loading.tsx`).
- **Mobile:** cards full-width; action row buttons `flex-1`; this page is the SMS landing zone — target one-thumb approve within 2 taps of the text message.

AC:
1. All four kinds render with their kind-specific context block from a seeded `approvalRequests` fixture set.
2. Approve/reject from the queue optimistically removes the card in < 100 ms perceived; a forced action error restores it in place with a toast.
3. `/a/{token}` works logged-out, resolves the item, and a second visit shows "Already handled"; a bogus token 404s.
4. Editing a `gbp_post` body beyond 1500 chars is impossible (counter + maxLength + zod re-check server-side).
5. Approving a `review_response` never calls a Google API from the portal process — it only flips DB state (engine owns publishing).
6. Nav dot on the Approvals tab appears when `countPendingApprovals > 0` and clears after resolving (router.refresh on the layout via revalidatePath).

---

## 4. Overview upgrade (existing dashboard page, extended — not replaced)

### 4.1 Route & files

- Route stays `/portal/dashboard`; nav label becomes **Overview** (§0.4.1).
- Edit `src/app/(portal)/portal/(authed)/dashboard/page.tsx`.
- Components: **NEW** `src/components/portal/overview-stats.tsx`, `opportunities-card.tsx`, `activity-feed.tsx`. Existing SoV widgets are kept as-is: `StatCards` (engine visibility), `VisibilityChart`, `AvgVisibilityList`, `CompetitorHeatmap`, `TrafficSummary`.

### 4.2 Data requirements

Tables (spec 01): `activityLog`, `metricsDaily`, `conversations`, `customers`, plus everything the page already reads (`dailyAggregates` etc. via `src/server/queries/dashboard.ts` and `src/server/queries/visitors.ts` — unchanged).

**NEW** `src/server/queries/activity.ts`:

```ts
export interface OverviewStats {                      // from metricsDaily, summed over rangeDays
  reviewRequestsSent: number; reviewsReceived: number;
  leadsAnswered: number; medianResponseSeconds: number | null;
  revenueRecoveredEst: number;                        // sum of module metrics' revenue_est fields
}
export async function getOverviewStats(clientId: string, rangeDays: number): Promise<OverviewStats>;

export async function getGrowthScore(clientId: string): Promise<{ score: number | null; delta: number | null }>;
// reads latest metricsDaily row where module='growth' → metrics.score (0–100) and metrics.score_delta_30d;
// the nightly rollup job (spec 02) computes it — this query never computes, only reads.

export interface ActivityRow { id: string; engine: string; action: string;
  detail: Record<string, unknown> | null; createdAt: Date; }
export async function listActivity(clientId: string, limit = 20): Promise<ActivityRow[]>;  // status='ok' only

export interface Opportunity { key: "unanswered_leads" | "lapsed_customers" | "review_gap";
  count: number; amountEst: number | null; href: string; }
export async function getOpportunities(clientId: string): Promise<Opportunity[]>;
// three cheap SQL counts: conversations active w/ no outbound in 24h; customers lapsed past
// clientProfile threshold; happy visits (sale.completed events, 30d) minus review requests sent.
// amountEst = count × clientProfile average customer value where meaningful, else null. Returns
// only rows with count > 0, max 3.
```

### 4.3 Merged layout (top → bottom)

```
OverviewPage (server)
├─ PageHeader title="Overview"
├─ 1. Growth hero row — grid grid-cols-2 gap-4 lg:grid-cols-4 (GrowthStatCard ×4):
│     "Growth score"        value="72" sub="up 4 this month"          (getGrowthScore; hide card if score null)
│     "Recovered this month" value=formatMoney(revenueRecoveredEst) sub="est., from your activity log"
│     "Leads answered"       value="23" sub="median reply 41s"
│     "Reviews"              value="+8" sub="14 requests sent"
├─ 2. grid gap-5 lg:grid-cols-3
│     ├─ OpportunitiesCard (lg:col-span-1) — "Found money" heading; per opportunity one row:
│     │     unanswered_leads: "3 leads waiting on a reply — don't let them go cold" → /portal/growth/leads
│     │     lapsed_customers: "38 customers haven't been back in 90+ days — est. {money} on the table" → /portal/growth/reactivation
│     │     review_gap: "11 happy visits didn't leave a review last month" → /portal/review
│     │     empty: "Nothing leaking right now — Seekly is watching."
│     └─ ActivityFeed (lg:col-span-2) — "Recent activity"; divide-y rows: engine dot (size-2 rounded-full
│           bg-primary) · humanized line · relative time. Humanizer: NEW src/lib/activity-copy.ts
│           mapping action → template, e.g. review_request.sent → "Asked {detail.name ?? "a customer"} for a review",
│           review_response.published → "Replied to a {detail.rating}★ review",
│           lead.first_response → "Answered a new {sourceLabel} lead in {detail.seconds}s",
│           campaign.batch_sent → "Sent {detail.count} win-back messages". Unknown actions render
│           "{MODULE_LABELS[engine]} ran" — never raw keys.
├─ 3. section heading "AI Visibility" (h3 pattern from current page) — the EXISTING SoV block, unchanged:
│     StatCards(engineStats) → VisibilityChart + AvgVisibilityList grid → CompetitorHeatmap card
└─ 4. existing "Website traffic" TrafficSummary section, unchanged
```

MVP cut (doc 05, 2-week milestone): ship rows 1 (minus growth score), 2, 3, 4 — growth-score card appears automatically once the rollup job writes `module='growth'` rows.

### 4.4 States / mobile / AC

- **Empty activity:** *"Activity will appear here as soon as your first engine switches on."*
- New sections render zero-shapes (cards with "–") when `metricsDaily` is empty — never hide row 1 (it is the renewal pitch).
- Mobile: hero row `grid-cols-2`; opportunities above activity (source order already does this).

AC:
1. Existing SoV widgets render byte-identical to today (no prop changes) below the new growth rows.
2. Every number in rows 1–2 is reproducible by SQL over `metricsDaily`/`activityLog`/`conversations` fixtures (test with seeded data).
3. Opportunity rows deep-link to the named pages and disappear at count 0.
4. Activity feed shows humanized strings only — a fixture row with an unknown `action` renders the fallback, not the key.
5. Page still renders (rows 3–4 fine, rows 1–2 zero-shapes) when ALL new tables are empty — safe to deploy before engines go live.

---

## 5. Settings / Profile

### 5.1 Routes & files

- `/portal/settings` — **NEW** `src/app/(portal)/portal/(authed)/settings/page.tsx`.
- Components: **NEW** `src/components/portal/business-profile-form.tsx`, `hours-editor.tsx`, `holiday-hours-editor.tsx`, `brand-voice-card.tsx`, `team-users.tsx`.
- Actions: **NEW** `src/server/actions/business-profile.ts`, plus `invitePortalUser` client-side variant in **NEW** `src/server/actions/team.ts`.

### 5.2 Data requirements

Tables (spec 01): `locations` (NAP, `hoursJson`, `holidayHoursJson`), `clientProfile` (`brandVoice`), `events` (profile.updated), existing `clientUsers`.

Queries: **NEW** `src/server/queries/locations.ts` → `getPrimaryLocation(clientId)` (`primary = true`, else first); reuse existing `listClientUsers` (`src/server/queries/portal.ts`).

### 5.3 Component tree

```
SettingsPage (server)
├─ PageHeader title="Settings" description="Your business details — changes sync to Google and the directories automatically."
├─ BusinessProfileForm (client; initial from locations row)
│   ├─ Name (Input) · Phone (Input) · Address (street/city/province/postal Inputs, grid sm:grid-cols-2)
│   ├─ HoursEditor: 7 rows (Mon–Sun): day label · Switch "Open" · two <Input type="time"> (open/close),
│   │     inputs disabled when the day's Switch is off. Value shape { mon: {open:"09:00", close:"21:00"} | null, … }
│   ├─ HolidayHoursEditor: list of rows { date (Input type="date"), label (Input, placeholder "Canada Day"),
│   │     closed (Checkbox) | open/close times } + Button variant="ghost" "+ Add holiday hours" + per-row remove (X)
│   └─ footer: Button "Save changes" + muted note "Saving updates your Google profile and 15+ directories.
│        Changes can take a few days to appear everywhere."
├─ BrandVoiceCard (server, read-only — client view of clientProfile.brandVoice)
│   ├─ dl rows: Tone · Phrases we use · Phrases we avoid · Persona name (render only keys present)
│   └─ footer: "Want it to sound different? <Link href='/portal/messages'>Message your Seekly team</Link> —
│        we'll retune it." (clients never edit voice directly; Seekly does — doc 01 "managed" principle)
└─ TeamUsers (client; initial listClientUsers)
    ├─ Table: Email · StatusBadge (pending="Invited" secondary / active="Active" default) · Last sign-in
    └─ invite row: Input type="email" + Button "Invite teammate"
```

### 5.4 Server actions

`src/server/actions/business-profile.ts`:

```ts
"use server";
const timeRe = /^([01]\d|2[0-3]):[0-5]\d$/;
const dayShape = z.object({ open: z.string().regex(timeRe), close: z.string().regex(timeRe) }).nullable();
const schema = z.object({
  name: z.string().min(1).max(200),
  phone: z.string().min(7).max(20),
  address: z.object({ street: z.string().min(1), city: z.string().min(1),
    region: z.string().min(1), postal: z.string().min(1) }),
  hours: z.object({ mon: dayShape, tue: dayShape, wed: dayShape, thu: dayShape,
    fri: dayShape, sat: dayShape, sun: dayShape }),
  holidayHours: z.array(z.object({ date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
    label: z.string().max(80), closed: z.boolean(),
    open: z.string().regex(timeRe).optional(), close: z.string().regex(timeRe).optional() })).max(30),
});
export async function updateBusinessProfile(input): Promise<{ error: string } | { ok: true }> {
  // requireClientSession → getPrimaryLocation → diff old vs new (JSON compare)
  // if unchanged: return { ok: true } WITHOUT emitting (no event spam)
  // update locations row; writePortalAudit(action:'profile.update')
  // emitEvent({ clientId, type: 'profile.updated', source: 'manual',
  //             payload: { changed: ['hours','phone', …], location_id } })   // spec 02 — feeds WF-5 directory sync
  // insert activityLog { engine:'settings', action:'profile.updated' }
  // revalidatePath('/portal/settings')
}
```

`src/server/actions/team.ts` → `invitePortalUser({ email })`: `requireClientSession`; reuse the exact invite mechanics the admin `PortalSettings` flow uses (`clientUsers` insert with `inviteTokenHash`/`inviteExpiresAt` + invite email → `/portal/set-password`); cap 5 users per client (`{ error: "You've reached the 5-teammate limit — message us if you need more." }`); duplicate email → `{ error: "That email already has access." }`.

### 5.5 States / mobile / AC

- **No location row yet:** form renders empty with banner *"We're still setting up your profile from onboarding — anything you save here becomes the source of truth."* (action creates the row, `primary = true`).
- Validation errors toast the zod-mapped message: *"Check the highlighted hours — closing time must be after opening."* (client-side compare open<close before submit; server re-checks).
- Mobile: hours rows wrap (`flex flex-wrap gap-2`); time inputs `w-28`.

AC:
1. Saving a changed phone emits exactly one `profile.updated` event with `changed:["phone"]`; saving with no diff emits none.
2. Holiday rows persist and round-trip (add 2, remove 1, save, reload).
3. Brand-voice section has no inputs (read-only) and the messages link navigates.
4. Invite creates a `pending` clientUser and the invite email path is identical to the admin-initiated one (same set-password flow).
5. All writes are scoped to the session's clientId (attempting another clientId is impossible by construction — no client-supplied clientId field exists in any schema above).

---

## 6. Admin switchboard & ops (in `(app)`, Seekly staff only)

All pages here use `requireSession()` (admin iron-session) — pattern: any page under `src/app/(app)/`. **Add `"switchboard"`? No —** per-client pages live under `[clientSlug]` and need no proxy change; the new top-level `ops` route MUST be appended to `NON_CLIENT_TOP_SEGMENTS` in `src/proxy.ts` (line ~30) or the last-client cookie logic misfires.

### 6.1 Switchboard — `/{clientSlug}/switchboard`

**File:** **NEW** `src/app/(app)/[clientSlug]/switchboard/page.tsx`; add nav item to the admin sidebar config (`src/lib/nav-clusters.ts` / `src/components/shell/sidebar.tsx`, wherever standalone items live — follow how `settings` is registered).

**Data:** `workflowConfig` (spec 01) + dependency signals: `listConnections`, `getCommsProvisioning`, `getCustomerImportStats` (§1.2). **NEW** `src/server/queries/workflow-config.ts`:

```ts
export interface ModuleConfigRow { module: string; enabled: boolean;
  settings: Record<string, unknown>; updatedBy: string | null; updatedAt: Date | null; }
export async function listModuleConfigs(clientId: string): Promise<ModuleConfigRow[]>;
// LEFT JOIN a static module list so all 8 modules always appear (missing row = disabled, settings = {})
```

**Module registry** — extend `src/lib/modules.ts`:

```ts
export interface ModuleFieldDef {
  key: string; label: string; help?: string;
  type: "boolean" | "number" | "string" | "select" | "string_list";
  options?: { value: string; label: string }[];    // select only
  min?: number; max?: number;                       // number only
}
export interface ModuleDef { key: string; fields: ModuleFieldDef[]; settingsSchema: z.ZodTypeAny;
  dependencies: { label: string; check: "google_business" | "meta" | "a2p" | "customers" | "wordpress" }[]; }
export const MODULE_REGISTRY: ModuleDef[] = [ /* all 8, defaults per docs/03-workflows.md */ ];
```

Two modules get full field defs now (the rest use the raw-JSON fallback until specced):
- `review_automation`: `send_delay_hours` number 0–72 (default 2) · `daily_cap` number 1–200 (25) · `cooldown_days` number 30–365 (90) · `approval_mode` select auto / approve_all / approve_negative_only · `negative_threshold` number 1–4 (3).
- `speed_to_lead`: `persona_name` string · `followup_touches` number 0–7 (5) · `handoff_keywords` string_list · `booking_link` string.

**Component tree:**

```
SwitchboardPage (server)
├─ PageHeader title="Switchboard" description="Module toggles and settings for {client.name}. Flips are instant and logged."
└─ ModuleMatrix (client; NEW src/components/switchboard/module-matrix.tsx)
    └─ per module: rounded-xl border bg-card row (grid md:grid-cols-[1fr_auto])
        ├─ left: MODULE_LABELS name (+ font-mono module key, text-muted-foreground) · description
        │        · dependency chips: unmet → amber chip (border-amber-600/40 text-amber-600 — same
        │          tone precedent as prospecting StatusBadge) "Missing: Google connection" / "A2P not approved" /
        │          "No customer list" · updatedAt/updatedBy line (font-mono text-xs)
        ├─ right: Switch (src/components/ui/switch.tsx) — flipping ON with unmet deps opens a confirm
        │         Dialog: "Dependencies missing: {list}. The engine will gate at runtime and do nothing
        │         until they're met. Turn on anyway?"
        └─ expandable "Settings" (<details> or chevron toggle):
             SettingsSchemaForm (NEW src/components/switchboard/settings-schema-form.tsx):
               renders MODULE_REGISTRY fields → Input/Switch/Select/Textarea(one per line for string_list);
               modules without field defs render <Textarea className="font-mono"> raw JSON with parse-on-save.
             Save → updateModuleSettings action
```

**Actions** — **NEW** `src/server/actions/switchboard.ts`:

```ts
const flipSchema = z.object({ clientId: z.string().uuid(), module: z.string().min(1), enabled: z.boolean() });
export async function flipModule(input)
// requireSession(); upsert workflowConfig (pk client_id+module), updatedBy = session email, updatedAt now
// insert activityLog { engine:'switchboard', action: enabled ? 'module.enabled' : 'module.disabled',
//   detail:{ module, by: session.email } }
// revalidatePath(`/${slug}/switchboard`); no n8n call — engines read the toggle at run start (doc 03)

const settingsSchema = z.object({ clientId: z.string().uuid(), module: z.string(), settings: z.record(z.string(), z.unknown()) });
export async function updateModuleSettings(input)
// validate against MODULE_REGISTRY.settingsSchema when the module has one; merge over stored settings;
// activityLog action 'module.settings_updated' with detail.diff (changed keys only, values elided)
```

**AC:** 1. All 8 modules always render, with or without `workflowConfig` rows. 2. Flip persists in one round-trip and the `updatedBy` line refreshes. 3. Unmet-dependency chips derive live from connections/A2P/customer queries, and the confirm dialog appears only then. 4. Settings save rejects out-of-range values with the field label in the error. 5. Every flip/settings change produces an `activityLog` row (the "logged" promise of doc 05).

### 6.2 Ops queues — `/ops`

**File:** **NEW** `src/app/(app)/ops/page.tsx` (+ `"ops"` in `NON_CLIENT_TOP_SEGMENTS`, `src/proxy.ts`). Tabs via `src/components/ui/tabs.tsx`: **Dead letters · Email parses · Failed sends**. Each tab body reuses the existing expandable `DetailLog` component (`src/components/shell/detail-log.tsx` — headers/rows/gridTemplate/empty props).

**Data** — **NEW** `src/server/queries/ops.ts`:

```ts
export async function listDeadLetterEvents(limit = 100)   // events status='failed', newest first, joined client name
export async function listLowConfidenceParses(limit = 100) // events source='email-parse' AND status='pending'
                                                           // AND (payload->>'confidence')::float < 0.8
export async function listFailedSends(limit = 100)         // messageLedger status IN ('failed','blocked')
export async function getOpsCounts(): Promise<{ deadLetters: number; parses: number; failedSends: number }>
```

Rows (DetailLog): cells = client · type/kind · when · error/blocked_reason; details = full payload pretty-printed (`mono: true`). Row actions (buttons rendered in the DetailLog `note` slot or an action column appended to `cells` — extend DetailLog with an optional `actions?: React.ReactNode` per row, additive prop):
- Dead letter: **Retry** → `retryDeadLetter({ eventId })` calls spec 02 `reprocessEvent`; **Discard** → status `ignored` + confirm.
- Parse: **Open editor** → Dialog with `Textarea` of `payload` JSON; **Approve & process** validates JSON, writes corrected payload, status `pending` → `reprocessEvent`; **Discard**. (Compliance rule from doc 03: low-confidence parses never auto-send — this queue is the only path out.)
- Failed send: **Retry** → `retrySend({ messageId })` re-enqueues via the spec 02 send-pipeline (never direct Twilio); blocked sends show reason chip and have no retry (blocked is correct behavior).

Tab labels carry counts: `Dead letters (3)`. Empty states: *"Queue clear."* **AC:** 1. Counts match list lengths. 2. Retrying a dead letter flips it out of the queue on success and stays with an error toast on failure. 3. Parse editor round-trips invalid JSON with inline error *"Not valid JSON."* 4. Blocked sends are visibly non-retryable.

### 6.3 Manual provisioning — `/provision`

**File:** **NEW** `src/app/(app)/provision/page.tsx` (+ `"provision"` in `NON_CLIENT_TOP_SEGMENTS`). Form component **NEW** `src/components/switchboard/provision-form.tsx` modeled on `src/components/prospecting/new-prospect.tsx` (the existing manual-create form).

Fields: Business name · Website URL · Industry/niche (select: recreation, home_services, other) · Timezone (select of common TZs) · Tier (core/growth/market_leader) · Owner email · Owner mobile · GBP URL (optional).

**Action** — `provisionClient` in **NEW** `src/server/actions/provisioning.ts` (zod on all fields; `requireSession`): creates `clients` row (reusing existing slug generation — see `slugify` in `src/lib/formatting.ts`), `clientProfile` skeleton, all-OFF `workflowConfig` rows, `commsProvisioning` stub, `clientUsers` invite (existing invite mechanics), then `notifyN8n('provisioning/manual', { clientId, tier })` to run the WF-0 saga (spec 02; idempotent — re-running skips completed steps per doc 03). Returns `{ ok: true, slug }` → redirect to `/{slug}/switchboard`. Error copy: duplicate website → *"A client with this website already exists."*

**AC:** 1. Submit → client visible in switcher, switchboard shows 8 disabled modules. 2. Re-submitting the same business errors cleanly, no partial rows (transaction). 3. Invite email path identical to existing portal invites.

### 6.4 Prospect-mode espionage runner — extend `/prospecting`

Prospecting already exists (`src/app/(app)/prospecting/*`, actions in `src/server/actions/prospecting.ts` — `createProspectWithPrompts`, `triggerProspectRun`). Additions:

- `src/app/(app)/prospecting/[clientSlug]/page.tsx`: add an **Espionage** card — competitor list editor (name + website URL + GBP/place id rows, add/remove — stored on the prospect's `clientProfile.competitors`) + Button "Run espionage digest" → **NEW** action `triggerProspectEspionage({ clientId })` in `src/server/actions/prospecting.ts`: validates ≥1 competitor, `notifyN8n('espionage/run', { clientId, mode: 'prospect' })` (WF-6 with `client_id = prospect`, doc 03), inserts `activityLog` row, returns `{ ok: true }`; card then polls status like `getProspectRunStatus` does for SoV runs.
- Digest render: **NEW** `src/app/(app)/prospecting/[clientSlug]/espionage/page.tsx` — reads `intelDigests` (`kind='competitor'`) for the prospect and renders `findings`/`bodyHtml` with the same `LeakageReport` prose treatment (§2c); PageHeader action Button "Copy for outreach" (CopyBlock of a plaintext summary — this digest is the outbound lead magnet, doc 01).

**AC:** 1. Run button disabled with helper text until a competitor is added. 2. Digest page renders the seeded digest fixture; empty state *"No digest yet — run espionage above."* 3. Prospect stays `kind='prospect'` (never appears in the client switcher — existing invariant in `src/server/queries/clients.ts` `listClients`).

---

## 7. Tickets & build order

Build order follows doc 10's revised sequence (§"Revised near-term sequence"): schema/API first → WF-6 + prospect tooling → Integration Hub → WF-1 pilot → WF-2 + Growth/Approvals. Portal MVP cut (doc 05) = UI-1..UI-5.

| # | Ticket | Pages/sections | Depends on | Doc-10 step |
|---|---|---|---|---|
| UI-1 | Shared plumbing: tab registry + redirect, modules.ts, GrowthStatCard, StatusBadge, GrowthTabs, error.tsx, formatMoney, activity-copy.ts | §0.4 | — (pure UI) | 2 |
| UI-2 | Admin switchboard (matrix + settings forms) | §6.1 | spec 01 `workflowConfig`; UI-1 | 2 |
| UI-3 | Manual provisioning form | §6.3 | spec 01; spec 02 WF-0 hook; UI-2 | 2 |
| UI-4 | Prospect espionage runner + digest page | §6.4 | spec 01 `intelDigests`, `competitorSnapshots`; spec 02 WF-6 hook | 3 |
| UI-5 | Integrations page + banners + CSV upload + legacy redirect | §1 | spec 01 `connections`/`commsProvisioning`/`customers`; spec 04 OAuth routes; UI-1 | 4 |
| UI-6 | Overview upgrade (growth rows + opportunities + activity feed) | §4 | spec 01 `activityLog`/`metricsDaily`; UI-1 | 5 |
| UI-7 | Approvals queue + `/a/[token]` deep links + nav dot | §3 | spec 01 `approvalRequests` + reviews response columns; spec 02 notify; UI-1 | 5 (WF-1 approval path) |
| UI-8 | Growth → Leads list + detail | §2a | spec 01 `conversations`/`messageLedger`; WF-2 producing data; UI-1 | 6 |
| UI-9 | Growth → Reactivation list/detail + approve flow | §2b | spec 01 `campaigns`/`campaignMembers`; spec 02 notify; UI-7 (shared approve helper) | 6 |
| UI-10 | Settings/Profile (NAP/hours/holidays, brand voice, team) | §5 | spec 01 `locations`/`clientProfile`; spec 02 emitEvent | 6 |
| UI-11 | Ops queues page | §6.2 | spec 01 `events`/`messageLedger`; spec 02 reprocess/retry helpers | 6 |
| UI-12 | Revenue Leakage report page | §2c | spec 01 `intelDigests.kind`; intelligence job writing artifacts (doc 09, weeks 5–6) | post-30d ramp |
| UI-13 | Copy pass: audit every page against §0.5 rules; admin mirrors for Integrations/Growth where useful | all | UI-2..UI-12 | hardening |

Definition of done per ticket: acceptance criteria of its section pass; `npm run lint` and `npm test` green; new tables touched only via `src/server/queries`/`actions`; zero raw color utilities (AGENTS.md); mobile check at 375 px.

## 8. Open questions (carried to summary)

1. Exact filenames/section anchors of sibling specs 01/02/04 assumed, and two spec-01 deltas needed: `approvalRequests` table (§3.2) and `intelDigests.kind` discriminator (§2c) — confirm they're added there.
2. Growth score formula is deferred to the spec 02 rollup job; the card hides until `module='growth'` rows exist — acceptable for MVP?
3. Doc 05's admin "tenant health list" is not in this spec's page list — deferred (ops counts partially cover it); confirm.
4. `/a/[token]` treats the uuid token as sole authorization for one-tap approve (mirrors `reviewRequests.feedbackToken`); if stricter auth is wanted, fall back to session deep-link `/portal/approvals/[id]` only.
5. Legacy `clientCredentials` "Other logins" section stays on the Integrations page indefinitely, or sunset after WordPress/native adapters cover all secrets?
