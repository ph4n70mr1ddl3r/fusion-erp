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

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.fa.v1;

service FixedAssetsService {
    rpc GetAsset(GetAssetRequest) returns (GetAssetResponse);
    rpc CreateAsset(CreateAssetRequest) returns (CreateAssetResponse);
    rpc CalculateDepreciation(CalculateDepreciationRequest) returns (CalculateDepreciationResponse);
    rpc GetAssetNetBookValue(GetAssetNetBookValueRequest) returns (GetAssetNetBookValueResponse);
    rpc GetDepreciationSchedule(GetDepreciationScheduleRequest) returns (GetDepreciationScheduleResponse);
}

// Asset messages
message GetAssetRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetAssetResponse {
    Asset data = 1;
}

message CreateAssetRequest {
    string tenant_id = 1;
    string asset_name = 2;
    string description = 3;
    string category_id = 4;
    string asset_type = 5;
    int64 original_cost_cents = 6;
    int64 salvage_value_cents = 7;
    string currency_code = 8;
    string depreciation_method = 9;
    int32 useful_life_months = 10;
    string acquisition_date = 11;
    string depreciation_start_date = 12;
    string location_id = 13;
    string department = 14;
    string assigned_to = 15;
    string serial_number = 16;
    string tag_number = 17;
    string source_type = 18;
    string source_id = 19;
}

message CreateAssetResponse {
    Asset data = 1;
}

message Asset {
    string id = 1;
    string tenant_id = 2;
    string asset_number = 3;
    string asset_name = 4;
    string description = 5;
    string category_id = 6;
    string asset_type = 7;
    string status = 8;
    int64 original_cost_cents = 9;
    int64 current_cost_cents = 10;
    int64 salvage_value_cents = 11;
    int64 accumulated_depreciation_cents = 12;
    int64 net_book_value_cents = 13;
    string currency_code = 14;
    int64 revaluation_reserve_cents = 15;
    string depreciation_method = 16;
    int32 useful_life_months = 17;
    int32 remaining_life_months = 18;
    double depreciation_rate = 19;
    string acquisition_date = 20;
    string depreciation_start_date = 21;
    string depreciation_end_date = 22;
    string last_depreciation_date = 23;
    string location_id = 24;
    string department = 25;
    string assigned_to = 26;
    string serial_number = 27;
    string tag_number = 28;
    string created_at = 29;
    string updated_at = 30;
}

// Depreciation messages
message CalculateDepreciationRequest {
    string tenant_id = 1;
    string period_name = 2;
    repeated string asset_ids = 3;
}

message CalculateDepreciationResponse {
    string depreciation_run_id = 1;
    int32 assets_processed = 2;
    int64 total_depreciation_cents = 3;
    int32 entries_created = 4;
}

message GetAssetNetBookValueRequest {
    string tenant_id = 1;
    string id = 2;
    string as_of_date = 3;
}

message GetAssetNetBookValueResponse {
    string asset_id = 1;
    int64 original_cost_cents = 2;
    int64 accumulated_depreciation_cents = 3;
    int64 net_book_value_cents = 4;
    string currency_code = 5;
}

message GetDepreciationScheduleRequest {
    string tenant_id = 1;
    string asset_id = 2;
    string period_from = 3;
    string period_to = 4;
}

message GetDepreciationScheduleResponse {
    repeated DepreciationEntry items = 1;
}

message DepreciationEntry {
    string id = 1;
    string tenant_id = 2;
    string asset_id = 3;
    string period_name = 4;
    int32 fiscal_year = 5;
    int64 depreciation_amount_cents = 6;
    int64 accumulated_depreciation_cents = 7;
    int64 net_book_value_cents = 8;
    string method = 9;
    string gl_journal_id = 10;
    string created_at = 11;
}
```

---

## 6. Inter-Service Integration

### 6.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| gl-service | `PostJournal` | Post depreciation, disposal, revaluation journal entries |
| ap-service | `GetInvoiceLine` | Retrieve asset source invoice details |

### 6.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| gl-service | `GetAssetValue` | Query asset values for financial reporting |
| expense-service | `GetAssetByCategory` | Query assets for expense capitalization |

---

## 7. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | asset_categories | — |
| V002 | assets | V001 |
| V003 | asset_transactions | V002 |
| V004 | depreciation_entries | V002 |
