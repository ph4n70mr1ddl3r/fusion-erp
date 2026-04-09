# 209 - Professional Services Industry Solution Specification

## 1. Domain Overview

Professional Services provides industry-specific capabilities for client engagement management, resource utilization optimization, time-and-expense billing, project profitability tracking, and practice management. Supports client opportunity pipeline with win probability, resource demand forecasting with skill matching, utilization tracking at individual and practice level, multiple billing methods (hourly, fixed fee, retainer, milestone), work-in-progress management, and project profitability analysis with margin tracking. Enables consulting, legal, accounting, and IT services firms to optimize resource allocation, maximize billable utilization, and track profitability across engagements. Integrates with Project Management, Resource Management, and Financials.

**Bounded Context:** Professional Services Automation & Practice Management
**Service Name:** `professional-services-service`
**Database:** `data/professional_services.db`
**HTTP Port:** 8227 | **gRPC Port:** 9227

---

## 2. Database Schema

### 2.1 Client Engagements
```sql
CREATE TABLE ps_engagements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    engagement_number TEXT NOT NULL,
    engagement_name TEXT NOT NULL,
    client_id TEXT NOT NULL,
    client_name TEXT NOT NULL,
    practice_area TEXT NOT NULL,
    engagement_type TEXT NOT NULL CHECK(engagement_type IN ('CONSULTING','ADVISORY','IMPLEMENTATION','MANAGED_SERVICE','AUDIT','LEGAL')),
    billing_method TEXT NOT NULL CHECK(billing_method IN ('HOURLY','FIXED_FEE','RETAINER','MILESTONE','BLENDED')),
    contract_value_cents INTEGER NOT NULL DEFAULT 0,
    budget_hours REAL NOT NULL DEFAULT 0,
    bill_rate_cents INTEGER NOT NULL DEFAULT 0,
    cost_rate_cents INTEGER NOT NULL DEFAULT 0,
    engagement_start TEXT NOT NULL,
    engagement_end TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    win_probability_pct REAL NOT NULL DEFAULT 50,
    stage TEXT NOT NULL DEFAULT 'PIPELINE'
        CHECK(stage IN ('PIPELINE','PROPOSAL','NEGOTIATION','WON','ACTIVE','COMPLETED','LOST')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, engagement_number)
);

CREATE INDEX idx_ps_eng_client ON ps_engagements(client_id, stage);
CREATE INDEX idx_ps_eng_practice ON ps_engagements(practice_area, stage);
CREATE INDEX idx_ps_eng_partner ON ps_engagements(partner_id);
CREATE INDEX idx_ps_eng_dates ON ps_engagements(engagement_start, engagement_end);
```

### 2.2 Time & Expense Entries
```sql
CREATE TABLE ps_time_expense (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    engagement_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    entry_date TEXT NOT NULL,
    entry_type TEXT NOT NULL CHECK(entry_type IN ('BILLABLE','NON_BILLABLE','ADMIN','BUSINESS_DEV','TRAINING','PTO')),
    hours REAL NOT NULL DEFAULT 0,
    bill_rate_cents INTEGER NOT NULL DEFAULT 0,
    cost_rate_cents INTEGER NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    expense_type TEXT CHECK(expense_type IN ('TRAVEL','MEALS','ACCOMMODATION','MATERIALS','OTHER')),
    expense_amount_cents INTEGER NOT NULL DEFAULT 0,
    expense_billable INTEGER NOT NULL DEFAULT 1,
    description TEXT NOT NULL,
    approved_by TEXT,
    approved_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED','INVOICED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(id)
);

CREATE INDEX idx_ps_te_engagement ON ps_time_expense(engagement_id, entry_date DESC);
CREATE INDEX idx_ps_te_resource ON ps_time_expense(resource_id, entry_date DESC);
CREATE INDEX idx_ps_te_type ON ps_time_expense(entry_type);
CREATE INDEX idx_ps_te_status ON ps_time_expense(tenant_id, status);
```

