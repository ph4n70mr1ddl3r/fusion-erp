# 55 - Joint Venture Management Service Specification

## 1. Domain Overview

Joint Venture Management handles cost sharing and revenue distribution among multiple partners in joint ventures, common in oil & gas, mining, real estate, and large capital projects. Manages partner agreements, ownership percentages, cost allocation methodologies, billing to partners, cash calls, and joint venture accounting entries. Supports multiple allocation methods including working interest, carried interest, and overriding royalty, with automatic partner invoicing and reconciliation.

**Bounded Context:** Joint Venture Accounting, Partner Cost Sharing & Revenue Distribution
**Service Name:** `jv-service`
**Database:** `data/jv.db`
**HTTP Port:** 8087 | **gRPC Port:** 9087

---

## 2. Database Schema

### 2.1 Joint Ventures
```sql
CREATE TABLE joint_ventures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_code TEXT NOT NULL,
    jv_name TEXT NOT NULL,
    jv_type TEXT NOT NULL
        CHECK(jv_type IN ('OPERATED','NON_OPERATED','CONSORTIUM','PARTNERSHIP','LLC')),
    operator_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','SUSPENDED','WOUND_UP')),
    start_date TEXT NOT NULL,
    end_date TEXT,
    description TEXT,
    accounting_method TEXT NOT NULL DEFAULT 'EQUITY'
        CHECK(accounting_method IN ('EQUITY','PROPORTIONATE_CONSOLIDATION','COST')),
    currency TEXT NOT NULL DEFAULT 'USD',
    operator_gl_account TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, jv_code)
);

CREATE INDEX idx_joint_ventures_tenant ON joint_ventures(tenant_id, status);
CREATE INDEX idx_joint_ventures_operator ON joint_ventures(tenant_id, operator_id);
```

### 2.2 JV Partners
```sql
CREATE TABLE jv_partners (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    partner_role TEXT NOT NULL
        CHECK(partner_role IN ('OPERATOR','NON_OPERATOR','CARRIED','CARRYING')),
    ownership_percentage REAL NOT NULL,
    interest_type TEXT NOT NULL DEFAULT 'WORKING'
        CHECK(interest_type IN ('WORKING','CARRIED','OVERRIDING_ROYALTY','ROYALTY')),
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    billing_contact TEXT,
    billing_email TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, jv_id, partner_id, effective_from)
);

CREATE INDEX idx_jv_partners_jv ON jv_partners(jv_id, effective_from);
```

### 2.3 JV Billing Cycles
```sql
CREATE TABLE jv_billing_cycles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    cycle_name TEXT NOT NULL,
    billing_frequency TEXT NOT NULL
        CHECK(billing_frequency IN ('MONTHLY','QUARTERLY','SEMI_ANNUALLY','ANNUALLY','ON_DEMAND')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','PROCESSING','BILLED','CLOSED')),
    total_costs INTEGER NOT NULL DEFAULT 0,
    total_revenues INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id),
    UNIQUE(tenant_id, jv_id, cycle_name)
);

CREATE INDEX idx_jv_billing_cycles_jv ON jv_billing_cycles(jv_id, status);
```

### 2.4 JV Cost Pools
```sql
CREATE TABLE jv_cost_pools (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    pool_code TEXT NOT NULL,
    pool_name TEXT NOT NULL,
    pool_type TEXT NOT NULL
        CHECK(pool_type IN ('CAPEX','OPEX','ABANDONMENT','EXPLORATION','DEVELOPMENT','PRODUCTION','G&A')),
    allocation_method TEXT NOT NULL DEFAULT 'WORKING_INTEREST'
        CHECK(allocation_method IN ('WORKING_INTEREST','EQUAL','CUSTOM_PERCENTAGE','UNIT_BASED')),
    gl_account_id TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, jv_id, pool_code)
);

CREATE INDEX idx_jv_cost_pools_jv ON jv_cost_pools(jv_id, pool_type);
```

