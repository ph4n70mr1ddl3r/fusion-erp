# 31 - Lease Accounting Service Specification

## 1. Domain Overview

Lease Accounting manages the full lifecycle of leases from both lessee and lessor perspectives, compliant with IFRS 16 and ASC 842. Handles lease classification, right-of-use asset recognition, lease liability amortization, payment scheduling, modifications, reassessments, and disclosures. Integrates with FA for asset tracking, AP for lease payments, CM for cash flows, and GL for journal entries.

**Bounded Context:** Lease Accounting & Management
**Service Name:** `lease-service`
**Database:** `data/lease.db`
**HTTP Port:** 8031 | **gRPC Port:** 9031

---

## 2. Database Schema

### 2.1 Lease Classifications
```sql
CREATE TABLE lease_classification_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    accounting_standard TEXT NOT NULL
        CHECK(accounting_standard IN ('IFRS_16','ASC_842')),
    ownership_transfer INTEGER NOT NULL DEFAULT 0,
    purchase_option_likely INTEGER NOT NULL DEFAULT 0,
    lease_term_majority_percent REAL DEFAULT 75,
    pv_payment_threshold_percent REAL DEFAULT 90,
    specialized_asset INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);
```

### 2.2 Leases
```sql
CREATE TABLE leases (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_number TEXT NOT NULL,             -- LEASE-2024-00001
    lease_name TEXT NOT NULL,
    description TEXT,

    -- Parties
    lessee_entity TEXT,                     -- Internal entity (lessee)
    lessor_id TEXT,                         -- Supplier ID (external lessor)
    lessor_name TEXT,

    -- Role
    role TEXT NOT NULL CHECK(role IN ('LESSEE','LESSOR')),

    -- Classification
    accounting_standard TEXT NOT NULL DEFAULT 'IFRS_16',
    classification TEXT NOT NULL
        CHECK(classification IN ('OPERATING','FINANCE')),
    classification_reason TEXT,

    -- Lease terms
    commencement_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    lease_term_months INTEGER NOT NULL,
    extension_option_months INTEGER,
    extension_option_exercised INTEGER NOT NULL DEFAULT 0,
    termination_option INTEGER NOT NULL DEFAULT 0,

    -- Financial
    discount_rate REAL NOT NULL,            -- Incremental borrowing rate or rate implicit
    total_lease_payments_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    payment_frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(payment_frequency IN ('MONTHLY','QUARTERLY','SEMI_ANNUAL','ANNUAL')),

    -- Initial recognition
    initial_rou_asset_cents INTEGER NOT NULL DEFAULT 0,
    initial_lease_liability_cents INTEGER NOT NULL DEFAULT 0,

    -- Current balances
    current_rou_asset_cents INTEGER NOT NULL DEFAULT 0,
    current_rou_depreciation_cents INTEGER NOT NULL DEFAULT 0,
    current_rou_net_cents INTEGER NOT NULL DEFAULT 0,
    current_lease_liability_cents INTEGER NOT NULL DEFAULT 0,

    -- Exemptions
    short_term_exemption INTEGER NOT NULL DEFAULT 0,
    low_value_exemption INTEGER NOT NULL DEFAULT 0,

    -- Asset details
    asset_description TEXT,
    asset_location TEXT,
    asset_category_id TEXT,

    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','MODIFIED','TERMINATED','EXPIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, lease_number)
);

CREATE INDEX idx_leases_tenant_status ON leases(tenant_id, status);
CREATE INDEX idx_leases_tenant_dates ON leases(tenant_id, commencement_date, end_date);
CREATE INDEX idx_leases_tenant_classification ON leases(tenant_id, classification);
```

### 2.3 Lease Payments
```sql
CREATE TABLE lease_payments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_id TEXT NOT NULL,
    payment_number INTEGER NOT NULL,
    payment_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    payment_type TEXT NOT NULL
        CHECK(payment_type IN ('FIXED','VARIABLE','INDEX_LINKED','PERCENTAGE_REVENUE')),
    gross_payment_cents INTEGER NOT NULL DEFAULT 0,
    principal_cents INTEGER NOT NULL DEFAULT 0,
    interest_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Index linking
    index_type TEXT,                        -- 'CPI', 'RPI', etc.
    index_base_value REAL,
    index_current_value REAL,
    index_adjustment_percent REAL,

    status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(status IN ('SCHEDULED','PAID','OVERDUE','WAIVED')),

    ap_payment_id TEXT,                     -- Link to AP payment

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (lease_id) REFERENCES leases(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, lease_id, payment_number)
);

CREATE INDEX idx_lease_payments_tenant_lease ON lease_payments(tenant_id, lease_id);
CREATE INDEX idx_lease_payments_tenant_date ON lease_payments(tenant_id, due_date);
```

