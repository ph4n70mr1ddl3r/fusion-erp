# 48 - Cost Management Service Specification

## 1. Domain Overview

Cost Management provides comprehensive product costing, cost rollups across multi-level BOM structures, overhead absorption, variance analysis between standard and actual costs, and cost simulation. The service supports multiple costing methods including standard costing, actual costing (FIFO, LIFO, weighted average, moving average), and handles cost calculations for manufactured items, purchased items, and service items. Integrates with MFG for BOM cost rollups, INV for inventory valuation, PROC for purchase price variance, and GL for cost journal entries.

**Bounded Context:** Product Costing, Cost Analysis & Inventory Valuation
**Service Name:** `costing-service`
**Database:** `data/costing.db`
**HTTP Port:** 8080 | **gRPC Port:** 9080

---

## 2. Database Schema

### 2.1 Cost Types
```sql
CREATE TABLE cost_types (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    type_code TEXT NOT NULL,
    type_name TEXT NOT NULL,
    costing_method TEXT NOT NULL
        CHECK(costing_method IN ('STANDARD','ACTUAL_FIFO','ACTUAL_LIFO','WEIGHTED_AVG','MOVING_AVG')),
    description TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, type_code)
);

CREATE INDEX idx_cost_types_tenant ON cost_types(tenant_id, is_active);
```

### 2.2 Cost Elements
```sql
CREATE TABLE cost_elements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    element_code TEXT NOT NULL,
    element_name TEXT NOT NULL,
    element_type TEXT NOT NULL
        CHECK(element_type IN ('MATERIAL','LABOR','OVERHEAD','OUTSIDE_PROCESSING','SUBCONTRACTING','OTHER')),
    gl_account_id TEXT,
    description TEXT,
    is_absorbed INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, element_code)
);

CREATE INDEX idx_cost_elements_tenant ON cost_elements(tenant_id, element_type);
```

### 2.3 Item Costs
```sql
CREATE TABLE item_costs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT,
    cost_type_id TEXT NOT NULL,
    cost_element_id TEXT NOT NULL,
    cost_amount INTEGER NOT NULL DEFAULT 0,
    cost_date TEXT NOT NULL,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cost_type_id) REFERENCES cost_types(id),
    FOREIGN KEY (cost_element_id) REFERENCES cost_elements(id),
    UNIQUE(tenant_id, item_id, warehouse_id, cost_type_id, cost_element_id, effective_from)
);

CREATE INDEX idx_item_costs_tenant_item ON item_costs(tenant_id, item_id, cost_type_id);
CREATE INDEX idx_item_costs_tenant_wh ON item_costs(tenant_id, warehouse_id, cost_type_id);
CREATE INDEX idx_item_costs_effective ON item_costs(tenant_id, effective_from, effective_to);
```

### 2.4 Cost Rollups
```sql
CREATE TABLE cost_rollups (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rollup_number TEXT NOT NULL,
    cost_type_id TEXT NOT NULL,
    rollup_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED')),
    total_items_processed INTEGER NOT NULL DEFAULT 0,
    total_items_failed INTEGER NOT NULL DEFAULT 0,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cost_type_id) REFERENCES cost_types(id),
    UNIQUE(tenant_id, rollup_number)
);

CREATE INDEX idx_cost_rollups_tenant ON cost_rollups(tenant_id, status, rollup_date);
```

### 2.5 Cost Rollup Lines
```sql
CREATE TABLE cost_rollup_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rollup_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    bom_level INTEGER NOT NULL DEFAULT 0,
    cost_element_id TEXT NOT NULL,
    previous_cost INTEGER NOT NULL DEFAULT 0,
    new_cost INTEGER NOT NULL DEFAULT 0,
    cost_change INTEGER NOT NULL DEFAULT 0,
    cost_change_pct REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (rollup_id) REFERENCES cost_rollups(id),
    FOREIGN KEY (cost_element_id) REFERENCES cost_elements(id)
);

CREATE INDEX idx_rollup_lines_rollup ON cost_rollup_lines(rollup_id, item_id);
CREATE INDEX idx_rollup_lines_tenant_item ON cost_rollup_lines(tenant_id, item_id);
```

