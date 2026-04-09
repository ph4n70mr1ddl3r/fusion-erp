# 99 - Manufacturing Execution System (MES) / Smart Operations Service Specification

## 1. Domain Overview

Manufacturing Execution System (MES) / Smart Operations provides real-time shop floor execution management covering production scheduling and dispatch, work center and machine management, real-time machine parameter monitoring via IIoT/SCADA integration, production data collection from operators and automated sources, OEE (Overall Equipment Effectiveness) tracking with availability, performance, and quality decomposition, downtime event management with root cause analysis, shift definition and scheduling, and production alert management with severity-based escalation. The system bridges the gap between ERP-level planning and shop-floor execution, enabling smart manufacturing with Industry 4.0 capabilities. Integrates with MFG for work order release, INV for material consumption, COST for production costing, and QUAL for quality integration.

**Bounded Context:** Shop Floor Execution & Smart Manufacturing Operations
**Service Name:** `mes-service`
**Database:** `data/mes.db`
**HTTP Port:** 8136 | **gRPC Port:** 9136

---

## 2. Database Schema

### 2.1 Work Centers
```sql
CREATE TABLE work_centers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_center_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    work_center_type TEXT NOT NULL
        CHECK(work_center_type IN ('MACHINE','LABOR','LINE','CELL','STATION')),
    plant_id TEXT NOT NULL,
    department_id TEXT,
    capacity_units_per_hour REAL NOT NULL DEFAULT 0.0,
    available_hours_per_day REAL NOT NULL DEFAULT 8.0,
    cost_per_hour_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','MAINTENANCE')),
    equipment_ids TEXT,                             -- JSON: list of equipment identifiers
    labor_skill_requirements TEXT,                  -- JSON: required skill types and levels

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, work_center_code)
);

CREATE INDEX idx_work_centers_tenant_plant ON work_centers(tenant_id, plant_id);
CREATE INDEX idx_work_centers_tenant_type ON work_centers(tenant_id, work_center_type);
CREATE INDEX idx_work_centers_tenant_status ON work_centers(tenant_id, status);
CREATE INDEX idx_work_centers_tenant_active ON work_centers(tenant_id, is_active);
```

### 2.2 Production Schedules
```sql
CREATE TABLE production_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    work_center_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    scheduled_quantity INTEGER NOT NULL,
    completed_quantity INTEGER NOT NULL DEFAULT 0,
    scrapped_quantity INTEGER NOT NULL DEFAULT 0,
    scheduled_start TEXT NOT NULL,
    scheduled_end TEXT NOT NULL,
    actual_start TEXT,
    actual_end TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','RELEASED','STARTED','COMPLETED','CLOSED')),
    priority INTEGER NOT NULL DEFAULT 0,
    shift_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_center_id) REFERENCES work_centers(id)
);

CREATE INDEX idx_prod_schedules_tenant_wo ON production_schedules(tenant_id, work_order_id);
CREATE INDEX idx_prod_schedules_tenant_wc ON production_schedules(tenant_id, work_center_id);
CREATE INDEX idx_prod_schedules_tenant_status ON production_schedules(tenant_id, status);
CREATE INDEX idx_prod_schedules_tenant_item ON production_schedules(tenant_id, item_id);
CREATE INDEX idx_prod_schedules_tenant_dates ON production_schedules(tenant_id, scheduled_start, scheduled_end);
CREATE INDEX idx_prod_schedules_tenant_priority ON production_schedules(tenant_id, priority);
CREATE INDEX idx_prod_schedules_tenant_active ON production_schedules(tenant_id, is_active);
```

