# 40 - Enterprise Asset Management / Maintenance Service Specification

## 1. Domain Overview

Enterprise Asset Management provides operational asset lifecycle management, preventive and corrective maintenance planning, work order execution, meter/sensor reading collection, failure analysis, service contract management, safety permits and lockout/tagout procedures, and asset decommission tracking. Integrates with FA for financial asset linkage, INV for parts reservation and consumption, Procurement for service contracts and parts purchasing, and GL for maintenance cost accounting.

**Bounded Context:** Asset Operations, Maintenance & Reliability
**Service Name:** `eam-service`
**Database:** `data/eam.db`
**HTTP Port:** 8067 | **gRPC Port:** 9067

---

## 2. Database Schema

### 2.1 Operating Assets
```sql
CREATE TABLE eam_operating_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    financial_asset_id TEXT,                   -- FK to fa-service fixed asset
    asset_tag TEXT NOT NULL,                   -- Physical tag/barcode
    asset_name TEXT NOT NULL,
    asset_category TEXT NOT NULL
        CHECK(asset_category IN ('MACHINERY','VEHICLE','IT_EQUIPMENT','FACILITY','TOOL')),
    location_id TEXT NOT NULL,
    parent_asset_id TEXT,                      -- For asset hierarchy
    manufacturer TEXT,
    model_number TEXT,
    serial_number TEXT,
    installation_date TEXT,
    warranty_expiry TEXT,
    criticality TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(criticality IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    operating_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(operating_status IN ('ACTIVE','IDLE','MAINTENANCE','DECOMMISSIONED')),
    condition_rating TEXT NOT NULL DEFAULT 'A'
        CHECK(condition_rating IN ('A','B','C','D','F')),
    specifications TEXT,                       -- JSON: technical specs
    gps_latitude REAL,
    gps_longitude REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, asset_tag)
);

CREATE INDEX idx_assets_tenant_location ON eam_operating_assets(tenant_id, location_id);
CREATE INDEX idx_assets_tenant_category ON eam_operating_assets(tenant_id, asset_category);
CREATE INDEX idx_assets_tenant_status ON eam_operating_assets(tenant_id, operating_status);
CREATE INDEX idx_assets_tenant_criticality ON eam_operating_assets(tenant_id, criticality);
CREATE INDEX idx_assets_tenant_parent ON eam_operating_assets(tenant_id, parent_asset_id);
```

### 2.2 Maintenance Plans
```sql
CREATE TABLE eam_maintenance_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    asset_id TEXT,                             -- Nullable for group/category plans
    asset_category TEXT,
    plan_type TEXT NOT NULL
        CHECK(plan_type IN ('PREVENTIVE','PREDICTIVE','CONDITION_BASED')),
    frequency_type TEXT NOT NULL
        CHECK(frequency_type IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY','ANNUAL','METER_BASED')),
    frequency_interval INTEGER NOT NULL DEFAULT 1,
    description TEXT,
    work_order_template_id TEXT,
    estimated_duration_hours REAL NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    next_due_date TEXT NOT NULL,
    last_completed_date TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    auto_generate INTEGER NOT NULL DEFAULT 0,
    tolerance_days INTEGER NOT NULL DEFAULT 7,
    triggers TEXT,                             -- JSON: meter reading/threshold triggers

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (asset_id) REFERENCES eam_operating_assets(id) ON DELETE SET NULL
);

CREATE INDEX idx_plans_tenant_asset ON eam_maintenance_plans(tenant_id, asset_id);
CREATE INDEX idx_plans_tenant_due ON eam_maintenance_plans(tenant_id, next_due_date);
CREATE INDEX idx_plans_tenant_active ON eam_maintenance_plans(tenant_id, is_active);
```

