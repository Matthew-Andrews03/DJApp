# Spec 04 — OAuth Integration Hub (Google, Meta, WordPress) + Token Lifecycle

**Repo:** `seekly-client-insights` (the platform app — see doc 10).
**Depends on:** Spec 01 (schema: `connections`, `events`, `workflow_config`, `activity_log` tables), Spec 06 (n8n orchestration — consumes tokens via the internal context API, never refreshes them itself).
**Source requirements:** docs/06-integrations.md (provider details), docs/02-architecture.md §Layer 2 (token vault, health monitor, auto-pause), docs/10 §5 (credentials page → Integration Hub; reuse `src/server/vault`).

## Purpose

Client-facing "Connect Google / Connect Facebook / Connect Website" flows in the portal. One
Seekly OAuth app per provider; the client authorizes through it; tokens are stored encrypted
via the existing vault module and surfaced to workflows only as short-lived access tokens.
Includes: the full token lifecycle (refresh with concurrency lock, daily health check, module
auto-pause, disconnect+revoke), the Meta leadgen webhook, the WordPress application-password
capture, and the portal Integrations page.

### Existing code you MUST reuse (do not reinvent)

| What | Where | Functions/exports to use by name |
|---|---|---|
| Secret vault (Bitwarden Secrets Manager via `bws` CLI) | `src/server/vault/bitwarden.ts` | `vaultConfigured()`, `vaultDiagnostics()`, `storeSecret()`, `getSecret()`, `deleteSecret()`, `VaultSecret`, `VaultError`, `VaultNotConfiguredError` |
| Portal session (iron-session) | `src/server/auth/session.ts` | `requireClientSession()`, `getClientSession()`, `ClientSessionData` |
| Admin session | `src/server/auth/session.ts` | `requireSession()` |
| Audit log | `src/server/audit/portal-audit.ts` | `writePortalAudit()` (same call pattern as `src/server/actions/portal-credentials.ts`) |
| DB | `src/server/db` | `db`, plus tables from `src/server/db/schema.ts` |
| pg-boss | `src/server/pipeline/queue.ts` | `getBoss()`, `QUEUES` (extend it), retry-queue pattern in `RETRY_QUEUES` |
| Env | `src/env.ts` | `env`, `appUrl` (extend the zod schema) |
| Logging | `src/lib/logger.ts` | `log()`, `logError()` |
| Twilio signature pattern (reference for HMAC verification style) | `src/app/api/reviews/sms/inbound/route.ts` | — |
| Portal nav registry | `src/lib/portal-tabs.ts` | `PORTAL_TAB_REGISTRY` (add `integrations` tab) |

### Secret-storage split (why two mechanisms)

Per doc 02, tokens must be unreadable even with DB access; per doc 10 the existing vault module
is the home for token secrets. `bws` is a CLI round-trip (~1–2 s) with create/get/delete only —
fine for durable secrets, wrong for a token consulted on every workflow run. So:

- **Durable secret** (Google refresh token; Meta long-lived user token; Meta page token;
  WordPress application password): stored in Bitwarden via `storeSecret()`; only the returned
  `itemId` lands in our DB (`connections.refreshTokenItemId`, `connections.metadata.pageTokenItemId`,
  `clientCredentials.bitwardenItemId`).
- **Short-lived access token** (Google, ≤1 h TTL): cached on the `connections` row,
  AES-256-GCM-encrypted at the application layer (new `src/server/vault/token-crypto.ts`,
  key in env `TOKEN_ENCRYPTION_KEY`, never in the DB). Losing this cache costs one refresh call.

### Interface with Spec 01 (the `connections` table)

Spec 01 owns the migration. This spec's code assumes exactly the following shape — if Spec 01
named anything differently, **Spec 01 wins**; rename identifiers below mechanically.

```ts
// src/server/db/schema.ts (owned by Spec 01 — shown for reference)
export const connectionProviderEnum = pgEnum("connection_provider", ["google", "meta", "wordpress"]);
export const connectionStatusEnum = pgEnum("connection_status", ["pending", "active", "broken", "revoked"]);

export const connections = pgTable(
  "connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull().references(() => clients.id, { onDelete: "cascade" }),
    provider: connectionProviderEnum("provider").notNull(),
    providerAccountId: text("provider_account_id"),          // Google sub / Meta user id / WP username
    accountHint: text("account_hint"),                        // NON-secret display hint (email / page name)
    scopes: text("scopes").array().notNull().default([]),
    refreshTokenItemId: text("refresh_token_item_id"),        // Bitwarden secret id (durable secret)
    accessTokenEnc: text("access_token_enc"),                 // AES-256-GCM blob (short-lived cache)
    accessTokenExpiresAt: timestamp("access_token_expires_at", { withTimezone: true }),
    status: connectionStatusEnum("status").notNull().default("pending"),
    statusReason: text("status_reason"),                      // human-readable, shown in portal on broken
    lastVerifiedAt: timestamp("last_verified_at", { withTimezone: true }),
    metadata: jsonb("metadata").notNull().default({}),        // provider-specific, see per-ticket docs
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("connections_client_provider_unique").on(t.clientId, t.provider)],
);
```

`metadata` contents by provider:

| Provider | metadata keys |
|---|---|
| google | `gbpAccountName` ("accounts/123"), `gbpLocationId` ("locations/456"), `placeId`, `locationTitle` |
| meta | `pageId`, `pageName`, `pageTokenItemId` (Bitwarden id of the page access token), `userTokenExpiresAt` (ISO) |
| wordpress | `siteUrl`, `username`, `clientCredentialId` (FK into `clientCredentials`) |

Also assumed from Spec 01: `events` (canonical events; `id uuid pk` used as idempotency key),
`workflow_config` (`clientId`, `module`, `enabled`, `settings jsonb`, `updatedBy`, `updatedAt`),
`activity_log` (`clientId`, `engine`, `action`, `status`, `entityType`, `entityId`, `detail jsonb`).

### New environment variables (add to the zod schema in `src/env.ts`)

```ts
  // --- OAuth Integration Hub (Spec 04) ---
  // Google: OAuth client from the Seekly GCP project (NOT the Gemini GOOGLE_API_KEY).
  GOOGLE_OAUTH_CLIENT_ID: z.string().optional(),
  GOOGLE_OAUTH_CLIENT_SECRET: z.string().optional(),
  // While the consent screen / GBP API application is unverified, portal shows the
  // interim-mode notice and workflows use the Places-API/manual fallbacks (doc 06).
  GOOGLE_OAUTH_MODE: z.enum(["pending_verification", "verified"]).default("pending_verification"),
  // Meta developer app.
  META_APP_ID: z.string().optional(),
  META_APP_SECRET: z.string().optional(),
  META_WEBHOOK_VERIFY_TOKEN: z.string().optional(),   // any random string; also entered in Meta dashboard
  META_GRAPH_VERSION: z.string().default("v21.0"),
  // HMAC key for the OAuth state JWT (generate: openssl rand -base64 32).
  OAUTH_STATE_SECRET: z.string().min(32).optional(),
  // 32-byte base64 key for AES-256-GCM access-token cache (generate: openssl rand -base64 32).
  TOKEN_ENCRYPTION_KEY: z.string().min(43).optional(),
```

All optional so dev/test boots without them; every flow checks its own vars and fails with a
clear portal message (same philosophy as `vaultConfigured()`).

---

## Ticket table

| Ticket | Title | Depends on |
|---|---|---|
| OA-1 | Shared plumbing: state JWT, token crypto, connections store | Spec 01 migration |
| OA-2 | Google OAuth: `/api/oauth/google/start` + `/api/oauth/google/callback` | OA-1 |
| OA-3 | GBP account/location listing + portal location picker | OA-2 |
| OA-4 | GCP console runbook + verification-pending interim mode | — (ops task) |
| OA-5 | Meta OAuth: start/callback + page picker + leadgen subscription | OA-1 |
| OA-6 | Meta leadgen webhook (verify + signature + lead fetch job) | OA-5 |
| OA-7 | Meta dev-mode tester runbook + App Review checklist | OA-5 (ops task) |
| OA-8 | `getFreshAccessToken(connectionId)` with anti-concurrency lock | OA-1, OA-2, OA-5 |
| OA-9 | Health-check cron + module auto-pause + portal banner + disconnect/revoke | OA-8 |
| OA-10 | WordPress connect (application password → `clientCredentials` + test call) | OA-1 |
| OA-11 | Portal Integrations page (all providers, all states, exact copy) | OA-2..OA-10 |

---

## OA-1 — Shared plumbing

### 1a. OAuth state JWT — `src/server/oauth/state.ts` (new file)

No `jose`/`jsonwebtoken` dependency exists; implement HS256 with `node:crypto` (25 lines, no
new package).

```ts
import { createHmac, randomBytes, timingSafeEqual } from "node:crypto";
import { env } from "@/env";

export interface OAuthStatePayload {
  clientId: string;
  userId: string;   // clientUsers.id that initiated the connect
  nonce: string;    // random, echoed in a cookie; single-use CSRF binding
  provider: "google" | "meta";
  iat: number;      // seconds
  exp: number;      // seconds; start + 10 minutes
}

const b64url = (b: Buffer | string) => Buffer.from(b).toString("base64url");

function secret(): string {
  if (!env.OAUTH_STATE_SECRET) throw new Error("OAUTH_STATE_SECRET is not set");
  return env.OAUTH_STATE_SECRET;
}

export function newNonce(): string {
  return randomBytes(16).toString("base64url");
}

/** Signed JWT (HS256): header.payload.signature, 10-minute expiry. */
export function signOAuthState(p: Omit<OAuthStatePayload, "iat" | "exp">): string {
  const now = Math.floor(Date.now() / 1000);
  const payload: OAuthStatePayload = { ...p, iat: now, exp: now + 600 };
  const head = b64url(JSON.stringify({ alg: "HS256", typ: "JWT" }));
  const body = b64url(JSON.stringify(payload));
  const sig = createHmac("sha256", secret()).update(`${head}.${body}`).digest("base64url");
  return `${head}.${body}.${sig}`;
}

/** Returns the payload, or null on ANY failure (bad sig, expired, wrong provider). */
export function verifyOAuthState(token: string, expectedProvider: "google" | "meta"): OAuthStatePayload | null {
  const parts = token.split(".");
  if (parts.length !== 3) return null;
  const [head, body, sig] = parts;
  const expected = createHmac("sha256", secret()).update(`${head}.${body}`).digest("base64url");
  const a = Buffer.from(sig);
  const b = Buffer.from(expected);
  if (a.length !== b.length || !timingSafeEqual(a, b)) return null;
  let payload: OAuthStatePayload;
  try {
    payload = JSON.parse(Buffer.from(body, "base64url").toString("utf8"));
  } catch {
    return null;
  }
  if (payload.provider !== expectedProvider) return null;
  if (typeof payload.exp !== "number" || payload.exp * 1000 < Date.now()) return null;
  if (!payload.clientId || !payload.userId || !payload.nonce) return null;
  return payload;
}
```

