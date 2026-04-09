# 205 - Siebel Migration Service Specification

## 1. Domain Overview

Siebel Migration Service provides comprehensive data migration tooling for transitioning from Siebel CRM to Oracle Fusion CX. It supports migration project management with phased execution across Siebel objects (accounts, contacts, opportunities, activities, service requests, quotes, orders), object-level mapping configuration between Siebel entities and Fusion CX equivalents with field-level data type transformation and LOV translation, data staging with raw data loading, validation, cleansing, and error handling workflows, batch migration execution with success/failure tracking and detailed error summaries, and delta sync configuration for ongoing bidirectional data synchronization between Siebel and Fusion during coexistence periods with conflict resolution strategies. Enables parallel-run validation with rollback capabilities and full audit trails for enterprise CRM migration programs. Integrates with Multi-Tenancy for tenant provisioning, Sales Automation for account and opportunity loading, Service Center for service request migration, Customer Data Platform for unified customer profiles, and Data Import/Export for bulk data operations.

**Bounded Context:** Siebel CRM-to-Oracle Fusion CX Data Migration
**Service Name:** `siebel-migration-service`
**Database:** `data/siebel_migration.db`
**HTTP Port:** 8223 | **gRPC Port:** 9223

---

## 2. Database Schema

### 2.1 Migration Projects
```sql
CREATE TABLE sm_migration_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_code TEXT NOT NULL,
    project_name TEXT NOT NULL,
    source_siebel_version TEXT NOT NULL,
    source_instance_url TEXT NOT NULL,
    target_fusion_modules TEXT NOT NULL,               -- JSON: array of target Fusion CX modules
    scope TEXT NOT NULL,                               -- JSON: migration scope definition
    phases TEXT NOT NULL,                              -- JSON: [{phase, status, start, end}]
    current_phase TEXT NOT NULL,
    total_objects INTEGER NOT NULL DEFAULT 0,
    migrated_objects INTEGER NOT NULL DEFAULT 0,
    failed_objects INTEGER NOT NULL DEFAULT 0,
    started_at TEXT,
    target_completion TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNING'
        CHECK(status IN ('PLANNING','IN_PROGRESS','COMPLETED','ON_HOLD','FAILED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_code)
);

CREATE INDEX idx_sm_projects_tenant_status ON sm_migration_projects(tenant_id, status);
CREATE INDEX idx_sm_projects_tenant_phase ON sm_migration_projects(tenant_id, current_phase);
CREATE INDEX idx_sm_projects_tenant_completion ON sm_migration_projects(tenant_id, target_completion);
CREATE INDEX idx_sm_projects_tenant_active ON sm_migration_projects(tenant_id, is_active);
```

### 2.2 Object Mappings
```sql
CREATE TABLE sm_object_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    siebel_object TEXT NOT NULL
        CHECK(siebel_object IN ('ACCOUNT','CONTACT','OPPORTUNITY','ACTIVITY','SERVICE_REQUEST','QUOTE','ORDER')),
    fusion_entity TEXT NOT NULL,
    field_mappings TEXT NOT NULL,                      -- JSON: [{siebel_field, fusion_field, data_type, transformation}]
    lov_translations TEXT,                             -- JSON: Siebel LOV to Fusion LOV mappings
    default_values TEXT,                               -- JSON: default values for unmapped Fusion fields
    estimated_records INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','ACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES sm_migration_projects(id),
    UNIQUE(tenant_id, project_id, siebel_object)
);

CREATE INDEX idx_sm_mappings_tenant_project ON sm_object_mappings(tenant_id, project_id);
CREATE INDEX idx_sm_mappings_tenant_object ON sm_object_mappings(tenant_id, siebel_object);
CREATE INDEX idx_sm_mappings_tenant_status ON sm_object_mappings(tenant_id, status);
CREATE INDEX idx_sm_mappings_tenant_active ON sm_object_mappings(tenant_id, is_active);
```

