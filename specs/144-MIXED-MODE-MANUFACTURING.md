# 144 - Mixed-Mode Manufacturing Service Specification

## 1. Domain Overview

Mixed-Mode Manufacturing supports organizations that use a combination of discrete, process, and lean manufacturing methods within the same facility or across the enterprise. Enables work orders for discrete assembly, batch orders for formula-based production, kanban for lean pull-based replenishment, and project-driven manufacturing — all within a unified planning and execution framework. Supports flexible routing that combines discrete operations with continuous process steps, hybrid BOM/formula structures, and mixed costing methods. Integrates with Discrete Manufacturing, Process Manufacturing, Production Scheduling, and Cost Management.

**Bounded Context:** Hybrid Manufacturing Planning & Execution
**Service Name:** `mixed-mode-mfg-service`
**Database:** `data/mixed_mode_mfg.db`
**HTTP Port:** 8162 | **gRPC Port:** 9162

---

## 2. Database Schema

### 2.1 Manufacturing Modes
```sql
CREATE TABLE mmm_manufacturing_modes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mode_code TEXT NOT NULL,
    mode_name TEXT NOT NULL,
    mode_type TEXT NOT NULL CHECK(mode_type IN ('DISCRETE','PROCESS','LEAN','PROJECT','HYBRID')),
    description TEXT,
    default_routing_type TEXT NOT NULL CHECK(default_routing_type IN ('SEQUENTIAL','PARALLEL','NETWORK','FLEXIBLE')),
    bom_type TEXT NOT NULL CHECK(bom_type IN ('DISCRETE_BOM','FORMULA','KANBAN_CARD','HYBRID')),
    costing_method TEXT NOT NULL CHECK(costing_method IN ('STANDARD','ACTUAL','WEIGHTED_AVG','JOB_ORDER')),
    planning_strategy TEXT NOT NULL CHECK(planning_strategy IN ('MTO','MTS','ATO','CTO','ETO','REPETITIVE')),
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    setup_time_hours REAL NOT NULL DEFAULT 0,
    lot_size_minimum DECIMAL(18,4),
    lot_size_maximum DECIMAL(18,4),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, mode_code)
);

CREATE INDEX idx_mmm_mode_tenant ON mmm_manufacturing_modes(tenant_id, mode_type);
```

### 2.2 Hybrid Production Orders
```sql
CREATE TABLE mmm_production_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_number TEXT NOT NULL,             -- MO-2024-00001
    order_type TEXT NOT NULL CHECK(order_type IN ('WORK_ORDER','BATCH_ORDER','KANBAN_ORDER','PROJECT_ORDER','HYBRID_ORDER')),
    primary_mode_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    department_id TEXT,
    planned_quantity DECIMAL(18,4) NOT NULL,
    completed_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    scrapped_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    uom TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'CREATED'
        CHECK(status IN ('CREATED','RELEASED','IN_PROCESS','COMPLETED','CLOSED','CANCELLED')),
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    scheduled_start TEXT NOT NULL,
    scheduled_end TEXT NOT NULL,
    actual_start TEXT,
    actual_end TEXT,
    parent_order_id TEXT,                  -- For sub-assemblies in hybrid flow
    project_id TEXT,                        -- If project-driven
    sales_order_id TEXT,                    -- If order-driven
    bom_id TEXT,                            -- BOM or formula reference
    routing_id TEXT,                        -- Routing reference
    batch_size DECIMAL(18,4),              -- For process steps
    yield_pct REAL NOT NULL DEFAULT 100,
    potency_target REAL,                   -- For process manufacturing
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_mmm_order_tenant ON mmm_production_orders(tenant_id, status);
CREATE INDEX idx_mmm_order_item ON mmm_production_orders(tenant_id, item_id);
CREATE INDEX idx_mmm_order_schedule ON mmm_production_orders(tenant_id, scheduled_start);
CREATE INDEX idx_mmm_order_parent ON mmm_production_orders(parent_order_id);
```

### 2.3 Hybrid Routings
```sql
CREATE TABLE mmm_hybrid_routings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    routing_code TEXT NOT NULL,
    routing_name TEXT NOT NULL,
    item_id TEXT NOT NULL,
    routing_type TEXT NOT NULL CHECK(routing_type IN ('DISCRETE','PROCESS','HYBRID','LEAN')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, routing_code)
);

CREATE TABLE mmm_routing_operations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    routing_id TEXT NOT NULL,
    operation_seq INTEGER NOT NULL,
    operation_code TEXT NOT NULL,
    operation_name TEXT NOT NULL,
    operation_type TEXT NOT NULL CHECK(operation_type IN ('DISCRETE','PROCESS','INSPECT','MOVE','WAIT','KANBAN')),
    work_center_id TEXT NOT NULL,
    resource_id TEXT,
    setup_time_hours REAL NOT NULL DEFAULT 0,
    run_time_hours REAL NOT NULL DEFAULT 0,
    run_time_per_unit INTEGER NOT NULL DEFAULT 0, -- 0=fixed, 1=per unit
    teardown_time_hours REAL NOT NULL DEFAULT 0,
    overlap_pct REAL NOT NULL DEFAULT 0,    -- % overlap with next operation
    yield_pct REAL NOT NULL DEFAULT 100,
    batch_operation INTEGER NOT NULL DEFAULT 0,
    continuous_flow INTEGER NOT NULL DEFAULT 0,
    can_run_parallel INTEGER NOT NULL DEFAULT 0,
    prerequisite_operation_ids TEXT,        -- JSON array: for network routings

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (routing_id) REFERENCES mmm_hybrid_routings(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, routing_id, operation_seq)
);
```

