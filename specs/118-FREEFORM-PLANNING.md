# 118 - Freeform Planning (Essbase) Service Specification

## 1. Domain Overview

The Freeform Planning service provides flexible freeform budgeting and planning leveraging the multidimensional OLAP engine for ad-hoc modeling, what-if analysis, and custom financial models. It gives power users and finance analysts spreadsheet-like flexibility backed by an enterprise-grade calculation engine. Supports sandboxed planning, scenario comparison, and flexible data entry.

**Bounded Context:** Freeform Planning & Ad-Hoc Modeling
**Service Name:** `freeform-service`
**Database:** `data/freeform.db`
**HTTP Port:** 8155 | **gRPC Port:** 9155

---

## 2. Database Schema

### 2.1 Freeform Models
```sql
CREATE TABLE freeform_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL CHECK(model_type IN ('BUDGET','FORECAST','WHAT_IF','SCENARIO','CUSTOM')),
    cube_config TEXT NOT NULL,  -- JSON
    dimensions TEXT NOT NULL,  -- JSON
    default_view TEXT,
    data_entry_mode TEXT NOT NULL DEFAULT 'FREEFORM' CHECK(data_entry_mode IN ('FREEFORM','STRUCTURED','MIXED')),
    base_period TEXT NOT NULL,
    planning_horizon_months INTEGER NOT NULL DEFAULT 12,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_name)
);

CREATE INDEX idx_fm_tenant_type ON freeform_models(tenant_id, model_type);
CREATE INDEX idx_fm_tenant_status ON freeform_models(tenant_id, status);
```

### 2.2 Planning Sandboxes
```sql
CREATE TABLE planning_sandboxes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    sandbox_name TEXT NOT NULL,
    owner_id TEXT NOT NULL,
    created_from_sandbox_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','SUBMITTED','APPROVED','MERGED','DISCARDED')),
    submitted_at TEXT,
    submitted_by TEXT,
    merged_at TEXT,
    merged_by TEXT,
    change_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES freeform_models(id) ON DELETE CASCADE
);

CREATE INDEX idx_ps_tenant_model ON planning_sandboxes(tenant_id, model_id);
CREATE INDEX idx_ps_tenant_owner ON planning_sandboxes(tenant_id, owner_id);
CREATE INDEX idx_ps_tenant_status ON planning_sandboxes(tenant_id, status);
```

### 2.3 Planning Cells
```sql
CREATE TABLE planning_cells (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    sandbox_id TEXT NOT NULL,
    dimension_coordinates TEXT NOT NULL,  -- JSON
    cell_value DECIMAL(20,4),
    cell_type TEXT NOT NULL DEFAULT 'DATA' CHECK(cell_type IN ('DATA','FORMULA','TEXT','VARIANCE')),
    formula TEXT,
    annotation TEXT,
    data_entry_user_id TEXT,
    entry_timestamp TEXT,
    cell_status TEXT NOT NULL DEFAULT 'ENTERED' CHECK(cell_status IN ('ENTERED','CALCULATED','IMPORTED','LOCKED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES freeform_models(id) ON DELETE CASCADE,
    FOREIGN KEY (sandbox_id) REFERENCES planning_sandboxes(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, sandbox_id, dimension_coordinates)
);

CREATE INDEX idx_pc_tenant_model ON planning_cells(tenant_id, model_id);
CREATE INDEX idx_pc_tenant_sandbox ON planning_cells(tenant_id, sandbox_id);
CREATE INDEX idx_pc_tenant_status ON planning_cells(tenant_id, cell_status);
CREATE INDEX idx_pc_tenant_type ON planning_cells(tenant_id, cell_type);
```

### 2.4 Planning Forms
```sql
CREATE TABLE planning_forms (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    form_name TEXT NOT NULL,
    form_type TEXT NOT NULL CHECK(form_type IN ('DATA_ENTRY','DASHBOARD','COMPARISON','VARIANCE_ANALYSIS')),
    layout_config TEXT NOT NULL,  -- JSON
    dimensions_on_rows TEXT NOT NULL,  -- JSON
    dimensions_on_columns TEXT NOT NULL,  -- JSON
    point_of_view TEXT NOT NULL,  -- JSON
    data_validation_rules TEXT,  -- JSON
    valid_combinations TEXT,  -- JSON
    is_template INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES freeform_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, form_name)
);

CREATE INDEX idx_pf_tenant_model ON planning_forms(tenant_id, model_id);
CREATE INDEX idx_pf_tenant_type ON planning_forms(tenant_id, form_type);
```

