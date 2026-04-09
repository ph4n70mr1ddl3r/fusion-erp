# 204 - Taleo Migration Service Specification

## 1. Domain Overview

Taleo Migration Service provides comprehensive data migration tooling for transitioning from Oracle Taleo Recruiting to Oracle Fusion HCM Recruiting. It supports migration project management with phased execution planning (planning, extraction, transformation, loading, validation), entity-level mapping configuration between Taleo objects (requisitions, candidates, applications, offers, employees) and Fusion HCM equivalents with field-level transformation rules and value lookups, automated data extraction from Taleo instances with checksum validation and error reporting, batch transformation engine with data quality scoring and detailed error logging per record and field, and pre/post migration validation with record count reconciliation, discrepancy tracking, and formal sign-off workflow. Enables zero-downtime migration with rollback capabilities and full audit trails for regulatory compliance. Integrates with Multi-Tenancy for tenant provisioning, Recruiting for target entity creation, Core HR for employee data loading, Auth & Security for access control, and Data Import/Export for bulk data operations.

**Bounded Context:** Taleo-to-Oracle Fusion HCM Data Migration
**Service Name:** `taleo-migration-service`
**Database:** `data/taleo_migration.db`
**HTTP Port:** 8222 | **gRPC Port:** 9222

---

## 2. Database Schema

### 2.1 Migration Projects
```sql
CREATE TABLE tm_migration_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_code TEXT NOT NULL,
    project_name TEXT NOT NULL,
    source_instance_url TEXT NOT NULL,
    target_instance_id TEXT NOT NULL,
    scope TEXT NOT NULL,                               -- JSON: array of modules to migrate
    phases TEXT NOT NULL,                              -- JSON: [{phase, status, start, end}]
    current_phase TEXT NOT NULL,
    total_entities INTEGER NOT NULL DEFAULT 0,
    migrated_entities INTEGER NOT NULL DEFAULT 0,
    failed_entities INTEGER NOT NULL DEFAULT 0,
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

CREATE INDEX idx_tm_projects_tenant_status ON tm_migration_projects(tenant_id, status);
CREATE INDEX idx_tm_projects_tenant_phase ON tm_migration_projects(tenant_id, current_phase);
CREATE INDEX idx_tm_projects_tenant_completion ON tm_migration_projects(tenant_id, target_completion);
CREATE INDEX idx_tm_projects_tenant_active ON tm_migration_projects(tenant_id, is_active);
```

### 2.2 Entity Mappings
```sql
CREATE TABLE tm_entity_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    source_entity TEXT NOT NULL
        CHECK(source_entity IN ('TALEO_REQUISITION','TALEO_CANDIDATE','TALEO_APPLICATION','TALEO_OFFER','TALEO_EMPLOYEE')),
    target_entity TEXT NOT NULL,
    field_mappings TEXT NOT NULL,                      -- JSON: [{source_field, target_field, transformation, default_value}]
    value_lookups TEXT,                                -- JSON: LOV value translations
    validation_rules TEXT,                             -- JSON: field validation configurations
    estimated_records INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','ACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES tm_migration_projects(id),
    UNIQUE(tenant_id, project_id, source_entity)
);

CREATE INDEX idx_tm_mappings_tenant_project ON tm_entity_mappings(tenant_id, project_id);
CREATE INDEX idx_tm_mappings_tenant_entity ON tm_entity_mappings(tenant_id, source_entity);
CREATE INDEX idx_tm_mappings_tenant_status ON tm_entity_mappings(tenant_id, status);
CREATE INDEX idx_tm_mappings_tenant_active ON tm_entity_mappings(tenant_id, is_active);
```

### 2.3 Data Extracts
```sql
CREATE TABLE tm_data_extracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('TALEO_REQUISITION','TALEO_CANDIDATE','TALEO_APPLICATION','TALEO_OFFER','TALEO_EMPLOYEE')),
    extract_date TEXT NOT NULL,
    record_count INTEGER NOT NULL DEFAULT 0,
    file_reference TEXT NOT NULL,
    file_size_kb INTEGER NOT NULL DEFAULT 0,
    checksum TEXT NOT NULL,
    validation_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(validation_status IN ('PENDING','VALIDATED','ERRORS')),
    validation_errors TEXT,                            -- JSON: array of validation error details

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES tm_migration_projects(id),
    UNIQUE(tenant_id, project_id, entity_type, extract_date)
);

CREATE INDEX idx_tm_extracts_tenant_project ON tm_data_extracts(tenant_id, project_id);
CREATE INDEX idx_tm_extracts_tenant_entity ON tm_data_extracts(tenant_id, entity_type);
CREATE INDEX idx_tm_extracts_tenant_status ON tm_data_extracts(tenant_id, validation_status);
CREATE INDEX idx_tm_extracts_tenant_date ON tm_data_extracts(tenant_id, extract_date);
CREATE INDEX idx_tm_extracts_tenant_active ON tm_data_extracts(tenant_id, is_active);
```

