---
name: refactoring
description: Refactor Python code following clean code principles. Use when the user asks to refactor code, break up long functions, improve code structure, reduce complexity, or apply clean code principles.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(isort:*), Bash(black:*)
---

# Code Refactoring

## What This Skill Does

Refactors Python code following clean code principles:
- Breaking up long functions
- Applying guard clauses
- Proper file organization
- Consistent patterns across the codebase

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

**DO NOT use broad try-except blocks.** Use guard clauses with early returns.

```python
# BAD - Swallows errors, unclear failure modes
def process_file(path: str) -> bool:
    try:
        data = read_file(path)
        return True
    except Exception as e:
        return False

# GOOD - Explicit validation, clear error messages
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

**When changing a pattern, update ALL similar code.**

- Search for similar implementations using Grep
- Apply changes consistently across the codebase
- Don't leave mixed patterns

```bash
# Find similar patterns before refactoring
grep -r "def process_" --include="*.py"
grep -r "try:" --include="*.py" -A 3
```

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

**When adding new features, ALWAYS search for similar existing code first.**

If similar functionality exists:
1. Extract the common logic into a shared function
2. Use parameters to handle differences between use cases
3. Have both old and new code call the shared function

```python
# BAD - Adding new feature with duplicated logic
# Existing code:
def get_active_users(database: Database) -> List[User]:
    connection = database.connect()
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM users WHERE status = 'active'")
    rows = cursor.fetchall()
    users = [User(**row) for row in rows]
    connection.close()
    return users

# New feature adds similar code - WRONG!
def get_premium_users(database: Database) -> List[User]:
    connection = database.connect()
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM users WHERE tier = 'premium'")
    rows = cursor.fetchall()
    users = [User(**row) for row in rows]
    connection.close()
    return users
```

```python
# GOOD - Extract common logic, parameterize differences
def _fetch_users(database: Database, where_clause: str) -> List[User]:
    """Internal helper for fetching users with custom filter."""
    connection = database.connect()
    cursor = connection.cursor()
    cursor.execute(f"SELECT * FROM users WHERE {where_clause}")
    rows = cursor.fetchall()
    users = [User(**row) for row in rows]
    connection.close()
    return users

def get_active_users(database: Database) -> List[User]:
    return _fetch_users(database, "status = 'active'")

def get_premium_users(database: Database) -> List[User]:
    return _fetch_users(database, "tier = 'premium'")

def get_users_by_status(database: Database, status: str) -> List[User]:
    return _fetch_users(database, f"status = '{status}'")
```

**More complex example with multiple differences:**

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

**Before adding any new feature:**

```bash
# Search for similar patterns in the codebase
grep -rn "similar_keyword" --include="*.py"
grep -rn "def.*similar_function" --include="*.py"

# Look for duplicated logic
grep -rn "repeated_code_pattern" --include="*.py" -A 5
```

**Signs you need to refactor:**
- Copy-pasting existing code to modify slightly
- Two functions that differ only in 1-2 lines
- Same sequence of operations in multiple places
- Similar error handling repeated across functions

## Refactoring Checklist

Before finishing refactoring:

1. [ ] No function exceeds ~100 lines
2. [ ] Guard clauses used instead of broad try-except
3. [ ] Public functions at top, private at bottom
4. [ ] All functions have type annotations
5. [ ] Similar patterns updated consistently
6. [ ] Each function has single responsibility
7. [ ] **No duplicated code** - similar logic extracted to shared functions
8. [ ] Imports organized (stdlib, third-party, local)
9. [ ] Code formatted with isort and black

## Post-Refactoring

**ALWAYS run formatters after refactoring:**

```bash
# Detect package directories first, then:
isort <package> <tests>
black -C --line-length 120 --skip-string-normalization <package> <tests>
```
