---
name: general-coding
description: Apply explicit-input and error-handling conventions when implementing or refactoring code.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# General coding

Repository contracts and explicit user choices override shared preferences. Apply conventions
to the affected implementation and callers, including any repository-required layout migration.

- Use early returns for validation, authorization, and failure paths unless nesting is clearer.
- Catch specific errors where recovery or translation is possible. A top-level service/task
  boundary may catch broadly to record failure; preserve cancellation and cleanup. Never
  return fake success or silently substitute defaults after a failure.
- Keep error ownership clear: retain useful context, log once at the responsible boundary,
  and keep secrets out of logs and public errors.
- Function arguments have no defaults unless requested or required by a framework/language
  contract. New arguments require explicit decisions at every caller.
- Replace obsolete internal APIs completely. Compatibility requires a concrete external or
  persisted-data constraint or explicit user requirement.
- Extract shared behavior before adding a near-duplicate; use explicit parameters or
  composition for real differences. Update affected callers to one coherent implementation.
- Parse CLI/config at the application boundary and pass typed configuration explicitly.
  Use flags for a small new executable or a config file for structured settings; preserve the
  project's chosen format. Axum services use TOML. Do not create a second configuration path.
- Do not add environment reads inside business code. Container entrypoints may translate
  environment inputs; preserve documented config-loader exceptions.

Contract-change and regression workflow is in `add-modify-codebase`; placement is in repository
instructions and `project-structure`. Consult them when those decisions arise, not on every edit.