### 2.4 Transformation Logs
```sql
CREATE TABLE tm_transformation_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    batch_number INTEGER NOT NULL,
    records_extracted INTEGER NOT NULL DEFAULT 0,
    records_transformed INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    transformation_errors TEXT,                        -- JSON: [{record_id, field, error, remediation}]
    data_quality_score REAL,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    duration_ms INTEGER,
    status TEXT NOT NULL DEFAULT 'IN_PROGRESS'
        CHECK(status IN ('IN_PROGRESS','COMPLETED','FAILED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES tm_migration_projects(id)
);

CREATE INDEX idx_tm_transforms_tenant_project ON tm_transformation_logs(tenant_id, project_id);
CREATE INDEX idx_tm_transforms_tenant_entity ON tm_transformation_logs(tenant_id, entity_type);
CREATE INDEX idx_tm_transforms_tenant_batch ON tm_transformation_logs(tenant_id, project_id, batch_number);
CREATE INDEX idx_tm_transforms_tenant_status ON tm_transformation_logs(tenant_id, status);
CREATE INDEX idx_tm_transforms_tenant_quality ON tm_transformation_logs(tenant_id, data_quality_score);
CREATE INDEX idx_tm_transforms_tenant_active ON tm_transformation_logs(tenant_id, is_active);
```

### 2.5 Migration Validations
```sql
CREATE TABLE tm_migration_validations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    pre_migration_count INTEGER NOT NULL DEFAULT 0,
    post_migration_count INTEGER NOT NULL DEFAULT 0,
    match_count INTEGER NOT NULL DEFAULT 0,
    discrepancy_count INTEGER NOT NULL DEFAULT 0,
    discrepancies TEXT,                                -- JSON: [{id, source_value, target_value, field}]
    sign_off_by TEXT,
    sign_off_at TEXT,
    sign_off_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(sign_off_status IN ('PENDING','APPROVED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES tm_migration_projects(id),
    UNIQUE(tenant_id, project_id, entity_type)
);

CREATE INDEX idx_tm_validations_tenant_project ON tm_migration_validations(tenant_id, project_id);
CREATE INDEX idx_tm_validations_tenant_entity ON tm_migration_validations(tenant_id, entity_type);
CREATE INDEX idx_tm_validations_tenant_signoff ON tm_migration_validations(tenant_id, sign_off_status);
CREATE INDEX idx_tm_validations_tenant_active ON tm_migration_validations(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Migration Projects
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/taleo-migration/projects` | List migration projects |
| POST | `/api/v1/taleo-migration/projects` | Create migration project |
| GET | `/api/v1/taleo-migration/projects/{id}` | Get project details |
| PUT | `/api/v1/taleo-migration/projects/{id}` | Update project |
| POST | `/api/v1/taleo-migration/projects/{id}/start` | Start migration |
| POST | `/api/v1/taleo-migration/projects/{id}/hold` | Put migration on hold |
| GET | `/api/v1/taleo-migration/projects/{id}/progress` | Get migration progress |

### 3.2 Entity Mappings
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/taleo-migration/projects/{projectId}/mappings` | List entity mappings |
| POST | `/api/v1/taleo-migration/projects/{projectId}/mappings` | Create entity mapping |
| GET | `/api/v1/taleo-migration/projects/{projectId}/mappings/{id}` | Get mapping details |
| PUT | `/api/v1/taleo-migration/projects/{projectId}/mappings/{id}` | Update mapping |
| POST | `/api/v1/taleo-migration/projects/{projectId}/mappings/{id}/approve` | Approve mapping |
| POST | `/api/v1/taleo-migration/projects/{projectId}/mappings/auto-detect` | Auto-detect field mappings |

### 3.3 Data Extracts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/taleo-migration/projects/{projectId}/extracts` | List data extracts |
| POST | `/api/v1/taleo-migration/projects/{projectId}/extracts` | Trigger data extraction |
| GET | `/api/v1/taleo-migration/projects/{projectId}/extracts/{id}` | Get extract details |
| POST | `/api/v1/taleo-migration/projects/{projectId}/extracts/{id}/validate` | Validate extracted data |
| GET | `/api/v1/taleo-migration/projects/{projectId}/extracts/{id}/download` | Download extract file |

