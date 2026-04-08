# AI Agent Implementation Guide

## 1. Purpose

This guide instructs AI agents on how to consume the specification documents in this directory to build the Fusion ERP system incrementally, following spec-driven development.

---

## 2. How to Read Specs

### 2.1 Spec Reading Order
Read specs in this exact order for each stage:

**Stage 0 — Foundation (Build First):**
1. `00-OVERVIEW.md` — Understand the big picture, module list, build order
2. `02-TECH-STACK.md` — Set up workspace, dependencies, tooling
3. `01-ARCHITECTURE.md` — Understand service decomposition and communication
4. `04-DATABASE.md` — Understand DB conventions, standard columns, migration patterns
5. `05-AUTH-SECURITY.md` — Implement auth service (everything depends on auth)

**Stage 1 — Core Financials:**
6. `06-GENERAL-LEDGER.md` — Build GL first (all financial modules depend on it)
7. `07-ACCOUNTS-PAYABLE.md`
8. `08-ACCOUNTS-RECEIVABLE.md`
9. `09-FIXED-ASSETS.md`
10. `10-CASH-MANAGEMENT.md`

**Stage 2 — Supply Chain:**
11. `11-PROCUREMENT.md`
12. `12-INVENTORY.md`
13. `13-ORDER-MANAGEMENT.md`
14. `14-MANUFACTURING.md`

**Stage 3 — Operations:**
15. `15-PROJECT-MANAGEMENT.md`
16. `16-WORKFLOW.md`

**Stage 4 — Financial Extensions:**
17. `23-TAX-MANAGEMENT.md` — Tax engine (AP/AR depend on tax calculation)
18. `24-INTERCOMPANY.md` — Intercompany transactions
19. `25-EXPENSE-MANAGEMENT.md` — Employee expenses, corporate cards
20. `26-REVENUE-MANAGEMENT.md` — ASC 606 revenue recognition
21. `31-LEASE-ACCOUNTING.md` — IFRS 16/ASC 842 lease accounting

**Stage 5 — Supply Chain Extensions:**
22. `27-ADVANCED-PRICING.md` — Price lists, promotions, discount engine
23. `28-SUPPLY-CHAIN-PLANNING.md` — MRP, demand forecasting
24. `32-COLLECTIONS-CREDIT.md` — Collections, credit scoring, dunning

**Stage 6 — Cross-Cutting Services & Delivery:**
25. `29-DOCUMENT-MANAGEMENT.md` — Universal file attachments
26. `30-DATA-IMPORT-EXPORT.md` — Bulk data loading, templates
27. `17-REPORTING.md`
28. `18-MULTI-TENANCY.md`
29. `19-INTEGRATION.md`
30. `20-FRONTEND.md`
31. `21-DEPLOYMENT.md`
32. `22-TESTING.md`

### 2.2 How to Read Each Spec
Every module spec follows this structure:
1. **Domain Overview** — What this service does
2. **Database Schema** — CREATE TABLE statements (copy directly as migrations)
3. **REST API Endpoints** — Method, path, request/response, permissions
4. **Business Rules** — Validation logic, status flows, calculations
5. **gRPC Service** — Proto definitions for inter-service calls
6. **Inter-Service Integration** — What events to publish/consume
7. **Events** — Event names and payloads

---

## 3. Implementation Process per Service

### 3.1 Step-by-Step Checklist

For each service, follow these steps IN ORDER:

```
□ 1. Create Cargo.toml for the service crate
□ 2. Create migration files from the spec's "Database Schema" section
□ 3. Create the database repository layer (SQL queries)
□ 4. Create domain types (structs matching table schemas)
□ 5. Implement business rules and validation
□ 6. Create gRPC service implementation (proto → Rust)
□ 7. Create REST API handlers (axum routes)
□ 8. Wire up middleware (auth, tenant, audit)
□ 9. Implement event publishing (outbox pattern)
□ 10. Implement event subscription (inbox pattern)
□ 11. Write unit tests for domain logic
□ 12. Write repository tests (in-memory SQLite)
□ 13. Write API handler tests
□ 14. Write gRPC contract tests
□ 15. Verify all tests pass
□ 16. Test multi-tenant isolation
```

