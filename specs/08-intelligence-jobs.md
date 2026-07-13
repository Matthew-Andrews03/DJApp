# Spec 08 — Intelligence Jobs (pg-boss, inside seekly-client-insights)

**Implements:** doc 09 (Intelligence Platform) — the L0–L4 memory hierarchy, anti-flooding
rules, experiment ledger, and WF-8 Growth Action Queue.
**Runs in:** the existing `seekly-client-insights` app as **pg-boss jobs in the worker
process** (`src/worker.ts`) — NOT n8n. Per doc 10: "data products in pg-boss,
outreach/integration automations in n8n."
**Depends on:** spec 01 (control-plane schema migrations: `activity_log`, `metrics_daily`,
`events`, `customers`, `messages` growth ledger, `conversations`, `campaigns`,
`campaign_members`, `content_items`, `syndicated_posts`, `client_profile`,
`workflow_config`). If spec 01's exported Drizzle names differ from the identifiers used
below (`activityLog`, `metricsDaily`, `conversations`, `campaigns`, `campaignMembers`,
`contentItems`, `syndicatedPosts`, `growthMessages`, `clientProfile`, `workflowConfig`),
adjust the imports — the SQL semantics do not change.

## 0. Purpose and non-negotiable constraints

The intelligence layer turns raw engine exhaust (L0) into compounding, auditable memory:

```
L4 playbooks (per niche, ~1k tok)  ← playbook-aggregate-monthly
L3 client brief (≤1.5k tok HARD)   ← brief-refresh-monthly (+ on-demand)
L2 insight notes (~200 tok each)   ← insight-writer-weekly (per client per domain)
L1 metrics_daily + anomalies       ← rollup-nightly, anomaly-detect-nightly (zero AI)
L0 activity_log / messages / conversations / campaigns / reviews  (jobs read; agents never)
```

**Anti-flooding rules (doc 09) — these are implementation constraints, enforce in code:**

1. **No job or agent ever loads raw L0 rows into an LLM context.** LLM inputs are: the
   brief (L3), retrieved insight notes (L2), and *pre-computed numbers* from L1. Rollup and
   anomaly jobs read L0 with SQL only.
2. **Every AI-written artifact has a token budget and a zod schema.** Notes ~200 tokens,
   brief ≤1.5k tokens hard cap (enforced via `usage.output_tokens`, retry-with-compress),
   playbooks ~1k tokens, action queue exactly 5 actions.
3. **Compression is scheduled:** nightly SQL → weekly notes → monthly brief. LLM cost scales
   with client count, never event count.
4. **Scoped retrieval by domain tag** (`reputation|leads|reactivation|content|visibility|competitive`),
   plus optional embedding search. Never "all notes."
5. **Numbers stay numeric.** All trends/z-scores/verdicts are computed in SQL/TS. The LLM
   narrates and prioritizes; it is never the calculator.

