# 33 - Enterprise Performance Management Service Specification

## 1. Domain Overview

Enterprise Performance Management (EPM) provides planning, budgeting, forecasting, financial consolidation, account reconciliation, cost allocation, and management reporting. Supports multi-scenario planning (baseline, optimistic, pessimistic, what-if), hierarchical legal entity consolidation with currency translation and intercompany elimination, automated account reconciliation workflows, and rule-based cost allocations. Integrates with GL for actuals and journal posting, Intercompany for elimination data, and Workflow for approval processes.

**Bounded Context:** Enterprise Performance Management
**Service Name:** `epm-service`
**Database:** `data/epm.db`
**HTTP Port:** 8060 | **gRPC Port:** 9060

---

## 2. Database Schema

### 2.1 Planning Scenarios
```sql
CREATE TABLE epm_planning_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    scenario_type TEXT NOT NULL
        CHECK(scenario_type IN ('BASELINE','OPTIMISTIC','PESSIMISTIC','WHATIF','CUSTOM')),
    description TEXT,
    fiscal_year TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','LOCKED','ARCHIVED')),
    created_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, scenario_name, fiscal_year)
);

CREATE INDEX idx_planning_scenarios_tenant_year ON epm_planning_scenarios(tenant_id, fiscal_year);
CREATE INDEX idx_planning_scenarios_tenant_status ON epm_planning_scenarios(tenant_id, status);
```

### 2.2 Plan Lines
```sql
CREATE TABLE epm_plan_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    line_type TEXT NOT NULL
        CHECK(line_type IN ('REVENUE','EXPENSE','HEADCOUNT','CAPEX','OTHER')),
    department_id TEXT,
    cost_center_id TEXT,
    description TEXT,
    version INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (scenario_id) REFERENCES epm_planning_scenarios(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, scenario_id, account_id, period_name, line_type, department_id)
);

CREATE INDEX idx_plan_lines_tenant_scenario ON epm_plan_lines(tenant_id, scenario_id);
CREATE INDEX idx_plan_lines_tenant_account ON epm_plan_lines(tenant_id, account_id);
CREATE INDEX idx_plan_lines_tenant_period ON epm_plan_lines(tenant_id, period_name);
```

### 2.3 Plan Assumptions
```sql
CREATE TABLE epm_plan_assumptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    assumption_name TEXT NOT NULL,
    assumption_type TEXT NOT NULL
        CHECK(assumption_type IN ('PERCENTAGE','AMOUNT','RATE','RATIO')),
    value_real REAL NOT NULL DEFAULT 0,
    value_text TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (scenario_id) REFERENCES epm_planning_scenarios(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, scenario_id, assumption_name)
);
```

### 2.4 Consolidation Entities
```sql
CREATE TABLE epm_consolidation_entities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    entity_code TEXT NOT NULL,
    parent_entity_id TEXT,
    ownership_percent REAL NOT NULL DEFAULT 100.0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    is_operating INTEGER NOT NULL DEFAULT 1,
    country_code TEXT,
    chart_of_accounts_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, entity_code)
);

CREATE INDEX idx_consolidation_entities_tenant_parent ON epm_consolidation_entities(tenant_id, parent_entity_id);
```

### 2.5 Consolidation Adjustments
```sql
CREATE TABLE epm_consolidation_adjustments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    consolidation_run_id TEXT NOT NULL,
    adjustment_type TEXT NOT NULL
        CHECK(adjustment_type IN ('ELIMINATION','RECLASS','TRANSLATION','MINORITY_INTEREST')),
    debit_account_id TEXT NOT NULL,
    credit_account_id TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (consolidation_run_id) REFERENCES epm_consolidation_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_consolidation_adjustments_tenant_run ON epm_consolidation_adjustments(tenant_id, consolidation_run_id);
```

### 2.6 Consolidation Runs
```sql
CREATE TABLE epm_consolidation_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_name TEXT NOT NULL,
    period_from TEXT NOT NULL,
    period_to TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED')),
    started_at TEXT,
    completed_at TEXT,
    journal_batch_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, run_name, period_from, period_to)
);

CREATE INDEX idx_consolidation_runs_tenant_status ON epm_consolidation_runs(tenant_id, status);
CREATE INDEX idx_consolidation_runs_tenant_period ON epm_consolidation_runs(tenant_id, period_from, period_to);
```

