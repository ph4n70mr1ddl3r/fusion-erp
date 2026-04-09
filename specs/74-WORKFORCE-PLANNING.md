# 74 - Workforce Planning Service Specification

## 1. Domain Overview
**Bounded Context:** Workforce Planning - Enables strategic workforce planning through headcount target setting, demand modeling, gap analysis, scenario planning, and actionable workforce metrics to align talent supply with business demand.

**Service Name:** wfplanning-service
**Database:** wfplanning_db
**HTTP Port:** 8106 | **gRPC Port:** 9106

## 2. Database Schema

```sql
CREATE TABLE workforce_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    fiscal_year INTEGER NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    business_unit_id TEXT,
    department_id TEXT,
    total_current_headcount INTEGER DEFAULT 0,
    total_target_headcount INTEGER DEFAULT 0,
    total_budget_cents INTEGER DEFAULT 0,
    owner_id TEXT NOT NULL,
    executive_sponsor_id TEXT,
    description TEXT,
    approved_at TEXT,
    approved_by TEXT
);

CREATE INDEX idx_wf_plans_tenant ON workforce_plans(tenant_id);
CREATE INDEX idx_wf_plans_status ON workforce_plans(status);
CREATE INDEX idx_wf_plans_fiscal_year ON workforce_plans(fiscal_year);
CREATE INDEX idx_wf_plans_department ON workforce_plans(department_id);

CREATE TABLE plan_headcount_targets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    plan_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    job_id TEXT,
    position_id TEXT,
    location_id TEXT,
    quarter TEXT NOT NULL,
    current_headcount INTEGER NOT NULL DEFAULT 0,
    target_headcount INTEGER NOT NULL DEFAULT 0,
    planned_hires INTEGER DEFAULT 0,
    planned_exits INTEGER DEFAULT 0,
    planned_transfers_in INTEGER DEFAULT 0,
    planned_transfers_out INTEGER DEFAULT 0,
    budget_cents INTEGER DEFAULT 0,
    rationale TEXT,
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_headcount_targets_tenant ON plan_headcount_targets(tenant_id);
CREATE INDEX idx_headcount_targets_plan ON plan_headcount_targets(plan_id);
CREATE INDEX idx_headcount_targets_quarter ON plan_headcount_targets(quarter);

CREATE TABLE workforce_demand_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    department_id TEXT,
    model_parameters TEXT NOT NULL,
    revenue_growth_assumption DECIMAL(10,4) DEFAULT 0.0,
    productivity_factor DECIMAL(10,4) DEFAULT 1.0,
    attrition_rate DECIMAL(10,4) DEFAULT 0.0,
    training_lead_time_days INTEGER DEFAULT 90,
    hiring_lead_time_days INTEGER DEFAULT 60,
    demand_multiplier DECIMAL(10,4) DEFAULT 1.0,
    output_data TEXT,
    confidence_level INTEGER DEFAULT 80,
    last_run_at TEXT,
    run_status TEXT,
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_demand_models_tenant ON workforce_demand_models(tenant_id);
CREATE INDEX idx_demand_models_plan ON workforce_demand_models(plan_id);

CREATE TABLE gap_analyses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    plan_id TEXT NOT NULL,
    model_id TEXT,
    department_id TEXT NOT NULL,
    job_id TEXT,
    skill_category TEXT,
    gap_type TEXT NOT NULL,
    current_supply INTEGER NOT NULL DEFAULT 0,
    projected_demand INTEGER NOT NULL DEFAULT 0,
    gap_quantity INTEGER NOT NULL DEFAULT 0,
    gap_percentage DECIMAL(10,2) DEFAULT 0.0,
    severity TEXT NOT NULL DEFAULT 'LOW',
    root_cause TEXT,
    recommended_actions TEXT,
    estimated_cost_cents INTEGER DEFAULT 0,
    time_to_close_days INTEGER,
    priority INTEGER DEFAULT 5,
    status TEXT NOT NULL DEFAULT 'IDENTIFIED',
    resolved_at TEXT,
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_gap_analyses_tenant ON gap_analyses(tenant_id);
CREATE INDEX idx_gap_analyses_plan ON gap_analyses(plan_id);
CREATE INDEX idx_gap_analyses_severity ON gap_analyses(severity);

CREATE TABLE action_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    gap_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    action_type TEXT NOT NULL,
    action_name TEXT NOT NULL,
    description TEXT,
    owner_id TEXT NOT NULL,
    assigned_to TEXT,
    start_date TEXT NOT NULL,
    target_date TEXT NOT NULL,
    completed_date TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED',
    budget_cents INTEGER DEFAULT 0,
    actual_cost_cents INTEGER DEFAULT 0,
    progress_percentage INTEGER DEFAULT 0,
    milestones TEXT,
    dependencies TEXT,
    outcome_notes TEXT,
    FOREIGN KEY (gap_id) REFERENCES gap_analyses(id),
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_action_plans_tenant ON action_plans(tenant_id);
CREATE INDEX idx_action_plans_gap ON action_plans(gap_id);
CREATE INDEX idx_action_plans_status ON action_plans(status);

CREATE TABLE plan_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    plan_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    scenario_type TEXT NOT NULL,
    assumptions TEXT NOT NULL,
    parameters TEXT NOT NULL,
    projected_headcount INTEGER DEFAULT 0,
    projected_cost_cents INTEGER DEFAULT 0,
    risk_score INTEGER DEFAULT 50,
    probability_score INTEGER DEFAULT 50,
    impact_score INTEGER DEFAULT 50,
    simulation_results TEXT,
    is_baseline INTEGER NOT NULL DEFAULT 0,
    comparison_data TEXT,
    notes TEXT,
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_scenarios_tenant ON plan_scenarios(tenant_id);
CREATE INDEX idx_scenarios_plan ON plan_scenarios(plan_id);

CREATE TABLE workforce_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    plan_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    metric_category TEXT NOT NULL,
    metric_value DECIMAL(15,4) NOT NULL,
    target_value DECIMAL(15,4),
    unit_of_measure TEXT,
    department_id TEXT,
    business_unit_id TEXT,
    trend_direction TEXT,
    previous_value DECIMAL(15,4),
    change_percentage DECIMAL(10,2),
    source_system TEXT,
    calculation_method TEXT,
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_wf_metrics_tenant ON workforce_metrics(tenant_id);
CREATE INDEX idx_wf_metrics_plan ON workforce_metrics(plan_id);
CREATE INDEX idx_wf_metrics_date ON workforce_metrics(metric_date);
CREATE INDEX idx_wf_metrics_category ON workforce_metrics(metric_category);

CREATE TABLE plan_approvals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    plan_id TEXT NOT NULL,
    approval_level INTEGER NOT NULL DEFAULT 1,
    approver_id TEXT NOT NULL,
    approver_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING',
    comments TEXT,
    submitted_at TEXT,
    reviewed_at TEXT,
    approved_at TEXT,
    rejected_reason TEXT,
    delegated_to TEXT,
    FOREIGN KEY (plan_id) REFERENCES workforce_plans(id)
);

CREATE INDEX idx_plan_approvals_tenant ON plan_approvals(tenant_id);
CREATE INDEX idx_plan_approvals_plan ON plan_approvals(plan_id);
CREATE INDEX idx_plan_approvals_status ON plan_approvals(status);
```

