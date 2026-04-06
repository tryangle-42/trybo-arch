# Platform Completion Status

> Ideal long-term state definition for all 11 platform modules, measured against what a production-grade agentic personal assistant requires — not what was scoped for any single sprint.
>
> Based on: Architecture vision (`0-main.md`), memory spec (`9-memory-architecture.md`), codebase at `/Users/avinash.nagar/Documents/git/trybot_api`, DB schema (`docs/entire-schema.sql`).
>
> Last updated: 2026-03-30

---

## Summary

| # | Module | Completion | Ideal Components | Built (approx) |
|---|--------|-----------|------------------|----------------|
| 1 | Core Platform | **~29%** | 14 | ~4 |
| 2 | Skill Management | **~17%** | 12 | ~2 |
| 3 | Task Decomposition | **~33%** | 12 | ~4 |
| 4 | Skill Ranking & Selection | **~5%** | 11 | ~0.5 |
| 5 | Execution Engine | **~62%** | 17 | ~10.5 |
| 6 | New Skill Creation (On the Fly) | **0%** | 10 | 0 |
| 7 | User Memory | **~20%** | 14 | ~3 |
| 8 | Agent Intelligence | **~16%** | 14 | ~2.25 |
| 9 | Front-End / UX | **~25%** | 16 | ~4 |
| 10 | Communication Skills | **~44%** | 16 | ~7 |
| 11 | Security, Privacy, Consent | **~34%** | 19 | ~6.5 |

**Overall platform: ~28%** (43.75 / 155 components) — execution engine and communication channels are solid; intelligence, memory, and operational maturity layers are the gap.

> Percentages are first-pass estimates against the ideal state. Open for calibration.

---

## Module 1: Core Platform (Orchestration Layer)

The backbone — request handling, adapter system, graph execution pipeline.

### Ideal State (14 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Request Pipeline** | Unified flow: GID → Planner → Executor → Finalize. Same pipeline for chat-triggered, scheduled, and webhook-triggered requests. |
| 2 | **Adapter System** | Pluggable architecture supporting Direct, MCP, Marketplace, Agent adapter types. Single `invoke()` interface. |
| 3 | **Config-Driven MCP** | Any MCP server addable via config entry + skill registration. No adapter code changes. Multiple MCP providers active. |
| 4 | **Adapter Health & Circuit Breakers** | Per-adapter health probes on schedule. Circuit breaker pattern — auto-degrade on repeated failures, auto-recover. Planner never sees degraded adapters. |
| 5 | **Multi-Model LLM Routing** | Route different LLM calls (planning vs synthesis vs extraction) to optimal models. Cost/quality tradeoff per call type. Model fallbacks. |
| 6 | **Rate Limiting & Backpressure** | Per-user and per-skill rate limits. Queue overflow handling. Fair scheduling across concurrent users. |
| 7 | **Horizontal Scalability** | Stateless request handling. No single-process bottlenecks. Distributed execution across workers. |
| 8 | **Caching Layer** | Cache idempotent skill outputs, LLM responses for identical prompts, adapter responses with TTL. Cache invalidation rules. |
| 9 | **Observability Stack** | Structured logging, distributed tracing (OpenTelemetry), per-skill/per-plan metrics, latency percentiles, cost tracking dashboards, alerting. |
| 10 | **Hot Configuration Reload** | Change adapter configs, feature flags, LLM model assignments without restart. |
| 11 | **Request Queuing** | Priority queue for concurrent requests. Background vs interactive priority levels. |
| 12 | **SSE Streaming** | Real-time execution progress to mobile clients per agent run. |
| 13 | **Graceful Degradation** | Defined fallback paths when primary services fail. Partial results over total failure. |
| 14 | **Multi-Tenant Foundation** | User isolation at query/data/execution level. Resource limits per user/tier. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Request Pipeline | **Partial** | Two-tier LangGraph compiler graph works. `agent_runs` table tracks execution state (status, approvals, clarifications). But GID is LLM-based (not semantic router). Pipeline works for chat-triggered and scheduled. Webhook inbound is limited. |
| Adapter System | **Done** | Direct (68+ handlers), MCP (Tavily only), Composio marketplace, Agent (Deal Hunter). Single `invoke()` interface. |
| Config-Driven MCP | **Partial** | Architecture supports it, but only Tavily is actually connected via MCP. Config exists for others (Google Workspace, Slack, etc.) but untested/unused. |
| Adapter Health | **Not built** | No health probes, no circuit breakers. If Tavily goes down, the system just fails. |
| Multi-Model LLM Routing | **Not built** | Single LLM provider per environment (OpenAI prod, Ollama local). No per-call-type routing. |
| Rate Limiting | **Not built** | No rate limiting at any level. |
| Horizontal Scalability | **Not built** | Single FastAPI process. InMemoryRunStore is process-local. |
| Caching Layer | **Not built** | No caching of any kind. |
| Observability | **Minimal** | structlog for logging, performance middleware for request timing. No distributed tracing, no metrics dashboards, no alerting. |
| Hot Config Reload | **Not built** | Config loaded once at startup from env vars. |
| Request Queuing | **Not built** | Requests handled synchronously, no queue. |
| SSE Streaming | **Done** | Agent run events streamed to mobile via SSE. `agent_run_events` table persists full event timeline to Postgres. |
| Graceful Degradation | **Minimal** | Perplexity → Tavily fallback for search. No systematic degradation handling. |
| Multi-Tenant Foundation | **Partial** | RLS on DB tables provides data isolation. No execution-level resource limits. |

