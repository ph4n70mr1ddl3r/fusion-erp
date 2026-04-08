# 09 - Fixed Assets Service Specification

## 1. Domain Overview

Fixed Assets (FA) manages the lifecycle of long-term assets: acquisition, depreciation, revaluation, transfer, and disposal. Integrates with AP for asset invoice lines and GL for depreciation journal entries.

**Bounded Context:** Fixed Asset Management & Depreciation
**Service Name:** `fa-service`
**Database:** `data/fa.db`
**HTTP Port:** 8013 | **gRPC Port:** 9013

---

## 2. Database Schema

### 2.1 Asset Categories
```sql
CREATE TABLE asset_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL,
    category_name TEXT NOT NULL,
    description TEXT,
    depreciation_method TEXT NOT NULL DEFAULT 'STRAIGHT_LINE'
        CHECK(depreciation_method IN ('STRAIGHT_LINE','DECLINING_BALANCE','DOUBLE_DECLINING','UNITS_OF_PRODUCTION','NONE')),
    useful_life_months INTEGER NOT NULL DEFAULT 60,
    residual_value_percent REAL NOT NULL DEFAULT 0,
    asset_account_id TEXT NOT NULL,        -- GL account for asset cost
    accumulated_dep_account_id TEXT NOT NULL, -- GL account for accumulated depreciation
    depreciation_expense_account_id TEXT NOT NULL, -- GL expense account
    gain_loss_account_id TEXT,             -- GL account for disposal gain/loss
    parent_category_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, category_code)
);
```

### 2.2 Assets
```sql
CREATE TABLE assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_number TEXT NOT NULL,            -- FA-00001
    asset_name TEXT NOT NULL,
    description TEXT,
    category_id TEXT NOT NULL,
    asset_type TEXT NOT NULL DEFAULT 'TANGIBLE'
        CHECK(asset_type IN ('TANGIBLE','INTANGIBLE','LEASED','CONSTRUCTION_IN_PROGRESS')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','DEPRECIATING','FULLY_DEPRECIATED','DISPOSED','WRITTEN_OFF','TRANSFERRED')),

    -- Financial
    original_cost_cents INTEGER NOT NULL DEFAULT 0,
    current_cost_cents INTEGER NOT NULL DEFAULT 0,
    salvage_value_cents INTEGER NOT NULL DEFAULT 0,
    accumulated_depreciation_cents INTEGER NOT NULL DEFAULT 0,
    net_book_value_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    revaluation_reserve_cents INTEGER NOT NULL DEFAULT 0,

    -- Depreciation
    depreciation_method TEXT NOT NULL DEFAULT 'STRAIGHT_LINE',
    useful_life_months INTEGER NOT NULL,
    remaining_life_months INTEGER,
    depreciation_rate REAL,                -- For declining balance
    units_of_production_total INTEGER,     -- For UOP method
    units_of_production_used INTEGER DEFAULT 0,

    -- Dates
    acquisition_date TEXT NOT NULL,
    depreciation_start_date TEXT,
    depreciation_end_date TEXT,
    last_depreciation_date TEXT,
    disposal_date TEXT,

    -- Location & assignment
    location_id TEXT,
    department TEXT,
    assigned_to TEXT,
    serial_number TEXT,
    tag_number TEXT,

    -- Source reference
    source_type TEXT,                      -- 'AP_INVOICE', 'MANUAL', 'CONSTRUCTION'
    source_id TEXT,

    -- Insurance
    insurance_policy TEXT,
    insured_value_cents INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (category_id) REFERENCES asset_categories(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, asset_number)
);

CREATE INDEX idx_assets_tenant_category ON assets(tenant_id, category_id);
CREATE INDEX idx_assets_tenant_status ON assets(tenant_id, status);
CREATE INDEX idx_assets_tenant_dept ON assets(tenant_id, department);
```

### 2.3 Depreciation Records
```sql
CREATE TABLE depreciation_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    fiscal_year INTEGER NOT NULL,
    depreciation_amount_cents INTEGER NOT NULL DEFAULT 0,
    accumulated_depreciation_cents INTEGER NOT NULL DEFAULT 0, -- Running total
    net_book_value_cents INTEGER NOT NULL DEFAULT 0,
    units_produced INTEGER,
    method TEXT NOT NULL,
    depreciation_run_id TEXT,
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (asset_id) REFERENCES assets(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, asset_id, period_name)
);

CREATE INDEX idx_depreciation_tenant_asset ON depreciation_entries(tenant_id, asset_id);
CREATE INDEX idx_depreciation_tenant_period ON depreciation_entries(tenant_id, period_name);
CREATE INDEX idx_depreciation_tenant_run ON depreciation_entries(tenant_id, depreciation_run_id);
```