### 2.3 Resource Utilization
```sql
CREATE TABLE ps_utilization (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    period TEXT NOT NULL,
    total_available_hours REAL NOT NULL DEFAULT 0,
    billable_hours REAL NOT NULL DEFAULT 0,
    non_billable_hours REAL NOT NULL DEFAULT 0,
    admin_hours REAL NOT NULL DEFAULT 0,
    biz_dev_hours REAL NOT NULL DEFAULT 0,
    training_hours REAL NOT NULL DEFAULT 0,
    pto_hours REAL NOT NULL DEFAULT 0,
    billable_utilization_pct REAL NOT NULL DEFAULT 0,
    total_utilization_pct REAL NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    margin_pct REAL NOT NULL DEFAULT 0,
    target_utilization_pct REAL NOT NULL DEFAULT 80,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, resource_id, period)
);

CREATE INDEX idx_ps_util_resource ON ps_utilization(resource_id, period DESC);
CREATE INDEX idx_ps_util_period ON ps_utilization(period DESC);
```

### 2.4 Work-in-Progress
```sql
CREATE TABLE ps_wip (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    engagement_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    period TEXT NOT NULL,
    unbilled_revenue_cents INTEGER NOT NULL DEFAULT 0,
    unbilled_cost_cents INTEGER NOT NULL DEFAULT 0,
    recognized_revenue_cents INTEGER NOT NULL DEFAULT 0,
    billed_amount_cents INTEGER NOT NULL DEFAULT 0,
    written_off_cents INTEGER NOT NULL DEFAULT 0,
    aging_days INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','BILLED','WRITTEN_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, engagement_id, resource_id, period)
);

CREATE INDEX idx_ps_wip_engagement ON ps_wip(engagement_id, status);
CREATE INDEX idx_ps_wip_aging ON ps_wip(aging_days DESC);
```

### 2.5 Practice Analytics
```sql
CREATE TABLE ps_practice_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    practice_area TEXT NOT NULL,
    period TEXT NOT NULL,
    total_headcount INTEGER NOT NULL DEFAULT 0,
    avg_billable_utilization_pct REAL NOT NULL DEFAULT 0,
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    gross_margin_pct REAL NOT NULL DEFAULT 0,
    avg_realization_rate_pct REAL NOT NULL DEFAULT 0,
    avg_bill_rate_cents INTEGER NOT NULL DEFAULT 0,
    new_engagements INTEGER NOT NULL DEFAULT 0,
    completed_engagements INTEGER NOT NULL DEFAULT 0,
    active_engagements INTEGER NOT NULL DEFAULT 0,
    pipeline_value_cents INTEGER NOT NULL DEFAULT 0,
    win_rate_pct REAL NOT NULL DEFAULT 0,
    client_satisfaction_score REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, practice_area, period)
);

CREATE INDEX idx_ps_pa_practice ON ps_practice_analytics(practice_area, period DESC);
```

---

## 3. API Endpoints

### 3.1 Engagements
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/professional-services/engagements` | Create engagement |
| GET | `/api/v1/professional-services/engagements` | List engagements |
| GET | `/api/v1/professional-services/engagements/{id}` | Get engagement |
| PUT | `/api/v1/professional-services/engagements/{id}` | Update engagement |
| GET | `/api/v1/professional-services/engagements/{id}/profitability` | Profitability analysis |

### 3.2 Time & Expense
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/professional-services/time-entries` | Submit time entry |
| POST | `/api/v1/professional-services/expenses` | Submit expense |
| GET | `/api/v1/professional-services/time-entries` | List entries |
| POST | `/api/v1/professional-services/time-entries/{id}/approve` | Approve entry |

### 3.3 Utilization
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/professional-services/utilization` | Utilization dashboard |
| GET | `/api/v1/professional-services/utilization/{resourceId}` | Resource utilization |
| GET | `/api/v1/professional-services/utilization/team` | Team utilization |

### 3.4 WIP Management
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/professional-services/wip` | WIP summary |
| GET | `/api/v1/professional-services/wip/{engagementId}` | Engagement WIP |
| POST | `/api/v1/professional-services/wip/invoice` | Generate invoice from WIP |