### 2.5 JV Distributions
```sql
CREATE TABLE jv_distributions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    billing_cycle_id TEXT NOT NULL,
    cost_pool_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    distribution_type TEXT NOT NULL
        CHECK(distribution_type IN ('COST_SHARE','REVENUE_SHARE','CAPITAL_CALL','ADJUSTMENT')),
    total_amount INTEGER NOT NULL DEFAULT 0,
    partner_share_pct REAL NOT NULL,
    partner_share_amount INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id),
    FOREIGN KEY (billing_cycle_id) REFERENCES jv_billing_cycles(id),
    FOREIGN KEY (cost_pool_id) REFERENCES jv_cost_pools(id)
);

CREATE INDEX idx_jv_dist_cycle ON jv_distributions(billing_cycle_id, partner_id);
CREATE INDEX idx_jv_dist_jv ON jv_distributions(jv_id, distribution_type);
```

### 2.6 JV Invoices
```sql
CREATE TABLE jv_invoices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    billing_cycle_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    invoice_number TEXT NOT NULL,
    invoice_type TEXT NOT NULL
        CHECK(invoice_type IN ('COST_SHARE','CASH_CALL','REVENUE_DISTRIBUTION','ADJUSTMENT')),
    invoice_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    amount INTEGER NOT NULL DEFAULT 0,
    currency TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ISSUED','PARTIALLY_PAID','PAID','DISPUTED','WRITTEN_OFF')),
    payment_reference TEXT,
    paid_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id),
    FOREIGN KEY (billing_cycle_id) REFERENCES jv_billing_cycles(id),
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_jv_invoices_jv ON jv_invoices(jv_id, status, due_date);
CREATE INDEX idx_jv_invoices_partner ON jv_invoices(partner_id, status);
```

### 2.7 JV Billing Rules
```sql
CREATE TABLE jv_billing_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    cost_pool_id TEXT,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('COST_ALLOCATION','REVENUE_ALLOCATION','OVERHEAD_MARKUP','MANAGEMENT_FEE')),
    calculation_method TEXT NOT NULL
        CHECK(calculation_method IN ('PERCENTAGE','FIXED_AMOUNT','PER_UNIT','TIERED')),
    rate_or_amount REAL NOT NULL,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, jv_id, rule_name)
);

CREATE INDEX idx_jv_billing_rules_jv ON jv_billing_rules(jv_id, rule_type);
```

### 2.8 JV Ownership Changes
```sql
CREATE TABLE jv_ownership_changes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    change_date TEXT NOT NULL,
    old_percentage REAL NOT NULL,
    new_percentage REAL NOT NULL,
    transfer_from_partner_id TEXT,
    transfer_to_partner_id TEXT,
    reason TEXT,
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','APPROVED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id)
);

CREATE INDEX idx_ownership_changes_jv ON jv_ownership_changes(jv_id, change_date);
```

### 2.9 JV Cash Calls
```sql
CREATE TABLE jv_cash_calls (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    cash_call_number TEXT NOT NULL,
    call_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    total_amount INTEGER NOT NULL DEFAULT 0,
    purpose TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ISSUED'
        CHECK(status IN ('ISSUED','PARTIALLY_RECEIVED','FULLY_RECEIVED','OVERDUE','CANCELLED')),
    received_amount INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES joint_ventures(id),
    UNIQUE(tenant_id, cash_call_number)
);

CREATE INDEX idx_cash_calls_jv ON jv_cash_calls(jv_id, status, due_date);
```

---

## 3. REST API Endpoints

### 3.1 Joint Ventures
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/jv/ventures` | List joint ventures |
| POST | `/api/v1/jv/ventures` | Create joint venture |
| GET | `/api/v1/jv/ventures/{id}` | Get JV detail |
| PUT | `/api/v1/jv/ventures/{id}` | Update JV |
| POST | `/api/v1/jv/ventures/{id}/activate` | Activate JV |
| POST | `/api/v1/jv/ventures/{id}/suspend` | Suspend JV |

### 3.2 Partners
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/jv/ventures/{id}/partners` | List JV partners |
| POST | `/api/v1/jv/ventures/{id}/partners` | Add partner |
| PUT | `/api/v1/jv/ventures/{id}/partners/{partnerId}` | Update partner |
| DELETE | `/api/v1/jv/ventures/{id}/partners/{partnerId}` | Remove partner |

