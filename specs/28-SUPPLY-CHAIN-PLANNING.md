# 28 - Supply Chain Planning / MRP Service Specification

## 1. Domain Overview

Supply Chain Planning provides Material Requirements Planning (MRP), demand forecasting, supply-demand matching, planned order generation, safety stock calculations, and what-if scenario analysis. Integrates with OM for demand, INV for on-hand data, Procurement for purchase orders, Manufacturing for work orders and BOMs, and GL for planning-related accounting.

**Bounded Context:** Supply Chain Planning & MRP
**Service Name:** `planning-service`
**Database:** `data/planning.db`
**HTTP Port:** 8027 | **gRPC Port:** 9027

---

## 2. Database Schema

### 2.1 Planning Calendars
```sql
CREATE TABLE planning_calendars (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    calendar_name TEXT NOT NULL,
    calendar_type TEXT NOT NULL DEFAULT 'WORKING'
        CHECK(calendar_type IN ('WORKING','HOLIDAY','SHIFT')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    monday INTEGER NOT NULL DEFAULT 1,
    tuesday INTEGER NOT NULL DEFAULT 1,
    wednesday INTEGER NOT NULL DEFAULT 1,
    thursday INTEGER NOT NULL DEFAULT 1,
    friday INTEGER NOT NULL DEFAULT 1,
    saturday INTEGER NOT NULL DEFAULT 0,
    sunday INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, calendar_name)
);

CREATE TABLE calendar_exceptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    calendar_id TEXT NOT NULL,
    exception_date TEXT NOT NULL,
    exception_type TEXT NOT NULL CHECK(exception_type IN ('HOLIDAY','SHUTDOWN','OVERTIME')),
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (calendar_id) REFERENCES planning_calendars(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, calendar_id, exception_date)
);
```

### 2.2 MRP Parameters (per item-warehouse)
```sql
CREATE TABLE mrp_parameters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,

    -- Planning method
    planning_method TEXT NOT NULL DEFAULT 'MRP'
        CHECK(planning_method IN ('MRP','MPS','KANBAN','REORDER_POINT','MANUAL')),

    -- Lot sizing
    lot_size_method TEXT NOT NULL DEFAULT 'FIXED'
        CHECK(lot_size_method IN ('FIXED','EOQ','PERIOD_ORDER_QUANTITY','MIN_MAX','LOT_FOR_LOT')),
    fixed_lot_quantity DECIMAL(18,4),
    min_order_quantity DECIMAL(18,4) DEFAULT 0,
    max_order_quantity DECIMAL(18,4),
    order_multiple DECIMAL(18,4) DEFAULT 1,

    -- Lead time
    fixed_lead_time_days INTEGER NOT NULL DEFAULT 0,
    variable_lead_time_days REAL DEFAULT 0,

    -- Safety stock
    safety_stock_method TEXT NOT NULL DEFAULT 'FIXED'
        CHECK(safety_stock_method IN ('FIXED','SERVICE_LEVEL','TIME_BASED')),
    safety_stock_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    service_level_percent REAL DEFAULT 95,
    safety_time_days INTEGER DEFAULT 0,

    -- Time fences
    planning_time_fence_days INTEGER DEFAULT 0,
    demand_time_fence_days INTEGER DEFAULT 0,

    -- Order policy
    order_policy TEXT NOT NULL DEFAULT 'PURCHASE'
        CHECK(order_policy IN ('PURCHASE','MANUFACTURE','TRANSFER')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, item_id, warehouse_id)
);
```

### 2.3 Demand Forecasts
```sql
CREATE TABLE demand_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    period_start_date TEXT NOT NULL,
    period_end_date TEXT NOT NULL,
    forecast_method TEXT NOT NULL
        CHECK(forecast_method IN ('MOVING_AVERAGE','EXPONENTIAL_SMOOTHING','WEIGHTED_AVERAGE','SEASONAL','MANUAL','COLLABORATIVE')),
    forecasted_demand DECIMAL(18,4) NOT NULL DEFAULT 0,
    actual_demand DECIMAL(18,4),
    accuracy_percent REAL,
    confidence_level TEXT DEFAULT 'MEDIUM'
        CHECK(confidence_level IN ('HIGH','MEDIUM','LOW')),
    forecast_set_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, item_id, warehouse_id, period_start_date, forecast_set_id)
);

CREATE INDEX idx_forecasts_tenant_item ON demand_forecasts(tenant_id, item_id);
CREATE INDEX idx_forecasts_tenant_dates ON demand_forecasts(tenant_id, period_start_date);
```

