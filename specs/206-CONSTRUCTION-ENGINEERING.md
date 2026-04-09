# 206 - Construction & Engineering Industry Solution Specification

## 1. Domain Overview

Construction & Engineering provides industry-specific capabilities for construction project delivery, progress billing with retainage, BIM coordination, change order management, and compliance documentation. Supports construction project lifecycle from bidding through close-out, unit price and lump sum contract management, progress claim management with retainage tracking, construction change directives and variation orders, daily log and site reporting, safety compliance and incident management, and BIM model coordination. Enables construction and engineering firms to manage complex projects with multiple stakeholders, maintain regulatory compliance, and control costs throughout the project lifecycle. Integrates with Project Management, Financials, and Procurement.

**Bounded Context:** Construction Project Delivery & Engineering Management
**Service Name:** `construction-engineering-service`
**Database:** `data/construction_engineering.db`
**HTTP Port:** 8224 | **gRPC Port:** 9224

---

## 2. Database Schema

### 2.1 Construction Projects
```sql
CREATE TABLE ce_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_number TEXT NOT NULL,
    project_name TEXT NOT NULL,
    project_type TEXT NOT NULL CHECK(project_type IN ('GENERAL_CONSTRUCTION','DESIGN_BUILD','EPC','RENOVATION','INFRASTRUCTURE','RESIDENTIAL','COMMERCIAL')),
    client_id TEXT NOT NULL,
    contract_type TEXT NOT NULL CHECK(contract_type IN ('LUMP_SUM','UNIT_PRICE','COST_PLUS','GMP','TIME_MATERIALS')),
    contract_value_cents INTEGER NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    location TEXT NOT NULL,
    site_address TEXT,
    start_date TEXT NOT NULL,
    planned_end_date TEXT NOT NULL,
    actual_end_date TEXT,
    project_manager_id TEXT NOT NULL,
    superintendent_id TEXT,
    prime_contractor_id TEXT,
    permit_number TEXT,
    permit_expiry TEXT,
    safety_plan_id TEXT,
    bim_model_reference TEXT,
    weather_delay_days INTEGER NOT NULL DEFAULT 0,
    completion_pct REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNING'
        CHECK(status IN ('PLANNING','BIDDING','IN_PROGRESS','SUBSTANTIALLY_COMPLETE','COMPLETED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_number)
);

CREATE INDEX idx_ce_proj_tenant ON ce_projects(tenant_id, status);
CREATE INDEX idx_ce_proj_client ON ce_projects(client_id);
CREATE INDEX idx_ce_proj_dates ON ce_projects(start_date, planned_end_date);
CREATE INDEX idx_ce_proj_pm ON ce_projects(project_manager_id);
```

### 2.2 Progress Claims
```sql
CREATE TABLE ce_progress_claims (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    claim_number TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    work_completed_pct REAL NOT NULL DEFAULT 0,
    work_completed_value_cents INTEGER NOT NULL DEFAULT 0,
    materials_stored_cents INTEGER NOT NULL DEFAULT 0,
    total_claimed_cents INTEGER NOT NULL DEFAULT 0,
    retainage_pct REAL NOT NULL DEFAULT 10,
    retainage_held_cents INTEGER NOT NULL DEFAULT 0,
    previous_certified_cents INTEGER NOT NULL DEFAULT 0,
    net_certified_cents INTEGER NOT NULL DEFAULT 0,
    change_orders_included TEXT,                  -- JSON: change order IDs
    supporting_documents TEXT,                    -- JSON: document references
    certified_by TEXT,
    certified_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','CERTIFIED','REJECTED','PAID')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES ce_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, claim_number)
);

CREATE INDEX idx_ce_claim_project ON ce_progress_claims(project_id, status);
CREATE INDEX idx_ce_claim_period ON ce_progress_claims(period_start, period_end);
CREATE INDEX idx_ce_claim_status ON ce_progress_claims(tenant_id, status);
```

