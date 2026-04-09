# 26 - Revenue Management & Recognition Specification

## 1. Domain Overview

Revenue Management handles ASC 606 / IFRS 15 compliant revenue recognition, contract management, performance obligation tracking, standalone selling price estimation, deferred revenue amortization, and contract cost capitalization. Integrates with AR for invoicing, OM for sales orders, PM for project billing, and GL for revenue journal entries.

**Bounded Context:** Revenue Recognition & Contract Management
**Service Name:** `rev-service`
**Database:** `data/rev.db`
**HTTP Port:** 8025 | **gRPC Port:** 9025

---

## 2. Database Schema

### 2.1 Revenue Policies
```sql
CREATE TABLE revenue_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_name TEXT NOT NULL,
    description TEXT,
    product_type TEXT NOT NULL
        CHECK(product_type IN ('GOODS','SERVICES','SOFTWARE','SUBSCRIPTION','BUNDLE','PROJECT','LEASE')),
    recognition_method TEXT NOT NULL
        CHECK(recognition_method IN ('POINT_IN_TIME','OVER_TIME','RATABLE','MILESTONE','PERCENT_COMPLETE')),
    recognition_timing TEXT NOT NULL DEFAULT 'DELIVERY'
        CHECK(recognition_timing IN ('DELIVERY','ACCEPTANCE','INSTALLATION','SUBSCRIPTION_PERIOD','PROGRESS')),
    ssp_method TEXT NOT NULL DEFAULT 'ADJUSTED_MARKET'
        CHECK(ssp_method IN ('ADJUSTED_MARKET','EXPECTED_COST_PLUS_MARGIN','RESIDUAL','BLENDED')),
    deferral_required INTEGER NOT NULL DEFAULT 0,
    allocation_method TEXT NOT NULL DEFAULT 'PROPORTIONAL'
        CHECK(allocation_method IN ('PROPORTIONAL','RESIDUAL','EQUAL')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_name)
);
```

### 2.2 Revenue Contracts
```sql
CREATE TABLE revenue_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,           -- RC-2024-00001
    customer_id TEXT NOT NULL,
    contract_date TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','COMPLETED','CANCELLED','MODIFIED')),
    total_transaction_price_cents INTEGER NOT NULL DEFAULT 0,
    recognized_revenue_cents INTEGER NOT NULL DEFAULT 0,
    deferred_revenue_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Source reference
    source_type TEXT DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','SALES_ORDER','PROJECT','SUBSCRIPTION')),
    source_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_rev_contracts_tenant_customer ON revenue_contracts(tenant_id, customer_id);
CREATE INDEX idx_rev_contracts_tenant_status ON revenue_contracts(tenant_id, status);
CREATE INDEX idx_rev_contracts_tenant_dates ON revenue_contracts(tenant_id, start_date, end_date);
```

### 2.3 Performance Obligations
```sql
CREATE TABLE performance_obligations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    pob_number TEXT NOT NULL,               -- POB-1, POB-2 within contract
    description TEXT NOT NULL,
    product_type TEXT NOT NULL,
    recognition_method TEXT NOT NULL,
    recognition_timing TEXT NOT NULL,

    -- Standalone Selling Price
    ssp_amount_cents INTEGER NOT NULL DEFAULT 0,
    ssp_method TEXT NOT NULL,
    ssp_estimation_basis TEXT,

    -- Allocated Transaction Price
    allocated_amount_cents INTEGER NOT NULL DEFAULT 0,

    -- Quantities
    quantity INTEGER DEFAULT 1000,
    unit_of_measure TEXT DEFAULT 'EA',

    -- Dates
    satisfaction_start_date TEXT,
    satisfaction_end_date TEXT,
    satisfaction_percent_complete REAL DEFAULT 0,

    -- Revenue tracking
    recognized_revenue_cents INTEGER NOT NULL DEFAULT 0,
    deferred_revenue_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','CANCELLED')),

    -- Links
    ar_invoice_line_id TEXT,
    sales_order_line_id TEXT,
    project_task_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES revenue_contracts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, contract_id, pob_number)
);

CREATE INDEX idx_pobs_tenant_contract ON performance_obligations(tenant_id, contract_id);
CREATE INDEX idx_pobs_tenant_status ON performance_obligations(tenant_id, status);
```