### 2.3 Machine Parameters
```sql
CREATE TABLE machine_parameters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_center_id TEXT NOT NULL,
    parameter_name TEXT NOT NULL,
    parameter_type TEXT NOT NULL
        CHECK(parameter_type IN ('TEMPERATURE','PRESSURE','SPEED','VIBRATION','CYCLE_TIME','POWER')),
    unit_of_measure TEXT NOT NULL,
    target_value REAL NOT NULL DEFAULT 0.0,
    min_threshold REAL NOT NULL DEFAULT 0.0,
    max_threshold REAL NOT NULL DEFAULT 0.0,
    current_value REAL NOT NULL DEFAULT 0.0,
    last_reading_at TEXT,
    source TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(source IN ('MANUAL','IOT_SENSOR','SCADA')),
    is_critical INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_center_id) REFERENCES work_centers(id)
);

CREATE INDEX idx_machine_params_tenant_wc ON machine_parameters(tenant_id, work_center_id);
CREATE INDEX idx_machine_params_tenant_type ON machine_parameters(tenant_id, parameter_type);
CREATE INDEX idx_machine_params_tenant_source ON machine_parameters(tenant_id, source);
CREATE INDEX idx_machine_params_tenant_critical ON machine_parameters(tenant_id, is_critical);
CREATE INDEX idx_machine_params_tenant_active ON machine_parameters(tenant_id, is_active);
```

### 2.4 Production Data Collections
```sql
CREATE TABLE production_data_collections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_id TEXT NOT NULL,
    work_center_id TEXT NOT NULL,
    collection_timestamp TEXT NOT NULL,
    operator_id TEXT NOT NULL,
    data_type TEXT NOT NULL
        CHECK(data_type IN ('COUNTER','MEASUREMENT','DEFECT','EVENT')),
    value REAL NOT NULL DEFAULT 0.0,
    unit_of_measure TEXT,
    good_quantity INTEGER NOT NULL DEFAULT 0,
    reject_quantity INTEGER NOT NULL DEFAULT 0,
    reject_reason TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (schedule_id) REFERENCES production_schedules(id),
    FOREIGN KEY (work_center_id) REFERENCES work_centers(id)
);

CREATE INDEX idx_data_coll_tenant_schedule ON production_data_collections(tenant_id, schedule_id);
CREATE INDEX idx_data_coll_tenant_wc ON production_data_collections(tenant_id, work_center_id);
CREATE INDEX idx_data_coll_tenant_type ON production_data_collections(tenant_id, data_type);
CREATE INDEX idx_data_coll_tenant_timestamp ON production_data_collections(tenant_id, collection_timestamp);
CREATE INDEX idx_data_coll_tenant_operator ON production_data_collections(tenant_id, operator_id);
```

### 2.5 OEE Records
```sql
CREATE TABLE oee_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_center_id TEXT NOT NULL,
    shift_date TEXT NOT NULL,
    shift_id TEXT,
    availability_pct REAL NOT NULL DEFAULT 0.0,
    performance_pct REAL NOT NULL DEFAULT 0.0,
    quality_pct REAL NOT NULL DEFAULT 0.0,
    oee_pct REAL NOT NULL DEFAULT 0.0,
    ideal_cycle_time_seconds REAL NOT NULL DEFAULT 0.0,
    actual_cycle_time_seconds REAL NOT NULL DEFAULT 0.0,
    planned_production_minutes INTEGER NOT NULL DEFAULT 0,
    unplanned_downtime_minutes INTEGER NOT NULL DEFAULT 0,
    run_time_minutes INTEGER NOT NULL DEFAULT 0,
    total_pieces INTEGER NOT NULL DEFAULT 0,
    good_pieces INTEGER NOT NULL DEFAULT 0,
    scrap_pieces INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_center_id) REFERENCES work_centers(id),
    UNIQUE(tenant_id, work_center_id, shift_date, shift_id)
);

CREATE INDEX idx_oee_tenant_wc ON oee_records(tenant_id, work_center_id);
CREATE INDEX idx_oee_tenant_date ON oee_records(tenant_id, shift_date);
CREATE INDEX idx_oee_tenant_pct ON oee_records(tenant_id, oee_pct);
CREATE INDEX idx_oee_tenant_shift ON oee_records(tenant_id, shift_id);
```

