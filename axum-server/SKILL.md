---
name: axum-server
description: Write or modify Axum (Rust) server code following clean architecture. Use when the user asks to create an Axum app, add API endpoints, set up Rust web server, add authentication, or modify Rust server code.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(cargo:*)
---

# Axum Server Development (Rust)

## What This Skill Does

Creates and modifies Axum (Rust) applications following clean architecture:
- Application factory pattern (no module-level app)
- YAML config with serde validation
- Centralized routes and database modules
- Global exception handling with request/response logging
- Session-based authentication
- Structured logging with request details

## Project Structure

```
src/
├── main.rs              # Entry point, calls create_application()
├── app.rs               # Application factory
├── config.rs            # Serde config schema
├── routes/
│   ├── mod.rs           # Main router, includes sub-routers
│   └── items.rs         # Item endpoints
├── auth.rs              # Authentication logic
├── database/
│   ├── mod.rs           # Database pool and queries
│   └── schema.sql       # SQL schema
├── error.rs             # Custom errors and global handler
├── middleware.rs        # Request logging middleware
└── models/
    ├── mod.rs           # Request/response models
    └── items.rs         # Item models
```

## Core Principle: Application Factory Pattern

**NEVER create app as module variable.** Always use factory function.

```rust
// BAD - Module-level app or router construction in main
fn main() {
    let app = Router::new().route("/", get(root));
    // ...
}

// GOOD - Application factory
pub fn create_application(state: AppState) -> Router {
    Router::new()
        .nest("/api", routes::router())
        .layer(middleware::request_logging())
        .with_state(state)
}
```

## src/config.rs - Serde Config Schema

All configuration must be validated with serde:

```rust
use serde::Deserialize;
use std::fs;
use std::path::Path;

#[derive(Debug, Clone, Deserialize)]
pub struct DatabaseConfig {
    pub path: String,
    #[serde(default = "default_pool_size")]
    pub pool_size: u32,
}

fn default_pool_size() -> u32 {
    5
}

#[derive(Debug, Clone, Deserialize)]
pub struct AuthConfig {
    #[serde(default)]
    pub enabled: bool,
    #[serde(default = "default_token_expiration")]
    pub token_expiration_hours: u64,
}

fn default_token_expiration() -> u64 {
    24
}

impl Default for AuthConfig {
    fn default() -> Self {
        Self {
            enabled: false,
            token_expiration_hours: 24,
        }
    }
}

#[derive(Debug, Clone, Deserialize)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,
    #[serde(default = "default_port")]
    pub port: u16,
    #[serde(default)]
    pub debug: bool,
    pub database: DatabaseConfig,
    #[serde(default)]
    pub auth: AuthConfig,
}

fn default_host() -> String {
    "0.0.0.0".to_string()
}

fn default_port() -> u16 {
    8000
}

pub fn load_config(config_path: &str) -> Result<ServerConfig, Box<dyn std::error::Error>> {
    let path = Path::new(config_path);
    if !path.exists() {
        return Err(format!("Config file not found: {}", config_path).into());
    }

    let content = fs::read_to_string(path)?;
    let config: ServerConfig = serde_yaml::from_str(&content)?;
    Ok(config)
}
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

## src/error.rs - Global Exception Handler

**Minimal error handling in endpoints.** Let errors propagate to global handler.

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;
use std::fmt;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    BadRequest(String),
    Authentication(String),
    Internal(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(msg) => write!(f, "Not found: {}", msg),
            AppError::BadRequest(msg) => write!(f, "Bad request: {}", msg),
            AppError::Authentication(msg) => write!(f, "Authentication error: {}", msg),
            AppError::Internal(msg) => write!(f, "Internal error: {}", msg),
        }
    }
}

impl std::error::Error for AppError {}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::Authentication(msg) => (StatusCode::UNAUTHORIZED, msg.clone()),
            AppError::Internal(msg) => (StatusCode::INTERNAL_SERVER_ERROR, msg.clone()),
        };

        let body = Json(ErrorResponse { error: message });
        (status, body).into_response()
    }
}

// Convert common errors to AppError
impl From<rusqlite::Error> for AppError {
    fn from(err: rusqlite::Error) -> Self {
        AppError::Internal(err.to_string())
    }
}

impl From<serde_json::Error> for AppError {
    fn from(err: serde_json::Error) -> Self {
        AppError::BadRequest(err.to_string())
    }
}
```

