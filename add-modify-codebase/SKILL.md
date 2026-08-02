---
name: add-modify-codebase
description: Rules for adding to or modifying an existing codebase. Use whenever writing a new feature into existing code, changing a function signature, implementing something similar to existing code, or touching any file that already has callers. Enforces breaking changes over compatibility shims, refactoring over copy-paste, and unit tests for every touched code path.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash
---

# Adding To / Modifying An Existing Codebase

## What This Skill Does

Governs how changes land in code that already exists:
- Break the existing code on purpose instead of tiptoeing around it
- Refactor a shared implementation instead of copy-pasting a near-duplicate
- Add unit tests for every line added or modified
- Defer to the general coding skill and the project structure skill

## Rule 1: Make Breaking Changes, Not Minimal Ones

**Do not preserve the existing codebase's shape at the cost of clarity. Change the existing code so that every call site states its intent explicitly.**

The instinct to "keep the change small" produces hidden behavior: an old call site silently gets one behavior, the new call site gets another, and nothing in the code says so. Break the signature instead. The compiler, type checker, or test suite will list every place that needs a decision, and each one gets made deliberately.

### BAD — minimal change with a default value

```python
# Existing:
def render_report(rows):
    ...

# "Minimal" change: new arg gets a default so old callers keep working
def render_report(rows, include_totals=False):
    ...

# Only the new feature passes the non-default value:
render_report(rows, include_totals=True)

# Every pre-existing call is untouched and implicitly False:
render_report(rows)
```

Why this is wrong:
- The old call sites never declared they want `False`; they inherited it
- Nobody can tell from `render_report(rows)` what the totals behavior is
- The default becomes permanent — it never gets revisited
- Two behaviors coexist with no record of which call sites were reviewed

### GOOD — breaking change with explicit call sites

```python
# New arg, no default
def render_report(rows, include_totals):
    ...
```

```bash
# Find every caller
grep -rn "render_report(" --include="*.py"
```

```python
# Update each one explicitly
render_report(rows, include_totals=True)    # new feature
render_report(rows, include_totals=False)   # existing report, reviewed and kept as-is
render_report(rows, include_totals=False)   # existing export, reviewed and kept as-is
```

### Procedure

1. Change the signature / type / shape without a default or fallback
2. `grep` (or let the compiler / type checker) enumerate every call site
3. Visit each call site and pass the value explicitly — this is a decision, not a mechanical fill-in
4. Delete the old path entirely; no shim, no deprecated alias, no `if arg is None:` branch
5. Build and run the tests to prove no call site was missed

Applies equally to: renamed fields on a struct/DTO, changed return types, changed enum variants, changed config keys, changed API request/response shapes. Update producers, consumers, tests, fixtures, and generated types in the same change.

**Exception (narrow):** keep a compatibility path only for a concrete external constraint — persisted data on disk, a shipped public API contract, or a third-party consumer that cannot be updated in this change. Internal code is never such a constraint.

## Rule 2: Refactor To A Common Implementation, Never Copy-Paste

**When implementing something similar to existing code with a slight difference, do not copy the existing lines and tweak them. Extract the shared behavior first, then have both the old code and the new feature call it.**

### BAD — copy the existing function and tweak it

```rust
// Existing
pub async fn list_media(pool: &Pool, user_id: i64) -> Result<Vec<MediaResponse>, AppError> {
    let conn = pool.get()?;
    let rows = fetch_all(&conn, "SELECT * FROM media WHERE user_id = ?1", params![user_id])?;
    let mut out = Vec::new();
    for row in rows {
        out.push(build_media_response(row)?);
    }
    Ok(out)
}

// New feature: 25 lines copied, one WHERE clause changed
pub async fn list_favorite_media(pool: &Pool, user_id: i64) -> Result<Vec<MediaResponse>, AppError> {
    let conn = pool.get()?;
    let rows = fetch_all(
        &conn,
        "SELECT * FROM media WHERE user_id = ?1 AND is_favorite = 1",
        params![user_id],
    )?;
    let mut out = Vec::new();
    for row in rows {
        out.push(build_media_response(row)?);
    }
    Ok(out)
}
```

### GOOD — extract the common function, both callers use it

