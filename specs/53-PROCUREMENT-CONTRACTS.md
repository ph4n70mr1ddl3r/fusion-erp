# 53 - Procurement Contracts Service Specification

## 1. Domain Overview

Procurement Contracts provides full contract lifecycle management within procurement — authoring, negotiation, execution, compliance tracking, amendments, renewals, and closeout. Integrates with purchasing for contract-based buying and includes clause libraries, templates, and regulatory compliance tracking. Supports multiple contract types including purchase agreements, service contracts, framework agreements, and lease contracts with spending tracking against contract limits.

**Bounded Context:** Procurement Contract Lifecycle Management & Compliance
**Service Name:** `contracts-service`
**Database:** `data/contracts.db`
**HTTP Port:** 8085 | **gRPC Port:** 9085

---

## 2. Database Schema

### 2.1 Contract Types
```sql
CREATE TABLE contract_types (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    type_code TEXT NOT NULL,
    type_name TEXT NOT NULL,
    type_category TEXT NOT NULL
        CHECK(type_category IN ('PURCHASE','SERVICE','FRAMEWORK','LEASE','BLANKET','CONSULTING')),
    description TEXT,
    default_duration_days INTEGER,
    requires_approval INTEGER NOT NULL DEFAULT 1,
    approval_workflow_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, type_code)
);

CREATE INDEX idx_contract_types_tenant ON contract_types(tenant_id, type_category);
```

### 2.2 Contracts
```sql
CREATE TABLE contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,
    contract_type_id TEXT NOT NULL,
    contract_name TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PENDING_REVIEW','IN_REVIEW','PENDING_APPROVAL','APPROVED','ACTIVE','SUSPENDED','EXPIRED','TERMINATED','CLOSED')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    total_value INTEGER NOT NULL DEFAULT 0,
    currency TEXT NOT NULL DEFAULT 'USD',
    payment_terms TEXT,
    delivery_terms TEXT,
    contract_owner_id TEXT NOT NULL,
    department_id TEXT,
    renewal_type TEXT NOT NULL DEFAULT 'NONE'
        CHECK(renewal_type IN ('NONE','AUTO','MANUAL')),
    auto_renewal_days INTEGER NOT NULL DEFAULT 0,
    parent_contract_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_type_id) REFERENCES contract_types(id),
    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_contracts_tenant ON contracts(tenant_id, supplier_id, status);
CREATE INDEX idx_contracts_dates ON contracts(tenant_id, start_date, end_date);
CREATE INDEX idx_contracts_owner ON contracts(tenant_id, contract_owner_id);
```

### 2.3 Contract Lines
```sql
CREATE TABLE contract_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT,
    item_description TEXT NOT NULL,
    quantity REAL,
    uom TEXT,
    unit_price INTEGER NOT NULL,
    line_amount INTEGER NOT NULL DEFAULT 0,
    delivery_schedule TEXT,
    minimum_order_qty REAL,
    maximum_order_qty REAL,
    lead_time_days INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES contracts(id) ON DELETE CASCADE
);

CREATE INDEX idx_contract_lines_contract ON contract_lines(contract_id, line_number);
```

### 2.4 Contract Terms and Conditions
```sql
CREATE TABLE contract_terms_and_conditions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    clause_code TEXT NOT NULL,
    clause_title TEXT NOT NULL,
    clause_text TEXT NOT NULL,
    clause_type TEXT NOT NULL
        CHECK(clause_type IN ('STANDARD','REGULATORY','CUSTOM','CONFIDENTIALITY','INDEMNIFICATION','TERMINATION','FORCE_MAJEURE','PAYMENT','DELIVERY','WARRANTY')),
    is_mandatory INTEGER NOT NULL DEFAULT 0,
    section_reference TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES contracts(id) ON DELETE CASCADE
);

CREATE INDEX idx_contract_terms_contract ON contract_terms_and_conditions(contract_id, sort_order);
```