### 2.6 Overhead Rates
```sql
CREATE TABLE overhead_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rate_name TEXT NOT NULL,
    cost_element_id TEXT NOT NULL,
    overhead_type TEXT NOT NULL
        CHECK(overhead_type IN ('FIXED','VARIABLE','STEP')),
    rate_basis TEXT NOT NULL
        CHECK(rate_basis IN ('PERCENTAGE_OF_MATERIAL','PERCENTAGE_OF_LABOR','PER_UNIT','PER_LABOR_HOUR','PER_MACHINE_HOUR')),
    rate_value REAL NOT NULL,
    item_category_id TEXT,
    department_id TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cost_element_id) REFERENCES cost_elements(id),
    UNIQUE(tenant_id, rate_name)
);

CREATE INDEX idx_overhead_rates_tenant ON overhead_rates(tenant_id, effective_from);
```

### 2.7 Cost Transactions
```sql
CREATE TABLE cost_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('RECEIPT','ISSUE','TRANSFER','RETURN','ADJUSTMENT','PRODUCTION_COMPLETE','VARIANCE')),
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    quantity REAL NOT NULL,
    unit_cost INTEGER NOT NULL,
    total_cost INTEGER NOT NULL,
    source_type TEXT,
    source_id TEXT,
    transaction_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_cost_trans_tenant_item ON cost_transactions(tenant_id, item_id, transaction_date);
CREATE INDEX idx_cost_trans_tenant_type ON cost_transactions(tenant_id, transaction_type, transaction_date);
```

### 2.8 Variance Analysis
```sql
CREATE TABLE variance_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    variance_type TEXT NOT NULL
        CHECK(variance_type IN ('PURCHASE_PRICE','MATERIAL_USAGE','LABOR_RATE','LABOR_EFFICIENCY','OVERHEAD_SPENDING','OVERHEAD_EFFICIENCY','OVERHEAD_VOLUME')),
    standard_cost INTEGER NOT NULL DEFAULT 0,
    actual_cost INTEGER NOT NULL DEFAULT 0,
    variance_amount INTEGER NOT NULL DEFAULT 0,
    variance_pct REAL NOT NULL DEFAULT 0.0,
    standard_quantity REAL NOT NULL DEFAULT 0,
    actual_quantity REAL NOT NULL DEFAULT 0,
    standard_rate INTEGER NOT NULL DEFAULT 0,
    actual_rate INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, item_id, warehouse_id, period_start, period_end, variance_type)
);

CREATE INDEX idx_variance_tenant_item ON variance_analysis(tenant_id, item_id, period_start);
CREATE INDEX idx_variance_tenant_type ON variance_analysis(tenant_id, variance_type, period_start);
```

---

## 3. REST API Endpoints

### 3.1 Cost Types
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/costing/cost-types` | List cost types |
| POST | `/api/v1/costing/cost-types` | Create cost type |
| GET | `/api/v1/costing/cost-types/{id}` | Get cost type |
| PUT | `/api/v1/costing/cost-types/{id}` | Update cost type |
| DELETE | `/api/v1/costing/cost-types/{id}` | Deactivate cost type |

### 3.2 Cost Elements
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/costing/cost-elements` | List cost elements |
| POST | `/api/v1/costing/cost-elements` | Create cost element |
| GET | `/api/v1/costing/cost-elements/{id}` | Get cost element |
| PUT | `/api/v1/costing/cost-elements/{id}` | Update cost element |

### 3.3 Item Costs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/costing/item-costs` | List item costs (filter by item, warehouse, cost type) |
| POST | `/api/v1/costing/item-costs` | Create/update item cost |
| GET | `/api/v1/costing/item-costs/{item_id}` | Get item cost summary |
| POST | `/api/v1/costing/item-costs/calculate` | Calculate cost for an item |
| POST | `/api/v1/costing/item-costs/bulk-update` | Bulk update standard costs |

### 3.4 Cost Rollups
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/costing/rollups` | Initiate cost rollup |
| GET | `/api/v1/costing/rollups` | List rollup runs |
| GET | `/api/v1/costing/rollups/{id}` | Get rollup details |
| GET | `/api/v1/costing/rollups/{id}/lines` | Get rollup line details |
| POST | `/api/v1/costing/rollups/{id}/cancel` | Cancel running rollup |

### 3.5 Overhead Rates
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/costing/overhead-rates` | List overhead rates |
| POST | `/api/v1/costing/overhead-rates` | Create overhead rate |
| PUT | `/api/v1/costing/overhead-rates/{id}` | Update overhead rate |
| DELETE | `/api/v1/costing/overhead-rates/{id}` | Deactivate overhead rate |

