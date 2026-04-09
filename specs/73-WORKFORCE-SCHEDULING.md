# 73 - Workforce Scheduling Service Specification

## 1. Domain Overview
**Bounded Context:** Workforce Scheduling - Manages shift templates, automated schedule generation, shift assignments, swap requests, labor demand forecasting, and schedule compliance tracking for optimal workforce deployment.

**Service Name:** scheduling-service
**Database:** scheduling_db
**HTTP Port:** 8105 | **gRPC Port:** 9105

## 2. Database Schema

```sql
CREATE TABLE shift_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    template_name TEXT NOT NULL,
    template_code TEXT NOT NULL,
    start_time TEXT NOT NULL,
    end_time TEXT NOT NULL,
    break_duration_minutes INTEGER NOT NULL DEFAULT 0,
    grace_period_minutes INTEGER NOT NULL DEFAULT 0,
    color_code TEXT,
    skill_requirements TEXT,
    max_headcount INTEGER,
    min_headcount INTEGER DEFAULT 1,
    applicable_days TEXT,
    department_id TEXT NOT NULL,
    location_id TEXT,
    description TEXT
);

CREATE INDEX idx_shift_templates_tenant ON shift_templates(tenant_id);
CREATE INDEX idx_shift_templates_department ON shift_templates(department_id);
CREATE INDEX idx_shift_templates_code ON shift_templates(template_code);

CREATE TABLE work_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    schedule_name TEXT NOT NULL,
    schedule_code TEXT NOT NULL,
    schedule_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    department_id TEXT NOT NULL,
    location_id TEXT,
    generated_algorithm TEXT,
    optimization_score INTEGER,
    total_shifts INTEGER DEFAULT 0,
    total_hours DECIMAL(10,2) DEFAULT 0,
    published_at TEXT,
    published_by TEXT,
    notes TEXT
);

CREATE INDEX idx_work_schedules_tenant ON work_schedules(tenant_id);
CREATE INDEX idx_work_schedules_department ON work_schedules(department_id);
CREATE INDEX idx_work_schedules_status ON work_schedules(status);
CREATE INDEX idx_work_schedules_dates ON work_schedules(start_date, end_date);

CREATE TABLE schedule_shifts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    schedule_id TEXT NOT NULL,
    template_id TEXT,
    employee_id TEXT NOT NULL,
    shift_date TEXT NOT NULL,
    start_time TEXT NOT NULL,
    end_time TEXT NOT NULL,
    break_start TEXT,
    break_end TEXT,
    role_id TEXT,
    position_id TEXT,
    location_id TEXT,
    status TEXT NOT NULL DEFAULT 'SCHEDULED',
    assigned_by TEXT,
    assignment_method TEXT,
    swap_eligible INTEGER NOT NULL DEFAULT 0,
    notes TEXT,
    FOREIGN KEY (schedule_id) REFERENCES work_schedules(id)
);

CREATE INDEX idx_schedule_shifts_tenant ON schedule_shifts(tenant_id);
CREATE INDEX idx_schedule_shifts_schedule ON schedule_shifts(schedule_id);
CREATE INDEX idx_schedule_shifts_employee ON schedule_shifts(employee_id);
CREATE INDEX idx_schedule_shifts_date ON schedule_shifts(shift_date);
CREATE INDEX idx_schedule_shifts_status ON schedule_shifts(status);

CREATE TABLE shift_swap_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    shift_id TEXT NOT NULL,
    requesting_employee_id TEXT NOT NULL,
    target_employee_id TEXT,
    target_shift_id TEXT,
    swap_type TEXT NOT NULL,
    reason TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING',
    supervisor_id TEXT,
    approved_by TEXT,
    approved_at TEXT,
    rejected_reason TEXT,
    expires_at TEXT,
    FOREIGN KEY (shift_id) REFERENCES schedule_shifts(id)
);

CREATE INDEX idx_shift_swap_tenant ON shift_swap_requests(tenant_id);
CREATE INDEX idx_shift_swap_requester ON shift_swap_requests(requesting_employee_id);
CREATE INDEX idx_shift_swap_status ON shift_swap_requests(status);

CREATE TABLE schedule_constraints (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    constraint_name TEXT NOT NULL,
    constraint_type TEXT NOT NULL,
    constraint_rule TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 5,
    department_id TEXT,
    employee_id TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    parameters TEXT,
    hard_constraint INTEGER NOT NULL DEFAULT 1,
    description TEXT
);

CREATE INDEX idx_schedule_constraints_tenant ON schedule_constraints(tenant_id);
CREATE INDEX idx_schedule_constraints_type ON schedule_constraints(constraint_type);

CREATE TABLE labor_demand_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    forecast_date TEXT NOT NULL,
    department_id TEXT NOT NULL,
    location_id TEXT,
    role_id TEXT,
    predicted_demand INTEGER NOT NULL,
    confidence_score INTEGER,
    model_type TEXT NOT NULL,
    historical_period_days INTEGER,
    seasonality_factor DECIMAL(10,4) DEFAULT 1.0,
    trend_factor DECIMAL(10,4) DEFAULT 1.0,
    actual_demand INTEGER,
    variance_percentage DECIMAL(10,2),
    metadata TEXT
);

CREATE INDEX idx_labor_demand_tenant ON labor_demand_forecasts(tenant_id);
CREATE INDEX idx_labor_demand_date ON labor_demand_forecasts(forecast_date);
CREATE INDEX idx_labor_demand_department ON labor_demand_forecasts(department_id);

CREATE TABLE schedule_optimization_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL,
    weight INTEGER NOT NULL DEFAULT 50,
    rule_expression TEXT NOT NULL,
    department_id TEXT,
    applies_to_roles TEXT,
    max_continuous_hours INTEGER,
    min_rest_hours INTEGER,
    max_consecutive_days INTEGER,
    max_overtime_hours_weekly INTEGER,
    description TEXT
);

CREATE INDEX idx_opt_rules_tenant ON schedule_optimization_rules(tenant_id);
CREATE INDEX idx_opt_rules_type ON schedule_optimization_rules(rule_type);

CREATE TABLE schedule_compliance_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    schedule_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    compliance_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'COMPLIANT',
    rule_reference TEXT NOT NULL,
    detected_at TEXT NOT NULL,
    resolved_at TEXT,
    violation_details TEXT,
    severity TEXT NOT NULL DEFAULT 'LOW',
    resolution_action TEXT,
    resolved_by TEXT,
    FOREIGN KEY (schedule_id) REFERENCES work_schedules(id)
);

CREATE INDEX idx_compliance_tenant ON schedule_compliance_records(tenant_id);
CREATE INDEX idx_compliance_schedule ON schedule_compliance_records(schedule_id);
CREATE INDEX idx_compliance_status ON schedule_compliance_records(status);

CREATE TABLE schedule_publish_history (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    schedule_id TEXT NOT NULL,
    published_by TEXT NOT NULL,
    published_at TEXT NOT NULL,
    previous_version INTEGER,
    change_summary TEXT,
    notification_sent INTEGER NOT NULL DEFAULT 0,
    notification_count INTEGER DEFAULT 0,
    employee_count INTEGER DEFAULT 0,
    FOREIGN KEY (schedule_id) REFERENCES work_schedules(id)
);

CREATE INDEX idx_publish_history_tenant ON schedule_publish_history(tenant_id);
CREATE INDEX idx_publish_history_schedule ON schedule_publish_history(schedule_id);
```