### 2.4 Forecast Methods Configuration
```sql
CREATE TABLE forecast_methods_config (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    method_name TEXT NOT NULL,
    method_type TEXT NOT NULL,
    parameters TEXT NOT NULL,               -- JSON: method-specific parameters
    periods_to_analyze INTEGER NOT NULL DEFAULT 12,
    seasonality_period INTEGER,
    smoothing_factor REAL,
    trend_factor REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, method_name)
);
```

### 2.5 MRP Runs
```sql
CREATE TABLE mrp_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_number TEXT NOT NULL,               -- MRP-2024-00001
    run_date TEXT NOT NULL,
    planning_horizon_start TEXT NOT NULL,
    planning_horizon_end TEXT NOT NULL,
    scope_type TEXT NOT NULL DEFAULT 'FULL'
        CHECK(scope_type IN ('FULL','ITEM','WAREHOUSE','CATEGORY')),
    scope_value TEXT,                       -- Item ID, warehouse ID, or category ID
    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED','CANCELLED')),

    total_items_planned INTEGER NOT NULL DEFAULT 0,
    planned_orders_generated INTEGER NOT NULL DEFAULT 0,
    action_messages INTEGER NOT NULL DEFAULT 0,
    execution_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, run_number)
);

CREATE INDEX idx_mrp_runs_tenant_status ON mrp_runs(tenant_id, status);
CREATE INDEX idx_mrp_runs_tenant_date ON mrp_runs(tenant_id, run_date);
```

### 2.6 MRP Supplies
```sql
CREATE TABLE mrp_supplies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mrp_run_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    supply_type TEXT NOT NULL
        CHECK(supply_type IN ('ON_HAND','PURCHASE_ORDER','WORK_ORDER','PLANNED_ORDER','TRANSFER_ORDER')),
    supply_document_id TEXT,
    quantity DECIMAL(18,4) NOT NULL,
    due_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','FIRMED','RELEASED','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (mrp_run_id) REFERENCES mrp_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_mrp_supplies_tenant_item ON mrp_supplies(tenant_id, item_id);
CREATE INDEX idx_mrp_supplies_tenant_date ON mrp_supplies(tenant_id, due_date);
```

### 2.7 MRP Demands
```sql
CREATE TABLE mrp_demands (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mrp_run_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    demand_type TEXT NOT NULL
        CHECK(demand_type IN ('SALES_ORDER','FORECAST','SAFETY_STOCK','WORK_ORDER_COMPONENT','TRANSFER_REQUIREMENT')),
    demand_document_id TEXT,
    quantity DECIMAL(18,4) NOT NULL,
    required_date TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 5,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (mrp_run_id) REFERENCES mrp_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_mrp_demands_tenant_item ON mrp_demands(tenant_id, item_id);
CREATE INDEX idx_mrp_demands_tenant_date ON mrp_demands(tenant_id, required_date);
```

### 2.8 Planned Orders
```sql
CREATE TABLE planned_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mrp_run_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    order_type TEXT NOT NULL
        CHECK(order_type IN ('PURCHASE','MANUFACTURE','TRANSFER')),
    quantity DECIMAL(18,4) NOT NULL,
    due_date TEXT NOT NULL,
    release_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','FIRMED','RELEASED','CANCELLED')),

    -- Pegging (links to demand)
    pegged_demand_id TEXT,
    pegged_demand_type TEXT,

    -- Source (for converting to actual orders)
    converted_document_id TEXT,
    converted_document_type TEXT,

    -- Action messages
    action_message TEXT,
    action_details TEXT,                    -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (mrp_run_id) REFERENCES mrp_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_planned_orders_tenant_item ON planned_orders(tenant_id, item_id);
CREATE INDEX idx_planned_orders_tenant_status ON planned_orders(tenant_id, status);
CREATE INDEX idx_planned_orders_tenant_dates ON planned_orders(tenant_id, due_date, release_date);
```

### 2.9 What-If Scenarios
```sql
CREATE TABLE scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    description TEXT,
    baseline_scenario_id TEXT,              -- Compare against baseline
    scenario_type TEXT NOT NULL DEFAULT 'DEMAND'
        CHECK(scenario_type IN ('DEMAND','SUPPLY','CAPACITY','MIXED')),
    parameters TEXT NOT NULL,               -- JSON: modified assumptions
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','RUNNING','COMPLETED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, scenario_name)
);

CREATE TABLE scenario_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    period_date TEXT NOT NULL,
    projected_available DECIMAL(18,4) NOT NULL DEFAULT 0,
    planned_orders_count INTEGER NOT NULL DEFAULT 0,
    shortage_quantity DECIMAL(18,4) DEFAULT 0,
    excess_quantity DECIMAL(18,4) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (scenario_id) REFERENCES scenarios(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, scenario_id, item_id, warehouse_id, period_date)
);
```

