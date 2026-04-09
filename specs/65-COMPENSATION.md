# 65 - Compensation Service Specification

## 1. Domain Overview

The Compensation service manages employee compensation planning, salary administration, grade-rate matrices, compensation budgets, worksheets, awards, stock option grants, salary benchmarking, and total compensation statements. It supports annual compensation cycles, market-based pricing, and workforce compensation distribution.

**Bounded Context:** Compensation Planning & Administration
**Service Name:** `compensation-service`
**Database:** `data/compensation.db`
**HTTP Port:** 8097 | **gRPC Port:** 9097

---

## 2. Database Schema

### 2.1 Compensation Plans
```sql
CREATE TABLE compensation_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('SALARY','BONUS','COMMISSION','STOCK','MIXED','TOTAL_COMPENSATION')),
    plan_period_start TEXT NOT NULL,
    plan_period_end TEXT NOT NULL,
    cycle_status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(cycle_status IN ('DRAFT','ACTIVE','IN_REVIEW','APPROVED','COMPLETED','CANCELLED')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    eligibility_criteria TEXT,  -- JSON: rules for eligible employees
    access_level TEXT DEFAULT 'MANAGER' CHECK(access_level IN ('HR_ONLY','MANAGER','EMPLOYEE','PUBLIC')),
    description TEXT,
    effective_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_comp_plans_tenant_status ON compensation_plans(tenant_id, cycle_status);
CREATE INDEX idx_comp_plans_tenant_period ON compensation_plans(tenant_id, plan_period_start, plan_period_end);
```

### 2.2 Compensation Components
```sql
CREATE TABLE compensation_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    component_name TEXT NOT NULL,
    component_code TEXT NOT NULL,
    component_type TEXT NOT NULL CHECK(component_type IN ('BASE_SALARY','MERIT_INCREASE','LUMP_SUM','BONUS','COMMISSION','STOCK_OPTION','RSU','ALLOWANCE','PREMIUM','ADJUSTMENT')),
    calculation_method TEXT NOT NULL DEFAULT 'FIXED' CHECK(calculation_method IN ('FIXED','PERCENTAGE','FORMULA','GRADE_BASED','MARKET_BASED')),
    default_value INTEGER,  -- cents or basis points
    min_value INTEGER,
    max_value INTEGER,
    currency_code TEXT DEFAULT 'USD',
    is_mandatory INTEGER NOT NULL DEFAULT 0,
    display_order INTEGER DEFAULT 0,
    eligible_component_flag INTEGER DEFAULT 1,
    proration_rule TEXT DEFAULT 'NONE' CHECK(proration_rule IN ('NONE','CALENDAR_DAYS','WORKING_DAYS')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES compensation_plans(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, plan_id, component_code)
);

CREATE INDEX idx_comp_components_tenant_plan ON compensation_components(tenant_id, plan_id);
CREATE INDEX idx_comp_components_tenant_type ON compensation_components(tenant_id, component_type);
```

### 2.3 Grade Rate Matrices
```sql
CREATE TABLE grade_rate_matrices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rate_name TEXT NOT NULL,
    rate_code TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    grade_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    minimum_rate INTEGER NOT NULL,   -- cents
    midpoint_rate INTEGER NOT NULL,  -- cents
    maximum_rate INTEGER NOT NULL,   -- cents
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rate_code, grade_id, step_number)
);

CREATE INDEX idx_grade_rates_tenant_grade ON grade_rate_matrices(tenant_id, grade_id, step_number);
CREATE INDEX idx_grade_rates_tenant_rate ON grade_rate_matrices(tenant_id, rate_code);
```

### 2.4 Compensation Budgets
```sql
CREATE TABLE compensation_budgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    budget_name TEXT NOT NULL,
    budget_type TEXT NOT NULL CHECK(budget_type IN ('MERIT','LUMP_SUM','BONUS','TOTAL_COMPENSATION','STOCK')),
    allocated_amount INTEGER NOT NULL,  -- cents
    distributed_amount INTEGER NOT NULL DEFAULT 0,
    remaining_amount INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    target_organization_id TEXT,
    target_level TEXT DEFAULT 'ORGANIZATION' CHECK(target_level IN ('ORGANIZATION','DEPARTMENT','BUSINESS_UNIT','LEGAL_EMPLOYER')),
    budget_percentage DECIMAL(5,2),
    fiscal_year INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','APPROVED','ACTIVE','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES compensation_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_comp_budgets_tenant_plan ON compensation_budgets(tenant_id, plan_id);
CREATE INDEX idx_comp_budgets_tenant_org ON compensation_budgets(tenant_id, target_organization_id);
```

