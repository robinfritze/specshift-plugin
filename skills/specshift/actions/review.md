# Requirements


### Requirement: PR State Assessment and Re-Entrancy

The `specshift review` action SHALL be re-entrant across sessions. On each invocation, the action SHALL identify the PR number from the current branch using available GitHub tooling (gh CLI, MCP tools, or API), then read the PR's current state including: draft status, requested reviews, review decisions (approved, changes-requested, commented), unresolved comment threads, and CI check status. The action SHALL report the assessed state before proceeding to the applicable phase. If no PR exists for the current branch, the action SHALL report the situation and stop. The action SHALL NOT store session-local state — all state SHALL be derived from the PR on GitHub and the local change artifacts.

**User Story:** As a developer using Claude Code Web I want the review action to pick up where it left off in any session, so that ephemeral sessions do not block my PR from being merged.

#### Scenario: Fresh invocation reads PR state from GitHub
- **GIVEN** a branch with an open draft PR
- **WHEN** `specshift review` is invoked
- **THEN** the action reads draft status, reviews, comments, and checks from GitHub using available tooling
- **AND** reports the assessed state before proceeding

#### Scenario: Re-entrant invocation continues from current state
- **GIVEN** a previous review session ended after requesting a review
- **AND** new review comments have arrived on the PR
- **WHEN** `specshift review` is invoked in a new session
- **THEN** the action reads current PR state
- **AND** detects the new unresolved comments
- **AND** proceeds to comment processing without restarting from draft transition

#### Scenario: No PR exists for current branch
- **GIVEN** the current branch has no associated pull request
- **WHEN** `specshift review` is invoked
- **THEN** the action reports that no PR was found for the branch
- **AND** suggests running `specshift finalize` first or creating a PR manually

### Requirement: Draft-to-Ready Transition

When the PR is in draft state, the review action SHALL mark it ready for review using available GitHub tooling and update the PR body with a change summary derived from the change artifacts (proposal.md summary and issue references). If the proposal references a GitHub issue, the PR body SHALL include a closing reference (e.g., `Closes #X`). If the PR is already marked ready for review, the action SHALL skip this step and proceed to the next applicable phase.

**User Story:** As a developer I want the review action to automatically prepare my PR for review, so that I don't have to manually update the PR status and description.

#### Scenario: Draft PR marked ready with updated body
- **GIVEN** a draft PR for the current branch
- **AND** a proposal.md describing the change
- **WHEN** the review action runs
- **THEN** it marks the PR ready for review using available GitHub tooling
- **AND** updates the PR body with a change summary from proposal.md

#### Scenario: Already-ready PR skips transition
- **GIVEN** a PR that is already marked ready for review
- **WHEN** the review action runs
- **THEN** it skips the draft-to-ready step
- **AND** proceeds to the next applicable phase

#### Scenario: PR body includes issue references
- **GIVEN** a proposal.md that references issue #42
- **WHEN** the review action updates the PR body
- **THEN** the body includes a closing reference (e.g., `Closes #42`)

### Requirement: Review Request Dispatch

