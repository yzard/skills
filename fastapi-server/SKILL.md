---
name: fastapi-server
description: Write or modify FastAPI server code following clean architecture. Use when the user asks to create a FastAPI app, add API endpoints, set up web server, add authentication, or modify server code.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(isort:*), Bash(black:*)
---

# FastAPI Server Development

## What This Skill Does

Creates and modifies FastAPI applications following clean architecture:
- Application factory pattern (no module-level app)
- YAML config with Pydantic validation
- Centralized routes and database modules
- Global exception handling
- Session-based authentication

## Project Structure

```
entrypoints/
├── server.py            # Entry point, calls create_application()
server/
├── __init__.py
├── app.py               # Application factory
├── config.py            # Pydantic config schema
├── routes.py            # Main router, includes sub-routers
├── auth.py              # Authentication logic
├── database.py          # Database queries (or database/ module)
├── exceptions.py        # Custom exceptions and global handler
├── middleware.py        # Request logging middleware
└── module/              # Optional sub-modules
    ├── __init__.py
    └── routes.py        # Sub-module endpoints
```

## Core Principle: Application Factory Pattern

**NEVER create app as module variable.** Always use factory function.

```python
# BAD - Module-level app
app = FastAPI()

@app.get("/")
def root():
    return {"status": "ok"}

# GOOD - Application factory
def create_application(config: ServerConfig) -> FastAPI:
    app = FastAPI()
    # Setup app...
    return app
```

## server/config.py - Pydantic Config Schema

All configuration must be validated with Pydantic:

```python
from pathlib import Path
from typing import Optional

import yaml
from pydantic import BaseModel, Field


class DatabaseConfig(BaseModel):
    path: str = Field(description="Path to SQLite database")
    pool_size: int = Field(default=5, description="Connection pool size")


class AuthConfig(BaseModel):
    enabled: bool = Field(default=False, description="Enable authentication")
    token_expiration_hours: int = Field(default=24, description="Session token expiration in hours")


class ServerConfig(BaseModel):
    host: str = Field(default="0.0.0.0", description="Server host")
    port: int = Field(default=8000, description="Server port")
    debug: bool = Field(default=False, description="Debug mode")
    database: DatabaseConfig
    auth: AuthConfig = Field(default_factory=AuthConfig)


def load_config(config_path: str) -> ServerConfig:
    """Load and validate config from YAML file."""
    path = Path(config_path)
    if not path.exists():
        raise FileNotFoundError(f"Config file not found: {config_path}")

    with open(path) as f:
        raw_config = yaml.safe_load(f)

    return ServerConfig(**raw_config)
```

Example `config.yaml`:

```yaml
host: "0.0.0.0"
port: 8000
debug: false

database:
  path: "/data/app.db"
  pool_size: 10

auth:
  enabled: true
  token_expiration_hours: 48
```

## server/app.py - Application Factory

```python
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from server.config import ServerConfig
from server.database import Database
from server.exceptions import setup_exception_handlers
from server.routes import router


logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage application lifecycle - startup and shutdown."""
    # Startup: Initialize resources
    logger.info("Starting application...")
    app.state.database = Database(app.state.config.database.path)

    yield

    # Shutdown: Cleanup resources
    logger.info("Shutting down application...")
    app.state.database.close()


def create_application(config: ServerConfig) -> FastAPI:
    """Create and configure FastAPI application."""
    app = FastAPI(
        title="API Server",
        lifespan=lifespan,
    )

    # Store config in app.state for access in endpoints
    app.state.config = config

    # Store shared managers/singletons in app.state
    app.state.sessions = {}  # For auth tokens

    # Setup global exception handler
    setup_exception_handlers(app)

    # Include routers
    app.include_router(router, prefix="/api")

    return app
```

## server/main.py - Entry Point

```python
import os
import logging

import uvicorn

from server.app import create_application
from server.config import load_config


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
)
logger = logging.getLogger(__name__)


def main() -> None:
    """Main entry point."""
    config_path = os.environ.get("CONFIG_PATH", "config.yaml")
    logger.info(f"Loading config from {config_path}")

    config = load_config(config_path)
    app = create_application(config)

    uvicorn.run(
        app,
        host=config.host,
        port=config.port,
    )


if __name__ == "__main__":
    main()
```

