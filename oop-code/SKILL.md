---
name: oop-code
description: Write object-oriented Python code following clean architecture principles. Use when the user asks to write classes, design OOP architecture, create interfaces, or implement composition patterns.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(isort:*), Bash(black:*)
---

# Object-Oriented Programming Guidelines

## What This Skill Does

Guides writing OOP code following the principle: **Composition over Inheritance**.

## Core Principles

### 1. Composition Over Inheritance

**Prefer composition and dependency injection over class inheritance.**

```python
# GOOD - Composition with dependency injection
class OrderProcessor:
    def __init__(self, database: Database, payment_client: PaymentClient):
        self.database = database
        self.payment_client = payment_client

    def process(self, order_id: str) -> None:
        order = self.database.get_order(order_id)
        self.payment_client.charge(order.amount)
        self.database.mark_paid(order_id)
```

### 2. Subtyping with Protocol

**Use `typing.Protocol` for interfaces (structural subtyping).**

```python
from typing import Protocol

class Repository(Protocol):
    """Interface for data access."""
    def get(self, id: str) -> dict: ...
    def save(self, id: str, data: dict) -> None: ...

class DatabaseRepository:
    """Concrete implementation - implicitly implements Repository."""
    def get(self, id: str) -> dict:
        return self.connection.fetch(id)

    def save(self, id: str, data: dict) -> None:
        self.connection.insert(id, data)

class InMemoryRepository:
    """Another implementation for testing."""
    def __init__(self) -> None:
        self.storage: dict[str, dict] = {}

    def get(self, id: str) -> dict:
        return self.storage.get(id, {})

    def save(self, id: str, data: dict) -> None:
        self.storage[id] = data

# Both work with code expecting Repository
def process_data(repo: Repository, id: str) -> None:
    data = repo.get(id)
    # ...
```

### 3. Abstract Base Classes (When Appropriate)

**Use ABC only for clear "is-a" relationships with shared implementation.**

```python
from abc import ABC, abstractmethod
from typing import List

class Parser(ABC):
    """Base class for parsers - clear shared behavior."""

    def __init__(self, base_url: str):
        self.base_url = base_url

    @abstractmethod
    def parse_content(self, html: str) -> List[dict]:
        """Subclasses must implement parsing logic."""
        ...

    def fetch_and_parse(self, url: str) -> List[dict]:
        """Shared implementation - template method pattern."""
        html = self._fetch(url)
        return self.parse_content(html)

    def _fetch(self, url: str) -> str:
        # Shared fetch logic
        response = httpx.get(url)
        return response.text

class JSONParser(Parser):
    def parse_content(self, html: str) -> List[dict]:
        return json.loads(html)

class HTMLParser(Parser):
    def parse_content(self, html: str) -> List[dict]:
        # HTML-specific parsing
        return []
```

### 4. When NOT to Use Inheritance

Avoid inheritance when:
- Classes only share data, not behavior
- The relationship is "has-a" not "is-a"
- You're tempted to override most parent methods
- Multiple inheritance would be needed

```python
# BAD - Inheritance just for code reuse
class BaseService:
    def __init__(self, config: Config):
        self.config = config

class UserService(BaseService):
    pass

class OrderService(BaseService):
    pass

# GOOD - Composition instead
class UserService:
    def __init__(self, config: Config, repository: UserRepository):
        self.config = config
        self.repository = repository

class OrderService:
    def __init__(self, config: Config, repository: OrderRepository):
        self.config = config
        self.repository = repository
```

## Class Design Patterns

### Factory Pattern

```python
def create_repository(config: Config) -> Repository:
    """Factory function - preferred over class hierarchies."""
    if config.use_database:
        return DatabaseRepository(config.db_connection)
    return InMemoryRepository()
```

### Dependency Injection

```python
class ReportGenerator:
    def __init__(
        self,
        data_source: DataSource,
        formatter: Formatter,
        exporter: Exporter,
    ):
        self.data_source = data_source
        self.formatter = formatter
        self.exporter = exporter

    def generate(self, report_id: str) -> None:
        data = self.data_source.fetch(report_id)
        formatted = self.formatter.format(data)
        self.exporter.export(formatted)

# Easy to test with mock dependencies
# Easy to swap implementations
```

### Builder Pattern (for complex objects)

```python
class QueryBuilder:
    def __init__(self) -> None:
        self._select: List[str] = []
        self._where: List[str] = []
        self._limit: Optional[int] = None

    def select(self, *columns: str) -> "QueryBuilder":
        self._select.extend(columns)
        return self

    def where(self, condition: str) -> "QueryBuilder":
        self._where.append(condition)
        return self

    def limit(self, n: int) -> "QueryBuilder":
        self._limit = n
        return self

    def build(self) -> str:
        query = f"SELECT {', '.join(self._select)}"
        if self._where:
            query += f" WHERE {' AND '.join(self._where)}"
        if self._limit:
            query += f" LIMIT {self._limit}"
        return query

# Usage
query = (
    QueryBuilder()
    .select("id", "name")
    .where("active = 1")
    .limit(10)
    .build()
)
```

## Type Annotations for Classes

**Always annotate:**
- Constructor parameters
- Method parameters and return types
- Class attributes (using class-level annotations)

```python
from typing import Optional, List

class Order:
    id: str
    customer_id: str
    items: List[str]
    total: float
    paid: bool

    def __init__(self, id: str, customer_id: str, items: List[str]) -> None:
        self.id = id
        self.customer_id = customer_id
        self.items = items
        self.total = 0.0
        self.paid = False

    def calculate_total(self, prices: dict[str, float]) -> float:
        self.total = sum(prices.get(item, 0) for item in self.items)
        return self.total

    def mark_paid(self) -> None:
        self.paid = True

    def get_status(self) -> str:
        return "Paid" if self.paid else "Pending"
```

## OOP Checklist

When writing classes:

1. [ ] Prefer composition over inheritance
2. [ ] Use Protocol for interfaces (structural typing)
3. [ ] Use ABC only for clear "is-a" with shared behavior
4. [ ] Dependencies injected via constructor
5. [ ] All methods have type annotations
6. [ ] Public methods at top, private (`_`) at bottom
7. [ ] No deep inheritance hierarchies (max 2 levels)
8. [ ] Classes have single responsibility
9. [ ] Easy to test with mock dependencies