**Model policy.** Use the models already priced in
`src/server/engines/pricing.ts` → `ANTHROPIC_MODEL_PRICING` so `anthropicUsageCost()` works
unchanged: `claude-sonnet-4-6` for all four LLM jobs (insight writer, brief, action queue,
playbook — these are reasoning jobs, matching doc 07's "Sonnet default"). No LLM anywhere
else in this spec. Every LLM module MUST implement the repo's mock pattern
(`liveEnabled()` = `env.ENGINE_MODE === "live" && ANTHROPIC_API_KEY`, deterministic mock
otherwise) exactly as done in `src/server/analysis/review-insights.ts`.

**Where code lives.** New directory `src/server/intelligence/`:

```
src/server/intelligence/
  schema-notes.md        (none — tables go in src/server/db/schema.ts)
  scheduler.ts           coordinators: fan-out per-client jobs on cron ticks
  rollup.ts              INT-2
  anomalies.ts           INT-3
  embeddings.ts          INT-4
  retrieval.ts           INT-4 (getBriefForClient / getInsights)
  insight-writer.ts      INT-5
  brief.ts               INT-6
  experiments.ts         INT-7
  action-queue.ts        INT-8
  playbooks.ts           INT-9
  spend.ts               INT-1 (weekly budget guard)
  mock.ts                shared deterministic mocks (hashCode seed pattern)
```

---

## 1. Ticket table

Phase-in follows doc 09 "Build sequencing": L0/L1 exist by construction weeks 1–4 (spec 01);
intelligence starts **weeks 5–6**; WF-8 + experiments weeks 7–8 (approval-mode only at
first); playbooks month 3+.

| Ticket | Title | Depends on | Phase (doc 09) |
|---|---|---|---|
| INT-1 | Intelligence schema, queues, worker registration, budget guard | spec 01 migrations merged | Week 5 |
| INT-2 | `rollup-nightly` — metrics_daily per client per module | INT-1 | Week 5 |
| INT-3 | `anomaly-detect-nightly` — pure-stat detectors | INT-2 | Week 5 |
| INT-4 | Embeddings + retrieval helpers (`getBriefForClient`, `getInsights`) + n8n context enrichment | INT-1 | Weeks 5–6 |
| INT-5 | `insight-writer-weekly` — L2 notes per client per domain | INT-2, INT-3, INT-4 | Week 6 |
| INT-6 | `brief-refresh-monthly` + on-demand trigger — L3 brief, 1.5k hard cap | INT-5 | Week 6 (first pilot briefs) |
| INT-7 | Experiment ledger + `experiment-closer-daily` + confidence write-back | INT-2 | Week 7 |
| INT-8 | WF-8 `action-queue-weekly` — 5 ranked actions, safety triage, auto-exec + approval queue | INT-6, INT-7 | Weeks 7–8 (approval-mode only until 3 clean weekly cycles, then flip auto flag) |
| INT-9 | `playbook-aggregate-monthly` — per-niche L4 with privacy validator | INT-7; ≥2 clients in a niche with closed experiments | Month 3+ |
| INT-10 | Hardening: idempotency drills, budget-guard tests, ops digest wiring | INT-2..INT-8 | Week 8 |

---

## 2. Job registry

All queues are added to `QUEUES` in `src/server/pipeline/queue.ts` and get retry policies in
`RETRY_QUEUES` (+ auto-created `<name>-dlq` dead-letter queues — this happens for free in
`getBoss()` once the entry exists in `RETRY_QUEUES`). Coordinators are scheduled with
`boss.schedule(queue, cron)` exactly like `QUEUES.checkSchedules` in `src/worker.ts:102`.
Per-client jobs are fanned out by the coordinator with `boss.insert(...)` +
`singletonKey` = idempotency, mirroring `runClient()` in
`src/server/pipeline/run-client.ts:115-124` and the per-day singleton in
`src/server/reports/scheduler.ts:36-41`.

Cron times are UTC (03:10 ET ≈ 07:10 UTC in DST; the exact wall-clock hour is not
load-bearing, the ORDER is: rollup → experiment-closer → anomaly → insights → action queue).

| Job (queue name) | Schedule (cron, UTC) | Fan-out / concurrency | Cost guard | Reads | Writes |
|---|---|---|---|---|---|
| `intel-nightly` (coordinator) | `10 7 * * *` | 1 (enqueues per-client jobs) | — | `clients` | pg-boss jobs |
| `intel-rollup` (rollup-nightly) | via coordinator, per client | `batchSize: 4`; singleton `${clientId}:${date}` | none (pure SQL) | `activity_log`, `messages`, `conversations`, `campaigns`, `campaign_members`, `reviews`, `review_requests`, `content_items`, `syndicated_posts`, `daily_aggregates` | `metrics_daily`, `activity_log` |
| `intel-experiment-close` (experiment-closer-daily) | `0 8 * * *` (coordinator enqueues per due experiment) | `batchSize: 2`; singleton `${experimentId}` | none (pure SQL) | `experiments`, `metrics_daily`, `campaigns` | `experiments`, `insights`, `activity_log`; may enqueue `intel-brief` |
| `intel-anomaly` (anomaly-detect-nightly) | `30 8 * * *` (coordinator enqueues per client) | `batchSize: 4`; singleton `${clientId}:${date}` | none (pure SQL) | `metrics_daily`, `conversations`, `reviews`, `campaigns`, `daily_aggregates`, `anomalies` | `anomalies`, `activity_log` |
| `intel-insights` (insight-writer-weekly) | `30 9 * * 1` (coordinator enqueues per client × domain) | `batchSize: 2`; singleton `${clientId}:${domain}:${isoWeek}` | est. $0.02/call vs `clients.weeklyBudgetUsd` via `spend.ts` | `metrics_daily`, `anomalies`, `experiments`, `insights` | `insights` (+embeddings), `intelligence_costs`, `activity_log` |
| `intel-brief` (brief-refresh-monthly + on-demand) | `0 10 1 * *` (coordinator per client) + `enqueueBriefRefresh()` on major events | `batchSize: 2`; singleton `${clientId}:${yyyymm}` (on-demand: `${clientId}:evt:${reason}:${date}`) | est. $0.04/call vs weekly budget | `insights`, `experiments`, `playbooks`, `client_profile`, `client_briefs` | `client_briefs`, `intelligence_costs`, `activity_log` |
| `intel-action-queue` (action-queue-weekly, WF-8) | `30 11 * * 1` (coordinator per client, gate: `workflow_config.intelligence` enabled) | `batchSize: 2`; singleton `${clientId}:${isoWeek}` | est. $0.06/call vs weekly budget | `client_briefs`, `metrics_daily`, `experiments`, `anomalies`, `playbooks`, `action_queue_items` (last week's skips) | `action_queue_items`, `experiments` (auto-exec), `workflow_config` (auto-exec), `intelligence_costs`, `activity_log` |
| `intel-playbook` (playbook-aggregate-monthly) | `0 12 2 * *` (coordinator per niche) | `batchSize: 1`; singleton `${niche}:${yyyymm}` | est. $0.10/call vs a fixed $2/mo global playbook budget | `experiments`, `insights`, `clients`, `client_profile` | `playbooks`, `intelligence_costs`, `activity_log` |

Notes:
- Every fan-out coordinator selects `clients` with `isActive = true`, `onboardingComplete = true`,
  `kind <> 'prospect'`, `deletedAt IS NULL` — same predicate as `checkSchedules()` in
  `src/server/pipeline/scheduler.ts:83-95`.
- Retry policy for all intel queues: `{ retryLimit: 2, retryDelay: 60, retryBackoff: true }`
  in `RETRY_QUEUES`. LLM job handlers must be idempotent per singleton key: check for an
  existing artifact for the same key (e.g. a brief for this `${clientId}:${yyyymm}`) and
  no-op if present, so a retry after a post-LLM crash never double-spends.
- Every job writes one `activity_log` row on completion
  (`engine: 'intelligence'`, `action: '<job>.completed'`, `detail: {counts, costUsd}`) and
  on skip/failure — this powers the ops digest (doc 07 "Observable").

---

## 3. INT-1 — Schema, queues, worker registration, budget guard

### 3.1 Drizzle tables (append to `src/server/db/schema.ts`)

Spec 01 owns the growth tables; **this ticket owns the five doc-09 tables plus two support
tables**. Follow the file's existing conventions (uuid pk `defaultRandom()`, `client_id` FK
with cascade, `pgEnum`, snake_case column names, indexes as the third arg array).

```ts
// ---------- Intelligence layer (doc 09 / spec 08) ----------

export const insightDomainEnum = pgEnum("insight_domain", [
  "reputation", "leads", "reactivation", "content", "visibility", "competitive",
]);
export const insightStatusEnum = pgEnum("insight_status", [
  "active", "superseded", "refuted",
]);

/** L2 — AI-written, ~200-token, versioned insight notes. */
export const insights = pgTable(
  "insights",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    domain: insightDomainEnum("domain").notNull(),
    finding: text("finding").notNull(),          // ≤600 chars enforced in code
    /** [{kind: 'metric'|'anomaly'|'experiment', ref, value}] — audit pointers, not prose. */
    evidence: jsonb("evidence").notNull(),
    confidence: real("confidence").notNull(),    // 0..1
    recommendedAction: text("recommended_action"),
    status: insightStatusEnum("status").notNull().default("active"),
    supersededBy: uuid("superseded_by"),         // self-ref set on supersede
    /** 'insight_writer' | 'experiment_closer' | 'wf6_digest' */
    source: text("source").notNull().default("insight_writer"),
    // pgvector column added by the hand-written migration in §6.1 (nullable).
    embedding: vector1024("embedding"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    index("insights_client_domain_idx").on(t.clientId, t.domain, t.status),
    index("insights_client_created_idx").on(t.clientId, t.createdAt),
  ],
);

/** L3 — the living client brief, versioned, ≤1.5k tokens hard cap. */
export const clientBriefs = pgTable(
  "client_briefs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    version: integer("version").notNull(),
    bodyMd: text("body_md").notNull(),
    /** Measured output tokens of bodyMd at generation time (from usage.output_tokens). */
    tokenCount: integer("token_count").notNull(),
    /** {noteIds: [], experimentIds: [], playbookId, trigger: 'monthly'|'experiment_verdict'|...} */
    inputs: jsonb("inputs").notNull(),
    generatedAt: timestamp("generated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("client_briefs_version_unique").on(t.clientId, t.version)],
);

export const experimentSourceEnum = pgEnum("experiment_source", [
  "action_queue", "campaign", "content", "config_change", "manual",
]);
export const experimentVerdictEnum = pgEnum("experiment_verdict", ["win", "loss", "flat"]);

/** The learning ledger (doc 09). */
export const experiments = pgTable(
  "experiments",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    source: experimentSourceEnum("source").notNull(),
    hypothesis: text("hypothesis").notNull(),
    /** {module, configDelta?, previousConfig?, campaignId?, sourceInsightIds: []} */
    action: jsonb("action").notNull(),
    /** {metric: 'reviews.reviews_received', direction: 'up'|'down', magnitudeGuess?} */
    expectedImpact: jsonb("expected_impact").notNull(),
    startedAt: timestamp("started_at", { withTimezone: true }).notNull().defaultNow(),
    measureUntil: timestamp("measure_until", { withTimezone: true }).notNull(),
    /** null until closed: {metricBefore, metricAfter, relDelta, revenueEst, verdict} */
    outcome: jsonb("outcome"),
    verdict: experimentVerdictEnum("verdict"),
    learnedNoteId: uuid("learned_note_id").references(() => insights.id),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("experiments_due_idx").on(t.measureUntil, t.verdict)],
);

/** L4 — niche playbooks, aggregate patterns only (privacy rule enforced in INT-9). */
export const playbooks = pgTable(
  "playbooks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    niche: text("niche").notNull(),              // e.g. 'golf_recreation', 'home_services'
    version: integer("version").notNull(),
    bodyMd: text("body_md").notNull(),
    supportingExperiments: integer("supporting_experiments").notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("playbooks_niche_version_unique").on(t.niche, t.version)],
);

export const anomalyStatusEnum = pgEnum("anomaly_status", ["open", "addressed", "expired"]);

/** Detector output queue — pure stats, no AI. */
export const anomalies = pgTable(
  "anomalies",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    metric: text("metric").notNull(),            // detector key, §5
    direction: text("direction").notNull(),      // 'up' | 'down' | 'shift'
    magnitude: real("magnitude").notNull(),      // z-score / ratio / TV distance
    window: text("window").notNull(),            // human-readable, e.g. '7d vs prior 28d'
    /** Detector-specific numbers for evidence refs: {current, baseline, n, ...} */
    detail: jsonb("detail").notNull(),
    status: anomalyStatusEnum("status").notNull().default("open"),
    detectedAt: timestamp("detected_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("anomalies_client_status_idx").on(t.clientId, t.status, t.detectedAt)],
);

export const actionQueueStatusEnum = pgEnum("action_queue_status", [
  "proposed", "auto_executed", "pending_approval", "approved", "rejected", "skipped",
]);

/** WF-8 output: one row per proposed action per weekly run. */
export const actionQueueItems = pgTable(
  "action_queue_items",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id").notNull()
      .references(() => clients.id, { onDelete: "cascade" }),
    weekOf: date("week_of").notNull(),           // Monday of the run week
    rank: smallint("rank").notNull(),            // 1..5
    title: text("title").notNull(),
    rationale: text("rationale").notNull(),
    expectedImpact: jsonb("expected_impact").notNull(),
    effort: text("effort").notNull(),            // 'low' | 'medium' | 'high'
    executionPath: text("execution_path").notNull(), // 'auto' | 'approval' | 'human'
    module: text("module").notNull(),
    proposal: jsonb("proposal").notNull(),       // §9.3
    sourceInsightIds: jsonb("source_insight_ids").notNull(), // string[]
    status: actionQueueStatusEnum("status").notNull().default("proposed"),
    statusReason: text("status_reason"),
    experimentId: uuid("experiment_id").references(() => experiments.id),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    decidedAt: timestamp("decided_at", { withTimezone: true }),
  },
  (t) => [uniqueIndex("action_queue_week_rank_unique").on(t.clientId, t.weekOf, t.rank)],
);

/** LLM spend ledger for the intelligence layer (feeds the weekly budget guard). */
export const intelligenceCosts = pgTable(
  "intelligence_costs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    clientId: uuid("client_id")
      .references(() => clients.id, { onDelete: "cascade" }), // null = cross-client (playbooks)
    job: text("job").notNull(),
    model: text("model").notNull(),
    costUsd: numeric("cost_usd", { precision: 8, scale: 4 }).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("intel_costs_client_created_idx").on(t.clientId, t.createdAt)],
);
```

`vector1024` is a `customType` (same mechanism as the existing `bytea` customType at the top
of `schema.ts`):

```ts
const vector1024 = customType<{ data: number[] }>({
  dataType() { return "vector(1024)"; },
  toDriver(v: number[]) { return `[${v.join(",")}]`; },
});
```

Migration: run `npm run db:generate`, then append to the generated SQL file (drizzle-kit
does not know pgvector): `CREATE EXTENSION IF NOT EXISTS vector;` as the first statement,
and an ivfflat index:
`CREATE INDEX insights_embedding_idx ON insights USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`
Migrations auto-apply on worker boot via `runMigrations()` (`scripts/migrate.ts`), already
called at the top of `src/worker.ts:28`.

### 3.2 Queue + worker registration

`src/server/pipeline/queue.ts` — add to `QUEUES`:

```ts
  intelNightly: "intel-nightly",
  intelRollup: "intel-rollup",
  intelAnomaly: "intel-anomaly",
  intelInsights: "intel-insights",
  intelBrief: "intel-brief",
  intelActionQueue: "intel-action-queue",
  intelExperimentClose: "intel-experiment-close",
  intelPlaybook: "intel-playbook",
```

Add each to `RETRY_QUEUES` with `{ retryLimit: 2, retryDelay: 60, retryBackoff: true }`,
and job payload interfaces:

```ts
export interface IntelClientJob { clientId: string; date: string /* YYYY-MM-DD */; }
export interface IntelInsightJob extends IntelClientJob { domain: InsightDomain; }
export interface IntelBriefJob { clientId: string; trigger: "monthly" | "experiment_verdict" | "config_change" | "manual"; }
export interface IntelExperimentCloseJob { experimentId: string; }
export interface IntelPlaybookJob { niche: string; month: string /* YYYY-MM */; }
```

`src/worker.ts` — register handlers following the existing shape exactly (log line, call
module function, batch handlers wrap each job in try/catch like the `promptTask` handler at
`src/worker.ts:42-58`):

```ts
import { runIntelNightlyCoordinator, runWeeklyCoordinators, runMonthlyCoordinators }
  from "./server/intelligence/scheduler";
import { runRollup } from "./server/intelligence/rollup";
import { detectAnomalies } from "./server/intelligence/anomalies";
import { writeInsights } from "./server/intelligence/insight-writer";
import { refreshBrief } from "./server/intelligence/brief";
import { runActionQueue } from "./server/intelligence/action-queue";
import { closeExperiment } from "./server/intelligence/experiments";
import { aggregatePlaybook } from "./server/intelligence/playbooks";

await boss.work<IntelClientJob>(QUEUES.intelRollup, { batchSize: 4 }, async (jobs) => {
  await Promise.all(jobs.map(async (job) => {
    try { await runRollup(job.data.clientId, job.data.date); }
    catch (err) { logError(`[worker] intel-rollup ${job.data.clientId} failed:`, err); throw err; }
  }));
});
// ... same pattern for intelAnomaly (batchSize 4), intelInsights / intelBrief /
// intelActionQueue (batchSize 2), intelExperimentClose (batchSize 2), intelPlaybook (1).

await boss.work(QUEUES.intelNightly, async () => { await runIntelNightlyCoordinator(); });
await boss.schedule(QUEUES.intelNightly, "10 7 * * *");
```

The weekly/monthly coordinators piggyback on the **existing hourly `check-schedules` tick**
(`src/worker.ts:93-102`) rather than adding more `boss.schedule` crons — add two calls inside
that handler: `await runWeeklyCoordinators(new Date())` and
`await runMonthlyCoordinators(new Date())`. Each checks "is it Monday && past 09:30 UTC &&
no job with this singleton key sent yet" — the singleton keys make re-checks no-ops, the
same trick `checkReportSchedules()` uses. (`intel-nightly` gets its own cron because the
rollup → closer → anomaly ordering needs sub-hourly offsets; the closer and anomaly
coordinators are `boss.schedule`d too: `0 8 * * *` and `30 8 * * *`.)

`src/server/intelligence/scheduler.ts`:

```ts
import { and, eq, isNull, ne, lte } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, experiments } from "@/server/db/schema";
import { getBoss, QUEUES, type IntelClientJob } from "@/server/pipeline/queue";

/** Same active-client predicate as checkSchedules() in pipeline/scheduler.ts. */
export async function activeClients() {
  return db.select({ id: clients.id }).from(clients).where(and(
    eq(clients.isActive, true),
    eq(clients.onboardingComplete, true),
    ne(clients.kind, "prospect"),
    isNull(clients.deletedAt),
  ));
}

export async function runIntelNightlyCoordinator(now = new Date()): Promise<void> {
  // Roll up YESTERDAY's UTC day — the day is complete.
  const date = new Date(now.getTime() - 24 * 60 * 60 * 1000).toISOString().slice(0, 10);
  const boss = await getBoss();
  const rows = await activeClients();
  await boss.insert(QUEUES.intelRollup, rows.map((c) => ({
    data: { clientId: c.id, date } satisfies IntelClientJob as unknown as object,
    singletonKey: `${c.id}:${date}`,
    expireInSeconds: 600,
  })));
}
// anomaly coordinator: identical, queue intelAnomaly, singleton `${clientId}:${date}`.
// experiment-close coordinator: select experiments where measureUntil <= now and verdict is null,
//   enqueue { experimentId } with singletonKey experimentId.
// weekly coordinator (Mondays): per client — enqueue intelInsights once per domain in
//   DOMAINS_FOR_CLIENT(clientId) (see §7.1), singleton `${clientId}:${domain}:${isoWeek}`,
//   then intelActionQueue (only if workflow_config.intelligence enabled), singleton `${clientId}:${isoWeek}`.
// monthly coordinator (1st): per client — intelBrief singleton `${clientId}:${yyyymm}`;
//   (2nd): per niche with ≥2 eligible clients — intelPlaybook singleton `${niche}:${yyyymm}`.
```

### 3.3 Budget guard — `src/server/intelligence/spend.ts`

Mirrors the pre-run cost guard in `runClient()` (`run-client.ts:96-108`) but against
`clients.weeklyBudgetUsd` (the schema comment on that column already designates it the
"rolling 7-day spend ceiling across ALL dispatched agent work"):

```ts
import { and, eq, gte, sql } from "drizzle-orm";
import { db } from "@/server/db";
import { clients, intelligenceCosts } from "@/server/db/schema";
import { log } from "@/lib/logger";

const EST_COST_USD: Record<string, number> = {
  "intel-insights": 0.02, "intel-brief": 0.04,
  "intel-action-queue": 0.06, "intel-playbook": 0.10,
};

/** Returns true if the job may run; false → caller logs + skips (never throws). */
export async function checkIntelBudget(clientId: string, job: string): Promise<boolean> {
  const [client] = await db.select({ budget: clients.weeklyBudgetUsd })
    .from(clients).where(eq(clients.id, clientId));
  if (!client) return false;
  const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const [row] = await db.select({ spent: sql<string>`coalesce(sum(${intelligenceCosts.costUsd}), 0)` })
    .from(intelligenceCosts)
    .where(and(eq(intelligenceCosts.clientId, clientId), gte(intelligenceCosts.createdAt, weekAgo)));
  const ok = parseFloat(row.spent) + (EST_COST_USD[job] ?? 0.05) <= parseFloat(client.budget);
  if (!ok) log(`[intel] ${job} skipped for ${clientId}: weekly intelligence budget exhausted`);
  return ok;
}

export async function recordIntelCost(
  clientId: string | null, job: string, model: string, costUsd: number,
): Promise<void> {
  await db.insert(intelligenceCosts)
    .values({ clientId, job, model, costUsd: costUsd.toFixed(4) });
}
```

Every LLM job: `if (!(await checkIntelBudget(clientId, JOB))) { writeActivity('skipped_budget'); return; }`
and after the call `recordIntelCost(clientId, JOB, MODEL, anthropicUsageCost(MODEL, response.usage))`.

**Acceptance (INT-1):** migrations apply on a fresh DB and on prod-shaped data; worker boots
with all handlers registered; enqueueing the same singleton key twice runs the job once;
`checkIntelBudget` returns false once the ledger sum crosses `weeklyBudgetUsd`.

---

## 4. INT-2 — `rollup-nightly` (`src/server/intelligence/rollup.ts`)

One handler per client per day. Computes one `metrics_daily` row per module for `date`
(UTC day), **idempotent upsert** on `(client_id, date, module)` — the exact pattern of
`recomputeDailyAggregates()` in `src/server/pipeline/aggregate.ts:149-166`
(`onConflictDoUpdate` with `sql\`excluded.<col>\``). Metric shapes are doc 04's, verbatim
keys; extension keys are suffixed and additive.

```ts
import { and, eq, gte, lt, sql, inArray } from "drizzle-orm";
import { db } from "@/server/db";
import {
  activityLog, metricsDaily, conversations, campaigns, campaignMembers,
  reviews, reviewRequests, contentItems, syndicatedPosts, dailyAggregates, brands,
  growthMessages, // spec-01 `messages` growth ledger; rename import to match spec 01
} from "@/server/db/schema";

export async function runRollup(clientId: string, date: string): Promise<void> {
  const dayStart = new Date(`${date}T00:00:00.000Z`);
  const dayEnd = new Date(dayStart.getTime() + 24 * 60 * 60 * 1000);
  const inDay = (col: AnyColumn) => and(gte(col, dayStart), lt(col, dayEnd));

  const rows: { clientId: string; date: string; module: string; metrics: unknown }[] = [];

  // ---- module: 'reviews' -> {requests_sent, reviews_received, avg_rating, responses_published}
  const [req] = await db.select({ n: sql<number>`count(*)::int` }).from(reviewRequests)
    .where(and(eq(reviewRequests.clientId, clientId), inDay(reviewRequests.sentAt)));
  const [rcv] = await db.select({
    n: sql<number>`count(*)::int`,
    avg: sql<number | null>`avg(${reviews.rating})::float`,
    neg: sql<number>`count(*) filter (where ${reviews.rating} <= 3)::int`,
  }).from(reviews)
    .where(and(eq(reviews.clientId, clientId), inDay(reviews.reviewedAt)));
  const [pub] = await db.select({ n: sql<number>`count(*)::int` }).from(activityLog)
    .where(and(
      eq(activityLog.clientId, clientId),
      eq(activityLog.action, "review_response.published"),
      eq(activityLog.status, "ok"),
      inDay(activityLog.createdAt),
    ));
  rows.push({ clientId, date, module: "reviews", metrics: {
    requests_sent: req.n, reviews_received: rcv.n,
    avg_rating: rcv.avg === null ? null : Number(rcv.avg.toFixed(2)),
    responses_published: pub.n,
    negative_reviews_ext: rcv.neg,                       // extension key
  }});

  // ---- module: 'leads' -> {leads, median_response_s, qualified, booked}
  const [leads] = await db.select({
    leads: sql<number>`count(*)::int`,
    median: sql<number | null>`percentile_cont(0.5) within group
      (order by ${conversations.firstResponseSeconds})`,
    qualified: sql<number>`count(*) filter (where ${conversations.status} in ('qualified','booked'))::int`,
    booked: sql<number>`count(*) filter (where ${conversations.booked})::int`,
  }).from(conversations).where(and(
    eq(conversations.clientId, clientId),
    eq(conversations.engine, "wf-2"),
    inDay(conversations.createdAt),
  ));
  rows.push({ clientId, date, module: "leads", metrics: {
    leads: leads.leads,
    median_response_s: leads.median === null ? null : Math.round(leads.median),
    qualified: leads.qualified, booked: leads.booked,
  }});

  // ---- module: 'reactivation' -> {sent, replies, bookings, revenue_est}
  const [sent] = await db.select({ n: sql<number>`count(*)::int` }).from(growthMessages)
    .where(and(
      eq(growthMessages.clientId, clientId), eq(growthMessages.engine, "wf-3"),
      eq(growthMessages.direction, "out"),
      inArray(growthMessages.status, ["sent", "delivered"]),
      inDay(growthMessages.createdAt),
    ));
  const [replies] = await db.select({ n: sql<number>`count(*)::int` }).from(growthMessages)
    .where(and(
      eq(growthMessages.clientId, clientId), eq(growthMessages.engine, "wf-3"),
      eq(growthMessages.direction, "in"), inDay(growthMessages.createdAt),
    ));
  const [react] = await db.select({
    bookings: sql<number>`count(*) filter (where ${conversations.booked})::int`,
  }).from(conversations).where(and(
    eq(conversations.clientId, clientId), eq(conversations.engine, "wf-3"),
    inDay(conversations.createdAt),
  ));
  // revenue_est: sum of the day's per-campaign revenue deltas is not reconstructible from
  // stats jsonb; instead attribute conversation-level revenue when WF-3 marks it:
  const [rev] = await db.select({
    est: sql<number>`coalesce(sum((${conversations.context} ->> 'revenue_est')::numeric), 0)::float`,
  }).from(conversations).where(and(
    eq(conversations.clientId, clientId), eq(conversations.engine, "wf-3"),
    eq(conversations.booked, true), inDay(conversations.createdAt),
  ));
  rows.push({ clientId, date, module: "reactivation", metrics: {
    sent: sent.n, replies: replies.n, bookings: react.bookings, revenue_est: rev.est,
  }});

  // ---- module: 'content' -> {items_published, syndicated_posts, utm_clicks}
  const [pubItems] = await db.select({ n: sql<number>`count(*)::int` }).from(contentItems)
    .where(and(eq(contentItems.clientId, clientId), inDay(contentItems.publishedAt)));
  const [synd] = await db.select({
    n: sql<number>`count(*)::int`,
    clicks: sql<number>`coalesce(sum(${syndicatedPosts.clicks}), 0)::int`,
  }).from(syndicatedPosts)
    .where(and(eq(syndicatedPosts.clientId, clientId), inDay(syndicatedPosts.postedAt)));
  rows.push({ clientId, date, module: "content", metrics: {
    items_published: pubItems.n, syndicated_posts: synd.n, utm_clicks: synd.clicks,
  }});

  // ---- module: 'visibility-growth' -> mirror of the SoV all-engines aggregate for the own brand
  // (daily_aggregates is the SoV product's table — see pipeline/aggregate.ts; engineId NULL row
  //  is the all-engines rollup, relation 'own' is the client's brand.)
  const [vis] = await db.select({
    pct: dailyAggregates.visibilityPct, scored: dailyAggregates.responsesScored,
    sentiment: dailyAggregates.avgSentiment,
  }).from(dailyAggregates)
    .innerJoin(brands, eq(brands.id, dailyAggregates.brandId))
    .where(and(
      eq(dailyAggregates.clientId, clientId), eq(dailyAggregates.date, date),
      sql`${dailyAggregates.engineId} is null`, eq(brands.relation, "own"),
    )).limit(1);
  if (vis) rows.push({ clientId, date, module: "visibility-growth", metrics: {
    visibility_pct: Number(vis.pct), responses_scored: vis.scored,
    avg_sentiment: vis.sentiment === null ? null : Number(vis.sentiment),
  }});

  await db.insert(metricsDaily).values(rows).onConflictDoUpdate({
    target: [metricsDaily.clientId, metricsDaily.date, metricsDaily.module],
    set: { metrics: sql`excluded.metrics` },
  });
}
```

Also export the **L1 query helper** other engines and the LLM context assemblers use
(anti-flooding rule 1 — "one number, not rows"):

```ts
/** e.g. metricValue(clientId, 'leads', 'median_response_s', 30) -> avg over last 30 days. */
export async function metricAvg(
  clientId: string, module: string, key: string, days: number, endDate?: string,
): Promise<number | null>;
/** Sum variant for count metrics. */
export async function metricSum(/* same signature */): Promise<number>;
```

Both are one `SELECT avg/sum((metrics->>$key)::numeric) FROM metrics_daily WHERE ...` query.

**Acceptance (INT-2):** re-running the job for the same `(client, date)` produces identical
rows (upsert proven by test); a day with zero activity writes all-zero metric rows (not
missing rows — detectors need the zeros); module shapes match doc 04 keys exactly.

---

## 5. INT-3 — `anomaly-detect-nightly` (`src/server/intelligence/anomalies.ts`)

Pure stat, **no LLM**. Reads `metrics_daily` (and `daily_aggregates` for visibility). Each
detector returns `null` or a candidate row. Shared guards:

- **Dedup:** skip insert if an `open` anomaly exists for the same `(clientId, metric)`, or
  any anomaly for that `(clientId, metric)` was detected in the last 7 days.
- **Expiry sweep (start of job):** `open` anomalies with `detectedAt < now − 30d` → `expired`.
- **Minimum data:** each detector has explicit `n` floors below which it stays silent —
  a pilot with 3 reviews must never fire "review velocity collapse."

Helper: `async function series(clientId, module, key, fromDay, toDay): Promise<number[]>`
(daily values from `metrics_daily`, missing days = 0 for counts / skipped for medians).

### Detector table (initial set — exact formulas)

| `metric` key | Inputs | Formula | Fires when | `direction`, `magnitude` |
|---|---|---|---|---|
| `review_velocity` | `reviews.reviews_received` daily counts | x = sum(last 7d). Baseline = the 4 weekly sums over days −35..−8: μ = mean, σ = pop. stddev, σ★ = max(σ, √μ, 1). z = (x − μ)/σ★ | z ≤ −2.0 (drop) or z ≥ +3.0 (spike — competitor-style review push or our campaign working) | `down`/`up`, magnitude = z. Floor: μ ≥ 2 |
| `response_time_regression` | `conversations.firstResponseSeconds` (engine wf-2) — computed from raw conversations because medians don't average | m7 = median over last 7d (n ≥ 5); m28 = median over days −35..−8 (n ≥ 10) | m7 ≥ max(1.5 × m28, m28 + 60) | `up`, magnitude = m7 / m28 |
| `lead_source_mix_shift` | `conversations.leadSource` distribution | p = share per source, last 14d (n ≥ 10); q = share per source, days −74..−15 (n ≥ 20); TV = ½ Σᵢ \|pᵢ − qᵢ\| over the union of sources | TV ≥ 0.25 | `shift`, magnitude = TV; `detail.top_mover` = source with max \|pᵢ−qᵢ\| |
| `rating_drop` | `reviews.rating` | a14 = avg rating last 14d (n ≥ 3); a90 = avg prior 90d (n ≥ 10) | a14 ≤ a90 − 0.5 | `down`, magnitude = a90 − a14 |
| `reactivation_reply_drop` | `reactivation.sent` / `reactivation.replies` daily sums | r14 = replies/sent last 14d (sent ≥ 30); rBase = replies/sent days −104..−15 (sent ≥ 100) | r14 ≤ 0.5 × rBase | `down`, magnitude = rBase / max(r14, 0.001) |
| `visibility_drop` | `visibility-growth.visibility_pct` daily values | v7 = mean last 7d; vPrev = mean days −14..−8 (both need ≥ 4 datapoints) | vPrev ≥ 15 and vPrev − v7 ≥ 10 (absolute points) | `down`, magnitude = vPrev − v7 |
| `booking_rate_drop` | `leads.booked` / `leads.leads` | b14 = booked/leads last 14d (leads ≥ 10); bBase = prior 60d (leads ≥ 30) | b14 ≤ 0.5 × bBase | `down`, magnitude = bBase / max(b14, 0.001) |

`detail` always records `{current, baseline, n_current, n_baseline}` plus detector extras —
these become the evidence refs the insight writer cites (`anomaly:<id>`).

Population stddev, computed in TS over the 4 weekly sums:
`σ = sqrt(Σ(wᵢ−μ)² / 4)`. The `√μ` floor is the Poisson noise floor so small-count weeks
don't produce huge z-scores.

Each fired detector also writes `activity_log`
(`engine:'intelligence'`, `action:'anomaly.detected'`, `entity_type:'anomaly'`, `entity_id`).

**Acceptance (INT-3):** unit tests per detector with synthetic series (fires / doesn't fire /
data-floor silence); dedup proven (second nightly run inserts nothing new); zero LLM imports
in the module (lint assertion: no `@anthropic-ai/sdk` import).

---

## 6. INT-4 — Embeddings + retrieval helpers

### 6.1 `src/server/intelligence/embeddings.ts`

Embeddings are an **optional enrichment** (doc 09: notes are "embedded for retrieval").
There is no embedding provider in the repo today; use Voyage AI's REST API (Anthropic's
recommended embedding partner) behind an optional key, degrading to null exactly like the
repo's Twilio-optional pattern in `src/env.ts`.