## server/exceptions.py - Global Exception Handler

**Minimal try-except in endpoints.** Let exceptions propagate to global handler.

```python
import logging
import traceback
from typing import Any

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse


logger = logging.getLogger(__name__)


class AppException(Exception):
    """Base exception for application errors."""
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code
        super().__init__(message)


class NotFoundError(AppException):
    """Resource not found."""
    def __init__(self, message: str = "Resource not found"):
        super().__init__(message, status_code=404)


class AuthenticationError(AppException):
    """Authentication failed."""
    def __init__(self, message: str = "Authentication required"):
        super().__init__(message, status_code=401)


def _format_request_info(request: Request, body: Any = None) -> str:
    """Format request information for logging."""
    info = [
        f"Method: {request.method}",
        f"URL: {request.url}",
        f"Client: {request.client.host if request.client else 'unknown'}",
        f"Headers: {dict(request.headers)}",
    ]
    if body:
        info.append(f"Body: {body}")
    return "\n".join(info)


async def global_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    """Handle all unhandled exceptions."""
    # Try to get request body for logging
    body = None
    try:
        body = await request.json()
    except Exception:
        pass

    # Log full details
    logger.error(
        f"Unhandled exception:\n"
        f"{_format_request_info(request, body)}\n"
        f"Exception: {exc}\n"
        f"Traceback:\n{traceback.format_exc()}"
    )

    return JSONResponse(
        status_code=500,
        content={"error": "Internal server error"},
    )


async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    """Handle application-specific exceptions."""
    logger.warning(f"App exception: {exc.message} (status={exc.status_code})")
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.message},
    )


def setup_exception_handlers(app: FastAPI) -> None:
    """Register exception handlers on the app."""
    app.add_exception_handler(AppException, app_exception_handler)
    app.add_exception_handler(Exception, global_exception_handler)
```

## server/middleware.py - Request Logging Middleware

**Log every request with method, path, status, duration, and request body.**

Format: `INFO POST /api/v1/media/list 200 03.98ms {"cursor":"2025-03-12T22:13:36+00:00_128","groupBy":"day","limit":100}`

```python
import json
import logging
import time
from typing import Callable

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware


logger = logging.getLogger(__name__)


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Log every request with method, path, status, duration, and body."""

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        start_time = time.perf_counter()

        # Read request body for logging
        body_bytes = await request.body()
        body_str = ""
        if body_bytes:
            try:
                body_json = json.loads(body_bytes)
                body_str = json.dumps(body_json, separators=(",", ":"))
            except json.JSONDecodeError:
                body_str = body_bytes.decode("utf-8", errors="replace")

        # Process request
        response = await call_next(request)

        # Calculate duration
        duration_ms = (time.perf_counter() - start_time) * 1000

        # Format log line: INFO POST /api/v1/media/list 200 03.98ms {"key":"value"}
        method = request.method
        path = request.url.path
        status = response.status_code

        if body_str:
            log_line = f"{method} {path} {status} {duration_ms:05.2f}ms {body_str}"
        else:
            log_line = f"{method} {path} {status} {duration_ms:05.2f}ms"

        # Log with appropriate level based on status code
        if status >= 500:
            logger.error(log_line)
        elif status >= 400:
            logger.warning(log_line)
        else:
            logger.info(log_line)

        return response


def setup_middleware(app) -> None:
    """Register middleware on the app."""
    app.add_middleware(RequestLoggingMiddleware)
```

**Update app.py to include middleware:**

```python
from server.middleware import setup_middleware

def create_application(config: ServerConfig) -> FastAPI:
    app = FastAPI(title="API Server", lifespan=lifespan)
    app.state.config = config
    app.state.sessions = {}

    # Setup middleware (order matters - logging should be outermost)
    setup_middleware(app)

    # Setup global exception handler
    setup_exception_handlers(app)

    # Include routers
    app.include_router(router, prefix="/api")

    return app
```

