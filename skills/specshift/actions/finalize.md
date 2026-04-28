# Requirements


### Requirement: Generate Changelog from Completed Changes
The `specshift finalize` command SHALL generate release notes from completed changes located in `.specshift/changes/`. The agent SHALL scan all change directories and identify completed changes by reading proposal frontmatter `status: completed` (falling back to tasks.md checkbox parsing if frontmatter is absent). For each completed change not yet in the changelog, the agent SHALL identify affected capabilities by reading the proposal's frontmatter `capabilities` field (falling back to parsing the Capabilities section if frontmatter is absent). The agent SHALL read `proposal.md` for motivation and the current specs at `docs/specs/<capability>.md` for user stories and scenario titles. The generated changelog SHALL follow the Keep a Changelog format with sections for Added, Changed, Deprecated, Removed, Fixed, and Security as applicable. Entries SHALL be ordered newest first. The changelog SHALL be written to `CHANGELOG.md` in the project root. If `CHANGELOG.md` already exists, the agent SHALL update it by adding new entries for changes not yet represented, preserving existing manually written entries.

**User Story:** As a user of the project I want a changelog that tells me what changed and when, so that I can understand the impact of updates without reading spec files or commit logs.

#### Scenario: Changelog generated from single completed change
- **GIVEN** a completed change at `.specshift/changes/2025-01-15-user-auth/` containing a proposal describing a new authentication feature
- **AND** the proposal lists capability `user-auth` as new
- **AND** `docs/specs/user-auth.md` contains user stories and scenarios
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent creates or updates `CHANGELOG.md` with an entry dated 2025-01-15 describing the new authentication feature using user stories from the spec

#### Scenario: Multiple completed changes ordered newest first
- **GIVEN** three completed changes dated 2025-01-10, 2025-02-05, and 2025-03-20
- **WHEN** the developer runs `specshift finalize`
- **THEN** the changelog lists the 2025-03-20 entry first, followed by 2025-02-05, then 2025-01-10

#### Scenario: Existing changelog preserved
- **GIVEN** a `CHANGELOG.md` that already contains manually written entries for versions 1.0 and 1.1
- **AND** a new completed change that has not been represented in the changelog
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent adds the new entry at the top of the changelog without modifying or removing the existing 1.0 and 1.1 entries

#### Scenario: No completed changes to process
- **GIVEN** no completed changes exist under `.specshift/changes/`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent informs the user that no completed changes were found and no changelog entries were generated

#### Scenario: Change with only internal refactoring
- **GIVEN** a completed change whose proposal and specs describe purely internal refactoring with no user-visible changes
- **WHEN** the agent processes the change
- **THEN** the agent either omits the entry entirely or includes it under a minimal note (e.g., "Internal improvements") rather than fabricating user-facing changes

### Requirement: Completion Workflow Next Steps

The post-apply workflow output SHALL include a "Next steps" section guiding the user through the complete post-completion workflow: generate changelog, generate docs, version bump (`src/VERSION`), compile, push, and update the local plugin. This is defined via the constitution convention.

#### Scenario: Next steps shown after verification

- **GIVEN** a successful verification of a completed change
- **WHEN** the verification summary is displayed
- **THEN** the output SHALL include next steps: `specshift finalize` → `src/VERSION` bump → compile → push → update plugin

### Requirement: Auto Patch Version Bump

The project constitution SHALL define a convention that instructs the post-apply workflow to automatically increment the patch version in `src/VERSION` after a successful change completion. `src/VERSION` is the single agnostic version source of truth — manifest and marketplace files at the repository root carry the version only as a stamped copy. The output SHALL display the new version. The subsequent compile run SHALL propagate the new version into the three root files (`.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json`).

**User Story:** As a plugin maintainer I want the patch version to auto-increment when a change is completed, so that consumers can detect updates without manual version bumps.

#### Scenario: Successful auto-bump after change completion

- **GIVEN** a plugin project with `src/VERSION` containing `1.0.3`
- **AND** the constitution defines the post-completion auto-bump convention
- **WHEN** the post-apply workflow runs for a completed change
- **THEN** the system SHALL update `src/VERSION` to `1.0.4`
- **AND** the subsequent compile run SHALL stamp `1.0.4` into all three root manifest/marketplace files
- **AND** the output SHALL display the new version

### Requirement: Version Sync Between Plugin Files

The `version` field in every per-target manifest and marketplace file at the repository root MUST equal the value in `src/VERSION`. The compile script SHALL enforce this by reading `src/VERSION` and stamping the value into `.claude-plugin/plugin.json` (`.version`), `.claude-plugin/marketplace.json` (`.plugins[].version`), and `.codex-plugin/plugin.json` (`.version`). After stamping, the script SHALL re-read each file and verify the stamped version equals the SoT; any mismatch SHALL fail the build with an error naming the offending file. The CI release workflow SHALL run the same cross-check before tag creation, ensuring that pushed manifests carry the version their `src/VERSION` declares (catches the foot-gun where a maintainer pushes a `src/VERSION` bump without recompiling). Hand-edits to a manifest's `version` field SHALL be considered transient — the next compile run overwrites them with the SoT value.

