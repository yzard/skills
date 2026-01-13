---
name: naming-conventions
description: Naming conventions for all code in the project including Python, JavaScript, SQL, and configuration files. Use when writing new code, reviewing code, or refactoring to ensure consistent and meaningful names across all layers.
allowed-tools: Read, Write, Edit, Grep, Glob
---

# Naming Conventions

## Scope

These naming conventions apply to **ALL** code and scripts in the project:
- Python (backend, scripts, tests)
- JavaScript (frontend)
- SQL (database schemas, queries)
- Configuration files (YAML, JSON, TOML)
- API endpoints and parameters

## Core Rules

### 1. NO Meaningless Names

**NEVER use single letters or non-descriptive names:**

| Forbidden | Use Instead |
|-----------|-------------|
| `a`, `b`, `c`, `x`, `y` | Descriptive names based on purpose |
| `tmp`, `temp` | What it actually holds: `cached_result`, `pending_items` |
| `data`, `info` | Specific: `user_profile`, `invoice_details` |
| `val`, `value` | What value: `hourly_rate`, `total_amount` |
| `item`, `obj` | What it represents: `time_entry`, `session_token` |
| `ret`, `result` | What result: `generated_invoice`, `validated_user` |

**Exception:** Loop counters in simple iterations where context is obvious:
```python
# Acceptable only for simple index iteration
for i in range(10):
    print(items[i])

# But prefer descriptive names when possible
for index, entry in enumerate(time_entries):
    process_entry(index, entry)
```

### 2. NO Abbreviations (with exceptions)

**Spell out words fully:**

| Avoid | Use Instead |
|-------|-------------|
| `db` | `database` |
| `usr`, `user` (when meaning username) | `username` |
| `pwd`, `pass` | `password` |
| `cfg`, `conf` | `config` or `configuration` |
| `msg` | `message` |
| `btn` | `button` |
| `req` | `request` |
| `res`, `resp` | `response` |
| `addr` | `address` |
| `num`, `cnt` | `number`, `count` |
| `idx` | `index` |
| `len` | `length` |
| `str` | `string` or actual meaning like `name` |
| `amt` | `amount` |
| `qty` | `quantity` |
| `desc` | `description` |
| `err` | `error` |
| `auth` | `authentication` or `authorization` |
| `conn` | `connection` |
| `stmt` | `statement` |
| `impl` | `implementation` |
| `init` | `initialize` |
| `calc` | `calculate` |
| `gen` | `generate` |
| `max`, `min` | `maximum`, `minimum` (in variable names) |

**Allowed abbreviations** (industry-standard, universally understood):

| Abbreviation | Meaning |
|--------------|---------|
| `tcp`, `udp`, `http`, `https` | Network protocols |
| `url`, `uri` | Web addresses |
| `html`, `css`, `json`, `xml`, `yaml` | Data/markup formats |
| `api` | Application Programming Interface |
| `id` | Identifier |
| `sql` | Structured Query Language |
| `ip` | Internet Protocol |
| `cpu`, `gpu`, `ram` | Hardware components |
| `pdf` | Portable Document Format |
| `uuid` | Universally Unique Identifier |
| `jwt` | JSON Web Token |
| `oauth` | Open Authorization |
| `ssl`, `tls` | Security protocols |
| `io` | Input/Output |
| `async` | Asynchronous |

### 3. Cross-Layer Consistency (CRITICAL)

**The same concept MUST use the same name across ALL layers:**

```
Database Column  →  API Parameter  →  Frontend Variable
────────────────────────────────────────────────────────
username         →  username       →  username
hourly_rate      →  hourly_rate    →  hourlyRate (JS camelCase only)
invoice_number   →  invoice_number →  invoiceNumber
```

**WRONG - Creating aliases:**
```python
# Database
CREATE TABLE users (username TEXT)

# Python API - WRONG: different name!
def login(user_name: str):  # Should be: username
    pass

# Frontend - WRONG: yet another name!
const usr_name = response.user_name;  # Should be: username
```

**CORRECT - Same name everywhere:**
```python
# Database
CREATE TABLE users (username TEXT)

# Python API
def login(username: str):
    pass

# Frontend (camelCase for JS, but same root word)
const username = response.username;
```

### 4. Layer-Specific Casing

While the **root name stays the same**, apply language-appropriate casing:

| Layer | Convention | Example |
|-------|------------|---------|
| SQL | snake_case | `hourly_rate`, `created_at` |
| Python | snake_case | `hourly_rate`, `created_at` |
| JavaScript | camelCase | `hourlyRate`, `createdAt` |
| JSON API | snake_case | `"hourly_rate": 100` |
| CSS classes | kebab-case | `hourly-rate-input` |

**The root words remain identical** - only casing changes.

## Practical Examples

### Database to Frontend Flow

```sql
-- SQL: Define the source of truth
CREATE TABLE time_entries (
    entry_date DATE,
    hours_worked REAL,
    hourly_rate REAL,
    is_billable INTEGER
);
```

```python
# Python Pydantic model: Same names
class TimeEntry(BaseModel):
    entry_date: str
    hours_worked: float
    hourly_rate: float
    is_billable: bool

# Python API endpoint: Same names in parameters
@router.get("/time_entries")
async def get_time_entries(
    start_date: str,
    end_date: str
) -> List[TimeEntry]:
    pass
```

```javascript
// JavaScript: Same names, JS casing
const timeEntry = {
    entryDate: response.entry_date,
    hoursWorked: response.hours_worked,
    hourlyRate: response.hourly_rate,
    isBillable: response.is_billable
};
```

### Function and Variable Naming

```python
# WRONG
def get_usr_info(u_id):
    d = fetch_data(u_id)
    return d

# CORRECT
def get_user_information(user_id):
    user_profile = fetch_user_profile(user_id)
    return user_profile
```

### API Endpoints

```python
# WRONG - inconsistent naming
@router.post("/api/usr/auth")
async def authenticate_user(usr_credentials: dict):
    pass

# CORRECT - consistent, full words
@router.post("/api/user/authenticate")
async def authenticate_user(credentials: UserCredentials):
    pass
```

## Checklist Before Committing Code

1. **No single-letter variables** (except simple loop counters)
2. **No abbreviations** (except allowed list above)
3. **Names match across layers:**
   - Database column name = API parameter name = Frontend variable name
4. **Names are descriptive** - someone unfamiliar with the code can understand the purpose
5. **Consistent casing** per language conventions

## Finding Inconsistencies

```bash
# Find potential abbreviations in Python
grep -rn "db\|usr\|pwd\|cfg\|msg\|btn\|req\|res\|addr\|num\|cnt\|idx" --include="*.py"

# Find single-letter variables (excluding common patterns)
grep -rn " [a-z] = " --include="*.py" | grep -v "for .* in\|lambda"

# Compare naming between files
grep -h "username\|user_name\|userName\|usr" --include="*.py" --include="*.js" --include="*.sql" -r
```

## When Refactoring Existing Code

If you find inconsistent names:

1. **Identify the source of truth** (usually the database schema)
2. **Trace the name through all layers** (API, models, frontend)
3. **Rename to match the source** in all locations
4. **Update tests** to use consistent names
5. **Search for any remaining inconsistencies**