### 1b. Access-token cache crypto — `src/server/vault/token-crypto.ts` (new file)

```ts
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";
import { env } from "@/env";

export class TokenCryptoNotConfiguredError extends Error {}

function key(): Buffer {
  if (!env.TOKEN_ENCRYPTION_KEY) {
    throw new TokenCryptoNotConfiguredError("TOKEN_ENCRYPTION_KEY is not set.");
  }
  const k = Buffer.from(env.TOKEN_ENCRYPTION_KEY, "base64");
  if (k.length !== 32) {
    throw new TokenCryptoNotConfiguredError("TOKEN_ENCRYPTION_KEY must be 32 bytes base64.");
  }
  return k;
}

export function tokenCryptoConfigured(): boolean {
  try {
    key();
    return true;
  } catch {
    return false;
  }
}

/** AES-256-GCM. Output format: "v1:<iv b64>:<ciphertext b64>:<authTag b64>". */
export function encryptToken(plaintext: string): string {
  const iv = randomBytes(12);
  const cipher = createCipheriv("aes-256-gcm", key(), iv);
  const ct = Buffer.concat([cipher.update(plaintext, "utf8"), cipher.final()]);
  return `v1:${iv.toString("base64")}:${ct.toString("base64")}:${cipher.getAuthTag().toString("base64")}`;
}

export function decryptToken(blob: string): string {
  const [v, ivB64, ctB64, tagB64] = blob.split(":");
  if (v !== "v1" || !ivB64 || !ctB64 || !tagB64) throw new Error("Malformed token blob.");
  const decipher = createDecipheriv("aes-256-gcm", key(), Buffer.from(ivB64, "base64"));
  decipher.setAuthTag(Buffer.from(tagB64, "base64"));
  return Buffer.concat([decipher.update(Buffer.from(ctB64, "base64")), decipher.final()]).toString("utf8");
}
```

### 1c. Connections store — `src/server/connections/store.ts` (new file)

```ts
import { and, eq } from "drizzle-orm";
import { db } from "@/server/db";
import { connections } from "@/server/db/schema";
import { deleteSecret, storeSecret } from "@/server/vault/bitwarden";
import { encryptToken } from "@/server/vault/token-crypto";

export type Provider = "google" | "meta" | "wordpress";
export type Connection = typeof connections.$inferSelect;

export async function getConnectionById(id: string): Promise<Connection | null> {
  const [row] = await db.select().from(connections).where(eq(connections.id, id)).limit(1);
  return row ?? null;
}

export async function getConnection(clientId: string, provider: Provider): Promise<Connection | null> {
  const [row] = await db
    .select()
    .from(connections)
    .where(and(eq(connections.clientId, clientId), eq(connections.provider, provider)))
    .limit(1);
  return row ?? null;
}

/**
 * Create-or-replace the client's connection for a provider after an OAuth
 * callback. Stores the durable secret in Bitwarden (storeSecret) and caches
 * the short-lived access token AES-encrypted on the row. If a previous
 * connection existed, its old Bitwarden item is deleted (deleteSecret) so
 * secrets never orphan.
 */
export async function saveOAuthConnection(input: {
  clientId: string;
  provider: "google" | "meta";
  providerAccountId: string;
  accountHint: string | null;
  scopes: string[];
  durableSecret: string;            // Google refresh token / Meta long-lived user token
  accessToken: string | null;
  accessTokenExpiresAt: Date | null;
  status: "pending" | "active";     // google: "pending" until location picked
  metadata?: Record<string, unknown>;
}): Promise<Connection> {
  const existing = await getConnection(input.clientId, input.provider);

  const { itemId } = await storeSecret({
    clientId: input.clientId,
    label: `oauth-${input.provider}-durable-token`,
    secret: { username: input.providerAccountId, password: input.durableSecret },
  });

  const values = {
    clientId: input.clientId,
    provider: input.provider,
    providerAccountId: input.providerAccountId,
    accountHint: input.accountHint,
    scopes: input.scopes,
    refreshTokenItemId: itemId,
    accessTokenEnc: input.accessToken ? encryptToken(input.accessToken) : null,
    accessTokenExpiresAt: input.accessTokenExpiresAt,
    status: input.status,
    statusReason: null,
    lastVerifiedAt: new Date(),
    metadata: { ...(existing?.metadata as object | undefined), ...input.metadata },
    updatedAt: new Date(),
  } as const;

  let row: Connection;
  if (existing) {
    [row] = await db.update(connections).set(values).where(eq(connections.id, existing.id)).returning();
    if (existing.refreshTokenItemId && existing.refreshTokenItemId !== itemId) {
      await deleteSecret(existing.refreshTokenItemId).catch(() => {});
    }
  } else {
    [row] = await db.insert(connections).values(values).returning();
  }
  return row;
}

export async function markConnectionBroken(id: string, reason: string): Promise<void> {
  await db
    .update(connections)
    .set({ status: "broken", statusReason: reason, updatedAt: new Date() })
    .where(eq(connections.id, id));
}
```

### 1d. Deterministic event id helper — `src/server/events/deterministic-id.ts` (new file)

`events.id` is a uuid used as the idempotency key (Spec 01 / doc 04). Webhooks need a
*deterministic* uuid from a provider id so redelivery is a no-op:

```ts
import { createHash } from "node:crypto";

/** UUIDv5-style deterministic uuid from a namespace + name (SHA-1, RFC 4122 layout). */
export function deterministicUuid(namespace: string, name: string): string {
  const h = createHash("sha1").update(`${namespace}:${name}`).digest();
  h[6] = (h[6] & 0x0f) | 0x50; // version 5
  h[8] = (h[8] & 0x3f) | 0x80; // variant
  const hex = h.subarray(0, 16).toString("hex");
  return `${hex.slice(0, 8)}-${hex.slice(8, 12)}-${hex.slice(12, 16)}-${hex.slice(16, 20)}-${hex.slice(20, 32)}`;
}
```

### OA-1 acceptance criteria

1. `signOAuthState` → `verifyOAuthState` round-trips; tampering with any of the three JWT
   segments returns `null`; a state older than 10 minutes returns `null`; a `google` state
   fails verification when `expectedProvider === "meta"`. (Vitest unit tests in
   `src/server/oauth/state.test.ts`.)
2. `encryptToken` → `decryptToken` round-trips; flipping one ciphertext byte throws (GCM auth);
   missing/short `TOKEN_ENCRYPTION_KEY` throws `TokenCryptoNotConfiguredError`, not a generic error.
3. `saveOAuthConnection` on a client with an existing connection replaces the row (unique
   `(clientId, provider)` never violated) and calls `deleteSecret` on the superseded Bitwarden item.
4. No secret value (refresh token, access token, app password) is ever written to any DB column
   other than `accessTokenEnc` (encrypted) — verified by grepping the diff for direct assignments.
5. `deterministicUuid("meta-leadgen", "123")` is stable across calls and formats as a valid uuid.

---

## OA-2 — Google OAuth start + callback

One Seekly GCP OAuth app (OA-4 creates it). Scopes requested (single consent, doc 06):

```
openid email
https://www.googleapis.com/auth/business.manage
https://www.googleapis.com/auth/webmasters.readonly
https://www.googleapis.com/auth/analytics.readonly
```

`openid email` is added so the callback gets an `id_token` → `sub` (stable
`providerAccountId`) and `email` (the non-secret `accountHint`).

### `src/app/api/oauth/google/start/route.ts` (new file)

```ts
import { cookies } from "next/headers";
import { NextResponse } from "next/server";
import { appUrl, env } from "@/env";
import { getClientSession } from "@/server/auth/session";
import { newNonce, signOAuthState } from "@/server/oauth/state";

export const GOOGLE_SCOPES = [
  "openid",
  "email",
  "https://www.googleapis.com/auth/business.manage",
  "https://www.googleapis.com/auth/webmasters.readonly",
  "https://www.googleapis.com/auth/analytics.readonly",
];

export async function GET() {
  const session = await getClientSession();
  if (!session.loggedIn || !session.clientId || !session.clientUserId) {
    return NextResponse.redirect(new URL("/portal/login", appUrl));
  }
  if (!env.GOOGLE_OAUTH_CLIENT_ID || !env.GOOGLE_OAUTH_CLIENT_SECRET) {
    return NextResponse.redirect(new URL("/portal/integrations?error=google_not_configured", appUrl));
  }

  const nonce = newNonce();
  const state = signOAuthState({
    clientId: session.clientId,
    userId: session.clientUserId,
    nonce,
    provider: "google",
  });

  const params = new URLSearchParams({
    client_id: env.GOOGLE_OAUTH_CLIENT_ID,
    redirect_uri: `${appUrl}/api/oauth/google/callback`,
    response_type: "code",
    scope: GOOGLE_SCOPES.join(" "),
    access_type: "offline",          // → refresh token
    prompt: "consent",               // → refresh token EVERY time, even on re-connect
    include_granted_scopes: "true",
    state,
  });

  const res = NextResponse.redirect(`https://accounts.google.com/o/oauth2/v2/auth?${params}`);
  // Nonce cookie binds the callback to THIS browser (CSRF, on top of the signed state).
  (await cookies()).set("seekly_oauth_nonce", nonce, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 600,
    path: "/api/oauth",
  });
  return res;
}
```

### `src/app/api/oauth/google/callback/route.ts` (new file)

```ts
import { cookies } from "next/headers";
import { NextResponse } from "next/server";
import { appUrl, env } from "@/env";
import { getClientSession } from "@/server/auth/session";
import { verifyOAuthState } from "@/server/oauth/state";
import { saveOAuthConnection } from "@/server/connections/store";
import { logError } from "@/lib/logger";

interface GoogleTokenResponse {
  access_token: string;
  expires_in: number;      // seconds
  refresh_token?: string;  // present because prompt=consent + access_type=offline
  scope: string;           // space-separated GRANTED scopes (user can untick!)
  id_token?: string;
}

function fail(reason: string): NextResponse {
  return NextResponse.redirect(new URL(`/portal/integrations?error=${reason}`, appUrl));
}

