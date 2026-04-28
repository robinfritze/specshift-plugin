# Requirements


### Requirement: Install Workflow
The system SHALL provide `specshift init` as the single entry point for project setup. The init command SHALL: (1) copy pipeline Smart Templates from the plugin's `templates/` directory into the project's `.specshift/templates/` directory — excluding bootstrap templates (`workflow.md`, `constitution.md`, `agents.md`, `claude.md`) which are used only to generate their target files, (2) generate `.specshift/WORKFLOW.md` from the plugin's workflow template (skip if WORKFLOW.md already exists), (3) generate `.specshift/CONSTITUTION.md` from the plugin's constitution template if none exists, and (4) generate the bootstrap files (`AGENTS.md` + `CLAUDE.md`) per the "Bootstrap Files Generation" requirement. The init command SHALL be idempotent — running it on an already-initialized project SHALL skip completed steps.

The init command SHALL check for GitHub tooling availability (gh CLI, MCP tools, or API). If GitHub tooling is available and authenticated, the init command SHALL ask the user whether to enable worktree-based change isolation. If the user opts in, the init command SHALL uncomment the `worktree:` section in the generated WORKFLOW.md and set `enabled: true`. The init command SHALL also offer to configure the GitHub repository merge strategy for rebase-merge using available GitHub tooling.

The init command SHALL NOT install any external CLI tools or require Node.js/npm as prerequisites.

The init command SHALL ensure target directories exist (via `mkdir -p`) before copying files.

**User Story:** As a new user I want a single `specshift init` command that sets up everything including optional worktree mode, so that I do not have to manually configure the project.

#### Scenario: First-time project initialization
- **GIVEN** a project directory without the spec-driven workflow installed
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL copy Smart Templates from the plugin's `templates/` directory to `.specshift/templates/`, copy WORKFLOW.md from the plugin's workflow template, create `.specshift/CONSTITUTION.md` placeholder, generate `AGENTS.md` (full body) and `CLAUDE.md` (`@AGENTS.md` import stub) from their respective bootstrap templates, and verify the setup

#### Scenario: Idempotent re-initialization
- **GIVEN** a project that has already been initialized
- **WHEN** the user runs `specshift init` again
- **THEN** the system SHALL skip already-completed steps, preserve existing CONSTITUTION.md and WORKFLOW.md, and report what was already in place

#### Scenario: WORKFLOW.md copied from template
- **GIVEN** a project directory without `.specshift/WORKFLOW.md`
- **AND** the plugin ships a `workflow.md` bootstrap template
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL copy workflow.md to `.specshift/WORKFLOW.md`

#### Scenario: Worktree opt-in during init
- **GIVEN** GitHub tooling is available and authenticated
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL ask whether to enable worktree-based change isolation
- **AND** if the user opts in, SHALL uncomment the `worktree:` section in WORKFLOW.md and set `enabled: true`
- **AND** SHALL offer to configure the GitHub repo for rebase-merge

#### Scenario: No GitHub tooling available
- **GIVEN** no GitHub tooling is available or not authenticated
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL skip the worktree opt-in question
- **AND** SHALL leave the `worktree:` section commented out in WORKFLOW.md

### Requirement: Bootstrap Files Generation
The `specshift init` command SHALL generate two bootstrap files at the project root from the plugin's bootstrap templates: `AGENTS.md` (full body, agnostic source of truth, generated from the `agents.md` template) and `CLAUDE.md` (one-line `@AGENTS.md` import stub, generated from the `claude.md` template). Both files are generated unconditionally on fresh init — there is no environment detection that picks one over the other. The agnostic body lives in `AGENTS.md`; Codex CLI reads it natively, Claude Code reads it via the `@AGENTS.md` import expanded from `CLAUDE.md`.

The generated `AGENTS.md` SHALL contain at minimum: (1) a `## Workflow` section directing all changes through the spec-driven workflow, (2) a `## Planning` section requiring explicit scope commitment before exiting plan mode and workflow routing through the specshift skill for implementation steps, and (3) a `## Knowledge Management` section directing the agent to use transparent artifacts instead of auto-memory for project knowledge. The agent SHALL adapt the template content to include project-specific rules discovered during the codebase scan, using REVIEW markers for uncertain items (same pattern as constitution generation).