### 2.6 Downtime Events
```sql
CREATE TABLE downtime_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_center_id TEXT NOT NULL,
    schedule_id TEXT,
    downtime_type TEXT NOT NULL
        CHECK(downtime_type IN ('PLANNED','UNPLANNED')),
    reason_category TEXT NOT NULL
        CHECK(reason_category IN ('MAINTENANCE','CHANGEOVER','MATERIAL_SHORTAGE','QUALITY_HOLD','OPERATOR_ABSENCE','POWER_OUTAGE','OTHER')),
    reason_code TEXT,
    start_time TEXT NOT NULL,
    end_time TEXT,
    duration_minutes INTEGER NOT NULL DEFAULT 0,
    operator_id TEXT,
    resolution_notes TEXT,
    root_cause TEXT,
    corrective_action TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_center_id) REFERENCES work_centers(id)
);

CREATE INDEX idx_downtime_tenant_wc ON downtime_events(tenant_id, work_center_id);
CREATE INDEX idx_downtime_tenant_type ON downtime_events(tenant_id, downtime_type);
CREATE INDEX idx_downtime_tenant_reason ON downtime_events(tenant_id, reason_category);
CREATE INDEX idx_downtime_tenant_times ON downtime_events(tenant_id, start_time, end_time);
CREATE INDEX idx_downtime_tenant_schedule ON downtime_events(tenant_id, schedule_id);
```

### 2.7 Shift Definitions
```sql
CREATE TABLE shift_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shift_name TEXT NOT NULL,
    start_time TEXT NOT NULL,
    end_time TEXT NOT NULL,
    break_duration_minutes INTEGER NOT NULL DEFAULT 0,
    applicable_days TEXT NOT NULL,                  -- JSON: array of days ["MON","TUE",...]
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, shift_name)
);

CREATE INDEX idx_shifts_tenant_active ON shift_definitions(tenant_id, is_active);
```

### 2.8 Production Alerts
```sql
CREATE TABLE production_alerts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_center_id TEXT NOT NULL,
    alert_type TEXT NOT NULL
        CHECK(alert_type IN ('MACHINE_DOWN','QUALITY_THRESHOLD','PARAMETER_DEVIATION','SCHEDULE_DELAY','SAFETY')),
    severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(severity IN ('INFO','WARNING','CRITICAL','EMERGENCY')),
    message TEXT NOT NULL,
    trigger_value REAL,
    threshold_value REAL,
    acknowledged_by TEXT,
    acknowledged_at TEXT,
    resolved_at TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ACKNOWLEDGED','RESOLVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_center_id) REFERENCES work_centers(id)
);

CREATE INDEX idx_alerts_tenant_wc ON production_alerts(tenant_id, work_center_id);
CREATE INDEX idx_alerts_tenant_type ON production_alerts(tenant_id, alert_type);
CREATE INDEX idx_alerts_tenant_severity ON production_alerts(tenant_id, severity);
CREATE INDEX idx_alerts_tenant_status ON production_alerts(tenant_id, status);
CREATE INDEX idx_alerts_tenant_dates ON production_alerts(tenant_id, created_at);
```

---

## 3. REST API Endpoints

### 3.1 Work Centers
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mes/work-centers` | List work centers |
| POST | `/api/v1/mes/work-centers` | Create work center |
| GET | `/api/v1/mes/work-centers/{id}` | Get work center details |
| PUT | `/api/v1/mes/work-centers/{id}` | Update work center |
| PATCH | `/api/v1/mes/work-centers/{id}/status` | Update work center status |

### 3.2 Production Schedules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mes/schedules` | List production schedules |
| POST | `/api/v1/mes/schedules` | Create production schedule |
| GET | `/api/v1/mes/schedules/{id}` | Get schedule details |
| PUT | `/api/v1/mes/schedules/{id}` | Update schedule |
| POST | `/api/v1/mes/schedules/{id}/release` | Release schedule to shop floor |
| POST | `/api/v1/mes/schedules/{id}/start` | Start production |
| POST | `/api/v1/mes/schedules/{id}/complete` | Complete production |
| POST | `/api/v1/mes/schedules/{id}/close` | Close schedule |