### 2.5 Scenario Comparisons
```sql
CREATE TABLE scenario_comparisons (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    comparison_name TEXT NOT NULL,
    scenario_a_sandbox_id TEXT NOT NULL,
    scenario_b_sandbox_id TEXT NOT NULL,
    comparison_dimensions TEXT NOT NULL,  -- JSON
    comparison_type TEXT NOT NULL CHECK(comparison_type IN ('VARIANCE','RATIO','ABSOLUTE','PERCENTAGE')),
    results TEXT,  -- JSON
    created_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES freeform_models(id) ON DELETE CASCADE
);

CREATE INDEX idx_sc_tenant_model ON scenario_comparisons(tenant_id, model_id);
```

### 2.6 Model Rules
```sql
CREATE TABLE model_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('ALLOCATION','SPREADING','GROWTH','RATIO','TREND','ROLLFORWARD')),
    target_coordinates TEXT NOT NULL,  -- JSON
    source_coordinates TEXT,  -- JSON
    parameters TEXT,  -- JSON
    execution_order INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES freeform_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, rule_name)
);

CREATE INDEX idx_mr_tenant_model ON model_rules(tenant_id, model_id);
CREATE INDEX idx_mr_tenant_type ON model_rules(tenant_id, rule_type);
CREATE INDEX idx_mr_tenant_active ON model_rules(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### Models
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/freeform-models` | Create a freeform model |
| GET | `/api/v1/freeform-models` | List freeform models |
| GET | `/api/v1/freeform-models/{id}` | Get model details |
| PUT | `/api/v1/freeform-models/{id}` | Update model |
| POST | `/api/v1/freeform-models/{id}/deploy` | Deploy model to active status |
| POST | `/api/v1/freeform-models/{id}/archive` | Archive a model |

### Sandboxes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/freeform-models/{id}/sandboxes` | Create a sandbox |
| GET | `/api/v1/freeform-models/{id}/sandboxes` | List sandboxes for model |
| GET | `/api/v1/sandboxes/{id}` | Get sandbox details |
| PUT | `/api/v1/sandboxes/{id}` | Update sandbox |
| POST | `/api/v1/sandboxes/{id}/submit` | Submit sandbox for approval |
| POST | `/api/v1/sandboxes/{id}/merge` | Merge sandbox into base |
| POST | `/api/v1/sandboxes/{id}/discard` | Discard sandbox changes |
| POST | `/api/v1/sandboxes/{id}/clone` | Clone a sandbox |

### Cells
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sandboxes/{id}/cells` | Get cells for sandbox |
| POST | `/api/v1/cells/batch-update` | Batch update cell values |
| POST | `/api/v1/cells/calculate` | Trigger cell calculation |
| POST | `/api/v1/cells/{id}/lock` | Lock a cell |
| POST | `/api/v1/cells/{id}/unlock` | Unlock a cell |

### Forms
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/freeform-models/{id}/forms` | Create a planning form |
| GET | `/api/v1/freeform-models/{id}/forms` | List forms for model |
| GET | `/api/v1/forms/{id}` | Get form details |
| PUT | `/api/v1/forms/{id}` | Update form |
| POST | `/api/v1/forms/{id}/render` | Render form with data |
| POST | `/api/v1/forms/{id}/validate-entry` | Validate data entry |

### Comparisons
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/scenario-comparisons/compare` | Run scenario comparison |
| GET | `/api/v1/scenario-comparisons/{id}/results` | Get comparison results |
| GET | `/api/v1/freeform-models/{id}/comparisons` | List comparisons for model |

### Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/freeform-models/{id}/model-rules` | Create a model rule |
| GET | `/api/v1/freeform-models/{id}/model-rules` | List rules for model |
| GET | `/api/v1/model-rules/{id}` | Get rule details |
| PUT | `/api/v1/model-rules/{id}` | Update rule |
| POST | `/api/v1/model-rules/{id}/execute` | Execute a model rule |
| POST | `/api/v1/model-rules/{id}/validate` | Validate a model rule |

---

## 4. Business Rules

