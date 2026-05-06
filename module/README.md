# `/module/` — Development Scoping Area (Derived From `/document/`)

This directory is the **development scoping layer**. It converts foundational truth from `/document/` into **actionable, isolated sub-modules** that can be designed, planned, and implemented.

## Purpose
- Break down BRD/FSD/Swimlane intent into **clear sub-modules**.
- Provide a stable, model-agnostic structure that downstream agents can rely on.

## Absolute Rules (The 4‑File Strict Rule)
Every sub-folder created inside `/module/` (e.g., `module/auth/`) **MUST contain ONLY** the following **exact 4 files**:

1. `README.md` — module overview, boundaries, dependencies, relationships
2. `requirements.md` — functional requirements **extracted from BRD**
3. `specifications.md` — technical/system specifications **extracted from FSD**
4. `user-stories.md` — user stories with **testable acceptance criteria**

### Enforcement Notes
- **No extra files** in a sub-module folder. No diagrams. No exports. No scratch notes.
- If you need diagrams/design artifacts → put them in `implementation/design/`.
- If you need planning artifacts → put them in `implementation/plan/`.
- If you need foundational truth → update `/document/` **only when explicitly instructed**.

## Workflow: Creating a New Module (Required)
1. **Read `/document/`** and identify a coherent capability boundary.
2. Define the sub-module scope so it is:
   - **Isolated** (clear inputs/outputs)
   - **Non-overlapping** (avoid duplicate ownership with other modules)
   - **Traceable** (each requirement maps back to `/document/`)
3. Create `module/<submodule-name>/` and add **exactly** the 4 required files.
4. Populate:
   - `requirements.md` from BRD (what/why)
   - `specifications.md` from FSD (how/constraints/behavior)
   - `user-stories.md` with acceptance criteria (verifiable)
   - `README.md` with dependencies and relationships
5. Hand off execution to `/implementation/` (design → plan → replit-handoff).

## Definition of Done (for a sub-module)
- The 4 required files exist and contain **non-speculative**, `/document/`-grounded content.
- Acceptance criteria are **testable** and unambiguous.
- Dependencies are explicit (what this module needs / provides).
