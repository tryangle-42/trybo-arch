# Module Ownership Map

> Definitive breakdown of what each module IS, what it owns, what it does NOT own, what interfaces it exposes, and what it depends on. Designed so that each module (or module cluster) can be assigned to and built independently by a developer.
>
> This document ignores current implementation state. It defines the target architecture.

---

## Executive Summary

Trybo's backend is 11 modules. Here's what each one does in plain English, how they connect, and who could own them.

### Module Cheatsheet

| ID | Module | One-liner | Analogy |
|----|--------|-----------|---------|
| **M1** | Adapter Framework | The universal plug that connects to any external tool or API | USB hub — same port, different devices |
| **M2** | Skill Management | The catalog of everything the agent knows how to do | App store listing — metadata, ratings, descriptions |
| **M3** | Task Decomposition | Turns "plan my Goa trip" into a step-by-step execution plan | Project manager breaking a goal into tasks |
| **M4** | Skill Ranking & Selection | Picks the best provider when multiple can do the same thing | Price comparison engine — cheapest, fastest, most reliable |
| **M5** | Execution Engine | Actually runs the plan — parallel tasks, approvals, error handling | Factory floor — the assembly line that builds the output |
| **M6** | New Skill Creation | Discovers and integrates new tools without developer help | Self-expanding app store (future R&D, not built) |
| **M7** | User Memory | Remembers everything across sessions — facts, past work, preferences | The agent's brain — short-term, long-term, and muscle memory |
| **M8** | Agent Intelligence | Makes the agent smart — knows when to ask, when to act, when to proact | The agent's personality and judgment |
| **M9** | Front-End / UX | What the user sees and touches — chat, cards, progress, controls | The face of the product (React Native app) |
| **M10** | Communication Skills | Reaches people — calls, WhatsApp, email, SMS, Slack | The agent's mouth and ears across every channel |
| **M11** | Security & Consent | Auth, permissions, PII protection, audit trails, compliance | The bouncer — nothing happens without authorization |

### How They Connect (simplified)

```
User says something
       │
       ▼
   ┌───────┐     ┌───────┐
   │  M11  │     │  M7   │    M11 (Security) guards every step
   │Security│     │Memory │    M7  (Memory) provides context at every step
   └───┬───┘     └───┬───┘
       │             │
       ▼             ▼
   ┌─────────────────────┐
   │   M3: Decompose     │    "What needs to happen?" → PlanSpec
   │   (uses M2 catalog) │
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────┐
   │   M5: Execute       │    Run each step, handle approvals
   │   (uses M4 ranking) │
   └──────┬───────┬──────┘
          │       │
          ▼       ▼
   ┌──────────┐ ┌──────────┐
   │ M1:Adapt │ │M10:Comms │   Call the actual tools and channels
   └──────────┘ └──────────┘
              │
              ▼
   ┌─────────────────────┐
   │   M9: Show result   │    Render to user
   │   M8: Learn from it │    Get smarter for next time
   └─────────────────────┘
```

### What can be built independently?

Three modules have **zero dependencies** on other modules — they're the foundation:

| Foundation module | Why it's foundational |
|---|---|
| **M2** (Skill Management) | Everything needs to know what skills exist |
| **M7** (User Memory) | Everything needs user context |
| **M11** (Security) | Everything needs auth and consent |

These three can be built first, in parallel, by separate developers. Everything else builds on top of them.

### Ownership at a glance (3-person team)

| Person | Modules | What they own in plain English |
|--------|---------|-------------------------------|
| **Avinash** | M2, M3, M4, M5, M7, M8, M11 | The engine + the brain + security. Everything that makes the agent think and act. |
| **Krishna** | M1, M10 | The connectors. How the agent talks to external tools and reaches people (voice, WhatsApp, email). |
| **Rishav** | M9 + skill handlers | The face. What users see, plus the business logic inside each skill. |

---