export async function GET(request: Request) {
  const url = new URL(request.url);
  if (url.searchParams.get("error")) return fail("google_denied"); // user clicked Cancel

  const code = url.searchParams.get("code");
  const stateToken = url.searchParams.get("state");
  if (!code || !stateToken) return fail("google_invalid");

  const state = verifyOAuthState(stateToken, "google");
  if (!state) return fail("google_state");

  const cookieStore = await cookies();
  const nonceCookie = cookieStore.get("seekly_oauth_nonce")?.value;
  cookieStore.delete("seekly_oauth_nonce"); // single-use
  if (!nonceCookie || nonceCookie !== state.nonce) return fail("google_state");

  // The signed-in portal user must be the one who started the flow.
  const session = await getClientSession();
  if (!session.loggedIn || session.clientId !== state.clientId || session.clientUserId !== state.userId) {
    return fail("google_session");
  }

  // --- Code exchange ---
  const tokenRes = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "content-type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      code,
      client_id: env.GOOGLE_OAUTH_CLIENT_ID!,
      client_secret: env.GOOGLE_OAUTH_CLIENT_SECRET!,
      redirect_uri: `${appUrl}/api/oauth/google/callback`,
      grant_type: "authorization_code",
    }),
  });
  if (!tokenRes.ok) {
    logError("[oauth-google] token exchange failed", tokenRes.status, await tokenRes.text());
    return fail("google_exchange");
  }
  const tokens = (await tokenRes.json()) as GoogleTokenResponse;

  if (!tokens.refresh_token) return fail("google_no_refresh"); // shouldn't happen with prompt=consent
  const granted = tokens.scope.split(" ");
  if (!granted.includes("https://www.googleapis.com/auth/business.manage")) {
    // User unticked the GBP checkbox on the consent screen — connection is useless.
    return fail("google_scope_missing");
  }

  // id_token came straight from Google's token endpoint over TLS in this same
  // exchange — decoding without signature verification is safe here.
  let providerAccountId = "unknown";
  let accountHint: string | null = null;
  if (tokens.id_token) {
    try {
      const claims = JSON.parse(Buffer.from(tokens.id_token.split(".")[1], "base64url").toString("utf8"));
      providerAccountId = String(claims.sub ?? "unknown");
      accountHint = typeof claims.email === "string" ? claims.email : null;
    } catch (err) {
      logError("[oauth-google] id_token decode failed", err);
    }
  }

  try {
    await saveOAuthConnection({
      clientId: state.clientId,
      provider: "google",
      providerAccountId,
      accountHint,
      scopes: granted,
      durableSecret: tokens.refresh_token,          // → Bitwarden via storeSecret()
      accessToken: tokens.access_token,             // → AES cache via encryptToken()
      accessTokenExpiresAt: new Date(Date.now() + tokens.expires_in * 1000),
      status: "pending",                            // active only after location picked (OA-3)
    });
  } catch (err) {
    logError("[oauth-google] saveOAuthConnection failed", err);
    return fail("google_vault");
  }

  return NextResponse.redirect(new URL("/portal/integrations/google/select-location", appUrl));
}
```

### OA-2 acceptance criteria

1. Hitting `/api/oauth/google/start` while logged out redirects to `/portal/login`; logged in,
   it 302s to `accounts.google.com` with exactly the 5 scopes above, `access_type=offline`,
   `prompt=consent`, and a 3-segment `state` JWT, and sets the `seekly_oauth_nonce` cookie.
2. Callback with a forged/expired `state`, a missing/mismatched nonce cookie, or a session whose
   `clientId` differs from the state → redirect to `/portal/integrations?error=google_state|google_session`;
   **no** token exchange request is made in these cases.
3. Callback with `?error=access_denied` → `?error=google_denied`, no exchange.
4. Successful callback: `connections` row exists with `provider='google'`, `status='pending'`,
   `refreshTokenItemId` set, `accessTokenEnc` decryptable to the access token, `scopes`
   containing `business.manage`, and browser lands on the location picker.
5. If the user unticks the GBP scope on Google's consent screen, no connection row is created
   and the redirect carries `error=google_scope_missing`.
6. Reconnecting (second full flow) leaves exactly one `google` row per client and deletes the
   previous Bitwarden item.

---

## OA-3 — GBP account/location listing + portal location picker

After the callback the connection is `pending`. The client must pick which GBP location Seekly
manages; saving it flips the connection `active` and records `gbp_location_id`.

### `src/server/connections/google-gbp.ts` (new file)

```ts
import { getFreshAccessToken } from "@/server/connections/tokens";

const ACCT_API = "https://mybusinessaccountmanagement.googleapis.com/v1";
const INFO_API = "https://mybusinessbusinessinformation.googleapis.com/v1";

export interface GbpAccount { name: string; accountName: string; type: string }
export interface GbpLocation {
  name: string;              // "locations/123456"
  title: string;
  placeId: string | null;
  address: string | null;
}

async function gbpFetch<T>(accessToken: string, url: string): Promise<T> {
  const res = await fetch(url, { headers: { authorization: `Bearer ${accessToken}` } });
  if (!res.ok) {
    throw new Error(`GBP API ${res.status}: ${(await res.text()).slice(0, 300)}`);
  }
  return (await res.json()) as T;
}

export async function listGbpAccounts(connectionId: string): Promise<GbpAccount[]> {
  const token = await getFreshAccessToken(connectionId);
  const data = await gbpFetch<{ accounts?: GbpAccount[] }>(token, `${ACCT_API}/accounts`);
  return data.accounts ?? [];
}

export async function listGbpLocations(connectionId: string, accountName: string): Promise<GbpLocation[]> {
  const token = await getFreshAccessToken(connectionId);
  const readMask = "name,title,storefrontAddress,metadata";
  const out: GbpLocation[] = [];
  let pageToken: string | undefined;
  do {
    const qs = new URLSearchParams({ readMask, pageSize: "100" });
    if (pageToken) qs.set("pageToken", pageToken);
    const data = await gbpFetch<{
      locations?: Array<{
        name: string;
        title: string;
        metadata?: { placeId?: string };
        storefrontAddress?: { addressLines?: string[]; locality?: string; administrativeArea?: string };
      }>;
      nextPageToken?: string;
    }>(token, `${INFO_API}/${accountName}/locations?${qs}`);
    for (const l of data.locations ?? []) {
      const a = l.storefrontAddress;
      out.push({
        name: l.name,
        title: l.title,
        placeId: l.metadata?.placeId ?? null,
        address: a ? [a.addressLines?.join(" "), a.locality, a.administrativeArea].filter(Boolean).join(", ") : null,
      });
    }
    pageToken = data.nextPageToken;
  } while (pageToken);
  return out;
}
```

> Note: until the GBP API application (OA-4) is approved, these endpoints return 403/429
> (quota 0). The picker page catches errors and shows the interim-mode notice (OA-11 copy),
> leaving the connection `pending`. Nothing else breaks.

### Server action — add to new `src/server/actions/integrations.ts`

```ts
"use server";

import { eq } from "drizzle-orm";
import { z } from "zod";
import { db } from "@/server/db";
import { connections } from "@/server/db/schema";
import { requireClientSession } from "@/server/auth/session";
import { getConnection } from "@/server/connections/store";
import { writePortalAudit } from "@/server/audit/portal-audit";

const pickSchema = z.object({
  gbpAccountName: z.string().regex(/^accounts\/\d+$/),
  gbpLocationId: z.string().regex(/^locations\/\d+$/),
  placeId: z.string().optional(),
  locationTitle: z.string().min(1),
});

export async function saveGbpLocation(
  input: z.infer<typeof pickSchema>,
): Promise<{ error: string } | { ok: true }> {
  const s = await requireClientSession();
  const parsed = pickSchema.safeParse(input);
  if (!parsed.success) return { error: "Pick a location to continue." };

  const conn = await getConnection(s.clientId!, "google");
  if (!conn || conn.status === "revoked") return { error: "Connect Google first." };

  await db
    .update(connections)
    .set({
      status: "active",
      statusReason: null,
      metadata: { ...(conn.metadata as object), ...parsed.data },
      updatedAt: new Date(),
    })
    .where(eq(connections.id, conn.id));

  await writePortalAudit({
    clientId: s.clientId!,
    actorKind: "client",
    actorId: s.clientUserId!,
    action: "integration.google.location_selected",
    detail: parsed.data.gbpLocationId,
  });
  return { ok: true };
}
```

(If `writePortalAudit`'s real signature differs, match the call pattern used in
`src/server/actions/portal-credentials.ts`.)

### Picker page — `src/app/(portal)/portal/(authed)/integrations/google/select-location/page.tsx` (new file)

Server component: `requireClientSession()` → `getConnection(clientId, "google")`; if no
connection redirect to `/portal/integrations`. Try `listGbpAccounts` → for the first account
(loop all accounts, flatten) `listGbpLocations`; render one radio card per location
(`title`, `address`) + a "Use this location" submit button that calls `saveGbpLocation` and
`redirect("/portal/integrations?connected=google")`. On API error (quota 0 pre-approval)
render the OA-11 `pending` copy card instead of a crash. Use existing `Card`/`Button`
components (see `credentials/page.tsx` for the established page pattern).

### OA-3 acceptance criteria

1. Completing OA-2 then choosing a location sets `connections.status='active'` and
   `metadata.gbpLocationId`/`placeId`/`gbpAccountName`/`locationTitle`.
2. A client with >1 GBP location sees all of them (pagination handled — seed test with a mocked
   fetch returning `nextPageToken`).
3. GBP API 403 (pre-approval) renders the "Waiting on Google approval" card, connection stays
   `pending`, no unhandled error in logs beyond one `logError`.
4. `saveGbpLocation` rejects malformed ids (`accounts/x`, arbitrary strings) with a form error.
5. A `portalAuditLog` row `integration.google.location_selected` exists after saving.

---

## OA-4 — GCP console runbook + verification-pending interim mode

Ops/founder task. Do these in order; items 6–7 are **day-1 critical path** (README).

1. **Create project:** console.cloud.google.com → New Project → name `seekly-platform`
   (prod; make a second `seekly-platform-dev` later for localhost testing).
2. **Enable APIs** (APIs & Services → Library), all in this project:
   - My Business Account Management API
   - My Business Business Information API
   - Google My Business API (the legacy v4 surface — still required for review list/reply)
   - Google Search Console API
   - Google Analytics Data API
   - Places API (New) — interim review reads while GBP access pends (doc 06)
3. **OAuth consent screen** (APIs & Services → OAuth consent screen):
   - User type: **External**. App name: `Seekly`. Support email: matthewa0903@gmail.com (until
     a branded mailbox exists). Developer contact: same.
   - Authorized domains: the apex of `APP_URL` (e.g. `seekly.ca`). Add links to the marketing
     site's privacy policy + terms pages (required for verification).
   - Scopes: add the three sensitive/restricted scopes from OA-2 plus `openid`, `email`.
   - **Scope justification text** (paste, adjust brand):
     > Seekly is a managed local-marketing platform. Business owners connect their Google
     > account so Seekly can (1) manage their Business Profile — reply to reviews, publish
     > posts, and keep hours accurate (`business.manage`); (2) read their Search Console
     > performance to report content results (`webmasters.readonly`); (3) read their GA4
     > traffic to attribute leads (`analytics.readonly`). Data is used only to operate the
     > customer's own account and is never sold or shared.
   - Publishing status: keep **Testing** until verification passes; add every pilot's Google
     account email under **Test users** (max 100 — plenty).
4. **Create credentials:** APIs & Services → Credentials → Create credentials → OAuth client ID
   → Web application → name `seekly-portal`. Authorized redirect URIs:
   - `https://<prod APP_URL host>/api/oauth/google/callback`
   - `http://localhost:3000/api/oauth/google/callback` (dev project only)
   Copy client id/secret → env `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`.
