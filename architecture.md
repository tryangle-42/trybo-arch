# Trybo: Agentic Orchestration Architecture

## 1. What Trybo Is

Trybo is a **mobile-first personal assistant that executes real-world tasks**. It decomposes a user goal into subtasks, selects the right tool/agent for each from a registry, executes them with full observability, and requires user approval before any side-effecting action.

It is not a chatbot wrapper. It is not a fixed-workflow automation tool. It is an **Agent-of-Agents OS**: an orchestration engine that routes tasks to the right capability at runtime, with human-in-the-loop governance baked in.

**Core architectural moat:**
- Dynamic task graph creation at runtime (not pre-defined workflows)
- Registry-driven tool/agent selection (planner cannot hallucinate tool names)
- Mandatory consent gates on all side-effects
- Bidirectional execution (same engine for outbound tasks and inbound events)
- Persistent structured data collection across sessions (proprietary data assets per user)

---

## 2. Existing Stack

| Layer | Tech | Notes |
|---|---|---|
| Mobile UI | React Native | Chat-first, OTP login via Twilio + Supabase |
| Backend | FastAPI (Python) | Hosts the entire orchestration engine |
| Data/Auth/Storage | Supabase (Postgres) | Source of truth. Schema migrations managed by UI repo — FastAPI assumes schema exists. |
| Chat system | Supabase Realtime | Last 20 messages as LLM context; paginated history |
| Auth | Mobile OTP | Twilio SMS → Supabase session |
| Existing services | WhatsApp (GreenAPI), Voice calls (Twilio + Knowlarity), Intent detection, Batch tasks | Pre-orchestration services. Orchestration engine wraps these as Direct adapter skills. |

---

## 3. Orchestration Engine — Full Architecture

The orchestration engine lives entirely in FastAPI. It replaces the flat "trigger a batch of tasks" model with a dynamic, plan-driven execution engine.

### 3.1 Request Flow

```
User message (chat) or Scheduled trigger or Inbound webhook
  │
  ▼
GID (Global Intent Detector)
  ├─ FAST_PATH  → single skill, no side-effect, confidence > 0.85 → Execute directly
  ├─ PLANNER    → multi-step, side-effects, or ambiguous → Send to Planner
  └─ CLARIFY    → missing required entity → Ask user
        │
        ▼
Planner (LLM)
  → Reads: user goal, conversation history, user memory (mem0), available skills (from Registry)
  → Produces: validated PlanSpec (task graph JSON)
  → Validation: all skill_ids exist in registry, inputs match schemas, no cycles, consent flags match
        │
        ▼
Executor (LangGraph — two-tier architecture)
  │
  ├─ Outer Graph: fixed lifecycle pipeline
  │   normalize → clarify → plan → validate → execute → finalize
  │
  └─ Inner Graph: dynamically compiled from PlanSpec
      → Each task becomes a LangGraph node
      → Independent nodes execute in parallel (fan-out)
      → Map nodes process variable-length lists concurrently
      → Side-effect nodes: interrupt() → Consent gate → resume on approval
      → Data flows between nodes via $memory.<node_id>.<field> references
      → On failure: retry → fallback skill → surface to user
        │
        ▼
Results surfaced in chat as Structured Artifacts (typed cards rendered by mobile)
Skill-produced data persisted to Skill Data Store
Learned facts stored to mem0
Audit log written per node
```

### 3.2 Module Map

```
/orchestration
  /gid               # Global Intent Detector (semantic-router)
  /planner            # LLM planner + PlanSpec validator + partial re-planner
  /executor           # LangGraph graph builder + runner (two-tier)
  /registry           # Skill registry CRUD + health monitor
  /consent            # Consent service + approval artifacts
  /memory             # mem0 integration (semantic, unstructured)
  /persistence        # Skill Data Store (structured, queryable)
  /scheduler          # Scheduled job executor (cron-like)
  /adapters
    /direct           # In-house tools (WhatsApp, calls, WebRTC, push notifications)
    /mcp              # MCP server clients (MultiServerMCPClient, config-driven)
    /marketplace      # External services (Composio SDK)
  /models             # Shared Pydantic models (PlanSpec, SkillResult, TaskContext, etc.)
  /audit              # Audit log + event taxonomy + SSE streaming
```

