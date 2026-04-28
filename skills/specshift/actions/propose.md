# Requirements


### Requirement: Propose as Single Entry Point for Pipeline Traversal
The `specshift propose` command SHALL serve as the single entry point for all pipeline traversal operations. This includes: (1) creating new change workspaces (with worktree if enabled), (2) checkpoint/resume of partially completed pipelines, and (3) full lifecycle execution from research through tasks. When invoked with a description or name and no existing change matches, propose SHALL create a new change workspace. When invoked without a change name and existing changes are present, propose SHALL list active changes and use AskUserQuestion to let the user select which change to continue, showing the most recently modified change as recommended. When invoked with a description of what to build, propose SHALL derive a kebab-case name and create a new change. The `auto_approve` workflow configuration (defaults to `true` in WORKFLOW.md frontmatter) controls whether pipeline traversal proceeds without user confirmation at checkpoints. When `auto_approve` is absent or `true`, checkpoints are skipped on success paths. When explicitly set to `false`, the pipeline pauses at each checkpoint for user confirmation. Propose SHALL display artifact status for the current change, showing which artifacts are complete, in progress, or blocked.

**User Story:** As a developer I want a single command that handles workspace creation, progress display, and artifact generation, so that I don't need to remember different commands for different pipeline states.

#### Scenario: Propose creates new workspace from description
- **GIVEN** the user invokes `specshift propose` with a description like "add user authentication"
- **AND** no change with a matching name exists
- **WHEN** the action processes the input
- **THEN** it SHALL derive a kebab-case name (e.g., `add-user-auth`) and create a new change directory
- **AND** SHALL create a worktree if worktree mode is enabled in WORKFLOW.md

#### Scenario: Propose displays artifact status
- **GIVEN** a change workspace where research.md and proposal.md are complete
- **WHEN** the user runs `specshift propose`
- **THEN** the system SHALL display the status of all pipeline artifacts (research: done, proposal: done, specs: ready, design: blocked, preflight: blocked, tasks: blocked)

#### Scenario: Propose detects existing change and offers selection
- **GIVEN** existing changes under `.specshift/changes/` and the user invokes `specshift propose` without specifying a name
- **WHEN** the action detects active changes
- **THEN** it SHALL present a list of active changes using AskUserQuestion
- **AND** SHALL mark the most recently modified change as recommended

#### Scenario: Propose asks what to build when no context provided
- **GIVEN** no active changes exist and the user invokes `specshift propose` without a description
- **WHEN** the action processes the input
- **THEN** it SHALL ask the user what they want to build

### Requirement: Eight-Stage Pipeline
The system SHALL define an 8-stage artifact pipeline with the following stages in order: research, proposal, specs, design, preflight, tests, tasks, and audit. Each stage SHALL produce a verifiable artifact file. The pipeline stages SHALL execute in strict dependency order: research has no dependencies, proposal requires research, specs requires proposal, design requires specs, preflight requires design, tests requires preflight, tasks requires tests, and audit requires tasks. The audit artifact is generated during the apply phase (after implementation) rather than during artifact-forward generation. No stage SHALL be skippable; each MUST complete before the change is considered complete. The pipeline order SHALL be declared in the `pipeline` array of `.specshift/WORKFLOW.md` frontmatter. Each stage's metadata (generates, requires, instruction) SHALL be defined in the corresponding Smart Template's YAML frontmatter.

**User Story:** As a developer I want a structured pipeline that guides me from research through to implementation tasks, so that no critical thinking step is skipped and every decision is documented.

#### Scenario: Pipeline stages execute in dependency order
- **GIVEN** a new change workspace with no artifacts generated
- **WHEN** the user progresses through the pipeline
- **THEN** the system SHALL enforce the order: research first, then proposal, then specs, then design, then preflight, then tests, then tasks, then audit

#### Scenario: Skipping a stage is prevented
- **GIVEN** a change workspace where only research.md has been generated
- **WHEN** a user or agent attempts to generate specs (skipping proposal)
- **THEN** the system SHALL reject the attempt and report that the proposal artifact must be completed first

