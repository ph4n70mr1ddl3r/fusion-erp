# 121 - Process Manufacturing Specification

## 1. Domain Overview

Process Manufacturing manages formula-based production for industries such as chemicals, pharmaceuticals, food & beverage, and cosmetics. Unlike discrete manufacturing (BOM-based), process manufacturing uses formulas and recipes with ingredients measured in multiple units, produces co-products and by-products, and handles batch production with full traceability.

**Bounded Context:** Process Manufacturing & Batch Production
**Service Name:** `processmfg-service`
**Database:** `data/processmfg.db`
**HTTP Port:** 8201 | **gRPC Port:** 9201

---

## 2. Database Schema

### 2.1 Formulas
```sql
CREATE TABLE pm_formulas (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    formula_code TEXT NOT NULL,
    formula_name TEXT NOT NULL,
    description TEXT,
    formula_type TEXT NOT NULL DEFAULT 'PRODUCTION'
        CHECK(formula_type IN ('PRODUCTION','QUALITY','COSTING','TEMPLATE')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','ACTIVE','OBSOLETE')),
    base_quantity REAL NOT NULL,         -- Base yield quantity
    base_uom TEXT NOT NULL,              -- Unit of measure for base quantity
    yield_percentage REAL NOT NULL DEFAULT 100.0,
    instructions TEXT,                   -- Production instructions / recipe steps
    version_number INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, formula_code, version_number)
);

CREATE INDEX idx_pm_formulas_tenant_status ON pm_formulas(tenant_id, status);
```

### 2.2 Formula Ingredients (Inputs)
```sql
CREATE TABLE pm_formula_ingredients (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    formula_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT NOT NULL,               -- Ingredient item
    ingredient_type TEXT NOT NULL DEFAULT 'MATERIAL'
        CHECK(ingredient_type IN ('MATERIAL','PACKAGING','CONSUMABLE')),
    quantity REAL NOT NULL,
    quantity_uom TEXT NOT NULL,
    scrap_percentage REAL NOT NULL DEFAULT 0.0,
    is_phantom INTEGER NOT NULL DEFAULT 0,
    substitute_group TEXT,               -- Group ID for interchangeable ingredients
    substitute_priority INTEGER,
    cost_allocation_pct REAL,            -- Percentage of total ingredient cost
    storage_conditions TEXT,             -- JSON: temperature, humidity requirements

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (formula_id) REFERENCES pm_formulas(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, formula_id, line_number)
);
```

### 2.3 Formula Products (Outputs)
```sql
CREATE TABLE pm_formula_products (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    formula_id TEXT NOT NULL,
    product_type TEXT NOT NULL CHECK(product_type IN ('PRIMARY','CO_PRODUCT','BY_PRODUCT','WASTE')),
    item_id TEXT NOT NULL,
    quantity REAL NOT NULL,
    quantity_uom TEXT NOT NULL,
    yield_percentage REAL NOT NULL DEFAULT 100.0,
    cost_allocation_pct REAL,            -- For co-product cost sharing
    is_planned INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (formula_id) REFERENCES pm_formulas(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, formula_id, item_id, product_type)
);
```

### 2.4 Recipes (Formula + Routing Combination)
```sql
CREATE TABLE pm_recipes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    recipe_code TEXT NOT NULL,
    recipe_name TEXT NOT NULL,
    description TEXT,
    formula_id TEXT NOT NULL,
    routing_id TEXT,                      -- Manufacturing routing
    equipment_class TEXT,                 -- Required equipment type
    minimum_batch_size REAL,
    maximum_batch_size REAL,
    batch_uom TEXT NOT NULL,
    standard_batch_size REAL NOT NULL,
    setup_time_minutes INTEGER NOT NULL DEFAULT 0,
    process_time_minutes REAL NOT NULL,   -- Per unit of batch size
    cleanup_time_minutes INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','ACTIVE','OBSOLETE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (formula_id) REFERENCES pm_formulas(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, recipe_code)
);
```

