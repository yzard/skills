---
name: sql-coding
description: Write SQL queries following coding standards. Use when the user asks to write SQL, create database queries, add database tables, or modify database code.
allowed-tools: Read, Edit, Write, Glob, Grep
---

# SQL Coding Standards

## What This Skill Does

Guides writing SQL queries following strict formatting conventions:
- Single `schema.sql` file for ALL table and index creation statements
- Single migration file per migration (never scatter migration SQL)
- Centralized query file with named constants (no inline SQL)
- Raw SQL only (no ORM)
- Cross-database compatibility
- Trailing commas in CREATE TABLE column definitions
- Leading commas in SELECT column lists
- Parameterized queries for security
- SQL code centralized, never scattered across source files

## Core Principles

### 1. Single Centralized Schema File

**Every project MUST have ONE `schema.sql` file** that contains ALL:
- Table creation statements
- Index creation statements
- Foreign key constraints

**NEVER scatter SQL DDL across multiple files.** All table and index definitions belong in `schema.sql`.

This file ensures:
- New databases are created with all necessary tables and columns
- Existing databases can be migrated to include any missing tables/columns
- Single source of truth for the entire database structure

```sql
-- schema.sql - Single source of truth for database schema
-- Contains ALL tables and ALL indexes
-- Run on startup to ensure all tables exist

CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    role TEXT DEFAULT 'user',
    is_active INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    total_amount REAL NOT NULL,
    status TEXT DEFAULT 'pending',
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- ALL indexes must also be in this file
CREATE INDEX IF NOT EXISTS idx_users_email ON users (email);
CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders (user_id);
CREATE INDEX IF NOT EXISTS idx_orders_status ON orders (status);
```

**Column Definition Format (CREATE TABLE):**
- Trailing commas: comma goes on the RIGHT of each column definition
- Each column on its own line with consistent indentation
- Last column or constraint has NO trailing comma

```sql
-- GOOD - Trailing commas (comma on right)
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- BAD - Leading commas in CREATE TABLE
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT
  , name TEXT NOT NULL
  , email TEXT NOT NULL UNIQUE
  , created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Best Practices:**
- Use `CREATE TABLE IF NOT EXISTS` for all tables
- Use `CREATE INDEX IF NOT EXISTS` for all indexes
- Include foreign key constraints
- Keep schema.sql as the ONLY file for DDL statements
- Run schema.sql on application startup
- NEVER put table/index creation in source code files

### 2. Migration Files

**When database schema changes require migration, use ONE file per migration.** Never scatter migration SQL across multiple source files.

**Migration File Structure:**
```
migrations/
├── 001_initial_schema.sql
├── 002_add_user_preferences.sql
├── 003_add_order_tracking.sql
└── 004_add_product_categories.sql
```

**Each migration file contains ALL SQL for that migration:**

```sql
-- migrations/002_add_user_preferences.sql
-- Migration: Add user preferences table and related columns
-- Date: 2024-01-15

