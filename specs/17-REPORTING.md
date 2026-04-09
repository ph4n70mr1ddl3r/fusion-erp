# 17 - Reporting & Analytics Service Specification

## 1. Domain Overview

The Reporting service aggregates data across all services, generates financial and operational reports, provides dashboard metrics, and supports ad-hoc queries and data export.

**Bounded Context:** Reporting, Analytics & Data Warehousing
**Service Name:** `report-service`
**Database:** `data/report.db`
**HTTP Port:** 8050 | **gRPC Port:** 9050

---

## 2. Database Schema

### 2.1 Report Definitions
```sql
CREATE TABLE report_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_code TEXT NOT NULL,
    report_name TEXT NOT NULL,
    description TEXT,
    report_type TEXT NOT NULL
        CHECK(report_type IN ('FINANCIAL','OPERATIONAL','REGULATORY','AD_HOC','DASHBOARD')),
    category TEXT,
    data_source TEXT NOT NULL,             -- Which service(s) to query
    query_template TEXT NOT NULL,          -- SQL template or API query definition
    parameters TEXT,                       -- JSON: parameter definitions
    output_formats TEXT NOT NULL DEFAULT '["JSON","CSV"]',
    is_scheduled INTEGER NOT NULL DEFAULT 0,
    default_sort TEXT,
    default_filters TEXT,                  -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, report_code)
);
```

### 2.2 Report Executions
```sql
CREATE TABLE report_executions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_definition_id TEXT NOT NULL,
    executed_by TEXT NOT NULL,
    executed_at TEXT NOT NULL DEFAULT (datetime('now')),
    parameters TEXT,                       -- JSON: actual parameters used
    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED','CANCELLED')),
    row_count INTEGER,
    execution_time_ms INTEGER,
    output_format TEXT NOT NULL DEFAULT 'JSON',
    output_location TEXT,                  -- File path for large reports
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (report_definition_id) REFERENCES report_definitions(id) ON DELETE RESTRICT
);
```

### 2.3 Scheduled Reports
```sql
CREATE TABLE scheduled_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_definition_id TEXT NOT NULL,
    schedule_cron TEXT NOT NULL,           -- Cron expression
    recipients TEXT NOT NULL,              -- JSON array of user IDs
    output_format TEXT NOT NULL DEFAULT 'CSV',
    parameters TEXT,                       -- JSON: fixed parameters
    is_active INTEGER NOT NULL DEFAULT 1,
    last_run_at TEXT,
    next_run_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (report_definition_id) REFERENCES report_definitions(id) ON DELETE RESTRICT
);
```

### 2.4 Dashboard Widgets
```sql
CREATE TABLE dashboard_widgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    widget_type TEXT NOT NULL
        CHECK(widget_type IN ('KPI','CHART','TABLE','GAUGE','LIST','IFRAME')),
    title TEXT NOT NULL,
    description TEXT,
    data_source TEXT NOT NULL,             -- API endpoint or metric query
    refresh_interval_seconds INTEGER DEFAULT 300,
    config TEXT NOT NULL,                  -- JSON: widget configuration (chart type, colors, etc.)
    position_x INTEGER NOT NULL DEFAULT 0,
    position_y INTEGER NOT NULL DEFAULT 0,
    width INTEGER NOT NULL DEFAULT 4,
    height INTEGER NOT NULL DEFAULT 2,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE user_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    widget_ids TEXT NOT NULL,              -- JSON array of widget IDs
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, user_id, dashboard_name)
);
```

