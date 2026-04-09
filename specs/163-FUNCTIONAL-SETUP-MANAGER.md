# 163 - Functional Setup Manager Service Specification

## 1. Domain Overview

Functional Setup Manager provides implementation project management, configuration lifecycle management, and setup data migration tools for configuring Fusion applications. Supports structured implementation projects with task lists, configuration workbooks, setup data export/import, reference data management, and environment promotion. Enables implementation teams to plan, execute, and track configuration tasks, validate setup data, and migrate configurations between environments. Integrates with all application modules for configuration management.

**Bounded Context:** Implementation & Configuration Management
**Service Name:** `fsm-service`
**Database:** `data/fsm.db`
**HTTP Port:** 8181 | **gRPC Port:** 9181

---

## 2. Database Schema

### 2.1 Implementation Projects
```sql
CREATE TABLE fsm_impl_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_code TEXT NOT NULL,
    project_name TEXT NOT NULL,
    project_type TEXT NOT NULL CHECK(project_type IN ('NEW_IMPLEMENTATION','UPGRADE','MIGRATION','CONFIGURATION','EXTENSION')),
    description TEXT,
    methodology TEXT NOT NULL DEFAULT 'AGILE'
        CHECK(methodology IN ('AGILE','WATERFALL','HYBRID')),
    start_date TEXT NOT NULL,
    target_go_live TEXT NOT NULL,
    actual_go_live TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','IN_PROGRESS','TESTING','READY_FOR_GO_LIVE','COMPLETED','ON_HOLD','CANCELLED')),
    project_manager_id TEXT NOT NULL,
    team_members TEXT NOT NULL,             -- JSON array of user IDs
    total_tasks INTEGER NOT NULL DEFAULT 0,
    completed_tasks INTEGER NOT NULL DEFAULT 0,
    completion_pct REAL NOT NULL DEFAULT 0,
    current_phase TEXT,
    environment TEXT NOT NULL CHECK(environment IN ('DEV','TEST','STAGING','PRODUCTION')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_code)
);

CREATE INDEX idx_fsm_proj_tenant ON fsm_impl_projects(tenant_id, status);
CREATE INDEX idx_fsm_proj_manager ON fsm_impl_projects(project_manager_id);
```

### 2.2 Setup Tasks
```sql
CREATE TABLE fsm_setup_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    task_code TEXT NOT NULL,
    task_name TEXT NOT NULL,
    task_type TEXT NOT NULL CHECK(task_type IN (
        'CONFIGURATION','DATA_MIGRATION','CUSTOMIZATION','INTEGRATION_SETUP',
        'TESTING','TRAINING','DOCUMENTATION','APPROVAL','VALIDATION'
    )),
    description TEXT,
    module TEXT NOT NULL,                   -- Which application module
    setup_url TEXT,                         -- Direct link to setup page
    assigned_to TEXT NOT NULL,
    prerequisites TEXT,                     -- JSON array of task codes
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','CRITICAL')),
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','BLOCKED','SKIPPED')),
    due_date TEXT,
    completed_date TEXT,
    completion_notes TEXT,
    validation_status TEXT CHECK(validation_status IN ('PENDING','PASSED','FAILED')),
    validation_notes TEXT,
    estimated_hours REAL NOT NULL DEFAULT 0,
    actual_hours REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (project_id) REFERENCES fsm_impl_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, project_id, task_code)
);

CREATE INDEX idx_fsm_task_project ON fsm_setup_tasks(project_id, status);
CREATE INDEX idx_fsm_task_assigned ON fsm_setup_tasks(assigned_to, status);
CREATE INDEX idx_fsm_task_module ON fsm_setup_tasks(tenant_id, module);
```

### 2.3 Configuration Snapshots
```sql
CREATE TABLE fsm_config_snapshots (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    snapshot_name TEXT NOT NULL,
    source_environment TEXT NOT NULL,
    snapshot_type TEXT NOT NULL CHECK(snapshot_type IN ('FULL','MODULE','DELTA','PRE_MIGRATION','POST_MIGRATION')),
    modules TEXT NOT NULL,                  -- JSON: included modules
    config_data_path TEXT NOT NULL,         -- Path to exported config file
    file_size_bytes INTEGER NOT NULL,
    checksum TEXT NOT NULL,
    configuration_count INTEGER NOT NULL DEFAULT 0,
    created_by TEXT NOT NULL,
    notes TEXT,
    status TEXT NOT NULL DEFAULT 'COMPLETED'
        CHECK(status IN ('COMPLETED','MIGRATED','RESTORED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, snapshot_name)
);

CREATE INDEX idx_fsm_snap_tenant ON fsm_config_snapshots(tenant_id, source_environment);
```

### 2.4 Environment Promotions
```sql
CREATE TABLE fsm_promotions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    promotion_number TEXT NOT NULL,
    snapshot_id TEXT NOT NULL,
    source_environment TEXT NOT NULL,
    target_environment TEXT NOT NULL,
    promotion_type TEXT NOT NULL CHECK(promotion_type IN ('FULL','DELTA','SELECTIVE')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','IN_PROGRESS','COMPLETED','FAILED','ROLLED_BACK')),
    approved_by TEXT,
    approved_at TEXT,
    started_at TEXT,
    completed_at TEXT,
    validation_results TEXT,                -- JSON: validation output
    error_details TEXT,
    items_promoted INTEGER NOT NULL DEFAULT 0,
    items_skipped INTEGER NOT NULL DEFAULT 0,
    items_failed INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (snapshot_id) REFERENCES fsm_config_snapshots(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, promotion_number)
);

CREATE INDEX idx_fsm_promo_tenant ON fsm_promotions(tenant_id, status);
CREATE INDEX idx_fsm_promo_env ON fsm_promotions(source_environment, target_environment);
```

