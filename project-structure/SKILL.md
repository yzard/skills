---
name: project-structure
description: Place or move source, tests, documentation, and build outputs using the repository layout and migration rules.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Project structure

Repository-specific layouts and explicit user choices override these shared defaults. Apply
this skill to file placement and directory migrations, not to every edit inside an existing file.

| Content | Default location |
|---------|------------------|
| Backend / frontend source | `src/backend/`, `src/frontend/` |
| Shared contracts or code | `src/shared/`, or the repository's named shared component |
| Tests | `tests/`, mirroring the source path and language's test filename convention |
| Architecture, guides, design notes | `docs/` |
| E2E configuration and runtime data | `playground/` |
| Intermediate artifacts / final products | Root `build/<component>/`, `dist/<component>/` |
| Dockerfiles, entrypoints, Compose | `docker/`, with documented repository exceptions |
| Build/run entrypoints | Root `build_docker.sh`, `run_playground.sh`, plus repository-owned platform entrypoints |

Root manifests, lockfiles, README and agent instruction files are normal exceptions. Skills
and other tool-native packages retain their required layouts (`SKILL.md`, `references/`,
`scripts/`, etc.); do not force them into application `src/` or `docs/` trees.

## Tests and moves

Map `src/backend/routes/media.rs` to `tests/backend/routes/media.rs`; keep intermediate
resource directories intact. Move/rename relevant tests with source and update test registration.
Use the repository's test harness; do not expose private APIs solely to satisfy file layout.
Coverage concerns executable behavior, not placeholder tests for types, exports, or documentation.

When the affected directory violates the required layout, migrate that affected tree even if
the requested edit is small. Update imports, callers, tests, fixtures, manifests, build/CI
references, and documentation together; remove the obsolete layout. Do not add the feature
in the wrong location or leave two parallel structures. Follow repository-specific migration
boundaries; do not sweep unrelated directories into the change.

## Conditional details

When changing playground/build/Docker paths, read [build and runtime paths](references/build-paths.md).
For image contents and entrypoint behavior use `docker-build` if relevant. Ordinary source or
documentation placement does not require reading those workflows.
