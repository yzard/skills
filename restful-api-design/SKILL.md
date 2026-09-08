---
name: restful-api-design
description: Define or review action-style API routes, HTTP methods, and protocol exceptions.
---

# API route conventions

This is an action-style project convention, not a claim that REST requires POST for reads.
Repository-defined contracts take precedence.

- Use `/api/v1/<component>/<operation>` with domain names matching existing components.
- Use POST for normal application operations, including queries under this convention.
- Preserve GET for media/static delivery and protocol handshakes, and protocol-specific
  methods such as WebDAV PUT/PATCH. Do not rewrite unrelated endpoints during a scoped task.
- Define validation, error status, retry/idempotency behavior, and access checks for the
  changed contract; update affected handlers, clients, docs, and tests together.
- Queries remain side-effect-free even with POST. The HTTP method does not select a database
  execution lane or replace authentication. Never put credentials in query strings.