### 2.4 Standalone Selling Prices
```sql
CREATE TABLE standalone_selling_prices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_type TEXT NOT NULL,
    product_id TEXT,
    price_cents INTEGER NOT NULL,
    estimation_method TEXT NOT NULL,
    observable INTEGER NOT NULL DEFAULT 1,  -- 1=directly observable, 0=estimated
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    confidence_level TEXT DEFAULT 'HIGH'
        CHECK(confidence_level IN ('HIGH','MEDIUM','LOW')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, product_type, product_id, effective_from)
);
```

### 2.5 Revenue Schedules
```sql
CREATE TABLE revenue_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pob_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    scheduled_amount_cents INTEGER NOT NULL DEFAULT 0,
    recognized_amount_cents INTEGER NOT NULL DEFAULT 0,
    recognition_date TEXT,
    status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(status IN ('SCHEDULED','RECOGNIZED','REVERSED','DEFERRED')),
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (pob_id) REFERENCES performance_obligations(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, pob_id, period_name)
);

CREATE INDEX idx_rev_schedules_tenant_pob ON revenue_schedules(tenant_id, pob_id);
CREATE INDEX idx_rev_schedules_tenant_period ON revenue_schedules(tenant_id, period_name);
CREATE INDEX idx_rev_schedules_tenant_status ON revenue_schedules(tenant_id, status);
```

### 2.6 Revenue Recognition Entries
```sql
CREATE TABLE revenue_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    pob_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    recognition_date TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    recognition_type TEXT NOT NULL
        CHECK(recognition_type IN ('INITIAL','ADJUSTMENT','REVERSAL','CUMULATIVE_CATCHUP','MODIFICATION')),
    description TEXT,
    gl_journal_id TEXT,
    reference_type TEXT,
    reference_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (contract_id) REFERENCES revenue_contracts(id) ON DELETE RESTRICT,
    FOREIGN KEY (pob_id) REFERENCES performance_obligations(id) ON DELETE RESTRICT
);

CREATE INDEX idx_rev_entries_tenant_contract ON revenue_entries(tenant_id, contract_id);
CREATE INDEX idx_rev_entries_tenant_period ON revenue_entries(tenant_id, period_name);
CREATE INDEX idx_rev_entries_tenant_date ON revenue_entries(tenant_id, recognition_date);
```

### 2.7 Deferred Revenue
```sql
CREATE TABLE deferred_revenue (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    pob_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    beginning_balance_cents INTEGER NOT NULL DEFAULT 0,
    additions_cents INTEGER NOT NULL DEFAULT 0,
    recognized_cents INTEGER NOT NULL DEFAULT 0,
    adjustments_cents INTEGER NOT NULL DEFAULT 0,
    ending_balance_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (contract_id) REFERENCES revenue_contracts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, pob_id, period_name)
);
```

### 2.8 Contract Modifications
```sql
CREATE TABLE contract_modifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    modification_number TEXT NOT NULL,
    modification_date TEXT NOT NULL,
    modification_type TEXT NOT NULL
        CHECK(modification_type IN ('SCOPE_CHANGE','PRICE_CHANGE','TERM_EXTENSION','TERM_REDUCTION','CANCELLATION_PARTIAL')),
    accounting_treatment TEXT NOT NULL
        CHECK(accounting_treatment IN ('PROSPECTIVE','CUMULATIVE_CATCHUP','SEPARATE_CONTRACT')),
    description TEXT,
    price_change_cents INTEGER NOT NULL DEFAULT 0,
    old_total_cents INTEGER NOT NULL,
    new_total_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','POSTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES revenue_contracts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, contract_id, modification_number)
);
```

