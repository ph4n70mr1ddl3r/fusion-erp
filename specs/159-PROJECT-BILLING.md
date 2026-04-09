# 159 - Project Billing Service Specification

## 1. Domain Overview

Project Billing manages contract billing, invoicing, revenue recognition, and collections for project-based work. Supports time and materials (T&M), fixed price, cost-plus, and milestone-based billing methods. Enables automated invoice generation from approved project costs, progress billing, retention management, billing event scheduling, and contract amount management. Integrates with Project Costing for billable costs, Accounts Receivable for invoicing, Revenue Management for recognition, and Project Management for project status.

**Bounded Context:** Project Billing & Invoicing
**Service Name:** `project-billing-service`
**Database:** `data/project_billing.db`
**HTTP Port:** 8177 | **gRPC Port:** 9177

---

## 2. Database Schema

### 2.1 Billing Contracts
```sql
CREATE TABLE pb_billing_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,          -- CTR-2024-00001
    contract_name TEXT NOT NULL,
    project_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    billing_method TEXT NOT NULL CHECK(billing_method IN (
        'TIME_AND_MATERIALS','FIXED_PRICE','COST_PLUS','MILESTONE','PROGRESS','RETAINER','HYBRID'
    )),
    total_contract_value_cents INTEGER NOT NULL DEFAULT 0,
    billed_to_date_cents INTEGER NOT NULL DEFAULT 0,
    retained_to_date_cents INTEGER NOT NULL DEFAULT 0,
    revenue_recognized_cents INTEGER NOT NULL DEFAULT 0,
    remaining_value_cents INTEGER NOT NULL DEFAULT 0,
    retention_pct REAL NOT NULL DEFAULT 0,
    max_retention_cents INTEGER NOT NULL DEFAULT 0,
    markup_pct REAL NOT NULL DEFAULT 0,    -- For cost-plus
    billing_cycle TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(billing_cycle IN ('WEEKLY','BIWEEKLY','MONTHLY','MILESTONE','ON_DEMAND')),
    next_billing_date TEXT,
    contract_start_date TEXT NOT NULL,
    contract_end_date TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    payment_terms TEXT NOT NULL DEFAULT 'NET_30',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ON_HOLD','COMPLETED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_pb_contract_project ON pb_billing_contracts(project_id);
CREATE INDEX idx_pb_contract_customer ON pb_billing_contracts(tenant_id, customer_id, status);
CREATE INDEX idx_pb_contract_billing ON pb_billing_contracts(tenant_id, next_billing_date, status);
```

### 2.2 Bill Rates
```sql
CREATE TABLE pb_bill_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    resource_role TEXT NOT NULL,
    resource_id TEXT,                       -- NULL = role-level rate
    rate_type TEXT NOT NULL CHECK(rate_type IN ('HOURLY','DAILY','FIXED','TIERED')),
    bill_rate_cents INTEGER NOT NULL,
    cost_rate_cents INTEGER,               -- For margin calculation
    currency_code TEXT NOT NULL DEFAULT 'USD',
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    tier_config TEXT,                       -- JSON: tiered rates

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (contract_id) REFERENCES pb_billing_contracts(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, contract_id, resource_role, resource_id, effective_from)
);
```

### 2.3 Billing Events
```sql
CREATE TABLE pb_billing_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    event_number TEXT NOT NULL,             -- BIL-EVT-2024-00001
    event_type TEXT NOT NULL CHECK(event_type IN (
        'MILESTONE','PROGRESS','TIME_MATERIALS','COST_PLUS','RETENTION_RELEASE',
        'MANUAL','CREDIT_MEMO','WRITE_OFF'
    )),
    description TEXT NOT NULL,
    billing_date TEXT NOT NULL,
    event_amount_cents INTEGER NOT NULL DEFAULT 0,
    billable_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    completion_pct REAL,                    -- For progress billing
    milestone_id TEXT,                      -- For milestone billing
    cost_transactions TEXT,                 -- JSON: linked cost transaction IDs
    invoice_id TEXT,                        -- After invoice generation
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','INVOICED','CANCELLED','ON_HOLD')),
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, event_number)
);

CREATE INDEX idx_pb_event_contract ON pb_billing_events(contract_id, billing_date DESC);
CREATE INDEX idx_pb_event_status ON pb_billing_events(tenant_id, status);
```

### 2.4 Project Invoices
```sql
CREATE TABLE pb_project_invoices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    invoice_number TEXT NOT NULL,           -- PJINV-2024-00001
    contract_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    invoice_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    billing_period_start TEXT NOT NULL,
    billing_period_end TEXT NOT NULL,
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    retention_held_cents INTEGER NOT NULL DEFAULT 0,
    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    ar_invoice_id TEXT,                     -- Reference to AR invoice
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','SENT','PARTIALLY_PAID','PAID','CANCELLED','CREDITED')),
    billing_event_ids TEXT NOT NULL,        -- JSON: included events
    notes TEXT,
    payment_terms TEXT NOT NULL DEFAULT 'NET_30',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_pb_inv_contract ON pb_project_invoices(contract_id, invoice_date DESC);
CREATE INDEX idx_pb_inv_customer ON pb_project_invoices(tenant_id, customer_id, status);
CREATE INDEX idx_pb_inv_status ON pb_project_invoices(tenant_id, status);
```