### 2.5 Reference Data Sets
```sql
CREATE TABLE fsm_reference_data_sets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    set_code TEXT NOT NULL,
    set_name TEXT NOT NULL,
    data_type TEXT NOT NULL CHECK(data_type IN (
        'LOOKUP_CODES','VALUE_SETS','FLEXFIELD_DEFINITIONS','CHART_OF_ACCOUNTS',
        'ORGANIZATION_HIERARCHY','BUSINESS_UNITS','TAX_CODES','PAYMENT_TERMS','CUSTOM'
    )),
    description TEXT,
    data_content TEXT NOT NULL,             -- JSON: the reference data
    applicable_modules TEXT NOT NULL,       -- JSON array
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, set_code)
);
```

---

## 3. API Endpoints

### 3.1 Implementation Projects
| Method | Path | Description |
|--------|------|-------------|
| POST | `/fsm/v1/projects` | Create implementation project |
| GET | `/fsm/v1/projects` | List projects |
| GET | `/fsm/v1/projects/{id}` | Get project details |
| PUT | `/fsm/v1/projects/{id}` | Update project |
| POST | `/fsm/v1/projects/{id}/activate` | Start implementation |
| GET | `/fsm/v1/projects/{id}/progress` | Get progress dashboard |

### 3.2 Setup Tasks
| Method | Path | Description |
|--------|------|-------------|
| POST | `/fsm/v1/projects/{id}/tasks` | Add setup task |
| GET | `/fsm/v1/projects/{id}/tasks` | List tasks |
| GET | `/fsm/v1/tasks/{id}` | Get task details |
| PUT | `/fsm/v1/tasks/{id}` | Update task |
| POST | `/fsm/v1/tasks/{id}/complete` | Mark task complete |
| POST | `/fsm/v1/tasks/{id}/validate` | Validate configuration |
| POST | `/fsm/v1/tasks/{id}/block` | Block task |
| POST | `/fsm/v1/tasks/{id}/unblock` | Unblock task |

### 3.3 Configuration Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/fsm/v1/snapshots` | Create configuration snapshot |
| GET | `/fsm/v1/snapshots` | List snapshots |
| GET | `/fsm/v1/snapshots/{id}` | Get snapshot details |
| POST | `/fsm/v1/snapshots/{id}/restore` | Restore snapshot |

### 3.4 Environment Promotion
| Method | Path | Description |
|--------|------|-------------|
| POST | `/fsm/v1/promotions` | Create promotion |
| GET | `/fsm/v1/promotions` | List promotions |
| POST | `/fsm/v1/promotions/{id}/approve` | Approve promotion |
| POST | `/fsm/v1/promotions/{id}/execute` | Execute promotion |
| POST | `/fsm/v1/promotions/{id}/rollback` | Rollback promotion |

### 3.5 Reference Data
| Method | Path | Description |
|--------|------|-------------|
| POST | `/fsm/v1/reference-data` | Create reference data set |
| GET | `/fsm/v1/reference-data` | List reference data |
| PUT | `/fsm/v1/reference-data/{id}` | Update reference data |
| POST | `/fsm/v1/reference-data/{id}/activate` | Activate reference data |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `fsm.project.created` | `{ project_id, type, go_live_date }` | Implementation project created |
| `fsm.task.completed` | `{ task_id, project_id, module }` | Setup task completed |
| `fsm.task.blocked` | `{ task_id, reason }` | Setup task blocked |
| `fsm.snapshot.created` | `{ snapshot_id, environment, configs }` | Configuration snapshot created |
| `fsm.promotion.completed` | `{ promotion_id, source, target }` | Environment promotion completed |
| `fsm.promotion.failed` | `{ promotion_id, errors }` | Promotion failed |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `deployment.completed` | Deployment | Update environment status |
| `deployment.failed` | Deployment | Block dependent setup tasks |

---

## 5. Business Rules

1. **Task Dependencies**: Tasks with prerequisites cannot start until prerequisites completed
2. **Configuration Validation**: All configurations validated before task marked complete
3. **Snapshot Before Promotion**: Snapshot required before any environment promotion
4. **Promotion Approval**: Promotions to PRODUCTION require explicit approval
5. **Rollback Support**: All promotions support rollback to pre-promotion snapshot
6. **Environment Isolation**: Configuration changes in one environment don't affect others
7. **Reference Data Sharing**: Reference data sets can be shared across modules
8. **Audit Trail**: All configuration changes tracked with who, when, and what

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Deployment (21) | Environment management and deployments |
| Auth & Security (05) | User role and permission setup |
| Multi-Tenancy (18) | Tenant configuration management |
| Data Import/Export (30) | Bulk data migration |
| Workflow (16) | Task approval workflows |
| All Modules | Configuration endpoints for each module |