If either file already exists, init SHALL skip its generation rather than overwrite — existing files are authoritative. If only one of the two files exists, init SHALL generate the missing one (so a project bootstrapped on Claude Code first picks up an `AGENTS.md` later when re-init runs, and vice versa).

For re-init on an already-bootstrapped project, the section-completeness check SHALL apply to `AGENTS.md` (which carries the normative content). Missing standard sections in `AGENTS.md` SHALL be reported as WARNING. The `CLAUDE.md` stub SHALL only be checked for the presence of an `@AGENTS.md` import line — if the import line is missing, the system SHALL report it as WARNING. The system SHALL NOT modify either file in re-init mode — user edits are authoritative.

**User Story:** As a developer adopting the spec-driven workflow I want init to generate both AGENTS.md and CLAUDE.md so that whichever AI tool I use picks up the standard agent directives, and so that I can switch between Claude Code and Codex without re-bootstrapping.

#### Scenario: Both bootstrap files generated on fresh init
- **GIVEN** a project without an `AGENTS.md` file and without a `CLAUDE.md` file
- **AND** the plugin ships `agents.md` and `claude.md` bootstrap templates
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL generate `AGENTS.md` (full body) at the project root
- **AND** SHALL generate `CLAUDE.md` (`@AGENTS.md` import stub) at the project root
- **AND** the AGENTS.md SHALL contain `## Workflow`, `## Planning`, and `## Knowledge Management` sections
- **AND** the CLAUDE.md SHALL contain a single `@AGENTS.md` import line

#### Scenario: Existing AGENTS.md preserved on re-init
- **GIVEN** a project with an existing `AGENTS.md` containing all standard sections
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL NOT overwrite `AGENTS.md`
- **AND** SHALL check AGENTS.md against the bootstrap template's section headings
- **AND** SHALL report "AGENTS.md already exists — skipped (all standard sections present)"

#### Scenario: Existing CLAUDE.md preserved on re-init
- **GIVEN** a project with an existing `CLAUDE.md` (whether stub or full content from a prior install)
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL NOT overwrite `CLAUDE.md`
- **AND** SHALL check that CLAUDE.md contains an `@AGENTS.md` import line
- **AND** SHALL report a WARNING if the import line is missing

#### Scenario: AGENTS.md generated alone when CLAUDE.md already exists
- **GIVEN** a project with an existing `CLAUDE.md` (e.g., legacy single-target install) but no `AGENTS.md`
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL generate `AGENTS.md` (full body) at the project root
- **AND** SHALL NOT modify the existing `CLAUDE.md`
- **AND** SHALL report a WARNING if the existing CLAUDE.md does not contain an `@AGENTS.md` import line

#### Scenario: CLAUDE.md stub generated alone when AGENTS.md already exists
- **GIVEN** a project with an existing `AGENTS.md` (e.g., from a prior Codex install) but no `CLAUDE.md`
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL generate `CLAUDE.md` (`@AGENTS.md` import stub) at the project root
- **AND** SHALL NOT modify the existing `AGENTS.md`

#### Scenario: AGENTS.md includes project-specific rules
- **GIVEN** a project with specific conventions discovered during the codebase scan
- **WHEN** the init command generates AGENTS.md
- **THEN** the generated file SHALL include project-specific agent rules beyond the standard sections
- **AND** uncertain items SHALL be marked with `<!-- REVIEW -->` for user resolution

#### Scenario: AGENTS.md missing standard section detected on re-init
- **GIVEN** a project with an existing `AGENTS.md` that contains a `## Workflow` section but lacks `## Planning` or `## Knowledge Management`
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL NOT overwrite or modify `AGENTS.md`
- **AND** SHALL report WARNING for each missing standard section (e.g., "AGENTS.md missing standard section: Planning")
- **AND** SHALL suggest the user add the missing section manually

### Requirement: Template Merge on Re-Init
When `specshift init` runs on an already-initialized project (re-init after plugin update), the system SHALL use Smart Template `template-version` fields to detect user customizations and merge plugin updates instead of blindly overwriting. For each template file in the plugin's `templates/` directory:

1. **Read** the plugin template's `template-version` field and the local template's `template-version` field at `.specshift/templates/<path>`.
2. **Compare versions:**
   - If the local template does not exist: copy the plugin template (fresh install).
   - If the local `template-version` matches the plugin `template-version` AND content is identical: skip (already up to date).
   - If the local `template-version` matches the plugin `template-version` BUT content differs: the user has customized the template. Keep the local version and report: "Template <name> has local customizations — skipped."
   - If the plugin `template-version` is higher than the local `template-version` AND the local content matches the previous plugin version (no user changes): update silently to the new plugin template.
   - If the plugin `template-version` is higher AND the local content has been customized: merge is needed. The system SHALL present both versions to the user and ask them to resolve differences. Report: "Template <name> has both local customizations and plugin updates — merge required."
   - If the local template has no `template-version` field (legacy): treat as version 0 and apply the same logic (likely results in silent update if content matches plugin template, or merge prompt if customized).

The merge detection SHALL apply to all Smart Templates including docs templates in subdirectories, WORKFLOW.md, and CONSTITUTION.md. WORKFLOW.md is especially important for merge detection because the plugin frequently updates behavioral fields (`apply.instruction`, `context`) while users customize project-specific fields (`worktree`, `docs_language`, pipeline order). The existing skip-if-exists behavior for WORKFLOW.md is replaced by version-based merge detection.

For CONSTITUTION.md, the merge operates at **section level**: the system SHALL compare the template's section headings (e.g., `## Tech Stack`, `## Architecture Rules`, `## Standard Tasks`) against the existing CONSTITUTION.md. Missing sections from a newer template version SHALL be offered to the user for interactive generation (the agent reads the codebase and proposes content for the new section, as bootstrap does). Existing sections with user content SHALL be preserved. The generated CONSTITUTION.md SHALL include a `template-version` field in YAML frontmatter to track which template version generated its structure.

For CLAUDE.md, the merge operates at **section level with WARNING-only reporting**: the system SHALL compare the bootstrap template's section headings (e.g., `## Workflow`, `## Knowledge Management`) against the existing CLAUDE.md. Missing standard sections SHALL be reported as WARNING (e.g., "CLAUDE.md missing standard section: Knowledge Management"). The system SHALL NOT modify the existing CLAUDE.md — user edits are authoritative. This check ensures that standard agent directives are not silently lost when CLAUDE.md is manually edited or created before the plugin is installed.

**User Story:** As a user who has customized my templates I want plugin updates to preserve my customizations, so that re-running `specshift init` after a plugin update does not silently destroy my changes.

#### Scenario: Unchanged template updated silently
- **GIVEN** a local template `.specshift/templates/changes/research.md` with `template-version: 1` matching the plugin template content exactly
- **AND** the plugin update has `template-version: 2` with updated instruction text
- **WHEN** the user runs `specshift init`
- **THEN** the local template SHALL be replaced with the plugin's template-version 2 template
- **AND** the report SHALL show "Template research.md updated (v1 → v2)"

#### Scenario: User-customized template preserved
- **GIVEN** a local template `.specshift/templates/changes/research.md` with `template-version: 1` but modified instruction content
- **AND** the plugin template also has `template-version: 1`
- **WHEN** the user runs `specshift init`
- **THEN** the local template SHALL NOT be overwritten
- **AND** the report SHALL show "Template research.md has local customizations — skipped"

#### Scenario: Customized template with plugin update triggers merge
- **GIVEN** a local template with `template-version: 1` and custom content
- **AND** the plugin template has `template-version: 2` with new content
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL present both versions and ask the user to resolve
- **AND** SHALL NOT overwrite the local template without user confirmation

#### Scenario: Constitution gets new section from template update
- **GIVEN** an existing `.specshift/CONSTITUTION.md` with `template-version: 1` containing Tech Stack, Architecture Rules, Code Style, Constraints, Conventions
- **AND** the plugin's constitution template has `template-version: 2` with a new `## Security Rules` section
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL detect the missing "Security Rules" section
- **AND** SHALL offer to generate content for it based on the codebase
- **AND** SHALL preserve all existing sections and their user content

#### Scenario: Constitution with all sections up to date
- **GIVEN** an existing CONSTITUTION.md with `template-version: 2` matching the plugin template
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL skip CONSTITUTION.md (already up to date)

#### Scenario: Legacy template without version field
- **GIVEN** a local template with no `template-version` field in frontmatter
- **AND** the plugin template has `template-version: 1`
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL treat the local version as 0
- **AND** SHALL update silently if content matches the plugin template, or prompt for merge if customized