## 3. REST API Endpoints

### Workforce Plans
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/workforce-plans` | Create a new workforce plan |
| GET | `/api/v1/workforce-plans` | List workforce plans with filters |
| GET | `/api/v1/workforce-plans/{id}` | Get plan by ID with full details |
| PUT | `/api/v1/workforce-plans/{id}` | Update plan metadata |
| DELETE | `/api/v1/workforce-plans/{id}` | Deactivate a plan |
| POST | `/api/v1/workforce-plans/{id}/submit` | Submit plan for approval |
| POST | `/api/v1/workforce-plans/{id}/clone` | Clone plan as new version |

### Headcount Targets
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/workforce-plans/{planId}/headcount-targets` | Set headcount targets |
| GET | `/api/v1/workforce-plans/{planId}/headcount-targets` | Get targets for plan |
| PUT | `/api/v1/headcount-targets/{id}` | Update target |
| POST | `/api/v1/headcount-targets/{id}/adjust` | Adjust target with audit |
| GET | `/api/v1/headcount-targets/summary` | Get headcount target summary |

### Demand Models
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/demand-models` | Create a demand model |
| GET | `/api/v1/demand-models` | List demand models |
| GET | `/api/v1/demand-models/{id}` | Get demand model details |
| PUT | `/api/v1/demand-models/{id}` | Update model parameters |
| POST | `/api/v1/demand-models/{id}/run` | Execute demand model |
| GET | `/api/v1/demand-models/{id}/results` | Get model run results |

### Gap Analyses
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/workforce-plans/{planId}/gap-analyses` | Run gap analysis |
| GET | `/api/v1/gap-analyses` | List gap analyses |
| GET | `/api/v1/gap-analyses/{id}` | Get gap analysis details |
| PUT | `/api/v1/gap-analyses/{id}` | Update gap analysis |
| POST | `/api/v1/gap-analyses/{id}/resolve` | Mark gap as resolved |
| GET | `/api/v1/gap-analyses/summary` | Get gap summary dashboard |

