# 115 - Project Asset Management Service Specification

## 1. Domain Overview

The Project Asset Management service manages assets created, acquired, or constructed through projects. It tracks capital project costs and automatically capitalizes them into fixed assets upon completion. It supports CIP (Construction In Progress) accounting, asset capitalization rules, cost allocation, and integration with both Project Management and Fixed Assets modules.

**Bounded Context:** Project Asset Management & Capitalization
**Service Name:** `projectasset-service`
**Database:** `data/projectasset.db`
**HTTP Port:** 8152 | **gRPC Port:** 9152

---

## 2. Database Schema

### 2.1 Capital Projects
```sql
CREATE TABLE capital_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    project_name TEXT NOT NULL,
    project_type TEXT NOT NULL CHECK(project_type IN ('CONSTRUCTION','ACQUISITION','DEVELOPMENT','IMPROVEMENT','RESEARCH')),
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    committed_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    remaining_budget_cents INTEGER NOT NULL DEFAULT 0,
    start_date TEXT NOT NULL,
    estimated_completion TEXT,
    actual_completion TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','CANCELLED')),
    capitalization_method TEXT NOT NULL DEFAULT 'MANUAL' CHECK(capitalization_method IN ('AUTOMATIC','MANUAL','THRESHOLD_BASED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_id)
);

CREATE INDEX idx_cp_tenant_status ON capital_projects(tenant_id, status);
CREATE INDEX idx_cp_tenant_type ON capital_projects(tenant_id, project_type);
CREATE INDEX idx_cp_tenant_dates ON capital_projects(tenant_id, start_date, estimated_completion);
```

### 2.2 CIP Accounts
```sql
CREATE TABLE cip_accounts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    capital_project_id TEXT NOT NULL,
    cip_account_id TEXT NOT NULL,
    cip_account_name TEXT NOT NULL,
    total_debits_cents INTEGER NOT NULL DEFAULT 0,
    total_credits_cents INTEGER NOT NULL DEFAULT 0,
    balance_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'OPEN' CHECK(status IN ('OPEN','CAPITALIZED','PARTIALLY_CAPITALIZED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (capital_project_id) REFERENCES capital_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, cip_account_id)
);

CREATE INDEX idx_cip_tenant_project ON cip_accounts(tenant_id, capital_project_id);
CREATE INDEX idx_cip_tenant_status ON cip_accounts(tenant_id, status);
```

### 2.3 Project Asset Lines
```sql
CREATE TABLE project_asset_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    capital_project_id TEXT NOT NULL,
    cip_account_id TEXT NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('PO_RECEIPT','AP_INVOICE','PROJECT_EXPENSE','LABOR','JOURNAL')),
    source_reference TEXT,
    amount_cents INTEGER NOT NULL,
    transaction_date TEXT NOT NULL,
    description TEXT,
    capitalizable INTEGER NOT NULL DEFAULT 1,
    capitalized INTEGER NOT NULL DEFAULT 0,
    capitalization_date TEXT,
    asset_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (capital_project_id) REFERENCES capital_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_pal_tenant_project ON project_asset_lines(tenant_id, capital_project_id);
CREATE INDEX idx_pal_tenant_cip ON project_asset_lines(tenant_id, cip_account_id);
CREATE INDEX idx_pal_tenant_source ON project_asset_lines(tenant_id, source_type);
CREATE INDEX idx_pal_tenant_capitalized ON project_asset_lines(tenant_id, capitalized);
```

### 2.4 Capitalization Rules
```sql
CREATE TABLE capitalization_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    project_type TEXT NOT NULL,
    asset_category_id TEXT NOT NULL,
    depreciation_method TEXT NOT NULL,
    useful_life_months INTEGER NOT NULL,
    salvage_value_pct DECIMAL(5,2) DEFAULT 0,
    cost_threshold_cents INTEGER DEFAULT 0,
    auto_capitalize INTEGER NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_cr_tenant_type ON capitalization_rules(tenant_id, project_type);
CREATE INDEX idx_cr_tenant_active ON capitalization_rules(tenant_id, is_active);
```

