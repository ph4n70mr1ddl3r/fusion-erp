# 111 - Workforce Modeling Service Specification

## 1. Domain Overview

The Workforce Modeling service provides scenario-based workforce planning tools enabling HR leaders to model the impact of business decisions on headcount, costs, and organizational structure. It supports what-if analysis for M&A planning, restructuring, growth scenarios, reduction-in-force modeling, and cost projections with interactive scenario comparison, enabling data-driven workforce strategy decisions.

**Bounded Context:** Workforce Planning & Scenario Modeling
**Service Name:** `wfmodel-service`
**Database:** `data/wfmodel.db`
**HTTP Port:** 8148 | **gRPC Port:** 9148

---

## 2. Database Schema

### 2.1 Workforce Scenarios
```sql
CREATE TABLE workforce_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    scenario_type TEXT NOT NULL CHECK(scenario_type IN ('GROWTH','RESTRUCTURING','MERGER','RIF','REORG','BUDGET','EXPANSION','CUSTOM')),
    baseline_date TEXT NOT NULL,
    projection_start TEXT NOT NULL,
    projection_end TEXT NOT NULL,
    description TEXT,
    assumptions TEXT,  -- JSON: scenario assumptions
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','RUNNING','COMPLETED','APPROVED','ARCHIVED')),
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, scenario_name)
);

CREATE INDEX idx_scenarios_tenant_type ON workforce_scenarios(tenant_id, scenario_type);
CREATE INDEX idx_scenarios_tenant_status ON workforce_scenarios(tenant_id, status);
CREATE INDEX idx_scenarios_tenant_dates ON workforce_scenarios(tenant_id, projection_start, projection_end);
```

### 2.2 Scenario Parameters
```sql
CREATE TABLE scenario_parameters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    parameter_name TEXT NOT NULL,
    parameter_type TEXT NOT NULL CHECK(parameter_type IN ('HEADCOUNT','COMPENSATION','BENEFIT_RATE','TURNOVER_RATE','HIRE_RATE','PROMOTION_RATE','COST_OF_LIVING','REVENUE_PER_EMPLOYEE')),
    current_value DECIMAL(15,4) NOT NULL,
    projected_value DECIMAL(15,4) NOT NULL,
    change_pct DECIMAL(5,2),
    effective_date TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (scenario_id) REFERENCES workforce_scenarios(id) ON DELETE CASCADE
);

CREATE INDEX idx_params_tenant_scenario ON scenario_parameters(tenant_id, scenario_id);
CREATE INDEX idx_params_tenant_type ON scenario_parameters(tenant_id, parameter_type);
```

### 2.3 Scenario Headcount Projections
```sql
CREATE TABLE scenario_headcount_projections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    department_id TEXT,
    location_id TEXT,
    job_family_id TEXT,
    period TEXT NOT NULL,
    starting_headcount INTEGER NOT NULL,
    projected_hires INTEGER NOT NULL DEFAULT 0,
    projected_terminations INTEGER NOT NULL DEFAULT 0,
    projected_transfers INTEGER NOT NULL DEFAULT 0,
    projected_promotions INTEGER NOT NULL DEFAULT 0,
    ending_headcount INTEGER NOT NULL,
    confidence DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (scenario_id) REFERENCES workforce_scenarios(id) ON DELETE CASCADE
);

CREATE INDEX idx_headcount_tenant_scenario ON scenario_headcount_projections(tenant_id, scenario_id);
CREATE INDEX idx_headcount_tenant_dept ON scenario_headcount_projections(tenant_id, department_id, period);
CREATE INDEX idx_headcount_tenant_period ON scenario_headcount_projections(tenant_id, period);
```

### 2.4 Scenario Cost Projections
```sql
CREATE TABLE scenario_cost_projections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    department_id TEXT,
    cost_category TEXT NOT NULL CHECK(cost_category IN ('SALARY','BENEFITS','TAXES','TRAINING','RECRUITING','SEVERANCE','OVERHEAD')),
    period TEXT NOT NULL,
    baseline_cost_cents INTEGER NOT NULL,
    projected_cost_cents INTEGER NOT NULL,
    variance_cents INTEGER NOT NULL,
    variance_pct DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (scenario_id) REFERENCES workforce_scenarios(id) ON DELETE CASCADE
);

CREATE INDEX idx_costs_tenant_scenario ON scenario_cost_projections(tenant_id, scenario_id);
CREATE INDEX idx_costs_tenant_dept ON scenario_cost_projections(tenant_id, department_id, period);
CREATE INDEX idx_costs_tenant_category ON scenario_cost_projections(tenant_id, cost_category);
```