## 3. REST API Endpoints

### Shift Templates
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/shift-templates` | Create a new shift template |
| GET | `/api/v1/shift-templates` | List shift templates with filters |
| GET | `/api/v1/shift-templates/{id}` | Get shift template by ID |
| PUT | `/api/v1/shift-templates/{id}` | Update shift template |
| DELETE | `/api/v1/shift-templates/{id}` | Deactivate shift template |

### Work Schedules
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/work-schedules` | Create a new work schedule |
| GET | `/api/v1/work-schedules` | List work schedules with filters |
| GET | `/api/v1/work-schedules/{id}` | Get work schedule by ID |
| PUT | `/api/v1/work-schedules/{id}` | Update work schedule metadata |
| POST | `/api/v1/work-schedules/{id}/generate` | Auto-generate shifts for schedule |
| POST | `/api/v1/work-schedules/{id}/optimize` | Optimize schedule using rules engine |
| POST | `/api/v1/work-schedules/{id}/publish` | Publish schedule to employees |
| GET | `/api/v1/work-schedules/{id}/compliance` | Check schedule compliance |

### Schedule Shifts
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/schedule-shifts` | Assign a shift manually |
| GET | `/api/v1/schedule-shifts` | List shifts with filters |
| GET | `/api/v1/schedule-shifts/{id}` | Get shift details |
| PUT | `/api/v1/schedule-shifts/{id}` | Update shift assignment |
| PATCH | `/api/v1/schedule-shifts/{id}/status` | Update shift status |
| GET | `/api/v1/employees/{employeeId}/shifts` | Get shifts for an employee |
| POST | `/api/v1/schedule-shifts/bulk-assign` | Bulk assign shifts |

### Shift Swap Requests
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/shift-swap-requests` | Create a shift swap request |
| GET | `/api/v1/shift-swap-requests` | List swap requests |
| GET | `/api/v1/shift-swap-requests/{id}` | Get swap request details |
| PUT | `/api/v1/shift-swap-requests/{id}` | Update swap request |
| POST | `/api/v1/shift-swap-requests/{id}/approve` | Approve swap request |
| POST | `/api/v1/shift-swap-requests/{id}/reject` | Reject swap request |