---

## 4. Global Intent Detector (GID)

The entry point for every user message. Routes to the cheapest execution path.

**Implementation:** `semantic-router` library — embedding-based classification, sub-10ms latency.

**Inputs:** raw text, conversation history (last 20 msgs), user memory (from mem0)
**Outputs:** `intent_type`, `entities`, `confidence_score`, `routing_decision`

| Routing Decision | Trigger | Action |
|---|---|---|
| `FAST_PATH` | Single skill needed, no side-effect, confidence > 0.85, no ambiguous entities | Execute skill directly, skip planner |
| `PLANNER` | Multi-step decomposition needed, any side-effect present, low confidence | Full planner pipeline |
| `CLARIFY` | Missing required entity (e.g., "send a message" with no recipient), conflicting instructions | Ask user for clarification |

**Entity extraction:** LLM call, triggered only on `PLANNER` and `CLARIFY` paths (not on fast path).

**Performance target:** Fast path must complete end-to-end in sub-500ms. Planner path can take 2–5s.

---

## 5. Planner

Decomposes a user goal into a validated task graph.

**Inputs:** user goal, conversation history, user memory (mem0), available skills (filtered from registry — only `health_status: active` skills)
**Outputs:** validated `PlanSpec` (JSON)

### 5.1 PlanSpec Schema

```json
{
  "plan_id": "uuid",
  "goal": "string",
  "tasks": [
    {
      "task_id": "string",
      "skill_id": "string (must exist in registry)",
      "node_type": "tool | agent | map",
      "input": { "resolved params, may contain $memory.<node_id>.<field> references" },
      "depends_on": ["task_id"],
      "requires_consent": true | false,
      "fallback_skill_id": "string | null",
      "map_config": {
        "inputs_from": "$memory.<node_id>.<field>",
        "concurrency_limit": 10
      }
    }
  ]
}
```

### 5.2 Plan Validation (Hallucination Control)

1. LLM generates plan draft in structured JSON
2. Validator checks:
   - All `skill_id` values exist in the registry
   - Input params match each skill's `input_schema`
   - No circular dependencies (cycle detection)
   - `requires_consent` matches the registry value (plan cannot escalate privileges)
3. If validation fails: re-prompt LLM with error context (max 3 retries), then surface clarification to user
4. Graceful degradation: if a skill is missing, remove the node and cascade-remove dependents
5. Graph is built in **deterministic Python code** from the validated PlanSpec — the LLM does not write graph code

### 5.3 Skill Selection Ranking

When multiple skills match a capability:

```
score = (success_rate * 0.4) + (1/latency_normalized * 0.25) + (cost_score * 0.2) + (consent_overhead_penalty * 0.15)
```

### 5.4 Fallback Chain

Per task, on failure:
1. Retry same skill with backoff (max 2x)
2. Try `fallback_skill_id` if set
3. If all fail: pause graph, surface to user with available alternatives
4. On consent denial: find no-side-effect alternative or return failure artifact

### 5.5 Partial Re-planning

The planner supports mid-flight plan modification: accept a completed/in-progress PlanSpec + a modification request → preserve already-completed node results → re-plan only the affected subtree → resume executor from the modification point. This enables users to pivot mid-task without losing work (e.g., swap a hotel in a trip plan, change a commute destination).

### 5.6 Clarification Phase

Before generating a plan, the planner runs a clarification check: asks the LLM if critical info is missing. If so, routes to the user for clarification before producing a PlanSpec. This prevents wasted planning on incomplete inputs.

---

## 6. Executor (LangGraph)

LangGraph is the runtime. The executor uses a **two-tier graph architecture**:

- **Outer graph:** Fixed lifecycle pipeline — `normalize → clarify → plan → validate → execute → finalize`. This is the same for every request.
- **Inner graph:** Dynamically compiled from the validated PlanSpec. Each task becomes a LangGraph node. This varies per request.

### 6.1 Task State Machine

```
PLANNED → RUNNING → WAITING_FOR_APPROVAL → EXECUTING_SIDE_EFFECT → DONE
                                                                    → FAILED → RETRYING → DONE / FAILED
                  → WAITING_FOR_CLARIFICATION → REPLANNING → RUNNING
```