### 3.3 Cost Pools
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/jv/ventures/{id}/cost-pools` | List cost pools |
| POST | `/api/v1/jv/ventures/{id}/cost-pools` | Create cost pool |
| PUT | `/api/v1/jv/ventures/{id}/cost-pools/{poolId}` | Update cost pool |

### 3.4 Billing Cycles
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/jv/ventures/{id}/billing-cycles` | List billing cycles |
| POST | `/api/v1/jv/ventures/{id}/billing-cycles` | Create billing cycle |
| POST | `/api/v1/jv/ventures/{id}/billing-cycles/{cycleId}/process` | Process billing |
| POST | `/api/v1/jv/ventures/{id}/billing-cycles/{cycleId}/close` | Close cycle |

### 3.5 Distributions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/jv/distributions/calculate` | Calculate distributions |
| GET | `/api/v1/jv/ventures/{id}/distributions` | List distributions |
| POST | `/api/v1/jv/distributions/{id}/approve` | Approve distribution |

### 3.6 Invoices
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/jv/invoices/generate` | Generate partner invoices |
| GET | `/api/v1/jv/ventures/{id}/invoices` | List JV invoices |
| GET | `/api/v1/jv/invoices/{id}` | Get invoice detail |
| POST | `/api/v1/jv/invoices/{id}/issue` | Issue invoice |
| POST | `/api/v1/jv/invoices/{id}/record-payment` | Record payment |

### 3.7 Cash Calls
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/jv/ventures/{id}/cash-calls` | Issue cash call |
| GET | `/api/v1/jv/ventures/{id}/cash-calls` | List cash calls |
| POST | `/api/v1/jv/cash-calls/{id}/receive` | Record cash call receipt |

### 3.8 Ownership Changes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/jv/ventures/{id}/ownership-changes` | Record ownership change |
| GET | `/api/v1/jv/ventures/{id}/ownership-history` | Get ownership history |

### 3.9 Billing Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/jv/ventures/{id}/billing-rules` | List billing rules |
| POST | `/api/v1/jv/ventures/{id}/billing-rules` | Create billing rule |
| PUT | `/api/v1/jv/ventures/{id}/billing-rules/{ruleId}` | Update rule |

---

## 4. Business Rules

1. Partner ownership percentages for a JV MUST total 100% at any point in time.
2. Cost distributions MUST be calculated based on the applicable ownership percentages for the billing period.
3. The operator partner MUST be responsible for billing other partners.
4. Carried interest partners MUST NOT be billed for costs during the carry period.
5. Billing cycles MUST be processed sequentially — a new cycle cannot start until the prior one is closed.
6. Cash calls MUST specify a due date and track partial receipts.
7. Ownership changes MUST be recorded with effective dates and approvals.
8. Revenue distributions MUST be allocated based on working interest after royalty deductions.
9. JV invoices to partners MUST create corresponding AP/AR entries.
10. Management fees and overhead markups MUST be calculated per billing rules before partner allocation.
11. Overdue cash calls MUST be flagged and may trigger penalty interest.
12. Winding up a JV MUST require all billing cycles to be closed and all partner balances reconciled.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.jv.v1;

service JointVentureService {
    rpc CreateJointVenture(CreateJVRequest) returns (JVResponse);
    rpc GetJointVenture(GetJVRequest) returns (JVDetailResponse);
    rpc ListJointVentures(ListJVsRequest) returns (ListJVsResponse);

    rpc AddPartner(AddPartnerRequest) returns (PartnerResponse);
    rpc UpdatePartner(UpdatePartnerRequest) returns (PartnerResponse);

    rpc CreateBillingCycle(CreateBillingCycleRequest) returns (BillingCycleResponse);
    rpc ProcessBillingCycle(ProcessBillingCycleRequest) returns (BillingCycleResponse);

    rpc CalculateDistributions(CalculateDistributionsRequest) returns (DistributionsResponse);

    rpc GenerateInvoices(GenerateInvoicesRequest) returns (InvoiceListResponse);
    rpc IssueInvoice(IssueInvoiceRequest) returns (InvoiceResponse);
    rpc RecordPayment(RecordPaymentRequest) returns (InvoiceResponse);

    rpc IssueCashCall(IssueCashCallRequest) returns (CashCallResponse);
    rpc RecordCashCallReceipt(RecordCashCallReceiptRequest) returns (CashCallResponse);

    rpc RecordOwnershipChange(OwnershipChangeRequest) returns (OwnershipChangeResponse);
}