---

## Module 2: Skill Management / Skill Map

The registry, lifecycle, and discoverability of all skills.

### Ideal State (12 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Central Registry** | All skills registered with full metadata — schemas, capabilities, health, cost tier, latency stats. Single source of truth. |
| 2 | **Skill Contracts** | Strongly typed input/output JSON schemas. Side-effect flags. Consent requirements. Idempotency declarations. |
| 3 | **Skill Hierarchy / Inheritance** | Capability trees: `communication → {whatsapp, call, email, sms}`. Abstract capabilities map to concrete skill implementations. Planner reasons at capability level. |
| 4 | **Skill Discovery (Embedding/Graph)** | At scale (1K+ skills), planner cannot scan all descriptions. Embedding-based skill search or capability graph traversal to find relevant skills for a goal. |
| 5 | **Skill Versioning** | Multiple versions of same skill coexist. Gradual rollout (canary %). Rollback to previous version. |
| 6 | **Skill Health Metrics** | Per-skill: success_rate, p50/p95/p99 latency, cost per invocation, error categorization, usage count. Rolling windows. |
| 7 | **Skill Caching** | Idempotent skill results cached with TTL. Cache key = skill_id + input hash. Invalidation rules per skill. |
| 8 | **Skill Dependency Declarations** | Skills declare what other skills or data they need. Enables dependency resolution and pre-fetching. |
| 9 | **Dynamic Skill Registration** | Register new skills at runtime without process restart. Hot-add via admin API. |
| 10 | **Skill Testing Framework** | Contract tests (schema validation), integration tests per skill, dry_run mode for every skill. CI-enforced. |
| 11 | **Skill Documentation** | Auto-generated docs from schemas. Usage examples. Capability catalog browsable by planner and humans. |
| 12 | **Skill Deprecation Lifecycle** | Deprecation warnings, migration path to replacement skills, sunset timeline enforcement. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Central Registry | **Done** | `seed_registry.py` — 70+ skill definitions. `SkillsRegistry` class with lookup by ID and capabilities. |
| Skill Contracts | **Done** | `SkillDefinition`, `PlanSpec`, `NodeSpec`, `Receipt` in `contracts.py`. JSON schema validation on inputs. |
| Skill Hierarchy | **Not built** | Flat registry. No capability trees. "Send a message" doesn't resolve to WhatsApp vs call vs email. |
| Skill Discovery | **Not built** | Planner receives full skill catalog in prompt. No embedding search. Works at 70 skills, won't scale to 500+. |
| Skill Versioning | **Not built** | One version per skill. No rollout/rollback. |
| Skill Health Metrics | **Not built** | No per-skill statistics tracked. No success_rate, no latency tracking. |
| Skill Caching | **Not built** | Every invocation hits the real provider. |
| Skill Dependencies | **Not built** | Implicit only (planner figures it out). |
| Dynamic Registration | **Not built** | Skills loaded at import time from seed files. Restart required for changes. |
| Skill Testing | **Minimal** | dry_run mode exists on some handlers. No systematic contract tests. No CI enforcement. |
| Skill Documentation | **Not built** | Descriptions in code only. No generated docs. |
| Skill Deprecation | **Not built** | No lifecycle management. |

---

## Module 3: Task Decomposition

Turning user goals into executable plans.

### Ideal State (12 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **LLM Planner** | Generates validated PlanSpec (task graph JSON) from user goal + skill catalog + full context. Structured output. |
| 2 | **GID (Semantic Router)** | Sub-10ms intent classification using embedding-based `semantic-router`. No LLM call for routing. Handles FAST_PATH / PLANNER / CLARIFY decisions. |
| 3 | **Fast Path Execution** | Single-skill, no-side-effect, high-confidence intents bypass planner entirely. Sub-500ms end-to-end. |
| 4 | **Plan Validation** | All skill_ids verified against registry. Input schemas validated. Cycle detection. Consent flags cross-checked. Max 3 retry on validation failure. |
| 5 | **Clarification Engine** | Detect missing required info before planning. Multi-turn clarification. Resume planning with answers. |
| 6 | **Partial Re-Planning** | Modify in-flight plans. Preserve completed node results. Re-plan only the affected subtree. |
| 7 | **Algorithmic Decomposition** | Known patterns (search+compare, fetch+summarize, contact-per-person) decomposed without LLM. Reduces planner latency and cost. |
| 8 | **Multi-Turn Planning** | Planner sees previous plans in conversation. "Now change the hotel" builds on prior trip plan, not from scratch. |
| 9 | **Plan Templates** | Pre-built plan structures for common goals (trip planning, price comparison, daily digest). Planner uses templates as starting points. |
| 10 | **Plan Optimization** | Minimize API calls, maximize parallelism, cost-aware plan generation. Avoid redundant skills. |
| 11 | **Confidence Scoring** | Planner rates its own plan confidence. Low confidence → escalate to user for confirmation before execution. |
| 12 | **Explainable Plans** | Human-readable plan explanations generated alongside PlanSpec. Shown in UI: "I'll search flights, compare prices, then build an itinerary." |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| LLM Planner | **Done** | `llm_planner.py` generates PlanSpec from goal + skill catalog. Structured JSON output. |
| GID (Semantic Router) | **Not built** | Current intent detection is LLM-based (`intent_router.py`, `intent_parser.py`). Every routing decision costs an LLM call. `semantic-router` library not integrated. |
| Fast Path | **Not built** | All intents go through the full planner pipeline. No fast path bypass. |
| Plan Validation | **Done** | skill_id existence check, input schema validation, cycle detection in `registry.py`. |
| Clarification Engine | **Done** | Interrupt-based clarification in compiler graph. Multi-turn supported. |
| Partial Re-Planning | **Partial** | Planner can generate new plans. But no episode-aware partial planning — can't load a previous plan's completed results and modify just a subtree. |
| Algorithmic Decomposition | **Not built** | All decomposition is LLM-based. |
| Multi-Turn Planning | **Partial** | Conversation buffer (Layer 1) is wired, so planner sees recent messages. But no episodic memory — can't reference yesterday's trip plan. |
| Plan Templates | **Not built** | Every plan generated from scratch. |
| Plan Optimization | **Not built** | Planner doesn't optimize for cost or parallelism — it just generates a plan. |
| Confidence Scoring | **Not built** | No self-assessment of plan quality. |
| Explainable Plans | **Not built** | PlanSpec is technical JSON. No human-readable explanation layer. |