### 2.3 Data Staging
```sql
CREATE TABLE sm_data_staging (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    siebel_object TEXT NOT NULL
        CHECK(siebel_object IN ('ACCOUNT','CONTACT','OPPORTUNITY','ACTIVITY','SERVICE_REQUEST','QUOTE','ORDER')),
    batch_id TEXT NOT NULL,
    raw_data TEXT NOT NULL,                            -- JSON: raw Siebel record data
    staging_status TEXT NOT NULL DEFAULT 'LOADED'
        CHECK(staging_status IN ('LOADED','VALIDATED','CLEANSED','ERROR')),
    cleansing_rules_applied TEXT,                     -- JSON: array of cleansing rules executed
    validation_errors TEXT,                            -- JSON: array of validation error details
    record_count INTEGER NOT NULL DEFAULT 0,
    loaded_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES sm_migration_projects(id)
);

CREATE INDEX idx_sm_staging_tenant_project ON sm_data_staging(tenant_id, project_id);
CREATE INDEX idx_sm_staging_tenant_object ON sm_data_staging(tenant_id, siebel_object);
CREATE INDEX idx_sm_staging_tenant_batch ON sm_data_staging(tenant_id, project_id, batch_id);
CREATE INDEX idx_sm_staging_tenant_status ON sm_data_staging(tenant_id, staging_status);
CREATE INDEX idx_sm_staging_tenant_loaded ON sm_data_staging(tenant_id, loaded_at);
CREATE INDEX idx_sm_staging_tenant_active ON sm_data_staging(tenant_id, is_active);
```

### 2.4 Migration Runs
```sql
CREATE TABLE sm_migration_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    run_number INTEGER NOT NULL,
    siebel_object TEXT NOT NULL,
    batch_id TEXT NOT NULL,
    records_attempted INTEGER NOT NULL DEFAULT 0,
    records_succeeded INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    error_summary TEXT,                                -- JSON: [{error_type, count, sample_errors}]
    started_at TEXT NOT NULL,
    completed_at TEXT,
    duration_ms INTEGER,
    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED','PARTIAL')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES sm_migration_projects(id),
    UNIQUE(tenant_id, project_id, run_number)
);

CREATE INDEX idx_sm_runs_tenant_project ON sm_migration_runs(tenant_id, project_id);
CREATE INDEX idx_sm_runs_tenant_object ON sm_migration_runs(tenant_id, siebel_object);
CREATE INDEX idx_sm_runs_tenant_status ON sm_migration_runs(tenant_id, status);
CREATE INDEX idx_sm_runs_tenant_batch ON sm_migration_runs(tenant_id, batch_id);
CREATE INDEX idx_sm_runs_tenant_active ON sm_migration_runs(tenant_id, is_active);
```

### 2.5 Delta Sync Configurations
```sql
CREATE TABLE sm_delta_sync_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    siebel_object TEXT NOT NULL
        CHECK(siebel_object IN ('ACCOUNT','CONTACT','OPPORTUNITY','ACTIVITY','SERVICE_REQUEST','QUOTE','ORDER')),
    sync_direction TEXT NOT NULL
        CHECK(sync_direction IN ('SIEBEL_TO_FUSION','FUSION_TO_SIEBEL','BIDIRECTIONAL')),
    sync_frequency TEXT NOT NULL
        CHECK(sync_frequency IN ('REALTIME','HOURLY','DAILY')),
    conflict_resolution TEXT NOT NULL DEFAULT 'SOURCE_WINS'
        CHECK(conflict_resolution IN ('SOURCE_WINS','TARGET_WINS','MANUAL')),
    last_sync_at TEXT,
    last_sync_records INTEGER NOT NULL DEFAULT 0,
    last_sync_conflicts INTEGER NOT NULL DEFAULT 0,
    active_flag INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','ERROR')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES sm_migration_projects(id),
    UNIQUE(tenant_id, project_id, siebel_object)
);

CREATE INDEX idx_sm_delta_tenant_project ON sm_delta_sync_configs(tenant_id, project_id);
CREATE INDEX idx_sm_delta_tenant_object ON sm_delta_sync_configs(tenant_id, siebel_object);
CREATE INDEX idx_sm_delta_tenant_direction ON sm_delta_sync_configs(tenant_id, sync_direction);
CREATE INDEX idx_sm_delta_tenant_frequency ON sm_delta_sync_configs(tenant_id, sync_frequency);
CREATE INDEX idx_sm_delta_tenant_status ON sm_delta_sync_configs(tenant_id, status);
CREATE INDEX idx_sm_delta_tenant_active ON sm_delta_sync_configs(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Migration Projects
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/siebel-migration/projects` | List migration projects |
| POST | `/api/v1/siebel-migration/projects` | Create migration project |
| GET | `/api/v1/siebel-migration/projects/{id}` | Get project details |
| PUT | `/api/v1/siebel-migration/projects/{id}` | Update project |
| POST | `/api/v1/siebel-migration/projects/{id}/start` | Start migration |
| POST | `/api/v1/siebel-migration/projects/{id}/hold` | Put migration on hold |
| GET | `/api/v1/siebel-migration/projects/{id}/progress` | Get migration progress |

