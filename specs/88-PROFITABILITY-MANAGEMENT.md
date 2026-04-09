# 88 - Profitability Management Service Specification

## 1. Domain Overview

Profitability Management provides activity-based costing (ABC), profitability analysis by multiple dimensions (product, customer, channel, region), cost pool definition and driver-based allocation, multi-step allocation rules, margin calculation and analysis, and what-if scenario simulation. Supports creation of profitability models with configurable cost pools and drivers, assignment of direct costs and allocation of indirect costs to segments, calculation of contribution margin and net margin by dimension, period-over-period comparison, and scenario modeling for pricing and volume changes. Integrates with GL for financial actuals, COST-MANAGEMENT for cost data, and EPM for planning and budgeting.

**Bounded Context:** Activity-Based Costing & Profitability Analysis
**Service Name:** `profitability-service`
**Database:** `data/profitability.db`
**HTTP Port:** 8120 | **gRPC Port:** 9120

---

## 2. Database Schema

### 2.1 Profitability Models
```sql
CREATE TABLE profitability_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL DEFAULT 'ABC'
        CHECK(model_type IN ('ABC','STANDARD_COST','MARGINAL_COST','THROUGHPUT','CUSTOM')),
    description TEXT,
    fiscal_year TEXT NOT NULL,
    period_type TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(period_type IN ('MONTHLY','QUARTERLY','ANNUAL')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','LOCKED','ARCHIVED')),
    base_currency TEXT NOT NULL DEFAULT 'USD',
    allocation_methodology TEXT NOT NULL DEFAULT 'SEQUENTIAL'
        CHECK(allocation_methodology IN ('SEQUENTIAL','RECIPROCAL','SIMULTANEOUS')),
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_margin_cents INTEGER NOT NULL DEFAULT 0,
    overall_margin_pct_real REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_name, fiscal_year)
);

CREATE INDEX idx_models_tenant_status ON profitability_models(tenant_id, status);
CREATE INDEX idx_models_tenant_year ON profitability_models(tenant_id, fiscal_year);
CREATE INDEX idx_models_tenant_type ON profitability_models(tenant_id, model_type);
```

### 2.2 Cost Pools
```sql
CREATE TABLE profitability_cost_pools (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    pool_name TEXT NOT NULL,
    pool_type TEXT NOT NULL
        CHECK(pool_type IN ('MANUFACTURING_OVERHEAD','SELLING','GENERAL_ADMIN','DISTRIBUTION','R_AND_D','CUSTOMER_SERVICE','IT','HR','FACILITIES','CUSTOM')),
    pool_category TEXT NOT NULL DEFAULT 'INDIRECT'
        CHECK(pool_category IN ('DIRECT','INDIRECT')),
    gl_account_id TEXT,
    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    allocated_amount_cents INTEGER NOT NULL DEFAULT 0,
    unallocated_amount_cents INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, pool_name)
);

CREATE INDEX idx_pools_tenant_model ON profitability_cost_pools(tenant_id, model_id);
CREATE INDEX idx_pools_tenant_type ON profitability_cost_pools(tenant_id, pool_type);
CREATE INDEX idx_pools_tenant_category ON profitability_cost_pools(tenant_id, pool_category);
```

### 2.3 Cost Drivers
```sql
CREATE TABLE profitability_cost_drivers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    driver_name TEXT NOT NULL,
    driver_type TEXT NOT NULL
        CHECK(driver_type IN ('TRANSACTION_COUNT','LABOR_HOURS','MACHINE_HOURS','UNIT_COUNT','REVENUE','FLOOR_SPACE','HEADCOUNT','CUSTOM')),
    unit_of_measure TEXT NOT NULL,
    driver_rate_cents INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, driver_name)
);

CREATE INDEX idx_drivers_tenant_model ON profitability_cost_drivers(tenant_id, model_id);
CREATE INDEX idx_drivers_tenant_type ON profitability_cost_drivers(tenant_id, driver_type);
CREATE INDEX idx_drivers_tenant_active ON profitability_cost_drivers(tenant_id, is_active);
```

