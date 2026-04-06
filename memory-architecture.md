# Memory Architecture for an Agentic Personal Assistant -> Only Conversation Buffer complete

## The Problem

Trybo has two execution paths that live in different worlds:

```
classify_intent(prompt, chat_history)
  ├─ "chat"  → _handle_chat_response()  → Loads last 20 messages → LLM has full context
  └─ "agent" → _run_agentic_mvp()       → Passes only req.prompt → LLM is amnesiac
```

The agentic planner receives:
- `query`: the current user message (just one string)
- `clarifications`: answers to questions asked in THIS run only
- `skill catalog`: what tools exist

It does NOT receive:
- Previous messages in the chat ("I have an HDFC card" said 2 messages ago)
- Results of previous agent executions ("the Goa trip you planned yesterday")
- Learned user facts ("user prefers window seats")
- Structured user data ("briefing preferences saved last week")

This means:
- "Find hotels near the beach" after "Search flights to Goa" → planner doesn't know about Goa
- "Change the hotel to a 5-star" → planner can't find the previous trip plan
- "Use my HDFC card for the comparison" after saying "I have HDFC and ICICI cards" → forgotten

The conversational path works because it dumps 20 messages into the prompt. But that approach doesn't scale for an agentic system that needs structured access to prior work, not just chat history.

---

## What's Currently Stored (and Unused by the Agentic Path)

| Data | Where It Lives | What's In It | Used by Agentic Path? |
|---|---|---|---|
| Chat messages | `chat_messages` table | Every user + AI message, ordered by `seq` | **No** — only used by chat intent path |
| Batch records | `batches` table | `raw_prompt`, `summary_text`, status, `chat_id` link | **No** |
| Task results | `bot_tasks` table | Per-task prompt, status, error info | **No** |
| Skill outputs | `skill_data_store` table | Structured data written by skills (preferences, saved plans) | **No** — skills read it at execution time, but planner doesn't |
| LangGraph state | Postgres checkpointer | Full graph state per thread, node outputs | **Partially** — used for interrupt/resume within one run, not across runs |
| Execution context | In-memory `ExecutionContext` | user_id, chat_id, location, API keys | **Yes** — but chat_id is never used to load history |

The data exists. The wiring doesn't.

---

## Why "Just Pass Chat History" Is Insufficient

The obvious fix: load last 20 messages → inject into planner prompt. This works for conversational continuity but fails at 4 things:

1. **Token waste.** 20 messages of "Good morning" / "Hello! How can I help?" burn tokens. The planner needs the *relevant* context, not the full transcript.

2. **Cross-session blindness.** User planned a Goa trip yesterday in a different chat. Today asks "change my hotel." Chat history won't contain yesterday's execution.

3. **Structured data is invisible.** The user saved briefing preferences, financial instruments, commute routes. These are in `skill_data_store`, not in chat messages. Dumping messages into the prompt won't surface them.

4. **Execution artifacts are lost.** The planner doesn't just need "the user asked about Goa" — it needs the actual flight options, hotel shortlist, and itinerary that were produced. Those are structured outputs from previous graph executions, not chat text.

Chat history is Layer 1 of 5. On its own, it produces a mediocre personal assistant. All 5 layers together produce Jarvis.

---

## The 5-Layer Memory Model