### 2.5 Production Batches
```sql
CREATE TABLE pm_batches (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    batch_number TEXT NOT NULL,
    recipe_id TEXT NOT NULL,
    formula_id TEXT NOT NULL,
    work_order_id TEXT,                   -- Link to discrete manufacturing work order
    planned_quantity REAL NOT NULL,
    actual_quantity REAL,
    batch_uom TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','RELEASED','IN_PROCESS','COMPLETED','CLOSED','CANCELLED')),
    start_date TEXT NOT NULL,
    end_date TEXT,
    actual_start_date TEXT,
    actual_end_date TEXT,
    equipment_id TEXT,
    operator_id TEXT,
    lot_number TEXT,                      -- Output lot number
    parent_batch_id TEXT,                 -- For multi-stage batch production
    grade TEXT,                           -- Quality grade: A, B, C, etc.

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (recipe_id) REFERENCES pm_recipes(id) ON DELETE RESTRICT,
    FOREIGN KEY (parent_batch_id) REFERENCES pm_batches(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, batch_number)
);

CREATE INDEX idx_pm_batches_tenant_status ON pm_batches(tenant_id, status);
CREATE INDEX idx_pm_batches_tenant_dates ON pm_batches(tenant_id, start_date, end_date);
```

### 2.6 Batch Transactions
```sql
CREATE TABLE pm_batch_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    batch_id TEXT NOT NULL,
    transaction_type TEXT NOT NULL CHECK(transaction_type IN ('CONSUME','PRODUCE','SCRAP','RETURN','QUALITY_SAMPLE')),
    item_id TEXT NOT NULL,
    quantity REAL NOT NULL,
    quantity_uom TEXT NOT NULL,
    lot_number TEXT,
    sublot_number TEXT,
    storage_location TEXT,
    transaction_date TEXT NOT NULL DEFAULT (datetime('now')),
    operator_id TEXT,
    notes TEXT,
    quality_sample_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    FOREIGN KEY (batch_id) REFERENCES pm_batches(id) ON DELETE RESTRICT
);

CREATE INDEX idx_pm_batch_txns_tenant_batch ON pm_batch_transactions(tenant_id, batch_id);
CREATE INDEX idx_pm_batch_txns_tenant_item ON pm_batch_transactions(tenant_id, item_id, lot_number);
```

### 2.7 Step Instructions
```sql
CREATE TABLE pm_recipe_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    recipe_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    step_name TEXT NOT NULL,
    description TEXT NOT NULL,
    instruction_type TEXT NOT NULL CHECK(instruction_type IN ('CHARGE','MIX','HEAT','COOL','FILTER','TRANSFER','WAIT','SAMPLE','CUSTOM')),
    parameters TEXT,                      -- JSON: { "temperature": 85, "duration_min": 30, "rpm": 200 }
    duration_minutes INTEGER,
    is_quality_checkpoint INTEGER NOT NULL DEFAULT 0,
    equipment_requirement TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (recipe_id) REFERENCES pm_recipes(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, recipe_id, step_number)
);
```

---

## 3. REST API Endpoints

### 3.1 Formulas
```
GET    /api/v1/processmfg/formulas                    Permission: pm.formulas.read
GET    /api/v1/processmfg/formulas/{id}               Permission: pm.formulas.read
POST   /api/v1/processmfg/formulas                    Permission: pm.formulas.create
PUT    /api/v1/processmfg/formulas/{id}               Permission: pm.formulas.update
POST   /api/v1/processmfg/formulas/{id}/approve       Permission: pm.formulas.approve
GET    /api/v1/processmfg/formulas/{id}/ingredients   Permission: pm.formulas.read
POST   /api/v1/processmfg/formulas/{id}/ingredients   Permission: pm.formulas.update
GET    /api/v1/processmfg/formulas/{id}/products      Permission: pm.formulas.read
POST   /api/v1/processmfg/formulas/{id}/products      Permission: pm.formulas.update
```

### 3.2 Recipes
```
GET    /api/v1/processmfg/recipes                     Permission: pm.recipes.read
GET    /api/v1/processmfg/recipes/{id}                Permission: pm.recipes.read
POST   /api/v1/processmfg/recipes                     Permission: pm.recipes.create
PUT    /api/v1/processmfg/recipes/{id}                Permission: pm.recipes.update
POST   /api/v1/processmfg/recipes/{id}/approve        Permission: pm.recipes.approve
GET    /api/v1/processmfg/recipes/{id}/steps          Permission: pm.recipes.read
POST   /api/v1/processmfg/recipes/{id}/steps          Permission: pm.recipes.update
```