### 6.2 Node Types

| Node Type | Behavior |
|---|---|
| **Tool node** | Calls `adapter.invoke()` → returns result → no LLM reasoning |
| **Agent node** | Wraps a sub-LLM loop (ReAct) with its own tool access. Used for complex subtasks like "research 3 vendors and extract pricing." |
| **Map node** | Takes a variable-length input list + a skill_id. Spawns N parallel skill invocations with configurable concurrency limit. Collects results into a list. Used for processing N emails, N product pages, N flight options, etc. |
| **Consent node** | Emits approval artifact → LangGraph `interrupt()` → graph suspends → graph state checkpointed to Postgres → resumes via `Command(resume=)` on user action |
| **Condition node** | Routes graph based on prior result (e.g., "has phone? → call : has email? → email") |

### 6.3 Parallel Execution

- **Fan-out:** Independent nodes (no dependency between them) execute in parallel using LangGraph's `Send` API or parallel state branches.
- **Map:** Map nodes process variable-length lists concurrently with a configurable concurrency cap.
- **Fan-in:** Results from parallel branches merge back into `TaskContext` before dependent nodes execute.

### 6.4 Data Flow Between Nodes

Nodes communicate via `$memory.<node_id>.<field>` late-binding references in the PlanSpec. At execution time, these resolve to actual values from prior node outputs stored in `TaskContext.task_results`. This is cleaner than passing raw data through the planner.

### 6.5 Shared Working Memory (per graph execution)

```python
class TaskContext:
    plan_id: str
    user_id: str
    task_results: dict[str, SkillResult]    # node outputs, keyed by task_id
    conversation_history: list[Message]
    user_memory: MemoryContext               # from mem0
    approval_artifacts: list[ApprovalArtifact]
    audit_log: list[AuditEntry]
    device_context: dict                     # location, push token, timezone, etc.
```

Every node reads from and writes to `TaskContext`.

### 6.6 Checkpointing & Resumability

- LangGraph checkpoints persisted to **Supabase via `langgraph-checkpoint-postgres`** — not in-memory
- Each checkpoint: `(plan_id, thread_id, state_snapshot, timestamp)`
- On app disconnect: graph suspends at current node
- On reconnect: load latest checkpoint → resume from last incomplete node
- Consent interrupts checkpoint automatically — graph survives process restarts

### 6.7 Idempotency

Content-addressable hashing per node execution (`skill_id + input hash`). On graph re-execution (e.g., after crash recovery), already-completed nodes with matching hashes skip re-execution. Prevents duplicate side effects.

---

## 7. Skill / Agent Registry

Single source of truth for everything the system can invoke. The planner only uses skills from the registry — it cannot hallucinate tool names.

### 7.1 Skill Schema

```json
{
  "id": "string (unique)",
  "name": "string",
  "kind": "tool | agent | mcp",
  "adapter_type": "direct | mcp | marketplace",
  "capabilities": ["capability.verb.noun"],
  "description": "string (used by LLM planner for selection)",
  "input_schema": { "JSON Schema" },
  "output_schema": { "JSON Schema" },
  "side_effect": true | false,
  "requires_consent": true | false,
  "consent_risk_level": "low | medium | high",
  "auth_required": true | false,
  "auth_scopes": ["string"],
  "success_rate": 0.0-1.0,
  "p50_latency_ms": "integer",
  "cost_tier": "free | low | medium | high",
  "rate_limits": { "rpm": "integer", "daily": "integer" },
  "health_status": "active | degraded | disabled",
  "last_health_check": "ISO timestamp",
  "fallback_skill_ids": ["string"]
}
```

### 7.2 Registry Health

- **Synthetic canary:** Auth check + lightweight probe per skill on schedule
- **Passive telemetry:** Failure rate, schema drift, auth errors per skill (from audit log)
- **Auto-degrade:** `health_status` → `degraded` or `disabled` on threshold breach
- **Planner isolation:** Planner never receives `disabled` skills in its available-skills list

### 7.3 Skill Registration

Skills are registered via `seed_registry.py` — configuration-as-code. Each skill is a Pydantic `SkillDefinition` object with full JSON schemas for inputs/outputs. New skills are added by appending to the seed file — no code changes to the orchestration engine.