### 2.9 Contract Costs (ASC 340-40)
```sql
CREATE TABLE contract_costs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    cost_type TEXT NOT NULL
        CHECK(cost_type IN ('INCREMENTAL_OBTAINING','FULFILLMENT','OTHER')),
    description TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    amortized_cents INTEGER NOT NULL DEFAULT 0,
    remaining_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    amortization_method TEXT NOT NULL DEFAULT 'STRAIGHT_LINE'
        CHECK(amortization_method IN ('STRAIGHT_LINE','PROPORTIONAL','MILESTONE')),
    amortization_start_date TEXT NOT NULL,
    amortization_end_date TEXT,
    expense_account_id TEXT NOT NULL,
    asset_account_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'CAPITALIZED'
        CHECK(status IN ('CAPITALIZED','AMORTIZING','FULLY_AMORTIZED','IMPAIRED','WRITTEN_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES revenue_contracts(id) ON DELETE RESTRICT
);
```

---

## 3. REST API Endpoints

```
# Revenue Policies
GET/POST      /api/v1/rev/policies                     Permission: rev.policies.read/create
GET/PUT       /api/v1/rev/policies/{id}                 Permission: rev.policies.read/update

# Revenue Contracts
GET/POST      /api/v1/rev/contracts                     Permission: rev.contracts.read/create
GET/PUT       /api/v1/rev/contracts/{id}                 Permission: rev.contracts.read/update
POST          /api/v1/rev/contracts/{id}/activate       Permission: rev.contracts.update
POST          /api/v1/rev/contracts/{id}/cancel         Permission: rev.contracts.update
GET           /api/v1/rev/contracts/{id}/summary        Permission: rev.contracts.read

# Performance Obligations
GET/POST      /api/v1/rev/contracts/{id}/obligations    Permission: rev.obligations.read/create
GET/PUT       /api/v1/rev/obligations/{id}               Permission: rev.obligations.read/update
POST          /api/v1/rev/obligations/{id}/complete      Permission: rev.obligations.update

# Standalone Selling Prices
GET/POST      /api/v1/rev/ssp                           Permission: rev.ssp.read/create
GET/PUT       /api/v1/rev/ssp/{id}                       Permission: rev.ssp.read/update
POST          /api/v1/rev/ssp/estimate                   Permission: rev.ssp.create

# Revenue Recognition
POST          /api/v1/rev/recognize/run                  Permission: rev.recognition.run
GET           /api/v1/rev/recognize/run/{id}             Permission: rev.recognition.read
GET           /api/v1/rev/recognize/entries              Permission: rev.recognition.read
POST          /api/v1/rev/recognize/reverse/{entry_id}   Permission: rev.recognition.reverse

# Deferred Revenue
GET           /api/v1/rev/deferred                       Permission: rev.deferred.read
GET           /api/v1/rev/deferred/roll-forward          Permission: rev.deferred.read

# Contract Modifications
GET/POST      /api/v1/rev/contracts/{id}/modifications   Permission: rev.modifications.read/create
POST          /api/v1/rev/modifications/{id}/approve     Permission: rev.modifications.approve

# Contract Costs
GET/POST      /api/v1/rev/contracts/{id}/costs           Permission: rev.costs.read/create
POST          /api/v1/rev/costs/{id}/amortize            Permission: rev.costs.update
POST          /api/v1/rev/costs/{id}/impair              Permission: rev.costs.update

# Reports
GET           /api/v1/rev/reports/waterfall              Permission: rev.reports.view
GET           /api/v1/rev/reports/contract-summary       Permission: rev.reports.view
GET           /api/v1/rev/reports/deferred-revenue       Permission: rev.reports.view
GET           /api/v1/rev/reports/recognition-by-period  Permission: rev.reports.view
```

---

## 4. Business Rules

### 4.1 ASC 606 Five-Step Revenue Recognition

**Step 1 — Identify the Contract:**
- Contract exists when: parties approved, rights identifiable, payment terms identified, commercial substance, collectibility probable
- Contract combinations: multiple contracts with same customer may be combined
- Contract modifications create new or adjusted contracts