Before dispatching a review request, the action SHALL verify the working tree is clean (no uncommitted changes). If uncommitted changes exist (e.g., from finalize's compilation step), the action SHALL commit and push them before proceeding to review dispatch. After the PR is marked ready, the review action SHALL request a review based on the `review.request_review` configuration from WORKFLOW.md frontmatter (defined in the Review Action Configuration requirement of workflow-contract.md). If `copilot`: request a Copilot review using available GitHub tooling. If `true`: request a review from the repository's default reviewers. If `false` or absent: skip the review request. If reviews have already been requested or completed, the action SHALL NOT re-request. If the review request fails (tool unavailable, reviewer not configured), the action SHALL log a warning and continue without blocking.

**User Story:** As a project maintainer I want configurable reviewer assignment, so that I can choose the right review strategy for my project without editing action instructions.

#### Scenario: Copilot review requested per configuration
- **GIVEN** `review.request_review: copilot` in WORKFLOW.md
- **AND** no reviews have been requested yet
- **WHEN** the review action runs
- **THEN** it requests a Copilot review using available GitHub tooling

#### Scenario: Review already requested is not re-requested
- **GIVEN** a review has already been requested from Copilot
- **WHEN** the review action is re-invoked in a new session
- **THEN** it detects the existing review request
- **AND** skips the review request step

#### Scenario: Review request failure does not block
- **GIVEN** `review.request_review: copilot`
- **AND** the Copilot review request fails (tool unavailable)
- **WHEN** the review action runs
- **THEN** it logs a warning with the failure reason
- **AND** continues to the next step without blocking

#### Scenario: Uncommitted changes committed before review dispatch
- **GIVEN** the working tree has uncommitted changes after finalize
- **WHEN** the review action reaches the review dispatch phase
- **THEN** it commits and pushes the uncommitted changes
- **AND** proceeds to request the review

### Requirement: Review Comment Processing

The review action SHALL process unresolved review comment threads on the PR. For each unresolved thread, the action SHALL: read the comment content, determine if the feedback is actionable within the current change scope, implement the fix if actionable, reply to the thread explaining the action taken, and resolve the thread. If a comment requires a fundamental change beyond the current scope (e.g., architectural redesign, new requirements), the action SHALL NOT attempt the fix; instead it SHALL inform the user and suggest starting a new `specshift propose` for the requested change. After processing all threads: the action SHALL commit and push the fixes, then run the built-in review skill for self-check to verify the fixes did not introduce regressions. If the self-review finds issues, the action SHALL fix them before proceeding.

**User Story:** As a developer I want review comments automatically addressed and verified, so that the review-fix cycle is handled without manual intervention for straightforward feedback.

#### Scenario: Actionable review comment processed and resolved
- **GIVEN** an unresolved review thread requesting a concrete code change
- **WHEN** the review action processes the thread
- **THEN** it implements the requested change
- **AND** replies to the thread explaining the fix
- **AND** resolves the thread
- **AND** commits and pushes the changes

#### Scenario: Out-of-scope comment deferred to user
- **GIVEN** an unresolved review thread requesting an architectural redesign
- **WHEN** the review action evaluates the comment
- **THEN** it determines the change is out of scope
- **AND** informs the user with a suggestion to run `specshift propose` for the requested change
- **AND** does NOT resolve the thread

#### Scenario: Self-review after fixes catches regression
- **GIVEN** the review action has implemented fixes for review comments
- **AND** committed and pushed the changes
- **WHEN** the built-in review skill runs as self-check
- **THEN** it detects a new issue introduced by the fixes
- **AND** the action fixes the regression before proceeding

### Requirement: Review Cycle Safety Limit

The review action SHALL support iterative review cycles: after processing comments and pushing fixes, if a reviewer posts new comments, the action SHALL return to comment processing. To prevent infinite loops, the action SHALL enforce a maximum of 3 review-fix cycles per invocation. After reaching the limit, the action SHALL pause and report the situation to the user, listing remaining unresolved threads. The user may then manually resolve remaining comments or re-invoke the review action. Each cycle consists of: processing all current unresolved threads, committing and pushing fixes, running self-review, and checking for new comments.

**User Story:** As a developer I want a safety limit on review cycles, so that AI reviewers that keep finding new issues do not cause an infinite loop.

#### Scenario: Second review cycle processes new comments
- **GIVEN** the first cycle resolved 3 threads and pushed fixes
- **AND** the reviewer posts 2 new comments after the push
- **WHEN** the review action detects the new comments
- **THEN** it enters cycle 2
- **AND** processes the 2 new threads

#### Scenario: Safety limit reached after 3 cycles
- **GIVEN** the action has completed 3 review-fix cycles
- **AND** the reviewer posts another comment
- **WHEN** the action detects new unresolved threads
- **THEN** it reports "Safety limit reached: 3 review cycles completed. N unresolved threads remain."
- **AND** pauses for user intervention
- **AND** does NOT process the new comments automatically

#### Scenario: No new comments after fixes proceeds to merge check
- **GIVEN** the action processed all review comments in cycle 1
- **AND** pushed fixes
- **AND** no new comments arrive
- **WHEN** the action checks for new threads
- **THEN** it proceeds to the merge readiness check

### Requirement: Pre-Merge Summary Comment

Before asking the user for merge confirmation, the review action SHALL post a PR comment summarizing the review activity. The summary comment SHALL include: (1) the number of review threads processed and resolved, (2) a brief list of fixes implemented, (3) the self-check result (pass or fail, and whether any findings were fixed), (4) the number of review cycles completed, and (5) a final status line in the format: `Ready for merge — N threads resolved, M fixes applied, self-check passed`. If no review threads were processed (e.g., no comments were posted by a reviewer), the summary SHALL still be posted with zeroed counts to confirm the action assessed the PR. If posting the comment fails (tooling error, permission issue), the action SHALL log a warning and continue to the merge confirmation — a failed summary SHALL NOT block the merge flow. On re-entrant invocations, the action SHALL check whether a summary comment has already been posted (by searching for a `<!-- specshift:review-summary -->` marker in the comment body); if found, the action SHALL update the existing comment rather than posting a duplicate.

**User Story:** As a developer (and anyone watching the PR) I want a summary comment posted before merge, so that there is a clear audit trail of what the AI did during the review process.

#### Scenario: Summary comment posted before merge confirmation
- **GIVEN** the review action has processed 4 review threads and resolved all of them
- **AND** implemented 3 fixes across 2 review cycles
- **AND** the self-check passed with no findings
- **WHEN** no unresolved comments remain and CI checks are passing
- **THEN** the action posts a PR comment containing the summary
- **AND** the comment includes "4 threads resolved, 3 fixes applied, self-check passed"
- **AND** the action proceeds to ask the user for merge confirmation

#### Scenario: Summary posted with zero counts when no review comments existed
- **GIVEN** the PR received no review comments
- **AND** CI checks are passing
- **WHEN** the review action reaches the pre-merge phase
- **THEN** the action posts a summary comment with "0 threads resolved, 0 fixes applied, self-check passed"
- **AND** proceeds to merge confirmation

#### Scenario: Summary comment failure does not block merge
- **GIVEN** the review action has completed processing
- **AND** posting the PR comment fails due to a tooling error
- **WHEN** the action attempts to post the summary
- **THEN** it logs a warning with the failure reason
- **AND** continues to ask the user for merge confirmation

#### Scenario: Re-entrant invocation updates existing summary instead of duplicating
- **GIVEN** a previous session posted a summary comment with "2 threads resolved, 2 fixes applied"
- **AND** a new review cycle resolved 1 additional thread with 1 fix
- **WHEN** the review action reaches the pre-merge phase in the new session
- **THEN** it detects the existing summary comment by its marker
- **AND** updates it to reflect the cumulative totals: "3 threads resolved, 3 fixes applied"
- **AND** does NOT post a second summary comment

### Requirement: Merge Execution with Mandatory Confirmation

When no unresolved review threads remain, CI checks are passing, and no requested review is pending without a decision (i.e., every requested reviewer has submitted a decision — approved, changes-requested, or commented), the review action SHALL ask the user for explicit merge confirmation before proceeding. If a review was requested (via `review.request_review` configuration) but no review decision has been submitted yet, the action SHALL report "Review pending — waiting for reviewer decision" and suggest re-running `specshift review` later; it SHALL NOT offer merge. This confirmation SHALL be required regardless of the `auto_approve` setting — `auto_approve` controls only whether the review action is auto-dispatched from finalize, not whether the merge itself is automatic (as defined in the Review Action Configuration requirement of workflow-contract.md). If CI checks are pending, the action SHALL report the status and suggest waiting or re-invoking later. If CI checks are failing, the action SHALL report the failures and stop without offering merge. After user confirmation, the action SHALL first set the proposal's `status` frontmatter to `completed` (completing the `active → review → completed` lifecycle), commit and push the change so it is included in the squash merge. The action SHALL then merge the PR via squash using available GitHub tooling. The squash commit message SHALL be composed rather than using GitHub's default (which concatenates individual commit messages). The commit title SHALL be the PR title followed by the PR number in parentheses (e.g., `Fix auth timeout (#42)`). The commit body SHALL contain the proposal's **Why** section (problem statement), followed by a blank line and the **What Changes** bullet list, followed by any issue-closing references (e.g., `Closes #31`). Post-merge cleanup (worktree removal, branch deletion) SHALL follow the Post-Merge Worktree Cleanup requirement in change-workspace.md.

**User Story:** As a developer I want the merge to always require my explicit approval, so that I maintain control over what reaches the main branch even in fully automated workflows.

#### Scenario: Merge after user confirmation with passing CI
- **GIVEN** no unresolved review threads
- **AND** all CI checks pass
- **AND** no requested review is pending without a decision
- **WHEN** the action asks for merge confirmation
- **AND** the user confirms
- **THEN** the action sets proposal `status` to `completed`, commits, and pushes
- **AND** merges the PR via squash with a composed commit message
- **AND** triggers post-merge cleanup per change-workspace.md

#### Scenario: CI checks pending delays merge
- **GIVEN** no unresolved review threads
- **AND** CI checks are still running
- **WHEN** the action checks CI status
- **THEN** it reports the pending checks
- **AND** suggests waiting or re-invoking `specshift review` later

#### Scenario: CI checks failing stops merge
- **GIVEN** no unresolved review threads
- **AND** one CI check has failed
- **WHEN** the action checks CI status
- **THEN** it reports the failure
- **AND** does NOT offer the merge confirmation
- **AND** suggests investigating the failure

#### Scenario: Merge confirmation required even with auto_approve
- **GIVEN** `auto_approve: true` in WORKFLOW.md
- **AND** PR is approved with all checks passing
- **WHEN** the review action reaches the merge phase
- **THEN** it SHALL pause and ask for explicit user confirmation
- **AND** SHALL only merge after the user confirms

#### Scenario: Review pending blocks merge offer
- **GIVEN** a review was requested from Copilot
- **AND** the review has not yet been submitted (no decision)
- **AND** no unresolved review threads exist
- **AND** CI checks are passing
- **WHEN** the action checks merge readiness
- **THEN** it reports "Review pending — waiting for reviewer decision"
- **AND** does NOT offer merge confirmation
- **AND** suggests re-running `specshift review` later

#### Scenario: Squash merge uses clean commit message from proposal
- **GIVEN** the user has confirmed merge
- **AND** the PR title is "Fix auth timeout" and the PR number is 42
- **AND** proposal.md has a Why section and What Changes section
- **AND** proposal.md references issue #31
- **WHEN** the action merges the PR
- **THEN** the commit title is `Fix auth timeout (#42)`
- **AND** the commit body contains the Why section text and What Changes bullets
- **AND** the commit body includes `Closes #31`
- **AND** the commit body does NOT contain pipeline commit messages

### Requirement: Review Action Configuration

WORKFLOW.md frontmatter SHALL support an optional `review` configuration object for the `review` built-in action. The `review` object SHALL contain:
- `request_review` (optional, defaults to `false`) — controls whether the review action requests an external review when marking the PR ready. Values: `false` (no reviewer assigned), `copilot` (request Copilot review using available GitHub tooling), `true` (request review from the repository's default reviewers using available GitHub tooling). If the review request fails (tool unavailable, reviewer not configured), the action SHALL log a warning and continue without blocking.

The `review` action is a built-in action with compiled requirements (extracted from `docs/specs/review-lifecycle.md`) and an instruction defined via `## Action: review` in the WORKFLOW.md body. Its behavior (re-entrant PR state machine, review comment processing, self-review via built-in tools, merge with user confirmation) is specified in the review-lifecycle spec. This requirement covers only the frontmatter configuration surface.

When `auto_approve` is `true` and `review` is listed in the `actions` array, the router SHALL auto-dispatch from finalize to review after finalize completes successfully. The review action SHALL always pause for explicit user confirmation before merging, regardless of the `auto_approve` setting — `auto_approve` controls only the dispatch (whether review is started automatically), not the merge itself.

**User Story:** As a consumer project maintainer I want configurable PR review automation, so that I can choose between no reviewer, Copilot review, or default reviewers without modifying the action instruction text.

#### Scenario: Review configuration with request_review false (default)
- **GIVEN** WORKFLOW.md contains `review: { request_review: false }`
- **AND** `review` is in the `actions` array
- **WHEN** the review action runs
- **THEN** it SHALL mark the PR ready for review and update the PR body
- **AND** SHALL NOT request a reviewer
- **AND** SHALL process any review comments that exist or arrive

#### Scenario: Review configuration with request_review copilot
- **GIVEN** WORKFLOW.md contains `review: { request_review: copilot }`
- **WHEN** the review action runs
- **THEN** it SHALL request a Copilot review using available GitHub tooling
- **AND** if the request fails, SHALL log a warning and continue

#### Scenario: Review configuration with request_review true
- **GIVEN** WORKFLOW.md contains `review: { request_review: true }`
- **WHEN** the review action runs
- **THEN** it SHALL request a review from the repository's default reviewers using available GitHub tooling

#### Scenario: Review action always requires user confirmation for merge
- **GIVEN** `auto_approve: true` in WORKFLOW.md
- **AND** the PR is approved with all checks passing
- **WHEN** the review action reaches the merge phase
- **THEN** it SHALL pause and ask the user for explicit confirmation before merging
- **AND** SHALL only merge after the user confirms

#### Scenario: Review action is re-entrant across sessions
- **GIVEN** the review action was started in a previous session that ended
- **AND** review comments were partially processed
- **WHEN** the user invokes `specshift review` in a new session
- **THEN** the action SHALL read the current PR state from GitHub
- **AND** SHALL continue processing from the current state (not restart from scratch)

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