### Action Plans
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/action-plans` | Create an action plan |
| GET | `/api/v1/action-plans` | List action plans |
| GET | `/api/v1/action-plans/{id}` | Get action plan details |
| PUT | `/api/v1/action-plans/{id}` | Update action plan |
| POST | `/api/v1/action-plans/{id}/progress` | Update progress |
| POST | `/api/v1/action-plans/{id}/complete` | Mark action plan complete |

### Scenarios
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/workforce-plans/{planId}/scenarios` | Create a scenario |
| GET | `/api/v1/scenarios` | List scenarios |
| GET | `/api/v1/scenarios/{id}` | Get scenario details |
| PUT | `/api/v1/scenarios/{id}` | Update scenario |
| POST | `/api/v1/scenarios/{id}/simulate` | Run scenario simulation |
| POST | `/api/v1/scenarios/compare` | Compare multiple scenarios |

### Metrics & Approvals
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/workforce-metrics` | List workforce metrics |
| POST | `/api/v1/workforce-metrics/calculate` | Calculate metrics for plan |
| GET | `/api/v1/workforce-metrics/dashboard` | Get metrics dashboard |
| GET | `/api/v1/plan-approvals/{planId}` | Get approval chain for plan |
| POST | `/api/v1/plan-approvals/{id}/approve` | Approve at current level |
| POST | `/api/v1/plan-approvals/{id}/reject` | Reject at current level |

## 4. Business Rules

1. A workforce plan MUST have a unique plan_code within a tenant and fiscal year.
2. Headcount targets MUST be set for each quarter within the plan period.
3. Total target headcount on the plan MUST equal the sum of all quarterly headcount targets.
4. The demand model MUST incorporate at least 3 years of historical workforce data for accurate projections.
5. Gap analyses MUST be run after demand model execution to ensure data consistency.
6. A gap with severity CRITICAL MUST have at least one associated action plan.
7. Action plan target dates MUST fall within the plan period.
8. Scenario simulation SHOULD complete within 60 seconds for standard complexity scenarios.
9. Only one scenario per plan MAY be marked as baseline.
10. Plan approvals MUST follow the configured approval chain in sequential order.
11. A plan MUST NOT be published until all approval levels have been completed.
12. Workforce metrics SHOULD be recalculated automatically when underlying data changes.
13. The system MUST prevent deletion of a plan that has approved action plans in progress.
14. Budget amounts on headcount targets SHOULD be validated against organizational budget limits.
15. Gap severity MUST be auto-calculated based on gap_percentage thresholds: >20% CRITICAL, >10% HIGH, >5% MEDIUM, <=5% LOW.

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package wfplanning;

service WorkforcePlanningService {
    // Workforce Plans
    rpc CreateWorkforcePlan(CreateWorkforcePlanRequest) returns (WorkforcePlanResponse);
    rpc GetWorkforcePlan(GetByIdRequest) returns (WorkforcePlanResponse);
    rpc ListWorkforcePlans(ListWorkforcePlansRequest) returns (WorkforcePlanListResponse);
    rpc UpdateWorkforcePlan(UpdateWorkforcePlanRequest) returns (WorkforcePlanResponse);
    rpc SubmitPlanForApproval(SubmitPlanRequest) returns (WorkforcePlanResponse);
    rpc ClonePlan(ClonePlanRequest) returns (WorkforcePlanResponse);

    // Headcount Targets
    rpc SetHeadcountTargets(SetHeadcountTargetsRequest) returns (HeadcountTargetListResponse);
    rpc GetHeadcountTargets(GetHeadcountTargetsRequest) returns (HeadcountTargetListResponse);
    rpc AdjustTarget(AdjustTargetRequest) returns (HeadcountTargetResponse);

    // Demand Models
    rpc CreateDemandModel(CreateDemandModelRequest) returns (DemandModelResponse);
    rpc RunDemandModel(RunDemandModelRequest) returns (DemandModelResultResponse);
    rpc GetDemandModelResults(GetDemandModelResultsRequest) returns (DemandModelResultResponse);

    // Gap Analysis
    rpc RunGapAnalysis(RunGapAnalysisRequest) returns (GapAnalysisListResponse);
    rpc GetGapAnalysis(GetByIdRequest) returns (GapAnalysisResponse);
    rpc ResolveGap(ResolveGapRequest) returns (GapAnalysisResponse);

    // Action Plans
    rpc CreateActionPlan(CreateActionPlanRequest) returns (ActionPlanResponse);
    rpc UpdateActionPlanProgress(UpdateProgressRequest) returns (ActionPlanResponse);

    // Scenarios
    rpc CreateScenario(CreateScenarioRequest) returns (ScenarioResponse);
    rpc SimulateScenario(SimulateScenarioRequest) returns (ScenarioSimulationResponse);
    rpc CompareScenarios(CompareScenariosRequest) returns (ScenarioComparisonResponse);

    // Metrics
    rpc CalculateMetrics(CalculateMetricsRequest) returns (MetricsResponse);
    rpc GetMetricsDashboard(GetMetricsDashboardRequest) returns (MetricsDashboardResponse);

    // Approvals
    rpc ApprovePlan(ApprovePlanRequest) returns (PlanApprovalResponse);
    rpc RejectPlan(RejectPlanRequest) returns (PlanApprovalResponse);
}

message GetByIdRequest {
    string id = 1;
    string tenant_id = 2;
}

message CreateWorkforcePlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    string plan_code = 3;
    string plan_type = 4;
    int32 fiscal_year = 5;
    string start_date = 6;
    string end_date = 7;
    string business_unit_id = 8;
    string department_id = 9;
    string owner_id = 10;
    string created_by = 11;
}

message WorkforcePlanResponse {
    string id = 1;
    string plan_name = 2;
    string plan_code = 3;
    string status = 4;
    int32 fiscal_year = 5;
    int32 total_current_headcount = 6;
    int32 total_target_headcount = 7;
    int64 total_budget_cents = 8;
    string created_at = 9;
    int32 version = 10;
}

message ListWorkforcePlansRequest {
    string tenant_id = 1;
    string status = 2;
    int32 fiscal_year = 3;
    int32 page = 4;
    int32 page_size = 5;
}

message WorkforcePlanListResponse {
    repeated WorkforcePlanResponse items = 1;
    int32 total_count = 2;
}

message UpdateWorkforcePlanRequest {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string updated_by = 4;
    int32 version = 5;
}

message SubmitPlanRequest {
    string id = 1;
    string tenant_id = 2;
    string submitted_by = 3;
}

message ClonePlanRequest {
    string id = 1;
    string tenant_id = 2;
    string new_plan_name = 3;
    string cloned_by = 4;
}

message SetHeadcountTargetsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    repeated HeadcountTargetInput targets = 3;
    string created_by = 4;
}

message HeadcountTargetInput {
    string department_id = 1;
    string job_id = 2;
    string quarter = 3;
    int32 target_headcount = 4;
    int32 planned_hires = 5;
    int32 planned_exits = 6;
    int64 budget_cents = 7;
}

message HeadcountTargetResponse {
    string id = 1;
    string plan_id = 2;
    string department_id = 3;
    string quarter = 4;
    int32 current_headcount = 5;
    int32 target_headcount = 6;
    int32 version = 7;
}

message HeadcountTargetListResponse {
    repeated HeadcountTargetResponse items = 1;
    int32 total_count = 2;
}

message GetHeadcountTargetsRequest {
    string plan_id = 1;
    string tenant_id = 2;
    string department_id = 3;
    string quarter = 4;
}

message AdjustTargetRequest {
    string id = 1;
    string tenant_id = 2;
    int32 new_target = 3;
    string rationale = 4;
    string adjusted_by = 5;
    int32 version = 6;
}

message CreateDemandModelRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string model_name = 3;
    string model_type = 4;
    string model_parameters = 5;
    string department_id = 6;
    string created_by = 7;
}

message DemandModelResponse {
    string id = 1;
    string model_name = 2;
    string model_type = 3;
    string status = 4;
    int32 confidence_level = 5;
    string last_run_at = 6;
}

message RunDemandModelRequest {
    string id = 1;
    string tenant_id = 2;
    string run_by = 3;
}

message DemandModelResultResponse {
    string model_id = 1;
    string run_status = 2;
    int32 confidence_level = 3;
    string output_data = 4;
    string run_at = 5;
}

message GetDemandModelResultsRequest {
    string model_id = 1;
    string tenant_id = 2;
}

message RunGapAnalysisRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string model_id = 3;
    string department_id = 4;
    string run_by = 5;
}

message GapAnalysisResponse {
    string id = 1;
    string plan_id = 2;
    string gap_type = 3;
    int32 current_supply = 4;
    int32 projected_demand = 5;
    int32 gap_quantity = 6;
    string severity = 7;
    string status = 8;
}

message GapAnalysisListResponse {
    repeated GapAnalysisResponse items = 1;
    int32 total_count = 2;
}

message ResolveGapRequest {
    string id = 1;
    string tenant_id = 2;
    string resolution_notes = 3;
    string resolved_by = 4;
}

message CreateActionPlanRequest {
    string tenant_id = 1;
    string gap_id = 2;
    string plan_id = 3;
    string action_type = 4;
    string action_name = 5;
    string owner_id = 6;
    string start_date = 7;
    string target_date = 8;
    int64 budget_cents = 9;
    string created_by = 10;
}

message ActionPlanResponse {
    string id = 1;
    string gap_id = 2;
    string action_type = 3;
    string status = 4;
    int32 progress_percentage = 5;
    string target_date = 6;
}

message UpdateProgressRequest {
    string id = 1;
    string tenant_id = 2;
    int32 progress_percentage = 3;
    string updated_by = 4;
}

message CreateScenarioRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string scenario_name = 3;
    string scenario_type = 4;
    string assumptions = 5;
    string parameters = 6;
    string created_by = 7;
}

message ScenarioResponse {
    string id = 1;
    string scenario_name = 2;
    string scenario_type = 3;
    int32 projected_headcount = 4;
    int64 projected_cost_cents = 5;
    int32 risk_score = 6;
    bool is_baseline = 7;
}

message SimulateScenarioRequest {
    string id = 1;
    string tenant_id = 2;
    string simulated_by = 3;
}

message ScenarioSimulationResponse {
    string scenario_id = 1;
    int32 projected_headcount = 2;
    int64 projected_cost_cents = 3;
    int32 risk_score = 4;
    string simulation_results = 5;
}

message CompareScenariosRequest {
    string tenant_id = 1;
    repeated string scenario_ids = 2;
}

message ScenarioComparisonResponse {
    repeated ScenarioResponse scenarios = 1;
    string comparison_data = 2;
}

message CalculateMetricsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string calculated_by = 3;
}

message MetricsResponse {
    string plan_id = 1;
    repeated MetricEntry metrics = 2;
}

message MetricEntry {
    string metric_name = 1;
    double metric_value = 2;
    double target_value = 3;
    string trend_direction = 4;
}

message GetMetricsDashboardRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message MetricsDashboardResponse {
    string plan_id = 1;
    repeated MetricEntry summary_metrics = 2;
    repeated GapSummary gaps_summary = 3;
    repeated ActionPlanSummary actions_summary = 4;
}

message GapSummary {
    string severity = 1;
    int32 count = 2;
}

message ActionPlanSummary {
    string status = 1;
    int32 count = 2;
}

message ApprovePlanRequest {
    string plan_id = 1;
    string tenant_id = 2;
    string approver_id = 3;
    string comments = 4;
}

message RejectPlanRequest {
    string plan_id = 1;
    string tenant_id = 2;
    string approver_id = 3;
    string rejected_reason = 4;
}

message PlanApprovalResponse {
    string plan_id = 1;
    string status = 2;
    string current_approval_level = 3;
    string approved_at = 4;
}
```

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data/Events | Purpose |
|---------------|-------------|---------|
| CORE-HR | Employee profiles, headcount data, org structure, skills inventory | Current workforce supply, organizational hierarchy |
| RECRUITING | Open positions, hiring pipeline, time-to-fill metrics | Planned hires tracking, recruitment capacity planning |
| COMPENSATION | Salary structures, budget allocations, cost projections | Workforce cost modeling, budget validation |

