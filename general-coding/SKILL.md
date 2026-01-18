---
name: general-coding
description: General coding principles for all languages. Use when writing or modifying any code to ensure guard clauses, proper error handling, no unnecessary defaults, and DRY principles are followed.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# General Coding Principles

## What This Skill Does

Enforces consistent coding practices across all languages:
- Guard clauses for early returns
- No broad try-except/catch blocks
- No default argument values unless requested
- No backward compatibility unless requested
- DRY - eliminate code duplication

## Principle 1: Always Use Guard Clauses

**Use guard clauses with early returns instead of nested conditionals.**

Applies to: Python, JavaScript/TypeScript, C++, Go, Rust, Java, etc.

```python
# BAD - Deeply nested
def process_user(user):
    if user:
        if user.is_active:
            if user.has_permission:
                do_something(user)
                return True
    return False

# GOOD - Guard clauses
def process_user(user):
    if not user:
        return False
    if not user.is_active:
        return False
    if not user.has_permission:
        return False

    do_something(user)
    return True
```

```typescript
// BAD - Nested
function processOrder(order: Order | null): boolean {
    if (order) {
        if (order.isValid) {
            if (order.items.length > 0) {
                submitOrder(order);
                return true;
            }
        }
    }
    return false;
}

// GOOD - Guard clauses
function processOrder(order: Order | null): boolean {
    if (!order) return false;
    if (!order.isValid) return false;
    if (order.items.length === 0) return false;

    submitOrder(order);
    return true;
}
```

```cpp
// BAD - Nested
bool processData(Data* data) {
    if (data != nullptr) {
        if (data->isValid()) {
            if (!data->isEmpty()) {
                process(data);
                return true;
            }
        }
    }
    return false;
}

// GOOD - Guard clauses
bool processData(Data* data) {
    if (data == nullptr) return false;
    if (!data->isValid()) return false;
    if (data->isEmpty()) return false;

    process(data);
    return true;
}
```

## Principle 2: Avoid Broad Try-Except/Catch

**Never use broad exception handling. Catch specific exceptions or let them propagate.**

```python
# BAD - Swallows all errors
def read_config(path):
    try:
        with open(path) as f:
            return json.load(f)
    except Exception:
        return {}

# GOOD - Specific handling with guard clauses
def read_config(path):
    if not path:
        raise ValueError("Config path cannot be empty")
    if not os.path.exists(path):
        raise FileNotFoundError(f"Config file not found: {path}")

    with open(path) as f:
        return json.load(f)  # Let JSONDecodeError propagate
```

```typescript
// BAD - Catches everything
async function fetchData(url: string): Promise<Data | null> {
    try {
        const response = await fetch(url);
        return await response.json();
    } catch (e) {
        return null;
    }
}

// GOOD - Specific handling
async function fetchData(url: string): Promise<Data> {
    if (!url) throw new Error("URL cannot be empty");

    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`HTTP error: ${response.status}`);
    }
    return await response.json();
}
```

```cpp
// BAD - Catches everything
std::optional<Data> readData(const std::string& path) {
    try {
        return parseFile(path);
    } catch (...) {
        return std::nullopt;
    }
}

// GOOD - Let exceptions propagate or handle specifically
Data readData(const std::string& path) {
    if (path.empty()) {
        throw std::invalid_argument("Path cannot be empty");
    }
    if (!std::filesystem::exists(path)) {
        throw std::runtime_error("File not found: " + path);
    }
    return parseFile(path);  // Let parsing errors propagate
}
```

## Principle 3: No Default Argument Values

**Function arguments must not have default values unless explicitly requested.**

When adding new arguments to existing functions:
1. Do NOT add default values
2. Find ALL callers of the function
3. Update ALL callers to pass the new argument explicitly

```python
# BAD - Adding a new argument with default
def process_items(items, max_count=100):  # DON'T add defaults
    pass

# GOOD - No defaults, update all callers
def process_items(items, max_count):
    pass

# Then find and update ALL callers:
# Before: process_items(my_items)
# After:  process_items(my_items, 100)
```

```typescript
// BAD - Default value
function sendEmail(to: string, subject: string, priority: string = "normal") {}

// GOOD - No default, update all callers
function sendEmail(to: string, subject: string, priority: string) {}

// Update callers:
// Before: sendEmail("user@example.com", "Hello")
// After:  sendEmail("user@example.com", "Hello", "normal")
```

```cpp
// BAD - Default parameter
void connect(const std::string& host, int port, int timeout = 30) {}

// GOOD - No default
void connect(const std::string& host, int port, int timeout) {}

// Update all callers to pass timeout explicitly
```

**Process for adding new arguments:**
1. Add the new argument WITHOUT a default value
2. Use grep/search to find all existing callers
3. Update each caller to pass an appropriate value
4. Compile/test to ensure no callers were missed

## Principle 4: No Backward Compatibility

**Do not provide backward compatibility unless explicitly requested.**

When refactoring or changing APIs:
- Remove old code completely
- Do not keep deprecated versions
- Do not add compatibility shims
- Do not re-export for compatibility
- Do not rename to `_unused_*` or similar

```python
# BAD - Keeping old function for compatibility
def old_function(x):  # Deprecated
    return new_function(x, default_y)

def new_function(x, y):
    pass

# GOOD - Just the new function, delete the old one entirely
def new_function(x, y):
    pass
# old_function is deleted, callers must update
```

```typescript
// BAD - Re-exporting for compatibility
export { NewClass as OldClass };  // Don't do this
export const oldFunction = newFunction;  // Don't do this

// GOOD - Only export the new API
export { NewClass };
export { newFunction };
// Callers must update their imports
```

## Principle 5: Refactor Similar Functions

**When adding or changing a function, search for similar functions and consolidate them.**

Before adding new code:
1. Search for similar existing functions
2. If found, extract common logic into a shared function
3. Have all similar functions call the shared implementation

```python
# BAD - Multiple similar functions
def get_active_users(db):
    conn = db.connect()
    result = conn.execute("SELECT * FROM users WHERE status = 'active'")
    users = [User(**row) for row in result]
    conn.close()
    return users

def get_premium_users(db):  # 90% duplicate code
    conn = db.connect()
    result = conn.execute("SELECT * FROM users WHERE tier = 'premium'")
    users = [User(**row) for row in result]
    conn.close()
    return users

# GOOD - Shared implementation
def _fetch_users(db, where_clause):
    conn = db.connect()
    result = conn.execute(f"SELECT * FROM users WHERE {where_clause}")
    users = [User(**row) for row in result]
    conn.close()
    return users

def get_active_users(db):
    return _fetch_users(db, "status = 'active'")

def get_premium_users(db):
    return _fetch_users(db, "tier = 'premium'")
```

**How to identify similar functions:**
```bash
# Search for similar patterns before adding code
grep -rn "def.*_users" --include="*.py"
grep -rn "SELECT \* FROM users" --include="*.py"
```

**Signs you need to consolidate:**
- Copy-pasting existing code to modify slightly
- Functions that differ in only 1-2 lines
- Same sequence of operations in multiple places

## Checklist

Before completing any code change:

1. [ ] Guard clauses used for all validation/early returns
2. [ ] No broad try-except/catch blocks
3. [ ] No default argument values (unless explicitly requested)
4. [ ] Old code deleted completely (no backward compatibility)
5. [ ] Similar functions consolidated into shared implementation
6. [ ] All callers updated when function signatures change