Each layer serves a different time horizon, query pattern, and decision point. No single layer subsumes the others.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Layer 5: Procedural Memory                    │
│    Explicit configurations + structured data (skill_data_store)  │
│    Time horizon: months–years | Query: key-value lookup          │
├──────────────────────────────────────────────────────────────────┤
│                    Layer 4: Semantic Memory                       │
│    Learned user facts extracted from interactions (mem0)          │
│    Time horizon: months–years | Query: semantic similarity       │
├──────────────────────────────────────────────────────────────────┤
│                    Layer 3: Episodic Memory                       │
│    Summaries + key outputs from past graph executions             │
│    Time horizon: days–months | Query: semantic + structured      │
├──────────────────────────────────────────────────────────────────┤
│                    Layer 2: Working Memory                        │
│    Inter-node data flow within a single graph execution           │
│    Time horizon: seconds–minutes | Query: $memory.node.field     │
├──────────────────────────────────────────────────────────────────┤
│                    Layer 1: Conversation Buffer                   │
│    Recent messages in the current chat thread                     │
│    Time horizon: minutes–hours | Query: last N, sequential       │
└──────────────────────────────────────────────────────────────────┘
```

### Layer 1: Conversation Buffer

**What:** Recent messages in the current chat session.
**Source:** `chat_messages` table, filtered by `chat_id`, ordered by `seq`.
**Time horizon:** Minutes to hours (current session).
**Query pattern:** Sequential — last N messages, chronological.
**Fed to:** GID, Planner, Clarification.

**Purpose:** Multi-turn continuity within a single conversation. "Find hotels near the beach" after "Search flights to Goa" → planner sees Goa context.

**Implementation:**
```
Before planning: load last N messages from chat_messages WHERE chat_id = current
Inject as conversation_history into planner prompt
N should be configurable (default 20, but summarize if token budget tight)
```

**Design decisions:**
- Raw messages vs. summarized? Start with raw (last 10-15). If token budget is a problem, summarize older messages and keep recent ones verbatim. Hybrid: last 5 verbatim + summary of previous 15.
- Include AI responses? Yes — the AI's previous outputs are context too ("I found 3 flights to Goa" tells the planner what was already done).
- Include system messages / milestone markers? Yes, selectively — "Batch completed: Goa trip plan saved" is useful context.

### Layer 2: Working Memory

**What:** Data flowing between nodes in a single graph execution.
**Source:** LangGraph state / `TaskContext.task_results`.
**Time horizon:** Seconds to minutes (single execution).
**Query pattern:** Key-value — `$memory.<node_id>.<field>`.
**Fed to:** Executor nodes (via late-binding resolution).

**Purpose:** Flight search returns data → hotel search needs destination → itinerary needs both.

**Status:** Already implemented via `$memory` references. Works well. No changes needed.

### Layer 3: Episodic Memory

**What:** Structured summaries of past graph executions — what was asked, what was produced, what decisions were made.
**Source:** New — generated after each graph execution completes.
**Time horizon:** Days to months.
**Query pattern:** Semantic search (pgvector) + structured filters (user_id, goal type, recency).
**Fed to:** Planner (for follow-up tasks, partial re-planning).

**Purpose:** "Change the hotel in my Goa trip" → find yesterday's trip plan execution → load its key outputs → partial re-plan. This is the hardest and most valuable layer — it gives the assistant continuity across sessions.

**What an episode contains:**
```json
{
  "episode_id": "uuid",
  "user_id": "uuid",
  "chat_id": "uuid",
  "goal": "Plan a 3-day Goa trip from Bangalore, budget 30K",
  "goal_embedding": [0.12, -0.34, ...],
  "completed_at": "2026-03-13T14:32:00Z",
  "status": "completed",
  "key_outputs": {
    "flights": {"airline": "IndiGo", "price": 4200, "departure": "11:30 AM"},
    "hotel": {"name": "Calangute Beach Resort", "price_per_night": 6000},
    "itinerary": {"days": 3, "artifact_id": "uuid"}
  },
  "decisions_made": ["chose IndiGo over Air India (₹800 cheaper)", "3-star hotel to fit budget"],
  "artifacts_produced": ["itinerary_card:uuid", "comparison_table:uuid"],
  "skills_used": ["booking.flights.search", "booking.hotels.search", "trip.itinerary.synthesize"],
  "summary": "Planned 3-day Goa trip. IndiGo flights ₹4,200. Calangute Beach Resort ₹18,000. Total ₹34,400."
}
```

**How it's queried:**
1. Before planning, retrieve recent episodes for the user
2. Semantic search: embed the current query → find episodes with similar goals
3. Structured filter: same chat_id (conversation continuity) or recent (last 7 days)
4. Inject top-K relevant episodes into planner prompt as context

**Storage options:**
- Option A: `skill_data_store` with `namespace="system", record_type="episode"` — reuses existing infra
- Option B: Dedicated `execution_episodes` table with a `goal_embedding` vector column — better for semantic search
- **Recommendation: Option B.** Episodes are a system concern, not a skill concern. They deserve their own table with a vector index. `skill_data_store` is for skill-produced business data.

```sql
CREATE TABLE execution_episodes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES user_profiles(id),
  chat_id UUID REFERENCES chats(id),
  batch_id UUID REFERENCES batches(id),
  goal TEXT NOT NULL,
  goal_embedding VECTOR(1536),
  status TEXT NOT NULL,
  key_outputs JSONB NOT NULL,
  decisions_made JSONB,
  artifacts_produced JSONB,
  skills_used TEXT[],
  summary TEXT NOT NULL,
  completed_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_episodes_user_recent
  ON execution_episodes(user_id, completed_at DESC);

