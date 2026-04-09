# 178 - Sandbox Management Service Specification

## 1. Domain Overview

Sandbox Management provides isolated test environment provisioning, configuration cloning, data masking, and environment lifecycle management for Fusion application development and testing. Supports sandbox creation from production snapshots, selective data masking, configuration-only vs. full-data sandboxes, environment refresh, change migration between environments, and sandbox auto-expiry. Enables implementation teams and developers to test changes in isolated environments without affecting production.

**Bounded Context:** Test Environment & Sandbox Management
**Service Name:** `sandbox-service`
**Database:** `data/sandbox.db`
**HTTP Port:** 8196 | **gRPC Port:** 9196

---

## 2. Database Schema

### 2.1 Sandbox Environments
```sql
CREATE TABLE sb_environments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    environment_name TEXT NOT NULL,
    environment_type TEXT NOT NULL CHECK(environment_type IN ('SANDBOX','DEV','TEST','STAGING','UAT','PRE_PRODUCTION')),
    source_environment_id TEXT,              -- What it was cloned from
    description TEXT,
    base_url TEXT NOT NULL,
    database_instance TEXT NOT NULL,
    config_snapshot_id TEXT,
    is_full_data INTEGER NOT NULL DEFAULT 0, -- Full data vs config-only
    data_masking_applied INTEGER NOT NULL DEFAULT 0,
    masking_profile_id TEXT,
    created_by_user TEXT NOT NULL,
    team TEXT,
    purpose TEXT,
    auto_expiry_date TEXT,
    status TEXT NOT NULL DEFAULT 'PROVISIONING'
        CHECK(status IN ('PROVISIONING','AVAILABLE','IN_USE','LOCKED','REFRESHING','EXPIRED','DECOMMISSIONED')),
    last_refreshed_at TEXT,
    total_refreshes INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, environment_name)
);

CREATE INDEX idx_sb_env_tenant ON sb_environments(tenant_id, status);
CREATE INDEX idx_sb_env_type ON sb_environments(environment_type);
CREATE INDEX idx_sb_env_expiry ON sb_environments(auto_expiry_date);
```

### 2.2 Refresh Operations
```sql
CREATE TABLE sb_refreshes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    target_environment_id TEXT NOT NULL,
    source_environment_id TEXT NOT NULL,
    refresh_type TEXT NOT NULL CHECK(refresh_type IN ('FULL','CONFIGURATION_ONLY','SELECTIVE_MODULES','DATA_ONLY')),
    modules TEXT,                            -- JSON: modules to refresh (for selective)
    data_masking INTEGER NOT NULL DEFAULT 1,
    masking_profile_id TEXT,
    requested_by TEXT NOT NULL,
    requested_at TEXT NOT NULL,
    started_at TEXT,
    completed_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','IN_PROGRESS','COMPLETED','FAILED','CANCELLED')),
    approval_required INTEGER NOT NULL DEFAULT 1,
    approved_by TEXT,
    approved_at TEXT,
    error_message TEXT,
    duration_seconds INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_sb_refresh_target ON sb_refreshes(target_environment_id, created_at DESC);
CREATE INDEX idx_sb_refresh_status ON sb_refreshes(tenant_id, status);
```

### 2.3 Data Masking Profiles
```sql
CREATE TABLE sb_masking_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_name TEXT NOT NULL,
    description TEXT,
    masking_rules TEXT NOT NULL,             -- JSON: [{ "table": "...", "column": "...", "method": "..." }]
    methods TEXT NOT NULL CHECK(methods IN ('HASH','SUBSTITUTE','RANDOMIZE','NULLIFY','FORMAT_PRESERVE','CUSTOM')),
    default_profile INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, profile_name)
);
```

