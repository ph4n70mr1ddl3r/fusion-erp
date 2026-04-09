# 143 - Backlog Management Service Specification

## 1. Domain Overview

Backlog Management provides order backlog monitoring, prioritization, allocation, and fulfillment tracking. Supports configurable prioritization rules based on customer tier, order value, margin, committed date, and custom criteria. Enables supply allocation from available inventory and planned production against backlog orders with fairness and business rule enforcement. Integrates with Order Management for sales order data, Inventory for available-to-promise, Supply Chain Planning for supply projections, and Manufacturing for production schedules.

**Bounded Context:** Order Backlog & Allocation Management
**Service Name:** `backlog-mgmt-service`
**Database:** `data/backlog_mgmt.db`
**HTTP Port:** 8161 | **gRPC Port:** 9161

---

## 2. Database Schema

### 2.1 Backlog Orders
```sql
CREATE TABLE bm_backlog_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_id TEXT NOT NULL,
    order_number TEXT NOT NULL,
    order_line_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    customer_tier TEXT NOT NULL CHECK(customer_tier IN ('PLATINUM','GOLD','SILVER','STANDARD','NEW')),
    item_id TEXT NOT NULL,
    requested_quantity DECIMAL(18,4) NOT NULL,
    allocated_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    shipped_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    open_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    unit_price_cents INTEGER NOT NULL,
    margin_pct REAL NOT NULL DEFAULT 0,
    requested_date TEXT NOT NULL,
    promised_date TEXT,
    original_promised_date TEXT,
    schedule_ship_date TEXT,
    backlog_days INTEGER NOT NULL DEFAULT 0,
    priority_score REAL NOT NULL DEFAULT 0,
    allocation_priority INTEGER NOT NULL DEFAULT 0,
    hold_reason TEXT CHECK(hold_reason IN ('CREDIT_HOLD','CUSTOMER_REQUEST','MATERIAL_SHORTAGE','CAPACITY_CONSTRAINT','QUALITY_HOLD','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'BACKLOG'
        CHECK(status IN ('BACKLOG','PARTIALLY_ALLOCATED','ALLOCATED','PARTIALLY_FULFILLED','FULFILLED','CANCELLED')),
    source_type TEXT NOT NULL CHECK(source_type IN ('STOCKOUT','LATE_SUPPLY','CAPACITY','HOLD','SPLIT')),
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_line_id)
);

CREATE INDEX idx_bm_backlog_tenant ON bm_backlog_orders(tenant_id, status);
CREATE INDEX idx_bm_backlog_customer ON bm_backlog_orders(tenant_id, customer_id);
CREATE INDEX idx_bm_backlog_priority ON bm_backlog_orders(tenant_id, priority_score DESC);
CREATE INDEX idx_bm_backlog_date ON bm_backlog_orders(tenant_id, promised_date);
CREATE INDEX idx_bm_backlog_item ON bm_backlog_orders(tenant_id, item_id, status);
```

### 2.2 Prioritization Rules
```sql
CREATE TABLE bm_priority_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('CUSTOMER_TIER','ORDER_AGE','MARGIN','VALUE','PRODUCT_CATEGORY','CUSTOM_FIELD','COMPOSITE')),
    priority INTEGER NOT NULL,              -- Execution order (1 = highest)
    conditions TEXT NOT NULL,               -- JSON: rule conditions
    weight REAL NOT NULL DEFAULT 1.0,       -- Weight in composite scoring
    score_formula TEXT NOT NULL,            -- Custom score formula
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_bm_rule_tenant ON bm_priority_rules(tenant_id, priority);
```

### 2.3 Supply Allocations
```sql
CREATE TABLE bm_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    backlog_order_id TEXT NOT NULL,
    allocation_number TEXT NOT NULL,         -- ALLOC-2024-00001
    item_id TEXT NOT NULL,
    allocated_quantity DECIMAL(18,4) NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('ON_HAND','PLANNED_ORDER','TRANSFER','PRODUCTION','PURCHASE')),
    source_id TEXT NOT NULL,                -- Reference to inventory lot, PO, work order, etc.
    source_location_id TEXT NOT NULL,
    allocated_date TEXT NOT NULL,
    expected_available_date TEXT NOT NULL,
    consumed_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','CONFIRMED','PARTIALLY_CONSUMED','CONSUMED','RELEASED','CANCELLED')),
    allocation_method TEXT NOT NULL CHECK(allocation_method IN ('AUTOMATIC','MANUAL','FAIR_SHARE','PRIORITY_BASED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (backlog_order_id) REFERENCES bm_backlog_orders(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, allocation_number)
);

CREATE INDEX idx_bm_alloc_backlog ON bm_allocations(backlog_order_id);
CREATE INDEX idx_bm_alloc_item ON bm_allocations(tenant_id, item_id, status);
CREATE INDEX idx_bm_alloc_date ON bm_allocations(tenant_id, expected_available_date);
```

### 2.4 Fair Share Rules
```sql
CREATE TABLE bm_fair_share_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    allocation_method TEXT NOT NULL CHECK(allocation_method IN ('PROPORTIONAL','EQUAL','WEIGHTED','PRIORITY_TIERED')),
    customer_tier_weights TEXT,             -- JSON: { "PLATINUM": 4, "GOLD": 3, ... }
    min_allocation_pct REAL NOT NULL DEFAULT 0,
    max_allocation_pct REAL NOT NULL DEFAULT 100,
    prioritize_by TEXT NOT NULL DEFAULT 'REQUESTED_DATE'
        CHECK(prioritize_by IN ('REQUESTED_DATE','PROMISED_DATE','CUSTOMER_TIER','ORDER_VALUE','MANUAL')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, item_id, rule_name)
);
```

