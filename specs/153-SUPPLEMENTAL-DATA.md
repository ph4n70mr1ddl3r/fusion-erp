# 153 - Supplemental Data Management Service Specification

## 1. Domain Overview

Supplemental Data Management enables collection, validation, and integration of non-financial and supplemental data from business users across the organization. Supports configurable data forms, multi-step data collection workflows, validation rules, and approval processes for data submitted by operational managers, business unit heads, and department leads. Used for budget inputs, forecast adjustments, operational metrics, headcount planning, and any data that supplements core financial data. Integrates with EPM for planning data, Workflow for approvals, and Reporting for consolidated views.

**Bounded Context:** Supplemental Data Collection & Validation
**Service Name:** `supplemental-data-service`
**Database:** `data/supplemental_data.db`
**HTTP Port:** 8171 | **gRPC Port:** 9171

---

## 2. Database Schema

### 2.1 Data Collection Templates
```sql
CREATE TABLE sd_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    template_type TEXT NOT NULL CHECK(template_type IN (
        'BUDGET_INPUT','FORECAST_ADJUSTMENT','HEADCOUNT_PLAN','CAPEX_REQUEST',
        'OPERATIONAL_METRIC','SURVEY','CUSTOM'
    )),
    period_type TEXT NOT NULL CHECK(period_type IN ('MONTHLY','QUARTERLY','ANNUAL','ADHOC')),
    columns TEXT NOT NULL,                  -- JSON: column definitions with types, validations
    rows_source TEXT NOT NULL CHECK(rows_source IN ('FIXED','DYNAMIC_QUERY','HIERARCHY','USER_DEFINED')),
    rows_config TEXT,                       -- JSON: fixed rows or query definition
    default_values TEXT,                    -- JSON: column default values
    validation_rules TEXT,                  -- JSON: cross-field validation rules
    approval_required INTEGER NOT NULL DEFAULT 1,
    allow_attachments INTEGER NOT NULL DEFAULT 1,
    submission_deadline TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_sd_template_tenant ON sd_templates(tenant_id, template_type, status);
```

### 2.2 Data Collection Tasks
```sql
CREATE TABLE sd_collection_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    task_name TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-Q1"
    assigned_to TEXT NOT NULL,              -- User or group ID
    assigned_org TEXT,                      -- Department/Business unit
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED','OVERDUE')),
    submitted_at TEXT,
    reviewed_at TEXT,
    reviewed_by TEXT,
    review_comments TEXT,
    due_date TEXT NOT NULL,
    completion_pct REAL NOT NULL DEFAULT 0,
    total_rows INTEGER NOT NULL DEFAULT 0,
    filled_rows INTEGER NOT NULL DEFAULT 0,
    validation_errors INTEGER NOT NULL DEFAULT 0,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES sd_templates(id) ON DELETE RESTRICT
);

CREATE INDEX idx_sd_task_tenant ON sd_collection_tasks(tenant_id, status);
CREATE INDEX idx_sd_task_assigned ON sd_collection_tasks(assigned_to, status);
CREATE INDEX idx_sd_task_due ON sd_collection_tasks(tenant_id, due_date, status);
```

### 2.3 Submitted Data
```sql
CREATE TABLE sd_submitted_data (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    row_key TEXT NOT NULL,                  -- Row identifier from template
    row_label TEXT NOT NULL,
    data_values TEXT NOT NULL,              -- JSON: { column_code: value }
    original_values TEXT,                   -- JSON: values before last edit
    validation_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(validation_status IN ('PENDING','VALID','INVALID','WARNING')),
    validation_errors TEXT,                 -- JSON: field-level error messages
    comments TEXT,
    attachments TEXT,                       -- JSON array of document IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_by TEXT NOT NULL,

    FOREIGN KEY (task_id) REFERENCES sd_collection_tasks(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, task_id, row_key)
);

CREATE INDEX idx_sd_data_task ON sd_submitted_data(task_id);
CREATE INDEX idx_sd_data_validation ON sd_submitted_data(tenant_id, validation_status);
```

### 2.4 Data Integration Mappings
```sql
CREATE TABLE sd_integration_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    mapping_name TEXT NOT NULL,
    target_system TEXT NOT NULL CHECK(target_system IN ('EPM_PLANNING','GL_BUDGET','WORKFORCE_PLAN','CUSTOM')),
    target_entity TEXT NOT NULL,            -- Target table or plan element
    column_mappings TEXT NOT NULL,          -- JSON: { source_column: target_field }
    row_mapping TEXT NOT NULL,              -- JSON: how rows map to target dimensions
    transformation_rules TEXT,              -- JSON: value transformations
    sync_mode TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(sync_mode IN ('AUTO_ON_APPROVAL','MANUAL','SCHEDULED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (template_id) REFERENCES sd_templates(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, template_id, mapping_name)
);
```