### 2.5 Scenario Org Changes
```sql
CREATE TABLE scenario_org_changes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    change_type TEXT NOT NULL CHECK(change_type IN ('ADD_DEPARTMENT','MERGE_DEPARTMENT','ELIMINATE_DEPARTMENT','RECLASSIFY_JOB','TRANSFER_TEAM','CHANGE_REPORTING')),
    source_entity_id TEXT,
    target_entity_id TEXT,
    effective_date TEXT NOT NULL,
    impact_description TEXT,
    affected_headcount INTEGER NOT NULL DEFAULT 0,
    cost_impact_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (scenario_id) REFERENCES workforce_scenarios(id) ON DELETE CASCADE
);

CREATE INDEX idx_orgchanges_tenant_scenario ON scenario_org_changes(tenant_id, scenario_id);
CREATE INDEX idx_orgchanges_tenant_type ON scenario_org_changes(tenant_id, change_type);
```

### 2.6 Scenario Comparisons
```sql
CREATE TABLE scenario_comparisons (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_ids TEXT NOT NULL,               -- JSON: array of scenario IDs
    comparison_date TEXT NOT NULL,
    metrics_compared TEXT NOT NULL,           -- JSON: list of compared metrics
    total_cost_baseline_cents INTEGER NOT NULL,
    total_cost_scenario_cents TEXT NOT NULL,  -- JSON: map of scenario_id -> cost
    headcount_baseline INTEGER NOT NULL,
    headcount_scenario TEXT NOT NULL,         -- JSON: map of scenario_id -> headcount
    summary TEXT,                             -- JSON: comparison summary

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_comparisons_tenant ON scenario_comparisons(tenant_id);
CREATE INDEX idx_comparisons_tenant_date ON scenario_comparisons(tenant_id, comparison_date);
```

### 2.7 Modeling Templates
```sql
CREATE TABLE modeling_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_type TEXT NOT NULL,
    default_parameters TEXT NOT NULL,  -- JSON: default parameter values
    default_assumptions TEXT,          -- JSON: default assumptions
    description TEXT,
    is_system INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_name)
);

CREATE INDEX idx_templates_tenant_type ON modeling_templates(tenant_id, template_type);
```

---

## 3. REST API Endpoints

### Scenarios
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/scenarios` | Create a workforce scenario |
| GET | `/api/v1/scenarios` | List workforce scenarios |
| GET | `/api/v1/scenarios/{id}` | Get scenario details |
| PUT | `/api/v1/scenarios/{id}` | Update scenario |
| POST | `/api/v1/scenarios/{id}/run` | Run scenario projections |
| POST | `/api/v1/scenarios/{id}/clone` | Clone a scenario |
| POST | `/api/v1/scenarios/{id}/approve` | Approve a scenario |

### Parameters
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/parameters` | Create a scenario parameter |
| GET | `/api/v1/parameters` | List scenario parameters |
| GET | `/api/v1/parameters/{id}` | Get parameter details |
| PUT | `/api/v1/parameters/{id}` | Update parameter |
| POST | `/api/v1/parameters/{id}/reset-to-baseline` | Reset parameter to baseline value |

### Headcount Projections
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/headcount-projections` | Get headcount projections by scenario |
| GET | `/api/v1/headcount-projections/by-department` | Get projections by department |
| POST | `/api/v1/headcount-projections/recalculate` | Recalculate headcount projections |

### Cost Projections
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/cost-projections` | Get cost projections by scenario |
| GET | `/api/v1/cost-projections/breakdown` | Get cost breakdown by category |

### Org Changes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/org-changes` | Create an organizational change |
| GET | `/api/v1/org-changes` | List org changes for scenario |
| GET | `/api/v1/org-changes/{id}` | Get org change details |
| PUT | `/api/v1/org-changes/{id}` | Update org change |
| POST | `/api/v1/org-changes/{id}/apply-to-scenario` | Apply org change to scenario |