## src/middleware.rs - Request Logging Middleware

**Log every request with method, path, status, duration, and request body.**

Format: `INFO POST /api/v1/media/list 200 03.98ms {"cursor":"2025-03-12T22:13:36+00:00_128","groupBy":"day","limit":100}`

```rust
use axum::{
    body::{Body, Bytes},
    extract::Request,
    http::StatusCode,
    middleware::Next,
    response::Response,
};
use http_body_util::BodyExt;
use std::time::Instant;
use tracing::{error, info, warn};

pub async fn request_logging(request: Request, next: Next) -> Response {
    let start = Instant::now();
    let method = request.method().clone();
    let uri = request.uri().clone();
    let path = uri.path().to_string();

    // Extract request body for logging
    let (parts, body) = request.into_parts();
    let bytes = match body.collect().await {
        Ok(collected) => collected.to_bytes(),
        Err(_) => Bytes::new(),
    };

    // Format request body as compact JSON string for logging
    let body_str = if bytes.is_empty() {
        String::new()
    } else {
        match serde_json::from_slice::<serde_json::Value>(&bytes) {
            Ok(json) => serde_json::to_string(&json).unwrap_or_default(),
            Err(_) => String::from_utf8_lossy(&bytes).to_string(),
        }
    };

    // Reconstruct request with body
    let request = Request::from_parts(parts, Body::from(bytes));

    // Process request
    let response = next.run(request).await;

    let duration = start.elapsed();
    let status = response.status();
    let duration_ms = duration.as_secs_f64() * 1000.0;

    // Log format: INFO POST /api/v1/media/list 200 03.98ms {"key":"value"}
    let log_line = if body_str.is_empty() {
        format!(
            "{} {} {} {:05.2}ms",
            method,
            path,
            status.as_u16(),
            duration_ms
        )
    } else {
        format!(
            "{} {} {} {:05.2}ms {}",
            method,
            path,
            status.as_u16(),
            duration_ms,
            body_str
        )
    };

    match status {
        s if s.is_server_error() => error!("{}", log_line),
        s if s.is_client_error() => warn!("{}", log_line),
        _ => info!("{}", log_line),
    }

    response
}
```

## src/app.rs - Application Factory

```rust
use axum::{middleware, Router};
use std::sync::Arc;
use tokio::sync::RwLock;

use crate::config::ServerConfig;
use crate::database::Database;
use crate::middleware::request_logging;
use crate::routes;

#[derive(Clone)]
pub struct AppState {
    pub config: ServerConfig,
    pub database: Arc<Database>,
    pub sessions: Arc<RwLock<std::collections::HashMap<String, crate::auth::Session>>>,
}

pub fn create_application(config: ServerConfig) -> Router {
    // Initialize database
    let database = Arc::new(Database::new(&config.database.path).expect("Failed to open database"));

    let state = AppState {
        config,
        database,
        sessions: Arc::new(RwLock::new(std::collections::HashMap::new())),
    };

    Router::new()
        .nest("/api", routes::router())
        .layer(middleware::from_fn(request_logging))
        .with_state(state)
}
```

## src/main.rs - Entry Point

```rust
use std::env;
use std::net::SocketAddr;
use tracing::info;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod app;
mod auth;
mod config;
mod database;
mod error;
mod middleware;
mod models;
mod routes;

#[tokio::main]
async fn main() {
    // Initialize tracing
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "info".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // Load config
    let config_path = env::var("CONFIG_PATH").unwrap_or_else(|_| "config.yaml".to_string());
    info!("Loading config from {}", config_path);

    let config = config::load_config(&config_path).expect("Failed to load config");
    let addr = SocketAddr::from(([0, 0, 0, 0], config.port));

    // Create application
    let app = app::create_application(config);

    info!("Starting server on {}", addr);
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## src/auth.rs - Session-Based Authentication

```rust
use axum::{
    async_trait,
    extract::{FromRef, FromRequestParts},
    http::request::Parts,
};
use chrono::{DateTime, Duration, Utc};
use rand::distributions::Alphanumeric;
use rand::Rng;
use std::sync::Arc;
use tokio::sync::RwLock;

