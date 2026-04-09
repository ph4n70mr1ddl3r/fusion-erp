# 176 - Spreadsheet Designer Service Specification

## 1. Domain Overview

Spreadsheet Designer provides web-based Excel-like report design with live data connections to Fusion application data sources. Enables business users to create formatted spreadsheet reports with dynamic data bindings, formulas, conditional formatting, charts, and multi-sheet layouts directly in the browser. Supports template-based report generation, Excel export with full formatting fidelity, and scheduled distribution. Distinct from Smart View (155) which connects desktop Excel to data; Spreadsheet Designer is a web-native designer that generates formatted Excel outputs.

**Bounded Context:** Web-Based Spreadsheet Report Design
**Service Name:** `spreadsheet-designer-service`
**Database:** `data/spreadsheet_designer.db`
**HTTP Port:** 8194 | **gRPC Port:** 9194

---

## 2. Database Schema

### 2.1 Spreadsheet Templates
```sql
CREATE TABLE sd_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    data_source_config TEXT NOT NULL,        -- JSON: { source: "...", connection: "..." }
    sheet_count INTEGER NOT NULL DEFAULT 1,
    layout_config TEXT NOT NULL,             -- JSON: sheet/cell layout definitions
    cell_definitions TEXT NOT NULL,          -- JSON: { cell_ref: { type, formula, binding, format } }
    conditional_formats TEXT,                -- JSON: conditional formatting rules
    chart_definitions TEXT,                  -- JSON: embedded chart configs
    named_ranges TEXT,                       -- JSON: named range definitions
    parameters TEXT,                         -- JSON: report parameters
    default_output_format TEXT NOT NULL DEFAULT 'XLSX'
        CHECK(default_output_format IN ('XLSX','XLSM','CSV','PDF')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_sd_template_tenant ON sd_templates(tenant_id, status);
```

### 2.2 Cell Definitions
```sql
CREATE TABLE sd_cell_defs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    sheet_index INTEGER NOT NULL DEFAULT 0,
    cell_reference TEXT NOT NULL,            -- "A1", "B2:G20" for ranges
    cell_type TEXT NOT NULL CHECK(cell_type IN ('LABEL','DATA_BINDING','FORMULA','AGGREGATION','IMAGE','CHART')),
    value TEXT,                              -- Static value or label
    data_binding TEXT,                       -- JSON: { field, dimension, filter }
    formula TEXT,                            -- Excel formula
    number_format TEXT,                      -- "#,##0.00", "$#,##0", "0%"
    font_config TEXT,                        -- JSON: font family, size, bold, color
    fill_config TEXT,                        -- JSON: background color, pattern
    border_config TEXT,                      -- JSON: border styles
    alignment TEXT,                          -- JSON: horizontal, vertical, wrap
    column_width REAL,
    row_height REAL,
    merge_span TEXT,                         -- "3x1" for 3-column merge

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (template_id) REFERENCES sd_templates(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, template_id, sheet_index, cell_reference)
);
```

### 2.3 Data Binding Rules
```sql
CREATE TABLE sd_data_bindings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    binding_name TEXT NOT NULL,
    source_entity TEXT NOT NULL,             -- Table/view name
    fields TEXT NOT NULL,                    -- JSON: selected fields
    filters TEXT,                            -- JSON: filter conditions
    sort_config TEXT,                        -- JSON: sort order
    grouping TEXT,                           -- JSON: group by fields
    aggregation TEXT,                        -- JSON: aggregate functions
    row_expansion TEXT NOT NULL CHECK(row_expansion IN ('FIXED','DYNAMIC','REPEATING')),
    start_cell TEXT NOT NULL,                -- Where data starts
    orientation TEXT NOT NULL DEFAULT 'VERTICAL'
        CHECK(orientation IN ('VERTICAL','HORIZONTAL','CROSSTAB')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (template_id) REFERENCES sd_templates(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, template_id, binding_name)
);
```

### 2.4 Generated Reports
```sql
CREATE TABLE sd_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    report_number TEXT NOT NULL,             -- SDR-2024-00001
    parameters TEXT,                         -- JSON: runtime parameters
    output_format TEXT NOT NULL DEFAULT 'XLSX',
    output_path TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL DEFAULT 0,
    sheet_count INTEGER NOT NULL DEFAULT 1,
    row_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','GENERATING','COMPLETED','FAILED')),
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (template_id) REFERENCES sd_templates(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, report_number)
);

CREATE INDEX idx_sd_report_template ON sd_reports(template_id, created_at DESC);
CREATE INDEX idx_sd_report_status ON sd_reports(tenant_id, status);
```

---

## 3. API Endpoints

### 3.1 Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/spreadsheet-designer/v1/templates` | Create template |
| GET | `/spreadsheet-designer/v1/templates` | List templates |
| GET | `/spreadsheet-designer/v1/templates/{id}` | Get template details |
| PUT | `/spreadsheet-designer/v1/templates/{id}` | Update template |
| DELETE | `/spreadsheet-designer/v1/templates/{id}` | Archive template |
| POST | `/spreadsheet-designer/v1/templates/{id}/clone` | Clone template |

### 3.2 Cell Design
| Method | Path | Description |
|--------|------|-------------|
| PUT | `/spreadsheet-designer/v1/templates/{id}/cells` | Bulk update cells |
| POST | `/spreadsheet-designer/v1/templates/{id}/data-bindings` | Add data binding |
| PUT | `/spreadsheet-designer/v1/templates/{id}/format` | Apply formatting |

### 3.3 Generation
| Method | Path | Description |
|--------|------|-------------|
| POST | `/spreadsheet-designer/v1/templates/{id}/generate` | Generate report |
| GET | `/spreadsheet-designer/v1/reports` | List generated reports |
| GET | `/spreadsheet-designer/v1/reports/{id}` | Get report details |
| GET | `/spreadsheet-designer/v1/reports/{id}/download` | Download file |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sdesigner.report.generated` | `{ report_id, template_id, format, rows }` | Report generated |
| `sdesigner.report.failed` | `{ report_id, error }` | Generation failed |

---

## 5. Business Rules

1. **Cell Limits**: Max 1M cells per sheet; max 50 sheets per template
2. **Data Binding**: Dynamic data bindings executed at generation time
3. **Formula Support**: Standard Excel formulas supported; custom functions via config
4. **Format Fidelity**: Generated XLSX maintains full Excel formatting compatibility
5. **Template Versioning**: Published templates versioned; changes create new version

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| BI Publisher (150) | Report rendering engine |
| OTBI (174) | Data source for analysis |
| Enterprise Scheduler (173) | Scheduled report generation |
| Data Import/Export (30) | Data source connections |
| Smart View (155) | Template compatibility |
| Reporting (17) | Report catalog |