### 2.5 Contract Amendments
```sql
CREATE TABLE contract_amendments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    amendment_number TEXT NOT NULL,
    amendment_type TEXT NOT NULL
        CHECK(amendment_type IN ('SCOPE_CHANGE','PRICE_CHANGE','TERM_EXTENSION','TERMINATION','RENEWAL','OTHER')),
    description TEXT NOT NULL,
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PENDING_APPROVAL','APPROVED','REJECTED')),
    old_value TEXT,
    new_value TEXT,
    effective_date TEXT NOT NULL,
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES contracts(id),
    UNIQUE(tenant_id, contract_id, amendment_number)
);

CREATE INDEX idx_amendments_contract ON contract_amendments(contract_id, status);
```

### 2.6 Contract Renewals
```sql
CREATE TABLE contract_renewals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    renewal_number INTEGER NOT NULL,
    old_end_date TEXT NOT NULL,
    new_end_date TEXT NOT NULL,
    new_total_value INTEGER,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED')),
    renewal_notes TEXT,
    renewed_by TEXT,
    renewed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES contracts(id)
);

CREATE INDEX idx_renewals_contract ON contract_renewals(contract_id);
```

### 2.7 Contract Compliance Rules
```sql
CREATE TABLE contract_compliance_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('SPEND_LIMIT','DELIVERY_DEADLINE','INSURANCE_REQUIREMENT','CERTIFICATION','REGULATORY','DOCUMENT_SUBMISSION')),
    rule_description TEXT NOT NULL,
    threshold_value TEXT,
    threshold_unit TEXT,
    is_blocking INTEGER NOT NULL DEFAULT 0,
    check_frequency TEXT NOT NULL DEFAULT 'CONTINUOUS'
        CHECK(check_frequency IN ('CONTINUOUS','DAILY','WEEKLY','MONTHLY')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES contracts(id) ON DELETE CASCADE
);

CREATE INDEX idx_compliance_rules_contract ON contract_compliance_rules(contract_id, rule_type);
```

### 2.8 Spending Against Contracts
```sql
CREATE TABLE spending_against_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    contract_line_id TEXT,
    purchase_order_id TEXT,
    po_line_id TEXT,
    spent_amount INTEGER NOT NULL DEFAULT 0,
    spent_date TEXT NOT NULL,
    invoice_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (contract_id) REFERENCES contracts(id),
    FOREIGN KEY (contract_line_id) REFERENCES contract_lines(id)
);

CREATE INDEX idx_spending_contract ON spending_against_contracts(contract_id, spent_date);
CREATE INDEX idx_spending_line ON spending_against_contracts(contract_line_id);
```

---

## 3. REST API Endpoints

### 3.1 Contract Types
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts/types` | List contract types |
| POST | `/api/v1/contracts/types` | Create contract type |
| PUT | `/api/v1/contracts/types/{id}` | Update contract type |

### 3.2 Contracts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts` | List contracts (filter by supplier, status, type) |
| POST | `/api/v1/contracts` | Create contract |
| GET | `/api/v1/contracts/{id}` | Get contract with lines and terms |
| PUT | `/api/v1/contracts/{id}` | Update contract |
| POST | `/api/v1/contracts/{id}/submit` | Submit for approval |
| POST | `/api/v1/contracts/{id}/approve` | Approve contract |
| POST | `/api/v1/contracts/{id}/activate` | Activate contract |
| POST | `/api/v1/contracts/{id}/suspend` | Suspend contract |
| POST | `/api/v1/contracts/{id}/terminate` | Terminate contract |
| POST | `/api/v1/contracts/{id}/close` | Close contract |

### 3.3 Contract Lines
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts/{id}/lines` | List contract lines |
| POST | `/api/v1/contracts/{id}/lines` | Add contract line |
| PUT | `/api/v1/contracts/{id}/lines/{lineId}` | Update contract line |

### 3.4 Terms and Conditions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts/{id}/terms` | List contract terms |
| POST | `/api/v1/contracts/{id}/terms` | Add term |
| PUT | `/api/v1/contracts/{id}/terms/{termId}` | Update term |
| DELETE | `/api/v1/contracts/{id}/terms/{termId}` | Remove term |

### 3.5 Amendments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts/{id}/amendments` | List amendments |
| POST | `/api/v1/contracts/{id}/amendments` | Create amendment |
| POST | `/api/v1/contracts/{id}/amendments/{amdId}/approve` | Approve amendment |