### 2.5 Collection History
```sql
CREATE TABLE sd_collection_history (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    action TEXT NOT NULL CHECK(action IN (
        'CREATED','OPENED','DATA_ENTERED','VALIDATED','SUBMITTED',
        'REVIEWED','APPROVED','REJECTED','RETURNED','DATA_SYNCED'
    )),
    user_id TEXT NOT NULL,
    user_name TEXT NOT NULL,
    details TEXT,                           -- JSON: action-specific details
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (task_id) REFERENCES sd_collection_tasks(id) ON DELETE CASCADE
);

CREATE INDEX idx_sd_history_task ON sd_collection_history(task_id, timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/supplemental-data/v1/templates` | Create template |
| GET | `/supplemental-data/v1/templates` | List templates |
| GET | `/supplemental-data/v1/templates/{id}` | Get template details |
| PUT | `/supplemental-data/v1/templates/{id}` | Update template |
| POST | `/supplemental-data/v1/templates/{id}/activate` | Activate template |

### 3.2 Collection Tasks
| Method | Path | Description |
|--------|------|-------------|
| POST | `/supplemental-data/v1/tasks` | Create collection task |
| POST | `/supplemental-data/v1/tasks/bulk-assign` | Bulk assign to users |
| GET | `/supplemental-data/v1/tasks` | List tasks |
| GET | `/supplemental-data/v1/tasks/{id}` | Get task with data |
| PUT | `/supplemental-data/v1/tasks/{id}/data` | Save data entry |
| POST | `/supplemental-data/v1/tasks/{id}/validate` | Validate submitted data |
| POST | `/supplemental-data/v1/tasks/{id}/submit` | Submit for approval |
| POST | `/supplemental-data/v1/tasks/{id}/approve` | Approve submission |
| POST | `/supplemental-data/v1/tasks/{id}/reject` | Reject and return |

### 3.3 Integration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/supplemental-data/v1/integration-mappings` | Create mapping |
| GET | `/supplemental-data/v1/integration-mappings` | List mappings |
| POST | `/supplemental-data/v1/tasks/{id}/sync` | Sync to target system |

### 3.4 Monitoring
| Method | Path | Description |
|--------|------|-------------|
| GET | `/supplemental-data/v1/dashboard` | Collection progress dashboard |
| GET | `/supplemental-data/v1/tasks/{id}/history` | Get task history |
| POST | `/supplemental-data/v1/tasks/send-reminders` | Send reminders for overdue |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sdata.task.assigned` | `{ task_id, assigned_to, due_date }` | Task assigned |
| `sdata.task.submitted` | `{ task_id, assigned_to }` | Data submitted |
| `sdata.task.approved` | `{ task_id, reviewer }` | Submission approved |
| `sdata.task.rejected` | `{ task_id, reason }` | Submission rejected |
| `sdata.task.overdue` | `{ task_id, due_date }` | Task overdue |
| `sdata.sync.completed` | `{ task_id, target_system }` | Data synced |
| `sdata.validation.failed` | `{ task_id, error_count }` | Validation errors |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `epm.period.opened` | EPM | Create collection tasks for period |
| `workflow.approval.completed` | Workflow | Update task approval status |

---

## 5. Business Rules

1. **Validation**: All data validated against template rules before submission
2. **Approval**: Approved data locked from editing; rejected data returned with comments
3. **Deadline Enforcement**: Overdue tasks auto-escalated to manager after 2 business days
4. **Auto-Sync**: Data auto-syncs to target system upon approval when configured
5. **Version Tracking**: Every data change tracked with user, timestamp, and previous value
6. **Row Locking**: Rows locked during edit to prevent concurrent modification conflicts
7. **Bulk Operations**: Support bulk data entry via spreadsheet upload (CSV/Excel)

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| EPM (33) | Planning data targets, period triggers |
| Workflow (16) | Approval workflows |
| Reporting (17) | Collection status dashboards |
| Notification Center (165) | Task assignment and reminder notifications |
| Data Import/Export (30) | Bulk data upload/download |
| Document Management (29) | Attachment storage |
| HCM Analytics (148) | Headcount and HR metric collection |
| ERP Analytics (149) | Operational metric collection |