### 2.4 Driver Quantities
```sql
CREATE TABLE profitability_driver_quantities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    driver_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    quantity_real REAL NOT NULL DEFAULT 0.0,
    source_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','IMPORTED','CALCULATED','GL_DERIVED')),
    source_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (driver_id) REFERENCES profitability_cost_drivers(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, driver_id, segment_id, period_name)
);

CREATE INDEX idx_quantities_tenant_driver ON profitability_driver_quantities(tenant_id, driver_id);
CREATE INDEX idx_quantities_tenant_segment ON profitability_driver_quantities(tenant_id, segment_id);
CREATE INDEX idx_quantities_tenant_period ON profitability_driver_quantities(tenant_id, period_name);
```

### 2.5 Allocation Rules
```sql
CREATE TABLE profitability_allocation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    pool_id TEXT NOT NULL,
    driver_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    allocation_method TEXT NOT NULL DEFAULT 'PROPORTIONAL'
        CHECK(allocation_method IN ('PROPORTIONAL','WEIGHTED','FIXED_PERCENTAGE','STEP_DOWN')),
    fixed_percentages TEXT,                        -- JSON: segment_id -> percentage (for FIXED_PERCENTAGE)
    allocation_order INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    FOREIGN KEY (pool_id) REFERENCES profitability_cost_pools(id),
    FOREIGN KEY (driver_id) REFERENCES profitability_cost_drivers(id),
    UNIQUE(tenant_id, model_id, rule_name)
);

CREATE INDEX idx_rules_tenant_model ON profitability_allocation_rules(tenant_id, model_id);
CREATE INDEX idx_rules_tenant_pool ON profitability_allocation_rules(tenant_id, pool_id);
CREATE INDEX idx_rules_tenant_driver ON profitability_allocation_rules(tenant_id, driver_id);
CREATE INDEX idx_rules_tenant_active ON profitability_allocation_rules(tenant_id, is_active);
CREATE INDEX idx_rules_tenant_order ON profitability_allocation_rules(tenant_id, allocation_order);
```

### 2.6 Profitability Segments
```sql
CREATE TABLE profitability_segments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    segment_code TEXT NOT NULL,
    segment_name TEXT NOT NULL,
    segment_type TEXT NOT NULL
        CHECK(segment_type IN ('PRODUCT','CUSTOMER','CHANNEL','REGION','BUSINESS_UNIT','PROJECT','CUSTOM')),
    parent_segment_id TEXT,
    segment_level INTEGER NOT NULL DEFAULT 0,
    attributes TEXT,                               -- JSON: extensible segment attributes
    is_leaf INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, segment_code)
);

CREATE INDEX idx_segments_tenant_model ON profitability_segments(tenant_id, model_id);
CREATE INDEX idx_segments_tenant_type ON profitability_segments(tenant_id, segment_type);
CREATE INDEX idx_segments_tenant_parent ON profitability_segments(tenant_id, parent_segment_id);
CREATE INDEX idx_segments_tenant_active ON profitability_segments(tenant_id, is_active);
```

### 2.7 Segment Revenues
```sql
CREATE TABLE profitability_segment_revenues (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    revenue_type TEXT NOT NULL DEFAULT 'PRODUCT'
        CHECK(revenue_type IN ('PRODUCT','SERVICE','SUBSCRIPTION','OTHER')),
    gross_revenue_cents INTEGER NOT NULL DEFAULT 0,
    discounts_cents INTEGER NOT NULL DEFAULT 0,
    returns_cents INTEGER NOT NULL DEFAULT 0,
    net_revenue_cents INTEGER NOT NULL DEFAULT 0,
    volume_units INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    source_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','GL_DERIVED','IMPORTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    FOREIGN KEY (segment_id) REFERENCES profitability_segments(id),
    UNIQUE(tenant_id, model_id, segment_id, period_name, revenue_type)
);

CREATE INDEX idx_revenues_tenant_model ON profitability_segment_revenues(tenant_id, model_id);
CREATE INDEX idx_revenues_tenant_segment ON profitability_segment_revenues(tenant_id, segment_id);
CREATE INDEX idx_revenues_tenant_period ON profitability_segment_revenues(tenant_id, period_name);
```

