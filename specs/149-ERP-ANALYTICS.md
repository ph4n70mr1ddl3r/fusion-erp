# 149 - ERP Analytics Service Specification

## 1. Domain Overview

ERP Analytics provides prebuilt financial and operational analytics, KPI dashboards, and predictive insights across the enterprise resource planning lifecycle. Covers revenue analysis, expense tracking, cash flow forecasting, accounts payable/receivable aging, procurement spend analysis, project profitability, and general ledger analytics. Supports custom financial report building, ad-hoc analysis, embedded visualizations, and scheduled report distribution with drill-down capabilities. Integrates with all ERP modules for comprehensive financial intelligence.

**Bounded Context:** Financial & ERP Analytics
**Service Name:** `erp-analytics-service`
**Database:** `data/erp_analytics.db`
**HTTP Port:** 8167 | **gRPC Port:** 9167

---

## 2. Database Schema

### 2.1 Analytics Dashboards
```sql
CREATE TABLE ea_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_code TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    dashboard_type TEXT NOT NULL CHECK(dashboard_type IN (
        'FINANCIAL_SUMMARY','REVENUE','EXPENSE','CASH_FLOW','AP','AR',
        'PROCUREMENT','PROJECT','GL','TAX','FIXED_ASSETS','CUSTOM'
    )),
    description TEXT,
    category TEXT NOT NULL,
    widgets TEXT NOT NULL,                  -- JSON array: widget configurations
    filters TEXT,                           -- JSON: available filter dimensions
    access_roles TEXT NOT NULL,             -- JSON array of role IDs
    is_default INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dashboard_code)
);
```

### 2.2 Financial KPI Definitions
```sql
CREATE TABLE ea_kpi_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_code TEXT NOT NULL,
    kpi_name TEXT NOT NULL,
    kpi_category TEXT NOT NULL CHECK(kpi_category IN (
        'REVENUE','PROFITABILITY','CASH','AP','AR','EXPENSE',
        'TAX','FIXED_ASSETS','PROCUREMENT','PROJECT','CUSTOM'
    )),
    description TEXT,
    calculation_type TEXT NOT NULL CHECK(calculation_type IN ('COUNT','RATIO','PERCENTAGE','AVERAGE','SUM','CUSTOM')),
    measure_expression TEXT NOT NULL,
    dimensions TEXT NOT NULL,               -- JSON: department, cost center, legal entity, etc.
    target_value REAL,
    warning_threshold REAL,
    critical_threshold REAL,
    threshold_direction TEXT NOT NULL DEFAULT 'BELOW'
        CHECK(threshold_direction IN ('BELOW','ABOVE')),
    unit TEXT NOT NULL DEFAULT 'NUMBER',
    currency_aware INTEGER NOT NULL DEFAULT 1,
    refresh_frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(refresh_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY','MONTHLY')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, kpi_code)
);
```