### 3.2 Object Mappings
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/siebel-migration/projects/{projectId}/mappings` | List object mappings |
| POST | `/api/v1/siebel-migration/projects/{projectId}/mappings` | Create object mapping |
| GET | `/api/v1/siebel-migration/projects/{projectId}/mappings/{id}` | Get mapping details |
| PUT | `/api/v1/siebel-migration/projects/{projectId}/mappings/{id}` | Update mapping |
| POST | `/api/v1/siebel-migration/projects/{projectId}/mappings/{id}/approve` | Approve mapping |
| POST | `/api/v1/siebel-migration/projects/{projectId}/mappings/auto-detect` | Auto-detect field mappings |
| POST | `/api/v1/siebel-migration/projects/{projectId}/mappings/validate-lov` | Validate LOV translations |

### 3.3 Data Staging
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/siebel-migration/projects/{projectId}/staging` | List staged data |
| POST | `/api/v1/siebel-migration/projects/{projectId}/staging/load` | Load data into staging |
| GET | `/api/v1/siebel-migration/projects/{projectId}/staging/{id}` | Get staging record |
| POST | `/api/v1/siebel-migration/projects/{projectId}/staging/{id}/validate` | Validate staged data |
| POST | `/api/v1/siebel-migration/projects/{projectId}/staging/{id}/cleanse` | Cleanse staged data |
| GET | `/api/v1/siebel-migration/projects/{projectId}/staging/errors` | List staging errors |
| POST | `/api/v1/siebel-migration/projects/{projectId}/staging/bulk-validate` | Bulk validate all staged data |

### 3.4 Migration Runs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/siebel-migration/projects/{projectId}/runs` | List migration runs |
| POST | `/api/v1/siebel-migration/projects/{projectId}/runs/execute` | Execute migration run |
| GET | `/api/v1/siebel-migration/projects/{projectId}/runs/{id}` | Get run details |
| GET | `/api/v1/siebel-migration/projects/{projectId}/runs/{id}/errors` | Get run error summary |
| POST | `/api/v1/siebel-migration/projects/{projectId}/runs/retry-failed` | Retry failed records |

