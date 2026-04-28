---
id: constitution
template-version: 1
description: Project governance template for bootstrap
generates: CONSTITUTION.md
requires: []
instruction: |
  Generate project-specific rules from codebase analysis.
  Fill in each section based on what you discover in the code.
  Use REVIEW markers for items needing user confirmation.
  Version-bump detection: scan for version files (package.json, pyproject.toml,
  Cargo.toml, plugin.json, setup.cfg, version.txt, etc.). If found, generate a
  version-bump convention in the Conventions section describing which file(s) to
  bump and how (e.g., "increment patch in package.json"). If no version file is
  found, omit the version-bump convention — finalize will skip the step.
---
# Project Constitution

## Tech Stack
(Language, Runtime, Framework, Database, Testing, Package Manager)

## Testing
(Framework, test directory, file pattern, import style, conventions.
 Leave empty or remove section if project has no test infrastructure.)

## Architecture Rules
(Architectural patterns, module boundaries, dependency direction)

## Code Style
(Coding conventions, formatting, naming patterns)

## Constraints
(Limits, requirements, compatibility rules)

## Conventions
(Naming, commits, branching, file organization.
 If the codebase has a version file, include a version-bump convention
 describing which file to bump and how. Finalize step 3 follows this
 convention — if none exists, version-bump is skipped.)

## Standard Tasks

<!-- Project-specific extras appended to the universal standard tasks in the tasks template.
     Pre-merge items use checkbox format and are executed during post-apply workflow.
     Post-merge items use plain bullet format and are manual reminders after PR merge. -->

### Pre-Merge
<!-- - [ ] Example: Update PR body with summary -->

### Post-Merge
<!-- - Example: Run deployment script -->
