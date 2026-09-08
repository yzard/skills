---
name: naming-conventions
description: Introduce or rename domain concepts consistently across schemas, APIs, code, and configuration.
allowed-tools: Read, Write, Edit, Grep, Glob
---

# Naming conventions

Keep descriptive root words consistent across layers; change only the casing required by
the language or wire contract. For example, `created_at` in SQL/Rust and `createdAt` in a
camelCase JSON API describe the same concept. Do not impose a new casing on an existing API.

- Prefer domain names over opaque abbreviations or vague placeholders. Standard acronyms,
  `min`/`max`, and obvious mathematical coordinates or short loop indices are acceptable.
- Use the repository's filename and acronym casing. Do not rename established `HttpClient`
  or `mediaId` contracts to enforce an uppercase-acronym preference.
- When flattening configuration domains, prefix ambiguous fields with the source concept;
  keep the typed config, serialized fields, and callers aligned without compatibility aliases.
- A rename includes affected schema/DTO mappings, producers, consumers, tests, fixtures,
  and documentation. Check old spellings and serialization behavior, not just compilation.

Follow repository-specific layout migrations where required. Naming review is not a reason
to rename unrelated code or mechanically replace standard library/framework terminology.