---

## Module 4: Skill Ranking & Selection

Choosing the best provider/implementation when multiple options exist for a capability.

### Ideal State (11 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Multi-Provider Registry** | Multiple implementations per capability. `web.search` → {Perplexity, Tavily, Google, Serper}. `telephony.call` → {Twilio, Knowlarity}. |
| 2 | **Performance Statistics** | Track success rate, latency, cost per provider over rolling windows (1h, 24h, 7d). Stored and queryable. |
| 3 | **Cost-Aware Routing** | Select cheapest provider meeting SLA requirements. Budget-constrained plans use low-cost providers. |
| 4 | **Quality-Aware Routing** | Route based on output quality metrics — user satisfaction signals, LLM judge scores, result completeness. |
| 5 | **A/B Testing** | Randomly route to providers, measure outcomes, auto-converge to winner. Configurable exploration rate. |
| 6 | **User Preference Override** | Users specify preferred providers. "Always use Google Maps." Preferences stored in procedural memory. |
| 7 | **Context-Aware Selection** | Different contexts trigger different providers. Budget trip → cheap API. Business trip → premium API. |
| 8 | **Automatic Fallback Chains** | On failure: retry same → try alternative provider → degrade gracefully. Configurable per skill. |
| 9 | **Provider Degradation Detection** | Passive telemetry detects rising error rates or latency. Auto-shift traffic away from degraded providers. |
| 10 | **Load Balancing** | Distribute load across providers to avoid rate limits. Round-robin or weighted distribution. |
| 11 | **Ranking Formula** | Composite score: `success_rate * w1 + 1/latency * w2 + cost_score * w3 + consent_overhead * w4`. Weights configurable. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Multi-Provider Registry | **Minimal** | Perplexity → Tavily fallback for search. Twilio + Knowlarity for telephony. But no formal multi-provider abstraction — each is a separate skill. |
| Performance Statistics | **Not built** | No per-provider stats tracked anywhere. |
| Cost-Aware Routing | **Not built** | No cost data on skills. |
| Quality-Aware Routing | **Not built** | No quality measurement. |
| A/B Testing | **Not built** | |
| User Preference Override | **Not built** | |
| Context-Aware Selection | **Not built** | |
| Automatic Fallback | **Minimal** | Perplexity → Tavily hardcoded. No general fallback system. |
| Degradation Detection | **Not built** | |
| Load Balancing | **Not built** | |
| Ranking Formula | **Not built** | Planner picks skills by description matching only. |

---

## Module 5: Execution Engine

The runtime that executes plans — graph compilation, parallel execution, state management.

