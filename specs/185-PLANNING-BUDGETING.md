# 185 - Planning and Budgeting Service Specification

## 1. Domain Overview

Planning and Budgeting provides enterprise planning, budgeting, and forecasting capabilities as part of the Enterprise Performance Management (EPM) suite. Supports configurable planning cycles (annual, quarterly, rolling, monthly) with approval hierarchies, multi-dimensional budget versions with scenario modeling, driver-based and statistical forecast models, planning assumption management, and comprehensive budget-to-actual variance analysis. Enables finance teams to build bottom-up and top-down plans, collaborate across departments, compare multiple what-if scenarios, and track performance against targets. Integrates with General Ledger for actuals, Workforce Planning for headcount plans, and Revenue Management for revenue forecasts.

**Bounded Context:** Enterprise Planning, Budgeting & Forecasting
**Service Name:** `planning-budgeting-service`
**Database:** `data/planning_budgeting.db`
**HTTP Port:** 8203 | **gRPC Port:** 9203

---

## 2. Database Schema

### 2.1 Plan Cycles
```sql
CREATE TABLE pb_plan_cycles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_code TEXT NOT NULL,
    cycle_name TEXT NOT NULL,
    cycle_type TEXT NOT NULL CHECK(cycle_type IN ('ANNUAL','QUARTERLY','ROLLING','MONTHLY')),
    description TEXT,
    fiscal_year INTEGER NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    submission_deadline TEXT NOT NULL,
    approval_deadline TEXT NOT NULL,
    approval_hierarchy TEXT NOT NULL,             -- JSON: ordered approval chain
    dimensions TEXT NOT NULL,                    -- JSON: enabled dimensions (dept, entity, etc.)
    scenarios TEXT NOT NULL,                     -- JSON: enabled scenarios (budget, forecast, etc.)
    submitted_count INTEGER NOT NULL DEFAULT 0,
    approved_count INTEGER NOT NULL DEFAULT 0,
    total_contributors INTEGER NOT NULL DEFAULT 0,
    baseline_version_id TEXT,                    -- Reference to baseline budget version
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','OPEN','IN_REVIEW','APPROVED','CLOSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, cycle_code)
);

CREATE INDEX idx_pb_cycle_tenant ON pb_plan_cycles(tenant_id, status);
CREATE INDEX idx_pb_cycle_year ON pb_plan_cycles(fiscal_year);
CREATE INDEX idx_pb_cycle_dates ON pb_plan_cycles(period_start, period_end);
CREATE INDEX idx_pb_cycle_owner ON pb_plan_cycles(owner_id);
```

### 2.2 Budget Versions
```sql
CREATE TABLE pb_budget_versions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    version_code TEXT NOT NULL,
    version_name TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    scenario TEXT NOT NULL CHECK(scenario IN ('BUDGET','FORECAST','ACTUAL','VARIANCE','WHAT_IF','BASELINE')),
    department TEXT,
    account TEXT,
    entity TEXT,
    product_line TEXT,
    geographic_region TEXT,
    period TEXT NOT NULL,                        -- e.g., "2025-Q1", "2025-01"
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    unit_quantity INTEGER,
    unit_of_measure TEXT,
    line_description TEXT,
    driver_name TEXT,                            -- Associated planning driver
    is_baseline INTEGER NOT NULL DEFAULT 0,
    is_locked INTEGER NOT NULL DEFAULT 0,
    parent_version_id TEXT,                      -- For rolled-up versions
    source_version_id TEXT,                      -- For copied/derived versions
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES pb_plan_cycles(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, version_code, period)
);

CREATE INDEX idx_pb_bv_cycle ON pb_budget_versions(cycle_id, scenario);
CREATE INDEX idx_pb_bv_dept ON pb_budget_versions(department, period);
CREATE INDEX idx_pb_bv_period ON pb_budget_versions(period);
CREATE INDEX idx_pb_bv_scenario ON pb_budget_versions(scenario, period);
```

