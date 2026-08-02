---
name: general-coding
description: General coding principles for all languages. Use when writing or modifying any code to enforce uniform codebase style, guard clauses, proper error handling, no unnecessary defaults, explicit caller updates, no default backward compatibility, and no environment variables in source code (configuration comes from CLI flags or config.yaml).
allowed-tools: Read, Edit, Write, Glob, Grep
---

# General Coding Principles

## What This Skill Does

Enforces consistent coding practices across all languages:
- Mandatory codebase-wide uniformity for touched logic and style
- Guard clauses are mandatory for validation and early returns unless they would make the code harder to follow
- No broad try-except/catch blocks
- No default argument values unless requested
- No backward compatibility unless explicitly requested or required by a concrete external constraint
- DRY - eliminate code duplication
- No environment variables in source code — configuration comes from the command line, or from a `config.yaml` once there are more than 5 settings

## Principle 1: Keep Touched Code Uniform

**When modifying code or adding features, it is mandatory to keep the codebase's coding logic and style uniform.**

Do not introduce one-off patterns, compatibility branches, partial migrations, or isolated calling conventions. If a change touches an existing pattern, update the touched area to use one unified style.

When changing behavior or APIs:
1. Search for the existing pattern before adding new code
2. Match the dominant style unless it is clearly wrong
3. If the current pattern must change, change all affected call sites and related code together
4. Prefer a complete migration over parallel old/new styles
5. Compile/test to catch any missed inconsistencies

Examples of required uniformity:
- If a function signature changes, update every caller explicitly
- If request/response shape changes, update all producers, consumers, and tests that use it
- If a helper replaces repeated logic, remove the duplicate variants in the touched area
- If a naming or argument-order pattern is corrected, apply it consistently across related code

## Principle 2: Guard Clauses Are Mandatory

**Use guard clauses with early returns instead of nested conditionals. Guard clauses are mandatory unless applying them would make the code harder to follow or require awkward restructuring.**

Prefer guard clauses for validation, preconditions, authorization checks, empty states, error states, and other early exits. Keep a small conditional at the end only when converting it into a guard clause would reduce readability.

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

## Principle 3: Avoid Broad Try-Except/Catch

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

## Principle 4: No Default Argument Values

**Function arguments must not have default values unless explicitly requested or required by the language/framework contract.**

When adding new arguments to existing functions:
1. Do NOT add default values
2. Find ALL callers of the function
3. Update ALL callers to pass the new argument explicitly
4. Do NOT only update the caller that needs the new behavior

The bad approach is adding a default value so old callers silently keep working while only one caller uses the new argument. That creates hidden behavior differences and makes the codebase harder to maintain. The preferred approach is to break the signature, let compile/type/test failures expose missed call sites, and make every caller pass the value intentionally.

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

**Process for adding new arguments:** the `add-modify-codebase` skill, Rule 1, owns the
step-by-step procedure for breaking a signature and updating every call site.

## Principle 5: Break Compatibility By Default

**Do not provide backward compatibility by default. Break compatibility when it keeps the codebase simpler, more uniform, and easier to maintain.**

Compatibility layers are allowed only when there is a concrete reason, such as persisted data, shipped external API contracts, migration windows explicitly requested by the user, or third-party consumers that cannot be updated in the same change. If that reason is not present, remove the old path and update all affected code.

When refactoring or changing APIs:
- Remove old code completely
- Do not keep deprecated versions
- Do not add compatibility shims
- Do not re-export for compatibility
- Do not rename to `_unused_*` or similar
- Do not keep old and new coding patterns side by side
- Update all internal callers, tests, docs, and generated types touched by the change

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

## Principle 6: Refactor Similar Functions

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

**Signs you need to consolidate:**
- Copy-pasting existing code to modify slightly
- Functions that differ in only 1-2 lines
- Same sequence of operations in multiple places

**Procedure:** the `add-modify-codebase` skill, Rule 2, owns the extract-then-delegate
steps and the duplication threshold.

## Principle 7: No Environment Variables In Source Code

**Source code must not read environment variables unless the user explicitly asks for it.**

No `os.environ` / `std::env::var` / `process.env` reads scattered through the code. An
environment variable is an invisible input: it does not appear in any signature, has no
type, has no default the reader can see, and cannot be traced to a caller. Configuration
must arrive through an explicit, inspectable path instead.

```python
# BAD - Reads the environment deep inside the code
def get_database(url: str) -> Database:
    timeout = int(os.environ.get("DB_TIMEOUT", "30"))  # Invisible input
    return Database(url, timeout)

# GOOD - The value is a parameter, supplied by the caller
def get_database(url: str, timeout: int) -> Database:
    return Database(url, timeout)
```

```rust
// BAD
let port = std::env::var("PORT").unwrap_or_else(|_| "8000".to_string());

// GOOD - port comes from the parsed config
let port = config.server.port;
```

### Where Configuration Comes From

**Every executable takes its configuration from the command line.** Parse arguments once
at the entry point (`main`), build a config value, and pass it down explicitly.

**Up to 5 configuration values: individual command-line flags.**

```bash
./server --port 8000 --data-dir /data --log-level info
```

**More than 5 configuration values: a `config.yaml`, passed as a single flag.**

```bash
./server --config /path/to/config.yaml
```

```yaml
# config.yaml
server:
  host: 0.0.0.0
  port: 8000
storage:
  data_dir: /data
  upload_dir: /upload
logging:
  level: info
```

The config file is parsed into a typed struct/dataclass at startup. Do not mix the two
styles — once a config file exists, new settings go into it, not into new flags.

### The Docker Exception

The **Dockerfile and entrypoint script are the one place environment variables are
expected**, because that is how container runtimes pass configuration in. The container
layer translates environment variables into the executable's real interface, one of two
ways:

```dockerfile
# Option 1: env vars become command-line flags
ENV PORT=8000 DATA_DIR=/data LOG_LEVEL=info
CMD ["sh", "-c", "exec /app/server --port \"$PORT\" --data-dir \"$DATA_DIR\" --log-level \"$LOG_LEVEL\""]
```

```bash
# Option 2 (docker/entrypoint.sh): env vars are rendered into a config.yaml
cat > /data/config.yaml <<EOF
server:
  host: 0.0.0.0
  port: ${PORT}
storage:
  data_dir: ${DATA_DIR}
  upload_dir: ${UPLOAD_DIR}
logging:
  level: ${LOG_LEVEL}
EOF

exec /app/server --config /data/config.yaml
```

Option 2 is the right one whenever the executable already uses a config file. Either way,
the executable itself never calls `getenv` — it sees only flags or a config path. Docker
file contents are owned by the `docker-build` skill.

## Checklist

Before completing any code change:

1. [ ] Guard clauses used for validation and early returns unless they would make the code harder to follow
2. [ ] No broad try-except/catch blocks
3. [ ] Touched code follows one uniform logic/style pattern
4. [ ] No default argument values (unless explicitly requested or required by a contract)
5. [ ] All callers updated explicitly when function signatures change
6. [ ] Old code deleted completely (no default backward compatibility)
7. [ ] Similar functions consolidated into shared implementation
8. [ ] No environment variable reads in source code (Dockerfile and entrypoint excepted)
9. [ ] Executable configuration arrives as command-line flags, or as `--config <file>` once there are more than 5 settings