### Ideal State (17 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Two-Tier Graph Architecture** | Outer graph: fixed lifecycle (normalize → clarify → plan → validate → execute → finalize). Inner graph: dynamically compiled from PlanSpec. |
| 2 | **Parallel Fan-Out** | Independent nodes (no dependency edges between them) execute concurrently. LangGraph Send API or parallel branches. |
| 3 | **Map Nodes** | Process variable-length input lists with configurable concurrency limit. Fan-out N skill invocations, collect results. |
| 4 | **Consent Gates** | LangGraph `interrupt()` before side-effecting skills. Graph suspends, checkpoints, resumes on user action via `Command(resume=)`. |
| 5 | **Checkpointing to Postgres** | All graph state persisted to Postgres via `langgraph-checkpoint-postgres`. Survives process restarts. No in-memory-only state. |
| 6 | **Idempotency** | Content-addressable hashing (`skill_id + input hash`). On re-execution, already-completed nodes with matching hashes skip. Prevents duplicate side-effects. |
| 7 | **Error Recovery & Retry** | Per-node retry with exponential backoff. Fallback skill on repeated failure. Partial plan recovery — completed nodes preserved, only failed subtree retried. |
| 8 | **Run Cancellation / Abort** | User can cancel mid-execution. Graceful cleanup: in-flight nodes complete or abort, no orphaned side-effects. |
| 9 | **Timeout Management** | Per-node and per-plan timeouts. Escalation on breach (notify user, abort, or continue with partial results). |
| 10 | **Resource Limits** | Max concurrent nodes per plan. Max API calls per plan. Max total execution time. Prevents runaway plans. |
| 11 | **Execution Isolation** | Failures in one plan/user don't affect others. No shared mutable state between plans. |
| 12 | **Streaming Progress** | Real-time node-level progress events via SSE. Plan started, node started, node completed, approval requested, etc. |
| 13 | **Conditional Routing** | Branch execution based on intermediate results. "Has phone? → call. Has email? → email." |
| 14 | **Batch Execution** | Multi-item batch processing. N items per plan stage. Batch progress tracking. Sequential multi-batch (N stages). |
| 15 | **SLA Tracking** | Deadline-aware execution. Escalation chain when SLA breached (notify → escalate channel → abort). |
| 16 | **Warm Restart** | Resume interrupted plans from last checkpoint. No re-execution of completed nodes. |
| 17 | **Execution Analytics** | Per-plan metrics: total duration, cost, node count, success/failure rate, parallelism utilization. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Two-Tier Graph | **Done** | `compiler_graph.py` (outer lifecycle) + `plan_builder.py` (inner dynamic graph). |
| Parallel Fan-Out | **Done** | LangGraph Send API. Feature-flagged (`PARALLEL_FANOUT_ENABLED`). |
| Map Nodes | **Partial** | Map node type exists in contracts. Multi-batch resume (TB-619) implemented. Full map with concurrency limit unclear. |
| Consent Gates | **Done** | `interrupt()` + `Command(resume=)`. `requires_consent` on SkillDefinition. Double-consent fix (TB-639). |
| Checkpointing | **Done** | `AsyncPostgresSaver` via `langgraph-checkpoint-postgres`. Confirmed deployed: `public.checkpoints`, `checkpoint_blobs`, `checkpoint_writes` tables exist. Fallback to `MemorySaver`. |
| Idempotency | **Done** | `InMemoryRunStore` with deterministic hashing in `store.py`. |
| Error Recovery | **Not built** | A single failed node fails the entire run. No retry with backoff. No fallback skill invocation. |
| Run Cancellation | **Not built** | No abort mechanism from UI or API. |
| Timeout Management | **Not built** | No per-node or per-plan timeouts. |
| Resource Limits | **Not built** | No limits on concurrent nodes or API calls. |
| Execution Isolation | **Partial** | Each run has its own LangGraph thread. But `InMemoryRunStore` is process-local. |
| Streaming Progress | **Done** | SSE event streaming to mobile. ~24 typed event types in `run_events.py`. Confirmed deployed: `agent_runs` + `agent_run_events` tables persist events to Postgres. |
| Conditional Routing | **Partial** | Condition node type defined in contracts. Implementation unclear. |
| Batch Execution | **Done** | Multi-batch graph resume (TB-619). `WAITING_FOR_BATCH_COMPLETION` status. |
| SLA Tracking | **Done** | SLA checker with 30s sweep, escalation chain (TB-623/624). Confirmed: `bot_tasks` has `deadline_at`, `sla_status`, `sla_breached_at`, `escalation_state`, `follow_up_policy` columns deployed. |
| Warm Restart | **Done** | Checkpointing enables resume from last state. Task restart with 10-ancestor context carry-forward (TB-625). |
| Execution Analytics | **Not built** | No per-plan metrics aggregation. |

---

## Module 6: New Skill Creation (On the Fly)

Autonomous capability expansion — the system discovers and integrates new tools without developer intervention.

### Ideal State (10 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **API Spec Discovery** | Discover third-party APIs from OpenAPI/AsyncAPI specs. Crawl API directories or accept user-provided specs. |
| 2 | **MCP Server Discovery** | Find and connect to new MCP servers at runtime. MCP registry/marketplace browsing. |
| 3 | **Auto-Skill Generation** | Generate `SkillDefinition` (schema, description, auth requirements) from API spec automatically. |
| 4 | **Dynamic Integration** | New skills usable immediately without process restart. Hot-register into live registry. |
| 5 | **Skill Composition** | Combine existing skills into new composite skills. User or agent defines multi-step workflows as reusable skills. |
| 6 | **User-Defined Skills** | Users define custom automations ("every morning: check weather → check calendar → send me a summary"). Stored as skill templates. |
| 7 | **Skill Learning from Demos** | Watch user perform a multi-step action → infer the workflow → generate automation skill. |
| 8 | **Safety Sandbox** | New/untrusted skills execute in sandboxed environment. Resource limits, network restrictions, output validation. |
| 9 | **Quality Gate** | Auto-test generated skills (schema compliance, dry-run, output validation) before promoting to production registry. |
| 10 | **Skill Marketplace** | Browse, install, rate community-contributed skills. Review/approval pipeline. Revenue sharing. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| All 10 components | **Not built** | This entire module is future R&D. No runtime skill discovery, generation, or dynamic integration exists. |

---

## Module 7: User Memory

Cross-session intelligence — the system remembers, learns, and personalizes.