### 2.5 Compensation Worksheets
```sql
CREATE TABLE compensation_worksheets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    worksheet_name TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    organization_id TEXT,
    status TEXT NOT NULL DEFAULT 'IN_PROGRESS' CHECK(status IN ('IN_PROGRESS','SUBMITTED','APPROVED','REJECTED','ROLLED_UP')),
    total_allocated INTEGER DEFAULT 0,  -- cents
    employee_count INTEGER DEFAULT 0,
    budget_id TEXT,
    submitted_at TEXT,
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES compensation_plans(id) ON DELETE CASCADE,
    FOREIGN KEY (budget_id) REFERENCES compensation_budgets(id) ON DELETE SET NULL
);

CREATE INDEX idx_comp_ws_tenant_plan ON compensation_worksheets(tenant_id, plan_id);
CREATE INDEX idx_comp_ws_tenant_manager ON compensation_worksheets(tenant_id, manager_id);
```

### 2.6 Compensation Awards
```sql
CREATE TABLE compensation_awards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    worksheet_id TEXT,
    employee_id TEXT NOT NULL,
    assignment_id TEXT NOT NULL,
    component_id TEXT NOT NULL,
    previous_amount INTEGER,      -- cents (current salary/component)
    proposed_amount INTEGER,      -- cents
    approved_amount INTEGER,      -- cents
    percentage_change DECIMAL(7,4),
    performance_rating TEXT,
    compa_ratio DECIMAL(7,4),
    position_in_range DECIMAL(7,4),
    effective_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PROPOSED' CHECK(status IN ('PROPOSED','SUBMITTED','APPROVED','REJECTED','CANCELLED','IMPLEMENTED')),
    approved_by TEXT,
    approved_at TEXT,
    implemented_at TEXT,
    reason_code TEXT,
    comments TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES compensation_plans(id) ON DELETE CASCADE,
    FOREIGN KEY (worksheet_id) REFERENCES compensation_worksheets(id) ON DELETE SET NULL,
    FOREIGN KEY (component_id) REFERENCES compensation_components(id) ON DELETE RESTRICT
);

CREATE INDEX idx_comp_awards_tenant_employee ON compensation_awards(tenant_id, employee_id);
CREATE INDEX idx_comp_awards_tenant_plan ON compensation_awards(tenant_id, plan_id, status);
CREATE INDEX idx_comp_awards_tenant_status ON compensation_awards(tenant_id, status);
```

### 2.7 Salary Survey Data
```sql
CREATE TABLE salary_survey_data (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    survey_source TEXT NOT NULL,
    survey_year INTEGER NOT NULL,
    job_code TEXT NOT NULL,
    job_title TEXT NOT NULL,
    industry TEXT,
    location_code TEXT,
    geographic_area TEXT,
    percentile_10 INTEGER,  -- cents
    percentile_25 INTEGER,
    percentile_50 INTEGER,  -- median
    percentile_75 INTEGER,
    percentile_90 INTEGER,
    mean_salary INTEGER,
    sample_size INTEGER,
    currency_code TEXT DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, survey_source, survey_year, job_code, location_code)
);

CREATE INDEX idx_survey_tenant_job ON salary_survey_data(tenant_id, job_code);
CREATE INDEX idx_survey_tenant_year ON salary_survey_data(tenant_id, survey_year);
```