#### Scenario: All three root files in sync after compile

- **GIVEN** `src/VERSION` contains `1.0.3`
- **WHEN** the compile script runs
- **THEN** all three root manifest/marketplace files SHALL declare version `1.0.3`

#### Scenario: Manifest version drifts from SoT

- **GIVEN** `src/VERSION` contains `1.0.3` and `.codex-plugin/plugin.json` declares version `1.0.0`
- **WHEN** the compile script runs
- **THEN** the system SHALL stamp `.codex-plugin/plugin.json` to version `1.0.3`
- **AND** the post-stamp cross-check SHALL pass

#### Scenario: Stamping failure caught by cross-check

- **GIVEN** `src/VERSION` contains `1.0.3`
- **AND** the in-place jq stamp on one of the three files fails silently
- **WHEN** the cross-check step runs
- **THEN** the script SHALL detect the mismatch
- **AND** SHALL exit non-zero with an error naming the offending file

#### Scenario: CI release workflow catches missing recompile

- **GIVEN** the maintainer edits `src/VERSION` from `1.0.3` to `1.0.4` and pushes to `main` without running `bash scripts/compile-skills.sh` first
- **AND** the three root manifest/marketplace files therefore still declare `1.0.3`
- **WHEN** the GitHub Actions release workflow runs
- **THEN** the cross-check step SHALL fail naming each offending file with the actual vs expected version
- **AND** the tag creation SHALL be skipped

### Requirement: Generate Enriched Capability Documentation
The `specshift finalize` command SHALL generate user-facing documentation from the specs located in `docs/specs/<capability>.md`. The command SHALL produce one documentation file per capability, placed under `docs/capabilities/<capability>.md`. The agent SHALL read the capability doc template at `.specshift/templates/docs/capability.md` for the expected output format. Generated documentation SHALL use clear, user-facing language that explains what the capability does, how to use it, and what behavior to expect. Documentation SHALL NOT include implementation details, internal architecture references, or normative specification language (SHALL/MUST). The agent SHALL transform requirement descriptions into natural explanations, convert Gherkin scenarios into readable usage examples or behavioral descriptions, and organize content with appropriate headings. If a User Story is present in the spec, the agent SHALL use it to inform the documentation's framing and context. The `docs/capabilities/` directory SHALL be created if it does not exist.

Additionally, when generating a capability doc, the agent SHALL look up completed change directories at `.specshift/changes/*/` to find changes whose proposal frontmatter `capabilities` field lists the capability being documented (falling back to parsing the Capabilities section if frontmatter is absent). For each relevant change found, the agent SHALL read `proposal.md`, `research.md`, `design.md`, and `preflight.md` (where they exist) and enrich the capability doc with:
- A "Purpose" section (max 3 sentences) describing what the capability does and why it matters. ALWAYS derive from the spec's `## Purpose` section using problem-framing: frame as what goes wrong WITHOUT this capability, consider the User Stories ("so that...") for motivation. Change proposals provide context for Rationale, NOT for Purpose. The agent SHALL NOT use proposal "Why" sections as the Purpose — they describe change motivation, not capability purpose. The agent SHALL NOT restate the spec Purpose section verbatim.
- A "Rationale" section (max 3-5 sentences) summarizing design context: key decisions, alternatives explored, why this approach was chosen. For enriched capabilities: derived from `research.md` and `design.md`. For initial-spec-only: derived from spec requirements, scenarios, and assumptions — explain WHY the design works this way (e.g., why kebab-case naming, why date-prefix sorting, why certain constraints exist). Only omit Rationale if truly no design reasoning is derivable from the spec itself. The Rationale SHALL describe the current design in present tense. The agent SHALL NOT narrate change history (e.g., "was later added", "initially... then...", "a subsequent change introduced..."). Every Rationale sentence SHALL answer "why does it work this way?" not "how did it get this way?".
- A "Known Limitations" section (max 5 bullets) derived from `design.md` Non-Goals that describe current technical constraints (rewritten as "Does not support X"), `design.md` Risks (user-relevant risks), and `preflight.md` Assumption Audit (assumptions rated "Acceptable Risk" that affect users). Omit this section entirely if empty.
- A "Future Enhancements" section (max 5 bullets) derived from `design.md` Non-Goals that are explicitly deferred or represent natural extensions of the capability. Items marked "(deferred)" or "(separate feature)" SHALL be included. Sensible out-of-scope items that would genuinely improve the capability MAY be included. Change-level scope boundaries (e.g., "No changes to other skills") SHALL NOT be included. Link to GitHub Issues where they exist. Omit this section entirely if no actionable future ideas.

**Section completeness:** The agent SHALL include ALL sections from the capability doc template when source data exists for that section. The agent SHALL only omit a section when no source data is available (e.g., no Non-Goals in any change → omit Known Limitations; no deferred items → omit Future Enhancements). Per-section maximum limits (Purpose max 3 sentences, Known Limitations max 5 bullets, etc.) are sufficient conciseness guards. The agent SHALL NOT treat any enriched section as optional when source data exists.