5. **App verification:** submit the consent screen for verification (sensitive + restricted
   scopes require it: demo video of the connect flow + in-product use of each scope, privacy
   policy URL, domain verification via Search Console). Expect days–weeks.
6. **GBP API access application** — separate from OAuth verification and ALSO gating:
   quota for all My Business APIs is **0 QPM until Google approves the project**.
   - Form: **https://support.google.com/business/contact/api_default** ("Request access to
     Business Profile APIs"). Prereqs doc: https://developers.google.com/my-business/content/prereqs
   - Steps: (a) sign in with the Google account that owns the GCP project; (b) fill the form
     with: project number (Console → Dashboard), project id, business name `Seekly`, website,
     use case — paste:
     > Seekly is a software platform for local businesses. On behalf of our customers (who
     > authorize via OAuth), we reply to their Google reviews, publish Business Profile posts,
     > and update their business hours. We are the developer and data processor; each
     > end customer connects their own Business Profile.
     (c) submit; (d) after approval email arrives, confirm quotas (IAM → Quotas → "My Business")
     rose above 0.
7. **Track it:** create ops checklist items "GBP API approved" and "OAuth verification passed";
   both must be checked before setting `GOOGLE_OAUTH_MODE=verified`.

### Interim mode flag

`GOOGLE_OAUTH_MODE` (env, OA-1) drives behavior while either approval pends:

| Area | `pending_verification` | `verified` |
|---|---|---|
| Portal Integrations card | Amber notice (copy in OA-11): connect works only for whitelisted test users; Google shows an "unverified app" interstitial | Normal |
| Location picker | On GBP 403/429 → "Waiting on Google approval" card, connection stays `pending` | Normal |
| Workflows (Spec 06) | Review reads via Places API; replies/posts via ops manual queue (doc 06 fallback) | GBP API |

Implementation: export `export const googleInterimMode = env.GOOGLE_OAUTH_MODE === "pending_verification";`
from `src/server/connections/google-gbp.ts` and branch UI copy on it.

### OA-4 acceptance criteria

1. `GOOGLE_OAUTH_CLIENT_ID`/`SECRET` set in prod env; `/api/oauth/google/start` reaches a real
   Google consent screen listing all three product scopes.
2. GBP access form submitted; submission confirmation saved to the ops checklist with the date.
3. Consent screen in Testing mode lists every pilot user under Test users; a pilot completes
   the full connect flow end-to-end despite the unverified-app interstitial.
4. With `GOOGLE_OAUTH_MODE=pending_verification`, the Integrations card shows the interim
   notice; flipping the env to `verified` (restart) removes it with no code change.

---

## OA-5 — Meta OAuth: start/callback + page picker + leadgen subscription

Scopes (doc 06): `leads_retrieval`, `pages_manage_posts`, `pages_show_list`,
`pages_read_engagement`. Same start/callback pattern as Google. Differences:

- No refresh token. Exchange the callback's short-lived user token for a **long-lived user
  token** (~60 days) — that is the durable secret in Bitwarden.
- From the long-lived user token, fetch `/me/accounts` → the client picks a Page → store the
  **page access token** (does not expire while the user token is valid) as a second Bitwarden
  item (`metadata.pageTokenItemId`), then subscribe the page to the app's `leadgen` webhook.

### `src/app/api/oauth/meta/start/route.ts` (new file)

```ts
import { cookies } from "next/headers";
import { NextResponse } from "next/server";
import { appUrl, env } from "@/env";
import { getClientSession } from "@/server/auth/session";
import { newNonce, signOAuthState } from "@/server/oauth/state";

export const META_SCOPES = [
  "leads_retrieval",
  "pages_manage_posts",
  "pages_show_list",
  "pages_read_engagement",
];

export async function GET() {
  const session = await getClientSession();
  if (!session.loggedIn || !session.clientId || !session.clientUserId) {
    return NextResponse.redirect(new URL("/portal/login", appUrl));
  }
  if (!env.META_APP_ID || !env.META_APP_SECRET) {
    return NextResponse.redirect(new URL("/portal/integrations?error=meta_not_configured", appUrl));
  }
  const nonce = newNonce();
  const state = signOAuthState({
    clientId: session.clientId,
    userId: session.clientUserId,
    nonce,
    provider: "meta",
  });
  const params = new URLSearchParams({
    client_id: env.META_APP_ID,
    redirect_uri: `${appUrl}/api/oauth/meta/callback`,
    response_type: "code",
    scope: META_SCOPES.join(","),
    state,
  });
  const res = NextResponse.redirect(
    `https://www.facebook.com/${env.META_GRAPH_VERSION}/dialog/oauth?${params}`,
  );
  (await cookies()).set("seekly_oauth_nonce", nonce, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 600,
    path: "/api/oauth",
  });
  return res;
}
```

### `src/app/api/oauth/meta/callback/route.ts` (new file)

```ts
import { cookies } from "next/headers";
import { NextResponse } from "next/server";
import { appUrl, env } from "@/env";
import { getClientSession } from "@/server/auth/session";
import { verifyOAuthState } from "@/server/oauth/state";
import { saveOAuthConnection } from "@/server/connections/store";
import { logError } from "@/lib/logger";

const GRAPH = () => `https://graph.facebook.com/${env.META_GRAPH_VERSION}`;

function fail(reason: string): NextResponse {
  return NextResponse.redirect(new URL(`/portal/integrations?error=${reason}`, appUrl));
}

export async function GET(request: Request) {
  const url = new URL(request.url);
  if (url.searchParams.get("error")) return fail("meta_denied");
  const code = url.searchParams.get("code");
  const stateToken = url.searchParams.get("state");
  if (!code || !stateToken) return fail("meta_invalid");

  const state = verifyOAuthState(stateToken, "meta");
  if (!state) return fail("meta_state");
  const cookieStore = await cookies();
  const nonceCookie = cookieStore.get("seekly_oauth_nonce")?.value;
  cookieStore.delete("seekly_oauth_nonce");
  if (!nonceCookie || nonceCookie !== state.nonce) return fail("meta_state");

  const session = await getClientSession();
  if (!session.loggedIn || session.clientId !== state.clientId || session.clientUserId !== state.userId) {
    return fail("meta_session");
  }

  // 1. code → short-lived user token
  const qs1 = new URLSearchParams({
    client_id: env.META_APP_ID!,
    client_secret: env.META_APP_SECRET!,
    redirect_uri: `${appUrl}/api/oauth/meta/callback`,
    code,
  });
  const shortRes = await fetch(`${GRAPH()}/oauth/access_token?${qs1}`);
  if (!shortRes.ok) {
    logError("[oauth-meta] code exchange failed", shortRes.status, await shortRes.text());
    return fail("meta_exchange");
  }
  const shortTok = (await shortRes.json()) as { access_token: string };

  // 2. short-lived → long-lived user token (~60 days) — the durable secret
  const qs2 = new URLSearchParams({
    grant_type: "fb_exchange_token",
    client_id: env.META_APP_ID!,
    client_secret: env.META_APP_SECRET!,
    fb_exchange_token: shortTok.access_token,
  });
  const longRes = await fetch(`${GRAPH()}/oauth/access_token?${qs2}`);
  if (!longRes.ok) {
    logError("[oauth-meta] long-lived exchange failed", longRes.status, await longRes.text());
    return fail("meta_exchange");
  }
  const longTok = (await longRes.json()) as { access_token: string; expires_in?: number };
  const userTokenExpiresAt = new Date(Date.now() + (longTok.expires_in ?? 60 * 86400) * 1000);

  // 3. identify the user (providerAccountId + hint)
  const meRes = await fetch(`${GRAPH()}/me?fields=id,name&access_token=${encodeURIComponent(longTok.access_token)}`);
  if (!meRes.ok) return fail("meta_exchange");
  const me = (await meRes.json()) as { id: string; name: string };

  try {
    await saveOAuthConnection({
      clientId: state.clientId,
      provider: "meta",
      providerAccountId: me.id,
      accountHint: me.name,
      scopes: ["leads_retrieval", "pages_manage_posts", "pages_show_list", "pages_read_engagement"],
      durableSecret: longTok.access_token,      // long-lived USER token → Bitwarden
      accessToken: null,                        // meta has no separate short-lived cache
      accessTokenExpiresAt: null,
      status: "pending",                        // active after page picked
      metadata: { userTokenExpiresAt: userTokenExpiresAt.toISOString() },
    });
  } catch (err) {
    logError("[oauth-meta] saveOAuthConnection failed", err);
    return fail("meta_vault");
  }
  return NextResponse.redirect(new URL("/portal/integrations/meta/select-page", appUrl));
}
```

### Page picker + leadgen subscription

`src/server/connections/meta.ts` (new file):

```ts
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { connections } from "@/server/db/schema";
import { env } from "@/env";
import { getConnection, type Connection } from "@/server/connections/store";
import { getSecret, storeSecret, deleteSecret } from "@/server/vault/bitwarden";

const GRAPH = () => `https://graph.facebook.com/${env.META_GRAPH_VERSION}`;

export interface MetaPage { id: string; name: string; access_token: string }