- Add to `src/env.ts` schema: `VOYAGE_API_KEY: z.string().optional(),`
- Model: `voyage-3.5-lite`, `output_dimension: 1024` (matches the `vector(1024)` column).

```ts
const VOYAGE_URL = "https://api.voyageai.com/v1/embeddings";

/** Returns null when no key configured or on any error — embedding is never load-bearing. */
export async function embed(text: string): Promise<number[] | null> {
  if (!env.VOYAGE_API_KEY) return null;
  try {
    const res = await fetch(VOYAGE_URL, {
      method: "POST",
      headers: { "content-type": "application/json", authorization: `Bearer ${env.VOYAGE_API_KEY}` },
      body: JSON.stringify({ model: "voyage-3.5-lite", input: [text.slice(0, 4000)], output_dimension: 1024 }),
      signal: AbortSignal.timeout(10_000),
    });
    if (!res.ok) return null;
    const json = (await res.json()) as { data: { embedding: number[] }[] };
    return json.data[0]?.embedding ?? null;
  } catch { return null; }
}
```

### 6.2 `src/server/intelligence/retrieval.ts` — the API other engines consume

```ts
export interface BriefResult { bodyMd: string; version: number; generatedAt: Date; tokenCount: number; }

/** Latest L3 brief, or null before the first refresh. NEVER substitutes raw data. */
export async function getBriefForClient(clientId: string): Promise<BriefResult | null> {
  const [row] = await db.select().from(clientBriefs)
    .where(eq(clientBriefs.clientId, clientId))
    .orderBy(desc(clientBriefs.version)).limit(1);
  return row ? { bodyMd: row.bodyMd, version: row.version, generatedAt: row.generatedAt, tokenCount: row.tokenCount } : null;
}

export interface InsightResult {
  id: string; domain: InsightDomain; finding: string;
  evidence: EvidenceRef[]; confidence: number; recommendedAction: string | null; createdAt: Date;
}

/**
 * Active L2 notes for one domain, best-first.
 * - No `query`: ORDER BY confidence DESC, created_at DESC (deterministic, embedding-free).
 * - With `query` and embeddings available: cosine distance via pgvector (`<=>`), falling
 *   back to the deterministic order when the query can't be embedded.
 * limit default 5 — callers assembling agent context should use 3.
 */
export async function getInsights(
  clientId: string, domain: InsightDomain, limit = 5, query?: string,
): Promise<InsightResult[]> {
  if (query) {
    const qe = await embed(query);
    if (qe) {
      return db.select(/* cols */).from(insights).where(and(
        eq(insights.clientId, clientId), eq(insights.domain, domain),
        eq(insights.status, "active"), sql`${insights.embedding} is not null`,
      )).orderBy(sql`${insights.embedding} <=> ${sql.raw(`'[${qe.join(",")}]'::vector`)}`)
        .limit(limit);
    }
  }
  return db.select(/* cols */).from(insights).where(and(
    eq(insights.clientId, clientId), eq(insights.domain, domain), eq(insights.status, "active"),
  )).orderBy(desc(insights.confidence), desc(insights.createdAt)).limit(limit);
}
```

