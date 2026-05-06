# `/document/` — Single Source of Truth (SSOT)

This directory contains the **foundational, authoritative project documents** (e.g., BRD, FSD, Swimlane Analysis). All scoping (`/module/`) and execution artifacts (`/implementation/`) **MUST** trace back to what is defined here.

## Purpose
- Store the **original requirements and business/system intent**.
- Provide the **ground truth** used to derive:
  - `module/<submodule>/{requirements,specifications,user-stories}.md`
  - `implementation/{design,plan,replit-handoff}/...`

## Absolute Rules for AI Agents
- **DO NOT** put implementation code, technical plans, or handoff prompts in `/document/`.
- **DO NOT** modify core documents unless explicitly instructed to update business logic/requirements.
- When producing or editing anything in `/module/` or `/implementation/`, **always re-check alignment** against the documents in this directory.
- Prefer **adding clarifying amendments** (when explicitly instructed) over silently rewriting intent.

## How This Connects to the Rest of the Repo
- `/document/` defines **what must be built** (truth).
- `/module/` decomposes `/document/` into **scoped, isolated sub-modules** (work units).
- `/implementation/` turns `/module/` into **designs, plans, and executable handoffs** (execution).

## Foundational Document Index (Maintain This)
Agents should keep this section up to date so future models can navigate instantly.

### Current Documents (index)
- **BRD:** `BRD-LPS-Analysis.docx`
- **BRD (Markdown):** `BRD-LPS-V3-Analysis.md`
- **Swimlane Analysis:** `Swimlane-Analysis-LPS V3.md`

### Index Template (optional, recommended)
- **Document:** `<filename>`
  - **Type:** BRD / FSD / Swimlane / Other
  - **Scope:** `<what it covers>`
  - **Status:** Draft / Final
  - **Notes:** `<links to related module(s)>`