### 2.3 Financial Metrics Snapshots
```sql
CREATE TABLE ea_financial_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-01"
    legal_entity_id TEXT,
    department_id TEXT,
    cost_center_id TEXT,
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    total_expense_cents INTEGER NOT NULL DEFAULT 0,
    gross_profit_cents INTEGER NOT NULL DEFAULT 0,
    net_income_cents INTEGER NOT NULL DEFAULT 0,
    ebitda_cents INTEGER NOT NULL DEFAULT 0,
    operating_margin_pct REAL NOT NULL DEFAULT 0,
    revenue_growth_pct REAL NOT NULL DEFAULT 0,
    expense_growth_pct REAL NOT NULL DEFAULT 0,
    ap_balance_cents INTEGER NOT NULL DEFAULT 0,
    ar_balance_cents INTEGER NOT NULL DEFAULT 0,
    cash_balance_cents INTEGER NOT NULL DEFAULT 0,
    ap_aging_current_cents INTEGER NOT NULL DEFAULT 0,
    ap_aging_30_cents INTEGER NOT NULL DEFAULT 0,
    ap_aging_60_cents INTEGER NOT NULL DEFAULT 0,
    ap_aging_90_plus_cents INTEGER NOT NULL DEFAULT 0,
    ar_aging_current_cents INTEGER NOT NULL DEFAULT 0,
    ar_aging_30_cents INTEGER NOT NULL DEFAULT 0,
    ar_aging_60_cents INTEGER NOT NULL DEFAULT 0,
    ar_aging_90_plus_cents INTEGER NOT NULL DEFAULT 0,
    dso_days REAL NOT NULL DEFAULT 0,       -- Days Sales Outstanding
    dpo_days REAL NOT NULL DEFAULT 0,       -- Days Payable Outstanding
    cash_conversion_cycle_days REAL NOT NULL DEFAULT 0,
    budget_variance_pct REAL NOT NULL DEFAULT 0,
    journal_entry_count INTEGER NOT NULL DEFAULT 0,
    invoice_count INTEGER NOT NULL DEFAULT 0,
    po_count INTEGER NOT NULL DEFAULT 0,
    project_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, period, legal_entity_id, department_id, cost_center_id)
);

CREATE INDEX idx_ea_metric_period ON ea_financial_metrics(tenant_id, period DESC);
CREATE INDEX idx_ea_metric_entity ON ea_financial_metrics(tenant_id, legal_entity_id, period DESC);
```

### 2.4 Spend Analysis
```sql
CREATE TABLE ea_spend_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,
    spend_category TEXT NOT NULL,
    supplier_id TEXT,
    department_id TEXT,
    cost_center_id TEXT,
    total_spend_cents INTEGER NOT NULL DEFAULT 0,
    invoice_count INTEGER NOT NULL DEFAULT 0,
    po_coverage_pct REAL NOT NULL DEFAULT 0,
    avg_payment_terms_days REAL NOT NULL DEFAULT 0,
    early_payment_discount_captured INTEGER NOT NULL DEFAULT 0,
    maverick_spend_pct REAL NOT NULL DEFAULT 0,
    contract_compliance_pct REAL NOT NULL DEFAULT 0,
    supplier_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, period, spend_category, supplier_id, department_id)
);

CREATE INDEX idx_ea_spend_tenant ON ea_spend_analysis(tenant_id, period DESC);
CREATE INDEX idx_ea_spend_category ON ea_spend_analysis(tenant_id, spend_category);
```

### 2.5 Custom Reports
```sql
CREATE TABLE ea_custom_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_name TEXT NOT NULL,
    report_type TEXT NOT NULL CHECK(report_type IN ('TABULAR','PIVOT','CHART','FINANCIAL_STATEMENT','AGING','CUSTOM')),
    description TEXT,
    data_source TEXT NOT NULL,
    columns TEXT NOT NULL,                  -- JSON: column definitions
    filters TEXT,
    grouping TEXT,
    sort_config TEXT,
    visualization_config TEXT,
    schedule TEXT,
    recipients TEXT,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, report_name)
);
```

---

## 3. API Endpoints

### 3.1 Dashboards
| Method | Path | Description |
|--------|------|-------------|
| POST | `/erp-analytics/v1/dashboards` | Create dashboard |
| GET | `/erp-analytics/v1/dashboards` | List dashboards |
| GET | `/erp-analytics/v1/dashboards/{id}` | Get dashboard with data |
| PUT | `/erp-analytics/v1/dashboards/{id}` | Update dashboard |
| DELETE | `/erp-analytics/v1/dashboards/{id}` | Archive dashboard |

### 3.2 KPIs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/erp-analytics/v1/kpis` | List KPIs with current values |
| GET | `/erp-analytics/v1/kpis/{id}/trend` | Get KPI trend |
| GET | `/erp-analytics/v1/kpis/{id}/breakdown` | Get KPI breakdown |
| POST | `/erp-analytics/v1/kpis/refresh` | Trigger KPI refresh |

