# 150 - BI Publisher Service Specification

## 1. Domain Overview

BI Publisher provides pixel-perfect, template-based report generation for operational and financial reporting. Supports RTF, PDF, Excel, HTML, and PowerPoint output formats with dynamic data binding, conditional formatting, charts, and multi-language support. Enables business users to design report templates using familiar desktop tools, connect to multiple data sources, schedule report generation, and distribute via email, print, or archive. Integrates with all application modules for enterprise-wide reporting.

**Bounded Context:** Template-Based Report Generation & Distribution
**Service Name:** `bi-publisher-service`
**Database:** `data/bi_publisher.db`
**HTTP Port:** 8168 | **gRPC Port:** 9168

---

## 2. Database Schema

### 2.1 Report Definitions
```sql
CREATE TABLE bp_report_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_code TEXT NOT NULL,
    report_name TEXT NOT NULL,
    description TEXT,
    report_category TEXT NOT NULL CHECK(report_category IN (
        'FINANCIAL','OPERATIONAL','REGULATORY','INVOICE','STATEMENT',
        'PURCHASING','SHIPPING','HR','PAYROLL','CUSTOM'
    )),
    data_source_config TEXT NOT NULL,       -- JSON: { type: "SQL|REST|GQL", connection: "...", query: "..." }
    default_template_id TEXT,
    default_output_format TEXT NOT NULL DEFAULT 'PDF'
        CHECK(default_output_format IN ('PDF','RTF','XLSX','HTML','PPTX','CSV','XML','JSON')),
    parameters TEXT,                        -- JSON: report parameter definitions
    cache_duration_minutes INTEGER NOT NULL DEFAULT 0,
    max_runtime_seconds INTEGER NOT NULL DEFAULT 300,
    row_limit INTEGER NOT NULL DEFAULT 100000,
    requires_parameters INTEGER NOT NULL DEFAULT 0,
    multi_language INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, report_code)
);

CREATE INDEX idx_bp_report_tenant ON bp_report_definitions(tenant_id, report_category, status);
```

### 2.2 Report Templates
```sql
CREATE TABLE bp_report_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_type TEXT NOT NULL CHECK(template_type IN ('RTF','PDF','XLSX','HTML','PPTX','ETEXT')),
    storage_path TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    layout_config TEXT,                     -- JSON: page size, orientation, margins
    data_mapping TEXT NOT NULL,             -- JSON: field bindings
    conditional_formatting TEXT,            -- JSON: conditional rules
    header_template TEXT,
    footer_template TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    language_code TEXT NOT NULL DEFAULT 'en',
    version_number INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (report_id) REFERENCES bp_report_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, report_id, template_name, language_code)
);

CREATE INDEX idx_bp_template_report ON bp_report_templates(report_id);
```

### 2.3 Report Jobs
```sql
CREATE TABLE bp_report_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_id TEXT NOT NULL,
    template_id TEXT,
    job_number TEXT NOT NULL,               -- RJ-2024-00001
    job_type TEXT NOT NULL CHECK(job_type IN ('IMMEDIATE','SCHEDULED','BURST','ON_DEMAND')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','CANCELLED')),
    parameters TEXT,                        -- JSON: runtime parameter values
    output_format TEXT NOT NULL DEFAULT 'PDF',
    output_file_path TEXT,
    output_file_size_bytes INTEGER,
    output_mime_type TEXT,
    row_count INTEGER NOT NULL DEFAULT 0,
    page_count INTEGER NOT NULL DEFAULT 0,
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    error_message TEXT,
    locale TEXT NOT NULL DEFAULT 'en_US',
    timezone TEXT NOT NULL DEFAULT 'UTC',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (report_id) REFERENCES bp_report_definitions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, job_number)
);

CREATE INDEX idx_bp_job_tenant ON bp_report_jobs(tenant_id, report_id, created_at DESC);
CREATE INDEX idx_bp_job_status ON bp_report_jobs(tenant_id, status);
```

### 2.4 Report Schedules
```sql
CREATE TABLE bp_report_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    cron_expression TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    template_id TEXT,
    output_format TEXT NOT NULL DEFAULT 'PDF',
    parameters TEXT,                        -- JSON: static parameters
    dynamic_parameters TEXT,                -- JSON: parameter calculation expressions
    distribution_method TEXT NOT NULL CHECK(distribution_method IN ('EMAIL','FTP','SHARE','ARCHIVE','PRINT','WEBHOOK')),
    distribution_config TEXT NOT NULL,      -- JSON: email addresses, FTP path, etc.
    burst_config TEXT,                      -- JSON: bursting rules for split delivery
    retain_output_days INTEGER NOT NULL DEFAULT 30,
    active_from TEXT,
    active_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','EXPIRED','CANCELLED')),
    last_run_at TEXT,
    next_run_at TEXT,
    run_count INTEGER NOT NULL DEFAULT 0,
    last_failure_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (report_id) REFERENCES bp_report_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, schedule_name)
);

CREATE INDEX idx_bp_sched_tenant ON bp_report_schedules(tenant_id, status);
CREATE INDEX idx_bp_sched_next ON bp_report_schedules(next_run_at, status);
```

