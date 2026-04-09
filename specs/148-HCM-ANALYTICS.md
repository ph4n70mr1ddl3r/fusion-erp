# 148 - HCM Analytics Service Specification

## 1. Domain Overview

HCM Analytics provides prebuilt workforce analytics, KPI dashboards, and predictive insights across the human capital management lifecycle. Covers workforce composition, turnover analysis, compensation benchmarking, diversity & inclusion metrics, recruitment funnel analytics, learning effectiveness, absence patterns, and performance distribution. Supports custom report building, ad-hoc analysis, embedded visualizations, and scheduled report distribution. Integrates with all HCM modules for comprehensive workforce data.

**Bounded Context:** Workforce Analytics & HCM Intelligence
**Service Name:** `hcm-analytics-service`
**Database:** `data/hcm_analytics.db`
**HTTP Port:** 8166 | **gRPC Port:** 9166

---

## 2. Database Schema

### 2.1 Analytics Dashboards
```sql
CREATE TABLE ha_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_code TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    dashboard_type TEXT NOT NULL CHECK(dashboard_type IN (
        'WORKFORCE_SUMMARY','TURNOVER','COMPENSATION','DIVERSITY_INCLUSION',
        'RECRUITING','LEARNING','PERFORMANCE','ABSENCE','SAFETY','CUSTOM'
    )),
    description TEXT,
    category TEXT NOT NULL,
    widgets TEXT NOT NULL,                  -- JSON array: widget configurations
    filters TEXT,                           -- JSON: available filter dimensions
    access_roles TEXT NOT NULL,             -- JSON array of role IDs with access
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

CREATE INDEX idx_ha_dash_tenant ON ha_dashboards(tenant_id, dashboard_type);
```

### 2.2 KPI Definitions
```sql
CREATE TABLE ha_kpi_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_code TEXT NOT NULL,
    kpi_name TEXT NOT NULL,
    kpi_category TEXT NOT NULL CHECK(kpi_category IN (
        'HEADCOUNT','TURNOVER','COMPENSATION','RECRUITING','PERFORMANCE',
        'LEARNING','DIVERSITY','ABSENCE','SAFETY','ENGAGEMENT','CUSTOM'
    )),
    description TEXT,
    calculation_type TEXT NOT NULL CHECK(calculation_type IN ('COUNT','RATIO','PERCENTAGE','AVERAGE','MEDIAN','SUM','CUSTOM')),
    measure_expression TEXT NOT NULL,       -- SQL-like expression for calculation
    dimensions TEXT NOT NULL,               -- JSON: available breakdown dimensions
    target_value REAL,
    warning_threshold REAL,
    critical_threshold REAL,
    threshold_direction TEXT NOT NULL DEFAULT 'BELOW'
        CHECK(threshold_direction IN ('BELOW','ABOVE')),
    unit TEXT NOT NULL DEFAULT 'NUMBER',    -- NUMBER, PERCENTAGE, CURRENCY, DAYS
    refresh_frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(refresh_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY','MONTHLY')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, kpi_code)
);

CREATE INDEX idx_ha_kpi_tenant ON ha_kpi_definitions(tenant_id, kpi_category);
```

### 2.3 KPI Snapshots
```sql
CREATE TABLE ha_kpi_snapshots (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_id TEXT NOT NULL,
    snapshot_date TEXT NOT NULL,
    dimension_type TEXT,                    -- DEPARTMENT, LOCATION, JOB, GRADE, etc.
    dimension_value TEXT,
    measure_value REAL NOT NULL,
    target_value REAL,
    status TEXT NOT NULL CHECK(status IN ('ON_TARGET','WARNING','CRITICAL')),
    record_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (kpi_id) REFERENCES ha_kpi_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, kpi_id, snapshot_date, dimension_type, dimension_value)
);

CREATE INDEX idx_ha_kpi_snap_date ON ha_kpi_snapshots(kpi_id, snapshot_date DESC);
CREATE INDEX idx_ha_kpi_snap_tenant ON ha_kpi_snapshots(tenant_id, snapshot_date DESC);
```

### 2.4 Workforce Metrics
```sql
CREATE TABLE ha_workforce_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-01"
    department_id TEXT,
    location_id TEXT,
    job_family TEXT,
    headcount_start INTEGER NOT NULL DEFAULT 0,
    headcount_end INTEGER NOT NULL DEFAULT 0,
    new_hires INTEGER NOT NULL DEFAULT 0,
    terminations_voluntary INTEGER NOT NULL DEFAULT 0,
    terminations_involuntary INTEGER NOT NULL DEFAULT 0,
    internal_movements INTEGER NOT NOT NULL DEFAULT 0,
    avg_tenure_months REAL NOT NULL DEFAULT 0,
    avg_age REAL NOT NULL DEFAULT 0,
    avg_compensation_cents INTEGER NOT NULL DEFAULT 0,
    avg_performance_rating REAL NOT NULL DEFAULT 0,
    diversity_female_pct REAL NOT NULL DEFAULT 0,
    diversity_minority_pct REAL NOT NULL DEFAULT 0,
    absence_rate_pct REAL NOT NULL DEFAULT 0,
    training_hours_per_employee REAL NOT NULL DEFAULT 0,
    open_positions INTEGER NOT NULL DEFAULT 0,
    time_to_fill_days REAL NOT NULL DEFAULT 0,
    turnover_rate_pct REAL NOT NULL DEFAULT 0,
    regrettable_turnover_pct REAL NOT NULL DEFAULT 0,
    span_of_control REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, period, department_id, location_id, job_family)
);

CREATE INDEX idx_ha_metric_period ON ha_workforce_metrics(tenant_id, period DESC);
CREATE INDEX idx_ha_metric_dept ON ha_workforce_metrics(tenant_id, department_id, period DESC);
```

