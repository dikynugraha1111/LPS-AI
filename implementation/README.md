# `/implementation/` — Practical Execution Space (From Scope to Execution)

This directory turns scoped modules (`/module/`) into **actionable designs, concrete plans, and final execution handoffs**.

## Purpose
- Translate **module scope** into:
  - architecture and design artifacts,
  - step-by-step implementation plans,
  - and final, executable instructions for the Replit Agent.

## Sub-Directory Pipeline (Strict)
Work must flow in this order:

1. **Read**: `/module/<submodule>/...`
2. **Design**: `/implementation/design/`
3. **Plan**: `/implementation/plan/`
4. **Handoff**: `/implementation/replit-handoff/`

## Sub-Directory Rules

### `design/` — Architecture & UI/UX Design
Use for:
- Architecture decisions and diagrams
- UI/UX flows
- Database schemas / data models
- API contracts and system diagrams

Rules:
- Must remain **grounded in `/module/`** (no drifting scope).
- Capture decisions explicitly so future agents don’t re-derive them.

### `plan/` — Step-by-Step Implementation Plans (Per Module)
Use for:
- Highly focused technical implementation steps
- Risk/edge-case notes
- Sequencing and dependency ordering

Rules:
- Plans **MUST be separated per module** (no monolithic mega-plans).
- Plans should be actionable and deterministic (reduce ambiguity).

### `replit-handoff/` — Final Staging for Replit Agent Execution
Use for:
- Concrete, unambiguous, stepwise handoff prompts formatted for the Replit Agent to execute code generation.

Rules:
- Must be **executable without missing context**.
- Must not contain unresolved “TBD/TODO/figure it out” instructions.
- Must reflect the final intended behavior from `/module/` + `design/` + `plan/`.

## Workflow for AI Agents (Required)
- Start in `/module/` → confirm scope and acceptance criteria
- Draft architecture in `design/` → document key decisions
- Convert to step sequence in `plan/` → per module
- Produce final execution prompt in `replit-handoff/` → deterministic and complete
