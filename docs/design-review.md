# Claude Solution Architecture Review
## Enterprise AI Knowledge Assistant

**Module 2 | The Anthropic Claude Platform — Design Review**

---

## 1. Solution Overview

This design proposes **"Ask"**, an internal knowledge assistant that lets employees ask
natural-language questions and get grounded, cited answers pulled from the company's own
systems — internal wikis/Confluence, a structured HR/product database, and an internal
ticketing system — with a fallback to public web search for questions those systems can't
answer. It is exposed through a web chat UI and a Slack app, both calling a single backend
orchestrator.

The core design goal is **production-readiness at enterprise scale**: correct, cited
answers; predictable and controllable per-query cost; and operational visibility (latency,
spend, cache effectiveness, tool reliability) from day one — not bolted on after launch.

## 2. Messages API Usage

The orchestrator is a thin backend service that calls Claude directly via the official
Anthropic SDK's Messages API (`POST /v1/messages`) — not Managed Agents. This is a
deliberate choice (see rationale doc): the assistant's tools all reach into the company's
own private network (internal wiki index, internal database, internal ticketing API), so
the tool-execution environment needs to run inside the company's own VPC, under the
company's own auth, logging, and data-residency controls. Anthropic's Messages API keeps
Anthropic strictly in the "generate the next turn" role; the enterprise keeps full custody
of the data plane.

Each turn follows the standard structure: a stable `system` block (assistant persona,
grounding instructions, tool-use policy, output-format contract) with `cache_control` on
it, a `tools` array (below), and the running `messages` history. The service uses the
SDK's `messages.create` in a manual agent loop with a max-iteration cap, since this gives
full control over tool-execution auditing (every tool call and result is logged, keyed to
the requesting employee's ID, before it's returned to Claude) — a requirement for an
internal system that touches HR and ticketing data.

## 3. Tool Integration Strategy

Four tools cover the required capability surface, each justified by a distinct real need:

| Tool | Type | Purpose |
|---|---|---|
| `knowledge_search` | Custom (client-side) | Retrieval-augmented lookup against a vector index of internal wiki/Confluence/SharePoint content — the primary source for "how do I…" and policy questions. |
| `employee_db_query` | Custom (client-side) | Read-only, parameterized lookups against structured internal data (org chart, product catalog, policy tables) — for questions a vector search answers poorly (exact facts, counts, lookups by ID). |
| `ticket_system_api` | Custom (client-side) | Calls the internal ticketing system (e.g. ServiceNow/Jira) to check ticket status or open a new one — turns the assistant into an action-taker, not just a Q&A bot. |
| `web_search` | Anthropic server-side (`web_search_20260209`), `allowed_domains`-restricted | Public information (e.g. a vendor's public docs) that isn't in any internal system, restricted to an approved domain allowlist to keep answers on-brand and avoid ungoverned public browsing. |

All four are declared with `strict: true` JSON schemas, so a malformed tool call is
rejected before it reaches internal systems rather than causing a downstream error.

## 4. Extended Thinking

Extended (adaptive) thinking is enabled on every request (`thinking: {"type":
"adaptive"}`), but **`output_config.effort` is set dynamically per query**, not fixed:

- **`low`/`medium`** for single-lookup questions ("What's the office Wi-Fi password
  policy?") — most employee traffic, and the cost/latency-sensitive majority case.
- **`high`/`xhigh`** for multi-hop questions that require synthesizing across
  systems ("Which of my team's open tickets are blocking the Q3 roadmap items assigned to
  me?") — genuinely benefits from more deliberate reasoning across several tool calls.

A lightweight upfront classifier (a lower-cost model can suffice) or a simple heuristic on
query length/tool-fanout expectation routes each request to the right effort tier. Running
every query at `high`/`xhigh` would materially inflate cost at enterprise query volume for
no quality benefit on the simple majority.

## 5. Prompt Caching Strategy

The system prompt and the four tool definitions are byte-identical across every request
from every employee, so a single `cache_control: {"type": "ephemeral"}` breakpoint on the
last system block caches both together (request render order is `tools → system →
messages`). At enterprise query volume, this prefix is re-read on nearly every request,
so:

- The **1-hour cache TTL** variant is used over the default 5-minute one — enterprise
  traffic is roughly continuous during business hours, so a 5-minute TTL would force
  repeated cache-write turns; the 1-hour extension keeps the shared prefix warm across
  normal request gaps.
- `usage.cache_read_input_tokens` / `cache_creation_input_tokens` are logged on every call
  (see §6) so cache effectiveness is a monitored metric, not an assumption.
- Per-department or per-role system-prompt variants (if introduced later) should share the
  longest possible common prefix, with role-specific instructions appended *after* the
  cache breakpoint — inserting them earlier would fragment the cache across every role.

## 6. Performance Monitoring

Every request logs, keyed by request ID and employee ID:

- **Cost/tokens**: input, output, cache-read, cache-creation tokens; computed cost per
  request, rolled up to per-team and organization-wide daily spend.
- **Cache effectiveness**: request-level and token-level cache hit rate, tracked as a
  first-class dashboard metric with an alert if it drops below a target threshold
  (indicating an accidental cache-busting change, e.g. a timestamp added to the system
  prompt).
- **Latency**: P50/P95 end-to-end and per-tool-call latency, to catch a slow internal
  system before it degrades the whole assistant.
- **Tool reliability**: error rate and `is_error` tool-result rate per tool, since a
  silently-failing internal API is worse than a visibly-failing one.
- **Answer quality proxies**: confidence-level distribution (from the structured output),
  escalation/human-handoff rate, and thumbs-up/down feedback rate.

## 7. Cost Optimization

- **Prompt caching** (above) is the single largest lever — cache reads are priced at
  roughly a tenth of the base input rate, and the shared system+tools prefix dominates
  input tokens on short employee questions.
- **Effort tuning per query** (§4) avoids paying for deep reasoning on simple lookups.
- **Model tiering**: a smaller/cheaper model (e.g. Claude Haiku 4.5) handles query
  classification and simple FAQ-style answers where the wiki lookup alone is sufficient;
  Claude Sonnet 5 handles the general case. This mirrors a router pattern rather than
  sending every request to the most capable (and most expensive) model by default.
- **Batching** for non-interactive workloads — e.g. nightly re-indexing summaries or
  bulk document classification for the knowledge base — via the Message Batches API,
  which runs at roughly half the synchronous price for latency-insensitive work.
- **Structured output** constrains response length and shape, avoiding the token waste of
  free-form, overly verbose answers.

See `cost-performance-summary.md` for the concrete cost model and monitored metrics table.