### 2.8 Segment Costs
```sql
CREATE TABLE profitability_segment_costs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    cost_type TEXT NOT NULL DEFAULT 'DIRECT'
        CHECK(cost_type IN ('DIRECT_MATERIAL','DIRECT_LABOR','DIRECT_EXPENSE','ALLOCATED_OVERHEAD','ALLOCATED_SELLING','ALLOCATED_GA','OTHER')),
    cost_category TEXT NOT NULL
        CHECK(cost_category IN ('VARIABLE','FIXED','SEMI_VARIABLE','STEP')),
    amount_cents INTEGER NOT NULL DEFAULT 0,
    allocation_rule_id TEXT,
    is_allocated INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    source_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','GL_DERIVED','ALLOCATED','IMPORTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    FOREIGN KEY (segment_id) REFERENCES profitability_segments(id),
    UNIQUE(tenant_id, model_id, segment_id, period_name, cost_type)
);

CREATE INDEX idx_costs_tenant_model ON profitability_segment_costs(tenant_id, model_id);
CREATE INDEX idx_costs_tenant_segment ON profitability_segment_costs(tenant_id, segment_id);
CREATE INDEX idx_costs_tenant_period ON profitability_segment_costs(tenant_id, period_name);
CREATE INDEX idx_costs_tenant_type ON profitability_segment_costs(tenant_id, cost_type);
CREATE INDEX idx_costs_tenant_allocated ON profitability_segment_costs(tenant_id, is_allocated);
```

### 2.9 Profitability Calculations
```sql
CREATE TABLE profitability_calculations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    calculation_type TEXT NOT NULL DEFAULT 'FULL'
        CHECK(calculation_type IN ('FULL','INCREMENTAL','CONTRIBUTION')),

    net_revenue_cents INTEGER NOT NULL DEFAULT 0,
    direct_cost_cents INTEGER NOT NULL DEFAULT 0,
    contribution_margin_cents INTEGER NOT NULL DEFAULT 0,
    contribution_margin_pct REAL NOT NULL DEFAULT 0.0,

    allocated_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    gross_margin_cents INTEGER NOT NULL DEFAULT 0,
    gross_margin_pct REAL NOT NULL DEFAULT 0.0,

    net_margin_cents INTEGER NOT NULL DEFAULT 0,
    net_margin_pct REAL NOT NULL DEFAULT 0.0,
    roi_pct REAL NOT NULL DEFAULT 0.0,

    revenue_per_unit_cents INTEGER NOT NULL DEFAULT 0,
    cost_per_unit_cents INTEGER NOT NULL DEFAULT 0,
    profit_per_unit_cents INTEGER NOT NULL DEFAULT 0,
    volume_units INTEGER NOT NULL DEFAULT 0,

    calculated_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    FOREIGN KEY (segment_id) REFERENCES profitability_segments(id),
    UNIQUE(tenant_id, model_id, segment_id, period_name, calculation_type)
);

CREATE INDEX idx_calculations_tenant_model ON profitability_calculations(tenant_id, model_id);
CREATE INDEX idx_calculations_tenant_segment ON profitability_calculations(tenant_id, segment_id);
CREATE INDEX idx_calculations_tenant_period ON profitability_calculations(tenant_id, period_name);
CREATE INDEX idx_calculations_tenant_type ON profitability_calculations(tenant_id, calculation_type);
```

