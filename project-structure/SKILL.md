---
name: project-structure
description: Directory layout rules for every project regardless of language. Use when creating a new file, adding a module, writing tests, adding documentation, dockerizing a project, setting up a new project, wiring up a build script, or deciding where any file belongs. Enforces src/ for all source, tests/ mirroring src/ 1:1, src/backend and src/frontend split, docs/ for documentation, playground/ for end-to-end config and data run by run_playground.sh, build/ and dist/ at the git root for intermediate and final build artifacts, and docker/ for all Docker files driven by build_docker.sh at the git root.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Project Structure

## What This Skill Does

Defines where every file goes, in every project, in every language:
- All source code under `src/`
- All tests under `tests/`, mirroring `src/` exactly
- Backend under `src/backend/`, frontend under `src/frontend/`
- All documentation under `docs/`
- All end-to-end config and data under `playground/`, run by `run_playground.sh` at the git root
- All intermediate build artifacts under `build/` and all final build products under `dist/`, both at the git root
- All Docker files under `docker/`, driven by `build_docker.sh` at the git root

## The Canonical Layout

```
<git root>/
├── src/
│   ├── backend/          # all server-side source, any language
│   └── frontend/         # all client-side source, any language
├── tests/
│   ├── backend/          # mirrors src/backend/ 1:1
│   └── frontend/         # mirrors src/frontend/ 1:1
├── docs/                 # all documentation
├── playground/
│   ├── config.yaml       # the config run_playground.sh starts the service with
│   ├── data/             # service data directory
│   ├── upload/           # service upload directory
│   └── output/           # run-time output: logs, generated reports
├── build/                # intermediate build artifacts (compiler caches, work trees)
├── dist/                 # final build products: binaries, bundled frontend assets
├── docker/               # Dockerfile, docker-compose.yml, .dockerignore, entrypoint.sh
├── build_docker.sh       # cds into docker/ and builds
├── run_playground.sh     # builds into build/ and dist/, starts the service against playground/
└── <build/tooling files> # package manifests, lockfiles, CI config
```

Exactly two scripts belong at the git root: `build_docker.sh` and `run_playground.sh`.
Everything else executable lives in `src/`.

## Rule 1: All Source Code Lives In `src/`

**Every source file, in every language, goes under `src/`. No exceptions for language, tooling convention, or "it's just one file."**

Rust, Python, TypeScript, C++, Go, SQL, shell scripts — all of it goes under `src/`. Do not scatter language-specific top-level directories at the repo root.

```
# BAD — language-specific roots scattered at top level
api/
web/
lib/
scripts/
server.py
```

```
# GOOD — one source root
src/
├── backend/
└── frontend/
```

Only build and tooling metadata belongs at the git root: package manifests (`Cargo.toml`, `package.json`, `pyproject.toml`), lockfiles, CI config, `.gitignore`, `README.md`, `AGENTS.md`, and the entry-point scripts `build_docker.sh` (Rule 6) and `run_playground.sh` (Rule 5). Docker files themselves go in `docker/`.

## Rule 2: Backend And Frontend Are Separate Top-Level Trees

**Server-side code goes in `src/backend/`. Client-side code goes in `src/frontend/`. The same split applies to `tests/backend/` and `tests/frontend/`.**

```
src/
├── backend/
│   ├── routes/
│   ├── models/
│   ├── database/
│   └── main.rs
└── frontend/
    ├── components/
    ├── pages/
    ├── hooks/
    └── api/

tests/
├── backend/
│   ├── routes/
│   ├── models/
│   └── database/
└── frontend/
    ├── components/
    ├── pages/
    └── hooks/
```

Code shared between the two — type definitions, protocol schemas, generated clients — goes in `src/shared/`, with `tests/shared/` mirroring it. Do not reach across from `src/frontend/` into `src/backend/` or vice versa; share through `src/shared/`.

## Rule 3: `tests/` Mirrors `src/` 1:1

**The `tests/` tree is an exact structural mirror of the `src/` tree. Every source file has a predictable test location derived purely from its source path.**