export async function listPages(clientId: string): Promise<MetaPage[]> {
  const conn = await getConnection(clientId, "meta");
  if (!conn?.refreshTokenItemId) throw new Error("Meta not connected.");
  const userToken = (await getSecret(conn.refreshTokenItemId)).password;
  const res = await fetch(
    `${GRAPH()}/me/accounts?fields=id,name,access_token&limit=100&access_token=${encodeURIComponent(userToken)}`,
  );
  if (!res.ok) throw new Error(`Meta /me/accounts ${res.status}: ${(await res.text()).slice(0, 300)}`);
  return ((await res.json()) as { data?: MetaPage[] }).data ?? [];
}

/** Save the chosen page: page token → Bitwarden, subscribe page to leadgen, activate. */
export async function selectPage(clientId: string, pageId: string): Promise<void> {
  const conn = await getConnection(clientId, "meta");
  if (!conn) throw new Error("Meta not connected.");
  const pages = await listPages(clientId);
  const page = pages.find((p) => p.id === pageId);
  if (!page) throw new Error("That page is not on your Facebook account.");

  // Replace any previous page-token secret.
  const meta = conn.metadata as Record<string, unknown>;
  if (typeof meta.pageTokenItemId === "string") await deleteSecret(meta.pageTokenItemId).catch(() => {});
  const { itemId } = await storeSecret({
    clientId,
    label: `oauth-meta-page-token-${page.id}`,
    secret: { username: page.id, password: page.access_token },
  });

  // Subscribe the page to the app's leadgen webhook (Spec: OA-6 receives the events).
  const sub = await fetch(
    `${GRAPH()}/${page.id}/subscribed_apps?subscribed_fields=leadgen&access_token=${encodeURIComponent(page.access_token)}`,
    { method: "POST" },
  );
  if (!sub.ok) throw new Error(`leadgen subscribe failed ${sub.status}: ${(await sub.text()).slice(0, 300)}`);

  await db
    .update(connections)
    .set({
      status: "active",
      statusReason: null,
      metadata: { ...meta, pageId: page.id, pageName: page.name, pageTokenItemId: itemId },
      updatedAt: new Date(),
    })
    .where(eq(connections.id, conn.id));
}
```

Server action `saveMetaPage(pageId)` in `src/server/actions/integrations.ts`: `requireClientSession()`,
zod `z.object({ pageId: z.string().regex(/^\d+$/) })`, call `selectPage`, `writePortalAudit`
action `integration.meta.page_selected`, return `{ok:true}` / `{error}` like `saveGbpLocation`.

Picker page `src/app/(portal)/portal/(authed)/integrations/meta/select-page/page.tsx`: same
structure as the Google picker; one radio card per page (`name`, `id`).

### OA-5 acceptance criteria

1. Start route 302s to `facebook.com/<version>/dialog/oauth` with the exact comma-joined scope
   list `leads_retrieval,pages_manage_posts,pages_show_list,pages_read_engagement` and a signed state.
2. Callback performs BOTH exchanges (code→short, short→long-lived) and stores only the
   long-lived token (Bitwarden); DB has no plaintext token anywhere.
3. State/nonce/session mismatch cases behave exactly as OA-2 AC 2 (shared code path).
4. Choosing a page: Bitwarden holds a page-token item, `metadata.pageId/pageName/pageTokenItemId`
   set, `POST /{page}/subscribed_apps` called with `subscribed_fields=leadgen`, status `active`.
5. Re-selecting a different page replaces the page-token secret (old item deleted) and
   re-subscribes the new page.
6. `GET /{page}/subscribed_apps` (manual check in Graph Explorer) shows the Seekly app with
   field `leadgen` after connecting.

---

## OA-6 — Meta leadgen webhook

Route: `src/app/api/webhooks/meta/leadgen/route.ts`. Configured in the Meta dashboard (OA-7,
step "Webhooks") with callback URL `https://<APP_URL host>/api/webhooks/meta/leadgen` and
verify token = env `META_WEBHOOK_VERIFY_TOKEN`.

Design: verify + ACK fast, do the Graph fetch in a pg-boss job (retries for free; a slow Graph
call never times out the webhook; lead loss is unacceptable per doc 03 WF-2).

### Queue additions — `src/server/pipeline/queue.ts` (edit)

```ts
// in QUEUES:
  metaLeadFetch: "meta-lead-fetch",
// in RETRY_QUEUES:
  metaLeadFetch: { retryLimit: 5, retryDelay: 60, retryBackoff: true },
// new job interface:
export interface MetaLeadFetchJob {
  pageId: string;
  leadgenId: string;
  formId: string | null;
  createdTime: number | null; // unix seconds from the webhook
}
```

### The route (complete)

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
import { env } from "@/env";
import { getBoss, QUEUES, type MetaLeadFetchJob } from "@/server/pipeline/queue";
import { log, logError } from "@/lib/logger";

/** Webhook verification handshake (Meta dashboard "Verify and save"). */
export async function GET(request: Request) {
  const url = new URL(request.url);
  const mode = url.searchParams.get("hub.mode");
  const token = url.searchParams.get("hub.verify_token");
  const challenge = url.searchParams.get("hub.challenge");
  if (mode === "subscribe" && token && env.META_WEBHOOK_VERIFY_TOKEN && token === env.META_WEBHOOK_VERIFY_TOKEN) {
    return new Response(challenge ?? "", { status: 200 });
  }
  return new Response("forbidden", { status: 403 });
}

/** X-Hub-Signature-256: "sha256=" + HMAC-SHA256(app secret, raw body). Fail closed. */
function validSignature(rawBody: string, header: string | null): boolean {
  if (!env.META_APP_SECRET || !header?.startsWith("sha256=")) return false;
  const expected = createHmac("sha256", env.META_APP_SECRET).update(rawBody, "utf8").digest("hex");
  const a = Buffer.from(header.slice(7));
  const b = Buffer.from(expected);
  return a.length === b.length && timingSafeEqual(a, b);
}

interface LeadgenChange {
  field: string;
  value: { page_id: string; leadgen_id: string; form_id?: string; created_time?: number };
}

export async function POST(request: Request) {
  const rawBody = await request.text(); // raw BEFORE parsing — signature is over exact bytes
  if (!validSignature(rawBody, request.headers.get("x-hub-signature-256"))) {
    return new Response("invalid signature", { status: 401 });
  }

  let body: { object?: string; entry?: Array<{ changes?: LeadgenChange[] }> };
  try {
    body = JSON.parse(rawBody);
  } catch {
    return new Response("bad json", { status: 400 });
  }
  if (body.object !== "page") return new Response("ok", { status: 200 });

  const boss = await getBoss();
  for (const entry of body.entry ?? []) {
    for (const change of entry.changes ?? []) {
      if (change.field !== "leadgen") continue;
      const job: MetaLeadFetchJob = {
        pageId: change.value.page_id,
        leadgenId: change.value.leadgen_id,
        formId: change.value.form_id ?? null,
        createdTime: change.value.created_time ?? null,
      };
      // singletonKey → webhook redelivery cannot double-enqueue the same lead.
      await boss
        .send(QUEUES.metaLeadFetch, job, { singletonKey: `leadgen-${job.leadgenId}` })
        .catch((err) => logError("[meta-webhook] enqueue failed", err));
      log(`[meta-webhook] leadgen ${job.leadgenId} for page ${job.pageId} enqueued`);
    }
  }
  // Always 200 — Meta disables webhooks that error repeatedly.
  return new Response("ok", { status: 200 });
}
```

### Worker handler — `src/server/connections/meta-lead-fetch.ts` (new file)

```ts
import { sql } from "drizzle-orm";
import { db } from "@/server/db";
import { connections, events } from "@/server/db/schema";
import { env } from "@/env";
import { getSecret } from "@/server/vault/bitwarden";
import { deterministicUuid } from "@/server/events/deterministic-id";
import type { MetaLeadFetchJob } from "@/server/pipeline/queue";
import { log } from "@/lib/logger";

const GRAPH = () => `https://graph.facebook.com/${env.META_GRAPH_VERSION}`;

/** Resolve which client owns this page, fetch full lead, write canonical lead.created. */
export async function fetchMetaLead(job: MetaLeadFetchJob): Promise<void> {
  const [conn] = await db
    .select()
    .from(connections)
    .where(sql`${connections.provider} = 'meta' AND ${connections.metadata} ->> 'pageId' = ${job.pageId}`)
    .limit(1);
  if (!conn) {
    log(`[meta-lead] no connection for page ${job.pageId} — ignoring`);
    return; // page unsubscribed/disconnected between webhook and job; not an error
  }
  const meta = conn.metadata as { pageTokenItemId?: string };
  if (!meta.pageTokenItemId) throw new Error(`connection ${conn.id} missing pageTokenItemId`);
  const pageToken = (await getSecret(meta.pageTokenItemId)).password;

  const res = await fetch(
    `${GRAPH()}/${job.leadgenId}?fields=id,created_time,field_data,ad_id,form_id&access_token=${encodeURIComponent(pageToken)}`,
  );
  if (!res.ok) throw new Error(`lead fetch ${res.status}: ${(await res.text()).slice(0, 300)}`); // pg-boss retries

  const lead = (await res.json()) as {
    id: string;
    created_time: string;
    field_data: Array<{ name: string; values: string[] }>;
    ad_id?: string;
    form_id?: string;
  };
  const fields = Object.fromEntries(lead.field_data.map((f) => [f.name.toLowerCase(), f.values[0] ?? ""]));

  // Canonical event envelope (doc 02). Deterministic id ⇒ webhook redelivery is a no-op.
  await db
    .insert(events)
    .values({
      id: deterministicUuid("meta-leadgen", lead.id),
      clientId: conn.clientId,
      type: "lead.created",
      occurredAt: new Date(lead.created_time),
      source: "meta",
      status: "pending",
      payload: {
        customer: {
          name: fields.full_name ?? fields.name ?? null,
          phone: fields.phone_number ?? fields.phone ?? null,
          email: fields.email ?? null,
        },
        form_id: lead.form_id ?? job.formId,
        ad_id: lead.ad_id ?? null,
        raw: lead.field_data,
      },
    })
    .onConflictDoNothing();
  log(`[meta-lead] lead.created for client ${conn.clientId} (lead ${lead.id})`);
}
```

Register in `src/worker.ts` next to the other handlers:

```ts
import { fetchMetaLead } from "./server/connections/meta-lead-fetch";
// ...
await boss.work<MetaLeadFetchJob>(QUEUES.metaLeadFetch, async ([job]) => {
  await fetchMetaLead(job.data);
});
```

(Spec 06's event forwarder picks `events.status='pending'` rows up and forwards to n8n; out of
scope here.)

### OA-6 acceptance criteria

1. `GET` with correct `hub.verify_token` echoes `hub.challenge` (200); wrong/absent token → 403.
2. `POST` with a body whose HMAC-SHA256 (app secret) doesn't match `X-Hub-Signature-256` → 401
   and nothing enqueued; missing header → 401. Signature is computed over the **raw** body.
3. Valid signed leadgen payload (use Meta's "Test" button in the Webhooks dashboard) →
   `meta-lead-fetch` job enqueued; worker writes exactly one `events` row with
   `type='lead.created'`, `source='meta'`, deterministic uuid.
4. Delivering the identical payload twice (Meta retries) produces one job (singletonKey) and,
   even if both run, one `events` row (`onConflictDoNothing`).
5. Graph fetch failure (kill network) → job retries per `RETRY_QUEUES` and lands in
   `meta-lead-fetch-dlq` after 5 attempts — verify the DLQ row.
6. A leadgen event for a page with no matching connection logs and completes without retry.

---

## OA-7 — Meta dev-mode tester runbook + App Review checklist

### Dev-mode setup (pilots run on this — no review needed)

1. developers.facebook.com → My Apps → Create App → Use case: **Other** → Type: **Business**
   → name `Seekly Platform`. Note the App ID/Secret → env `META_APP_ID`, `META_APP_SECRET`.
2. App Settings → Basic: fill Privacy Policy URL, App icon, Category (`Business and pages`).
   Add Platform → Website → Site URL = `APP_URL`.
3. Add products: **Facebook Login for Business** (set Valid OAuth Redirect URIs:
   `https://<APP_URL host>/api/oauth/meta/callback`) and **Webhooks**.