### 2.5 Project Assets
```sql
CREATE TABLE project_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    capital_project_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    asset_tag TEXT NOT NULL,
    asset_name TEXT NOT NULL,
    asset_category_id TEXT NOT NULL,
    capitalized_amount_cents INTEGER NOT NULL DEFAULT 0,
    capitalization_date TEXT,
    depreciation_start_date TEXT,
    depreciation_method TEXT,
    useful_life_months INTEGER,
    salvage_value_cents INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'CIP' CHECK(status IN ('CIP','CAPITALIZED','DISPOSED','TRANSFERRED')),
    location_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (capital_project_id) REFERENCES capital_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, asset_tag)
);

CREATE INDEX idx_pa_tenant_project ON project_assets(tenant_id, capital_project_id);
CREATE INDEX idx_pa_tenant_status ON project_assets(tenant_id, status);
CREATE INDEX idx_pa_tenant_category ON project_assets(tenant_id, asset_category_id);
```

### 2.6 Asset Cost Allocations
```sql
CREATE TABLE asset_cost_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    capital_project_id TEXT NOT NULL,
    source_amount_cents INTEGER NOT NULL,
    allocation_method TEXT NOT NULL CHECK(allocation_method IN ('EQUAL','PERCENTAGE','WEIGHTED','RATIO')),
    allocations TEXT NOT NULL,  -- JSON: [{ asset_id, amount_cents, percentage }]
    allocated_by TEXT NOT NULL,
    allocated_at TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','COMPLETED','REVERSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (capital_project_id) REFERENCES capital_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_aca_tenant_project ON asset_cost_allocations(tenant_id, capital_project_id);
CREATE INDEX idx_aca_tenant_status ON asset_cost_allocations(tenant_id, status);
```

---

## 3. REST API Endpoints

### Capital Projects
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/capital-projects` | Create a capital project |
| GET | `/api/v1/capital-projects` | List capital projects |
| GET | `/api/v1/capital-projects/{id}` | Get capital project details |
| PUT | `/api/v1/capital-projects/{id}` | Update capital project |
| POST | `/api/v1/capital-projects/{id}/complete` | Complete a capital project |
| GET | `/api/v1/capital-projects/{id}/budget-vs-actual` | Get budget vs actual report |

### CIP Accounts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/capital-projects/{id}/cip-accounts` | List CIP accounts for project |
| GET | `/api/v1/cip-accounts/{id}/balance` | Get CIP account balance |
| POST | `/api/v1/cip-accounts/{id}/adjust` | Adjust CIP account balance |

### Asset Lines
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/capital-projects/{id}/asset-lines` | List asset lines for project |
| POST | `/api/v1/asset-lines/capitalize-lines` | Capitalize selected asset lines |
| POST | `/api/v1/asset-lines/{id}/reverse` | Reverse a capitalized asset line |

### Capitalization Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/capitalization-rules` | Create a capitalization rule |
| GET | `/api/v1/capitalization-rules` | List capitalization rules |
| GET | `/api/v1/capitalization-rules/{id}` | Get rule details |
| PUT | `/api/v1/capitalization-rules/{id}` | Update capitalization rule |
| POST | `/api/v1/capitalization-rules/{id}/evaluate` | Evaluate rule against a project |

### Project Assets
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/capital-projects/{id}/project-assets` | List assets for a project |
| POST | `/api/v1/project-assets/{id}/capitalize` | Capitalize a project asset |
| POST | `/api/v1/project-assets/{id}/transfer-to-asset` | Transfer to fixed asset register |
| GET | `/api/v1/project-assets/{id}/depreciation-preview` | Preview depreciation schedule |

### Allocations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/allocations` | Create a cost allocation |
| GET | `/api/v1/capital-projects/{id}/allocations` | List allocations for project |
| POST | `/api/v1/allocations/{id}/reallocate` | Reallocate costs |

---

## 4. Business Rules