### 3.4 Transformations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/taleo-migration/projects/{projectId}/transformations` | List transformation logs |
| POST | `/api/v1/taleo-migration/projects/{projectId}/transformations/execute` | Execute transformation batch |
| GET | `/api/v1/taleo-migration/projects/{projectId}/transformations/{id}` | Get transformation details |
| GET | `/api/v1/taleo-migration/projects/{projectId}/transformations/errors` | List transformation errors |
| POST | `/api/v1/taleo-migration/projects/{projectId}/transformations/retry-failed` | Retry failed transformations |

### 3.5 Validations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/taleo-migration/projects/{projectId}/validations` | List validation results |
| POST | `/api/v1/taleo-migration/projects/{projectId}/validations/run` | Run migration validation |
| GET | `/api/v1/taleo-migration/projects/{projectId}/validations/{id}` | Get validation details |
| POST | `/api/v1/taleo-migration/projects/{projectId}/validations/{id}/sign-off` | Sign off on validation |
| GET | `/api/v1/taleo-migration/projects/{projectId}/validations/discrepancies` | List all discrepancies |

---

## 4. Business Rules

### 4.1 Migration Project Management
1. project_code MUST be unique within a tenant.
2. Project status transitions MUST follow: PLANNING -> IN_PROGRESS -> COMPLETED; ON_HOLD and FAILED are valid intermediate states.
3. current_phase MUST align with one of the phases defined in the phases JSON.
4. migrated_entities MUST NOT exceed total_entities; failed_entities MUST NOT exceed total_entities.
5. A project in COMPLETED or FAILED status MUST NOT allow new entity mappings or data extracts.

### 4.2 Entity Mapping Configuration
6. The combination of (tenant_id, project_id, source_entity) MUST be unique.
7. source_entity MUST be one of the five Taleo entity types: TALEO_REQUISITION, TALEO_CANDIDATE, TALEO_APPLICATION, TALEO_OFFER, TALEO_EMPLOYEE.
8. field_mappings JSON MUST contain at least one field mapping with source_field, target_field, and transformation.
9. Mapping status transitions MUST follow: DRAFT -> APPROVED -> ACTIVE; only ACTIVE mappings can be used in transformation execution.
10. value_lookups JSON SHOULD cover all Taleo LOV values that have no direct Fusion equivalent.

### 4.3 Data Extraction
11. The combination of (tenant_id, project_id, entity_type, extract_date) MUST be unique.
12. checksum MUST be computed on the extract file and validated on any subsequent read operation.
13. Extracts with validation_status ERRORS MUST list all validation failures in validation_errors JSON before proceeding.
14. record_count MUST match the actual number of records in the referenced extract file.
15. Only APPROVED or ACTIVE entity mappings can be used as the basis for data extraction.

### 4.4 Transformation Execution
16. records_transformed MUST equal records_extracted minus records_failed for a COMPLETED batch.
17. data_quality_score MUST be between 0.0 and 1.0; batches below 0.8 SHOULD be flagged for review.
18. transformation_errors JSON MUST include record_id, field, error, and remediation for each failed record.
19. duration_ms MUST be populated upon batch completion; IN_PROGRESS batches MUST NOT have duration_ms set.
20. Failed batches (status FAILED) MAY be retried; retry creates a new transformation log entry with incremented batch_number.

### 4.5 Migration Validation
21. The combination of (tenant_id, project_id, entity_type) MUST be unique for validations.
22. discrepancy_count MUST equal the length of the discrepancies JSON array.
23. match_count MUST be less than or equal to both pre_migration_count and post_migration_count.
24. sign_off_status transitions MUST follow: PENDING -> APPROVED or PENDING -> REJECTED.
25. sign_off_by and sign_off_at MUST be populated when sign_off_status transitions from PENDING.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.taleomigration.v1;