use crate::app::AppState;
use crate::config::ServerConfig;
use crate::error::AppError;

#[derive(Debug, Clone)]
pub struct Session {
    pub username: String,
    pub token: String,
    pub expires_at: DateTime<Utc>,
}

impl Session {
    pub fn new(username: String, expiration_hours: u64) -> Self {
        let token: String = rand::thread_rng()
            .sample_iter(&Alphanumeric)
            .take(32)
            .map(char::from)
            .collect();

        Self {
            username,
            token,
            expires_at: Utc::now() + Duration::hours(expiration_hours as i64),
        }
    }
}

pub struct CurrentUser(pub String);

#[async_trait]
impl<S> FromRequestParts<S> for CurrentUser
where
    AppState: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let app_state = AppState::from_ref(state);

        // Skip auth if disabled
        if !app_state.config.auth.enabled {
            return Ok(CurrentUser("anonymous".to_string()));
        }

        // Get token from header
        let token = parts
            .headers
            .get("X-Auth-Token")
            .and_then(|v| v.to_str().ok())
            .ok_or_else(|| AppError::Authentication("Missing authentication token".to_string()))?;

        // Validate token
        let sessions = app_state.sessions.read().await;
        let session = sessions
            .get(token)
            .ok_or_else(|| AppError::Authentication("Invalid authentication token".to_string()))?;

        if Utc::now() > session.expires_at {
            drop(sessions);
            // Remove expired session
            app_state.sessions.write().await.remove(token);
            return Err(AppError::Authentication("Session expired".to_string()));
        }

        Ok(CurrentUser(session.username.clone()))
    }
}

pub fn authenticate_user(config: &ServerConfig, username: &str, password: &str) -> bool {
    // In real app, check against database or config
    // This is a placeholder
    username == "admin" && password == "password"
}

pub async fn create_session(
    sessions: &Arc<RwLock<std::collections::HashMap<String, Session>>>,
    username: String,
    expiration_hours: u64,
) -> Session {
    let session = Session::new(username, expiration_hours);
    sessions
        .write()
        .await
        .insert(session.token.clone(), session.clone());
    session
}
```

## src/routes/mod.rs - Main Router

**All endpoints use POST with request body** (except file downloads).

```rust
use axum::{
    extract::{Path, State},
    routing::{get, post},
    Json, Router,
};

use crate::app::AppState;
use crate::auth::{authenticate_user, create_session, CurrentUser};
use crate::database::Database;
use crate::error::AppError;
use crate::models::*;

mod items;

pub fn router() -> Router<AppState> {
    Router::new()
        .route("/login", post(login))
        .route("/items/get", post(get_item))
        .route("/items/create", post(create_item))
        .route("/files/{file_id}", get(download_file))
        .nest("/items", items::router())
}

// === Auth Endpoints ===

async fn login(
    State(state): State<AppState>,
    Json(credentials): Json<LoginRequest>,
) -> Result<Json<LoginResponse>, AppError> {
    if !authenticate_user(&state.config, &credentials.username, &credentials.password) {
        return Err(AppError::Authentication(
            "Invalid username or password".to_string(),
        ));
    }

    let session = create_session(
        &state.sessions,
        credentials.username,
        state.config.auth.token_expiration_hours,
    )
    .await;

    Ok(Json(LoginResponse {
        token: session.token,
        expires_in_hours: state.config.auth.token_expiration_hours,
    }))
}

// === Protected Endpoints ===

