---
name: refactoring
description: Refactor Python functions or classes while preserving intended behavior and updating affected callers.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(isort:*), Bash(black:*)
---

# Python refactoring

Use the behavior, callers, and tests in the affected area to guide structural changes.
Follow repository-required directory migrations even for a small refactor; update imports,
build references, and mirrored tests together. Otherwise avoid unrelated cleanup.

- Split mixed responsibilities into named operations without mechanically fragmenting a
  coherent algorithm to meet a line-count limit.
- Put public functions/methods before private helpers and annotate arguments and returns.
- Use guard clauses and specific exceptions; introduce a custom exception when callers
  need that distinction. Log at the responsible boundary. Filesystem prechecks do not
  replace handling the operation's own errors.
- Extract shared behavior before adding a variant. Use explicit parameters, a callable,
  or composition for differences; update every affected caller and remove superseded paths.
- Preserve intended behavior unless asked to change it. Cover the motivating behavior and
  relevant error/edge cases, reusing existing regression coverage where sufficient.
- Format affected Python files with the repository's configured tools and run relevant
  checks. Report behavior changes separately from structural changes.

Consult `general-coding` for shared conventions and `add-modify-codebase` for contract-change
workflow when needed. These references do not require reading every skill before editing.