service TaleoMigrationService {
    // Migration projects
    rpc CreateMigrationProject(CreateMigrationProjectRequest) returns (CreateMigrationProjectResponse);
    rpc GetMigrationProject(GetMigrationProjectRequest) returns (GetMigrationProjectResponse);
    rpc ListMigrationProjects(ListMigrationProjectsRequest) returns (ListMigrationProjectsResponse);
    rpc StartMigration(StartMigrationRequest) returns (StartMigrationResponse);
    rpc GetMigrationProgress(GetMigrationProgressRequest) returns (GetMigrationProgressResponse);

    // Entity mappings
    rpc CreateEntityMapping(CreateEntityMappingRequest) returns (CreateEntityMappingResponse);
    rpc GetEntityMapping(GetEntityMappingRequest) returns (GetEntityMappingResponse);
    rpc ListEntityMappings(ListEntityMappingsRequest) returns (ListEntityMappingsResponse);
    rpc ApproveEntityMapping(ApproveEntityMappingRequest) returns (ApproveEntityMappingResponse);

    // Data extracts
    rpc TriggerExtraction(TriggerExtractionRequest) returns (TriggerExtractionResponse);
    rpc GetDataExtract(GetDataExtractRequest) returns (GetDataExtractResponse);
    rpc ListDataExtracts(ListDataExtractsRequest) returns (ListDataExtractsResponse);
    rpc ValidateExtract(ValidateExtractRequest) returns (ValidateExtractResponse);

    // Transformations
    rpc ExecuteTransformation(ExecuteTransformationRequest) returns (ExecuteTransformationResponse);
    rpc GetTransformationLog(GetTransformationLogRequest) returns (GetTransformationLogResponse);
    rpc ListTransformationLogs(ListTransformationLogsRequest) returns (ListTransformationLogsResponse);
    rpc RetryFailedTransformations(RetryFailedTransformationsRequest) returns (RetryFailedTransformationsResponse);

    // Validations
    rpc RunValidation(RunValidationRequest) returns (RunValidationResponse);
    rpc GetValidation(GetValidationRequest) returns (GetValidationResponse);
    rpc ListValidations(ListValidationsRequest) returns (ListValidationsResponse);
    rpc SignOffValidation(SignOffValidationRequest) returns (SignOffValidationResponse);
}

message CreateMigrationProjectRequest {
    string tenant_id = 1;
    string project_code = 2;
    string project_name = 3;
    string source_instance_url = 4;
    string target_instance_id = 5;
    string scope = 6;
    string phases = 7;
    string target_completion = 8;
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
    string current_phase = 4;
    int32 total_entities = 5;
    int32 migrated_entities = 6;
    int32 failed_entities = 7;
    string status = 8;
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
    int32 total_entities = 2;
    int32 migrated_entities = 3;
    int32 failed_entities = 4;
    double completion_pct = 5;
    string current_phase = 6;
    string estimated_completion = 7;
}

message CreateEntityMappingRequest {
    string tenant_id = 1;
    string project_id = 2;
    string source_entity = 3;
    string target_entity = 4;
    string field_mappings = 5;
    string value_lookups = 6;
    string validation_rules = 7;
    int32 estimated_records = 8;
}

message CreateEntityMappingResponse {
    string mapping_id = 1;
    string source_entity = 2;
    string target_entity = 3;
    string status = 4;
}

message GetEntityMappingRequest {
    string tenant_id = 1;
    string mapping_id = 2;
}

message GetEntityMappingResponse {
    string mapping_id = 1;
    string source_entity = 2;
    string target_entity = 3;
    string field_mappings = 4;
    int32 estimated_records = 5;
    string status = 6;
}

message ListEntityMappingsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string source_entity = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListEntityMappingsResponse {
    repeated GetEntityMappingResponse mappings = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ApproveEntityMappingRequest {
    string tenant_id = 1;
    string mapping_id = 2;
}

message ApproveEntityMappingResponse {
    string mapping_id = 1;
    string status = 2;
}

message TriggerExtractionRequest {
    string tenant_id = 1;
    string project_id = 2;
    string entity_type = 3;
}

message TriggerExtractionResponse {
    string extract_id = 1;
    string entity_type = 2;
    string extract_date = 3;
    string validation_status = 4;
}

message GetDataExtractRequest {
    string tenant_id = 1;
    string extract_id = 2;
}

message GetDataExtractResponse {
    string extract_id = 1;
    string entity_type = 2;
    string extract_date = 3;
    int32 record_count = 4;
    string file_reference = 5;
    string checksum = 6;
    string validation_status = 7;
}

message ListDataExtractsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string entity_type = 3;
    string validation_status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListDataExtractsResponse {
    repeated GetDataExtractResponse extracts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ValidateExtractRequest {
    string tenant_id = 1;
    string extract_id = 2;
}

message ValidateExtractResponse {
    string extract_id = 1;
    string validation_status = 2;
    int32 error_count = 3;
    string validation_errors = 4;
}

message ExecuteTransformationRequest {
    string tenant_id = 1;
    string project_id = 2;
    string entity_type = 3;
    string extract_id = 4;
}

message ExecuteTransformationResponse {
    string log_id = 1;
    int32 batch_number = 2;
    int32 records_transformed = 3;
    int32 records_failed = 4;
    double data_quality_score = 5;
    string status = 6;
}

message GetTransformationLogRequest {
    string tenant_id = 1;
    string log_id = 2;
}

message GetTransformationLogResponse {
    string log_id = 1;
    string entity_type = 2;
    int32 batch_number = 3;
    int32 records_extracted = 4;
    int32 records_transformed = 5;
    int32 records_failed = 6;
    double data_quality_score = 7;
    string transformation_errors = 8;
    string status = 9;
}

message ListTransformationLogsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string entity_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListTransformationLogsResponse {
    repeated GetTransformationLogResponse logs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RetryFailedTransformationsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string entity_type = 3;
    string original_log_id = 4;
}

message RetryFailedTransformationsResponse {
    string new_log_id = 1;
    int32 batch_number = 2;
    int32 records_retried = 3;
    string status = 4;
}

message RunValidationRequest {
    string tenant_id = 1;
    string project_id = 2;
    string entity_type = 3;
}

message RunValidationResponse {
    string validation_id = 1;
    int32 pre_migration_count = 2;
    int32 post_migration_count = 3;
    int32 match_count = 4;
    int32 discrepancy_count = 5;
    string sign_off_status = 6;
}

message GetValidationRequest {
    string tenant_id = 1;
    string validation_id = 2;
}

message GetValidationResponse {
    string validation_id = 1;
    string entity_type = 2;
    int32 pre_migration_count = 3;
    int32 post_migration_count = 4;
    int32 match_count = 5;
    int32 discrepancy_count = 6;
    string discrepancies = 7;
    string sign_off_status = 8;
}

message ListValidationsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string sign_off_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListValidationsResponse {
    repeated GetValidationResponse validations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SignOffValidationRequest {
    string tenant_id = 1;
    string validation_id = 2;
    string sign_off_status = 3;
    string notes = 4;
}

message SignOffValidationResponse {
    string validation_id = 1;
    string sign_off_status = 2;
    string sign_off_by = 3;
    string sign_off_at = 4;
}
```

---

## 6. Events

### Published Events
| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `taleo.extract.completed` | `taleo-migration.events` | `{ extract_id, project_id, entity_type, record_count, checksum, validation_status }` | Data extraction from Taleo completed |
| `taleo.transformation.error` | `taleo-migration.events` | `{ log_id, project_id, entity_type, batch_number, records_failed, sample_errors }` | Transformation batch encountered errors |
| `taleo.migration.phase.completed` | `taleo-migration.events` | `{ project_id, completed_phase, next_phase, migrated_entities, total_entities }` | Migration phase completed |
| `taleo.validation.passed` | `taleo-migration.events` | `{ validation_id, project_id, entity_type, match_count, discrepancy_count, sign_off_status }` | Migration validation completed |

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
| `recruiting-service` (67) | Target entity schemas, field definitions, and LOV values for mapping |
| `hr-service` (47) | Employee data model, job profiles, and organizational structure for employee migration |
| `auth-service` (05) | User authentication for migration operations, role-based access for sign-offs |
| `dataimport-service` (30) | Bulk data import APIs for loading transformed records into Fusion |

### Published To
| Service | Data |
|---------|------|
| `recruiting-service` (67) | Migrated requisitions, candidates, applications, and offers |
| `hr-service` (47) | Migrated employee records and associated job data |
| `dataimport-service` (30) | Transformation output files for bulk import execution |
| `notification-service` | Migration progress alerts, error notifications, and sign-off requests |
| `audit-service` | Full migration audit trail for compliance and rollback support |

---

## 8. Migrations

1. V001: `tm_migration_projects`
2. V002: `tm_entity_mappings`
3. V003: `tm_data_extracts`
4. V004: `tm_transformation_logs`
5. V005: `tm_migration_validations`
6. V006: Triggers for `updated_at`
