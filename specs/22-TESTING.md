# 22 - Testing Specification

## 1. Overview

Comprehensive testing strategy covering unit tests, integration tests, contract tests, and end-to-end tests for the entire ERP system.

---

## 2. Testing Tools

| Tool | Purpose |
|------|---------|
| `cargo nextest` | Test runner (faster than built-in) |
| `proptest` | Property-based testing |
| `tokio::test` | Async test runtime |
| `tower::ServiceExt` | Axum handler testing |
| `tonic::test` | gRPC service testing |
| Custom test harness | Integration test orchestration |

---

## 3. Test Categories

### 3.1 Unit Tests
- Location: `src/**/*.rs` within each service crate (inline `#[cfg(test)]` modules)
- Scope: Individual functions, domain logic, validation, calculations
- Dependencies: Mocked (no database, no network)
- Run: `cargo nextest run -p {service}`

**Example — GL balance calculation:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_ending_balance_asset_account() {
        let beginning = 1_000_000; // $10,000.00
        let debits = 500_000;       // $5,000.00
        let credits = 200_000;      // $2,000.00
        // Assets: ending = beginning + debits - credits
        let ending = calculate_ending_balance(beginning, debits, credits, AccountType::Asset);
        assert_eq!(ending, 1_300_000); // $13,000.00
    }

    #[test]
    fn test_journal_balances() {
        let lines = vec![
            JournalLine { debit: 1000, credit: 0 },
            JournalLine { debit: 0, credit: 1000 },
        ];
        assert!(is_balanced(&lines));
    }

    #[test]
    fn test_unbalanced_journal_rejected() {
        let lines = vec![
            JournalLine { debit: 1000, credit: 0 },
            JournalLine { debit: 0, credit: 900 },
        ];
        assert!(!is_balanced(&lines));
    }
}
```

### 3.2 Repository Tests (Database Layer)
- Location: `tests/` directory within each service crate
- Scope: SQL queries, transactions, migrations
- Dependencies: Real SQLite in-memory database
- Each test creates a fresh in-memory database with migrations applied

**Example:**
```rust
#[tokio::test]
async fn test_create_and_get_account() {
    let pool = create_test_pool(); // :memory: database
    run_migrations(&pool).await.unwrap();

    let repo = GlAccountRepository::new(pool);
    let account = repo.create("tenant-1", NewAccount {
        account_code: "1000-010-0001-0000",
        account_name: "Cash",
        account_type: "ASSET",
        // ...
    }).await.unwrap();

    let fetched = repo.find_by_id("tenant-1", &account.id).await.unwrap().unwrap();
    assert_eq!(fetched.account_code, "1000-010-0001-0000");
    assert_eq!(fetched.tenant_id, "tenant-1");
}