4. Webhooks → Page object → Subscribe to `leadgen` → Callback URL
   `https://<APP_URL host>/api/webhooks/meta/leadgen`, Verify token = env
   `META_WEBHOOK_VERIFY_TOKEN` → "Verify and save" (OA-6 GET must be deployed first).
5. **Testers for each pilot:** App Roles → Roles → Add Testers → enter the pilot owner's
   personal Facebook username/profile URL (from intake question A.6). They accept the invite
   at facebook.com → Settings → Apps and Websites → Developer requests. While the app is in
   Development mode, ONLY admins/developers/testers can complete the OAuth flow — this is the
   doc 06 pilot path.
6. The pilot's user must have a Page role (Admin) on the business page — verify before the
   connect call; `/me/accounts` returns only pages they manage.

### App Review submission checklist (submit by week 3, doc 06)

- [ ] Business verification completed (Meta Business Suite → Security Center) — required for
      `leads_retrieval` Advanced Access.
- [ ] Request **Advanced Access** for: `leads_retrieval`, `pages_manage_posts`,
      `pages_show_list`, `pages_read_engagement` (App Review → Permissions and Features).
- [ ] Per-permission written use case (template):
      `pages_show_list` — "Business owners select which of their Pages Seekly manages.";
      `leads_retrieval` — "Seekly retrieves our customer's own Lead Ads leads so our platform
      can respond to the lead within seconds on the customer's behalf.";
      `pages_manage_posts` — "Seekly publishes the customer's own blog-derived updates to
      their Page on a schedule they configure.";
      `pages_read_engagement` — "Read the customer's own Page content and metadata to report
      post performance back to them."
- [ ] Screencast per permission: full flow on a test client — portal login → Connect Facebook
      → consent → page picker → (leads_retrieval) submit a test lead via the Lead Ads Testing
      Tool (developers.facebook.com/tools/lead-ads-testing) and show it appearing in the
      portal → (pages_manage_posts) show a post published from Seekly.
- [ ] Test credentials for the reviewer: a dedicated portal login on a demo client + a test
      Page owned by a test user; put both in the review notes.
- [ ] Privacy policy URL + Data deletion instructions URL set (App Settings → Basic). A static
      page "email privacy@<domain> to delete your data" is acceptable.
- [ ] After approval: flip App Mode → Live. Dev-mode connections keep working.

### OA-7 acceptance criteria

1. A tester account (not admin/developer) completes connect + page select end-to-end in dev mode.
2. Lead Ads Testing Tool lead arrives as an `events` row within 60 seconds (proves webhook +
   OA-6 path with a real Meta delivery, not just the dashboard Test button).
3. App Review submitted with all four permissions; submission id recorded in the ops checklist.

---

## OA-8 — Token refresh: `getFreshAccessToken(connectionId)`

`src/server/connections/tokens.ts` (new file). Contract: returns a provider access token valid
for ≥60 seconds, refreshing if needed. Concurrent callers (web + worker + n8n context endpoint)
must not double-refresh: a Postgres **transaction-scoped advisory lock** keyed on the
connection id serializes refreshes across all processes.

```ts
import { eq, sql } from "drizzle-orm";
import { db } from "@/server/db";
import { connections } from "@/server/db/schema";
import { env } from "@/env";
import { getSecret, storeSecret, deleteSecret } from "@/server/vault/bitwarden";
import { decryptToken, encryptToken } from "@/server/vault/token-crypto";
import { markConnectionBroken, type Connection } from "@/server/connections/store";
import { log, logError } from "@/lib/logger";

export class ConnectionUnusableError extends Error {
  constructor(public readonly connectionId: string, message: string) {
    super(message);
  }
}

const SKEW_MS = 120_000; // treat tokens expiring within 2 min as expired

function cachedTokenIfFresh(conn: Connection): string | null {
  if (!conn.accessTokenEnc || !conn.accessTokenExpiresAt) return null;
  if (conn.accessTokenExpiresAt.getTime() - Date.now() < SKEW_MS) return null;
  return decryptToken(conn.accessTokenEnc);
}

/**
 * Returns a fresh access token for the connection.
 * - google: cached AES token if >2min left, else refresh-token grant (locked).
 * - meta: the page access token from Bitwarden (long-lived; validated by OA-9 cron).
 * Throws ConnectionUnusableError when status is revoked/broken or refresh fails
 * permanently (caller surfaces module pause per doc 02).
 */
export async function getFreshAccessToken(connectionId: string): Promise<string> {
  const [conn] = await db.select().from(connections).where(eq(connections.id, connectionId)).limit(1);
  if (!conn) throw new ConnectionUnusableError(connectionId, "Connection not found.");
  if (conn.status === "revoked") throw new ConnectionUnusableError(connectionId, "Connection was disconnected.");

  if (conn.provider === "meta") {
    const meta = conn.metadata as { pageTokenItemId?: string };
    if (!meta.pageTokenItemId) throw new ConnectionUnusableError(connectionId, "No page selected yet.");
    return (await getSecret(meta.pageTokenItemId)).password;
  }
  if (conn.provider === "wordpress") {
    throw new ConnectionUnusableError(connectionId, "WordPress uses Basic auth, not bearer tokens (see OA-10).");
  }

  // --- google ---
  const cached = cachedTokenIfFresh(conn);
  if (cached) return cached;

  return db.transaction(async (tx) => {
    // Two-int advisory lock: (namespace, connection). hashtext() is stable per DB.
    await tx.execute(
      sql`SELECT pg_advisory_xact_lock(hashtext('connection-refresh'), hashtext(${connectionId}))`,
    );
    // Someone may have refreshed while we waited on the lock — re-read.
    const [fresh] = await tx.select().from(connections).where(eq(connections.id, connectionId)).limit(1);
    const nowCached = fresh ? cachedTokenIfFresh(fresh) : null;
    if (nowCached) return nowCached;
    if (!fresh?.refreshTokenItemId) throw new ConnectionUnusableError(connectionId, "No refresh token stored.");

    const refreshToken = (await getSecret(fresh.refreshTokenItemId)).password;
    const res = await fetch("https://oauth2.googleapis.com/token", {
      method: "POST",
      headers: { "content-type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        client_id: env.GOOGLE_OAUTH_CLIENT_ID!,
        client_secret: env.GOOGLE_OAUTH_CLIENT_SECRET!,
        refresh_token: refreshToken,
        grant_type: "refresh_token",
      }),
    });

    if (!res.ok) {
      const bodyText = await res.text();
      // invalid_grant = user revoked access / token expired → permanently broken.
      if (res.status === 400 && bodyText.includes("invalid_grant")) {
        await markConnectionBroken(connectionId, "Google access was revoked. Reconnect Google.");
        throw new ConnectionUnusableError(connectionId, "Google refresh token revoked.");
      }
      // Transient (5xx / 429): DON'T mark broken; let the caller retry.
      logError(`[tokens] google refresh transient failure ${res.status}`, bodyText.slice(0, 300));
      throw new Error(`Google token refresh failed (${res.status}).`);
    }

    const tok = (await res.json()) as { access_token: string; expires_in: number; refresh_token?: string };

    // Rare: Google rotates the refresh token. Store new secret, delete old.
    let refreshTokenItemId = fresh.refreshTokenItemId;
    if (tok.refresh_token && tok.refresh_token !== refreshToken) {
      const { itemId } = await storeSecret({
        clientId: fresh.clientId,
        label: "oauth-google-durable-token",
        secret: { username: fresh.providerAccountId ?? undefined, password: tok.refresh_token },
      });
      await deleteSecret(fresh.refreshTokenItemId).catch(() => {});
      refreshTokenItemId = itemId;
    }

    await tx
      .update(connections)
      .set({
        accessTokenEnc: encryptToken(tok.access_token),
        accessTokenExpiresAt: new Date(Date.now() + tok.expires_in * 1000),
        refreshTokenItemId,
        status: fresh.status === "broken" ? "active" : fresh.status, // successful refresh heals
        statusReason: fresh.status === "broken" ? null : fresh.statusReason,
        lastVerifiedAt: new Date(),
        updatedAt: new Date(),
      })
      .where(eq(connections.id, connectionId));

    log(`[tokens] refreshed google token for connection ${connectionId}`);
    return tok.access_token;
  });
}
```