### 2.5 Invoice Lines
```sql
CREATE TABLE pb_invoice_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    invoice_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    line_type TEXT NOT NULL CHECK(line_type IN ('LABOR','MATERIAL','EXPENSE','OVERHEAD','MILESTONE','RETENTION','OTHER')),
    description TEXT NOT NULL,
    task_id TEXT,
    quantity DECIMAL(18,4),
    uom TEXT,
    unit_price_cents INTEGER NOT NULL,
    line_amount_cents INTEGER NOT NULL,
    tax_code TEXT,
    cost_transaction_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (invoice_id) REFERENCES pb_project_invoices(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, invoice_id, line_number)
);
```

---

## 3. API Endpoints

### 3.1 Billing Contracts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-billing/v1/contracts` | Create billing contract |
| GET | `/project-billing/v1/contracts` | List contracts |
| GET | `/project-billing/v1/contracts/{id}` | Get contract details |
| PUT | `/project-billing/v1/contracts/{id}` | Update contract |
| POST | `/project-billing/v1/contracts/{id}/activate` | Activate contract |

### 3.2 Bill Rates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-billing/v1/contracts/{id}/rates` | Set bill rate |
| GET | `/project-billing/v1/contracts/{id}/rates` | List rates |
| PUT | `/project-billing/v1/rates/{id}` | Update rate |

### 3.3 Billing Events
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-billing/v1/events` | Create billing event |
| GET | `/project-billing/v1/events` | List events |
| GET | `/project-billing/v1/events/{id}` | Get event details |
| PUT | `/project-billing/v1/events/{id}` | Update event |
| POST | `/project-billing/v1/events/{id}/approve` | Approve event |
| POST | `/project-billing/v1/events/auto-generate` | Auto-generate from costs |

### 3.4 Invoices
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-billing/v1/invoices` | Create invoice |
| GET | `/project-billing/v1/invoices` | List invoices |
| GET | `/project-billing/v1/invoices/{id}` | Get invoice details |
| PUT | `/project-billing/v1/invoices/{id}` | Update invoice |
| POST | `/project-billing/v1/invoices/{id}/submit` | Submit for approval |
| POST | `/project-billing/v1/invoices/{id}/approve` | Approve and send to AR |
| POST | `/project-billing/v1/invoices/{id}/credit` | Create credit memo |

### 3.5 Contract Summary
| Method | Path | Description |
|--------|------|-------------|
| GET | `/project-billing/v1/contracts/{id}/summary` | Contract billing summary |
| GET | `/project-billing/v1/contracts/{id}/aging` | Billing aging |
| GET | `/project-billing/v1/contracts/{id}/revenue-summary` | Revenue recognized |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `pbilling.event.created` | `{ event_id, contract_id, amount }` | Billing event created |
| `pbilling.event.approved` | `{ event_id, amount }` | Billing event approved |
| `pbilling.invoice.created` | `{ invoice_id, contract_id, total }` | Invoice generated |
| `pbilling.invoice.approved` | `{ invoice_id, ar_invoice_id }` | Invoice approved |
| `pbilling.retention.released` | `{ contract_id, amount }` | Retention released |
| `pbilling.contract.threshold` | `{ contract_id, billed_pct }` | Contract threshold reached |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `pcost.transaction.created` | Project Costing | Create billable event for T&M |
| `project.milestone.completed` | Project Mgmt | Trigger milestone billing |
| `project.completed` | Project Mgmt | Trigger final billing and retention release |
| `payment.received` | AR | Update invoice payment status |

---

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.project_billing.v1;

service ProjectBillingService {
    // Contract management
    rpc CreateContract(CreateContractRequest) returns (BillingContract);
    rpc GetContract(GetContractRequest) returns (BillingContract);
    rpc ListContracts(ListContractsRequest) returns (ContractList);
    rpc ActivateContract(ActivateContractRequest) returns (BillingContract);

    // Billing events
    rpc CreateBillingEvent(CreateBillingEventRequest) returns (BillingEvent);
    rpc ApproveBillingEvent(ApproveBillingEventRequest) returns (BillingEvent);
    rpc AutoGenerateEvents(AutoGenerateEventsRequest) returns (AutoGenerateEventsResponse);

    // Invoice management
    rpc CreateInvoice(CreateInvoiceRequest) returns (ProjectInvoice);
    rpc GetInvoice(GetInvoiceRequest) returns (ProjectInvoice);
    rpc ApproveInvoice(ApproveInvoiceRequest) returns (ProjectInvoice);
}

