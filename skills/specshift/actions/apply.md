# Requirements


### Requirement: Implement Tasks from Task List

The system SHALL work through pending task checkboxes in the change's `tasks.md` file when the user invokes `specshift apply`. For each task, the system SHALL read the task description, make the required code changes, and mark the task as complete by changing `- [ ]` to `- [x]` in the tasks file. The system SHALL read all context files (proposal, design, tasks) from the change directory and specs from `docs/specs/` for the capabilities listed in the proposal before beginning implementation. The system SHALL read the `apply.instruction` field from WORKFLOW.md for apply guidance. The system SHALL pause and request clarification if a task is ambiguous, if implementation reveals a design issue, or if a blocker is encountered. The system SHALL NOT guess when requirements are unclear.

The QA Loop's Metric Check and Auto-Verify steps are **automated steps** — the system SHALL execute them without pausing for user confirmation. The first human gate in the QA Loop is User Testing. The system SHALL NOT pause or ask for permission before generating `audit.md`; it SHALL generate the audit artifact automatically after the metric check passes using the audit template.

**User Story:** As a developer I want the AI to systematically work through my task list and implement each item, so that I can focus on review and guidance rather than manual coding of each task.

#### Scenario: Implement all tasks sequentially

- **GIVEN** a change named "add-user-auth" with a tasks.md containing 5 pending tasks
- **AND** all pipeline artifacts (proposal, specs, design) are available as context in the change directory
- **WHEN** the user invokes `specshift apply add-user-auth`
- **THEN** the system reads all context files from the change directory and the apply instruction from WORKFLOW.md
- **AND** works through each pending task in order
- **AND** makes the code changes described by each task
- **AND** marks each task `- [x]` immediately after completing it
- **AND** continues to the next task until all are done or a blocker is hit

#### Scenario: Resume partially completed task list

- **GIVEN** a change with tasks.md containing 3 completed and 4 pending tasks
- **WHEN** the user invokes `specshift apply`
- **THEN** the system skips the 3 already-completed tasks
- **AND** begins working from the first pending task (task 4 of 7)
- **AND** displays "3/7 tasks already complete, continuing from task 4"

#### Scenario: QA automated steps run without pausing

- **GIVEN** a change with all implementation tasks complete
- **AND** the system reaches the QA Loop Metric Check
- **WHEN** the metric check passes
- **THEN** the system SHALL immediately proceed to Auto-Verify without pausing
- **AND** SHALL generate `audit.md` in the change directory automatically using the audit template
- **AND** SHALL only pause at User Testing to wait for human approval

#### Scenario: Pause on ambiguous task

- **GIVEN** a task description that is unclear or could be interpreted multiple ways
- **WHEN** the system encounters this task during apply
- **THEN** the system SHALL pause implementation
- **AND** SHALL present the ambiguity to the user with suggested interpretations
- **AND** SHALL wait for user guidance before proceeding

### Requirement: Progress Tracking

The system SHALL report implementation progress using checkbox counts from the tasks file. Progress SHALL be displayed as "N/M tasks complete" where N is the count of `- [x]` items and M is the total count of checkbox items. The system SHALL show progress at the start of an apply session, after each task completion, and in the final summary. When all tasks are complete, the system SHALL display a completion message with the total count.

**User Story:** As a developer I want to see clear progress counts as tasks are completed, so that I can track how much work remains and estimate time to completion.

#### Scenario: Display progress at session start

- **GIVEN** a change with tasks.md containing 2 completed and 5 pending tasks
- **WHEN** the user invokes `specshift apply`
- **THEN** the system displays "Progress: 2/7 tasks complete" before starting implementation

#### Scenario: Update progress after each task

- **GIVEN** the system has just completed task 3 of 7
- **WHEN** the checkbox is marked complete in tasks.md
- **THEN** the system displays "Progress: 3/7 tasks complete"
- **AND** announces which task it will work on next

#### Scenario: Display final summary on completion

- **GIVEN** the system has completed the last pending task (task 7 of 7)
- **WHEN** the final checkbox is marked complete
- **THEN** the system displays "Progress: 7/7 tasks complete"
- **AND** lists all tasks completed during the current session
- **AND** suggests running the post-apply workflow

#### Scenario: Display progress on pause

- **GIVEN** the system pauses at task 4 of 7 due to a blocker
- **WHEN** the pause message is displayed
- **THEN** the system includes "Progress: 3/7 tasks complete"
- **AND** shows which task caused the pause
- **AND** presents options for how to proceed

#### Scenario: Handle parallelizable task markers

- **GIVEN** tasks.md contains tasks marked with `[P]` indicating they can be done in parallel
- **WHEN** the system displays progress
- **THEN** the progress count SHALL still be based on individual checkbox completion
- **AND** the `[P]` marker SHALL be informational only (no change to counting logic)

### Requirement: Standard Tasks Exclusion from Apply Scope