The mapping is mechanical: replace the leading `src/` with `tests/`, keep every intermediate directory, and apply the language's test-file naming convention to the filename.

```
src/backend/routes/media.rs        →  tests/backend/routes/media.rs
src/backend/database/pool.py       →  tests/backend/database/test_pool.py
src/frontend/components/Card.tsx   →  tests/frontend/components/Card.test.tsx
src/shared/types/media.ts          →  tests/shared/types/media.test.ts
```

```
# BAD — flat test directory, no mirror
tests/
├── test_media.py
├── test_pool.py
└── test_everything_else.py

# BAD — tests colocated next to source
src/backend/routes/media.rs
src/backend/routes/media_test.rs

# GOOD — structural mirror
tests/backend/routes/media.rs
tests/backend/database/test_pool.py
```

Rules that follow from the mirror:
- Creating `src/<path>/<file>` means creating `tests/<path>/<test file>` in the same change
- Moving or renaming a source file means moving or renaming its test file identically
- Deleting a source file means deleting its test file
- A directory that exists in `src/` but not in `tests/` is a coverage gap — treat it as one

**Language-specific caveat:** some toolchains default to colocated or in-file tests (Rust `#[cfg(test)]` modules, Go `_test.go` siblings, Vitest colocation). Configure the toolchain to use the mirrored `tests/` tree instead of accepting the default. In Rust that means integration tests under `tests/` with the crate's public API exercised from outside; keep unit tests that genuinely need private access minimal and prefer widening visibility for testability over breaking the mirror.

## Rule 4: All Documentation Lives In `docs/`

**Every documentation file goes under `docs/`.** Design notes, architecture, API references, runbooks, ADRs, guides, diagrams.

```
docs/
├── architecture.md
├── api/
│   └── endpoints.md
├── runbooks/
│   └── deployment.md
└── decisions/
    └── 0001-sqlite-over-postgres.md
```

The only documentation permitted at the repo root is `README.md` (entry point, links into `docs/`) and agent instruction files (`AGENTS.md`, `CLAUDE.md`). Do not leave stray `NOTES.md`, `TODO.md`, or `DESIGN.md` files scattered in source directories.

## Rule 5: End-To-End Configs And Data Live In `playground/`, Run By `run_playground.sh`

**Anything a full end-to-end run needs — config, seed data, fixtures, sample media, generated databases, uploads — goes under `playground/`, and `run_playground.sh` at the git root starts the service against it.**

```
<git root>/
├── run_playground.sh     # builds, then starts the service against playground/
├── build/                # intermediate build artifacts — never run from here
├── dist/                 # the binaries and bundles run_playground.sh executes
└── playground/
    ├── config.yaml       # the one config file the service reads
    ├── data/             # database files, seed data, service working data
    ├── upload/           # upload destination, if the service accepts uploads
    └── output/           # run-time output: logs, generated reports
```

### Fixed paths

- **`playground/config.yaml`** — a single config file at the top of `playground/`, not a `config/` subdirectory. The extension follows whatever format the service actually parses: `config.yaml`, `config.json`, `config.toml`. One file, one name, so `run_playground.sh` never has to guess.
- **`playground/data/`** — every data directory the service is configured with points here. Databases, seed data, working files.
- **`playground/upload/`** — if the service accepts uploads, its upload directory points here. Do not nest it inside `data/`; uploads are user-supplied input and get wiped independently.
- **`playground/output/`** — output the *running service* produces: logs, generated reports. Not build artifacts — those go to `build/` and `dist/` at the git root.

`playground/config.yaml` is checked in — it is the documented way to run the service locally. `data/`, `upload/`, and `output/` are gitignored.

### Build artifacts go to `build/` and `dist/` at the git root

**Nothing `run_playground.sh` compiles is written into `playground/`, and nothing is written into `src/`. Intermediate build artifacts go to `<git root>/build/`; final build products go to `<git root>/dist/`.**

The split is by lifetime, not by producer:

- **`build/`** — everything the toolchain needs while building and nothing anyone runs afterwards: compiler caches and object files (`CARGO_TARGET_DIR`, `.o`, `.tsbuildinfo`), staged work trees, intermediate bundler output. Deleting it costs a rebuild and nothing else.
- **`dist/`** — what actually gets executed or served: compiled binaries, the bundled frontend, anything a Dockerfile or deployment would ship. `run_playground.sh` starts the service from `dist/`, never from `build/` and never from a toolchain default like `src/backend/target/release/`.

Give each component its own subdirectory under both, named for the component:

```
<git root>/
├── build/
│   ├── backend/          # CARGO_TARGET_DIR for the backend crate
│   ├── llm/              # CARGO_TARGET_DIR for the LLM service crate
│   └── frontend/         # staged pnpm workspace, Vite intermediates
└── dist/
    ├── backend/          # momento-api
    ├── llm/              # llm-service
    └── frontend/         # index.html, assets/ — what static_dir points at
```

```
# BAD — build output buried under playground/
playground/output/build/backend/target/
playground/output/dist/backend/momento-api

# BAD — running out of the toolchain's default location
src/backend/target/release/momento-api

# GOOD — one build root, one dist root, at the git root
build/backend/target/
dist/backend/momento-api
```

Both directories are generated, so both are gitignored (`/build/` and `/dist/` — anchor
the patterns to the root so nested `dist/` directories inside `src/` are still caught by
whatever rules already cover them). `run_playground.sh` removes the component
subdirectories it is about to write before building, so a run never picks up a stale
binary from a previous one.

A service config that points at built assets — a frontend `static_dir`, a template
directory — points into `dist/`, not into `build/` or a toolchain default path.

### `run_playground.sh`

Lives at the git root, is executable, resolves paths from its own location so it works
from any working directory, and creates the runtime directories before starting:

```bash
#!/bin/bash
set -euo pipefail

ROOT="$(cd "$(dirname "$0")" && pwd)"
BUILD_DIR="$ROOT/build/backend"
DIST_DIR="$ROOT/dist/backend"

mkdir -p "$ROOT/playground/data" "$ROOT/playground/upload" "$ROOT/playground/output"

rm -rf "$BUILD_DIR" "$DIST_DIR"
mkdir -p "$DIST_DIR"

CARGO_TARGET_DIR="$BUILD_DIR/target" \
    cargo build --release --manifest-path "$ROOT/src/backend/Cargo.toml"
cp "$BUILD_DIR/target/release/<binary>" "$DIST_DIR/<binary>"

exec "$DIST_DIR/<binary>" --config "$ROOT/playground/config.yaml"
```

Same shape for other stacks — the build step writes into `build/`, the artifact lands in
`dist/`, and the exec line runs the thing in `dist/`:

```bash
pnpm --dir "$ROOT/src/frontend" build --outDir "$DIST_DIR"
```

When the project also has a frontend dev server, `run_playground.sh` starts that too —
one script, one name, whatever the stack. Do not add a second `run_*.sh` at the root.

### Requirements

- The service reads its config path from an argument or environment variable — never a hardcoded `playground/` path inside `src/`. The dependency points one way: `playground/` → nothing, `run_playground.sh` → both.
- `playground/config.yaml` points the service's data and upload settings at `playground/data/` and `playground/upload/`, and any built-asset setting at `dist/`
- Compilation writes into `<git root>/build/`, artifacts land in `<git root>/dist/`, and the service is started from `dist/` — nothing built is written under `playground/` or `src/`
- Runtime-generated content under `playground/` is gitignored; the config file is not
- Unit test fixtures small enough to live beside the test stay in `tests/`; only end-to-end scale configs and data go to `playground/`

## Rule 6: All Docker Files Live In `docker/`, Built By `build_docker.sh`

**Every Docker-related file goes under `<git root>/docker/` — `Dockerfile`, `docker-compose.yml`, `.dockerignore`, `entrypoint.sh`, and any compose overrides. The single exception is `<git root>/build_docker.sh`, which cds into `docker/` and runs the build.**

```
<git root>/
├── build_docker.sh       # the only Docker-related file at the root
└── docker/
    ├── Dockerfile
    ├── Dockerfile.dockerignore
    ├── docker-compose.yml
    ├── docker-compose.dev.yml
    └── entrypoint.sh
```