### Requirement: First-Run Codebase Scan
The `specshift init` command SHALL analyze the entire project codebase on first run (when no `.specshift/CONSTITUTION.md` exists or it is a placeholder). The scan SHALL identify the tech stack, frameworks, languages, file structure, configuration patterns, dependency management approach, coding conventions, and version files (e.g., `package.json`, `pyproject.toml`, `Cargo.toml`, `plugin.json`, `setup.cfg`). The scan results SHALL be used as input for constitution generation. The init command SHALL handle projects of any size, skipping binary files and respecting `.gitignore` patterns.

**User Story:** As a developer adopting spec-driven development on an existing project I want init to understand my codebase automatically, so that the generated constitution reflects my actual project rather than generic defaults.

#### Scenario: First-run scan of an existing project
- **GIVEN** a project with source code, configuration files, and dependencies but no `.specshift/CONSTITUTION.md`
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL scan the entire codebase and identify the tech stack, languages, frameworks, file structure, and coding conventions

#### Scenario: Large project with binary files
- **GIVEN** a project containing source code, images, compiled binaries, and a `.gitignore` file
- **WHEN** the init scan runs
- **THEN** the system SHALL skip binary files and files matching `.gitignore` patterns, analyzing only text-based source and configuration files

### Requirement: Constitution Generation
The `specshift init` command SHALL generate a `CONSTITUTION.md` file based on the observed patterns from the codebase scan. The constitution SHALL include Tech Stack, Architecture Rules, Code Style, Constraints, Conventions, and Standard Tasks sections. The Tech Stack, Architecture Rules, Code Style, Constraints, and Conventions sections SHALL be populated with project-specific values from the scan. If version files are detected, the Conventions section SHALL include a version-bump convention describing which file(s) to bump and how. If no version files are found, the version-bump convention SHALL be omitted — the finalize workflow skips the version-bump step when no convention is defined. The Standard Tasks section SHALL be generated empty with an HTML comment explaining its purpose, so that new projects are aware of the feature and know where to define project-specific post-implementation steps.

**User Story:** As a developer I want the constitution to be auto-generated from my codebase, so that it accurately captures my project's existing patterns rather than requiring me to write it from scratch.

#### Scenario: Constitution generated from scan results
- **GIVEN** the codebase scan has completed and identified TypeScript, React, and Jest as the primary technologies
- **WHEN** the constitution generation phase runs
- **THEN** the system SHALL create `.specshift/CONSTITUTION.md` with Tech Stack listing TypeScript, React, and Jest, along with Architecture Rules, Code Style, Constraints, Conventions, and an empty Standard Tasks section reflecting the observed patterns

#### Scenario: Standard Tasks section present but empty on first run
- **GIVEN** a new project being initialized for the first time
- **WHEN** the constitution generation phase runs
- **THEN** the generated constitution SHALL contain a `## Standard Tasks` section
- **AND** the section SHALL be empty except for an HTML comment explaining that project-specific extras can be added here to appear in every tasks.md after the universal standard tasks

#### Scenario: Constitution respects existing conventions
- **GIVEN** a project using 4-space indentation, camelCase variables, and conventional commits
- **WHEN** the constitution is generated
- **THEN** the Code Style section SHALL reflect the 4-space indentation and camelCase convention, and the Conventions section SHALL reference the conventional commits format

### Requirement: Documentation Drift Verification (Health Check)

The system SHALL verify that generated documentation accurately reflects the current state of specs as part of `specshift init` project-level health checks. The verification SHALL assess three dimensions:

1. **Capability Docs vs Specs** — For each spec in `docs/specs/*.md`, the system SHALL check that a corresponding capability doc exists in `docs/capabilities/` and that the doc's Purpose section aligns with the spec's Purpose, and that documented features cover the spec's requirements. Missing capability docs SHALL be classified as CRITICAL. Capability docs that omit requirements present in the spec SHALL be classified as WARNING.

2. **ADRs vs Design Decisions** — The system SHALL scan all completed change directories' `design.md` files in `.specshift/changes/*/design.md`. For each design.md, the system SHALL first check the frontmatter `has_decisions` field — if `false` or absent, skip this design.md. If `true`, scan for Decisions tables and verify that each decision has a corresponding ADR in `docs/decisions/`. Missing ADRs SHALL be classified as WARNING. The system SHALL recognize manual ADRs (prefix `adr-MNNN`) and skip them during the cross-check, since they have no corresponding design.md entry.