### 3.3 Financial Metrics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/erp-analytics/v1/financial-metrics` | Query financial metrics |
| GET | `/erp-analytics/v1/financial-metrics/trend` | Metric trends |
| GET | `/erp-analytics/v1/financial-metrics/comparison` | Period comparison |
| GET | `/erp-analytics/v1/financial-metrics/variance` | Budget vs actual variance |

### 3.4 Spend Analysis
| Method | Path | Description |
|--------|------|-------------|
| GET | `/erp-analytics/v1/spend-analysis` | Get spend analysis |
| GET | `/erp-analytics/v1/spend-analysis/by-category` | Spend by category |
| GET | `/erp-analytics/v1/spend-analysis/by-supplier` | Spend by supplier |
| GET | `/erp-analytics/v1/spend-analysis/trend` | Spend trends |

### 3.5 Custom Reports
| Method | Path | Description |
|--------|------|-------------|
| POST | `/erp-analytics/v1/reports` | Create custom report |
| GET | `/erp-analytics/v1/reports` | List reports |
| POST | `/erp-analytics/v1/reports/{id}/execute` | Execute report |
| POST | `/erp-analytics/v1/reports/{id}/export` | Export report |
| POST | `/erp-analytics/v1/reports/{id}/schedule` | Schedule report |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `erp.kpi.threshold_breach` | `{ kpi_id, value, threshold }` | KPI crossed threshold |
| `erp.metrics.calculated` | `{ period, entity_count }` | Financial metrics calculated |
| `erp.report.completed` | `{ report_id, row_count }` | Report execution completed |
| `erp.spend.anomaly` | `{ category, amount, deviation_pct }` | Spend anomaly detected |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `gl.journal.posted` | General Ledger | Update GL metrics |
| `invoice.created` | AP/AR | Update AP/AR metrics |
| `payment.completed` | Cash Management | Update cash metrics |
| `po.approved` | Procurement | Update procurement metrics |
| `project.cost.updated` | Project Mgmt | Update project metrics |
| `budget.approved` | EPM | Set budget targets for variance |

---


---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.erp_analytics.v1;

service ErpAnalyticsService {
    // Dashboard management
    rpc CreateDashboard(CreateDashboardRequest) returns (Dashboard);
    rpc GetDashboard(GetDashboardRequest) returns (Dashboard);
    rpc ListDashboards(ListDashboardsRequest) returns (DashboardList);

    // Financial metrics
    rpc GetFinancialMetrics(GetFinancialMetricsRequest) returns (FinancialMetricsList);
    rpc GetSpendAnalysis(GetSpendAnalysisRequest) returns (SpendAnalysisList);

    // KPI operations
    rpc GetKpiTrend(GetKpiTrendRequest) returns (KpiTrendResponse);
    rpc RefreshKpis(RefreshKpisRequest) returns (RefreshKpisResponse);

    // Report execution
    rpc ExecuteReport(ExecuteReportRequest) returns (ReportExecutionResult);
}

// Entity messages
message Dashboard {
    string id = 1;
    string tenant_id = 2;
    string dashboard_code = 3;
    string dashboard_name = 4;
    string dashboard_type = 5;
    string description = 6;
    string category = 7;
    string widgets = 8;
    string filters = 9;
    string access_roles = 10;
    bool is_default = 11;
    string status = 12;
    int32 version = 13;
}

message FinancialMetrics {
    string id = 1;
    string tenant_id = 2;
    string period = 3;
    string legal_entity_id = 4;
    string department_id = 5;
    string cost_center_id = 6;
    int64 total_revenue_cents = 7;
    int64 total_expense_cents = 8;
    int64 gross_profit_cents = 9;
    int64 net_income_cents = 10;
    int64 ebitda_cents = 11;
    double operating_margin_pct = 12;
    double revenue_growth_pct = 13;
    double expense_growth_pct = 14;
    int64 ap_balance_cents = 15;
    int64 ar_balance_cents = 16;
    int64 cash_balance_cents = 17;
    double dso_days = 18;
    double dpo_days = 19;
    double cash_conversion_cycle_days = 20;
    double budget_variance_pct = 21;
    int32 journal_entry_count = 22;
    int32 invoice_count = 23;
    int32 po_count = 24;
    int32 project_count = 25;
}

