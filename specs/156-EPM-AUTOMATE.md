# 156 - EPM Automate Service Specification

## 1. Domain Overview

EPM Automate provides command-line interface (CLI) automation, scripting, and job scheduling for EPM cloud operations. Supports automated data loads, metadata imports/exports, application snapshots, business rule execution, report generation, user management, and integration batch processing. Enables administrators to create automation scripts for recurring EPM tasks, schedule batch operations, and monitor execution history. Integrates with all EPM modules, Smart View for Office, and external systems via REST API and file-based interfaces.

**Bounded Context:** EPM Automation & Batch Operations
**Service Name:** `epm-automate-service`
**Database:** `data/epm_automate.db`
**HTTP Port:** 8174 | **gRPC Port:** 9174

---

## 2. Database Schema

### 2.1 Automation Scripts
```sql
CREATE TABLE ea_scripts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    script_name TEXT NOT NULL,
    description TEXT,
    script_type TEXT NOT NULL CHECK(script_type IN ('GROOVY','PYTHON','SHELL','CUSTOM')),
    source_code TEXT NOT NULL,
    parameters TEXT,                        -- JSON: parameter definitions
    target_application TEXT,                -- EPM application name
    timeout_seconds INTEGER NOT NULL DEFAULT 3600,
    retry_count INTEGER NOT NULL DEFAULT 0,
    retry_delay_seconds INTEGER NOT NULL DEFAULT 60,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','DISABLED','ARCHIVED')),
    last_modified_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, script_name)
);

CREATE INDEX idx_ea_script_tenant ON ea_scripts(tenant_id, status);
```

### 2.2 Scheduled Jobs
```sql
CREATE TABLE ea_scheduled_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    job_name TEXT NOT NULL,
    script_id TEXT,
    cron_expression TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    parameters TEXT,                        -- JSON: runtime parameter values
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','CRITICAL')),
    max_concurrent INTEGER NOT NULL DEFAULT 1,
    active_from TEXT,
    active_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','COMPLETED','FAILED','CANCELLED')),
    last_run_at TEXT,
    last_run_status TEXT CHECK(last_run_status IN ('SUCCESS','FAILURE','TIMEOUT')),
    next_run_at TEXT,
    total_runs INTEGER NOT NULL DEFAULT 0,
    success_count INTEGER NOT NULL DEFAULT 0,
    failure_count INTEGER NOT NULL DEFAULT 0,
    notification_on_failure INTEGER NOT NULL DEFAULT 1,
    notification_recipients TEXT,           -- JSON array
    dependent_job_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (script_id) REFERENCES ea_scripts(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, job_name)
);

CREATE INDEX idx_ea_sched_tenant ON ea_scheduled_jobs(tenant_id, status);
CREATE INDEX idx_ea_sched_next ON ea_scheduled_jobs(next_run_at, status);
```

### 2.3 Job Executions
```sql
CREATE TABLE ea_job_executions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    job_id TEXT NOT NULL,
    execution_number TEXT NOT NULL,         -- EA-EXEC-2024-00001
    trigger_type TEXT NOT NULL CHECK(trigger_type IN ('SCHEDULED','MANUAL','EVENT','API','DEPENDENCY')),
    status TEXT NOT NULL DEFAULT 'QUEUED'
        CHECK(status IN ('QUEUED','RUNNING','COMPLETED','FAILED','TIMEOUT','CANCELLED','RETRYING')),
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    parameters TEXT,                        -- JSON: actual parameters used
    output_log TEXT,                        -- Execution log output
    error_log TEXT,
    records_processed INTEGER NOT NULL DEFAULT 0,
    records_succeeded INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    artifacts_generated TEXT,               -- JSON: files, reports generated
    retry_attempt INTEGER NOT NULL DEFAULT 0,
    triggered_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (job_id) REFERENCES ea_scheduled_jobs(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, execution_number)
);

CREATE INDEX idx_ea_exec_job ON ea_job_executions(job_id, started_at DESC);
CREATE INDEX idx_ea_exec_status ON ea_job_executions(tenant_id, status);
CREATE INDEX idx_ea_exec_tenant ON ea_job_executions(tenant_id, started_at DESC);
```

### 2.4 EPM Operations Catalog
```sql
CREATE TABLE ea_operations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    operation_code TEXT NOT NULL,
    operation_name TEXT NOT NULL,
    operation_type TEXT NOT NULL CHECK(operation_type IN (
        'DATA_LOAD','DATA_EXPORT','METADATA_IMPORT','METADATA_EXPORT',
        'APPLICATION_SNAPSHOT','APPLICATION_RESTORE','BUSINESS_RULE_EXECUTE',
        'REPORT_GENERATE','USER_MANAGE','DIMENSION_BUILD','CURRENCY_RATE_LOAD',
        'INTERCOMPANY_MATCH','CONSOLIDATION_RUN','ALLOCATION_RUN','CUSTOM'
    )),
    description TEXT,
    required_parameters TEXT NOT NULL,      -- JSON: required parameter definitions
    optional_parameters TEXT,               -- JSON: optional parameters
    target_application TEXT,
    estimated_duration_seconds INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, operation_code)
);
```