### 2.4 Right-of-Use Assets
```sql
CREATE TABLE right_of_use_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_id TEXT NOT NULL,
    rou_asset_number TEXT NOT NULL,         -- ROU-2024-00001
    asset_name TEXT NOT NULL,

    -- Initial values
    initial_cost_cents INTEGER NOT NULL DEFAULT 0,
    residual_value_cents INTEGER NOT NULL DEFAULT 0,

    -- Depreciation
    depreciation_method TEXT NOT NULL DEFAULT 'STRAIGHT_LINE',
    useful_life_months INTEGER NOT NULL,
    depreciation_start_date TEXT NOT NULL,
    depreciation_end_date TEXT,
    accumulated_depreciation_cents INTEGER NOT NULL DEFAULT 0,
    net_book_value_cents INTEGER NOT NULL DEFAULT 0,

    -- GL accounts
    rou_asset_account_id TEXT NOT NULL,
    accumulated_dep_account_id TEXT NOT NULL,
    depreciation_expense_account_id TEXT NOT NULL,
    interest_expense_account_id TEXT NOT NULL,
    lease_liability_account_id TEXT NOT NULL,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','FULLY_DEPRECIATED','IMPAIRED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (lease_id) REFERENCES leases(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, rou_asset_number)
);
```

### 2.5 Lease Liability Amortization
```sql
CREATE TABLE lease_liability_schedule (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_id TEXT NOT NULL,
    period_date TEXT NOT NULL,
    opening_balance_cents INTEGER NOT NULL DEFAULT 0,
    payment_cents INTEGER NOT NULL DEFAULT 0,
    interest_expense_cents INTEGER NOT NULL DEFAULT 0,
    principal_repayment_cents INTEGER NOT NULL DEFAULT 0,
    closing_balance_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (lease_id) REFERENCES leases(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, lease_id, period_date)
);

CREATE INDEX idx_lease_liability_tenant_lease ON lease_liability_schedule(tenant_id, lease_id);
```

### 2.6 Lease Modifications
```sql
CREATE TABLE lease_modifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_id TEXT NOT NULL,
    modification_number TEXT NOT NULL,
    modification_date TEXT NOT NULL,
    modification_type TEXT NOT NULL
        CHECK(modification_type IN ('SCOPE_INCREASE','SCOPE_DECREASE','LEASE_EXTENSION','PRICE_CHANGE','REEVALUATION')),
    accounting_treatment TEXT NOT NULL
        CHECK(accounting_treatment IN ('SEPARATE_LEASE','PROSPECTIVE','CUMULATIVE_CATCHUP')),

    -- Changes
    old_lease_term_months INTEGER,
    new_lease_term_months INTEGER,
    old_annual_payment_cents INTEGER,
    new_annual_payment_cents INTEGER,
    old_discount_rate REAL,
    new_discount_rate REAL,

    rou_asset_adjustment_cents INTEGER NOT NULL DEFAULT 0,
    liability_adjustment_cents INTEGER NOT NULL DEFAULT 0,
    gain_loss_cents INTEGER,

    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','POSTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (lease_id) REFERENCES leases(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, lease_id, modification_number)
);
```

### 2.7 Lease Incentives
```sql
CREATE TABLE lease_incentives (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_id TEXT NOT NULL,
    incentive_type TEXT NOT NULL
        CHECK(incentive_type IN ('RENT_FREE_PERIOD','TENANT_ALLOWANCE','CASH_PAYMENT','OTHER')),
    description TEXT,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    effective_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (lease_id) REFERENCES leases(id) ON DELETE CASCADE
);
```

---

## 3. REST API Endpoints

```
# Leases
GET/POST      /api/v1/lease/leases                      Permission: lease.leases.read/create
GET/PUT       /api/v1/lease/leases/{id}                  Permission: lease.leases.read/update
POST          /api/v1/lease/leases/{id}/classify         Permission: lease.leases.update
POST          /api/v1/lease/leases/{id}/activate         Permission: lease.leases.update
POST          /api/v1/lease/leases/{id}/terminate        Permission: lease.leases.update

# Lease Payments
GET/POST      /api/v1/lease/leases/{id}/payments         Permission: lease.payments.read/create
GET           /api/v1/lease/payments/{id}                 Permission: lease.payments.read

# Right-of-Use Assets
GET           /api/v1/lease/rou-assets                    Permission: lease.rou-assets.read
GET           /api/v1/lease/rou-assets/{id}               Permission: lease.rou-assets.read
POST          /api/v1/lease/rou-assets/{id}/depreciate    Permission: lease.rou-assets.update

# Liability Schedule
GET           /api/v1/lease/leases/{id}/liability-schedule Permission: lease.schedules.read

# Modifications
GET/POST      /api/v1/lease/leases/{id}/modifications    Permission: lease.modifications.read/create
POST          /api/v1/lease/modifications/{id}/approve   Permission: lease.modifications.approve

# Incentives
GET/POST      /api/v1/lease/leases/{id}/incentives       Permission: lease.incentives.read/create

# Month-End Processing
POST          /api/v1/lease/process/interest             Permission: lease.process.run
POST          /api/v1/lease/process/depreciation         Permission: lease.process.run

# Reports & Disclosures
GET           /api/v1/lease/reports/lease-register       Permission: lease.reports.view
GET           /api/v1/lease/reports/liability-summary    Permission: lease.reports.view
GET           /api/v1/lease/reports/rou-asset-summary    Permission: lease.reports.view
GET           /api/v1/lease/reports/disclosures          Permission: lease.reports.view
GET           /api/v1/lease/reports/expense-breakdown    Permission: lease.reports.view
```