-- Add new table
CREATE TABLE IF NOT EXISTS user_preferences (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    theme TEXT DEFAULT 'light',
    language TEXT DEFAULT 'en',
    notifications_enabled INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Add new columns to existing table
ALTER TABLE users ADD COLUMN preferences_id INTEGER;

-- Add indexes for new table
CREATE INDEX IF NOT EXISTS idx_user_preferences_user_id ON user_preferences (user_id);
```

**Migration Best Practices:**
- One migration file = one logical change
- Include all related DDL in the same file (tables, columns, indexes)
- Use sequential numbering (001_, 002_, etc.)
- Include descriptive filename and header comment
- NEVER put migration SQL in application source code
- NEVER scatter a single migration across multiple files
- Keep migrations idempotent where possible (`IF NOT EXISTS`, `IF EXISTS`)

**Update schema.sql after migration:**
After a migration is applied, update `schema.sql` to reflect the new complete schema. The `schema.sql` should always represent the current database structure.

### 3. Centralized Query File

**All SQL queries MUST be defined in a centralized query file** with named variables. Other source files import and reference these variables instead of writing inline SQL.

**Python Example (`queries.py`):**
```python
# queries.py - All SQL queries defined as named constants

# User queries
GET_USER_BY_ID = """
SELECT id
     , name
     , email
     , created_at
  FROM users
 WHERE id = ?
"""

GET_ACTIVE_USERS = """
SELECT id
     , name
     , email
  FROM users
 WHERE is_active = 1
 ORDER BY name
"""

CREATE_USER = """
INSERT INTO users (name, email, role)
VALUES (?, ?, ?)
"""

UPDATE_USER = """
UPDATE users
   SET name = ?
     , email = ?
     , updated_at = CURRENT_TIMESTAMP
 WHERE id = ?
"""

# Order queries
GET_USER_ORDERS = """
SELECT o.id
     , o.total_amount
     , o.status
     , o.order_date
  FROM orders AS o
 WHERE o.user_id = ?
 ORDER BY o.order_date DESC
"""

GET_ORDER_SUMMARY = """
SELECT u.id
     , u.name
     , COUNT(o.id) AS order_count
     , SUM(o.total_amount) AS total_spent
  FROM users AS u
  LEFT JOIN orders AS o ON u.id = o.user_id
 GROUP BY u.id
        , u.name
"""
```

**Using queries in other files:**
```python
# user_service.py
from queries import GET_USER_BY_ID, CREATE_USER, GET_ACTIVE_USERS

class UserService:
    def get_user(self, user_id: str):
        cursor = self.connection.cursor()
        cursor.execute(GET_USER_BY_ID, (user_id,))
        return cursor.fetchone()

    def get_active_users(self):
        cursor = self.connection.cursor()
        cursor.execute(GET_ACTIVE_USERS)
        return cursor.fetchall()
```

**Rust Example (`queries.rs`):**
```rust
// queries.rs - All SQL queries defined as constants

// User queries
pub const GET_USER_BY_ID: &str = r#"
SELECT id
     , name
     , email
     , created_at
  FROM users
 WHERE id = ?
"#;

pub const GET_ACTIVE_USERS: &str = r#"
SELECT id
     , name
     , email
  FROM users
 WHERE is_active = 1
 ORDER BY name
"#;

pub const CREATE_USER: &str = r#"
INSERT INTO users (name, email, role)
VALUES (?, ?, ?)
"#;
```

**TypeScript/JavaScript Example (`queries.ts`):**
```typescript
// queries.ts - All SQL queries defined as constants

// User queries
export const GET_USER_BY_ID = `
SELECT id
     , name
     , email
     , created_at
  FROM users
 WHERE id = ?
`;

export const GET_ACTIVE_USERS = `
SELECT id
     , name
     , email
  FROM users
 WHERE is_active = 1
 ORDER BY name
`;

export const CREATE_USER = `
INSERT INTO users (name, email, role)
VALUES (?, ?, ?)
`;
```

**Benefits:**
- Single location to find and update all queries
- Easier to review SQL for security and performance
- Prevents query duplication
- Enables query reuse across services
- Simplifies testing and mocking

### 4. No Scattered SQL

**NEVER scatter SQL across multiple source files.** All SQL must be centralized:

| SQL Type | Location | Example |
|----------|----------|---------|
| Table creation | `schema.sql` | `CREATE TABLE IF NOT EXISTS...` |
| Index creation | `schema.sql` | `CREATE INDEX IF NOT EXISTS...` |
| Migrations | `migrations/NNN_description.sql` | One file per migration |
| Queries | `queries.py`, `queries.rs`, `queries.ts` | Named constants |

**BAD - SQL scattered across files:**
```
src/
├── user_service.py      # Contains CREATE TABLE for users
├── order_service.py     # Contains CREATE TABLE for orders
├── analytics.py         # Contains inline queries
└── migrations/
    ├── add_users.py     # SQL mixed with Python
    └── add_orders.py    # SQL mixed with Python
```

**GOOD - SQL centralized:**
```
src/
├── schema.sql           # ALL table and index creation
├── queries.py           # ALL query constants
├── migrations/
│   ├── 001_initial.sql  # Complete migration in one file
│   └── 002_add_prefs.sql
├── user_service.py      # Imports from queries.py
└── order_service.py     # Imports from queries.py
```

### 6. No ORM - Use Raw SQL

**DO NOT use SQLAlchemy or any ORM.** Write raw SQL for maximum control and portability.

```python
# GOOD - Raw SQL with parameterized query
sql = """
SELECT id
     , name
     , created_at
  FROM users
 WHERE id = ?
"""
cursor.execute(sql, (user_id,))

# BAD - Using SQLAlchemy ORM
from sqlalchemy.orm import Session
session.query(User).filter(User.id == user_id).first()
```

### 7. Parameterized Queries

**ALWAYS use `?` placeholders** to prevent SQL injection.

```python
# GOOD - Parameterized
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))

