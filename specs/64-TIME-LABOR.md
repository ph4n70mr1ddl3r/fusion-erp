# 64 - Time and Labor Service Specification

## 1. Domain Overview

The Time and Labor service manages employee time tracking, work schedules, time approvals, absence entries, labor distribution, overtime calculations, and time device integration. It captures detailed time entries, enforces work schedule policies, and feeds data to payroll for compensation calculation and project management for cost tracking.

**Bounded Context:** Time Tracking & Labor Management
**Service Name:** `timelabor-service`
**Database:** `data/timelabor.db`
**HTTP Port:** 8096 | **gRPC Port:** 9096

---

## 2. Database Schema

### 2.1 Time Card Templates
```sql
CREATE TABLE time_card_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    period_type TEXT NOT NULL CHECK(period_type IN ('WEEKLY','BIWEEKLY','SEMI_MONTHLY','MONTHLY')),
    start_day_of_week INTEGER NOT NULL DEFAULT 1 CHECK(start_day_of_week BETWEEN 1 AND 7),
    hours_per_day INTEGER DEFAULT 8,
    hours_per_week INTEGER DEFAULT 40,
    overtime_threshold_daily INTEGER DEFAULT 480,   -- minutes
    overtime_threshold_weekly INTEGER DEFAULT 2400,  -- minutes
    double_time_threshold INTEGER DEFAULT 720,       -- minutes
    rounding_rule TEXT DEFAULT 'NEAREST_15' CHECK(rounding_rule IN ('NONE','NEAREST_15','NEAREST_30','UP_15','DOWN_15')),
    grace_period_minutes INTEGER DEFAULT 7,
    auto_break_rule TEXT,  -- JSON: { after_minutes: 360, break_minutes: 30 }
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_name)
);

CREATE INDEX idx_timecard_tmpls_tenant ON time_card_templates(tenant_id, status);
```

### 2.2 Time Cards
```sql
CREATE TABLE time_cards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    assignment_id TEXT NOT NULL,
    template_id TEXT,
    card_number TEXT NOT NULL,
    period_start_date TEXT NOT NULL,
    period_end_date TEXT NOT NULL,
    total_regular_hours DECIMAL(10,2) DEFAULT 0,
    total_overtime_hours DECIMAL(10,2) DEFAULT 0,
    total_double_time_hours DECIMAL(10,2) DEFAULT 0,
    total_absence_hours DECIMAL(10,2) DEFAULT 0,
    total_hours DECIMAL(10,2) DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED','PROCESSING','PROCESSED','EXCEPTION')),
    submitted_at TEXT,
    approved_by TEXT,
    approved_at TEXT,
    rejected_reason TEXT,
    comments TEXT,
    exception_count INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, card_number)
);

CREATE INDEX idx_timecards_tenant_employee ON time_cards(tenant_id, employee_id, period_start_date);
CREATE INDEX idx_timecards_tenant_status ON time_cards(tenant_id, status);
CREATE INDEX idx_timecards_tenant_period ON time_cards(tenant_id, period_start_date, period_end_date);
```

### 2.3 Time Entries
```sql
CREATE TABLE time_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    time_card_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    entry_date TEXT NOT NULL,
    start_time TEXT,
    end_time TEXT,
    duration_minutes INTEGER NOT NULL DEFAULT 0,
    entry_type TEXT NOT NULL CHECK(entry_type IN ('REGULAR','OVERTIME','DOUBLE_TIME','ABSENCE','HOLIDAY','TRAINING','ON_CALL','TRAVEL','BREAK')),
    project_id TEXT,
    task_id TEXT,
    labor_distribution_id TEXT,
    work_area_id TEXT,
    cost_center TEXT,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'MANUAL' CHECK(source IN ('MANUAL','CLOCK_IN_OUT','BULK_IMPORT','FORMULA','DEVICE')),
    device_id TEXT,
    location_code TEXT,
    exception_flag INTEGER DEFAULT 0,
    exception_type TEXT CHECK(exception_type IN ('MISSING_PUNCH','EARLY_IN','LATE_OUT','LONG_BREAK','MISSING_BREAK','DUPLICATE','SCHEDULE_CONFLICT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (time_card_id) REFERENCES time_cards(id) ON DELETE CASCADE
);

CREATE INDEX idx_time_entries_tenant_card ON time_entries(tenant_id, time_card_id);
CREATE INDEX idx_time_entries_tenant_employee ON time_entries(tenant_id, employee_id, entry_date);
CREATE INDEX idx_time_entries_tenant_project ON time_entries(tenant_id, project_id, entry_date);
CREATE INDEX idx_time_entries_tenant_date ON time_entries(tenant_id, entry_date);
```