### 2.5 Automation Audit
```sql
CREATE TABLE ea_audit_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    execution_id TEXT NOT NULL,
    operation TEXT NOT NULL,
    action TEXT NOT NULL,
    target TEXT NOT NULL,                   -- What was operated on
    parameters TEXT,                        -- JSON
    result TEXT NOT NULL CHECK(result IN ('SUCCESS','FAILURE','WARNING')),
    message TEXT,
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),
    user_id TEXT NOT NULL,

    FOREIGN KEY (execution_id) REFERENCES ea_job_executions(id) ON DELETE CASCADE
);

CREATE INDEX idx_ea_audit_exec ON ea_audit_log(execution_id);
CREATE INDEX idx_ea_audit_tenant ON ea_audit_log(tenant_id, timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Scripts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/epm-automate/v1/scripts` | Create script |
| GET | `/epm-automate/v1/scripts` | List scripts |
| GET | `/epm-automate/v1/scripts/{id}` | Get script details |
| PUT | `/epm-automate/v1/scripts/{id}` | Update script |
| DELETE | `/epm-automate/v1/scripts/{id}` | Archive script |
| POST | `/epm-automate/v1/scripts/{id}/validate` | Validate script syntax |

### 3.2 Scheduled Jobs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/epm-automate/v1/jobs` | Create scheduled job |
| GET | `/epm-automate/v1/jobs` | List jobs |
| GET | `/epm-automate/v1/jobs/{id}` | Get job details |
| PUT | `/epm-automate/v1/jobs/{id}` | Update job |
| POST | `/epm-automate/v1/jobs/{id}/pause` | Pause job |
| POST | `/epm-automate/v1/jobs/{id}/resume` | Resume job |
| DELETE | `/epm-automate/v1/jobs/{id}` | Cancel job |

### 3.3 Execution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/epm-automate/v1/jobs/{id}/execute` | Execute job now |
| POST | `/epm-automate/v1/scripts/{id}/run` | Run script ad-hoc |
| GET | `/epm-automate/v1/executions` | List executions |
| GET | `/epm-automate/v1/executions/{id}` | Get execution details |
| GET | `/epm-automate/v1/executions/{id}/log` | Get execution log |
| POST | `/epm-automate/v1/executions/{id}/cancel` | Cancel execution |
| POST | `/epm-automate/v1/executions/{id}/retry` | Retry execution |

### 3.4 Operations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/epm-automate/v1/operations` | List available operations |
| GET | `/epm-automate/v1/operations/{code}` | Get operation details |
| POST | `/epm-automate/v1/operations/{code}/execute` | Execute operation directly |

### 3.5 Monitoring
| Method | Path | Description |
|--------|------|-------------|
| GET | `/epm-automate/v1/dashboard` | Automation dashboard |
| GET | `/epm-automate/v1/executions/{id}/audit` | Get audit trail |
| GET | `/epm-automate/v1/health` | Service health check |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `epmauto.job.started` | `{ execution_id, job_name, trigger_type }` | Job execution started |
| `epmauto.job.completed` | `{ execution_id, duration_ms, records }` | Job completed successfully |
| `epmauto.job.failed` | `{ execution_id, error, retry_count }` | Job failed |
| `epmauto.operation.completed` | `{ operation, target, result }` | Individual operation completed |
| `epmauto.schedule.paused` | `{ job_id, reason }` | Schedule paused |
| `epmauto.notification.sent` | `{ execution_id, type, recipients }` | Notification sent |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `epm.period.opened` | EPM | Trigger data load jobs |
| `consolidation.completed` | Financial Consolidation | Trigger post-consolidation jobs |
| `deployment.completed` | Deployment | Restart scheduled jobs |

---

## 5. Business Rules

1. **Script Validation**: Scripts validated for syntax and parameter references before saving
2. **Execution Queue**: Jobs queued and executed based on priority; max concurrent per tenant configurable
3. **Retry Logic**: Failed jobs retried up to configured retry count with exponential backoff
4. **Timeout**: Jobs exceeding timeout are killed and marked as TIMED_OUT
5. **Dependency Chains**: Jobs can define dependencies; execution triggered after dependency completion
6. **Audit Completeness**: Every operation within a job logged with parameters and results
7. **Notification**: Failure notifications sent immediately; success summaries sent per schedule
8. **Script Versioning**: Script changes create new version; running jobs use version at start time

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| EPM (33) | Planning data loads, business rule execution |
| Financial Consolidation (100) | Consolidation triggers |
| Account Reconciliation (101) | Reconciliation automation |
| Smart View for Office (155) | Coordinated data operations |
| Data Import/Export (30) | File-based data operations |
| Reporting (17) | Automated report generation |
| BI Publisher (150) | Report generation scheduling |
| Integration Cloud (89) | External system integration |
| Workflow (16) | Approval automation |