## System Topology

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                          ENTRY POINTS                                   │
 │              Chat  ·  Scheduled Job  ·  Webhook  ·  Proactive           │
 └────────────────────────────────┬────────────────────────────────────────┘
                                  │
 ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
   M11: SECURITY (cross-cutting)  │  Auth · PII · RLS · Rate Limits
 │ Wraps every layer below        │  Consent Enforcement · Audit Trail   │
 └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
                                  │
                                  ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │                     M3: TASK DECOMPOSITION                             │
 │         GID (semantic router) ──▶ Clarify ──▶ Plan ──▶ Validate        │
 │                                                                        │
 │   reads from ──▶ M2 (skill catalog)                                    │
 │   reads from ──▶ M7 (assembled context: conversation + episodes +      │
 │                      semantic memories + procedural summary)            │
 └───────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ PlanSpec (validated task graph)
                                 ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │                     M5: EXECUTION ENGINE                               │
 │    Compile ──▶ Fan-out ──▶ Consent Gate ──▶ Execute ──▶ Finalize       │
 │                                                                        │
 │   calls ──▶ M1  (adapter.invoke per node)                              │
 │   calls ──▶ M4  (select best provider)                                 │
 │   calls ──▶ M11 (consent enforcement)                                  │
 │   emits ──▶ SSE events to M9 (frontend)                                │
 │   writes ─▶ M7  (episodes + facts on finalize)                         │
 └──────────┬─────────────────────────────────────┬───────────────────────┘
            │                                     │
            ▼                                     ▼
 ┌─────────────────────────┐       ┌──────────────────────────────┐
 │   M1: ADAPTER FRAMEWORK │       │  M10: COMMUNICATION SKILLS   │
 │                         │       │                              │
 │   Direct  (Python fns)  │◀──────│  Voice (Twilio, Knowlarity)  │
 │   MCP     (config-driven│       │  WhatsApp (Meta, Pinnacle)   │
 │   Marketplace (Composio)│       │  WebRTC (LiveKit)            │
 │   Agent   (external)    │       │  Email · SMS · Slack         │
 │                         │       │  Voice Cloning               │
 │   Health probes         │       │  Unified channel abstraction │
 │   Circuit breakers      │       │                              │
 └─────────────────────────┘       └──────────────────────────────┘

 ── FOUNDATIONAL SERVICES (zero upstream dependencies) ──────────────────

 ┌──────────────────────┐  ┌──────────────────────┐  ┌─────────────────┐
 │  M2: SKILL MGMT      │  │  M7: USER MEMORY     │  │  M4: RANKING &  │
 │                      │  │                      │  │  SELECTION      │
 │  Registry (70+ skills│  │  L1: Chat buffer     │  │                 │
 │  Contracts + schemas │  │  L2: Working memory   │  │  Multi-provider │
 │  Hierarchy / trees   │  │  L3: Episodic memory  │  │  Stats tracking │
 │  Discovery (embed)   │  │  L4: Semantic (mem0)  │  │  Cost routing   │
 │  Health metrics      │  │  L5: Procedural       │  │  A/B testing    │
 │  Versioning          │  │  Context assembly     │  │  Fallback chains│
 │  Testing framework   │  │  Read/write patterns  │  │  Load balancing │
 └──────────────────────┘  └──────────────────────┘  └─────────────────┘

 ── HIGHER-ORDER MODULES ────────────────────────────────────────────────

 ┌──────────────────────────┐  ┌────────────────────────────────────────┐
 │  M8: AGENT INTELLIGENCE  │  │  M9: FRONT-END / UX                   │
 │                          │  │                                        │
 │  Autonomy levels         │  │  Chat UI · Consent cards               │
 │  Channel auto-selection  │  │  Planning view · Task abort            │
 │  Proactive actions       │  │  Artifact rendering · Batch progress   │
 │  Preference learning     │  │  Memory dashboard · Skill browser      │
 │  Goal inference          │  │  Notifications · History / timeline    │
 │  Explainability          │  │  Preferences / settings                │
 │  Feedback loop           │  │                                        │
 └──────────────────────────┘  └────────────────────────────────────────┘

 ── FUTURE R&D ──────────────────────────────────────────────────────────

 ┌──────────────────────────────────────────────────────────────────────┐
 │  M6: NEW SKILL CREATION (ON THE FLY)                                 │
 │  API discovery · MCP discovery · Auto-generate skills · Sandbox      │
 │  Skill composition · User-defined workflows · Marketplace            │
 └──────────────────────────────────────────────────────────────────────┘
```

---

## Module Definitions

---

### M1: Core Platform — Adapter Framework

**Mission:** Provide a uniform interface for the execution engine to invoke any external capability — whether it's a Python function, an MCP server, a marketplace API, or an external agent.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| `SkillAdapter` ABC and `invoke()` contract | Deciding WHICH adapter to call (that's M4) |
| `DirectAdapter` — routes skill_id to Python handler functions | The handler functions themselves (those belong to the module that owns the domain — M10 for comms, etc.) |
| `MCPAdapter` — config-driven MCP server connections via `MultiServerMCPClient` | Skill registration metadata (that's M2) |
| `MarketplaceAdapter` — Composio SDK integration for third-party tools | Plan creation or task decomposition (that's M3) |
| `AgentAdapter` — bidirectional lifecycle for external agents | Consent enforcement (that's M11 via M5) |
| Adapter health probes and circuit breaker pattern | |
| MCP server config management (add/remove/update MCP servers) | |
| Adapter response normalization to `SkillResult` | |

#### Interfaces Exposed

```
SkillAdapter.invoke(skill_id, input, context) → SkillResult
AdapterHealthMonitor.check(adapter_type, skill_id) → HealthStatus
MCPConfigManager.register_server(name, config) → void
MCPConfigManager.list_servers() → list[MCPServerConfig]
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M2 (Skill Management) | Skill definitions to know adapter_type and handler routing |
| M11 (Security) | API key management, credential storage |

#### Data Owned

No dedicated tables. MCP server configs stored in application config or `app_configurations` table.

---

### M2: Skill Management

**Mission:** Be the single source of truth for every capability the system can invoke. No skill exists outside the registry. The planner cannot hallucinate a skill that isn't registered.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| `SkillDefinition` schema (id, name, kind, adapter_type, input/output schemas, side_effect, requires_consent, cost_tier, health_status) | Implementing the actual skill logic (that's M1 adapters + domain modules) |
| `SkillsRegistry` — CRUD, lookup by ID, lookup by capability, filtered catalog for planner | Deciding which skill to pick when multiple match (that's M4) |
| Skill hierarchy / capability trees (`communication → {whatsapp, call, email}`) | Running skills (that's M5) |
| Skill discovery — embedding-based search for planner when catalog is too large to fit in prompt | |
| Skill health metrics collection — success_rate, latency, cost per skill (aggregated from execution telemetry) | |
| Skill versioning — multiple versions coexist, gradual rollout | |
| Skill contracts — JSON schema validation on inputs/outputs | |
| Skill registration API — seed files, dynamic registration, deprecation lifecycle | |
| Skill testing framework — contract tests, dry_run enforcement | |
| Skill documentation generation from schemas | |

#### Interfaces Exposed

```
Registry.get(skill_id) → SkillDefinition
Registry.get_catalog(filter: health=active) → list[SkillDefinition]
Registry.search(query_embedding, top_k) → list[SkillDefinition]       # for discovery at scale
Registry.get_by_capability(capability) → list[SkillDefinition]
Registry.validate_inputs(skill_id, input_dict) → ValidationResult
Registry.record_execution(skill_id, duration_ms, success, cost) → void  # telemetry sink
Registry.get_stats(skill_id) → SkillStats                               # for M4 ranking
Registry.register(skill_def) → void
Registry.deprecate(skill_id, replacement_id) → void
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| None at runtime | Registry is a foundational service with no upstream dependencies |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| In-memory registry (loaded from seed files) | Primary skill catalog |
| `skill_execution_stats` (new, or use `app_configurations`) | Per-skill telemetry aggregates |
| Embedding index (pgvector or in-memory FAISS) | Skill discovery at scale |

---

### M3: Task Decomposition

**Mission:** Turn a user's natural language goal into a validated, executable PlanSpec — choosing the right skills, structuring dependencies, and knowing when to ask for clarification.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| **GID (Global Intent Detector)** — sub-10ms routing: FAST_PATH / PLANNER / CLARIFY | Running the plan (that's M5) |
| **Clarification engine** — detect missing info, multi-turn Q&A, resume | Adapter invocation (that's M1) |
| **LLM Planner** — generate PlanSpec from goal + context + skill catalog | Memory storage/retrieval (that's M7 — but M3 consumes assembled context) |
| **Plan validation** — skill_id existence, schema conformance, cycle detection, consent flag cross-check | Consent UI (that's M9/M11) |
| **Partial re-planning** — modify in-flight plans preserving completed work | |
| **Algorithmic decomposition** — known patterns decomposed without LLM | |
| **Plan templates** — pre-built structures for common goals | |
| **Plan optimization** — maximize parallelism, minimize API calls, cost-aware | |
| **Confidence scoring** — planner self-assesses plan quality | |
| **Explainable plans** — human-readable explanation alongside PlanSpec | |

#### Interfaces Exposed

```
GID.route(message, conversation_context, user_memories) → RoutingDecision(FAST_PATH|PLANNER|CLARIFY)
Planner.create_plan(goal, context: AssembledContext, catalog: list[SkillDefinition]) → PlanSpec
Planner.replan(existing_plan, completed_results, modification_request) → PlanSpec
PlanValidator.validate(plan: PlanSpec, registry: Registry) → ValidationResult
ClarificationEngine.check(goal, context) → ClarificationNeeded | None
```

**PlanSpec schema (the key contract):**
```json
{
  "plan_id": "uuid",
  "goal": "string",
  "explanation": "human-readable plan description",
  "confidence": 0.0-1.0,
  "tasks": [
    {
      "task_id": "string",
      "skill_id": "string (must exist in registry)",
      "node_type": "tool | agent | map | condition",
      "input": { "params, may contain $memory.<node_id>.<field>" },
      "depends_on": ["task_id"],
      "requires_consent": true | false,
      "fallback_skill_id": "string | null",
      "timeout_ms": "integer | null",
      "map_config": { "inputs_from": "$memory ref", "concurrency_limit": 10 }
    }
  ]
}
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M2 (Skill Management) | Skill catalog for planner prompt, validation against registry |
| M7 (User Memory) | Assembled context (conversation buffer, episodes, semantic memories, procedural summary) |
| M2 (Skill Management) | Skill discovery/search when catalog is too large for prompt |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| GID route definitions | Semantic router embeddings for intent classification |
| Plan templates | Pre-built PlanSpec structures for common goals |

---

### M4: Skill Ranking & Selection

**Mission:** When multiple skills or providers can fulfill a capability, pick the best one based on performance, cost, quality, and user preferences.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| Multi-provider abstraction — multiple implementations per capability | Registering skills (that's M2) |
| Performance statistics — success rate, latency percentiles, cost tracking over rolling windows | Running skills (that's M5) |
| Ranking formula — composite score from stats, cost, consent overhead | Defining skill schemas (that's M2) |
| Cost-aware routing — cheapest provider meeting SLA | |
| Quality-aware routing — user satisfaction signals, output completeness | |
| A/B testing between providers — exploration vs exploitation | |
| User preference overrides — "always use Google Maps" | |
| Automatic fallback chains — primary → secondary → tertiary | |
| Provider degradation detection — auto-shift traffic from failing providers | |
| Load balancing — distribute to avoid rate limits | |

#### Interfaces Exposed

```
Ranker.select(capability, context: RankingContext) → RankedSkillList
Ranker.get_fallback_chain(skill_id) → list[skill_id]
Ranker.record_outcome(skill_id, success, latency_ms, cost, quality_score) → void
Ranker.set_user_preference(user_id, capability, preferred_skill_id) → void
Ranker.get_provider_health(capability) → list[ProviderHealth]
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M2 (Skill Management) | Skill stats, capability lookups, health status |
| M7 (User Memory) | User preference overrides (from procedural memory / skill_data_store) |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| `provider_stats` (new) or aggregated in M2 | Rolling window performance data per provider |
| A/B test configuration | Experiment definitions and traffic splits |

---

### M5: Execution Engine

**Mission:** Take a validated PlanSpec and execute it to completion — compiling it into a LangGraph state machine, running nodes in parallel where possible, handling consent interrupts, persisting state, and recovering from failures.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| **Graph compilation** — PlanSpec → LangGraph StateGraph (inner graph) | Creating the plan (that's M3) |
| **Outer lifecycle graph** — normalize → clarify → plan → validate → execute → finalize | Choosing skills (that's M3 + M4) |
| **Parallel fan-out** — independent nodes run concurrently (LangGraph Send API) | Implementing adapters (that's M1) |
| **Map nodes** — process variable-length lists with concurrency limits | Managing skill registry (that's M2) |
| **Consent gate orchestration** — `interrupt()` → checkpoint → `Command(resume=)` | Consent UI rendering (that's M9) |
| **Checkpointing** — all state persisted to Postgres via `langgraph-checkpoint-postgres` | Memory retrieval/storage (that's M7, but M5 triggers reads/writes at lifecycle hooks) |
| **Idempotency** — content-addressable hashing, skip re-execution of completed nodes | |
| **Error recovery** — per-node retry with backoff, fallback skill invocation, partial plan recovery | |
| **Run cancellation / abort** — graceful mid-execution termination | |
| **Timeout management** — per-node and per-plan timeouts | |
| **Resource limits** — max concurrent nodes, max API calls per plan | |
| **SSE event streaming** — emit typed events for every lifecycle transition | |
| **Batch execution** — multi-item, multi-stage batch processing | |
| **SLA tracking** — deadline monitoring, escalation triggers | |
| **Warm restart** — resume from last checkpoint after crash | |
| **Execution analytics** — per-plan duration, cost, success metrics | |
| **Finalize hook** — trigger post-execution writes (episodes to M7, telemetry to M2) | |

#### Interfaces Exposed

```
Executor.run(plan: PlanSpec, context: ExecutionContext) → ExecutionResult
Executor.resume(run_id, user_decision) → ExecutionResult
Executor.cancel(run_id) → void
Executor.get_status(run_id) → RunStatus
ExecutionEventStream.subscribe(run_id) → AsyncIterator[Event]

# Lifecycle hooks (called by executor, implemented by other modules)
on_before_plan(context) → AssembledContext           # M7 reads
on_node_complete(node_result) → void                 # M2 telemetry
on_execution_complete(plan, results) → void          # M7 writes (episodes, facts)
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M1 (Adapters) | `adapter.invoke()` to run each node |
| M2 (Skill Management) | Skill lookup for adapter routing, telemetry recording |
| M3 (Task Decomposition) | Receives PlanSpec to execute. Calls planner for partial re-plan on modification. |
| M4 (Ranking) | Provider selection when node has multiple provider options |
| M7 (Memory) | Context assembly before planning (lifecycle hook). Episode/fact writes after execution (lifecycle hook). |
| M11 (Security) | Consent enforcement, PII checks before external calls |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| `agent_runs` | Execution state — run_id, status, approvals, clarifications |
| `agent_run_events` | Event timeline — seq-ordered events per run |
| `checkpoints` / `checkpoint_blobs` / `checkpoint_writes` (public schema) | LangGraph state persistence |
| `bot_tasks` | Per-task state — SLA, output, escalation |

---

### M6: New Skill Creation (On the Fly)

**Mission:** Enable the system to autonomously discover, generate, test, and integrate new skills at runtime — from API specs, MCP registries, or user demonstrations.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| API spec discovery (OpenAPI, AsyncAPI crawling) | Running discovered skills in production (that's M5 + M1) |
| MCP server discovery and connection | Managing the existing registry (that's M2) |
| Auto-generate SkillDefinition from API spec | |
| Dynamic skill registration without restart | |
| Skill composition — combine existing skills into new ones | |
| User-defined automation workflows ("every morning: weather → calendar → briefing") | |
| Safety sandbox — run untrusted skills with resource limits | |
| Quality gate — auto-test generated skills before promotion | |
| Skill marketplace — browse/install community skills | |

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M2 (Skill Management) | Registry.register() to add discovered skills |
| M1 (Adapters) | MCPConfigManager to connect new MCP servers |
| M11 (Security) | Sandbox execution, trust verification |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| Discovered API spec cache | Raw specs from crawling |
| Skill draft storage | Generated but not yet promoted skills |
| Marketplace index | Community skill catalog |

**Note:** This module is future R&D. It has clear interfaces but no near-term deliverables.

---

### M7: User Memory

**Mission:** Make the assistant smarter with every interaction. Assemble the right context for every decision point. Remember across sessions. Never dump — curate.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| **Layer 1 — Conversation Buffer** — load recent messages, summarize older ones | Chat UI (that's M9) |
| **Layer 2 — Working Memory** — `$memory.<node_id>.<field>` inter-node data flow | Graph execution (that's M5, but M5 uses Layer 2) |
| **Layer 3 — Episodic Memory** — generate episode summaries after execution, vector search for relevant past work | Deciding what skills to use (that's M3) |
| **Layer 4 — Semantic Memory (mem0)** — extract durable user facts, semantic retrieval, deduplication | |
| **Layer 5 — Procedural Memory** — surface skill_data_store contents as summaries for planner | |
| **Context Assembly Engine** — compose the right layers for each decision point (GID, Planner, Executor, Finalize) | |
| **Three-Read Pattern** — conversation buffer + episodes/facts + procedural summary before every plan | |
| **Three-Write Pattern** — AI response + episode + facts after every execution | |
| **Message summarization** — hybrid: recent verbatim + older summarized | |
| **Relevance filtering** — semantic similarity + recency ranking for memory retrieval | |
| **Memory decay & archival** — old episodes → semantic facts → archive | |
| **Memory conflict resolution** — handle contradictions ("cancelled my HDFC card" vs old fact) | |
| **Token budget management** — dynamic allocation across layers within ~4K token budget | |
| **User memory dashboard API** — view/edit/delete stored memories | |

#### Interfaces Exposed

```
# Context assembly (consumed by M3 and M5)
ContextAssembler.assemble_for_gid(user_id, message) → GIDContext
ContextAssembler.assemble_for_planner(user_id, message, chat_id) → PlannerContext
ContextAssembler.assemble_for_synthesis(user_id, query) → SynthesisContext

# Post-execution writes (called by M5 finalize hook)
MemoryWriter.write_episode(user_id, plan, results) → void
MemoryWriter.extract_and_store_facts(user_id, interaction) → void
MemoryWriter.write_response_to_chat(chat_id, response) → void

# Procedural memory (direct access for skills at execution time)
SkillDataStore.write(user_id, namespace, record_type, payload, recorded_at) → str
SkillDataStore.query(user_id, namespace, record_type, filters) → list[dict]
SkillDataStore.query_latest(user_id, namespace, record_type) → dict | None

# Semantic memory
SemanticMemory.search(user_id, query, limit) → list[MemoryItem]
SemanticMemory.store(user_id, content, metadata) → void
SemanticMemory.get_all(user_id) → list[MemoryItem]
SemanticMemory.delete(memory_id) → void

# User-facing
MemoryDashboard.get_user_memories(user_id) → MemorySummary
MemoryDashboard.delete_memory(user_id, memory_id) → void
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| None at runtime | Memory is a foundational service. It reads from its own tables and provides assembled context to consumers. |

**Note:** M7 has no upstream dependencies. It is consumed by M3 (context for planning), M5 (lifecycle hooks for reads/writes), and M8 (learning from history). This makes it a high-priority standalone module.

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| `chat_messages` (read) | Layer 1 — conversation buffer (owned by chat system, read by M7) |
| `execution_episodes` (new) | Layer 3 — episode summaries with vector embeddings |
| mem0 / pgvector tables | Layer 4 — semantic memory storage |
| `skill_data_store` | Layer 5 — structured procedural data |

---

### M8: Agent Intelligence

**Mission:** Make the agent genuinely intelligent — not just a plan executor, but an entity that reasons about autonomy, learns from outcomes, acts proactively, and explains its decisions.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| **Autonomy levels** — user-configurable per-domain: "always ask" / "auto for scheduling" / "full auto" | Consent gate mechanics (that's M5 + M11) |
| **Automatic channel selection** — rules engine for "contact them" → evaluate channels → pick best | Channel implementation (that's M10) |
| **Channel escalation** — SLA-driven escalation chains (WhatsApp → call → voicemail) | Plan creation (that's M3) |
| **Proactive intelligence** — agent initiates actions without user trigger (alerts, conflict detection, pattern recognition) | |
| **Preference learning** — infer preferences from behavior (always picks cheapest → "budget-conscious") | |
| **Goal inference** — understand implicit goals ("meeting at 9" → check traffic, set alarm) | |
| **Strategy persistence** — remember which approaches work per user/contact | |
| **Feedback loop** — user corrections become persistent rules | |
| **Delegation authority matrix** — per-domain, per-action autonomy config | |
| **Explainability engine** — "why did you choose IndiGo?" → reasoning trace | |
| **Uncertainty quantification** — low confidence → ask rather than guess | |
| **PII detection** — scan content before external API calls, redact/mask | |

#### Interfaces Exposed

```
Intelligence.select_channel(user_id, contact, urgency, context) → ChannelRecommendation
Intelligence.get_autonomy_level(user_id, domain, action) → AutonomyLevel
Intelligence.should_auto_approve(user_id, consent_request) → bool
Intelligence.get_proactive_actions(user_id, context) → list[ProactiveAction]
Intelligence.explain_decision(run_id, node_id) → Explanation
Intelligence.record_feedback(user_id, feedback) → void
Intelligence.check_pii(content, target_service) → PIICheckResult
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M7 (User Memory) | User preferences, episodic history, behavioral patterns |
| M2 (Skill Management) | Skill capabilities for channel selection |
| M5 (Execution Engine) | Execution results for strategy learning, reasoning traces for explainability |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| Autonomy config per user | Domain-action autonomy matrix |
| Feedback rules per user | Persistent corrections ("never WhatsApp my boss") |
| PII detection config | Patterns, sensitivity classifications |

---

### M9: Front-End / UX

**Mission:** Give users full visibility and control over what the agent is doing, has done, and knows. The UI should make the agent's intelligence tangible.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| Chat UI — conversational interface, message rendering, rich text | Backend API design (M9 consumes APIs from M5, M7, M8) |
| SSE event consumption — real-time progress display | SSE event production (that's M5) |
| Consent cards — approve/edit/deny with draft editing | Consent enforcement logic (that's M11) |
| Planning view — show plan being built, step-by-step | Plan creation logic (that's M3) |
| Structured artifact rendering — flight cards, hotel cards, itineraries, comparisons, briefings, digests | Artifact data production (that's domain skills) |
| Task abort UI — cancel mid-execution with confirmation | Abort mechanics (that's M5) |
| Reasoning/citations display — show how agent decided | Reasoning generation (that's M8) |
| Batch progress visualization — N tasks, per-item status | |
| Memory dashboard — view/edit/delete agent's knowledge about you | Memory storage (that's M7) |
| Skill browser — browse capabilities by category | Skill registration (that's M2) |
| Notification management — frequency, channels, quiet hours | Push delivery (that's M10) |
| Multi-modal input — voice, image, document upload | Document parsing (that's backend) |
| Proactive suggestion cards — agent suggests actions | Proactive logic (that's M8) |
| History/timeline — browse past executions | Execution storage (that's M5) |
| Preferences/settings UI — autonomy, channels, connected accounts | |

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M5 (Execution) | SSE event stream, run status, execution history |
| M7 (Memory) | Memory dashboard data |
| M8 (Intelligence) | Proactive suggestions, explainability data |
| M2 (Skill Management) | Skill catalog for browser |
| M11 (Security) | Auth session, consent request data |

#### Data Owned

Frontend-local state only. No backend tables owned by M9. All persistent data comes from backend modules via APIs.

---

### M10: Communication Skills

**Mission:** Enable the agent to reach any person through any channel — voice, text, email — with unified context, format adaptation, and delivery tracking.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| **Voice calls (Twilio)** — outbound/inbound, AI voice agent, recording, transcription, DTMF, conferencing | Choosing which channel to use (that's M8) |
| **Voice calls (Knowlarity)** — alternative telephony provider | Execution orchestration (that's M5) |
| **WhatsApp (Meta/Pinnacle)** — send/receive text, media, templates, interactive messages | Skill registration (that's M2) |
| **WebRTC (LiveKit)** — browser-based calls, link sharing, multi-participant | |
| **Voice cloning** — user voice profiles, multi-provider (ElevenLabs, CosyVoice, Cartesia) | |
| **Email (Gmail/SMTP)** — read, compose, send, search, thread management, attachments | |
| **SMS** — send, delivery tracking, two-way, MMS | |
| **Slack** — post messages, DMs, thread replies, rich formatting | |
| **Telegram** — bot integration, rich media | |
| **Unified channel abstraction** — single interface, multiple backends | |
| **Cross-channel context** — call summary → WhatsApp follow-up with full context | |
| **Rich media handling** — images, documents, voice notes adapted per channel constraints | |
| **Delivery tracking** — delivery/read receipts, retry on failure | |
| **Smart templates** — dynamic message drafting per channel constraints | |
| **Handler functions** — the actual Python handlers registered in DirectAdapter for each channel skill | |

#### Interfaces Exposed

Each channel exposes skills registered in M2 and invoked via M1 adapters:
```
channel.whatsapp.send(recipient, message, media?) → DeliveryResult
channel.call.initiate(recipient, purpose, voice_profile?) → CallResult
channel.email.send(recipient, subject, body, attachments?) → DeliveryResult
channel.sms.send(recipient, message) → DeliveryResult
channel.slack.post(channel, message, thread?) → DeliveryResult
channel.webrtc.create_room(participants) → RoomResult

# Unified interface (implemented by M8 intelligence + M10 handlers)
channel.contact.reach(contact_id, message, urgency) → DeliveryResult
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| M1 (Adapters) | DirectAdapter dispatches to channel handlers |
| M2 (Skill Management) | Skill registration for each channel capability |
| M11 (Security) | Consent enforcement for side-effecting communications |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| `calls` + `call_memberships` + `recordings` + `transcripts` + `transcript_segments` | Voice pipeline |
| `whatsapp_conversations` + `whatsapp_messages` + `whatsapp_schedules` | WhatsApp pipeline |
| `voice_profiles` + `voip_numbers` | Voice cloning and telephony |
| `contacts` + `inbox_assignments` | Contact directory and channel routing |

---

### M11: Security, Privacy, Consent

**Mission:** Ensure every action is authorized, every data flow is auditable, and user data is protected at every boundary. This module is cross-cutting — it wraps around everything else.

#### Scope

| In scope | NOT in scope |
|----------|-------------|
| **Authentication** — JWT auth (Supabase), Firebase push auth, session management | Chat UI login flow (that's M9) |
| **Consent lifecycle** — request creation, user decision, enforcement, audit | Consent card rendering (that's M9) |
| **Row-Level Security** — RLS policies on all tables, per-user data isolation | |
| **PII detection & redaction** — scan prompts/data before external API calls | Intelligence decisions about PII (that's M8, but enforcement is M11) |
| **Data retention policies** — auto-expire by data class (recordings: 90d, messages: 1y) | |
| **Audit trail** — every consent decision, data access, external API call logged | |
| **Compliance** — GDPR (right to access, erasure, portability), India DPDP Act | |
| **Rate limiting / abuse prevention** — per-user, per-skill, per-IP limits | |
| **API key management** — storage, rotation, usage tracking | |
| **Encryption** — application-level encryption of sensitive fields (financial, PII) | |
| **Input validation** — prompt injection defense, SQL injection prevention, sanitization | |
| **Role-Based Access Control** — admin/user/viewer roles, org support | |
| **Security logging & alerting** — anomaly detection, suspicious activity alerts | |
| **Account deletion** — GDPR-compliant full data removal | |
| **Data classification** — sensitivity levels (public/internal/confidential/restricted) | |
| **Third-party data policies** — per-service data sharing rules | |

#### Interfaces Exposed

```
Auth.authenticate(request) → AuthenticatedUser
Auth.authorize(user, action, resource) → bool

Consent.create_request(user_id, action, draft, risk_level) → ConsentRequest
Consent.enforce(skill_id, requires_consent, context) → Allowed | Interrupted
Consent.record_decision(request_id, decision) → void

PII.scan(content) → PIIScanResult
PII.redact(content, policy) → RedactedContent

RateLimit.check(user_id, skill_id) → Allowed | RateLimited
AuditLog.record(event_type, actor, action, resource, metadata) → void
DataRetention.enforce() → void  # scheduled job
GDPR.export_user_data(user_id) → DataExport
GDPR.delete_user(user_id) → DeletionResult
```

#### Dependencies

| Depends on | For what |
|-----------|---------|
| None | Security is foundational. Other modules depend on it, not the reverse. |

#### Data Owned

| Table / Store | Purpose |
|--------------|---------|
| `consent_requests` | Consent lifecycle tracking |
| `user_permissions` + `permission_events` | Permission management and audit |
| `otp_events` | OTP security audit |
| `account_deletion_log` | GDPR deletion records |
| `audit_log` (new) | Unified cross-module audit trail |

---

## Dependency Graph

```
                    DEPENDS ON (runtime) ──▶

 ┌───────────────────────────────────────────────────────────────┐
 │ FOUNDATION LAYER  (zero upstream dependencies)                │
 │                                                               │
 │   M2: Skill Mgmt     M7: User Memory     M11: Security       │
 │                                                               │
 └──────────┬──────────────────┬──────────────────┬──────────────┘
            │                  │                  │
            ▼                  ▼                  ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ ORCHESTRATION LAYER                                          │
 │                                                              │
 │   M3: Task Decomposition ──────▶ M5: Execution Engine        │
 │       depends: M2, M7               depends: M1, M2, M3,    │
 │                                              M4, M7, M11     │
 │                                                              │
 │   M4: Ranking & Selection                                    │
 │       depends: M2, M7                                        │
 └──────────────────────────────────┬───────────────────────────┘
                                    │
                                    ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ CAPABILITY LAYER                                             │
 │                                                              │
 │   M1: Adapter Framework          M10: Communication Skills   │
 │       depends: M2                     depends: M1, M2, M11   │
 │       (invoked BY M5)                 (handlers live in M1)  │
 └──────────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────┐
 │ CONSUMER LAYER                                               │
 │                                                              │
 │   M8: Agent Intelligence          M9: Front-End / UX         │
 │       depends: M7, M2, M5            depends: M5, M7, M8,   │
 │                                               M2, M11        │
 │   M6: New Skill Creation (future)                            │
 │       depends: M2, M1, M11                                   │
 └──────────────────────────────────────────────────────────────┘
```

### How to read this

- **Foundation Layer** modules have **zero upstream dependencies**. They can be built first and in parallel. Everything else consumes them.
- **Orchestration Layer** is where plans are created (M3) and executed (M5). M5 is the integration hub with the most dependencies.
- **Capability Layer** is where external systems are actually called. M1 provides the uniform `invoke()` interface; M10 provides channel-specific handler logic.
- **Consumer Layer** modules sit on top. M8 adds intelligence, M9 presents to users, M6 is future R&D.

**Key insight:** M2, M7, and M11 are load-bearing — if these are solid, everything above them can be built independently.

---

## Ownership Model

### Option A: 6 developers, module clusters

| Cluster | Modules | Rationale |
|---------|---------|-----------|
| **Orchestration Core** | M3 (Decomposition) + M5 (Execution) | Tightly coupled — PlanSpec is the contract between them. Same developer understands the full plan→execute→finalize lifecycle. |
| **Skill Ecosystem** | M2 (Skill Mgmt) + M4 (Ranking) + M1 (Adapters) | Registry, ranking, and adapter invocation form a vertical. One developer owns "how skills are defined, selected, and called." |
| **Intelligence & Memory** | M7 (Memory) + M8 (Intelligence) | Intelligence is built ON memory. One developer owns "how the agent gets smarter." |
| **Communication** | M10 (Communication Skills) | All channels. Voice, WhatsApp, email, SMS, Slack. Large surface area but self-contained — each channel is independent. |
| **Frontend** | M9 (Front-End / UX) | Separate repo, separate skillset (React Native). |
| **Platform Security** | M11 (Security) | Cross-cutting but can be built independently — provides interfaces that other modules call. |

M6 (New Skill Creation) is deferred. When it starts, it can be owned by the Skill Ecosystem developer.

### Option B: 4 developers, larger clusters

| Cluster | Modules |
|---------|---------|
| **Core Engine** | M1 + M2 + M3 + M4 + M5 |
| **Intelligence** | M7 + M8 + M11 |
| **Channels** | M10 |
| **Frontend** | M9 |

### Option C: 3 developers (current team)

| Developer | Modules | Rationale |
|-----------|---------|-----------|
| **Avinash** | M2, M3, M4, M5, M7, M8, M11 | Orchestration core + intelligence. Architect-level ownership of the engine and its brain. |
| **Krishna** | M1, M10 | Adapters + all communication channels. Deep telephony/WebRTC/voice expertise. |
| **Rishav** | M9 + domain skills (handler functions) | Frontend + skill handler implementations (the actual business logic inside DirectAdapter). |

---

## Interface Contracts Between Modules

The key contracts that define module boundaries:

| Contract | Producer | Consumer | Shape |
|----------|----------|----------|-------|
| `SkillDefinition` | M2 | M1, M3, M4, M5 | Pydantic model — skill metadata, schemas, flags |
| `PlanSpec` | M3 | M5 | JSON — task graph with dependencies, inputs, consent flags |
| `SkillResult` | M1 | M5 | Pydantic model — output, status, duration, artifacts |
| `AssembledContext` | M7 | M3 | Dict — conversation buffer, episodes, facts, procedural summary |
| `ConsentRequest` | M5 | M11, M9 | Pydantic model — action, draft, risk level, decision |
| `ExecutionEvent` | M5 | M9 | JSON via SSE — typed event stream (plan_created, node_started, etc.) |
| `SkillStats` | M2 | M4 | Pydantic model — success_rate, latency_ms, cost, sample_count |
| `RankedSkillList` | M4 | M5 | Ordered list of skill_ids with scores |
| `ChannelRecommendation` | M8 | M5 | Selected channel + reasoning |
| `StructuredArtifact` | Domain skills (via M1) | M9 | JSON — artifact_type + typed payload for mobile rendering |

---

## Data Ownership Summary

Every table has exactly one owning module. Other modules may read but should not write directly.

| Table | Owner | Readers |
|-------|-------|---------|
| `user_profiles` | M11 | All |
| `chats` + `chat_messages` | M7 (read), M9 (write via API) | M3, M5 |
| `agent_runs` + `agent_run_events` | M5 | M9 |
| `bot_tasks` | M5 | M9 |
| `batches` | M5 | M9 |
| `checkpoints` + `checkpoint_*` | M5 | — |
| `skill_data_store` | M7 | M1 (skill handlers read at execution time) |
| `scheduled_jobs` | M5 | M3 (registers jobs via skill handlers) |
| `consent_requests` | M11 | M5, M9 |
| `calls` + `call_memberships` + `recordings` | M10 | M5, M9 |
| `transcripts` + `transcript_segments` | M10 | M7 (for fact extraction), M9 |
| `whatsapp_*` | M10 | M7, M9 |
| `voice_profiles` + `voip_numbers` | M10 | M9 |
| `contacts` + `inbox_assignments` | M10 | M8 (channel selection), M9 |
| `user_permissions` + `permission_events` | M11 | M5 |
| `otp_events` | M11 | — |
| `account_deletion_log` | M11 | — |
| `app_configurations` | M1 | All |
| `execution_episodes` (new) | M7 | M3, M8 |