# GOOD - Multiple parameters
cursor.execute("""
    INSERT INTO users (name, email, role)
    VALUES (?, ?, ?)
""", (name, email, role))

# BAD - String formatting (SQL injection risk!)
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
cursor.execute("SELECT * FROM users WHERE id = " + user_id)
```

### 8. Database Compatibility

Write SQL compatible across **SQLite**, **PostgreSQL**, and **DuckDB**.

| Feature | SQLite | PostgreSQL | DuckDB | Recommendation |
|---------|--------|------------|--------|----------------|
| Placeholder | `?` | `%s` or `$1` | `?` or `$1` | Use `?` with SQLite-compatible drivers |
| String concat | `\|\|` | `\|\|` | `\|\|` | Use `\|\|` |
| Null handling | `IFNULL()` | `COALESCE()` | `COALESCE()` | Use `COALESCE()` |
| String aggregation | `GROUP_CONCAT()` | `STRING_AGG()` | `STRING_AGG()` | Be aware of differences |
| Boolean | `0`/`1` | `true`/`false` | Both | Use `0`/`1` for portability |

## SQL Formatting Rules

### Keywords: UPPERCASE

All SQL keywords must be **UPPERCASE**:

```sql
SELECT, FROM, WHERE, JOIN, LEFT JOIN, RIGHT JOIN, INNER JOIN,
ON, AND, OR, NOT, IN, EXISTS, BETWEEN, LIKE, IS NULL, IS NOT NULL,
ORDER BY, GROUP BY, HAVING, LIMIT, OFFSET,
INSERT, INTO, VALUES, UPDATE, SET, DELETE,
CREATE, TABLE, INDEX, VIEW, ALTER, DROP,
WITH, AS, UNION, EXCEPT, INTERSECT,
CASE, WHEN, THEN, ELSE, END,
COUNT, SUM, AVG, MIN, MAX, COALESCE, CAST
```

### Table/Column Names: lowercase

All identifiers in **lowercase**:

```sql
-- GOOD
SELECT user_id, user_name, created_at
  FROM users
 WHERE is_active = 1

-- BAD
SELECT User_ID, UserName, Created_At
  FROM Users
 WHERE Is_Active = 1
```

### Comma Placement Rules

**Different rules for different statements:**

| Statement Type | Comma Position | Example |
|----------------|----------------|---------|
| SELECT, INSERT, UPDATE | Leading (left) | `, column_name` |
| CREATE TABLE | Trailing (right) | `column_name,` |

#### SELECT/DML: Leading Commas

Use **leading commas** (comma at beginning of line) for SELECT, INSERT column lists, UPDATE SET clauses:

```sql
-- GOOD - Leading commas in SELECT
SELECT id
     , name
     , email
     , created_at
  FROM users

-- BAD - Trailing commas in SELECT
SELECT id,
       name,
       email,
       created_at
  FROM users
```

**Why leading commas for SELECT?**
- Easier to add/remove columns
- Easier to spot missing commas
- Cleaner diffs in version control

#### CREATE TABLE: Trailing Commas

Use **trailing commas** (comma at end of line) for CREATE TABLE column definitions:

```sql
-- GOOD - Trailing commas in CREATE TABLE
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- BAD - Leading commas in CREATE TABLE
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT
  , name TEXT NOT NULL
  , email TEXT NOT NULL UNIQUE
  , created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Column Alignment

