# 12 - Inventory Service Specification

## 1. Domain Overview

Inventory manages items, warehouses, stock levels, movements, costing, and physical counting. Integrates with Procurement for receiving, OM for fulfillment, Manufacturing for material consumption/output, and GL for accounting.

**Bounded Context:** Inventory & Warehouse Management
**Service Name:** `inv-service`
**Database:** `data/inv.db`
**HTTP Port:** 8021 | **gRPC Port:** 9021

---

## 2. Database Schema

### 2.1 Items
```sql
CREATE TABLE items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_code TEXT NOT NULL,
    item_name TEXT NOT NULL,
    description TEXT,
    item_type TEXT NOT NULL DEFAULT 'STOCK'
        CHECK(item_type IN ('STOCK','SERVICE','EXPENSE','ASSET','KIT','PHANTOM')),
    category_id TEXT,
    subcategory TEXT,
    uom TEXT NOT NULL,                     -- Base unit of measure: EA, KG, M, L, etc.
    secondary_uom TEXT,
    uom_conversion_factor REAL,

    -- Physical attributes
    weight_kg REAL,
    volume_l REAL,
    height_m REAL, width_m REAL, depth_m REAL,
    is_serial_tracked INTEGER NOT NULL DEFAULT 0,
    is_lot_tracked INTEGER NOT NULL DEFAULT 0,
    is_expiry_tracked INTEGER NOT NULL DEFAULT 0,
    shelf_life_days INTEGER,

    -- Costing
    costing_method TEXT NOT NULL DEFAULT 'WEIGHTED_AVG'
        CHECK(costing_method IN ('FIFO','LIFO','WEIGHTED_AVG','STANDARD')),
    standard_cost_cents INTEGER,
    last_purchase_cost_cents INTEGER,
    average_cost_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Purchasing
    lead_time_days INTEGER,
    min_order_quantity REAL,
    max_order_quantity REAL,
    reorder_point REAL,
    reorder_quantity REAL,
    preferred_supplier_id TEXT,

    -- Sales
    is_saleable INTEGER NOT NULL DEFAULT 1,
    list_price_cents INTEGER,

    -- Status
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','OBSOLETE','DISCONTINUED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, item_code)
);

CREATE INDEX idx_items_tenant_type ON items(tenant_id, item_type);
CREATE INDEX idx_items_tenant_category ON items(tenant_id, category_id);
CREATE INDEX idx_items_tenant_status ON items(tenant_id, status);
```

### 2.2 Item Categories
```sql
CREATE TABLE item_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL,
    category_name TEXT NOT NULL,
    description TEXT,
    parent_category_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_category_id) REFERENCES item_categories(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, category_code)
);
```

### 2.3 Warehouses & Locations
```sql
CREATE TABLE warehouses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_code TEXT NOT NULL,
    warehouse_name TEXT NOT NULL,
    description TEXT,
    warehouse_type TEXT DEFAULT 'GENERAL'
        CHECK(warehouse_type IN ('GENERAL','BONDED','CONSIGNMENT','VIRTUAL','IN_TRANSIT')),
    address_line1 TEXT, address_line2 TEXT,
    city TEXT, state TEXT, postal_code TEXT, country TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    manager_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, warehouse_code)
);

CREATE TABLE warehouse_locations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    location_code TEXT NOT NULL,           -- e.g., "A-01-03" (Aisle-Rack-Shelf)
    location_type TEXT DEFAULT 'STORAGE'
        CHECK(location_type IN ('RECEIVING','STORAGE','PICKING','SHIPPING','QUARANTINE','SCRAP')),
    zone TEXT,
    aisle TEXT,
    rack TEXT,
    shelf TEXT,
    bin TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, warehouse_id, location_code)
);
```

### 2.4 On-Hand Quantities
```sql
CREATE TABLE on_hand_quantities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    location_id TEXT,
    lot_number TEXT,
    serial_number TEXT,
    quantity_on_hand REAL NOT NULL DEFAULT 0,
    quantity_reserved REAL NOT NULL DEFAULT 0,
    quantity_available REAL NOT NULL DEFAULT 0,
    quantity_in_inspection REAL NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    expiry_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (item_id) REFERENCES items(id) ON DELETE RESTRICT,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, item_id, warehouse_id, location_id, lot_number, serial_number)
);

CREATE INDEX idx_on_hand_tenant_item ON on_hand_quantities(tenant_id, item_id);
CREATE INDEX idx_on_hand_tenant_warehouse ON on_hand_quantities(tenant_id, warehouse_id);
CREATE INDEX idx_on_hand_tenant_expiry ON on_hand_quantities(tenant_id, expiry_date) WHERE expiry_date IS NOT NULL;
```