### Schedule Constraints
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/schedule-constraints` | Create a scheduling constraint |
| GET | `/api/v1/schedule-constraints` | List constraints |
| PUT | `/api/v1/schedule-constraints/{id}` | Update constraint |
| DELETE | `/api/v1/schedule-constraints/{id}` | Deactivate constraint |

### Labor Demand Forecasts
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/labor-demand-forecasts` | Create a demand forecast |
| GET | `/api/v1/labor-demand-forecasts` | List demand forecasts |
| POST | `/api/v1/labor-demand-forecasts/generate` | Generate forecast using ML model |
| GET | `/api/v1/labor-demand-forecasts/actuals` | Get actual vs predicted comparison |

### Schedule Optimization
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/optimization-rules` | Create optimization rule |
| GET | `/api/v1/optimization-rules` | List optimization rules |
| PUT | `/api/v1/optimization-rules/{id}` | Update optimization rule |
| POST | `/api/v1/optimization-rules/evaluate` | Evaluate schedule against rules |

### Compliance & Publishing
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/compliance-records` | List compliance records |
| GET | `/api/v1/compliance-records/{id}` | Get compliance record detail |
| POST | `/api/v1/compliance-records/{id}/resolve` | Resolve compliance violation |
| GET | `/api/v1/schedules/{id}/publish-history` | Get publish history |
| GET | `/api/v1/scheduling-analytics/dashboard` | Get scheduling analytics dashboard |

## 4. Business Rules

1. A shift template MUST have a unique template_code within a tenant.
2. A work schedule MUST have a start_date that is before end_date.
3. The schedule generation algorithm MUST respect all hard constraints defined in schedule_constraints.
4. An employee MUST NOT be assigned overlapping shifts on the same date.
5. Shift swap requests MUST be approved by the supervisor of the requesting employee.
6. A shift swap request MUST expire after the configured expiration period if not acted upon.
7. Schedule optimization SHOULD maximize employee preference satisfaction while minimizing labor cost.
8. The system MUST enforce minimum rest hours between consecutive shifts for the same employee as defined in optimization rules.
9. The system MUST NOT allow publishing a schedule with unresolved hard constraint violations.
10. Labor demand forecasts SHOULD use at least 90 days of historical data for accurate predictions.
11. The system MUST track and record all schedule publish events for audit purposes.
12. An employee MUST NOT exceed the max_consecutive_days limit defined in optimization rules.
13. The system SHOULD notify affected employees within 5 minutes of schedule publication.
14. Compliance checks MUST run automatically before any schedule publish action.
15. Overtime shifts MUST be flagged and require additional approval if they exceed weekly limits.

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package scheduling;

