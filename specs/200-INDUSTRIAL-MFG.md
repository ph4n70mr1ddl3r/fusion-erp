# 200 - Industrial Manufacturing Industry Solution Specification

## 1. Domain Overview

Industrial Manufacturing provides industry-specific capabilities for heavy equipment registry, production line management with OEE tracking, work order lifecycle management, quality inspection with tolerance-based measurements, and safety incident management. Supports comprehensive equipment registry with criticality ratings and specifications, production line monitoring with availability, performance, and quality OEE components, planned and corrective work orders with technician assignment, variable and attribute quality inspections with sampling methods, and safety incident reporting with root cause analysis and corrective actions. Enables industrial manufacturers to maximize equipment uptime, maintain production quality, and ensure workplace safety. Integrates with Maintenance Cloud, Quality Management, Inventory, and Manufacturing.

**Bounded Context:** Industrial Manufacturing, Heavy Equipment & Project-Based Production
**Service Name:** `industrial-mfg-service`
**Database:** `data/industrial_mfg.db`
**HTTP Port:** 8218 | **gRPC Port:** 9218

---

## 2. Database Schema

### 2.1 Equipment Registry
```sql
CREATE TABLE im_equipment_registry (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_tag TEXT NOT NULL,
    equipment_name TEXT NOT NULL,
    equipment_type TEXT NOT NULL,
    manufacturer TEXT,
    model_number TEXT,
    serial_number TEXT,
    installation_date TEXT,
    location TEXT NOT NULL,                        -- PLANT/BAY/LINE format
    specifications TEXT,                           -- JSON: equipment specs
    criticality_rating TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(criticality_rating IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    warranty_expiry TEXT,
    status TEXT NOT NULL DEFAULT 'OPERATIONAL'
        CHECK(status IN ('OPERATIONAL','DOWN','MAINTENANCE','RETIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, asset_tag)
);

CREATE INDEX idx_im_equip_tenant ON im_equipment_registry(tenant_id, status);
CREATE INDEX idx_im_equip_criticality ON im_equipment_registry(criticality_rating);
CREATE INDEX idx_im_equip_location ON im_equipment_registry(location);
CREATE INDEX idx_im_equip_type ON im_equipment_registry(equipment_type);
```

### 2.2 Production Lines
```sql
CREATE TABLE im_production_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    line_code TEXT NOT NULL,
    line_name TEXT NOT NULL,
    plant_id TEXT NOT NULL,
    capacity_units_per_hour REAL NOT NULL DEFAULT 0,
    shift_pattern TEXT NOT NULL,                   -- JSON: shift definitions
    product_assignments TEXT,                      -- JSON: products assigned to line
    oee_pct REAL NOT NULL DEFAULT 0,
    availability_pct REAL NOT NULL DEFAULT 0,
    performance_pct REAL NOT NULL DEFAULT 0,
    quality_pct REAL NOT NULL DEFAULT 0,
    downtime_reasons TEXT,                         -- JSON: [{reason, minutes, count}]
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','MAINTENANCE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, line_code)
);

CREATE INDEX idx_im_line_plant ON im_production_lines(plant_id, status);
CREATE INDEX idx_im_line_oee ON im_production_lines(oee_pct);
```

### 2.3 Work Orders
```sql
CREATE TABLE im_work_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_number TEXT NOT NULL,
    work_order_type TEXT NOT NULL CHECK(work_order_type IN ('PLANNED','PREVENTIVE','CORRECTIVE','EMERGENCY')),
    equipment_id TEXT NOT NULL,
    description TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 5,
    assigned_technicians TEXT,                     -- JSON: technician IDs
    estimated_hours REAL NOT NULL DEFAULT 0,
    actual_hours REAL NOT NULL DEFAULT 0,
    scheduled_start TEXT,
    scheduled_end TEXT,
    completed_at TEXT,
    materials_used TEXT,                           -- JSON: materials list
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_PROGRESS','COMPLETED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (equipment_id) REFERENCES im_equipment_registry(id),
    UNIQUE(tenant_id, work_order_number)
);

CREATE INDEX idx_im_wo_equipment ON im_work_orders(equipment_id, status);
CREATE INDEX idx_im_wo_status ON im_work_orders(tenant_id, status);
CREATE INDEX idx_im_wo_priority ON im_work_orders(priority, status);
CREATE INDEX idx_im_wo_dates ON im_work_orders(scheduled_start);
```

### 2.4 Quality Inspections
```sql
CREATE TABLE im_quality_inspections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    inspection_plan_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    batch_number TEXT NOT NULL,
    checkpoint TEXT NOT NULL,
    measurement_type TEXT NOT NULL CHECK(measurement_type IN ('VARIABLE','ATTRIBUTE')),
    specification TEXT NOT NULL,
    tolerance_min REAL,
    tolerance_max REAL,
    measured_value REAL,
    result TEXT NOT NULL CHECK(result IN ('PASS','FAIL','MARGINAL')),
    inspector_id TEXT NOT NULL,
    inspected_at TEXT NOT NULL DEFAULT (datetime('now')),
    corrective_action TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(id)
);

CREATE INDEX idx_im_qi_product ON im_quality_inspections(product_id, inspected_at DESC);
CREATE INDEX idx_im_qi_batch ON im_quality_inspections(batch_number);
CREATE INDEX idx_im_qi_result ON im_quality_inspections(tenant_id, result);
CREATE INDEX idx_im_qi_plan ON im_quality_inspections(inspection_plan_id);
```