### 2.3 Work Orders
```sql
CREATE TABLE eam_work_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_number TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    work_type TEXT NOT NULL
        CHECK(work_type IN ('CORRECTIVE','PREVENTIVE','PREDICTIVE','EMERGENCY','INSPECTION')),
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','URGENT')),
    description TEXT NOT NULL,
    problem_code TEXT,
    cause_code TEXT,
    remedy_code TEXT,
    status TEXT NOT NULL DEFAULT 'REQUESTED'
        CHECK(status IN ('REQUESTED','APPROVED','SCHEDULED','IN_PROGRESS','ON_HOLD','COMPLETED','CLOSED')),
    requested_by TEXT NOT NULL,
    assigned_to TEXT,
    scheduled_start TEXT,
    scheduled_end TEXT,
    actual_start TEXT,
    actual_end TEXT,
    downtime_hours REAL NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    location_id TEXT,
    maintenance_plan_id TEXT,
    parent_work_order_id TEXT,
    safety_requirements TEXT,
    completion_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (asset_id) REFERENCES eam_operating_assets(id),
    FOREIGN KEY (maintenance_plan_id) REFERENCES eam_maintenance_plans(id) ON DELETE SET NULL,

    UNIQUE(tenant_id, work_order_number)
);

CREATE INDEX idx_work_orders_tenant_asset ON eam_work_orders(tenant_id, asset_id);
CREATE INDEX idx_work_orders_tenant_status ON eam_work_orders(tenant_id, status);
CREATE INDEX idx_work_orders_tenant_type ON eam_work_orders(tenant_id, work_type);
CREATE INDEX idx_work_orders_tenant_priority ON eam_work_orders(tenant_id, priority);
CREATE INDEX idx_work_orders_tenant_assigned ON eam_work_orders(tenant_id, assigned_to);
CREATE INDEX idx_work_orders_tenant_dates ON eam_work_orders(tenant_id, scheduled_start, scheduled_end);
```

### 2.4 Work Order Tasks
```sql
CREATE TABLE eam_work_order_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    task_sequence INTEGER NOT NULL,
    task_description TEXT NOT NULL,
    task_type TEXT NOT NULL
        CHECK(task_type IN ('INSPECT','REPLACE','ADJUST','LUBRICATE','MEASURE','CLEAN')),
    is_completed INTEGER NOT NULL DEFAULT 0,
    completed_by TEXT,
    completed_at TEXT,
    measurements TEXT,                         -- JSON: measurement readings
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (work_order_id) REFERENCES eam_work_orders(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, work_order_id, task_sequence)
);

CREATE INDEX idx_tasks_tenant_work_order ON eam_work_order_tasks(tenant_id, work_order_id);
```

### 2.5 Work Order Parts
```sql
CREATE TABLE eam_work_order_parts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    quantity_required DECIMAL(18,4) NOT NULL DEFAULT 0,
    quantity_used DECIMAL(18,4) NOT NULL DEFAULT 0,
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    warehouse_id TEXT NOT NULL,
    is_reserved INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (work_order_id) REFERENCES eam_work_orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_parts_tenant_work_order ON eam_work_order_parts(tenant_id, work_order_id);
CREATE INDEX idx_parts_tenant_item ON eam_work_order_parts(tenant_id, item_id);
```

### 2.6 Meter Readings
```sql
CREATE TABLE eam_meter_readings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    reading_type TEXT NOT NULL
        CHECK(reading_type IN ('HOURS','KILOMETERS','CYCLES','TEMPERATURE','VIBRATION','PRESSURE','FLOW')),
    reading_value REAL NOT NULL,
    reading_unit TEXT NOT NULL,
    reading_date TEXT NOT NULL,
    reading_source TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(reading_source IN ('MANUAL','IOT_SENSOR','TELEMETRY')),
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (asset_id) REFERENCES eam_operating_assets(id)
);

CREATE INDEX idx_meter_readings_tenant_asset ON eam_meter_readings(tenant_id, asset_id);
CREATE INDEX idx_meter_readings_tenant_date ON eam_meter_readings(tenant_id, reading_date);
CREATE INDEX idx_meter_readings_tenant_type ON eam_meter_readings(tenant_id, reading_type);
```