### 2.10 Profitability Reports
```sql
CREATE TABLE profitability_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    report_name TEXT NOT NULL,
    report_type TEXT NOT NULL
        CHECK(report_type IN ('MARGIN_BY_PRODUCT','MARGIN_BY_CUSTOMER','MARGIN_BY_CHANNEL','MARGIN_BY_REGION','WATERFALL','PARETO','TREND','CUSTOM')),
    period_from TEXT NOT NULL,
    period_to TEXT NOT NULL,
    segment_type TEXT,
    filters TEXT,                                  -- JSON: filter criteria
    sort_by TEXT NOT NULL DEFAULT 'NET_MARGIN_PCT',
    sort_direction TEXT NOT NULL DEFAULT 'DESC'
        CHECK(sort_direction IN ('ASC','DESC')),
    include_details INTEGER NOT NULL DEFAULT 0,
    generated_at TEXT,
    report_data TEXT,                              -- JSON: calculated report results

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE
);

CREATE INDEX idx_reports_tenant_model ON profitability_reports(tenant_id, model_id);
CREATE INDEX idx_reports_tenant_type ON profitability_reports(tenant_id, report_type);
CREATE INDEX idx_reports_tenant_dates ON profitability_reports(tenant_id, period_from, period_to);
```

### 2.11 What-If Scenarios
```sql
CREATE TABLE profitability_whatif_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    base_period TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','CALCULATED','APPROVED','ARCHIVED')),

    revenue_change_pct REAL NOT NULL DEFAULT 0.0,
    volume_change_pct REAL NOT NULL DEFAULT 0.0,
    cost_change_pct REAL NOT NULL DEFAULT 0.0,
    price_change_pct REAL NOT NULL DEFAULT 0.0,
    mix_change TEXT,                               -- JSON: segment mix adjustments

    base_margin_cents INTEGER NOT NULL DEFAULT 0,
    base_margin_pct REAL NOT NULL DEFAULT 0.0,
    scenario_margin_cents INTEGER NOT NULL DEFAULT 0,
    scenario_margin_pct REAL NOT NULL DEFAULT 0.0,
    margin_delta_cents INTEGER NOT NULL DEFAULT 0,
    margin_delta_pct REAL NOT NULL DEFAULT 0.0,

    segment_overrides TEXT,                        -- JSON: segment-level overrides
    calculated_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES profitability_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, scenario_name)
);

CREATE INDEX idx_whatif_tenant_model ON profitability_whatif_scenarios(tenant_id, model_id);
CREATE INDEX idx_whatif_tenant_status ON profitability_whatif_scenarios(tenant_id, status);
```

---

## 3. REST API Endpoints

