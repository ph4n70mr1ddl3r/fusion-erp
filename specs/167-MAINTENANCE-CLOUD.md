# 167 - Maintenance Cloud Service Specification

## 1. Domain Overview

Maintenance Cloud provides production asset maintenance management including preventive, predictive, and condition-based maintenance programs. Covers maintenance planning, work order execution, failure code analysis, maintenance schedules, meter-based triggers, and maintenance cost tracking for production equipment. Distinct from Enterprise Asset Management which covers the full asset financial lifecycle; Maintenance Cloud focuses specifically on keeping production equipment operational and optimizing maintenance strategies. Integrates with Manufacturing for production schedules, Inventory for spare parts, Quality for inspection results, and IoT for condition monitoring.

**Bounded Context:** Production Maintenance & Reliability Engineering
**Service Name:** `maintenance-service`
**Database:** `data/maintenance.db`
**HTTP Port:** 8185 | **gRPC Port:** 9185

---

## 2. Database Schema

### 2.1 Maintenance Assets (Production Equipment)
```sql
CREATE TABLE mc_production_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_code TEXT NOT NULL,
    asset_name TEXT NOT NULL,
    asset_type TEXT NOT NULL CHECK(asset_type IN ('MACHINE','LINE','ROBOT','TOOL','VEHICLE','UTILITY','CUSTOM')),
    manufacturer TEXT,
    model TEXT,
    serial_number TEXT,
    installation_date TEXT,
    facility_id TEXT NOT NULL,
    department_id TEXT,
    production_line_id TEXT,
    work_center_id TEXT,
    parent_asset_id TEXT,
    criticality TEXT NOT NULL DEFAULT 'MODERATE'
        CHECK(criticality IN ('LOW','MODERATE','HIGH','MISSION_CRITICAL')),
    current_condition TEXT NOT NULL DEFAULT 'OPERATIONAL'
        CHECK(current_condition IN ('OPERATIONAL','DEGRADED','DOWN','OUT_OF_SERVICE')),
    condition_score REAL NOT NULL DEFAULT 100, -- 0-100 health index
    total_runtime_hours REAL NOT NULL DEFAULT 0,
    total_downtime_hours REAL NOT NULL DEFAULT 0,
    mtbf_hours REAL,                        -- Mean Time Between Failures
    mttr_hours REAL,                        -- Mean Time To Repair
    availability_pct REAL NOT NULL DEFAULT 100,
    oee_pct REAL NOT NULL DEFAULT 0,        -- Overall Equipment Effectiveness
    replacement_value_cents INTEGER NOT NULL DEFAULT 0,
    eam_asset_id TEXT,                      -- Link to Enterprise Asset Management

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, asset_code)
);

CREATE INDEX idx_mc_asset_facility ON mc_production_assets(facility_id, current_condition);
CREATE INDEX idx_mc_asset_criticality ON mc_production_assets(tenant_id, criticality);
CREATE INDEX idx_mc_asset_wc ON mc_production_assets(work_center_id);
```

### 2.2 Maintenance Programs
```sql
CREATE TABLE mc_maintenance_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_name TEXT NOT NULL,
    strategy_type TEXT NOT NULL CHECK(strategy_type IN (
        'PREVENTIVE_TIME','PREVENTIVE_USAGE','PREDICTIVE','CONDITION_BASED',
        'REACTIVE','RELIABILITY_CENTERED','TOTAL_PRODUCTIVE','CUSTOM'
    )),
    description TEXT,
    asset_id TEXT NOT NULL,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','CRITICAL')),
    trigger_type TEXT NOT NULL CHECK(trigger_type IN (
        'CALENDAR','METER_READING','CONDITION_THRESHOLD','IOT_SIGNAL','MANUAL','EVENT'
    )),
    trigger_config TEXT NOT NULL,           -- JSON: trigger parameters (interval, thresholds, etc.)
    estimated_duration_hours REAL NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    required_skills TEXT,                   -- JSON: required technician skills
    required_parts TEXT,                    -- JSON: [{ "item_id": "...", "quantity": 1 }]
    safety_requirements TEXT,               -- JSON: permits, LOTO procedures, PPE
    checklists TEXT,                        -- JSON: inspection checklist items
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','SUSPENDED','ARCHIVED')),
    last_execution_date TEXT,
    next_due_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_code)
);

CREATE INDEX idx_mc_prog_asset ON mc_maintenance_programs(asset_id, status);
CREATE INDEX idx_mc_prog_due ON mc_maintenance_programs(tenant_id, next_due_date, status);
CREATE INDEX idx_mc_prog_strategy ON mc_maintenance_programs(tenant_id, strategy_type);
```

