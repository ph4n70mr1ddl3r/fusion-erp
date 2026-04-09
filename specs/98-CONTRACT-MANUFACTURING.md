# 98 - Contract Manufacturing Service Specification

## 1. Domain Overview

Contract Manufacturing manages outsourced and contract manufacturing operations including contract manufacturer registration and capability tracking, work order outsourcing with full lifecycle management, raw material consignment inventory at manufacturer sites, quality inspections at contract manufacturing locations, detailed cost tracking for outsourced production across material/labor/overhead/tooling categories, contract BOM management with ownership assignment, and comprehensive manufacturer performance scoring based on on-time delivery, quality, and cost variance metrics. Integrates with MFG for work order synchronization, INV for consignment inventory, PROC for supplier management, QUAL for inspection standards, and GL for cost accounting.

**Bounded Context:** Outsourced Manufacturing & Contract Partner Management
**Service Name:** `contractmfg-service`
**Database:** `data/contractmfg.db`
**HTTP Port:** 8135 | **gRPC Port:** 9135

---

## 2. Database Schema

### 2.1 Contract Manufacturers
```sql
CREATE TABLE contract_manufacturers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    manufacturer_code TEXT NOT NULL,
    capabilities TEXT,                               -- JSON: manufacturing capabilities list
    certifications TEXT,                             -- JSON: certifications (ISO, GMP, etc.)
    capacity_units_per_month INTEGER,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    quality_rating REAL NOT NULL DEFAULT 0.0,
    on_time_delivery_rate REAL NOT NULL DEFAULT 0.0,
    preferred_status TEXT NOT NULL DEFAULT 'PROVISIONAL'
        CHECK(preferred_status IN ('PREFERRED','APPROVED','PROVISIONAL','SUSPENDED')),
    contract_start_date TEXT NOT NULL,
    contract_end_date TEXT,
    payment_terms TEXT,
    minimum_order_quantity INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, manufacturer_code)
);

CREATE INDEX idx_contract_mfg_tenant_supplier ON contract_manufacturers(tenant_id, supplier_id);
CREATE INDEX idx_contract_mfg_tenant_status ON contract_manufacturers(tenant_id, preferred_status);
CREATE INDEX idx_contract_mfg_tenant_rating ON contract_manufacturers(tenant_id, quality_rating);
CREATE INDEX idx_contract_mfg_tenant_active ON contract_manufacturers(tenant_id, is_active);
```

### 2.2 Outsourced Work Orders
```sql
CREATE TABLE outsourced_work_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manufacturer_id TEXT NOT NULL,
    internal_work_order_id TEXT,
    item_id TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    unit_of_measure TEXT NOT NULL,
    planned_start_date TEXT NOT NULL,
    planned_completion_date TEXT NOT NULL,
    actual_start_date TEXT,
    actual_completion_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','ACKNOWLEDGED','IN_PRODUCTION','COMPLETED','REJECTED','CANCELLED')),
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    quality_check_required INTEGER NOT NULL DEFAULT 1,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (manufacturer_id) REFERENCES contract_manufacturers(id)
);

CREATE INDEX idx_outsourced_wo_tenant_mfg ON outsourced_work_orders(tenant_id, manufacturer_id);
CREATE INDEX idx_outsourced_wo_tenant_status ON outsourced_work_orders(tenant_id, status);
CREATE INDEX idx_outsourced_wo_tenant_item ON outsourced_work_orders(tenant_id, item_id);
CREATE INDEX idx_outsourced_wo_tenant_dates ON outsourced_work_orders(tenant_id, planned_start_date, planned_completion_date);
CREATE INDEX idx_outsourced_wo_tenant_priority ON outsourced_work_orders(tenant_id, priority);
CREATE INDEX idx_outsourced_wo_tenant_active ON outsourced_work_orders(tenant_id, is_active);
```

