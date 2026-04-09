# 105 - Workforce Labor Optimization Service Specification

## 1. Domain Overview

Workforce Labor Optimization provides advanced labor demand forecasting, shift optimization, labor cost modeling, workforce demand planning based on business volume predictions, budget-based scheduling constraints, labor compliance tracking, and optimization analytics for operational workforce management. It enables data-driven labor planning to minimize costs while maximizing coverage and compliance.

**Bounded Context:** Workforce Labor Optimization & Analytics
**Service Name:** `laboropt-service`
**Database:** `data/laboropt.db`
**HTTP Port:** 8142 | **gRPC Port:** 9142

---

## 2. Database Schema

### 2.1 Labor Demand Forecasts
```sql
CREATE TABLE labor_demand_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    forecast_date TEXT NOT NULL,
    forecast_type TEXT NOT NULL
        CHECK(forecast_type IN ('DAILY','WEEKLY','MONTHLY')),
    predicted_demand_hours DECIMAL(10,2) NOT NULL DEFAULT 0,
    confidence_level DECIMAL(5,2) NOT NULL DEFAULT 0,
    business_volume_drivers TEXT NOT NULL,  -- JSON
    seasonality_factor DECIMAL(5,2) NOT NULL DEFAULT 1.00,
    trend_factor DECIMAL(5,2) NOT NULL DEFAULT 1.00,
    model_type TEXT NOT NULL DEFAULT 'ML_REGRESSION'
        CHECK(model_type IN ('ARIMA','PROPHET','ML_REGRESSION')),
    model_version TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUPERSEDED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, location_id, department_id, forecast_date, forecast_type)
);

CREATE INDEX idx_labor_forecasts_tenant_loc ON labor_demand_forecasts(tenant_id, location_id);
CREATE INDEX idx_labor_forecasts_tenant_dept ON labor_demand_forecasts(tenant_id, department_id);
CREATE INDEX idx_labor_forecasts_tenant_date ON labor_demand_forecasts(tenant_id, forecast_date);
```

### 2.2 Labor Budgets
```sql
CREATE TABLE labor_budgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    budget_period_type TEXT NOT NULL
        CHECK(budget_period_type IN ('WEEKLY','MONTHLY','QUARTERLY','ANNUAL')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    budgeted_hours DECIMAL(10,2) NOT NULL DEFAULT 0,
    budgeted_labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_hours DECIMAL(10,2) NOT NULL DEFAULT 0,
    actual_labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    variance_hours DECIMAL(10,2) NOT NULL DEFAULT 0,
    variance_cost_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','IN_PROGRESS','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, location_id, department_id, period_start, budget_period_type)
);

CREATE INDEX idx_labor_budgets_tenant_loc ON labor_budgets(tenant_id, location_id);
CREATE INDEX idx_labor_budgets_tenant_dept ON labor_budgets(tenant_id, department_id);
CREATE INDEX idx_labor_budgets_tenant_status ON labor_budgets(tenant_id, status);
CREATE INDEX idx_labor_budgets_tenant_period ON labor_budgets(tenant_id, period_start, period_end);
```

### 2.3 Shift Optimization Results
```sql
CREATE TABLE shift_optimization_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    optimization_date TEXT NOT NULL,
    algorithm_type TEXT NOT NULL
        CHECK(algorithm_type IN ('LINEAR_PROGRAMMING','GENETIC','CONSTRAINT_SATISFACTION')),
    input_constraints TEXT NOT NULL,  -- JSON
    optimal_shifts TEXT NOT NULL,     -- JSON
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    coverage_score DECIMAL(5,2) NOT NULL DEFAULT 0,
    employee_satisfaction_score DECIMAL(5,2) NOT NULL DEFAULT 0,
    compliance_score DECIMAL(5,2) NOT NULL DEFAULT 0,
    computation_time_ms INTEGER NOT NULL DEFAULT 0,
    accepted INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, location_id, department_id, optimization_date, algorithm_type)
);

CREATE INDEX idx_shift_opt_tenant_loc ON shift_optimization_results(tenant_id, location_id);
CREATE INDEX idx_shift_opt_tenant_dept ON shift_optimization_results(tenant_id, department_id);
CREATE INDEX idx_shift_opt_tenant_date ON shift_optimization_results(tenant_id, optimization_date);
```