### 2.3 Change Orders
```sql
CREATE TABLE ce_change_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    co_number TEXT NOT NULL,
    co_type TEXT NOT NULL CHECK(co_type IN ('OWNER_DIRECTED','CONSTRUCTION_CHANGE_DIRECTIVE','VARIATION','FIELD_ORDER','CLAIM')),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    reason TEXT NOT NULL,
    impact_schedule_days INTEGER NOT NULL DEFAULT 0,
    impact_cost_cents INTEGER NOT NULL DEFAULT 0,
    proposed_cost_cents INTEGER NOT NULL DEFAULT 0,
    approved_cost_cents INTEGER,
    affected_work TEXT NOT NULL,                  -- JSON: work items affected
    documents TEXT,                               -- JSON: supporting documents
    requested_by TEXT NOT NULL,
    approved_by TEXT,
    approved_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED','EXECUTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES ce_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, co_number)
);

CREATE INDEX idx_ce_co_project ON ce_change_orders(project_id, status);
CREATE INDEX idx_ce_co_type ON ce_change_orders(co_type);
```

### 2.4 Daily Logs
```sql
CREATE TABLE ce_daily_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    log_date TEXT NOT NULL,
    weather_conditions TEXT,                      -- JSON: temp, precipitation, wind
    weather_delay_hours REAL NOT NULL DEFAULT 0,
    workers_on_site INTEGER NOT NULL DEFAULT 0,
    subcontractors_on_site TEXT,                  -- JSON: [{company, workers, trade}]
    equipment_on_site TEXT,                       -- JSON: [{equipment, hours}]
    work_performed TEXT NOT NULL,
    areas_worked TEXT,
    deliveries TEXT,                              -- JSON: deliveries received
    inspections TEXT,                             -- JSON: inspections conducted
    safety_incidents INTEGER NOT NULL DEFAULT 0,
    visitors TEXT,                                -- JSON: site visitors
    issues TEXT,                                  -- JSON: issues noted
    photos TEXT,                                  -- JSON: photo references
    reported_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, project_id, log_date)
);

CREATE INDEX idx_ce_log_project ON ce_daily_logs(project_id, log_date DESC);
CREATE INDEX idx_ce_log_date ON ce_daily_logs(log_date DESC);
```

### 2.5 Submittals & RFIs
```sql
CREATE TABLE ce_submittals_rfis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    item_type TEXT NOT NULL CHECK(item_type IN ('SUBMITTAL','RFI','SHOP_DRAWING','SAMPLE')),
    item_number TEXT NOT NULL,
    subject TEXT NOT NULL,
    description TEXT NOT NULL,
    specification_reference TEXT,
    drawing_reference TEXT,
    due_date TEXT,
    response_required_date TEXT,
    assigned_to TEXT NOT NULL,
    response TEXT,
    response_by TEXT,
    response_at TEXT,
    attachments TEXT,                             -- JSON: document references
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    cost_impact INTEGER NOT NULL DEFAULT 0,
    schedule_impact_days INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_REVIEW','RESPONDED','CLOSED','VOID')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (project_id) REFERENCES ce_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, item_number)
);

CREATE INDEX idx_ce_rfi_project ON ce_submittals_rfis(project_id, status);
CREATE INDEX idx_ce_rfi_type ON ce_submittals_rfis(item_type, status);
CREATE INDEX idx_ce_rfi_due ON ce_submittals_rfis(due_date);
```

---

## 3. API Endpoints

### 3.1 Projects
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/construction/projects` | Create project |
| GET | `/api/v1/construction/projects` | List projects |
| GET | `/api/v1/construction/projects/{id}` | Get project |
| PUT | `/api/v1/construction/projects/{id}` | Update project |
| GET | `/api/v1/construction/projects/{id}/dashboard` | Project dashboard |

### 3.2 Progress Claims
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/construction/claims` | Create progress claim |
| GET | `/api/v1/construction/claims` | List claims |
| GET | `/api/v1/construction/claims/{id}` | Get claim |
| POST | `/api/v1/construction/claims/{id}/certify` | Certify claim |

