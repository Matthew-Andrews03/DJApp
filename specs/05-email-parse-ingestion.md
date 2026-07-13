# Spec 05 — Universal Email-Parse Ingestion (adapter ladder rung 3) + CSV Import (rung 4)

**Repo:** all file paths below are in `seekly-client-insights` (Next.js 16 + Drizzle + Postgres + pg-boss + zod).
**Source requirements:** doc 02 §"The adapter ladder" (rungs 3–4), doc 06 §"Email-parse infrastructure" + §"POS / booking systems", doc 03 WF-1/WF-2 triggers, doc 07 week-1 ("email-parse infra live, tuned on the pilot's real POS notification sample"). Doc 10 supersedes stack assumptions.
**Depends on spec 03:** the `commsProvisioning` table (holds `inboundEmailAddress`), the pg-boss `QUEUES` pattern, `sendOpsAlert`, and `toE164` from `src/server/messaging/compliance.ts`. Depends on spec 02 for `emitCanonicalEvent` (`src/server/events/emit.ts`) and spec 01 for `customers`, `workflow_config`, and `ops_review_queue` (minimal definitions reproduced below so this spec is buildable stand-alone).

## Purpose

Rung 3 of the adapter ladder: **literally any software that sends email** becomes a Seekly event source with zero client-side code. Each client gets `{slug}@in.seekly.app`; they forward/CC their POS "sale completed" receipts and website-form notification emails there. SendGrid Inbound Parse POSTs the email to us; Claude Haiku extracts a structured record; high-confidence extractions become canonical `sale.completed` / `lead.created` events (which drive WF-1 review requests and WF-2 speed-to-lead); low-confidence extractions go to an ops review queue — **a low-confidence parse must never trigger a customer-facing message** (doc 03 WF-2 failure handling: "never a wrong text to a customer").

Rung 4 rides along: a portal CSV upload loads the client's customer history into `customers` with `consent_basis = 'imported_list'` (the WF-3 reactivation audience).

```
POS / form email → {slug}@in.seekly.app → SendGrid MX → POST /api/inbound/email
  → inbound_emails row (durable, idempotent on Message-ID)
  → pg-boss "email-parse" job
  → Claude Haiku extraction (few-shot from per-sender-domain examples)
      ├─ confidence ≥ 0.8 → emitCanonicalEvent(sale.completed | lead.created)
      └─ confidence < 0.8 or no contact info → ops_review_queue (admin approves/edits/discards)
```

---

## Ticket table

| # | Ticket | Depends on | Size |
|---|---|---|---|
| EP-1 | SendGrid Inbound Parse setup: DNS for `in.seekly.app`, dashboard config, env vars | — | S |
| EP-2 | Schema: `inbound_emails`, `email_extraction_examples`, `ops_review_queue` (+ address assignment) | SP-1 | M |
| EP-3 | `POST /api/inbound/email` route (auth, size limits, client resolution, dedupe) + `email-parse` queue | EP-1, EP-2 | M |
| EP-4 | Claude Haiku extraction module: prompt template, JSON schema, zod validation, cost tracking | EP-2 | M |
| EP-5 | Confidence-threshold flow: canonical event emit vs ops review queue; admin approval actions | EP-3, EP-4, spec 02 | M |
| EP-6 | Per-client template learning: sender-domain few-shot example store | EP-4, EP-5 | S |
| EP-7 | CSV import (rung 4): preview + commit endpoints, column-mapping contract, `customers` upsert | spec 01 `customers` | M |
| EP-8 | Test pack & pilot tuning (real POS sample from intake Q16) | EP-3–EP-7 | M |

New env vars (add to the zod schema in `src/env.ts`):

```ts
  // --- Optional: inbound email parse (SendGrid Inbound Parse webhook) ---
  // Shared secret in the webhook URL; absent = route returns 503 (not wired up yet).
  EMAIL_INBOUND_WEBHOOK_KEY: z.string().min(32).optional(),
```

Existing env vars used: `ANTHROPIC_API_KEY` + `ENGINE_MODE` (extraction runs live only when both allow, same convention as `src/server/analysis/review-insights.ts`), `SPINE_SERVICE_TOKEN` (spec 02 event emit), `OPS_ALERT_EMAIL` (spec 03).

New dependency (EP-7): `npm install csv-parse` (battle-tested streaming CSV parser; do not hand-roll quoting rules).

---

## EP-1 — SendGrid Inbound Parse setup

