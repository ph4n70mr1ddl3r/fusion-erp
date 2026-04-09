# 114 - SCM Analytics Service Specification

## 1. Domain Overview

The SCM Analytics service provides pre-built supply chain analytics delivering dashboards, KPIs, and predictive insights across the plan-to-deliver lifecycle. It includes inventory analytics, procurement spend analysis, supplier performance dashboards, order fulfillment metrics, logistics analytics, and demand forecasting visualization. Distinct from general Fusion Data Intelligence by focusing exclusively on the supply chain domain.

**Bounded Context:** Supply Chain Analytics
**Service Name:** `scmanalytics-service`
**Database:** `data/scmanalytics.db`
**HTTP Port:** 8151 | **gRPC Port:** 9151

---

## 2. Database Schema

### 2.1 SCM KPI Definitions
```sql
CREATE TABLE scm_kpi_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_name TEXT NOT NULL,
    kpi_category TEXT NOT NULL CHECK(kpi_category IN ('INVENTORY','PROCUREMENT','FULFILLMENT','LOGISTICS','DEMAND','SUPPLIER','QUALITY','MANUFACTURING')),
    calculation_formula TEXT NOT NULL,
    unit_type TEXT NOT NULL CHECK(unit_type IN ('PERCENTAGE','COUNT','CURRENCY','DAYS','RATIO')),
    target_value DECIMAL(15,4),
    warning_threshold DECIMAL(15,4),
    critical_threshold DECIMAL(15,4),
    dimensions TEXT,  -- JSON
    refresh_frequency TEXT NOT NULL DEFAULT 'DAILY' CHECK(refresh_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY')),
    data_source_config TEXT,  -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, kpi_name)
);

CREATE INDEX idx_kpid_tenant_category ON scm_kpi_definitions(tenant_id, kpi_category);
CREATE INDEX idx_kpid_tenant_active ON scm_kpi_definitions(tenant_id, is_active);
```

### 2.2 SCM KPI Values
```sql
CREATE TABLE scm_kpi_values (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_id TEXT NOT NULL,
    period_type TEXT NOT NULL CHECK(period_type IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    actual_value DECIMAL(15,4) NOT NULL,
    target_value DECIMAL(15,4),
    previous_value DECIMAL(15,4),
    trend TEXT CHECK(trend IN ('IMPROVING','STABLE','DECLINING')),
    dimensions_applied TEXT,  -- JSON
    calculated_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (kpi_id) REFERENCES scm_kpi_definitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_kpiv_tenant_kpi ON scm_kpi_values(tenant_id, kpi_id);
CREATE INDEX idx_kpiv_tenant_period ON scm_kpi_values(tenant_id, period_type, period_start, period_end);
CREATE INDEX idx_kpiv_tenant_trend ON scm_kpi_values(tenant_id, trend);
```

### 2.3 SCM Dashboards
```sql
CREATE TABLE scm_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    dashboard_category TEXT NOT NULL CHECK(dashboard_category IN ('EXECUTIVE','INVENTORY','PROCUREMENT','ORDER','LOGISTICS','SUPPLIER','QUALITY')),
    description TEXT,
    layout_config TEXT NOT NULL,  -- JSON
    widgets TEXT NOT NULL,  -- JSON
    owner_id TEXT NOT NULL,
    sharing_type TEXT NOT NULL DEFAULT 'PRIVATE' CHECK(sharing_type IN ('PRIVATE','SHARED','PUBLIC')),
    refresh_interval_seconds INTEGER DEFAULT 300,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dashboard_name)
);

CREATE INDEX idx_dash_tenant_category ON scm_dashboards(tenant_id, dashboard_category);
CREATE INDEX idx_dash_tenant_owner ON scm_dashboards(tenant_id, owner_id);
```

### 2.4 Inventory Analytics
```sql
CREATE TABLE inventory_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    item_category_id TEXT,
    period_date TEXT NOT NULL,
    total_on_hand_value_cents INTEGER NOT NULL DEFAULT 0,
    turnover_rate DECIMAL(5,2) DEFAULT 0,
    days_of_supply DECIMAL(10,2) DEFAULT 0,
    stockout_count INTEGER NOT NULL DEFAULT 0,
    excess_stock_value_cents INTEGER NOT NULL DEFAULT 0,
    slow_moving_value_cents INTEGER NOT NULL DEFAULT 0,
    abc_classification TEXT CHECK(abc_classification IN ('A','B','C','D')),
    dead_stock_value_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, warehouse_id, item_category_id, period_date)
);

CREATE INDEX idx_ia_tenant_warehouse ON inventory_analytics(tenant_id, warehouse_id);
CREATE INDEX idx_ia_tenant_category ON inventory_analytics(tenant_id, item_category_id);
CREATE INDEX idx_ia_tenant_date ON inventory_analytics(tenant_id, period_date);
CREATE INDEX idx_ia_tenant_abc ON inventory_analytics(tenant_id, abc_classification);
```

