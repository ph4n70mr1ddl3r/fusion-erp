# 36 - Warehouse Management System (WMS) Service Specification

## 1. Domain Overview

Warehouse Management provides comprehensive warehouse operations including warehouse/zone/location setup, wave planning and release, task management (pick, put, replenish, pack, ship, count, move), inbound receiving with quality inspection, outbound shipping with carrier integration, cross-dock routing, pick strategy configuration, and material handling equipment tracking. Integrates with Inventory for on-hand and reservations, Order Management for fulfillment, Procurement for inbound receiving, Manufacturing for production material flow, and GL for warehouse-related accounting.

**Bounded Context:** Warehouse Operations & Fulfillment
**Service Name:** `wms-service`
**Database:** `data/wms.db`
**HTTP Port:** 8063 | **gRPC Port:** 9063

---

## 2. Database Schema

### 2.1 Warehouses
```sql
CREATE TABLE wms_warehouses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_code TEXT NOT NULL,
    warehouse_name TEXT NOT NULL,
    warehouse_type TEXT NOT NULL DEFAULT 'DISTRIBUTION'
        CHECK(warehouse_type IN ('DISTRIBUTION','PRODUCTION','STAGING','3PL')),
    address_id TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    capacity_cubic_meters REAL,
    operating_hours TEXT,                     -- JSON: { "mon": {"open":"08:00","close":"17:00"}, ... }
    timezone TEXT NOT NULL DEFAULT 'UTC',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, warehouse_code)
);

CREATE INDEX idx_wms_wh_tenant ON wms_warehouses(tenant_id);
```

### 2.2 Zones
```sql
CREATE TABLE wms_zones (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    zone_code TEXT NOT NULL,
    zone_name TEXT NOT NULL,
    zone_type TEXT NOT NULL
        CHECK(zone_type IN ('RECEIVING','STORAGE','PICKING','PACKING','SHIPPING','STAGING','QUARANTINE')),
    temperature_range TEXT,                   -- e.g., "-20 to -10 C"
    is_hazardous INTEGER NOT NULL DEFAULT 0,
    sequence INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (warehouse_id) REFERENCES wms_warehouses(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, warehouse_id, zone_code)
);

CREATE INDEX idx_wms_zone_tenant ON wms_zones(tenant_id);
CREATE INDEX idx_wms_zone_wh ON wms_zones(warehouse_id);
```

### 2.3 Locations (Bins)
```sql
CREATE TABLE wms_locations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    zone_id TEXT NOT NULL,
    location_code TEXT NOT NULL,
    location_type TEXT NOT NULL DEFAULT 'BIN'
        CHECK(location_type IN ('BIN','SHELF','AISLE','DOOR','STAGING')),
    row_code TEXT,
    bay_code TEXT,
    level_code TEXT,
    position_code TEXT,
    max_weight_kg REAL,
    max_volume_cubic REAL,
    is_pickable INTEGER NOT NULL DEFAULT 1,
    is_receivable INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (warehouse_id) REFERENCES wms_warehouses(id) ON DELETE CASCADE,
    FOREIGN KEY (zone_id) REFERENCES wms_zones(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, warehouse_id, location_code)
);

CREATE INDEX idx_wms_loc_tenant ON wms_locations(tenant_id);
CREATE INDEX idx_wms_loc_wh ON wms_locations(warehouse_id);
CREATE INDEX idx_wms_loc_zone ON wms_locations(zone_id);
CREATE INDEX idx_wms_loc_type ON wms_locations(tenant_id, location_type);
```

### 2.4 Waves
```sql
CREATE TABLE wms_waves (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    wave_number TEXT NOT NULL,                -- WAVE-2024-00001
    wave_type TEXT NOT NULL DEFAULT 'PICKING'
        CHECK(wave_type IN ('PICKING','REPLENISHMENT','CROSS_DOCK')),
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','RELEASED','IN_PROGRESS','COMPLETED','CANCELLED')),
    planned_date TEXT NOT NULL,
    released_at TEXT,
    completed_at TEXT,
    priority INTEGER NOT NULL DEFAULT 5,
    total_tasks INTEGER NOT NULL DEFAULT 0,
    completed_tasks INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, wave_number)
);

CREATE INDEX idx_wms_wave_tenant ON wms_waves(tenant_id);
CREATE INDEX idx_wms_wave_wh ON wms_waves(warehouse_id);
CREATE INDEX idx_wms_wave_status ON wms_waves(tenant_id, status);
CREATE INDEX idx_wms_wave_date ON wms_waves(tenant_id, planned_date);
```