### 3.5 Delta Sync
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/siebel-migration/projects/{projectId}/delta-sync` | List delta sync configs |
| POST | `/api/v1/siebel-migration/projects/{projectId}/delta-sync` | Create delta sync config |
| GET | `/api/v1/siebel-migration/projects/{projectId}/delta-sync/{id}` | Get delta sync config |
| PUT | `/api/v1/siebel-migration/projects/{projectId}/delta-sync/{id}` | Update delta sync config |
| POST | `/api/v1/siebel-migration/projects/{projectId}/delta-sync/{id}/pause` | Pause delta sync |
| POST | `/api/v1/siebel-migration/projects/{projectId}/delta-sync/{id}/resume` | Resume delta sync |
| POST | `/api/v1/siebel-migration/projects/{projectId}/delta-sync/{id}/trigger` | Trigger manual sync |
| GET | `/api/v1/siebel-migration/projects/{projectId}/delta-sync/conflicts` | List unresolved conflicts |

---

## 4. Business Rules

### 4.1 Migration Project Management
1. project_code MUST be unique within a tenant.
2. Project status transitions MUST follow: PLANNING -> IN_PROGRESS -> COMPLETED; ON_HOLD and FAILED are valid intermediate states.
3. current_phase MUST align with one of the phases defined in the phases JSON.
4. migrated_objects MUST NOT exceed total_objects; failed_objects MUST NOT exceed total_objects.
5. A project in COMPLETED or FAILED status MUST NOT allow new object mappings or migration runs.

### 4.2 Object Mapping Configuration
6. The combination of (tenant_id, project_id, siebel_object) MUST be unique.
7. siebel_object MUST be one of: ACCOUNT, CONTACT, OPPORTUNITY, ACTIVITY, SERVICE_REQUEST, QUOTE, ORDER.
8. field_mappings JSON MUST contain at least one field mapping with siebel_field, fusion_field, data_type, and transformation.
9. Mapping status transitions MUST follow: DRAFT -> APPROVED -> ACTIVE; only ACTIVE mappings can be used in migration runs.
10. lov_translations JSON SHOULD cover all Siebel LOV values that have no direct Fusion equivalent.

### 4.3 Data Staging
11. staging_status transitions MUST follow: LOADED -> VALIDATED -> CLEANSED; ERROR is a terminal state requiring correction.
12. raw_data JSON MUST be preserved in original form regardless of staging_status changes.
13. cleansing_rules_applied JSON MUST document every transformation applied during cleansing.
14. Records in ERROR staging_status MUST have validation_errors JSON populated with failure details.
15. Only VALIDATED or CLEANSED staging records can be included in migration runs.

### 4.4 Migration Run Execution
16. The combination of (tenant_id, project_id, run_number) MUST be unique.
17. Run status MUST be RUNNING during execution; final states are COMPLETED, FAILED, or PARTIAL.
18. PARTIAL status indicates records_succeeded > 0 and records_failed > 0.
19. error_summary JSON MUST aggregate errors by type with count and sample error messages.
20. records_attempted MUST equal records_succeeded plus records_failed for COMPLETED or PARTIAL runs.

### 4.5 Delta Synchronization
21. The combination of (tenant_id, project_id, siebel_object) MUST be unique for delta sync configs.
22. BIDIRECTIONAL sync_direction with REALTIME frequency MUST use conflict_resolution of MANUAL.
23. last_sync_conflicts MUST be incremented when conflict_resolution is MANUAL and a conflict is detected.
24. Delta sync configs with status ERROR MUST have their active_flag set to 0 automatically.
25. A delta sync config MUST NOT be created for an object that is still in the initial migration pipeline.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.siebelmigration.v1;

service SiebelMigrationService {
    // Migration projects
    rpc CreateMigrationProject(CreateMigrationProjectRequest) returns (CreateMigrationProjectResponse);
    rpc GetMigrationProject(GetMigrationProjectRequest) returns (GetMigrationProjectResponse);
    rpc ListMigrationProjects(ListMigrationProjectsRequest) returns (ListMigrationProjectsResponse);
    rpc StartMigration(StartMigrationRequest) returns (StartMigrationResponse);
    rpc GetMigrationProgress(GetMigrationProgressRequest) returns (GetMigrationProgressResponse);

    // Object mappings
    rpc CreateObjectMapping(CreateObjectMappingRequest) returns (CreateObjectMappingResponse);
    rpc GetObjectMapping(GetObjectMappingRequest) returns (GetObjectMappingResponse);
    rpc ListObjectMappings(ListObjectMappingsRequest) returns (ListObjectMappingsResponse);
    rpc ApproveObjectMapping(ApproveObjectMappingRequest) returns (ApproveObjectMappingResponse);

    // Data staging
    rpc LoadStagingData(LoadStagingDataRequest) returns (LoadStagingDataResponse);
    rpc GetStagingRecord(GetStagingRecordRequest) returns (GetStagingRecordResponse);
    rpc ListStagingRecords(ListStagingRecordsRequest) returns (ListStagingRecordsResponse);
    rpc ValidateStagingData(ValidateStagingDataRequest) returns (ValidateStagingDataResponse);
    rpc CleanseStagingData(CleanseStagingDataRequest) returns (CleanseStagingDataResponse);

    // Migration runs
    rpc ExecuteMigrationRun(ExecuteMigrationRunRequest) returns (ExecuteMigrationRunResponse);
    rpc GetMigrationRun(GetMigrationRunRequest) returns (GetMigrationRunResponse);
    rpc ListMigrationRuns(ListMigrationRunsRequest) returns (ListMigrationRunsResponse);
    rpc RetryFailedRecords(RetryFailedRecordsRequest) returns (RetryFailedRecordsResponse);

    // Delta sync
    rpc CreateDeltaSyncConfig(CreateDeltaSyncConfigRequest) returns (CreateDeltaSyncConfigResponse);
    rpc GetDeltaSyncConfig(GetDeltaSyncConfigRequest) returns (GetDeltaSyncConfigResponse);
    rpc ListDeltaSyncConfigs(ListDeltaSyncConfigsRequest) returns (ListDeltaSyncConfigsResponse);
    rpc TriggerDeltaSync(TriggerDeltaSyncRequest) returns (TriggerDeltaSyncResponse);
    rpc ResolveConflict(ResolveConflictRequest) returns (ResolveConflictResponse);
}

message CreateMigrationProjectRequest {
    string tenant_id = 1;
    string project_code = 2;
    string project_name = 3;
    string source_siebel_version = 4;
    string source_instance_url = 5;
    string target_fusion_modules = 6;
    string scope = 7;
    string phases = 8;
    string target_completion = 9;
}

message CreateMigrationProjectResponse {
    string project_id = 1;
    string project_code = 2;
    string project_name = 3;
    string status = 4;
}

message GetMigrationProjectRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message GetMigrationProjectResponse {
    string project_id = 1;
    string project_code = 2;
    string project_name = 3;
    string source_siebel_version = 4;
    string current_phase = 5;
    int32 total_objects = 6;
    int32 migrated_objects = 7;
    int32 failed_objects = 8;
    string status = 9;
}

message ListMigrationProjectsRequest {
    string tenant_id = 1;
    string status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListMigrationProjectsResponse {
    repeated GetMigrationProjectResponse projects = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message StartMigrationRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message StartMigrationResponse {
    string project_id = 1;
    string current_phase = 2;
    string status = 3;
}

message GetMigrationProgressRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message GetMigrationProgressResponse {
    string project_id = 1;
    int32 total_objects = 2;
    int32 migrated_objects = 3;
    int32 failed_objects = 4;
    double completion_pct = 5;
    string current_phase = 6;
    string estimated_completion = 7;
}

message CreateObjectMappingRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string fusion_entity = 4;
    string field_mappings = 5;
    string lov_translations = 6;
    string default_values = 7;
    int32 estimated_records = 8;
}

message CreateObjectMappingResponse {
    string mapping_id = 1;
    string siebel_object = 2;
    string fusion_entity = 3;
    string status = 4;
}

message GetObjectMappingRequest {
    string tenant_id = 1;
    string mapping_id = 2;
}

message GetObjectMappingResponse {
    string mapping_id = 1;
    string siebel_object = 2;
    string fusion_entity = 3;
    string field_mappings = 4;
    int32 estimated_records = 5;
    string status = 6;
}

message ListObjectMappingsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListObjectMappingsResponse {
    repeated GetObjectMappingResponse mappings = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ApproveObjectMappingRequest {
    string tenant_id = 1;
    string mapping_id = 2;
}

message ApproveObjectMappingResponse {
    string mapping_id = 1;
    string status = 2;
}

message LoadStagingDataRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string batch_id = 4;
    string raw_data = 5;
}

message LoadStagingDataResponse {
    string staging_id = 1;
    string batch_id = 2;
    int32 record_count = 3;
    string staging_status = 4;
}

message GetStagingRecordRequest {
    string tenant_id = 1;
    string staging_id = 2;
}

message GetStagingRecordResponse {
    string staging_id = 1;
    string siebel_object = 2;
    string batch_id = 3;
    string staging_status = 4;
    int32 record_count = 5;
    string validation_errors = 6;
}

message ListStagingRecordsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string staging_status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListStagingRecordsResponse {
    repeated GetStagingRecordResponse records = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ValidateStagingDataRequest {
    string tenant_id = 1;
    string staging_id = 2;
}

message ValidateStagingDataResponse {
    string staging_id = 1;
    string staging_status = 2;
    int32 error_count = 3;
    string validation_errors = 4;
}

message CleanseStagingDataRequest {
    string tenant_id = 1;
    string staging_id = 2;
    string cleansing_rules = 3;
}

message CleanseStagingDataResponse {
    string staging_id = 1;
    string staging_status = 2;
    string cleansing_rules_applied = 3;
    int32 records_cleansed = 4;
}

message ExecuteMigrationRunRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string batch_id = 4;
}

message ExecuteMigrationRunResponse {
    string run_id = 1;
    int32 run_number = 2;
    int32 records_attempted = 3;
    int32 records_succeeded = 4;
    int32 records_failed = 5;
    string status = 6;
}

message GetMigrationRunRequest {
    string tenant_id = 1;
    string run_id = 2;
}

message GetMigrationRunResponse {
    string run_id = 1;
    int32 run_number = 2;
    string siebel_object = 3;
    int32 records_attempted = 4;
    int32 records_succeeded = 5;
    int32 records_failed = 6;
    string error_summary = 7;
    string started_at = 8;
    string completed_at = 9;
    string status = 10;
}

message ListMigrationRunsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListMigrationRunsResponse {
    repeated GetMigrationRunResponse runs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RetryFailedRecordsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string original_run_id = 3;
}

message RetryFailedRecordsResponse {
    string new_run_id = 1;
    int32 run_number = 2;
    int32 records_retried = 3;
    string status = 4;
}

message CreateDeltaSyncConfigRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string sync_direction = 4;
    string sync_frequency = 5;
    string conflict_resolution = 6;
}

message CreateDeltaSyncConfigResponse {
    string config_id = 1;
    string siebel_object = 2;
    string sync_direction = 3;
    string status = 4;
}

message GetDeltaSyncConfigRequest {
    string tenant_id = 1;
    string config_id = 2;
}

message GetDeltaSyncConfigResponse {
    string config_id = 1;
    string siebel_object = 2;
    string sync_direction = 3;
    string sync_frequency = 4;
    string conflict_resolution = 5;
    string last_sync_at = 6;
    int32 last_sync_records = 7;
    int32 last_sync_conflicts = 8;
    string status = 9;
}

message ListDeltaSyncConfigsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string siebel_object = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListDeltaSyncConfigsResponse {
    repeated GetDeltaSyncConfigResponse configs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TriggerDeltaSyncRequest {
    string tenant_id = 1;
    string config_id = 2;
}

message TriggerDeltaSyncResponse {
    string config_id = 1;
    int32 records_synced = 2;
    int32 conflicts = 3;
    string sync_completed_at = 4;
}

message ResolveConflictRequest {
    string tenant_id = 1;
    string config_id = 2;
    string conflict_id = 3;
    string resolution = 4;   // SOURCE or TARGET
    string resolved_value = 5;
}

message ResolveConflictResponse {
    string conflict_id = 1;
    string resolution = 2;
    string resolved_at = 3;
}
```