### 2.5 Procurement Analytics
```sql
CREATE TABLE procurement_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT,
    buyer_id TEXT,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    total_spend_cents INTEGER NOT NULL DEFAULT 0,
    po_count INTEGER NOT NULL DEFAULT 0,
    avg_lead_time_days DECIMAL(10,2) DEFAULT 0,
    on_time_delivery_pct DECIMAL(5,2) DEFAULT 0,
    cost_savings_cents INTEGER NOT NULL DEFAULT 0,
    maverick_spend_pct DECIMAL(5,2) DEFAULT 0,
    contract_compliance_pct DECIMAL(5,2) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, buyer_id, period_start, period_end)
);

CREATE INDEX idx_pa_tenant_supplier ON procurement_analytics(tenant_id, supplier_id);
CREATE INDEX idx_pa_tenant_buyer ON procurement_analytics(tenant_id, buyer_id);
CREATE INDEX idx_pa_tenant_period ON procurement_analytics(tenant_id, period_start, period_end);
```

### 2.6 Fulfillment Analytics
```sql
CREATE TABLE fulfillment_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    warehouse_id TEXT,
    shipping_method TEXT,
    period_date TEXT NOT NULL,
    total_orders INTEGER NOT NULL DEFAULT 0,
    on_time_shipments INTEGER NOT NULL DEFAULT 0,
    on_time_delivery_pct DECIMAL(5,2) DEFAULT 0,
    avg_order_cycle_time_days DECIMAL(5,2) DEFAULT 0,
    perfect_order_pct DECIMAL(5,2) DEFAULT 0,
    backorder_count INTEGER NOT NULL DEFAULT 0,
    fill_rate DECIMAL(5,2) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, warehouse_id, shipping_method, period_date)
);

CREATE INDEX idx_fa_tenant_warehouse ON fulfillment_analytics(tenant_id, warehouse_id);
CREATE INDEX idx_fa_tenant_date ON fulfillment_analytics(tenant_id, period_date);
CREATE INDEX idx_fa_tenant_method ON fulfillment_analytics(tenant_id, shipping_method);
```

### 2.7 Supplier Scorecards
```sql
CREATE TABLE supplier_scorecards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    quality_score DECIMAL(5,2) DEFAULT 0,
    delivery_score DECIMAL(5,2) DEFAULT 0,
    cost_score DECIMAL(5,2) DEFAULT 0,
    responsiveness_score DECIMAL(5,2) DEFAULT 0,
    overall_score DECIMAL(5,2) DEFAULT 0,
    tier TEXT NOT NULL DEFAULT 'APPROVED' CHECK(tier IN ('PREFERRED','APPROVED','PROVISIONAL','AT_RISK')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, period_start, period_end)
);

CREATE INDEX idx_ss_tenant_supplier ON supplier_scorecards(tenant_id, supplier_id);
CREATE INDEX idx_ss_tenant_tier ON supplier_scorecards(tenant_id, tier);
CREATE INDEX idx_ss_tenant_period ON supplier_scorecards(tenant_id, period_start, period_end);
```

---

## 3. REST API Endpoints

### KPIs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/kpis` | Create a KPI definition |
| GET | `/api/v1/kpis` | List KPI definitions |
| GET | `/api/v1/kpis/{id}` | Get KPI definition |
| PUT | `/api/v1/kpis/{id}` | Update KPI definition |
| GET | `/api/v1/kpis/{id}/values` | Get KPI values over time |
| GET | `/api/v1/kpis/{id}/trends` | Get KPI trend analysis |
| POST | `/api/v1/kpis/{id}/calculate` | Trigger KPI calculation |

### Dashboards
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/dashboards` | Create a dashboard |
| GET | `/api/v1/dashboards` | List dashboards |
| GET | `/api/v1/dashboards/{id}` | Get dashboard details |
| PUT | `/api/v1/dashboards/{id}` | Update dashboard |
| DELETE | `/api/v1/dashboards/{id}` | Delete dashboard |
| POST | `/api/v1/dashboards/{id}/share` | Share dashboard with users |

### Inventory Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/inventory-analytics/dashboard` | Get inventory analytics dashboard |
| GET | `/api/v1/inventory-analytics/by-warehouse` | Get analytics by warehouse |
| GET | `/api/v1/inventory-analytics/by-category` | Get analytics by item category |
| GET | `/api/v1/inventory-analytics/aging` | Get inventory aging analysis |

