# Agent Architecture Findings + Trybo Applications

## What the Ecosystem Is Converging On

Two independent sources — Anthropic's Skills team and OpenClaw (Peter Steinberger) breakdown — are converging on the same architectural philosophy: **one general agent, thin orchestration layer, intelligence encoded in skills, context loaded on-demand.** The era of building specialized agents per domain is over. The era of building skills for a general agent has begun.

***

## The Five Genuinely Novel Ideas

### 1. Progressive Disclosure (Claude Skills)

The mechanism: skill metadata costs ~100 tokens at startup. The full instruction body (up to 5,000 tokens) loads only when the model selects that skill. Sub-files and scripts load only when referenced inside the body. Idle context cost for 1,000 skills = ~100,000 tokens. Idle cost with traditional all-or-nothing loading = potentially millions.

**The Notion comparison makes the stakes concrete:** Notion's MCP server dumps ~20,000 tokens at startup. A Notion skill dumps ~100 tokens. 200x difference. This is a scalability cliff, not a minor optimization.

**Trybo application — Priority 1:** The planner currently receives every skill's full `description + input_schema + output_schema` regardless of relevance. Two-phase loading fixes this:
- Phase 1: planner gets `skill_id + name + one-line description` (~50 tokens/skill). Selects relevant skills.
- Phase 2: full schemas load only for selected skills. Validation runs against complete definitions.
- Result: 60-80% planner prompt reduction at 50+ skill registries. Better plan quality from less noise.

***

### 2. Skills as Procedural Memory, Not Tool Definitions (Claude Skills)

Current skill registries describe **what** a skill does (input/output schemas). Claude Skills adds a layer that describes **how** to use it well — when to invoke it, what patterns work, what to avoid, how to combine it with other skills. The "body" of a skill.md is a prompt engineering document, not an API spec.

Separately: the agent can **write new skills during work**, saving them for its future self. Day 1 = generic. Day 30 = knows your workflows, preferences, and patterns as concrete reusable instructions — not fuzzy semantic memories. This is a compounding knowledge base with a standardized format guaranteeing that anything the agent writes, a future version of itself can use efficiently.

**Trybo application — Priority 4 (v2):** Add `usage_notes` / `best_practices` field to `SkillDefinition` that loads when a skill is selected, not at startup. Encode things like: *"Google Maps: never set future departure_time — always use now"* or *"Tavily: use search_depth: advanced for product pages."* Longer term: after a complex plan executes successfully, save the `PlanSpec` structure as a reusable template in `skill_data_store` with `namespace: "plan_templates"`. Planner retrieves and adapts proven templates instead of planning from scratch.

***

### 3. Heartbeat — Context-Aware Proactive Wake (OpenClaw)

Not a cron job in the useful sense. A general-purpose trigger that fires every ~30 minutes and asks the agent: *"you have full context about this user — is there anything worth doing right now?"* The agent decides. Most of the time: nothing. When context contains something significant (upcoming surgery, flight in 4 hours, approaching deadline): it acts.

The shoulder surgery story is the proof of concept. Nobody programmed "check in after surgery." The model inferred significance from context. No task-specific code. This is categorically different from Trybo's scheduled jobs, which are pre-defined tasks someone explicitly configured. Heartbeat is open-ended inference over context.

**Trybo application — Priority 2:** New `job_type: "heartbeat"` in the existing scheduler. Runs every N minutes per active user:
1. Load recent context (last messages, calendar events, pending tasks, recent `skill_data_store` writes)
2. Load mem0 user profile
3. Lightweight LLM call (GPT-4o-mini, ~$0.001): *"Given this context, is there anything time-sensitive worth surfacing?"*
4. If yes → trigger mini-plan or notification. If no → discard.

No new infrastructure. The "magic" moments this creates are disproportionate to the implementation cost.

***

### 4. Output Filtering / Composability (OpenClaw)

Peter's CLI composability argument is really about **output filtering**. An MCP weather tool returns a 50-field blob — the model must consume all 50 fields. A CLI can pipe through `jq .temperature` and consume only what's needed. Zero context pollution.

Trybo has this exact problem. Every `SkillResult` goes into `TaskContext.task_results` in full. If a Maps API response is 2KB and the next node only needs `duration_seconds`, 1.5KB of irrelevant JSON is carried through every downstream node's context.

**Trybo application — Priority 3:** Add `output_fields: []` array per task in `PlanSpec`. Executor strips everything else before writing to `TaskContext.task_results`. Medium effort (schema change + executor logic). Keeps context lean as plan complexity grows.

***

### 5. Gateway as Dumb Router + Agent Isolation (OpenClaw)

The architectural insight: agents never communicate directly. Everything — human messages, heartbeats, cron jobs, webhooks, and inter-agent messages — routes through one dumb Gateway into a FIFO queue. An agent sending a message to another agent looks identical to a human sending a message. Same pipeline, zero special-casing.

Each agent is fully isolated: it only knows its own context window + its own markdown memory files. No shared state. Coordination emerges from message passing, not from a central orchestrator or supervisor agent.

**Takeaway for Trybo:** If/when Trybo moves to multi-agent, resist the temptation to build direct agent-to-agent channels or a hierarchical orchestrator. Route everything through the existing webhook/scheduler infrastructure. Agent B receives Agent A's output as just another inbound event.

***

## What Doesn't Apply to Trybo (Being Direct)

- **CLIs over MCP** — OpenClaw runs in a shell. Trybo is a FastAPI server with async Python skill adapters. The composability insight (output filtering) applies. The CLI-as-interface pattern does not.
- **Memory as markdown** — Works for single-user desktop agents. Doesn't work for multi-tenant mobile product requiring per-user isolation. mem0 + pgvector is the right call.
- **Skills as filesystem folders** — Optimized for Claude Code's filesystem context. Trybo's registry + adapter pattern is better for a server-side engine serving N concurrent users.
- **Agent modifies its own tools** — Powerful for developer tools. Risky for a consumer product where side-effects have real-world consequences (phone calls, messages). Schema-validated contract-first approach is the right tradeoff.

***

## Priority Table

| Priority | What | Source | Effort | Impact |
|---|---|---|---|---|
| **1** | Progressive skill disclosure in planner | Claude Skills | Low — planner prompt restructure | High — scales registry to 100+ skills |
| **2** | Heartbeat proactive wake | OpenClaw | Low — new job_type in existing scheduler | High — creates "magical" moments |
| **3** | Output field filtering in PlanSpec | OpenClaw composability | Medium — schema change + executor logic | Medium — keeps context lean at scale |
| **4** | Skill usage notes / best practices field | Claude Skills body concept | Low — schema addition | Medium — better plan quality |
| **5** | Plan template procedural memory | Claude Skills self-improvement | High — new store + retrieval logic | High — but v2 territory |

***

## The Single Most Important Insight

The pattern across all three sources is the same principle stated different ways:

> **Never load context you don't need yet. Make the orchestration layer dumb. Put all intelligence in the skills.**

Claude Skills calls it progressive disclosure. OpenClaw calls it the Gateway routing everything through a queue. Claude calls it three-level context injection. They're all saying the same thing: the orchestration layer's only job is routing. The agent's only job is deciding what to load. The skill's job is knowing how to work. Keep those three concerns completely separated and the system scales.