### 6.3 Usage — n8n context endpoint enrichment

Spec 02's internal API route `GET /api/internal/clients/:id/context?module=<module>`
(doc 10 §Internal API) adds an `intelligence` block using these two helpers — **this is the
only path by which n8n engines ever see intelligence data**:

```ts
const MODULE_TO_DOMAIN: Record<string, InsightDomain> = {
  review_automation: "reputation", speed_to_lead: "leads", reactivation: "reactivation",
  content_engine: "content", social_syndication: "content",
  directory_sync: "visibility", competitor_intel: "competitive",
};

// inside the route handler:
const [brief, notes] = await Promise.all([
  getBriefForClient(clientId),
  getInsights(clientId, MODULE_TO_DOMAIN[module], 3),
]);
payload.intelligence = {
  brief: brief?.bodyMd ?? null,                      // ≤1.5k tokens by construction
  insights: notes.map(({ id, finding, confidence, recommendedAction }) =>
    ({ id, finding, confidence, recommended_action: recommendedAction })),
};                                                    // ≤ ~2.1k tokens added, total, ever
```

Usage notes for engine builders (put this comment block in `retrieval.ts`):
- WF-2/WF-3 composers: load `brief` + domain notes into the system prompt. Never request
  more than 3 notes. Never pass transcripts of other customers.