**Example log output:**
```
INFO POST /api/v1/media/list 200 03.98ms {"cursor":"2025-03-12T22:13:36+00:00_128","groupBy":"day","limit":100}
WARN POST /api/v1/items/get 404 01.23ms {"item_id":"nonexistent"}
ERROR POST /api/v1/items/create 500 15.67ms {"name":"test"}
```

## Exception Handling Pattern

### ❌ Endpoints: NO try-except - Always Let Exceptions Propagate

Endpoints should NEVER use try-except. The global exception handler will catch and log everything.

```python
# ❌ BAD - Don't catch exceptions in endpoints
@router.post("/items/get")
def get_item(request: Request, data: ItemRequest, user: str = Depends(get_current_user)) -> ItemResponse:
    try:
        db: Database = request.app.state.database
        item = db.get_item(data.item_id)
        return ItemResponse(id=item["id"], name=item["name"])
    except Exception as exc:
        logging.exception(f"Error: {exc}")
        raise

# ✅ GOOD - Let exceptions propagate to global handler
@router.post("/items/get")
def get_item(request: Request, data: ItemRequest, user: str = Depends(get_current_user)) -> ItemResponse:
    db: Database = request.app.state.database
    item = db.get_item(data.item_id)
    return ItemResponse(id=item["id"], name=item["name"])
```

### ✅ Background Routines: ONE Top-Level try-except with logging.exception()

Background routines MUST have exactly ONE try-except at the top level. The routine implementation should have NO try-except blocks.

```python
# ✅ GOOD - One top-level try-except, no try-except in implementation
class Routine:
    async def process_queue(self):
        """Process queue with top-level exception handling."""
        try:
            logging.info("Processing queue")

            # Implementation - NO try-except here
            item = self._database.get_next_item()
            if item:
                self._validate_item(item)  # Can raise
                self._process_item(item)   # Can raise
                self._update_status(item)  # Can raise

            logging.info("Queue processing completed")
        except Exception as exc:
            logging.exception(f"Error in process_queue routine: {exc}")
            # Don't re-raise - keep scheduler running

    def _validate_item(self, item):
        """Validate item - NO try-except, let exceptions bubble up."""
        if not item.get("id"):
            raise ValueError("Item missing ID")

    def _process_item(self, item):
        """Process item - NO try-except, let exceptions bubble up."""
        result = self._external_service.call(item)  # Can raise
        self._database.save_result(result)          # Can raise

# ❌ BAD - Multiple try-except blocks in routine
class Routine:
    async def process_queue(self):
        try:
            item = self._database.get_next_item()

            try:
                self._validate_item(item)  # ❌ Don't catch here
            except ValueError as e:
                logging.error(f"Validation error: {e}")
                return

            try:
                self._process_item(item)   # ❌ Don't catch here
            except Exception as e:
                logging.error(f"Process error: {e}")
                return
        except Exception as exc:
            logging.exception(f"Error: {exc}")
```

### Why This Pattern?

**Endpoints:**
- Global handler provides consistent error logging and response format
- Avoids duplicate logging and scattered error handling
- Single place to modify error response structure

**Background Routines:**
- Global handler doesn't catch background task exceptions
- One top-level handler prevents scheduler from crashing
- Implementation without try-except keeps code clean and lets errors surface naturally
- All errors get logged with full traceback at the top level

## server/auth.py - Session-Based Authentication