// Entity messages
message BillingContract {
    string id = 1;
    string tenant_id = 2;
    string contract_number = 3;
    string contract_name = 4;
    string project_id = 5;
    string customer_id = 6;
    string billing_method = 7;
    int64 total_contract_value_cents = 8;
    int64 billed_to_date_cents = 9;
    int64 retained_to_date_cents = 10;
    int64 revenue_recognized_cents = 11;
    int64 remaining_value_cents = 12;
    double retention_pct = 13;
    int64 max_retention_cents = 14;
    double markup_pct = 15;
    string billing_cycle = 16;
    string next_billing_date = 17;
    string contract_start_date = 18;
    string contract_end_date = 19;
    string currency_code = 20;
    string payment_terms = 21;
    string status = 22;
    int32 version = 23;
}

message BillingEvent {
    string id = 1;
    string tenant_id = 2;
    string contract_id = 3;
    string project_id = 4;
    string event_number = 5;
    string event_type = 6;
    string description = 7;
    string billing_date = 8;
    int64 event_amount_cents = 9;
    int64 billable_amount_cents = 10;
    int64 tax_amount_cents = 11;
    int64 total_amount_cents = 12;
    double completion_pct = 13;
    string milestone_id = 14;
    string cost_transactions = 15;
    string invoice_id = 16;
    string status = 17;
    string approved_by = 18;
    string approved_at = 19;
}

message ProjectInvoice {
    string id = 1;
    string tenant_id = 2;
    string invoice_number = 3;
    string contract_id = 4;
    string project_id = 5;
    string customer_id = 6;
    string invoice_date = 7;
    string due_date = 8;
    string billing_period_start = 9;
    string billing_period_end = 10;
    int64 subtotal_cents = 11;
    int64 tax_cents = 12;
    int64 retention_held_cents = 13;
    int64 total_amount_cents = 14;
    string currency_code = 15;
    string ar_invoice_id = 16;
    string status = 17;
    string billing_event_ids = 18;
    string notes = 19;
    string payment_terms = 20;
}

// Request/Response messages
message CreateContractRequest {
    string tenant_id = 1;
    string contract_name = 2;
    string project_id = 3;
    string customer_id = 4;
    string billing_method = 5;
    int64 total_contract_value_cents = 6;
    double retention_pct = 7;
    double markup_pct = 8;
    string billing_cycle = 9;
    string contract_start_date = 10;
    string contract_end_date = 11;
    string currency_code = 12;
    string payment_terms = 13;
}

message GetContractRequest {
    string id = 1;
    string tenant_id = 2;
}

message ListContractsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string customer_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ContractList {
    repeated BillingContract contracts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ActivateContractRequest {
    string id = 1;
    string tenant_id = 2;
}

message CreateBillingEventRequest {
    string tenant_id = 1;
    string contract_id = 2;
    string event_type = 3;
    string description = 4;
    string billing_date = 5;
    int64 event_amount_cents = 6;
    double completion_pct = 7;
    string cost_transactions = 8;
}

message ApproveBillingEventRequest {
    string event_id = 1;
    string tenant_id = 2;
    string approved_by = 3;
}

message AutoGenerateEventsRequest {
    string tenant_id = 1;
    string contract_id = 2;
    string billing_date = 3;
}

message AutoGenerateEventsResponse {
    repeated BillingEvent events = 1;
    int32 events_created = 2;
}

message CreateInvoiceRequest {
    string tenant_id = 1;
    string contract_id = 2;
    repeated string billing_event_ids = 3;
    string invoice_date = 4;
}

message GetInvoiceRequest {
    string id = 1;
    string tenant_id = 2;
}

message ApproveInvoiceRequest {
    string invoice_id = 1;
    string tenant_id = 2;
    string approved_by = 3;
}
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | pb_billing_contracts | — |
| V002 | pb_bill_rates | V001 |
| V003 | pb_billing_events | V001 |
| V004 | pb_project_invoices | V001 |
| V005 | pb_invoice_lines | V004 |

---

## 7. Business Rules

1. **Billable Costs**: Only costs marked billable on eligible tasks generate billing events
2. **Rate Determination**: Bill rate lookup by resource > role > contract default
3. **Retention**: Retention percentage held from each invoice; released per contract terms
4. **Progress Billing**: Percentage-based billing validated against project completion
5. **Contract Ceiling**: Alert when billing approaches 80% of contract value; block at 100%
6. **Invoice Sequentiality**: Invoice numbers sequential and gapless per tenant
7. **Cost-Plus Markup**: Markup applied to eligible cost base per contract configuration
8. **Credit Memo**: Credit memos reference original invoice; reduce contract billed amount

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Project Costing (158) | Billable cost transactions |
| Project Management (15) | Project milestones, completion status |
| Accounts Receivable (08) | Invoice posting and payment |
| Revenue Management (26) | Revenue recognition rules |
| General Ledger (06) | Billing journal entries |
| Tax Management (23) | Tax calculation on invoices |
| BI Publisher (150) | Invoice document generation |
| Customer Service (81) | Customer billing inquiries |