CREATE INDEX idx_episodes_embedding
  ON execution_episodes USING ivfflat (goal_embedding vector_cosine_ops);
```

**Lifecycle:**
- **Write:** After `finalize` node in the outer graph → LLM generates episode summary from execution results → embed goal → insert
- **Read:** Before `build_plan` node → query recent/relevant episodes → inject into planner context
- **Expire:** Episodes older than 90 days get summarized into semantic memory (Layer 4) and archived

### Layer 4: Semantic Memory

**What:** Persistent facts learned about the user across all interactions. Not tied to any specific execution.
**Source:** mem0 (with pgvector backend).
**Time horizon:** Months to years.
**Query pattern:** Semantic similarity search.
**Fed to:** GID (context for routing), Planner (context for planning), Synthesis nodes (personalization).

**Purpose:** "User has HDFC and ICICI credit cards." "User's office is in Sector 62, Noida." "User prefers non-stop flights." These are durable facts that inform every future interaction.

**How it works:**
1. **Extraction:** After each execution (or significant conversation), an LLM extracts durable facts from the interaction. mem0 handles deduplication and conflict resolution.
2. **Retrieval:** Before GID/Planning, query mem0 with the current user message. Retrieve top-K relevant memories. Inject as "Known about this user:" section in the prompt.
3. **Decay:** mem0 handles relevance scoring. Stale memories naturally rank lower.

**What gets extracted (examples):**
- User preferences: "prefers window seats", "vegetarian", "morning person"
- User facts: "works at [company]", "lives in Noida Sector 62", "has 2 kids"
- Financial data: "has HDFC Regalia credit card", "uses PhonePe for UPI"
- Behavioral patterns: "usually plans trips 2-3 weeks in advance"

**What does NOT go here:**
- Specific execution outputs (those are episodic memory, Layer 3)
- Explicit configurations (those are procedural memory, Layer 5)
- Temporary states ("currently planning a Goa trip" — that's conversation buffer)

**Implementation:** Already designed in `5.1-Workstream.1.md`. mem0 with Supabase/pgvector backend.

### Layer 5: Procedural Memory

**What:** Explicit user configurations and structured data written by skills.
**Source:** `skill_data_store` table.
**Time horizon:** Until explicitly changed.
**Query pattern:** Key-value lookup by `namespace` + `record_type`.
**Fed to:** Skill nodes (direct data access at execution time), Planner (summary of what's configured).

**Purpose:** Briefing preferences ("daily at 7 AM, include weather + calendar"), financial instruments ("HDFC Regalia, ICICI Amazon Pay"), saved locations ("home: Noida Sector 62", "office: Hauz Khas").

**Why it's separate from semantic memory:** These are structured, schema-driven records that skills read programmatically. You don't semantically search for "what's the user's briefing time?" — you query `namespace=briefing, record_type=preference`.

**Status:** Already implemented. `SkillDataStore` service exists and skills use it.

**Missing piece:** The planner doesn't know what's in the skill_data_store. Two options:
1. Planner queries a summary: "User has: briefing preferences (daily 7 AM), financial instruments (HDFC, ICICI), 2 saved locations"
2. Skills self-declare what stored data they need and load it themselves at execution time

Recommendation: Both. Give the planner a lightweight summary so it knows what data exists. Let skills load the full records themselves.

---

## Context Assembly at Each Decision Point

Every LLM call in the pipeline should receive the right layers — not all of them, not none of them.

### GID (Intent Detection)

```
System: GID routing rules + available intents
Layer 1: Last 5 messages (lightweight — GID needs context, not full history)
Layer 4: Top 3 semantic memories (relevant to current message)
User: Current message
```

Why not more? GID is a classifier, not a planner. It needs enough context to route correctly ("chat" vs "agent") but doesn't need execution history or procedural memory.

### Planner (Task Decomposition)

```
System: Planner instructions + skill catalog
Layer 1: Last 10-15 messages (conversation continuity)
Layer 3: Top 3 relevant episodes (what was done before)
Layer 4: Top 5 semantic memories (user facts)
Layer 5: Summary of stored configurations (what data exists in skill_data_store)
User: Current message + clarifications
```

This is the heaviest context assembly. The planner makes the most consequential decisions — it needs the richest context.

**Token budget:** ~2,000 tokens for conversation buffer + ~500 for episodes + ~300 for memories + ~200 for config summary + skill catalog + system prompt. Well within any modern model's context window.

### Executor Nodes (Skill Invocation)

```
Layer 2: $memory references from prior nodes (already implemented)
Layer 5: Direct skill_data_store reads (already implemented)
ExecutionContext: user_id, API keys, location (already implemented)
```

Executor nodes are stateless tool calls. They don't need conversation history or semantic memory — they receive resolved inputs from the plan.

Exception: **Synthesis nodes** (briefing.synthesize, digest.synthesize, trip.itinerary.synthesize) benefit from Layer 4 (user preferences from mem0) for personalization. These nodes should receive a `user_context` input populated from mem0.

### Finalize Node (Post-Execution)

```
Layer 1: Last 3 messages (to format response naturally)
Layer 2: All node outputs from this execution
→ Writes:
  - AI response → chat_messages (Layer 1 for next turn)
  - Episode summary → execution_episodes (Layer 3)
  - Extracted facts → mem0 (Layer 4)
  - Structured outputs → skill_data_store (Layer 5)
