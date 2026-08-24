# Cost & Performance Summary

## 1. Model pricing used in this model (per million tokens)

| Model | Role in this design | Input | Output | Cache write (5m / 1h) | Cache read |
|---|---|---:|---:|---:|---:|
| Claude Sonnet 5 | General-purpose answering | $3.00¹ | $15.00¹ | $3.75 / $6.00 | $0.30 |
| Claude Haiku 4.5 | Query classification, simple FAQ tier | $1.00 | $5.00 | $1.25 / $2.00 | $0.10 |
| Claude Opus 4.8 | Fallback if a hard-guaranteed-throughput (Priority Tier) SLA is required later | $5.00 | $25.00 | $6.25 / $10.00 | $0.50 |

¹ Sonnet 5 carries an introductory rate of $2.00 / $10.00 per MTok through **2026-08-31**;
this table uses the standard post-introductory rate ($3.00 / $15.00) since a
production rollout should be costed against the rate it will actually run at, not a
short-lived promotional window. *(Claude Opus 5 and Sonnet 5 are currently excluded from
Priority Tier — see rationale doc — hence Opus 4.8 as the fallback for that specific
requirement, not Opus 5.)*

## 2. Per-query cost model (illustrative)

Assumes a ~1,200-token shared system+tools prefix (cached) and typical per-query token
counts:

| Query type | Effort | Input (uncached) | Cache read | Output | Est. cost |
|---|---|---:|---:|---:|---:|
| Simple lookup, cache hit (e.g. "What's the PTO policy?") | low/medium | ~150 tok | ~1,200 tok | ~250 tok | **~$0.0045** |
| Simple lookup, cache miss (first request after idle) | low/medium | ~1,350 tok | 0 (writes cache) | ~250 tok | ~$0.0090 |
| Complex multi-hop (2–3 tool calls, synthesis) | high/xhigh | ~500 tok | ~1,200 tok | ~900 tok | **~$0.0157** |

At steady state (cache warm), a simple cached query costs roughly **half** of an
uncached one — this is the concrete effect of the caching strategy in §5 of the design
review, not just a general claim.

## 3. Aggregate cost projection (illustrative enterprise scale)

Assumptions: 2,000 employees, ~3 queries/employee/week, 80% simple-tier (Haiku-routed or
low-effort Sonnet), 20% complex-tier (high-effort Sonnet):

| | Volume/month | Avg cost/query | Monthly cost |
|---|---:|---:|---:|
| Simple tier | ~19,200 | ~$0.003 | ~$58 |
| Complex tier | ~4,800 | ~$0.016 | ~$77 |
| **Total** | **24,000** | — | **~$135/month** |

This is deliberately conservative (illustrative token counts, not a live measurement) —
its purpose is to show the *shape* of the cost model (cache-driven, effort-tiered,
volume-linear) that a real deployment would calibrate with production telemetry, not to
serve as a binding quote.

## 4. Performance & monitoring targets

| Metric | Target | Why it's monitored |
|---|---|---|
| Token-level cache hit rate | > 70% during business hours | Directly measures whether the caching strategy (§5) is working; a drop flags an accidental cache-busting change (e.g. a prompt edit, a timestamp added upstream). |
| P95 end-to-end latency | < 4s for single-tool queries, < 10s for multi-tool | User-facing responsiveness target for a chat-style internal tool. |
| Tool error rate (`is_error` results) | < 1% per tool | An internal system (DB, ticketing) failing silently is worse than the assistant visibly saying it can't answer. |
| Cost per query (rolling 7-day avg) | Tracked against the projection in §3, alert on >25% deviation | Catches cost regressions (e.g. effort routing misclassifying queries as complex) before the monthly bill does. |
| Escalation/human-handoff rate | Tracked as a trend, no fixed target | Rising escalation rate signals a knowledge-base gap or a classification issue worth investigating. |
| Extended-thinking usage rate (% of queries at high/xhigh effort) | Tracked against the assumed 20% complex-tier split | Confirms the effort router is behaving as designed rather than drifting toward always-high-effort (cost risk) or always-low-effort (quality risk). |
