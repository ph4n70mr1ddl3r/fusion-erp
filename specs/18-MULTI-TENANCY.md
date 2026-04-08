# 18 - Multi-Tenancy Specification

## 1. Overview

Multi-tenancy enables a single deployment to serve multiple organizations (tenants) with complete data isolation, configurable business rules, and independent administration per tenant.

---

## 2. Tenancy Model

### 2.1 Strategy: Shared Database, Shared Schema, Discriminator Column
- All tenants share the same database schema within each service's SQLite database
- Data isolation enforced via `tenant_id` column on every table
- Application-layer enforcement — no database-level row security (SQLite limitation)
- Every query MUST include `WHERE tenant_id = ?` as the first filter condition

### 2.2 Tenant Registry (auth-service database)
```sql
CREATE TABLE tenants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_code TEXT NOT NULL,             -- Short code: "ACME", "GLOBEX"
    tenant_name TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUSPENDED','TRIAL','CLOSED')),

    -- Configuration
    default_currency_code TEXT NOT NULL DEFAULT 'USD',
    fiscal_year_start_month INTEGER NOT NULL DEFAULT 1,
    date_format TEXT NOT NULL DEFAULT 'YYYY-MM-DD',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    locale TEXT NOT NULL DEFAULT 'en_US',

    -- Limits
    max_users INTEGER NOT NULL DEFAULT 100,
    max_storage_mb INTEGER NOT NULL DEFAULT 1024,

    -- Subscription
    plan TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(plan IN ('TRIAL','STARTER','STANDARD','PROFESSIONAL','ENTERPRISE')),
    subscription_start TEXT,
    subscription_end TEXT,
    trial_ends_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    UNIQUE(tenant_code)
);
```

### 2.3 Tenant Configuration Table
```sql
CREATE TABLE tenant_config (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    config_key TEXT NOT NULL,
    config_value TEXT NOT NULL,            -- JSON string
    config_type TEXT NOT NULL DEFAULT 'STRING'
        CHECK(config_type IN ('STRING','NUMBER','BOOLEAN','JSON')),
    description TEXT,
    is_encrypted INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, config_key)
);
```

---

## 3. Tenant Lifecycle

### 3.1 Provisioning Flow
1. Create tenant record in `tenants` table
2. Create default admin user for the tenant
3. Seed default roles and permissions
4. Seed reference data (currencies, payment terms, tax codes)
5. Seed default chart of accounts template
6. Generate default accounting periods for current fiscal year
7. Activate tenant

### 3.2 Tenant Operations
```
POST   /api/v1/auth/tenants                    — Create tenant (super-admin only)
GET    /api/v1/auth/tenants                     — List tenants (super-admin)
GET    /api/v1/auth/tenants/{id}                — Get tenant details
PUT    /api/v1/auth/tenants/{id}                — Update tenant settings
PATCH  /api/v1/auth/tenants/{id}/suspend        — Suspend tenant
PATCH  /api/v1/auth/tenants/{id}/activate       — Reactivate tenant
GET    /api/v1/auth/tenants/{id}/usage          — Get tenant usage stats
GET/PUT /api/v1/auth/tenants/{id}/config        — Tenant configuration
```

---

## 4. Data Isolation Enforcement

### 4.1 Application-Layer Isolation
- Auth middleware extracts `tenant_id` from JWT `tid` claim
- Tenant ID injected into request context (axum extension)
- Repository layer MUST accept `tenant_id` as first parameter
- All SQL queries MUST include `WHERE tenant_id = ?1`

### 4.2 Validation Middleware
```rust
pub async fn tenant_middleware(
    axum::extract::Request req: Request,
    next: Next,
) -> Result<Response, (StatusCode, Json<ApiError>)> {
    let claims = req.extensions().get::<TokenClaims>()
        .ok_or((StatusCode::UNAUTHORIZED, /* ... */))?;

    let tenant_header = req.headers()
        .get("X-Tenant-Id")
        .and_then(|v| v.to_str().ok());

    // Verify tenant header matches JWT claim
    if let Some(header_tid) = tenant_header {
        if header_tid != claims.tid {
            return Err((StatusCode::FORBIDDEN, /* ... */));
        }
    }

    // Verify tenant exists and is active
    // ... database lookup ...

    Ok(next.run(req).await)
}
```

### 4.3 Index Design for Multi-Tenancy
Every index MUST include `tenant_id` as the leftmost column:
```sql
-- CORRECT
CREATE INDEX idx_accounts_tenant_code ON gl_accounts(tenant_id, account_code);
CREATE INDEX idx_invoices_tenant_date ON ap_invoices(tenant_id, invoice_date);

-- WRONG — missing tenant_id
CREATE INDEX idx_accounts_code ON gl_accounts(account_code);
```

### 4.4 Cross-Tenant Prevention
- Services MUST NOT expose endpoints that operate across tenants
- Bulk operations MUST be scoped to a single tenant
- Admin reports (super-admin) MUST explicitly request cross-tenant data

---

## 5. Tenant Configuration

### 5.1 Standard Configuration Keys

| Key | Default | Description |
|-----|---------|-------------|
| `finance.default_currency` | USD | Default accounting currency |
| `finance.fiscal_year_start` | 1 | Fiscal year start month (1=January) |
| `finance.journal_number_prefix` | JE- | Journal entry number prefix |
| `gl.auto_period_close` | false | Automatically close periods at month end |
| `ap.three_way_match` | true | Require 3-way match for PO invoices |
| `ap.match_tolerance_percent` | 1 | Invoice-PO match tolerance |
| `ap.match_tolerance_cents` | 500 | Invoice-PO match tolerance in cents |
| `ar.auto_credit_check` | true | Check customer credit on invoice |
| `ar.dunning_enabled` | false | Enable automatic dunning |
| `inv.costing_method` | WEIGHTED_AVG | Default item costing method |
| `inv.negative_stock_allowed` | false | Allow negative inventory |
| `inv.auto_reorder` | false | Auto-generate requisitions at reorder point |
| `workflow.approval_required_amount` | 0 | Amount above which approval required |
| `ui.date_format` | YYYY-MM-DD | Display date format |
| `ui.decimal_places` | 2 | Number of decimal places for amounts |

### 5.2 Per-Tenant Feature Flags
```json
{
  "features": {
    "multi_currency": true,
    "budget_management": true,
    "project_billing": false,
    "manufacturing": false,
    "bank_reconciliation": true,
    "fixed_assets": true
  }
}
```

---

## 6. Tenant Data Seeding

### 6.1 Default Data Created per Tenant
1. **System Roles:** SuperAdmin, TenantAdmin, FinanceManager, Accountant, APManager, ARManager, ReadOnly, etc.
2. **Permissions:** All standard `{service}.{resource}.{action}` combinations
3. **Reference Data:**
   - Currencies (USD, EUR, GBP, CAD, etc.)
   - Payment terms (Net 30, Net 60, Due on Receipt, etc.)
   - Tax codes (standard rates)
   - Units of measure (EA, KG, M, L, etc.)
4. **Chart of Accounts Template:**
   - Standard account structure: Assets (1000), Liabilities (2000), Equity (3000), Revenue (4000), Expenses (5000)
5. **Accounting Periods:** 12 monthly periods for current fiscal year
6. **Default User:** Admin user with temporary password

---

## 7. Tenant Isolation Testing

### 7.1 Verification Tests
Every service MUST include tests that verify:
1. User from Tenant A cannot read Tenant B's data
2. User from Tenant A cannot modify Tenant B's data
3. Queries with tenant_id filter return only matching records
4. Cross-tenant ID reference returns 404 (not the other tenant's data)