```

The finalize node is the write-back point. Every execution enriches the memory system for next time.

---

## The Three-Read, Three-Write Pattern

**Before every plan (three reads):**
1. Read `chat_messages` → conversation buffer (Layer 1)
2. Read `execution_episodes` → relevant past work (Layer 3) + read mem0 → user facts (Layer 4)
3. Read `skill_data_store` → configuration summary (Layer 5)

**After every execution (three writes):**
1. Write response → `chat_messages` (Layer 1 for next turn)
2. Write episode summary → `execution_episodes` (Layer 3)
3. Write extracted facts → mem0 (Layer 4) + write outputs → `skill_data_store` (Layer 5, if applicable)

This pattern ensures the system gets smarter with every interaction. The first execution is context-free. The tenth execution in the same chat is deeply contextual. The hundredth execution across sessions leverages months of learned facts and episodic history.

---

## How This Compares to Industry

| Framework | Thread Memory | Working Memory | Episodic Memory | Semantic Memory | Procedural Memory |
|---|---|---|---|---|---|
| OpenAI Assistants | Thread messages | Run steps | None | None built-in | None built-in |
| LangGraph (vanilla) | Checkpointer state | State channels | Store (basic KV) | None built-in | None built-in |
| Google ADK | Session state | Agent state | Artifact store | None | None |
| CrewAI | Agent context | Shared crew memory | Long-term (basic) | RAG memory | None |
| Mem0 (standalone) | None | None | None | Core product | None |
| **Trybo (proposed)** | **chat_messages** | **LangGraph state** | **execution_episodes** | **mem0** | **skill_data_store** |

Trybo is the only system with all 5 layers purpose-built. Most frameworks handle 1-2 layers and leave the rest to the developer. This is a genuine architectural differentiator — the memory system IS the moat for a personal assistant.

---

## Implementation Sequence

### Phase 0: Wire the Obvious (~2 hours)

The lowest-hanging fruit: the planner doesn't receive conversation history even though the data is already in the DB and the loading function already exists.

**Changes:**
1. Add `chat_history: list[dict]` to `CompilerState` TypedDict
2. In `ingress_normalize` node: load `chat_messages` via existing `load_recent_chat_history(db, chat_id, limit=15)`
3. In `llm_planner.py`: accept `chat_history` parameter, inject into planner system prompt as "Conversation so far:" section
4. In `clarify_intent`: include history so clarification questions account for what was already said

**Result:** The planner can now see "Search flights to Goa" when the user follows up with "Now find hotels near the beach." Immediate, massive improvement in multi-turn agentic conversations.

### Phase 1: Episodic Memory (~1 day)

**Changes:**
1. Create `execution_episodes` table with pgvector index
2. In `finalize` node: generate episode summary via lightweight LLM call → embed goal → insert
3. In `build_plan` node (pre-step): query recent episodes by semantic similarity + recency → inject top 3 into planner prompt
4. Planner prompt section: "Recent relevant work:" with episode summaries

**Result:** "Change the hotel in my Goa trip" now works cross-session. The planner sees yesterday's trip plan episode and can do partial re-planning.

### Phase 2: Semantic Memory — mem0 (1-2 days)

**Changes:**
1. Deploy mem0 with pgvector backend (already designed in 5.1-Workstream.1.md)
2. In `finalize` node: extract durable facts from execution results → store to mem0
3. In `ingress_normalize`: retrieve relevant memories → inject into planner prompt as "Known about this user:"
4. In GID: include top 3 memories for better routing

**Result:** "I have HDFC and ICICI cards" persists across all future sessions. Planner automatically considers user preferences when building plans.

### Phase 3: Procedural Memory Surfacing (~half day)

**Changes:**
1. In `ingress_normalize`: query `skill_data_store` for user's configured items (briefing prefs, saved locations, financial instruments) → produce a lightweight summary
2. Inject summary into planner prompt: "User has configured: daily briefing at 7 AM (weather + calendar), saved home location (Noida Sector 62), financial instruments (HDFC Regalia, ICICI Amazon Pay)"

**Result:** Planner knows what data is already available without the user restating it. "What's the best deal on a Samsung phone?" automatically uses saved card info.

### Phase 4: Intelligent Context Management (ongoing)

**Changes:**
1. Summarization: old messages get summarized instead of raw-injected (saves tokens)
2. Relevance filtering: not all episodes are relevant — rank by semantic similarity to current query
3. Episode expiry: old episodes get distilled into semantic memory facts, then archived
4. Feedback loop: track which memories the planner actually uses → improve retrieval

---

## Architecture Diagram

```
User Message
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                  CONTEXT ASSEMBLY                        │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │   Layer 1    │  │   Layer 3    │  │   Layer 4     │  │
│  │ chat_messages│  │  episodes    │  │   mem0        │  │
│  │ (last 15)   │  │ (top 3 sim)  │  │ (top 5 rel)   │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬────────┘  │
│         │                │                  │           │
│         └────────────────┼──────────────────┘           │
│                          │                              │
│                 ┌────────▼────────┐                     │
│                 │  Layer 5 summary │                     │
│                 │ (skill_data_store│                     │
│                 │  configurations) │                     │
│                 └────────┬────────┘                     │
│                          │                              │
│              ┌───────────▼──────────┐                   │
│              │   Assembled Context   │                   │
│              │   (< 4K tokens)       │                   │
│              └───────────┬──────────┘                   │
└──────────────────────────┼──────────────────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Planner    │
                    │  (LLM call)  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Executor   │──── Layer 2: $memory refs
                    │  (LangGraph) │     between nodes
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Finalize   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Layer 1  │ │ Layer 3  │ │ Layer 4  │
        │ Write AI │ │ Write    │ │ Write    │
        │ message  │ │ episode  │ │ facts to │
        │ to chat  │ │ summary  │ │ mem0     │
        └──────────┘ └──────────┘ └──────────┘
