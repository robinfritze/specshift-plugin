---
id: agents
template-version: 1
description: AGENTS.md bootstrap template (full body) — single source of truth for agent directives, read by Codex natively and by Claude Code via the @AGENTS.md import in CLAUDE.md
generates: AGENTS.md
requires: []
instruction: |
  Generate AGENTS.md with project-specific agent instructions.
  Always include the Workflow, Planning, and Knowledge Management sections.
  Add project-specific rules discovered during codebase analysis (e.g., a File Ownership section).
  Use REVIEW markers for items needing user confirmation.
---
# Project Rules

## Workflow

All changes to this project MUST go through the spec-driven workflow. Before editing ANY file (source code, specs, skills, templates, docs, or configuration), invoke the specshift skill with the appropriate action (`specshift propose` to plan changes, `specshift apply` to implement them, `specshift finalize` to wrap up, `specshift init` to bootstrap or update). Never edit files directly.

## Planning

When using plan mode to discuss a change before invoking `specshift propose`, conclude the planning discussion with an explicit scope summary before exiting plan mode. The summary must cover:

- **In scope** — what this change will do
- **Out of scope / Non-goals** — what this change will explicitly not do, and why

Present the summary to the user for review. Do not exit plan mode until the user has confirmed the scope. This confirmed scope feeds directly into the proposal's Scope & Boundaries section and the design's Non-Goals — treat it as a commitment, not a suggestion.

For trivial changes, a one-line scope statement is sufficient. The goal is explicitness, not ceremony.

When describing implementation steps, the plan MUST route implementation through the specshift workflow skill (starting with `specshift propose`). Plans that describe direct file edits without invoking specshift are non-conforming.

## Knowledge Management

Do not use auto-memory for project knowledge (architecture decisions, conventions, design rationale, workflow patterns). Instead:
- **Rules/conventions** → propose a CONSTITUTION.md update via specshift propose
- **Decisions with rationale** → these emerge naturally as design.md artifacts and ADRs during the change flow
- **Requirements** → propose spec updates via specshift propose
- **Friction/bugs** → file a GitHub Issue

Auto-memory is appropriate only for user preferences and session-specific feedback that do not belong in project artifacts.