3. **README vs Current State** — The system SHALL verify that `docs/README.md` lists all current capabilities from `docs/specs/` in its capabilities table, that the Key Design Decisions table references existing ADRs, and that the architecture overview is consistent with `.specshift/CONSTITUTION.md`. Missing capabilities in the README SHALL be classified as CRITICAL. Stale ADR references (pointing to deleted or renamed ADRs) SHALL be classified as WARNING.

Each issue found SHALL be classified as:
- **CRITICAL** — documentation is missing or fundamentally incorrect (e.g., capability doc missing entirely, README omits a capability)
- **WARNING** — documentation exists but has drifted from specs (e.g., requirement not reflected in capability doc, stale ADR reference)
- **INFO** — minor discrepancy or observation that may be intentional (e.g., manual ADR with no matching design decision, capability doc has extra context beyond spec)

The system SHALL produce a verification report with a summary (total issues by severity), followed by findings grouped by dimension, with file references for each issue. The report SHALL conclude with a verdict: **CLEAN** (0 critical, 0 warnings), **DRIFTED** (warnings but no criticals), or **OUT OF SYNC** (at least one critical). The system SHALL NOT automatically fix any issues; it SHALL recommend running `specshift finalize` to regenerate drifted documentation.

The system SHALL gracefully handle missing documentation directories: if `docs/capabilities/` does not exist, the system SHALL report all capabilities as missing (CRITICAL) rather than erroring. If `docs/decisions/` does not exist, the system SHALL skip the ADR dimension and note it. If `docs/README.md` does not exist, the system SHALL report it as a single CRITICAL issue.

**User Story:** As a developer I want to verify that my generated documentation still matches the current specs, so that I can detect documentation drift before it causes confusion or misinformation.

#### Scenario: All documentation is in sync

- **GIVEN** a project with 5 capabilities, each having a corresponding capability doc in `docs/capabilities/`
- **AND** all completed changes' design decisions have corresponding ADRs in `docs/decisions/`
- **AND** `docs/README.md` lists all 5 capabilities and references valid ADRs
- **WHEN** the user invokes `specshift init`
- **THEN** the system produces a verification report
- **AND** all three dimensions show no issues
- **AND** the verdict is "CLEAN"

#### Scenario: Capability doc missing for a spec

- **GIVEN** a project with specs for "user-auth" and "data-export"
- **AND** `docs/capabilities/` contains only `user-auth.md` (no `data-export.md`)
- **WHEN** the user invokes `specshift init`
- **THEN** the Capability Docs dimension reports a CRITICAL issue: "Missing capability doc for data-export"
- **AND** the recommendation is "Run `specshift finalize` to generate the missing documentation"
- **AND** the verdict is "OUT OF SYNC"

#### Scenario: Capability doc omits a requirement from spec

- **GIVEN** a spec for "quality-gates" with requirements for Preflight, Verify, and Docs-Verify
- **AND** the capability doc `docs/capabilities/quality-gates.md` only documents Preflight and Verify
- **WHEN** the system checks Capability Docs vs Specs
- **THEN** the report includes a WARNING: "Capability doc for quality-gates missing requirement: Documentation Drift Verification"
- **AND** the verdict is "DRIFTED"

#### Scenario: README missing a capability

- **GIVEN** a project with 6 specs in `docs/specs/`
- **AND** `docs/README.md` capabilities table lists only 5 of them
- **WHEN** the system checks README vs Current State
- **THEN** the report includes a CRITICAL issue: "README capabilities table missing: <capability-name>"
- **AND** recommends "Run `specshift finalize` to regenerate the README"

#### Scenario: Stale ADR reference in README

- **GIVEN** a README Key Design Decisions table referencing `adr-005-old-caching.md`
- **AND** the file `docs/decisions/adr-005-old-caching.md` does not exist
- **WHEN** the system checks README vs Current State
- **THEN** the report includes a WARNING: "README references non-existent ADR: adr-005-old-caching.md"

#### Scenario: Documentation directory does not exist

- **GIVEN** a project where `docs/capabilities/` has not been created yet
- **WHEN** the user invokes `specshift init`
- **THEN** the system reports each spec as a CRITICAL missing capability doc
- **AND** does not error or abort
- **AND** recommends "Run `specshift finalize` to generate initial documentation"