### 2.4 Environment Migrations
```sql
CREATE TABLE sb_migrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    migration_number TEXT NOT NULL,
    source_environment_id TEXT NOT NULL,
    target_environment_id TEXT NOT NULL,
    migration_type TEXT NOT NULL CHECK(migration_type IN ('CONFIGURATION','CUSTOMIZATION','EXTENSION','DATA','FULL')),
    snapshot_id TEXT,
    requested_by TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','IN_PROGRESS','COMPLETED','FAILED','ROLLED_BACK')),
    approved_by TEXT,
    started_at TEXT,
    completed_at TEXT,
    items_migrated INTEGER NOT NULL DEFAULT 0,
    items_failed INTEGER NOT NULL DEFAULT 0,
    error_details TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, migration_number)
);

CREATE INDEX idx_sb_migration_status ON sb_migrations(tenant_id, status);
```

### 2.5 Environment Activity Log
```sql
CREATE TABLE sb_activity_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    environment_id TEXT NOT NULL,
    activity_type TEXT NOT NULL CHECK(activity_type IN ('PROVISIONED','REFRESHED','LOCKED','UNLOCKED','MIGRATED_TO','MIGRATED_FROM','EXPIRED','DECOMMISSIONED','ACCESSED')),
    user_id TEXT NOT NULL,
    details TEXT,

    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (environment_id) REFERENCES sb_environments(id) ON DELETE CASCADE
);

CREATE INDEX idx_sb_activity_env ON sb_activity_log(environment_id, timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Environments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sandbox/v1/environments` | Create environment |
| GET | `/sandbox/v1/environments` | List environments |
| GET | `/sandbox/v1/environments/{id}` | Get details |
| PUT | `/sandbox/v1/environments/{id}` | Update environment |
| DELETE | `/sandbox/v1/environments/{id}` | Decommission |
| POST | `/sandbox/v1/environments/{id}/lock` | Lock environment |
| POST | `/sandbox/v1/environments/{id}/unlock` | Unlock environment |

### 3.2 Refresh
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sandbox/v1/refreshes` | Request refresh |
| GET | `/sandbox/v1/refreshes` | List refreshes |
| GET | `/sandbox/v1/refreshes/{id}` | Get refresh status |
| POST | `/sandbox/v1/refreshes/{id}/approve` | Approve refresh |
| POST | `/sandbox/v1/refreshes/{id}/cancel` | Cancel refresh |

### 3.3 Masking Profiles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sandbox/v1/masking-profiles` | Create profile |
| GET | `/sandbox/v1/masking-profiles` | List profiles |
| PUT | `/sandbox/v1/masking-profiles/{id}` | Update profile |