### Published To
| Target Service | Data/Events | Purpose |
|---------------|-------------|---------|
| RECRUITING | Headcount gaps, hiring targets | Trigger requisition creation for planned hires |
| COMPENSATION | Budget projections, headcount changes | Budget allocation and cost planning |
| CORE-HR | Plan approvals, workforce metrics | Organizational reporting and analytics |

## 7. Events

### Produced Events
| Event | Payload | Description |
|-------|---------|-------------|
| `PlanCreated` | `{ plan_id, tenant_id, plan_name, fiscal_year, owner_id }` | Emitted when a new workforce plan is created |
| `GapIdentified` | `{ gap_id, plan_id, gap_type, severity, department_id, gap_quantity, tenant_id }` | Emitted when a workforce gap is identified through analysis |
| `ScenarioModeled` | `{ scenario_id, plan_id, projected_headcount, risk_score, tenant_id }` | Emitted when a scenario simulation completes |
| `PlanApproved` | `{ plan_id, tenant_id, approved_by, approval_level, approved_at }` | Emitted when a plan approval level is completed |
| `PlanRejected` | `{ plan_id, tenant_id, rejected_by, rejected_reason }` | Emitted when a plan is rejected at any approval level |
| `ActionPlanCompleted` | `{ action_plan_id, gap_id, plan_id, outcome_notes, tenant_id }` | Emitted when an action plan is marked complete |
| `DemandModelExecuted` | `{ model_id, plan_id, confidence_level, run_status, tenant_id }` | Emitted when a demand model run completes |
| `TargetAdjusted` | `{ target_id, plan_id, old_target, new_target, rationale, tenant_id }` | Emitted when a headcount target is adjusted |
