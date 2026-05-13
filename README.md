# LPS-AI — Base Knowledge & Navigation Guide (for AI Agents)

This repository is **designed to be managed by AI agents**. It is **not** a traditional user-facing README.

## 1) Project Overview & AI Onboarding

### Purpose of this repository
- This repo acts as a **structured knowledge base + execution workspace** for building and evolving the project.
- **Foundational truth lives in `/document/`**. Everything else is derived from it.

### Mandatory onboarding (before doing any task)
Any AI agent (any model, any toolchain) **MUST**:
- Read this file **completely**: `README.md`
- Read constraints and operating rules: `rules.md`
- Read historical context and prior decisions: `work-log.md`

Then, depending on the task, the agent **MUST** read the relevant:
- Source documents in `document/` (BRD/FSD/Swimlane/etc.)
- Module scope in `module/<module-name>/` (requirements/specs/stories)
- Implementation artifacts in `implementation/` (design/plan/handoff)

### Standard operating loop (SOP)
When executing any task, follow this loop:
1. **Orient**: read `rules.md` + `work-log.md` + the relevant module/documents.
2. **Plan**: determine what must change and where it belongs (see directory rules below).
3. **Execute**: make the smallest correct set of changes.
4. **Log**: append a structured entry to `work-log.md` (format below).

> If any instruction conflicts with `rules.md`, treat `rules.md` as the **hard boundary**.

---

## 2) Repository Architecture & Directory Mapping

### Quick navigation
- **Foundational documents (source of truth):** [`document/`](./document/)
- **Module decomposition (scoped development units):** [`module/`](./module/)
- **Execution workspace:** [`implementation/`](./implementation/)
  - [`implementation/design/`](./implementation/design/)
  - [`implementation/plan/`](./implementation/plan/)
  - [`implementation/replit-handoff/`](./implementation/replit-handoff/)

### `/document/` — Foundational Source of Truth
**Description:**
- The **single source of truth** for all foundational project documents required to build the project.

**Examples of what belongs here:**
- BRD (Business Requirements Document)
- FSD (Functional Specifications Document)
- Swimlane analysis
- Any core “ground truth” reference that future agents must trust

**Rules:**
- Keep documents **authoritative and durable**.
- Avoid duplicating “source of truth” content elsewhere.

#### `/document/brd/` — BRD Per Modul (Derived, Read-Optimized)
**Description:**
- Contains **one Markdown file per module**, split from the main BRD (`document/BRD-LPS-V3-Analysis.md`) for easier reading.
- Each file covers: Definisi, Tujuan, Cakupan, Functional Requirements, Role & Access, and cross-references for one specific module.

**Critical rule — Dual-Update Invariant:**
- `document/BRD-LPS-V3-Analysis.md` is **always the primary source of truth**.
- `document/brd/<module>.md` files are **derived views**. They must never be edited in isolation.
- **When a BRD update is needed:** update `BRD-LPS-V3-Analysis.md` first, then immediately update the relevant `document/brd/<module>.md` file in the same work session.
- Never let the two fall out of sync.

**Index:** [`document/brd/README.md`](./document/brd/README.md)

---

### `/module/` — Derived Modules (Development Scoping)
**Description:**
- Contains **highly detailed, broken-down modules** derived from the documents in `/document/`.
- This is where development scope is decomposed into implementable units.

**Hard rule (structure invariant):**
- Every **single sub-module folder** inside `module/` **MUST contain exactly these 4 files**:
  1. `README.md`
  2. `requirements.md`
  3. `specifications.md`
  4. `user-stories.md`

**Meaning of “exactly”:**
- No extra files at the sub-module root.
- If supporting artifacts are needed (diagrams, drafts, exports), place them in:
  - `document/` if they are foundational, or
  - `implementation/design/` if they are design artifacts, or
  - `implementation/plan/` if they are planning artifacts, or
  - `implementation/replit-handoff/` if they are execution handoffs.

**File responsibilities (must be respected):**
- `module/<submodule>/README.md`
  - Module overview, boundaries, dependencies, and relationship to other modules
- `module/<submodule>/requirements.md`
  - **Functional requirements** extracted from the BRD (what the business needs)
- `module/<submodule>/specifications.md`
  - **Technical specifications** extracted from the FSD (how the system should behave)
- `module/<submodule>/user-stories.md`
  - User stories with **acceptance criteria** (testable conditions of completion)

---

### `/implementation/` — Practical Execution Space
**Description:**
- This is where implementation becomes actionable.
- It is strictly divided into **three** sub-folders.