#[tokio::test]
async fn test_tenant_isolation() {
    let pool = create_test_pool();
    run_migrations(&pool).await.unwrap();

    let repo = GlAccountRepository::new(pool);
    // Create account for tenant A
    repo.create("tenant-A", NewAccount { account_code: "1000", ... }).await.unwrap();
    // Create account for tenant B
    repo.create("tenant-B", NewAccount { account_code: "1000", ... }).await.unwrap();

    // Tenant A should see only 1 account
    let accounts_a = repo.find_all("tenant-A", &Filter::default()).await.unwrap();
    assert_eq!(accounts_a.len(), 1);

    // Tenant B should see only 1 account
    let accounts_b = repo.find_all("tenant-B", &Filter::default()).await.unwrap();
    assert_eq!(accounts_b.len(), 1);
}
```

### 3.3 API Handler Tests
- Location: `tests/api/` within each service
- Scope: HTTP request/response handling, middleware, status codes
- Dependencies: In-memory database, axum test harness
- Tool: `axum::Router` with `tower::ServiceExt`

**Example:**
```rust
#[tokio::test]
async fn test_create_account_returns_201() {
    let app = create_test_app().await; // Sets up router with test state
    let token = create_test_token("tenant-1", vec!["gl.accounts.create"]);

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/v1/gl/accounts")
                .header("Authorization", format!("Bearer {}", token))
                .header("X-Tenant-Id", "tenant-1")
                .header("Content-Type", "application/json")
                .body(Body::from(r#"{"account_code":"1000","account_name":"Cash","account_type":"ASSET"}"#))
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);
}

#[tokio::test]
async fn test_unauthenticated_returns_401() {
    let app = create_test_app().await;

    let response = app
        .oneshot(
            Request::builder()
                .method("GET")
                .uri("/api/v1/gl/accounts")
                .body(Body::empty())
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::UNAUTHORIZED);
}

#[tokio::test]
async fn test_cross_tenant_returns_404() {
    let app = create_test_app().await;
    let token_a = create_test_token("tenant-A", vec!["gl.accounts.read"]);
    // Account exists in tenant B
    let account_id = create_test_account(&app, "tenant-B").await;

    let response = app
        .oneshot(
            Request::builder()
                .method("GET")
                .uri(&format!("/api/v1/gl/accounts/{}", account_id))
                .header("Authorization", format!("Bearer {}", token_a))
                .header("X-Tenant-Id", "tenant-A")
                .body(Body::empty())
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::NOT_FOUND);
}
```

### 3.4 gRPC Contract Tests
- Location: `tests/grpc/` within each service
- Scope: gRPC service implementations
- Dependencies: In-memory database, tonic test client

**Example:**
```rust
#[tokio::test]
async fn test_gl_create_journal_grpc() {
    let (mut client, _guard) = create_test_grpc_client::<GeneralLedgerServiceClient<Channel>>().await;

    let response = client.create_journal(CreateJournalRequest {
        tenant_id: Some(TenantId { value: "tenant-1".into() }),
        description: "Test journal".into(),
        journal_date: "2024-01-15".into(),
        currency_code: "USD".into(),
        lines: vec![
            JournalLineInput { account_id: "acc-1".into(), entered_debit_cents: 1000, ..Default::default() },
            JournalLineInput { account_id: "acc-2".into(), entered_credit_cents: 1000, ..Default::default() },
        ],
        ..Default::default()
    }).await.unwrap();

    assert!(!response.get_ref().journal_id.is_empty());
    assert!(!response.get_ref().journal_number.is_empty());
}
```

---

## 4. Integration Tests

### 4.1 Cross-Service Integration Tests
- Location: `/tests/integration/`
- Scope: End-to-end business flows across multiple services
- Dependencies: All services running
- Run: `cargo nextest run --test integration`

**Example — Procure-to-Pay Flow:**
```rust
#[tokio::test]
async fn test_procure_to_pay_flow() {
    // 1. Create supplier (AP)
    let supplier = ap_client.create_supplier("tenant-1", NewSupplier { ... }).await.unwrap();

    // 2. Create requisition (Proc)
    let req = proc_client.create_requisition("tenant-1", NewRequisition { ... }).await.unwrap();

    // 3. Approve and convert to PO (Proc)
    let po = proc_client.approve_and_convert("tenant-1", &req.id).await.unwrap();

    // 4. Receive goods (Proc + INV)
    let receipt = proc_client.receive_goods("tenant-1", &po.id, vec![...]).await.unwrap();
    let stock = inv_client.get_stock("tenant-1", item_id).await.unwrap();
    assert!(stock.quantity_on_hand > 0);

    // 5. Create and post invoice (AP)
    let invoice = ap_client.create_invoice("tenant-1", NewInvoice { po_id: po.id, ... }).await.unwrap();
    ap_client.post_invoice("tenant-1", &invoice.id).await.unwrap();

    // 6. Verify GL entries (GL)
    let journal = gl_client.get_journal_by_reference("tenant-1", "AP_INVOICE", &invoice.id).await.unwrap();
    assert_eq!(journal.status, "POSTED");

    // 7. Create payment (AP)
    let payment = ap_client.create_payment("tenant-1", NewPayment { invoice_id: invoice.id, ... }).await.unwrap();

    // 8. Verify invoice is paid
    let invoice = ap_client.get_invoice("tenant-1", &invoice.id).await.unwrap();
    assert_eq!(invoice.status, "PAID");
}
```

### 4.2 Integration Test Matrix

| Flow | Services | Test |
|------|----------|------|
| Procure-to-Pay | Proc → INV → AP → GL | Full PO to payment cycle |
| Order-to-Cash | OM → INV → AR → GL | Full order to receipt cycle |
| Fixed Asset Lifecycle | AP → FA → GL | Asset purchase through disposal |
| Cash Reconciliation | AR/CM → CM → GL | Receipt to bank reconciliation |
| Project Delivery | PM → AR → GL | Time entry to project billing |
| Period Close | GL | Post journals, close period, verify balances |
| Multi-tenant Isolation | All | Cross-tenant access returns 404 |

---

## 5. Test Utilities

### 5.1 Test Harness
```rust
// tests/common/mod.rs
pub struct TestHarness {
    pub pool: Pool<SqliteConnectionManager>,
    pub auth_state: MockAuthState,
}

impl TestHarness {
    pub async fn new() -> Self {
        let pool = create_in_memory_pool();
        run_migrations(&pool).unwrap();
        Self { pool, auth_state: MockAuthState::default() }
    }

    pub fn create_token(&self, tenant_id: &str, permissions: Vec<&str>) -> String {
        generate_test_jwt(tenant_id, permissions)
    }
}

pub fn create_in_memory_pool() -> Pool<SqliteConnectionManager> {
    let manager = SqliteConnectionManager::memory()
        .with_init(|conn| {
            conn.execute_batch("PRAGMA foreign_keys=ON;")?;
            Ok(())
        });
    Pool::builder().max_size(4).build(manager).unwrap()
}
```

### 5.2 Test Data Factories
```rust
pub fn new_account(tenant_id: &str) -> NewAccount {
    NewAccount {
        tenant_id: tenant_id.into(),
        account_code: "1000-010-0001-0000".into(),
        account_name: "Test Cash Account".into(),
        account_type: "ASSET".into(),
        is_postable: true,
        ..Default::default()
    }
}

pub fn new_journal(tenant_id: &str, account_id: &str) -> NewJournal {
    NewJournal {
        tenant_id: tenant_id.into(),
        description: "Test journal".into(),
        journal_date: "2024-01-15".into(),
        lines: vec![
            NewJournalLine { account_id: account_id.into(), debit: 100000, credit: 0 },
            NewJournalLine { account_id: account_id.into(), debit: 0, credit: 100000 },
        ],
    }
}
```

---

## 6. Coverage Requirements

### 6.1 Minimum Coverage Targets
| Layer | Coverage |
|-------|----------|
| Domain logic (calculations, validations) | 90% |
| Repository (SQL queries) | 80% |
| API handlers | 80% |
| gRPC services | 70% |
| Integration flows | All critical paths |

### 6.2 Critical Path Tests (MUST pass before merge)
- GL: journal creation, posting, balancing, period close
- AP: invoice creation, approval, posting, payment
- AR: invoice creation, posting, receipt application
- Auth: login, token refresh, permission check, tenant isolation
- Multi-tenant: cross-tenant access prevention