### 2.7 Asset Failures
```sql
CREATE TABLE eam_asset_failures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    work_order_id TEXT,
    failure_date TEXT NOT NULL,
    failure_type TEXT NOT NULL
        CHECK(failure_type IN ('MECHANICAL','ELECTRICAL','SOFTWARE','OPERATOR','WEAR')),
    failure_code TEXT NOT NULL,
    component_failed TEXT,
    root_cause TEXT,
    corrective_action TEXT,
    downtime_hours REAL NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (asset_id) REFERENCES eam_operating_assets(id),
    FOREIGN KEY (work_order_id) REFERENCES eam_work_orders(id) ON DELETE SET NULL
);

CREATE INDEX idx_failures_tenant_asset ON eam_asset_failures(tenant_id, asset_id);
CREATE INDEX idx_failures_tenant_date ON eam_asset_failures(tenant_id, failure_date);
CREATE INDEX idx_failures_tenant_type ON eam_asset_failures(tenant_id, failure_type);
```

### 2.8 Service Contracts
```sql
CREATE TABLE eam_service_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,
    vendor_id TEXT NOT NULL,
    contract_type TEXT NOT NULL
        CHECK(contract_type IN ('FULL_SERVICE','PARTS_ONLY','LABOR_ONLY','TIME_MATERIALS')),
    coverage_scope TEXT,
    annual_cost_cents INTEGER NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL,
    effective_to TEXT NOT NULL,
    response_time_hours INTEGER NOT NULL DEFAULT 24,
    sla_terms TEXT,
    auto_renew INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_contracts_tenant_vendor ON eam_service_contracts(tenant_id, vendor_id);
CREATE INDEX idx_contracts_tenant_status ON eam_service_contracts(tenant_id, status);
CREATE INDEX idx_contracts_tenant_expiry ON eam_service_contracts(tenant_id, effective_to);
```

### 2.9 Safety Permits
```sql
CREATE TABLE eam_safety_permits (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    work_order_id TEXT,
    permit_type TEXT NOT NULL
        CHECK(permit_type IN ('HOT_WORK','CONFINED_SPACE','LOCKOUT_TAGOUT','ELECTRICAL','EXCAVATION')),
    status TEXT NOT NULL DEFAULT 'REQUESTED'
        CHECK(status IN ('REQUESTED','APPROVED','ACTIVE','CLOSED','EXPIRED')),
    issued_by TEXT,
    issued_at TEXT,
    expires_at TEXT,
    closed_by TEXT,
    closed_at TEXT,
    safety_checklist TEXT,                     -- JSON: checklist items and responses

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (asset_id) REFERENCES eam_operating_assets(id),
    FOREIGN KEY (work_order_id) REFERENCES eam_work_orders(id) ON DELETE SET NULL
);

CREATE INDEX idx_permits_tenant_asset ON eam_safety_permits(tenant_id, asset_id);
CREATE INDEX idx_permits_tenant_status ON eam_safety_permits(tenant_id, status);
CREATE INDEX idx_permits_tenant_expiry ON eam_safety_permits(tenant_id, expires_at);
```

### 2.10 Asset Decommission
```sql
CREATE TABLE eam_asset_decommission (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    decommission_date TEXT NOT NULL,
    decommission_reason TEXT NOT NULL
        CHECK(decommission_reason IN ('REPLACEMENT','BEYOND_REPAIR','OBSOLETE','SAFETY')),
    disposal_method TEXT NOT NULL
        CHECK(disposal_method IN ('SELL','SCRAP','DONATE','RECYCLE')),
    disposal_value_cents INTEGER NOT NULL DEFAULT 0,
    environmental_compliance TEXT,
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','APPROVED','REJECTED')),
    approved_by TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (asset_id) REFERENCES eam_operating_assets(id),
    UNIQUE(tenant_id, asset_id)
);
```

---

