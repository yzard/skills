---
name: project-structure
description: Place or move source, tests, documentation, and build outputs using the repository layout and migration rules.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Project structure

## Client navigation and editing

For client application tabs and editors, keep the original view state and open editors as overlays.
Do not reload the original view on tab return or editor close.
Follow the [client navigation and editing contract](../add-modify-codebase/references/client-navigation.md) when this work affects those flows.
Apply it to designs, implementation, reviews, or changed-client release checks within this skill's existing scope.


Repository-specific layouts and explicit user choices override these shared defaults. Apply
this skill to file placement and directory migrations, not to every edit inside an existing file.

| Content | Default location |
|---------|------------------|
| Backend / frontend source | `src/backend/`, `src/frontend/` |
| Android / iOS platform source | `src/android/`, `src/ios/` |
| API backend / OCR backend, when implemented | `src/backend_api/`, `src/backend_ocr/` |
| Shared contracts or code | `src/shared/`, or the repository's named shared component |
| Tests | `tests/`, mirroring the source path and language's test filename convention |
| Architecture, guides, design notes | `docs/` |
| E2E configuration and runtime data | `playground/` |
| Intermediate artifacts / final products | Root `build/<component>/`, `dist/<component>/` |
| Dockerfiles, entrypoints, Compose | `docker/`, with documented repository exceptions |
| Build/run entrypoints | Root `build_docker.sh`, `run_playground.sh`, and applicable `build_android.sh` / `build_ios.sh` |

README, agent instructions, and genuinely project-wide manifests and lockfiles are normal
root exceptions. Component-owned manifests, lockfiles, and tool configuration belong with
their component, not at the repository root merely because the tool supports it. Skills and
other tool-native packages retain their required layouts (`SKILL.md`, `references/`,
`scripts/`, etc.); do not force them into application `src/` or `docs/` trees.

## Manifest and lockfile ownership

Place a Rust backend's `Cargo.toml`, `Cargo.lock`, and component-specific `.cargo/` in
`src/backend/` (or its actual named component directory). An independently managed shared
Rust library belongs in `src/shared/` with its own Cargo files. Do not create an empty shared
component or a root Cargo workspace just to retain root manifests. Root `Cargo.*` files are
appropriate only when they actually manage the whole project's Rust workspace, as explicitly
chosen by the user or established by a repository-wide contract.

On migration, update lib/bin/test paths, build working directories, Docker COPY/context
rules, documentation and callers together. Keep mirrored tests in `tests/` and outputs in
root `build/<component>/` and `dist/<component>/`. Cargo discovers `.cargo/config.toml`
from the invocation directory and its ancestors, not from `--manifest-path` alone; validate
the documented invocation's resolved package, workspace and target directories. Preserve
the lockfile's dependency resolution unless a dependency change is part of the task.

## Mobile applications

Keep Android and iOS platform projects in `src/android/` and `src/ios/` and reusable
application code in `src/shared/`. An implemented OCR backend belongs in
`src/backend_ocr/`, with the separate API backend in `src/backend_api/`; do not create an empty backend or introduce a server just to fill the
layout. Preserve other real platform components in their own source directories.

If a cross-platform tool requires platforms nested under a common application root,
assemble that tool-specific structure under root `build/` through repository-owned
scripts. Keep a single canonical source tree; do not retain `src/mobile/` as a duplicate
or compatibility directory. Update developer commands, dependency paths, mirrored tests,
Docker context/COPY rules, CI and docs, then validate the assembled build. Android APK
compilation remains inside Docker; iOS validation requires macOS/Xcode. Mobile workspace
preparation belongs in `build_android.sh` (with a preparation-only mode for iOS), and
checks belong in the relevant build workflow, not separate `prepare_mobile.sh` or
`run_*checks.sh` entrypoints. For combined Android/backend builds, `build_docker.sh`
calls `build_android.sh` and stages its APK for backend download; `build_ios.sh` remains
independent. See `docker-build` for the build and APK delivery contracts.

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