Each column on its own line, aligned:

```sql
-- GOOD - Aligned columns
SELECT u.id
     , u.name
     , u.email
     , o.order_count
     , o.total_amount
  FROM users AS u
  LEFT JOIN order_summary AS o ON u.id = o.user_id

-- BAD - Not aligned
SELECT u.id, u.name, u.email, o.order_count, o.total_amount
FROM users AS u LEFT JOIN order_summary AS o ON u.id = o.user_id
```

### Keyword Alignment

Align major keywords (`SELECT`, `FROM`, `WHERE`, `ORDER BY`) at consistent positions:

```sql
SELECT u.id
     , u.name
     , u.email
  FROM users AS u
 WHERE u.is_active = 1
   AND u.created_at > ?
 ORDER BY u.created_at DESC
 LIMIT 10
```

Note the alignment pattern:
- `SELECT` at column 0
- `,` (commas) at column 5
- `FROM`, `WHERE`, `ORDER BY` right-aligned to column 5
- `AND`, `OR` indented under `WHERE`

## Query Templates

### Simple SELECT

```sql
SELECT id
     , name
     , email
     , created_at
  FROM users
 WHERE is_active = 1
 ORDER BY created_at DESC
```

### SELECT with JOIN

```sql
SELECT u.id
     , u.name
     , o.order_id
     , o.total_amount
     , o.order_date
  FROM users AS u
 INNER JOIN orders AS o ON u.id = o.user_id
 WHERE u.is_active = 1
   AND o.status = 'completed'
 ORDER BY o.order_date DESC
```

### SELECT with Multiple JOINs

```sql
SELECT u.id
     , u.name
     , o.order_id
     , oi.product_name
     , oi.quantity
     , oi.unit_price
  FROM users AS u
 INNER JOIN orders AS o ON u.id = o.user_id
 INNER JOIN order_items AS oi ON o.order_id = oi.order_id
  LEFT JOIN products AS p ON oi.product_id = p.id
 WHERE u.is_active = 1
 ORDER BY o.order_date DESC
       , oi.line_number ASC
```

### SELECT with GROUP BY

```sql
SELECT u.id
     , u.name
     , COUNT(o.order_id) AS order_count
     , SUM(o.total_amount) AS total_spent
     , MAX(o.order_date) AS last_order_date
  FROM users AS u
  LEFT JOIN orders AS o ON u.id = o.user_id
 WHERE u.is_active = 1
 GROUP BY u.id
        , u.name
HAVING COUNT(o.order_id) >= 5
 ORDER BY total_spent DESC
```

### CTE (Common Table Expression)

```sql
WITH active_users AS (
    SELECT id
         , name
         , email
      FROM users
     WHERE is_active = 1
       AND created_at > ?
)
, user_orders AS (
    SELECT user_id
         , COUNT(*) AS order_count
         , SUM(total_amount) AS total_spent
      FROM orders
     WHERE status = 'completed'
  GROUP BY user_id
)
SELECT au.id
     , au.name
     , au.email
     , COALESCE(uo.order_count, 0) AS order_count
     , COALESCE(uo.total_spent, 0) AS total_spent
  FROM active_users AS au
  LEFT JOIN user_orders AS uo ON au.id = uo.user_id
 ORDER BY uo.total_spent DESC NULLS LAST
```

### INSERT

```sql
INSERT INTO users
  ( name
  , email
  , role
  , is_active
  , created_at
) VALUES
  ( ?
  , ?
  , ?
  , 1
  , CURRENT_TIMESTAMP
)
```

### UPDATE

```sql
UPDATE users
   SET name = ?
     , email = ?
     , updated_at = CURRENT_TIMESTAMP
 WHERE id = ?
```

### DELETE

```sql
DELETE FROM users
 WHERE id = ?
   AND is_active = 0
```

### CREATE TABLE

Use trailing commas (comma on right of each column definition):

```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    role TEXT DEFAULT 'user',
    is_active INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);
```

### CREATE INDEX

