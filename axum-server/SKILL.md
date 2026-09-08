---
name: axum-server
description: Build or modify Axum handlers and service infrastructure using JWT authentication and TOML configuration.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(cargo:*)
---

# Axum services

Use JWT authentication and TOML configuration. Keep construction in an application factory,
startup in `main.rs`, and dependencies in typed application state. Follow the crate's installed
Axum version and repository-owned execution model rather than copying a server template.

- Parse the required CLI config path into typed TOML settings and validate ranges and
  cross-field constraints. Missing/malformed required config fails startup. Use the existing
  config manager for live snapshots; do not retain stale copies or introduce YAML config.
- Authenticate user API requests with `Authorization: Bearer <JWT>`, through the shared
  authentication extractor. Do not introduce an in-memory session-token map or `X-Auth-Token`.
  Preserve deliberately separate login, service API-key, WebDAV, and WebSocket contracts.
- Handlers extract typed input, enforce resource access, and call business operations.
  Keep body-consuming extractors last and preserve streaming/backpressure.
- Map errors through `Result<T, AppError>` and the existing `IntoResponse` implementation.
  Distinguish absent data from infrastructure failure; keep internal diagnostics out of
  public responses.
- Log method, path, status, and duration. Additional body fields must be bounded and safe
  to log; never buffer a streaming body just for logging or log credentials/tokens.
- Keep blocking SQL, filesystem work, hashing, and image work off network threads. Use the
  repository's bounded scheduler/executor domains; do not add private pools or synchronous
  connection checkout inside async handlers.
- Put SQL in centralized resource query modules with explicit transaction ownership.
  Follow the repository's API routes, module hierarchy, and mirrored test locations.
- Background work uses the established durable lifecycle, cancellation, retry, and recovery
  owners. Adding an endpoint is not permission to replace that architecture.

For authentication or configuration implementation, read
[JWT and TOML contracts](references/auth-config.md). A change to an unrelated handler does
not require that reference or a service-wide authentication/configuration rewrite.

Verify the changed behavior and its relevant authentication, validation, and error paths.
Use the repository's registered tests, formatter, and appropriate Rust checks; do not run
an entire service test matrix for a documentation edit.