### 3.5 Practice Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/professional-services/analytics/dashboard` | Practice dashboard |
| GET | `/api/v1/professional-services/analytics/pipeline` | Pipeline analysis |
| GET | `/api/v1/professional-services/analytics/margins` | Margin trends |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `psvc.engagement.won` | `{ engagement_id, client, value }` | Engagement won |
| `psvc.time.submitted` | `{ entry_id, resource, hours, billable }` | Time entry submitted |
| `psvc.wip.threshold` | `{ engagement_id, unbilled_amount, days }` | WIP aging threshold |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `opportunity.closed_won` | Sales (77) | Create engagement |
| `invoice.sent` | AR (08) | Update WIP status |
| `resource.assigned` | Resource Mgmt (160) | Update utilization |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fusion.professional_services.v1;

import "google/protobuf/timestamp.proto";

// ── Service ──────────────────────────────────────────────────────────
service ProfessionalServicesService {
  // Engagements
  rpc CreateEngagement(CreateEngagementRequest) returns (Engagement);
  rpc GetEngagement(GetEngagementRequest) returns (Engagement);
  rpc ListEngagements(ListEngagementsRequest) returns (ListEngagementsResponse);
  rpc UpdateEngagement(UpdateEngagementRequest) returns (Engagement);

  // Time & Expense
  rpc SubmitTimeEntry(SubmitTimeEntryRequest) returns (TimeExpenseEntry);
  rpc ApproveTimeEntry(ApproveTimeEntryRequest) returns (TimeExpenseEntry);

  // Utilization
  rpc GetResourceUtilization(GetResourceUtilizationRequest) returns (Utilization);
  rpc ListTeamUtilization(ListTeamUtilizationRequest) returns (ListTeamUtilizationResponse);

  // WIP Management
  rpc GetEngagementWip(GetEngagementWipRequest) returns (ListWipResponse);
  rpc GenerateInvoiceFromWip(GenerateInvoiceFromWipRequest) returns (GenerateInvoiceFromWipResponse);
}

// ── Enums ────────────────────────────────────────────────────────────
enum EngagementType {
  ENGAGEMENT_TYPE_UNSPECIFIED = 0;
  ENGAGEMENT_TYPE_CONSULTING = 1;
  ENGAGEMENT_TYPE_ADVISORY = 2;
  ENGAGEMENT_TYPE_IMPLEMENTATION = 3;
  ENGAGEMENT_TYPE_MANAGED_SERVICE = 4;
  ENGAGEMENT_TYPE_AUDIT = 5;
  ENGAGEMENT_TYPE_LEGAL = 6;
}

enum BillingMethod {
  BILLING_METHOD_UNSPECIFIED = 0;
  BILLING_METHOD_HOURLY = 1;
  BILLING_METHOD_FIXED_FEE = 2;
  BILLING_METHOD_RETAINER = 3;
  BILLING_METHOD_MILESTONE = 4;
  BILLING_METHOD_BLENDED = 5;
}

enum EngagementStage {
  ENGAGEMENT_STAGE_UNSPECIFIED = 0;
  ENGAGEMENT_STAGE_PIPELINE = 1;
  ENGAGEMENT_STAGE_PROPOSAL = 2;
  ENGAGEMENT_STAGE_NEGOTIATION = 3;
  ENGAGEMENT_STAGE_WON = 4;
  ENGAGEMENT_STAGE_ACTIVE = 5;
  ENGAGEMENT_STAGE_COMPLETED = 6;
  ENGAGEMENT_STAGE_LOST = 7;
}

enum EntryType {
  ENTRY_TYPE_UNSPECIFIED = 0;
  ENTRY_TYPE_BILLABLE = 1;
  ENTRY_TYPE_NON_BILLABLE = 2;
  ENTRY_TYPE_ADMIN = 3;
  ENTRY_TYPE_BUSINESS_DEV = 4;
  ENTRY_TYPE_TRAINING = 5;
  ENTRY_TYPE_PTO = 6;
}