### 2.3 Consignment Inventory
```sql
CREATE TABLE consignment_inventory (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manufacturer_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_location TEXT,
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_in_transit INTEGER NOT NULL DEFAULT 0,
    last_replenishment_date TEXT,
    reorder_point INTEGER NOT NULL DEFAULT 0,
    reorder_quantity INTEGER NOT NULL DEFAULT 0,
    ownership_type TEXT NOT NULL DEFAULT 'CONSIGNMENT'
        CHECK(ownership_type IN ('CONSIGNMENT','VENDOR_MANAGED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (manufacturer_id) REFERENCES contract_manufacturers(id),
    UNIQUE(tenant_id, manufacturer_id, item_id, warehouse_location)
);

CREATE INDEX idx_consignment_tenant_mfg ON consignment_inventory(tenant_id, manufacturer_id);
CREATE INDEX idx_consignment_tenant_item ON consignment_inventory(tenant_id, item_id);
CREATE INDEX idx_consignment_tenant_reorder ON consignment_inventory(tenant_id, reorder_point);
CREATE INDEX idx_consignment_tenant_active ON consignment_inventory(tenant_id, is_active);
```

### 2.4 Contract BOM
```sql
CREATE TABLE contract_bom (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    component_item_id TEXT NOT NULL,
    required_quantity INTEGER NOT NULL,
    unit_of_measure TEXT NOT NULL,
    provided_by TEXT NOT NULL DEFAULT 'CUSTOMER'
        CHECK(provided_by IN ('CUSTOMER','MANUFACTURER')),
    component_cost_cents INTEGER NOT NULL DEFAULT 0,
    yield_factor REAL NOT NULL DEFAULT 1.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_order_id) REFERENCES outsourced_work_orders(id)
);

CREATE INDEX idx_contract_bom_tenant_wo ON contract_bom(tenant_id, work_order_id);
CREATE INDEX idx_contract_bom_tenant_item ON contract_bom(tenant_id, component_item_id);
CREATE INDEX idx_contract_bom_tenant_provided ON contract_bom(tenant_id, provided_by);
CREATE INDEX idx_contract_bom_tenant_active ON contract_bom(tenant_id, is_active);
```

### 2.5 Contract Quality Inspections
```sql
CREATE TABLE contract_quality_inspections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    manufacturer_id TEXT NOT NULL,
    inspection_type TEXT NOT NULL
        CHECK(inspection_type IN ('IN_PROCESS','FINAL','SAMPLING','RECEIVING')),
    inspector_id TEXT NOT NULL,
    inspection_date TEXT NOT NULL,
    sample_size INTEGER NOT NULL DEFAULT 0,
    defects_found INTEGER NOT NULL DEFAULT 0,
    pass_fail TEXT NOT NULL DEFAULT 'PASS'
        CHECK(pass_fail IN ('PASS','FAIL','CONDITIONAL')),
    disposition TEXT NOT NULL DEFAULT 'ACCEPT'
        CHECK(disposition IN ('ACCEPT','REJECT','REWORK','SCRAP')),
    inspection_report_reference TEXT,
    non_conformances TEXT,                           -- JSON: non-conformance details

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_order_id) REFERENCES outsourced_work_orders(id),
    FOREIGN KEY (manufacturer_id) REFERENCES contract_manufacturers(id)
);

CREATE INDEX idx_quality_insp_tenant_wo ON contract_quality_inspections(tenant_id, work_order_id);
CREATE INDEX idx_quality_insp_tenant_mfg ON contract_quality_inspections(tenant_id, manufacturer_id);
CREATE INDEX idx_quality_insp_tenant_type ON contract_quality_inspections(tenant_id, inspection_type);
CREATE INDEX idx_quality_insp_tenant_result ON contract_quality_inspections(tenant_id, pass_fail);
CREATE INDEX idx_quality_insp_tenant_date ON contract_quality_inspections(tenant_id, inspection_date);
```

### 2.6 Contract Cost Tracking
```sql
CREATE TABLE contract_cost_tracking (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    manufacturer_id TEXT NOT NULL,
    cost_category TEXT NOT NULL
        CHECK(cost_category IN ('MATERIAL','LABOR','OVERHEAD','TOOLING','SETUP','SHIPPING','QUALITY')),
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    variance_percentage REAL NOT NULL DEFAULT 0.0,
    cost_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (work_order_id) REFERENCES outsourced_work_orders(id),
    FOREIGN KEY (manufacturer_id) REFERENCES contract_manufacturers(id),
    UNIQUE(tenant_id, work_order_id, cost_category)
);

CREATE INDEX idx_cost_track_tenant_wo ON contract_cost_tracking(tenant_id, work_order_id);
CREATE INDEX idx_cost_track_tenant_mfg ON contract_cost_tracking(tenant_id, manufacturer_id);
CREATE INDEX idx_cost_track_tenant_category ON contract_cost_tracking(tenant_id, cost_category);
CREATE INDEX idx_cost_track_tenant_date ON contract_cost_tracking(tenant_id, cost_date);
```