#### Scenario: All stages produce verifiable artifacts
- **GIVEN** a completed pipeline run
- **WHEN** the change workspace is inspected
- **THEN** it SHALL contain research.md, proposal.md, one or more `docs/specs/<capability>.md` files, design.md, preflight.md, tests.md, tasks.md, and audit.md

### Requirement: Artifact Dependencies
Each artifact in the pipeline SHALL declare its dependencies explicitly in the Smart Template's YAML frontmatter `requires` field. Skills SHALL enforce these dependencies by reading WORKFLOW.md and Smart Templates and checking artifact completion status via file existence before allowing generation of a dependent artifact. An artifact SHALL be considered complete when its corresponding file exists and is non-empty in the change workspace. For artifacts with glob patterns in the `generates` field (e.g., `specs/**/*.md`), completion SHALL be determined by at least one matching file existing.

**User Story:** As a developer I want the system to enforce artifact dependencies automatically, so that I cannot accidentally generate a design before the specs are written.

#### Scenario: Dependency check passes
- **GIVEN** a change workspace with completed research.md and proposal.md
- **WHEN** the system checks dependencies for the specs artifact
- **THEN** the dependency check SHALL pass because both research and proposal (the transitive chain) are complete

#### Scenario: Dependency check fails
- **GIVEN** a change workspace with only research.md completed
- **WHEN** the system checks dependencies for the design artifact
- **THEN** the dependency check SHALL fail and report that proposal and specs must be completed first

#### Scenario: Smart Template declares dependencies explicitly
- **GIVEN** a Smart Template file (e.g., `.specshift/templates/changes/proposal.md`)
- **WHEN** its YAML frontmatter is inspected
- **THEN** it SHALL have a `requires` field listing its direct dependencies by artifact ID (e.g., `[research]`)

#### Scenario: Artifact status computed from file existence
- **GIVEN** a change workspace with research.md and proposal.md present
- **WHEN** a skill computes artifact status by reading WORKFLOW.md and Smart Templates
- **THEN** research and proposal SHALL be marked as "done", specs as "ready", and design/preflight/tasks as "blocked"

### Requirement: Post-Artifact Commit and PR Integration
The `specshift propose` skill SHALL execute post-artifact commit logic after creating each artifact during pipeline traversal. The skill SHALL: (1) check the current branch — if already on `<change-name>` branch (e.g., in a worktree), skip branch creation; if on main, create the branch via `git checkout -b <change-name>`; if on another branch, switch to it via `git checkout <change-name>`, (2) stage and commit the change artifacts with a commit message in the format `specshift(<change-name>): <artifact-id>` (e.g., `specshift(fix-auth): research`), (3) push the branch to the remote, and (4) on the first push only, create a draft PR using available GitHub tooling. This logic lives in the skill (SKILL.md), not in WORKFLOW.md.

**User Story:** As a developer I want every artifact committed incrementally with a draft PR created on the first commit, so that my team has early visibility and every pipeline stage is tracked in version control.

#### Scenario: First artifact triggers branch and PR creation
- **GIVEN** a change workspace where no feature branch exists yet
- **AND** GitHub tooling is available (gh CLI, MCP tools, or API)
- **WHEN** the agent finishes creating the first artifact
- **THEN** the agent SHALL create a feature branch, commit, push, and create a draft PR

#### Scenario: Subsequent artifacts commit and push only
- **GIVEN** a change workspace with an existing feature branch and draft PR
- **WHEN** the agent finishes creating a subsequent artifact
- **THEN** the agent SHALL commit and push but SHALL NOT create a new PR

#### Scenario: Worktree skips branch creation
- **GIVEN** a change workspace in a git worktree already on the `<change-name>` branch
- **WHEN** the agent finishes creating an artifact
- **THEN** the agent SHALL skip the branch creation step and proceed directly to staging, committing, and pushing

#### Scenario: Graceful degradation without GitHub tooling
- **GIVEN** no GitHub tooling is available (no gh CLI, no MCP tools, no API access)
- **WHEN** the agent finishes creating the first artifact
- **THEN** the agent SHALL create the branch, commit, attempt push, and skip PR creation