The system SHALL distinguish between implementation tasks (Foundation, Implementation, QA Loop sections) and standard tasks (Post-Implementation section) in the generated `tasks.md`. During `specshift apply`, the system SHALL only process tasks in the implementation and QA sections. Standard tasks in the Post-Implementation section SHALL NOT be executed by the apply phase. Standard tasks SHALL remain as unchecked `- [ ]` items after apply completes. The standard tasks section SHALL be included in the total checkbox count for progress reporting, reflecting the full workflow completion state. During the post-apply workflow, the system SHALL mark all standard task checkboxes as complete in `tasks.md` — including universal steps and constitution-defined pre-merge extras — before creating the final commit, so that the committed file reflects the fully-checked state. Constitution-defined post-merge items SHALL appear in a separate "Post-Merge Reminders" section using plain bullet format (`- ` without checkbox). Plain bullets are not counted in progress totals and are not parsed by the apply skill. Post-merge items MAY include a scope hint describing when the reminder is relevant (e.g., "applies when src/ files change"). During task generation, the agent SHALL evaluate each post-merge item's relevance against the proposal's "What Changes" and "Scope & Boundaries" sections, including only items whose scope matches the change. Items without a scope hint SHALL always be included. If no post-merge items exist in the constitution, or no items are relevant to the change scope, the Post-Merge Reminders section SHALL be omitted.

**User Story:** As a developer I want post-implementation workflow steps tracked as checkboxes in my task list but not executed by apply, so that I have a visible, auditable checklist for post-apply steps without conflating them with implementation work.

#### Scenario: Apply skips standard tasks section

- **GIVEN** a tasks.md with Foundation, Implementation, and QA Loop sections, a Standard Tasks (Post-Implementation) section containing 3 unchecked items, and a Post-Merge Reminders section containing plain bullet reminders
- **WHEN** the user invokes `specshift apply`
- **THEN** the system SHALL work through pending tasks in Foundation, Implementation, and QA Loop sections only
- **AND** SHALL NOT attempt to execute Standard Tasks or Post-Merge Reminders
- **AND** Standard Tasks items SHALL remain as `- [ ]` after apply completes
- **AND** Post-Merge Reminders plain bullets SHALL remain unchanged

#### Scenario: Progress count includes standard tasks

- **GIVEN** a tasks.md with 5 implementation tasks (all complete) and 3 standard tasks (all unchecked)
- **WHEN** the system reports progress
- **THEN** the system SHALL display "5/8 tasks complete"
- **AND** SHALL indicate that standard tasks remain for post-apply workflow

#### Scenario: All standard tasks marked before commit

- **GIVEN** a tasks.md with all implementation tasks complete, Standard Tasks including universal steps (changelog, docs, version bump, commit and push) and pre-merge extras (e.g., update PR) as unchecked checkboxes, and Post-Merge Reminders (e.g., update plugin) as plain bullets
- **AND** the post-apply workflow has completed all steps including pre-merge constitution extras
- **WHEN** the system is about to create the final commit
- **THEN** the system SHALL mark universal and pre-merge standard task checkboxes as `- [x]` in tasks.md
- **AND** Post-Merge Reminders SHALL remain as plain bullets (they have no checkbox to mark)
- **AND** the committed tasks.md SHALL reflect the Standard Tasks / Post-Merge Reminders distinction
- **AND** no extra follow-up commit SHALL be needed for pre-merge standard task checkboxes

#### Scenario: Conditional post-merge item excluded by scope

- **GIVEN** a constitution with a Post-Merge item that includes a scope hint indicating it applies when plugin files change
- **AND** a proposal whose Scope & Boundaries state only docs/ files are in scope
- **WHEN** the tasks artifact is generated
- **THEN** the Post-Merge Reminders section SHALL be omitted
- **AND** the conditional item SHALL NOT appear in the generated tasks.md

#### Scenario: Conditional post-merge item included by scope

- **GIVEN** a constitution with a Post-Merge item that includes a scope hint indicating it applies when src/ files change
- **AND** a proposal whose What Changes lists modifications to files under src/
- **WHEN** the tasks artifact is generated
- **THEN** the Post-Merge Reminders section SHALL include the item as a plain bullet
- **AND** the scope hint SHALL be stripped from the output

### Requirement: Spec Edits During Implementation

The system SHALL allow implementation tasks to modify spec files (`docs/specs/`) when required by the task description. Specs are edited directly during the specs stage and may need refinements during implementation. Implementation tasks (Foundation, Implementation sections) SHALL NOT include post-apply workflow steps (changelog, docs, version bump). These steps SHALL appear in the Standard Tasks section.

**User Story:** As a developer I want to be able to refine specs during implementation if needed, so that specs stay accurate as implementation reveals edge cases.

#### Scenario: Task refines a spec during implementation

- **GIVEN** a task that says "Add edge case for empty input to user-auth spec"
- **WHEN** the system implements this task
- **THEN** the system SHALL edit `docs/specs/user-auth.md` to add the edge case
- **AND** SHALL mark the task as complete

#### Scenario: Implementation tasks exclude post-apply workflow steps

- **GIVEN** a change being processed through the artifact pipeline
- **WHEN** the tasks artifact is generated
- **THEN** implementation task sections (Foundation, Implementation) SHALL NOT include changelog, docs, or version bump steps
- **AND** these steps SHALL appear in the Standard Tasks section

### Requirement: Apply Gate
Implementation (the apply phase) SHALL be gated by completion of the tasks artifact. The apply phase SHALL NOT begin until tasks.md exists and is non-empty. The apply phase SHALL track progress against the task checklist in tasks.md, marking items complete as implementation proceeds. WORKFLOW.md SHALL declare this gate via the `apply.requires` field referencing the tasks artifact.

**User Story:** As a project lead I want implementation to be gated by task completion in the pipeline, so that developers cannot start coding before the full analysis and planning cycle is done.