### 3.6 Renewals
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts/{id}/renewals` | List renewals |
| POST | `/api/v1/contracts/{id}/renew` | Renew contract |

### 3.7 Compliance & Spending
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contracts/{id}/compliance` | Get compliance status |
| POST | `/api/v1/contracts/{id}/compliance-rules` | Add compliance rule |
| GET | `/api/v1/contracts/{id}/spending` | Get spending summary |
| GET | `/api/v1/contracts/spending-analysis` | Cross-contract spending analysis |

---

## 4. Business Rules

1. Contracts MUST have at least one line item before submission.
2. Total spending against a contract MUST NOT exceed the contract total value unless allowed by amendment.
3. Contract status transitions MUST follow: DRAFT → PENDING_REVIEW → PENDING_APPROVAL → APPROVED → ACTIVE → CLOSED/EXPIRED/TERMINATED.
4. Amendments MUST be approved before taking effect.
5. Auto-renewal MUST trigger a notification before the renewal date.
6. Expired contracts MUST NOT be used for new purchase orders.
7. Compliance rules with is_blocking=1 MUST prevent contract use when violated.
8. Contract line prices MUST be used when creating purchase orders against the contract.
9. Spending records MUST be immutable once created.
10. Contract terms MUST include all mandatory clauses before approval.
11. Terminated contracts MUST go through compensation/closeout process.
12. Renewal MUST create a new contract version with updated dates and terms.

---

## 5. gRPC Service Definition

```protobuf
service ContractsService {
    rpc CreateContract(CreateContractRequest) returns (ContractResponse);
    rpc GetContract(GetContractRequest) returns (ContractDetailResponse);
    rpc ListContracts(ListContractsRequest) returns (ListContractsResponse);
    rpc SubmitContract(SubmitContractRequest) returns (ContractResponse);
    rpc ApproveContract(ApproveContractRequest) returns (ContractResponse);
    rpc ActivateContract(ActivateContractRequest) returns (ContractResponse);

    rpc AddContractLine(AddContractLineRequest) returns (ContractLineResponse);

    rpc CreateAmendment(CreateAmendmentRequest) returns (AmendmentResponse);
    rpc ApproveAmendment(ApproveAmendmentRequest) returns (AmendmentResponse);

    rpc RenewContract(RenewContractRequest) returns (ContractResponse);

    rpc GetSpendingSummary(GetSpendingSummaryRequest) returns (SpendingSummaryResponse);
    rpc CheckCompliance(CheckComplianceRequest) returns (ComplianceResponse);

    rpc RecordSpending(RecordSpendingRequest) returns (SpendingResponse);
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `proc-service` | Purchase orders against contracts, spending data |
| `supplier-service` | Supplier details, performance data |
| `workflow-service` | Contract approval workflows |
| `gl-service` | GL accounts for contract accruals |

### Published To
| Service | Data |
|---------|------|
| `proc-service` | Contract pricing and terms for PO creation |
| `supplier-service` | Contract status for supplier scorecard |
| `workflow-service` | Approval requests for contract/amendment approval |
| `report-service` | Contract spending analytics, compliance reports |
| `gl-service` | Contract accrual journal entries |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `contracts.contract.created` | `{contract_id, number, supplier_id, type}` | Contract created |
| `contracts.contract.approved` | `{contract_id, approved_by}` | Contract approved |
| `contracts.contract.activated` | `{contract_id, start_date, end_date}` | Contract activated |
| `contracts.contract.expiring` | `{contract_id, days_until_expiry}` | Contract nearing expiry |
| `contracts.contract.expired` | `{contract_id}` | Contract expired |
| `contracts.contract.terminated` | `{contract_id, reason}` | Contract terminated |
| `contracts.amendment.created` | `{contract_id, amendment_id, type}` | Amendment created |
| `contracts.amendment.approved` | `{amendment_id}` | Amendment approved |
| `contracts.renewal.completed` | `{contract_id, new_end_date}` | Contract renewed |
| `contracts.compliance.violated` | `{contract_id, rule_type, description}` | Compliance rule violated |
| `contracts.spending.threshold` | `{contract_id, spent_pct, threshold_pct}` | Spending threshold reached |
