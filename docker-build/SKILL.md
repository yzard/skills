---
name: docker-build
description: Create or change Python/Rust Docker images, entrypoints, and build or publish scripts.
allowed-tools: Write, Read, Bash, Glob, Grep
---

# Docker builds

Use repository-owned build scripts and mandatory platform entrypoints. Preserve existing
runtime/deployment contracts; a generic template is not a reason to replace them.

- Default layout is `docker/` for image files and root `build_docker.sh`, with repository
  exceptions respected. Build context stays at the git root; COPY uses paths inside it.
- Build scripts resolve their own location, validate arguments, and fail on command errors.
  Local build is the default. Publishing requires an explicit publish command/flag and user
  authorization; do not add an unconditional registry push to a build script.
- Preserve the project's tag scheme. Date tags plus latest are an option for a new script,
  not a reason to change an existing release contract.
- Cache dependency installation separately from source changes; use lockfiles and the
  project's toolchain. Choose a base compatible with native/GPU dependencies rather than
  mandating Alpine. Preserve existing Python uv/pip/virtual-environment decisions.
- Inspect actual binary linkage and include required runtime libraries, certificates, and
  timezone data. Installing static libraries does not prove a binary is statically linked.
- Use Dockerfile-specific ignore files beside each Dockerfile. Exclude secrets/runtime data
  while retaining required COPY inputs, including entrypoints. Compose bind/context paths
  resolve relative to the Compose file; update build/CI/docs when moving them.

When changing identity, configuration initialization, or process startup/shutdown, read
[entrypoint contracts](references/entrypoints.md). Ordinary image dependency edits do not
require redesigning the entrypoint.

Validate the changed script/config/image path and relevant startup behavior. Complete local
checks within the authorized task scope. Build instructions alone do not authorize deploying,
publishing, or modifying an existing data volume.
