# 02 - Technology Stack Specification

## 1. Language & Runtime

### 1.1 Rust Configuration
- **Edition:** 2021
- **Minimum Supported Rust Version (MSRV):** 1.82.0
- **Compilation Profile:** Release with optimizations for production
- **Linting:** cargo clippy with `-D warnings` (deny warnings)
- **Formatting:** rustfmt with default settings, enforced in CI

### 1.2 Async Runtime
- **tokio** 1.x with `features = ["full"]`
- Multi-threaded scheduler (default)
- All I/O operations MUST be async
- Blocking operations (SQLite writes) MUST use `tokio::task::spawn_blocking`

### 1.3 Workspace Structure

```toml
# /Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = [
    "crates/fusion-core",
    "crates/fusion-db",
    "crates/fusion-auth",
    "crates/fusion-client",
    "crates/fusion-proto",
    "crates/fusion-ui",
    "services/auth-service",
    "services/gateway-service",
    "services/gl-service",
    "services/ap-service",
    "services/ar-service",
    "services/fa-service",
    "services/cm-service",
    "services/proc-service",
    "services/inv-service",
    "services/om-service",
    "services/mfg-service",
    "services/pm-service",
    "services/workflow-service",
    "services/report-service",
]

[workspace.dependencies]
# Async runtime
tokio = { version = "1", features = ["full"] }

# Web framework
axum = "0.8"
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace", "request-id", "compression-gzip", "limit"] }

# gRPC
tonic = "0.12"
prost = "0.13"
tonic-build = "0.12"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Database
rusqlite = { version = "0.32", features = ["bundled", "column_decltype"] }
r2d2 = "0.8"
r2d2_sqlite = "0.25"

# Auth
jsonwebtoken = "9"
bcrypt = "0.16"

# Observability
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }

# Utilities
uuid = { version = "1", features = ["v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
validator = { version = "0.19", features = ["derive"] }
rand = "0.8"
hex = "0.4"
sha2 = "0.10"

# Metrics
prometheus = "0.13"
lazy_static = "1"

# Testing
proptest = "1"

# Frontend (WASM)
leptos = { version = "0.7", features = ["csr"] }
wasm-bindgen = "0.2"
web-sys = "0.3"
```

---

## 2. Backend Framework (per Service)

### 2.1 HTTP Layer — axum 0.8+
Each service exposes a REST API using axum with this standard setup:

```rust
// Standard axum app setup per service
use axum::{Router, middleware};
use tower_http::cors::CorsLayer;
use tower_http::trace::TraceLayer;
use tower_http::request_id::RequestIdLayer;

fn app(state: AppState) -> Router {
    Router::new()
        .merge(health_routes())
        .nest("/api/v1/{service}", api_routes())
        .layer(CorsLayer::permissive()) // configured per environment
        .layer(TraceLayer::new_for_http())
        .layer(RequestIdLayer::new())
        .with_state(state)
}
```

**Axum middleware ordering (outer to inner):**
1. `RequestIdLayer` — assign unique request ID
2. `TraceLayer` — request/response logging
3. `CorsLayer` — CORS headers
4. Auth middleware (custom axum middleware) — JWT validation
5. Tenant middleware — extract and validate tenant_id
6. Route handlers

### 2.2 gRPC Layer — tonic 0.12+
Each service exposes gRPC endpoints for inter-service communication:

```rust
// Standard tonic server setup per service
use tonic::transport::Server;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = format!("0.0.0.0:{}", config.service.port).parse()?;

    // Separate ports for HTTP and gRPC
    let grpc_addr = format!("0.0.0.0:{}", config.service.port + 1000).parse()?;

    // Run both servers concurrently
    tokio::spawn(async move {
        Server::builder()
            .add_service(GlServiceServer::new(gl_impl))
            .serve(grpc_addr)
            .await
            .unwrap();
    });

    // HTTP server on main port
    let listener = tokio::net::TcpListener::bind(addr).await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

### 2.3 Serialization
- **serde** + **serde_json** for all JSON serialization
- **prost** for protobuf serialization (via tonic)
- All domain types MUST derive `Serialize` and `Deserialize`
- Custom serialization for special types (Money, TenantId) via `serde_with` or manual impl

### 2.4 Validation
- **validator** crate with derive macros
- All input structs MUST derive `Validate`
- Custom validators for business rules

```rust
use validator::Validate;

#[derive(serde::Deserialize, Validate)]
pub struct CreateJournalRequest {
    #[validate(length(min = 1, max = 255))]
    pub description: String,
    #[validate(custom(function = "validate_date"))]
    pub journal_date: String,
    #[validate(length(min = 2))]
    pub lines: Vec<CreateJournalLineRequest>,
}
```

---

## 3. Database Layer

### 3.1 SQLite via rusqlite 0.32+
- **rusqlite** with `bundled` feature (no system SQLite dependency)
- Direct SQL — no ORM
- Parameterized queries only — MUST NOT use string formatting for SQL

```rust
use rusqlite::{params, Connection, Result};