### 3.3 Change Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/construction/change-orders` | Create change order |
| GET | `/api/v1/construction/change-orders` | List change orders |
| GET | `/api/v1/construction/change-orders/{id}` | Get change order |
| POST | `/api/v1/construction/change-orders/{id}/approve` | Approve change order |

### 3.4 Daily Logs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/construction/daily-logs` | Submit daily log |
| GET | `/api/v1/construction/daily-logs` | List daily logs |
| GET | `/api/v1/construction/daily-logs/{id}` | Get log details |

### 3.5 Submittals & RFIs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/construction/rfis` | Create RFI/submittal |
| GET | `/api/v1/construction/rfis` | List RFIs |
| GET | `/api/v1/construction/rfis/{id}` | Get RFI |
| POST | `/api/v1/construction/rfis/{id}/respond` | Respond to RFI |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `construct.project.started` | `{ project_id, contract_value, start_date }` | Construction started |
| `construct.claim.certified` | `{ claim_id, amount, retainage }` | Progress claim certified |
| `construct.co.approved` | `{ co_id, impact_cost, impact_days }` | Change order approved |
| `construct.rfi.responded` | `{ rfi_id, schedule_impact }` | RFI responded |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `project.milestone.completed` | Project Management (15) | Update completion % |
| `invoice.approved` | AP (07) | Process progress payment |
| `permit.approved` | External | Update project status |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fusion.construction.v1;

import "google/protobuf/timestamp.proto";

// ── Service ──────────────────────────────────────────────────────────
service ConstructionEngineeringService {
  // Projects
  rpc CreateProject(CreateProjectRequest) returns (Project);
  rpc GetProject(GetProjectRequest) returns (Project);
  rpc ListProjects(ListProjectsRequest) returns (ListProjectsResponse);
  rpc UpdateProject(UpdateProjectRequest) returns (Project);

  // Progress Claims
  rpc CreateProgressClaim(CreateProgressClaimRequest) returns (ProgressClaim);
  rpc CertifyProgressClaim(CertifyProgressClaimRequest) returns (ProgressClaim);

  // Change Orders
  rpc CreateChangeOrder(CreateChangeOrderRequest) returns (ChangeOrder);
  rpc ApproveChangeOrder(ApproveChangeOrderRequest) returns (ChangeOrder);

  // Daily Logs
  rpc SubmitDailyLog(SubmitDailyLogRequest) returns (DailyLog);
  rpc ListDailyLogs(ListDailyLogsRequest) returns (ListDailyLogsResponse);

  // Submittals & RFIs
  rpc CreateSubmittalRfi(CreateSubmittalRfiRequest) returns (SubmittalRfi);
  rpc RespondToSubmittalRfi(RespondToSubmittalRfiRequest) returns (SubmittalRfi);
}

// ── Enums ────────────────────────────────────────────────────────────
enum ProjectType {
  PROJECT_TYPE_UNSPECIFIED = 0;
  PROJECT_TYPE_GENERAL_CONSTRUCTION = 1;
  PROJECT_TYPE_DESIGN_BUILD = 2;
  PROJECT_TYPE_EPC = 3;
  PROJECT_TYPE_RENOVATION = 4;
  PROJECT_TYPE_INFRASTRUCTURE = 5;
  PROJECT_TYPE_RESIDENTIAL = 6;
  PROJECT_TYPE_COMMERCIAL = 7;
}

enum ContractType {
  CONTRACT_TYPE_UNSPECIFIED = 0;
  CONTRACT_TYPE_LUMP_SUM = 1;
  CONTRACT_TYPE_UNIT_PRICE = 2;
  CONTRACT_TYPE_COST_PLUS = 3;
  CONTRACT_TYPE_GMP = 4;
  CONTRACT_TYPE_TIME_MATERIALS = 5;
}