### 3.6 Cost Transactions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/costing/transactions` | List cost transactions |
| GET | `/api/v1/costing/transactions/{id}` | Get transaction detail |
| POST | `/api/v1/costing/transactions` | Record cost transaction |

### 3.7 Variance Analysis
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/costing/variances` | List variances (filter by period, item, type) |
| POST | `/api/v1/costing/variances/calculate` | Calculate variances for a period |
| GET | `/api/v1/costing/variances/summary` | Variance summary dashboard |
| GET | `/api/v1/costing/variances/{id}` | Get variance detail |

### 3.8 Cost Simulation
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/costing/simulate` | Simulate cost change impact |
| GET | `/api/v1/costing/simulate/{id}/results` | Get simulation results |

---

## 4. Business Rules

1. Each tenant MUST define at least one cost type before recording item costs.
2. Standard cost updates MUST go through a cost rollup or explicit update process with audit trail.
3. Cost rollups MUST traverse the full BOM hierarchy and aggregate costs from lowest level upward.
4. Overhead absorption rates MUST be applied based on the configured rate basis (material %, labor %, per unit, etc.).
5. Variance analysis MUST calculate purchase price variance as `(Standard Cost - Actual Cost) * Actual Quantity`.
6. Material usage variance MUST be calculated as `(Standard Quantity - Actual Quantity) * Standard Cost`.
7. Labor efficiency variance MUST be calculated as `(Standard Hours - Actual Hours) * Standard Rate`.
8. Cost transactions MUST be immutable once created — corrections require new reversing transactions.
9. Inventory valuation MUST use the costing method defined on the item's cost type.
10. Cost rollup MUST NOT update standard costs until the rollup is explicitly approved.
11. Negative cost amounts MUST NOT be allowed for standard costs.
12. Historical cost records MUST be retained with effective_from/effective_to for audit purposes.
13. Each cost element MUST map to a GL account for journal entry generation.

---

## 5. gRPC Service Definition

```protobuf
service CostingService {
    rpc GetCostType(GetCostTypeRequest) returns (CostTypeResponse);
    rpc ListCostTypes(ListCostTypesRequest) returns (ListCostTypesResponse);
    rpc CreateCostType(CreateCostTypeRequest) returns (CostTypeResponse);

    rpc GetItemCost(GetItemCostRequest) returns (ItemCostResponse);
    rpc ListItemCosts(ListItemCostsRequest) returns (ListItemCostsResponse);
    rpc CalculateItemCost(CalculateItemCostRequest) returns (CalculateItemCostResponse);

    rpc InitiateCostRollup(InitiateRollupRequest) returns (RollupResponse);
    rpc GetRollupStatus(GetRollupStatusRequest) returns (RollupResponse);

    rpc RecordCostTransaction(RecordCostTransactionRequest) returns (CostTransactionResponse);

    rpc CalculateVariances(CalculateVariancesRequest) returns (VarianceListResponse);
    rpc GetVarianceSummary(GetVarianceSummaryRequest) returns (VarianceSummaryResponse);

    rpc SimulateCostChange(SimulateCostChangeRequest) returns (SimulationResponse);
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `mfg-service` | BOM structures, routing operations, work order completions |
| `inv-service` | Item master, on-hand quantities, warehouse details, inventory movements |
| `proc-service` | Purchase order prices, goods receipt actual costs |
| `gl-service` | GL account mappings for cost elements |
| `pricing-service` | Item prices for selling price comparison |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | Cost journal entries (inventory valuation, variance postings, WIP accounting) |
| `inv-service` | Updated item unit costs for inventory valuation |
| `mfg-service` | Production cost information, WIP valuation |
| `report-service` | Cost analysis data, variance reports |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `costing.cost-type.created` | `{cost_type_id, type_code, costing_method}` | New cost type defined |
| `costing.item-cost.updated` | `{item_id, cost_type_id, element_id, old_amount, new_amount}` | Item cost changed |
| `costing.rollup.initiated` | `{rollup_id, rollup_number, cost_type_id}` | Cost rollup started |
| `costing.rollup.completed` | `{rollup_id, items_processed, items_failed}` | Cost rollup finished |
| `costing.transaction.recorded` | `{transaction_id, item_id, type, amount}` | Cost transaction recorded |
| `costing.variance.calculated` | `{item_id, variance_type, amount, period}` | Variance calculated |
| `costing.standard-cost.changed` | `{item_id, old_cost, new_cost, change_pct}` | Standard cost updated |
| `costing.simulation.completed` | `{simulation_id, total_impact}` | Cost simulation finished |