### 2.4 Time Entry Attributes
```sql
CREATE TABLE time_entry_attributes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    time_entry_id TEXT NOT NULL,
    attribute_name TEXT NOT NULL,
    attribute_value TEXT NOT NULL,
    attribute_type TEXT DEFAULT 'TEXT' CHECK(attribute_type IN ('TEXT','NUMBER','DATE','BOOLEAN')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (time_entry_id) REFERENCES time_entries(id) ON DELETE CASCADE
);

CREATE INDEX idx_time_attrs_tenant_entry ON time_entry_attributes(tenant_id, time_entry_id);
```

### 2.5 Time Approval Routes
```sql
CREATE TABLE time_approval_routes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    time_card_id TEXT NOT NULL,
    approver_id TEXT NOT NULL,
    approval_level INTEGER NOT NULL DEFAULT 1,
    approval_type TEXT NOT NULL DEFAULT 'SEQUENTIAL' CHECK(approval_type IN ('SEQUENTIAL','PARALLEL','ANY')),
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','APPROVED','REJECTED','SKIPPED')),
    approved_at TEXT,
    rejected_reason TEXT,
    delegated_to TEXT,
    notification_sent INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (time_card_id) REFERENCES time_cards(id) ON DELETE CASCADE
);

CREATE INDEX idx_time_approvals_tenant_card ON time_approval_routes(tenant_id, time_card_id);
CREATE INDEX idx_time_approvals_tenant_approver ON time_approval_routes(tenant_id, approver_id, status);
```

### 2.6 Absence Entries
```sql
CREATE TABLE absence_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    time_card_id TEXT,
    absence_type_id TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    start_time TEXT,
    end_time TEXT,
    duration_hours DECIMAL(10,2) NOT NULL,
    duration_days DECIMAL(10,2),
    reason TEXT,
    approval_status TEXT DEFAULT 'PENDING' CHECK(approval_status IN ('PENDING','APPROVED','REJECTED','CANCELLED')),
    approved_by TEXT,
    approved_at TEXT,
    absence_plan_id TEXT,
    certification_required INTEGER DEFAULT 0,
    certification_provided INTEGER DEFAULT 0,
    source TEXT DEFAULT 'SELF_SERVICE' CHECK(source IN ('SELF_SERVICE','MANAGER','SYSTEM','IMPORT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (time_card_id) REFERENCES time_cards(id) ON DELETE SET NULL
);

CREATE INDEX idx_absence_entries_tenant_employee ON absence_entries(tenant_id, employee_id, start_date);
CREATE INDEX idx_absence_entries_tenant_dates ON absence_entries(tenant_id, start_date, end_date);
CREATE INDEX idx_absence_entries_tenant_status ON absence_entries(tenant_id, approval_status);
```

### 2.7 Work Schedule Patterns
```sql
CREATE TABLE work_schedule_patterns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pattern_name TEXT NOT NULL,
    pattern_type TEXT NOT NULL CHECK(pattern_type IN ('FIXED','ROTATING','FLEXIBLE','SHIFT')),
    monday_start TEXT,
    monday_end TEXT,
    monday_hours DECIMAL(4,2) DEFAULT 0,
    tuesday_start TEXT,
    tuesday_end TEXT,
    tuesday_hours DECIMAL(4,2) DEFAULT 0,
    wednesday_start TEXT,
    wednesday_end TEXT,
    wednesday_hours DECIMAL(4,2) DEFAULT 0,
    thursday_start TEXT,
    thursday_end TEXT,
    thursday_hours DECIMAL(4,2) DEFAULT 0,
    friday_start TEXT,
    friday_end TEXT,
    friday_hours DECIMAL(4,2) DEFAULT 0,
    saturday_start TEXT,
    saturday_end TEXT,
    saturday_hours DECIMAL(4,2) DEFAULT 0,
    sunday_start TEXT,
    sunday_end TEXT,
    sunday_hours DECIMAL(4,2) DEFAULT 0,
    rotation_length_days INTEGER,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, pattern_name)
);

CREATE INDEX idx_work_sched_tenant_type ON work_schedule_patterns(tenant_id, pattern_type);
```