### 2.4 Asset Transactions (Transfers, Revaluations, Disposals)
```sql
CREATE TABLE asset_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('TRANSFER','REVALUATION','ADJUSTMENT','DISPOSAL_SALE','DISPOSAL_SCRAP','WRITE_OFF')),
    transaction_date TEXT NOT NULL,
    period_name TEXT NOT NULL,

    -- Financial impact
    amount_cents INTEGER NOT NULL DEFAULT 0,
    gain_loss_cents INTEGER,               -- For disposals

    -- Transfer details
    from_location_id TEXT,
    to_location_id TEXT,
    from_department TEXT,
    to_department TEXT,

    -- Revaluation details
    revalued_cost_cents INTEGER,
    revaluation_reserve_cents INTEGER,

    -- Disposal details
    disposal_proceeds_cents INTEGER,
    disposal_method TEXT,

    description TEXT,
    reference TEXT,
    gl_journal_id TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','POSTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (asset_id) REFERENCES assets(id) ON DELETE RESTRICT
);

CREATE INDEX idx_asset_trans_tenant_asset ON asset_transactions(tenant_id, asset_id);
CREATE INDEX idx_asset_trans_tenant_type ON asset_transactions(tenant_id, transaction_type);
```

---

## 3. REST API Endpoints

```
# Asset Categories
GET/POST      /api/v1/fa/categories              Permission: fa.categories.read/create
GET/PUT       /api/v1/fa/categories/{id}          Permission: fa.categories.read/update

# Assets
GET/POST      /api/v1/fa/assets                   Permission: fa.assets.read/create
GET/PUT       /api/v1/fa/assets/{id}              Permission: fa.assets.read/update
PATCH         /api/v1/fa/assets/{id}/activate     Permission: fa.assets.update
POST          /api/v1/fa/assets/{id}/transfer     Permission: fa.assets.update
POST          /api/v1/fa/assets/{id}/revalue      Permission: fa.assets.update
POST          /api/v1/fa/assets/{id}/dispose      Permission: fa.assets.dispose
POST          /api/v1/fa/assets/{id}/write-off    Permission: fa.assets.dispose

# Depreciation
POST          /api/v1/fa/depreciation/run         Permission: fa.depreciation.run
GET           /api/v1/fa/depreciation/run/{id}    Permission: fa.depreciation.read
GET           /api/v1/fa/depreciation/entries      Permission: fa.depreciation.read

# Reports
GET           /api/v1/fa/reports/asset-register   Permission: fa.reports.view
GET           /api/v1/fa/reports/depreciation-schedule  Permission: fa.reports.view
GET           /api/v1/fa/reports/net-book-value    Permission: fa.reports.view
```

---

## 4. Business Rules

### 4.1 Depreciation Methods
- **Straight-Line:** `(cost - salvage) / useful_life_months` per month
- **Declining Balance:** `net_book_value × (rate / 12)` per month
- **Double Declining:** `net_book_value × (2 / useful_life_months)`
- **Units of Production:** `(cost - salvage) × (units_this_period / total_units)`

### 4.2 Depreciation Run
1. Select all assets with status = DEPRECIATING
2. Calculate depreciation for current period
3. Create depreciation_entries records
4. Update asset accumulated_depreciation and net_book_value
5. Create GL journal: Debit depreciation expense, Credit accumulated depreciation
6. If net_book_value <= salvage_value, transition to FULLY_DEPRECIATED

### 4.3 Disposal
- Calculate gain/loss: `proceeds - net_book_value`
- Create GL journal entries to remove asset and accumulated depreciation
- Post gain/loss to gain/loss account
- Update asset status to DISPOSED

### 4.4 Events Published
| Event | Trigger |
|-------|---------|
| `fa.asset.created` | Asset registered |
| `fa.depreciation.calculated` | Depreciation run complete |
| `fa.asset.disposed` | Asset disposed |
| `fa.asset.transferred` | Asset location/department changed |