```python
import hashlib
import secrets
from datetime import datetime, timedelta
from typing import Optional

from fastapi import Depends, Request
from fastapi.security import HTTPBasic, HTTPBasicCredentials

from server.config import ServerConfig
from server.exceptions import AuthenticationError


security = HTTPBasic()


class Session:
    """User session with token."""
    def __init__(self, username: str, expires_at: datetime):
        self.username = username
        self.token = secrets.token_urlsafe(32)
        self.expires_at = expires_at


def _hash_password(password: str) -> str:
    """Hash password for comparison."""
    return hashlib.sha256(password.encode()).hexdigest()


def authenticate_user(
    config: ServerConfig,
    username: str,
    password: str,
) -> bool:
    """Validate username and password against config."""
    # Users should be defined in config
    allowed_users = getattr(config, 'allowed_users', {})
    if username not in allowed_users:
        return False
    return allowed_users[username] == password


def create_session(request: Request, username: str) -> Session:
    """Create a new session for authenticated user."""
    config: ServerConfig = request.app.state.config
    expires_at = datetime.utcnow() + timedelta(hours=config.auth.token_expiration_hours)

    session = Session(username=username, expires_at=expires_at)

    # Store in app.state.sessions
    request.app.state.sessions[session.token] = session

    return session


def get_current_user(request: Request) -> str:
    """Dependency to get current authenticated user from session token."""
    config: ServerConfig = request.app.state.config

    # Skip auth if disabled
    if not config.auth.enabled:
        return "anonymous"

    # Get token from header
    token = request.headers.get("X-Auth-Token")
    if not token:
        raise AuthenticationError("Missing authentication token")

    # Validate token
    sessions = request.app.state.sessions
    session = sessions.get(token)

    if not session:
        raise AuthenticationError("Invalid authentication token")

    if datetime.utcnow() > session.expires_at:
        # Remove expired session
        del sessions[token]
        raise AuthenticationError("Session expired")

    return session.username


def login_with_basic_auth(
    request: Request,
    credentials: HTTPBasicCredentials = Depends(security),
) -> Session:
    """Login endpoint dependency using HTTP Basic Auth."""
    config: ServerConfig = request.app.state.config

    if not authenticate_user(config, credentials.username, credentials.password):
        raise AuthenticationError("Invalid username or password")

    return create_session(request, credentials.username)
```

## server/routes.py - Main Router

**All endpoints use POST with request body** (except file downloads).

```python
from typing import Optional

from fastapi import APIRouter, Depends, Request
from fastapi.responses import FileResponse
from fastapi.security import HTTPBasicCredentials
from pydantic import BaseModel

from server.auth import get_current_user, login_with_basic_auth, security
from server.database import Database


router = APIRouter()


# === Request/Response Models ===

class LoginResponse(BaseModel):
    token: str
    expires_in_hours: int


class ItemRequest(BaseModel):
    item_id: str
    include_details: bool = False


class ItemResponse(BaseModel):
    id: str
    name: str
    details: Optional[str] = None


class CreateItemRequest(BaseModel):
    name: str
    description: Optional[str] = None


# === Auth Endpoints ===

@router.post("/login")
def login(
    request: Request,
    credentials: HTTPBasicCredentials = Depends(security),
) -> LoginResponse:
    """Login with HTTP Basic Auth, returns session token."""
    session = login_with_basic_auth(request, credentials)
    config = request.app.state.config
    return LoginResponse(
        token=session.token,
        expires_in_hours=config.auth.token_expiration_hours,
    )


# === Protected Endpoints ===

@router.post("/items/get")
def get_item(
    request: Request,
    data: ItemRequest,
    user: str = Depends(get_current_user),
) -> ItemResponse:
    """Get item by ID. Uses POST with request body."""
    db: Database = request.app.state.database
    item = db.get_item(data.item_id)
    return ItemResponse(
        id=item["id"],
        name=item["name"],
        details=item.get("details") if data.include_details else None,
    )


@router.post("/items/create")
def create_item(
    request: Request,
    data: CreateItemRequest,
    user: str = Depends(get_current_user),
) -> ItemResponse:
    """Create new item. Uses POST with request body."""
    db: Database = request.app.state.database
    item = db.create_item(data.name, data.description)
    return ItemResponse(id=item["id"], name=item["name"])


# === File Downloads (Exception: uses GET) ===

@router.get("/files/{file_id}")
def download_file(
    request: Request,
    file_id: str,
    user: str = Depends(get_current_user),
) -> FileResponse:
    """Download file. Uses GET because FileResponse convention."""
    db: Database = request.app.state.database
    file_path = db.get_file_path(file_id)
    return FileResponse(path=file_path)


# === Include Sub-Module Routers ===

# from server.module.routes import router as module_router
# router.include_router(module_router, prefix="/module", tags=["module"])
```

## server/database.py - Database Module

**Minimal try-except.** Let exceptions propagate to global handler.