### 2.7 Manufacturer Performance
```sql
CREATE TABLE manufacturer_performance (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manufacturer_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    orders_placed INTEGER NOT NULL DEFAULT 0,
    orders_completed INTEGER NOT NULL DEFAULT 0,
    on_time_count INTEGER NOT NULL DEFAULT 0,
    quality_pass_count INTEGER NOT NULL DEFAULT 0,
    avg_lead_time_days REAL NOT NULL DEFAULT 0.0,
    avg_cost_variance_pct REAL NOT NULL DEFAULT 0.0,
    overall_score REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (manufacturer_id) REFERENCES contract_manufacturers(id),
    UNIQUE(tenant_id, manufacturer_id, period_start, period_end)
);

CREATE INDEX idx_mfg_perf_tenant_mfg ON manufacturer_performance(tenant_id, manufacturer_id);
CREATE INDEX idx_mfg_perf_tenant_score ON manufacturer_performance(tenant_id, overall_score);
CREATE INDEX idx_mfg_perf_tenant_dates ON manufacturer_performance(tenant_id, period_start, period_end);
```

---

## 3. REST API Endpoints

### 3.1 Manufacturers
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contractmfg/manufacturers` | List contract manufacturers |
| POST | `/api/v1/contractmfg/manufacturers` | Register contract manufacturer |
| GET | `/api/v1/contractmfg/manufacturers/{id}` | Get manufacturer details |
| PUT | `/api/v1/contractmfg/manufacturers/{id}` | Update manufacturer |
| POST | `/api/v1/contractmfg/manufacturers/{id}/rate` | Rate manufacturer performance |
| POST | `/api/v1/contractmfg/manufacturers/{id}/approve` | Approve manufacturer |
| POST | `/api/v1/contractmfg/manufacturers/{id}/suspend` | Suspend manufacturer |

### 3.2 Work Orders
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contractmfg/work-orders` | List outsourced work orders |
| POST | `/api/v1/contractmfg/work-orders` | Create outsourced work order |
| GET | `/api/v1/contractmfg/work-orders/{id}` | Get work order details |
| PUT | `/api/v1/contractmfg/work-orders/{id}` | Update work order |
| POST | `/api/v1/contractmfg/work-orders/{id}/submit` | Submit work order to manufacturer |
| POST | `/api/v1/contractmfg/work-orders/{id}/acknowledge` | Acknowledge work order receipt |
| POST | `/api/v1/contractmfg/work-orders/{id}/complete` | Mark work order as completed |

### 3.3 Consignment
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contractmfg/consignment/balance` | Get consignment inventory balances |
| POST | `/api/v1/contractmfg/consignment/replenish` | Replenish consignment stock |
| POST | `/api/v1/contractmfg/consignment/adjust` | Adjust consignment inventory |
| GET | `/api/v1/contractmfg/consignment/reorder-alerts` | Get items below reorder point |

### 3.4 BOM
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contractmfg/work-orders/{id}/bom` | Get contract BOM for work order |
| POST | `/api/v1/contractmfg/work-orders/{id}/bom` | Add BOM component |
| PUT | `/api/v1/contractmfg/bom/{id}` | Update BOM component |
| DELETE | `/api/v1/contractmfg/bom/{id}` | Remove BOM component |

### 3.5 Quality Inspections
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/contractmfg/inspections` | Record quality inspection |
| GET | `/api/v1/contractmfg/work-orders/{id}/inspections` | Get inspections for work order |
| GET | `/api/v1/contractmfg/inspections/{id}` | Get inspection detail |
| GET | `/api/v1/contractmfg/manufacturers/{id}/quality-summary` | Get manufacturer quality summary |

### 3.6 Cost Tracking
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/contractmfg/costs/record` | Record actual cost |
| GET | `/api/v1/contractmfg/work-orders/{id}/costs` | Get cost analysis for work order |
| GET | `/api/v1/contractmfg/costs/variance` | Get cost variance analysis |
| GET | `/api/v1/contractmfg/costs/summary` | Get cost summary across orders |