Ops/infra ticket — no code except the env var. Doc 06 names SendGrid Inbound Parse as the chosen service (Postmark is the fallback; nothing below leaks SendGrid specifics past the route handler, so a swap later only touches EP-3's form-field parsing).

### 1. DNS — exactly one record

At the DNS provider for `seekly.app`, add:

| Host | Type | Priority | Value | TTL |
|---|---|---|---|---|
| `in.seekly.app` | MX | 10 | `mx.sendgrid.net.` | 3600 |

Nothing else. Do **not** add an MX for the root `seekly.app` (normal mail is unaffected); do not add SPF/DKIM for `in.` (those govern *outbound* mail — irrelevant for a receive-only subdomain). Verify with:

```bash
dig +short MX in.seekly.app   # expect: 10 mx.sendgrid.net.
```

### 2. SendGrid dashboard

1. Log in → **Settings → Inbound Parse** → **Add Host & URL**.
2. If prompted to pick an authenticated domain: first add `seekly.app` under **Settings → Sender Authentication → Authenticate Your Domain** (SendGrid requires the root domain on the account; complete its CNAME records). This is account plumbing, not mail-flow.
3. Form fields:
   - **Receiving subdomain:** `in`
   - **Domain:** `seekly.app`
   - **Destination URL:** `https://<production host of APP_URL>/api/inbound/email?key=<EMAIL_INBOUND_WEBHOOK_KEY>`
   - **Check "POST the raw, full MIME message":** ❌ leave unchecked (we want SendGrid's parsed fields: `from`, `to`, `subject`, `text`, `html`, `envelope`, `SPF`, `dkim`, `spam_score`).
   - **Check "Check incoming emails for spam":** ✅ check it (adds `spam_score` + `spam_report` fields we store).
4. Save. SendGrid now accepts mail for `*@in.seekly.app` and POSTs each message as `multipart/form-data` to the URL, retrying on non-2xx for up to ~72 h (which is why EP-3 answers 200 for everything except auth failures).

### 3. Secret

```bash
openssl rand -hex 32   # → EMAIL_INBOUND_WEBHOOK_KEY in .env / .env.production
```

The `?key=` query parameter is the only authentication SendGrid supports (Inbound Parse does not sign requests). Treat the full URL as a secret; rotating = generate new value, update env, update dashboard URL.

**Acceptance criteria**

1. `dig +short MX in.seekly.app` returns `10 mx.sendgrid.net.`.
2. Sending any email to `test@in.seekly.app` produces a POST hit in the app logs (deploy EP-3 first, or point the URL temporarily at a request-bin to verify plumbing).
3. `EMAIL_INBOUND_WEBHOOK_KEY` present in prod env; the dashboard URL contains it.
4. A runbook note in the ticket records where the dashboard entry lives and the rotation procedure.

---

## EP-2 — Schema

Add to `src/server/db/schema.ts` (then `npm run db:generate` + `npm run db:migrate`).

```ts
// ---------------------------------------------------------------------------
// Email-parse ingestion (spec 05): durable inbound-email log, per-sender-domain
// extraction examples (few-shot learning), and the ops review queue.
// ---------------------------------------------------------------------------

export const inboundEmailStatusEnum = pgEnum("inbound_email_status", [
  "received", // stored, parse job queued
  "parsed",   // high-confidence → canonical event emitted
  "review",   // low-confidence → sitting in ops_review_queue
  "ignored",  // classification "other", unknown recipient, or ops discarded
  "failed",   // extraction crashed after retries (job in email-parse-dlq)
]);

/**
 * One row per email received at {slug}@in.seekly.app. Durable + replayable:
 * the raw text is kept so an extraction can be re-run after prompt tuning.
 * Idempotent on the RFC-5322 Message-ID header (SendGrid redelivers on
 * non-2xx and some POS systems re-send).
 */
export const inboundEmails = pgTable(
  "inbound_emails",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    /** Null when the recipient slug didn't resolve (row kept for ops forensics). */
    clientId: uuid("client_id").references(() => clients.id, { onDelete: "cascade" }),
    toAddress: text("to_address").notNull(),
    fromAddress: text("from_address").notNull(),
    /** Lowercased domain of fromAddress — the few-shot example key. */
    senderDomain: text("sender_domain").notNull(),
    subject: text("subject").notNull().default(""),
    /** Plain text body (or naive-stripped HTML), truncated to 100k chars. */
    textBody: text("text_body").notNull(),
    /** RFC-5322 Message-ID, angle brackets stripped. Null if absent. */
    messageId: text("message_id"),
    spf: text("spf"),          // SendGrid "SPF" field, e.g. "pass" / "fail" / "neutral"
    dkim: text("dkim"),        // SendGrid "dkim" field, e.g. "{@gmail.com : pass}"
    spamScore: text("spam_score"),
    status: inboundEmailStatusEnum("status").notNull().default("received"),
    /** The canonical events.id emitted from this email (= this row's id, see EP-5). */
    eventId: uuid("event_id"),
    /** Last extraction attempt (raw model output) — shown in the review UI. */
    extraction: jsonb("extraction"),
    confidence: real("confidence"),
    costUsd: numeric("cost_usd", { precision: 8, scale: 5 }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    // Postgres treats NULLs as distinct, so emails without a Message-ID never collide.
    uniqueIndex("inbound_emails_message_id_unique").on(t.messageId),
    index("inbound_emails_client_status_idx").on(t.clientId, t.status, t.createdAt),
  ],
);

/**
 * Few-shot memory: verified (extraction, excerpt) pairs per (client, sender
 * domain). Prepended to future extraction prompts for the same sender so the
 * parser "learns" each POS template (doc 06: "prompt-tuned per template,
 * improves per client"). Per-client only — never shared across clients (PII).
 */
export const emailExtractionExamples = pgTable(
  "email_extraction_examples",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    senderDomain: text("sender_domain").notNull(),
    /** First 2,000 chars of the email the extraction was verified against. */
    emailExcerpt: text("email_excerpt").notNull(),
    /** The verified ExtractionResult JSON (post-ops-edit when source="ops"). */
    extraction: jsonb("extraction").notNull(),
    /** "auto" = first high-confidence parse; "ops" = human-verified (preferred). */
    source: text("source").notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("email_examples_lookup_idx").on(t.clientId, t.senderDomain, t.createdAt)],
);

export const opsReviewStatusEnum = pgEnum("ops_review_status", [
  "open",
  "approved",
  "discarded",
]);

/**
 * Generic ops review queue (doc 05 admin "Ops queues"; owned by spec 01 —
 * if spec 01 already defined it, reconcile: these columns are required).
 */
export const opsReviewQueue = pgTable(
  "ops_review_queue",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    /** "email_parse_low_confidence" here; other specs add their own kinds. */
    kind: text("kind").notNull(),
    /** For this kind: { inbound_email_id } */
    ref: jsonb("ref").notNull(),
    payload: jsonb("payload").notNull(),
    status: opsReviewStatusEnum("status").notNull().default("open"),
    resolution: jsonb("resolution"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    resolvedAt: timestamp("resolved_at", { withTimezone: true }),
    resolvedBy: text("resolved_by"),
  },
  (t) => [index("ops_review_open_idx").on(t.status, t.kind, t.createdAt)],
);
```

(Imports needed at the top of `schema.ts`: `real`, `numeric` from `drizzle-orm/pg-core` — `numeric` is already imported.)

**Address assignment:** `commsProvisioning.inboundEmailAddress` is set to `${slug}@in.seekly.app` by spec 03's `createSubaccount()` and the SP-1 backfill SQL. For clients provisioned before that, run:

```sql
UPDATE comms_provisioning cp SET inbound_email_address = c.slug || '@in.seekly.app'
FROM clients c WHERE c.id = cp.client_id AND cp.inbound_email_address IS NULL;
```

**Acceptance criteria**

1. Migration applies cleanly; inserting two `inbound_emails` rows with the same non-null `messageId` violates the unique index; two rows with null `messageId` coexist.
2. Every active client has a non-null `inbound_email_address` equal to `slug || '@in.seekly.app'`.
3. `ops_review_queue` accepts rows of kind `email_parse_low_confidence` with `ref.inbound_email_id`.

---

## EP-3 — `POST /api/inbound/email` + `email-parse` queue

### Queue registration — `src/server/pipeline/queue.ts`

```ts
export const QUEUES = {
  // ...existing (incl. messageSend from spec 03)...
  emailParse: "email-parse",
} as const;

const RETRY_QUEUES = {
  // ...existing...
  // Anthropic 429/5xx retries; final failure → email-parse-dlq + row status "failed".
  emailParse: { retryLimit: 3, retryDelay: 60, retryBackoff: true },
};

export interface EmailParseJob {
  inboundEmailId: string;
}
```

### Worker — `src/worker.ts`

```ts
import { runEmailParse } from "./server/ingestion/email-parse";
// inside main():
await boss.work<EmailParseJob>(QUEUES.emailParse, async ([job]) => {
  await runEmailParse(job.data.inboundEmailId);
});
```

### Route — `src/app/api/inbound/email/route.ts` (complete)

Design rules: **fast and dumb** — authenticate, bound size, store, enqueue, ack. All content problems return **200** (SendGrid retries non-2xx for days; retrying can't fix a bad email). Only a bad/missing key returns 401 and a missing env key returns 503.

```ts
import { timingSafeEqual } from "node:crypto";
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, commsProvisioning, inboundEmails } from "@/server/db/schema";
import { getBoss, QUEUES } from "@/server/pipeline/queue";
import { log, logError } from "@/lib/logger";

const MAX_BODY_BYTES = 10 * 1024 * 1024; // SendGrid posts up to 30 MB; we cap at 10
const MAX_TEXT_CHARS = 100_000;

function keyValid(url: string): boolean {
  const expected = process.env.EMAIL_INBOUND_WEBHOOK_KEY;
  if (!expected) return false;
  const got = new URL(url).searchParams.get("key") ?? "";
  const a = Buffer.from(got);
  const b = Buffer.from(expected);
  return a.length === b.length && timingSafeEqual(a, b);
}

/** "Golf Pro Shop <receipts@squareup.com>" → "receipts@squareup.com" */
function extractAddress(raw: string): string {
  const m = raw.match(/<([^>]+)>/);
  return (m ? m[1] : raw).trim().toLowerCase();
}

/** All @in.seekly.app recipients across the envelope + To/Cc headers. */
function seeklyRecipients(envelopeJson: string, toHeader: string, ccHeader: string): string[] {
  const out = new Set<string>();
  try {
    const env = JSON.parse(envelopeJson) as { to?: string[] };
    for (const a of env.to ?? []) out.add(a.trim().toLowerCase());
  } catch { /* envelope malformed — fall back to headers */ }
  for (const part of `${toHeader},${ccHeader}`.split(",")) {
    const addr = extractAddress(part);
    if (addr.includes("@")) out.add(addr);
  }
  return [...out].filter((a) => a.endsWith("@in.seekly.app"));
}

/** "brendas-golf+receipts@in.seekly.app" → "brendas-golf" (plus-addressing supported). */
function slugOf(address: string): string {
  return address.split("@")[0].split("+")[0];
}

function stripHtml(html: string): string {
  return html
    .replace(/<style[\s\S]*?<\/style>/gi, " ")
    .replace(/<script[\s\S]*?<\/script>/gi, " ")
    .replace(/<br\s*\/?>/gi, "\n")
    .replace(/<\/(p|div|tr|li|h[1-6])>/gi, "\n")
    .replace(/<[^>]+>/g, " ")
    .replace(/&nbsp;/g, " ")
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/[ \t]+/g, " ")
    .replace(/\n{3,}/g, "\n\n")
    .trim();
}

function messageIdOf(headersBlob: string): string | null {
  const m = headersBlob.match(/^Message-ID:\s*<([^>]+)>/im);
  return m ? m[1].slice(0, 500) : null;
}

export async function POST(request: Request) {
  if (!process.env.EMAIL_INBOUND_WEBHOOK_KEY) {
    return Response.json({ error: "inbound email not configured" }, { status: 503 });
  }
  if (!keyValid(request.url)) {
    return Response.json({ error: "unauthorized" }, { status: 401 });
  }
  const contentLength = parseInt(request.headers.get("content-length") ?? "0", 10);
  if (contentLength > MAX_BODY_BYTES) {
    log(`[inbound-email] dropped oversize post (${contentLength} bytes)`);
    return new Response("ok", { status: 200 }); // 200: a retry can't shrink it
  }

  let form: FormData;
  try {
    form = await request.formData();
  } catch (err) {
    logError("[inbound-email] unparseable multipart body:", err);
    return new Response("ok", { status: 200 });
  }
  const f = (name: string) => String(form.get(name) ?? "");

  const recipients = seeklyRecipients(f("envelope"), f("to"), f("cc"));
  const fromAddress = extractAddress(f("from"));
  const senderDomain = fromAddress.split("@")[1] ?? "unknown";
  const text = (f("text") || stripHtml(f("html"))).slice(0, MAX_TEXT_CHARS);
  const messageId = messageIdOf(f("headers"));

  if (recipients.length === 0 || !fromAddress.includes("@") || !text) {
    log(`[inbound-email] ignored: recipients=${recipients.length} from=${fromAddress}`);
    return new Response("ok", { status: 200 });
  }

  // Resolve client by exact provisioned address first, then by slug (covers
  // plus-addressing: brendas-golf+forms@ still routes to brendas-golf).
  const address = recipients[0];
  let clientId: string | null = null;
  const [byAddress] = await db
    .select({ clientId: commsProvisioning.clientId })
    .from(commsProvisioning)
    .where(eq(commsProvisioning.inboundEmailAddress, address));
  if (byAddress) {
    clientId = byAddress.clientId;
  } else {
    const [bySlug] = await db
      .select({ id: clients.id })
      .from(clients)
      .where(eq(clients.slug, slugOf(address)));
    clientId = bySlug?.id ?? null;
  }

  const [row] = await db
    .insert(inboundEmails)
    .values({
      clientId,
      toAddress: address,
      fromAddress,
      senderDomain,
      subject: f("subject").slice(0, 500),
      textBody: text,
      messageId,
      spf: f("SPF") || null,
      dkim: f("dkim") || null,
      spamScore: f("spam_score") || null,
      status: clientId ? "received" : "ignored",
    })
    .onConflictDoNothing({ target: inboundEmails.messageId })
    .returning({ id: inboundEmails.id });

  if (!row) {
    log(`[inbound-email] duplicate Message-ID ${messageId} — already stored`);
    return new Response("ok", { status: 200 });
  }
  if (!clientId) {
    log(`[inbound-email] unknown recipient ${address} — stored as ignored`);
    return new Response("ok", { status: 200 });
  }

  const boss = await getBoss();
  await boss.send(QUEUES.emailParse, { inboundEmailId: row.id }, { singletonKey: row.id });
  log(`[inbound-email] ${row.id} queued for parse (client=${clientId}, from=${senderDomain})`);
  return new Response("ok", { status: 200 });
}
```

**Sender verification policy** (consumed in EP-5, stored here): the `spf`/`dkim`/`spamScore` fields are recorded verbatim. Emails are *not* rejected on SPF/DKIM failure at the route (forwarded mail routinely breaks SPF — the golf pilot literally forwards receipts). Instead, EP-5 treats an email whose SPF is `fail`/`softfail` **and** whose sender domain has no verified extraction examples as automatically low-confidence (→ review queue), so a spoofed email can never directly trigger a customer-facing event.

**Acceptance criteria**

1. POST without `?key=` or with a wrong key → 401, no DB row. Correct key → 200.
2. `Content-Length` > 10 MB → 200, no DB row, log line.
3. A well-formed SendGrid payload for `brendas-golf@in.seekly.app` creates one `inbound_emails` row (status `received`) and one `email-parse` pg-boss job; replaying the identical payload (same Message-ID) creates nothing new.
4. `brendas-golf+forms@in.seekly.app` resolves to the same client (plus-addressing).
5. Unknown slug → row with `clientId=null`, status `ignored`, no job, 200.
6. HTML-only email (no `text` field) is stored with the stripped-HTML body.

---

## EP-4 — Claude Haiku extraction module

`src/server/ingestion/email-extract.ts`. Follows `src/server/analysis/review-insights.ts` exactly: dynamic SDK import, `claude-haiku-4-5`, 30 s timeout / 1 retry, `output_config` JSON schema, zod `safeParse` of the output, cost via `anthropicUsageCost`, and a deterministic mock so dev/tests run free (`ENGINE_MODE=mock`).

```ts
/**
 * AI extraction for email-parse ingestion (adapter ladder rung 3). Turns a raw
 * POS/form notification email into a classified, confidence-scored candidate
 * for a canonical sale.completed or lead.created event. Model: claude-haiku-4-5
 * (cheapest tier — this runs on every inbound email). Extraction NEVER invents
 * contact data; the prompt + downstream guardrails (EP-5) enforce that a
 * low-confidence parse cannot reach a customer.
 */
import { z } from "zod";
import { logError } from "@/lib/logger";

export const EXTRACTION_MODEL = "claude-haiku-4-5";
export const MAX_EMAIL_CHARS = 12_000;

// --- Output contract -------------------------------------------------------

export const ExtractionResult = z.object({
  classification: z.enum(["sale_receipt", "lead_notification", "other"]),
  confidence: z.number().min(0).max(1),
  sale: z
    .object({
      customer: z.object({
        name: z.string(),
        phone: z.string().nullable(),
        email: z.string().nullable(),
      }),
      amount: z.number().nullable(),
      occurredAt: z.string().nullable(), // ISO-8601 or null
      externalId: z.string().nullable(), // POS order/booking id
      items: z.array(z.string()).nullable(),
    })
    .nullable(),
  lead: z
    .object({
      contact: z.object({
        name: z.string().nullable(),
        phone: z.string().nullable(),
        email: z.string().nullable(),
      }),
      message: z.string().nullable(),       // the free-text inquiry
      service: z.string().nullable(),       // what they asked about
      requestedDate: z.string().nullable(), // ISO-8601 or null
      partySize: z.number().nullable(),
      formSource: z.string().nullable(),    // e.g. "contact form", "quote request"
    })
    .nullable(),
  notes: z.string(), // one-line model rationale, surfaced in the review UI
});
export type ExtractionResult = z.infer<typeof ExtractionResult>;

// JSON schema handed to the API (mirror of the zod schema; additionalProperties
// false everywhere, all fields required — nullability expresses optionality).
const EXTRACTION_SCHEMA = {
  type: "object",
  properties: {
    classification: { type: "string", enum: ["sale_receipt", "lead_notification", "other"] },
    confidence: { type: "number" },
    sale: {
      type: ["object", "null"],
      properties: {
        customer: {
          type: "object",
          properties: {
            name: { type: "string" },
            phone: { type: ["string", "null"] },
            email: { type: ["string", "null"] },
          },
          required: ["name", "phone", "email"],
          additionalProperties: false,
        },
        amount: { type: ["number", "null"] },
        occurredAt: { type: ["string", "null"] },
        externalId: { type: ["string", "null"] },
        items: { type: ["array", "null"], items: { type: "string" } },
      },
      required: ["customer", "amount", "occurredAt", "externalId", "items"],
      additionalProperties: false,
    },
    lead: {
      type: ["object", "null"],
      properties: {
        contact: {
          type: "object",
          properties: {
            name: { type: ["string", "null"] },
            phone: { type: ["string", "null"] },
            email: { type: ["string", "null"] },
          },
          required: ["name", "phone", "email"],
          additionalProperties: false,
        },
        message: { type: ["string", "null"] },
        service: { type: ["string", "null"] },
        requestedDate: { type: ["string", "null"] },
        partySize: { type: ["number", "null"] },
        formSource: { type: ["string", "null"] },
      },
      required: ["contact", "message", "service", "requestedDate", "partySize", "formSource"],
      additionalProperties: false,
    },
    notes: { type: "string" },
  },
  required: ["classification", "confidence", "sale", "lead", "notes"],
  additionalProperties: false,
} as const;

// --- Prompt ----------------------------------------------------------------

export interface FewShotExample {
  emailExcerpt: string;
  extraction: unknown; // verified ExtractionResult
}

export function buildExtractionPrompt(input: {
  businessName: string;
  industry?: string | null;
  from: string;
  subject: string;
  textBody: string;
  examples: FewShotExample[];
}): string {
  const fewShot =
    input.examples.length === 0
      ? ""
      : `\nPreviously VERIFIED extractions from this same sender. Follow their field\nmapping exactly — the sender uses a fixed template:\n\n${input.examples
          .map(
            (ex, i) =>
              `### Verified example ${i + 1}\nEmail excerpt:\n"""\n${ex.emailExcerpt}\n"""\nCorrect extraction:\n${JSON.stringify(ex.extraction)}\n`,
          )
          .join("\n")}`;

  return `You are extracting structured data from a notification email received by \
"${input.businessName}"${input.industry ? ` (a ${input.industry} business)` : ""}, \
a local business whose growth automation is managed by Seekly.

Classify the email as exactly one of:
- "sale_receipt": a COMPLETED sale, booking confirmation, or payment receipt for a specific customer visit.
- "lead_notification": a NEW inquiry — website form submission, quote request, booking REQUEST (not yet paid/confirmed), or a message from a prospective customer.
- "other": anything else — newsletters, invoices addressed TO the business, shipping notices, system alerts, marketing, spam.

Then extract the fields. Hard rules:
- Extract ONLY what is literally present in the email. NEVER invent, infer, or autocomplete names, phone numbers, email addresses, amounts, or dates. A wrong phone number would cause a text message to a stranger.
- Phone numbers: copy digits exactly as written (formatting may be kept); null if absent.
- amount: the numeric total (e.g. 125.50), digits only, no currency symbol; null if absent.
- occurredAt / requestedDate: ISO-8601 only when the email states an unambiguous date (assume the business's locale for day/month order only if the format is unambiguous); otherwise null.
- Populate "sale" only for sale_receipt, "lead" only for lead_notification; the other MUST be null. For "other", both are null.
- notes: one sentence explaining your classification.

confidence is your calibrated probability that the classification AND every extracted field are correct:
- 0.90–1.00: unambiguous machine-generated template; every field copied verbatim.
- 0.80–0.90: minor ambiguity (name casing, date format) but classification certain.
- below 0.80: free-form / human-written email, any required field guessed, or classification uncertain.
- A sale_receipt where the customer has NO phone AND NO email must never exceed 0.70.
- If the email looks forwarded or quoted multiple times, reduce confidence one band.
${fewShot}
Email to extract:
From: ${input.from}
Subject: ${input.subject}
"""
${input.textBody.slice(0, MAX_EMAIL_CHARS)}
"""`;
}

// --- Live / mock execution --------------------------------------------------

function liveEnabled(): boolean {
  return process.env.ENGINE_MODE === "live" && Boolean(process.env.ANTHROPIC_API_KEY);
}

/** Deterministic mock: classification by keyword, everything else null-ish. */
export function mockExtraction(textBody: string): ExtractionResult {
  const lower = textBody.toLowerCase();
  const isSale = /receipt|payment|order #|booking confirmed|total/.test(lower);
  const isLead = /form submission|new inquiry|contact form|quote request/.test(lower);
  const phone = textBody.match(/\+?1?[\s.(-]*\d{3}[\s.)-]*\d{3}[\s.-]*\d{4}/)?.[0] ?? null;
  const email = textBody.match(/[\w.+-]+@[\w-]+\.[\w.]+/)?.[0] ?? null;
  if (isSale) {
    return {
      classification: "sale_receipt",
      confidence: phone || email ? 0.95 : 0.5,
      sale: { customer: { name: "Mock Customer", phone, email }, amount: null, occurredAt: null, externalId: null, items: null },
      lead: null,
      notes: "mock",
    };
  }
  if (isLead) {
    return {
      classification: "lead_notification",
      confidence: phone || email ? 0.9 : 0.5,
      lead: { contact: { name: null, phone, email }, message: textBody.slice(0, 200), service: null, requestedDate: null, partySize: null, formSource: "mock" },
      sale: null,
      notes: "mock",
    };
  }
  return { classification: "other", confidence: 0.95, sale: null, lead: null, notes: "mock" };
}

export async function extractFromEmail(input: {
  businessName: string;
  industry?: string | null;
  from: string;
  subject: string;
  textBody: string;
  examples: FewShotExample[];
}): Promise<{ result: ExtractionResult; costUsd: number } | null> {
  if (!liveEnabled()) return { result: mockExtraction(input.textBody), costUsd: 0 };

  const { default: Anthropic } = await import("@anthropic-ai/sdk");
  const { anthropicUsageCost } = await import("@/server/engines/pricing");
  const client = new Anthropic({ timeout: 30_000, maxRetries: 1 });

  const response = await client.messages.create({
    model: EXTRACTION_MODEL,
    max_tokens: 1500,
    output_config: { format: { type: "json_schema", schema: EXTRACTION_SCHEMA } },
    messages: [{ role: "user", content: buildExtractionPrompt(input) }],
  });

  const costUsd = anthropicUsageCost(EXTRACTION_MODEL, response.usage);
  const textBlock = response.content.find((b) => b.type === "text");
  if (!textBlock || textBlock.type !== "text") return null;
  const parsed = ExtractionResult.safeParse(JSON.parse(textBlock.text));
  if (!parsed.success) {
    logError("[email-extract] output failed schema validation:", parsed.error.issues);
    return null; // caller treats as extraction failure → review, never an event
  }
  return { result: parsed.data, costUsd };
}
```

**Acceptance criteria**

1. `ExtractionResult.safeParse` rejects: missing `confidence`, `sale` present alongside `lead`-shaped junk fields, confidence > 1.
2. With `ENGINE_MODE=mock`, a Square-style receipt fixture returns `sale_receipt` deterministically (byte-identical across runs).
3. Live mode (manual check with a real key): the pilot's actual POS sample email (intake Q16) returns `sale_receipt` with the customer's real name/phone and confidence ≥ 0.8; a hand-written personal email returns `other` or confidence < 0.8.
4. `buildExtractionPrompt` with 2 examples renders both excerpts and both verified JSON blobs; with 0 examples the few-shot block is absent.
5. The unit test asserts the prompt contains the "NEVER invent" rule and the ≤0.70 no-contact cap (guards against prompt regressions).

---

## EP-5 — Confidence-threshold flow, canonical events, ops review

### Worker logic — `src/server/ingestion/email-parse.ts`

```ts
/**
 * email-parse job: inbound_emails row → extraction → canonical event OR ops
 * review. THE GUARDRAIL: only extractFromEmail results with confidence ≥
 * threshold, a usable contact, and a trusted sender path may emit events —
 * everything else goes to a human. A low-confidence parse must never cause a
 * customer-facing message (doc 03).
 */
import { and, desc, eq } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, emailExtractionExamples, inboundEmails, opsReviewQueue } from "@/server/db/schema";
import { emitCanonicalEvent } from "@/server/events/emit"; // spec 02
import { toE164 } from "@/server/messaging/compliance";     // spec 03
import { log, logError } from "@/lib/logger";
import { extractFromEmail, type ExtractionResult } from "./email-extract";
import { getExamples, maybeStoreAutoExample } from "./examples"; // EP-6

export const DEFAULT_MIN_CONFIDENCE = 0.8;
// Per-client override lives in workflow_config settings (spec 01):
// { module: "email_parse", settings: { minConfidence: number } }. Until that
// table lands, the default is compiled in.

export async function runEmailParse(inboundEmailId: string): Promise<void> {
  const [email] = await db.select().from(inboundEmails).where(eq(inboundEmails.id, inboundEmailId));
  if (!email || email.status !== "received" || !email.clientId) return; // idempotent re-run guard

  const [client] = await db
    .select({ name: clients.name })
    .from(clients)
    .where(eq(clients.id, email.clientId));

  let extracted;
  try {
    extracted = await extractFromEmail({
      businessName: client.name,
      from: email.fromAddress,
      subject: email.subject,
      textBody: email.textBody,
      examples: await getExamples(email.clientId, email.senderDomain),
    });
  } catch (err) {
    logError(`[email-parse] extraction crashed for ${inboundEmailId}:`, err);
    throw err; // let pg-boss retry; final failure → dlq + the sweep below marks "failed"
  }

  if (!extracted) {
    await sendToReview(email, null, 0, "extraction returned no valid result");
    return;
  }
  const { result, costUsd } = extracted;

  await db.update(inboundEmails)
    .set({ extraction: result, confidence: result.confidence, costUsd: String(costUsd) })
    .where(eq(inboundEmails.id, inboundEmailId));

  if (result.classification === "other") {
    await db.update(inboundEmails).set({ status: "ignored" }).where(eq(inboundEmails.id, inboundEmailId));
    log(`[email-parse] ${inboundEmailId} classified other — ignored`);
    return;
  }

  // Trust gate (EP-3 policy): a failing-SPF sender with no verified example
  // history is never auto-emitted, regardless of model confidence.
  const spfFailed = /fail/i.test(email.spf ?? "");
  const hasVerifiedHistory = (await getExamples(email.clientId, email.senderDomain)).length > 0;
  const trusted = !spfFailed || hasVerifiedHistory;

  const contact =
    result.classification === "sale_receipt" ? result.sale?.customer : result.lead?.contact;
  const hasContact = Boolean(contact && (contact.phone || contact.email));

  if (result.confidence >= DEFAULT_MIN_CONFIDENCE && hasContact && trusted) {
    await emitFromExtraction(email.id, email.clientId, result);
    await db.update(inboundEmails)
      .set({ status: "parsed", eventId: email.id })
      .where(eq(inboundEmails.id, inboundEmailId));
    await maybeStoreAutoExample(email.clientId, email.senderDomain, email.textBody, result);
    log(`[email-parse] ${inboundEmailId} → ${result.classification} event (conf=${result.confidence})`);
    return;
  }

  await sendToReview(
    email, result, result.confidence,
    !trusted ? "spf failed, unverified sender" : !hasContact ? "no phone or email extracted" : "below confidence threshold",
  );
}

/**
 * Build + emit the canonical event (doc 02 envelope). event_id = the
 * inbound_emails row id: replays and ops re-approvals dedupe on events.id.
 */
export async function emitFromExtraction(
  eventId: string,
  clientId: string,
  result: ExtractionResult,
): Promise<void> {
  if (result.classification === "sale_receipt" && result.sale) {
    const s = result.sale;
    await emitCanonicalEvent({
      eventId,
      clientId,
      type: "sale.completed",
      occurredAt: s.occurredAt ?? new Date().toISOString(),
      source: "email-parse",
      payload: {
        customer: {
          name: s.customer.name,
          phone: s.customer.phone ? toE164(s.customer.phone) : null,
          email: s.customer.email,
        },
        amount: s.amount,
        items: s.items,
        external_id: s.externalId,
        raw: { inbound_email_id: eventId },
      },
    });
    return;
  }
  if (result.classification === "lead_notification" && result.lead) {
    const l = result.lead;
    await emitCanonicalEvent({
      eventId,
      clientId,
      type: "lead.created",
      occurredAt: new Date().toISOString(),
      source: "email-parse",
      payload: {
        contact: {
          name: l.contact.name,
          phone: l.contact.phone ? toE164(l.contact.phone) : null,
          email: l.contact.email,
        },
        message: l.message,
        service: l.service,
        requested_date: l.requestedDate,
        party_size: l.partySize,
        form_source: l.formSource ?? "email-parse",
        raw: { inbound_email_id: eventId },
      },
    });
  }
}

async function sendToReview(
  email: typeof inboundEmails.$inferSelect,
  result: ExtractionResult | null,
  confidence: number,
  reason: string,
): Promise<void> {
  await db.update(inboundEmails).set({ status: "review" }).where(eq(inboundEmails.id, email.id));
  await db.insert(opsReviewQueue).values({
    clientId: email.clientId!,
    kind: "email_parse_low_confidence",
    ref: { inbound_email_id: email.id },
    payload: {
      reason,
      confidence,
      from: email.fromAddress,
      subject: email.subject,
      text_preview: email.textBody.slice(0, 2000),
      extraction: result,
    },
  });
  log(`[email-parse] ${email.id} → ops review (${reason})`);
}
```

> **`emitCanonicalEvent` contract (spec 02):** `{ eventId, clientId, type, occurredAt, source, payload }` → idempotent insert into `events` (PK = eventId; conflict = no-op) then forward to the type's n8n webhook. If spec 02 hasn't merged when you build this, implement `src/server/events/emit.ts` as insert-plus-`log()` with the same signature — the call sites here must not change. Downstream (n8n WF-1/WF-2 or the spec-02 event processor) is responsible for upserting `customers` from `sale.completed` — this module never writes `customers` directly.

### Admin review UI (note + server actions — UI ticket references doc 05 "Ops queues")

The `(app)` admin area gets an **Ops queue** page (follow the existing admin table/card patterns — shadcn/radix + the `(app)` layout; no new design language). Contract for the page:

- List `ops_review_queue` rows with `status='open'`, `kind='email_parse_low_confidence'`, newest first, grouped by client. Each row shows: from, subject, reason, confidence, `text_preview` (collapsible), and the proposed `extraction` JSON in an editable `<textarea>`.
- Buttons: **Approve** (uses the possibly-edited JSON), **Discard**.
- Both call the server actions below; the page revalidates.

`src/server/ingestion/review-actions.ts`:

```ts
"use server";
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { inboundEmails, opsReviewQueue } from "@/server/db/schema";
import { requireSession } from "@/server/auth/session"; // admin session
import { ExtractionResult } from "./email-extract";
import { emitFromExtraction } from "./email-parse";
import { storeOpsExample } from "./examples"; // EP-6

export async function approveEmailParse(opsRowId: string, editedExtractionJson: string) {
  const session = await requireSession();
  const [row] = await db.select().from(opsReviewQueue).where(eq(opsReviewQueue.id, opsRowId));
  if (!row || row.status !== "open") throw new Error("review row not open");

  const parsed = ExtractionResult.safeParse(JSON.parse(editedExtractionJson));
  if (!parsed.success) throw new Error(`invalid extraction: ${parsed.error.issues[0]?.message}`);
  if (parsed.data.classification === "other") throw new Error("cannot approve 'other' — discard instead");

  const inboundEmailId = (row.ref as { inbound_email_id: string }).inbound_email_id;
  const [email] = await db.select().from(inboundEmails).where(eq(inboundEmails.id, inboundEmailId));

  // event_id = inbound email id → double-approve is a no-op at the events table.
  await emitFromExtraction(inboundEmailId, row.clientId, parsed.data);
  await db.update(inboundEmails)
    .set({ status: "parsed", eventId: inboundEmailId, extraction: parsed.data })
    .where(eq(inboundEmails.id, inboundEmailId));
  await storeOpsExample(row.clientId, email.senderDomain, email.textBody, parsed.data);
  await db.update(opsReviewQueue)
    .set({ status: "approved", resolution: parsed.data, resolvedAt: new Date(), resolvedBy: session.email })
    .where(eq(opsReviewQueue.id, opsRowId));
}

export async function discardEmailParse(opsRowId: string) {
  const session = await requireSession();
  const [row] = await db.select().from(opsReviewQueue).where(eq(opsReviewQueue.id, opsRowId));
  if (!row || row.status !== "open") return;
  const inboundEmailId = (row.ref as { inbound_email_id: string }).inbound_email_id;
  await db.update(inboundEmails).set({ status: "ignored" }).where(eq(inboundEmails.id, inboundEmailId));
  await db.update(opsReviewQueue)
    .set({ status: "discarded", resolvedAt: new Date(), resolvedBy: session.email })
    .where(eq(opsReviewQueue.id, opsRowId));
}
```

**Acceptance criteria**

1. Confidence 0.85 with a phone → one `events` row (`type='sale.completed'`, `id` = inbound email id, `source='email-parse'`, payload phone in E.164); `inbound_emails.status='parsed'`.
2. Confidence 0.6 → NO events row; `ops_review_queue` row with reason "below confidence threshold"; `inbound_emails.status='review'`.
3. Confidence 0.95 sale with `phone=null` AND `email=null` → review, never an event (belt on top of the prompt's 0.70 cap).
4. SPF `fail` + zero examples for the domain → review even at confidence 0.99; same email after one ops approval for the domain → auto-emits.
5. `approveEmailParse` with edited JSON emits the event, stores an `ops` example, closes the review row; approving twice throws (row not open) and cannot double-emit (events PK).
6. `discardEmailParse` marks the email `ignored` and emits nothing.
7. Running `runEmailParse` twice for the same row (worker retry after success) is a no-op the second time (status guard).

---

## EP-6 — Per-client template learning

`src/server/ingestion/examples.ts`:

```ts
/**
 * Few-shot memory for the email parser. Keyed by (clientId, senderDomain);
 * NEVER shared across clients (excerpts contain customer PII). Ops-verified
 * examples outrank auto-captured ones; at most EXAMPLES_PER_PROMPT are
 * injected per parse, at most MAX_STORED kept per key (oldest auto pruned).
 */
import { and, asc, desc, eq } from "drizzle-orm";
import { db } from "@/server/db";
import { emailExtractionExamples } from "@/server/db/schema";
import type { ExtractionResult } from "./email-extract";
import type { FewShotExample } from "./email-extract";

export const EXAMPLES_PER_PROMPT = 3;
export const MAX_STORED = 5;
export const EXCERPT_CHARS = 2000;

export async function getExamples(clientId: string, senderDomain: string): Promise<FewShotExample[]> {
  const rows = await db
    .select({ emailExcerpt: emailExtractionExamples.emailExcerpt, extraction: emailExtractionExamples.extraction, source: emailExtractionExamples.source })
    .from(emailExtractionExamples)
    .where(and(eq(emailExtractionExamples.clientId, clientId), eq(emailExtractionExamples.senderDomain, senderDomain)))
    .orderBy(
      // "ops" before "auto", then newest first.
      asc(emailExtractionExamples.source), // "auto" < "ops" alphabetically → use sql CASE if reversed
      desc(emailExtractionExamples.createdAt),
    );
  // Explicit re-sort in JS (alphabetical trick above is fragile — keep it obvious):
  rows.sort((a, b) => (a.source === b.source ? 0 : a.source === "ops" ? -1 : 1));
  return rows.slice(0, EXAMPLES_PER_PROMPT).map((r) => ({ emailExcerpt: r.emailExcerpt, extraction: r.extraction }));
}

async function prune(clientId: string, senderDomain: string): Promise<void> {
  const rows = await db
    .select({ id: emailExtractionExamples.id, source: emailExtractionExamples.source })
    .from(emailExtractionExamples)
    .where(and(eq(emailExtractionExamples.clientId, clientId), eq(emailExtractionExamples.senderDomain, senderDomain)))
    .orderBy(desc(emailExtractionExamples.createdAt));
  const keep = new Set<string>();
  // Keep newest MAX_STORED, preferring ops rows.
  for (const r of [...rows].sort((a, b) => (a.source === b.source ? 0 : a.source === "ops" ? -1 : 1))) {
    if (keep.size < MAX_STORED) keep.add(r.id);
  }
  for (const r of rows) {
    if (!keep.has(r.id)) {
      await db.delete(emailExtractionExamples).where(eq(emailExtractionExamples.id, r.id));
    }
  }
}

/** Ops-verified example (from approveEmailParse). Always stored. */
export async function storeOpsExample(
  clientId: string, senderDomain: string, textBody: string, extraction: ExtractionResult,
): Promise<void> {
  await db.insert(emailExtractionExamples).values({
    clientId, senderDomain,
    emailExcerpt: textBody.slice(0, EXCERPT_CHARS),
    extraction, source: "ops",
  });
  await prune(clientId, senderDomain);
}

/** Auto example: only the FIRST high-confidence parse for a bare domain. */
export async function maybeStoreAutoExample(
  clientId: string, senderDomain: string, textBody: string, extraction: ExtractionResult,
): Promise<void> {
  const existing = await db
    .select({ id: emailExtractionExamples.id })
    .from(emailExtractionExamples)
    .where(and(eq(emailExtractionExamples.clientId, clientId), eq(emailExtractionExamples.senderDomain, senderDomain)))
    .limit(1);
  if (existing.length > 0) return;
  await db.insert(emailExtractionExamples).values({
    clientId, senderDomain,
    emailExcerpt: textBody.slice(0, EXCERPT_CHARS),
    extraction, source: "auto",
  });
}
```

**Acceptance criteria**

1. `getExamples` returns at most 3, ops rows first, per (client, domain) only — a second client's examples for the same domain never appear.
2. `maybeStoreAutoExample` stores once per domain and never overwrites; `storeOpsExample` always stores and prunes to ≤ 5 rows keeping ops rows preferentially.
3. After one `storeOpsExample`, the next `buildExtractionPrompt` for that domain includes the verified extraction (integration test through `getExamples`).

---

## EP-7 — CSV import (adapter ladder rung 4)

Portal upload → column mapping → `customers` upsert with `consent_basis='imported_list'` (the WF-3 reactivation audience; doc 02: "reactivation sends only to customers with a recorded prior relationship").

### `customers` table (spec 01 — reproduced; reconcile if already landed)

```ts
export const consentBasisEnum = pgEnum("consent_basis", [
  "existing_customer",
  "lead_inbound",
  "imported_list",
]);

export const customers = pgTable(
  "customers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull().references(() => clients.id, { onDelete: "cascade" }),
    name: text("name"),
    phone: text("phone").notNull(), // E.164
    email: text("email"),
    firstSeenAt: timestamp("first_seen_at", { withTimezone: true }),
    lastVisitAt: timestamp("last_visit_at", { withTimezone: true }),
    visitCount: integer("visit_count").notNull().default(0),
    consentBasis: consentBasisEnum("consent_basis").notNull(),
    source: text("source").notNull(), // "csv" | "email-parse" | "webhook" | ...
    tags: jsonb("tags").$type<string[]>(),
    externalIds: jsonb("external_ids"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("customers_client_phone_unique").on(t.clientId, t.phone)],
);
```

### Column-mapping UI contract (portal page `/portal/customers/import`)

Two-step flow; the portal UI ticket implements the React, this spec fixes the wire contract:

1. **Preview.** User picks a `.csv` file → `POST /api/portal/customers/import/preview` (multipart, field `file`). Response:
   ```json
   {
     "columns": ["First Name", "Cell", "Email", "Last Visit"],
     "sample": [["Dana", "613-555-0142", "dana@x.com", "2025-11-02"], ...],  // ≤ 5 rows
     "rowCountEstimate": 1240
   }
   ```
2. **Map.** The UI renders one `<select>` per target field over `columns`. Target fields: `phone` (required), `name`, `email`, `lastVisitAt`, `firstSeenAt` (each optional), plus `dateFormat` (`"ISO" | "MDY" | "DMY"`, default `ISO`). The UI must block submit until `phone` is mapped and show the consent attestation checkbox: *"I confirm these are existing customers of my business who provided their phone numbers"* (maps to nothing server-side yet; it is the imported-list consent basis being attested — keep the copy).
3. **Commit.** `POST /api/portal/customers/import` (multipart: `file` again + `mapping` as a JSON string field):
   ```json
   { "phone": "Cell", "name": "First Name", "email": "Email", "lastVisitAt": "Last Visit", "dateFormat": "MDY" }
   ```
   Response:
   ```json
   { "imported": 1103, "updated": 84, "skipped": 53,
     "skippedDetails": [{ "row": 7, "reason": "invalid phone: '555-0142'" }, ...] }  // capped at 100
   ```

### Route handlers

`src/app/api/portal/customers/import/preview/route.ts`:

```ts
import { parse } from "csv-parse/sync";
import { requireClientSession } from "@/server/auth/session";

const MAX_CSV_BYTES = 5 * 1024 * 1024;

export async function POST(request: Request) {
  const session = await requireClientSession(); // throws → 401 via error boundary; portal-scoped
  const form = await request.formData();
  const file = form.get("file");
  if (!(file instanceof File)) return Response.json({ error: "file required" }, { status: 400 });
  if (file.size > MAX_CSV_BYTES) return Response.json({ error: "CSV too large (max 5 MB)" }, { status: 413 });

  const text = await file.text();
  let records: string[][];
  try {
    records = parse(text, { to: 6, relax_column_count: true }) as string[][];
  } catch {
    return Response.json({ error: "not a parseable CSV" }, { status: 400 });
  }
  if (records.length < 2) return Response.json({ error: "CSV needs a header row and at least one data row" }, { status: 400 });
  return Response.json({
    columns: records[0],
    sample: records.slice(1, 6),
    rowCountEstimate: Math.max(0, text.split("\n").length - 1),
  });
}
```

`src/app/api/portal/customers/import/route.ts`:

```ts
import { z } from "zod";
import { parse } from "csv-parse/sync";
import { sql } from "drizzle-orm";
import { db } from "@/server/db";
import { customers } from "@/server/db/schema";
import { requireClientSession } from "@/server/auth/session";
import { toE164 } from "@/server/messaging/compliance";
import { log } from "@/lib/logger";

const MAX_CSV_BYTES = 5 * 1024 * 1024;
const MAX_ROWS = 20_000;

const Mapping = z.object({
  phone: z.string().min(1),
  name: z.string().optional(),
  email: z.string().optional(),
  lastVisitAt: z.string().optional(),
  firstSeenAt: z.string().optional(),
  dateFormat: z.enum(["ISO", "MDY", "DMY"]).default("ISO"),
});

function parseDate(value: string, format: "ISO" | "MDY" | "DMY"): Date | null {
  const v = value.trim();
  if (!v) return null;
  if (format !== "ISO") {
    const m = v.match(/^(\d{1,2})[\/\-.](\d{1,2})[\/\-.](\d{2,4})$/);
    if (m) {
      const [a, b] = format === "MDY" ? [m[1], m[2]] : [m[2], m[1]];
      const year = m[3].length === 2 ? `20${m[3]}` : m[3];
      const d = new Date(`${year}-${a.padStart(2, "0")}-${b.padStart(2, "0")}T00:00:00Z`);
      return isNaN(d.getTime()) ? null : d;
    }
  }
  const d = new Date(v);
  return isNaN(d.getTime()) ? null : d;
}

export async function POST(request: Request) {
  const session = await requireClientSession();
  const clientId = session.clientId;

  const form = await request.formData();
  const file = form.get("file");
  if (!(file instanceof File)) return Response.json({ error: "file required" }, { status: 400 });
  if (file.size > MAX_CSV_BYTES) return Response.json({ error: "CSV too large (max 5 MB)" }, { status: 413 });
  const mappingParsed = Mapping.safeParse(JSON.parse(String(form.get("mapping") ?? "{}")));
  if (!mappingParsed.success) return Response.json({ error: mappingParsed.error.issues }, { status: 400 });
  const mapping = mappingParsed.data;

  const rows = parse(await file.text(), { columns: true, relax_column_count: true, skip_empty_lines: true }) as Record<string, string>[];
  if (rows.length > MAX_ROWS) return Response.json({ error: `too many rows (max ${MAX_ROWS})` }, { status: 413 });

  let imported = 0, updated = 0;
  const skippedDetails: { row: number; reason: string }[] = [];

  for (let i = 0; i < rows.length; i++) {
    const r = rows[i];
    const phone = toE164(r[mapping.phone] ?? "");
    if (!phone) {
      if (skippedDetails.length < 100) skippedDetails.push({ row: i + 2, reason: `invalid phone: '${r[mapping.phone] ?? ""}'` });
      continue;
    }
    const lastVisitAt = mapping.lastVisitAt ? parseDate(r[mapping.lastVisitAt] ?? "", mapping.dateFormat) : null;
    const firstSeenAt = mapping.firstSeenAt ? parseDate(r[mapping.firstSeenAt] ?? "", mapping.dateFormat) : null;

    const [result] = await db
      .insert(customers)
      .values({
        clientId,
        phone,
        name: mapping.name ? r[mapping.name]?.trim() || null : null,
        email: mapping.email ? r[mapping.email]?.trim().toLowerCase() || null : null,
        lastVisitAt,
        firstSeenAt,
        consentBasis: "imported_list",
        source: "csv",
      })
      .onConflictDoUpdate({
        target: [customers.clientId, customers.phone],
        set: {
          // Enrich, never destroy: COALESCE keeps existing values; consentBasis is
          // NOT touched on update (an existing_customer must never be downgraded
          // to imported_list).
          name: sql`coalesce(excluded.name, ${customers.name})`,
          email: sql`coalesce(excluded.email, ${customers.email})`,
          lastVisitAt: sql`greatest(coalesce(excluded.last_visit_at, ${customers.lastVisitAt}), coalesce(${customers.lastVisitAt}, excluded.last_visit_at))`,
          firstSeenAt: sql`least(coalesce(excluded.first_seen_at, ${customers.firstSeenAt}), coalesce(${customers.firstSeenAt}, excluded.first_seen_at))`,
        },
      })
      .returning({ inserted: sql<boolean>`(xmax = 0)` }); // xmax=0 ⇔ fresh insert
    if (result.inserted) imported++; else updated++;
  }

  log(`[csv-import] client=${clientId} imported=${imported} updated=${updated} skipped=${skippedDetails.length}`);
  // Activity log (spec 01 activity_log): one row per import — action "customers.csv_import",
  // detail { imported, updated, skipped }. Add when the table exists; log() suffices until then.
  return Response.json({ imported, updated, skipped: rows.length - imported - updated, skippedDetails });
}
```

**No per-row canonical events.** Doc 02 lists `customer.imported`, but 20k events for one upload is noise with no consumer — the `customers` table *is* the reactivation audience. Emit nothing per row; the activity-log entry records the batch. (If a future engine needs the event, add a single batch-level `customer.imported` event then.)

**Acceptance criteria**

1. A 3-row CSV with headers `First Name,Cell,Email,Last Visit` and mapping `{phone:"Cell",name:"First Name",email:"Email",lastVisitAt:"Last Visit",dateFormat:"MDY"}` creates 3 `customers` rows: E.164 phones, `consentBasis='imported_list'`, `source='csv'`, parsed dates.
2. Re-uploading the identical file → `imported=0, updated=3`, no duplicates (unique `(clientId, phone)`), and an existing row's `consentBasis='existing_customer'` is untouched.
3. Rows with unparseable phones are skipped and itemized (row number is the 1-based CSV line incl. header).
4. A customer that already has `lastVisitAt=2026-01-01` updated by a CSV row with `2025-06-01` keeps `2026-01-01` (GREATEST).
5. 6 MB file → 413; 25k-row file → 413; portal session for client A can never write rows for client B (clientId comes from the session, not the request).
6. Preview endpoint returns headers + ≤5 sample rows for a quoted-field CSV (`"Smith, Dana"` stays one cell — csv-parse, not `split(",")`).

---

## EP-8 — Test pack & pilot tuning

`src/server/ingestion/email-parse.test.ts` + fixtures in `src/server/ingestion/fixtures/` (three realistic emails: a Square-style receipt, a WordPress contact-form notification, a personal thank-you note). Everything runs under `ENGINE_MODE=mock` except the two explicitly-live checks.

| # | Test | Assertion |
|---|---|---|
| 1 | route auth + idempotency | wrong key 401; same Message-ID twice → one row, one job |
| 2 | threshold routing | conf ≥ 0.8 + phone → event; conf < 0.8 → review row; `other` → ignored |
| 3 | never-text-a-stranger guard | sale with no phone/email → review even at conf 0.95 |
| 4 | SPF trust gate | SPF fail + no examples → review; after ops approval → auto-emit |
| 5 | event idempotency | approve after auto-emit (or double approve) → single `events` row |
| 6 | few-shot learning | ops approval stores example; next prompt for the domain contains it |
| 7 | CSV round-trip | acceptance criteria 1–4 of EP-7 as tests |
| 8 | LIVE: pilot POS sample | the real forwarded sample from intake Q16 → `sale_receipt`, conf ≥ 0.8, correct name+phone (manual, gated behind `ENGINE_MODE=live`, run before flipping the pilot switch) |
| 9 | LIVE: adversarial email | a hand-written email claiming a fake sale from a random Gmail → lands in review, never an event |

**Pilot tuning procedure (runbook, part of this ticket):** forward the pilot's real POS notification sample to `{slug}@in.seekly.app` in staging → inspect the `inbound_emails` row + extraction → if conf < 0.8, approve via the ops queue (this seeds the ops example) → forward a second sample → verify auto-emit. Two emails end-to-end is the definition of "email-parse infra live" from doc 07 week 1.

**Acceptance criteria**

1. Tests 1–7 green in `npm test` (CI, mock mode).
2. Tests 8–9 executed and recorded in the ticket before the pilot's `review_automation`/`speed_to_lead` switches flip to live.
3. Total Claude spend per parsed email logged (`costUsd`) and < $0.01 at Haiku pricing for the pilot's templates.