1. A freeform model MUST define at least one dimension in the cube_config before deployment.
2. Each model MUST have exactly one base sandbox created automatically on deployment.
3. Sandbox changes MUST be isolated and MUST NOT affect other sandboxes or the base model until merged.
4. A cell with cell_status LOCKED MUST NOT be modified through data entry or batch update.
5. Formula cells MUST be recalculated when any source cell referenced by the formula changes.
6. Sandbox merge MUST detect and report conflicts when the same cell has been modified in both the sandbox and the base since the sandbox was created.
7. The system MUST validate that dimension_coordinates in a planning cell match the dimensions defined in the model.
8. Scenario comparisons MUST NOT be performed between sandboxes from different models.
9. Data validation rules on a planning form MUST be evaluated on every data entry submission.
10. Model rules MUST be executed in execution_order sequence within a single calculation pass.
11. The system SHOULD auto-populate cell_value from formula when cell_type is FORMULA.
12. A sandbox in SUBMITTED or APPROVED status MUST be read-only until merged or discarded.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package freeform.v1;

service FreeformPlanningService {
    // Models
    rpc CreateModel(CreateModelRequest) returns (CreateModelResponse);
    rpc GetModel(GetModelRequest) returns (GetModelResponse);
    rpc ListModels(ListModelsRequest) returns (ListModelsResponse);
    rpc DeployModel(DeployModelRequest) returns (DeployModelResponse);
    rpc ArchiveModel(ArchiveModelRequest) returns (ArchiveModelResponse);

    // Sandboxes
    rpc CreateSandbox(CreateSandboxRequest) returns (CreateSandboxResponse);
    rpc GetSandbox(GetSandboxRequest) returns (GetSandboxResponse);
    rpc ListSandboxes(ListSandboxesRequest) returns (ListSandboxesResponse);
    rpc SubmitSandbox(SubmitSandboxRequest) returns (SubmitSandboxResponse);
    rpc MergeSandbox(MergeSandboxRequest) returns (MergeSandboxResponse);
    rpc DiscardSandbox(DiscardSandboxRequest) returns (DiscardSandboxResponse);
    rpc CloneSandbox(CloneSandboxRequest) returns (CloneSandboxResponse);

    // Cells
    rpc GetCells(GetCellsRequest) returns (GetCellsResponse);
    rpc BatchUpdateCells(BatchUpdateCellsRequest) returns (BatchUpdateCellsResponse);
    rpc CalculateCells(CalculateCellsRequest) returns (CalculateCellsResponse);
    rpc LockCell(LockCellRequest) returns (LockCellResponse);
    rpc UnlockCell(UnlockCellRequest) returns (UnlockCellResponse);

    // Forms
    rpc CreateForm(CreateFormRequest) returns (CreateFormResponse);
    rpc GetForm(GetFormRequest) returns (GetFormResponse);
    rpc ListForms(ListFormsRequest) returns (ListFormsResponse);
    rpc RenderForm(RenderFormRequest) returns (RenderFormResponse);
    rpc ValidateEntry(ValidateEntryRequest) returns (ValidateEntryResponse);

    // Comparisons
    rpc CompareScenarios(CompareScenariosRequest) returns (CompareScenariosResponse);
    rpc GetComparisonResults(GetComparisonResultsRequest) returns (GetComparisonResultsResponse);
    rpc ListComparisons(ListComparisonsRequest) returns (ListComparisonsResponse);

    // Rules
    rpc CreateModelRule(CreateModelRuleRequest) returns (CreateModelRuleResponse);
    rpc ListModelRules(ListModelRulesRequest) returns (ListModelRulesResponse);
    rpc ExecuteModelRule(ExecuteModelRuleRequest) returns (ExecuteModelRuleResponse);
    rpc ValidateModelRule(ValidateModelRuleRequest) returns (ValidateModelRuleResponse);
}

message FreeformModel {
    string id = 1;
    string tenant_id = 2;
    string model_name = 3;
    string model_type = 4;
    string cube_config = 5;
    string dimensions = 6;
    string default_view = 7;
    string data_entry_mode = 8;
    string base_period = 9;
    int32 planning_horizon_months = 10;
    string status = 11;
}

message CreateModelRequest {
    string tenant_id = 1;
    string model_name = 2;
    string model_type = 3;
    string cube_config = 4;
    string dimensions = 5;
    string default_view = 6;
    string data_entry_mode = 7;
    string base_period = 8;
    int32 planning_horizon_months = 9;
    string created_by = 10;
}

message CreateModelResponse {
    FreeformModel model = 1;
}

message GetModelRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetModelResponse {
    FreeformModel model = 1;
}