### 2.8 Stock Option Grants
```sql
CREATE TABLE stock_option_grants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_id TEXT,
    grant_number TEXT NOT NULL,
    grant_type TEXT NOT NULL CHECK(grant_type IN ('STOCK_OPTION','RSU','RESTRICTED_STOCK','PERFORMANCE_SHARE','ESPP','SAR')),
    total_shares INTEGER NOT NULL,
    granted_shares INTEGER NOT NULL,
    exercised_shares INTEGER NOT NULL DEFAULT 0,
    vested_shares INTEGER NOT NULL DEFAULT 0,
    forfeited_shares INTEGER NOT NULL DEFAULT 0,
    grant_price INTEGER NOT NULL,  -- cents per share
    current_price INTEGER,         -- cents per share
    grant_date TEXT NOT NULL,
    vesting_start_date TEXT NOT NULL,
    vesting_end_date TEXT,
    vesting_schedule TEXT,  -- JSON: { type: "CLIFF"|"GRADED", periods: [...] }
    expiration_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','ACTIVE','PARTIALLY_VESTED','FULLY_VESTED','EXERCISED','EXPIRED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, grant_number)
);

CREATE INDEX idx_stock_grants_tenant_employee ON stock_option_grants(tenant_id, employee_id);
CREATE INDEX idx_stock_grants_tenant_status ON stock_option_grants(tenant_id, status);
CREATE INDEX idx_stock_grants_tenant_expiration ON stock_option_grants(tenant_id, expiration_date);
```

### 2.9 Total Compensation Statements
```sql
CREATE TABLE total_compensation_statements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    statement_period_start TEXT NOT NULL,
    statement_period_end TEXT NOT NULL,
    base_salary INTEGER NOT NULL DEFAULT 0,          -- cents
    variable_pay INTEGER NOT NULL DEFAULT 0,
    overtime_pay INTEGER NOT NULL DEFAULT 0,
    employer_benefit_contributions INTEGER NOT NULL DEFAULT 0,
    employer_tax_contributions INTEGER NOT NULL DEFAULT 0,
    stock_grants_value INTEGER NOT NULL DEFAULT 0,
    allowances INTEGER NOT NULL DEFAULT 0,
    other_compensation INTEGER NOT NULL DEFAULT 0,
    total_compensation INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT DEFAULT 'USD',
    generated_at TEXT,
    document_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_id, statement_period_start, statement_period_end)
);

CREATE INDEX idx_total_comp_tenant_employee ON total_compensation_statements(tenant_id, employee_id);
CREATE INDEX idx_total_comp_tenant_period ON total_compensation_statements(tenant_id, statement_period_start);
```

---

## 3. REST API Endpoints

### Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans` | Create a compensation plan |
| GET | `/api/v1/plans` | List compensation plans |
| GET | `/api/v1/plans/{id}` | Get plan with components |
| PUT | `/api/v1/plans/{id}` | Update plan details |
| PATCH | `/api/v1/plans/{id}/activate` | Activate a plan cycle |
| PATCH | `/api/v1/plans/{id}/complete` | Complete a plan cycle |

### Components
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans/{planId}/components` | Add a component to a plan |
| GET | `/api/v1/plans/{planId}/components` | List plan components |
| PUT | `/api/v1/components/{id}` | Update component |

### Grade Rates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/grade-rates` | Create grade rate entry |
| GET | `/api/v1/grade-rates` | List grade rates |
| PUT | `/api/v1/grade-rates/{id}` | Update grade rate |
| GET | `/api/v1/grade-rates/{gradeId}/matrix` | Get full matrix for a grade |

### Budgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/budgets` | Create compensation budget |
| GET | `/api/v1/budgets` | List budgets |
| GET | `/api/v1/budgets/{id}` | Get budget details |
| PUT | `/api/v1/budgets/{id}` | Update budget allocation |
| GET | `/api/v1/budgets/{id}/utilization` | Get budget utilization summary |

### Worksheets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/worksheets` | Create a compensation worksheet |
| GET | `/api/v1/worksheets` | List worksheets |
| GET | `/api/v1/worksheets/{id}` | Get worksheet with employee lines |
| POST | `/api/v1/worksheets/{id}/distribute` | Distribute worksheet to employees |
| POST | `/api/v1/worksheets/{id}/submit` | Submit worksheet for approval |
| POST | `/api/v1/worksheets/{id}/approve` | Approve worksheet |
| POST | `/api/v1/worksheets/{id}/rollup` | Roll up worksheet to higher level |