### 3.3 Machine Parameters
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mes/work-centers/{id}/parameters` | Get parameter readings for work center |
| POST | `/api/v1/mes/parameters/record` | Record parameter reading |
| GET | `/api/v1/mes/parameters/alerts` | Get parameter deviation alerts |
| GET | `/api/v1/mes/parameters/{id}/history` | Get parameter history |

### 3.4 Data Collection
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/mes/data-collection/record` | Record production data |
| GET | `/api/v1/mes/schedules/{id}/data` | Get data collection for schedule |
| GET | `/api/v1/mes/data-collection/summary` | Get production data summary |

### 3.5 OEE
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mes/oee/dashboard` | Get OEE dashboard |
| GET | `/api/v1/mes/oee/trends` | Get OEE trend data |
| GET | `/api/v1/mes/oee/by-work-center` | Get OEE by work center |
| GET | `/api/v1/mes/oee/by-shift` | Get OEE by shift |

### 3.6 Downtime
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/mes/downtime/log` | Log downtime event |
| GET | `/api/v1/mes/downtime/analysis` | Get downtime analysis |
| GET | `/api/v1/mes/downtime/reasons` | Get downtime reason breakdown |
| POST | `/api/v1/mes/downtime/{id}/resolve` | Resolve downtime event |

### 3.7 Shifts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mes/shifts` | List shift definitions |
| POST | `/api/v1/mes/shifts` | Create shift definition |
| GET | `/api/v1/mes/shifts/{id}` | Get shift details |
| PUT | `/api/v1/mes/shifts/{id}` | Update shift |

### 3.8 Alerts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mes/alerts/active` | Get active alerts |
| POST | `/api/v1/mes/alerts/{id}/acknowledge` | Acknowledge an alert |
| POST | `/api/v1/mes/alerts/{id}/resolve` | Resolve an alert |
| GET | `/api/v1/mes/alerts/history` | Get alert history |

---

## 4. Business Rules

### 4.1 Work Center Management
1. Each work center MUST have a unique work_center_code within a tenant.
2. A work center in INACTIVE or MAINTENANCE status MUST NOT be assigned new production schedules.
3. Work center capacity (capacity_units_per_hour) MUST be a positive value when the status is ACTIVE.
4. Equipment IDs and labor skill requirements stored as JSON MUST be validated against the master data on creation and update.

### 4.2 Production Scheduling
5. Production schedule status transitions MUST follow: PLANNED -> RELEASED -> STARTED -> COMPLETED -> CLOSED.
6. A schedule MUST NOT be started (status STARTED) until it has been released (status RELEASED).
7. completed_quantity plus scrapped_quantity MUST NOT exceed scheduled_quantity.
8. When a schedule is started, the actual_start timestamp MUST be recorded; upon completion, actual_end MUST be recorded.
9. Schedules with higher priority values SHOULD be dispatched before lower priority schedules within the same work center.

### 4.3 Machine Parameters and Monitoring
10. A parameter reading outside the min_threshold/max_threshold range MUST generate a PARAMETER_DEVIATION alert.
11. Parameters marked as is_critical = 1 that exceed thresholds MUST generate an EMERGENCY severity alert.
12. Readings from IOT_SENSOR or SCADA sources MUST include a timestamp and SHOULD be processed within 5 seconds of receipt.
13. The current_value on machine_parameters MUST be updated with every new reading; last_reading_at MUST reflect the most recent reading time.

### 4.4 OEE Calculation
14. OEE MUST be calculated as: Availability * Performance * Quality, where:
    - Availability = Run Time / Planned Production Time
    - Performance = (Total Pieces * Ideal Cycle Time) / Run Time
    - Quality = Good Pieces / Total Pieces
15. OEE records MUST be computed per work center per shift date and MUST NOT be modified after creation.
16. OEE percentages MUST be stored as values between 0.00 and 100.00.

### 4.5 Downtime Management
17. Downtime events with downtime_type UNPLANNED MUST have a reason_category specified.
18. Downtime duration MUST be calculated as the difference between start_time and end_time in minutes when end_time is provided.
19. Unresolved downtime events (end_time is NULL) MUST be included in current availability calculations as ongoing downtime.
20. Downtime events categorized as MAINTENANCE or POWER_OUTAGE exceeding 60 minutes SHOULD trigger a CRITICAL alert.