### 2.5 Wave Lines
```sql
CREATE TABLE wms_wave_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    wave_id TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('SALES_ORDER','TRANSFER_ORDER','PRODUCTION')),
    source_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT NOT NULL,
    quantity_required DECIMAL(18,4) NOT NULL,
    quantity_allocated DECIMAL(18,4) NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','ALLOCATED','PICKED','PACKED','SHIPPED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (wave_id) REFERENCES wms_waves(id) ON DELETE CASCADE
);

CREATE INDEX idx_wms_wline_tenant ON wms_wave_lines(tenant_id);
CREATE INDEX idx_wms_wline_wave ON wms_wave_lines(wave_id);
CREATE INDEX idx_wms_wline_item ON wms_wave_lines(item_id);
CREATE INDEX idx_wms_wline_status ON wms_wave_lines(tenant_id, status);
```

### 2.6 Tasks
```sql
CREATE TABLE wms_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    task_type TEXT NOT NULL
        CHECK(task_type IN ('PICK','PUT','REPLENISH','PACK','SHIP','COUNT','MOVE')),
    task_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(task_status IN ('PENDING','ASSIGNED','IN_PROGRESS','COMPLETED','CANCELLED')),
    wave_id TEXT,
    assigned_to_user_id TEXT,
    from_location_id TEXT,
    to_location_id TEXT,
    item_id TEXT NOT NULL,
    lot_number TEXT,
    quantity DECIMAL(18,4) NOT NULL,
    uom TEXT NOT NULL,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    started_at TEXT,
    completed_at TEXT,
    device_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    FOREIGN KEY (warehouse_id) REFERENCES wms_warehouses(id),
    FOREIGN KEY (wave_id) REFERENCES wms_waves(id),
    FOREIGN KEY (from_location_id) REFERENCES wms_locations(id),
    FOREIGN KEY (to_location_id) REFERENCES wms_locations(id)
);

CREATE INDEX idx_wms_task_tenant ON wms_tasks(tenant_id);
CREATE INDEX idx_wms_task_wh ON wms_tasks(warehouse_id);
CREATE INDEX idx_wms_task_status ON wms_tasks(tenant_id, task_status);
CREATE INDEX idx_wms_task_user ON wms_tasks(assigned_to_user_id);
CREATE INDEX idx_wms_task_type ON wms_tasks(tenant_id, task_type);
CREATE INDEX idx_wms_task_wave ON wms_tasks(wave_id);
```

### 2.7 Pick Strategies
```sql
CREATE TABLE wms_pick_strategies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    strategy_name TEXT NOT NULL,
    strategy_type TEXT NOT NULL
        CHECK(strategy_type IN ('DISCRETE','BATCH','ZONE','WAVE','CLUSTER')),
    config_json TEXT NOT NULL,                -- JSON: strategy-specific configuration
    is_default INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (warehouse_id) REFERENCES wms_warehouses(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, warehouse_id, strategy_name)
);

CREATE INDEX idx_wms_pstrat_tenant ON wms_pick_strategies(tenant_id);
CREATE INDEX idx_wms_pstrat_wh ON wms_pick_strategies(warehouse_id);
```

### 2.8 Receipts (Inbound)
```sql
CREATE TABLE wms_receipts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    receipt_number TEXT NOT NULL,             -- RCV-2024-00001
    source_type TEXT NOT NULL
        CHECK(source_type IN ('PO','TRANSFER','RETURN')),
    source_id TEXT NOT NULL,
    supplier_id TEXT,
    dock_door_id TEXT,
    status TEXT NOT NULL DEFAULT 'EXPECTED'
        CHECK(status IN ('EXPECTED','RECEIVING','COMPLETED','SHORT','CANCELLED')),
    expected_date TEXT,
    received_date TEXT,
    total_lines INTEGER NOT NULL DEFAULT 0,
    completed_lines INTEGER NOT NULL DEFAULT 0,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, receipt_number)
);

CREATE INDEX idx_wms_rcpt_tenant ON wms_receipts(tenant_id);
CREATE INDEX idx_wms_rcpt_wh ON wms_receipts(warehouse_id);
CREATE INDEX idx_wms_rcpt_status ON wms_receipts(tenant_id, status);
CREATE INDEX idx_wms_rcpt_source ON wms_receipts(source_type, source_id);
```