### 2.7 Reconciliation Items
```sql
CREATE TABLE epm_reconciliation_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    reconciliation_type TEXT NOT NULL
        CHECK(reconciliation_type IN ('BALANCE','TRANSACTION','INTERCOMPANY')),
    status TEXT NOT NULL DEFAULT 'UNPREPARED'
        CHECK(status IN ('UNPREPARED','IN_PROGRESS','SUBMITTED','APPROVED','EXCEPTION')),
    preparer_id TEXT,
    reviewer_id TEXT,
    explanation TEXT,
    risk_rating TEXT NOT NULL DEFAULT 'LOW'
        CHECK(risk_rating IN ('LOW','MEDIUM','HIGH')),
    supporting_document_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, account_id, period_name, reconciliation_type)
);

CREATE INDEX idx_reconciliation_tenant_status ON epm_reconciliation_items(tenant_id, status);
CREATE INDEX idx_reconciliation_tenant_period ON epm_reconciliation_items(tenant_id, period_name);
CREATE INDEX idx_reconciliation_tenant_risk ON epm_reconciliation_items(tenant_id, risk_rating);
```

### 2.8 Allocation Rules
```sql
CREATE TABLE epm_allocation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('PERCENTAGE','EVEN_SPLIT','BASED_ON_STATISTICAL','STEP_DOWN')),
    source_account_id TEXT NOT NULL,
    allocation_basis TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_allocation_rules_tenant_active ON epm_allocation_rules(tenant_id, is_active);
```

### 2.9 Allocation Rule Lines
```sql
CREATE TABLE epm_allocation_rule_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    allocation_rule_id TEXT NOT NULL,
    target_account_id TEXT NOT NULL,
    target_department_id TEXT,
    percentage REAL NOT NULL DEFAULT 0,
    statistical_value REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (allocation_rule_id) REFERENCES epm_allocation_rules(id) ON DELETE CASCADE
);

CREATE INDEX idx_allocation_rule_lines_tenant_rule ON epm_allocation_rule_lines(tenant_id, allocation_rule_id);
```

### 2.10 Budget Versions
```sql
CREATE TABLE epm_budget_versions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    budget_name TEXT NOT NULL,
    fiscal_year TEXT NOT NULL,
    version_number INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','SUPERSEDED')),
    parent_version_id TEXT,
    change_description TEXT,
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    total_expense_cents INTEGER NOT NULL DEFAULT 0,
    net_income_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, budget_name, fiscal_year, version_number)
);

CREATE INDEX idx_budget_versions_tenant_year ON epm_budget_versions(tenant_id, fiscal_year);
CREATE INDEX idx_budget_versions_tenant_status ON epm_budget_versions(tenant_id, status);
```

---

## 3. REST API Endpoints