#### Scenario: No design decisions to check

- **GIVEN** a project with no completed changes in `.specshift/changes/`
- **WHEN** the user invokes `specshift init`
- **THEN** the system skips the ADR dimension
- **AND** notes "No design decisions to verify against"
- **AND** still checks the other two dimensions

#### Scenario: Manual ADR without design decision is not flagged

- **GIVEN** a manual ADR `docs/decisions/adr-M001-initial-approach.md`
- **AND** no corresponding entry in any completed change's design.md Decisions table
- **WHEN** the system checks ADRs vs Design Decisions
- **THEN** the manual ADR is recognized by its `adr-MNNN` prefix
- **AND** no issue is raised for it

### Requirement: Recovery Mode (Spec Drift Detection)
The `specshift init` command SHALL detect when specs already exist in `docs/specs/`. When existing specs are found, the init command SHALL enter recovery mode: scanning the current codebase, comparing it against existing specs, and reporting drift findings. Recovery mode SHALL NOT overwrite existing specs or the constitution. Instead, it SHALL produce a drift report listing discrepancies between the codebase and the specs, and suggest corrective actions (e.g., `specshift propose hotfix-xyz` for small drift or a full re-bootstrap for large drift).

**User Story:** As a maintainer whose codebase has drifted from its specs I want init to detect and report the drift, so that I can decide how to reconcile without losing existing spec work.

#### Scenario: Recovery mode with minor drift
- **GIVEN** a project with existing specs and a codebase where two functions have been renamed without spec updates
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL detect the existing specs, enter recovery mode, report the two naming discrepancies, and suggest using `specshift propose hotfix-xyz` to create a targeted change for the drift

#### Scenario: Recovery mode with major drift
- **GIVEN** a project with existing specs and a codebase where an entire module has been rewritten without spec updates
- **WHEN** the user runs `specshift init`
- **THEN** the system SHALL detect the existing specs, enter recovery mode, report the extensive drift, and suggest a full re-bootstrap after backing up existing specs

#### Scenario: Recovery mode does not overwrite
- **GIVEN** a project with existing specs and constitution
- **WHEN** the user runs `specshift init` and recovery mode activates
- **THEN** the system SHALL NOT modify any existing spec files or the constitution, only producing a read-only drift report

### Requirement: Constitution Update
The constitution SHALL be updated when the design phase introduces new technologies, patterns, or architectural changes that establish new project-wide conventions. During the design artifact generation, if the agent determines that the proposed design introduces a technology not listed in the Tech Stack, a new architectural pattern, or a convention that should apply globally, the agent SHALL update `.specshift/CONSTITUTION.md` to reflect the addition. Updates SHALL be additive by default -- existing entries SHALL NOT be removed without explicit user approval. When a design proposes replacing an existing technology, the agent SHALL ask the user directly whether to replace the entry, rather than leaving a `<!-- REVIEW -->` marker. The agent SHALL note constitution changes in the design document so they are visible during review.

**User Story:** As a developer I want the constitution to evolve as my project grows, with replacements confirmed through direct questions rather than invisible markers.

#### Scenario: New dependency added to constitution during design
- **GIVEN** a design document that introduces Redis as a caching layer
- **AND** the current constitution does not mention Redis
- **WHEN** the agent generates or updates the design artifact
- **THEN** the agent adds Redis to the Tech Stack section of `.specshift/CONSTITUTION.md` and notes this addition in the design document

#### Scenario: New architectural pattern captured
- **GIVEN** a design that introduces an event-driven messaging pattern between services
- **AND** the constitution's Architecture Rules do not mention event-driven patterns
- **WHEN** the agent processes the design
- **THEN** the agent adds the event-driven pattern to Architecture Rules with a description of where and how it applies

#### Scenario: Technology replacement resolved through direct question
- **GIVEN** a design that replaces Jest with Vitest for testing
- **WHEN** the agent updates the constitution
- **THEN** the agent asks the user: "The design replaces Jest with Vitest. Should I remove the Jest entry and add Vitest, or keep both?"
- **AND** the agent applies the user's decision without leaving a REVIEW marker

#### Scenario: Constitution changes documented in design
- **GIVEN** a design phase that triggers constitution updates
- **WHEN** the agent modifies `.specshift/CONSTITUTION.md`
- **THEN** the design document includes a section or note listing the specific constitution changes made