#### Scenario: Apply is blocked without tasks
- **GIVEN** a change workspace where preflight.md is the latest completed artifact and tasks.md does not exist
- **WHEN** the user invokes `specshift apply`
- **THEN** the system SHALL reject the apply attempt and report that tasks.md must be generated first

#### Scenario: Apply proceeds after tasks completion
- **GIVEN** a change workspace with all 6 artifacts completed including tasks.md
- **WHEN** the user invokes `specshift apply`
- **THEN** the system SHALL begin implementation, reading the task checklist from tasks.md

#### Scenario: Apply tracks progress in tasks.md
- **GIVEN** the apply phase is active and tasks.md contains 10 unchecked task items
- **WHEN** the agent completes a task item
- **THEN** the system SHALL mark the corresponding `- [ ]` checkbox as `- [x]` in tasks.md

### Requirement: Post-Implementation Commit Before Approval

The WORKFLOW.md `apply.instruction` SHALL direct the agent to commit and push all implementation changes after `specshift apply` passes and before pausing for user approval. The commit message SHALL use the format `specshift(<change-name>): implementation`. This follows the same pattern as post-artifact commit (commit + push per checkpoint) but applies to the implementation phase rather than the artifact phase, and is defined in `apply.instruction` rather than the tasks template. If push fails, the system SHALL continue with a local commit and note that the user should review changes locally. This commit is distinct from the final commit in the Standard Tasks section, which includes changelog, docs, and version bump.

**User Story:** As a developer I want implementation changes committed and pushed before I'm asked for approval, so that I can review the actual PR diff rather than being asked to approve uncommitted local changes.

#### Scenario: Implementation committed before user testing

- **GIVEN** a change with all implementation tasks complete and Auto-Verify passed
- **WHEN** the post-apply workflow reaches the commit-before-approval step
- **THEN** the system SHALL commit all changed files with message `specshift(<change-name>): implementation`
- **AND** SHALL push to the remote branch
- **AND** the PR diff SHALL be available for the user to review before User Testing

#### Scenario: Graceful degradation on push failure

- **GIVEN** a change where Auto-Verify passed but `git push` fails
- **WHEN** the post-apply workflow reaches the commit-before-approval step
- **THEN** the system SHALL create a local commit
- **AND** SHALL note that the user should review changes locally
- **AND** SHALL proceed to User Testing

#### Scenario: Implementation commit does not replace final commit

- **GIVEN** a change where the implementation commit was created during the post-apply workflow
- **AND** the post-apply workflow completes changelog, docs, and version bump
- **WHEN** the Standard Tasks commit step is reached
- **THEN** the system SHALL create a separate commit with all post-apply changes
- **AND** the implementation commit and the final commit SHALL be distinct commits in the git history

### Requirement: Post-Implementation Verification (audit.md)

The system SHALL verify the implementation against change artifacts as part of `specshift apply`, producing an `audit.md` artifact in the change directory. Verification SHALL assess three dimensions: **Implementation** (Completeness + Correctness: task completion, requirement coverage, and scenario coverage), **Testing** (test coverage: automated tests pass, manual test checklist items verified), and **Scope** (Coherence + Side-Effects: design adherence, diff scope, side-effects, and code pattern consistency). The system SHALL read the proposal's frontmatter `capabilities` field to identify affected specs (falling back to parsing the Capabilities section if frontmatter is absent), then read each spec at `docs/specs/<capability>.md` to verify implementation against.

**Draft spec gate:** As part of verification, the system SHALL check all specs listed in the change's proposal for `status: draft` with `change` matching the current change. If any such specs remain in draft status, the verify report SHALL include a CRITICAL issue: "Spec <name> is still in draft status — must be finalized before merge." This gate ensures no draft specs reach the main branch.

**Verify completion (draft→stable flip):** When verify passes (no CRITICAL issues) and the change is approved for merge, the system SHALL finalize tracking fields:
- **Specs**: For all specs modified by this change: set `status: stable`, remove the `change` field, increment `version` by 1, and set `lastModified` to the current date.
- **Proposal**: Set `proposal.md` frontmatter `status` to `review` (indicating the change is verified and ready for PR review; `completed` is set later by the review action after merge).

This completion step runs as part of the post-apply workflow, after user approval and before the merge commit. Each issue found SHALL be classified as CRITICAL (must fix before proceeding), WARNING (should fix), or SUGGESTION (nice to fix). The system SHALL produce a verification report with a summary scorecard, issues grouped by priority, and specific actionable recommendations with file and line references where applicable. The system SHALL err on the side of lower severity when uncertain (SUGGESTION over WARNING, WARNING over CRITICAL).

When a WARNING is **mechanically fixable** — i.e., it involves stale cross-references between artifacts, inconsistent naming, or outdated text that can be corrected by simple text replacement without judgment — the system SHALL auto-fix the issue inline before presenting the report. Auto-fixed issues SHALL still appear in the report as resolved WARNINGs with a note indicating the fix applied. WARNINGs that require user judgment (e.g., spec/design divergence where the user must choose which is correct) SHALL NOT be auto-fixed and SHALL be presented as open issues for user resolution.