### Procurement Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/procurement-analytics/spend-analysis` | Get procurement spend analysis |
| GET | `/api/v1/procurement-analytics/by-supplier` | Get spend by supplier |
| GET | `/api/v1/procurement-analytics/by-buyer` | Get spend by buyer |
| GET | `/api/v1/procurement-analytics/savings` | Get cost savings analysis |

### Fulfillment Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fulfillment-analytics/dashboard` | Get fulfillment analytics dashboard |
| GET | `/api/v1/fulfillment-analytics/by-warehouse` | Get fulfillment by warehouse |
| GET | `/api/v1/fulfillment-analytics/order-metrics` | Get order metrics summary |

### Supplier Scorecards
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/supplier-scorecards/by-supplier/{supplierId}` | Get scorecard for a supplier |
| GET | `/api/v1/supplier-scorecards/comparison` | Compare supplier scorecards |
| GET | `/api/v1/supplier-scorecards/trends` | Get scorecard trends over time |

---

## 4. Business Rules

1. KPI definitions MUST include a valid calculation_formula and at least one data_source_config entry.
2. KPI values MUST be recalculated whenever the source data changes for REAL_TIME refresh frequency KPIs.
3. The system MUST set the trend field to IMPROVING, STABLE, or DECLINING by comparing actual_value to previous_value using the KPI definition directionality.
4. A KPI value that crosses the critical_threshold MUST trigger a KPIThresholdBreached event.
5. Supplier scorecard overall_score MUST be calculated as a weighted average of quality_score, delivery_score, cost_score, and responsiveness_score.
6. Supplier tier MUST be automatically updated based on overall_score thresholds: PREFERRED >= 90, APPROVED >= 70, PROVISIONAL >= 50, AT_RISK < 50.
7. Inventory turnover_rate MUST be calculated as cost of goods sold divided by average inventory value for the period.
8. ABC classification MUST be recalculated at least weekly using the Pareto principle on total_on_hand_value_cents.
9. Dashboard sharing_type of PUBLIC MUST make the dashboard visible to all users within the tenant.
10. Dead stock value MUST include only items with no transaction activity for more than 365 days.
11. Procurement maverick_spend_pct MUST represent spend outside of approved contracts as a percentage of total spend.
12. The system SHOULD cache dashboard results for the configured refresh_interval_seconds before recalculating.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package scmanalytics.v1;

service SCMAnalyticsService {
    // KPIs
    rpc CreateKPI(CreateKPIRequest) returns (CreateKPIResponse);
    rpc GetKPI(GetKPIRequest) returns (GetKPIResponse);
    rpc ListKPIs(ListKPIsRequest) returns (ListKPIsResponse);
    rpc GetKPIValues(GetKPIValuesRequest) returns (GetKPIValuesResponse);
    rpc GetKPITrends(GetKPITrendsRequest) returns (GetKPITrendsResponse);
    rpc CalculateKPI(CalculateKPIRequest) returns (CalculateKPIResponse);

    // Dashboards
    rpc CreateDashboard(CreateDashboardRequest) returns (CreateDashboardResponse);
    rpc GetDashboard(GetDashboardRequest) returns (GetDashboardResponse);
    rpc ListDashboards(ListDashboardsRequest) returns (ListDashboardsResponse);
    rpc ShareDashboard(ShareDashboardRequest) returns (ShareDashboardResponse);

    // Inventory Analytics
    rpc GetInventoryDashboard(GetInventoryDashboardRequest) returns (GetInventoryDashboardResponse);
    rpc GetInventoryByWarehouse(GetInventoryByWarehouseRequest) returns (GetInventoryByWarehouseResponse);
    rpc GetInventoryAging(GetInventoryAgingRequest) returns (GetInventoryAgingResponse);

    // Procurement Analytics
    rpc GetSpendAnalysis(GetSpendAnalysisRequest) returns (GetSpendAnalysisResponse);
    rpc GetSpendBySupplier(GetSpendBySupplierRequest) returns (GetSpendBySupplierResponse);
    rpc GetCostSavings(GetCostSavingsRequest) returns (GetCostSavingsResponse);

    // Fulfillment Analytics
    rpc GetFulfillmentDashboard(GetFulfillmentDashboardRequest) returns (GetFulfillmentDashboardResponse);
    rpc GetOrderMetrics(GetOrderMetricsRequest) returns (GetOrderMetricsResponse);

    // Supplier Scorecards
    rpc GetSupplierScorecard(GetSupplierScorecardRequest) returns (GetSupplierScorecardResponse);
    rpc CompareSuppliers(CompareSuppliersRequest) returns (CompareSuppliersResponse);
    rpc GetScorecardTrends(GetScorecardTrendsRequest) returns (GetScorecardTrendsResponse);
}

message KPIDefinition {
    string id = 1;
    string tenant_id = 2;
    string kpi_name = 3;
    string kpi_category = 4;
    string calculation_formula = 5;
    string unit_type = 6;
    double target_value = 7;
    double warning_threshold = 8;
    double critical_threshold = 9;
    string refresh_frequency = 10;
    bool is_active = 11;
}

message CreateKPIRequest {
    string tenant_id = 1;
    string kpi_name = 2;
    string kpi_category = 3;
    string calculation_formula = 4;
    string unit_type = 5;
    double target_value = 6;
    double warning_threshold = 7;
    double critical_threshold = 8;
    string refresh_frequency = 9;
    string created_by = 10;
}

message CreateKPIResponse {
    KPIDefinition kpi = 1;
}

message GetKPIRequest {
    string tenant_id = 1;
    string kpi_id = 2;
}

message GetKPIResponse {
    KPIDefinition kpi = 1;
}

message ListKPIsRequest {
    string tenant_id = 1;
    string kpi_category = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListKPIsResponse {
    repeated KPIDefinition kpis = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message KPIValue {
    string id = 1;
    string kpi_id = 2;
    string period_type = 3;
    string period_start = 4;
    string period_end = 5;
    double actual_value = 6;
    double target_value = 7;
    double previous_value = 8;
    string trend = 9;
    string calculated_at = 10;
}

message GetKPIValuesRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    string period_type = 3;
    string date_from = 4;
    string date_to = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message GetKPIValuesResponse {
    repeated KPIValue values = 1;
    string next_page_token = 2;
}

message GetKPITrendsRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    int32 periods = 3;
}

message GetKPITrendsResponse {
    string trend = 1;
    double current_value = 2;
    double previous_value = 3;
    double change_percentage = 4;
    repeated KPIValue historical_values = 5;
}

message CalculateKPIRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    string calculated_by = 3;
}

message CalculateKPIResponse {
    KPIValue value = 1;
}

message Dashboard {
    string id = 1;
    string tenant_id = 2;
    string dashboard_name = 3;
    string dashboard_category = 4;
    string description = 5;
    string sharing_type = 6;
    string owner_id = 7;
    int32 refresh_interval_seconds = 8;
    bool is_default = 9;
}

message CreateDashboardRequest {
    string tenant_id = 1;
    string dashboard_name = 2;
    string dashboard_category = 3;
    string description = 4;
    string layout_config = 5;
    string widgets = 6;
    string owner_id = 7;
    string created_by = 8;
}

message CreateDashboardResponse {
    Dashboard dashboard = 1;
}

message GetDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
}

message GetDashboardResponse {
    Dashboard dashboard = 1;
}

message ListDashboardsRequest {
    string tenant_id = 1;
    string dashboard_category = 2;
    string owner_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListDashboardsResponse {
    repeated Dashboard dashboards = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ShareDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    string sharing_type = 3;
    repeated string user_ids = 4;
    string shared_by = 5;
}

message ShareDashboardResponse {
    bool success = 1;
    string sharing_type = 2;
}

message InventoryAnalyticsItem {
    string id = 1;
    string warehouse_id = 2;
    string item_category_id = 3;
    string period_date = 4;
    int64 total_on_hand_value_cents = 5;
    double turnover_rate = 6;
    double days_of_supply = 7;
    int32 stockout_count = 8;
    string abc_classification = 9;
}

message GetInventoryDashboardRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message GetInventoryDashboardResponse {
    repeated InventoryAnalyticsItem items = 1;
    int64 total_value_cents = 2;
    double avg_turnover_rate = 3;
    int32 total_stockouts = 4;
}

message GetInventoryByWarehouseRequest {
    string tenant_id = 1;
    string warehouse_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetInventoryByWarehouseResponse {
    repeated InventoryAnalyticsItem items = 1;
}

message GetInventoryAgingRequest {
    string tenant_id = 1;
    string warehouse_id = 2;
    string as_of_date = 3;
}

message GetInventoryAgingResponse {
    int64 current_value_cents = 1;
    int64 slow_moving_value_cents = 2;
    int64 excess_value_cents = 3;
    int64 dead_stock_value_cents = 4;
    repeated AgingBucket aging_buckets = 5;
}

message AgingBucket {
    string bucket_name = 1;
    int64 value_cents = 2;
    double percentage = 3;
}

message SpendAnalysisItem {
    string supplier_id = 1;
    int64 total_spend_cents = 2;
    int32 po_count = 3;
    double avg_lead_time_days = 4;
    double on_time_delivery_pct = 5;
    double maverick_spend_pct = 6;
    double contract_compliance_pct = 7;
}

message GetSpendAnalysisRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message GetSpendAnalysisResponse {
    repeated SpendAnalysisItem items = 1;
    int64 total_spend_cents = 2;
    int64 total_savings_cents = 3;
}

message GetSpendBySupplierRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetSpendBySupplierResponse {
    repeated SpendAnalysisItem items = 1;
}

message GetCostSavingsRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message GetCostSavingsResponse {
    int64 total_savings_cents = 1;
    int64 negotiated_savings_cents = 2;
    int64 compliance_savings_cents = 3;
    int64 process_savings_cents = 4;
}

message FulfillmentDashboardItem {
    string warehouse_id = 1;
    string period_date = 2;
    int32 total_orders = 3;
    double on_time_delivery_pct = 4;
    double perfect_order_pct = 5;
    double fill_rate = 6;
}

message GetFulfillmentDashboardRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message GetFulfillmentDashboardResponse {
    repeated FulfillmentDashboardItem items = 1;
    double avg_on_time_delivery_pct = 2;
    double avg_perfect_order_pct = 3;
    int32 total_backorders = 4;
}

message GetOrderMetricsRequest {
    string tenant_id = 1;
    string warehouse_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetOrderMetricsResponse {
    int32 total_orders = 1;
    double avg_cycle_time_days = 2;
    double on_time_ship_pct = 3;
    double on_time_delivery_pct = 4;
    double perfect_order_pct = 5;
    double fill_rate = 6;
    int32 backorder_count = 7;
}

message SupplierScorecard {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string period_start = 4;
    string period_end = 5;
    double quality_score = 6;
    double delivery_score = 7;
    double cost_score = 8;
    double responsiveness_score = 9;
    double overall_score = 10;
    string tier = 11;
}

message GetSupplierScorecardRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetSupplierScorecardResponse {
    SupplierScorecard scorecard = 1;
}

message CompareSuppliersRequest {
    string tenant_id = 1;
    repeated string supplier_ids = 2;
    string period_start = 3;
    string period_end = 4;
}

message CompareSuppliersResponse {
    repeated SupplierScorecard scorecards = 1;
}

message GetScorecardTrendsRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    int32 periods = 3;
}

message GetScorecardTrendsResponse {
    repeated SupplierScorecard scorecards = 1;
    double avg_overall_score = 2;
    string overall_trend = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `inventory-service` | Stock levels, transactions, warehouse data | Calculate inventory analytics and KPIs |
| `procurement-service` | PO data, supplier invoices, contract spend | Calculate procurement analytics |
| `order-service` | Order data, shipment tracking, fulfillment status | Calculate fulfillment analytics |
| `logistics-service` | Shipment data, delivery confirmations, carrier performance | Calculate logistics analytics |
| `supplier-service` | Supplier profiles, certifications, quality records | Calculate supplier scorecards |
| `manufacturing-service` | Production orders, yield, scrap data | Calculate manufacturing KPIs |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `reporting-service` | Analytics results, dashboards, KPIs | Enterprise reporting and BI |
| `alert-service` | Threshold breaches, critical KPIs | Proactive notifications |
| `procurement-service` | Supplier scorecards, tier changes | Supplier management decisions |
| `inventory-service` | ABC classifications, dead stock alerts | Inventory optimization |
| `planning-service` | Demand trends, forecast accuracy | Supply chain planning |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `KPIValueCalculated` | `scmanalytics.kpi.calculated` | `{ tenant_id, kpi_id, kpi_name, actual_value, target_value, trend, calculated_at }` | Published when a KPI value is calculated |
| `KPIThresholdBreached` | `scmanalytics.kpi.threshold-breached` | `{ tenant_id, kpi_id, kpi_name, actual_value, threshold_type, threshold_value }` | Published when a KPI value crosses a critical or warning threshold |
| `ScorecardUpdated` | `scmanalytics.scorecard.updated` | `{ tenant_id, supplier_id, overall_score, tier, period_start, period_end }` | Published when a supplier scorecard is recalculated |
| `DashboardRefreshed` | `scmanalytics.dashboard.refreshed` | `{ tenant_id, dashboard_id, dashboard_name, refreshed_at, widget_count }` | Published when a dashboard data refresh completes |