The enriched sections SHALL appear in this order: Overview, Purpose, Rationale, Features, Behavior, Known Limitations, Future Enhancements, Edge Cases.

**Behavior depth:** Each distinct Gherkin scenario group in the spec SHALL produce at least one Behavior subsection. The agent SHALL NOT merge multiple distinct scenarios into fewer subsections than the spec defines.

For capabilities that involve multiple workflow phases (e.g., quality-gates covers preflight during `specshift propose` and audit.md during `specshift apply`), the agent SHALL add a brief workflow sequence note at the top of the Behavior section explaining when each phase is used relative to the overall workflow. For multi-phase capabilities, the agent SHALL include the action name in behavior subsection headers for quick scanning (e.g., `### Step-by-Step Generation (specshift propose)` rather than just `### Step-by-Step Generation`).

The Edge Cases section SHALL only include surprising states, error conditions, or non-obvious interactions. Normal flow variants and expected UX behaviors SHALL be placed in the Behavior section instead. A good test: if a user would not be surprised by the behavior, it belongs in Behavior, not Edge Cases.

If no relevant completed changes exist for a capability, the agent SHALL fall back to generating a spec-only doc (no enriched sections except Rationale where initial-spec research data is available).

**Step independence:** Each documentation generation step SHALL read its own source materials independently. The agent SHALL NOT assume that data loaded during an earlier step (e.g., change enrichment in Step 2) is still available in later steps — steps may execute in separate contexts.

**Incremental generation:** The agent SHALL only regenerate capability docs that need updating, as determined by the "Incremental Capability Documentation Generation" requirement. Capabilities with no newer changes SHALL be skipped entirely — no change reading, no generation, no file writes. The "read before write" guardrail still applies for capabilities that ARE regenerated.

**User Story:** As a user of the plugin I want clear documentation for each capability that explains not just what it does but why it exists and what its limitations are, so that I can make informed decisions about using it.

#### Scenario: Documentation generated for single capability
- **GIVEN** a spec exists at `docs/specs/user-auth.md` with two requirements and four scenarios
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent creates `docs/capabilities/user-auth.md` containing a description of the user-auth capability, explanations of each feature, and usage examples derived from the scenarios

#### Scenario: Normative language replaced with user-facing language
- **GIVEN** a spec containing "The system SHALL validate email format before account creation"
- **WHEN** the agent generates documentation for this capability
- **THEN** the documentation reads something like "Email addresses are validated when you create an account" — natural language without SHALL/MUST

#### Scenario: Gherkin scenarios converted to usage examples
- **GIVEN** a spec scenario with GIVEN a user on the dashboard, WHEN they click Export, THEN a CSV downloads
- **WHEN** the agent generates documentation
- **THEN** the documentation includes a section describing the export workflow in natural terms

#### Scenario: Implementation details excluded
- **GIVEN** a spec requirement that mentions internal module names or database schema details
- **WHEN** the agent generates documentation
- **THEN** the documentation omits internal references and focuses only on user-visible behavior

#### Scenario: Enriched doc with change data
- **GIVEN** a spec exists at `docs/specs/release-workflow.md`
- **AND** a completed change at `.specshift/changes/2026-03-04-release-workflow/` has a proposal listing `release-workflow` in its Capabilities section
- **AND** the change contains `design.md` with Non-Goals and `preflight.md` with an Assumption Audit
- **WHEN** the developer runs `specshift finalize`
- **THEN** the generated `docs/capabilities/release-workflow.md` includes enriched sections derived from the change's artifacts

#### Scenario: Capability with no change data falls back to spec-only
- **GIVEN** a spec exists at `docs/specs/new-feature.md`
- **AND** no completed change directory contains a proposal listing `new-feature`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the generated doc contains Overview, Features, Behavior, and Edge Cases sections only — no enriched sections

#### Scenario: Capability Purpose uses problem-framing
- **GIVEN** a capability being documented
- **WHEN** the agent generates the "Purpose" section
- **THEN** the agent derives it from the spec's `## Purpose` section using problem-framing (what goes wrong without this capability), not from change proposal "Why" sections and not by restating the Purpose verbatim

#### Scenario: Initial-spec-only capability gets Rationale from spec
- **GIVEN** a capability whose only change is `initial-spec`
- **AND** the spec contains requirements with design reasoning (e.g., kebab-case naming conventions, date-prefix sorting, specific constraints)
- **WHEN** the agent generates documentation
- **THEN** the doc includes a "Rationale" section explaining why the design works this way, derived from spec requirements and scenarios

#### Scenario: Rationale uses present tense without change history
- **GIVEN** a capability with change data spanning multiple changes
- **WHEN** the agent generates the "Rationale" section
- **THEN** the Rationale describes the current design in present tense (e.g., "X handles the common case. Y covers edge cases where Z occurs.") and does NOT narrate change history (e.g., "The initial design used X. A later change added Y.")