### Comparisons
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/comparisons` | Compare multiple scenarios |
| GET | `/api/v1/comparisons/{id}` | Get comparison results |

### Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/templates` | Create a modeling template |
| GET | `/api/v1/templates` | List modeling templates |
| GET | `/api/v1/templates/{id}` | Get template details |
| PUT | `/api/v1/templates/{id}` | Update template |
| POST | `/api/v1/templates/{id}/instantiate` | Create scenario from template |

---

## 4. Business Rules

1. Each scenario MUST maintain isolation from others; changes to one scenario's parameters MUST NOT affect any other scenario's projections.
2. Parameter values MUST be validated against reasonable bounds (e.g., turnover_rate MUST be between 0 and 100, hire_rate MUST be non-negative).
3. Cost projections MUST be calculated in cents to avoid floating-point precision errors in financial calculations.
4. Scenarios in RUNNING status MUST NOT be modified until the projection run completes or is cancelled.
5. Scenario approval MUST require at least one authorized approver distinct from the scenario creator.
6. Headcount projections MUST account for cascading effects (e.g., terminations reducing promotion pools, transfers adjusting both source and target departments).
7. Comparisons MUST support a minimum of 2 and a maximum of 10 scenarios simultaneously.
8. Org changes with type ELIMINATE_DEPARTMENT MUST generate severance cost projections automatically based on affected_headcount.
9. The system SHOULD cache completed projection results and MUST NOT recompute them unless parameters are explicitly changed.
10. Templates marked as is_system MUST NOT be deleted or modified by tenant users.
11. Projection periods MUST align to calendar months or quarters depending on the projection duration.
12. The variance_pct on cost projections MUST be calculated as ((projected - baseline) / baseline) * 100 and MUST handle zero-baseline gracefully.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package wfmodel.v1;

service WorkforceModelingService {
    // Scenarios
    rpc CreateScenario(CreateScenarioRequest) returns (CreateScenarioResponse);
    rpc GetScenario(GetScenarioRequest) returns (GetScenarioResponse);
    rpc ListScenarios(ListScenariosRequest) returns (ListScenariosResponse);
    rpc UpdateScenario(UpdateScenarioRequest) returns (UpdateScenarioResponse);
    rpc RunScenario(RunScenarioRequest) returns (RunScenarioResponse);
    rpc CloneScenario(CloneScenarioRequest) returns (CloneScenarioResponse);
    rpc ApproveScenario(ApproveScenarioRequest) returns (ApproveScenarioResponse);

    // Parameters
    rpc CreateParameter(CreateParameterRequest) returns (CreateParameterResponse);
    rpc ListParameters(ListParametersRequest) returns (ListParametersResponse);
    rpc UpdateParameter(UpdateParameterRequest) returns (UpdateParameterResponse);
    rpc ResetParameterToBaseline(ResetParameterToBaselineRequest) returns (ResetParameterToBaselineResponse);

    // Headcount Projections
    rpc GetHeadcountProjections(GetHeadcountProjectionsRequest) returns (GetHeadcountProjectionsResponse);
    rpc GetHeadcountByDepartment(GetHeadcountByDepartmentRequest) returns (GetHeadcountByDepartmentResponse);
    rpc RecalculateHeadcount(RecalculateHeadcountRequest) returns (RecalculateHeadcountResponse);

    // Cost Projections
    rpc GetCostProjections(GetCostProjectionsRequest) returns (GetCostProjectionsResponse);
    rpc GetCostBreakdown(GetCostBreakdownRequest) returns (GetCostBreakdownResponse);

    // Org Changes
    rpc CreateOrgChange(CreateOrgChangeRequest) returns (CreateOrgChangeResponse);
    rpc ListOrgChanges(ListOrgChangesRequest) returns (ListOrgChangesResponse);
    rpc ApplyOrgChange(ApplyOrgChangeRequest) returns (ApplyOrgChangeResponse);

    // Comparisons
    rpc CompareScenarios(CompareScenariosRequest) returns (CompareScenariosResponse);
    rpc GetComparison(GetComparisonRequest) returns (GetComparisonResponse);

    // Templates
    rpc CreateTemplate(CreateTemplateRequest) returns (CreateTemplateResponse);
    rpc ListTemplates(ListTemplatesRequest) returns (ListTemplatesResponse);
    rpc InstantiateTemplate(InstantiateTemplateRequest) returns (InstantiateTemplateResponse);
}