async fn get_item(
    State(state): State<AppState>,
    _user: CurrentUser,
    Json(data): Json<ItemRequest>,
) -> Result<Json<ItemResponse>, AppError> {
    let item = state.database.get_item(&data.item_id)?;
    Ok(Json(ItemResponse {
        id: item.id,
        name: item.name,
        details: if data.include_details {
            item.description
        } else {
            None
        },
    }))
}

async fn create_item(
    State(state): State<AppState>,
    _user: CurrentUser,
    Json(data): Json<CreateItemRequest>,
) -> Result<Json<ItemResponse>, AppError> {
    let item = state.database.create_item(&data.name, data.description.as_deref())?;
    Ok(Json(ItemResponse {
        id: item.id,
        name: item.name,
        details: item.description,
    }))
}

// === File Downloads (Exception: uses GET) ===

async fn download_file(
    State(state): State<AppState>,
    _user: CurrentUser,
    Path(file_id): Path<String>,
) -> Result<axum::response::Response, AppError> {
    let file_path = state.database.get_file_path(&file_id)?;

    // Use tokio_util for file streaming
    let file = tokio::fs::File::open(&file_path)
        .await
        .map_err(|e| AppError::NotFound(format!("File not accessible: {}", e)))?;

    let stream = tokio_util::io::ReaderStream::new(file);
    let body = axum::body::Body::from_stream(stream);

    Ok(axum::response::Response::builder()
        .header("Content-Type", "application/octet-stream")
        .body(body)
        .unwrap())
}
```

## src/models/mod.rs - Request/Response Models

```rust
use serde::{Deserialize, Serialize};

// === Auth Models ===

#[derive(Debug, Deserialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

#[derive(Debug, Serialize)]
pub struct LoginResponse {
    pub token: String,
    pub expires_in_hours: u64,
}

// === Item Models ===

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ItemRequest {
    pub item_id: String,
    #[serde(default)]
    pub include_details: bool,
}

#[derive(Debug, Serialize)]
pub struct ItemResponse {
    pub id: String,
    pub name: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct CreateItemRequest {
    pub name: String,
    pub description: Option<String>,
}

// === Database Row Types ===

#[derive(Debug)]
pub struct Item {
    pub id: String,
    pub name: String,
    pub description: Option<String>,
}
```

## src/database/mod.rs - Database Module

**Minimal error handling.** Let errors propagate via `?` operator.

```rust
use r2d2::Pool;
use r2d2_sqlite::SqliteConnectionManager;
use rusqlite::params;
use std::fs;

use crate::error::AppError;
use crate::models::Item;

pub type DbPool = Pool<SqliteConnectionManager>;

pub struct Database {
    pool: DbPool,
}

impl Database {
    pub fn new(db_path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let manager = SqliteConnectionManager::file(db_path);
        let pool = Pool::new(manager)?;

        let db = Self { pool };
        db.initialize_schema()?;

        Ok(db)
    }

    fn initialize_schema(&self) -> Result<(), Box<dyn std::error::Error>> {
        let conn = self.pool.get()?;

        // Load and execute schema.sql
        let schema_path = "src/database/schema.sql";
        if std::path::Path::new(schema_path).exists() {
            let schema = fs::read_to_string(schema_path)?;
            conn.execute_batch(&schema)?;
        }

        Ok(())
    }

    pub fn get_item(&self, item_id: &str) -> Result<Item, AppError> {
        let conn = self.pool.get().map_err(|e| AppError::Internal(e.to_string()))?;

        let mut stmt = conn.prepare(
            r#"
            SELECT id
                 , name
                 , description
              FROM items
             WHERE id = ?
            "#,
        )?;

        let item = stmt
            .query_row(params![item_id], |row| {
                Ok(Item {
                    id: row.get(0)?,
                    name: row.get(1)?,
                    description: row.get(2)?,
                })
            })
            .map_err(|_| AppError::NotFound(format!("Item not found: {}", item_id)))?;

        Ok(item)
    }

    pub fn create_item(&self, name: &str, description: Option<&str>) -> Result<Item, AppError> {
        let conn = self.pool.get().map_err(|e| AppError::Internal(e.to_string()))?;
        let id = uuid::Uuid::new_v4().to_string();

        conn.execute(
            r#"
            INSERT INTO items (id, name, description)
            VALUES (?, ?, ?)
            "#,
            params![id, name, description],
        )?;

        Ok(Item {
            id,
            name: name.to_string(),
            description: description.map(String::from),
        })
    }