### 3.3 Production Batches
```
GET    /api/v1/processmfg/batches                     Permission: pm.batches.read
GET    /api/v1/processmfg/batches/{id}                Permission: pm.batches.read
POST   /api/v1/processmfg/batches                     Permission: pm.batches.create
PUT    /api/v1/processmfg/batches/{id}                Permission: pm.batches.update
POST   /api/v1/processmfg/batches/{id}/release        Permission: pm.batches.release
POST   /api/v1/processmfg/batches/{id}/start          Permission: pm.batches.start
POST   /api/v1/processmfg/batches/{id}/complete       Permission: pm.batches.complete
POST   /api/v1/processmfg/batches/{id}/close          Permission: pm.batches.close
POST   /api/v1/processmfg/batches/{id}/cancel         Permission: pm.batches.cancel
GET    /api/v1/processmfg/batches/{id}/transactions   Permission: pm.batches.read
POST   /api/v1/processmfg/batches/{id}/transactions   Permission: pm.batches.transact
```

### 3.4 Batch Traceability
```
GET    /api/v1/processmfg/traceability/lot/{lot_number}       Permission: pm.traceability.read
GET    /api/v1/processmfg/traceability/batch/{batch_id}/tree  Permission: pm.traceability.read
GET    /api/v1/processmfg/traceability/where-used/{item_id}   Permission: pm.traceability.read
```

---

## 4. Business Rules

### 4.1 Formula Rules
- Formula ingredient quantities scale proportionally to batch size
- Co-product cost allocation percentages MUST total 100%
- Formulas MUST be approved before use in production
- Formula versioning preserves full history; new versions get incremented version_number
- Ingredient substitutions MUST be within the same substitute_group

### 4.2 Batch Rules
- Batch quantity MUST be between recipe minimum and maximum batch size
- Consuming materials creates negative inventory transactions (calls inv-service)
- Producing output creates positive inventory transactions with lot numbers
- Actual quantities MUST be recorded before batch completion
- Batch closure locks all transactions and triggers cost calculation
- Lot numbers auto-generated or manually assigned, unique per tenant

### 4.3 Traceability
- Every material consumed is tracked by lot number
- Forward traceability: lot → batch → output lot → customer shipment
- Backward traceability: customer complaint → output lot → batch → ingredient lots → supplier
- Multi-stage batches linked via parent_batch_id for complete genealogy

### 4.4 Yield & Variance
- Actual yield vs planned yield calculated on batch completion
- Yield variance = (actual_quantity / planned_quantity) * 100 - 100
- Significant yield variances trigger quality investigation workflows
- Scrap is tracked separately with reason codes

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.processmfg.v1;

service ProcessManufacturingService {
    rpc GetFormula(GetFormulaRequest) returns (GetFormulaResponse);
    rpc CreateBatch(CreateBatchRequest) returns (CreateBatchResponse);
    rpc RecordBatchTransaction(RecordBatchTransactionRequest) returns (RecordBatchTransactionResponse);
    rpc CompleteBatch(CompleteBatchRequest) returns (CompleteBatchResponse);
    rpc GetLotTraceability(GetLotTraceabilityRequest) returns (GetLotTraceabilityResponse);
    rpc ScaleFormula(ScaleFormulaRequest) returns (ScaleFormulaResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **mfg-service**: Work order integration for batch production orders
- **inv-service**: Material consumption and production receipt (lot-controlled)
- **quality-service**: Quality samples, test results, grade assignment
- **costing-service**: Batch cost calculation, co-product cost allocation
- **gl-service**: Production cost posting to GL
- **planning-service**: Planned batch orders from MRP

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `pm.formula.approved` | Formula approved for production | formula_id, version |
| `pm.batch.created` | Batch planned | batch_id, recipe_id, planned_qty |
| `pm.batch.started` | Batch production started | batch_id, actual_start_date |
| `pm.batch.material.consumed` | Material consumed in batch | batch_id, item_id, lot_number, qty |
| `pm.batch.output.produced` | Output produced from batch | batch_id, item_id, lot_number, qty |
| `pm.batch.completed` | Batch finished | batch_id, actual_qty, yield_variance |
| `pm.batch.closed` | Batch closed and costed | batch_id, total_cost_cents |
| `pm.lot.created` | New lot number generated | lot_number, item_id, batch_id |

---

## 7. Migrations

### Migration Order for processmfg-service:
1. V001: `pm_formulas`
2. V002: `pm_formula_ingredients`
3. V003: `pm_formula_products`
4. V004: `pm_recipes`
5. V005: `pm_recipe_steps`
6. V006: `pm_batches`
7. V007: `pm_batch_transactions`
8. V008: Triggers for `updated_at`
9. V009: Seed data (sample formula template, standard UOM conversions)