### 2.5 Custom Reports
```sql
CREATE TABLE ha_custom_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_name TEXT NOT NULL,
    report_type TEXT NOT NULL CHECK(report_type IN ('TABULAR','PIVOT','CHART','DASHBOARD','EXPORT')),
    description TEXT,
    data_source TEXT NOT NULL,              -- Which metrics/tables to query
    columns TEXT NOT NULL,                  -- JSON: column definitions
    filters TEXT,                           -- JSON: default filters
    grouping TEXT,                          -- JSON: group by dimensions
    sort_config TEXT,                       -- JSON: sort order
    visualization_config TEXT,              -- JSON: chart type, colors, etc.
    schedule TEXT,                          -- JSON: scheduled run config
    recipients TEXT,                        -- JSON: email/distribution list
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
| POST | `/hcm-analytics/v1/dashboards` | Create dashboard |
| GET | `/hcm-analytics/v1/dashboards` | List dashboards |
| GET | `/hcm-analytics/v1/dashboards/{id}` | Get dashboard with data |
| PUT | `/hcm-analytics/v1/dashboards/{id}` | Update dashboard |
| DELETE | `/hcm-analytics/v1/dashboards/{id}` | Archive dashboard |

### 3.2 KPIs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/hcm-analytics/v1/kpis` | List KPIs with current values |
| GET | `/hcm-analytics/v1/kpis/{id}/trend` | Get KPI trend over time |
| GET | `/hcm-analytics/v1/kpis/{id}/breakdown` | Get KPI breakdown by dimension |
| POST | `/hcm-analytics/v1/kpis/refresh` | Trigger KPI refresh |

### 3.3 Workforce Metrics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/hcm-analytics/v1/workforce-metrics` | Query workforce metrics |
| GET | `/hcm-analytics/v1/workforce-metrics/trend` | Metric trends |
| GET | `/hcm-analytics/v1/workforce-metrics/comparison` | Period-over-period comparison |
| POST | `/hcm-analytics/v1/workforce-metrics/calculate` | Calculate metrics for period |

### 3.4 Custom Reports
| Method | Path | Description |
|--------|------|-------------|
| POST | `/hcm-analytics/v1/reports` | Create custom report |
| GET | `/hcm-analytics/v1/reports` | List reports |
| POST | `/hcm-analytics/v1/reports/{id}/execute` | Execute report |
| GET | `/hcm-analytics/v1/reports/{id}/results` | Get report results |
| POST | `/hcm-analytics/v1/reports/{id}/schedule` | Schedule report |
| POST | `/hcm-analytics/v1/reports/{id}/export` | Export report (CSV, PDF, Excel) |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `hcm.kpi.threshold_breach` | `{ kpi_id, value, threshold, direction }` | KPI crossed threshold |
| `hcm.metrics.calculated` | `{ period, metrics_count }` | Workforce metrics calculated |
| `hcm.report.completed` | `{ report_id, row_count }` | Custom report execution completed |
| `hcm.dashboard.updated` | `{ dashboard_id }` | Dashboard data refreshed |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `employee.hired` | Core HR | Update headcount, new hire metrics |
| `employee.terminated` | Core HR | Update turnover metrics |
| `compensation.changed` | Compensation | Update compensation metrics |
| `performance.review.completed` | Performance Mgmt | Update performance metrics |
| `absence.approved` | Absence Mgmt | Update absence metrics |
| `learning.course.completed` | Learning | Update training metrics |
| `requisition.filled` | Recruiting | Update time-to-fill metrics |

---

## 5. Business Rules

1. **Data Aggregation**: Workforce metrics aggregated daily from transactional HCM data
2. **KPI Refresh**: KPIs refreshed at configured frequency; REAL_TIME for critical KPIs
3. **Threshold Alerts**: Automatic notification when KPI crosses warning or critical threshold
4. **Access Control**: Dashboard and report access controlled by HCM role assignments
5. **Data Privacy**: PII data masked based on viewer permissions (e.g., managers see only their org)
6. **Historical Retention**: Monthly metrics retained for 5 years; daily snapshots for 2 years
7. **Custom Report Limits**: Maximum 50 custom reports per tenant; max 100K rows per execution

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Core HR (62) | Employee master data, organizational data |
| Compensation (65) | Compensation data for benchmarking |
| Performance Management (68) | Performance ratings |
| Recruiting (67) | Recruitment funnel data |
| Learning & Development (69) | Training completion data |
| Absence Management (72) | Leave and absence data |
| Benefits (66) | Benefits enrollment data |
| Workforce Safety (76) | Safety incident data |
| Fusion Data Intelligence (102) | Cross-domain analytics platform |
| Reporting (17) | Report generation and distribution |