## 3. REST API Endpoints

```
# Operating Assets
GET/POST      /api/v1/eam/assets                            Permission: eam.assets.read/create
GET/PUT       /api/v1/eam/assets/{id}                       Permission: eam.assets.read/update
GET           /api/v1/eam/assets/{id}/hierarchy             Permission: eam.assets.read
GET           /api/v1/eam/assets/{id}/history               Permission: eam.assets.read
GET           /api/v1/eam/assets/{id}/health                Permission: eam.assets.read
POST          /api/v1/eam/assets/{id}/decommission          Permission: eam.assets.decommission

# Maintenance Plans
GET/POST      /api/v1/eam/maintenance-plans                 Permission: eam.plans.read/create
GET/PUT       /api/v1/eam/maintenance-plans/{id}            Permission: eam.plans.read/update
POST          /api/v1/eam/maintenance-plans/{id}/generate-work-order  Permission: eam.plans.execute
GET           /api/v1/eam/maintenance-plans/calendar        Permission: eam.plans.read

# Work Orders
GET/POST      /api/v1/eam/work-orders                       Permission: eam.work-orders.read/create
GET/PUT       /api/v1/eam/work-orders/{id}                  Permission: eam.work-orders.read/update
POST          /api/v1/eam/work-orders/{id}/approve          Permission: eam.work-orders.approve
POST          /api/v1/eam/work-orders/{id}/start            Permission: eam.work-orders.execute
POST          /api/v1/eam/work-orders/{id}/hold             Permission: eam.work-orders.execute
POST          /api/v1/eam/work-orders/{id}/complete         Permission: eam.work-orders.execute
POST          /api/v1/eam/work-orders/{id}/close            Permission: eam.work-orders.close
GET/POST      /api/v1/eam/work-orders/{id}/tasks            Permission: eam.work-orders.read/create
GET/POST      /api/v1/eam/work-orders/{id}/parts            Permission: eam.work-orders.read/create
POST          /api/v1/eam/work-orders/{id}/parts/{pid}/reserve  Permission: eam.work-orders.execute

# Meter Readings
GET/POST      /api/v1/eam/meter-readings                    Permission: eam.readings.read/create
GET           /api/v1/eam/meter-readings/asset/{asset_id}   Permission: eam.readings.read

# Failures
GET/POST      /api/v1/eam/failures                          Permission: eam.failures.read/create
GET           /api/v1/eam/failures/{id}                     Permission: eam.failures.read

# Service Contracts
GET/POST      /api/v1/eam/service-contracts                 Permission: eam.contracts.read/create
GET/PUT       /api/v1/eam/service-contracts/{id}            Permission: eam.contracts.read/update

# Safety Permits
GET/POST      /api/v1/eam/safety-permits                    Permission: eam.permits.read/create
GET/PUT       /api/v1/eam/safety-permits/{id}               Permission: eam.permits.read/update
POST          /api/v1/eam/safety-permits/{id}/approve       Permission: eam.permits.approve
POST          /api/v1/eam/safety-permits/{id}/close         Permission: eam.permits.execute

# Reports
GET           /api/v1/eam/reports/asset-reliability         Permission: eam.reports.view
GET           /api/v1/eam/reports/maintenance-cost          Permission: eam.reports.view
GET           /api/v1/eam/reports/mtbf-mttr                 Permission: eam.reports.view
GET           /api/v1/eam/reports/work-order-backlog        Permission: eam.reports.view
GET           /api/v1/eam/reports/warranty-tracking         Permission: eam.reports.view
GET           /api/v1/eam/reports/planned-vs-unplanned      Permission: eam.reports.view
```

---

## 4. Business Rules