### 2.5 Report Burst Config
```sql
CREATE TABLE bp_burst_config (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_id TEXT NOT NULL,
    burst_by TEXT NOT NULL CHECK(burst_by IN ('CUSTOMER','DEPARTMENT','LEGAL_ENTITY','LOCATION','CUSTOM_FIELD')),
    burst_field TEXT NOT NULL,              -- Field name to split on
    delivery_rules TEXT NOT NULL,           -- JSON: per-burst-value delivery config
    filename_pattern TEXT NOT NULL,         -- e.g., "Invoice_{burst_value}_{date}.pdf"
    combine_threshold INTEGER,              -- Max burst sections before combining

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (schedule_id) REFERENCES bp_report_schedules(id) ON DELETE CASCADE
);
```

---

## 3. API Endpoints

### 3.1 Report Definitions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/bi-publisher/v1/reports` | Create report definition |
| GET | `/bi-publisher/v1/reports` | List reports |
| GET | `/bi-publisher/v1/reports/{id}` | Get report details |
| PUT | `/bi-publisher/v1/reports/{id}` | Update report |
| DELETE | `/bi-publisher/v1/reports/{id}` | Archive report |
| POST | `/bi-publisher/v1/reports/{id}/test-data` | Test data source connection |

### 3.2 Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/bi-publisher/v1/templates` | Upload template |
| GET | `/bi-publisher/v1/templates` | List templates |
| GET | `/bi-publisher/v1/templates/{id}` | Get template details |
| PUT | `/bi-publisher/v1/templates/{id}` | Update template |
| DELETE | `/bi-publisher/v1/templates/{id}` | Delete template |
| GET | `/bi-publisher/v1/templates/{id}/preview` | Preview template with sample data |

### 3.3 Report Execution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/bi-publisher/v1/reports/{id}/run` | Run report immediately |
| GET | `/bi-publisher/v1/jobs` | List report jobs |
| GET | `/bi-publisher/v1/jobs/{id}` | Get job status |
| GET | `/bi-publisher/v1/jobs/{id}/output` | Download report output |
| POST | `/bi-publisher/v1/jobs/{id}/cancel` | Cancel running job |
| POST | `/bi-publisher/v1/jobs/{id}/retry` | Retry failed job |

### 3.4 Scheduling & Distribution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/bi-publisher/v1/schedules` | Create schedule |
| GET | `/bi-publisher/v1/schedules` | List schedules |
| PUT | `/bi-publisher/v1/schedules/{id}` | Update schedule |
| POST | `/bi-publisher/v1/schedules/{id}/pause` | Pause schedule |
| POST | `/bi-publisher/v1/schedules/{id}/resume` | Resume schedule |
| DELETE | `/bi-publisher/v1/schedules/{id}` | Cancel schedule |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `bip.report.completed` | `{ job_id, report_id, page_count, file_size }` | Report generation completed |
| `bip.report.failed` | `{ job_id, report_id, error }` | Report generation failed |
| `bip.report.distributed` | `{ job_id, method, recipients }` | Report distributed |
| `bip.schedule.executed` | `{ schedule_id, job_id }` | Scheduled report triggered |
| `bip.schedule.failed` | `{ schedule_id, error, consecutive_failures }` | Schedule execution failed |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `gl.period.closed` | General Ledger | Trigger financial statement generation |
| `invoice.created` | AP/AR | Trigger invoice/statement generation |
| `payroll.run.completed` | Payroll | Trigger payslip generation |
| `po.approved` | Procurement | Trigger PO document generation |

---

## 5. Business Rules

1. **Template Validation**: Templates validated against data source schema before saving
2. **Row Limit**: Reports exceeding row limit fail with partial results warning
3. **Timeout**: Reports exceeding max runtime are cancelled automatically
4. **Output Retention**: Generated outputs retained per schedule config (default 30 days)
5. **Bursting**: Burst reports split by specified field; each section delivered independently
6. **Concurrent Limits**: Max 5 concurrent report jobs per tenant; others queued
7. **Schedule Failure**: After 3 consecutive failures, schedule auto-paused with notification
8. **Multi-Language**: Template translations auto-selected based on report locale parameter

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Reporting (17) | Report catalog integration |
| General Ledger (06) | Financial statement data |
| Accounts Payable (07) | Invoice/check printing |
| Accounts Receivable (08) | Customer statements/dunning letters |
| Payroll (63) | Payslip and tax form generation |
| Order Management (13) | Order confirmation, packing slips |
| Document Management (29) | Template and output storage |
| Cash Management (10) | Bank correspondence |
| Tax Management (23) | Tax filing reports |
| Procurement (11) | PO and RFQ documents |