enum ExpenseType {
  EXPENSE_TYPE_UNSPECIFIED = 0;
  EXPENSE_TYPE_TRAVEL = 1;
  EXPENSE_TYPE_MEALS = 2;
  EXPENSE_TYPE_ACCOMMODATION = 3;
  EXPENSE_TYPE_MATERIALS = 4;
  EXPENSE_TYPE_OTHER = 5;
}

enum TimeEntryStatus {
  TIME_ENTRY_STATUS_UNSPECIFIED = 0;
  TIME_ENTRY_STATUS_DRAFT = 1;
  TIME_ENTRY_STATUS_SUBMITTED = 2;
  TIME_ENTRY_STATUS_APPROVED = 3;
  TIME_ENTRY_STATUS_REJECTED = 4;
  TIME_ENTRY_STATUS_INVOICED = 5;
}

enum WipStatus {
  WIP_STATUS_UNSPECIFIED = 0;
  WIP_STATUS_OPEN = 1;
  WIP_STATUS_BILLED = 2;
  WIP_STATUS_WRITTEN_OFF = 3;
}

// ── Common Messages ──────────────────────────────────────────────────
message AuditInfo {
  string created_at = 1;
  string updated_at = 2;
  string created_by = 3;
  string updated_by = 4;
  int32 version = 5;
}

// ── Engagement Messages ──────────────────────────────────────────────
message Engagement {
  string id = 1;
  string tenant_id = 2;
  string engagement_number = 3;
  string engagement_name = 4;
  string client_id = 5;
  string client_name = 6;
  string practice_area = 7;
  EngagementType engagement_type = 8;
  BillingMethod billing_method = 9;
  int64 contract_value_cents = 10;
  double budget_hours = 11;
  int64 bill_rate_cents = 12;
  int64 cost_rate_cents = 13;
  string engagement_start = 14;
  string engagement_end = 15;
  string partner_id = 16;
  string manager_id = 17;
  double win_probability_pct = 18;
  EngagementStage stage = 19;
  AuditInfo audit = 20;
}

message CreateEngagementRequest {
  string tenant_id = 1;
  string engagement_number = 2;
  string engagement_name = 3;
  string client_id = 4;
  string client_name = 5;
  string practice_area = 6;
  EngagementType engagement_type = 7;
  BillingMethod billing_method = 8;
  int64 contract_value_cents = 9;
  double budget_hours = 10;
  int64 bill_rate_cents = 11;
  int64 cost_rate_cents = 12;
  string engagement_start = 13;
  string engagement_end = 14;
  string partner_id = 15;
  string manager_id = 16;
  string user_id = 17;
}

message GetEngagementRequest {
  string id = 1;
  string tenant_id = 2;
}

message ListEngagementsRequest {
  string tenant_id = 1;
  EngagementStage stage = 2;
  string client_id = 3;
  string practice_area = 4;
  string partner_id = 5;
  int32 page_size = 6;
  string page_token = 7;
}