---

## 4. Business Rules

### 4.1 Lease Classification
A lease is classified as **FINANCE** if any criterion is met:
1. Ownership transfers to lessee by end of lease
2. Lessee has option to purchase at below fair value (reasonably certain to exercise)
3. Lease term ≥ 75% of asset's economic life
4. PV of lease payments ≥ 90% of asset's fair value
5. Asset is specialized for lessee's use

Otherwise classified as **OPERATING**.

### 4.2 Initial Recognition (Lessee)

**Finance Lease:**
- ROU Asset = PV of lease payments + initial direct costs + prepayments - incentives
- Lease Liability = PV of future lease payments
- PV calculation using discount rate (rate implicit or incremental borrowing rate)

**Operating Lease (IFRS 16):**
- Same initial recognition as finance (single model under IFRS 16)

**Operating Lease (ASC 842):**
- ROU Asset = PV of lease payments + initial direct costs + prepayments - incentives
- Lease Liability = PV of future lease payments

### 4.3 Subsequent Measurement

**Finance Lease:**
- ROU: Amortize on straight-line basis over lease term (or useful life if ownership transfers)
- Liability: Reduce by principal portion of payment, increase by interest expense

**Operating Lease (ASC 842):**
- Single lease cost = (total lease payments - incentives) / lease term (straight-line)
- Liability reduced by payment, increased by interest
- ROU adjusted for difference between straight-line cost and interest

### 4.4 Month-End Processing
1. Calculate interest on lease liability (opening balance × periodic rate)
2. Split payment into principal and interest
3. Calculate ROU depreciation
4. Post GL journal entries:
   - **Interest:** Debit Interest Expense, Credit Lease Liability
   - **Depreciation:** Debit Depreciation Expense, Credit Accumulated Depreciation
   - **Payment:** Debit Lease Liability, Credit Cash

### 4.5 Lease Modification Accounting
- **Separate lease:** Additional scope at standalone price → new ROU + liability
- **Prospective:** Adjust ROU and liability using revised discount rate
- **Cumulative catch-up:** Adjust ROU and liability, recognize gain/loss

### 4.6 Short-Term and Low-Value Exemptions
- Short-term: ≤ 12 months, no purchase option → expense on straight-line basis
- Low-value: Asset value below threshold (e.g., $5,000) → expense on straight-line basis

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `lease.lease.created` | Lease registered | GL |
| `lease.lease.classified` | Classification determined | — |
| `lease.payment.processed` | Payment processed | GL, CM |
| `lease.depreciation.calculated` | ROU depreciated | GL, FA |
| `lease.modification.approved` | Modification applied | GL |
| `lease.lease.terminated` | Lease ended | GL, FA |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.lease.v1;

service LeaseAccountingService {
    rpc CreateLease(CreateLeaseRequest) returns (CreateLeaseResponse);
    rpc GetLease(GetLeaseRequest) returns (GetLeaseResponse);
    rpc ClassifyLease(ClassifyLeaseRequest) returns (ClassifyLeaseResponse);
    rpc GetLiabilitySchedule(GetLiabilityScheduleRequest) returns (stream LiabilityScheduleEntry);
    rpc ProcessMonthEnd(ProcessMonthEndRequest) returns (ProcessMonthEndResponse);
    rpc GetLeaseDisclosures(GetLeaseDisclosuresRequest) returns (GetLeaseDisclosuresResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Calls To
- **GL:** Post interest, depreciation, and payment journal entries
- **AP:** Create recurring supplier invoices for lease payments
- **CM:** Record cash disbursements for lease payments
- **FA:** Track ROU assets (optional integration)

---

## 7. Migrations

1. V001: `lease_classification_rules`
2. V002: `leases`
3. V003: `lease_payments`
4. V004: `right_of_use_assets`
5. V005: `lease_liability_schedule`
6. V006: `lease_modifications`
7. V007: `lease_incentives`
8. V008: Triggers for `updated_at`