### 2.4 Kanban Cards
```sql
CREATE TABLE mmm_kanban_cards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    card_number TEXT NOT NULL,
    item_id TEXT NOT NULL,
    supplying_work_center TEXT NOT NULL,
    consuming_work_center TEXT NOT NULL,
    card_quantity DECIMAL(18,4) NOT NULL,
    uom TEXT NOT NULL,
    card_type TEXT NOT NULL CHECK(card_type IN ('SINGLE_CARD','TWO_CARD','SIGNAL','ELECTRONIC')),
    signal_method TEXT NOT NULL CHECK(signal_method IN ('EMPTY_CONTAINER','QUANTITY_THRESHOLD','MANUAL','AUTOMATIC')),
    replenishment_trigger_qty DECIMAL(18,4) NOT NULL,
    max_quantity DECIMAL(18,4) NOT NULL,
    current_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'FULL'
        CHECK(status IN ('FULL','IN_USE','EMPTY','SIGNALLED','REPLENISHING')),
    last_replenished_at TEXT,
    avg_replenishment_hours REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, card_number)
);

CREATE INDEX idx_mmm_kanban_item ON mmm_kanban_cards(tenant_id, item_id, status);
CREATE INDEX idx_mmm_kanban_wc ON mmm_kanban_cards(tenant_id, supplying_work_center);
```

### 2.5 Mode Transition Rules
```sql
CREATE TABLE mmm_mode_transitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    from_mode_id TEXT NOT NULL,
    to_mode_id TEXT NOT NULL,
    trigger_condition TEXT NOT NULL CHECK(trigger_condition IN ('VOLUME_THRESHOLD','PRODUCT_CHANGE','DEMAND_PATTERN','MANUAL','SCHEDULED')),
    condition_parameters TEXT NOT NULL,      -- JSON: threshold values, schedules
    transition_lead_time_hours REAL NOT NULL DEFAULT 0,
    setup_required INTEGER NOT NULL DEFAULT 1,
    quality_check_required INTEGER NOT NULL DEFAULT 1,
    approval_required INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, from_mode_id, to_mode_id)
);
```

---

## 3. API Endpoints

### 3.1 Manufacturing Modes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mixed-mode-mfg/v1/modes` | Define manufacturing mode |
| GET | `/mixed-mode-mfg/v1/modes` | List modes |
| PUT | `/mixed-mode-mfg/v1/modes/{id}` | Update mode configuration |

### 3.2 Hybrid Production Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mixed-mode-mfg/v1/orders` | Create hybrid production order |
| GET | `/mixed-mode-mfg/v1/orders` | List orders |
| GET | `/mixed-mode-mfg/v1/orders/{id}` | Get order details |
| PUT | `/mixed-mode-mfg/v1/orders/{id}` | Update order |
| POST | `/mixed-mode-mfg/v1/orders/{id}/release` | Release to shop floor |
| POST | `/mixed-mode-mfg/v1/orders/{id}/complete` | Complete order |
| POST | `/mixed-mode-mfg/v1/orders/{id}/split` | Split into sub-orders |

### 3.3 Hybrid Routings
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mixed-mode-mfg/v1/routings` | Create hybrid routing |
| GET | `/mixed-mode-mfg/v1/routings` | List routings |
| PUT | `/mixed-mode-mfg/v1/routings/{id}` | Update routing |
| POST | `/mixed-mode-mfg/v1/routings/{id}/operations` | Add operation |
| POST | `/mixed-mode-mfg/v1/routings/{id}/validate` | Validate routing |

### 3.4 Kanban Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mixed-mode-mfg/v1/kanban-cards` | Create kanban card |
| GET | `/mixed-mode-mfg/v1/kanban-cards` | List cards |
| POST | `/mixed-mode-mfg/v1/kanban-cards/{id}/signal` | Signal replenishment |
| POST | `/mixed-mode-mfg/v1/kanban-cards/{id}/replenish` | Complete replenishment |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `mixedmode.order.created` | `{ order_id, order_type, item_id, mode }` | Hybrid order created |
| `mixedmode.order.completed` | `{ order_id, qty_completed, yield_pct }` | Order completed |
| `mixedmode.kanban.signal` | `{ card_id, item_id, quantity }` | Kanban replenishment signaled |
| `mixedmode.mode.transition` | `{ order_id, from_mode, to_mode }` | Manufacturing mode transitioned |
| `mixedmode.routing.validated` | `{ routing_id, operations_count }` | Routing validated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `supply.plan.firmed` | Supply Chain Planning | Create production orders |
| `inventory.below_reorder` | Inventory | Trigger kanban replenishment |
| `quality.inspection.failed` | Quality Management | Hold production order |
| `workorder.completed` | Discrete Mfg | Check for mode transition |

---

## 5. Business Rules

1. **Mode Selection**: Auto-select manufacturing mode based on item attributes and volume
2. **Routing Flexibility**: Hybrid routings can combine discrete and process operations in sequence
3. **Kanban Loop**: Electronic kanban cards auto-signal when quantity drops below threshold
4. **Yield Tracking**: Yield calculated at each operation; affects downstream quantities
5. **Mode Transition**: Volume-based auto-transition with manual override capability
6. **Cost Rollup**: Costs rolled up across modes using mode-specific costing method
7. **Parent-Child Orders**: Hybrid orders can be decomposed into sub-orders for each mode

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Manufacturing (14) | Discrete work order execution |
| Process Manufacturing (121) | Batch order execution |
| Production Scheduling (122) | Shop floor scheduling |
| Cost Management (48) | Multi-mode costing |
| Inventory (12) | Material availability, kanban quantities |
| Quality Management (41) | In-process quality checks |
| Supply Chain Planning (28) | Production plan input |
| Project Management (15) | Project-driven manufacturing |