The system SHALL load the branch diff (full content and file list) as part of context loading. The diff content SHALL be the **primary evidence source** for verifying what this change introduced. Codebase keyword search SHALL serve as a fallback for requirements that may have been implemented in prior changes or pre-existing code. If no common ancestor with the base branch is available (e.g., orphan branch, first commit), the system SHALL skip all diff-based checks and note "No merge base available — diff checks skipped" in the report. Files under `.specshift/changes/` and `docs/specs/` SHALL be excluded from scope checks as they are expected in the diff for any workflow change.

**Task-Diff Mapping**: For each task marked complete in `tasks.md`, the system SHALL check whether at least one file in the diff corresponds to the task description (keyword match against file paths and diff content). A file-level match alone is insufficient — the system SHALL additionally verify that the diff content relates to the task (e.g., a task about error handling should show error-handling code, not just a comment change). Tasks marked complete that produced no corresponding changes in the diff SHALL be flagged as WARNING.

**Requirement Verification**: For each requirement in the relevant specs, the system SHALL search the diff content for evidence of implementation (primary), falling back to codebase keyword search (secondary). The system SHALL assess both existence and correctness in a single pass: missing requirements are CRITICAL, divergent implementations are WARNING, and matching implementations are satisfied.

**Diff Scope Check**: For each file in the diff, the system SHALL check whether the file is traceable to a task description in `tasks.md` or a component listed in `design.md`'s Architecture & Components section. Files that appear in the diff but have no connection to any task or design component SHALL be collected and reported as a single SUGGESTION with the list of untraced files, rather than one issue per file.

**Preflight Side-Effect Cross-Check**: The system SHALL read `preflight.md` Section C and cross-check each identified side-effect against `tasks.md` entries, diff content, and codebase evidence. Side-effects with neither a matching task nor detectable evidence SHALL be reported as WARNING. If Section C contains no actionable side-effects, the system SHALL skip the cross-check and note it in the report.

The audit.md generation SHALL serve as both the initial verification (tasks.md step 3.2) and the final verification (step 3.5) in the QA loop. When run as a final verify after the fix loop, the verification SHALL operate identically — checking implementation and scope against the current state of code and artifacts. No special flags or modes are needed; the verification is stateless and always checks the current state. The audit.md artifact is persisted in the change directory, replacing the previous transient verify report.

**User Story:** As a developer I want post-implementation verification that checks my code against the specs, so that I can catch gaps, divergences, and inconsistencies before proceeding.

#### Scenario: Verify gates on draft spec status
- **GIVEN** a change `2026-04-08-my-change` that modified spec `quality-gates`
- **AND** `docs/specs/quality-gates.md` has `status: draft` and `change: 2026-04-08-my-change`
- **WHEN** apply generates audit.md for `2026-04-08-my-change`
- **THEN** the report SHALL include a CRITICAL issue: "Spec quality-gates is still in draft status — must be finalized before merge"

#### Scenario: Verify completion flips draft to stable and proposal to completed
- **GIVEN** a change `2026-04-08-my-change` that modified specs `quality-gates` (version 3) and `spec-format` (version 5)
- **AND** `proposal.md` has `status: active`
- **AND** verify passes with no CRITICAL issues
- **AND** the user approves the change for merge
- **WHEN** the verify completion step runs
- **THEN** `quality-gates` frontmatter SHALL be updated to `status: stable`, `change` removed, `version: 4`, `lastModified: 2026-04-08`
- **AND** `spec-format` frontmatter SHALL be updated to `status: stable`, `change` removed, `version: 6`, `lastModified: 2026-04-08`
- **AND** `proposal.md` frontmatter SHALL be updated to `status: review`

#### Scenario: Test coverage verification with automated tests
- **GIVEN** a change with `tests.md` listing 5 automated test files and 2 manual test items
- **AND** all 5 automated test files exist in the project's test directory
- **AND** all 2 manual test items are checked off in tests.md
- **WHEN** apply generates audit.md
- **THEN** the Testing dimension SHALL report "5/5 automated tests present, 2/2 manual items verified"
- **AND** SHALL NOT raise any test coverage issues

#### Scenario: Test coverage with missing automated test file
- **GIVEN** a change with `tests.md` listing 3 automated test files
- **AND** only 2 of the 3 test files exist in the project's test directory
- **WHEN** apply generates audit.md
- **THEN** the Testing dimension SHALL report a WARNING: "Missing test file: <path>"

#### Scenario: Test coverage with unchecked manual items
- **GIVEN** a change with `tests.md` containing 4 manual test checklist items
- **AND** only 2 of the 4 items are checked off
- **WHEN** apply generates audit.md
- **THEN** the Testing dimension SHALL report a WARNING: "2 manual test items not verified"

#### Scenario: Test coverage for project without framework
- **GIVEN** a project with no test framework configured
- **AND** `tests.md` contains only manual test items (no automated tests section)
- **WHEN** apply generates audit.md
- **THEN** the Testing dimension SHALL verify only manual checklist completion
- **AND** SHALL NOT check for automated test files

#### Scenario: Verification with all checks passing

- **GIVEN** a change "add-user-auth" with all tasks complete
- **AND** all spec requirements are implemented and all design decisions are followed
- **WHEN** apply generates audit.md for `add-user-auth`
- **THEN** the system produces a verification report
- **AND** the Implementation dimension shows all tasks complete, requirements verified, and scenarios covered
- **AND** the Scope dimension shows design adherence and no untraced changes
- **AND** the final assessment is "All checks passed. Ready to proceed."

