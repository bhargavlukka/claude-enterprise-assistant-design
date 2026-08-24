# Claude Solution Architecture Review — Enterprise Knowledge Assistant

A production-ready solution architecture for an enterprise AI knowledge assistant built on
the Anthropic Claude platform. Built for the "Claude Solution Architecture Review" lab
(Module 2, The Anthropic Claude Platform).

This repo is a **design review**, not a running application — it addresses every point
required by the brief: solution overview, Messages API usage, tool integration strategy,
extended thinking policy, prompt caching strategy, performance monitoring, and cost
optimization, plus a rationale for each major decision and a cost/performance summary.

## Contents

- [`docs/design-review.md`](docs/design-review.md) — the 1–2 page design review document
- [`docs/rationale.md`](docs/rationale.md) — rationale for each major architectural decision
- [`docs/cost-performance-summary.md`](docs/cost-performance-summary.md) — cost model and monitored metrics
- Architecture diagram — below, and duplicated at the top of the design review

## Architecture

```mermaid
flowchart TD
    subgraph Clients
        WEB[Web chat UI]
        SLACK[Slack app]
    end

    WEB --> ORCH
    SLACK --> ORCH

    subgraph Orchestrator["Backend orchestrator (self-hosted, in enterprise VPC)"]
        ORCH["Agent loop\nclient.messages.create()\ncache_control on system+tools\nthinking: adaptive, effort routed per query\noutput_config.format: structured JSON"]
        ROUTER["Effort/model router\n(query complexity -> Haiku 4.5 vs Sonnet 5,\nlow/medium vs high/xhigh effort)"]
        LOG["Structured logging\ntokens, cache stats, latency,\ntool errors -> dashboards + alerts"]
    end

    ORCH --> ROUTER
    ORCH --> LOG

    ORCH -->|tool_use| KS["knowledge_search\n(custom tool)"]
    ORCH -->|tool_use| DB["employee_db_query\n(custom tool)"]
    ORCH -->|tool_use| TIX["ticket_system_api\n(custom tool)"]
    ORCH -->|hosted tool| WS["web_search\n(Anthropic server-side,\nallowed_domains restricted)"]

    KS --> VDB[("Vector index\nwiki / Confluence / SharePoint")]
    DB --> SQL[("Internal SQL DB\nHR / product / policy tables")]
    TIX --> TICKETS[("Internal ticketing system\ne.g. ServiceNow / Jira")]
    WS --> PUBWEB["Approved public sources\n(allowlisted domains)"]

    ORCH --> ANS["Structured JSON answer\nanswer, sources, confidence, escalate"]
    ANS --> WEB
    ANS --> SLACK
```

See [`docs/design-review.md`](docs/design-review.md) for the full narrative behind this
diagram, and [`docs/rationale.md`](docs/rationale.md) for why each box is there instead of
an alternative.

## Evaluation criteria mapping

| Criterion | Where it's addressed |
|---|---|
| Technical accuracy | Model pricing, thinking/effort behavior, caching mechanics, and Priority Tier eligibility in this design reflect the current Claude API (not deprecated patterns like fixed `budget_tokens`) — see `docs/cost-performance-summary.md` footnotes and `docs/rationale.md` |
| Understanding of Claude capabilities | `docs/design-review.md` §2–5 (Messages API usage, tools, extended thinking, caching) |
| Quality of architectural decisions | `docs/rationale.md` — alternative(s) considered for every major decision |
| Cost and performance considerations | `docs/cost-performance-summary.md` — pricing, per-query cost model, aggregate projection, monitoring targets |
| Clarity of communication | Design review kept to ~1–2 pages; supporting detail kept in separate, clearly-scoped files rather than one long document |