### 3.7 Performance
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/contractmfg/manufacturers/{id}/scorecard` | Get manufacturer scorecard |
| GET | `/api/v1/contractmfg/performance/trends` | Get performance trends |
| GET | `/api/v1/contractmfg/performance/comparison` | Compare manufacturer performance |

---

## 4. Business Rules

### 4.1 Manufacturer Management
1. Each contract manufacturer MUST have a unique manufacturer_code within a tenant.
2. A manufacturer with SUSPENDED preferred_status MUST NOT be assigned new work orders.
3. Manufacturer quality_rating MUST be recalculated after each quality inspection and expressed as a percentage of passing inspections.
4. The preferred_status transition MUST follow: PROVISIONAL -> APPROVED -> PREFERRED; any status MAY transition to SUSPENDED.
5. Contract end dates MUST be monitored; work orders MUST NOT be submitted to a manufacturer whose contract has expired.

### 4.2 Work Order Management
6. Outsourced work order status transitions MUST follow: DRAFT -> SUBMITTED -> ACKNOWLEDGED -> IN_PRODUCTION -> COMPLETED; REJECTED and CANCELLED are terminal states.
7. The total_cost_cents MUST equal unit_cost_cents multiplied by quantity for each work order.
8. A work order in DRAFT status MUST NOT be acknowledged or moved to IN_PRODUCTION without first being SUBMITTED.
9. Completed work orders with quality_check_required = 1 MUST have at least one quality inspection before final acceptance.

### 4.3 Consignment Inventory
10. Consignment replenishment MUST update quantity_in_transit immediately and quantity_on_hand upon confirmed delivery.
11. When quantity_on_hand for a consignment item falls below the reorder_point, the system MUST generate a replenishment alert.
12. Consignment adjustments MUST document the reason and SHOULD require supervisor approval for adjustments exceeding a configurable threshold.

### 4.4 Cost Tracking and Quality
13. Cost variance MUST be calculated as actual_cost_cents minus estimated_cost_cents; a negative variance indicates cost savings.
14. Cost variance exceeding 10% SHOULD trigger a review alert to the procurement team.
15. Quality inspections with pass_fail = FAIL MUST set disposition to REJECT or REWORK and MUST increment the manufacturer's defect count.
16. The overall_score for manufacturer performance MUST be a weighted composite of on-time delivery rate, quality pass rate, and inverse cost variance.
17. Manufacturer performance records MUST be generated at the end of each evaluation period and MUST NOT be modified after creation.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.contractmfg.v1;

service ContractManufacturingService {
    // Manufacturer management
    rpc CreateManufacturer(CreateManufacturerRequest) returns (CreateManufacturerResponse);
    rpc GetManufacturer(GetManufacturerRequest) returns (GetManufacturerResponse);
    rpc ListManufacturers(ListManufacturersRequest) returns (ListManufacturersResponse);
    rpc RateManufacturer(RateManufacturerRequest) returns (RateManufacturerResponse);

    // Work order outsourcing
    rpc CreateWorkOrder(CreateWorkOrderRequest) returns (CreateWorkOrderResponse);
    rpc GetWorkOrder(GetWorkOrderRequest) returns (GetWorkOrderResponse);
    rpc ListWorkOrders(ListWorkOrdersRequest) returns (ListWorkOrdersResponse);
    rpc SubmitWorkOrder(SubmitWorkOrderRequest) returns (SubmitWorkOrderResponse);
    rpc AcknowledgeWorkOrder(AcknowledgeWorkOrderRequest) returns (AcknowledgeWorkOrderResponse);
    rpc CompleteWorkOrder(CompleteWorkOrderRequest) returns (CompleteWorkOrderResponse);

    // Consignment inventory
    rpc GetConsignmentBalance(GetConsignmentBalanceRequest) returns (GetConsignmentBalanceResponse);
    rpc ReplenishConsignment(ReplenishConsignmentRequest) returns (ReplenishConsignmentResponse);

    // Quality inspections
    rpc RecordInspection(RecordInspectionRequest) returns (RecordInspectionResponse);
    rpc GetInspectionResults(GetInspectionResultsRequest) returns (GetInspectionResultsResponse);

    // Cost tracking
    rpc RecordCost(RecordCostRequest) returns (RecordCostResponse);
    rpc GetCostAnalysis(GetCostAnalysisRequest) returns (GetCostAnalysisResponse);

    // Performance
    rpc GetScorecard(GetScorecardRequest) returns (GetScorecardResponse);
}

message CreateManufacturerRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string manufacturer_code = 3;
    string capabilities = 4;
    string certifications = 5;
    int32 capacity_units_per_month = 6;
    int32 lead_time_days = 7;
    string contract_start_date = 8;
    string contract_end_date = 9;
    string payment_terms = 10;
    int32 minimum_order_quantity = 11;
}

message CreateManufacturerResponse {
    string manufacturer_id = 1;
    string manufacturer_code = 2;
    string preferred_status = 3;
}

message GetManufacturerRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
}

message GetManufacturerResponse {
    string manufacturer_id = 1;
    string manufacturer_code = 2;
    string supplier_id = 3;
    string preferred_status = 4;
    double quality_rating = 5;
    double on_time_delivery_rate = 6;
    string capabilities = 7;
    string certifications = 8;
    string contract_end_date = 9;
}

message ListManufacturersRequest {
    string tenant_id = 1;
    string preferred_status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListManufacturersResponse {
    repeated GetManufacturerResponse manufacturers = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RateManufacturerRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
    double quality_rating = 3;
    double on_time_delivery_rate = 4;
    string notes = 5;
}

message RateManufacturerResponse {
    string manufacturer_id = 1;
    double quality_rating = 2;
    double on_time_delivery_rate = 3;
}

message CreateWorkOrderRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
    string internal_work_order_id = 3;
    string item_id = 4;
    int32 quantity = 5;
    string unit_of_measure = 6;
    string planned_start_date = 7;
    string planned_completion_date = 8;
    int64 unit_cost_cents = 9;
    string currency_code = 10;
    bool quality_check_required = 11;
    string priority = 12;
}

message CreateWorkOrderResponse {
    string work_order_id = 1;
    string manufacturer_id = 2;
    int64 total_cost_cents = 3;
    string status = 4;
}

message GetWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
}

message GetWorkOrderResponse {
    string work_order_id = 1;
    string manufacturer_id = 2;
    string item_id = 3;
    int32 quantity = 4;
    string status = 5;
    int64 unit_cost_cents = 6;
    int64 total_cost_cents = 7;
    string planned_start_date = 8;
    string planned_completion_date = 9;
    string actual_start_date = 10;
    string actual_completion_date = 11;
    string priority = 12;
}

message ListWorkOrdersRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
    string status = 3;
    string priority = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListWorkOrdersResponse {
    repeated GetWorkOrderResponse work_orders = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
}

message SubmitWorkOrderResponse {
    string work_order_id = 1;
    string status = 2;
}

message AcknowledgeWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string acknowledged_by = 3;
}

message AcknowledgeWorkOrderResponse {
    string work_order_id = 1;
    string status = 2;
}

message CompleteWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string actual_completion_date = 3;
    int32 actual_quantity = 4;
}

message CompleteWorkOrderResponse {
    string work_order_id = 1;
    string status = 2;
    string actual_completion_date = 3;
}

message GetConsignmentBalanceRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
    string item_id = 3;
}

message GetConsignmentBalanceResponse {
    repeated ConsignmentBalanceInfo balances = 1;
}

message ConsignmentBalanceInfo {
    string item_id = 1;
    string manufacturer_id = 2;
    string warehouse_location = 3;
    int32 quantity_on_hand = 4;
    int32 quantity_reserved = 5;
    int32 quantity_in_transit = 6;
    int32 reorder_point = 7;
    string ownership_type = 8;
}

message ReplenishConsignmentRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
    string item_id = 3;
    int32 quantity = 4;
    string warehouse_location = 5;
}

message ReplenishConsignmentResponse {
    string consignment_id = 1;
    int32 quantity_in_transit = 2;
}

message RecordInspectionRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string manufacturer_id = 3;
    string inspection_type = 4;
    string inspector_id = 5;
    string inspection_date = 6;
    int32 sample_size = 7;
    int32 defects_found = 8;
    string pass_fail = 9;
    string disposition = 10;
    string non_conformances = 11;
}

message RecordInspectionResponse {
    string inspection_id = 1;
    string pass_fail = 2;
    string disposition = 3;
}

message GetInspectionResultsRequest {
    string tenant_id = 1;
    string work_order_id = 2;
}

message GetInspectionResultsResponse {
    repeated InspectionInfo inspections = 1;
}

message InspectionInfo {
    string inspection_id = 1;
    string inspection_type = 2;
    string inspection_date = 3;
    string pass_fail = 4;
    string disposition = 5;
    int32 sample_size = 6;
    int32 defects_found = 7;
}

message RecordCostRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string manufacturer_id = 3;
    string cost_category = 4;
    int64 estimated_cost_cents = 5;
    int64 actual_cost_cents = 6;
    string cost_date = 7;
}

message RecordCostResponse {
    string cost_id = 1;
    int64 variance_cents = 2;
    double variance_percentage = 3;
}

message GetCostAnalysisRequest {
    string tenant_id = 1;
    string work_order_id = 2;
}

message GetCostAnalysisResponse {
    string work_order_id = 1;
    int64 total_estimated_cents = 2;
    int64 total_actual_cents = 3;
    int64 total_variance_cents = 4;
    repeated CostCategoryInfo categories = 5;
}

message CostCategoryInfo {
    string cost_category = 1;
    int64 estimated_cents = 2;
    int64 actual_cents = 3;
    int64 variance_cents = 4;
    double variance_percentage = 5;
}

message GetScorecardRequest {
    string tenant_id = 1;
    string manufacturer_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetScorecardResponse {
    string manufacturer_id = 1;
    string period_start = 2;
    string period_end = 3;
    int32 orders_placed = 4;
    int32 orders_completed = 5;
    int32 on_time_count = 6;
    int32 quality_pass_count = 7;
    double avg_lead_time_days = 8;
    double avg_cost_variance_pct = 9;
    double overall_score = 10;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `mfg-service` | Internal work order data, BOM structures, routing operations, production schedules |
| `inv-service` | Item master data, on-hand quantities, warehouse details, inventory movements |
| `proc-service` | Supplier records for manufacturer registration, purchase order integration |
| `qual-service` | Quality inspection templates, acceptance criteria, non-conformance codes |
| `costing-service` | Standard cost data for variance analysis, cost element definitions |
| `gl-service` | GL account codes for cost posting, project accounting segments |

### Published To
| Service | Data |
|---------|------|
| `mfg-service` | Outsourced work order status updates, completion notifications |
| `inv-service` | Consignment inventory movements, material consumption at manufacturer sites |
| `proc-service` | Purchase requisitions for consignment replenishment, supplier performance data |
| `gl-service` | Cost journal entries, accrual postings for outsourced production |
| `costing-service` | Actual cost data from outsourced production for variance analysis |
| `reporting-service` | Manufacturer performance metrics, cost analysis reports, quality dashboards |
| `notification-service` | Reorder alerts, inspection failure alerts, work order status change notifications |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `contractmfg.work-order.outsourced` | `contractmfg.events` | `{ work_order_id, manufacturer_id, item_id, quantity, planned_completion_date }` | Work order outsourced to manufacturer |
| `contractmfg.consignment.replenished` | `contractmfg.events` | `{ consignment_id, manufacturer_id, item_id, quantity, warehouse_location }` | Consignment stock replenished |
| `contractmfg.inspection.completed` | `contractmfg.events` | `{ inspection_id, work_order_id, manufacturer_id, pass_fail, disposition }` | Quality inspection completed |
| `contractmfg.performance.scored` | `contractmfg.events` | `{ manufacturer_id, period_start, period_end, overall_score, on_time_rate, quality_rate }` | Manufacturer performance scored |

---

## 8. Migrations

1. V001: `contract_manufacturers`
2. V002: `outsourced_work_orders`
3. V003: `consignment_inventory`
4. V004: `contract_bom`
5. V005: `contract_quality_inspections`
6. V006: `contract_cost_tracking`
7. V007: `manufacturer_performance`
8. V008: Triggers for `updated_at`