```

---

## Key Design Principles

1. **Curated, not dumped.** Never dump all available context into the prompt. Each decision point gets only the layers it needs, retrieved by relevance, within a token budget.

2. **Every execution enriches the system.** The three-write pattern ensures each interaction makes the next one better. This is the compounding moat — a 6-month user has an assistant that knows them deeply.

3. **Structured where possible, semantic where necessary.** Briefing preferences are structured (Layer 5). "User prefers window seats" is semantic (Layer 4). Don't force structured data into semantic search or vice versa.

4. **Episodic memory is the missing piece.** Most agentic frameworks skip this entirely. But for a personal assistant that executes multi-step workflows, the ability to reference and build on previous work is transformative. "Modify my Goa trip" is impossible without it.

5. **The memory system is the product.** A personal assistant without memory is a chatbot. A personal assistant with 5 layers of contextual memory is Jarvis. The difference between "useful tool" and "indispensable companion" is memory depth.

---


# Above things implemented! Pending items:

```                                                                     
⏺ Here's my assessment:                
                                                                                             
  Prompt changes (done):                                                                  
  - Removed ALL hardcoded location examples (Noida, CyberHub, Patna, Connaught Place)        
  - Generalized commute/directions rule → "LOCATION-BASED SKILLS" — applies to any skill that
   takes location inputs                                                                     
  - Generalized clarification rule 7 → "LOCATION-BASED SKILLS" — covers directions, commute, 
  weather, anything needing a place                                   
  - Added explicit "CONVERSATION CONTEXT" rule in both prompts telling the LLM to extract
  details from chat history
  - Removed the specific +91... phone examples from channel upstream data examples
  - Kept DEFAULT_TIMEZONE as IST — this should eventually be per-user, but that's a separate
  system change

  Conversation buffer gaps vs your spec:

  ┌──────────────────────────────────┬───────────────────────────────────┬──────────────┐
  │            Your Spec             │           Current State           │     Gap      │
  ├──────────────────────────────────┼───────────────────────────────────┼──────────────┤
  │ Configurable N (default 20)      │ Hardcoded 15 in agent_runs, 20 in │ Minor — easy │
  │                                  │  chat_service                     │  fix         │
  ├──────────────────────────────────┼───────────────────────────────────┼──────────────┤
  │ Summarize if token budget tight  │ No summarization                  │ Real gap     │
  ├──────────────────────────────────┼───────────────────────────────────┼──────────────┤
  │ Include system/milestone         │ Only user and ai sender_types,    │ Missing      │
  │ messages                         │ skips system                      │              │
  ├──────────────────────────────────┼───────────────────────────────────┼──────────────┤
  │ Hybrid: last 5 verbatim +        │ Raw dump only                     │ Missing      │
  │ summary of older                 │                                   │              │
  └──────────────────────────────────┴───────────────────────────────────┴──────────────┘

  The summarization and hybrid approach are meaningful engineering work — not a quick fix.
  Should I address those now, or should we first verify the cleaned-up prompts work correctly
   with the current buffer implementation, then tackle summarization as a separate iteration?

✻ Cooked for 2m 0s

❯ 2                                                                                          
                                                                                        
⏺ Let me verify the server picked up the prompt changes and test that the commute query still
   works with the cleaned-up generic prompts:                                                
                                                                                             
⏺ Read 1 file (ctrl+o to expand)                                                             
                                                                                             
⏺ Server reloaded and is up. Try the commute query again in a new chat — the prompts are now 
  generic with no hardcoded examples. The LLM should still extract location from conversation
   history and build the plan correctly.                                                     
                                                                      
```