```sql
CREATE INDEX IF NOT EXISTS idx_users_email
    ON users (email)

CREATE INDEX IF NOT EXISTS idx_orders_user_date
    ON orders (user_id, order_date DESC)
```

## Python Integration

### Database Class Pattern

```python
import sqlite3
from typing import Optional


class Database:
    """Database access with raw SQL. No ORM."""

    def __init__(self, db_path: str):
        self.connection = sqlite3.connect(db_path, check_same_thread=False)
        self.connection.row_factory = sqlite3.Row

    def get_user(self, user_id: str) -> Optional[dict]:
        """Get user by ID."""
        cursor = self.connection.cursor()
        cursor.execute("""
            SELECT id
                 , name
                 , email
                 , created_at
              FROM users
             WHERE id = ?
        """, (user_id,))
        row = cursor.fetchone()
        return dict(row) if row else None

    def get_users_with_orders(self, min_orders: int = 1) -> list[dict]:
        """Get users with minimum order count."""
        cursor = self.connection.cursor()
        cursor.execute("""
            SELECT u.id
                 , u.name
                 , COUNT(o.id) AS order_count
                 , SUM(o.total_amount) AS total_spent
              FROM users AS u
             INNER JOIN orders AS o ON u.id = o.user_id
             WHERE u.is_active = 1
             GROUP BY u.id
                    , u.name
            HAVING COUNT(o.id) >= ?
             ORDER BY total_spent DESC
        """, (min_orders,))
        return [dict(row) for row in cursor.fetchall()]

    def create_user(self, name: str, email: str) -> str:
        """Create new user, return ID."""
        cursor = self.connection.cursor()
        cursor.execute("""
            INSERT INTO users (name, email)
            VALUES (?, ?)
        """, (name, email))
        self.connection.commit()
        return str(cursor.lastrowid)
```

### Multi-line SQL in Python

Use triple-quoted strings with proper indentation:

```python
# GOOD - Clean multi-line SQL
sql = """
SELECT u.id
     , u.name
     , o.order_count
  FROM users AS u
  LEFT JOIN order_summary AS o ON u.id = o.user_id
 WHERE u.is_active = ?
 ORDER BY u.name
"""
cursor.execute(sql, (is_active,))

# BAD - Hard to read
sql = "SELECT u.id, u.name, o.order_count FROM users AS u LEFT JOIN order_summary AS o ON u.id = o.user_id WHERE u.is_active = ? ORDER BY u.name"
```

## Compatibility Tips

| Situation | Solution |
|-----------|----------|
| NULL handling | Use `COALESCE(value, default)` |
| Boolean values | Use `0`/`1` instead of `true`/`false` |
| String concatenation | Use `\|\|` operator |
| Current timestamp | Use `CURRENT_TIMESTAMP` |
| Auto-increment | SQLite: `AUTOINCREMENT`, PostgreSQL: `SERIAL` |
| UPSERT | Use `INSERT ... ON CONFLICT` (SQLite 3.24+, PostgreSQL) |

## Checklist

When writing SQL:

**File Organization:**
1. [ ] Single `schema.sql` file contains ALL table and index creation statements
2. [ ] Migrations use ONE file per migration (never scatter migration SQL)
3. [ ] All queries defined in centralized query file (e.g., `queries.py`, `queries.rs`, `queries.ts`)
4. [ ] Source files import query constants instead of inline SQL
5. [ ] NO SQL DDL scattered across application source files

**Formatting:**
6. [ ] Keywords in UPPERCASE
7. [ ] Table/column names in lowercase
8. [ ] CREATE TABLE uses trailing commas (comma on right of column definition)
9. [ ] SELECT uses leading commas (comma on left of column)
10. [ ] Each column on its own line
11. [ ] Parameterized queries with `?`
12. [ ] Aligned keywords (SELECT, FROM, WHERE)
13. [ ] JOINs on separate lines with aligned ON
14. [ ] GROUP BY / HAVING properly indented

**Other:**
15. [ ] No ORM usage
16. [ ] Compatible across SQLite/PostgreSQL/DuckDB