### 4.6 Shift Scheduling
21. Shift definitions MUST NOT have overlapping time ranges for the same applicable_days within a tenant.
22. applicable_days MUST contain at least one valid day abbreviation.

### 4.7 Alert Management
23. Alerts in ACTIVE status MUST persist until explicitly acknowledged or resolved.
24. EMERGENCY severity alerts MUST NOT be resolved without both acknowledged_by and resolved_at being populated.
25. Production alerts SHOULD trigger notifications to the assigned shift supervisor and plant manager based on severity level.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.mes.v1;

service ManufacturingExecutionService {
    // Work center management
    rpc CreateWorkCenter(CreateWorkCenterRequest) returns (CreateWorkCenterResponse);
    rpc GetWorkCenter(GetWorkCenterRequest) returns (GetWorkCenterResponse);
    rpc ListWorkCenters(ListWorkCentersRequest) returns (ListWorkCentersResponse);

    // Production scheduling
    rpc CreateSchedule(CreateScheduleRequest) returns (CreateScheduleResponse);
    rpc GetSchedule(GetScheduleRequest) returns (GetScheduleResponse);
    rpc ListSchedules(ListSchedulesRequest) returns (ListSchedulesResponse);
    rpc ReleaseSchedule(ReleaseScheduleRequest) returns (ReleaseScheduleResponse);
    rpc StartProduction(StartProductionRequest) returns (StartProductionResponse);
    rpc CompleteProduction(CompleteProductionRequest) returns (CompleteProductionResponse);

    // Machine parameters
    rpc RecordParameter(RecordParameterRequest) returns (RecordParameterResponse);
    rpc GetParameterReadings(GetParameterReadingsRequest) returns (GetParameterReadingsResponse);
    rpc StreamParameters(StreamParametersRequest) returns (stream ParameterReading);

    // Data collection
    rpc RecordProductionData(RecordProductionDataRequest) returns (RecordProductionDataResponse);
    rpc GetDataSummary(GetDataSummaryRequest) returns (GetDataSummaryResponse);

    // OEE
    rpc GetOEEDashboard(GetOEEDashboardRequest) returns (GetOEEDashboardResponse);
    rpc GetOEETrends(GetOEETrendsRequest) returns (GetOEETrendsResponse);

    // Downtime
    rpc LogDowntime(LogDowntimeRequest) returns (LogDowntimeResponse);
    rpc GetDowntimeAnalysis(GetDowntimeAnalysisRequest) returns (GetDowntimeAnalysisResponse);

    // Alerts
    rpc GetActiveAlerts(GetActiveAlertsRequest) returns (GetActiveAlertsResponse);
    rpc AcknowledgeAlert(AcknowledgeAlertRequest) returns (AcknowledgeAlertResponse);
    rpc ResolveAlert(ResolveAlertRequest) returns (ResolveAlertResponse);
}

message CreateWorkCenterRequest {
    string tenant_id = 1;
    string work_center_code = 2;
    string name = 3;
    string description = 4;
    string work_center_type = 5;
    string plant_id = 6;
    string department_id = 7;
    double capacity_units_per_hour = 8;
    double available_hours_per_day = 9;
    int64 cost_per_hour_cents = 10;
    string equipment_ids = 11;
    string labor_skill_requirements = 12;
}

message CreateWorkCenterResponse {
    string work_center_id = 1;
    string work_center_code = 2;
    string status = 3;
}

message GetWorkCenterRequest {
    string tenant_id = 1;
    string work_center_id = 2;
}

message GetWorkCenterResponse {
    string work_center_id = 1;
    string work_center_code = 2;
    string name = 3;
    string work_center_type = 4;
    string plant_id = 5;
    string status = 6;
    double capacity_units_per_hour = 7;
    int64 cost_per_hour_cents = 8;
    string equipment_ids = 9;
}