// Entity messages

message JointVenture {
    string id = 1;
    string tenant_id = 2;
    string jv_code = 3;
    string jv_name = 4;
    string jv_type = 5;
    string operator_id = 6;
    string status = 7;
    string start_date = 8;
    string end_date = 9;
    string description = 10;
    string accounting_method = 11;
    string currency = 12;
    string operator_gl_account = 13;
    string created_at = 14;
    string updated_at = 15;
}

message JVPartner {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string partner_id = 4;
    string partner_role = 5;
    double ownership_percentage = 6;
    string interest_type = 7;
    string effective_from = 8;
    string effective_to = 9;
    string billing_contact = 10;
    string billing_email = 11;
    string created_at = 12;
    string updated_at = 13;
}

message JVBillingCycle {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string cycle_name = 4;
    string billing_frequency = 5;
    string period_start = 6;
    string period_end = 7;
    string status = 8;
    int64 total_costs = 9;
    int64 total_revenues = 10;
    string created_at = 11;
    string updated_at = 12;
}

message JVCostPool {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string pool_code = 4;
    string pool_name = 5;
    string pool_type = 6;
    string allocation_method = 7;
    string gl_account_id = 8;
    string description = 9;
    string created_at = 10;
    string updated_at = 11;
}

message JVDistribution {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string billing_cycle_id = 4;
    string cost_pool_id = 5;
    string partner_id = 6;
    string distribution_type = 7;
    int64 total_amount = 8;
    double partner_share_pct = 9;
    int64 partner_share_amount = 10;
    string description = 11;
    string created_at = 12;
    string updated_at = 13;
}

message JVInvoice {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string billing_cycle_id = 4;
    string partner_id = 5;
    string invoice_number = 6;
    string invoice_type = 7;
    string invoice_date = 8;
    string due_date = 9;
    int64 amount = 10;
    string currency = 11;
    string status = 12;
    string payment_reference = 13;
    string paid_date = 14;
    string created_at = 15;
    string updated_at = 16;
}

message JVBillingRule {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string rule_name = 4;
    string cost_pool_id = 5;
    string rule_type = 6;
    string calculation_method = 7;
    double rate_or_amount = 8;
    string effective_from = 9;
    string effective_to = 10;
    string created_at = 11;
    string updated_at = 12;
}

message JVOwnershipChange {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string partner_id = 4;
    string change_date = 5;
    double old_percentage = 6;
    double new_percentage = 7;
    string transfer_from_partner_id = 8;
    string transfer_to_partner_id = 9;
    string reason = 10;
    string approval_status = 11;
    string created_at = 12;
    string updated_at = 13;
}

message JVCashCall {
    string id = 1;
    string tenant_id = 2;
    string jv_id = 3;
    string cash_call_number = 4;
    string call_date = 5;
    string due_date = 6;
    int64 total_amount = 7;
    string purpose = 8;
    string status = 9;
    int64 received_amount = 10;
    string created_at = 11;
    string updated_at = 12;
}

// Request/Response messages

message CreateJVRequest {
    string tenant_id = 1;
    string jv_code = 2;
    string jv_name = 3;
    string jv_type = 4;
    string operator_id = 5;
    string start_date = 6;
    string end_date = 7;
    string description = 8;
    string accounting_method = 9;
    string currency = 10;
    string operator_gl_account = 11;
}

message JVResponse {
    JointVenture data = 1;
}

message GetJVRequest {
    string tenant_id = 1;
    string id = 2;
}

message JVDetailResponse {
    JointVenture data = 1;
    repeated JVPartner partners = 2;
    repeated JVCostPool cost_pools = 3;
}