### Ideal State (14 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Layer 1 — Conversation Buffer** | Last N messages loaded into planner. Configurable N. Summarization of older messages to save tokens. Include AI responses and system milestones. |
| 2 | **Layer 2 — Working Memory** | Inter-node data flow via `$memory.<node_id>.<field>` late-binding. Resolved at execution time from `TaskContext.task_results`. |
| 3 | **Layer 3 — Episodic Memory** | Post-execution episode summaries stored with vector embeddings. Semantic + structured retrieval. Enables "change the hotel in my Goa trip" across sessions. |
| 4 | **Layer 4 — Semantic Memory (mem0)** | Durable user facts extracted from interactions. "User has HDFC card", "prefers window seats". Semantic retrieval via pgvector. Wired into planner + GID prompts. |
| 5 | **Layer 5 — Procedural Memory** | Skill data store contents (preferences, saved configs) surfaced as lightweight summary to planner. Planner knows what data exists without user restating it. |
| 6 | **Context Assembly Engine** | Right memory layers at each decision point. GID gets lightweight context. Planner gets full context. Executor nodes get only resolved inputs. |
| 7 | **Three-Read Pattern** | Before every plan: (1) load conversation buffer, (2) query relevant episodes + semantic memories, (3) summarize procedural memory. |
| 8 | **Three-Write Pattern** | After every execution: (1) write AI response to chat, (2) generate + store episode summary, (3) extract + store durable facts to mem0. |
| 9 | **Message Summarization** | Older messages summarized instead of raw-injected. Hybrid: last 5 verbatim + summary of previous 15. Token budget management. |
| 10 | **Relevance Filtering** | Not all memories are relevant. Rank by semantic similarity to current query + recency. Top-K injection. |
| 11 | **Memory Decay & Archival** | Old episodes distilled into semantic memory facts, then archived. Stale memories rank lower. Configurable TTLs. |
| 12 | **Memory Conflict Resolution** | Handle contradictions: user said "HDFC card" last week but "I cancelled it" today. Update/invalidate stale facts. |
| 13 | **User Memory Dashboard** | Users can view, edit, and delete what the system knows about them. Transparency requirement. |
| 14 | **Token Budget Management** | Total memory context stays within budget (~4K tokens). Dynamic allocation across layers based on relevance. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Layer 1 — Conversation Buffer | **Partial** | Last 15 messages loaded in `agent_runs.py`. Wired into planner. But: no summarization, no system messages, no configurable N, hardcoded limit. |
| Layer 2 — Working Memory | **Done** | `$memory.<node_id>.<field>` late-binding works. `ExecutionContext` carries all per-run state. |
| Layer 3 — Episodic Memory | **Not built** | No `execution_episodes` table in DB. No episode generation after runs. No vector search. This is the biggest memory gap. |
| Layer 4 — Semantic Memory | **Service built, not wired** | `app/services/memory/` — MemoryService with Supabase/pgvector backend exists in code. But NOT injected into planner prompt. NOT called before planning. NOT called after execution to extract facts. Effectively unused. |
| Layer 5 — Procedural Memory | **Data store exists, not surfaced** | `skill_data_store` table deployed in DB. Service code exists. Skills can read/write structured data. But planner doesn't see a summary of stored data — can't reason about what's already configured. |
| Context Assembly | **Not built** | No coordinated context assembly. Planner gets conversation buffer only. No layer-aware assembly. |
| Three-Read Pattern | **1 of 3** | Only conversation buffer read is implemented. Episodes and semantic memory not read. |
| Three-Write Pattern | **~0.5 of 3** | AI response written to chat. Skills write to `skill_data_store` at execution time (Layer 5 write works). But no episode generation (Layer 3). No fact extraction to mem0 (Layer 4). |
| Message Summarization | **Not built** | Raw dump only. |
| Relevance Filtering | **Not built** | No semantic ranking of memories. |
| Memory Decay | **Not built** | |
| Conflict Resolution | **Not built** | |
| User Dashboard | **Not built** | |
| Token Budget | **Not built** | No budget management. |

---

## Module 8: Agent Intelligence / Decision Authority

How smart, autonomous, and proactive the agent is beyond just executing plans.

### Ideal State (14 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Consent Gates** | Mandatory approval before all side-effects. Risk-level-based policies (low → auto-approve if previously approved, high → explicit confirmation). |
| 2 | **Autonomy Levels** | User-configurable autonomy per domain. "Always ask for payments." "Auto-approve scheduling." "Full auto for search." Stored in procedural memory. |
| 3 | **Automatic Channel Selection** | Rules engine: "contact them" → evaluate contact's available channels → pick best based on urgency, time of day, contact preference, past success. |
| 4 | **Channel Escalation** | Auto-escalate when SLA breached. WhatsApp no response in 30m → try call. Call no answer → leave voicemail. Configurable escalation chains. |
| 5 | **PII Detection & Redaction** | Detect PII (phone numbers, addresses, financial data) before sending to external APIs. Redact or warn. Per-skill PII policies. |
| 6 | **Strategy Persistence** | Agent learns which approaches work for each user. "Calling at 10 AM works better for this contact." Stored in episodic memory. |
| 7 | **Proactive Intelligence** | Agent initiates actions without user prompting. Price drop alerts, schedule conflict detection, weather warnings for planned trips, recurring pattern recognition. |
| 8 | **Uncertainty Quantification** | Agent knows what it doesn't know. Low confidence → ask user rather than guess. "I found 2 flights but I'm not sure about your budget — should I filter?" |
| 9 | **Preference Learning** | Infer preferences from behavior, not just explicit statements. User always picks cheapest flight → learn "budget-conscious traveler." |
| 10 | **Goal Inference** | Understand implicit goals. "I have a meeting at 9 in Gurgaon" → check traffic, prepare briefing, set alarm. Contextual intelligence. |
| 11 | **Feedback Loop** | User corrections improve future behavior. "Don't send WhatsApp to my boss" → permanent rule. Stored and enforced. |
| 12 | **Delegation Authority Matrix** | Per-domain, per-action autonomy settings. Configurable by user. Default-safe (ask for everything). |
| 13 | **Explainability** | Agent explains its decisions when asked. "Why did you choose IndiGo?" → "₹800 cheaper than Air India, same timings, you've flown IndiGo before." |
| 14 | **Multi-Agent Collaboration** | Specialized sub-agents collaborate on complex goals. Research agent + booking agent + communication agent working in concert. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Consent Gates | **Done** | LangGraph `interrupt()` on `requires_consent` skills. Approval artifacts. TB-639 double-consent fix. |
| Autonomy Levels | **Not built** | No user-configurable autonomy. Everything requires explicit approval (conservative default). |
| Channel Selection | **Not built** | Planner picks channel in plan. No rules engine. No automatic selection. |
| Channel Escalation | **Done** | WhatsApp → call escalation on SLA overdue (TB-624). |
| PII Detection | **Not built** | No PII scanning. User data sent to LLMs and external APIs without filtering. |
| Strategy Persistence | **Not built** | No learning from outcomes. |
| Proactive Intelligence | **Not built** | Agent is purely reactive — only responds to user messages or scheduled triggers. No autonomous initiation. |
| Uncertainty Quantification | **Not built** | |
| Preference Learning | **Not built** | |
| Goal Inference | **Not built** | |
| Feedback Loop | **Not built** | User corrections don't persist as rules. |
| Delegation Matrix | **Not built** | |
| Explainability | **Not built** | |
| Multi-Agent Collaboration | **Minimal** | Agent adapter supports external agents (Deal Hunter). But no true multi-agent collaboration protocol. |