1. A capital project MUST have a valid project_id from the Project Management module before creation.
2. CIP account balance MUST always equal total_debits_cents minus total_credits_cents.
3. An asset line MUST NOT be capitalized if the capitalizable flag is set to 0.
4. The system MUST enforce that total capitalized amounts across all asset lines for a CIP account MUST NOT exceed the CIP account balance.
5. Capitalization rules with auto_capitalize set to 1 MUST be evaluated automatically when a capital project status transitions to COMPLETED.
6. THRESHOLD_BASED capitalization MUST only capitalize asset lines where amount_cents meets or exceeds the cost_threshold_cents defined in the matching rule.
7. A project asset in CIP status MUST NOT have depreciation calculated until its status transitions to CAPITALIZED.
8. Cost allocations using PERCENTAGE method MUST ensure all allocation percentages sum to exactly 100.
9. The system MUST recalculate remaining_budget_cents as total_budget_cents minus actual_cost_cents whenever an asset line is added.
10. A capitalization_date on a project asset MUST NOT be set to a future date.
11. The system SHOULD validate that depreciation_start_date falls on or after the capitalization_date.
12. Reversal of a capitalized asset line MUST restore the CIP account balance and set the asset line capitalized flag back to 0.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package projectasset.v1;

service ProjectAssetManagementService {
    // Capital Projects
    rpc CreateCapitalProject(CreateCapitalProjectRequest) returns (CreateCapitalProjectResponse);
    rpc GetCapitalProject(GetCapitalProjectRequest) returns (GetCapitalProjectResponse);
    rpc ListCapitalProjects(ListCapitalProjectsRequest) returns (ListCapitalProjectsResponse);
    rpc CompleteCapitalProject(CompleteCapitalProjectRequest) returns (CompleteCapitalProjectResponse);
    rpc GetBudgetVsActual(GetBudgetVsActualRequest) returns (GetBudgetVsActualResponse);

    // CIP Accounts
    rpc GetCIPAccounts(GetCIPAccountsRequest) returns (GetCIPAccountsResponse);
    rpc GetCIPBalance(GetCIPBalanceRequest) returns (GetCIPBalanceResponse);
    rpc AdjustCIPAccount(AdjustCIPAccountRequest) returns (AdjustCIPAccountResponse);

    // Asset Lines
    rpc GetAssetLines(GetAssetLinesRequest) returns (GetAssetLinesResponse);
    rpc CapitalizeLines(CapitalizeLinesRequest) returns (CapitalizeLinesResponse);
    rpc ReverseAssetLine(ReverseAssetLineRequest) returns (ReverseAssetLineResponse);

    // Capitalization Rules
    rpc CreateCapitalizationRule(CreateCapitalizationRuleRequest) returns (CreateCapitalizationRuleResponse);
    rpc GetCapitalizationRule(GetCapitalizationRuleRequest) returns (GetCapitalizationRuleResponse);
    rpc ListCapitalizationRules(ListCapitalizationRulesRequest) returns (ListCapitalizationRulesResponse);
    rpc EvaluateRule(EvaluateRuleRequest) returns (EvaluateRuleResponse);

    // Project Assets
    rpc GetProjectAssets(GetProjectAssetsRequest) returns (GetProjectAssetsResponse);
    rpc CapitalizeAsset(CapitalizeAssetRequest) returns (CapitalizeAssetResponse);
    rpc TransferToAsset(TransferToAssetRequest) returns (TransferToAssetResponse);
    rpc GetDepreciationPreview(GetDepreciationPreviewRequest) returns (GetDepreciationPreviewResponse);

    // Allocations
    rpc CreateAllocation(CreateAllocationRequest) returns (CreateAllocationResponse);
    rpc GetAllocations(GetAllocationsRequest) returns (GetAllocationsResponse);
    rpc Reallocate(ReallocateRequest) returns (ReallocateResponse);
}

message CapitalProject {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string project_name = 4;
    string project_type = 5;
    int64 total_budget_cents = 6;
    int64 committed_cost_cents = 7;
    int64 actual_cost_cents = 8;
    int64 remaining_budget_cents = 9;
    string start_date = 10;
    string estimated_completion = 11;
    string actual_completion = 12;
    string status = 13;
    string capitalization_method = 14;
}

message CreateCapitalProjectRequest {
    string tenant_id = 1;
    string project_id = 2;
    string project_name = 3;
    string project_type = 4;
    int64 total_budget_cents = 5;
    string start_date = 6;
    string estimated_completion = 7;
    string capitalization_method = 8;
    string created_by = 9;
}

message CreateCapitalProjectResponse {
    CapitalProject capital_project = 1;
}

message GetCapitalProjectRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
}

message GetCapitalProjectResponse {
    CapitalProject capital_project = 1;
}