```
# BAD — Docker files scattered at the root
Dockerfile
docker-compose.yml
.dockerignore
entrypoint.sh
build.sh
```

### `build_docker.sh`

Lives at the git root, is executable, and enters `docker/` before building so the script works from any working directory:

```bash
#!/bin/bash
set -euo pipefail

cd "$(dirname "$0")/docker"
docker build -f Dockerfile -t "<image>:latest" ..   # `..` = git root as build context
```

The full script — tagging, registry push, and the `Dockerfile` / `entrypoint.sh` /
ignore-file templates — belongs to the `docker-build` skill. This rule owns only
*where* those files go.

### Path consequences of moving Docker out of the root

Moving these files changes how every relative path in them resolves. Get all four right:

1. **Build context stays the git root.** The Dockerfile must `COPY src/ ...`, not `COPY ../src/ ...` — Docker refuses paths outside the context. So the build passes `..` as the context with `-f Dockerfile`, as above.

2. **`.dockerignore` must be named `Dockerfile.dockerignore`.** Docker resolves `.dockerignore` relative to the *build context*, not the Dockerfile — a plain `docker/.dockerignore` would be silently ignored when the context is the git root. BuildKit looks for `<dockerfile-name>.dockerignore` beside the Dockerfile first, so `docker/Dockerfile.dockerignore` is the form that keeps the file in `docker/` and actually takes effect. Requires BuildKit (the default in current Docker); if a build must run with BuildKit disabled, the ignore file has to sit at the context root instead.

3. **Compose paths resolve relative to the compose file.** `docker/docker-compose.yml` must declare the context explicitly:
   ```yaml
   services:
     app:
       build:
         context: ..
         dockerfile: docker/Dockerfile
       volumes:
         - ../playground/data:/data
   ```

4. **CI and docs must point at the new paths** — any `docker build`, `docker compose -f`, or published run instructions get updated in the same change.

## Applying This To An Existing Repo

When a repo already violates this layout, do not build the new feature into the wrong place to match what is there. Follow the `add-modify-codebase` skill: move the affected tree to the canonical layout, update every import path, build config, and CI reference, then land the feature in the right location. Partial migrations that leave two parallel structures are worse than either structure alone.

If the repo's `AGENTS.md` / `CLAUDE.md` documents a different layout, the documented layout wins for that repo — and the mismatch is worth raising with the user rather than silently working around.

## Checklist

Before adding any file:

1. [ ] Source file is under `src/`, not at the repo root or in a language-specific top-level directory
2. [ ] Correctly placed in `src/backend/`, `src/frontend/`, or `src/shared/`
3. [ ] Its test file exists at the mirrored path under `tests/`, with the same intermediate directories
4. [ ] Test toolchain configured to run from `tests/`, not colocated defaults
5. [ ] Documentation is in `docs/`, not scattered as loose markdown in source directories
6. [ ] E2E config is `playground/config.{yaml,json,toml}`, data in `playground/data/`, uploads in `playground/upload/`, and nothing in `src/` hardcodes those paths
7. [ ] `run_playground.sh` at the git root starts the service against that config, creating `data/` and `upload/` first
8. [ ] Intermediate build artifacts go to `<git root>/build/<component>/`, final artifacts to `<git root>/dist/<component>/`, and the service runs from `dist/` — nothing compiled is written under `playground/` or `src/`
9. [ ] `/build/` and `/dist/` are gitignored; generated `playground/` content is gitignored; `playground/config.*` is checked in
10. [ ] Moves and renames applied to `src/` and `tests/` together, keeping the mirror intact
11. [ ] All Docker files in `docker/`; only `build_docker.sh` and `run_playground.sh` sit at the git root
12. [ ] `build_docker.sh` cds into `docker/`, passes `..` as the build context, and is executable
13. [ ] Ignore file named `docker/Dockerfile.dockerignore` so the root build context honors it
14. [ ] Compose files declare `context: ..` and `dockerfile: docker/Dockerfile`; CI and docs updated