---

## Module 9: Front-End / UX & Transparency

User-facing experience — how users interact with, monitor, and control the agent.

### Ideal State (16 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Chat UI** | Conversational interface. Message history. Rich text rendering. |
| 2 | **SSE Event Streaming** | Real-time progress events from backend to UI. |
| 3 | **Consent Request UI** | Approval/edit/deny cards. Batch approval. Editable drafts. |
| 4 | **Planning View** | Show plan being built — step-by-step reasoning, like Replit Agent or Devin. User sees what the agent is about to do. |
| 5 | **Task Abort** | Cancel mid-execution from UI. Confirmation dialog. Graceful cleanup. |
| 6 | **Edit Prompt Mid-Execution** | Modify the request while plan runs. Triggers partial re-planning. |
| 7 | **Reasoning / Citations Display** | Show how agent arrived at decisions. Source links. Confidence indicators. |
| 8 | **Structured Artifact Rendering** | Rich cards: flights, hotels, itineraries, price comparisons, briefings, digests. Renderer registry per artifact type. |
| 9 | **Batch Progress Visualization** | See N tasks running in parallel. Per-item status. Overall progress bar. |
| 10 | **Memory Dashboard** | View/edit/delete what the agent knows. Fact list. Episode history. Preference editor. |
| 11 | **Skill Browser** | Browse available capabilities. Category navigation. Search. |
| 12 | **Notification Management** | Control notification frequency, channels, quiet hours. |
| 13 | **Multi-Modal Input** | Voice input, image upload, document upload as conversation input. |
| 14 | **Proactive Suggestion Cards** | Agent suggests actions in UI without being asked. "You have a trip tomorrow — check weather?" |
| 15 | **History / Timeline** | Browse past executions and their results. Searchable. Filterable. |
| 16 | **Preferences / Settings UI** | Configure autonomy levels, default channels, briefing times, connected accounts. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Chat UI | **Done** | React Native chat interface. |
| SSE Streaming | **Done** | Agent run events streamed to client. |
| Consent UI | **Done** | Mobile consent cards with approve/deny. |
| Planning View | **Not built** | |
| Task Abort | **Not built** | |
| Edit Mid-Execution | **Not built** | |
| Reasoning/Citations | **Not built** | |
| Artifact Rendering | **Partial** | `StructuredArtifact` model exists in backend. `ARTIFACT_TYPES` defined. But mobile renderers not built for most types. |
| Batch Progress | **Not built** | |
| Memory Dashboard | **Not built** | |
| Skill Browser | **Not built** | |
| Notification Management | **Not built** | |
| Multi-Modal Input | **Partial** | Document upload exists (Kreuzberg extraction). No image or voice-to-text in chat. |
| Proactive Suggestions | **Not built** | |
| History/Timeline | **Not built** | |
| Preferences UI | **Not built** | |

**Note:** Frontend is not in the backend repo's scope. But backend APIs for many of these features are also missing or incomplete.

---

## Module 10: Communication Skills as Foundational

Multi-channel communication — the agent's ability to reach people and services.