> n8n never calls Google/Meta token endpoints. Spec 06's
> `GET /api/internal/clients/:id/context` calls `getFreshAccessToken` and hands n8n the
> short-lived access token only (doc 02: "n8n receives short-lived access tokens per run,
> never refresh tokens").

### OA-8 acceptance criteria

1. With a fresh cache (>2 min left), `getFreshAccessToken` makes zero network calls (assert
   with a mocked `fetch` in vitest).
2. Two concurrent calls on an expired token perform exactly ONE token-endpoint request
   (test: `Promise.all` of two calls against a mocked fetch with a 200 ms delay; assert
   `fetch` called once — the advisory lock + re-read makes the second a cache hit).
3. `invalid_grant` marks the connection `broken` with `statusReason` "Google access was
   revoked. Reconnect Google." and throws `ConnectionUnusableError`; a 503 from Google throws
   a plain `Error` and does NOT change status.
4. Refresh-token rotation path stores the new secret and deletes the old Bitwarden item.
5. `meta` connections return the page token from `getSecret(metadata.pageTokenItemId)`
   without touching Google endpoints; a meta connection with no page selected throws
   `ConnectionUnusableError`.
6. A successful refresh on a previously-`broken` connection flips it back to `active`.

---

## OA-9 — Health-check cron, module auto-pause, portal banner, disconnect

### Cron job

Queue: add to `QUEUES` → `connectionsHealth: "connections-health"`, `RETRY_QUEUES` →
`connectionsHealth: { retryLimit: 1, retryDelay: 300 }`. In `src/worker.ts`:

```ts
import { sweepConnectionsHealth } from "./server/connections/health";
// ...
await boss.work(QUEUES.connectionsHealth, async () => {
  const r = await sweepConnectionsHealth();
  log(`[worker] connections-health: ${r.checked} checked, ${r.broken} broken, ${r.paused} modules paused`);
});
await boss.schedule(QUEUES.connectionsHealth, "30 6 * * *"); // daily 06:30 UTC
```

### `src/server/connections/health.ts` (new file)

```ts
import { and, eq, inArray } from "drizzle-orm";
import { db } from "@/server/db";
import { activityLog, connections, workflowConfig } from "@/server/db/schema";
import { env } from "@/env";
import { getSecret } from "@/server/vault/bitwarden";
import { getFreshAccessToken, ConnectionUnusableError } from "@/server/connections/tokens";
import { markConnectionBroken } from "@/server/connections/store";
import { log, logError } from "@/lib/logger";

/** Which modules depend on which provider (doc 02: broken connection → auto-pause). */
const MODULES_BY_PROVIDER: Record<string, string[]> = {
  google: ["review_automation", "social_syndication", "directory_sync", "content_engine"],
  meta: ["speed_to_lead", "social_syndication"],
  wordpress: ["content_engine"],
};

async function checkOne(conn: typeof connections.$inferSelect): Promise<{ ok: boolean; reason?: string }> {
  try {
    if (conn.provider === "google") {
      // Force a real refresh-token exercise by ignoring the cache: expire it first.
      await db
        .update(connections)
        .set({ accessTokenExpiresAt: new Date(0) })
        .where(eq(connections.id, conn.id));
      await getFreshAccessToken(conn.id);
      return { ok: true };
    }
    if (conn.provider === "meta") {
      const meta = conn.metadata as { pageTokenItemId?: string };
      if (!meta.pageTokenItemId) return { ok: false, reason: "No Facebook page selected." };
      const pageToken = (await getSecret(meta.pageTokenItemId)).password;
      const appToken = `${env.META_APP_ID}|${env.META_APP_SECRET}`;
      const res = await fetch(
        `https://graph.facebook.com/${env.META_GRAPH_VERSION}/debug_token?input_token=${encodeURIComponent(pageToken)}&access_token=${encodeURIComponent(appToken)}`,
      );
      const data = (await res.json()) as { data?: { is_valid?: boolean } };
      return data.data?.is_valid
        ? { ok: true }
        : { ok: false, reason: "Facebook access is no longer valid. Reconnect Facebook." };
    }
    if (conn.provider === "wordpress") {
      const meta = conn.metadata as { siteUrl?: string; username?: string; clientCredentialId?: string };
      // App password lives in clientCredentials.bitwardenItemId — see OA-10.
      const item = await wordpressSecretItemId(conn); // helper below
      if (!item || !meta.siteUrl || !meta.username) return { ok: false, reason: "WordPress details incomplete." };
      const appPassword = (await getSecret(item)).password;
      const res = await fetch(`${meta.siteUrl.replace(/\/$/, "")}/wp-json/wp/v2/users/me`, {
        headers: { authorization: `Basic ${Buffer.from(`${meta.username}:${appPassword}`).toString("base64")}` },
      });
      return res.ok
        ? { ok: true }
        : { ok: false, reason: `WordPress rejected the application password (HTTP ${res.status}).` };
    }
    return { ok: true };
  } catch (err) {
    if (err instanceof ConnectionUnusableError) {
      return { ok: false, reason: "Google access was revoked. Reconnect Google." };
    }
    // Transient (network/5xx): report ok=true-ish? No — skip status change, log only.
    logError(`[health] transient failure checking connection ${conn.id}`, err);
    return { ok: true }; // never mark broken on a transient error; tomorrow re-checks
  }
}

export async function sweepConnectionsHealth(): Promise<{ checked: number; broken: number; paused: number }> {
  const rows = await db
    .select()
    .from(connections)
    .where(inArray(connections.status, ["active", "broken"]));

  let broken = 0;
  let paused = 0;
  for (const conn of rows) {
    const result = await checkOne(conn);
    if (result.ok) {
      await db
        .update(connections)
        .set({ status: "active", statusReason: null, lastVerifiedAt: new Date(), updatedAt: new Date() })
        .where(eq(connections.id, conn.id));
      continue;
    }
    broken++;
    if (conn.status !== "broken") {
      await markConnectionBroken(conn.id, result.reason ?? "Connection stopped working.");
      // Auto-pause dependent modules that are currently on (doc 02 / doc 03 switch table).
      const modules = MODULES_BY_PROVIDER[conn.provider] ?? [];
      const flipped = await db
        .update(workflowConfig)
        .set({ enabled: false, updatedBy: "system:connections-health", updatedAt: new Date() })
        .where(
          and(
            eq(workflowConfig.clientId, conn.clientId),
            inArray(workflowConfig.module, modules),
            eq(workflowConfig.enabled, true),
          ),
        )
        .returning({ module: workflowConfig.module });
      paused += flipped.length;
      for (const f of flipped) {
        await db.insert(activityLog).values({
          clientId: conn.clientId,
          engine: f.module,
          action: "module.auto_paused",
          status: "ok",
          entityType: "connection",
          entityId: conn.id,
          detail: { reason: result.reason, provider: conn.provider },
        });
      }
      log(`[health] connection ${conn.id} (${conn.provider}) broken; paused ${flipped.length} module(s)`);
    }
  }
  return { checked: rows.length, broken, paused };
}

async function wordpressSecretItemId(conn: typeof connections.$inferSelect): Promise<string | null> {
  const meta = conn.metadata as { clientCredentialId?: string };
  if (!meta.clientCredentialId) return null;
  const { clientCredentials } = await import("@/server/db/schema");
  const [cred] = await db
    .select({ bitwardenItemId: clientCredentials.bitwardenItemId })
    .from(clientCredentials)
    .where(eq(clientCredentials.id, meta.clientCredentialId))
    .limit(1);
  return cred?.bitwardenItemId ?? null;
}
```

Notes:
- Auto-pause is **one-way**: a healed connection flips back to `active` but modules stay off
  until a Seekly admin re-enables them in the switchboard (deliberate — doc 05 onboarding flow
  gives module-on authority to admins).
- Recovered vs. broken transitions both surface in the ops daily digest via `activity_log`.

### Portal banner

In `src/app/(portal)/portal/(authed)/layout.tsx`, after `getClientById`, query broken
connections and render a banner above `{children}`:

```ts
const broken = await db
  .select({ provider: connections.provider, statusReason: connections.statusReason })
  .from(connections)
  .where(and(eq(connections.clientId, s.clientId!), eq(connections.status, "broken")));
```

```tsx
{broken.length > 0 && (
  <div className="border-b border-amber-300 bg-amber-50 px-4 py-3 text-sm text-amber-900">
    <strong>Action needed:</strong>{" "}
    {broken.map((b) => b.statusReason ?? `Your ${b.provider} connection stopped working.`).join(" ")}{" "}
    Some automations are paused until you reconnect.{" "}
    <a href="/portal/integrations" className="font-medium underline">Fix it in Integrations →</a>
  </div>
)}
```

### Disconnect flow (server action in `src/server/actions/integrations.ts`)

```ts
export async function disconnectIntegration(
  provider: "google" | "meta" | "wordpress",
): Promise<{ error: string } | { ok: true }> {
  const s = await requireClientSession();
  const conn = await getConnection(s.clientId!, provider);
  if (!conn) return { error: "Nothing to disconnect." };

  // 1. Best-effort remote revoke (never block deletion on it).
  try {
    if (provider === "google" && conn.refreshTokenItemId) {
      const refreshToken = (await getSecret(conn.refreshTokenItemId)).password;
      await fetch(`https://oauth2.googleapis.com/revoke?token=${encodeURIComponent(refreshToken)}`, { method: "POST" });
    }
    if (provider === "meta" && conn.refreshTokenItemId) {
      const userToken = (await getSecret(conn.refreshTokenItemId)).password;
      await fetch(
        `https://graph.facebook.com/${env.META_GRAPH_VERSION}/me/permissions?access_token=${encodeURIComponent(userToken)}`,
        { method: "DELETE" },
      );
    }
  } catch (err) {
    logError(`[integrations] remote revoke failed for ${provider}`, err); // continue regardless
  }

  // 2. Delete secrets from the vault (doc 02: revoked + deleted on disconnect).
  if (conn.refreshTokenItemId) await deleteSecret(conn.refreshTokenItemId).catch(() => {});
  const meta = conn.metadata as { pageTokenItemId?: string };
  if (meta.pageTokenItemId) await deleteSecret(meta.pageTokenItemId).catch(() => {});

  // 3. Null the row's token material, keep the row for history/UX.
  await db
    .update(connections)
    .set({
      status: "revoked",
      statusReason: null,
      refreshTokenItemId: null,
      accessTokenEnc: null,
      accessTokenExpiresAt: null,
      updatedAt: new Date(),
    })
    .where(eq(connections.id, conn.id));

  // 4. Auto-pause dependent modules (same mapping as health.ts — export MODULES_BY_PROVIDER).
  // ...identical update to sweepConnectionsHealth's pause block...

  await writePortalAudit({
    clientId: s.clientId!,
    actorKind: "client",
    actorId: s.clientUserId!,
    action: `integration.${provider}.disconnected`,
    detail: conn.providerAccountId ?? "",
  });
  return { ok: true };
}
```

### OA-9 acceptance criteria

1. `connections-health` is scheduled at `30 6 * * *` and shows in `pgboss.schedule`.
2. Revoking Seekly's access from a Google test account (myaccount.google.com → Security →
   Third-party access) causes the next sweep to: mark the connection `broken`, flip every
   enabled google-dependent `workflow_config` row to `enabled=false` with
   `updatedBy='system:connections-health'`, and write one `activity_log`
   `module.auto_paused` row per paused module.
3. The portal shows the amber banner on every authed page while any connection is `broken`;
   it disappears (without deploy) after reconnect.
4. Transient failure (health check run with network blackholed) changes **no** statuses.
5. Disconnect: Google token no longer appears in myaccount.google.com third-party access;
   both Bitwarden items deleted; row `status='revoked'` with null token columns; dependent
   modules off; audit row written.
6. A healed connection (sweep succeeds after broken) becomes `active` but previously paused
   modules remain `enabled=false`.

---

## OA-10 — WordPress connect (application password)

No OAuth — the client creates a WordPress **Application Password** and pastes it. Secret goes
to the existing `clientCredentials` table exactly like `src/server/actions/portal-credentials.ts`
does (Bitwarden via `storeSecret`, only `bitwardenItemId` in the DB), plus a `connections` row
so the Integrations page and health cron treat it like any other provider.

### Server action (add to `src/server/actions/integrations.ts`)

```ts
const wpSchema = z.object({
  siteUrl: z.string().url().refine((u) => u.startsWith("https://"), "Must be an https:// URL"),
  username: z.string().min(1),
  appPassword: z.string().min(8), // WP shows "xxxx xxxx xxxx xxxx xxxx xxxx"; spaces are OK
});