### 2.5 Safety Incidents
```sql
CREATE TABLE im_safety_incidents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    incident_number TEXT NOT NULL,
    incident_type TEXT NOT NULL CHECK(incident_type IN ('INJURY','NEAR_MISS','EQUIPMENT','ENVIRONMENTAL')),
    severity TEXT NOT NULL CHECK(severity IN ('MINOR','MODERATE','MAJOR','CRITICAL')),
    description TEXT NOT NULL,
    location TEXT NOT NULL,
    involved_persons TEXT,                         -- JSON: person IDs
    investigation_status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(investigation_status IN ('OPEN','INVESTIGATING','RESOLVED')),
    root_cause TEXT,
    corrective_actions TEXT,                       -- JSON: [{action, responsible, due_date, status}]
    reported_by TEXT NOT NULL,
    reported_at TEXT NOT NULL DEFAULT (datetime('now')),
    resolved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, incident_number)
);

CREATE INDEX idx_im_safety_type ON im_safety_incidents(incident_type, severity);
CREATE INDEX idx_im_safety_status ON im_safety_incidents(tenant_id, investigation_status);
CREATE INDEX idx_im_safety_location ON im_safety_incidents(location);
CREATE INDEX idx_im_safety_date ON im_safety_incidents(reported_at DESC);
```

---

## 3. API Endpoints

### 3.1 Equipment
| Method | Path | Description |
|--------|------|-------------|
| POST | `/industrial/v1/equipment` | Register equipment |
| GET | `/industrial/v1/equipment` | List equipment |
| GET | `/industrial/v1/equipment/{id}` | Get equipment |
| PUT | `/industrial/v1/equipment/{id}` | Update equipment |
| GET | `/industrial/v1/equipment/critical` | List critical equipment |

### 3.2 Production Lines
| Method | Path | Description |
|--------|------|-------------|
| POST | `/industrial/v1/lines` | Create production line |
| GET | `/industrial/v1/lines` | List lines |
| GET | `/industrial/v1/lines/{id}` | Get line with OEE |
| PUT | `/industrial/v1/lines/{id}` | Update line |
| POST | `/industrial/v1/lines/{id}/oee` | Update OEE metrics |

### 3.3 Work Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/industrial/v1/work-orders` | Create work order |
| GET | `/industrial/v1/work-orders` | List work orders |
| GET | `/industrial/v1/work-orders/{id}` | Get work order |
| POST | `/industrial/v1/work-orders/{id}/start` | Start work order |
| POST | `/industrial/v1/work-orders/{id}/complete` | Complete work order |

### 3.4 Quality
| Method | Path | Description |
|--------|------|-------------|
| POST | `/industrial/v1/inspections` | Create inspection |
| GET | `/industrial/v1/inspections` | List inspections |
| GET | `/industrial/v1/inspections/{id}` | Get inspection |
| POST | `/industrial/v1/inspections/{id}/record` | Record measurement |
| GET | `/industrial/v1/inspections/dashboard` | Quality dashboard |

### 3.5 Safety
| Method | Path | Description |
|--------|------|-------------|
| POST | `/industrial/v1/incidents` | Report incident |
| GET | `/industrial/v1/incidents` | List incidents |
| GET | `/industrial/v1/incidents/{id}` | Get incident |
| POST | `/industrial/v1/incidents/{id}/investigate` | Start investigation |
| POST | `/industrial/v1/incidents/{id}/resolve` | Resolve incident |
| GET | `/industrial/v1/incidents/dashboard` | Safety dashboard |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `indmfg.work_order.created` | `{ wo_number, equipment_id, type, priority }` | Work order created |
| `indmfg.quality.deviation` | `{ inspection_id, product, deviation }` | Quality deviation detected |
| `indmfg.safety.incident` | `{ incident_number, type, severity }` | Safety incident reported |
| `indmfg.oee.calculated` | `{ line_id, oee, availability, performance, quality }` | OEE updated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `maintenance.due` | Maintenance Cloud (167) | Create preventive work order |
| `inventory.consumed` | Inventory (34) | Update materials used |
| `order.placed` | Order Management (32) | Schedule production |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.industrial_mfg.v1;

service IndustrialMfgService {
    rpc GetEquipment(GetEquipmentRequest) returns (GetEquipmentResponse);
    rpc CreateEquipment(CreateEquipmentRequest) returns (CreateEquipmentResponse);
    rpc CreateWorkOrder(CreateWorkOrderRequest) returns (CreateWorkOrderResponse);
    rpc GetWorkOrder(GetWorkOrderRequest) returns (GetWorkOrderResponse);
    rpc CreateInspection(CreateInspectionRequest) returns (CreateInspectionResponse);
    rpc ReportIncident(ReportIncidentRequest) returns (ReportIncidentResponse);
}

