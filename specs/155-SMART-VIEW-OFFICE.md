# 155 - Smart View for Office Service Specification

## 1. Domain Overview

Smart View for Office provides Microsoft Office integration (Excel, Word, PowerPoint, Outlook) for accessing, analyzing, and reporting on Fusion application data directly from desktop productivity tools. Enables Excel-based data entry, ad-hoc analysis, and refreshable data grids connected to GL, EPM, and other data sources. Supports Word-based report authoring with linked data fields, PowerPoint chart generation from live data, and Outlook integration for report distribution. Provides a secure connection layer between Office clients and Fusion services.

**Bounded Context:** Microsoft Office Integration
**Service Name:** `smartview-service`
**Database:** `data/smartview.db`
**HTTP Port:** 8173 | **gRPC Port:** 9173

---

## 2. Database Schema

### 2.1 Connection Definitions
```sql
CREATE TABLE sv_connections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    connection_name TEXT NOT NULL,
    connection_type TEXT NOT NULL CHECK(connection_type IN ('GL','EPM_PLANNING','EPM_CONSOLIDATION','CRM','SCM','CUSTOM')),
    data_source_url TEXT NOT NULL,
    authentication_method TEXT NOT NULL DEFAULT 'OAUTH2'
        CHECK(authentication_method IN ('OAUTH2','TOKEN','WINDOWS_SSO')),
    default_cube TEXT,                      -- For EPM connections
    default_dimensionality TEXT,            -- JSON: default POV settings
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, connection_name)
);
```

### 2.2 Saved Queries (Ad-Hoc Analysis)
```sql
CREATE TABLE sv_saved_queries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    query_name TEXT NOT NULL,
    connection_id TEXT NOT NULL,
    application TEXT NOT NULL CHECK(application IN ('EXCEL','WORD','POWERPOINT','OUTLOOK')),
    query_type TEXT NOT NULL CHECK(query_type IN ('ADHOC_GRID','DATA_ENTRY','REPORT_LINK','CHART','TABLE')),
    pov_settings TEXT NOT NULL,             -- JSON: Point of View dimension selections
    row_dimensions TEXT NOT NULL,           -- JSON: row dimension members
    column_dimensions TEXT NOT NULL,        -- JSON: column dimension members
    filters TEXT,                           -- JSON: additional filters
    formatting_rules TEXT,                  -- JSON: cell formatting
    refresh_on_open INTEGER NOT NULL DEFAULT 1,
    refresh_interval_minutes INTEGER,
    owner_id TEXT NOT NULL,
    is_shared INTEGER NOT NULL DEFAULT 0,
    shared_with TEXT,                       -- JSON array of user/group IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (connection_id) REFERENCES sv_connections(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, query_name, owner_id)
);

CREATE INDEX idx_sv_query_tenant ON sv_saved_queries(tenant_id, application);
CREATE INDEX idx_sv_query_owner ON sv_saved_queries(owner_id);
```

### 2.3 Office Document Links
```sql
CREATE TABLE sv_document_links (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_name TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN ('EXCEL','WORD','POWERPOINT')),
    storage_path TEXT NOT NULL,
    query_ids TEXT NOT NULL,                -- JSON: embedded query references
    connection_id TEXT NOT NULL,
    last_refreshed_at TEXT,
    auto_refresh INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    is_shared INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (connection_id) REFERENCES sv_connections(id) ON DELETE SET NULL
);

CREATE INDEX idx_sv_doc_tenant ON sv_document_links(tenant_id, owner_id);
CREATE INDEX idx_sv_doc_type ON sv_document_links(tenant_id, document_type);
```

### 2.4 Data Entry Submissions
```sql
CREATE TABLE sv_data_submissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    connection_id TEXT NOT NULL,
    submission_number TEXT NOT NULL,        -- SV-SUB-2024-00001
    source_document TEXT NOT NULL,
    source_worksheet TEXT,
    data_range TEXT NOT NULL,               -- e.g., "A1:D50"
    cell_count INTEGER NOT NULL DEFAULT 0,
    data_payload TEXT NOT NULL,             -- JSON: grid of submitted values
    submission_type TEXT NOT NULL CHECK(submission_type IN ('PLANNING_DATA','GL_JOURNAL','BUDGET_ENTRY','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','VALIDATING','VALIDATED','SUBMITTED','APPROVED','REJECTED','ERROR')),
    validation_errors TEXT,                 -- JSON: validation error details
    submitted_at TEXT,
    reviewed_at TEXT,
    reviewed_by TEXT,
    review_comments TEXT,
    target_system_ref TEXT,                 -- Reference to created record in target system

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, submission_number)
);

CREATE INDEX idx_sv_sub_tenant ON sv_data_submissions(tenant_id, status);
CREATE INDEX idx_sv_sub_user ON sv_data_submissions(created_by, created_at DESC);
```