#### Scenario: Multi-command capability includes workflow sequence note
- **GIVEN** a capability that spans multiple workflow phases (e.g., quality-gates covers preflight during `specshift propose` and audit.md during `specshift apply`)
- **WHEN** the agent generates the Behavior section
- **THEN** the top of the Behavior section includes a brief workflow sequence note explaining when each command is used

#### Scenario: Multi-command behavior headers include command names
- **GIVEN** a capability that spans multiple commands (e.g., artifact-pipeline covers `specshift propose`)
- **WHEN** the agent generates behavior subsection headers
- **THEN** each subsection header includes the command name for quick scanning (e.g., `### Step-by-Step Generation (specshift propose)`)

#### Scenario: Edge cases contain only surprising behavior
- **GIVEN** a spec with edge cases including "If multiple active changes exist, the system lists them and asks you to select one"
- **WHEN** the agent generates the Edge Cases section
- **THEN** that item is placed in the Behavior section instead, because prompting for selection is normal UX, not a surprising state

#### Scenario: Conciseness guards enforced
- **GIVEN** a capability with extensive change data across multiple changes
- **WHEN** the agent generates the enriched doc
- **THEN** "Purpose" contains at most 3 sentences, "Rationale" at most 3-5 sentences, and "Known Limitations" at most 5 bullets

#### Scenario: Agent reads capability template
- **GIVEN** a doc template exists at `.specshift/templates/docs/capability.md`
- **WHEN** the agent generates a capability doc
- **THEN** the agent reads the template and uses it as the structural format for the output

#### Scenario: All sections included when source data exists
- **GIVEN** a capability with completed changes containing `design.md` with Non-Goals items
- **WHEN** the agent generates the enriched doc from scratch (no existing doc)
- **THEN** the doc includes a "Known Limitations" section derived from the Non-Goals
- **AND** the agent does NOT omit the section based on space or priority considerations

#### Scenario: Behavior subsections match scenario groups
- **GIVEN** a spec with 5 distinct Gherkin scenario groups
- **WHEN** the agent generates the Behavior section
- **THEN** the Behavior section contains at least 5 subsections — one per scenario group
- **AND** the agent does NOT merge distinct scenarios into fewer subsections

### Requirement: Incremental Capability Documentation Generation
The `specshift finalize` command SHALL support incremental generation by default. Before generating each capability doc, the agent SHALL determine whether regeneration is needed by comparing change dates against the doc's `lastUpdated` frontmatter field.

**Change detection logic:** For each capability, the agent SHALL:
1. Read the existing `docs/capabilities/<capability>.md` and extract the `lastUpdated` value from YAML frontmatter. If the file does not exist, the capability needs generation.
2. Read the spec's `lastModified` frontmatter field from `docs/specs/<capability>.md`. If `lastModified` is absent, the capability needs regeneration (treated as unknown modification date).
3. If `lastModified` is newer than (or equal to) the doc's `lastUpdated`, the capability needs regeneration.
4. If `lastModified` is older than the doc's `lastUpdated`, skip this capability entirely.

**First run:** When no capability docs exist yet, all capabilities SHALL be generated (equivalent to full generation).

**Single-capability mode:** When the user provides a single capability name argument (e.g., `specshift finalize auth`), the agent SHALL only read completed changes relevant to that capability. The specified capability SHALL always be regenerated regardless of dates.

**Multi-capability mode:** When the user provides multiple capability names as a comma-separated list (e.g., `specshift finalize artifact-pipeline,documentation`), the agent SHALL process only the listed capabilities. Each listed capability SHALL always be regenerated regardless of change dates (same as single-capability mode). The agent SHALL NOT perform the full change date scan for unlisted capabilities. Changes SHALL only be read for the listed capabilities. If a listed capability does not exist in `docs/specs/`, the agent SHALL warn and skip it. This mode is designed for the post-apply workflow where the caller already knows which capabilities were affected.

**Content-aware writes:** After generating a capability doc, the agent SHALL compare the generated content against the existing file content, excluding the `lastUpdated` frontmatter field. If the content is identical, the agent SHALL NOT write the file and SHALL NOT bump the `lastUpdated` timestamp. This prevents false timestamp updates when regeneration produces unchanged output.

**Output summary:** The agent SHALL report which capabilities were regenerated, which were skipped (no newer changes), and which had unchanged content (regenerated but not written).

**User Story:** As a user I want `specshift finalize` to only regenerate documentation for capabilities that have new change data, so that unchanged docs are not touched and timestamps remain accurate.

#### Scenario: Capability skipped when spec lastModified is older
- **GIVEN** `docs/capabilities/release-workflow.md` exists with `lastUpdated: "2026-03-20"`
- **AND** `docs/specs/release-workflow.md` has `lastModified: "2026-03-04"`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent skips `release-workflow` and does not read its changes or regenerate its doc