    pub fn get_file_path(&self, file_id: &str) -> Result<String, AppError> {
        let conn = self.pool.get().map_err(|e| AppError::Internal(e.to_string()))?;

        let mut stmt = conn.prepare(
            r#"
            SELECT path
              FROM files
             WHERE id = ?
            "#,
        )?;

        let path: String = stmt
            .query_row(params![file_id], |row| row.get(0))
            .map_err(|_| AppError::NotFound(format!("File not found: {}", file_id)))?;

        Ok(path)
    }
}
```

## src/database/schema.sql

```sql
-- schema.sql - Single source of truth for database schema
-- Contains ALL tables and ALL indexes

CREATE TABLE IF NOT EXISTS items (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS files (
    id TEXT PRIMARY KEY,
    path TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_items_name ON items (name);
CREATE INDEX IF NOT EXISTS idx_files_path ON files (path);
```

## Exception Handling Pattern

### Endpoints: Use `?` Operator - Let Errors Propagate

Endpoints should use `?` to propagate errors. The `AppError` type implements `IntoResponse` for automatic conversion.

```rust
// GOOD - Use ? operator, errors auto-convert to responses
async fn get_item(
    State(state): State<AppState>,
    _user: CurrentUser,
    Json(data): Json<ItemRequest>,
) -> Result<Json<ItemResponse>, AppError> {
    let item = state.database.get_item(&data.item_id)?;  // Propagates error
    Ok(Json(ItemResponse { ... }))
}

// BAD - Manual error handling in endpoint
async fn get_item(
    State(state): State<AppState>,
    Json(data): Json<ItemRequest>,
) -> Result<Json<ItemResponse>, AppError> {
    match state.database.get_item(&data.item_id) {
        Ok(item) => Ok(Json(ItemResponse { ... })),
        Err(e) => {
            tracing::error!("Error: {}", e);  // Don't do this
            Err(e)
        }
    }
}
```

### Background Tasks: One Top-Level Error Handler

```rust
// GOOD - One top-level error handler
async fn process_queue(state: AppState) {
    loop {
        if let Err(e) = process_queue_inner(&state).await {
            tracing::error!("Error in process_queue: {}", e);
        }
        tokio::time::sleep(Duration::from_secs(60)).await;
    }
}

async fn process_queue_inner(state: &AppState) -> Result<(), AppError> {
    // All operations use ? - no try-catch here
    let item = state.database.get_next_item()?;
    validate_item(&item)?;
    process_item(&item)?;
    Ok(())
}
```

## Cargo.toml Dependencies

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tokio-util = { version = "0.7", features = ["io"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
rusqlite = { version = "0.32", features = ["bundled"] }
r2d2 = "0.8"
r2d2_sqlite = "0.25"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
rand = "0.8"
http-body-util = "0.1"
```

## Key Rules Summary

| Rule | Description |
|------|-------------|
| **No module-level app** | Use `create_application()` factory |
| **Config via AppState** | Access via `State(state): State<AppState>` |
| **Shared state in AppState** | Database, sessions, config |
| **POST for all endpoints** | Except file downloads, streaming |
| **Serde request models** | All POST data validated with `#[derive(Deserialize)]` |
| **Use `?` in endpoints** | Let errors propagate to `AppError` |
| **Log every request** | Format: `INFO POST /path 200 03.98ms {body}` |
| **Session-based auth** | Login returns token, use `X-Auth-Token` header |

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

```rust
#[axum::debug_handler]
async fn action_name(
    State(state): State<AppState>,
    _user: CurrentUser,  // If auth needed
    Json(data): Json<RequestModel>,
) -> Result<Json<ResponseModel>, AppError> {
    // Business logic - use ? for error propagation
    let result = state.database.do_something(&data.param)?;

    Ok(Json(ResponseModel { ... }))
}
```