---

## 6. Events

### Published Events
| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `siebel.extract.completed` | `siebel-migration.events` | `{ project_id, siebel_object, batch_id, record_count, staging_status }` | Siebel data extraction completed and staged |
| `siebel.run.completed` | `siebel-migration.events` | `{ run_id, project_id, siebel_object, run_number, records_succeeded, records_failed, status }` | Migration run completed |
| `siebel.delta.sync.executed` | `siebel-migration.events` | `{ config_id, project_id, siebel_object, records_synced, conflicts, sync_direction }` | Delta synchronization executed |
| `siebel.conflict.detected` | `siebel-migration.events` | `{ config_id, project_id, siebel_object, conflict_id, source_value, target_value, field }` | Data conflict detected during delta sync |

### Consumed Events
| Event | Source | Description |
|-------|--------|-------------|
| `migration.approved` | Admin | Administrative approval to begin migration execution |
| `tenant.ready` | Multi-Tenancy (18) | Target Fusion tenant provisioned and ready for data loading |

---

## 7. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `multitenancy-service` (18) | Target tenant configuration, instance readiness, and provisioning status |
| `sales-service` (77) | Target account, contact, and opportunity entity schemas for mapping |
| `servicecenter-service` (80) | Target service request and activity entity schemas for mapping |
| `cdp-service` (60) | Unified customer profile model for contact deduplication and enrichment |
| `dataimport-service` (30) | Bulk data import APIs for loading transformed records into Fusion |

### Published To
| Service | Data |
|---------|------|
| `sales-service` (77) | Migrated accounts, contacts, opportunities, quotes, and orders |
| `servicecenter-service` (80) | Migrated service requests and activities |
| `cdp-service` (60) | Unified customer profiles from merged Siebel and Fusion contact data |
| `dataimport-service` (30) | Transformation output files for bulk import execution |
| `notification-service` | Migration progress alerts, conflict notifications, and run status updates |
| `audit-service` | Full migration audit trail for compliance and rollback support |

---

## 8. Migrations

1. V001: `sm_migration_projects`
2. V002: `sm_object_mappings`
3. V003: `sm_data_staging`
4. V004: `sm_migration_runs`
5. V005: `sm_delta_sync_configs`
6. V006: Triggers for `updated_at`