### 3.2 File Structure per Service
```
services/{service-name}/
├── Cargo.toml
├── src/
│   ├── main.rs                    # Entry point, config, server startup
│   ├── config.rs                  # Configuration loading
│   ├── db/
│   │   ├── mod.rs
│   │   ├── pool.rs                # Connection pool creation
│   │   └── migrations.rs          # Migration runner
│   ├── models/
│   │   ├── mod.rs
│   │   ├── {entity}.rs            # Domain structs per table
│   │   └── ...
│   ├── repository/
│   │   ├── mod.rs
│   │   ├── {entity}_repo.rs       # SQL queries per table
│   │   └── ...
│   ├── service/
│   │   ├── mod.rs
│   │   ├── {entity}_service.rs    # Business logic
│   │   └── ...
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── {entity}_handler.rs    # Axum route handlers
│   │   └── ...
│   ├── grpc/
│   │   ├── mod.rs
│   │   └── {service}_grpc.rs      # Tonic gRPC implementation
│   ├── events/
│   │   ├── mod.rs
│   │   ├── publisher.rs           # Event outbox publisher
│   │   └── subscriber.rs          # Event inbox subscriber
│   ├── middleware/
│   │   ├── mod.rs
│   │   ├── auth.rs                # JWT validation
│   │   ├── tenant.rs              # Tenant extraction
│   │   └── audit.rs               # Audit logging
│   └── routes.rs                  # Route definitions
├── migrations/
│   ├── V001_{name}.up.sql
│   ├── V001_{name}.down.sql
│   └── ...
└── tests/
    ├── api_tests.rs
    ├── repo_tests.rs
    └── grpc_tests.rs
```

---

## 4. Implementation Rules

### 4.1 MUST Follow
- Every table MUST have the standard columns (id, tenant_id, created_at, updated_at, created_by, updated_by, version, is_active)
- Every query MUST include tenant_id in WHERE clause
- Every monetary value MUST be stored as INTEGER cents
- Every API endpoint MUST check permissions before executing
- Every mutation MUST log to audit trail
- Every migration MUST have up AND down files
- Every gRPC call MUST have timeout and retry logic
- Every service MUST expose /health and /ready endpoints

### 4.2 MUST NOT
- MUST NOT use FLOAT/REAL for money (use INTEGER cents)
- MUST NOT build SQL via string concatenation (use parameterized queries)
- MUST NOT skip tenant_id in queries
- MUST NOT create cross-service database dependencies
- MUST NOT store passwords in plaintext
- MUST NOT add features not in the specs (YAGNI)

### 4.3 Code Style
- Use `thiserror` for error types, not anyhow
- Use `#[instrument]` for tracing on all handlers and service methods
- Use Rust idioms: `Result<T, E>`, `Option<T>`, pattern matching
- No `unwrap()` in production code — use `?` and proper error handling
- No `clone()` unless necessary — prefer references

---

## 5. Dependencies Between Services

### 5.1 Build Order Dependency Graph
```
auth-service ──────────────────────────────────────────────┐
  │                                                        │
gl-service ────────────────────────────────────────────────┤
  │                                                         │
  ├── ap-service ─── (also depends on: Tax, Workflow) ─────┤
  ├── ar-service ─── (also depends on: Tax, Pricing) ──────┤
  ├── fa-service ───────────────────────────────────────────┤
  ├── cm-service ───────────────────────────────────────────┤
  │                                                         │
inv-service ───────────────────────────────────────────────┤
  │                                                         │
  ├── proc-service ─── (also depends on: AP, Tax, Workflow)┤
  ├── om-service ───── (also depends on: AR, Pricing, INV) │
  ├── mfg-service ──── (also depends on: GL) ──────────────┤
  │                                                         │
workflow-service ──────────────────────────────────────────┤
  │                                                         │
pm-service ──── (depends on: GL, AR, RevMgmt, Workflow) ──┤
                                                           │
tax-service ────── (depends on: GL) ───────────────────────┤
ic-service ─────── (depends on: GL, AP, AR) ───────────────┤
expense-service ── (depends on: AP, GL, PM, Workflow) ─────┤
rev-service ────── (depends on: AR, OM, PM, GL) ──────────┤
pricing-service ── (depends on: OM, AR, Proc) ─────────────┤
planning-service ─ (depends on: INV, Proc, MFG, OM) ──────┤
dms-service ────── (cross-cutting, no service deps) ───────┤
etl-service ────── (depends on: ALL services for data) ────┤
lease-service ──── (depends on: GL, AP, FA, CM) ──────────┤
collections-service (depends on: AR, OM, GL, Workflow) ────┤
                                                           │
report-service ──── (depends on: ALL services) ────────────┘
```