**Step 2 — Identify Performance Obligations:**
- Each distinct good/service = separate POB
- Distinct if: customer can benefit from it on its own OR it's readily available separately
- Series of distinct services satisfied over time = single POB

**Step 3 — Determine Transaction Price:**
- Fixed amounts, variable consideration, significant financing component, non-cash consideration
- Variable consideration: estimate using expected value or most likely amount
- Apply constraint: recognize only when not subject to significant reversal

**Step 4 — Allocate Transaction Price:**
- Allocate based on relative standalone selling prices
- SSP methods: adjusted market assessment, expected cost + margin, residual
- Discounts allocated to specific POBs if observable, otherwise pro-rata

**Step 5 — Recognize Revenue:**
- **Point-in-time:** At transfer of control (delivery, acceptance, installation)
- **Over time:** As customer receives/consumes benefits (measured by output or input method)
- Ratable recognition for subscriptions (straight-line over subscription period)
- Milestone-based for project deliverables

### 4.2 Deferred Revenue Management
- Initial deferred balance = allocated transaction price (for over-time recognition)
- Each period: amortize portion based on recognition schedule
- Revenue waterfall: show future recognition schedule
- Roll-forward: beginning + additions - recognized + adjustments = ending

### 4.3 Contract Modification Accounting
- **Separate contract:** Additional goods/services at standalone price
- **Prospective:** Additional goods/services not at standalone price, no change to existing POBs
- **Cumulative catch-up:** Change to existing POBs, adjust cumulative revenue

### 4.4 Contract Cost Capitalization
- **Incremental costs of obtaining contract:** Capitalize (e.g., sales commissions), amortize over contract term
- **Fulfillment costs:** Capitalize if they generate/enhance resources used to satisfy POBs
- **Amortization:** Straight-line over contract term or proportional to revenue recognition
- **Impairment:** Write down when carrying amount > remaining consideration - costs to provide

### 4.5 GL Integration
- **Revenue recognition:** Debit Deferred Revenue, Credit Revenue
- **Contract cost amortization:** Debit Expense, Credit Capitalized Contract Cost Asset
- **Contract cost impairment:** Debit Impairment Expense, Credit Contract Cost Asset

### 4.6 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `rev.contract.created` | Contract created | GL |
| `rev.obligation.completed` | POB satisfied | GL, AR |
| `rev.revenue.recognized` | Revenue recognized | GL, Report |
| `rev.contract.modified` | Contract modified | GL |
| `rev.deferred.adjusted` | Deferred revenue changed | Report |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.rev.v1;

service RevenueManagementService {
    rpc CreateContract(CreateContractRequest) returns (CreateContractResponse);
    rpc GetContract(GetContractRequest) returns (GetContractResponse);
    rpc AddPerformanceObligation(AddPerformanceObligationRequest) returns (AddPerformanceObligationResponse);
    rpc AllocateTransactionPrice(AllocateTransactionPriceRequest) returns (AllocateTransactionPriceResponse);
    rpc RunRecognition(RunRecognitionRequest) returns (RunRecognitionResponse);
    rpc GetDeferredRevenue(GetDeferredRevenueRequest) returns (GetDeferredRevenueResponse);
    rpc GetRevenueWaterfall(GetRevenueWaterfallRequest) returns (GetRevenueWaterfallResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Source Contracts From
- **OM:** Sales order confirmed → create revenue contract
- **AR:** Invoice posted → create/update contract
- **PM:** Project milestone → create/update contract

### 6.2 Call To
- **GL:** Post revenue recognition journal entries
- **AR:** Generate invoices from contract billing schedules

---

## 7. Migrations

### Migration Order for rev-service:
1. V001: `revenue_policies`
2. V002: `revenue_contracts`
3. V003: `performance_obligations`
4. V004: `standalone_selling_prices`
5. V005: `revenue_schedules`
6. V006: `revenue_entries`
7. V007: `deferred_revenue`
8. V008: `contract_modifications`
9. V009: `contract_costs`
10. V010: Triggers for `updated_at`