### 2.9 Receipt Lines
```sql
CREATE TABLE wms_receipt_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    receipt_id TEXT NOT NULL,
    source_line_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    lot_number TEXT,
    serial_number TEXT,
    quantity_expected DECIMAL(18,4) NOT NULL,
    quantity_received DECIMAL(18,4) NOT NULL DEFAULT 0,
    quantity_damaged DECIMAL(18,4) NOT NULL DEFAULT 0,
    uom TEXT NOT NULL,
    location_id TEXT,                         -- putaway location
    quality_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(quality_status IN ('PENDING','PASS','FAIL','HOLD')),
    inspection_required INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (receipt_id) REFERENCES wms_receipts(id) ON DELETE CASCADE,
    FOREIGN KEY (location_id) REFERENCES wms_locations(id)
);

CREATE INDEX idx_wms_rline_tenant ON wms_receipt_lines(tenant_id);
CREATE INDEX idx_wms_rline_receipt ON wms_receipt_lines(receipt_id);
CREATE INDEX idx_wms_rline_item ON wms_receipt_lines(item_id);
CREATE INDEX idx_wms_rline_quality ON wms_receipt_lines(tenant_id, quality_status);
```

### 2.10 Shipments (Outbound)
```sql
CREATE TABLE wms_shipments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    shipment_number TEXT NOT NULL,            -- SHP-2024-00001
    carrier_id TEXT,
    carrier_service TEXT,
    tracking_number TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','STAGED','LOADED','SHIPPED','DELIVERED')),
    ship_to_address_id TEXT,
    total_weight_kg REAL,
    total_volume_cubic REAL,
    package_count INTEGER NOT NULL DEFAULT 0,
    shipped_at TEXT,
    delivered_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, shipment_number)
);

CREATE INDEX idx_wms_ship_tenant ON wms_shipments(tenant_id);
CREATE INDEX idx_wms_ship_wh ON wms_shipments(warehouse_id);
CREATE INDEX idx_wms_ship_status ON wms_shipments(tenant_id, status);
CREATE INDEX idx_wms_ship_tracking ON wms_shipments(tracking_number);
```

### 2.11 Cross-Dock Rules
```sql
CREATE TABLE wms_crossdock_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('DEMAND_DRIVEN','SCHEDULE_DRIVEN','SUPPLIER_DRIVEN')),
    matching_criteria TEXT NOT NULL,          -- JSON: item, supplier, order matching rules
    priority INTEGER NOT NULL DEFAULT 5,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (warehouse_id) REFERENCES wms_warehouses(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, warehouse_id, rule_name)
);

CREATE INDEX idx_wms_xdock_tenant ON wms_crossdock_rules(tenant_id);
CREATE INDEX idx_wms_xdock_wh ON wms_crossdock_rules(warehouse_id);
```

### 2.12 Equipment
```sql
CREATE TABLE wms_equipment (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    equipment_code TEXT NOT NULL,
    equipment_type TEXT NOT NULL
        CHECK(equipment_type IN ('FORKLIFT','CONVEYOR','ROBOT','SCANNER','SCALE')),
    status TEXT NOT NULL DEFAULT 'AVAILABLE'
        CHECK(status IN ('AVAILABLE','IN_USE','MAINTENANCE','OUT_OF_SERVICE')),
    assigned_location_id TEXT,
    last_maintenance_date TEXT,
    next_maintenance_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (warehouse_id) REFERENCES wms_warehouses(id),
    FOREIGN KEY (assigned_location_id) REFERENCES wms_locations(id),
    UNIQUE(tenant_id, warehouse_id, equipment_code)
);

CREATE INDEX idx_wms_equip_tenant ON wms_equipment(tenant_id);
CREATE INDEX idx_wms_equip_wh ON wms_equipment(warehouse_id);
CREATE INDEX idx_wms_equip_status ON wms_equipment(tenant_id, status);
```

---

## 3. REST API Endpoints