#### `/implementation/design/` — Architecture & UI/UX Design
**Contains:**
- Architecture decisions, diagrams, UX flows, API design notes, data model sketches
- Any design artifacts that translate module specs into an implementable design
- **`lps-design-system.md`** — UI/UX **single source of truth** (foundation tokens, color/typography/spacing, status palette, component library, dua surface preset: Customer Portal & Internal Operator). **WAJIB dibaca sebelum pekerjaan UI apapun.**
- **`m<N>-<name>-ui.md`** — per-modul UI design doc (page inventory, layout, komponen, state transitions, copy reference). Setiap modul yang punya UI wajib punya file ini.

**Aturan UI/UX (wajib):**
- Setiap kerja UI (baru, edit, refactor) **wajib** baca `lps-design-system.md` dulu, lalu invoke skill `ui-ux-pro-max` untuk validasi/generate komponen.
- Surface mapping: M7 (customer side), M8, M9, M9b, M10 → Surface A (Customer Portal, Bahasa Indonesia). M7 (admin side), M12 → Surface B (Internal Operator, English).
- Detail kebijakan ada di `CLAUDE.md` section "UI/UX Standard".

#### `/implementation/plan/` — Implementation Plans & Brainstorming
**Contains:**
- Hyper-focused implementation plans derived from designs
- Planning must be **separated per module** (do not create monolithic plans)

**Rule of thumb:**
- If it’s “what we will do and in what order,” it belongs here.

#### `/implementation/replit-handoff/` — Replit Agent Execution Handoffs
**Contains:**
- Finalized handoff documents explicitly formatted so a Replit Agent can execute
- Should be concrete, stepwise, and free of unresolved ambiguity

---

## 3) System Files & Boundaries

### `rules.md` — Absolute Constraints & Non-Negotiables
**Role:**
- `rules.md` is the project’s **constraint system**.
- It is the hard boundary for:
  - Security protocols
  - Coding standards
  - Naming conventions
  - What an AI **must not** do

**AI rule:**
- If you are about to do something and you are not sure whether it violates constraints, **stop and consult `rules.md` first**.

### `work-log.md` — Cross-Model Continuity (Most Critical)
**Role:**
- `work-log.md` is the **primary continuity mechanism** across different AI models/agents.
- Every agent **MUST** read it before resuming work.
- Every meaningful action **MUST** be recorded so the next agent can continue without re-discovery.

**Required log format (append-only):**
- **Timestamp:** ISO-8601 with timezone
- **Task Performed:** what was attempted/completed
- **Files Modified:** explicit list of files created/edited
- **Logic / Decisions:** key reasoning, tradeoffs, assumptions
- **Results / Next Steps:** outcomes, follow-ups, blockers

**Example entry template:**
```md
## 2026-04-25T21:03:56+07:00
- **Task Performed:** …
- **Files Modified:**
  - …
- **Logic / Decisions Made:** …
- **Results / Next Steps:** …
```

---

## Operating Expectations (AI-Optimized)

### Where to put new information
- **Truth/requirements/specs:** `document/`
- **Module scoping artifacts:** `module/<submodule>/*` (only the 4 required files)
- **Design decisions and diagrams:** `implementation/design/`
- **Step-by-step plans per module:** `implementation/plan/`
- **Final execution handoff for Replit:** `implementation/replit-handoff/`

### What not to do
- Do **not** invent requirements that contradict `/document/`.
- Do **not** change folder meanings or add “misc” dumping grounds.
- Do **not** skip updating `work-log.md` after completing work.

### Common agent actions (checklists)

**Creating a new sub-module under `module/`:**
- Create folder: `module/<submodule-name>/`
- Add **exactly** these files (no extras):
  - `module/<submodule-name>/README.md`
  - `module/<submodule-name>/requirements.md`
  - `module/<submodule-name>/specifications.md`
  - `module/<submodule-name>/user-stories.md`
- Populate them by **deriving** content from `/document/` (do not speculate).
- If you generate diagrams/design artifacts, put them in `implementation/design/` (not in the module folder).

**Planning implementation work:**
- Write per-module plans in: `implementation/plan/<submodule-name>/...`
- Keep plans actionable and bounded; escalate ambiguity back to `/document/` or module specs.

**Handing off to Replit Agent:**
- Produce a final, stepwise handoff in: `implementation/replit-handoff/<submodule-name>/...`
- Ensure it is executable without requiring the Replit Agent to “read your mind”.

