# 154 - Financial Reporting Service Specification

## 1. Domain Overview

Financial Reporting provides pixel-perfect financial report generation for management, statutory, and regulatory purposes. Supports income statements, balance sheets, cash flow statements, and custom financial reports with dimensional analysis, drill-down capabilities, and side-by-side period comparisons. Includes report designer with grid-based layout, auto-calculating formulas, row/column dimensional intersections, and dynamic member selection. Supports report books (collections of reports), bursting by entity/department, and multi-GAAP reporting. Integrates with General Ledger for actuals, EPM for budgets, and Financial Consolidation for consolidated data.

**Bounded Context:** Financial Report Design & Generation
**Service Name:** `financial-reporting-service`
**Database:** `data/financial_reporting.db`
**HTTP Port:** 8172 | **gRPC Port:** 9172

---

## 2. Database Schema

### 2.1 Report Definitions
```sql
CREATE TABLE fr_report_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_code TEXT NOT NULL,
    report_name TEXT NOT NULL,
    report_type TEXT NOT NULL CHECK(report_type IN (
        'INCOME_STATEMENT','BALANCE_SHEET','CASH_FLOW','TRIAL_BALANCE',
        'STATEMENT_OF_EQUITY','CUSTOM_FINANCIAL','VARIANCE','TREND'
    )),
    description TEXT,
    gaap_standard TEXT NOT NULL DEFAULT 'US_GAAP'
        CHECK(gaap_standard IN ('US_GAAP','IFRS','LOCAL_GAAP','MULTI_GAAP')),
    base_currency TEXT NOT NULL DEFAULT 'USD',
    row_dimension TEXT NOT NULL CHECK(row_dimension IN ('ACCOUNT','DEPARTMENT','ENTITY','CUSTOM')),
    column_dimension TEXT NOT NULL CHECK(column_dimension IN ('PERIOD','SCENARIO','ENTITY','CUSTOM')),
    default_columns TEXT NOT NULL,          -- JSON: column definitions
    default_rows TEXT NOT NULL,             -- JSON: row definitions with formulas
    formatting_rules TEXT,                  -- JSON: conditional formatting
    header_text TEXT,
    footer_text TEXT,
    page_layout TEXT NOT NULL DEFAULT 'PORTRAIT'
        CHECK(page_layout IN ('PORTRAIT','LANDSCAPE')),
    paper_size TEXT NOT NULL DEFAULT 'A4'
        CHECK(paper_size IN ('A4','LETTER','LEGAL','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, report_code)
);

CREATE INDEX idx_fr_report_tenant ON fr_report_definitions(tenant_id, report_type, status);
```

### 2.2 Report Grid Definitions
```sql
CREATE TABLE fr_grid_rows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    report_id TEXT NOT NULL,
    row_number INTEGER NOT NULL,
    row_label TEXT NOT NULL,
    row_type TEXT NOT NULL CHECK(row_type IN ('DATA','FORMULA','TEXT','HEADER','TOTAL','SUBTOTAL','BLANK')),
    account_range TEXT,                     -- e.g., "4000-4999" for revenue
    formula_expression TEXT,                -- e.g., "ROW_10 + ROW_20 - ROW_30"
    dimension_filters TEXT,                 -- JSON: dimension filters for this row
    indent_level INTEGER NOT NULL DEFAULT 0,
    bold INTEGER NOT NULL DEFAULT 0,
    underline INTEGER NOT NULL DEFAULT 0,
    number_format TEXT DEFAULT '#,##0',
    scaling_factor REAL NOT NULL DEFAULT 1, -- e.g., 1000 for thousands
    suppress_zero INTEGER NOT NULL DEFAULT 0,
    show_sign_reverse INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (report_id) REFERENCES fr_report_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, report_id, row_number)
);

CREATE TABLE fr_grid_columns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    report_id TEXT NOT NULL,
    column_number INTEGER NOT NULL,
    column_label TEXT NOT NULL,
    column_type TEXT NOT NULL CHECK(column_type IN ('ACTUAL','BUDGET','VARIANCE','VARIANCE_PCT','FORMULA','TEXT')),
    period_type TEXT NOT NULL CHECK(period_type IN ('CURRENT','YTD','QTD','PRIOR_YEAR','PRIOR_PERIOD','CUSTOM')),
    period_offset INTEGER NOT NULL DEFAULT 0, -- Relative to report period
    scenario_id TEXT,
    entity_id TEXT,
    formula_expression TEXT,                -- For formula columns
    column_width INTEGER NOT NULL DEFAULT 15,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (report_id) REFERENCES fr_report_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, report_id, column_number)
);
```