### 2.3 Maintenance Work Orders
```sql
CREATE TABLE mc_work_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_number TEXT NOT NULL,         -- MWO-2024-00001
    asset_id TEXT NOT NULL,
    program_id TEXT,                         -- NULL for reactive/unplanned
    work_type TEXT NOT NULL CHECK(work_type IN (
        'PREVENTIVE','CORRECTIVE','PREDICTIVE','EMERGENCY','INSPECTION',
        'CALIBRATION','OVERHAUL','MODIFICATION','PROJECT','CUSTOM'
    )),
    description TEXT NOT NULL,
    problem_code TEXT,
    cause_code TEXT,
    remedy_code TEXT,
    failure_code TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT','EMERGENCY')),
    status TEXT NOT NULL DEFAULT 'CREATED'
        CHECK(status IN ('CREATED','PLANNED','SCHEDULED','ASSIGNED','IN_PROGRESS',
                         'ON_HOLD','COMPLETED','CLOSED','CANCELLED')),
    assigned_technician_id TEXT,
    assigned_team_id TEXT,
    scheduled_start TEXT NOT NULL,
    scheduled_end TEXT NOT NULL,
    actual_start TEXT,
    actual_end TEXT,
    downtime_start TEXT,
    downtime_end TEXT,
    downtime_hours REAL NOT NULL DEFAULT 0,
    production_impact TEXT CHECK(production_impact IN ('NONE','REDUCED','STOPPED')),
    estimated_hours REAL NOT NULL DEFAULT 0,
    actual_hours REAL NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    completion_notes TEXT,
    root_cause_analysis TEXT,
    parent_wo_id TEXT,                      -- For break-down work orders

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, work_order_number)
);

CREATE INDEX idx_mc_wo_asset ON mc_work_orders(asset_id, status);
CREATE INDEX idx_mc_wo_status ON mc_work_orders(tenant_id, status);
CREATE INDEX idx_mc_wo_schedule ON mc_work_orders(tenant_id, scheduled_start);
CREATE INDEX idx_mc_wo_tech ON mc_work_orders(assigned_technician_id, status);
CREATE INDEX idx_mc_wo_type ON mc_work_orders(tenant_id, work_type);
```

### 2.4 Work Order Operations
```sql
CREATE TABLE mc_wo_operations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    operation_seq INTEGER NOT NULL,
    operation_description TEXT NOT NULL,
    operation_type TEXT NOT NULL CHECK(operation_type IN ('INSPECT','REPAIR','REPLACE','ADJUST','LUBRICATE','TEST','CLEAN','CUSTOM')),
    estimated_hours REAL NOT NULL DEFAULT 0,
    actual_hours REAL NOT NULL DEFAULT 0,
    assigned_technician_id TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','SKIPPED')),
    checklist_items TEXT,                   -- JSON: [{ "item": "...", "passed": true }]
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (work_order_id) REFERENCES mc_work_orders(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, work_order_id, operation_seq)
);
```

### 2.5 Meter Readings
```sql
CREATE TABLE mc_meter_readings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    meter_type TEXT NOT NULL CHECK(meter_type IN ('RUNTIME_HOURS','CYCLES','PRODUCTION_UNITS','TEMPERATURE','VIBRATION','PRESSURE','CUSTOM')),
    reading_value REAL NOT NULL,
    unit TEXT NOT NULL,
    reading_date TEXT NOT NULL,
    reading_source TEXT NOT NULL CHECK(reading_source IN ('MANUAL','AUTOMATED','IOT_SENSOR','CALCULATED')),
    previous_value REAL,
    delta_value REAL,
    triggered_program_id TEXT,              -- If reading triggered a maintenance program

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(id)
);

CREATE INDEX idx_mc_meter_asset ON mc_meter_readings(asset_id, meter_type, reading_date DESC);
```

### 2.6 Failure Analysis
```sql
CREATE TABLE mc_failure_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    failure_date TEXT NOT NULL,
    failure_type TEXT NOT NULL CHECK(failure_type IN ('MECHANICAL','ELECTRICAL','HYDRAULIC','PNEUMATIC','SOFTWARE','OPERATOR_ERROR','WEAR','CORROSION','OTHER')),
    failure_severity TEXT NOT NULL CHECK(failure_severity IN ('MINOR','MAJOR','CRITICAL','CATASTROPHIC')),
    failure_mode TEXT,                      -- How it failed
    failure_effect TEXT,                    -- What happened
    failure_cause TEXT,                     -- Why it failed
    failure_mechanism TEXT,                 -- Physical root cause
    detection_method TEXT,
    corrective_action TEXT,
    preventive_action TEXT,
    downtime_hours REAL NOT NULL DEFAULT 0,
    repair_cost_cents INTEGER NOT NULL DEFAULT 0,
    production_loss_cents INTEGER NOT NULL DEFAULT 0,
    rpn_score REAL,                         -- Risk Priority Number (Severity × Occurrence × Detection)

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (work_order_id) REFERENCES mc_work_orders(id) ON DELETE RESTRICT
);

CREATE INDEX idx_mc_fail_asset ON mc_failure_records(asset_id, failure_date DESC);
CREATE INDEX idx_mc_fail_type ON mc_failure_records(tenant_id, failure_type);
CREATE INDEX idx_mc_fail_severity ON mc_failure_records(tenant_id, failure_severity);
```