// Equipment messages
message GetEquipmentRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetEquipmentResponse {
    ImEquipmentRegistry data = 1;
}

message CreateEquipmentRequest {
    string tenant_id = 1;
    string asset_tag = 2;
    string equipment_name = 3;
    string equipment_type = 4;
    string manufacturer = 5;
    string model_number = 6;
    string serial_number = 7;
    string installation_date = 8;
    string location = 9;
    string specifications = 10;
    string criticality_rating = 11;
    string warranty_expiry = 12;
}

message CreateEquipmentResponse {
    ImEquipmentRegistry data = 1;
}

message ImEquipmentRegistry {
    string id = 1;
    string tenant_id = 2;
    string asset_tag = 3;
    string equipment_name = 4;
    string equipment_type = 5;
    string manufacturer = 6;
    string model_number = 7;
    string serial_number = 8;
    string installation_date = 9;
    string location = 10;
    string specifications = 11;
    string criticality_rating = 12;
    string warranty_expiry = 13;
    string status = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Work order messages
message CreateWorkOrderRequest {
    string tenant_id = 1;
    string work_order_number = 2;
    string work_order_type = 3;
    string equipment_id = 4;
    string description = 5;
    int32 priority = 6;
    string assigned_technicians = 7;
    double estimated_hours = 8;
    string scheduled_start = 9;
    string scheduled_end = 10;
}

message CreateWorkOrderResponse {
    ImWorkOrder data = 1;
}

message GetWorkOrderRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetWorkOrderResponse {
    ImWorkOrder data = 1;
}

message ImWorkOrder {
    string id = 1;
    string tenant_id = 2;
    string work_order_number = 3;
    string work_order_type = 4;
    string equipment_id = 5;
    string description = 6;
    int32 priority = 7;
    string assigned_technicians = 8;
    double estimated_hours = 9;
    double actual_hours = 10;
    string scheduled_start = 11;
    string scheduled_end = 12;
    string completed_at = 13;
    string materials_used = 14;
    string status = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Inspection messages
message CreateInspectionRequest {
    string tenant_id = 1;
    string inspection_plan_id = 2;
    string product_id = 3;
    string batch_number = 4;
    string checkpoint = 5;
    string measurement_type = 6;
    string specification = 7;
    double tolerance_min = 8;
    double tolerance_max = 9;
    string inspector_id = 10;
}

message CreateInspectionResponse {
    ImQualityInspection data = 1;
}

message ImQualityInspection {
    string id = 1;
    string tenant_id = 2;
    string inspection_plan_id = 3;
    string product_id = 4;
    string batch_number = 5;
    string checkpoint = 6;
    string measurement_type = 7;
    string specification = 8;
    double tolerance_min = 9;
    double tolerance_max = 10;
    double measured_value = 11;
    string result = 12;
    string inspector_id = 13;
    string inspected_at = 14;
    string corrective_action = 15;
    string status = 16;
    string created_at = 17;
    string updated_at = 18;
}

// Safety incident messages
message ReportIncidentRequest {
    string tenant_id = 1;
    string incident_number = 2;
    string incident_type = 3;
    string severity = 4;
    string description = 5;
    string location = 6;
    string involved_persons = 7;
    string reported_by = 8;
}

message ReportIncidentResponse {
    ImSafetyIncident data = 1;
}

message ImSafetyIncident {
    string id = 1;
    string tenant_id = 2;
    string incident_number = 3;
    string incident_type = 4;
    string severity = 5;
    string description = 6;
    string location = 7;
    string involved_persons = 8;
    string investigation_status = 9;
    string root_cause = 10;
    string corrective_actions = 11;
    string reported_by = 12;
    string reported_at = 13;
    string resolved_at = 14;
    string created_at = 15;
    string updated_at = 16;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | im_equipment_registry | -- |
| V002 | im_production_lines | -- |
| V003 | im_work_orders | V001 |
| V004 | im_quality_inspections | -- |
| V005 | im_safety_incidents | -- |

---

## 7. Business Rules

1. **Criticality-Based Priority**: CRITICAL equipment work orders prioritized above all others
2. **OEE Calculation**: OEE = Availability x Performance x Quality; recalculated per shift
3. **Quality Tolerance**: Measurements outside tolerance auto-fail; marginal flagged for review
4. **Safety Escalation**: MAJOR and CRITICAL incidents auto-escalate to plant manager
5. **Work Order Dependencies**: Preventive work orders auto-generated from maintenance schedule
6. **Root Cause Analysis**: Resolved incidents require documented root cause and corrective actions

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| maintenance-service | `GetSchedule` | Preventive maintenance scheduling |
| quality-service | `GetSpec` / `GetNonConformance` | Quality specs and non-conformances |
| inventory-service | `ConsumeMaterial` | Material consumption tracking |
| order-service | `GetOrder` | Production order triggering |
| manufacturing-service | `GetProductionOrder` | Production execution |
| asset-service | `GetAsset` | Equipment asset registry |
| notification-service | `SendAlert` | Safety and quality alerts |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| maintenance-service | `GetWorkOrder` | Work order data for maintenance planning |
| quality-service | `GetInspection` | Quality inspection data |