- WF-6: write digest findings back as `competitive`-domain insights via a plain
  `db.insert(insights)` with `source: 'wf6_digest'`, confidence 0.5, ≤600-char finding.

**Acceptance (INT-4):** with `VOYAGE_API_KEY` unset everything works (null embeddings,
deterministic ordering); with it set, a seeded similarity test retrieves the semantically
closest note first; the context endpoint's `intelligence` block serializes to < 9,000 chars.

---

## 7. INT-5 — `insight-writer-weekly` (`src/server/intelligence/insight-writer.ts`)

One job per **client × domain** per week. Module structure copies
`src/server/pipeline/review-insights.ts` (orchestration + upsert) and
`src/server/analysis/review-insights.ts` (LLM call + zod + mock) — cite both in the PR.

### 7.1 Which domains run

```ts
const DOMAIN_MODULES: Record<InsightDomain, string[]> = {
  reputation: ["reviews"], leads: ["leads"], reactivation: ["reactivation"],
  content: ["content"], visibility: ["visibility-growth"],
  competitive: [], // written by WF-6, never by this job
};
/** A domain runs when its workflow switch is on AND it has ≥7 metrics_daily rows. */
export async function domainsForClient(clientId: string): Promise<InsightDomain[]>;
```

### 7.2 Context assembly — numbers only, fixed size

```ts
interface InsightContext {
  clientName: string; domain: InsightDomain;
  deltas: string;        // ≤ 1,200 chars: per metric key "last7=…, prior28avg=…, Δ=…%"
  anomalies: string;     // ≤ 800 chars: open anomalies for the domain, "[anomaly:<id>] …"
  experiments: string;   // ≤ 800 chars: experiments closed in last 7d touching the domain
  existingNotes: { id: string; finding: string; confidence: number }[]; // active, max 8
}
```

