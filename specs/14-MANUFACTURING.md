# 14 - Manufacturing Service Specification

## 1. Domain Overview

Manufacturing manages Bills of Materials (BOM), work orders, routing, production scheduling, material consumption, and cost roll-up. Integrates with Inventory for materials and finished goods, and GL for production accounting.

**Bounded Context:** Manufacturing & Production
**Service Name:** `mfg-service`
**Database:** `data/mfg.db`
**HTTP Port:** 8023 | **gRPC Port:** 9023

---

## 2. Database Schema

### 2.1 Bills of Materials (BOM)
```sql
CREATE TABLE bills_of_materials (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bom_number TEXT NOT NULL,
    finished_item_id TEXT NOT NULL,
    bom_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(bom_type IN ('STANDARD','ALTERNATIVE','ENGINEERING')),
    description TEXT,
    quantity DECIMAL(18,4) NOT NULL DEFAULT 1,
    uom TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','OBSOLETE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, bom_number)
);

CREATE TABLE bom_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bom_id TEXT NOT NULL,
    component_item_id TEXT NOT NULL,
    quantity DECIMAL(18,4) NOT NULL,
    unit_of_measure TEXT NOT NULL,
    scrap_percent REAL DEFAULT 0,
    operation_sequence INTEGER,            -- Linked to routing operation
    is_phantom INTEGER NOT NULL DEFAULT 0, -- Phantom (sub-assembly exploded)
    substitute_item_id TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (bom_id) REFERENCES bills_of_materials(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, bom_id, component_item_id)
);
```

### 2.2 Routings (Operations)
```sql
CREATE TABLE routings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    routing_number TEXT NOT NULL,
    finished_item_id TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, routing_number)
);

CREATE TABLE routing_operations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    routing_id TEXT NOT NULL,
    operation_sequence INTEGER NOT NULL,
    operation_name TEXT NOT NULL,
    operation_type TEXT NOT NULL
        CHECK(operation_type IN ('SETUP','RUN','INSPECT','MOVE','QUEUE')),
    work_center TEXT NOT NULL,
    description TEXT,

    -- Time standards
    setup_time_hours REAL DEFAULT 0,
    run_time_hours REAL DEFAULT 0,         -- Per unit
    queue_time_hours REAL DEFAULT 0,
    move_time_hours REAL DEFAULT 0,

    -- Cost
    labor_rate_cents INTEGER DEFAULT 0,    -- Per hour
    overhead_rate_cents INTEGER DEFAULT 0, -- Per hour

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (routing_id) REFERENCES routings(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, routing_id, operation_sequence)
);
```

### 2.3 Work Orders
```sql
CREATE TABLE work_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_number TEXT NOT NULL,       -- WO-2024-00001
    finished_item_id TEXT NOT NULL,
    bom_id TEXT,
    routing_id TEXT,
    description TEXT,
    order_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(order_type IN ('STANDARD','DISASSEMBLY','REWORK','PROTOYPE')),

    -- Quantities
    quantity_ordered DECIMAL(18,4) NOT NULL,
    quantity_completed DECIMAL(18,4) NOT NULL DEFAULT 0,
    quantity_scrapped DECIMAL(18,4) NOT NULL DEFAULT 0,
    quantity_remaining DECIMAL(18,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,

    -- Dates
    order_date TEXT NOT NULL,
    scheduled_start_date TEXT,
    scheduled_end_date TEXT,
    actual_start_date TEXT,
    actual_end_date TEXT,
    due_date TEXT,

    -- Production
    warehouse_id TEXT NOT NULL,            -- Output warehouse
    work_center TEXT,
    priority TEXT DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),

    -- Costing
    estimated_material_cost_cents INTEGER NOT NULL DEFAULT 0,
    estimated_labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    estimated_overhead_cost_cents INTEGER NOT NULL DEFAULT 0,
    estimated_total_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_material_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_overhead_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_total_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','RELEASED','IN_PROGRESS','COMPLETED','CLOSED','CANCELLED')),

    -- Source reference
    source_type TEXT,                      -- 'MANUAL', 'SALES_ORDER', 'MRP'
    source_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, work_order_number)
);

CREATE INDEX idx_work_orders_tenant_item ON work_orders(tenant_id, finished_item_id);
CREATE INDEX idx_work_orders_tenant_status ON work_orders(tenant_id, status);
CREATE INDEX idx_work_orders_tenant_dates ON work_orders(tenant_id, scheduled_start_date, scheduled_end_date);
```

### 2.4 Work Order Material Requirements
```sql
CREATE TABLE work_order_materials (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    component_item_id TEXT NOT NULL,
    bom_component_id TEXT,
    quantity_required DECIMAL(18,4) NOT NULL,
    quantity_issued DECIMAL(18,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    warehouse_id TEXT,
    cost_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (work_order_id) REFERENCES work_orders(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, work_order_id, component_item_id)
);
```