### 2.8 Time Rules
```sql
CREATE TABLE time_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('OVERTIME','PREMIUM','ROUNDING','ELIGIBILITY','VALIDATION','ATTENDANCE')),
    priority INTEGER NOT NULL DEFAULT 100,
    condition_expression TEXT NOT NULL,
    action_expression TEXT NOT NULL,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_time_rules_tenant_type ON time_rules(tenant_id, rule_type, priority);
```

### 2.9 Labor Distribution
```sql
CREATE TABLE labor_distribution (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    time_entry_id TEXT,
    project_id TEXT,
    task_id TEXT,
    cost_center TEXT,
    work_type TEXT,
    percentage DECIMAL(5,2) NOT NULL DEFAULT 100.00,
    hours_split DECIMAL(10,2),
    effective_date TEXT NOT NULL,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (time_entry_id) REFERENCES time_entries(id) ON DELETE SET NULL
);

CREATE INDEX idx_labor_dist_tenant_employee ON labor_distribution(tenant_id, employee_id, effective_date);
CREATE INDEX idx_labor_dist_tenant_project ON labor_distribution(tenant_id, project_id);
```

### 2.10 Time Device Events
```sql
CREATE TABLE time_device_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN ('CLOCK_IN','CLOCK_OUT','BREAK_START','BREAK_END','TRANSFER_IN','TRANSFER_OUT')),
    event_timestamp TEXT NOT NULL,
    location_code TEXT,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    processed_flag INTEGER NOT NULL DEFAULT 0,
    time_entry_id TEXT,
    processing_error TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_time_device_tenant_employee ON time_device_events(tenant_id, employee_id, event_timestamp);
CREATE INDEX idx_time_device_tenant_processed ON time_device_events(tenant_id, processed_flag);
```

---

## 3. REST API Endpoints

### Time Cards
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/time-cards` | Create a new time card for a period |
| GET | `/api/v1/time-cards` | List time cards with filters |
| GET | `/api/v1/time-cards/{id}` | Get time card with all entries |
| PUT | `/api/v1/time-cards/{id}` | Update time card header |
| POST | `/api/v1/time-cards/{id}/submit` | Submit time card for approval |
| POST | `/api/v1/time-cards/{id}/recall` | Recall a submitted time card |
| DELETE | `/api/v1/time-cards/{id}` | Delete a draft time card |

### Time Entries
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/time-cards/{id}/entries` | Add time entry to a card |
| PUT | `/api/v1/time-entries/{entryId}` | Update a time entry |
| DELETE | `/api/v1/time-entries/{entryId}` | Delete a time entry |
| POST | `/api/v1/time-entries/bulk` | Bulk create time entries |
| GET | `/api/v1/time-entries` | Search time entries across cards |

### Approvals
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/approvals/pending` | List pending approvals for approver |
| POST | `/api/v1/time-cards/{id}/approve` | Approve a time card |
| POST | `/api/v1/time-cards/{id}/reject` | Reject a time card |
| POST | `/api/v1/approvals/bulk-approve` | Bulk approve time cards |

### Exceptions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/exceptions` | List time entry exceptions |
| POST | `/api/v1/time-entries/{id}/resolve-exception` | Resolve an exception |
| GET | `/api/v1/exceptions/summary` | Get exception summary statistics |

### Overtime
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/overtime/summary` | Get overtime summary for period |
| GET | `/api/v1/overtime/employee/{employeeId}` | Get employee overtime details |
| POST | `/api/v1/overtime/calculate` | Trigger overtime calculation |

### Labor Distribution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/labor-distribution` | Create labor distribution |
| GET | `/api/v1/labor-distribution` | List labor distributions |
| PUT | `/api/v1/labor-distribution/{id}` | Update labor distribution |
| GET | `/api/v1/labor-distribution/project/{projectId}` | Get project labor summary |

### Work Schedules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/schedules` | Create work schedule pattern |
| GET | `/api/v1/schedules` | List schedule patterns |
| PUT | `/api/v1/schedules/{id}` | Update schedule pattern |
| GET | `/api/v1/employees/{employeeId}/schedule` | Get employee schedule |