message ListEngagementsResponse {
  repeated Engagement engagements = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

message UpdateEngagementRequest {
  string id = 1;
  string tenant_id = 2;
  string engagement_name = 3;
  int64 contract_value_cents = 4;
  double budget_hours = 5;
  int64 bill_rate_cents = 6;
  int64 cost_rate_cents = 7;
  string engagement_end = 8;
  double win_probability_pct = 9;
  EngagementStage stage = 10;
  string user_id = 11;
}

// ── Time & Expense Messages ──────────────────────────────────────────
message TimeExpenseEntry {
  string id = 1;
  string tenant_id = 2;
  string engagement_id = 3;
  string resource_id = 4;
  string entry_date = 5;
  EntryType entry_type = 6;
  double hours = 7;
  int64 bill_rate_cents = 8;
  int64 cost_rate_cents = 9;
  int64 revenue_cents = 10;
  int64 cost_cents = 11;
  ExpenseType expense_type = 12;
  int64 expense_amount_cents = 13;
  bool expense_billable = 14;
  string description = 15;
  string approved_by = 16;
  string approved_at = 17;
  TimeEntryStatus status = 18;
  AuditInfo audit = 19;
}

message SubmitTimeEntryRequest {
  string tenant_id = 1;
  string engagement_id = 2;
  string resource_id = 3;
  string entry_date = 4;
  EntryType entry_type = 5;
  double hours = 6;
  int64 bill_rate_cents = 7;
  int64 cost_rate_cents = 8;
  ExpenseType expense_type = 9;
  int64 expense_amount_cents = 10;
  bool expense_billable = 11;
  string description = 12;
  string user_id = 13;
}

message ApproveTimeEntryRequest {
  string id = 1;
  string tenant_id = 2;
  string approved_by = 3;
}

// ── Utilization Messages ─────────────────────────────────────────────
message Utilization {
  string id = 1;
  string tenant_id = 2;
  string resource_id = 3;
  string period = 4;
  double total_available_hours = 5;
  double billable_hours = 6;
  double non_billable_hours = 7;
  double admin_hours = 8;
  double biz_dev_hours = 9;
  double training_hours = 10;
  double pto_hours = 11;
  double billable_utilization_pct = 12;
  double total_utilization_pct = 13;
  int64 revenue_cents = 14;
  int64 cost_cents = 15;
  double margin_pct = 16;
  double target_utilization_pct = 17;
  string created_at = 18;
  string updated_at = 19;
}

message GetResourceUtilizationRequest {
  string resource_id = 1;
  string tenant_id = 2;
  string period = 3;
}

message ListTeamUtilizationRequest {
  string tenant_id = 1;
  string practice_area = 2;
  string period = 3;
  int32 page_size = 4;
  string page_token = 5;
}

message ListTeamUtilizationResponse {
  repeated Utilization utilizations = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

// ── WIP Messages ─────────────────────────────────────────────────────
message WipEntry {
  string id = 1;
  string tenant_id = 2;
  string engagement_id = 3;
  string resource_id = 4;
  string period = 5;
  int64 unbilled_revenue_cents = 6;
  int64 unbilled_cost_cents = 7;
  int64 recognized_revenue_cents = 8;
  int64 billed_amount_cents = 9;
  int64 written_off_cents = 10;
  int32 aging_days = 11;
  WipStatus status = 12;
  string created_at = 13;
  string updated_at = 14;
}

message GetEngagementWipRequest {
  string engagement_id = 1;
  string tenant_id = 2;
  WipStatus status = 3;
}

message ListWipResponse {
  repeated WipEntry entries = 1;
  int64 total_unbilled_cents = 2;
}

message GenerateInvoiceFromWipRequest {
  string tenant_id = 1;
  string engagement_id = 2;
  repeated string wip_entry_ids = 3;
  string user_id = 4;
}

message GenerateInvoiceFromWipResponse {
  string invoice_id = 1;
  int64 total_invoiced_cents = 2;
  int32 entries_billed = 3;
}
```

## 6. Migration Order

| Order | Table | Depends On |
|-------|-------|------------|
| 1 | `ps_engagements` | — |
| 2 | `ps_time_expense` | `ps_engagements` |
| 3 | `ps_utilization` | — |
| 4 | `ps_wip` | `ps_engagements` |
| 5 | `ps_practice_analytics` | — |

---

## 7. Business Rules

1. **Utilization Targets**: Individual utilization targets tracked; underutilization flagged weekly
2. **WIP Aging**: Unbilled WIP exceeding 60 days flagged for review
3. **Billing Realization**: Actual billed vs standard rates tracked; write-downs require approval
4. **Engagement Margins**: Gross margin monitored against practice benchmarks
5. **Time Entry Compliance**: Time entries required daily; missing entries flagged
6. **Fixed Fee Revenue**: Revenue recognized based on completion percentage for fixed-fee engagements

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Project Management (15) | Project structure and tasks |
| Resource Management (160) | Resource allocation |
| Accounts Receivable (08) | Invoicing |
| Project Costing (158) | Cost tracking |
| Sales Automation (77) | Opportunity pipeline |
| HCM Analytics (148) | Workforce metrics |
