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

## 5. Business Rules

1. **Data Aggregation**: Financial metrics aggregated daily from ERP transactional data
2. **Multi-Currency**: All currency-aware KPIs converted to tenant base currency
3. **Threshold Alerts**: Auto-notify CFO/controller when financial KPIs breach thresholds
4. **Drill-Down**: All dashboard widgets support drill-down to transactional detail
5. **Budget Variance**: Budget vs actual calculated at period close; real-time estimate during period
6. **Access Control**: Financial data access restricted by cost center/legal entity permissions
7. **Spend Classification**: Auto-categorize spend using ML classification on invoice descriptions

---

## 6. Integration Points

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