message WorkforceScenario {
    string id = 1;
    string tenant_id = 2;
    string scenario_name = 3;
    string scenario_type = 4;
    string baseline_date = 5;
    string projection_start = 6;
    string projection_end = 7;
    string description = 8;
    string assumptions = 9;
    string status = 10;
    string created_by = 11;
    string approved_by = 12;
}

message CreateScenarioRequest {
    string tenant_id = 1;
    string scenario_name = 2;
    string scenario_type = 3;
    string baseline_date = 4;
    string projection_start = 5;
    string projection_end = 6;
    string description = 7;
    string assumptions = 8;
    string created_by = 9;
}

message CreateScenarioResponse {
    WorkforceScenario scenario = 1;
}

message GetScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
}

message GetScenarioResponse {
    WorkforceScenario scenario = 1;
}

message ListScenariosRequest {
    string tenant_id = 1;
    string scenario_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListScenariosResponse {
    repeated WorkforceScenario scenarios = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string scenario_name = 3;
    string description = 4;
    string assumptions = 5;
    string updated_by = 6;
}

message UpdateScenarioResponse {
    WorkforceScenario scenario = 1;
}

message RunScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string run_by = 3;
}

message RunScenarioResponse {
    WorkforceScenario scenario = 1;
    string run_id = 2;
}

message CloneScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string new_scenario_name = 3;
    string cloned_by = 4;
}

message CloneScenarioResponse {
    WorkforceScenario scenario = 1;
}

message ApproveScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string approved_by = 3;
}

message ApproveScenarioResponse {
    WorkforceScenario scenario = 1;
}

message ScenarioParameter {
    string id = 1;
    string tenant_id = 2;
    string scenario_id = 3;
    string parameter_name = 4;
    string parameter_type = 5;
    double current_value = 6;
    double projected_value = 7;
    double change_pct = 8;
    string effective_date = 9;
}

message CreateParameterRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string parameter_name = 3;
    string parameter_type = 4;
    double current_value = 5;
    double projected_value = 6;
    string effective_date = 7;
    string created_by = 8;
}

message CreateParameterResponse {
    ScenarioParameter parameter = 1;
}

message ListParametersRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string parameter_type = 3;
}

message ListParametersResponse {
    repeated ScenarioParameter parameters = 1;
}

message UpdateParameterRequest {
    string tenant_id = 1;
    string parameter_id = 2;
    double projected_value = 3;
    string notes = 4;
    string updated_by = 5;
}

message UpdateParameterResponse {
    ScenarioParameter parameter = 1;
}

message ResetParameterToBaselineRequest {
    string tenant_id = 1;
    string parameter_id = 2;
    string reset_by = 3;
}

message ResetParameterToBaselineResponse {
    ScenarioParameter parameter = 1;
}

message HeadcountProjection {
    string id = 1;
    string scenario_id = 2;
    string department_id = 3;
    string location_id = 4;
    string job_family_id = 5;
    string period = 6;
    int32 starting_headcount = 7;
    int32 projected_hires = 8;
    int32 projected_terminations = 9;
    int32 projected_transfers = 10;
    int32 projected_promotions = 11;
    int32 ending_headcount = 12;
    double confidence = 13;
}

message GetHeadcountProjectionsRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string period = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetHeadcountProjectionsResponse {
    repeated HeadcountProjection projections = 1;
    string next_page_token = 2;
}

message GetHeadcountByDepartmentRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string department_id = 3;
}

message GetHeadcountByDepartmentResponse {
    repeated HeadcountProjection projections = 1;
}

message RecalculateHeadcountRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string recalculated_by = 3;
}

message RecalculateHeadcountResponse {
    int32 projection_count = 1;
}

message CostProjection {
    string id = 1;
    string scenario_id = 2;
    string department_id = 3;
    string cost_category = 4;
    string period = 5;
    int64 baseline_cost_cents = 6;
    int64 projected_cost_cents = 7;
    int64 variance_cents = 8;
    double variance_pct = 9;
}

message GetCostProjectionsRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string cost_category = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetCostProjectionsResponse {
    repeated CostProjection projections = 1;
    string next_page_token = 2;
}

message CostBreakdownItem {
    string cost_category = 1;
    int64 total_baseline_cents = 2;
    int64 total_projected_cents = 3;
    int64 total_variance_cents = 4;
}

message GetCostBreakdownRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string period = 3;
}

message GetCostBreakdownResponse {
    repeated CostBreakdownItem items = 1;
    int64 grand_total_baseline_cents = 2;
    int64 grand_total_projected_cents = 3;
}