### 2.5 Stock Movements
```sql
CREATE TABLE stock_movements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    movement_number TEXT NOT NULL,         -- SM-2024-00001
    movement_type TEXT NOT NULL
        CHECK(movement_type IN ('RECEIPT','ISSUE','TRANSFER','ADJUSTMENT','RETURN','SCRAP','PRODUCTION_RECEIPT','PRODUCTION_ISSUE')),
    movement_date TEXT NOT NULL,
    item_id TEXT NOT NULL,
    quantity REAL NOT NULL,
    unit_of_measure TEXT NOT NULL,
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- From/To
    from_warehouse_id TEXT,
    from_location_id TEXT,
    to_warehouse_id TEXT,
    to_location_id TEXT,

    -- References
    reference_type TEXT,                   -- 'PO_RECEIPT', 'SO_ISSUE', 'WORK_ORDER', 'MANUAL', etc.
    reference_id TEXT,
    reference_line_id TEXT,
    lot_number TEXT,
    serial_number TEXT,
    reason TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, movement_number)
);

CREATE INDEX idx_stock_mov_tenant_item ON stock_movements(tenant_id, item_id);
CREATE INDEX idx_stock_mov_tenant_date ON stock_movements(tenant_id, movement_date);
CREATE INDEX idx_stock_mov_tenant_type ON stock_movements(tenant_id, movement_type);
CREATE INDEX idx_stock_mov_tenant_ref ON stock_movements(tenant_id, reference_type, reference_id);
```

### 2.6 Cost Layers (for FIFO/LIFO)
```sql
CREATE TABLE cost_layers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    layer_date TEXT NOT NULL,
    quantity_received REAL NOT NULL,
    quantity_remaining REAL NOT NULL,
    unit_cost_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    lot_number TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (item_id) REFERENCES items(id) ON DELETE RESTRICT
);

CREATE INDEX idx_cost_layers_tenant_item ON cost_layers(tenant_id, item_id, layer_date);
```

### 2.7 Physical Count
```sql
CREATE TABLE physical_counts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    count_number TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    count_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_PROGRESS','COMPLETED','POSTED')),
    count_type TEXT DEFAULT 'FULL'
        CHECK(count_type IN ('FULL','CYCLE','ABC','SPOT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, count_number)
);

CREATE TABLE physical_count_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    physical_count_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    location_id TEXT,
    lot_number TEXT,
    system_quantity REAL NOT NULL,
    counted_quantity REAL,
    variance_quantity REAL,
    counted_by TEXT,
    counted_at TEXT,
    status TEXT DEFAULT 'PENDING' CHECK(status IN ('PENDING','COUNTED','VERIFIED','ADJUSTED')),

    FOREIGN KEY (physical_count_id) REFERENCES physical_counts(id) ON DELETE CASCADE
);
```

---

## 3. REST API Endpoints

```
# Items
GET/POST      /api/v1/inv/items                     Permission: inv.items.read/create
GET/PUT       /api/v1/inv/items/{id}                 Permission: inv.items.read/update
GET           /api/v1/inv/items/{id}/stock            Permission: inv.items.read
GET           /api/v1/inv/items/{id}/cost-history     Permission: inv.items.read

# Categories
GET/POST      /api/v1/inv/categories                 Permission: inv.categories.read/create

# Warehouses
GET/POST      /api/v1/inv/warehouses                 Permission: inv.warehouses.read/create
GET/PUT       /api/v1/inv/warehouses/{id}             Permission: inv.warehouses.read/update
GET/POST      /api/v1/inv/warehouses/{id}/locations   Permission: inv.warehouses.read/create

# Stock Movements
GET/POST      /api/v1/inv/movements                  Permission: inv.movements.read/create
GET           /api/v1/inv/movements/{id}              Permission: inv.movements.read
POST          /api/v1/inv/movements/receive           Permission: inv.movements.create
POST          /api/v1/inv/movements/issue             Permission: inv.movements.create
POST          /api/v1/inv/movements/transfer          Permission: inv.movements.create
POST          /api/v1/inv/movements/adjust            Permission: inv.movements.create

# Physical Counts
GET/POST      /api/v1/inv/physical-counts            Permission: inv.counts.read/create
POST          /api/v1/inv/physical-counts/{id}/start  Permission: inv.counts.update
POST          /api/v1/inv/physical-counts/{id}/count  Permission: inv.counts.update
POST          /api/v1/inv/physical-counts/{id}/post   Permission: inv.counts.update

# Reports
GET           /api/v1/inv/reports/stock-status        Permission: inv.reports.view
GET           /api/v1/inv/reports/stock-valuation     Permission: inv.reports.view
GET           /api/v1/inv/reports/movement-history    Permission: inv.reports.view
GET           /api/v1/inv/reports/aging               Permission: inv.reports.view
```