---

## 3. API Endpoints

### 3.1 Production Assets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maintenance/v1/assets` | Register production asset |
| GET | `/maintenance/v1/assets` | List assets |
| GET | `/maintenance/v1/assets/{id}` | Get asset details |
| PUT | `/maintenance/v1/assets/{id}` | Update asset |
| GET | `/maintenance/v1/assets/{id}/health` | Get asset health score |
| GET | `/maintenance/v1/assets/{id}/history` | Get maintenance history |

### 3.2 Maintenance Programs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maintenance/v1/programs` | Create maintenance program |
| GET | `/maintenance/v1/programs` | List programs |
| GET | `/maintenance/v1/programs/{id}` | Get program details |
| PUT | `/maintenance/v1/programs/{id}` | Update program |
| POST | `/maintenance/v1/programs/{id}/activate` | Activate program |
| POST | `/maintenance/v1/programs/generate-work-orders` | Generate due work orders |

### 3.3 Work Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maintenance/v1/work-orders` | Create work order |
| GET | `/maintenance/v1/work-orders` | List work orders |
| GET | `/maintenance/v1/work-orders/{id}` | Get work order details |
| PUT | `/maintenance/v1/work-orders/{id}` | Update work order |
| POST | `/maintenance/v1/work-orders/{id}/assign` | Assign technician |
| POST | `/maintenance/v1/work-orders/{id}/start` | Start work |
| POST | `/maintenance/v1/work-orders/{id}/complete` | Complete work order |
| POST | `/maintenance/v1/work-orders/{id}/close` | Close work order |

### 3.4 Meter Readings & Failure Analysis
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maintenance/v1/meter-readings` | Submit meter reading |
| GET | `/maintenance/v1/meter-readings` | List readings |
| POST | `/maintenance/v1/failures` | Record failure |
| GET | `/maintenance/v1/failures` | List failures |
| GET | `/maintenance/v1/failures/analysis` | Failure analysis reports |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/maintenance/v1/dashboard` | Maintenance dashboard |
| GET | `/maintenance/v1/kpi/mtbf` | MTBF metrics |
| GET | `/maintenance/v1/kpi/mttr` | MTTR metrics |
| GET | `/maintenance/v1/kpi/oee` | OEE metrics |
| GET | `/maintenance/v1/costs` | Maintenance cost analysis |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `maint.work_order.created` | `{ wo_id, asset_id, work_type, priority }` | Work order created |
| `maint.work_order.completed` | `{ wo_id, asset_id, downtime_hours }` | Work order completed |
| `maint.asset.condition_changed` | `{ asset_id, old_condition, new_condition }` | Asset condition changed |
| `maint.program.due` | `{ program_id, asset_id, due_date }` | Maintenance program due |
| `maint.meter.threshold` | `{ asset_id, meter_type, value, threshold }` | Meter reading crossed threshold |
| `maint.failure.recorded` | `{ asset_id, failure_type, severity }` | Equipment failure recorded |
| `maint.emergency.created` | `{ wo_id, asset_id, production_impact }` | Emergency work order |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `production.equipment.stopped` | Manufacturing | Create emergency work order |
| `iot.sensor.alert` | IoT Integration | Update condition score, trigger predictive maintenance |
| `quality.inspection.failed` | Quality Management | Create corrective work order |
| `inventory.parts.received` | Inventory | Update parts availability for scheduled WOs |
| `production.schedule.updated` | Manufacturing | Adjust maintenance schedule windows |

---

## 5. Business Rules

1. **PM Generation**: Preventive work orders auto-generated based on program trigger configuration
2. **Emergency Priority**: EMERGENCY work orders bypass scheduling; immediate technician assignment
3. **Downtime Tracking**: Production downtime recorded when asset status affects production
4. **Failure Code Hierarchy**: Problems → Causes → Remedies follow ISO 14224 failure coding
5. **Condition Scoring**: Asset health index calculated from failure history, age, meter readings
6. **Safety Permits**: Work orders in hazardous areas require active safety permits before execution
7. **Parts Reservation**: Required parts auto-reserved from inventory when work order scheduled
8. **Cost Rollup**: Actual costs rolled up from labor hours, parts consumed, and external services

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Manufacturing (14) | Production schedules, equipment status |
| Enterprise Asset Mgmt (40) | Financial asset data, asset lifecycle |
| Inventory (12) | Spare parts availability and reservation |
| Procurement (11) | Parts purchasing for maintenance |
| IoT Integration (44) | Condition monitoring, sensor data |
| Quality Management (41) | Quality-triggered maintenance |
| Cost Management (48) | Maintenance cost accounting |
| General Ledger (06) | Maintenance expense posting |
| Workforce Scheduling (73) | Technician scheduling |
| Production Scheduling (122) | Maintenance window coordination |
| Smart Operations (123) | AI-driven maintenance optimization |