### Ideal State (16 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Voice Calls (Twilio)** | Full outbound/inbound. AI voice agent. Call recording. Transcription. DTMF. Conference. |
| 2 | **Voice Calls (Knowlarity)** | Alternative telephony provider. Same capabilities as Twilio path. |
| 3 | **WhatsApp (Meta)** | Send/receive text, media, documents. Template messages. Read receipts. Interactive messages (buttons, lists). |
| 4 | **WebRTC (LiveKit)** | Browser-based agent calls. Link sharing. Screen share. Multi-participant. |
| 5 | **Voice Cloning** | User voice profiles. Cloned voice for outbound calls. Multiple voice profiles per user. |
| 6 | **Email (Gmail/SMTP)** | Full read/write/search/summarize. Thread management. Attachment handling. Multiple accounts. |
| 7 | **SMS (Rich)** | Beyond basic send. Delivery tracking. Two-way SMS. MMS (media). |
| 8 | **Slack** | Full channel interaction. DMs. Thread replies. Rich formatting. Bot presence. |
| 9 | **Telegram** | Bot integration. Inline queries. Rich media. Group support. |
| 10 | **Unified Channel Abstraction** | "Contact them" → agent evaluates all available channels → picks best. Single skill interface, multiple backends. |
| 11 | **Cross-Channel Context** | Call summary → WhatsApp follow-up with context. Email thread → Slack notification with summary. Conversation continuity across channels. |
| 12 | **Channel Preference Learning** | Learn which channels work for which contacts at which times. "Boss responds faster on WhatsApp before 10 AM." |
| 13 | **Rich Media Handling** | Send/receive images, documents, voice notes, location pins across all channels. Format adaptation per channel. |
| 14 | **Conversation Threading** | Maintain context when switching channels mid-conversation. Agent remembers what was discussed on call when sending WhatsApp follow-up. |
| 15 | **Delivery Tracking** | Know when messages were delivered, read. Retry on failure. Escalate on non-delivery. |
| 16 | **Smart Templates** | Dynamic template generation per channel constraints. WhatsApp template approval workflow. Personalized message drafting. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Voice (Twilio) | **Done** | Full outbound/inbound, AI voice agent, recording, transcription, media streams. |
| Voice (Knowlarity) | **Done** | Alternative telephony. Stream service implemented. |
| WhatsApp (Meta) | **Done** | Send/receive, template messages. GreenAPI + Meta BSP (Pinnacle) dual provider. |
| WebRTC (LiveKit) | **Done** | Browser-based calls. Link sharing via SMS/WhatsApp. Session management. |
| Voice Cloning | **Done** | ElevenLabs voice profiles. Multiple providers (ElevenLabs, CosyVoice, Cartesia, Coqui). |
| Email | **Partial** | Gmail digest handler exists (`google_workspace_handlers.py`). But: read-only summarization. No send, no compose, no thread management. Google OAuth wired but limited. |
| SMS | **Minimal** | Via Twilio, basic send only. No delivery tracking, no two-way, no MMS. |
| Slack | **Partial** | Composio marketplace skill for posting messages. No full integration (no DMs, threads, presence). |
| Telegram | **Not built** | |
| Unified Channel Abstraction | **Not built** | Each channel is a separate skill. No "contact them" resolver. |
| Cross-Channel Context | **Not built** | Channels are siloed. No context passing between channels. |
| Channel Preference Learning | **Not built** | |
| Rich Media | **Partial** | WhatsApp supports media. Voice calls support audio. No unified media handling. |
| Conversation Threading | **Not built** | |
| Delivery Tracking | **Minimal** | WhatsApp delivery/read receipts from Meta webhooks. Nothing unified. |
| Smart Templates | **Not built** | |

---

## Module 11: Security, Privacy, Consent

Trust, compliance, and safety infrastructure.

### Ideal State (19 components)

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Consent Lifecycle** | Request → user decision (approve/edit/deny) → execution. Full audit trail. Expiry on stale requests. |
| 2 | **Row-Level Security** | Per-user data isolation at DB level. All tables RLS-enabled. |
| 3 | **JWT Auth** | Supabase session authentication. Token refresh. Session management. |
| 4 | **Firebase Auth** | Push notification authentication. Device token management. |
| 5 | **Account Deletion** | GDPR-compliant deletion. Full data removal. Audit log of deletion. |
| 6 | **PII Detection in LLM Prompts** | Scan prompts before sending to external LLMs/APIs. Detect phone numbers, addresses, financial data. Redact or mask. |
| 7 | **Data Retention Policies** | Auto-expire old data by type. Chat messages: 1 year. Call recordings: 90 days. Configurable per data class. |
| 8 | **Full Audit Trail** | Every consent decision, every data access, every external API call logged. Queryable. Tamper-evident. |
| 9 | **Compliance Framework (GDPR)** | Right to access, right to erasure, data portability, processing records. |
| 10 | **Compliance Framework (India DPDP)** | Data localization awareness. Consent management per DPDP Act requirements. |
| 11 | **Rate Limiting / Abuse Prevention** | Per-user, per-skill, per-IP rate limits. Anomaly detection. Account suspension on abuse. |
| 12 | **API Key Rotation** | Automated rotation of external API keys. Zero-downtime key swap. Key usage tracking. |
| 13 | **Encryption at Rest** | Sensitive data encrypted in DB (financial info, PII, voice recordings). Key management (KMS or Vault). |
| 14 | **Input Validation / Injection Prevention** | Prompt injection defense. SQL injection prevention. XSS protection. Input sanitization on all user inputs. |
| 15 | **Role-Based Access Control** | Admin, user, viewer roles. Feature gating per role. Organization support (multi-user teams). |
| 16 | **Security Logging & Alerting** | Anomaly detection on login patterns, API usage, data access. Alerts on suspicious activity. |
| 17 | **Data Classification** | Classify data by sensitivity (public, internal, confidential, restricted). Apply handling rules per class. |
| 18 | **Third-Party Data Policies** | Define what data can be shared with each external service. Consent requirements per data-service pair. |
| 19 | **Penetration Testing Ready** | Security hardened. OWASP top 10 mitigated. Regular security audits. Bug bounty readiness. |

### What's Built Today