### 2.5 Materialized Aggregations (CQRS Read Model)
```sql
-- Pre-computed daily summaries for fast dashboard/report queries
CREATE TABLE daily_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,
    metric_type TEXT NOT NULL,             -- 'REVENUE', 'EXPENSE', 'CASH_FLOW', 'AR_AGING', etc.
    dimension_1 TEXT,                      -- e.g., customer_id, supplier_id, account_id
    dimension_2 TEXT,                      -- e.g., department, category
    value_cents INTEGER NOT NULL DEFAULT 0,
    quantity DECIMAL(18,4),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    source_service TEXT NOT NULL,
    computed_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, metric_date, metric_type, dimension_1, dimension_2)
);

CREATE INDEX idx_daily_metrics_tenant_date ON daily_metrics(tenant_id, metric_date);
CREATE INDEX idx_daily_metrics_tenant_type ON daily_metrics(tenant_id, metric_type);
```

---

## 3. REST API Endpoints

```
# Report Definitions
GET/POST      /api/v1/reports/definitions             Permission: reports.definitions.read/create
GET/PUT       /api/v1/reports/definitions/{id}         Permission: reports.definitions.read/update

# Report Execution
POST          /api/v1/reports/execute                  Permission: reports.execute
GET           /api/v1/reports/executions/{id}           Permission: reports.execute
GET           /api/v1/reports/executions/{id}/download  Permission: reports.execute
POST          /api/v1/reports/execute/{id}/cancel       Permission: reports.execute

# Pre-Built Financial Reports (shortcut endpoints)
GET           /api/v1/reports/trial-balance            Permission: reports.financial.view
GET           /api/v1/reports/balance-sheet            Permission: reports.financial.view
GET           /api/v1/reports/income-statement         Permission: reports.financial.view
GET           /api/v1/reports/cash-flow                Permission: reports.financial.view
GET           /api/v1/reports/general-ledger           Permission: reports.financial.view
GET           /api/v1/reports/ap-aging                 Permission: reports.financial.view
GET           /api/v1/reports/ar-aging                 Permission: reports.financial.view
GET           /api/v1/reports/budget-vs-actual         Permission: reports.financial.view

# Scheduled Reports
GET/POST      /api/v1/reports/schedules                Permission: reports.schedules.read/create
PUT/DELETE    /api/v1/reports/schedules/{id}            Permission: reports.schedules.update

# Dashboards
GET/POST      /api/v1/reports/dashboards               Permission: reports.dashboards.read/create
GET/PUT       /api/v1/reports/dashboards/{id}           Permission: reports.dashboards.read/update
GET/POST      /api/v1/reports/widgets                   Permission: reports.widgets.read/create
GET           /api/v1/reports/widget-data/{id}          Permission: reports.widgets.read

# Data Export
POST          /api/v1/reports/export                   Permission: reports.export
  Request: { "query": "...", "format": "CSV", "filters": {...} }
```

---

## 4. Business Rules

### 4.1 Report Data Collection
- Report service calls other services via gRPC to collect data
- Uses streaming gRPC for large data sets
- Results cached in materialized aggregations (daily_metrics)
- Real-time reports query source services directly

### 4.2 Standard Financial Reports

**Trial Balance:**
```
GET /api/v1/reports/trial-balance?period=Mar-2024
→ Calls GL service: GetTrialBalance(period)
→ Returns accounts with debit/credit balances
```

**Balance Sheet:**
```
GET /api/v1/reports/balance-sheet?as_of=2024-03-31
→ Calls GL service: GetBalances(account_type IN [ASSET, LIABILITY, EQUITY], period)
→ Aggregates by account category
```

**Income Statement:**
```
GET /api/v1/reports/income-statement?from=Jan-2024&to=Mar-2024
→ Calls GL service: GetBalances(account_type IN [REVENUE, EXPENSE], periods)
→ Computes net income
```

**AP/AR Aging:**
```
GET /api/v1/reports/ap-aging?as_of=2024-01-31
→ Calls AP/AR service: GetAging(as_of_date)
→ Buckets: Current, 1-30, 31-60, 61-90, 90+ days
```

### 4.3 Data Aggregation Pipeline
1. **Event-driven:** Subscribe to events from all services
2. **Batch:** Periodic (hourly/daily) refresh of aggregation tables
3. **On-demand:** Query source services for real-time data
4. Metrics computed: revenue, expenses, cash position, open AR/AP, inventory value

