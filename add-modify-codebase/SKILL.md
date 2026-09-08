---
name: add-modify-codebase
description: Change existing behavior, signatures, or shared implementations with explicit caller updates and regression coverage.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash
---

# Changing an existing codebase

Use the repository's contracts and the user's requested outcome. This skill governs behavior
and contract changes; it does not require a full workflow for documentation or formatting edits.

## Contract changes

New arguments have no implicit defaults merely to preserve callers. Change the signature and
update every affected producer, consumer, fixture, generated type, and test explicitly. Remove
superseded internal paths instead of keeping aliases or deprecated wrappers. Compatibility
requires a concrete shipped external or persisted-data contract, or an explicit user request.

## Shared implementations

Before adding a variant of existing behavior, extract its shared implementation and make both
callers use it. Keep real differences in explicit parameters or composed collaborators. Avoid
abstractions based only on incidental similarity; preserve distinct domain responsibilities.

## Layout and scope

Use the repository's layout rules for the affected tree and mirrored tests. If those rules
require migrating a nonconforming directory even for a small edit, complete that migration
with its imports, callers, build/CI references, and tests in the same change. Do not leave
parallel old/new layouts or compatibility re-exports. A small diff is not a reason to skip
a required migration. Otherwise keep structural changes within the requested feature.

## Completion

Cover changed behavior and meaningful failure paths with regression tests. Existing tests can
supply coverage; do not add one test per line or tests that merely repeat the implementation.
Use the repository's actual test runner and layout, not paths from a generic example.

Run affected tests and relevant type/build/lint checks. Broaden validation when contract or
layout changes affect multiple components. Once checks pass, repeat them only for subsequent
changes, failures, or a specific unresolved risk. Finish the authorized implementation and
verification; report remaining blockers without claiming unrun checks passed.

For error/configuration conventions consult `general-coding` when needed; file-placement
questions belong to repository guidance and `project-structure`. Load stack-specific skills
only for the mechanisms being changed, not as a mandatory chain.