---

## 4. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.inv.v1;

service InventoryService {
    rpc GetItem(GetItemRequest) returns (GetItemResponse);
    rpc GetOnHandQuantity(GetOnHandQuantityRequest) returns (GetOnHandQuantityResponse);
    rpc RecordMovement(RecordMovementRequest) returns (RecordMovementResponse);
    rpc GetItemAvailability(GetItemAvailabilityRequest) returns (GetItemAvailabilityResponse);
}

message Item { string id = 1; string tenant_id = 2; string item_code = 3; string name = 4; string description = 5; string item_type = 6; string uom_code = 7; int64 unit_cost_cents = 8; string status = 9; string created_at = 10; string updated_at = 11; }
message StockMovement { string id = 1; string tenant_id = 2; string item_id = 3; string warehouse_id = 4; string movement_type = 5; int64 quantity = 6; string reference = 7; string movement_date = 8; string created_at = 9; }
message WarehouseQuantity { string warehouse_id = 1; string warehouse_name = 2; int64 on_hand = 3; int64 reserved = 4; int64 available = 5; }

message GetItemRequest { string tenant_id = 1; string id = 2; }
message GetItemResponse { Item data = 1; }
message GetOnHandQuantityRequest { string tenant_id = 1; string item_id = 2; string warehouse_id = 3; }
message GetOnHandQuantityResponse { int64 quantity = 1; int64 reserved_quantity = 2; int64 available_quantity = 3; }
message RecordMovementRequest { string tenant_id = 1; string item_id = 2; string warehouse_id = 3; string movement_type = 4; int64 quantity = 5; }
message RecordMovementResponse { StockMovement data = 1; }
message GetItemAvailabilityRequest { string tenant_id = 1; string item_id = 2; }
message GetItemAvailabilityResponse { repeated WarehouseQuantity warehouses = 1; }
```

## 5. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | items | — |
| V002 | item_categories | V001 |
| V003 | warehouses | V002 |
| V004 | warehouse_locations | V003 |
| V005 | on_hand_quantities | V004 |
| V006 | stock_movements | V005 |
| V007 | cost_layers | V006 |
| V008 | physical_counts | V007 |
| V009 | physical_count_lines | V008 |

---

## 6. Business Rules

### 7.1 Stock Movement Processing
1. Create movement record
2. Update on_hand_quantities (add to `to`, subtract from `from`)
3. Update cost layers for costing
4. If movement has GL impact, create journal entry

### 7.2 Costing Methods
- **FIFO:** Issue from oldest cost layer first
- **LIFO:** Issue from newest cost layer first
- **Weighted Average:** `new_avg = (existing_qty * existing_cost + received_qty * received_cost) / total_qty`
- **Standard:** Fixed cost per item, variances tracked separately

### 7.3 Quantity Logic
- `quantity_available = quantity_on_hand - quantity_reserved - quantity_in_inspection`
- Cannot issue more than `quantity_available`
- Receiving increments `quantity_on_hand` (and `quantity_in_inspection` if inspection required)

### 7.4 GL Integration
Stock movements create GL journals:
- **Receipt:** Debit Inventory, Credit GRNI (or Cash/AP)
- **Issue:** Debit COGS/Expense, Credit Inventory
- **Adjustment:** Debit/Credit Inventory, Credit/Debit Expense/Variance
- **Transfer:** No GL impact (same asset, different location within tenant)

### 7.5 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `inv.item.created` | Item created | — |
| `inv.stock.received` | Stock received | GL, Proc |
| `inv.stock.issued` | Stock issued | GL, OM |
| `inv.stock.adjusted` | Stock adjusted | GL |
| `inv.stock.low` | Stock below reorder point | Proc |
| `inv.count.posted` | Physical count finalized | GL |