### 5.2 Shared Crate Build Order
1. `fusion-core` (no dependencies) — Error types, domain primitives
2. `fusion-db` (depends on fusion-core) — SQLite pool, migrations
3. `fusion-auth` (depends on fusion-core) — JWT, RBAC types
4. `fusion-proto` (depends on fusion-core) — Protobuf definitions
5. `fusion-client` (depends on fusion-proto, fusion-auth) — gRPC clients
6. `fusion-ui` (depends on fusion-core) — Leptos components

---

## 6. Testing Strategy

### 6.1 When to Test
- Write unit tests alongside the implementation (TDD preferred)
- Repository tests: test every SQL query with in-memory SQLite
- API tests: test every endpoint with mock auth
- Run `cargo nextest run` after every service completion
- Do NOT proceed to the next service until current service tests pass

### 6.2 Test Priority
1. Multi-tenant isolation (highest — data security)
2. Business rule validation (high — correctness)
3. API contract (high — integration)
4. gRPC contract (medium — inter-service)
5. Edge cases (medium — robustness)

---

## 7. Common Patterns

### 7.1 Creating a New Entity
```rust
// 1. Define model
pub struct NewAccount {
    pub account_code: String,
    pub account_name: String,
    pub account_type: String,
    // ...
}

// 2. Repository function
pub fn create(conn: &Connection, tenant_id: &str, input: &NewAccount, user_id: &str) -> Result<Account> {
    let id = uuid::Uuid::now_v7().to_string();
    conn.execute(
        "INSERT INTO gl_accounts (id, tenant_id, account_code, account_name, account_type, created_by, updated_by)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?6)",
        params![id, tenant_id, input.account_code, input.account_name, input.account_type, user_id],
    )?;
    find_by_id(conn, tenant_id, &id)
}

// 3. Service layer (business rules + validation)
pub async fn create_account(pool: &Pool, tenant_id: &str, input: NewAccount, user_id: &str) -> FusionResult<Account> {
    input.validate()?;
    // Check for duplicate code
    // ...
    let pool = pool.clone();
    tokio::task::spawn_blocking(move || {
        let conn = pool.get()?;
        create(&conn, tenant_id, &input, user_id)
    }).await?
}

// 4. API handler
async fn handle_create(
    State(state): State<AppState>,
    claims: TokenClaims,
    Json(input): Json<NewAccount>,
) -> impl IntoResponse {
    check_permission(&claims, "gl.accounts.create")?;
    let account = create_account(&state.pool, &claims.tid, input, &claims.sub).await?;
    Ok((StatusCode::CREATED, Json(ApiResponse { data: account, meta: default_meta() })))
}
```

### 7.2 Publishing Events
```rust
pub fn publish_event(conn: &Connection, event: OutboxEvent) -> Result<()> {
    conn.execute(
        "INSERT INTO outbox_events (id, tenant_id, event_type, aggregate_type, aggregate_id, payload)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6)",
        params![event.id, event.tenant_id, event.event_type, event.aggregate_type, event.aggregate_id, event.payload_json()],
    )?;
    Ok(())
}
```

---

## 8. Incremental Delivery

### 8.1 Milestone Definitions

**Milestone 0 — Workspace Ready:**
- Cargo workspace builds
- Shared crates compile
- Migrations framework works
- Auth service runs and issues JWTs

**Milestone 1 — GL Operational:**
- GL service: accounts CRUD, journal CRUD, posting, period management
- Tests pass
- Can create chart of accounts and post journal entries

**Milestone 2 — Financial Suite:**
- AP, AR, FA, CM services operational
- Can process full procure-to-pay and order-to-cash flows
- All services post to GL

**Milestone 3 — Supply Chain:**
- Procurement, Inventory, OM, Manufacturing operational
- Full supply chain integration tested

**Milestone 4 — Complete System:**
- All services operational
- Frontend functional
- Integration tests pass
- Multi-tenant isolation verified

### 8.2 Definition of Done per Service
A service is "done" when:
- [ ] All migrations run successfully
- [ ] All CRUD operations implemented
- [ ] All business rules enforced
- [ ] All REST endpoints return correct status codes
- [ ] gRPC endpoints respond correctly
- [ ] Events published for all mutations
- [ ] Multi-tenant isolation tested and verified
- [ ] Unit test coverage >= 80% for domain logic
- [ ] All tests pass: `cargo nextest run -p {service}`