#### Scenario: Verification finds critical issues

- **GIVEN** a change with 5 of 7 tasks marked complete
- **AND** one spec requirement has no corresponding implementation in the codebase
- **WHEN** apply generates audit.md
- **THEN** the report lists 2 CRITICAL issues: incomplete tasks and missing requirement implementation
- **AND** each issue includes a specific recommendation (e.g., "Complete task: Add rate limiting middleware" and "Implement requirement: Session Timeout -- no session timeout logic found in auth module")
- **AND** the final assessment states "2 critical issue(s) found. Fix before proceeding."

#### Scenario: Verification finds implementation diverging from spec

- **GIVEN** a spec requiring JWT-based authentication
- **AND** the implementation uses session cookies instead
- **WHEN** the system verifies requirements against the diff content
- **THEN** the report includes a WARNING: "Implementation may diverge from spec: auth uses session cookies, spec requires JWT"
- **AND** recommends "Review src/auth/handler.ts:45 against requirement: JWT Authentication"

#### Scenario: Verify auto-fixes stale artifact cross-reference

- **GIVEN** a change where `proposal.md` references an artifact filename that was renamed during the specs stage
- **AND** the stale reference is a simple text replacement (old filename → new filename)
- **WHEN** the system generates audit.md
- **THEN** the system SHALL auto-fix the stale reference in `proposal.md`
- **AND** the verification report SHALL list the fix as "WARNING (auto-fixed): Updated stale reference in proposal.md"
- **AND** the report SHALL NOT present it as an open issue requiring user action

#### Scenario: Verify does not auto-fix judgment-requiring divergence

- **GIVEN** a spec requiring JWT authentication
- **AND** the implementation uses session cookies instead
- **WHEN** the system generates audit.md
- **THEN** the system SHALL NOT auto-fix the divergence
- **AND** SHALL present it as an open WARNING for the user to decide whether to update the spec or the code

#### Scenario: Verification finds code pattern inconsistency

- **GIVEN** the project uses kebab-case file naming throughout
- **AND** a new file is named `userAuth.ts` (camelCase)
- **WHEN** the system verifies scope
- **THEN** the report includes a SUGGESTION: "Code pattern deviation: file userAuth.ts uses camelCase, project convention is kebab-case"
- **AND** recommends "Consider renaming to user-auth.ts to follow project pattern"

#### Scenario: Graceful degradation with missing artifacts

- **GIVEN** a change with only tasks.md (no specs or design)
- **WHEN** apply generates audit.md
- **THEN** the system verifies task completion only
- **AND** skips requirement verification and scope checks
- **AND** notes in the report which checks were skipped and why

#### Scenario: Verification with no spec changes

- **GIVEN** a change that has tasks but the proposal lists no capability modifications
- **WHEN** apply generates audit.md
- **THEN** the system skips requirement-level verification
- **AND** focuses on task completion and scope checks

#### Scenario: Final verify confirms fix loop resolved all issues

- **GIVEN** a change where the initial verify found 2 CRITICAL issues
- **AND** the developer fixed both issues in the fix loop
- **WHEN** the system regenerates audit.md as the final verification step (3.5)
- **THEN** the verification report SHALL show 0 CRITICAL issues
- **AND** the report SHALL reflect the current state of all artifacts (including any specs updated during the fix loop)
- **AND** the final assessment SHALL be "All checks passed. Ready to proceed." or note remaining warnings

#### Scenario: Side-effect from preflight not addressed

- **GIVEN** a change where preflight Section C identifies "Regression to existing auth middleware" as a side-effect
- **AND** no task in tasks.md addresses auth middleware regression
- **AND** no codebase evidence of auth middleware changes
- **WHEN** apply generates audit.md
- **THEN** the report includes a WARNING: "Preflight side-effect not addressed: Regression to existing auth middleware"
- **AND** recommends "Add a task or verify that this side-effect is handled in the implementation"

#### Scenario: Side-effect addressed in code but not as task

- **GIVEN** a change where preflight Section C identifies "Schema migration needed for user table"
- **AND** no explicit task in tasks.md mentions schema migration
- **BUT** a migration file exists in the codebase matching the keyword "user table"
- **WHEN** the system cross-checks preflight side-effects
- **THEN** the side-effect is marked as addressed (implementation evidence found)
- **AND** no issue is raised for this side-effect

#### Scenario: Preflight Section C has no side-effects

- **GIVEN** a change where preflight Section C shows all risks assessed as NONE
- **WHEN** apply generates audit.md
- **THEN** the system skips the side-effect cross-check
- **AND** notes "No preflight side-effects to verify" in the report

#### Scenario: Diff scope check finds all files traceable

- **GIVEN** a change with design.md listing `src/skills/verify/SKILL.md` and `docs/specs/quality-gates.md` as components
- **AND** the branch diff contains changes to exactly those two files
- **WHEN** the system performs diff scope verification
- **THEN** all changed files are traceable to design components
- **AND** no diff scope issues are raised

#### Scenario: Task marked complete with no corresponding diff

- **GIVEN** a change with tasks.md containing a completed task "Update error messages in auth module"
- **AND** the branch diff contains no changes to any auth-related files
- **WHEN** the system performs task-diff mapping
- **THEN** the report includes a WARNING: "Task marked complete but no corresponding changes in diff: Update error messages in auth module"

#### Scenario: Task-diff mapping with matching changes