message ListCapitalProjectsRequest {
    string tenant_id = 1;
    string status = 2;
    string project_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCapitalProjectsResponse {
    repeated CapitalProject capital_projects = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CompleteCapitalProjectRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
    string actual_completion = 3;
    string completed_by = 4;
}

message CompleteCapitalProjectResponse {
    CapitalProject capital_project = 1;
}

message GetBudgetVsActualRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
}

message GetBudgetVsActualResponse {
    int64 total_budget_cents = 1;
    int64 committed_cost_cents = 2;
    int64 actual_cost_cents = 3;
    int64 remaining_budget_cents = 4;
    double utilization_percentage = 5;
    repeated BudgetLineItem line_items = 6;
}

message BudgetLineItem {
    string cost_element = 1;
    int64 budget_cents = 2;
    int64 actual_cents = 3;
    int64 variance_cents = 4;
}

message CIPAccount {
    string id = 1;
    string tenant_id = 2;
    string capital_project_id = 3;
    string cip_account_id = 4;
    string cip_account_name = 5;
    int64 total_debits_cents = 6;
    int64 total_credits_cents = 7;
    int64 balance_cents = 8;
    string status = 9;
}

message GetCIPAccountsRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
}

message GetCIPAccountsResponse {
    repeated CIPAccount accounts = 1;
}

message GetCIPBalanceRequest {
    string tenant_id = 1;
    string cip_account_id = 2;
}

message GetCIPBalanceResponse {
    CIPAccount account = 1;
}

message AdjustCIPAccountRequest {
    string tenant_id = 1;
    string cip_account_id = 2;
    int64 adjustment_cents = 3;
    string adjustment_reason = 4;
    string adjusted_by = 5;
}

message AdjustCIPAccountResponse {
    CIPAccount account = 1;
}

message AssetLine {
    string id = 1;
    string tenant_id = 2;
    string capital_project_id = 3;
    string cip_account_id = 4;
    string source_type = 5;
    string source_reference = 6;
    int64 amount_cents = 7;
    string transaction_date = 8;
    string description = 9;
    bool capitalizable = 10;
    bool capitalized = 11;
    string capitalization_date = 12;
    string asset_id = 13;
}

message GetAssetLinesRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
    bool capitalized_only = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetAssetLinesResponse {
    repeated AssetLine lines = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CapitalizeLinesRequest {
    string tenant_id = 1;
    repeated string asset_line_ids = 2;
    string capitalization_date = 3;
    string capitalized_by = 4;
}

message CapitalizeLinesResponse {
    repeated AssetLine capitalized_lines = 1;
    int64 total_capitalized_cents = 2;
}

message ReverseAssetLineRequest {
    string tenant_id = 1;
    string asset_line_id = 2;
    string reversal_reason = 3;
    string reversed_by = 4;
}

message ReverseAssetLineResponse {
    AssetLine asset_line = 1;
}

message CapitalizationRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string project_type = 4;
    string asset_category_id = 5;
    string depreciation_method = 6;
    int32 useful_life_months = 7;
    double salvage_value_pct = 8;
    int64 cost_threshold_cents = 9;
    bool auto_capitalize = 10;
    string effective_from = 11;
    string effective_to = 12;
    bool is_active = 13;
}

message CreateCapitalizationRuleRequest {
    string tenant_id = 1;
    string rule_name = 2;
    string project_type = 3;
    string asset_category_id = 4;
    string depreciation_method = 5;
    int32 useful_life_months = 6;
    double salvage_value_pct = 7;
    int64 cost_threshold_cents = 8;
    bool auto_capitalize = 9;
    string effective_from = 10;
    string created_by = 11;
}

message CreateCapitalizationRuleResponse {
    CapitalizationRule rule = 1;
}

message GetCapitalizationRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message GetCapitalizationRuleResponse {
    CapitalizationRule rule = 1;
}