#### Scenario: Capability regenerated when spec lastModified is newer
- **GIVEN** `docs/capabilities/documentation.md` exists with `lastUpdated: "2026-03-05"`
- **AND** `docs/specs/documentation.md` has `lastModified: "2026-03-23"`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent regenerates `docs/capabilities/documentation.md`

#### Scenario: Spec without lastModified always regenerated
- **GIVEN** `docs/capabilities/legacy-cap.md` exists with `lastUpdated: "2026-03-05"`
- **AND** `docs/specs/legacy-cap.md` has no `lastModified` field
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent treats the spec as needing regeneration (unknown modification date)

#### Scenario: First run generates all capabilities
- **GIVEN** no capability doc files exist under `docs/capabilities/`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent generates documentation for all capabilities

#### Scenario: Content-aware write prevents false timestamp bump
- **GIVEN** `docs/capabilities/change-workspace.md` exists with `lastUpdated: "2026-03-20"`
- **AND** a newer change touches `change-workspace` but only changes assumption comments in the spec
- **WHEN** the agent regenerates the capability doc
- **AND** the generated content (excluding `lastUpdated`) is identical to the existing file
- **THEN** the agent does NOT write the file and `lastUpdated` remains `"2026-03-20"`

#### Scenario: Single-capability mode scopes change reading
- **GIVEN** the developer runs `specshift finalize release-workflow`
- **WHEN** the agent looks up change enrichment
- **THEN** the agent only reads completed changes whose proposal lists `release-workflow`
- **AND** the agent does NOT read changes for other capabilities

#### Scenario: Multi-capability mode processes only listed capabilities
- **GIVEN** the developer runs `specshift finalize artifact-pipeline,documentation`
- **WHEN** the agent performs change detection
- **THEN** the agent processes only `artifact-pipeline` and `documentation`
- **AND** both capabilities are always regenerated regardless of change dates
- **AND** the agent does NOT scan changes for the other capabilities

#### Scenario: Multi-capability mode with nonexistent capability
- **GIVEN** the developer runs `specshift finalize artifact-pipeline,nonexistent-cap`
- **WHEN** the agent processes the list
- **THEN** the agent regenerates `artifact-pipeline`
- **AND** the agent warns that `nonexistent-cap` does not exist in `docs/specs/` and skips it

#### Scenario: Output summary shows skipped and unchanged capabilities
- **GIVEN** 18 capabilities exist, 2 have newer completed changes, and 1 of those produces identical content
- **WHEN** the developer runs `specshift finalize`
- **THEN** the output shows 1 capability regenerated, 1 skipped (unchanged content), and 16 skipped (no newer changes)

### Requirement: Generate Architecture Overview
The `specshift finalize` command SHALL generate a cross-cutting architecture overview as part of the consolidated `docs/README.md` file. The overview content SHALL be synthesized from the project constitution (`.specshift/CONSTITUTION.md`), the three-layer-architecture spec (`docs/specs/three-layer-architecture.md`), and all generated ADR files at `docs/decisions/adr-*.md`. The architecture overview SHALL include: a "System Architecture" section describing the three-layer model, a "Tech Stack" section from the constitution, a "Key Design Decisions" section sourced from ADR files, and a "Conventions" section from the constitution.

**Key Design Decisions table — ADR-sourced:** The table SHALL be built by reading all ADR files in `docs/decisions/` (both generated `adr-NNN-*.md` and manual `adr-M*.md`). For each ADR, the agent SHALL extract:
- **Decision**: A summary derived from the `## Decision` section content. For consolidated ADRs with numbered sub-decisions, summarize the overarching decision. For single-decision ADRs, use the inline decision text.
- **Rationale**: For generated ADRs, extract the inline rationale from the Decision section (the text after the em-dash `—`). For manual ADRs, extract from the `## Rationale` section if present.
- **ADR link**: Link directly to the ADR file (e.g., `[ADR-001](decisions/adr-001-slug.md)`).

The table SHALL list generated ADRs first (ordered by number), followed by manual ADRs (ordered by M-number). The agent SHALL NOT read `design.md` from change directories for the Key Design Decisions table — ADR files are the single canonical source.

The overview SHALL surface notable trade-offs from ADR Consequences sections — if a decision has a significant negative consequence, it SHALL be mentioned as a brief note in the table or in a "Notable Trade-offs" subsection below the table. The Notable Trade-offs subsection SHALL include trade-offs that affect documentation consumers or represent meaningful constraints. Every ADR with a substantive negative consequence SHOULD be represented. The document SHALL be written in user-facing language. The agent SHALL read the README template at `.specshift/templates/docs/readme.md` for the expected output format.

**Conditional regeneration:** The `docs/README.md` SHALL be regenerated only when at least one of the following conditions is met during the current `specshift finalize` run:
1. Any capability doc was created or updated (written to disk) in this run.
2. Any ADR was created in this run.
3. No `docs/README.md` exists yet (first run).
4. The content of `.specshift/CONSTITUTION.md` (Tech Stack, Architecture Rules, Conventions sections) has diverged from the corresponding sections in the existing `docs/README.md`. The agent SHALL read the constitution and compare its key content against the README to detect drift.