```
# Profitability Models
GET/POST      /api/v1/profitability/models                    Permission: profitability.models.read/create
GET/PUT       /api/v1/profitability/models/{id}               Permission: profitability.models.read/update
PATCH         /api/v1/profitability/models/{id}/lock          Permission: profitability.models.update
DELETE        /api/v1/profitability/models/{id}               Permission: profitability.models.delete

# Cost Pools
GET/POST      /api/v1/profitability/models/{id}/cost-pools    Permission: profitability.pools.read/create
GET/PUT       /api/v1/profitability/cost-pools/{id}           Permission: profitability.pools.read/update
DELETE        /api/v1/profitability/cost-pools/{id}           Permission: profitability.pools.delete

# Cost Drivers
GET/POST      /api/v1/profitability/models/{id}/drivers       Permission: profitability.drivers.read/create
GET/PUT       /api/v1/profitability/drivers/{id}              Permission: profitability.drivers.read/update
DELETE        /api/v1/profitability/drivers/{id}              Permission: profitability.drivers.delete

# Driver Quantities
GET/POST      /api/v1/profitability/drivers/{id}/quantities   Permission: profitability.quantities.read/create
PUT           /api/v1/profitability/quantities/bulk           Permission: profitability.quantities.update
GET           /api/v1/profitability/quantities/period/{name}  Permission: profitability.quantities.read

# Allocation Rules
GET/POST      /api/v1/profitability/models/{id}/allocations   Permission: profitability.allocations.read/create
GET/PUT       /api/v1/profitability/allocations/{id}          Permission: profitability.allocations.read/update
POST          /api/v1/profitability/allocations/execute       Permission: profitability.allocations.execute
GET           /api/v1/profitability/allocations/history        Permission: profitability.allocations.read
DELETE        /api/v1/profitability/allocations/{id}          Permission: profitability.allocations.delete

# Profitability Segments
GET/POST      /api/v1/profitability/models/{id}/segments      Permission: profitability.segments.read/create
GET/PUT       /api/v1/profitability/segments/{id}             Permission: profitability.segments.read/update
GET           /api/v1/profitability/models/{id}/segments/tree Permission: profitability.segments.read
DELETE        /api/v1/profitability/segments/{id}             Permission: profitability.segments.delete

# Segment Revenues
GET/POST      /api/v1/profitability/segments/{id}/revenues    Permission: profitability.revenues.read/create
PUT           /api/v1/profitability/revenues/bulk             Permission: profitability.revenues.update
GET           /api/v1/profitability/revenues/period/{name}    Permission: profitability.revenues.read

# Segment Costs
GET/POST      /api/v1/profitability/segments/{id}/costs       Permission: profitability.costs.read/create
PUT           /api/v1/profitability/costs/bulk                Permission: profitability.costs.update
GET           /api/v1/profitability/costs/period/{name}       Permission: profitability.costs.read

# Profitability Calculations
POST          /api/v1/profitability/models/{id}/calculate     Permission: profitability.calculate.execute
GET           /api/v1/profitability/models/{id}/calculations  Permission: profitability.calculate.read
GET           /api/v1/profitability/calculations/{id}         Permission: profitability.calculate.read
GET           /api/v1/profitability/models/{id}/margins       Permission: profitability.calculate.read

# What-If Scenarios
GET/POST      /api/v1/profitability/models/{id}/scenarios    Permission: profitability.scenarios.read/create
GET/PUT       /api/v1/profitability/scenarios/{id}           Permission: profitability.scenarios.read/update
POST          /api/v1/profitability/scenarios/{id}/run        Permission: profitability.scenarios.execute
GET           /api/v1/profitability/scenarios/{id}/comparison Permission: profitability.scenarios.read
DELETE        /api/v1/profitability/scenarios/{id}           Permission: profitability.scenarios.delete

# Reports
GET/POST      /api/v1/profitability/reports                   Permission: profitability.reports.read/create
GET           /api/v1/profitability/reports/{id}              Permission: profitability.reports.read
GET           /api/v1/profitability/reports/{id}/export       Permission: profitability.reports.export
GET           /api/v1/profitability/reports/margin-analysis   Permission: profitability.reports.view
GET           /api/v1/profitability/reports/waterfall         Permission: profitability.reports.view
GET           /api/v1/profitability/reports/trend             Permission: profitability.reports.view
GET           /api/v1/profitability/reports/pareto            Permission: profitability.reports.view
```

---

## 4. Business Rules

### 4.1 Model Management
1. A profitability model MUST define at least one cost pool and one segment before calculations can run
2. Models with status LOCKED cannot have their structure modified; only driver quantities may be updated
3. Only one model per fiscal year MAY be designated as the primary model
4. Model allocation_methodology determines the order and method of cost pool allocation
5. The overall margin percentage MUST be recalculated whenever underlying data changes
6. Archived models retain all calculation results for historical reference

### 4.2 Cost Pool Rules
1. Each cost pool MUST be linked to either a GL account or have a manually entered total amount
2. Direct cost pools bypass the allocation process and are assigned directly to segments
3. Indirect cost pools MUST have at least one allocation rule defined before allocation can run
4. The total allocated amount MUST equal the total pool amount (no unallocated balance permitted)
5. Cost pools with zero amounts SHOULD be flagged for review
6. Pool amounts are entered in cents to avoid floating-point precision issues