`deltas` is built with `metricSum`/`metricAvg` from §4 (weekly sum vs prior-28d weekly
average per metric key of the domain's modules). Anomaly → its `metric`, `direction`,
`magnitude`, `window`, `detail.current/baseline`. Experiment → hypothesis + verdict +
`outcome.relDelta`. Existing notes are included **so the model can supersede/refute them**.

### 7.3 Prompt (single user message, exactly like `analyzeReviews`)

```
You are the weekly intelligence analyst for "${clientName}", a local business managed by
the Seekly growth platform. Domain under review: ${domain}.

You receive ONLY pre-computed numbers. Never invent numbers; cite the refs you were given.

WEEKLY METRIC DELTAS (last 7 days vs trailing 28-day weekly average):
${deltas}

OPEN ANOMALIES (statistical detectors; each has a ref like [anomaly:<id>]):
${anomalies || "none"}

EXPERIMENTS CLOSED THIS WEEK:
${experiments || "none"}

EXISTING ACTIVE NOTES for this domain (id | confidence | finding):
${existingNotes.map((n) => `${n.id} | ${n.confidence} | ${n.finding}`).join("\n") || "none"}

Write 0 to 3 NEW insight notes. Rules:
- A note is a durable, decision-relevant pattern — not a restatement of one week's delta.
  If nothing durable emerged this week, return an empty list. Empty is a good answer.
- finding: ≤ 2 sentences, ≤ 500 characters, concrete and falsifiable.
- evidence: 1-4 refs pointing at the inputs above. kind "metric" ref format
  "<module>.<key>:last7", kind "anomaly" ref format "anomaly:<id>", kind "experiment"
  ref format "experiment:<id>". value = the number you are citing, as a short string.
- confidence: 0.3 = plausible pattern, 0.5 = supported by one clean signal,
  0.7 = supported by multiple signals or a closed experiment. Never exceed 0.8 —
  only experiment verdicts raise confidence above that.
- recommended_action: one imperative sentence an operator could execute, or null.
- supersedes: ids of existing notes above that this note REPLACES (refines/updates).
- refutes: ids of existing notes above that this week's evidence CONTRADICTS.
Do not repeat an existing note without superseding it.
```

### 7.4 Schema + budget enforcement

JSON schema (for `output_config.format`) mirrors the zod schema; zod is the gate:

```ts
const evidenceRef = z.object({
  kind: z.enum(["metric", "anomaly", "experiment"]),
  ref: z.string().max(80),
  value: z.string().max(60),
});
const insightNote = z.object({
  finding: z.string().min(20).max(600),
  evidence: z.array(evidenceRef).min(1).max(4),
  confidence: z.number().min(0).max(0.8),
  recommended_action: z.string().max(240).nullable(),
  supersedes: z.array(z.string()).max(3),
  refutes: z.array(z.string()).max(3),
});
const InsightsResponse = z.object({ notes: z.array(insightNote).max(3) });
```

Call: `getAnthropicClient({ timeoutMs: 60_000 })` (from `src/server/analysis/anthropic.ts`),
`model: "claude-sonnet-4-6"`, **`max_tokens: 1000`** (3 notes × ~200 tokens + JSON syntax —
this is the ~200-token budget made physical), `output_config: { format: { type: "json_schema",
schema: INSIGHT_JSON_SCHEMA } }`. Post-validation: drop any note whose serialized JSON
exceeds 1,000 chars (≈250 tokens) and log it; drop notes whose `supersedes`/`refutes` ids
are not in `existingNotes` (the model may not touch notes it wasn't shown).

### 7.5 Persistence + supersede/refute

```ts
for (const note of accepted) {
  const [row] = await db.insert(insights).values({
    clientId, domain, finding: note.finding, evidence: note.evidence,
    confidence: note.confidence, recommendedAction: note.recommended_action,
    embedding: await embed(`${domain}: ${note.finding}`),
  }).returning();
  if (note.supersedes.length) {
    await db.update(insights)
      .set({ status: "superseded", supersededBy: row.id })
      .where(and(inArray(insights.id, note.supersedes),
                 eq(insights.clientId, clientId), eq(insights.status, "active")));
  }
  if (note.refutes.length) {
    await db.update(insights).set({ status: "refuted" })
      .where(and(inArray(insights.id, note.refutes),
                 eq(insights.clientId, clientId), eq(insights.status, "active")));
  }
}
```

Superseded/refuted rows are **kept forever** (L2 history — doc 09 rule 2: "superseded
content falls back to L2 history"). Mock mode: deterministic 0–1 notes seeded from
`hashCode(clientId + domain + isoWeek)` (reuse the `mockInsights` approach).

**Acceptance (INT-5):** validation-failure path degrades to zero notes without throwing
(matching `analyzeReviews`'s `safeParse` + log pattern); a superseded note disappears from
`getInsights` results; per-call cost recorded; `usage.output_tokens ≤ 1000` in test fixtures.

---

## 8. INT-6 — `brief-refresh-monthly` (`src/server/intelligence/brief.ts`)

### 8.1 Inputs

- Active insights: top 4 per domain by `confidence DESC, createdAt DESC` (max 24 notes).
- Experiment ledger summary: last 6 closed experiments (hypothesis, verdict, relDelta) +
  win/loss/flat counts all-time. **Losses are included explicitly as anti-patterns.**
- Current playbook `bodyMd` for the client's niche (from `client_profile.industry` →
  niche mapping), if any — the model may borrow defaults but must not copy it wholesale.
- Static facts: business name, niche, services (from `client_profile`), enabled modules.
- Previous brief `bodyMd` (so durable content survives regeneration).

### 8.2 Prompt

```
You maintain the CLIENT BRIEF for "${clientName}" (${niche}) — the single document every
Seekly AI agent loads before acting for this client. It must contain only durable,
operational knowledge. HARD LIMIT: 1400 tokens (~5,600 characters). If everything cannot
fit, DROP the least action-relevant content — the underlying notes remain retrievable.

Structure (markdown, these five H2 sections, nothing else):
## Business — 3-5 bullet facts (services, seasonality, audience).
## What works — proven angles/settings, each ending with (evidence: experiment/insight id).
## What failed — anti-patterns we must not retry, each with its evidence id.
## Watchouts — active risks from anomalies/low-confidence notes.
## Current focus — the 2-3 levers the next month's actions should pull.

Rules: no raw transcripts, no customer PII, no numbers you were not given, no praise or
filler prose. Prefer editing the previous brief over rewriting; keep ids as (evidence: ...).

PREVIOUS BRIEF:
${previousBriefMd || "none — first brief"}

ACTIVE INSIGHT NOTES (id | domain | confidence | finding | recommended_action):
${notesBlock}

EXPERIMENT LEDGER (id | verdict | relDelta | hypothesis):
${experimentsBlock}

NICHE PLAYBOOK EXCERPT (aggregate patterns from other ${niche} clients):
${playbookExcerpt || "none yet"}

Return ONLY the markdown brief.
```

Plain-text response (no JSON schema — the artifact is markdown).

### 8.3 HARD 1.5k-token cap — retry-with-compress loop (exact code)

The measurement is the API's own `usage.output_tokens` (no separate count-tokens call
needed), the same `usage` object `anthropicUsageCost()` already consumes:

```ts
const BRIEF_MODEL = "claude-sonnet-4-6";
const HARD_CAP_TOKENS = 1500;     // doc 09 hard cap
const TARGET_TOKENS = 1400;       // what we ask for, leaving headroom

export async function generateBriefBody(
  prompt: string,
): Promise<{ bodyMd: string; tokenCount: number; costUsd: number }> {
  const client = getAnthropicClient({ timeoutMs: 90_000 });
  let costUsd = 0;
  let lastDraft = "";
  let lastTokens = 0;

  for (let attempt = 0; attempt < 3; attempt++) {
    const messages: Anthropic.MessageParam[] =
      attempt === 0
        ? [{ role: "user", content: prompt }]
        : [
            { role: "user", content: prompt },
            {
              role: "user",
              content:
                `Your draft below is ${lastTokens} tokens — OVER the ${TARGET_TOKENS}-token ` +
                `limit. Rewrite it under the limit by deleting the least action-relevant ` +
                `bullets entirely (do not compress wording into fragments; drop whole items).` +
                `\n\nDRAFT:\n${lastDraft}`,
            },
          ];

    const response = await client.messages.create({
      model: BRIEF_MODEL,
      // Physical ceiling slightly above the cap so we can DETECT overrun via
      // stop_reason instead of silently truncating mid-sentence.
      max_tokens: HARD_CAP_TOKENS + 200,
      messages,
    });
    costUsd += anthropicUsageCost(BRIEF_MODEL, response.usage);
    const text = response.content.find((b) => b.type === "text");
    if (!text || text.type !== "text") continue;
    lastDraft = text.text.trim();
    lastTokens = response.usage.output_tokens;

    if (response.stop_reason === "end_turn" && lastTokens <= HARD_CAP_TOKENS) {
      return { bodyMd: lastDraft, tokenCount: lastTokens, costUsd };
    }
  }

  // Final fallback: deterministic truncation at the last complete "## " section that fits
  // ~HARD_CAP_TOKENS * 4 chars. The brief NEVER exceeds the cap. Log loudly.
  logError(`[intel-brief] compress loop failed; hard-truncating`);
  const truncated = truncateAtSectionBoundary(lastDraft, HARD_CAP_TOKENS * 4);
  return { bodyMd: truncated, tokenCount: HARD_CAP_TOKENS, costUsd };
}
```

### 8.4 Versioning + on-demand trigger

- Insert `client_briefs` with `version = (previous ?? 0) + 1`, `tokenCount`, `inputs`
  (`{noteIds, experimentIds, playbookId, trigger}`). Never update rows — versions are the
  audit trail. Idempotency: if a brief with `inputs.trigger === 'monthly'` already exists
  for the current month, the monthly job no-ops (retry safety).
- `enqueueBriefRefresh(clientId, trigger)` — a one-line `boss.send(QUEUES.intelBrief, ...)`
  with singleton `${clientId}:evt:${trigger}:${date}`. Called from:
  - `closeExperiment()` when `verdict !== 'flat'` and `|relDelta| ≥ 0.25` (major event),
  - spec 02's switchboard write path when a module flips on/off,
  - the admin UI ("Refresh brief" button → trigger `'manual'`).

**Acceptance (INT-6):** cap proven by test with a fake client whose fixture forces a long
draft (mock returns >1500-token text; loop truncates at section boundary); version
increments; on-demand trigger dedupes within a day; brief renders in the admin client view.

---

## 9. INT-8 — WF-8 `action-queue-weekly` (`src/server/intelligence/action-queue.ts`)

Gate at the top of the handler: `workflow_config` row `(clientId, 'intelligence')` enabled —
doc 09: internal always-on once active, but the switch controls it like every engine.

### 9.1 Fixed context assembly — ≤4k tokens, shown in full

Char budgets, 4 chars ≈ 1 token. `clamp()` truncates at a line boundary and appends `"…"`.

```ts
const CTX_BUDGET_CHARS = {
  brief: 6000,        // L3 — already ≤1.5k tokens by construction
  deltas: 2400,       // this week's numbers, all modules
  experiments: 1600,  // open experiments (so it doesn't re-propose running tests)
  anomalies: 1200,    // top unaddressed anomalies
  playbook: 3200,     // niche playbook excerpt (L4)
  skipped: 800,       // last week's skipped/rejected actions + reasons (doc 09 step 5)
} as const;           // total 15,200 chars ≈ 3.8k tokens — FIXED, independent of data volume

export async function buildActionQueueContext(clientId: string): Promise<string> {
  const [brief, deltas, openExps, topAnoms, playbook, lastWeek] = await Promise.all([
    getBriefForClient(clientId),
    weeklyDeltaBlock(clientId),          // §7.2 deltas, but across ALL enabled modules
    db.select().from(experiments).where(and(
      eq(experiments.clientId, clientId), isNull(experiments.verdict))).limit(6),
    db.select().from(anomalies).where(and(
      eq(anomalies.clientId, clientId), eq(anomalies.status, "open")))
      .orderBy(desc(anomalies.magnitude)).limit(5),
    playbookForClient(clientId),
    db.select().from(actionQueueItems).where(and(
      eq(actionQueueItems.clientId, clientId),
      inArray(actionQueueItems.status, ["skipped", "rejected"]),
      gte(actionQueueItems.createdAt, daysAgo(14)))).limit(5),
  ]);

  return [
    "== CLIENT BRIEF (L3) ==",
    clamp(brief?.bodyMd ?? "No brief yet — propose conservative, data-gathering actions.",
          CTX_BUDGET_CHARS.brief),
    "== THIS WEEK'S DELTAS (L1, last 7d vs trailing 28d weekly avg) ==",
    clamp(deltas, CTX_BUDGET_CHARS.deltas),
    "== OPEN EXPERIMENTS (do NOT propose overlapping changes) ==",
    clamp(openExps.map(fmtExperiment).join("\n") || "none", CTX_BUDGET_CHARS.experiments),
    "== TOP UNADDRESSED ANOMALIES ==",
    clamp(topAnoms.map(fmtAnomaly).join("\n") || "none", CTX_BUDGET_CHARS.anomalies),
    "== NICHE PLAYBOOK EXCERPT (L4, aggregate patterns) ==",
    clamp(playbook?.bodyMd ?? "none", CTX_BUDGET_CHARS.playbook),
    "== LAST WEEK'S SKIPPED/REJECTED ACTIONS (do not re-propose unchanged) ==",
    clamp(lastWeek.map((a) => `- ${a.title}: ${a.statusReason ?? "no reason"}`).join("\n") || "none",
          CTX_BUDGET_CHARS.skipped),
  ].join("\n\n");
}
```

### 9.2 Prompt

```
You are Seekly's weekly growth planner for "${clientName}". Using ONLY the context below,
propose exactly the FIVE highest-impact actions available THIS WEEK, ranked 1 (highest) to 5.

${context}

Rules for each action:
- rationale must point at evidence: reference brief bullets, delta numbers, anomaly ids,
  or playbook lines. No evidence, no action.
- expected_impact: the metrics_daily metric it should move ("<module>.<key>"), direction,
  and an honest magnitude guess (e.g. "+10-20% over 4 weeks").
- effort: low (config tweak) | medium (needs copy/setup) | high (multi-week initiative).
- execution_path:
  - "auto": a small reversible config change within existing engine settings.
  - "approval": campaigns, new outreach, copy changes, anything customer-visible in a new way.
  - "human": requires the owner or Seekly ops to act outside the platform.
- proposal.kind and fields per the schema. For config changes, config_delta must use real
  workflow_config setting keys with concrete values.
- Never propose changes overlapping an open experiment's module+setting.
- Prefer actions that open clean experiments (one variable, measurable in metrics_daily).
```

### 9.3 Zod schema

```ts
const actionSchema = z.object({
  title: z.string().max(90),
  rationale: z.string().max(350),
  expected_impact: z.object({
    metric: z.string().max(60),                 // "<module>.<key>"
    direction: z.enum(["up", "down"]),
    magnitude_guess: z.string().max(60),
  }),
  effort: z.enum(["low", "medium", "high"]),
  execution_path: z.enum(["auto", "approval", "human"]),
  module: z.enum(["review_automation", "speed_to_lead", "reactivation",
                  "content_engine", "directory_sync", "competitor_intel",
                  "social_syndication", "intelligence"]),
  proposal: z.object({
    kind: z.enum(["config_change", "campaign", "content_topic", "ops_task", "other"]),
    config_delta: z.record(z.string(), z.unknown()).optional(),
    campaign_angle: z.string().max(200).optional(),
    details: z.string().max(400),
  }),
  source_insight_ids: z.array(z.string()).max(3),
});
export const ActionQueueResponse = z.object({ actions: z.array(actionSchema).length(5) });
```

Call: `claude-sonnet-4-6`, `max_tokens: 2500`, `output_config` JSON-schema mirror of the
above **without** the exact-length constraint (structured outputs doesn't support complex
array constraints) — the `.length(5)` is enforced by zod; on the first failure re-prompt
once with `"Return exactly 5 actions."`, then fail the job (retry policy takes over).

### 9.4 Safety triage — server-side, the model's `execution_path` is only a hint

```
triage(action):
  model says "human"                        -> pending_approval routed to ops (never dropped)
  model says "approval"                     -> pending_approval
  model says "auto":
    qualifiesForAuto(action) === true       -> AUTO path
    else                                    -> pending_approval, statusReason "failed auto triage: <why>"
```

`qualifiesForAuto` — ALL must hold:

1. `AUTO_EXECUTE_ENABLED` (new optional env flag, default **false** — weeks 7–8 run
   approval-mode only per doc 09; flip per deployment after ≥3 trusted weekly cycles).
2. `proposal.kind === "config_change"` with a non-empty `config_delta`.
3. Every `(module, key)` pair is in the allowlist AND every value is within bounds:

| module | setting key | allowed values / bounds | doc 03 default |
|---|---|---|---|
| `review_automation` | `send_delay_hours` | integer 1–8 | 2h |
| `review_automation` | `daily_send_cap` | integer 10–40 | 25 |
| `review_automation` | `reask_cooldown_days` | integer 60–180 | 90 |
| `speed_to_lead` | `followup_touches` | integer 3–7 | 5 |
| `speed_to_lead` | `followup_schedule_days` | array, subset of 0–14, length 3–7 | [0,1,3,5,7] |
| `content_engine` | `topic_queue_add` | array of ≤3 strings ≤120 chars (additive only) | — |
| `social_syndication` | `gbp_post_cadence_per_month` | integer 2–8 | 4 |
| `social_syndication` | `posting_window` | one of "morning"\|"midday"\|"evening" | — |

4. The target module's switch is enabled for the client.
5. No open experiment already touches the same `(module, key)`.
6. Fewer than **2** auto-executions for this client in the trailing 7 days (`MAX_AUTO_PER_WEEK = 2`).

**Never auto — hard denylist regardless of allowlist evolution:** anything that initiates
messages to a new audience (campaign launches, list imports), spend/pricing/offer changes,
review-response tone/template changes, deleting or disabling anything, quiet-hours or
frequency-cap changes (compliance settings are ops-only), directory/NAP data changes.

### 9.5 Execution

```ts
// AUTO path — jobs run inside the app, so call the spec-02 switchboard service directly
// (the same code the internal API route uses); do NOT loop back over HTTP.
async function executeAuto(item: ActionQueueItem): Promise<void> {
  const previous = await getWorkflowSettings(item.clientId, item.module);     // spec 02
  const experiment = await openExperiment({                                   // §10
    clientId: item.clientId, source: "action_queue",
    hypothesis: item.rationale,
    action: { module: item.module, configDelta: item.proposal.config_delta,
              previousConfig: pick(previous, Object.keys(item.proposal.config_delta)),
              sourceInsightIds: item.sourceInsightIds },
    expectedImpact: item.expectedImpact, measureDays: 28,
  });
  await updateWorkflowSettings(item.clientId, item.module, item.proposal.config_delta,
                               { updatedBy: "intelligence:action-queue" });   // spec 02
  await db.update(actionQueueItems)
    .set({ status: "auto_executed", experimentId: experiment.id, decidedAt: new Date() })
    .where(eq(actionQueueItems.id, item.id));
  await writeActivity(item.clientId, "action.auto_executed", { itemId: item.id, experimentId: experiment.id });
}
```

- **approval path:** row stays `pending_approval`. Surfaced in (a) the portal/admin
  Approvals queue (spec 05/portal work — reads `action_queue_items` where status
  `pending_approval`) and (b) the weekly ops digest email. The approval route handler calls
  the same `executeAuto`-shaped function (`approveAction(itemId, approver)`) → status
  `approved` + experiment; or `rejectAction(itemId, reason)` → status `rejected` +
  `statusReason` (feeds next week's context, §9.1).
- **stale sweep:** items still `proposed`/`pending_approval` when the next weekly run starts
  → status `skipped`, `statusReason: 'expired_unactioned'`.

**Acceptance (INT-8):** with `AUTO_EXECUTE_ENABLED=false` nothing ever writes
`workflow_config` (test); out-of-bounds `config_delta` fails triage with reason; exactly 5
rows per client per week with unique ranks; skipped actions appear in the next week's
context block; every executed action has an experiment row.

---

## 10. INT-7 — Experiment lifecycle (`src/server/intelligence/experiments.ts`)

### 10.1 Open / measure / close

```ts
export async function openExperiment(input: {
  clientId: string; source: ExperimentSource; hypothesis: string;
  action: ExperimentAction; expectedImpact: ExpectedImpact; measureDays?: number;
}): Promise<{ id: string }> {
  const days = input.measureDays ?? 28;
  const before = await metricWindowMean(input.clientId, input.expectedImpact.metric,
                                        /* from */ -28, /* to */ 0);   // trailing 28d BEFORE start
  const [row] = await db.insert(experiments).values({
    ...input,
    measureUntil: new Date(Date.now() + days * 86_400_000),
    outcome: { metricBefore: before },        // partial; completed at close
  }).returning({ id: experiments.id });
  await writeActivity(input.clientId, "experiment.opened", { id: row.id });
  return row;
}

/** metric = "<module>.<key>"; mean of the daily value over the window from metrics_daily. */
async function metricWindowMean(clientId: string, metric: string,
                                fromDayOffset: number, toDayOffset: number): Promise<number | null>;
```

`experiment-closer-daily` handler (`closeExperiment(experimentId)`):

```ts
const after = await metricWindowMean(clientId, metric,
  /* startedAt..measureUntil expressed as day offsets */);
const before = outcome.metricBefore;

// Verdict computation — deterministic, no LLM:
const MIN_REL_DELTA = 0.10;                       // ±10% band = flat
if (before === null || after === null) verdict = "flat";  // insufficient data ⇒ never a fake win
else {
  const relDelta = (after - before) / Math.max(Math.abs(before), 1e-9);
  const signed = expectedImpact.direction === "down" ? -relDelta : relDelta;
  verdict = signed >= MIN_REL_DELTA ? "win" : signed <= -MIN_REL_DELTA ? "loss" : "flat";
}
// revenueEst: only when action.campaignId is set — read campaigns.stats.revenue_est; else null.
```

Update row: `outcome = {metricBefore, metricAfter, relDelta, revenueEst, verdict}`,
`verdict` column set. **Never reopen** a closed experiment.

### 10.2 Write-back to insight confidence (the learning step)

Deterministic — no LLM writes here:

```ts
// 1. Write the learned L2 note (source 'experiment_closer'):
const finding =
  verdict === "win"
    ? `Proven: ${hypothesis} — ${metric} moved ${pct(relDelta)} over ${days}d (experiment).`
    : verdict === "loss"
      ? `Anti-pattern: ${hypothesis} — ${metric} moved ${pct(relDelta)} AGAINST expectation. Do not retry unchanged.`
      : `No effect: ${hypothesis} — ${metric} within ±10% band.`;
const [note] = await db.insert(insights).values({
  clientId, domain: domainForModule(action.module),
  finding: finding.slice(0, 600),
  evidence: [{ kind: "experiment", ref: `experiment:${id}`, value: pct(relDelta) }],
  confidence: verdict === "win" ? 0.85 : verdict === "loss" ? 0.75 : 0.4,
  source: "experiment_closer",
  embedding: await embed(finding),
}).returning();
await db.update(experiments).set({ learnedNoteId: note.id }).where(eq(experiments.id, id));

// 2. Adjust the confidence of the source notes that motivated the action:
for (const noteId of action.sourceInsightIds ?? []) {
  const delta = verdict === "win" ? +0.10 : verdict === "loss" ? -0.15 : 0;
  if (!delta) continue;
  await db.update(insights)
    .set({ confidence: sql`least(0.95, greatest(0.05, ${insights.confidence} + ${delta}))` })
    .where(and(eq(insights.id, noteId), eq(insights.clientId, clientId)));
}
// 3. Any active insight whose confidence fell below 0.2 -> status 'refuted'.
// 4. Major event: if verdict !== 'flat' && |relDelta| >= 0.25 -> enqueueBriefRefresh(clientId, 'experiment_verdict').
```

**Acceptance (INT-7):** verdict unit tests (win/loss/flat, direction "down" metrics like
`leads.median_response_s`, null-data ⇒ flat); confidence clamp respected; learned note
retrievable via `getInsights`; brief refresh enqueued exactly once per major verdict.

---

## 11. INT-9 — `playbook-aggregate-monthly` (`src/server/intelligence/playbooks.ts`)

Per niche (`client_profile.industry` normalized via a fixed `NICHE_MAP`, e.g.
`golf_recreation`, `home_services`). Eligibility gate: **≥2 active clients in the niche AND
≥1 closed experiment each** (doc 09 "Month 3+" condition) — otherwise skip with an activity
log entry.

### 11.1 Anonymization — in code, before the prompt

```ts
// Label clients deterministically within the run: Client A, Client B, ...
// Every input line is scrubbed BEFORE prompt assembly:
//  - replace each client's business name, slug, and website host with its label
//  - round metric deltas to 5-point buckets ("+23%" -> "+20-25%")
//  - strip evidence ids (playbooks carry patterns, not pointers)
function anonymize(text: string, nameMap: Map<string, string>): string;
```

Inputs per client: closed experiments (hypothesis, module, verdict, bucketed relDelta) and
active insights with `confidence ≥ 0.6` (finding only, scrubbed).

### 11.2 Prompt (privacy rule stated in-prompt AND enforced by validator)

```
You maintain the ${niche} PLAYBOOK for the Seekly platform — aggregate patterns that make
every new ${niche} client start smarter. HARD LIMIT: 1000 tokens.

PRIVACY RULE (absolute): the playbook must contain ZERO client-identifiable information.
No business names, no city/venue names, no exact revenue figures, no phone numbers, no
URLs. Refer to sources only as "one client" / "two of three clients". If a pattern cannot
be stated without identifying a client, omit it.

Structure: ## Proven angles / ## Anti-patterns / ## Defaults for new clients
(each bullet: the pattern + strength of evidence, e.g. "won in 2/3 clients").
Only include patterns supported by at least one closed experiment or two independent
high-confidence insights.

INPUTS (already anonymized to Client A/B/C):
${anonymizedBlock}

PREVIOUS PLAYBOOK VERSION:
${previousBodyMd || "none"}

Return ONLY the markdown playbook.
```

`claude-sonnet-4-6`, `max_tokens: 1200`, plain text, same output-token check pattern as the
brief (cap 1000, one compress retry).

### 11.3 Output validator — privacy enforced mechanically

```ts
function validatePlaybookPrivacy(bodyMd: string, clientsInNiche: ClientFacts[]): string[] {
  const violations: string[] = [];
  const lower = bodyMd.toLowerCase();
  for (const c of clientsInNiche) {
    for (const needle of [c.name, c.slug, hostOf(c.website), c.contactName, c.napPhone]
        .filter((s): s is string => !!s && s.length >= 4)) {
      if (lower.includes(needle.toLowerCase())) violations.push(`client string: ${needle}`);
    }
  }
  if (/\+?1?[\s.-]?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}/.test(bodyMd)) violations.push("phone number");
  if (/[\w.+-]+@[\w-]+\.[\w.]+/.test(bodyMd)) violations.push("email address");
  if (/https?:\/\//.test(bodyMd)) violations.push("URL");
  if (/\$\s?\d{3,}/.test(bodyMd)) violations.push("exact dollar figure");
  return violations;
}
```

On violations: retry once appending
`"Your draft violated the privacy rule (${violations}). Remove all such content."`;
if it still violates, **fail the job** (dead-letter + ops alert). A leaked playbook is worse
than a missing one. On pass: insert `playbooks` row `version = prev + 1`,
`supportingExperiments = count`.

**Acceptance (INT-9):** validator unit tests (each violation class caught; clean text
passes); job skips below the 2-client gate; new clients' onboarding defaults (spec 02) can
read the latest playbook version per niche.

---

## 12. INT-10 — Hardening & observability (week 8)

- **Idempotency drill:** for every queue, send the same singleton payload twice and assert
  single execution/artifact (mirrors doc 07 "replaying any event produces zero duplicates").
- **Budget drill:** set a client's `weeklyBudgetUsd` to 0.01, run the weekly coordinators,
  assert every LLM job skips with `intelligence.skipped_budget` activity rows and zero
  Anthropic calls (assert in mock mode via a spy).
- **Stuck-job reaping:** intel jobs rely on pg-boss `expireInSeconds` (set on every
  send/insert above) — verify expired jobs land in the `-dlq` queues; add the dlq depths to
  the existing ops digest.
- **Dashboards:** admin page section reading `intelligence_costs` (spend per client per
  job, trailing 30d) and `activity_log` intelligence rows (runs/failures/skips per day).
- **Lint guard:** ESLint `no-restricted-imports` rule scoped to `src/server/intelligence/
  {rollup,anomalies,experiments}.ts` banning `@anthropic-ai/sdk` — L1 stays AI-free forever.

---

## 13. Out of scope (tracked elsewhere)

- Revenue Leakage Report v1 and health-score HubSpot sync (doc 09 "also produced") — they
  are *readers* of `metrics_daily`/`anomalies` built with the reports stack
  (`src/server/reports/`); spec separately once INT-2/INT-3 land.
- Portal Approvals UI + Opportunities feed rendering of `action_queue_items` — portal spec.
- WF-6 espionage writing `competitive` insights — WF-6's n8n spec calls a small internal
  API route (`POST /api/internal/insights`) that wraps the same `db.insert(insights)` used
  in §7.5 with `source: 'wf6_digest'`.