If none of these conditions are met, the agent SHALL skip README regeneration and report "README is up-to-date — no capability or ADR changes detected."

The agent SHALL NOT generate `docs/architecture-overview.md` as a separate file. If this file exists from a previous run, the agent SHALL delete it.

**User Story:** As a developer or contributor I want a single document that explains the system architecture and key decisions with direct links to ADRs and visible trade-offs, so that I can understand the project structure and the reasoning behind it without navigating multiple files.

#### Scenario: Architecture overview generated as part of consolidated README
- **GIVEN** a constitution at `.specshift/CONSTITUTION.md` with Tech Stack and Architecture Rules sections
- **AND** ADR files exist at `docs/decisions/adr-*.md`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent writes architecture overview content (System Architecture, Tech Stack, Key Design Decisions, Conventions) into `docs/README.md`

#### Scenario: Architecture overview with no ADR files
- **GIVEN** a constitution exists but no ADR files exist in `docs/decisions/`
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent generates architecture content in `docs/README.md` with System Architecture, Tech Stack, and Conventions sections, omitting Key Design Decisions

#### Scenario: Design decisions table sourced from ADR files
- **GIVEN** ADR files exist at `docs/decisions/adr-001-*.md` through `adr-015-*.md`
- **WHEN** the agent generates the Key Design Decisions table
- **THEN** each row's Decision and Rationale are extracted from the ADR file content (not from design.md from completed changes)
- **AND** each row includes an ADR link column pointing to the corresponding ADR file

#### Scenario: Manual ADRs included in design decisions table
- **GIVEN** `docs/decisions/adr-M001-init-model-invocable.md` exists with `## Decision` and `## Rationale` sections
- **AND** generated ADRs exist at `docs/decisions/adr-001-*.md` through `adr-033-*.md`
- **WHEN** the agent generates the Key Design Decisions table
- **THEN** ADR-M001 appears as a row after all generated ADRs, with the decision text, rationale, and a link to the file

#### Scenario: Trade-offs surfaced comprehensively from ADR consequences
- **GIVEN** ADR Consequences sections contain significant negative consequences across multiple ADRs
- **WHEN** the agent generates the Key Design Decisions section
- **THEN** the Notable Trade-offs subsection includes trade-offs that affect documentation consumers or represent meaningful constraints — every ADR with a substantive negative consequence is represented

#### Scenario: Stale architecture-overview.md deleted
- **GIVEN** `docs/architecture-overview.md` exists from a previous run
- **WHEN** the agent runs `specshift finalize`
- **THEN** the agent deletes `docs/architecture-overview.md`

#### Scenario: README skipped when no changes detected
- **GIVEN** no capability docs were created or updated in this run
- **AND** no ADRs were created in this run
- **AND** `docs/README.md` already exists
- **WHEN** the agent reaches the README generation step
- **THEN** the agent skips README regeneration and reports "README is up-to-date"

#### Scenario: README regenerated when new capability doc written
- **GIVEN** one capability doc was updated in this run
- **AND** `docs/README.md` already exists
- **WHEN** the agent reaches the README generation step
- **THEN** the agent regenerates `docs/README.md` to reflect the updated capability

#### Scenario: README regenerated on first run
- **GIVEN** no `docs/README.md` exists
- **WHEN** the developer runs `specshift finalize`
- **THEN** the agent generates `docs/README.md`

#### Scenario: README regenerated when constitution content drifted
- **GIVEN** no capability docs or ADRs were changed in this run
- **AND** `docs/README.md` already exists
- **AND** `.specshift/CONSTITUTION.md` (Tech Stack, Architecture Rules, or Conventions) has been updated since the last README generation
- **WHEN** the agent reaches the README generation step
- **THEN** the agent regenerates `docs/README.md` to reflect the updated constitution content

### Requirement: Generate Documentation Table of Contents
The `specshift finalize` command SHALL create or update `docs/README.md` as the single entry point for all generated documentation. The README SHALL include the architecture overview content (System Architecture, Tech Stack, Key Design Decisions with ADR links, Conventions) followed by a capabilities section. The capabilities section SHALL be grouped by the `category` frontmatter field from specs, rendered as category group headers (title-case). Within each category group, capabilities SHALL be ordered by the `order` frontmatter field (lower first). The Key Design Decisions table SHALL include an "ADR" column linking directly to the corresponding ADR file instead of a "Source" column. The README SHALL surface notable trade-offs from ADR Consequences for the most significant decisions. The `docs/README.md` SHALL be regenerated when capability docs or ADRs change, as specified by the conditional regeneration logic in the "Generate Architecture Overview" requirement. The agent SHALL read the README template at `.specshift/templates/docs/readme.md` for the expected output format. Capability descriptions in the capabilities table SHALL be concise: max 80 characters or 15 words. Each description SHALL be one short phrase, not a multi-clause sentence.