### Device Events
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/device-events` | Record a device clock event |
| GET | `/api/v1/device-events` | List device events |
| POST | `/api/v1/device-events/process` | Process unprocessed device events |

---

## 4. Business Rules

1. A time card MUST cover exactly one reporting period as defined by the associated template.
2. An employee MUST NOT have overlapping time entries on the same date for the same entry type unless labor distribution is used.
3. Total hours on a time card MUST equal the sum of all time entry durations for that card.
4. Overtime MUST be automatically calculated based on configured thresholds (daily and weekly) when time entries exceed the threshold.
5. A time card in `SUBMITTED` status or beyond MUST NOT have entries added, modified, or deleted without recall.
6. Time entries with exceptions MUST be flagged and MUST NOT be processed for payroll until resolved.
7. Labor distribution percentages for a single time entry MUST sum to 100%.
8. Clock-in and clock-out device events MUST be paired; an unpaired event MUST generate a `MISSING_PUNCH` exception.
9. Time rounding MUST be applied consistently at the time entry level before overtime calculation.
10. Approval routes MUST support at least 3 levels of sequential approval.
11. The system SHOULD auto-generate time cards at the start of each period based on template assignments.
12. Break rules MUST be validated against the configured auto-break thresholds.
13. Absence entries MUST be validated against the employee's absence balance before submission.
14. Double time MUST only apply when both daily and weekly thresholds have been exceeded.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package timelabor.v1;

service TimeLaborService {
    // Time cards
    rpc CreateTimeCard(CreateTimeCardRequest) returns (CreateTimeCardResponse);
    rpc GetTimeCard(GetTimeCardRequest) returns (GetTimeCardResponse);
    rpc ListTimeCards(ListTimeCardsRequest) returns (ListTimeCardsResponse);
    rpc SubmitTimeCard(SubmitTimeCardRequest) returns (SubmitTimeCardResponse);

    // Time entries
    rpc AddTimeEntry(AddTimeEntryRequest) returns (AddTimeEntryResponse);
    rpc UpdateTimeEntry(UpdateTimeEntryRequest) returns (UpdateTimeEntryResponse);
    rpc GetTimeEntries(GetTimeEntriesRequest) returns (GetTimeEntriesResponse);

    // Approvals
    rpc ApproveTimeCard(ApproveTimeCardRequest) returns (ApproveTimeCardResponse);
    rpc RejectTimeCard(RejectTimeCardRequest) returns (RejectTimeCardResponse);
    rpc GetPendingApprovals(GetPendingApprovalsRequest) returns (GetPendingApprovalsResponse);

    // Overtime
    rpc CalculateOvertime(CalculateOvertimeRequest) returns (CalculateOvertimeResponse);
    rpc GetOvertimeSummary(GetOvertimeSummaryRequest) returns (GetOvertimeSummaryResponse);

    // Labor distribution
    rpc DistributeLabor(DistributeLaborRequest) returns (DistributeLaborResponse);
    rpc GetProjectLabor(GetProjectLaborRequest) returns (GetProjectLaborResponse);

    // Device events
    rpc RecordDeviceEvent(RecordDeviceEventRequest) returns (RecordDeviceEventResponse);
    rpc ProcessDeviceEvents(ProcessDeviceEventsRequest) returns (ProcessDeviceEventsResponse);
}

message TimeCard {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string card_number = 4;
    string period_start_date = 5;
    string period_end_date = 6;
    double total_hours = 7;
    double total_regular_hours = 8;
    double total_overtime_hours = 9;
    string status = 10;
}

message TimeEntry {
    string id = 1;
    string time_card_id = 2;
    string entry_date = 3;
    string start_time = 4;
    string end_time = 5;
    int32 duration_minutes = 6;
    string entry_type = 7;
    string project_id = 8;
}

message CreateTimeCardRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string assignment_id = 3;
    string period_start_date = 4;
    string period_end_date = 5;
    string created_by = 6;
}

message CreateTimeCardResponse {
    TimeCard time_card = 1;
}

message GetTimeCardRequest {
    string tenant_id = 1;
    string time_card_id = 2;
}

message GetTimeCardResponse {
    TimeCard time_card = 1;
    repeated TimeEntry entries = 2;
}

message ListTimeCardsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string status = 3;
    string period_start = 4;
    string period_end = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListTimeCardsResponse {
    repeated TimeCard time_cards = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitTimeCardRequest {
    string tenant_id = 1;
    string time_card_id = 2;
    string submitted_by = 3;
}

message SubmitTimeCardResponse {
    TimeCard time_card = 1;
}

message AddTimeEntryRequest {
    string tenant_id = 1;
    string time_card_id = 2;
    string entry_date = 3;
    string start_time = 4;
    string end_time = 5;
    string entry_type = 6;
    string project_id = 7;
    string created_by = 8;
}

message AddTimeEntryResponse {
    TimeEntry entry = 1;
}

message UpdateTimeEntryRequest {
    string tenant_id = 1;
    string entry_id = 2;
    string start_time = 3;
    string end_time = 4;
    string entry_type = 5;
    string updated_by = 6;
    int32 version = 7;
}

message UpdateTimeEntryResponse {
    TimeEntry entry = 1;
}

message GetTimeEntriesRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string entry_date_start = 3;
    string entry_date_end = 4;
    string project_id = 5;
}

message GetTimeEntriesResponse {
    repeated TimeEntry entries = 1;
}

message ApproveTimeCardRequest {
    string tenant_id = 1;
    string time_card_id = 2;
    string approved_by = 3;
}

message ApproveTimeCardResponse {
    TimeCard time_card = 1;
}

message RejectTimeCardRequest {
    string tenant_id = 1;
    string time_card_id = 2;
    string rejected_by = 3;
    string reason = 4;
}

message RejectTimeCardResponse {
    TimeCard time_card = 1;
}

message GetPendingApprovalsRequest {
    string tenant_id = 1;
    string approver_id = 2;
}

message GetPendingApprovalsResponse {
    repeated TimeCard pending_cards = 1;
}

message CalculateOvertimeRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message CalculateOvertimeResponse {
    double regular_hours = 1;
    double overtime_hours = 2;
    double double_time_hours = 3;
    repeated string exception_ids = 4;
}

message GetOvertimeSummaryRequest {
    string tenant_id = 1;
    string department_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetOvertimeSummaryResponse {
    double total_regular_hours = 1;
    double total_overtime_hours = 2;
    double total_double_time_hours = 3;
    int32 employee_count = 4;
}

message LaborDistributionEntry {
    string id = 1;
    string project_id = 2;
    string cost_center = 3;
    double percentage = 4;
    double hours_split = 5;
}

message DistributeLaborRequest {
    string tenant_id = 1;
    string time_entry_id = 2;
    repeated LaborDistributionEntry distributions = 3;
    string created_by = 4;
}

message DistributeLaborResponse {
    repeated LaborDistributionEntry distributions = 1;
}

message GetProjectLaborRequest {
    string tenant_id = 1;
    string project_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetProjectLaborResponse {
    double total_hours = 1;
    int32 employee_count = 2;
    repeated LaborDistributionEntry entries = 3;
}

message RecordDeviceEventRequest {
    string tenant_id = 1;
    string device_id = 2;
    string employee_id = 3;
    string event_type = 4;
    string event_timestamp = 5;
    string location_code = 6;
}

message RecordDeviceEventResponse {
    string event_id = 1;
    bool processed = 2;
}

message ProcessDeviceEventsRequest {
    string tenant_id = 1;
    repeated string event_ids = 2;
}

message ProcessDeviceEventsResponse {
    int32 processed_count = 1;
    int32 error_count = 2;
    repeated string time_entry_ids = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, work schedules | Determine eligible employees and schedules |
| `absence-service` | Absence plans, balances | Validate absence entries against balances |
| `project-management` | Projects, tasks | Labor distribution to project tasks |
| `workflow-service` | Approval workflows | Route time card approvals |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `payroll-service` | Approved time entries, hours | Calculate earnings and overtime |
| `project-management` | Labor hours and costs | Project costing and billing |
| `reporting-service` | Time and labor analytics | Workforce utilization reports |
| `compensation-service` | Overtime hours | Calculate overtime premiums |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `TimeCardSubmitted` | `timelabor.card.submitted` | `{ tenant_id, time_card_id, employee_id, period_start, period_end, total_hours }` | Published when a time card is submitted |
| `TimeCardApproved` | `timelabor.card.approved` | `{ tenant_id, time_card_id, employee_id, approved_by, total_regular_hours, total_overtime_hours }` | Published when a time card is approved |
| `TimeCardRejected` | `timelabor.card.rejected` | `{ tenant_id, time_card_id, employee_id, rejected_by, reason }` | Published when a time card is rejected |
| `TimeEntryException` | `timelabor.entry.exception` | `{ tenant_id, entry_id, employee_id, exception_type, entry_date }` | Published when a time entry exception is detected |
| `OvertimeThresholdExceeded` | `timelabor.overtime.exceeded` | `{ tenant_id, employee_id, entry_date, threshold_type, hours_over_threshold }` | Published when overtime threshold is crossed |
| `DeviceEventRecorded` | `timelabor.device.recorded` | `{ tenant_id, event_id, device_id, employee_id, event_type, timestamp }` | Published when a device event is captured |
| `LaborDistributed` | `timelabor.labor.distributed` | `{ tenant_id, time_entry_id, distributions }` | Published when labor is distributed to cost centers |
