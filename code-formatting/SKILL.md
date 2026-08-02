---
name: code-formatting
description: Format Python code using isort and black. Use when the user asks to format code, fix imports, run formatters, or clean up Python files after making changes.
allowed-tools: Bash(isort:*), Bash(black:*), Read, Glob
---

# Code Formatting

## What This Skill Does

Formats Python code using:
- **isort**: Organizes imports into 3 sections (stdlib, third-party, local)
- **black**: Formats code with consistent style

## Before Formatting

1. Check for existing config in `pyproject.toml`, `setup.cfg`, or `.isort.cfg`
2. Format targets are `src/` and `tests/` — the layout owned by the `project-structure` skill

```bash
# Confirm the layout before formatting
ls -d src tests
```

If a repo predates that layout, detect its directories instead:

```bash
find . -name "__init__.py" -maxdepth 2
```

## Default Configuration

| Tool   | Setting                          | Value |
|--------|----------------------------------|-------|
| black  | Line length                      | 120   |
| black  | Skip string normalization        | Yes   |
| black  | Skip magic trailing comma (`-C`) | Yes   |
| isort  | Default profile                  | Yes   |

## Commands

### Format entire project

```bash
# Sort imports first
isort src tests

# Then format with black
black -C --line-length 120 --skip-string-normalization src tests
```

### Format a single file

```bash
isort path/to/file.py
black -C --line-length 120 --skip-string-normalization path/to/file.py
```

### Check without modifying

```bash
isort --check-only --diff <package>
black --check --diff <package>
```

## When to Run

**ALWAYS run formatting after:**
- Writing new Python code
- Modifying existing Python files
- Adding new imports
- Refactoring code

## Import Organization Rules

Imports must be organized into 3 sections separated by blank lines:

1. **Python standard library** (built-in modules)
2. **Third-party packages** (installed via pip)
3. **Local project imports** (from the current project)

Example:
```python
# Section 1: Standard library
import hashlib
import logging
from datetime import datetime
from typing import List, Optional

# Section 2: Third-party
import httpx
from fastapi import FastAPI

# Section 3: Local project
from myproject.config import Config
from myproject.database import Database
```

## Project-Specific Overrides

If the project has a `pyproject.toml` with `[tool.black]` or `[tool.isort]` sections, use those settings instead of the defaults above.

```bash
# Check for existing config
cat pyproject.toml | grep -A 10 "\[tool.black\]"
cat pyproject.toml | grep -A 10 "\[tool.isort\]"
```