### 2.3 Report Books
```sql
CREATE TABLE fr_report_books (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    book_code TEXT NOT NULL,
    book_name TEXT NOT NULL,
    description TEXT,
    report_ids TEXT NOT NULL,               -- JSON ordered array of report IDs
    default_period TEXT NOT NULL,           -- e.g., "2024-Q4"
    burst_by TEXT CHECK(burst_by IN ('ENTITY','DEPARTMENT','NONE')),
    cover_page_template TEXT,
    table_of_contents INTEGER NOT NULL DEFAULT 1,
    page_numbering TEXT NOT NULL DEFAULT 'CONTINUOUS'
        CHECK(page_numbering IN ('CONTINUOUS','PER_REPORT')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, book_code)
);

CREATE INDEX idx_fr_book_tenant ON fr_report_books(tenant_id, status);
```

### 2.4 Report Execution
```sql
CREATE TABLE fr_report_executions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_id TEXT NOT NULL,
    book_id TEXT,
    execution_number TEXT NOT NULL,         -- FR-EXEC-2024-00001
    report_period TEXT NOT NULL,            -- "2024-12"
    scenario_id TEXT,
    entity_scope TEXT,                      -- JSON: entity/department filters
    output_format TEXT NOT NULL DEFAULT 'PDF'
        CHECK(output_format IN ('PDF','XLSX','HTML','CSV')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED')),
    output_path TEXT,
    file_size_bytes INTEGER,
    page_count INTEGER,
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    error_message TEXT,
    parameters TEXT,                        -- JSON: runtime parameters

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (report_id) REFERENCES fr_report_definitions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, execution_number)
);

CREATE INDEX idx_fr_exec_tenant ON fr_report_executions(tenant_id, created_at DESC);
CREATE INDEX idx_fr_exec_report ON fr_report_executions(report_id, created_at DESC);
```

---

## 3. API Endpoints

### 3.1 Report Definitions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-reporting/v1/reports` | Create report definition |
| GET | `/financial-reporting/v1/reports` | List reports |
| GET | `/financial-reporting/v1/reports/{id}` | Get report with grid |
| PUT | `/financial-reporting/v1/reports/{id}` | Update report |
| DELETE | `/financial-reporting/v1/reports/{id}` | Archive report |
| POST | `/financial-reporting/v1/reports/{id}/copy` | Copy report |

### 3.2 Grid Design
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-reporting/v1/reports/{id}/rows` | Add row |
| PUT | `/financial-reporting/v1/rows/{id}` | Update row |
| DELETE | `/financial-reporting/v1/rows/{id}` | Remove row |
| POST | `/financial-reporting/v1/reports/{id}/rows/reorder` | Reorder rows |
| POST | `/financial-reporting/v1/reports/{id}/columns` | Add column |
| PUT | `/financial-reporting/v1/columns/{id}` | Update column |
| DELETE | `/financial-reporting/v1/columns/{id}` | Remove column |

### 3.3 Report Books
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-reporting/v1/books` | Create report book |
| GET | `/financial-reporting/v1/books` | List books |
| GET | `/financial-reporting/v1/books/{id}` | Get book details |
| PUT | `/financial-reporting/v1/books/{id}` | Update book |

### 3.4 Execution & Output
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-reporting/v1/reports/{id}/run` | Run report |
| POST | `/financial-reporting/v1/books/{id}/run` | Run report book |
| GET | `/financial-reporting/v1/executions` | List executions |
| GET | `/financial-reporting/v1/executions/{id}` | Get execution status |
| GET | `/financial-reporting/v1/executions/{id}/output` | Download output |
| GET | `/financial-reporting/v1/executions/{id}/drilldown` | Drill-down to detail |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `fr.report.executed` | `{ execution_id, report_id, period, pages }` | Report generated |
| `fr.report.failed` | `{ execution_id, error }` | Report generation failed |
| `fr.book.executed` | `{ execution_id, book_id, report_count }` | Report book generated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `gl.period.closed` | General Ledger | Refresh available reporting periods |
| `consolidation.completed` | Financial Consolidation | Enable consolidated reports |
| `epm.budget.approved` | EPM | Enable budget variance columns |

---

## 5. Business Rules

1. **Account Mapping**: Row account ranges validated against GL chart of accounts
2. **Formula Validation**: All formulas validated for circular references before saving
3. **Zero Suppression**: Rows with all zero values optionally suppressed
4. **Scaling**: Financial amounts can be displayed in thousands, millions, or full precision
5. **Multi-Period**: Support current, prior, YTD, QTD, and custom period columns
6. **Drill-Down**: All data cells support drill-down to journal detail level
7. **Report Book Sequence**: Reports in books generated in defined order with continuous pagination

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| General Ledger (06) | Actual financial data |
| EPM (33) | Budget and forecast data |
| Financial Consolidation (100) | Consolidated financial data |
| Account Reconciliation (101) | Verified account balances |
| BI Publisher (150) | Report rendering engine |
| Reporting (17) | Report catalog integration |
| Tax Reporting (85) | Tax provision data |
| Intercompany (24) | Intercompany elimination data |