service SchedulingService {
    // Shift Templates
    rpc CreateShiftTemplate(CreateShiftTemplateRequest) returns (ShiftTemplateResponse);
    rpc GetShiftTemplate(GetByIdRequest) returns (ShiftTemplateResponse);
    rpc ListShiftTemplates(ListShiftTemplatesRequest) returns (ShiftTemplateListResponse);
    rpc UpdateShiftTemplate(UpdateShiftTemplateRequest) returns (ShiftTemplateResponse);

    // Work Schedules
    rpc CreateWorkSchedule(CreateWorkScheduleRequest) returns (WorkScheduleResponse);
    rpc GetWorkSchedule(GetByIdRequest) returns (WorkScheduleResponse);
    rpc ListWorkSchedules(ListWorkSchedulesRequest) returns (WorkScheduleListResponse);
    rpc GenerateSchedule(GenerateScheduleRequest) returns (WorkScheduleResponse);
    rpc OptimizeSchedule(OptimizeScheduleRequest) returns (OptimizeScheduleResponse);
    rpc PublishSchedule(PublishScheduleRequest) returns (PublishScheduleResponse);

    // Schedule Shifts
    rpc AssignShift(AssignShiftRequest) returns (ScheduleShiftResponse);
    rpc BulkAssignShifts(BulkAssignShiftsRequest) returns (BulkAssignShiftsResponse);
    rpc GetEmployeeShifts(GetEmployeeShiftsRequest) returns (ScheduleShiftListResponse);
    rpc UpdateShiftStatus(UpdateShiftStatusRequest) returns (ScheduleShiftResponse);

    // Shift Swaps
    rpc CreateSwapRequest(CreateSwapRequestRequest) returns (SwapRequestResponse);
    rpc ApproveSwapRequest(ApproveSwapRequestRequest) returns (SwapRequestResponse);
    rpc RejectSwapRequest(RejectSwapRequestRequest) returns (SwapRequestResponse);

    // Compliance & Forecasting
    rpc CheckCompliance(CheckComplianceRequest) returns (ComplianceResponse);
    rpc GenerateDemandForecast(GenerateDemandForecastRequest) returns (DemandForecastResponse);
}

message GetByIdRequest {
    string id = 1;
    string tenant_id = 2;
}

message CreateShiftTemplateRequest {
    string tenant_id = 1;
    string template_name = 2;
    string template_code = 3;
    string start_time = 4;
    string end_time = 5;
    int32 break_duration_minutes = 6;
    int32 grace_period_minutes = 7;
    string color_code = 8;
    string skill_requirements = 9;
    int32 max_headcount = 10;
    int32 min_headcount = 11;
    string applicable_days = 12;
    string department_id = 13;
    string location_id = 14;
    string description = 15;
    string created_by = 16;
}

message ShiftTemplateResponse {
    string id = 1;
    string template_name = 2;
    string template_code = 3;
    string start_time = 4;
    string end_time = 5;
    int32 break_duration_minutes = 6;
    string department_id = 7;
    string created_at = 8;
    int32 version = 9;
}

message ListShiftTemplatesRequest {
    string tenant_id = 1;
    string department_id = 2;
    int32 page = 3;
    int32 page_size = 4;
}

message ShiftTemplateListResponse {
    repeated ShiftTemplateResponse items = 1;
    int32 total_count = 2;
}

message UpdateShiftTemplateRequest {
    string id = 1;
    string tenant_id = 2;
    string template_name = 3;
    string start_time = 4;
    string end_time = 5;
    string updated_by = 6;
    int32 version = 7;
}

message CreateWorkScheduleRequest {
    string tenant_id = 1;
    string schedule_name = 2;
    string schedule_code = 3;
    string schedule_type = 4;
    string start_date = 5;
    string end_date = 6;
    string department_id = 7;
    string location_id = 8;
    string created_by = 9;
}

message WorkScheduleResponse {
    string id = 1;
    string schedule_name = 2;
    string schedule_code = 3;
    string status = 4;
    string start_date = 5;
    string end_date = 6;
    int32 total_shifts = 7;
    int32 optimization_score = 8;
    string created_at = 9;
    int32 version = 10;
}

message ListWorkSchedulesRequest {
    string tenant_id = 1;
    string department_id = 2;
    string status = 3;
    int32 page = 4;
    int32 page_size = 5;
}

message WorkScheduleListResponse {
    repeated WorkScheduleResponse items = 1;
    int32 total_count = 2;
}

message GenerateScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    string algorithm = 3;
    string generated_by = 4;
}

message OptimizeScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    repeated string rule_ids = 3;
    string optimized_by = 4;
}

message OptimizeScheduleResponse {
    string schedule_id = 1;
    int32 optimization_score = 2;
    int32 violations_resolved = 3;
    repeated string changes_made = 4;
}

message PublishScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    string published_by = 3;
}

message PublishScheduleResponse {
    string schedule_id = 1;
    string published_at = 2;
    int32 employee_count = 3;
    int32 notification_sent = 4;
}

message AssignShiftRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    string employee_id = 3;
    string shift_date = 4;
    string start_time = 5;
    string end_time = 6;
    string template_id = 7;
    string role_id = 8;
    string assigned_by = 9;
}

message BulkAssignShiftsRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    repeated AssignShiftRequest shifts = 3;
}