### 3.4 Migrations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sandbox/v1/migrations` | Request migration |
| GET | `/sandbox/v1/migrations` | List migrations |
| POST | `/sandbox/v1/migrations/{id}/approve` | Approve migration |
| POST | `/sandbox/v1/migrations/{id}/rollback` | Rollback migration |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sandbox.provisioned` | `{ env_id, type, source }` | Environment provisioned |
| `sandbox.refresh.completed` | `{ refresh_id, env_id, duration }` | Refresh completed |
| `sandbox.refresh.failed` | `{ refresh_id, error }` | Refresh failed |
| `sandbox.migration.completed` | `{ migration_id, source, target }` | Migration completed |
| `sandbox.expiring` | `{ env_id, expiry_date }` | Environment expiring soon |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `deployment.completed` | Deployment | Update environment status |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.sandbox.v1;

service SandboxManagementService {
    rpc GetEnvironment(GetEnvironmentRequest) returns (GetEnvironmentResponse);
    rpc CreateEnvironment(CreateEnvironmentRequest) returns (CreateEnvironmentResponse);
    rpc RequestRefresh(RequestRefreshRequest) returns (RequestRefreshResponse);
    rpc GetRefresh(GetRefreshRequest) returns (GetRefreshResponse);
    rpc RequestMigration(RequestMigrationRequest) returns (RequestMigrationResponse);
}

// Environment messages
message GetEnvironmentRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetEnvironmentResponse {
    SbEnvironment data = 1;
}

message CreateEnvironmentRequest {
    string tenant_id = 1;
    string environment_name = 2;
    string environment_type = 3;
    string source_environment_id = 4;
    string description = 5;
    string base_url = 6;
    string database_instance = 7;
    bool is_full_data = 8;
    bool data_masking_applied = 9;
    string masking_profile_id = 10;
    string created_by_user = 11;
    string team = 12;
    string purpose = 13;
    string auto_expiry_date = 14;
}

message CreateEnvironmentResponse {
    SbEnvironment data = 1;
}

message SbEnvironment {
    string id = 1;
    string tenant_id = 2;
    string environment_name = 3;
    string environment_type = 4;
    string source_environment_id = 5;
    string description = 6;
    string base_url = 7;
    string database_instance = 8;
    string config_snapshot_id = 9;
    bool is_full_data = 10;
    bool data_masking_applied = 11;
    string masking_profile_id = 12;
    string created_by_user = 13;
    string team = 14;
    string purpose = 15;
    string auto_expiry_date = 16;
    string status = 17;
    string last_refreshed_at = 18;
    int32 total_refreshes = 19;
    string created_at = 20;
    string updated_at = 21;
}

// Refresh messages
message RequestRefreshRequest {
    string tenant_id = 1;
    string target_environment_id = 2;
    string source_environment_id = 3;
    string refresh_type = 4;
    string modules = 5;
    bool data_masking = 6;
    string masking_profile_id = 7;
    string requested_by = 8;
}

message RequestRefreshResponse {
    SbRefresh data = 1;
}

message GetRefreshRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetRefreshResponse {
    SbRefresh data = 1;
}

message SbRefresh {
    string id = 1;
    string tenant_id = 2;
    string target_environment_id = 3;
    string source_environment_id = 4;
    string refresh_type = 5;
    string modules = 6;
    bool data_masking = 7;
    string masking_profile_id = 8;
    string requested_by = 9;
    string requested_at = 10;
    string started_at = 11;
    string completed_at = 12;
    string status = 13;
    bool approval_required = 14;
    string approved_by = 15;
    string approved_at = 16;
    string error_message = 17;
    int32 duration_seconds = 18;
    string created_at = 19;
}

// Migration messages
message RequestMigrationRequest {
    string tenant_id = 1;
    string source_environment_id = 2;
    string target_environment_id = 3;
    string migration_type = 4;
    string requested_by = 5;
}

message RequestMigrationResponse {
    SbMigration data = 1;
}

message SbMigration {
    string id = 1;
    string tenant_id = 2;
    string migration_number = 3;
    string source_environment_id = 4;
    string target_environment_id = 5;
    string migration_type = 6;
    string snapshot_id = 7;
    string requested_by = 8;
    string status = 9;
    string approved_by = 10;
    string started_at = 11;
    string completed_at = 12;
    int32 items_migrated = 13;
    int32 items_failed = 14;
    string error_details = 15;
    string created_at = 16;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | sb_environments | -- |
| V002 | sb_refreshes | V001 |
| V003 | sb_masking_profiles | -- |
| V004 | sb_migrations | V001 |
| V005 | sb_activity_log | V001 |

---

## 7. Business Rules

1. **Auto-Expiry**: Sandboxes auto-expire after configured period (default 30 days)
2. **Data Masking**: Production-to-sandbox refreshes require data masking by default
3. **Refresh Approval**: Refreshes from production to available environments require approval
4. **Migration Lock**: Target environments locked during migration
5. **Environment Limits**: Max concurrent sandboxes per tenant configurable
6. **Activity Logging**: All environment activities logged for audit compliance

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| deployment-service | `DeployEnvironment` | CI/CD pipeline integration |
| setup-manager-service | `MigrateConfig` | Configuration migration |
| data-import-service | `ExportData` / `ImportData` | Data movement |
| auth-service | `CheckAccess` | Environment access control |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| multi-tenancy-service | `CreateEnvironment` | Tenant data isolation |
| access-governor-service | `GetEnvironment` | Access policy enforcement |
