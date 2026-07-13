# 09 — Seekly Intelligence Platform

The Intelligence Engine is what makes Seekly a *growth system* rather than seven disconnected
automations: competitor monitoring, attribution, opportunity scoring, demand patterns, and the
weekly **Growth Action Queue** — the loop from the brainstorm:

> **Observe → Identify opportunity → Execute → Measure revenue → Learn → Reallocate effort**

## The core design problem

Naive approach: dump activity logs, transcripts, and metrics into an LLM context and ask
"what should we do?" That fails three ways: context floods (costs explode, quality degrades),
nothing is *retained* between runs (no learning), and conclusions can't be audited.

Seekly instead uses a **layered memory hierarchy**. Each layer compresses the one below it on a
schedule. Agents only ever read the top layers; everything below is reachable through scoped
query tools, never by dumping.

```
L4  NICHE PLAYBOOKS      cross-client patterns per vertical      (~1k tokens each)
     ▲ aggregated monthly
L3  CLIENT INTELLIGENCE  one living brief per client — THE       (hard cap ~1.5k tokens)
    BRIEF                 base context every agent loads
     ▲ regenerated monthly + on major events
L2  INSIGHT NOTES        AI-written, structured, per domain,     (~200 tokens each,
                          versioned, with confidence + evidence    embedded for retrieval)
     ▲ weekly jobs read rollups, write notes
L1  DETERMINISTIC        metrics_daily + SQL rollups — exact     (never enters context
    ROLLUPS               numbers, zero AI                         wholesale; queried via tools)
     ▲ nightly aggregation
L0  RAW FACTS            events, messages, conversations,        (agents never read this,
                          activity_log, reviews                    except the single active
                                                                   conversation thread)
```

### Anti-flooding rules (hard constraints)

1. **Agents never read raw logs.** An agent gets: the client brief (L3) + task-relevant insight
   notes (L2, retrieved by domain/embedding) + **query tools** against L1 for exact numbers
   ("median lead response time, last 30d" returns one number, not rows).
2. **Every AI-written artifact has a token budget and a schema.** Insight notes ~200 tokens,
   briefs ~1.5k hard cap. Regeneration must *compress or drop* — the brief cannot grow
   unboundedly; superseded content falls back to L2 history.
3. **Compression is scheduled, not reactive:** nightly SQL rollups → weekly insight jobs →
   monthly brief refresh. Cost scales with number of clients, not number of events.
4. **Scoped retrieval by task:** the review-response agent pulls reputation-domain notes only;
   the reactivation composer pulls campaign-history notes only. Domain tags, not one big pile.
5. **Numbers stay numeric.** Trends, attribution, and scoring are SQL over L1. The LLM
   interprets and narrates; it is never the calculator or the database.

## How it learns over time (the experiment ledger)

Learning = structured outcomes, not accumulated prose.

Every meaningful action any engine takes can be framed as an **experiment**:

```
experiments
  id, client_id, source (action_queue|campaign|content|config_change|manual),
  hypothesis            "Birthday-party angle will outperform league-night for 90-180d lapsed"
  action                {module, config_delta or campaign_id}
  expected_impact       {metric: reactivation_bookings, direction: up, magnitude_guess}
  started_at, measure_until,
  outcome               {metric_before, metric_after, revenue_est, verdict: win|loss|flat}
  learned_note_id       → the L2 insight written when the experiment closes
```

The loop in practice:

1. **Observe** — nightly rollups (L1) + anomaly detection (pure SQL/stats: review velocity
   drop, response-time spike, lead-source shift, seasonal pattern).
2. **Identify** — weekly insight jobs turn anomalies + rollups into L2 notes with confidence
   and evidence pointers.
3. **Execute** — the Growth Action Queue (below) turns notes into ranked actions; approved
   actions open experiment records and change engine configs / launch campaigns.
4. **Measure revenue** — attribution already built into every engine (UTM links, conversation
   outcomes, campaign booking counts, `revenue_est`) closes the experiment with real numbers.
5. **Learn** — experiment verdicts write back: the L2 note's confidence rises or falls; the
   monthly brief refresh promotes durable findings ("what works for this client"); losses are
   retained as anti-patterns so they're not retried.
6. **Reallocate** — next queue generation reads the updated brief: proven angles get budget,
   losers get dropped. Effort follows measured revenue, mechanically.