message ListModelsRequest {
    string tenant_id = 1;
    string model_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListModelsResponse {
    repeated FreeformModel models = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message DeployModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string deployed_by = 3;
}

message DeployModelResponse {
    FreeformModel model = 1;
    string base_sandbox_id = 2;
}

message ArchiveModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string archived_by = 3;
}

message ArchiveModelResponse {
    FreeformModel model = 1;
}

message PlanningSandbox {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string sandbox_name = 4;
    string owner_id = 5;
    string created_from_sandbox_id = 6;
    string status = 7;
    string submitted_at = 8;
    int32 change_count = 9;
}

message CreateSandboxRequest {
    string tenant_id = 1;
    string model_id = 2;
    string sandbox_name = 3;
    string owner_id = 4;
    string created_from_sandbox_id = 5;
    string created_by = 6;
}

message CreateSandboxResponse {
    PlanningSandbox sandbox = 1;
}

message GetSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
}

message GetSandboxResponse {
    PlanningSandbox sandbox = 1;
}

message ListSandboxesRequest {
    string tenant_id = 1;
    string model_id = 2;
    string status = 3;
    string owner_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListSandboxesResponse {
    repeated PlanningSandbox sandboxes = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string submitted_by = 3;
}

message SubmitSandboxResponse {
    PlanningSandbox sandbox = 1;
}

message MergeConflict {
    string cell_id = 1;
    string dimension_coordinates = 2;
    double sandbox_value = 3;
    double base_value = 4;
}

message MergeSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    bool force = 3;
    string merged_by = 4;
}

message MergeSandboxResponse {
    bool success = 1;
    int32 cells_merged = 2;
    repeated MergeConflict conflicts = 3;
}

message DiscardSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string discarded_by = 3;
}

message DiscardSandboxResponse {
    bool success = 1;
}

message CloneSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string new_sandbox_name = 3;
    string cloned_by = 4;
}

message CloneSandboxResponse {
    PlanningSandbox sandbox = 1;
}

message PlanningCell {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string sandbox_id = 4;
    string dimension_coordinates = 5;
    double cell_value = 6;
    string cell_type = 7;
    string formula = 8;
    string annotation = 9;
    string data_entry_user_id = 10;
    string entry_timestamp = 11;
    string cell_status = 12;
}

message GetCellsRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string dimension_filter = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetCellsResponse {
    repeated PlanningCell cells = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CellUpdate {
    string cell_id = 1;
    string dimension_coordinates = 2;
    double cell_value = 3;
    string cell_type = 4;
    string formula = 5;
}

message BatchUpdateCellsRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    repeated CellUpdate updates = 3;
    string updated_by = 4;
}

message BatchUpdateCellsResponse {
    int32 cells_updated = 1;
    int32 cells_failed = 2;
    repeated string error_messages = 3;
}

message CalculateCellsRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    repeated string cell_ids = 3;
    string calculated_by = 4;
}

message CalculateCellsResponse {
    int32 cells_calculated = 1;
    double execution_seconds = 2;
}

message LockCellRequest {
    string tenant_id = 1;
    string cell_id = 2;
    string locked_by = 3;
}

message LockCellResponse {
    PlanningCell cell = 1;
}

message UnlockCellRequest {
    string tenant_id = 1;
    string cell_id = 2;
    string unlocked_by = 3;
}

message UnlockCellResponse {
    PlanningCell cell = 1;
}

message PlanningForm {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string form_name = 4;
    string form_type = 5;
    string layout_config = 6;
    string dimensions_on_rows = 7;
    string dimensions_on_columns = 8;
    string point_of_view = 9;
    bool is_template = 10;
}

message CreateFormRequest {
    string tenant_id = 1;
    string model_id = 2;
    string form_name = 3;
    string form_type = 4;
    string layout_config = 5;
    string dimensions_on_rows = 6;
    string dimensions_on_columns = 7;
    string point_of_view = 8;
    string data_validation_rules = 9;
    string valid_combinations = 10;
    string created_by = 11;
}

message CreateFormResponse {
    PlanningForm form = 1;
}

message GetFormRequest {
    string tenant_id = 1;
    string form_id = 2;
}

message GetFormResponse {
    PlanningForm form = 1;
}