---

## 3. REST API Endpoints

```
# Planning Calendars
GET/POST      /api/v1/planning/calendars               Permission: planning.calendars.read/create
GET/PUT       /api/v1/planning/calendars/{id}           Permission: planning.calendars.read/update
GET/POST      /api/v1/planning/calendars/{id}/exceptions Permission: planning.calendars.read/create

# MRP Parameters
GET/POST      /api/v1/planning/parameters               Permission: planning.parameters.read/create
GET/PUT       /api/v1/planning/parameters/{id}           Permission: planning.parameters.read/update

# Demand Forecasts
GET/POST      /api/v1/planning/forecasts                 Permission: planning.forecasts.read/create
PUT           /api/v1/planning/forecasts/{id}             Permission: planning.forecasts.update
POST          /api/v1/planning/forecasts/generate        Permission: planning.forecasts.create
GET           /api/v1/planning/forecasts/accuracy        Permission: planning.forecasts.read

# MRP Runs
POST          /api/v1/planning/mrp/run                   Permission: planning.mrp.run
GET           /api/v1/planning/mrp/runs                   Permission: planning.mrp.read
GET           /api/v1/planning/mrp/runs/{id}              Permission: planning.mrp.read
GET           /api/v1/planning/mrp/runs/{id}/supplies     Permission: planning.mrp.read
GET           /api/v1/planning/mrp/runs/{id}/demands      Permission: planning.mrp.read

# Planned Orders
GET           /api/v1/planning/planned-orders            Permission: planning.orders.read
POST          /api/v1/planning/planned-orders/{id}/firm  Permission: planning.orders.update
POST          /api/v1/planning/planned-orders/{id}/release Permission: planning.orders.release
POST          /api/v1/planning/planned-orders/bulk-firm  Permission: planning.orders.update
POST          /api/v1/planning/planned-orders/bulk-release Permission: planning.orders.release

# Scenarios
GET/POST      /api/v1/planning/scenarios                 Permission: planning.scenarios.read/create
GET           /api/v1/planning/scenarios/{id}             Permission: planning.scenarios.read
POST          /api/v1/planning/scenarios/{id}/run         Permission: planning.scenarios.run
GET           /api/v1/planning/scenarios/{id}/results     Permission: planning.scenarios.read
GET           /api/v1/planning/scenarios/compare          Permission: planning.scenarios.read

# Reports
GET           /api/v1/planning/reports/supply-demand     Permission: planning.reports.view
GET           /api/v1/planning/reports/item-plan          Permission: planning.reports.view
GET           /api/v1/planning/reports/action-messages   Permission: planning.reports.view
GET           /api/v1/planning/reports/exception-report  Permission: planning.reports.view
```

---

## 4. Business Rules

### 4.1 MRP Run Logic
```
For each item in scope (bottom-up BOM explosion):
  1. Gather demands: sales orders + forecasts + safety stock requirements + component demands
  2. Gather supplies: on-hand + POs + WOs in progress + planned orders + transfers
  3. Time-phase demands and supplies by date
  4. Calculate net requirements per period:
     Net Requirement = Gross Demand - Scheduled Receipts - Projected Available + Safety Stock
  5. If net requirement > 0: create planned order
  6. Apply lot sizing rule to planned order quantity
  7. Calculate release date = due date - lead time
  8. For manufactured items: explode BOM, create component demands
  9. Generate action messages for existing orders
```

### 4.2 Demand Forecasting Methods
- **Moving Average:** `forecast = SUM(demand_last_N_periods) / N`
- **Exponential Smoothing:** `forecast = α × last_actual + (1-α) × last_forecast`
- **Weighted Average:** `forecast = Σ(weight_i × demand_i) / Σ(weights)`
- **Seasonal:** Apply seasonal indices to base forecast
- **Accuracy:** MAPE = `AVG(|actual - forecast| / actual) × 100`

### 4.3 Lot Sizing Rules
- **Fixed Lot:** Order in fixed quantities, may cover multiple periods
- **EOQ:** `sqrt(2 × annual_demand × order_cost / holding_cost_per_unit)`
- **Period Order Quantity:** EOQ converted to periods of coverage
- **Min-Max:** Order when projected < min, order up to max
- **Lot-for-Lot:** Exact quantity needed per period

### 4.4 Safety Stock Calculation
- **Fixed:** Manually specified quantity
- **Service Level:** `Z-score × σ_demand × sqrt(lead_time)` where Z from service level %
- **Time-Based:** Cover N days of average demand