enum ProjectStatus {
  PROJECT_STATUS_UNSPECIFIED = 0;
  PROJECT_STATUS_PLANNING = 1;
  PROJECT_STATUS_BIDDING = 2;
  PROJECT_STATUS_IN_PROGRESS = 3;
  PROJECT_STATUS_SUBSTANTIALLY_COMPLETE = 4;
  PROJECT_STATUS_COMPLETED = 5;
  PROJECT_STATUS_CLOSED = 6;
}

enum ClaimStatus {
  CLAIM_STATUS_UNSPECIFIED = 0;
  CLAIM_STATUS_DRAFT = 1;
  CLAIM_STATUS_SUBMITTED = 2;
  CLAIM_STATUS_UNDER_REVIEW = 3;
  CLAIM_STATUS_CERTIFIED = 4;
  CLAIM_STATUS_REJECTED = 5;
  CLAIM_STATUS_PAID = 6;
}

enum ChangeOrderType {
  CO_TYPE_UNSPECIFIED = 0;
  CO_TYPE_OWNER_DIRECTED = 1;
  CO_TYPE_CONSTRUCTION_CHANGE_DIRECTIVE = 2;
  CO_TYPE_VARIATION = 3;
  CO_TYPE_FIELD_ORDER = 4;
  CO_TYPE_CLAIM = 5;
}

enum ChangeOrderStatus {
  CO_STATUS_UNSPECIFIED = 0;
  CO_STATUS_DRAFT = 1;
  CO_STATUS_SUBMITTED = 2;
  CO_STATUS_UNDER_REVIEW = 3;
  CO_STATUS_APPROVED = 4;
  CO_STATUS_REJECTED = 5;
  CO_STATUS_EXECUTED = 6;
}

enum ItemType {
  ITEM_TYPE_UNSPECIFIED = 0;
  ITEM_TYPE_SUBMITTAL = 1;
  ITEM_TYPE_RFI = 2;
  ITEM_TYPE_SHOP_DRAWING = 3;
  ITEM_TYPE_SAMPLE = 4;
}

enum Priority {
  PRIORITY_UNSPECIFIED = 0;
  PRIORITY_LOW = 1;
  PRIORITY_NORMAL = 2;
  PRIORITY_HIGH = 3;
  PRIORITY_URGENT = 4;
}

enum RfiStatus {
  RFI_STATUS_UNSPECIFIED = 0;
  RFI_STATUS_OPEN = 1;
  RFI_STATUS_IN_REVIEW = 2;
  RFI_STATUS_RESPONDED = 3;
  RFI_STATUS_CLOSED = 4;
  RFI_STATUS_VOID = 5;
}

// ── Common Messages ──────────────────────────────────────────────────
message AuditInfo {
  string created_at = 1;
  string updated_at = 2;
  string created_by = 3;
  string updated_by = 4;
  int32 version = 5;
}

// ── Project Messages ─────────────────────────────────────────────────
message Project {
  string id = 1;
  string tenant_id = 2;
  string project_number = 3;
  string project_name = 4;
  ProjectType project_type = 5;
  string client_id = 6;
  ContractType contract_type = 7;
  int64 contract_value_cents = 8;
  int64 estimated_cost_cents = 9;
  string location = 10;
  string site_address = 11;
  string start_date = 12;
  string planned_end_date = 13;
  string actual_end_date = 14;
  string project_manager_id = 15;
  string superintendent_id = 16;
  string prime_contractor_id = 17;
  string permit_number = 18;
  string permit_expiry = 19;
  string safety_plan_id = 20;
  string bim_model_reference = 21;
  int32 weather_delay_days = 22;
  double completion_pct = 23;
  ProjectStatus status = 24;
  AuditInfo audit = 25;
}