```
# Warehouse Setup
GET/POST      /api/v1/wms/warehouses                          Permission: wms.warehouses.read/create
GET/PUT       /api/v1/wms/warehouses/{id}                     Permission: wms.warehouses.read/update
GET/POST      /api/v1/wms/warehouses/{id}/zones               Permission: wms.zones.read/create
GET/POST      /api/v1/wms/warehouses/{id}/locations           Permission: wms.locations.read/create
PUT           /api/v1/wms/warehouses/{id}/locations/bulk      Permission: wms.locations.update

# Wave Management
GET/POST      /api/v1/wms/waves                               Permission: wms.waves.read/create
GET/PUT       /api/v1/wms/waves/{id}                          Permission: wms.waves.read/update
POST          /api/v1/wms/waves/{id}/release                  Permission: wms.waves.release
POST          /api/v1/wms/waves/{id}/plan                     Permission: wms.waves.plan
GET/POST      /api/v1/wms/waves/{id}/lines                    Permission: wms.waves.read/create

# Task Management
GET           /api/v1/wms/tasks                               Permission: wms.tasks.read
POST          /api/v1/wms/tasks/create                        Permission: wms.tasks.create
GET/PUT       /api/v1/wms/tasks/{id}                          Permission: wms.tasks.read/update
POST          /api/v1/wms/tasks/{id}/assign                   Permission: wms.tasks.assign
POST          /api/v1/wms/tasks/{id}/complete                 Permission: wms.tasks.complete
POST          /api/v1/wms/tasks/{id}/cancel                   Permission: wms.tasks.cancel
GET           /api/v1/wms/tasks/my-tasks                      Permission: wms.tasks.read

# Receiving
GET/POST      /api/v1/wms/receipts                            Permission: wms.receipts.read/create
GET/PUT       /api/v1/wms/receipts/{id}                       Permission: wms.receipts.read/update
POST          /api/v1/wms/receipts/{id}/receive-line          Permission: wms.receipts.receive
POST          /api/v1/wms/receipts/{id}/putaway               Permission: wms.receipts.putaway
POST          /api/v1/wms/receipts/{id}/complete              Permission: wms.receipts.complete

# Shipping
GET/POST      /api/v1/wms/shipments                           Permission: wms.shipments.read/create
GET/PUT       /api/v1/wms/shipments/{id}                      Permission: wms.shipments.read/update
POST          /api/v1/wms/shipments/{id}/pack                 Permission: wms.shipments.pack
POST          /api/v1/wms/shipments/{id}/stage                Permission: wms.shipments.stage
POST          /api/v1/wms/shipments/{id}/ship                 Permission: wms.shipments.ship
POST          /api/v1/wms/shipments/{id}/confirm-delivery     Permission: wms.shipments.deliver

# Pick Strategies
GET/POST      /api/v1/wms/pick-strategies                     Permission: wms.strategies.read/create
GET/PUT       /api/v1/wms/pick-strategies/{id}                Permission: wms.strategies.read/update

# Cross-Dock
GET/POST      /api/v1/wms/crossdock/rules                     Permission: wms.crossdock.read/create
GET/PUT       /api/v1/wms/crossdock/rules/{id}                Permission: wms.crossdock.read/update

# Equipment
GET/POST      /api/v1/wms/equipment                           Permission: wms.equipment.read/create
GET/PUT       /api/v1/wms/equipment/{id}                      Permission: wms.equipment.read/update

# Reports
GET           /api/v1/wms/reports/warehouse-utilization       Permission: wms.reports.view
GET           /api/v1/wms/reports/pick-efficiency             Permission: wms.reports.view
GET           /api/v1/wms/reports/receiving-summary           Permission: wms.reports.view
GET           /api/v1/wms/reports/shipping-summary            Permission: wms.reports.view
GET           /api/v1/wms/reports/task-productivity           Permission: wms.reports.view
```

---

## 4. Business Rules

### 4.1 Wave Planning
```
1. Wave planning auto-groups orders by zone, priority, and carrier
2. Wave creation algorithm:
   a. Gather eligible orders/transfer orders by planned_date
   b. Group by ship-to zone, carrier, and priority
   c. Allocate inventory from pickable locations
   d. Generate wave lines for each allocated order line
   e. Create pick tasks based on configured pick strategy
3. Wave release triggers task generation and assignment
4. Wave status flow: PLANNED -> RELEASED -> IN_PROGRESS -> COMPLETED
5. A wave cannot be released if any line has zero allocation (short pick)
```

### 4.2 Pick Strategy Rules
- **Discrete:** One order = one pick trip, simplest method
- **Batch:** Multiple orders picked in one trip, sorted later
- **Zone:** Orders split by zone, each picker works one zone
- **Wave:** All orders in wave picked together by area
- **Cluster:** Multiple orders picked simultaneously using cart slots
- Pick tasks are generated based on the warehouse's configured default strategy
- Strategy can be overridden per wave at creation time

