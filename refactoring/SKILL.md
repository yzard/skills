---
name: refactoring
description: Refactor Python code following clean code principles. Use when the user asks to refactor code, break up long functions, improve code structure, reduce complexity, or apply clean code principles.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(isort:*), Bash(black:*)
---

# Code Refactoring

## What This Skill Does

Refactors Python code following clean code principles:
- Breaking up long functions
- Proper file organization (public at top, private at bottom)
- Type annotations and single responsibility
- Python techniques for parameterizing extracted common logic

The cross-language rules this builds on — guard clauses, no broad try-except, uniform
patterns, no default arguments, DRY — are owned by the `general-coding` skill. The
procedure for landing a change (breaking signatures, extracting shared implementations,
unit tests for touched code) is owned by `add-modify-codebase`. This skill only adds
what is Python-specific.

## Key Refactoring Rules

### 1. Function Length

**Maximum ~100 lines per function.** If longer, extract helper functions.

```python
# BAD - Too long and does too many things
def process_items(self):
    # 150 lines of mixed logic...
    pass

# GOOD - Refactored into focused functions
def process_items(self) -> None:
    item = self._get_next_item()
    if not item:
        return

    candidates = self._fetch_candidates(item)
    selected = self._select_best(candidates)

    if selected:
        self._process_selected(selected)
        self._mark_complete(item)
    else:
        self._retry_later(item)

def _fetch_candidates(self, item: Item) -> List[Candidate]:
    # Focused logic for fetching
    pass

def _select_best(self, candidates: List[Candidate]) -> Optional[Candidate]:
    # Focused selection logic
    pass
```

### 2. Guard Clauses Over Try-Except

Owned by the `general-coding` skill, Principles 2 and 3 — guard clauses mandatory for
validation and early exits, no broad try-except.

The Python-specific addition here: each guard raises a **named exception class** and logs
before raising.

```python
class InvalidFilePathError(ValueError):
    """Raised when file path is invalid."""
    pass

def process_file(path: str) -> None:
    if not path:
        logging.error("process_file failed: path is empty")
        raise InvalidFilePathError("File path cannot be empty")

    if not os.path.exists(path):
        logging.error(f"process_file failed: file not found at {path}")
        raise InvalidFilePathError(f"File not found: {path}")

    if not os.path.isfile(path):
        logging.error(f"process_file failed: not a file: {path}")
        raise InvalidFilePathError(f"Path is not a file: {path}")

    # Process file - let exceptions propagate naturally
    data = read_file(path)
    process(data)
```

### 3. File Organization

**Public functions at TOP, private functions at BOTTOM.**

```python
# === Public API at top ===

def create_app(config: Config) -> Application:
    """Public - creates the application."""
    app = _initialize_app(config)
    _setup_routes(app)
    return app

def get_database(path: str) -> Database:
    """Public - returns database instance."""
    return Database(path)


# === Private helpers at bottom (with _ prefix) ===

def _initialize_app(config: Config) -> Application:
    """Internal helper."""
    return Application()

def _setup_routes(app: Application) -> None:
    """Internal helper."""
    app.include_router(router)
```

### 4. Consistent Patterns

Owned by the `general-coding` skill, Principle 1 — when changing a pattern, update ALL
similar code and never leave mixed patterns behind.

### 5. Type Annotations

**MANDATORY on all function arguments and return values.**

```python
# GOOD - Fully annotated
def process_download(identifier: str, retry_count: int = 3) -> bool:
    return True

def get_items(limit: Optional[int] = None) -> List[Item]:
    return []

# BAD - Missing annotations
def process_download(identifier, retry_count):
    return True
```

### 6. Single Responsibility

Each function should do ONE thing well.

```python
# BAD - Does multiple things
def handle_user(user_data: dict) -> None:
    # Validates, saves, sends email, logs...
    pass

# GOOD - Single responsibility
def validate_user(user_data: dict) -> User:
    pass

def save_user(user: User) -> None:
    pass

def send_welcome_email(user: User) -> None:
    pass
```

### 7. Eliminate Duplication When Adding Features

Owned by the `general-coding` skill, Principle 6 (the rule) and the `add-modify-codebase`
skill, Rule 2 (the extract-then-delegate procedure and the duplication threshold).

What this skill adds is the Python technique for **parameterizing the differences** once
they exceed a simple value — pass a `Callable` for the varying step rather than branching
inside the shared function:

```python
# BAD - Two functions with 80% similar code
def generate_monthly_report(data: ReportData) -> Report:
    report = Report()
    report.title = "Monthly Report"
    report.period = "month"
    # ... 50 lines of common formatting logic ...
    report.add_summary(calculate_totals(data))
    report.add_charts(create_monthly_charts(data))  # Different
    return report

def generate_quarterly_report(data: ReportData) -> Report:
    report = Report()
    report.title = "Quarterly Report"
    report.period = "quarter"
    # ... same 50 lines of common formatting logic ...
    report.add_summary(calculate_totals(data))
    report.add_charts(create_quarterly_charts(data))  # Different
    return report
```

```python
# GOOD - Common function with parameters for differences
from typing import Callable

def _generate_report(
    data: ReportData,
    title: str,
    period: str,
    chart_generator: Callable[[ReportData], List[Chart]]
) -> Report:
    """Internal helper for generating reports."""
    report = Report()
    report.title = title
    report.period = period
    # ... 50 lines of common formatting logic (written once) ...
    report.add_summary(calculate_totals(data))
    report.add_charts(chart_generator(data))
    return report

def generate_monthly_report(data: ReportData) -> Report:
    return _generate_report(
        data,
        title="Monthly Report",
        period="month",
        chart_generator=create_monthly_charts
    )

def generate_quarterly_report(data: ReportData) -> Report:
    return _generate_report(
        data,
        title="Quarterly Report",
        period="quarter",
        chart_generator=create_quarterly_charts
    )
```

## Refactoring Checklist

Before finishing refactoring:

1. [ ] No function exceeds ~100 lines
2. [ ] Guard clauses used for validation and early exits unless they would make the code harder to follow
3. [ ] Public functions at top, private at bottom
4. [ ] All functions have type annotations
5. [ ] Similar patterns updated consistently
6. [ ] Each function has single responsibility
7. [ ] **No duplicated code** - similar logic extracted to shared functions
8. [ ] Imports organized (stdlib, third-party, local)
9. [ ] Code formatted with isort and black

## Post-Refactoring

**ALWAYS run formatters after refactoring** — commands and settings are owned by the
`code-formatting` skill:

```bash
isort src tests
black -C --line-length 120 --skip-string-normalization src tests
```