message ListWorkCentersRequest {
    string tenant_id = 1;
    string plant_id = 2;
    string work_center_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListWorkCentersResponse {
    repeated GetWorkCenterResponse work_centers = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CreateScheduleRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string work_center_id = 3;
    string item_id = 4;
    int32 scheduled_quantity = 5;
    string scheduled_start = 6;
    string scheduled_end = 7;
    int32 priority = 8;
    string shift_id = 9;
}

message CreateScheduleResponse {
    string schedule_id = 1;
    string work_center_id = 2;
    string status = 3;
}

message GetScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
}

message GetScheduleResponse {
    string schedule_id = 1;
    string work_order_id = 2;
    string work_center_id = 3;
    string item_id = 4;
    int32 scheduled_quantity = 5;
    int32 completed_quantity = 6;
    int32 scrapped_quantity = 7;
    string scheduled_start = 8;
    string scheduled_end = 9;
    string actual_start = 10;
    string actual_end = 11;
    string status = 12;
    int32 priority = 13;
}

message ListSchedulesRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string status = 3;
    string scheduled_start = 4;
    string scheduled_end = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListSchedulesResponse {
    repeated GetScheduleResponse schedules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReleaseScheduleRequest {
    string tenant_id = 1;
    string schedule_id = 2;
}

message ReleaseScheduleResponse {
    string schedule_id = 1;
    string status = 2;
}

message StartProductionRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    string operator_id = 3;
}

message StartProductionResponse {
    string schedule_id = 1;
    string status = 2;
    string actual_start = 3;
}

message CompleteProductionRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    int32 completed_quantity = 3;
    int32 scrapped_quantity = 4;
}

message CompleteProductionResponse {
    string schedule_id = 1;
    string status = 2;
    string actual_end = 3;
}

message RecordParameterRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string parameter_name = 3;
    double value = 4;
    string source = 5;
    string reading_timestamp = 6;
}

message RecordParameterResponse {
    string parameter_id = 1;
    double value = 2;
    bool threshold_exceeded = 3;
    string alert_id = 4;
}

message GetParameterReadingsRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string parameter_type = 3;
    string from_timestamp = 4;
    string to_timestamp = 5;
}

message GetParameterReadingsResponse {
    repeated ParameterReading readings = 1;
}

message ParameterReading {
    string parameter_id = 1;
    string parameter_name = 2;
    string parameter_type = 3;
    double value = 4;
    double target_value = 5;
    double min_threshold = 6;
    double max_threshold = 7;
    string unit_of_measure = 8;
    string reading_timestamp = 9;
}

message StreamParametersRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    repeated string parameter_names = 3;
}

message RecordProductionDataRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    string work_center_id = 3;
    string operator_id = 4;
    string collection_timestamp = 5;
    string data_type = 6;
    double value = 7;
    string unit_of_measure = 8;
    int32 good_quantity = 9;
    int32 reject_quantity = 10;
    string reject_reason = 11;
    string notes = 12;
}

message RecordProductionDataResponse {
    string data_id = 1;
    string data_type = 2;
}

message GetDataSummaryRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    string work_center_id = 3;
    string from_date = 4;
    string to_date = 5;
}

message GetDataSummaryResponse {
    int32 total_good_quantity = 1;
    int32 total_reject_quantity = 2;
    int32 total_data_points = 3;
    double avg_value = 4;
}

message GetOEEDashboardRequest {
    string tenant_id = 1;
    string plant_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetOEEDashboardResponse {
    double overall_oee = 1;
    double overall_availability = 2;
    double overall_performance = 3;
    double overall_quality = 4;
    repeated WorkCenterOEE work_centers = 5;
}

message WorkCenterOEE {
    string work_center_id = 1;
    string work_center_name = 2;
    double oee_pct = 3;
    double availability_pct = 4;
    double performance_pct = 5;
    double quality_pct = 6;
}

message GetOEETrendsRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetOEETrendsResponse {
    repeated OEEDataPoint data_points = 1;
}

message OEEDataPoint {
    string date = 1;
    double oee_pct = 2;
    double availability_pct = 3;
    double performance_pct = 4;
    double quality_pct = 5;
}

message LogDowntimeRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string schedule_id = 3;
    string downtime_type = 4;
    string reason_category = 5;
    string reason_code = 6;
    string start_time = 7;
    string end_time = 8;
    string operator_id = 9;
    string resolution_notes = 10;
}