```
# Planning Scenarios
GET/POST      /api/v1/epm/scenarios                        Permission: epm.scenarios.read/create
GET/PUT       /api/v1/epm/scenarios/{id}                    Permission: epm.scenarios.read/update
POST          /api/v1/epm/scenarios/{id}/copy               Permission: epm.scenarios.create
POST          /api/v1/epm/scenarios/{id}/lock               Permission: epm.scenarios.update

# Plan Lines
GET/POST      /api/v1/epm/scenarios/{id}/lines              Permission: epm.lines.read/create
PUT           /api/v1/epm/scenarios/{id}/lines/bulk         Permission: epm.lines.update

# Plan Assumptions
GET/POST      /api/v1/epm/scenarios/{id}/assumptions        Permission: epm.assumptions.read/create

# Consolidation Entities
GET/POST      /api/v1/epm/consolidation/entities            Permission: epm.consolidation.read/create
GET/PUT       /api/v1/epm/consolidation/entities/{id}        Permission: epm.consolidation.read/update

# Consolidation Runs
POST          /api/v1/epm/consolidation/run                 Permission: epm.consolidation.execute
GET           /api/v1/epm/consolidation/runs                 Permission: epm.consolidation.read
GET           /api/v1/epm/consolidation/runs/{id}/adjustments Permission: epm.consolidation.read
POST          /api/v1/epm/consolidation/runs/{id}/post-to-gl Permission: epm.consolidation.update

# Reconciliation
GET/POST      /api/v1/epm/reconciliation                   Permission: epm.reconciliation.read/create
GET/PUT       /api/v1/epm/reconciliation/{id}               Permission: epm.reconciliation.read/update
POST          /api/v1/epm/reconciliation/{id}/submit        Permission: epm.reconciliation.update
POST          /api/v1/epm/reconciliation/{id}/approve       Permission: epm.reconciliation.approve
POST          /api/v1/epm/reconciliation/auto-match         Permission: epm.reconciliation.update

# Allocation Rules
GET/POST      /api/v1/epm/allocations/rules                 Permission: epm.allocations.read/create
GET/PUT       /api/v1/epm/allocations/rules/{id}            Permission: epm.allocations.read/update

# Allocation Targets
GET/POST      /api/v1/epm/allocations/rules/{id}/targets    Permission: epm.allocations.read/create

# Allocation Execution
POST          /api/v1/epm/allocations/execute               Permission: epm.allocations.execute
GET           /api/v1/epm/allocations/history                Permission: epm.allocations.read

# Budget Versions
GET/POST      /api/v1/epm/budgets                           Permission: epm.budgets.read/create
GET/PUT       /api/v1/epm/budgets/{id}                      Permission: epm.budgets.read/update
POST          /api/v1/epm/budgets/{id}/submit               Permission: epm.budgets.update
POST          /api/v1/epm/budgets/{id}/approve              Permission: epm.budgets.approve
GET           /api/v1/epm/budgets/{id}/vs-actual            Permission: epm.budgets.read
POST          /api/v1/epm/budgets/{id}/version              Permission: epm.budgets.create

# Reports
GET           /api/v1/epm/reports/consolidated-balance-sheet Permission: epm.reports.view
GET           /api/v1/epm/reports/consolidated-income-statement Permission: epm.reports.view
GET           /api/v1/epm/reports/variance-analysis          Permission: epm.reports.view
GET           /api/v1/epm/reports/allocation-summary         Permission: epm.reports.view
```

---

## 4. Business Rules

### 4.1 Consolidation Processing
1. Consolidation runs MUST create elimination journal entries for intercompany transactions
2. Currency translation MUST use period-end rates for balance sheet accounts
3. Currency translation MUST use average rates for income statement accounts
4. Minority interest MUST be calculated based on ownership percentages in consolidation entities
5. Consolidation adjustments auto-generate from intercompany service data
6. Each consolidation run produces a batch of adjustments that can be reviewed before posting to GL

### 4.2 Budget Version Management
1. Budget versions are immutable once approved; changes require a new version
2. Creating a new version copies all lines from the parent version
3. The previous version status changes to SUPERSEDED when a newer version is approved
4. Budget-to-actual comparison uses the latest approved version

### 4.3 Cost Allocation Rules
1. Allocations MUST balance (sum of target percentages = 100%)
2. Step-down allocations execute in sequential order across departments
3. Statistical allocations use driver values from allocation_rule_lines
4. Allocation execution generates GL journal entries for the allocated amounts

### 4.4 Account Reconciliation
1. Reconciliation items with HIGH risk rating MUST require reviewer approval
2. Auto-match attempts to match subledger balances to GL balances
3. Exceptions are flagged when differences exceed configurable thresholds
4. Intercompany reconciliation verifies matching balances across entities

### 4.5 Planning Scenarios
1. Planning scenarios can be copied to create what-if analysis
2. A scenario must be ACTIVE before plan lines can be edited
3. Locking a scenario prevents further modifications to plan lines
4. Assumptions drive calculated plan lines (e.g., revenue growth rate applied to baseline)