### 2.3 Forecast Models
```sql
CREATE TABLE pb_forecast_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    description TEXT,
    methodology TEXT NOT NULL CHECK(methodology IN ('DRIVER_BASED','TREND','SIMULATION','STATISTICAL')),
    parameters TEXT NOT NULL,                    -- JSON: model parameters and coefficients
    input_variables TEXT NOT NULL,               -- JSON: required input variables
    output_dimensions TEXT NOT NULL,             -- JSON: output dimensions and measures
    training_period_start TEXT,
    training_period_end TEXT,
    forecast_horizon_months INTEGER NOT NULL DEFAULT 12,
    confidence_level REAL NOT NULL DEFAULT 0.95,
    accuracy_mape REAL,                          -- Mean Absolute Percentage Error
    accuracy_mad REAL,                           -- Mean Absolute Deviation
    last_run_at TEXT,
    last_run_duration_ms INTEGER,
    next_scheduled_run TEXT,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_pb_fm_tenant ON pb_forecast_models(tenant_id, status);
CREATE INDEX idx_pb_fm_method ON pb_forecast_models(methodology);
CREATE INDEX idx_pb_fm_owner ON pb_forecast_models(owner_id);
```

### 2.4 Plan Assumptions
```sql
CREATE TABLE pb_plan_assumptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    assumption_code TEXT NOT NULL,
    assumption_name TEXT NOT NULL,
    description TEXT,
    dimension TEXT NOT NULL,                     -- Category: inflation, growth, headcount, FX, etc.
    applicable_period_start TEXT NOT NULL,
    applicable_period_end TEXT NOT NULL,
    assumed_value REAL NOT NULL,
    unit TEXT NOT NULL,                          -- PERCENT, RATE, COUNT, RATIO
    previous_value REAL,
    change_pct REAL,
    source TEXT,                                 -- Manual, historical, external
    justification TEXT,
    sensitivity TEXT,                            -- JSON: sensitivity analysis parameters
    approved_by TEXT,
    approved_at TEXT,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','SUPERSEDED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, assumption_code, applicable_period_start)
);

CREATE INDEX idx_pb_pa_tenant ON pb_plan_assumptions(tenant_id, dimension);
CREATE INDEX idx_pb_pa_period ON pb_plan_assumptions(applicable_period_start, applicable_period_end);
CREATE INDEX idx_pb_pa_status ON pb_plan_assumptions(status);
```

### 2.5 Variance Analysis
```sql
CREATE TABLE pb_variance_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    period TEXT NOT NULL,
    department TEXT NOT NULL,
    account TEXT NOT NULL,
    entity TEXT,
    product_line TEXT,
    scenario TEXT NOT NULL DEFAULT 'BUDGET',
    budget_amount_cents INTEGER NOT NULL DEFAULT 0,
    actual_amount_cents INTEGER NOT NULL DEFAULT 0,
    variance_amount_cents INTEGER NOT NULL DEFAULT 0,
    variance_pct REAL NOT NULL DEFAULT 0,
    favorable INTEGER NOT NULL DEFAULT 1,       -- 1 = favorable, 0 = unfavorable
    explanation TEXT,
    corrective_action TEXT,
    root_cause TEXT,
    reported_by TEXT,
    reported_at TEXT,
    reviewed_by TEXT,
    reviewed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES pb_plan_cycles(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, cycle_id, period, department, account, scenario)
);

CREATE INDEX idx_pb_va_cycle ON pb_variance_analysis(cycle_id, period);
CREATE INDEX idx_pb_va_dept ON pb_variance_analysis(department, period);
CREATE INDEX idx_pb_va_favorable ON pb_variance_analysis(favorable, variance_pct);
CREATE INDEX idx_pb_va_period ON pb_variance_analysis(period DESC);
```

---

## 3. API Endpoints

### 3.1 Plan Cycles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/planning-budgeting/cycles` | Create plan cycle |
| GET | `/api/v1/planning-budgeting/cycles` | List plan cycles |
| GET | `/api/v1/planning-budgeting/cycles/{id}` | Get cycle details |
| PUT | `/api/v1/planning-budgeting/cycles/{id}` | Update cycle |
| POST | `/api/v1/planning-budgeting/cycles/{id}/open` | Open cycle for input |
| POST | `/api/v1/planning-budgeting/cycles/{id}/approve` | Approve cycle |
| POST | `/api/v1/planning-budgeting/cycles/{id}/close` | Close cycle |

### 3.2 Budgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/planning-budgeting/budgets` | Create budget version |
| GET | `/api/v1/planning-budgeting/budgets` | List budget versions |
| GET | `/api/v1/planning-budgeting/budgets/{id}` | Get budget details |
| PUT | `/api/v1/planning-budgeting/budgets/{id}` | Update budget |
| POST | `/api/v1/planning-budgeting/budgets/bulk` | Bulk budget entry |
| POST | `/api/v1/planning-budgeting/budgets/{id}/submit` | Submit for approval |
| POST | `/api/v1/planning-budgeting/budgets/{id}/lock` | Lock budget version |
| GET | `/api/v1/planning-budgeting/budgets/rollup` | Get rolled-up budget |