message ListCapitalizationRulesRequest {
    string tenant_id = 1;
    string project_type = 2;
    bool active_only = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCapitalizationRulesResponse {
    repeated CapitalizationRule rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message EvaluateRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string capital_project_id = 3;
}

message EvaluateRuleResponse {
    bool is_applicable = 1;
    repeated string eligible_line_ids = 2;
    int64 total_capitalizable_cents = 3;
    string asset_category_id = 4;
}

message ProjectAsset {
    string id = 1;
    string tenant_id = 2;
    string capital_project_id = 3;
    string asset_id = 4;
    string asset_tag = 5;
    string asset_name = 6;
    string asset_category_id = 7;
    int64 capitalized_amount_cents = 8;
    string capitalization_date = 9;
    string depreciation_start_date = 10;
    string depreciation_method = 11;
    int32 useful_life_months = 12;
    int64 salvage_value_cents = 13;
    string status = 14;
    string location_id = 15;
}

message GetProjectAssetsRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetProjectAssetsResponse {
    repeated ProjectAsset assets = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CapitalizeAssetRequest {
    string tenant_id = 1;
    string project_asset_id = 2;
    string capitalization_date = 3;
    string capitalized_by = 4;
}

message CapitalizeAssetResponse {
    ProjectAsset asset = 1;
}

message TransferToAssetRequest {
    string tenant_id = 1;
    string project_asset_id = 2;
    string transferred_by = 3;
}

message TransferToAssetResponse {
    ProjectAsset asset = 1;
    string fixed_asset_id = 2;
}

message GetDepreciationPreviewRequest {
    string tenant_id = 1;
    string project_asset_id = 2;
}

message GetDepreciationPreviewResponse {
    repeated DepreciationScheduleEntry entries = 1;
    int64 total_depreciable_cents = 2;
    int64 total_depreciation_cents = 3;
    int64 salvage_value_cents = 4;
}

message DepreciationScheduleEntry {
    string period = 1;
    int64 depreciation_cents = 2;
    int64 accumulated_cents = 3;
    int64 net_book_value_cents = 4;
}

message AllocationEntry {
    string asset_id = 1;
    int64 amount_cents = 2;
    double percentage = 3;
}

message AssetCostAllocation {
    string id = 1;
    string tenant_id = 2;
    string capital_project_id = 3;
    int64 source_amount_cents = 4;
    string allocation_method = 5;
    repeated AllocationEntry allocations = 6;
    string allocated_by = 7;
    string allocated_at = 8;
    string status = 9;
}

message CreateAllocationRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
    int64 source_amount_cents = 3;
    string allocation_method = 4;
    repeated AllocationEntry allocations = 5;
    string allocated_by = 6;
}

message CreateAllocationResponse {
    AssetCostAllocation allocation = 1;
}

message GetAllocationsRequest {
    string tenant_id = 1;
    string capital_project_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetAllocationsResponse {
    repeated AssetCostAllocation allocations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReallocateRequest {
    string tenant_id = 1;
    string allocation_id = 2;
    repeated AllocationEntry new_allocations = 3;
    string reallocated_by = 4;
}

message ReallocateResponse {
    AssetCostAllocation allocation = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `project-service` | Project definitions, tasks, status | Link capital projects to PM projects |
| `fixed-assets-service` | Asset categories, depreciation methods | Define asset parameters |
| `gl-service` | CIP GL account balances | Validate CIP accounting |
| `ap-service` | Invoice data, payment details | Create asset lines from AP |
| `po-service` | Purchase order receipts | Create asset lines from PO receipts |
| `costing-service` | Labor and expense costs | Create asset lines from project costs |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `fixed-assets-service` | Capitalized assets, depreciation schedules | Add to fixed asset register |
| `gl-service` | Capitalization journal entries, CIP clearing | Post accounting entries |
| `project-service` | Capital project completion, asset tracking | Update project status |
| `reporting-service` | CIP reports, budget vs actual | Financial reporting |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `CapitalProjectCreated` | `projectasset.capital-project.created` | `{ tenant_id, capital_project_id, project_id, project_name, project_type, total_budget_cents }` | Published when a new capital project is created |
| `CIPTransactionRecorded` | `projectasset.cip-transaction.recorded` | `{ tenant_id, cip_account_id, capital_project_id, source_type, amount_cents, balance_cents }` | Published when a transaction is recorded against a CIP account |
| `AssetCapitalized` | `projectasset.asset.capitalized` | `{ tenant_id, project_asset_id, asset_id, asset_tag, capitalized_amount_cents, capitalization_date }` | Published when a project asset is capitalized |
| `CostAllocated` | `projectasset.cost.allocated` | `{ tenant_id, allocation_id, capital_project_id, source_amount_cents, allocation_method, allocation_count }` | Published when costs are allocated across assets |