message CreateProjectRequest {
  string tenant_id = 1;
  string project_number = 2;
  string project_name = 3;
  ProjectType project_type = 4;
  string client_id = 5;
  ContractType contract_type = 6;
  int64 contract_value_cents = 7;
  int64 estimated_cost_cents = 8;
  string location = 9;
  string site_address = 10;
  string start_date = 11;
  string planned_end_date = 12;
  string project_manager_id = 13;
  string superintendent_id = 14;
  string prime_contractor_id = 15;
  string permit_number = 16;
  string permit_expiry = 17;
  string safety_plan_id = 18;
  string bim_model_reference = 19;
  string user_id = 20;
}

message GetProjectRequest {
  string id = 1;
  string tenant_id = 2;
}

message ListProjectsRequest {
  string tenant_id = 1;
  ProjectStatus status = 2;
  string client_id = 3;
  string project_manager_id = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListProjectsResponse {
  repeated Project projects = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

message UpdateProjectRequest {
  string id = 1;
  string tenant_id = 2;
  string project_name = 3;
  int64 contract_value_cents = 4;
  int64 estimated_cost_cents = 5;
  string planned_end_date = 6;
  string actual_end_date = 7;
  double completion_pct = 8;
  ProjectStatus status = 9;
  string user_id = 10;
}

// ── Progress Claim Messages ──────────────────────────────────────────
message ProgressClaim {
  string id = 1;
  string tenant_id = 2;
  string project_id = 3;
  string claim_number = 4;
  string period_start = 5;
  string period_end = 6;
  double work_completed_pct = 7;
  int64 work_completed_value_cents = 8;
  int64 materials_stored_cents = 9;
  int64 total_claimed_cents = 10;
  double retainage_pct = 11;
  int64 retainage_held_cents = 12;
  int64 previous_certified_cents = 13;
  int64 net_certified_cents = 14;
  string change_orders_included = 15;  // JSON
  string supporting_documents = 16;    // JSON
  string certified_by = 17;
  string certified_at = 18;
  ClaimStatus status = 19;
  AuditInfo audit = 20;
}

message CreateProgressClaimRequest {
  string tenant_id = 1;
  string project_id = 2;
  string period_start = 3;
  string period_end = 4;
  double work_completed_pct = 5;
  int64 work_completed_value_cents = 6;
  int64 materials_stored_cents = 7;
  double retainage_pct = 8;
  string change_orders_included = 9;   // JSON
  string user_id = 10;
}

message CertifyProgressClaimRequest {
  string id = 1;
  string tenant_id = 2;
  string certified_by = 3;
}

// ── Change Order Messages ────────────────────────────────────────────
message ChangeOrder {
  string id = 1;
  string tenant_id = 2;
  string project_id = 3;
  string co_number = 4;
  ChangeOrderType co_type = 5;
  string title = 6;
  string description = 7;
  string reason = 8;
  int32 impact_schedule_days = 9;
  int64 impact_cost_cents = 10;
  int64 proposed_cost_cents = 11;
  int64 approved_cost_cents = 12;
  string affected_work = 13;           // JSON
  string documents = 14;               // JSON
  string requested_by = 15;
  string approved_by = 16;
  string approved_at = 17;
  ChangeOrderStatus status = 18;
  AuditInfo audit = 19;
}

message CreateChangeOrderRequest {
  string tenant_id = 1;
  string project_id = 2;
  string co_number = 3;
  ChangeOrderType co_type = 4;
  string title = 5;
  string description = 6;
  string reason = 7;
  int32 impact_schedule_days = 8;
  int64 impact_cost_cents = 9;
  int64 proposed_cost_cents = 10;
  string affected_work = 11;           // JSON
  string requested_by = 12;
  string user_id = 13;
}

message ApproveChangeOrderRequest {
  string id = 1;
  string tenant_id = 2;
  string approved_by = 3;
  int64 approved_cost_cents = 4;
}

// ── Daily Log Messages ───────────────────────────────────────────────
message DailyLog {
  string id = 1;
  string tenant_id = 2;
  string project_id = 3;
  string log_date = 4;
  string weather_conditions = 5;       // JSON
  double weather_delay_hours = 6;
  int32 workers_on_site = 7;
  string subcontractors_on_site = 8;   // JSON
  string equipment_on_site = 9;        // JSON
  string work_performed = 10;
  string areas_worked = 11;
  string deliveries = 12;              // JSON
  string inspections = 13;             // JSON
  int32 safety_incidents = 14;
  string visitors = 15;                // JSON
  string issues = 16;                  // JSON
  string photos = 17;                  // JSON
  string reported_by = 18;
  AuditInfo audit = 19;
}

message SubmitDailyLogRequest {
  string tenant_id = 1;
  string project_id = 2;
  string log_date = 3;
  string weather_conditions = 4;       // JSON
  double weather_delay_hours = 5;
  int32 workers_on_site = 6;
  string subcontractors_on_site = 7;   // JSON
  string equipment_on_site = 8;        // JSON
  string work_performed = 9;
  string areas_worked = 10;
  string deliveries = 11;              // JSON
  string inspections = 12;             // JSON
  int32 safety_incidents = 13;
  string visitors = 14;                // JSON
  string issues = 15;                  // JSON
  string photos = 16;                  // JSON
  string reported_by = 17;
  string user_id = 18;
}

message ListDailyLogsRequest {
  string tenant_id = 1;
  string project_id = 2;
  string log_date_from = 3;
  string log_date_to = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListDailyLogsResponse {
  repeated DailyLog daily_logs = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

// ── Submittal / RFI Messages ─────────────────────────────────────────
message SubmittalRfi {
  string id = 1;
  string tenant_id = 2;
  string project_id = 3;
  ItemType item_type = 4;
  string item_number = 5;
  string subject = 6;
  string description = 7;
  string specification_reference = 8;
  string drawing_reference = 9;
  string due_date = 10;
  string response_required_date = 11;
  string assigned_to = 12;
  string response = 13;
  string response_by = 14;
  string response_at = 15;
  string attachments = 16;             // JSON
  Priority priority = 17;
  bool cost_impact = 18;
  int32 schedule_impact_days = 19;
  RfiStatus status = 20;
  AuditInfo audit = 21;
}

message CreateSubmittalRfiRequest {
  string tenant_id = 1;
  string project_id = 2;
  ItemType item_type = 3;
  string item_number = 4;
  string subject = 5;
  string description = 6;
  string specification_reference = 7;
  string drawing_reference = 8;
  string due_date = 9;
  string response_required_date = 10;
  string assigned_to = 11;
  string attachments = 12;             // JSON
  Priority priority = 13;
  bool cost_impact = 14;
  int32 schedule_impact_days = 15;
  string user_id = 16;
}

message RespondToSubmittalRfiRequest {
  string id = 1;
  string tenant_id = 2;
  string response = 3;
  string responded_by = 4;
}
```

## 6. Migration Order

| Order | Table | Depends On |
|-------|-------|------------|
| 1 | `ce_projects` | — |
| 2 | `ce_progress_claims` | `ce_projects` |
| 3 | `ce_change_orders` | `ce_projects` |
| 4 | `ce_daily_logs` | `ce_projects` |
| 5 | `ce_submittals_rfis` | `ce_projects` |

---

## 7. Business Rules

1. **Retainage**: Default 10% retainage held until substantial completion; released per contract terms
2. **Change Order Impact**: All change orders require cost and schedule impact assessment
3. **Daily Log Required**: Active projects require daily log submission by end of business day
4. **RFI Response SLA**: RFIs must be responded to within contractually specified timeframe
5. **Weather Delays**: Weather delays tracked and applied to schedule extension calculations
6. **Progress Verification**: Progress claims verified against on-site inspection before certification

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Project Management (15) | Project structure and milestones |
| Accounts Payable (07) | Progress payment processing |
| Project Costing (158) | Cost tracking and budgeting |
| Document Management (29) | Drawing and submittal storage |
| Procurement (11) | Material and subcontractor ordering |
| Enterprise Scheduler (173) | Scheduled reporting |