### 4.5 BOM Explosion
- Explode demand through multi-level BOMs
- Apply scrap percentages
- Account for phantom assemblies (explode through)
- Handle co-products and by-products

### 4.6 Action Messages
| Message | Condition |
|---------|-----------|
| Cancel | Existing order no longer needed |
| Expedite | Order needed sooner than scheduled |
| De-expedite | Order can be delayed |
| Reschedule In | Move due date earlier |
| Reschedule Out | Move due date later |
| Reduce Quantity | Decrease order quantity |
| Increase Quantity | Increase order quantity |

### 4.7 Planned Order Conversion
- **Purchase** → Creates requisition/PO in Procurement
- **Manufacture** → Creates work order in Manufacturing
- **Transfer** → Creates transfer order in Inventory

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `planning.mrp.completed` | MRP run finished | — |
| `planning.planned_order.created` | New planned order | Proc, MFG |
| `planning.planned_order.released` | Order converted to actual | Proc, MFG, INV |
| `planning.forecast.updated` | Forecast revised | Report |
| `planning.shortage.detected` | Item shortage predicted | Proc |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.planning.v1;

service PlanningService {
    rpc RunMRP(RunMRPRequest) returns (RunMRPResponse);
    rpc GetMRPResults(GetMRPResultsRequest) returns (GetMRPResultsResponse);
    rpc GetPlannedOrders(GetPlannedOrdersRequest) returns (stream PlannedOrder);
    rpc ReleasePlannedOrder(ReleasePlannedOrderRequest) returns (ReleasePlannedOrderResponse);
    rpc GetItemPlan(GetItemPlanRequest) returns (GetItemPlanResponse);
    rpc GenerateForecast(GenerateForecastRequest) returns (GenerateForecastResponse);
    rpc RunScenario(RunScenarioRequest) returns (RunScenarioResponse);
}
```

```protobuf
message RunMRPRequest {
    string tenant_id = 1;
    string planning_horizon_start = 2;
    string planning_horizon_end = 3;
    string scope_type = 4;
    string scope_value = 5;
    string created_by = 6;
}

message RunMRPResponse {
    string mrp_run_id = 1;
    string run_number = 2;
    int32 total_items_planned = 3;
    int32 planned_orders_generated = 4;
    int32 action_messages = 5;
    int32 execution_time_ms = 6;
    string status = 7;
}

message GetMRPResultsRequest {
    string tenant_id = 1;
    string mrp_run_id = 2;
    string item_id = 3;
    string warehouse_id = 4;
}

message GetMRPResultsResponse {
    repeated MrpSupply supplies = 1;
    repeated MrpDemand demands = 2;
    repeated PlannedOrder planned_orders = 3;
}

message GetPlannedOrdersRequest {
    string tenant_id = 1;
    string mrp_run_id = 2;
    string item_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ReleasePlannedOrderRequest {
    string tenant_id = 1;
    string planned_order_id = 2;
    string released_by = 3;
}

message ReleasePlannedOrderResponse {
    bool success = 1;
    string converted_document_id = 2;
    string converted_document_type = 3;
}

message GetItemPlanRequest {
    string tenant_id = 1;
    string item_id = 2;
    string warehouse_id = 3;
    string horizon_start = 4;
    string horizon_end = 5;
}

message GetItemPlanResponse {
    repeated MrpSupply supplies = 1;
    repeated MrpDemand demands = 2;
    repeated PlannedOrder planned_orders = 3;
}

message GenerateForecastRequest {
    string tenant_id = 1;
    string item_id = 2;
    string warehouse_id = 3;
    string forecast_method = 4;
    int32 periods = 5;
    string created_by = 6;
}

message GenerateForecastResponse {
    repeated DemandForecast forecasts = 1;
}

message RunScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
    string parameters = 3;
    string created_by = 4;
}

message RunScenarioResponse {
    string scenario_id = 1;
    string status = 2;
    repeated ScenarioResult results = 3;
}

message PlanningCalendar {
    string id = 1;
    string tenant_id = 2;
    string calendar_name = 3;
    string calendar_type = 4;
    string effective_from = 5;
    string effective_to = 6;
    int32 monday = 7;
    int32 tuesday = 8;
    int32 wednesday = 9;
    int32 thursday = 10;
    int32 friday = 11;
    int32 saturday = 12;
    int32 sunday = 13;
    string created_at = 14;
    string updated_at = 15;
    string created_by = 16;
    string updated_by = 17;
    int32 version = 18;
    int32 is_active = 19;
}