### 4.6 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `epm.consolidation.completed` | Consolidation run finishes | GL |
| `epm.budget.approved` | Budget version approved | GL, Reporting |
| `epm.allocation.executed` | Allocation run completes | GL |
| `epm.reconciliation.exception` | Reconciliation discrepancy found | Workflow |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.epm.v1;

service EpmService {
    rpc GetScenario(GetScenarioRequest) returns (GetScenarioResponse);
    rpc CreateConsolidationRun(CreateConsolidationRunRequest) returns (CreateConsolidationRunResponse);
    rpc ExecuteAllocations(ExecuteAllocationsRequest) returns (ExecuteAllocationsResponse);
    rpc GetBudgetVsActual(GetBudgetVsActualRequest) returns (GetBudgetVsActualResponse);
}

message PlanningScenario {
    string id = 1;
    string tenant_id = 2;
    string scenario_name = 3;
    string scenario_type = 4;
    string description = 5;
    string fiscal_year = 6;
    string status = 7;
    string created_by = 8;
    int32 version = 9;
}

message GetScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
}

message GetScenarioResponse {
    PlanningScenario scenario = 1;
}

message ConsolidationRun {
    string id = 1;
    string tenant_id = 2;
    string run_name = 3;
    string period_from = 4;
    string period_to = 5;
    string status = 6;
    string started_at = 7;
    string completed_at = 8;
}

message CreateConsolidationRunRequest {
    string tenant_id = 1;
    string run_name = 2;
    string period_from = 3;
    string period_to = 4;
    string created_by = 5;
}

message CreateConsolidationRunResponse {
    ConsolidationRun run = 1;
}

message AllocationRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string source_account_id = 5;
    string allocation_basis = 6;
    bool is_active = 7;
    string effective_from = 8;
    string effective_to = 9;
}

message ExecuteAllocationsRequest {
    string tenant_id = 1;
    repeated string rule_ids = 2;
    string period_name = 3;
    string executed_by = 4;
}

message ExecuteAllocationsResponse {
    int32 rules_executed = 1;
    int32 journal_entries_created = 2;
    int64 total_allocated_cents = 3;
}

message BudgetVersion {
    string id = 1;
    string tenant_id = 2;
    string budget_name = 3;
    string fiscal_year = 4;
    int32 version_number = 5;
    string status = 6;
    string parent_version_id = 7;
    string change_description = 8;
    int64 total_revenue_cents = 9;
    int64 total_expense_cents = 10;
    int64 net_income_cents = 11;
}

message GetBudgetVsActualRequest {
    string tenant_id = 1;
    string budget_version_id = 2;
}

message BudgetVsActualLine {
    string account_id = 1;
    string period_name = 2;
    int64 budget_amount_cents = 3;
    int64 actual_amount_cents = 4;
    int64 variance_cents = 5;
    double variance_pct = 6;
}

message GetBudgetVsActualResponse {
    BudgetVersion budget = 1;
    repeated BudgetVsActualLine lines = 2;
    int64 total_budget_cents = 3;
    int64 total_actual_cents = 4;
    int64 total_variance_cents = 5;
}
```

---

## 6. Inter-Service Integration

### 6.1 Consumes From
- **GL:** Actual balances for budget-vs-actual comparison, journal posting targets
- **Intercompany:** Intercompany transaction data for consolidation eliminations
- **Auth:** User data for preparer/reviewer assignment
- **Workflow:** Approval processes for budget submissions and reconciliation

### 6.2 Publishes To
- **GL:** Consolidation adjustment journal entries, allocation journal entries
- **Reporting:** Budget data, consolidation results, variance analysis
- **Workflow:** Budget approval requests, reconciliation review tasks

---

## 7. Migrations

1. V001: `epm_planning_scenarios`
2. V002: `epm_plan_lines`
3. V003: `epm_plan_assumptions`
4. V004: `epm_consolidation_entities`
5. V005: `epm_consolidation_runs`
6. V006: `epm_consolidation_adjustments`
7. V007: `epm_reconciliation_items`
8. V008: `epm_allocation_rules`
9. V009: `epm_allocation_rule_lines`
10. V010: `epm_budget_versions`
11. V011: Triggers for `updated_at`