- **GIVEN** a change with tasks.md containing a completed task "Add diff scope check to verify SKILL.md"
- **AND** the branch diff includes `src/skills/verify/SKILL.md`
- **WHEN** the system performs task-diff mapping
- **THEN** the task is confirmed as having corresponding changes
- **AND** no task-diff mapping issue is raised

#### Scenario: Untraced files reported as single suggestion

- **GIVEN** a change where the branch diff includes 3 files not referenced in design.md or any task
- **WHEN** the system performs diff scope verification
- **THEN** the report includes a single SUGGESTION listing all 3 untraced files
- **AND** does not create separate issues per file

#### Scenario: Diff checks skipped when no merge base available

- **GIVEN** a change on an orphan branch with no common ancestor to main
- **WHEN** the system attempts to determine the common ancestor with main
- **AND** no common ancestor exists
- **THEN** the system skips all diff-based checks
- **AND** notes "No merge base available — diff checks skipped" in the report
- **AND** proceeds with keyword-based verification as normal

### Requirement: QA Loop with Mandatory Approval

The system SHALL require explicit human approval before a change can proceed to the post-apply workflow, unless `auto_approve: true` is set in WORKFLOW.md and the audit.md verdict is PASS (no CRITICAL or WARNING issues). When auto_approve is true and audit passes cleanly, the system SHALL auto-approve and proceed without pausing. The QA loop consists of: (1) generating `audit.md` in the change directory using the audit template to produce a persisted verification report, (2) presenting findings to the user, and (3) waiting for an explicit "Approved" response. The system SHALL NOT proceed without receiving explicit human approval. Approval SHALL only be requested after verification has been run and all CRITICAL issues have been resolved. The tasks.md template SHALL include a QA Loop section with an explicit human approval checkbox that MUST be checked before proceeding. Every Success Metric from design.md SHALL be carried over as a PASS/FAIL checkbox in the QA Loop section.

Approval SHALL be gated by a final verification pass. After the Fix Loop completes (all CRITICAL issues resolved, code and specs in sync), a final `audit.md` SHALL be regenerated (Final Verify step) before the user is asked for approval. This ensures that all changes made during the Fix Loop — including spec updates, design changes, and code fixes — are verified as consistent before finalizing. If the Fix Loop was not entered (first verify was clean), the Final Verify step can be marked complete immediately.

The QA Loop SHALL include the following steps in order: Metric Check, Auto-Verify, User Testing, Fix Loop, Final Verify, and Approval. The exact step numbering is a template concern defined in the tasks Smart Template. Implementation changes are committed and pushed before User Testing via the `apply.instruction` in WORKFLOW.md (not as a template step).

**User Story:** As a developer I want a mandatory human approval step before finalizing, so that no change is finalized without my explicit review and sign-off.

#### Scenario: Approval after clean verification

- **GIVEN** a change "add-user-auth" has been implemented and all tasks are complete
- **AND** apply generates `audit.md` which shows no CRITICAL or WARNING issues
- **AND** all success metric checkboxes in the QA Loop section are marked PASS
- **WHEN** the system presents the verification report
- **THEN** the system asks for explicit approval
- **AND** the user responds "Approved"
- **AND** the system proceeds to allow the post-apply workflow

#### Scenario: Approval blocked by critical issues

- **GIVEN** a verification report contains 2 CRITICAL issues
- **WHEN** the system presents the findings to the user
- **THEN** the system SHALL NOT request approval
- **AND** SHALL state that CRITICAL issues must be resolved first
- **AND** SHALL list the specific issues that need resolution

#### Scenario: Approval with warnings acknowledged

- **GIVEN** a verification report contains 0 CRITICAL issues but 3 WARNING issues
- **WHEN** the system presents the findings
- **THEN** the system SHALL request approval while highlighting the warnings
- **AND** the user may respond "Approved" to accept the warnings
- **AND** the system SHALL proceed to allow the post-apply workflow

#### Scenario: Success metrics carried into QA loop

- **GIVEN** a design.md containing 3 success metrics: "Auth response time < 200ms", "All endpoints require valid JWT", "Session expiry after 30 min idle"
- **WHEN** tasks.md is generated
- **THEN** the QA Loop section SHALL contain 3 PASS/FAIL checkboxes, one for each success metric
- **AND** each checkbox SHALL be marked by the user or verifier during the QA phase
- **AND** all checkboxes MUST be marked PASS before approval can be granted

#### Scenario: Final verify after fix loop

- **GIVEN** a change where Auto-Verify found CRITICAL issues
- **AND** the developer completed the Fix Loop, fixing all issues
- **WHEN** the Fix Loop is complete
- **THEN** the system SHALL regenerate `audit.md` one final time (Final Verify)
- **AND** the final verify report SHALL confirm 0 CRITICAL issues
- **AND** only then SHALL the system proceed to request Approval

#### Scenario: Auto-approve bypasses user testing when PASS and auto_approve true

- **GIVEN** `auto_approve: true` in WORKFLOW.md
- **AND** apply generates `audit.md` with verdict PASS and 0 CRITICAL / 0 WARNING issues
- **WHEN** the QA loop reaches the User Testing step
- **THEN** the system SHALL skip the user testing pause
- **AND** SHALL auto-mark the Approval checkbox as complete
- **AND** SHALL proceed to the post-apply workflow

#### Scenario: Auto-approve does NOT bypass when warnings present