pub fn get_account(conn: &Connection, tenant_id: &str, id: &str) -> Result<Account> {
    conn.query_row(
        "SELECT id, tenant_id, account_code, account_name, account_type
         FROM gl_accounts
         WHERE tenant_id = ?1 AND id = ?2 AND is_active = 1",
        params![tenant_id, id],
        |row| {
            Ok(Account {
                id: row.get(0)?,
                tenant_id: row.get(1)?,
                account_code: row.get(2)?,
                account_name: row.get(3)?,
                account_type: row.get(4)?,
            })
        }
    )
}
```

### 3.2 Connection Pooling
- **r2d2** with **r2d2_sqlite** for connection pooling
- Pool configuration per service

```rust
use r2d2::Pool;
use r2d2_sqlite::SqliteConnectionManager;

pub fn create_pool(db_path: &str) -> Pool<SqliteConnectionManager> {
    let manager = SqliteConnectionManager::file(db_path)
        .with_init(|conn| {
            conn.execute_batch("PRAGMA journal_mode=WAL;
                               PRAGMA foreign_keys=ON;
                               PRAGMA busy_timeout=5000;
                               PRAGMA synchronous=NORMAL;
                               PRAGMA mmap_size=268435456;
                               PRAGMA cache_size=-64000;")?;
            Ok(())
        });

    Pool::builder()
        .max_size(8)
        .min_idle(Some(2))
        .build(manager)
        .expect("Failed to create connection pool")
}
```

### 3.3 Migration Framework
- Custom migration runner embedded in each service
- Migration files: `migrations/{service}/V{number}_{name}.up.sql`
- Migrations run on service startup before accepting requests
- Each migration tracked in `_migrations` table

```rust
use std::sync::LazyLock;

static MIGRATIONS: LazyLock<Vec<Migration>> = LazyLock::new(|| {
    vec![
        Migration::new(1, "create_gl_accounts", include_str!("../../migrations/gl/V001_create_gl_accounts.up.sql")),
        Migration::new(2, "create_gl_journals", include_str!("../../migrations/gl/V002_create_gl_journals.up.sql")),
    ]
});

pub fn run_migrations(conn: &Connection) -> Result<()> {
    conn.execute_batch("CREATE TABLE IF NOT EXISTS _migrations (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        applied_at TEXT NOT NULL DEFAULT (datetime('now')),
        checksum TEXT NOT NULL
    )")?;

    for migration in MIGRATIONS.iter() {
        let applied: bool = conn.query_row(
            "SELECT COUNT(*) > 0 FROM _migrations WHERE id = ?1",
            params![migration.id],
            |row| row.get(0),
        )?;

        if !applied {
            conn.execute_batch(&migration.sql)?;
            conn.execute(
                "INSERT INTO _migrations (id, name, checksum) VALUES (?1, ?2, ?3)",
                params![migration.id, migration.name, migration.checksum()],
            )?;
        }
    }
    Ok(())
}
```

### 3.4 Write Concurrency
SQLite is single-writer. The service MUST handle this via:
- `tokio::task::spawn_blocking` for all database write operations
- Application-level write queue if write contention is observed
- WAL mode allows concurrent readers during writes

```rust
pub async fn create_journal(pool: &Pool, journal: NewJournal) -> Result<Journal> {
    let pool = pool.clone();
    tokio::task::spawn_blocking(move || {
        let mut conn = pool.get()?;
        let tx = conn.transaction()?;
        // ... insert journal and lines ...
        tx.commit()?;
        Ok(journal)
    }).await?
}
```

---

## 4. Frontend

### 4.1 Framework — Leptos 0.7+
- Full-stack Rust UI framework with CSR (Client-Side Rendering) mode
- Reactive signals for state management
- Component-based architecture
- Compiles to WebAssembly

### 4.2 UI Components
- Shared component library in `crates/fusion-ui`
- Components: DataTable, Form, Modal, Tabs, Sidebar, Breadcrumb, DatePicker, FileUpload, Chart

### 4.3 Styling
- Tailwind CSS via CDN (loaded in index.html)
- Utility-first CSS — no custom CSS files required
- Dark/light theme via CSS variables and Tailwind's dark mode

### 4.4 State Management
- Leptos signals (`create_signal`, `create_rw_signal`)
- Global app state via Leptos context
- Per-page state via local signals
- Server state synchronized via fetch API calls to gateway

### 4.5 Routing
- Leptos router for client-side routing
- Route structure mirrors API service structure:
  - `/dashboard`
  - `/gl/accounts`, `/gl/journals`, `/gl/periods`, `/gl/budgets`
  - `/ap/suppliers`, `/ap/invoices`, `/ap/payments`
  - `/ar/customers`, `/ar/invoices`, `/ar/receipts`
  - `/inv/items`, `/inv/warehouses`, `/inv/movements`
  - etc.

---

## 5. Shared Crates

### 5.1 fusion-core
Common types, traits, and error handling shared across all services.

**Key types:**
```rust
// Domain primitives
pub struct TenantId(pub String);
pub struct UserId(pub String);
pub struct Money { pub cents: i64, pub currency: String }
pub struct Pagination { pub cursor: Option<String>, pub limit: u32 }

// Standard result type
pub type FusionResult<T> = Result<T, FusionError>;

// Error hierarchy
pub enum FusionError {
    NotFound(String),
    Validation(Vec<FieldError>),
    Conflict(String),
    Unauthorized(String),
    Forbidden(String),
    Internal(String),
    Database(String),
    ServiceUnavailable(String),
}

// Standard API response envelope
pub struct ApiResponse<T> {
    pub data: T,
    pub meta: ResponseMeta,
}

pub struct ApiError {
    pub error: ErrorDetail,
}

// Standard model fields
pub struct AuditFields {
    pub created_at: String,
    pub updated_at: String,
    pub created_by: String,
    pub updated_by: String,
    pub version: i64,
    pub is_active: bool,
}
```

### 5.2 fusion-db
Database utilities shared across all services.

**Key features:**
- Connection pool creation with standard PRAGMAs
- Migration runner framework
- Transaction helper functions
- Standard column mapping helpers
- Tenant-scoped query builder base

### 5.3 fusion-auth
Authentication primitives shared across all services.

**Key features:**
- JWT token creation and validation
- Claims structure definition
- Auth middleware for axum
- Permission checking utilities
- RBAC type definitions (Role, Permission)

### 5.4 fusion-client
Generated API client for inter-service communication.

**Key features:**
- gRPC client wrappers for each service
- Connection pooling for gRPC channels
- Retry logic with circuit breaker
- Error mapping from gRPC status to FusionError

### 5.5 fusion-proto
Shared protobuf definitions.

**Key features:**
- Common message types (Money, Pagination, TenantId)
- Service proto definitions
- Build script for code generation

### 5.6 fusion-ui
Shared Leptos UI components.

**Key features:**
- DataTable with sorting, filtering, pagination
- Form components (TextInput, Select, DatePicker, etc.)
- Layout components (Sidebar, Header, PageContainer)
- Feedback components (Toast, Modal, ConfirmationDialog)
- Chart components (Bar, Line, Pie)

---

## 6. Observability

### 6.1 Structured Logging
```rust
use tracing::{info, error, instrument};
use tracing_subscriber::{fmt, EnvFilter};

// Initialize at service startup
tracing_subscriber::fmt()
    .json()
    .with_env_filter(EnvFilter::from_default_env().add_directive("info".parse().unwrap()))
    .init();

// Usage
#[instrument(skip(pool), fields(tenant_id = %req.tenant_id))]
pub async fn create_journal(pool: &Pool, req: CreateJournalRequest) -> FusionResult<Journal> {
    info!(journal_number = %number, "Creating journal entry");
    // ...
}
```

### 6.2 Metrics
- Prometheus format exposed on `/metrics` endpoint
- Standard HTTP metrics via `tower-http` metrics middleware
- Custom business metrics per service

### 6.3 Health Checks
- `/health` — liveness (always returns 200 if process is alive)
- `/ready` — readiness (checks database connectivity, returns 503 if unhealthy)

---

## 7. Build & Tooling

### 7.1 Task Runner — just
```make
# justfile

# Build all services
build:
    cargo build --workspace

# Build a specific service
build-service service:
    cargo build -p {{service}}

# Run all tests
test:
    cargo nextest run --workspace

# Run tests for a specific service
test-service service:
    cargo nextest run -p {{service}}

# Lint
lint:
    cargo clippy --workspace -- -D warnings

# Format check
fmt-check:
    cargo fmt --check

# Format
fmt:
    cargo fmt

# Run a specific service
run service:
    cargo run -p {{service}}

# Generate proto files
proto:
    cargo run -p fusion-proto-build
```

### 7.2 CI Pipeline
1. `cargo fmt --check`
2. `cargo clippy --workspace -- -D warnings`
3. `cargo nextest run --workspace`
4. Build Docker images for changed services

### 7.3 Development Setup
```bash
# Prerequisites
rustup toolchain install 1.82.0
rustup default 1.82.0
cargo install cargo-nextest
cargo install just

# Build
just build

# Run auth service + gateway
just run auth-service &
just run gateway-service &
```

---

## 8. Cargo.toml Template (per Service)

```toml
[package]
name = "gl-service"
version = "0.1.0"
edition = "2021"

[dependencies]
fusion-core = { path = "../../crates/fusion-core" }
fusion-db = { path = "../../crates/fusion-db" }
fusion-auth = { path = "../../crates/fusion-auth" }
fusion-proto = { path = "../../crates/fusion-proto" }
fusion-client = { path = "../../crates/fusion-client" }
tokio = { workspace = true }
axum = { workspace = true }
tonic = { workspace = true }
prost = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
rusqlite = { workspace = true }
r2d2 = { workspace = true }
r2d2_sqlite = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
validator = { workspace = true }
tower = { workspace = true }
tower-http = { workspace = true }
prometheus = { workspace = true }

[dev-dependencies]
proptest = { workspace = true }
```