### 3.3 Forecasts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/planning-budgeting/forecasts/models` | Create forecast model |
| GET | `/api/v1/planning-budgeting/forecasts/models` | List models |
| GET | `/api/v1/planning-budgeting/forecasts/models/{id}` | Get model details |
| PUT | `/api/v1/planning-budgeting/forecasts/models/{id}` | Update model |
| POST | `/api/v1/planning-budgeting/forecasts/models/{id}/run` | Run forecast |
| GET | `/api/v1/planning-budgeting/forecasts/models/{id}/accuracy` | Get accuracy metrics |

### 3.4 Assumptions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/planning-budgeting/assumptions` | Create assumption |
| GET | `/api/v1/planning-budgeting/assumptions` | List assumptions |
| GET | `/api/v1/planning-budgeting/assumptions/{id}` | Get assumption details |
| PUT | `/api/v1/planning-budgeting/assumptions/{id}` | Update assumption |
| POST | `/api/v1/planning-budgeting/assumptions/{id}/approve` | Approve assumption |
| GET | `/api/v1/planning-budgeting/assumptions/dimensions` | List assumption dimensions |

### 3.5 Variance Analysis
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/planning-budgeting/variance` | List variance analysis |
| GET | `/api/v1/planning-budgeting/variance/{id}` | Get variance detail |
| POST | `/api/v1/planning-budgeting/variance/{id}/explain` | Submit variance explanation |
| GET | `/api/v1/planning-budgeting/variance/summary` | Variance summary report |
| GET | `/api/v1/planning-budgeting/variance/trends` | Variance trend analysis |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `plan.cycle.created` | `{ cycle_id, cycle_type, fiscal_year, deadline }` | New planning cycle created |
| `budget.submitted` | `{ cycle_id, version_id, department, total_amount }` | Budget version submitted for approval |
| `forecast.updated` | `{ model_id, period, accuracy_mape }` | Forecast model produced new results |
| `variance.threshold_exceeded` | `{ variance_id, department, variance_pct, threshold }` | Variance exceeded threshold |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `gl.period.closed` | General Ledger | Load actuals for variance calculation |
| `workforce.plan.updated` | Workforce Planning | Incorporate headcount and compensation plans |
| `revenue.forecast.changed` | Revenue Management | Update revenue forecast assumptions |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.planning_budgeting.v1;

service PlanningBudgetingService {
    rpc GetPlanCycle(GetPlanCycleRequest) returns (GetPlanCycleResponse);
    rpc CreatePlanCycle(CreatePlanCycleRequest) returns (CreatePlanCycleResponse);
    rpc GetBudgetVersion(GetBudgetVersionRequest) returns (GetBudgetVersionResponse);
    rpc CreateBudgetVersion(CreateBudgetVersionRequest) returns (CreateBudgetVersionResponse);
    rpc RunForecast(RunForecastRequest) returns (RunForecastResponse);
    rpc GetVarianceAnalysis(GetVarianceAnalysisRequest) returns (GetVarianceAnalysisResponse);
}

// Plan cycle messages
message GetPlanCycleRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetPlanCycleResponse {
    PbPlanCycle data = 1;
}

message CreatePlanCycleRequest {
    string tenant_id = 1;
    string cycle_code = 2;
    string cycle_name = 3;
    string cycle_type = 4;
    string description = 5;
    int32 fiscal_year = 6;
    string period_start = 7;
    string period_end = 8;
    string submission_deadline = 9;
    string approval_deadline = 10;
    string approval_hierarchy = 11;
    string dimensions = 12;
    string scenarios = 13;
    string owner_id = 14;
}

message CreatePlanCycleResponse {
    PbPlanCycle data = 1;
}

message PbPlanCycle {
    string id = 1;
    string tenant_id = 2;
    string cycle_code = 3;
    string cycle_name = 4;
    string cycle_type = 5;
    string description = 6;
    int32 fiscal_year = 7;
    string period_start = 8;
    string period_end = 9;
    string submission_deadline = 10;
    string approval_deadline = 11;
    string approval_hierarchy = 12;
    string dimensions = 13;
    string scenarios = 14;
    int32 submitted_count = 15;
    int32 approved_count = 16;
    int32 total_contributors = 17;
    string baseline_version_id = 18;
    string owner_id = 19;
    string status = 20;
    string created_at = 21;
    string updated_at = 22;
}

// Budget version messages
message GetBudgetVersionRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetBudgetVersionResponse {
    PbBudgetVersion data = 1;
}

message CreateBudgetVersionRequest {
    string tenant_id = 1;
    string version_code = 2;
    string version_name = 3;
    string cycle_id = 4;
    string scenario = 5;
    string department = 6;
    string account = 7;
    string entity = 8;
    string product_line = 9;
    string geographic_region = 10;
    string period = 11;
    int64 amount_cents = 12;
    string currency_code = 13;
    int32 unit_quantity = 14;
    string unit_of_measure = 15;
    string line_description = 16;
    string driver_name = 17;
}

message CreateBudgetVersionResponse {
    PbBudgetVersion data = 1;
}

message PbBudgetVersion {
    string id = 1;
    string tenant_id = 2;
    string version_code = 3;
    string version_name = 4;
    string cycle_id = 5;
    string scenario = 6;
    string department = 7;
    string account = 8;
    string entity = 9;
    string product_line = 10;
    string geographic_region = 11;
    string period = 12;
    int64 amount_cents = 13;
    string currency_code = 14;
    int32 unit_quantity = 15;
    string unit_of_measure = 16;
    string line_description = 17;
    string driver_name = 18;
    bool is_baseline = 19;
    bool is_locked = 20;
    string parent_version_id = 21;
    string source_version_id = 22;
    string approved_by = 23;
    string approved_at = 24;
    string created_at = 25;
    string updated_at = 26;
}

// Forecast messages
message RunForecastRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message RunForecastResponse {
    string model_id = 1;
    string status = 2;
    double accuracy_mape = 3;
    double accuracy_mad = 4;
    string run_at = 5;
    int32 duration_ms = 6;
}

// Variance messages
message GetVarianceAnalysisRequest {
    string tenant_id = 1;
    string cycle_id = 2;
    string period = 3;
    string department = 4;
}

message GetVarianceAnalysisResponse {
    repeated PbVarianceAnalysis data = 1;
}

message PbVarianceAnalysis {
    string id = 1;
    string tenant_id = 2;
    string cycle_id = 3;
    string period = 4;
    string department = 5;
    string account = 6;
    string entity = 7;
    string product_line = 8;
    string scenario = 9;
    int64 budget_amount_cents = 10;
    int64 actual_amount_cents = 11;
    int64 variance_amount_cents = 12;
    double variance_pct = 13;
    bool favorable = 14;
    string explanation = 15;
    string corrective_action = 16;
    string root_cause = 17;
    string reported_by = 18;
    string reported_at = 19;
    string created_at = 20;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | pb_plan_cycles | -- |
| V002 | pb_budget_versions | V001 |
| V003 | pb_forecast_models | -- |
| V004 | pb_plan_assumptions | -- |
| V005 | pb_variance_analysis | V001 |

---

## 7. Business Rules

1. **Cycle Deadlines**: Budget submissions rejected after cycle submission deadline
2. **Approval Hierarchy**: Budgets must follow configured multi-level approval chain
3. **Version Immutability**: Approved and locked budget versions cannot be modified
4. **Variance Thresholds**: Unfavorable variances exceeding threshold trigger alerts and require explanation
5. **Forecast Accuracy**: Models with MAPE exceeding 15% flagged for recalibration
6. **Assumption Consistency**: Approved assumptions cascade to all dependent budget calculations
7. **Rolling Forecasts**: Rolling forecast types automatically extend horizon as periods close

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| gl-service | `GetActuals` | Actual financial data for variance analysis |
| workforce-service | `GetPlan` | Headcount and compensation planning inputs |
| revenue-service | `GetForecast` | Revenue forecast integration |
| consolidation-service | `GetData` | Consolidated plan data |
| workflow-service | `SubmitApproval` | Approval workflow orchestration |
| notification-service | `SendAlert` | Deadline and threshold alerts |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| reconciliation-service | `GetBudgetVersion` | Reconciled actuals for accuracy |
| profitability-service | `GetBudgetVersion` / `GetVariance` | Profit plan targets and actuals |