---

## 8. Adapter Layer

All adapter types expose a single interface to the executor:

```python
class SkillAdapter(ABC):
    async def invoke(self, skill_id: str, input: dict, context: TaskContext) -> SkillResult
```

### 8.1 Direct Adapter

In-house tools with full control. Each skill maps to a handler function that calls an existing service:

- WhatsApp send/receive (GreenAPI)
- Voice calls (Twilio + Knowlarity)
- WebRTC room creation + SMS link
- Push notifications (FCM/APNs)
- LLM summarize/analyze
- Google Maps Directions (direct HTTP)
- RSS/Feed aggregation (feedparser + httpx)
- Document text extraction (Kreuzberg)
- Structured entity extraction (Instructor + Pydantic)

### 8.2 MCP Adapter

**Config-driven, not code-driven.** A single generic adapter handles all MCP skills using `langchain-mcp-adapters` `MultiServerMCPClient`:

```python
MCP_SERVERS = {
    "tavily":            {"url": "https://mcp.tavily.com/mcp/...", "transport": "http"},
    "google-workspace":  {"command": "npx", "args": ["-y", "@anthropic/google-workspace-mcp"], "transport": "stdio"},
    "slack":             {"url": "https://mcp.slack.com/mcp", "transport": "http"},
    "weather":           {"command": "...", "transport": "stdio"},
    "playwright":        {"command": "...", "transport": "stdio"},
    "notion":            {"command": "npx", "args": ["-y", "@notionhq/notion-mcp-server"], "transport": "stdio"},
    "github":            {"command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"], "transport": "stdio"},
}
```

Adding a new MCP skill = adding one entry to this config + one `SkillDefinition` to the seed registry. No new adapter code.

### 8.3 Marketplace Adapter

External HTTP agents/services via `composio-langgraph` SDK:

- Handles OAuth token refresh, tool discovery, and multi-app support
- Currently: Composio (Slack, Notion, GitHub, Linear, Salesforce, etc.)
- Extensible to any external HTTP agent with an agent card

### 8.4 Agent Adapter

For external agents that are genuinely different from tools (async, bidirectional lifecycle):

- `start()` + `forward_answer()` lifecycle (breaks the standard `invoke()` interface — intentionally)
- Bidirectional message channels (`AgentChannel`)
- HTTP-based agent communication protocol

---

## 9. Consent / HITL Layer

**No side-effect action executes without passing through the consent gate.** This is enforced by the executor, not optional.

### 9.1 Implementation

Uses LangGraph's native `interrupt()` / `Command(resume=)`:
- Side-effect node emits approval artifact → `interrupt()` suspends the graph
- Graph state checkpoints to Postgres (survives process restarts)
- User approves/edits/denies via mobile UI
- `Command(resume=)` resumes the graph with the user's decision

### 9.2 Consent Service Contract

**Input:**
```json
{
  "action_type": "whatsapp_send | email_send | call_place | order_place | ...",
  "draft_content": "string",
  "recipient": { "name": "string", "channel": "string", "id": "string" },
  "risk_level": "low | medium | high",
  "tool_id": "string",
  "context": { "task_id": "string", "plan_id": "string" }
}
```

**Output:**
```json
{
  "decision": "APPROVED | EDIT_REQUIRED | DENIED",
  "edited_content": "string | null",
  "approval_artifact_id": "uuid",
  "timestamp": "ISO"
}
```

### 9.3 Approval Policies

| Condition | Behavior |
|---|---|
| `risk_level = low` + previously approved same `action_type` + same recipient | Auto-approve |
| `risk_level = medium` | Surface artifact → require explicit tap-to-approve |
| `risk_level = high` (new recipient, PII detected, financial action) | Require explicit approval + confirmation text |
| Batch actions (e.g., 5 recipients) | Single batch approval card showing all recipients |
| Post-approval failure | Retry idempotently; if not idempotent, re-ask |
| Consent denied | Planner selects fallback no-side-effect skill or surfaces options |

### 9.4 Mobile UI for Consent

