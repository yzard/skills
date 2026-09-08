---
name: sql-coding
description: Write or change raw SQL schemas, queries, and transactions using the target database dialect.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# SQL

Use the repository's schema policy and database dialect. Keep raw SQL in centralized resource
query modules; do not introduce an ORM or replace an unrelated persistence layer.

- Current tables, indexes, constraints, and triggers belong in the canonical `schema.sql`.
  `CREATE TABLE IF NOT EXISTS` does not migrate existing columns or constraints.
- Follow the actual migration policy. If migrations are required, keep one logical migration
  in one SQL file and update the current schema too. Do not add migration scaffolding to a
  current-schema-only repository or discard existing user data to apply a schema change.
- Application code imports named query constants; do not scatter SQL across handlers.
- Bind data values using the driver's placeholder syntax. Dynamic identifiers and ordering
  clauses come from a closed validated mapping, never untrusted string concatenation.
- Use the target engine's actual boolean, ID, aggregation, conflict, and type semantics;
  SQLite, PostgreSQL, and DuckDB are not interchangeable dialects.
- Keep connection/transaction ownership explicit. Preserve foreign keys, uniqueness,
  access filtering, and deterministic ordering before pagination. A read that claims or
  cleans work is a write; use the repository's correct executor/transaction lane.

## Formatting

Use UPPERCASE keywords and snake_case identifiers. Multiline SELECT/INSERT/UPDATE lists use
leading commas; CREATE TABLE definitions use trailing commas, except on the final definition.
Match surrounding indentation.

```sql
SELECT user_id
     , username
  FROM users
 WHERE is_active = ?1
 ORDER BY username
        , user_id
```

Verify changed queries against the configured engine, covering relevant nulls, ordering,
access boundaries, and transactional failure behavior. Inspect query plans when changing
performance-sensitive queries; do not require them for every SQL edit.