message ListFormsRequest {
    string tenant_id = 1;
    string model_id = 2;
    string form_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListFormsResponse {
    repeated PlanningForm forms = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RenderFormRequest {
    string tenant_id = 1;
    string form_id = 2;
    string sandbox_id = 3;
    string point_of_view_override = 4;
}

message RenderFormResponse {
    repeated FormRow rows = 1;
    repeated FormColumn columns = 2;
    string rendered_at = 3;
}

message FormRow {
    string dimension_member = 1;
    string label = 2;
}

message FormColumn {
    string dimension_member = 1;
    string label = 2;
}

message ValidateEntryRequest {
    string tenant_id = 1;
    string form_id = 2;
    string dimension_coordinates = 3;
    double cell_value = 4;
}

message ValidateEntryResponse {
    bool is_valid = 1;
    repeated string validation_errors = 2;
    repeated string warnings = 3;
}

message ScenarioComparison {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string comparison_name = 4;
    string scenario_a_sandbox_id = 5;
    string scenario_b_sandbox_id = 6;
    string comparison_dimensions = 7;
    string comparison_type = 8;
    string results = 9;
}

message CompareScenariosRequest {
    string tenant_id = 1;
    string model_id = 2;
    string comparison_name = 3;
    string scenario_a_sandbox_id = 4;
    string scenario_b_sandbox_id = 5;
    string comparison_dimensions = 6;
    string comparison_type = 7;
    string created_by = 8;
}

message CompareScenariosResponse {
    ScenarioComparison comparison = 1;
}

message GetComparisonResultsRequest {
    string tenant_id = 1;
    string comparison_id = 2;
}

message GetComparisonResultsResponse {
    ScenarioComparison comparison = 1;
}

message ListComparisonsRequest {
    string tenant_id = 1;
    string model_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListComparisonsResponse {
    repeated ScenarioComparison comparisons = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ModelRule {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string rule_name = 4;
    string rule_type = 5;
    string target_coordinates = 6;
    string source_coordinates = 7;
    string parameters = 8;
    int32 execution_order = 9;
    bool is_active = 10;
}

message CreateModelRuleRequest {
    string tenant_id = 1;
    string model_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string target_coordinates = 5;
    string source_coordinates = 6;
    string parameters = 7;
    int32 execution_order = 8;
    string created_by = 9;
}

message CreateModelRuleResponse {
    ModelRule rule = 1;
}

message ListModelRulesRequest {
    string tenant_id = 1;
    string model_id = 2;
    string rule_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListModelRulesResponse {
    repeated ModelRule rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ExecuteModelRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string sandbox_id = 3;
    string executed_by = 4;
}

message ExecuteModelRuleResponse {
    bool success = 1;
    int32 cells_updated = 2;
    double execution_seconds = 3;
}

message ValidateModelRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message ValidateModelRuleResponse {
    bool is_valid = 1;
    repeated string errors = 2;
    repeated string warnings = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `epmplatform-service` | Application metadata, dimensions, cube config | Initialize model structures |
| `gl-service` | GL actuals, trial balances | Load actual data into base sandbox |
| `auth-service` | User roles, permissions | Enforce sandbox ownership and access |
| `hr-service` | Organization hierarchy, cost centers | Populate workforce planning dimensions |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `epmplatform-service` | Calculation results, cell updates | Sync data back to EPM platform |
| `budgeting-service` | Approved budget scenarios | Publish final budget numbers |
| `reporting-service` | Planning data, variance comparisons | Planning dashboards and reports |
| `consolidation-service` | Legal entity plans | Consolidate freeform plans |
| `notification-service` | Sandbox events, approval requests | User notifications |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ModelDeployed` | `freeform.model.deployed` | `{ tenant_id, model_id, model_name, model_type, base_sandbox_id }` | Published when a freeform model is deployed |
| `SandboxCreated` | `freeform.sandbox.created` | `{ tenant_id, sandbox_id, model_id, sandbox_name, owner_id, created_from_sandbox_id }` | Published when a new sandbox is created |
| `CellUpdated` | `freeform.cell.updated` | `{ tenant_id, cell_id, sandbox_id, dimension_coordinates, cell_value, cell_type }` | Published when a planning cell is updated |
| `SandboxMerged` | `freeform.sandbox.merged` | `{ tenant_id, sandbox_id, model_id, cells_merged, merged_by, conflicts_detected }` | Published when a sandbox is merged into base |
| `ScenarioCompared` | `freeform.scenario.compared` | `{ tenant_id, comparison_id, model_id, scenario_a_sandbox_id, scenario_b_sandbox_id, comparison_type }` | Published when a scenario comparison is executed |
