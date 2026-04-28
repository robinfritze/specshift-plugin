---
id: claude
template-version: 5
description: CLAUDE.md import stub for Claude Code — delegates to AGENTS.md via @-import (single source of truth pattern)
generates: CLAUDE.md
requires: []
instruction: |
  Generate a minimal CLAUDE.md that imports AGENTS.md.
  Claude Code reads CLAUDE.md natively and expands the @AGENTS.md import at session start, loading AGENTS.md content into context.
  This keeps the bootstrap rules in a single source of truth (AGENTS.md) without duplication.
  Optionally append a `## Claude Code` section below the import for Claude-specific instructions that do not apply to Codex or other targets.
  Do not duplicate normative rules from AGENTS.md.
---
@AGENTS.md