- **Outbound approvals:** Artifact card in chat (draft content, recipient, action type, Edit / Approve / Deny buttons)
- **Inbound approvals** (e.g., incoming call response): Pre-filled response text, Send / Ignore
- "To respond" badge count visible in nav tab
- Edit-by-default: tapping an artifact opens an editable draft, not a read-only view

---

## 10. Memory Layer

Three tiers of memory, each serving a different purpose:

### 10.1 Short-Term Memory

Last 20 messages in Supabase `messages` table (already exists). Passed as conversation history to GID and Planner.

### 10.2 Working Memory (per graph execution)

`TaskContext` — shared across all nodes in a single graph execution. Contains node outputs (`task_results`), approval artifacts, user memory snapshot. Lives for the duration of one plan execution. Checkpointed to Postgres.

### 10.3 Long-Term Memory (mem0)

Semantic, unstructured memory for user preferences, entity relationships, and learned patterns.

**Implementation:** Self-hosted `mem0ai` with Supabase/pgvector backend.

| Component | Choice |
|---|---|
| Vector store | Supabase + pgvector extension |
| Embedding model | `text-embedding-3-small` (OpenAI) |
| Extraction LLM | `gpt-4o-mini` (OpenAI) |
| Collection | `memories` table (auto-created by mem0) |

**Service interface:**

```python
class MemoryService:
    async def retrieve_context(user_id, query, limit=10) -> MemoryContext
        # Called before GID/Planner — injects relevant memories into LLM prompt
    async def store_interaction(user_id, content, metadata=None) -> None
        # Called after graph completion — extracts and stores learned facts
    async def search(user_id, query, limit=10) -> list[MemoryItem]
    async def get_all(user_id) -> list[MemoryItem]
    async def delete(memory_id) -> None
```

**Design principles:**
- Never-crash contract: memory failures log errors but never propagate exceptions
- Lazy initialization: mem0 only imported on first use
- Feature-flagged: `MEM0_ENABLED=False` by default
- All sync mem0 calls wrapped in `asyncio.to_thread()`

Memory is scoped per user. Planner has read access at planning time.

---

## 11. Skill Data Persistence Layer

**Problem:** Skills produce structured data that must persist beyond a single graph execution and be queryable later — commute measurements, digest histories, user preferences, route configs. This is not semantic memory (mem0) and not observability (audit log).

**Solution:** One generic table, one service interface, zero migrations for new use cases.

```sql
CREATE TABLE skill_data_store (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  namespace TEXT NOT NULL,          -- skill/domain identifier: "commute", "digest", "briefing"
  record_type TEXT NOT NULL,        -- data category: "measurement", "profile", "preference", "daily_digest"
  payload JSONB NOT NULL,           -- the actual data (schema varies by namespace + record_type)
  recorded_at TIMESTAMPTZ NOT NULL, -- business timestamp (when the data was observed/extracted)
  created_at TIMESTAMPTZ DEFAULT now(),
  metadata JSONB                    -- optional context: source references, route IDs, etc.
);

CREATE INDEX idx_skill_data_lookup
  ON skill_data_store(user_id, namespace, record_type, recorded_at);

CREATE INDEX idx_skill_data_payload
  ON skill_data_store USING GIN (payload);
```

**Service interface:**

```python
class SkillDataStore:
    async def write(user_id, namespace, record_type, payload, recorded_at, metadata=None) -> str
    async def query(user_id, namespace, record_type, since=None, until=None,
                    payload_filter=None, limit=None, order="desc") -> list[dict]
    async def query_latest(user_id, namespace, record_type) -> dict | None
    async def delete(user_id, namespace, record_type=None, before=None) -> int
    async def count(user_id, namespace, record_type, since=None) -> int
```

**How it's used:** Skills write raw data (measurements, records) and computed results (profiles, aggregates) to the same table, differentiated by `record_type`. Aggregation is done in application code (Python background jobs), not SQL views — no DDL changes per use case.

**Platform principle:** No feature-specific tables. Every use case writes to and reads from this one table via the `SkillDataStore` service. Namespace isolation keeps skills from colliding.

---

## 12. Scheduled Job Executor

**Problem:** The orchestration engine is one-shot (user message → graph → result). Some capabilities require recurring execution without a user trigger — periodic data collection, daily briefings, scheduled notifications.

**Implementation:**