### 2.4 Labor Compliance Rules
```sql
CREATE TABLE labor_compliance_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('MAX_HOURS_DAILY','MAX_HOURS_WEEKLY','MIN_REST_PERIOD','OVERTIME_THRESHOLD','MIN_WAGE','CONSECUTIVE_DAYS','BREAK_REQUIREMENT')),
    parameters TEXT NOT NULL,  -- JSON
    jurisdiction TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    violation_penalty INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_compliance_rules_tenant_type ON labor_compliance_rules(tenant_id, rule_type);
CREATE INDEX idx_compliance_rules_tenant_jurisdiction ON labor_compliance_rules(tenant_id, jurisdiction);
CREATE INDEX idx_compliance_rules_tenant_active ON labor_compliance_rules(tenant_id, is_active);
```

### 2.5 Labor Compliance Checks
```sql
CREATE TABLE labor_compliance_checks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    schedule_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    violation_type TEXT NOT NULL,
    violation_date TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(severity IN ('WARNING','VIOLATION','CRITICAL')),
    description TEXT NOT NULL,
    resolution_status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(resolution_status IN ('OPEN','RESOLVED','WAIVED')),
    resolved_by TEXT,
    resolved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (rule_id) REFERENCES labor_compliance_rules(id) ON DELETE RESTRICT
);

CREATE INDEX idx_compliance_checks_tenant_employee ON labor_compliance_checks(tenant_id, employee_id);
CREATE INDEX idx_compliance_checks_tenant_schedule ON labor_compliance_checks(tenant_id, schedule_id);
CREATE INDEX idx_compliance_checks_tenant_severity ON labor_compliance_checks(tenant_id, severity);
CREATE INDEX idx_compliance_checks_tenant_resolution ON labor_compliance_checks(tenant_id, resolution_status);
```

### 2.6 Labor Cost Models
```sql
CREATE TABLE labor_cost_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    cost_components TEXT NOT NULL,     -- JSON
    hourly_rates TEXT NOT NULL,        -- JSON
    overtime_multipliers TEXT NOT NULL, -- JSON
    benefit_load_pct DECIMAL(5,2) NOT NULL DEFAULT 0,
    tax_load_pct DECIMAL(5,2) NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_name)
);

CREATE INDEX idx_cost_models_tenant_active ON labor_cost_models(tenant_id, is_active);
CREATE INDEX idx_cost_models_tenant_dates ON labor_cost_models(tenant_id, effective_from, effective_to);
```

### 2.7 Optimization Analytics
```sql
CREATE TABLE optimization_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    schedule_efficiency_pct DECIMAL(5,2) NOT NULL DEFAULT 0,
    labor_utilization_pct DECIMAL(5,2) NOT NULL DEFAULT 0,
    overtime_pct DECIMAL(5,2) NOT NULL DEFAULT 0,
    absenteeism_pct DECIMAL(5,2) NOT NULL DEFAULT 0,
    cost_per_unit_output INTEGER NOT NULL DEFAULT 0,
    benchmark_comparison TEXT,  -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, location_id, period_start, period_end)
);

CREATE INDEX idx_opt_analytics_tenant_loc ON optimization_analytics(tenant_id, location_id);
CREATE INDEX idx_opt_analytics_tenant_period ON optimization_analytics(tenant_id, period_start, period_end);
```

---

## 3. REST API Endpoints

### Forecasts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/laboropt/forecasts` | Create a labor demand forecast |
| GET | `/api/v1/laboropt/forecasts` | List forecasts |
| GET | `/api/v1/laboropt/forecasts/{id}` | Get forecast details |
| PUT | `/api/v1/laboropt/forecasts/{id}` | Update forecast |
| POST | `/api/v1/laboropt/forecasts/generate` | Generate demand forecast |
| GET | `/api/v1/laboropt/forecasts/accuracy` | Get forecast accuracy metrics |

### Budgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/laboropt/budgets` | Create a labor budget |
| GET | `/api/v1/laboropt/budgets` | List budgets |
| GET | `/api/v1/laboropt/budgets/{id}` | Get budget details |
| PUT | `/api/v1/laboropt/budgets/{id}` | Update budget |
| POST | `/api/v1/laboropt/budgets/{id}/approve` | Approve a budget |
| GET | `/api/v1/laboropt/budgets/{id}/variance` | Get budget variance |

### Optimization
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/laboropt/shifts/optimize` | Run shift optimization |
| GET | `/api/v1/laboropt/shifts/results` | List optimization results |
| GET | `/api/v1/laboropt/shifts/results/{id}` | Get optimization result details |
| POST | `/api/v1/laboropt/shifts/results/{id}/accept` | Accept optimization result |

### Compliance Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/laboropt/compliance/rules` | Create a compliance rule |
| GET | `/api/v1/laboropt/compliance/rules` | List compliance rules |
| GET | `/api/v1/laboropt/compliance/rules/{id}` | Get rule details |
| PUT | `/api/v1/laboropt/compliance/rules/{id}` | Update rule |
| DELETE | `/api/v1/laboropt/compliance/rules/{id}` | Delete rule |

### Compliance Checks
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/laboropt/compliance/violations` | List compliance violations |
| GET | `/api/v1/laboropt/compliance/violations/{id}` | Get violation details |
| POST | `/api/v1/laboropt/compliance/violations/{id}/resolve` | Resolve a violation |

### Cost Models
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/laboropt/cost-models` | Create a cost model |
| GET | `/api/v1/laboropt/cost-models` | List cost models |
| GET | `/api/v1/laboropt/cost-models/{id}` | Get cost model details |
| PUT | `/api/v1/laboropt/cost-models/{id}` | Update cost model |
| POST | `/api/v1/laboropt/cost-models/{id}/calculate-cost` | Calculate labor cost |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/laboropt/analytics/labor-utilization` | Get labor utilization metrics |
| GET | `/api/v1/laboropt/analytics/cost-trends` | Get labor cost trends |
| GET | `/api/v1/laboropt/analytics/forecast-accuracy` | Get forecast accuracy analytics |

---

## 4. Business Rules

1. A labor demand forecast MUST include at least one business volume driver to establish correlation with predicted demand.
2. Shift optimization MUST respect all active compliance rules for the applicable jurisdiction; any schedule violating a rule MUST be flagged before acceptance.
3. A labor budget in `DRAFT` status MAY be modified; once `APPROVED`, modifications MUST create a new budget version.
4. The system MUST NOT accept a shift optimization result with a compliance score below 95%.
5. Labor compliance checks MUST be executed against every published schedule; violations of type `CRITICAL` MUST block schedule publication.
6. A cost model MUST have at least one hourly rate defined and at least one cost component configured.
7. Overtime percentages exceeding 10% of total scheduled hours SHOULD trigger a warning to the scheduling manager.
8. Budget variance calculations MUST use the formula: `variance = actual - budgeted`, with positive values indicating over-budget.
9. Forecast accuracy MUST be tracked by comparing predicted demand hours to actual scheduled hours for completed periods.
10. The system MAY auto-resolve compliance violations flagged as `WARNING` if the underlying schedule is modified within 24 hours.
11. Shift optimization MUST consider employee preferences as a soft constraint, weighted by the `employee_satisfaction_score` target.
12. A labor budget with status `CLOSED` MUST NOT be modified or reopened.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package laboropt.v1;

service WorkforceLaborOptimizationService {
    // Forecasts
    rpc CreateForecast(CreateForecastRequest) returns (CreateForecastResponse);
    rpc GetForecast(GetForecastRequest) returns (GetForecastResponse);
    rpc ListForecasts(ListForecastsRequest) returns (ListForecastsResponse);
    rpc UpdateForecast(UpdateForecastRequest) returns (UpdateForecastResponse);
    rpc GenerateForecast(GenerateForecastRequest) returns (GenerateForecastResponse);
    rpc GetForecastAccuracy(GetForecastAccuracyRequest) returns (GetForecastAccuracyResponse);

    // Budgets
    rpc CreateBudget(CreateBudgetRequest) returns (CreateBudgetResponse);
    rpc GetBudget(GetBudgetRequest) returns (GetBudgetResponse);
    rpc ListBudgets(ListBudgetsRequest) returns (ListBudgetsResponse);
    rpc UpdateBudget(UpdateBudgetRequest) returns (UpdateBudgetResponse);
    rpc ApproveBudget(ApproveBudgetRequest) returns (ApproveBudgetResponse);
    rpc GetBudgetVariance(GetBudgetVarianceRequest) returns (GetBudgetVarianceResponse);

    // Shift Optimization
    rpc OptimizeShifts(OptimizeShiftsRequest) returns (OptimizeShiftsResponse);
    rpc GetOptimizationResult(GetOptimizationResultRequest) returns (GetOptimizationResultResponse);
    rpc ListOptimizationResults(ListOptimizationResultsRequest) returns (ListOptimizationResultsResponse);
    rpc AcceptOptimizationResult(AcceptOptimizationResultRequest) returns (AcceptOptimizationResultResponse);

    // Compliance Rules
    rpc CreateComplianceRule(CreateComplianceRuleRequest) returns (CreateComplianceRuleResponse);
    rpc GetComplianceRule(GetComplianceRuleRequest) returns (GetComplianceRuleResponse);
    rpc ListComplianceRules(ListComplianceRulesRequest) returns (ListComplianceRulesResponse);
    rpc UpdateComplianceRule(UpdateComplianceRuleRequest) returns (UpdateComplianceRuleResponse);
    rpc DeleteComplianceRule(DeleteComplianceRuleRequest) returns (DeleteComplianceRuleResponse);

    // Compliance Checks
    rpc ListComplianceViolations(ListComplianceViolationsRequest) returns (ListComplianceViolationsResponse);
    rpc GetComplianceViolation(GetComplianceViolationRequest) returns (GetComplianceViolationResponse);
    rpc ResolveViolation(ResolveViolationRequest) returns (ResolveViolationResponse);

    // Cost Models
    rpc CreateCostModel(CreateCostModelRequest) returns (CreateCostModelResponse);
    rpc GetCostModel(GetCostModelRequest) returns (GetCostModelResponse);
    rpc ListCostModels(ListCostModelsRequest) returns (ListCostModelsResponse);
    rpc UpdateCostModel(UpdateCostModelRequest) returns (UpdateCostModelResponse);
    rpc CalculateLaborCost(CalculateLaborCostRequest) returns (CalculateLaborCostResponse);

    // Analytics
    rpc GetLaborUtilization(GetLaborUtilizationRequest) returns (GetLaborUtilizationResponse);
    rpc GetCostTrends(GetCostTrendsRequest) returns (GetCostTrendsResponse);
    rpc GetForecastAccuracyAnalytics(GetForecastAccuracyAnalyticsRequest) returns (GetForecastAccuracyAnalyticsResponse);
}

message LaborDemandForecast {
    string id = 1;
    string tenant_id = 2;
    string location_id = 3;
    string department_id = 4;
    string forecast_date = 5;
    string forecast_type = 6;
    double predicted_demand_hours = 7;
    double confidence_level = 8;
    string business_volume_drivers = 9;
    double seasonality_factor = 10;
    double trend_factor = 11;
    string model_type = 12;
    string model_version = 13;
    string status = 14;
}

message CreateForecastRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string forecast_date = 4;
    string forecast_type = 5;
    double predicted_demand_hours = 6;
    double confidence_level = 7;
    string business_volume_drivers = 8;
    string created_by = 9;
}

message CreateForecastResponse {
    LaborDemandForecast forecast = 1;
}

message GetForecastRequest {
    string tenant_id = 1;
    string forecast_id = 2;
}

message GetForecastResponse {
    LaborDemandForecast forecast = 1;
}

message ListForecastsRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string forecast_type = 4;
    string forecast_date_from = 5;
    string forecast_date_to = 6;
    int32 page_size = 7;
    string page_token = 8;
}

message ListForecastsResponse {
    repeated LaborDemandForecast forecasts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateForecastRequest {
    string tenant_id = 1;
    string forecast_id = 2;
    double predicted_demand_hours = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateForecastResponse {
    LaborDemandForecast forecast = 1;
}

message GenerateForecastRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string forecast_date = 4;
    string forecast_type = 5;
    string model_type = 6;
    string generated_by = 7;
}

message GenerateForecastResponse {
    LaborDemandForecast forecast = 1;
    double confidence_level = 2;
}

message ForecastAccuracyEntry {
    string forecast_date = 1;
    double predicted_hours = 2;
    double actual_hours = 3;
    double absolute_error = 4;
    double pct_error = 5;
}

message GetForecastAccuracyRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string period_start = 4;
    string period_end = 5;
}

message GetForecastAccuracyResponse {
    repeated ForecastAccuracyEntry entries = 1;
    double mean_absolute_pct_error = 2;
    double overall_accuracy = 3;
}

message LaborBudget {
    string id = 1;
    string tenant_id = 2;
    string location_id = 3;
    string department_id = 4;
    string budget_period_type = 5;
    string period_start = 6;
    string period_end = 7;
    double budgeted_hours = 8;
    int64 budgeted_labor_cost_cents = 9;
    double actual_hours = 10;
    int64 actual_labor_cost_cents = 11;
    double variance_hours = 12;
    int64 variance_cost_cents = 13;
    string status = 14;
}

message CreateBudgetRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string budget_period_type = 4;
    string period_start = 5;
    string period_end = 6;
    double budgeted_hours = 7;
    int64 budgeted_labor_cost_cents = 8;
    string created_by = 9;
}

message CreateBudgetResponse {
    LaborBudget budget = 1;
}

message GetBudgetRequest {
    string tenant_id = 1;
    string budget_id = 2;
}

message GetBudgetResponse {
    LaborBudget budget = 1;
}

message ListBudgetsRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListBudgetsResponse {
    repeated LaborBudget budgets = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateBudgetRequest {
    string tenant_id = 1;
    string budget_id = 2;
    double budgeted_hours = 3;
    int64 budgeted_labor_cost_cents = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateBudgetResponse {
    LaborBudget budget = 1;
}

message ApproveBudgetRequest {
    string tenant_id = 1;
    string budget_id = 2;
    string approved_by = 3;
}

message ApproveBudgetResponse {
    LaborBudget budget = 1;
}

message BudgetVariance {
    string budget_id = 1;
    double hours_variance = 2;
    int64 cost_variance_cents = 3;
    double hours_variance_pct = 4;
    double cost_variance_pct = 5;
    string trend = 6;
}

message GetBudgetVarianceRequest {
    string tenant_id = 1;
    string budget_id = 2;
}

message GetBudgetVarianceResponse {
    BudgetVariance variance = 1;
}

message ShiftOptimizationResult {
    string id = 1;
    string tenant_id = 2;
    string location_id = 3;
    string department_id = 4;
    string optimization_date = 5;
    string algorithm_type = 6;
    string input_constraints = 7;
    string optimal_shifts = 8;
    int64 total_cost_cents = 9;
    double coverage_score = 10;
    double employee_satisfaction_score = 11;
    double compliance_score = 12;
    int64 computation_time_ms = 13;
    bool accepted = 14;
}

message OptimizeShiftsRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string optimization_date = 4;
    string algorithm_type = 5;
    string constraints = 6;  -- JSON
    string cost_model_id = 7;
    string optimized_by = 8;
}

message OptimizeShiftsResponse {
    ShiftOptimizationResult result = 1;
}

message GetOptimizationResultRequest {
    string tenant_id = 1;
    string result_id = 2;
}

message GetOptimizationResultResponse {
    ShiftOptimizationResult result = 1;
}

message ListOptimizationResultsRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string optimization_date_from = 4;
    string optimization_date_to = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListOptimizationResultsResponse {
    repeated ShiftOptimizationResult results = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message AcceptOptimizationResultRequest {
    string tenant_id = 1;
    string result_id = 2;
    string accepted_by = 3;
}

message AcceptOptimizationResultResponse {
    ShiftOptimizationResult result = 1;
}

message ComplianceRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string parameters = 5;
    string jurisdiction = 6;
    bool is_active = 7;
    int64 violation_penalty = 8;
}

message CreateComplianceRuleRequest {
    string tenant_id = 1;
    string rule_name = 2;
    string rule_type = 3;
    string parameters = 4;
    string jurisdiction = 5;
    int64 violation_penalty = 6;
    string created_by = 7;
}

message CreateComplianceRuleResponse {
    ComplianceRule rule = 1;
}

message GetComplianceRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message GetComplianceRuleResponse {
    ComplianceRule rule = 1;
}

message ListComplianceRulesRequest {
    string tenant_id = 1;
    string rule_type = 2;
    string jurisdiction = 3;
    bool is_active = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListComplianceRulesResponse {
    repeated ComplianceRule rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateComplianceRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string parameters = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateComplianceRuleResponse {
    ComplianceRule rule = 1;
}

message DeleteComplianceRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string deleted_by = 3;
}

message DeleteComplianceRuleResponse {
    bool success = 1;
}

message ComplianceViolation {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string schedule_id = 4;
    string rule_id = 5;
    string violation_type = 6;
    string violation_date = 7;
    string severity = 8;
    string description = 9;
    string resolution_status = 10;
}

message ListComplianceViolationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string schedule_id = 3;
    string severity = 4;
    string resolution_status = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListComplianceViolationsResponse {
    repeated ComplianceViolation violations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetComplianceViolationRequest {
    string tenant_id = 1;
    string violation_id = 2;
}

message GetComplianceViolationResponse {
    ComplianceViolation violation = 1;
}

message ResolveViolationRequest {
    string tenant_id = 1;
    string violation_id = 2;
    string resolution_notes = 3;
    string resolved_by = 4;
}

message ResolveViolationResponse {
    ComplianceViolation violation = 1;
}

message CostModel {
    string id = 1;
    string tenant_id = 2;
    string model_name = 3;
    string cost_components = 4;
    string hourly_rates = 5;
    string overtime_multipliers = 6;
    double benefit_load_pct = 7;
    double tax_load_pct = 8;
    string effective_from = 9;
    string effective_to = 10;
    bool is_active = 11;
}

message CreateCostModelRequest {
    string tenant_id = 1;
    string model_name = 2;
    string cost_components = 3;
    string hourly_rates = 4;
    string overtime_multipliers = 5;
    double benefit_load_pct = 6;
    double tax_load_pct = 7;
    string effective_from = 8;
    string created_by = 9;
}

message CreateCostModelResponse {
    CostModel cost_model = 1;
}

message GetCostModelRequest {
    string tenant_id = 1;
    string cost_model_id = 2;
}

message GetCostModelResponse {
    CostModel cost_model = 1;
}

message ListCostModelsRequest {
    string tenant_id = 1;
    bool is_active = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCostModelsResponse {
    repeated CostModel models = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateCostModelRequest {
    string tenant_id = 1;
    string cost_model_id = 2;
    string cost_components = 3;
    string hourly_rates = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateCostModelResponse {
    CostModel cost_model = 1;
}

message CalculateLaborCostRequest {
    string tenant_id = 1;
    string cost_model_id = 2;
    string schedule_data = 3;  -- JSON
}

message CalculateLaborCostResponse {
    int64 base_cost_cents = 1;
    int64 overtime_cost_cents = 2;
    int64 benefit_cost_cents = 3;
    int64 tax_cost_cents = 4;
    int64 total_cost_cents = 5;
    double total_hours = 6;
}

message LaborUtilizationEntry {
    string location_id = 1;
    string department_id = 2;
    double utilization_pct = 3;
    double overtime_pct = 4;
    double absenteeism_pct = 5;
    double efficiency_pct = 6;
}

message GetLaborUtilizationRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string period_start = 4;
    string period_end = 5;
}

message GetLaborUtilizationResponse {
    repeated LaborUtilizationEntry entries = 1;
    double avg_utilization_pct = 2;
    double avg_overtime_pct = 3;
}

message CostTrendPoint {
    string period = 1;
    int64 total_cost_cents = 2;
    int64 base_cost_cents = 3;
    int64 overtime_cost_cents = 4;
    double total_hours = 5;
    double cost_per_hour_cents = 6;
}

message GetCostTrendsRequest {
    string tenant_id = 1;
    string location_id = 2;
    string department_id = 3;
    string period_start = 4;
    string period_end = 5;
    string granularity = 6;  -- DAILY, WEEKLY, MONTHLY
}

message GetCostTrendsResponse {
    repeated CostTrendPoint trends = 1;
    double avg_cost_per_hour_cents = 2;
    double trend_direction = 3;  -- positive = increasing
}

message ForecastAccuracyAnalytics {
    string location_id = 1;
    string department_id = 2;
    double mape = 3;  -- Mean Absolute Percentage Error
    double bias = 4;
    int32 total_forecasts = 5;
    int32 within_tolerance = 6;
}

message GetForecastAccuracyAnalyticsRequest {
    string tenant_id = 1;
    string period_start = 2;
    string period_end = 3;
}

message GetForecastAccuracyAnalyticsResponse {
    repeated ForecastAccuracyAnalytics analytics = 1;
    double overall_mape = 2;
    double overall_accuracy_pct = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `workforce-scheduling` | Published schedules, shift assignments | Compliance checking and optimization input |
| `timelabor-service` | Time cards, actual hours worked | Budget variance and utilization calculation |
| `payroll-service` | Payroll totals, overtime data | Labor cost model calibration |
| `core-hr-service` | Employee profiles, skills, availability | Shift optimization constraints |
| `compensation-service` | Hourly rates, overtime rates | Cost model configuration |
| `absence-service` | Absence records, leave balances | Absenteeism analytics and demand adjustment |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `workforce-scheduling` | Optimized shift assignments | Schedule creation from optimization results |
| `payroll-service` | Labor cost calculations | Cost projection and budgeting |
| `reporting-service` | Utilization metrics, compliance reports | Workforce analytics reporting |
| `notification-service` | Compliance violation alerts, budget threshold warnings | Proactive alerting |
| `fdi-service` | Labor analytics, forecast accuracy | Cross-domain workforce intelligence |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `LaborForecastGenerated` | `laboropt.forecast.generated` | `{ tenant_id, forecast_id, location_id, department_id, forecast_date, predicted_demand_hours, confidence_level }` | Published when a labor demand forecast is generated |
| `ShiftsOptimized` | `laboropt.shifts.optimized` | `{ tenant_id, result_id, location_id, department_id, optimization_date, coverage_score, compliance_score, total_cost_cents }` | Published when a shift optimization completes |
| `ComplianceViolation` | `laboropt.compliance.violation` | `{ tenant_id, violation_id, employee_id, schedule_id, rule_id, severity, violation_type, description }` | Published when a compliance violation is detected |
| `BudgetThresholdExceeded` | `laboropt.budget.threshold-exceeded` | `{ tenant_id, budget_id, location_id, department_id, variance_cost_cents, variance_pct, threshold_type }` | Published when actual labor costs exceed a budget threshold |