### 2.5 Backlog Snapshots
```sql
CREATE TABLE bm_backlog_snapshots (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    snapshot_date TEXT NOT NULL,
    snapshot_type TEXT NOT NULL CHECK(snapshot_type IN ('DAILY','WEEKLY','ADHOC')),
    total_backlog_lines INTEGER NOT NULL DEFAULT 0,
    total_backlog_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    total_backlog_value_cents INTEGER NOT NULL DEFAULT 0,
    avg_backlog_days REAL NOT NULL DEFAULT 0,
    max_backlog_days INTEGER NOT NULL DEFAULT 0,
    lines_by_status TEXT NOT NULL,          -- JSON: status counts
    lines_by_customer_tier TEXT NOT NULL,   -- JSON: tier counts
    lines_by_item TEXT NOT NULL,            -- JSON: top items by backlog
    aging_buckets TEXT NOT NULL,            -- JSON: { "0-7": n, "8-14": n, ... }

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, snapshot_date, snapshot_type)
);

CREATE INDEX idx_bm_snapshot_tenant ON bm_backlog_snapshots(tenant_id, snapshot_date DESC);
```

---

## 3. API Endpoints

### 3.1 Backlog Orders
| Method | Path | Description |
|--------|------|-------------|
| GET | `/backlog-mgmt/v1/backlog-orders` | List backlog orders |
| GET | `/backlog-mgmt/v1/backlog-orders/{id}` | Get backlog order details |
| PUT | `/backlog-mgmt/v1/backlog-orders/{id}/priority` | Update priority manually |
| POST | `/backlog-mgmt/v1/backlog-orders/{id}/cancel` | Cancel backlog order |
| POST | `/backlog-mgmt/v1/backlog-orders/{id}/hold` | Place on hold |
| POST | `/backlog-mgmt/v1/backlog-orders/{id}/release-hold` | Release hold |

### 3.2 Prioritization
| Method | Path | Description |
|--------|------|-------------|
| POST | `/backlog-mgmt/v1/priority-rules` | Create priority rule |
| GET | `/backlog-mgmt/v1/priority-rules` | List rules |
| PUT | `/backlog-mgmt/v1/priority-rules/{id}` | Update rule |
| POST | `/backlog-mgmt/v1/priority-rules/reorder` | Reorder rule execution |
| POST | `/backlog-mgmt/v1/backlog-orders/recalculate-priorities` | Recalculate all priorities |

### 3.3 Allocation
| Method | Path | Description |
|--------|------|-------------|
| POST | `/backlog-mgmt/v1/allocations/auto` | Run auto-allocation |
| POST | `/backlog-mgmt/v1/allocations/manual` | Manual allocation |
| GET | `/backlog-mgmt/v1/allocations` | List allocations |
| GET | `/backlog-mgmt/v1/allocations/{id}` | Get allocation details |
| POST | `/backlog-mgmt/v1/allocations/{id}/cancel` | Cancel allocation |
| POST | `/backlog-mgmt/v1/allocations/rebalance` | Rebalance allocations |

### 3.4 Fair Share & Analytics
| Method | Path | Description |
|--------|------|-------------|
| POST | `/backlog-mgmt/v1/fair-share-rules` | Create fair share rule |
| GET | `/backlog-mgmt/v1/fair-share-rules` | List fair share rules |
| GET | `/backlog-mgmt/v1/dashboard` | Backlog dashboard data |
| GET | `/backlog-mgmt/v1/aging-report` | Backlog aging analysis |
| GET | `/backlog-mgmt/v1/trends` | Backlog trend data |
| POST | `/backlog-mgmt/v1/snapshots/generate` | Generate snapshot |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `backlog.order.entered` | `{ order_id, item_id, quantity, requested_date }` | New order entered backlog |
| `backlog.order.allocated` | `{ order_id, allocated_qty, source }` | Order allocated supply |
| `backlog.order.fulfilled` | `{ order_id, shipped_qty, backlog_days }` | Backlog order fulfilled |
| `backlog.allocation.shortfall` | `{ item_id, total_backlog, available_supply }` | Supply shortfall detected |
| `backlog.aging.alert` | `{ order_id, backlog_days, promised_date }` | Order aging past threshold |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `order.line.unfulfilled` | Order Management | Add to backlog |
| `inventory.received` | Inventory | Trigger allocation for pending backlog |
| `production.completed` | Manufacturing | Trigger allocation for produced items |
| `po.goods_receipt.completed` | Procurement | Trigger allocation against PO receipt |
| `order.cancelled` | Order Management | Remove from backlog |

---

## 5. Business Rules

1. **Auto-Prioritization**: Priority scores recalculated daily using active rule set
2. **Allocation Fairness**: When supply < total backlog, apply fair share rules per item
3. **Customer Tier Priority**: PLATINUM orders always allocated before STANDARD
4. **Aging Escalation**: Orders backlogged >14 days auto-escalate; >30 days to VP level
5. **Allocation Reservation**: Allocated supply is reserved; cannot be re-allocated without release
6. **Snapshot Retention**: Daily snapshots retained for 90 days; weekly for 2 years
7. **Hold Management**: Held orders excluded from auto-allocation until released

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Order Management (13) | Sales order and fulfillment data |
| Inventory (12) | On-hand and available-to-promise quantities |
| Supply Chain Planning (28) | Supply projections and planned orders |
| Manufacturing (14) | Production schedules and completions |
| Procurement (11) | Purchase order receipt dates |
| Global Order Promising (50) | ATP/CTP for realistic date promising |
| Customer Service (81) | Customer inquiry on backlog status |
| AI Agents (134) | AI-powered allocation optimization |