message BulkAssignShiftsResponse {
    int32 assigned_count = 1;
    int32 failed_count = 2;
    repeated string errors = 3;
}

message ScheduleShiftResponse {
    string id = 1;
    string schedule_id = 2;
    string employee_id = 3;
    string shift_date = 4;
    string start_time = 5;
    string end_time = 6;
    string status = 7;
    int32 version = 8;
}

message GetEmployeeShiftsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string start_date = 3;
    string end_date = 4;
}

message ScheduleShiftListResponse {
    repeated ScheduleShiftResponse items = 1;
    int32 total_count = 2;
}

message UpdateShiftStatusRequest {
    string tenant_id = 1;
    string shift_id = 2;
    string status = 3;
    string updated_by = 4;
    int32 version = 5;
}

message CreateSwapRequestRequest {
    string tenant_id = 1;
    string shift_id = 2;
    string requesting_employee_id = 3;
    string target_employee_id = 4;
    string target_shift_id = 5;
    string swap_type = 6;
    string reason = 7;
    string created_by = 8;
}

message SwapRequestResponse {
    string id = 1;
    string shift_id = 2;
    string requesting_employee_id = 3;
    string status = 4;
    string created_at = 5;
}

message ApproveSwapRequestRequest {
    string tenant_id = 1;
    string swap_request_id = 2;
    string approved_by = 3;
}

message RejectSwapRequestRequest {
    string tenant_id = 1;
    string swap_request_id = 2;
    string rejected_reason = 3;
    string rejected_by = 4;
}

message CheckComplianceRequest {
    string tenant_id = 1;
    string schedule_id = 2;
}

message ComplianceResponse {
    string schedule_id = 1;
    bool is_compliant = 2;
    int32 total_violations = 3;
    repeated ComplianceViolation violations = 4;
}

message ComplianceViolation {
    string employee_id = 1;
    string compliance_type = 2;
    string severity = 3;
    string details = 4;
}

message GenerateDemandForecastRequest {
    string tenant_id = 1;
    string department_id = 2;
    string start_date = 3;
    string end_date = 4;
    string model_type = 5;
    int32 historical_period_days = 6;
    string generated_by = 7;
}

message DemandForecastResponse {
    string department_id = 1;
    repeated DemandForecastEntry entries = 2;
    int32 confidence_score = 3;
}

message DemandForecastEntry {
    string forecast_date = 1;
    int32 predicted_demand = 2;
    int32 confidence_score = 3;
}
```

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data/Events | Purpose |
|---------------|-------------|---------|
| CORE-HR | Employee profiles, departments, positions, skills | Employee availability, role matching, department structure |
| TIME-LABOR | Time entries, attendance records, punches | Actual hours worked vs scheduled, overtime calculation |
| ABSENCE | Leave requests, absence records, PTO balances | Employee unavailability, leave conflict detection |

### Published To
| Target Service | Data/Events | Purpose |
|---------------|-------------|---------|
| TIME-LABOR | Schedule shifts, published schedules | Expected attendance, timecard validation |
| CORE-HR | Schedule assignments, compliance records | Employee work hour tracking, violations |
| NOTIFICATION | Schedule published alerts, shift change alerts | Employee notifications |

## 7. Events

### Produced Events
| Event | Payload | Description |
|-------|---------|-------------|
| `SchedulePublished` | `{ schedule_id, tenant_id, department_id, published_by, published_at, employee_count }` | Emitted when a schedule is published to employees |
| `ShiftAssigned` | `{ shift_id, schedule_id, employee_id, shift_date, start_time, end_time, tenant_id }` | Emitted when a shift is assigned to an employee |
| `ShiftSwapRequested` | `{ swap_request_id, shift_id, requesting_employee_id, target_employee_id, tenant_id }` | Emitted when an employee requests a shift swap |
| `ComplianceViolation` | `{ compliance_id, schedule_id, employee_id, compliance_type, severity, tenant_id }` | Emitted when a scheduling compliance violation is detected |
| `ShiftSwapApproved` | `{ swap_request_id, original_shift_id, new_shift_id, tenant_id }` | Emitted when a shift swap is approved |
| `DemandForecastGenerated` | `{ department_id, start_date, end_date, confidence_score, tenant_id }` | Emitted when a labor demand forecast is generated |
| `ScheduleOptimized` | `{ schedule_id, optimization_score, violations_resolved, tenant_id }` | Emitted when schedule optimization completes |
