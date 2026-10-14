---
name: general-coding
description: Apply explicit-input and error-handling conventions when implementing or refactoring code.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# General coding

## Client navigation and editing

For client application tabs and editors, keep the original view state and open editors as overlays.
Do not reload the original view on tab return or editor close.
Follow the [client navigation and editing contract](../add-modify-codebase/references/client-navigation.md) when this work affects those flows.
Apply it to designs, implementation, reviews, or changed-client release checks within this skill's existing scope.


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
  project's chosen format for tools; background services use `config.toml`. Do not create a second configuration path.
- Do not add environment reads inside business code. Container entrypoints may translate
  noncredential environment inputs; background-service credentials come only from TOML.

## Background service credentials

For every background service, including API and OCR services, store all configured API keys,
tokens, signing secrets and passwords as embedded values in that service's `config.toml`.
Do not use separate secret/key/password files, `*_file` credential settings, Docker secrets,
or environment/CLI credential overrides as a second source. Load and validate the TOML at
the service boundary and pass typed settings explicitly. Preserve the repository's data-root
and startup CLI conventions. Templates contain empty placeholders, never real credentials;
secret-bearing operator TOMLs are owner-readable only and excluded from Git, image build
contexts, client payloads, logs and ordinary business-data exports.

When converting an existing credential-file setup, preserve key values and operator settings,
validate the complete replacement, persist it atomically, and remove project-owned obsolete
files only after the TOMLs are durable. Stop affected services before replacing live
configuration/layout. User-account password hashes and issued-session records remain business
data in their established stores; build/signing artifacts and platform-managed certificates
are outside this background-service configuration rule.

Contract-change and regression workflow is in `add-modify-codebase`; placement is in repository
instructions and `project-structure`. Consult them when those decisions arise, not on every edit.