message ListJVsRequest {
    string tenant_id = 1;
    string status = 2;
    string jv_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListJVsResponse {
    repeated JointVenture items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message AddPartnerRequest {
    string tenant_id = 1;
    string jv_id = 2;
    string partner_id = 3;
    string partner_role = 4;
    double ownership_percentage = 5;
    string interest_type = 6;
    string effective_from = 7;
    string effective_to = 8;
    string billing_contact = 9;
    string billing_email = 10;
}

message PartnerResponse {
    JVPartner data = 1;
}

message UpdatePartnerRequest {
    string tenant_id = 1;
    string id = 2;
    double ownership_percentage = 3;
    string effective_to = 4;
    string billing_contact = 5;
    string billing_email = 6;
}

message CreateBillingCycleRequest {
    string tenant_id = 1;
    string jv_id = 2;
    string cycle_name = 3;
    string billing_frequency = 4;
    string period_start = 5;
    string period_end = 6;
}

message BillingCycleResponse {
    JVBillingCycle data = 1;
}

message ProcessBillingCycleRequest {
    string tenant_id = 1;
    string cycle_id = 2;
}

message CalculateDistributionsRequest {
    string tenant_id = 1;
    string jv_id = 2;
    string billing_cycle_id = 3;
    string cost_pool_id = 4;
}

message DistributionsResponse {
    repeated JVDistribution distributions = 1;
}

message GenerateInvoicesRequest {
    string tenant_id = 1;
    string jv_id = 2;
    string billing_cycle_id = 3;
}

message InvoiceListResponse {
    repeated JVInvoice invoices = 1;
}

message IssueInvoiceRequest {
    string tenant_id = 1;
    string invoice_id = 2;
}

message InvoiceResponse {
    JVInvoice data = 1;
}

message RecordPaymentRequest {
    string tenant_id = 1;
    string invoice_id = 2;
    int64 amount = 3;
    string payment_reference = 4;
    string paid_date = 5;
}

message IssueCashCallRequest {
    string tenant_id = 1;
    string jv_id = 2;
    string call_date = 3;
    string due_date = 4;
    int64 total_amount = 5;
    string purpose = 6;
}

message CashCallResponse {
    JVCashCall data = 1;
}

message RecordCashCallReceiptRequest {
    string tenant_id = 1;
    string cash_call_id = 2;
    int64 received_amount = 3;
}

message OwnershipChangeRequest {
    string tenant_id = 1;
    string jv_id = 2;
    string partner_id = 3;
    string change_date = 4;
    double new_percentage = 5;
    string transfer_from_partner_id = 6;
    string transfer_to_partner_id = 7;
    string reason = 8;
}

message OwnershipChangeResponse {
    JVOwnershipChange data = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `gl-service` | GL accounts, journal validation |
| `ap-service` | Supplier/partner payment processing |
| `ar-service` | Partner invoicing, receipt processing |
| `pm-service` | Project costs allocated to JV |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | JV accounting entries, elimination entries |
| `ap-service` | Partner payment requests |
| `ar-service` | Partner invoices, receipts |
| `report-service` | JV financial reports, partner statements |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `jv.venture.created` | `{jv_id, code, type, operator}` | JV created |
| `jv.venture.activated` | `{jv_id}` | JV activated |
| `jv.partner.added` | `{jv_id, partner_id, ownership_pct}` | Partner added |
| `jv.ownership.changed` | `{jv_id, partner_id, old_pct, new_pct}` | Ownership changed |
| `jv.distribution.calculated` | `{jv_id, cycle_id, total_amount}` | Distribution calculated |
| `jv.invoice.generated` | `{jv_id, invoice_id, partner_id, amount}` | Invoice generated |
| `jv.invoice.issued` | `{invoice_id, amount, due_date}` | Invoice issued |
| `jv.invoice.paid` | `{invoice_id, amount, paid_date}` | Invoice paid |
| `jv.cash-call.issued` | `{cash_call_id, jv_id, total_amount, due_date}` | Cash call issued |
| `jv.cash-call.received` | `{cash_call_id, received_amount}` | Cash call receipt |
| `jv.billing-cycle.closed` | `{cycle_id, jv_id}` | Billing cycle closed |
