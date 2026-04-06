# Trybo Architecture Repo

## What This Repo Is

Gold standard finalized architecture for Trybo. Changes only via reviewed PRs. This is what developers and agents reference for module definitions, interfaces, and design decisions. NOT a scratchpad — planning and iteration happen in trybo-planning.

## Repo Ecosystem

| Repo | Purpose | Location |
|------|---------|----------|
| **tryangle-42/trybo-arch** (this repo) | Finalized architecture — source of truth | `/Users/avinash.nagar/Documents/git/trybo-arch` |
| **tryangle-42/trybo-planning** | Planning scratchpad — meetings, sprints, milestones, gap analysis | `/Users/avinash.nagar/Documents/git/trybo-planning` |
| **tryangle-42/trybot-api** | Backend codebase — FastAPI, LangGraph, Python | `/Users/avinash.nagar/Documents/git/trybot_api` |
| **tryangle-42/Agentic_Orchestration_Components** | Shared architecture components (KPR's contributions) | — |

## Key Files in This Repo

| File | What it is |
|------|-----------|
| `architecture.md` | THE source of truth — full system architecture (Agent-of-Agents, 11 modules) |
| `module-ownership-map.md` | 11 modules: boundaries, interfaces, dependencies, ownership |
| `memory-architecture.md` | 5-layer memory model (buffer, working, episodic, semantic, procedural) |
| `skill-strategy.md` | Use-case-first skill approach, hero use cases |
| `testing-strategy.md` | Testing approach |
| `platform-completion-status.md` | Per-module: what's built, what's not, completion % |
| `research-agent-architecture.md` | External research reference |
| `investor-meetup/` | Investor-facing materials |

## Rules

- This repo is the **gold standard** — only finalized, reviewed content
- Planning docs live in trybo-planning, NOT here
- Changes via PR with team review — no direct pushes for architecture changes
- Backend repo (`trybot-api/docs/arch/`) contains condensed versions pointing back here