### 2.5 Session Activity
```sql
CREATE TABLE sv_session_activity (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    application TEXT NOT NULL,
    action TEXT NOT NULL CHECK(action IN ('CONNECT','REFRESH','SUBMIT','QUERY','EXPORT','DISCONNECT')),
    connection_id TEXT,
    query_id TEXT,
    duration_ms INTEGER,
    rows_affected INTEGER,
    status TEXT NOT NULL CHECK(status IN ('SUCCESS','FAILURE')),

    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_sv_activity_user ON sv_session_activity(tenant_id, user_id, timestamp DESC);
CREATE INDEX idx_sv_activity_action ON sv_session_activity(tenant_id, action, timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Connections
| Method | Path | Description |
|--------|------|-------------|
| POST | `/smartview/v1/connections` | Create connection |
| GET | `/smartview/v1/connections` | List connections |
| GET | `/smartview/v1/connections/{id}` | Get connection details |
| PUT | `/smartview/v1/connections/{id}` | Update connection |
| POST | `/smartview/v1/connections/{id}/test` | Test connection |

### 3.2 Queries
| Method | Path | Description |
|--------|------|-------------|
| POST | `/smartview/v1/queries` | Save query |
| GET | `/smartview/v1/queries` | List queries |
| GET | `/smartview/v1/queries/{id}` | Get query definition |
| PUT | `/smartview/v1/queries/{id}` | Update query |
| DELETE | `/smartview/v1/queries/{id}` | Delete query |
| POST | `/smartview/v1/queries/{id}/execute` | Execute query and return data |

### 3.3 Data Entry
| Method | Path | Description |
|--------|------|-------------|
| POST | `/smartview/v1/submissions` | Submit data entry |
| GET | `/smartview/v1/submissions` | List submissions |
| GET | `/smartview/v1/submissions/{id}` | Get submission status |
| POST | `/smartview/v1/submissions/{id}/validate` | Validate submission |
| POST | `/smartview/v1/submissions/{id}/commit` | Commit to target system |

### 3.4 Document Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/smartview/v1/documents` | Register linked document |
| GET | `/smartview/v1/documents` | List documents |
| POST | `/smartview/v1/documents/{id}/refresh` | Refresh all linked queries |

### 3.5 Metadata
| Method | Path | Description |
|--------|------|-------------|
| GET | `/smartview/v1/connections/{id}/dimensions` | Get available dimensions |
| GET | `/smartview/v1/connections/{id}/members` | Get dimension members |
| GET | `/smartview/v1/connections/{id}/cubes` | Get available cubes |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sv.data.submitted` | `{ submission_id, user_id, target_system }` | Data submitted from Office |
| `sv.data.approved` | `{ submission_id, target_ref }` | Data entry approved |
| `sv.data.rejected` | `{ submission_id, reason }` | Data entry rejected |
| `sv.query.executed` | `{ query_id, user_id, rows }` | Query executed |
| `sv.connection.error` | `{ connection_id, error }` | Connection failure |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `gl.period.closed` | General Ledger | Invalidate cached GL data |
| `epm.scenario.approved` | EPM | Enable new scenario data for queries |
| `workflow.approval.completed` | Workflow | Update data submission status |

---

## 5. Business Rules

1. **Authentication**: All connections authenticated via OAuth2; tokens refreshed automatically
2. **Data Caching**: Query results cached client-side; refresh on demand or scheduled interval
3. **Cell-Level Security**: Data access respects GL/EPM data security by user/role
4. **Concurrent Editing**: Detect and warn on concurrent data entry conflicts
5. **Validation Before Submit**: All data entry validated against target system rules before submission
6. **Audit Trail**: All data submissions tracked with full before/after values
7. **Offline Support**: Excel grids can be edited offline; synced on reconnection
8. **Session Timeout**: Office sessions timeout after 8 hours of inactivity

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| General Ledger (06) | GL data queries and journal entry |
| EPM (33) | Planning data entry and queries |
| Financial Consolidation (100) | Consolidated data queries |
| Financial Reporting (154) | Report data access |
| Workflow (16) | Data entry approval workflows |
| Auth & Security (05) | Authentication and authorization |
| EPM Automate (156) | Batch operations coordination |
| Data Import/Export (30) | Data format compatibility |