Because outcomes are rows, "what have we learned about this client's revenue?" is a query —
and the brief is just its narrated, compressed projection.

## WF-8 · Growth Action Queue (weekly master workflow)

**Switch:** `intelligence` (client-facing digest optional; internal always-on once active).

**Flow (per client, weekly):**
1. Load: client brief (L3) + this week's rollup deltas (L1 summary) + open experiments +
   top unaddressed anomalies + relevant niche playbook (L4). Total context: ~3-4k tokens, fixed.
2. AI proposes the **five highest-impact actions available this week**, each with: rationale
   (pointing at evidence), expected revenue impact, effort class, and execution path
   (auto-executable config/campaign vs. needs-approval vs. needs-human).
3. Safety triage: auto-executable + low-risk (e.g., shift review-request send time, add
   follow-up touch, refresh GBP post cadence) → executed immediately as experiments.
   Everything else → approval queue (Seekly ops or client per approval prefs).
4. Queue rendered in portal (Overview → Opportunities) + weekly ops digest to Seekly.
5. Every executed action = experiment record; every skipped action logged with reason
   (feeds next week's ranking).

**Also produced by the intelligence layer:**
- **Revenue Leakage Report (monthly):** opportunity scoring made client-visible — quantified
  found-money list (unanswered leads, lapsed customers, review gaps, weak pages) with what
  Seekly recovered vs. what remains. Doubles as prospect-mode sales artifact.
- **Demand patterns:** seasonal/weekly curves from L1 powering send-time and content-calendar
  defaults per client and per niche.
- **Health score:** composite from rollups (synced to HubSpot summary fields; drives churn
  attention).

## L4 — Niche playbooks (the compounding moat)

Monthly cross-client job per vertical: reads closed experiments + high-confidence insights
across all clients in a niche → writes/updates the playbook ("golf venues: reactivation reply
rate 3× higher Tue–Thu; birthday angle beats league-night for 90–180d lapsed; review requests
convert best 2–4h post-visit").

- **Privacy rule:** playbooks contain aggregate patterns only — no client-identifiable data,
  no client names, no raw numbers traceable to one business.
- New clients onboard with playbook-derived defaults instead of generic guesses — every client
  makes the next one smarter, which is the agency's structural advantage over point tools.

## Storage additions (extends 04-data-model.md)

```sql
insights            -- L2
  id, client_id, domain (reputation|leads|reactivation|content|visibility|competitive),
  finding, evidence jsonb (metric refs, event samples), confidence (0-1),
  status (active|superseded|refuted), embedding vector, created_at, superseded_by

client_briefs       -- L3, versioned
  id, client_id, version, body_md (≤1.5k tokens), generated_at, inputs jsonb (note ids)

experiments         -- the learning ledger (schema above)

playbooks           -- L4
  id, niche, version, body_md, supporting_experiments int, updated_at

anomalies           -- detector output queue
  id, client_id, metric, direction, magnitude, window, detected_at, status
```

## How other engines consume intelligence

- **WF-2 Speed-to-Lead / WF-3 Reactivation composers:** load brief + domain notes → better
  qualification emphasis and proven campaign angles (never raw transcripts of other customers).
- **WF-4 Content Engine:** topic queue replenished from competitive gaps (WF-6 snapshots) +
  demand patterns + playbook topics.
- **WF-6 Espionage digests:** findings land as `competitive`-domain L2 notes, so the Action
  Queue can respond to competitor moves ("counter their $25 league night") — the digest is for
  humans, the notes are for the system.

## Build sequencing

Intelligence rides on data the other engines generate, so it phases in:
- **Weeks 1–4 (with the platform build):** L0/L1 exist by construction (activity log,
  metrics_daily, attribution fields) — *this is why every engine logs everything from day one.*
- **Weeks 5–6:** anomaly detector + L2 insight jobs + first client briefs (pilot data is rich
  enough by then); Revenue Leakage Report v1 for the pilot conversion pitch.
- **Weeks 7–8:** WF-8 Growth Action Queue (approval-mode only at first; auto-execution after
  a few trusted cycles) + experiment ledger wired into campaigns/config changes.
- **Month 3+:** niche playbooks once ≥2 clients in a vertical have closed experiments.