#### Scenario: Post-artifact commit logic is in the skill
- **GIVEN** the propose skill with post-artifact commit logic defined in SKILL.md
- **WHEN** the agent finishes creating an artifact
- **THEN** the skill SHALL execute its built-in commit/push/PR logic without requiring a WORKFLOW.md field

### Requirement: Create Change Workspace

The system SHALL create a change workspace when the user invokes `specshift propose <change-name>`. The workspace directory SHALL use a creation-date prefix in the format `YYYY-MM-DD-<change-name>`, set at creation time and never changed. If `worktree.enabled` is `true` in WORKFLOW.md, the system SHALL create a git worktree (see "Create Worktree-Based Workspace" requirement) and then create the change directory inside it via `mkdir -p <worktree>/.specshift/changes/YYYY-MM-DD-<name>`. If worktree mode is not enabled, the workspace SHALL be created by running `mkdir -p .specshift/changes/YYYY-MM-DD-<name>`. The change name MUST be in kebab-case format. If the user provides a description instead of a name, the system SHALL derive a kebab-case name from the description. The system SHALL NOT proceed without a valid change name.

When the proposal artifact (`proposal.md`) is created for a change, the propose action SHALL include YAML frontmatter with tracking fields:
- `status: active` — change lifecycle (`active` → `review` when audit passes, `review` → `completed` after PR merge)
- `branch: <branch-name>` — git branch association
- `worktree: <worktree-path>` — only when worktree mode is enabled
- `capabilities` — structured list of affected capabilities:
  ```yaml
  capabilities:
    new: [capability-one]
    modified: [quality-gates, spec-format]
    removed: []
  ```
  This machine-readable field mirrors the Capabilities section in the proposal body and eliminates the need for skills to parse markdown sections.

These fields enable automated change context detection, active/completed filtering, worktree association, and capability lookup without relying on naming conventions or markdown parsing.

**User Story:** As a developer I want to create a new change workspace with a date-prefixed directory, so that changes are chronologically ordered from creation and I can immediately begin the spec-driven workflow.

#### Scenario: Create workspace with date prefix

- **GIVEN** no change named "add-user-auth" exists
- **AND** today's date is 2026-04-01
- **WHEN** the user invokes `specshift propose add-user-auth`
- **THEN** the system creates the workspace at `.specshift/changes/2026-04-01-add-user-auth/`
- **AND** reads WORKFLOW.md to determine the artifact pipeline and displays the artifact status

#### Scenario: Derive name from description

- **WHEN** the user invokes `specshift propose` and provides the description "add user authentication"
- **AND** today's date is 2026-04-01
- **THEN** the system derives the kebab-case name `add-user-auth`
- **AND** creates the workspace at `.specshift/changes/2026-04-01-add-user-auth/`

#### Scenario: Proposal created with tracking frontmatter

- **GIVEN** a new change `add-user-auth` created on branch `add-user-auth` in worktree `.specshift/worktrees/add-user-auth`
- **AND** the proposal lists `user-auth` as a new capability and `quality-gates` as modified
- **WHEN** the proposal artifact is generated
- **THEN** `proposal.md` SHALL include YAML frontmatter with `status: active`, `branch: add-user-auth`, `worktree: .specshift/worktrees/add-user-auth`, and `capabilities: { new: [user-auth], modified: [quality-gates], removed: [] }`

#### Scenario: Proposal frontmatter without worktree

- **GIVEN** a new change created without worktree mode (`worktree.enabled: false`)
- **WHEN** the proposal artifact is generated
- **THEN** `proposal.md` SHALL include YAML frontmatter with `status: active`, `branch: <branch-name>`, and `capabilities`
- **AND** the `worktree` field SHALL be absent

#### Scenario: Reject invalid name format

- **GIVEN** the user provides a name containing uppercase letters or special characters (e.g., "Add_User Auth")
- **WHEN** the system attempts to create the workspace
- **THEN** the system SHALL ask the user for a valid kebab-case name
- **AND** SHALL NOT create the workspace until a valid name is provided