```sql
CREATE TABLE scheduled_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  job_type TEXT NOT NULL,           -- "skill_invocation" | "graph_execution"
  cron_expression TEXT NOT NULL,    -- standard cron syntax
  config JSONB NOT NULL,            -- job-specific config (skill_id, input params, plan template, etc.)
  next_run TIMESTAMPTZ NOT NULL,
  last_run TIMESTAMPTZ,
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

**Two execution modes:**
- **Lightweight skill invocation:** Single skill call + data store write. Used for periodic data collection (e.g., Maps API query every 15 min). No full graph execution overhead.
- **Full graph execution:** Triggers a complete GID → Planner → Executor pipeline. Used for complex scheduled tasks (e.g., morning briefing that fans out to 5 APIs).

**Runtime:** APScheduler or Celery Beat as the background worker. Polls `scheduled_jobs` table, fires jobs on schedule.

**Any skill can register a scheduled job** through the service interface — this is a platform module, not tied to any specific use case.

---

## 13. Audit & Observability

### 13.1 Audit Log

Every node execution writes an audit entry:

```json
{
  "task_id": "string",
  "skill_id": "string",
  "input": {},
  "output": {},
  "status": "success | failed | skipped",
  "duration_ms": "integer",
  "timestamp": "ISO",
  "idempotency_hash": "string"
}
```

Persisted to Supabase `task_audit_log` table. Surfaced in mobile UI as TaskTimeline.

### 13.2 Event Taxonomy

~24 typed event types covering the full lifecycle: plan created, node started, node completed, approval requested, approval received, node failed, retry initiated, graph completed, etc.

- Each event has a version and structured payload
- PII redaction utilities built-in
- SSE streaming to mobile frontend for real-time progress updates

### 13.3 Event Persistence

All events persisted to Supabase (not in-memory) — survives process restarts. SSE pubsub layer on top for real-time streaming.

---

## 14. Bidirectional Execution

Same orchestration engine handles both outbound and inbound flows. Only the entry point differs.

**Outbound (chat-triggered or scheduled):**
```
User message / Scheduled trigger → GID → Planner → Executor → Consent gates → External delivery
```

**Inbound (webhook/event-triggered):**
```
Incoming event (WhatsApp reply, call, webhook)
  → Identify user + context
  → Load memory + relevant task history
  → GID
  → (if needs response) → Draft → Consent gate → Deliver
  → (if needs new plan) → Planner → Executor → ...
```

Inbound events hit a FastAPI webhook endpoint per channel. They resolve to the same orchestration pipeline with a different `entrypoint_type` flag in `TaskContext`.

---

## 15. Structured Artifact System (Mobile)

Rich results that go beyond plain chat text.

**Protocol:** Backend returns `{ artifact_type: string, payload: JSONB }`. Mobile app has a renderer registry mapping `artifact_type` → React Native component.

**Adding a new card type** = adding one React Native component + registering it in the renderer. No backend schema change.

**Artifact types include:** flight cards, hotel cards, itinerary day timelines, comparison tables, briefing cards, digest cards, consent/approval cards, map views.

**Push notifications:** FCM (Android) + APNs (iOS) via a Direct adapter skill (`notification.push.send`). Used for proactive delivery of scheduled results (morning briefing, commute alerts). Push token registered at app startup, stored in `skill_data_store` with `namespace: "user_profile"`.

---

## 16. Non-Negotiables

1. **Planner never invents skill names** — registry lookup only, validated before graph build
2. **No side-effect executes without consent gate** — `requires_consent = true` in registry is enforced by executor, not optional
3. **LangGraph checkpoints go to Supabase** — in-memory state is not acceptable for production
4. **Consent uses LangGraph `interrupt()`** — no custom polling loops, no in-memory event stores
5. **Audit log on every node** — non-optional, written before result is returned to caller
6. **Registry health checks run on schedule** — planner never sees `disabled` skills
7. **GID fast path sub-500ms** — planner path can be 2–5s, but trivial tasks must feel instant
8. **No feature-specific tables** — all skill-produced persistent data goes through `skill_data_store`
9. **MCP adapter is config-driven** — adding a new MCP skill is a config entry, not new adapter code
10. **Memory never crashes the request** — mem0 failures are logged, never propagated