- **GIVEN** `auto_approve: true` in WORKFLOW.md
- **AND** apply generates `audit.md` with verdict PASS WITH WARNINGS
- **WHEN** the QA loop reaches the User Testing step
- **THEN** the system SHALL pause for user review of the warnings
- **AND** SHALL NOT auto-approve

#### Scenario: Final verify skipped when first verify is clean

- **GIVEN** a change where Auto-Verify found no CRITICAL or WARNING issues
- **AND** User Testing found no bugs
- **AND** the Fix Loop was not entered
- **WHEN** the QA loop reaches Final Verify
- **THEN** the Final Verify step SHALL be marked complete immediately
- **AND** the system SHALL proceed to Approval

#### Scenario: Final verify finds new issues from fix loop changes

- **GIVEN** a Fix Loop where the developer updated specs and code
- **WHEN** Final Verify is run
- **AND** it discovers that a spec update introduced an inconsistency
- **THEN** the system SHALL report the new issue
- **AND** the developer SHALL return to the Fix Loop to resolve it
- **AND** SHALL re-run Final Verify after the fix

### Requirement: Fix Loop

Verify issues and user correction requests SHALL be resolved via a tiered re-entry process before re-verification. Before applying any fix, the system SHALL classify the correction into one of three tiers and apply the matching re-entry depth:

**Tier 1 — Tweak**: The correction changes a value, line, or detail *within* the current approach (wrong value, typo, missing line, formatting error). Re-entry depth: fix in place, then regenerate `audit.md`.

**Tier 2 — Design Pivot**: The correction changes *which files are modified* or *which approach/abstraction is used*, but requirements are still correct (wrong file edited, wrong architectural pattern, wrong abstraction level). Re-entry depth: update `design.md` to reflect the corrected approach, discard and re-generate the affected task sections in `tasks.md`, re-implement from the updated design, then regenerate `audit.md`.

**Tier 3 — Scope Change**: The correction changes *which requirements apply* or *who the target audience is* (wrong capability scope, missing requirement, wrong consumer model). Re-entry depth: update `docs/specs/<capability>.md` and `proposal.md` to reflect the corrected scope, update `design.md`, re-generate affected tasks, re-implement fully, then regenerate `audit.md`.

**Detection signals** — the system SHALL check these before classifying a correction:
- A completed task needs to be reverted or undone → Design Pivot or Scope Change
- A success metric from `design.md` no longer applies to the corrected implementation → Design Pivot or Scope Change
- A design decision in `design.md` is factually reversed by the correction → Design Pivot
- The correction touches files outside those listed in `design.md` Architecture & Components → Design Pivot
- The correction reveals that a listed requirement does not apply to the correct audience → Scope Change
- More than two incremental fix commits on the same issue → heuristic signal; treat as Design Pivot or Scope Change unless the agent can confirm each commit was a genuine independent Tweak

**Artifact staleness rule**: For Tier 2 and Tier 3 corrections, ALL stale change artifacts SHALL be updated before re-implementing. A stale artifact is any change file (design.md, tasks.md, preflight.md, audit.md) that still describes the original (wrong) approach. The system SHALL NOT leave stale artifacts in the change directory that contradict the corrected implementation.

The bidirectional feedback principle applies at all tiers: updating a spec or design to match the intended implementation is always a valid resolution path.

After all fixes are applied at the appropriate re-entry depth, the system SHALL regenerate `audit.md` to confirm resolution. The system SHALL support iterative fix-verify cycles until all CRITICAL issues are resolved and the user is satisfied with remaining warnings.

**User Story:** As a developer I want a structured fix-verify loop with explicit re-entry tiers, so that approach changes trigger artifact updates and clean re-implementation instead of patch commits on top of a wrong design.

#### Scenario: Classify correction as Tweak — fix in place

- **GIVEN** a review correction that changes a wrong value in an edited file (e.g., wrong version string, missing newline)
- **AND** the approach and affected files remain the same
- **WHEN** the system classifies the correction
- **THEN** it SHALL identify this as Tier 1 — Tweak
- **AND** SHALL fix the value in place
- **AND** SHALL regenerate `audit.md` after the fix

#### Scenario: Classify correction as Design Pivot — update design and re-implement

- **GIVEN** a review correction that points out the wrong file was edited (e.g., `CONSTITUTION.md` was changed instead of `src/templates/constitution.md`)
- **AND** the requirements are still correct, only the implementation target changed
- **WHEN** the system checks detection signals and finds "correction touches files outside those listed in design.md Architecture & Components"
- **THEN** it SHALL identify this as Tier 2 — Design Pivot
- **AND** SHALL update `design.md` Architecture & Components to reflect the correct file targets
- **AND** SHALL discard and re-generate the affected task sections
- **AND** SHALL re-implement the affected tasks from the corrected design
- **AND** SHALL regenerate `audit.md` after re-implementation

#### Scenario: Design Pivot updates all stale artifacts

- **GIVEN** a Design Pivot correction has occurred
- **AND** the existing `audit.md` in the change directory still shows PASS against the original (wrong) approach
- **WHEN** the system applies the Tier 2 re-entry
- **THEN** it SHALL update `design.md` to reflect the corrected approach
- **AND** SHALL update `tasks.md` affected sections to remove old tasks and add corrected ones
- **AND** SHALL delete `audit.md` (stale) before re-implementing
- **AND** SHALL generate a new `audit.md` from the corrected implementation