| Component | Status | Notes |
|-----------|--------|-------|
| Consent Lifecycle | **Done** | Consent request → approve/decline → action. `consent_requests` table with full status tracking. |
| Row-Level Security | **Done** | RLS policies on all `app` schema tables. |
| JWT Auth | **Done** | Supabase auth + `get_authenticated_user` dependency. |
| Firebase Auth | **Done** | Firebase service for push notifications. |
| Account Deletion | **Done** | `account_deletion_log` table. Deletion flow implemented. |
| PII Detection | **Not built** | User data sent raw to LLMs and external APIs. |
| Data Retention | **Not built** | No auto-expiry. Data accumulates indefinitely. Audio retention flag exists (`AUDIO_FILE_RETENTION_HOURS`) but no general policy. |
| Audit Trail | **Partial** | Agent run events logged. But no unified audit trail across all operations. No consent decision audit. |
| GDPR Compliance | **Partial** | Account deletion exists. No right-to-access, no data export, no processing records. |
| India DPDP | **Not built** | |
| Rate Limiting | **Not built** | |
| API Key Rotation | **Not built** | Keys stored as env vars. Manual rotation only. |
| Encryption at Rest | **Not built** | Supabase default encryption only. No application-level encryption of sensitive fields. Token encryption exists for Google OAuth tokens. |
| Input Validation | **Partial** | Pydantic schema validation on API inputs. No prompt injection defense. |
| RBAC | **Not built** | Single user role. No admin/viewer distinction. No org support. |
| Security Logging | **Not built** | |
| Data Classification | **Not built** | |
| Third-Party Policies | **Not built** | |
| Pen Test Ready | **Not built** | |

---

## DB Schema Reality Check

Schema from `docs/entire-schema.sql` (updated 2026-03-30).

### app schema (32 tables)

| Table | Category | Notes |
|-------|----------|-------|
| `user_profiles` | Users | Core user table. Account status, voice settings, referral code. |
| `chats` | Chat | Per-user chat sessions. |
| `chat_messages` | Chat | Messages with sender_type (user/ai), seq ordering, attachments, citations, metadata. |
| `agent_runs` | Orchestration | Agent execution tracking — run_id, status, approvals, clarifications, agent_answers, waiting_deadline. |
| `agent_run_events` | Orchestration | Event timeline per run — seq-ordered, event_type + payload JSONB. |
| `batches` | Tasks | Batch task containers — progress tracking, channel_mix, summary. |
| `bot_tasks` | Tasks | Individual tasks — SLA tracking (deadline_at, sla_status, sla_breached_at), escalation_state, structured output (output_raw, output_summary, output_extracted), root_task_id for restart chains. |
| `consent_requests` | Consent | Consent lifecycle — pending/approved/declined/expired/delivered. Linked to calls and tasks. |
| `skill_data_store` | Data | Generic skill persistence — namespace, record_type, payload JSONB, recorded_at. |
| `scheduled_jobs` | Scheduling | Cron jobs — job_type (skill_invocation/graph_execution), cron_expression, config JSONB, next_run. |
| `calls` | Voice | Call records — direction, status, outcome, agent_mode, consent_request_ids. |
| `call_memberships` | Voice | Call participants with roles. |
| `recordings` | Voice | Twilio recordings — status, duration, media_url. |
| `transcripts` | Voice | Call transcripts with full-text search (tsvector). |
| `transcript_segments` | Voice | Per-segment transcription with speaker, timestamps, full-text search. |
| `voice_profiles` | Voice | Cloned voice profiles — provider (elevenlabs), status lifecycle. |
| `voip_numbers` | Voice | Twilio phone numbers per user. |
| `whatsapp_conversations` | WhatsApp | Conversation threads — summary, summary_model. |
| `whatsapp_messages` | WhatsApp | Individual messages — direction, status, media, full-text search. |
| `whatsapp_schedules` | WhatsApp | Scheduled WhatsApp messages — cron-like scheduling. |
| `contacts` | Contacts | User contact book — phone, blocked, force_agent flags. |
| `inbox_assignments` | Contacts | Channel-to-contact routing (whatsapp/sms/gmail/slack). |
| `user_permissions` | Security | Per-permission consent tracking — consent_status, os_grant_state, policy versioning. |
| `permission_events` | Security | Permission change audit log. |
| `otp_events` | Security | OTP delivery and decision audit. |
| `app_configurations` | Config | Key-value system config with public/private flag. |
| `account_deletion_log` | GDPR | Deletion audit trail. |
| `referral_events` | Growth | Referral tracking. |

### public schema (LangGraph checkpoint tables)

| Table | Purpose |
|-------|---------|
| `checkpoints` | LangGraph graph state checkpoints — thread_id, checkpoint_id, checkpoint JSONB, metadata. |
| `checkpoint_blobs` | Binary state blobs per channel/version. |
| `checkpoint_writes` | Write operations per checkpoint — task_id, channel, blob. |
| `checkpoint_migrations` | Schema version tracking. |

### What's NOT in the schema

| Expected | Status | Impact |
|----------|--------|--------|
| `execution_episodes` | **Not deployed** | Episodic memory (Layer 3) has no storage. Biggest memory gap. |
| mem0 / pgvector tables | **Not in main DB** | Semantic memory (Layer 4) — mem0 may use its own managed storage, or pgvector extension may not be enabled. |

---

## Next Step

Review the ideal state definitions and component breakdowns. Percentages are first-pass estimates — calibrate based on how much weight each component deserves (not all components are equal complexity).