The agent SHALL NOT generate a separate `docs/architecture-overview.md` file. The agent SHALL NOT generate a separate `docs/decisions/README.md` file. If either file exists from a previous run, the agent SHALL delete it.

**User Story:** As a user I want a single documentation entry point that gives me the architecture overview, lists all capabilities grouped by workflow phase, and links to ADRs inline, so that I don't have to navigate between multiple index files.

#### Scenario: Consolidated README generated
- **GIVEN** the agent has generated capability docs and ADR files
- **WHEN** the agent updates `docs/README.md`
- **THEN** the README contains System Architecture, Tech Stack, Key Design Decisions (with ADR links), Conventions, and a category-grouped capabilities section — all in one file

#### Scenario: Stale files cleaned up
- **GIVEN** `docs/architecture-overview.md` and `docs/decisions/README.md` exist from a previous run
- **WHEN** the agent generates `docs/README.md`
- **THEN** the agent deletes `docs/architecture-overview.md` and `docs/decisions/README.md`

#### Scenario: Capabilities grouped by category
- **GIVEN** specs with `category` frontmatter values like `setup`, `change-workflow`, `development`
- **WHEN** the agent generates the capabilities section
- **THEN** capabilities appear under category group headers (e.g., "### Setup", "### Change Workflow"), ordered by `order` within each group

#### Scenario: Capabilities without category appear in Other group
- **GIVEN** a spec with no `category` frontmatter
- **WHEN** the agent generates the capabilities section
- **THEN** the capability appears under an "Other" group header

#### Scenario: Capability descriptions are concise
- **GIVEN** specs with varying description lengths
- **WHEN** the agent generates the capabilities table
- **THEN** each description is at most 80 characters or 15 words — one short phrase, not a multi-clause sentence

#### Scenario: Design decisions table includes ADR links
- **GIVEN** ADR files have been generated at `docs/decisions/adr-NNN-slug.md`
- **WHEN** the agent generates the Key Design Decisions table
- **THEN** each row includes an ADR link column pointing to the corresponding ADR file

#### Scenario: Notable trade-offs surfaced
- **GIVEN** ADR Consequences sections contain significant negative consequences
- **WHEN** the agent generates the Key Design Decisions section
- **THEN** notable trade-offs are surfaced as brief notes in the table or a subsection

### Requirement: ADR Generation from Change Decisions

The `specshift finalize` command SHALL generate formal Architecture Decision Records (ADRs) from `## Decisions` tables in completed changes' `design.md` files.

Each ADR SHALL include:
- **Status**: "Accepted" with the change date
- **Context**: From the `design.md` Context section, enriched with `research.md` Approaches and findings from the same change where available. The Context section SHALL be at least 4-6 sentences. It SHALL include what motivated the decision (the problem being solved), what was investigated or researched, and key constraints or trade-offs that shaped the decision. The agent SHALL NOT write thin Context sections like "we chose X over Y because Z".
- **Decision**: From the Decisions table "Decision" column. Each decision item SHALL include its rationale inline using the em-dash pattern: `**Decision text** — rationale explaining why`. For consolidated ADRs, this is a numbered list of sub-decisions each with inline rationale. For single-decision ADRs, this is a single statement with inline rationale. The Rationale column from the Decisions table SHALL be used as the inline rationale text. There is no separate Rationale section.
- **Alternatives Considered**: From the Decisions table "Alternatives" column, expanded into bullets
- **Consequences**: Split into two subsections:
  - **Positive**: Benefits of this decision, derived from the rationale, context, and positive outcomes
  - **Negative**: Drawbacks, risks, or trade-offs, derived from the `design.md` "Risks & Trade-offs" section filtered to relevance for this specific decision where possible
- **References**: Internal relative links only — no external URLs (GitHub issues, external docs). The first reference SHALL be the source change backlink (see "ADR Change Backlinks" requirement). The agent SHALL use descriptive text like `[Spec: three-layer-architecture](path)`, NOT raw file paths as link text like `[../../docs/specs/three-layer-architecture.md](path)`. Include the relevant spec file and related ADRs if the decision connects to other decisions. The change backlink provides traceability to GitHub issues via the change's proposal.md.

**References determination:** The agent SHALL determine which specs are relevant to each decision by checking the change's `proposal.md` Capabilities section to find which capabilities were affected. The agent SHALL link to those specs using semantic link text: `[Spec: capability-name](../../docs/specs/capability.md)`. The agent SHALL cross-reference other ADRs from the same change when decisions are related.

**Reference validation with auto-resolution:** After generating the References section for each ADR, the agent SHALL validate all links and actively resolve broken references:
1. **Spec links**: For every `[Spec: <name>]` link, glob `docs/specs/<name>.md` to verify the spec exists. If a linked spec does not exist (e.g., it was renamed or split), the agent SHALL attempt to find the successor spec(s) by searching for the capability name in existing specs. If found, replace the broken link with the successor(s). If the successor cannot be determined, the agent SHALL ask the user to identify the correct spec and update the link accordingly. No `<!-- REVIEW -->` markers SHALL be left in the final output.
2. **Change links**: For every `[Change: <name>]` link, verify the change directory exists under `.specshift/changes/`. If the change does not exist, the agent SHALL ask the user whether to remove the link or provide the correct change name. No `<!-- REVIEW -->` markers SHALL be left in the final output.