#### Scenario: Classify correction as Scope Change — update specs and re-implement

- **GIVEN** a review correction that reveals the wrong capability scope was targeted (e.g., a requirement listed in the proposal does not apply to the correct audience)
- **AND** the implementation approach may be sound, but the requirements themselves need revision
- **WHEN** the system checks detection signals and finds "correction reveals that a listed requirement does not apply to the correct audience"
- **THEN** it SHALL identify this as Tier 3 — Scope Change
- **AND** SHALL update `docs/specs/<capability>.md` and `proposal.md` to reflect the corrected scope
- **AND** SHALL update `design.md` to align with the corrected requirements
- **AND** SHALL re-generate affected task sections in `tasks.md`
- **AND** SHALL re-implement fully from the corrected design
- **AND** SHALL regenerate `audit.md` after re-implementation

#### Scenario: Fix code to resolve critical issue

- **GIVEN** a verification report with CRITICAL issue "Requirement not found: Session Timeout"
- **WHEN** the developer implements session timeout logic in the auth module
- **AND** regenerates `audit.md`
- **THEN** the new verification report no longer lists the session timeout issue as CRITICAL
- **AND** the Completeness dimension reflects the additional requirement coverage

#### Scenario: Update spec to resolve warning

- **GIVEN** a verification report with WARNING "Implementation may diverge from spec: auth uses session cookies, spec requires JWT"
- **AND** the developer intentionally chose session cookies over JWT
- **WHEN** the developer updates the spec to reflect session cookie authentication
- **AND** regenerates `audit.md`
- **THEN** the new verification report no longer lists the divergence warning
- **AND** the spec accurately reflects the implementation

#### Scenario: Multiple fix-verify iterations

- **GIVEN** a first verification finds 3 CRITICAL and 2 WARNING issues
- **WHEN** the developer fixes all 3 CRITICAL issues and regenerates `audit.md`
- **THEN** the second report shows 0 CRITICAL issues
- **AND** may show the same 2 warnings plus any new issues introduced by the fixes
- **AND** the developer may choose to address warnings or approve with acknowledged warnings

#### Scenario: Fix introduces new issue

- **GIVEN** a developer fixes a CRITICAL issue by refactoring the auth module
- **AND** the refactoring removes a function that another requirement depends on
- **WHEN** the developer regenerates `audit.md`
- **THEN** the original CRITICAL issue is resolved
- **BUT** a new CRITICAL issue appears for the broken dependency
- **AND** the developer must address the new issue before approval

#### Scenario: Bidirectional feedback -- update design

- **GIVEN** a verification finds that the implementation uses a different architectural pattern than design.md specifies
- **AND** the new pattern is superior and the developer wants to keep it
- **WHEN** the developer updates design.md to document the actual architecture
- **AND** regenerates `audit.md`
- **THEN** the coherence check passes because design.md now matches the implementation

### Requirement: Active vs Completed Change Detection

The router SHALL distinguish change phases using the proposal's `status` frontmatter field. The lifecycle has three states:
- **`active`** — change is being developed (propose, apply phases). Set at change creation.
- **`review`** — implementation verified, PR under review. Set when audit.md passes (same step that flips spec `draft → stable`).
- **`completed`** — PR merged, change finished. Set by the review action after successful merge.

A change is considered **active** if its `proposal.md` has `status: active` or has no `status` field (legacy/early pipeline). A change is considered **in review** if its `proposal.md` has `status: review`. A change is considered **completed** if its `proposal.md` has `status: completed`.

**Fallback** (for proposals without frontmatter): A change is active if its `tasks.md` contains at least one unchecked item (`- [ ]`) or if `tasks.md` does not exist. A change is completed if its `tasks.md` exists and all items are checked (`- [x]`).

Actions that operate on active changes (propose, apply) SHALL filter to active changes. Actions that operate on changes in review (finalize, review) SHALL filter to changes with `status: review` or `status: completed`. The review action SHALL also accept `status: review` changes (its primary input).

**User Story:** As a developer I want the system to distinguish active from completed changes using structured metadata, so that detection is instant and does not require parsing task checkboxes.

#### Scenario: Active change detected via proposal status

- **GIVEN** a change at `.specshift/changes/2026-04-01-add-auth/` with `proposal.md` containing `status: active`
- **WHEN** `specshift apply` lists available changes
- **THEN** the change is shown as available for implementation

#### Scenario: Completed change detected via proposal status

- **GIVEN** a change at `.specshift/changes/2026-04-01-add-auth/` with `proposal.md` containing `status: completed`
- **WHEN** `specshift finalize` lists available changes
- **THEN** the change is included in changelog generation

#### Scenario: Change without proposal frontmatter falls back to tasks.md

- **GIVEN** a change at `.specshift/changes/2026-03-01-legacy/` with `proposal.md` without YAML frontmatter
- **AND** `tasks.md` contains unchecked items
- **WHEN** `specshift apply` lists available changes
- **THEN** the change is shown as active (fallback to tasks.md parsing)

#### Scenario: Change without tasks.md is active

- **GIVEN** a change at `.specshift/changes/2026-04-01-add-auth/` with research.md and proposal.md (`status: active`) but no tasks.md
- **WHEN** `specshift propose` lists available changes
- **THEN** the change is shown as available for artifact generation