message OrgChange {
    string id = 1;
    string tenant_id = 2;
    string scenario_id = 3;
    string change_type = 4;
    string source_entity_id = 5;
    string target_entity_id = 6;
    string effective_date = 7;
    string impact_description = 8;
    int32 affected_headcount = 9;
    int64 cost_impact_cents = 10;
}

message CreateOrgChangeRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string change_type = 3;
    string source_entity_id = 4;
    string target_entity_id = 5;
    string effective_date = 6;
    string impact_description = 7;
    string created_by = 8;
}

message CreateOrgChangeResponse {
    OrgChange change = 1;
}

message ListOrgChangesRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string change_type = 3;
}

message ListOrgChangesResponse {
    repeated OrgChange changes = 1;
}

message ApplyOrgChangeRequest {
    string tenant_id = 1;
    string org_change_id = 2;
    string applied_by = 3;
}

message ApplyOrgChangeResponse {
    OrgChange change = 1;
    bool success = 2;
}

message ScenarioComparison {
    string id = 1;
    string tenant_id = 2;
    repeated string scenario_ids = 3;
    string comparison_date = 4;
    string metrics_compared = 5;
    int64 total_cost_baseline_cents = 6;
    string total_cost_scenario_cents = 7;
    int32 headcount_baseline = 8;
    string headcount_scenario = 9;
    string summary = 10;
}

message CompareScenariosRequest {
    string tenant_id = 1;
    repeated string scenario_ids = 2;
    repeated string metrics = 3;
    string compared_by = 4;
}

message CompareScenariosResponse {
    ScenarioComparison comparison = 1;
}

message GetComparisonRequest {
    string tenant_id = 1;
    string comparison_id = 2;
}

message GetComparisonResponse {
    ScenarioComparison comparison = 1;
}

message ModelingTemplate {
    string id = 1;
    string tenant_id = 2;
    string template_name = 3;
    string template_type = 4;
    string default_parameters = 5;
    string default_assumptions = 6;
    string description = 7;
    bool is_system = 8;
}

message CreateTemplateRequest {
    string tenant_id = 1;
    string template_name = 2;
    string template_type = 3;
    string default_parameters = 4;
    string default_assumptions = 5;
    string description = 6;
    string created_by = 7;
}

message CreateTemplateResponse {
    ModelingTemplate template = 1;
}

message ListTemplatesRequest {
    string tenant_id = 1;
    string template_type = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListTemplatesResponse {
    repeated ModelingTemplate templates = 1;
    string next_page_token = 2;
}

message InstantiateTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
    string scenario_name = 3;
    string parameter_overrides = 4;
    string instantiated_by = 5;
}

message InstantiateTemplateResponse {
    WorkforceScenario scenario = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Current headcount, org structure, job families | Establish baseline workforce data |
| `compensation-service` | Salary ranges, benefits cost rates, compensation budgets | Calculate cost projections |
| `payroll-service` | Actual payroll costs, tax rates | Validate and calibrate cost models |
| `recruiting-service` | Open requisitions, hiring pipeline, cost-per-hire | Factor in recruiting costs and planned hires |
| `finance-service` | Budget allocations, revenue projections | Align workforce costs with financial plans |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Approved org changes, headcount targets | Execute organizational restructuring |
| `reporting-service` | Scenario results, cost projections, comparisons | Workforce planning dashboards |
| `notification-service` | Scenario completion, approval requests | Alert planners of modeling results |
| `budget-service` | Approved workforce cost projections | Feed into budget planning cycles |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ScenarioCreated` | `wfmodel.scenario.created` | `{ tenant_id, scenario_id, scenario_name, scenario_type, baseline_date, created_by }` | Published when a new workforce scenario is created |
| `ScenarioRunCompleted` | `wfmodel.scenario.run.completed` | `{ tenant_id, scenario_id, scenario_name, status, total_projected_cost_cents, ending_headcount }` | Published when a scenario projection run finishes |
| `ScenarioApproved` | `wfmodel.scenario.approved` | `{ tenant_id, scenario_id, scenario_name, approved_by, total_cost_impact_cents }` | Published when a scenario is approved for execution |
| `ScenarioComparisonRequested` | `wfmodel.scenario.comparison.requested` | `{ tenant_id, comparison_id, scenario_ids, comparison_date }` | Published when a scenario comparison is initiated |