### 4.1 Preventive Maintenance Auto-Generation
```
Based on maintenance plan schedule or meter readings:
  1. Check all active maintenance plans with auto_generate enabled
  2. For time-based plans: generate work order when next_due_date within tolerance_days
  3. For meter-based plans: compare latest meter reading against trigger thresholds
  4. Create work order from template with tasks and estimated parts
  5. Set priority based on asset criticality:
     CRITICAL asset -> HIGH priority
     HIGH asset -> MEDIUM priority
     MEDIUM/LOW asset -> LOW priority
  6. Update plan last_completed_date and next_due_date
```

### 4.2 Work Order Lifecycle
- **REQUESTED:** Initial state, requires approval (except EMERGENCY)
- **APPROVED:** Approved by authorized personnel
- **SCHEDULED:** Assigned to technician with start/end dates
- **IN_PROGRESS:** Work has begun (actual_start recorded)
- **ON_HOLD:** Temporarily paused with reason
- **COMPLETED:** Work finished (actual_end, downtime_hours recorded)
- **CLOSED:** Final review and cost reconciliation complete
- Emergency work orders bypass REQUESTED -> APPROVED flow; go directly to IN_PROGRESS

### 4.3 Parts and Inventory
- Parts reservation triggers inventory reservation via inv-service
- Reserved parts decremented from available quantity
- quantity_used recorded on work order completion
- Variance between required and used tracked for analysis
- Cost calculated from actual quantity_used * unit_cost

### 4.4 Asset Criticality and Priority
- Asset criticality determines maintenance priority and escalation rules
- CRITICAL assets: emergency response within 1 hour, preventive adherence tracked
- HIGH assets: emergency response within 4 hours, preventive adherence tracked
- MEDIUM assets: emergency response within 24 hours
- LOW assets: standard scheduling
- Priority escalation: overdue CRITICAL/HIGH work orders escalate automatically

### 4.5 MTBF and MTTR Calculation
- **MTBF (Mean Time Between Failures):** `Total operating hours / Number of failures`
- **MTTR (Mean Time To Repair):** `Total downtime hours / Number of failures`
- Calculated per asset, per asset category, and per location
- Rolling 12-month calculation window
- Used for reliability reporting and predictive maintenance triggers

### 4.6 Safety Permits and Lockout/Tagout
- Safety permits MUST be approved before work begins on applicable assets
- Permit types: HOT_WORK, CONFINED_SPACE, LOCKOUT_TAGOUT, ELECTRICAL, EXCAVATION
- Lockout/tagout procedures MUST be closed before work order can be completed
- Expired permits auto-flagged; work must stop until renewal
- Safety checklist must be fully completed before permit closure
- All permit activities logged with timestamps and responsible parties

### 4.7 Asset Decommission
- Asset decommission requires financial asset disposal in fa-service
- Decommission sets operating_status to DECOMMISSIONED
- Active maintenance plans for decommissioned assets are deactivated
- Open work orders for decommissioned assets are cancelled
- Disposal value posted to financial records
- Environmental compliance requirements documented and verified

### 4.8 Condition Rating Updates
- Condition ratings update based on failure history and inspection results
- Rating scale: A (Excellent) through F (Replace Immediately)
- Auto-downgrade triggers: multiple failures within 90 days, critical failure
- Upgrade only through formal inspection with documented evidence
- Rating changes tracked in asset history

### 4.9 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `eam.work_order.created` | New work order created | Workflow, FA |
| `eam.work_order.completed` | Work order marked complete | GL, FA |
| `eam.failure.reported` | Asset failure logged | FA, Workflow |
| `eam.maintenance.due` | Maintenance plan due date reached | Workflow |
| `eam.safety_permit.expired` | Safety permit past expiry | Workflow |
| `eam.asset.decommissioned` | Asset decommission finalized | FA, GL |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.eam.v1;

service EamService {
    rpc GetAssetHealth(GetAssetHealthRequest) returns (GetAssetHealthResponse);
    rpc CreateMaintenanceOrder(CreateMaintenanceOrderRequest) returns (CreateMaintenanceOrderResponse);
    rpc RecordMeterReading(RecordMeterReadingRequest) returns (RecordMeterReadingResponse);
    rpc CheckMaintenanceDue(CheckMaintenanceDueRequest) returns (CheckMaintenanceDueResponse);
}
```

```protobuf
message GetAssetHealthRequest {
    string tenant_id = 1;
    string asset_id = 2;
}

