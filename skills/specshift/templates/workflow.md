---
template-version: 9
plugin-version: 0.2.5-beta
templates_dir: .specshift/templates
pipeline: [research, proposal, specs, design, preflight, tests, tasks, audit]

actions: [init, propose, apply, finalize, review]
# Add custom actions here (e.g. qa-review) and define matching
# ## Action: <name> sections in the body below.

# worktree:
#   enabled: false
#   path_pattern: .specshift/worktrees/{change}
#   auto_cleanup: false
#   stale_days: 14

auto_approve: true

review:
  request_review: false
  # request_review: copilot  # Request Copilot review
  # request_review: true     # Request repo default reviewers

# docs_language: English
---

# Workflow

Research → Propose → Specs → Design → Pre-Flight → Tests → Tasks → Apply → Audit → Finalize → Review

## Context

Always read and follow .specshift/CONSTITUTION.md before proceeding.
All workflow artifacts (research, proposal, specs, design, preflight, tests, tasks, audit)
must be written in English regardless of docs_language.

## Action: propose

### Instruction

Create change workspace if needed, then traverse the pipeline generating artifacts.
If no change exists: ask user what to build, derive kebab-case name, create workspace (with worktree if enabled).
Lazy worktree cleanup: before creating, check for stale worktrees. Auto-clean completed proposals and merged PRs. For closed PRs or branches inactive beyond stale_days, prompt the user before cleanup. Read proposals from worktree filesystem paths.
Checkpoint/resume: skip completed artifacts, resume from first incomplete step.
Design review checkpoint: when auto_approve is false, pause after design for user alignment. When auto_approve is true, skip the design checkpoint and continue.
Preflight checkpoint: PASS → continue, PASS WITH WARNINGS → pause for acknowledgment, BLOCKED → stop.
audit artifact: stop before audit and suggest specshift apply.

## Action: init

### Instruction

Project initialization and health check.
Mode detection:
- Fresh (no WORKFLOW.md): install templates, scan codebase, generate constitution, generate AGENTS.md (full body) and CLAUDE.md (one-line @AGENTS.md import stub)
- Update (templates outdated): merge plugin template updates with local customizations
- Re-sync (all installed): detect spec drift (code vs specs) + docs drift (docs vs specs)
Bootstrap files: AGENTS.md is the agnostic single source of truth for agent directives (read by Codex natively, by Claude Code via the documented @AGENTS.md memory-import). On fresh init (neither file exists), generate both AGENTS.md (full body, from templates/agents.md) and CLAUDE.md (one-line @AGENTS.md import stub, from templates/claude.md). The CLAUDE.md stub is a pointer, not a content duplicate — SSOT is preserved. On re-init: never overwrite existing AGENTS.md or CLAUDE.md. If AGENTS.md exists, run a standard-sections check and report WARNING per missing section (passive — no auto-edit). If CLAUDE.md exists, run a WARNING-only check for the @AGENTS.md import line. If only CLAUDE.md exists (legacy project), generate AGENTS.md alongside it and leave the existing CLAUDE.md untouched. If only AGENTS.md exists, generate the CLAUDE.md import stub.
Report findings, suggest specshift propose for changes needed.

## Action: apply

### Instruction

Implement tasks from tasks.md, then generate audit.md.
QA loop: implement → generate audit.md → fix if FAIL → regenerate audit.md → until PASS.
Delete existing audit.md before starting implementation.
When auto_approve is false, pause at user testing gate. When auto_approve is true and audit.md verdict is PASS, skip user testing pause.
Fix loop: before applying any fix, classify the correction — Tweak (wrong value/typo/missing line → fix in place), Design Pivot (wrong files/approach/abstraction → update design.md + discard affected tasks → re-implement), or Scope Change (wrong requirements/target audience → update specs + design → full re-implementation). After any fix, regenerate audit.md before presenting to user.
Artifact staleness: for Design Pivot or Scope Change corrections, update ALL stale change artifacts (design.md, tasks.md affected sections, preflight.md if needed) before re-implementing. A stale artifact is one that still describes the original wrong approach. Specs must match implementation before proceeding.
Standard Tasks (post-implementation section) are NOT part of apply.
Constitution standard tasks: pre-merge executed during post-apply, post-merge remain as reminders.
Before committing, mark all standard task checkboxes as complete except post-merge.
After audit.md PASS, commit and push implementation.

## Action: finalize

### Instruction

Post-approval finalization, executed sequentially:
1. Changelog: incremental entries from completed change
2. Docs: regenerate affected capability docs, ADRs, README
3. Version-bump: if the constitution defines a version-bump convention, follow it; otherwise skip
On error in one step: continue with next, report failures at end.
Check audit.md exists with verdict PASS before proceeding.

## Action: review

### Instruction

PR review-to-merge lifecycle. Re-entrant: can be run in any session.
State assessment: determine PR number from current branch, read PR state (draft, reviews, comments, checks) using available GitHub tooling.
- **Draft transition:** If PR is draft, mark ready for review, update body with change summary.
- **Clean-tree check:** Before review dispatch, verify the working tree is clean. If uncommitted changes exist (e.g., from finalize compilation), commit and push them first.
- **Review dispatch:** If no reviews requested and `review.request_review` is configured, request external review using available GitHub tooling. If unavailable or request fails, log warning and continue.
- **Activity subscription:** Subscribe to PR activity for real-time updates if tooling supports it.
- **Comment processing:** Process unresolved review comments: read each thread, implement fixes, reply explaining action taken, resolve threads. If a comment requires a fundamental change, inform the user and suggest a new specshift propose.
- **Self-check:** After fixes, commit, push, run built-in self-check. Fix any findings.
- **Cycle limit:** If reviewer posts new comments, return to Comment processing. Safety limit: max 3 cycles, then pause.
- **CI gate:** When no unresolved comments remain, check CI. If pending, report status and suggest waiting. If failing, report failures and stop.
- **Pre-merge summary:** If CI is passing, post a summary comment on the PR (threads processed/resolved, fixes list, self-check result, cycles completed). Use `<!-- specshift:review-summary -->` marker to detect and update existing summary on re-entrant runs. If posting fails, log warning and continue.
- **Review-pending gate:** If a review was requested (via `review.request_review` config) but no review decision has been submitted yet, report "Review pending — waiting for reviewer decision" and suggest re-running `specshift review` later. Do NOT offer merge.
- **Merge confirmation:** If no review is pending, ask user for explicit merge confirmation.
- **Merge execution:** After user confirms, set proposal status to `completed`, commit and push (so the status change is included in the squash). Then merge the PR via squash. Compose the commit message — title: `<PR title> (#<number>)`, body: proposal Why section, blank line, What Changes bullets, then issue-closing references (e.g., `Closes #N`). Do not duplicate issue-closing references already present in the Why section. Do not use GitHub's default squash message. Post-merge: clean up worktree if applicable.
When auto_approve is true and `review.request_review` is `false` or absent: skip Review dispatch and review waiting, proceed to CI gate + Pre-merge summary + Merge confirmation.
When auto_approve is true and `review.request_review` is configured (`copilot` or `true`): dispatch the review and wait for the decision normally — auto_approve does NOT skip review when a review is explicitly configured.
If session may end before review arrives: report state and suggest re-running specshift review later.