```rust
// Common implementation, extracted from the original
async fn query_media(pool: &Pool, user_id: i64, filter: MediaFilter) -> Result<Vec<MediaResponse>, AppError> {
    let conn = pool.get()?;
    let rows = fetch_all(&conn, &filter.to_sql(), params![user_id])?;
    let mut out = Vec::new();
    for row in rows {
        out.push(build_media_response(row)?);
    }
    Ok(out)
}

// Existing function, rewritten to delegate
pub async fn list_media(pool: &Pool, user_id: i64) -> Result<Vec<MediaResponse>, AppError> {
    query_media(pool, user_id, MediaFilter::All).await
}

// New feature, also delegating
pub async fn list_favorite_media(pool: &Pool, user_id: i64) -> Result<Vec<MediaResponse>, AppError> {
    query_media(pool, user_id, MediaFilter::Favorites).await
}
```

The same rule applies to classes: pull the shared behavior into a base class or a composed collaborator, and let both the existing type and the new one use it. Do not subclass just to redeclare copied methods.

### Procedure

1. Before writing the new code, search for the function/class you were about to copy from
2. Extract the shared body into a common function/class — this **modifies the existing code**, which is correct and expected
3. Rewrite the existing function to call the common one; verify its tests still pass
4. Write the new feature as a second thin caller of the common one
5. Keep the difference in one place — a parameter, a strategy object, or an overridden hook — not duplicated across both branches

**Threshold:** if you are about to duplicate more than a couple of lines, extract. "I'll deduplicate it later" is not an outcome that happens.

## Rule 3: Unit Tests For Every Touched Code Path

**Every time you finish a change, add unit tests for the code you added or modified. This is part of finishing the code, not a follow-up task.**

Required coverage:
- The new function / new branch, including its edge cases
- The refactored common implementation extracted under Rule 2
- The existing function after its signature broke under Rule 1, at each distinct explicit value now passed
- Error and guard-clause paths, not only the happy path

Follow the project's existing test framework and layout — do not introduce a second one. In this repo: `cargo test` for `src/api`, the configured frontend runner for `src/web`.

```bash
# Run the tests before declaring the change done
cd src/api && cargo test
```

A change is not complete until its tests exist and pass. Report the actual result; if a test fails, say so with the output.

## Rule 4: Follow The General Coding Skill

Every add / modify / delete follows the `general-coding` skill: uniform style across touched code, mandatory guard clauses, no broad try-except/catch, no default argument values, explicit caller updates, and no backward compatibility by default. Rules 1 and 2 above are the concrete procedure for its Principles 4–6.

## Rule 5: Follow The Project Structure Skill

Every add / modify / delete follows the `project-structure` skill: all source under `src/` split into `src/backend/` and `src/frontend/`, all tests under `tests/` mirroring `src/` 1:1, documentation in `docs/`, and end-to-end configs and data in `playground/`. A new source file means a new test file at the mirrored path in the same change.

Within that layout, follow the stack-specific layering and file placement rules:

- Rust / Axum backend → `axum-server`
- Python / FastAPI backend → `fastapi-server`
- C++ → `cpp-coding`
- Python OOP / architecture → `oop-code`
- Naming across all layers → `naming-conventions`
- API paths and methods → `restful-api-design`
- SQL → `sql-coding`

New files go where that structure says they go. Do not create a parallel directory tree or a new layering convention for one feature. If the repo has an `AGENTS.md` / `CLAUDE.md` describing structure, it wins over habit.

## Checklist

Before declaring an add/modify change complete:

1. [ ] New arguments / fields added **without** defaults or fallbacks
2. [ ] Every call site found via grep/compiler and updated to pass values explicitly
3. [ ] Old path deleted — no shim, alias, deprecated wrapper, or implicit-default branch left behind
4. [ ] Nothing was copy-pasted; shared behavior extracted into a common function/class
5. [ ] Existing code rewritten to call that common implementation, and still passing its tests
6. [ ] Unit tests added for every added/modified path, including error and edge cases
7. [ ] Full test suite run; results reported honestly
8. [ ] `general-coding` skill satisfied
9. [ ] `project-structure` skill satisfied — files under `src/`, tests mirrored under `tests/`, plus the stack-specific layering rules