message GetAssetHealthResponse {
    OperatingAsset asset = 1;
    string condition_rating = 2;
    double mtbf_hours = 3;
    double mttr_hours = 4;
    int32 open_work_orders = 5;
    int32 total_failures_12m = 6;
    double maintenance_cost_12m_cents = 7;
}

message CreateMaintenanceOrderRequest {
    string tenant_id = 1;
    string asset_id = 2;
    string work_type = 3;
    string priority = 4;
    string description = 5;
    string problem_code = 6;
    string requested_by = 7;
    string scheduled_start = 8;
    string scheduled_end = 9;
    int64 estimated_cost_cents = 10;
    string maintenance_plan_id = 11;
    string created_by = 12;
}

message CreateMaintenanceOrderResponse {
    WorkOrder work_order = 1;
}

message RecordMeterReadingRequest {
    string tenant_id = 1;
    string asset_id = 2;
    string reading_type = 3;
    double reading_value = 4;
    string reading_unit = 5;
    string reading_date = 6;
    string reading_source = 7;
    string notes = 8;
    string created_by = 9;
}

message RecordMeterReadingResponse {
    bool success = 1;
    string reading_id = 2;
}

message CheckMaintenanceDueRequest {
    string tenant_id = 1;
    string asset_id = 2;
    string due_within_days = 3;
}

message CheckMaintenanceDueResponse {
    repeated MaintenancePlan due_plans = 1;
    int32 overdue_count = 2;
    int32 due_soon_count = 3;
}

message OperatingAsset {
    string id = 1;
    string tenant_id = 2;
    string financial_asset_id = 3;
    string asset_tag = 4;
    string asset_name = 5;
    string asset_category = 6;
    string location_id = 7;
    string parent_asset_id = 8;
    string manufacturer = 9;
    string model_number = 10;
    string serial_number = 11;
    string installation_date = 12;
    string warranty_expiry = 13;
    string criticality = 14;
    string operating_status = 15;
    string condition_rating = 16;
    string specifications = 17;
    double gps_latitude = 18;
    double gps_longitude = 19;
    string created_at = 20;
    string updated_at = 21;
    string created_by = 22;
    string updated_by = 23;
    int32 version = 24;
    int32 is_active = 25;
}

message MaintenancePlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string asset_id = 4;
    string asset_category = 5;
    string plan_type = 6;
    string frequency_type = 7;
    int32 frequency_interval = 8;
    string description = 9;
    string work_order_template_id = 10;
    double estimated_duration_hours = 11;
    int64 estimated_cost_cents = 12;
    string next_due_date = 13;
    string last_completed_date = 14;
    int32 is_active = 15;
    int32 auto_generate = 16;
    int32 tolerance_days = 17;
    string triggers = 18;
    string created_at = 19;
    string updated_at = 20;
    string created_by = 21;
    string updated_by = 22;
    int32 version = 23;
}

message WorkOrder {
    string id = 1;
    string tenant_id = 2;
    string work_order_number = 3;
    string asset_id = 4;
    string work_type = 5;
    string priority = 6;
    string description = 7;
    string problem_code = 8;
    string cause_code = 9;
    string remedy_code = 10;
    string status = 11;
    string requested_by = 12;
    string assigned_to = 13;
    string scheduled_start = 14;
    string scheduled_end = 15;
    string actual_start = 16;
    string actual_end = 17;
    double downtime_hours = 18;
    int64 estimated_cost_cents = 19;
    int64 actual_cost_cents = 20;
    string currency_code = 21;
    string location_id = 22;
    string maintenance_plan_id = 23;
    string parent_work_order_id = 24;
    string safety_requirements = 25;
    string completion_notes = 26;
    string created_at = 27;
    string updated_at = 28;
    string created_by = 29;
    string updated_by = 30;
    int32 version = 31;
    int32 is_active = 32;
}