### 4.3 Driver and Allocation Rules
1. Each allocation rule MUST reference exactly one cost pool and one cost driver
2. Driver quantities MUST be entered for every segment before proportional allocation runs
3. Fixed percentage allocations MUST sum to 100% across all target segments
4. Step-down allocations MUST execute in the order defined by allocation_order
5. Reciprocal allocation methodology MUST use iterative calculation for mutual cost sharing
6. Driver rates are computed as: pool amount / total driver quantity

### 4.4 Segment Management
1. Segments MUST have a unique code within a model
2. Segments support hierarchical grouping via parent_segment_id
3. Leaf segments (is_leaf = 1) receive cost allocations; parent segments aggregate children
4. Segment types determine the analytical dimension (product, customer, channel, region)
5. Multiple segment types MAY coexist in a single model for multi-dimensional analysis
6. Deactivated segments MUST NOT receive new allocations but retain historical data

### 4.5 Revenue and Cost Assignment
1. Net revenue MUST be calculated as: gross revenue - discounts - returns
2. Direct costs MUST be assigned to segments before allocated costs are computed
3. Contribution margin is: net revenue - direct costs
4. Gross margin is: net revenue - direct costs - allocated costs
5. Net margin includes all cost categories (direct + all allocated overhead)
6. Revenue and cost amounts MUST use the same currency as the model's base currency

### 4.6 What-If Scenarios
1. What-if scenarios MUST reference a base period from the model's actual data
2. Scenario parameters (revenue, volume, cost, price changes) are expressed as percentage adjustments
3. Segment overrides allow per-segment adjustments beyond global percentages
4. Calculated scenario results MUST show the delta (change) compared to the base period
5. Scenarios with status CALCULATED are read-only; further changes require resetting to DRAFT
6. Multiple scenarios MAY be compared side-by-side for decision support

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `profitability.model.created` | New model created | EPM, Reporting |
| `profitability.allocation.run` | Allocation execution begins | Notification |
| `profitability.allocation.completed` | Allocation finishes | GL, Reporting |
| `profitability.calculated` | Profitability calculation completes | EPM, Reporting |
| `profitability.scenario.modeled` | What-if scenario calculated | EPM, Notification |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.profitability.v1;

service ProfitabilityService {
    rpc GetModel(GetModelRequest) returns (GetModelResponse);
    rpc RunAllocation(RunAllocationRequest) returns (RunAllocationResponse);
    rpc CalculateProfitability(CalculateProfitabilityRequest) returns (CalculateProfitabilityResponse);
    rpc GetMarginAnalysis(GetMarginAnalysisRequest) returns (GetMarginAnalysisResponse);
    rpc RunWhatIfScenario(RunWhatIfScenarioRequest) returns (RunWhatIfScenarioResponse);
    rpc ComparePeriods(ComparePeriodsRequest) returns (ComparePeriodsResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **GL:** Trial balance data, account balances for revenue and cost pool amounts, journal entry details
- **COST-MANAGEMENT:** Standard cost data, overhead rates, cost center allocations
- **EPM:** Planning budgets and forecasts for scenario comparison, planned cost rates
- **Auth:** User identity for model ownership and calculation permissions
- **Inventory:** Product cost data for direct material cost assignment
- **Order-Management:** Revenue data by product, customer, and channel for segment revenue assignment

### 6.2 Data Published To
- **GL:** Allocation journal entries from cost pool distributions
- **EPM:** Profitability results for budget variance analysis and forecast adjustments
- **Reporting:** Margin analysis data, waterfall charts, trend data for executive dashboards
- **Notification:** Calculation completion alerts, allocation warnings, threshold breach notifications

---

## 7. Migrations

1. V001: `profitability_models`
2. V002: `profitability_cost_pools`
3. V003: `profitability_cost_drivers`
4. V004: `profitability_driver_quantities`
5. V005: `profitability_allocation_rules`
6. V006: `profitability_segments`
7. V007: `profitability_segment_revenues`
8. V008: `profitability_segment_costs`
9. V009: `profitability_calculations`
10. V010: `profitability_reports`
11. V011: `profitability_whatif_scenarios`
12. V012: Triggers for `updated_at`