message SpendAnalysisEntry {
    string id = 1;
    string tenant_id = 2;
    string period = 3;
    string spend_category = 4;
    string supplier_id = 5;
    string department_id = 6;
    string cost_center_id = 7;
    int64 total_spend_cents = 8;
    int32 invoice_count = 9;
    int32 po_count = 10;
    int64 po_backed_spend_cents = 11;
    int64 non_po_spend_cents = 12;
    int64 contract_spend_cents = 13;
    int64 maverick_spend_cents = 14;
    double maverick_spend_pct = 15;
    double po_coverage_pct = 16;
    double avg_payment_terms_days = 17;
    double contract_compliance_pct = 18;
    int32 supplier_count = 19;
}

// Request/Response messages
message CreateDashboardRequest {
    string tenant_id = 1;
    string dashboard_code = 2;
    string dashboard_name = 3;
    string dashboard_type = 4;
    string description = 5;
    string category = 6;
    string widgets = 7;
    string access_roles = 8;
}

message GetDashboardRequest {
    string id = 1;
    string tenant_id = 2;
}

message ListDashboardsRequest {
    string tenant_id = 1;
    string dashboard_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message DashboardList {
    repeated Dashboard dashboards = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetFinancialMetricsRequest {
    string tenant_id = 1;
    string period = 2;
    string legal_entity_id = 3;
    string department_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message FinancialMetricsList {
    repeated FinancialMetrics metrics = 1;
    string next_page_token = 2;
}

message GetSpendAnalysisRequest {
    string tenant_id = 1;
    string period = 2;
    string spend_category = 3;
    string supplier_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message SpendAnalysisList {
    repeated SpendAnalysisEntry entries = 1;
    string next_page_token = 2;
}

message GetKpiTrendRequest {
    string kpi_id = 1;
    string tenant_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message KpiTrendResponse {
    string kpi_id = 1;
    repeated double values = 2;
    repeated string dates = 3;
}

message RefreshKpisRequest {
    string tenant_id = 1;
    repeated string kpi_ids = 2;
}

message RefreshKpisResponse {
    int32 refreshed_count = 1;
}

message ExecuteReportRequest {
    string tenant_id = 1;
    string report_id = 2;
    string filters = 3;
    string output_format = 4;
}

message ReportExecutionResult {
    string execution_id = 1;
    int32 row_count = 2;
    string output_path = 3;
    string status = 4;
}
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | ea_dashboards | -- |
| V002 | ea_kpi_definitions | -- |
| V003 | ea_financial_metrics | -- |
| V004 | ea_spend_analysis | -- |
| V005 | ea_custom_reports | -- |

---

## 7. Business Rules

1. **Data Aggregation**: Financial metrics aggregated daily from ERP transactional data
2. **Multi-Currency**: All currency-aware KPIs converted to tenant base currency
3. **Threshold Alerts**: Auto-notify CFO/controller when financial KPIs breach thresholds
4. **Drill-Down**: All dashboard widgets support drill-down to transactional detail
5. **Budget Variance**: Budget vs actual calculated at period close; real-time estimate during period
6. **Access Control**: Financial data access restricted by cost center/legal entity permissions
7. **Spend Classification**: Auto-categorize spend using ML classification on invoice descriptions

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| General Ledger (06) | GL balances and journal data |
| Accounts Payable (07) | AP aging and payment data |
| Accounts Receivable (08) | AR aging and collection data |
| Cash Management (10) | Cash position and forecasting |
| Procurement (11) | Purchase order and spend data |
| Project Management (15) | Project cost and revenue data |
| Revenue Management (26) | Revenue recognition data |
| Tax Management (23) | Tax liability data |
| EPM (33) | Budget targets for variance analysis |
| Fusion Data Intelligence (102) | Cross-domain analytics platform |
| AI Agents (134) | Anomaly detection and insights |