message LogDowntimeResponse {
    string downtime_id = 1;
    int32 duration_minutes = 2;
}

message GetDowntimeAnalysisRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetDowntimeAnalysisResponse {
    int32 total_events = 1;
    int32 total_downtime_minutes = 2;
    repeated DowntimeByReason breakdown = 3;
}

message DowntimeByReason {
    string reason_category = 1;
    int32 event_count = 2;
    int32 total_minutes = 3;
    double percentage = 4;
}

message GetActiveAlertsRequest {
    string tenant_id = 1;
    string work_center_id = 2;
    string severity = 3;
    int32 page_size = 4;
}

message GetActiveAlertsResponse {
    repeated AlertInfo alerts = 1;
    int32 total_count = 2;
}

message AlertInfo {
    string alert_id = 1;
    string work_center_id = 2;
    string alert_type = 3;
    string severity = 4;
    string message = 5;
    double trigger_value = 6;
    double threshold_value = 7;
    string status = 8;
    string created_at = 9;
}

message AcknowledgeAlertRequest {
    string tenant_id = 1;
    string alert_id = 2;
    string acknowledged_by = 3;
}

message AcknowledgeAlertResponse {
    string alert_id = 1;
    string status = 2;
}

message ResolveAlertRequest {
    string tenant_id = 1;
    string alert_id = 2;
    string resolution_notes = 3;
}

message ResolveAlertResponse {
    string alert_id = 1;
    string status = 2;
    string resolved_at = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `mfg-service` | Work order releases, BOM and routing data, production item specifications |
| `inv-service` | Item master data, material availability, warehouse and location data |
| `costing-service` | Standard cost data, cost element definitions, overhead rates |
| `qual-service` | Quality inspection plans, acceptance criteria, defect classification codes |
| `iot-service` | IIoT sensor readings, SCADA data feeds, equipment health telemetry |
| `auth-service` | User identity for operator assignment, shift supervisor role resolution |

### Published To
| Service | Data |
|---------|------|
| `mfg-service` | Production completion data, work order progress updates, scrap reporting |
| `inv-service` | Material consumption transactions, scrap disposal movements |
| `costing-service` | Actual production costs, labor and machine hours for variance calculation |
| `qual-service` | In-process quality data, defect counts for inspection triggering |
| `reporting-service` | OEE metrics, downtime analytics, production efficiency dashboards |
| `notification-service` | Alert notifications, shift change alerts, parameter deviation warnings |
| `workflow-service` | Escalation routing for unresolved alerts, maintenance request workflows |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `mes.production.started` | `mes.events` | `{ schedule_id, work_order_id, work_center_id, item_id, actual_start }` | Production run started on shop floor |
| `mes.production.completed` | `mes.events` | `{ schedule_id, work_order_id, work_center_id, completed_quantity, scrapped_quantity, actual_end }` | Production run completed |
| `mes.parameter.deviation` | `mes.events` | `{ parameter_id, work_center_id, parameter_name, trigger_value, threshold_value, severity }` | Machine parameter deviated from threshold |
| `mes.downtime.started` | `mes.events` | `{ downtime_id, work_center_id, downtime_type, reason_category, start_time }` | Downtime event recorded |
| `mes.oee.calculated` | `mes.events` | `{ work_center_id, shift_date, oee_pct, availability_pct, performance_pct, quality_pct }` | OEE record calculated for shift |
| `mes.alert.triggered` | `mes.events` | `{ alert_id, work_center_id, alert_type, severity, message }` | Production alert triggered |

---

## 8. Migrations

1. V001: `shift_definitions`
2. V002: `work_centers`
3. V003: `production_schedules`
4. V004: `machine_parameters`
5. V005: `production_data_collections`
6. V006: `oee_records`
7. V007: `downtime_events`
8. V008: `production_alerts`
9. V009: Triggers for `updated_at`