message WorkOrderTask {
    string id = 1;
    string tenant_id = 2;
    string work_order_id = 3;
    int32 task_sequence = 4;
    string task_description = 5;
    string task_type = 6;
    int32 is_completed = 7;
    string completed_by = 8;
    string completed_at = 9;
    string measurements = 10;
    string notes = 11;
    string created_at = 12;
    string updated_at = 13;
    string created_by = 14;
    string updated_by = 15;
}

message WorkOrderPart {
    string id = 1;
    string tenant_id = 2;
    string work_order_id = 3;
    string item_id = 4;
    string quantity_required = 5;
    string quantity_used = 6;
    int64 unit_cost_cents = 7;
    string warehouse_id = 8;
    int32 is_reserved = 9;
    string created_at = 10;
    string updated_at = 11;
    string created_by = 12;
    string updated_by = 13;
}

message MeterReading {
    string id = 1;
    string tenant_id = 2;
    string asset_id = 3;
    string reading_type = 4;
    double reading_value = 5;
    string reading_unit = 6;
    string reading_date = 7;
    string reading_source = 8;
    string notes = 9;
    string created_at = 10;
    string created_by = 11;
}

message AssetFailure {
    string id = 1;
    string tenant_id = 2;
    string asset_id = 3;
    string work_order_id = 4;
    string failure_date = 5;
    string failure_type = 6;
    string failure_code = 7;
    string component_failed = 8;
    string root_cause = 9;
    string corrective_action = 10;
    double downtime_hours = 11;
    int64 cost_cents = 12;
    string created_at = 13;
    string updated_at = 14;
    string created_by = 15;
    string updated_by = 16;
}

message ServiceContract {
    string id = 1;
    string tenant_id = 2;
    string contract_number = 3;
    string vendor_id = 4;
    string contract_type = 5;
    string coverage_scope = 6;
    int64 annual_cost_cents = 7;
    string effective_from = 8;
    string effective_to = 9;
    int32 response_time_hours = 10;
    string sla_terms = 11;
    int32 auto_renew = 12;
    string status = 13;
    string created_at = 14;
    string updated_at = 15;
    string created_by = 16;
    string updated_by = 17;
    int32 version = 18;
}

message SafetyPermit {
    string id = 1;
    string tenant_id = 2;
    string asset_id = 3;
    string work_order_id = 4;
    string permit_type = 5;
    string status = 6;
    string issued_by = 7;
    string issued_at = 8;
    string expires_at = 9;
    string closed_by = 10;
    string closed_at = 11;
    string safety_checklist = 12;
    string created_at = 13;
    string updated_at = 14;
    string created_by = 15;
    string updated_by = 16;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **FA:** Financial asset records linked to operating assets, asset disposal triggers
- **INV:** Item master for parts, inventory availability checks, warehouse locations
- **Procurement:** Vendor details for service contracts, purchase orders for parts
- **CM:** Carrier information for asset transfers

### 6.2 Data Published To
- **FA:** Maintenance cost updates, asset condition changes, decommission/disposal events
- **INV:** Parts reservation requests, parts consumption on work order completion
- **Procurement:** Parts requisition when stock below reorder point
- **GL:** Maintenance cost accounting entries, disposal gain/loss postings
- **Workflow:** Approval workflows for work orders, safety permits, decommission requests

---

## 7. Migrations

1. V001: `eam_operating_assets`
2. V002: `eam_maintenance_plans`
3. V003: `eam_work_orders`
4. V004: `eam_work_order_tasks`
5. V005: `eam_work_order_parts`
6. V006: `eam_meter_readings`
7. V007: `eam_asset_failures`
8. V008: `eam_service_contracts`
9. V009: `eam_safety_permits`
10. V010: `eam_asset_decommission`
11. V011: Triggers for `updated_at`