**Cross-reference heuristic for related ADRs:** Beyond cross-referencing ADRs from the same change, the agent SHALL check whether the current ADR modifies, extends, or supersedes a system established by an earlier ADR. Specifically:
1. If the current change's `proposal.md` or `design.md` references another change by name (e.g., "supersedes the full regeneration from doc-ecosystem"), the agent SHALL link to the ADR from that referenced change.
2. If the current change modifies the same capabilities as an earlier change's ADR (determined by overlapping capabilities in proposal.md Capabilities sections), the agent SHOULD add a cross-reference to the most relevant earlier ADR.
3. The agent SHALL NOT add cross-references speculatively — only when a clear thematic relationship is evident from the change content.

The slug SHALL be derived from the Decision column text using this deterministic algorithm:
1. Lowercase the entire string
2. Replace any character that is NOT in `[a-z0-9]` with a hyphen
3. Collapse consecutive hyphens into a single hyphen
4. Trim leading and trailing hyphens
5. Truncate to 50 characters
6. Trim trailing hyphens again (in case truncation split a word)

For consolidated ADRs, the slug SHALL be derived from the overarching decision title (e.g., "Rename init skill to setup" → `rename-init-skill-to-setup`), not from individual sub-decision texts.

**Step independence:** ADR generation SHALL read its own source materials independently. The agent SHALL NOT assume that data loaded during an earlier step (e.g., change enrichment in Step 2) is still available — steps may execute in separate contexts. This is especially critical for ADR generation, which needs full change data (design.md, research.md, proposal.md) independently of capability doc generation.

**User Story:** As a developer or contributor I want formal decision records with fully resolved references, so that I never encounter invisible REVIEW markers or broken links in generated ADRs.

#### Scenario: Spec link validated and auto-resolved
- **GIVEN** an ADR references `[Spec: docs-generation](../../docs/specs/docs-generation.md)`
- **AND** `docs/specs/docs-generation.md` does not exist
- **WHEN** the agent validates the References section
- **THEN** the agent searches existing specs for successors and replaces the broken link with the correct successor specs (e.g., `documentation`)

#### Scenario: Change link validated and resolved via user question
- **GIVEN** an ADR references `[Change: old-feature](../../.specshift/changes/2026-01-01-old-feature/)`
- **AND** the change directory does not exist
- **WHEN** the agent validates the References section
- **THEN** the agent asks the user: "Change 'old-feature' not found. Should I remove this reference or provide the correct change name?"
- **AND** the agent applies the user's decision with no `<!-- REVIEW -->` marker left in the output

#### Scenario: No REVIEW markers in generated ADRs
- **GIVEN** a `specshift finalize` run that generates multiple ADRs
- **WHEN** all ADRs are written to disk
- **THEN** zero `<!-- REVIEW -->` or `<!-- REVIEW: ... -->` markers exist in any generated ADR file

### Requirement: Post-Merge Worktree Cleanup

When a PR is merged from within a worktree (via any merge method), the system SHALL perform immediate cleanup of the completed worktree. The cleanup sequence SHALL be: (1) switch working directory to the main worktree, (2) remove the completed worktree, (3) delete the local branch, (4) delete the remote branch. The system SHALL detect that it is inside a worktree by checking `git rev-parse --git-dir` for a path containing `/worktrees/`. This immediate cleanup complements lazy cleanup at `specshift propose` — lazy cleanup catches worktrees from merges that happened outside the agent session, while immediate cleanup handles in-session merges.

**User Story:** As a developer I want the worktree cleaned up immediately after my PR is merged, so that I don't have stale worktrees lingering until the next `specshift propose`.

#### Scenario: Cleanup after successful local merge

- **GIVEN** the agent is working inside a worktree at `.specshift/worktrees/fix-auth` on branch `fix-auth`
- **AND** the agent merges the PR which succeeds
- **WHEN** the merge completes
- **THEN** the system SHALL switch the working directory to the main worktree
- **AND** remove the worktree
- **AND** delete the local branch
- **AND** delete the remote branch
- **AND** report "Cleaned up worktree: fix-auth (merged)"

#### Scenario: Cleanup skipped when not in worktree

- **GIVEN** the agent is working in the main working tree (not a worktree)
- **WHEN** a merge completes
- **THEN** the system SHALL NOT attempt worktree cleanup

#### Scenario: Worktree removal fails due to dirty state

- **GIVEN** the agent is inside a worktree and a merge completes
- **AND** the worktree has uncommitted changes
- **WHEN** `git worktree remove` fails
- **THEN** the system SHALL report the failure and suggest manual cleanup
- **AND** SHALL NOT block the workflow