### 2.5 Production Reporting
```sql
CREATE TABLE production_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    report_date TEXT NOT NULL,
    operation_sequence INTEGER,

    -- Output
    quantity_completed DECIMAL(18,4) NOT NULL DEFAULT 0,
    quantity_scrapped DECIMAL(18,4) NOT NULL DEFAULT 0,
    warehouse_id TEXT,                     -- Output warehouse

    -- Labor
    labor_hours REAL DEFAULT 0,
    labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    overhead_cost_cents INTEGER NOT NULL DEFAULT 0,

    -- Material consumption (auto-deducted from BOM)
    material_cost_cents INTEGER NOT NULL DEFAULT 0,

    notes TEXT,
    reported_by TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'POSTED'
        CHECK(status IN ('DRAFT','POSTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (work_order_id) REFERENCES work_orders(id) ON DELETE RESTRICT
);

CREATE INDEX idx_production_reports_tenant_wo ON production_reports(tenant_id, work_order_id);
```

---

## 3. REST API Endpoints

```
# BOM
GET/POST      /api/v1/mfg/boms                        Permission: mfg.boms.read/create
GET/PUT       /api/v1/mfg/boms/{id}                    Permission: mfg.boms.read/update
GET/POST      /api/v1/mfg/boms/{id}/components         Permission: mfg.boms.read/create
PUT/DELETE    /api/v1/mfg/boms/{id}/components/{cid}   Permission: mfg.boms.update

# Routings
GET/POST      /api/v1/mfg/routings                     Permission: mfg.routings.read/create
GET/PUT       /api/v1/mfg/routings/{id}                 Permission: mfg.routings.read/update
GET/POST      /api/v1/mfg/routings/{id}/operations      Permission: mfg.routings.read/create

# Work Orders
GET/POST      /api/v1/mfg/work-orders                  Permission: mfg.work-orders.read/create
GET/PUT       /api/v1/mfg/work-orders/{id}              Permission: mfg.work-orders.read/update
POST          /api/v1/mfg/work-orders/{id}/release      Permission: mfg.work-orders.update
POST          /api/v1/mfg/work-orders/{id}/start        Permission: mfg.work-orders.update
POST          /api/v1/mfg/work-orders/{id}/complete     Permission: mfg.work-orders.update
POST          /api/v1/mfg/work-orders/{id}/close        Permission: mfg.work-orders.update
POST          /api/v1/mfg/work-orders/{id}/cancel       Permission: mfg.work-orders.update
GET           /api/v1/mfg/work-orders/{id}/materials    Permission: mfg.work-orders.read

# Production Reporting
POST          /api/v1/mfg/production/report             Permission: mfg.production.create
GET           /api/v1/mfg/production/history            Permission: mfg.production.read

# Cost Roll-up
POST          /api/v1/mfg/cost-rollup/{item_id}         Permission: mfg.costing.update
GET           /api/v1/mfg/cost-rollup/{item_id}         Permission: mfg.costing.read

# Reports
GET           /api/v1/mfg/reports/production-summary    Permission: mfg.reports.view
GET           /api/v1/mfg/reports/work-in-progress      Permission: mfg.reports.view
GET           /api/v1/mfg/reports/variance-analysis     Permission: mfg.reports.view
```

---

## 4. Business Rules

### 4.1 BOM Rules
- Circular references MUST NOT be allowed (component cannot contain its parent)
- Phantom items are exploded during work order creation
- BOM components reference routing operations for sequencing
- Cost roll-up: sum of component costs + routing labor + overhead

### 4.2 Work Order Processing
```
DRAFT → RELEASED → IN_PROGRESS → COMPLETED → CLOSED
```
1. **DRAFT:** Editable, materials not reserved
2. **RELEASED:** Materials reserved in inventory, cannot edit BOM
3. **IN_PROGRESS:** Material consumption starts, production reporting active
4. **COMPLETED:** All quantity produced, remaining materials returned
5. **CLOSED:** Financial settlement complete, GL entries finalized

### 4.3 Material Consumption
- On production report: auto-deduct materials per BOM ratio
- Overconsumption flagged for variance tracking
- Underconsumption: remaining stays reserved until work order close

### 4.4 Cost Calculation
- `estimated_cost = material_cost (from BOM × component cost) + labor (routing × rate) + overhead`
- `actual_cost` updated with each production report
- Variance = actual - estimated (tracked per work order)

### 4.5 GL Integration
- **Material Issue:** Debit WIP, Credit Inventory
- **Production Output:** Debit Finished Goods, Credit WIP
- **Variance:** Debit/Credit Variance Account, Credit/Debit WIP

### 4.6 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `mfg.work_order.released` | WO released | INV (reserve materials) |
| `mfg.material.issued` | Materials consumed | INV, GL |
| `mfg.production.completed` | Output reported | INV (receive goods), GL |
| `mfg.work_order.closed` | WO closed | GL (variance posting) |