### 4.4 Export Formats
- **JSON:** Default API response
- **CSV:** Flat table export, streamed for large datasets
- **PDF:** Formatted report (using Rust PDF generation library)
- **Excel (XLSX):** Multi-tab workbook (using rust_xlsxwriter crate)

### 4.5 Access Control
- Users can only run reports for their tenant
- Permission-based: `reports.financial.view`, `reports.operational.view`, `reports.export`
- Row-level filtering based on user's department (configurable)

### 4.6 gRPC Service
```protobuf
service ReportingService {
    rpc GetMetric(GetMetricRequest) returns (GetMetricResponse);
    rpc GetDashboard(GetDashboardRequest) returns (GetDashboardResponse);
    rpc StreamReportData(StreamReportDataRequest) returns (stream ReportDataRow);
}
```

```protobuf
message GetMetricRequest {
    string tenant_id = 1;
    string metric_type = 2;
    string metric_date = 3;
    string dimension_1 = 4;
    string dimension_2 = 5;
}

message GetMetricResponse {
    DailyMetric metric = 1;
    repeated DailyMetric metrics = 2;
}

message GetDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    string user_id = 3;
}

message GetDashboardResponse {
    UserDashboard dashboard = 1;
    repeated DashboardWidget widgets = 2;
}

message StreamReportDataRequest {
    string tenant_id = 1;
    string report_definition_id = 2;
    string output_format = 3;
    string parameters = 4;
}

message ReportDataRow {
    string row_data = 1;
    int32 row_number = 2;
}

message ReportDefinition {
    string id = 1;
    string tenant_id = 2;
    string report_code = 3;
    string report_name = 4;
    string description = 5;
    string report_type = 6;
    string category = 7;
    string data_source = 8;
    string query_template = 9;
    string parameters = 10;
    string output_formats = 11;
    int32 is_scheduled = 12;
    string default_sort = 13;
    string default_filters = 14;
    string created_at = 15;
    string updated_at = 16;
    string created_by = 17;
    string updated_by = 18;
    int32 version = 19;
    int32 is_active = 20;
}

message ReportExecution {
    string id = 1;
    string tenant_id = 2;
    string report_definition_id = 3;
    string executed_by = 4;
    string executed_at = 5;
    string parameters = 6;
    string status = 7;
    int32 row_count = 8;
    int32 execution_time_ms = 9;
    string output_format = 10;
    string output_location = 11;
    string error_message = 12;
    string created_at = 13;
}

message ScheduledReport {
    string id = 1;
    string tenant_id = 2;
    string report_definition_id = 3;
    string schedule_cron = 4;
    string recipients = 5;
    string output_format = 6;
    string parameters = 7;
    int32 is_active = 8;
    string last_run_at = 9;
    string next_run_at = 10;
    string created_at = 11;
    string updated_at = 12;
    string created_by = 13;
    string updated_by = 14;
    int32 version = 15;
}

message DashboardWidget {
    string id = 1;
    string tenant_id = 2;
    string widget_type = 3;
    string title = 4;
    string description = 5;
    string data_source = 6;
    int32 refresh_interval_seconds = 7;
    string config = 8;
    int32 position_x = 9;
    int32 position_y = 10;
    int32 width = 11;
    int32 height = 12;
    string created_at = 13;
    string updated_at = 14;
    string created_by = 15;
    string updated_by = 16;
    int32 version = 17;
    int32 is_active = 18;
}

message UserDashboard {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string dashboard_name = 4;
    string widget_ids = 5;
    int32 is_default = 6;
    string created_at = 7;
    string updated_at = 8;
}

message DailyMetric {
    string id = 1;
    string tenant_id = 2;
    string metric_date = 3;
    string metric_type = 4;
    string dimension_1 = 5;
    string dimension_2 = 6;
    int64 value_cents = 7;
    string quantity = 8;
    string currency_code = 9;
    string source_service = 10;
    string computed_at = 11;
}
```