### 4.3 Receiving & Putaway
```
1. Receipt is created from PO, transfer order, or return
2. Each receipt line is received against expected quantity
3. Quantity received is validated:
   - Over-receipt tolerance checked against PO terms
   - Damaged quantity tracked separately
4. Quality inspection:
   - Items flagged inspection_required go to HOLD status
   - Inspection PASS -> available for putaway
   - Inspection FAIL -> quarantine zone
5. Putaway assignment:
   - Location capacity MUST be validated before putaway assignment
   - System suggests location based on zone rules and item velocity
   - Putaway task is created and assigned
```

### 4.4 Cross-Dock Processing
- Cross-dock eligible items bypass putaway and go directly to shipping
- Matching criteria evaluated against inbound receipt lines
- Demand-driven: match against open sales/transfer orders
- Schedule-driven: match against planned shipment windows
- Supplier-driven: match against pre-arranged supplier agreements
- Cross-dock tasks are created with highest priority

### 4.5 Task Management
- Tasks can be auto-assigned based on zone and equipment availability
- Task interleaving allows operators to perform picks and putaways in the same trip
- Task priority escalation: tasks not started within SLA get priority boost
- Count tasks support cycle counting with variance thresholds
- Move tasks support ad-hoc location-to-location transfers

### 4.6 Shipping & Fulfillment
- Shipment status flow: PLANNED -> STAGED -> LOADED -> SHIPPED -> DELIVERED
- Packing creates package records with weight and dimensions
- Staging moves packed items to staging/shipping zone
- Shipping confirmation updates OM order status and triggers invoicing
- Delivery confirmation updates tracking and closes the fulfillment loop

### 4.7 Lot & Serial Tracking
- Lot/serial tracking is enforced for items marked as lot/serial controlled
- Receipt lines must capture lot and serial numbers for controlled items
- Pick tasks specify the lot/serial to pick (FIFO/FEFO enforcement)
- Lot expiration dates are validated during pick task generation

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.wms.v1;

service WmsService {
    rpc ReserveInventory(ReserveInventoryRequest) returns (ReserveInventoryResponse);
    rpc ReleaseInventory(ReleaseInventoryRequest) returns (ReleaseInventoryResponse);
    rpc GetPickSuggestions(GetPickSuggestionsRequest) returns (GetPickSuggestionsResponse);
    rpc ConfirmShipment(ConfirmShipmentRequest) returns (ConfirmShipmentResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **INV:** Item master, on-hand quantities, lot/serial tracking flags, unit of measure
- **OM:** Sales orders for fulfillment wave creation
- **Proc:** Purchase orders for inbound receiving
- **MFG:** Production orders for material issue/receipt
- **GL:** Warehouse-related accounting entries
- **Auth:** User assignments and warehouse access permissions
- **DMS:** Attached documents for receipts and shipments

### 6.2 Data Published To
- **INV:** Inventory reservations, receipts, issues, and adjustments
- **OM:** Order fulfillment status updates, shipment confirmations
- **Proc:** Receipt confirmations against purchase orders
- **MFG:** Material issue confirmations, finished goods receipt
- **GL:** Warehouse activity cost postings
- **Reporting:** Warehouse utilization, pick efficiency, task productivity metrics

### 6.3 Events Consumed
| Event | Action |
|-------|--------|
| `om.order.confirmed` | Create wave line for fulfillment |
| `proc.po.approved` | Create expected receipt |
| `inv.transfer.created` | Create expected receipt |
| `mfg.work_order.released` | Create material issue tasks |

### 6.4 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `wms.wave.released` | Wave released for execution | OM, Reporting |
| `wms.task.completed` | Task finished (pick/put/ship) | INV, OM, Reporting |
| `wms.receipt.completed` | Inbound receipt finalized | INV, Proc, GL |
| `wms.shipment.shipped` | Shipment dispatched | OM, GL, Reporting |
| `wms.replenishment.needed` | Location stock below threshold | INV, Planning |
| `wms.crossdock.matched` | Cross-dock opportunity identified | INV, OM |

---

## 7. Migrations

1. V001: `wms_warehouses`
2. V002: `wms_zones`
3. V003: `wms_locations`
4. V004: `wms_waves`
5. V005: `wms_wave_lines`
6. V006: `wms_tasks`
7. V007: `wms_pick_strategies`
8. V008: `wms_receipts`
9. V009: `wms_receipt_lines`
10. V010: `wms_shipments`
11. V011: `wms_crossdock_rules`
12. V012: `wms_equipment`
13. V013: Indexes for all tables
14. V014: Triggers for `updated_at`