export async function connectWordPress(
  input: z.infer<typeof wpSchema>,
): Promise<{ error: string } | { ok: true; displayName: string }> {
  const s = await requireClientSession();
  const parsed = wpSchema.safeParse(input);
  if (!parsed.success) return { error: "Enter your site URL, username, and application password." };
  const { siteUrl, username, appPassword } = parsed.data;
  const base = siteUrl.replace(/\/$/, "");

  // 1. TEST CALL before storing anything.
  let me: { name?: string; capabilities?: Record<string, boolean> };
  try {
    const res = await fetch(`${base}/wp-json/wp/v2/users/me`, {
      headers: { authorization: `Basic ${Buffer.from(`${username}:${appPassword}`).toString("base64")}` },
      signal: AbortSignal.timeout(10_000),
    });
    if (res.status === 401 || res.status === 403) {
      return { error: "WordPress rejected those credentials. Check the username and application password." };
    }
    if (!res.ok) return { error: `Couldn't reach the WordPress API (HTTP ${res.status}). Is the REST API enabled?` };
    me = await res.json();
  } catch {
    return { error: "Couldn't reach that site. Check the URL (it must be the WordPress site address)." };
  }
  if (me.capabilities && !me.capabilities.edit_posts) {
    return { error: "That user can't publish posts. Use an Editor or Administrator account." };
  }

  // 2. Secret → Bitwarden via the existing vault module.
  let itemId: string;
  try {
    ({ itemId } = await storeSecret({
      clientId: s.clientId!,
      label: "WordPress application password",
      secret: { username, password: appPassword, url: base },
    }));
  } catch (err) {
    if (err instanceof VaultNotConfiguredError || err instanceof VaultError) {
      logError("[integrations] vault error", vaultDiagnostics(), err);
      return { error: "The credential vault isn't set up yet. Ask your Seekly contact." };
    }
    throw err;
  }

  // 3. clientCredentials row (existing table) + connections row.
  const [cred] = await db
    .insert(clientCredentials)
    .values({
      clientId: s.clientId!,
      label: "Website (WordPress)",
      platform: "WordPress",
      accessKind: "password",
      accountHint: username,
      bitwardenItemId: itemId,
      notes: base,
    })
    .returning({ id: clientCredentials.id });

  await db
    .insert(connections)
    .values({
      clientId: s.clientId!,
      provider: "wordpress",
      providerAccountId: username,
      accountHint: me.name ?? username,
      scopes: [],
      status: "active",
      lastVerifiedAt: new Date(),
      metadata: { siteUrl: base, username, clientCredentialId: cred.id },
    })
    .onConflictDoUpdate({
      target: [connections.clientId, connections.provider],
      set: {
        providerAccountId: username,
        accountHint: me.name ?? username,
        status: "active",
        statusReason: null,
        lastVerifiedAt: new Date(),
        metadata: { siteUrl: base, username, clientCredentialId: cred.id },
        updatedAt: new Date(),
      },
    });

  await writePortalAudit({
    clientId: s.clientId!,
    actorKind: "client",
    actorId: s.clientUserId!,
    action: "integration.wordpress.connected",
    detail: base,
  });
  return { ok: true, displayName: me.name ?? username };
}
```

### Portal form

`src/app/(portal)/portal/(authed)/integrations/wordpress/page.tsx` — client component form
with three fields and inline how-to copy:

> **How to create an application password**
> 1. In your WordPress admin, go to **Users → Profile** (log in as an Administrator or Editor).
> 2. Scroll to **Application Passwords**, enter the name `Seekly`, click **Add New Application Password**.
> 3. Copy the generated password (spaces included are fine) and paste it below with your
>    WordPress username and site address.

On success: toast "Connected — we can now publish content to {displayName}'s site." and
redirect to `/portal/integrations?connected=wordpress`.

### OA-10 acceptance criteria

1. Correct credentials against a real WordPress site: `clientCredentials` row with
   `bitwardenItemId` set and NO password column anywhere; `connections` row
   `provider='wordpress'`, `status='active'`, `metadata.siteUrl/username/clientCredentialId`.
2. Wrong app password → "WordPress rejected those credentials..." and **nothing stored**
   (neither table, no Bitwarden item).
3. Non-WordPress URL / unreachable host → friendly error within ~10 s (timeout enforced).
4. A Subscriber-role user is rejected with the "can't publish posts" error.
5. Reconnecting updates the single `wordpress` connection row (`onConflictDoUpdate`) rather
   than violating the unique index; a NEW credentials row is added (history preserved,
   consistent with the credentials page's model).
6. Vault-not-configured environments return the friendly vault error string, same as the
   credentials page.

---

## OA-11 — Portal Integrations page

### Nav registration

`src/lib/portal-tabs.ts` — add to `PORTAL_TAB_REGISTRY` (import `Plug` from lucide-react):

```ts
{ key: "integrations", label: "Integrations", href: "/portal/integrations", icon: Plug, group: "Account" },
```

Enable the tab in the portal-config default set (same mechanism that gates `credentials` —
see the `portal_config` default in `src/server/db/schema.ts`).

### Page — `src/app/(portal)/portal/(authed)/integrations/page.tsx`

Server component: `requireClientSession()` → load all `connections` rows for the client →
render one card per provider (Google, Facebook, Website/WordPress) with the state derived from
the row (`no row or status='revoked'` ⇒ not_connected). Buttons: plain `<a>` to
`/api/oauth/google/start` / `/api/oauth/meta/start`; link to `/portal/integrations/wordpress`;
"Disconnect" wired to `disconnectIntegration` with a confirm dialog. Read the `?error=` and
`?connected=` query params and render a toast/inline alert (map every `error` code used in
OA-2/OA-5 to the copy below).

### Exact UI copy (required — use verbatim)

**Google card** (title: "Google", subtitle: "Business Profile, Search Console & Analytics")

| State | Body copy | Button |
|---|---|---|
| not_connected | "Connect your Google account so Seekly can manage your Business Profile — reviews, posts, and hours — and report your search performance." | `Connect Google` |
| pending (no location yet) | "Almost done — choose which business location Seekly should manage." | `Choose location` |
| pending + `GOOGLE_OAUTH_MODE=pending_verification` and GBP list failed | "You're connected. Google is still approving Seekly's Business Profile access — we'll finish this step automatically, and your Seekly team is covering it manually in the meantime." | (none) |
| active | "Connected as **{accountHint}** · Location: **{locationTitle}**" | `Disconnect` |
| broken | "We've lost access to your Google account, so some automations are paused. Reconnecting takes about a minute." + statusReason in muted text | `Reconnect Google` |

**Facebook card** (title: "Facebook", subtitle: "Lead Ads & Page posts")

| State | Body copy | Button |
|---|---|---|
| not_connected | "Connect Facebook so Seekly can answer your Lead Ads inquiries in seconds and post updates to your Page." | `Connect Facebook` |
| pending | "Almost done — choose which Facebook Page Seekly should manage." | `Choose Page` |
| active | "Connected · Page: **{pageName}**" | `Disconnect` |
| broken | "We've lost access to your Facebook Page, so lead replies and Page posts are paused. Reconnecting takes about a minute." + statusReason | `Reconnect Facebook` |

**Website card** (title: "Website", subtitle: "Content publishing (WordPress)")

| State | Body copy | Button |
|---|---|---|
| not_connected | "Connect your WordPress site so Seekly can publish your content directly. Not on WordPress? Your Seekly team will set up email delivery instead." | `Connect WordPress` |
| active | "Connected as **{accountHint}** · {metadata.siteUrl}" | `Disconnect` |
| broken | "Your WordPress application password stopped working — publishing is paused. Create a new application password and reconnect." | `Reconnect WordPress` |

Error-param alerts (single amber inline alert atop the page):
`google_denied|meta_denied` → "No problem — you can connect any time."
`google_scope_missing` → "Google connected, but the Business Profile permission was unticked. Please reconnect and leave all boxes checked."
`google_not_configured|meta_not_configured` → "This connection isn't available yet — your Seekly team is on it."
anything else → "Something went wrong connecting. Please try again, or message your Seekly team."

### OA-11 acceptance criteria

1. `/portal/integrations` renders three cards; each shows the correct state for: no row,
   `pending`, `active`, `broken`, `revoked` (revoked renders as not_connected).
2. Copy matches the tables above verbatim (checked by a snapshot/string test on the rendered
   card body for each state).
3. The Integrations tab appears in the portal nav in the Account group; deep links from the
   broken-connection banner (OA-9) land here.
4. Every `?error=` code produced by OA-2/OA-5 maps to one of the four alert strings — no raw
   codes ever shown.
5. Mobile (375 px): cards stack, buttons full-width tappable (doc 05: mobile-first).

---

## Out of scope (owned elsewhere)

- `connections`/`events`/`workflow_config`/`activity_log` migrations — Spec 01.
- `GET /api/internal/clients/:id/context` (n8n token hand-off) and the event forwarder to n8n
  — Spec 06 / internal-API spec; they consume `getFreshAccessToken` from this spec.
- GBP review polling, review replies, GBP/Facebook post publishing — engine specs (they call
  `getFreshAccessToken`).
- Twilio/A2P and POS email-parse — comms spec (doc 06).