### Awards
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/awards` | Create a compensation award |
| GET | `/api/v1/awards` | List awards |
| PUT | `/api/v1/awards/{id}` | Update award |
| POST | `/api/v1/awards/{id}/approve` | Approve an award |
| POST | `/api/v1/awards/bulk-approve` | Bulk approve awards |
| GET | `/api/v1/plans/{planId}/awards/summary` | Get awards summary for plan |

### Stock Options
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/stock-grants` | Create stock option grant |
| GET | `/api/v1/stock-grants` | List stock grants |
| GET | `/api/v1/stock-grants/{id}` | Get grant details |
| POST | `/api/v1/stock-grants/{id}/vest` | Process vesting event |
| GET | `/api/v1/employees/{employeeId}/stock-grants` | List employee grants |

### Benchmarking
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/survey-data` | Import salary survey data |
| GET | `/api/v1/survey-data` | List survey data |
| GET | `/api/v1/benchmark/{jobCode}` | Get benchmark comparison |
| GET | `/api/v1/market-analysis` | Get market analysis report |

### Statements
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/statements/generate` | Generate total compensation statements |
| GET | `/api/v1/employees/{employeeId}/statement` | Get employee total comp statement |
| GET | `/api/v1/statements/{id}/download` | Download statement PDF |

---

## 4. Business Rules

1. A compensation plan MUST have at least one component before it can be activated.
2. Budget allocated amounts MUST NOT be negative and distributed amounts MUST NOT exceed allocated amounts.
3. An award's approved amount MUST fall within the min and max values of the associated component.
4. Grade rate entries MUST have minimum rate less than midpoint and midpoint less than maximum rate.
5. Compa-ratio MUST be calculated as (current salary / midpoint rate) and stored for each award.
6. Stock option grants MUST NOT have a total shares value of zero.
7. Total compensation statements MUST aggregate data from payroll, benefits, and stock services.
8. A worksheet MUST be associated with exactly one budget and one plan.
9. Managers MUST NOT view or modify worksheets outside their reporting hierarchy.
10. The system SHOULD support automatic budget allocation based on headcount and current salaries.
11. Vesting schedules for stock grants MUST define at least one vesting period.
12. Compensation changes implemented via awards MUST trigger salary update events to HR and Payroll services.
13. Budget utilization exceeding 90% SHOULD generate a warning notification.
14. Merit increase percentages MUST be validated against the configured eligibility criteria.
15. Survey data used for benchmarking MUST NOT be older than 2 years.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package compensation.v1;

service CompensationService {
    // Plans
    rpc CreatePlan(CreatePlanRequest) returns (CreatePlanResponse);
    rpc GetPlan(GetPlanRequest) returns (GetPlanResponse);
    rpc ListPlans(ListPlansRequest) returns (ListPlansResponse);
    rpc ActivatePlan(ActivatePlanRequest) returns (ActivatePlanResponse);

    // Grade rates
    rpc CreateGradeRate(CreateGradeRateRequest) returns (CreateGradeRateResponse);
    rpc GetGradeMatrix(GetGradeMatrixRequest) returns (GetGradeMatrixResponse);

    // Budgets
    rpc CreateBudget(CreateBudgetRequest) returns (CreateBudgetResponse);
    rpc GetBudgetUtilization(GetBudgetUtilizationRequest) returns (GetBudgetUtilizationResponse);

    // Worksheets
    rpc CreateWorksheet(CreateWorksheetRequest) returns (CreateWorksheetResponse);
    rpc GetWorksheet(GetWorksheetRequest) returns (GetWorksheetResponse);
    rpc SubmitWorksheet(SubmitWorksheetRequest) returns (SubmitWorksheetResponse);

    // Awards
    rpc CreateAward(CreateAwardRequest) returns (CreateAwardResponse);
    rpc ApproveAward(ApproveAwardRequest) returns (ApproveAwardResponse);
    rpc GetEmployeeAwards(GetEmployeeAwardsRequest) returns (GetEmployeeAwardsResponse);

    // Stock
    rpc CreateStockGrant(CreateStockGrantRequest) returns (CreateStockGrantResponse);
    rpc GetStockGrant(GetStockGrantRequest) returns (GetStockGrantResponse);
    rpc ProcessVesting(ProcessVestingRequest) returns (ProcessVestingResponse);

    // Benchmarking
    rpc GetBenchmarkData(GetBenchmarkDataRequest) returns (GetBenchmarkDataResponse);

    // Statements
    rpc GenerateTotalCompStatement(GenerateTotalCompStatementRequest) returns (GenerateTotalCompStatementResponse);
    rpc GetTotalCompStatement(GetTotalCompStatementRequest) returns (GetTotalCompStatementResponse);
}

message CompensationPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_code = 4;
    string plan_type = 5;
    string cycle_status = 6;
    string plan_period_start = 7;
    string plan_period_end = 8;
}

message CreatePlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    string plan_code = 3;
    string plan_type = 4;
    string plan_period_start = 5;
    string plan_period_end = 6;
    string created_by = 7;
}

message CreatePlanResponse {
    CompensationPlan plan = 1;
}

message GetPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetPlanResponse {
    CompensationPlan plan = 1;
}

message ListPlansRequest {
    string tenant_id = 1;
    string cycle_status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListPlansResponse {
    repeated CompensationPlan plans = 1;
    string next_page_token = 2;
}

message ActivatePlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string activated_by = 3;
}

message ActivatePlanResponse {
    CompensationPlan plan = 1;
}

message GradeRateEntry {
    string id = 1;
    string grade_id = 2;
    int32 step_number = 3;
    int64 minimum_rate = 4;
    int64 midpoint_rate = 5;
    int64 maximum_rate = 6;
}

message CreateGradeRateRequest {
    string tenant_id = 1;
    string rate_name = 2;
    string rate_code = 3;
    string grade_id = 4;
    int32 step_number = 5;
    int64 minimum_rate = 6;
    int64 midpoint_rate = 7;
    int64 maximum_rate = 8;
    string created_by = 9;
}

message CreateGradeRateResponse {
    GradeRateEntry entry = 1;
}

message GetGradeMatrixRequest {
    string tenant_id = 1;
    string grade_id = 2;
    string rate_code = 3;
}

message GetGradeMatrixResponse {
    repeated GradeRateEntry entries = 1;
}

message CreateBudgetRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string budget_name = 3;
    string budget_type = 4;
    int64 allocated_amount = 5;
    string target_organization_id = 6;
    string created_by = 7;
}

message CreateBudgetResponse {
    string budget_id = 1;
    int64 allocated_amount = 2;
    int64 remaining_amount = 3;
}

message GetBudgetUtilizationRequest {
    string tenant_id = 1;
    string budget_id = 2;
}

message GetBudgetUtilizationResponse {
    int64 allocated = 1;
    int64 distributed = 2;
    int64 remaining = 3;
    double utilization_percentage = 4;
    int32 employee_count = 5;
}

message CreateWorksheetRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string worksheet_name = 3;
    string manager_id = 4;
    string budget_id = 5;
    string created_by = 6;
}

message CreateWorksheetResponse {
    string worksheet_id = 1;
    int32 employee_count = 2;
}

message GetWorksheetRequest {
    string tenant_id = 1;
    string worksheet_id = 2;
}

message GetWorksheetResponse {
    string id = 1;
    string plan_id = 2;
    string manager_id = 3;
    string status = 4;
    int64 total_allocated = 5;
    int32 employee_count = 6;
}

message SubmitWorksheetRequest {
    string tenant_id = 1;
    string worksheet_id = 2;
    string submitted_by = 3;
}

message SubmitWorksheetResponse {
    string worksheet_id = 1;
    string status = 2;
}

message CreateAwardRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string employee_id = 3;
    string component_id = 4;
    int64 proposed_amount = 5;
    string effective_date = 6;
    string created_by = 7;
}

message CreateAwardResponse {
    string award_id = 1;
    int64 proposed_amount = 2;
    double percentage_change = 3;
}

message ApproveAwardRequest {
    string tenant_id = 1;
    string award_id = 2;
    int64 approved_amount = 3;
    string approved_by = 4;
}

message ApproveAwardResponse {
    string award_id = 1;
    int64 approved_amount = 2;
    string status = 3;
}

message GetEmployeeAwardsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
}

message GetEmployeeAwardsResponse {
    repeated string award_ids = 1;
    int64 total_awarded = 2;
}

message StockGrant {
    string id = 1;
    string employee_id = 2;
    string grant_number = 3;
    string grant_type = 4;
    int64 total_shares = 5;
    int64 granted_shares = 6;
    int64 vested_shares = 7;
    int64 grant_price = 8;
    string status = 9;
}

message CreateStockGrantRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string grant_type = 3;
    int64 total_shares = 4;
    int64 grant_price = 5;
    string grant_date = 6;
    string vesting_start_date = 7;
    string vesting_end_date = 8;
    string created_by = 9;
}

message CreateStockGrantResponse {
    StockGrant grant = 1;
}

message GetStockGrantRequest {
    string tenant_id = 1;
    string grant_id = 2;
}

message GetStockGrantResponse {
    StockGrant grant = 1;
}

message ProcessVestingRequest {
    string tenant_id = 1;
    string grant_id = 2;
    int64 shares_to_vest = 3;
    string vesting_date = 4;
}

message ProcessVestingResponse {
    int64 vested_shares = 1;
    int64 remaining_unvested = 2;
}

message BenchmarkDataPoint {
    string job_code = 1;
    int64 percentile_25 = 2;
    int64 percentile_50 = 3;
    int64 percentile_75 = 4;
    int64 mean_salary = 5;
}

message GetBenchmarkDataRequest {
    string tenant_id = 1;
    string job_code = 2;
    string location_code = 3;
    int32 survey_year = 4;
}

message GetBenchmarkDataResponse {
    repeated BenchmarkDataPoint data_points = 1;
    string survey_source = 2;
}

message GenerateTotalCompStatementRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string period_start = 3;
    string period_end = 4;
    string generated_by = 5;
}

message GenerateTotalCompStatementResponse {
    string statement_id = 1;
    int64 total_compensation = 2;
}

message GetTotalCompStatementRequest {
    string tenant_id = 1;
    string statement_id = 2;
}

message GetTotalCompStatementResponse {
    string id = 1;
    string employee_id = 2;
    int64 base_salary = 3;
    int64 variable_pay = 4;
    int64 employer_benefit_contributions = 5;
    int64 stock_grants_value = 6;
    int64 total_compensation = 7;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, grades, jobs, org hierarchy | Determine eligible population |
| `payroll-service` | Current salary, earnings history | Populate current compensation |
| `performance-service` | Performance ratings | Factor ratings into compensation decisions |
| `benefits-service` | Employer benefit contributions | Calculate total compensation |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Salary changes, grade changes | Update employee records |
| `payroll-service` | New salary amounts, elements | Update payroll calculations |
| `reporting-service` | Compensation analytics | Workforce cost reports |
| `career-service` | Compensation benchmarks | Career path planning |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `CompensationCycleStarted` | `compensation.cycle.started` | `{ tenant_id, plan_id, plan_name, plan_period_start, plan_period_end }` | Published when a compensation plan is activated |
| `AwardsSubmitted` | `compensation.awards.submitted` | `{ tenant_id, plan_id, worksheet_id, employee_count, total_amount }` | Published when worksheet awards are submitted |
| `CompensationBudgetExceeded` | `compensation.budget.exceeded` | `{ tenant_id, budget_id, allocated_amount, distributed_amount, overage_amount }` | Published when budget utilization exceeds allocation |
| `AwardApproved` | `compensation.award.approved` | `{ tenant_id, award_id, employee_id, approved_amount, component_type, effective_date }` | Published when an individual award is approved |
| `AwardImplemented` | `compensation.award.implemented` | `{ tenant_id, award_id, employee_id, new_salary, previous_salary }` | Published when award changes take effect |
| `StockGrantVested` | `compensation.stock.vested` | `{ tenant_id, grant_id, employee_id, vested_shares, total_vested }` | Published when stock shares vest |
| `TotalCompStatementGenerated` | `compensation.statement.generated` | `{ tenant_id, statement_id, employee_id, total_compensation }` | Published when total comp statement is created |