#### Scenario: Change name already exists

- **GIVEN** a change directory matching `*-add-user-auth` already exists under `.specshift/changes/`
- **WHEN** the user invokes `specshift propose add-user-auth`
- **THEN** the system SHALL NOT create a duplicate workspace
- **AND** SHALL suggest continuing the existing change instead

### Requirement: Create Worktree-Based Workspace

The system SHALL create a git worktree with a dedicated feature branch when the user invokes `specshift propose <change-name>` and `worktree.enabled` is `true` in WORKFLOW.md. The worktree path SHALL be computed by replacing `{change}` in the `worktree.path_pattern` field with the change name (without date prefix). Before creating the worktree, the system SHALL fetch the latest remote main branch. The system SHALL use the fetched remote main as the start-point for the new branch. The system SHALL then run `mkdir -p .specshift/changes/YYYY-MM-DD-<name>` inside the worktree. The system SHALL report the worktree path and branch name in its output. If the worktree path already exists as a git worktree, the system SHALL suggest switching to it instead of creating a new one.

**User Story:** As a developer I want each change to be created in its own git worktree, so that parallel changes are fully isolated and cannot conflict with each other.

#### Scenario: Create worktree when enabled

- **GIVEN** WORKFLOW.md has `worktree.enabled: true` and `worktree.path_pattern: .specshift/worktrees/{change}`
- **AND** no worktree for "add-user-auth" exists
- **AND** today's date is 2026-04-01
- **WHEN** the user invokes `specshift propose add-user-auth`
- **THEN** the system fetches the latest remote main branch
- **AND** creates a worktree based on the fetched remote main
- **AND** creates `.specshift/changes/2026-04-01-add-user-auth/` inside the worktree
- **AND** reports the worktree path and branch name

#### Scenario: Worktree already exists

- **GIVEN** WORKFLOW.md has `worktree.enabled: true`
- **AND** a worktree at `.specshift/worktrees/add-user-auth` already exists
- **WHEN** the user invokes `specshift propose add-user-auth`
- **THEN** the system SHALL NOT create a duplicate worktree
- **AND** SHALL suggest switching to the existing worktree

#### Scenario: Worktree disabled falls back to directory creation

- **GIVEN** WORKFLOW.md does not have `worktree.enabled: true` (absent or false)
- **AND** today's date is 2026-04-01
- **WHEN** the user invokes `specshift propose add-user-auth`
- **THEN** the system SHALL create `.specshift/changes/2026-04-01-add-user-auth/` in the current working tree

### Requirement: Lazy Worktree Cleanup at Change Creation

The `specshift propose` skill SHALL check for stale worktrees before creating a new change. The system SHALL run `git worktree list` and for each worktree (excluding the main working tree), determine if the associated change is completed or abandoned:

1. **Proposal status check**: Read `<worktree-path>/.specshift/changes/*/proposal.md` (using the worktree's filesystem path from `git worktree list`). If the proposal has `status: completed`, the change is done — auto-clean.
2. **Fallback — PR status (MERGED)**: If no proposal status is available or not `completed`, check the PR state for the branch using available GitHub tooling. If `MERGED`, the change is done — auto-clean.
3. **Fallback — PR status (CLOSED)**: If the PR state is `CLOSED`, the change is abandoned. Prompt the user for confirmation before cleanup, displaying the branch name and PR URL. This prompt SHALL NOT be suppressed by `auto_approve`.
4. **Fallback — inactivity**: If no PR exists (or no GitHub tooling is available), check the last commit date on the branch via `git log -1 --format=%ci <branch-name>`. If older than `worktree.stale_days` (default: 14), prompt the user for confirmation before cleanup.
5. **Fallback — git branch**: If none of the above apply, try `git branch -d <branch-name>` (only deletes if merged).

For completed changes (tiers 1–2), the system SHALL auto-remove the worktree without prompting. For abandoned or stale changes (tiers 3–4), the system SHALL prompt the user before removal. The cleanup sequence SHALL be: `git worktree remove <path>`, `git branch -D <branch-name>`, delete the remote branch. The system SHALL report cleaned-up worktrees before proceeding.

**User Story:** As a developer I want stale worktrees cleaned up automatically when I start a new change, so that completed, abandoned, and inactive branches don't accumulate on disk.

#### Scenario: Cleanup worktree with completed proposal status

- **GIVEN** a worktree exists at `.specshift/worktrees/feature-x` on branch `worktree-feature-x`
- **AND** `<worktree-path>/.specshift/changes/2026-04-10-feature-x/proposal.md` has `status: completed`
- **WHEN** the user invokes `specshift propose add-logging`
- **THEN** the system reads the proposal from the worktree filesystem path
- **AND** removes the worktree and deletes the branch
- **AND** reports "Cleaned up stale worktree: feature-x (completed)"

#### Scenario: Cleanup worktree via PR status fallback

- **GIVEN** a worktree on branch `fix-auth` with no proposal `status` field
- **AND** the PR for `fix-auth` has state "MERGED"
- **WHEN** the user invokes `specshift propose add-logging`
- **THEN** the system removes the worktree and deletes the branch

#### Scenario: Abandoned worktree detected via closed PR

- **GIVEN** a worktree on branch `worktree-abandoned-feature` with `status: active`
- **AND** the PR for `worktree-abandoned-feature` has state `CLOSED`
- **WHEN** the user invokes `specshift propose new-change`
- **THEN** the system prompts: "Worktree 'abandoned-feature' has a closed PR (not merged). Clean up? [y/N]"
- **AND** if the user confirms, the system removes the worktree and deletes the branch
- **AND** if the user declines, the worktree is preserved

#### Scenario: Inactive worktree detected via commit age

- **GIVEN** a worktree on branch `worktree-old-experiment` with `status: active`
- **AND** no PR exists for `worktree-old-experiment`
- **AND** the last commit on the branch is 21 days old
- **AND** `worktree.stale_days` is 14
- **WHEN** the user invokes `specshift propose new-change`
- **THEN** the system prompts: "Worktree 'old-experiment' has had no commits for 21 days. Clean up? [y/N]"
- **AND** if the user confirms, the system removes the worktree and deletes the branch
- **AND** if the user declines, the worktree is preserved

#### Scenario: Recent worktree within inactivity threshold preserved

- **GIVEN** a worktree on branch `worktree-recent-work` with `status: active`
- **AND** no PR exists, and the last commit is 5 days old
- **AND** `worktree.stale_days` is 14
- **WHEN** the user invokes `specshift propose new-change`
- **THEN** the system SHALL NOT prompt for cleanup of `recent-work`

#### Scenario: No stale worktrees

- **GIVEN** no worktrees exist besides the main working tree
- **WHEN** the user invokes `specshift propose add-logging`
- **THEN** the system proceeds directly to change creation without cleanup messages

#### Scenario: Worktree with active change preserved

- **GIVEN** a worktree exists at `.specshift/worktrees/wip-feature` on branch `wip-feature`
- **AND** the proposal has `status: active`
- **AND** the last commit is within the `stale_days` threshold
- **WHEN** the user invokes `specshift propose add-logging`
- **THEN** the system SHALL NOT remove the `wip-feature` worktree

#### Scenario: Cleanup without GitHub tooling and no proposal status

- **GIVEN** a worktree exists on branch `fix-auth`
- **AND** proposal has no `status` field and no GitHub tooling is available
- **AND** the last commit is within the `stale_days` threshold
- **WHEN** the system attempts cleanup
- **THEN** the system falls back to `git branch -d fix-auth`
- **AND** if the branch was fully merged to main, it is deleted and the worktree removed
- **AND** if the branch was NOT merged, `git branch -d` fails and the worktree is preserved

### Requirement: Change Context Detection

The router SHALL detect the active change using proposal frontmatter before falling through to legacy detection, and pass the resolved context to the dispatched action. The detection sequence SHALL be:

1. **Proposal frontmatter lookup**: Scan `.specshift/changes/*/proposal.md` for a proposal whose `branch` field matches the current branch (`git rev-parse --abbrev-ref HEAD`). If found, auto-select that change.
2. **Fallback — worktree convention**: If no proposal has a matching `branch` field, check if the current working directory is inside a git worktree (inspect `git rev-parse --git-dir` for `/worktrees/`), derive the change name from the branch, and search for `.specshift/changes/*-<branch-name>/`.
3. **Fallback — directory listing**: If not in a worktree, list active changes and prompt the user.

If a match is found, the router SHALL auto-select the change and announce: "Detected change context: using change '<name>'". An explicit argument always overrides auto-detection.

**User Story:** As a developer I want the router to automatically know which change I'm working on using structured metadata, so that detection is reliable regardless of naming conventions.

#### Scenario: Auto-detect change via proposal branch field

- **GIVEN** the user is on branch `add-user-auth`
- **AND** `.specshift/changes/2026-04-01-add-user-auth/proposal.md` has frontmatter `branch: add-user-auth`
- **WHEN** any command is invoked via the router without an explicit change name
- **THEN** the router SHALL auto-select "2026-04-01-add-user-auth"
- **AND** announce "Detected change context: using change 'add-user-auth'"

#### Scenario: Fallback to worktree convention when proposal has no branch field

- **GIVEN** the user is in a git worktree on branch `legacy-change`
- **AND** `.specshift/changes/2026-03-01-legacy-change/proposal.md` has no YAML frontmatter
- **WHEN** any command is invoked via the router
- **THEN** the router SHALL fall back to worktree convention detection (branch name → directory glob)

#### Scenario: Fall through to directory listing when not in worktree

- **GIVEN** the user is in the main working tree (not a worktree)
- **AND** no proposal has a `branch` field matching the current branch
- **WHEN** any command is invoked via the router without an explicit change name
- **THEN** the router SHALL list active changes and prompt the user

#### Scenario: Explicit argument overrides auto-detection

- **GIVEN** the user is on branch `add-user-auth`
- **WHEN** a skill is invoked with explicit argument `other-change`
- **THEN** the router SHALL use "other-change" regardless of proposal frontmatter or worktree context

### Requirement: Preflight Quality Check

The system SHALL run a mandatory quality review before task creation when the user invokes `specshift propose`. The preflight check SHALL cover seven dimensions: (A) Traceability Matrix -- mapping each capability from the proposal's frontmatter `capabilities` field (falling back to parsing the Capabilities section if frontmatter is absent) to its corresponding spec at `docs/specs/<capability>.md` and verifying that the spec has been updated to reflect the proposed changes, (B) Gap Analysis -- identifying missing edge cases, error handling, and empty states, (C) Side-Effect Analysis -- assessing impact on existing systems and regression risks, (D) Constitution Check -- verifying consistency with project rules in constitution.md, (E) Duplication and Consistency -- detecting overlaps and contradictions across specs, (F) Marker Audit -- auditing all assumption and review markers from spec.md and design.md, and (G) Draft Spec Validation -- verifying that all specs with `status: draft` have a `change` value matching the current change directory name. Specs with `status: draft` belonging to a different change SHALL be flagged as BLOCKED. Specs with `status: draft` and no `change` field SHALL be flagged as WARNING. The Marker Audit SHALL:
1. Collect all `<!-- ASSUMPTION: ... -->` tags and verify each has an accompanying visible list item. Assumptions written entirely inside HTML comments (no visible text) SHALL be flagged as format violations.
2. Rate each valid assumption as Acceptable Risk, Needs Clarification, or Blocking.
3. Scan for any remaining `<!-- REVIEW -->` or `<!-- REVIEW: ... -->` markers. Any REVIEW marker found SHALL be rated as Blocking, because REVIEW markers must be resolved before implementation.

The system SHALL produce a `preflight.md` artifact containing findings and a verdict of PASS, PASS WITH WARNINGS, or BLOCKED. The system SHALL NOT auto-fix issues; it SHALL report findings for the user to resolve. The system SHALL NOT proceed to task creation if blockers are found. If the verdict is PASS WITH WARNINGS, the system SHALL pause and require explicit user acknowledgment of the warnings before proceeding to task creation. The system SHALL NOT auto-accept warnings or continue without the user reviewing each warning.

- All change artifacts (specs, design) are available and up to date when preflight is invoked. <!-- ASSUMPTION: Artifact availability -->

**User Story:** As a developer I want a thorough quality review of my specs and design before tasks are created, so that implementation is based on complete, consistent, and well-traced requirements with no unresolved markers.

#### Scenario: Preflight passes with no issues

- **GIVEN** a change named "add-user-auth" with complete specs and design artifacts
- **AND** all requirements have scenarios, no gaps are detected, all assumptions have visible text, and no REVIEW markers remain
- **WHEN** the user invokes `specshift propose add-user-auth`
- **THEN** the system reads constitution.md, all change artifacts, and existing specs
- **AND** produces `preflight.md` covering all dimensions
- **AND** the verdict is "PASS"
- **AND** the summary shows 0 blockers, 0 warnings

#### Scenario: Preflight finds invisible assumption

- **GIVEN** a change with a spec containing `<!-- ASSUMPTION: External API rate limit is 1000/min -->` with no visible list item
- **WHEN** the user invokes `specshift propose`
- **THEN** the Marker Audit flags the invisible assumption as a format violation
- **AND** the verdict is "BLOCKED"
- **AND** the system recommends adding visible text: `- External API rate limit is 1000/min. <!-- ASSUMPTION: API rate limit -->`

#### Scenario: Preflight finds unresolved REVIEW marker

- **GIVEN** a change where design.md contains `<!-- REVIEW: confirm caching strategy -->`
- **WHEN** the user invokes `specshift propose`
- **THEN** the Marker Audit flags the REVIEW marker as Blocking
- **AND** the verdict is "BLOCKED"
- **AND** the system informs the user that REVIEW markers must be resolved before proceeding

#### Scenario: Preflight detects contradiction with constitution

- **GIVEN** a design.md that proposes adding a project-level package.json
- **AND** the constitution states "Package manager: npm (global installs only -- no project-level package.json)"
- **WHEN** the system runs the Constitution Check
- **THEN** the system flags a contradiction between design.md and the constitution
- **AND** classifies it as a blocker
- **AND** recommends either updating the design to comply or updating the constitution if the rule should change

#### Scenario: Preflight with warnings requires user acknowledgment

- **GIVEN** a change where all requirements have scenarios, all assumptions have visible text, and no REVIEW markers remain
- **BUT** a minor gap is detected (missing error handling for an unlikely edge case)
- **WHEN** the user invokes `specshift propose`
- **THEN** the verdict is "PASS WITH WARNINGS"
- **AND** each warning is listed with a recommendation
- **AND** the system SHALL pause and ask the user to acknowledge each warning
- **AND** the system SHALL NOT proceed to task creation until the user explicitly confirms

#### Scenario: Preflight validates draft spec ownership
- **GIVEN** a change named `2026-04-08-my-change`
- **AND** `docs/specs/quality-gates.md` has `status: draft` and `change: 2026-04-08-my-change`
- **AND** `docs/specs/user-auth.md` has `status: draft` and `change: 2026-04-01-other-change`
- **WHEN** the user invokes `specshift propose 2026-04-08-my-change`
- **THEN** the Draft Spec Validation dimension SHALL flag `user-auth` as BLOCKED (draft owned by different change)
- **AND** SHALL confirm `quality-gates` as valid (draft owned by current change)

#### Scenario: Preflight detects orphaned draft spec
- **GIVEN** a spec with `status: draft` but no `change` field
- **WHEN** the user invokes `specshift propose`
- **THEN** the Draft Spec Validation SHALL flag it as WARNING: "Draft spec with no change owner"

#### Scenario: Required artifacts missing

- **GIVEN** a change where specs exist but design.md has not been created
- **WHEN** the user invokes `specshift propose`
- **THEN** the system SHALL abort the preflight
- **AND** SHALL report which required artifacts are missing
- **AND** SHALL suggest running `specshift propose` to generate them
