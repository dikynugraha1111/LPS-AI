# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is **not a software project** — it is an AI-managed knowledge base and execution workspace for the **Local Port System (LPS) Platform** project. There is no code to build, compile, or test. All artifacts are Markdown documents.

The LPS Platform is a Vessel Traffic System (VTS) for the STS Bunati area, managed by PT. TBK. It handles vessel monitoring, communications, safety, and government integration (Inaportnet, SIMoPEL, Bea Cukai). It is tightly integrated with the STS Platform (which owns master data, billing, and reporting).

---

## Mandatory Onboarding (Do This Before Any Task)

1. Read `README.md` — repo structure and directory rules
2. Read `rules.md` — hard constraints and non-negotiables
3. Read `work-log.md` — prior decisions and current state
4. Read relevant files in `document/`, `module/`, or `implementation/` depending on your task

---

## Directory Architecture

```
document/          ← Single source of truth (BRD, FSD, Swimlane)
module/            ← Derived sub-modules (scoped dev units)
implementation/
  design/          ← Architecture, UI/UX, data models, API contracts
  plan/            ← Per-module implementation plans
  replit-handoff/  ← Final executable prompts for Replit Agent
rules.md           ← Hard constraints (treat as inviolable)
work-log.md        ← Continuity log (append-only; read before work, write after)
```

### Key Rules

**`module/` sub-folders must contain exactly 4 files — no more, no less:**
- `README.md` — boundaries, dependencies, relationships
- `requirements.md` — functional requirements from BRD
- `specifications.md` — technical specs from FSD
- `user-stories.md` — user stories with testable acceptance criteria

**Implementation pipeline is strictly ordered:**
`module/` → `implementation/design/` → `implementation/plan/` → `implementation/replit-handoff/`

Do not skip stages or work backwards.

---

## Foundational Documents

- `document/BRD-LPS-V3-Analysis.md` — current BRD (v3, April 2026, DRAFT)
- `document/Swimlane-Analysis-LPS V3.md` — swimlane analysis
- `document/BRD-LPS-Analysis.docx` — original BRD (binary, for reference only)

**LPS scope (V3):** vessel monitoring, communications, safety, government integration, nomination request portal, monitoring dashboard.

**Out of LPS scope (owned by STS Platform):** master data (vessels, stakeholders, rate cards), billing/PNBP/invoicing, reporting & analytics.

---

## Standard Operating Loop

1. **Orient** — read `rules.md` + `work-log.md` + relevant module/documents
2. **Plan** — determine what must change and where it belongs
3. **Execute** — make the smallest correct set of changes
4. **Log** — append a structured entry to `work-log.md`

### work-log.md Entry Format (required)

```md
## <ISO-8601 timestamp with timezone>
- **Task Performed:** …
- **Files Modified:**
  - …
- **Logic / Decisions Made:** …
- **Results / Next Steps:** …
```

---

## Placement Rules (Where New Content Goes)

| Content Type | Location |
|---|---|
| Business/system requirements, ground truth | `document/` |
| Module scoping (requirements, specs, stories) | `module/<submodule>/` (4-file rule) |
| Architecture, diagrams, data models, API design | `implementation/design/` |
| Step-by-step per-module plans | `implementation/plan/<module>/` |
| Final Replit Agent execution prompts | `implementation/replit-handoff/<module>/` |

**Never** put diagrams or design artifacts inside a `module/` sub-folder. **Never** invent requirements that contradict `document/`.