message MrpParameters {
    string id = 1;
    string tenant_id = 2;
    string item_id = 3;
    string warehouse_id = 4;
    string planning_method = 5;
    string lot_size_method = 6;
    string fixed_lot_quantity = 7;
    string min_order_quantity = 8;
    string max_order_quantity = 9;
    string order_multiple = 10;
    int32 fixed_lead_time_days = 11;
    double variable_lead_time_days = 12;
    string safety_stock_method = 13;
    string safety_stock_quantity = 14;
    double service_level_percent = 15;
    int32 safety_time_days = 16;
    int32 planning_time_fence_days = 17;
    int32 demand_time_fence_days = 18;
    string order_policy = 19;
    string created_at = 20;
    string updated_at = 21;
    string created_by = 22;
    string updated_by = 23;
    int32 version = 24;
    int32 is_active = 25;
}

message DemandForecast {
    string id = 1;
    string tenant_id = 2;
    string item_id = 3;
    string warehouse_id = 4;
    string period_start_date = 5;
    string period_end_date = 6;
    string forecast_method = 7;
    string forecasted_demand = 8;
    string actual_demand = 9;
    double accuracy_percent = 10;
    string confidence_level = 11;
    string forecast_set_id = 12;
    string created_at = 13;
    string updated_at = 14;
    string created_by = 15;
    string updated_by = 16;
}

message MrpRun {
    string id = 1;
    string tenant_id = 2;
    string run_number = 3;
    string run_date = 4;
    string planning_horizon_start = 5;
    string planning_horizon_end = 6;
    string scope_type = 7;
    string scope_value = 8;
    string status = 9;
    int32 total_items_planned = 10;
    int32 planned_orders_generated = 11;
    int32 action_messages = 12;
    int32 execution_time_ms = 13;
    string created_at = 14;
    string updated_at = 15;
    string created_by = 16;
}

message MrpSupply {
    string id = 1;
    string tenant_id = 2;
    string mrp_run_id = 3;
    string item_id = 4;
    string warehouse_id = 5;
    string supply_type = 6;
    string supply_document_id = 7;
    string quantity = 8;
    string due_date = 9;
    string status = 10;
    string created_at = 11;
}

message MrpDemand {
    string id = 1;
    string tenant_id = 2;
    string mrp_run_id = 3;
    string item_id = 4;
    string warehouse_id = 5;
    string demand_type = 6;
    string demand_document_id = 7;
    string quantity = 8;
    string required_date = 9;
    int32 priority = 10;
    string created_at = 11;
}

message PlannedOrder {
    string id = 1;
    string tenant_id = 2;
    string mrp_run_id = 3;
    string item_id = 4;
    string warehouse_id = 5;
    string order_type = 6;
    string quantity = 7;
    string due_date = 8;
    string release_date = 9;
    string status = 10;
    string pegged_demand_id = 11;
    string pegged_demand_type = 12;
    string converted_document_id = 13;
    string converted_document_type = 14;
    string action_message = 15;
    string action_details = 16;
    string created_at = 17;
    string updated_at = 18;
    string created_by = 19;
}

message Scenario {
    string id = 1;
    string tenant_id = 2;
    string scenario_name = 3;
    string description = 4;
    string baseline_scenario_id = 5;
    string scenario_type = 6;
    string parameters = 7;
    string status = 8;
    string created_at = 9;
    string updated_at = 10;
    string created_by = 11;
    string updated_by = 12;
    int32 version = 13;
}

message ScenarioResult {
    string id = 1;
    string tenant_id = 2;
    string scenario_id = 3;
    string item_id = 4;
    string warehouse_id = 5;
    string period_date = 6;
    string projected_available = 7;
    int32 planned_orders_count = 8;
    string shortage_quantity = 9;
    string excess_quantity = 10;
    string created_at = 11;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **OM:** Sales orders (demand)
- **INV:** On-hand quantities, stock levels
- **Proc:** Open purchase orders (supply)
- **MFG:** Open work orders (supply), BOM structures
- **CM:** Lead time information

### 6.2 Data Published To
- **Proc:** Convert planned purchase orders to requisitions/POs
- **MFG:** Convert planned work orders to work orders
- **INV:** Planned transfer orders

---

## 7. Migrations

1. V001: `planning_calendars`
2. V002: `calendar_exceptions`
3. V003: `mrp_parameters`
4. V004: `demand_forecasts`
5. V005: `forecast_methods_config`
6. V006: `mrp_runs`
7. V007: `mrp_supplies`
8. V008: `mrp_demands`
9. V009: `planned_orders`
10. V010: `scenarios`
11. V011: `scenario_results`
12. V012: Triggers for `updated_at`