```python
import logging
import sqlite3
from typing import Any, Optional


logger = logging.getLogger(__name__)


class Database:
    """Database access layer. No exception handling - let errors propagate."""

    def __init__(self, db_path: str):
        self.db_path = db_path
        self.connection = sqlite3.connect(db_path, check_same_thread=False)
        self.connection.row_factory = sqlite3.Row
        self._initialize_schema()

    def _initialize_schema(self) -> None:
        """Create tables if they don't exist."""
        cursor = self.connection.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS items (
                id TEXT PRIMARY KEY
              , name TEXT NOT NULL
              , description TEXT
              , created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        self.connection.commit()

    def close(self) -> None:
        """Close database connection."""
        self.connection.close()

    def get_item(self, item_id: str) -> dict:
        """Get item by ID. Raises if not found."""
        cursor = self.connection.cursor()
        cursor.execute("""
            SELECT id
                 , name
                 , description
              FROM items
             WHERE id = ?
        """, (item_id,))
        row = cursor.fetchone()
        if not row:
            from server.exceptions import NotFoundError
            raise NotFoundError(f"Item not found: {item_id}")
        return dict(row)

    def create_item(self, name: str, description: Optional[str] = None) -> dict:
        """Create new item."""
        import uuid
        item_id = str(uuid.uuid4())
        cursor = self.connection.cursor()
        cursor.execute("""
            INSERT INTO items (id, name, description)
            VALUES (?, ?, ?)
        """, (item_id, name, description))
        self.connection.commit()
        return {"id": item_id, "name": name, "description": description}

    def get_file_path(self, file_id: str) -> str:
        """Get file path by ID."""
        cursor = self.connection.cursor()
        cursor.execute("""
            SELECT path
              FROM files
             WHERE id = ?
        """, (file_id,))
        row = cursor.fetchone()
        if not row:
            from server.exceptions import NotFoundError
            raise NotFoundError(f"File not found: {file_id}")
        return row["path"]
```

## Sub-Module Example: server/module/routes.py

```python
from fastapi import APIRouter, Depends, Request
from pydantic import BaseModel

from server.auth import get_current_user


router = APIRouter()


class ModuleRequest(BaseModel):
    action: str
    params: dict = {}


class ModuleResponse(BaseModel):
    result: str


@router.post("/action")
def perform_action(
    request: Request,
    data: ModuleRequest,
    user: str = Depends(get_current_user),
) -> ModuleResponse:
    """Perform module action. Uses POST with request body."""
    # Access shared state
    config = request.app.state.config

    # Module logic here
    result = f"Executed {data.action}"

    return ModuleResponse(result=result)
```

## Key Rules Summary

| Rule | Description |
|------|-------------|
| **No module-level app** | Use `create_application()` factory |
| **Config in app.state** | Access via `request.app.state.config` |
| **Shared state in app.state** | Singletons, managers, sessions |
| **POST for all endpoints** | Except FileResponse, streaming |
| **Pydantic request models** | All POST data validated |
| **No try-except in endpoints** | Global handler catches all |
| **Log every request** | Format: `INFO POST /path 200 03.98ms {body}` |
| **Session-based auth** | HTTP Basic → token → X-Auth-Token header |

## Request Logging Format

Every request is logged with:
- Log level (INFO/WARN/ERROR based on status code)
- HTTP method
- Request path
- Response status code
- Duration in milliseconds (05.2 format)
- Request body as compact JSON

```
INFO POST /api/v1/media/list 200 03.98ms {"cursor":"2025-03-12T22:13:36+00:00_128","groupBy":"day","limit":100}
WARN POST /api/v1/items/get 404 01.23ms {"item_id":"nonexistent"}
ERROR POST /api/v1/items/create 500 15.67ms {"name":"test"}
```

## Endpoint Pattern

```python
@router.post("/resource/action")
def action_name(
    request: Request,
    data: RequestModel,
    user: str = Depends(get_current_user),  # If auth needed
) -> ResponseModel:
    """Docstring."""
    # Access state
    config = request.app.state.config
    db = request.app.state.database

    # Business logic (no try-except)
    result = db.do_something(data.param)

    return ResponseModel(...)
```
