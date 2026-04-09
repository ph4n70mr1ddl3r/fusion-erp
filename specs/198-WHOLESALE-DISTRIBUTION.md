# 198 - Wholesale Distribution Industry Solution Specification

## 1. Domain Overview

Wholesale Distribution provides industry-specific capabilities for warehouse zone management, pick wave planning, delivery route optimization, customer-specific pricing, and vendor-managed inventory. Supports configurable warehouse zones (receiving, storage, picking, packing, shipping, cold chain) with capacity tracking, pick wave planning with priority queuing and picker assignment, route planning with stop sequencing and capacity utilization, tiered customer price lists with volume breaks and contract pricing, and VMI configurations with auto-replenishment. Enables wholesale distributors to optimize warehouse operations, reduce delivery costs, and automate replenishment for key customers. Integrates with Inventory, Order Management, Shipping, Pricing, and Warehouse Management.

**Bounded Context:** Wholesale Distribution, Warehouse Operations & Route Management
**Service Name:** `wholesale-distribution-service`
**Database:** `data/wholesale_distribution.db`
**HTTP Port:** 8216 | **gRPC Port:** 9216

---

## 2. Database Schema

### 2.1 Warehouse Zones
```sql
CREATE TABLE wd_warehouse_zones (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    zone_code TEXT NOT NULL,
    zone_name TEXT NOT NULL,
    zone_type TEXT NOT NULL CHECK(zone_type IN ('RECEIVING','STORAGE','PICKING','PACKING','SHIPPING','COLD_CHAIN')),
    warehouse_id TEXT NOT NULL,
    capacity_cubic_meters REAL NOT NULL DEFAULT 0,
    current_utilization_pct REAL NOT NULL DEFAULT 0,
    temperature_range TEXT,                        -- JSON: {min_celsius, max_celsius}
    equipment_assigned TEXT,                       -- JSON: equipment IDs
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','MAINTENANCE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, zone_code)
);

CREATE INDEX idx_wd_zone_tenant ON wd_warehouse_zones(tenant_id, status);
CREATE INDEX idx_wd_zone_warehouse ON wd_warehouse_zones(warehouse_id);
CREATE INDEX idx_wd_zone_type ON wd_warehouse_zones(zone_type);
```

### 2.2 Pick Wave Plans
```sql
CREATE TABLE wd_pick_wave_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    wave_number TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 5,
    cutoff_time TEXT NOT NULL,
    assigned_pickers TEXT,                         -- JSON: picker IDs
    total_lines INTEGER NOT NULL DEFAULT 0,
    picked_lines INTEGER NOT NULL DEFAULT 0,
    wave_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(wave_status IN ('PENDING','IN_PROGRESS','COMPLETED','CANCELLED')),
    started_at TEXT,
    completed_at TEXT,
    completion_pct REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, wave_number)
);

CREATE INDEX idx_wd_wave_warehouse ON wd_pick_wave_plans(warehouse_id, wave_status);
CREATE INDEX idx_wd_wave_status ON wd_pick_wave_plans(tenant_id, wave_status);
CREATE INDEX idx_wd_wave_cutoff ON wd_pick_wave_plans(cutoff_time);
```

### 2.3 Route Plans
```sql
CREATE TABLE wd_route_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    route_code TEXT NOT NULL,
    driver_id TEXT NOT NULL,
    vehicle_id TEXT NOT NULL,
    stops TEXT NOT NULL,                           -- JSON: [{stop_number, customer_id, address, delivery_window, items_count, weight_kg}]
    estimated_distance_km REAL NOT NULL DEFAULT 0,
    estimated_duration_minutes INTEGER NOT NULL DEFAULT 0,
    capacity_utilization_pct REAL NOT NULL DEFAULT 0,
    route_date TEXT NOT NULL,
    route_status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(route_status IN ('PLANNED','IN_PROGRESS','COMPLETED','EXCEPTION')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, route_code)
);

CREATE INDEX idx_wd_route_date ON wd_route_plans(route_date, route_status);
CREATE INDEX idx_wd_route_driver ON wd_route_plans(driver_id, route_date);
CREATE INDEX idx_wd_route_status ON wd_route_plans(tenant_id, route_status);
```

### 2.4 Customer Price Lists
```sql
CREATE TABLE wd_customer_price_lists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    price_list_code TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    customer_tier TEXT NOT NULL CHECK(customer_tier IN ('BRONZE','SILVER','GOLD','PLATINUM')),
    pricing_rules TEXT NOT NULL,                   -- JSON: [{sku, base_price_cents, volume_breaks, discount_pct}]
    contract_id TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, price_list_code)
);

CREATE INDEX idx_wd_price_customer ON wd_customer_price_lists(customer_id, status);
CREATE INDEX idx_wd_price_tier ON wd_customer_price_lists(customer_tier);
CREATE INDEX idx_wd_price_dates ON wd_customer_price_lists(effective_from, effective_to);
```

### 2.5 Vendor-Managed Inventory
```sql
CREATE TABLE wd_vmi_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    sku TEXT NOT NULL,
    min_level INTEGER NOT NULL DEFAULT 0,
    max_level INTEGER NOT NULL DEFAULT 0,
    reorder_point INTEGER NOT NULL DEFAULT 0,
    reorder_qty INTEGER NOT NULL DEFAULT 0,
    replenishment_frequency TEXT NOT NULL DEFAULT 'WEEKLY'
        CHECK(replenishment_frequency IN ('DAILY','WEEKLY','BIWEEKLY','MONTHLY')),
    last_replenished_at TEXT,
    last_replenished_qty INTEGER NOT NULL DEFAULT 0,
    performance_metrics TEXT,                      -- JSON: fill_rate, stockout_count
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, customer_id, sku)
);

CREATE INDEX idx_wd_vmi_customer ON wd_vmi_configs(customer_id, status);
CREATE INDEX idx_wd_vmi_sku ON wd_vmi_configs(sku);
CREATE INDEX idx_wd_vmi_frequency ON wd_vmi_configs(replenishment_frequency);
```

---

## 3. API Endpoints

### 3.1 Warehouse Zones
| Method | Path | Description |
|--------|------|-------------|
| POST | `/wholesale/v1/zones` | Create zone |
| GET | `/wholesale/v1/zones` | List zones |
| GET | `/wholesale/v1/zones/{id}` | Get zone |
| PUT | `/wholesale/v1/zones/{id}` | Update zone |

### 3.2 Pick Waves
| Method | Path | Description |
|--------|------|-------------|
| POST | `/wholesale/v1/waves` | Create pick wave |
| GET | `/wholesale/v1/waves` | List waves |
| GET | `/wholesale/v1/waves/{id}` | Get wave |
| POST | `/wholesale/v1/waves/{id}/start` | Start wave |
| POST | `/wholesale/v1/waves/{id}/complete` | Complete wave |

### 3.3 Route Planning
| Method | Path | Description |
|--------|------|-------------|
| POST | `/wholesale/v1/routes` | Plan route |
| GET | `/wholesale/v1/routes` | List routes |
| GET | `/wholesale/v1/routes/{id}` | Get route |
| POST | `/wholesale/v1/routes/{id}/dispatch` | Dispatch route |
| POST | `/wholesale/v1/routes/optimize` | Optimize routes |

### 3.4 Pricing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/wholesale/v1/price-lists` | Create price list |
| GET | `/wholesale/v1/price-lists` | List price lists |
| GET | `/wholesale/v1/price-lists/{id}` | Get price list |
| PUT | `/wholesale/v1/price-lists/{id}` | Update price list |

### 3.5 Vendor-Managed Inventory
| Method | Path | Description |
|--------|------|-------------|
| POST | `/wholesale/v1/vmi` | Create VMI config |
| GET | `/wholesale/v1/vmi` | List VMI configs |
| PUT | `/wholesale/v1/vmi/{id}` | Update VMI config |
| POST | `/wholesale/v1/vmi/refresh` | Trigger replenishment check |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `wholesale.wave.completed` | `{ wave_id, lines_picked, duration }` | Pick wave completed |
| `wholesale.route.dispatched` | `{ route_id, stops, driver }` | Route dispatched |
| `wholesale.vmi.replenished` | `{ customer_id, sku, qty }` | VMI replenishment |
| `wholesale.price.updated` | `{ price_list_id, customer_tier }` | Price list updated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `order.created` | Order Management (32) | Create pick wave |
| `inventory.threshold` | Inventory (34) | Trigger VMI replenishment |
| `shipment.delivered` | Shipping (145) | Update route status |

---

## 5. Business Rules

1. **Zone Capacity**: Alerts when zone utilization exceeds 85% threshold
2. **Wave Priority**: Higher priority waves processed first within warehouse
3. **Route Optimization**: Routes optimized for minimum distance while respecting delivery windows
4. **Tiered Pricing**: Customer tier determines baseline discount; volume breaks applied on top
5. **VMI Auto-Replenishment**: Orders auto-generated when inventory drops below reorder point
6. **Cold Chain Compliance**: Cold chain zones require temperature monitoring and logging

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Inventory (34) | Stock levels and thresholds |
| Order Management (32) | Order processing |
| Shipping (145) | Shipment and delivery |
| Pricing (78) | Base pricing engine |
| Warehouse Management (37) | Warehouse operations |
| BI Publisher (150) | Operational reporting |
