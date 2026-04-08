# 30 - Data Import/Export Framework Specification

## 1. Domain Overview

The Data Import/Export Framework provides template-based bulk data loading, transformation, validation, and export capabilities. Supports CSV, XLSX, and JSON formats with field mapping, reference data lookups, and scheduled execution. Essential for data migration, bulk updates, and periodic data exchange.

**Bounded Context:** Data Integration & ETL
**Service Name:** `etl-service`
**Database:** `data/etl.db`
**HTTP Port:** 8029 | **gRPC Port:** 9029

---

## 2. Database Schema

### 2.1 Import Templates
```sql
CREATE TABLE import_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    target_service TEXT NOT NULL,           -- 'gl', 'ap', 'ar', 'inv', etc.
    target_entity TEXT NOT NULL,            -- 'accounts', 'suppliers', 'items', etc.
    target_endpoint TEXT NOT NULL,          -- REST endpoint for data loading
    import_mode TEXT NOT NULL DEFAULT 'UPSERT'
        CHECK(import_mode IN ('CREATE_ONLY','UPDATE_ONLY','UPSERT','REPLACE')),

    -- File format
    supported_formats TEXT NOT NULL DEFAULT '["CSV","XLSX"]',

    -- Processing options
    batch_size INTEGER NOT NULL DEFAULT 100,
    stop_on_error INTEGER NOT NULL DEFAULT 0,
    max_errors INTEGER DEFAULT 100,
    commit_mode TEXT NOT NULL DEFAULT 'BATCH'
        CHECK(commit_mode IN ('ROW','BATCH','ALL_OR_NOTHING')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);
```

### 2.2 Import Template Columns
```sql
CREATE TABLE import_template_columns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    column_position INTEGER NOT NULL,
    source_column_name TEXT NOT NULL,        -- Name in the import file
    target_field_name TEXT NOT NULL,         -- API field name
    data_type TEXT NOT NULL
        CHECK(data_type IN ('STRING','INTEGER','DECIMAL','DATE','BOOLEAN','ENUM','JSON')),
    is_required INTEGER NOT NULL DEFAULT 0,
    max_length INTEGER,
    default_value TEXT,
    validation_regex TEXT,

    -- Transformation
    transformation TEXT,                    -- JSON: {"type":"TRIM|UPPER|LOWER|DATE_FORMAT|LOOKUP","params":{...}}
    lookup_entity TEXT,                     -- Reference entity for code-to-ID lookup
    lookup_field TEXT,                      -- Field to match for lookup

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (template_id) REFERENCES import_templates(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, template_id, column_position)
);
```

### 2.3 Import Jobs
```sql
CREATE TABLE import_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    job_number TEXT NOT NULL,               -- IMP-2024-00001
    original_filename TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    file_format TEXT NOT NULL
        CHECK(file_format IN ('CSV','XLSX','JSON')),

    -- Processing status
    status TEXT NOT NULL DEFAULT 'UPLOADED'
        CHECK(status IN ('UPLOADED','VALIDATING','VALIDATED','IMPORTING','COMPLETED','FAILED','CANCELLED')),

    -- Row counts
    total_rows INTEGER NOT NULL DEFAULT 0,
    processed_rows INTEGER NOT NULL DEFAULT 0,
    successful_rows INTEGER NOT NULL DEFAULT 0,
    failed_rows INTEGER NOT NULL DEFAULT 0,
    skipped_rows INTEGER NOT NULL DEFAULT 0,

    -- Execution
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    error_message TEXT,

    -- File storage
    file_storage_path TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (template_id) REFERENCES import_templates(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, job_number)
);

CREATE INDEX idx_import_jobs_tenant_status ON import_jobs(tenant_id, status);
CREATE INDEX idx_import_jobs_tenant_template ON import_jobs(tenant_id, template_id);
```

### 2.4 Import Job Row Results
```sql
CREATE TABLE import_job_rows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    job_id TEXT NOT NULL,
    row_number INTEGER NOT NULL,
    raw_data TEXT NOT NULL,                  -- JSON: original row data
    transformed_data TEXT,                  -- JSON: transformed data
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','SUCCESS','FAILED','SKIPPED')),
    error_messages TEXT,                    -- JSON array of error objects
    target_id TEXT,                         -- ID of created/updated entity
    processed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (job_id) REFERENCES import_jobs(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, job_id, row_number)
);

CREATE INDEX idx_import_rows_tenant_job ON import_job_rows(tenant_id, job_id);
CREATE INDEX idx_import_rows_tenant_status ON import_job_rows(tenant_id, job_id, status);
```

### 2.5 Export Definitions
```sql
CREATE TABLE export_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    export_code TEXT NOT NULL,
    export_name TEXT NOT NULL,
    description TEXT,
    source_service TEXT NOT NULL,
    source_endpoint TEXT NOT NULL,
    export_format TEXT NOT NULL DEFAULT 'CSV'
        CHECK(export_format IN ('CSV','XLSX','JSON','PDF')),
    field_selection TEXT NOT NULL,           -- JSON array of field names
    default_filters TEXT,                   -- JSON
    default_sort TEXT,
    column_headers TEXT,                    -- JSON: field_name → display_header mapping
    max_rows INTEGER DEFAULT 100000,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, export_code)
);
```

### 2.6 Export Jobs
```sql
CREATE TABLE export_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    export_definition_id TEXT,
    job_number TEXT NOT NULL,               -- EXP-2024-00001
    export_format TEXT NOT NULL,
    filters TEXT,                           -- JSON: actual filters used
    row_count INTEGER NOT NULL DEFAULT 0,
    file_size_bytes INTEGER NOT NULL DEFAULT 0,
    file_storage_path TEXT,

    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED','CANCELLED')),
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (export_definition_id) REFERENCES export_definitions(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, job_number)
);

CREATE INDEX idx_export_jobs_tenant_status ON export_jobs(tenant_id, status);
```

### 2.7 Data Transformations
```sql
CREATE TABLE data_transformations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transformation_name TEXT NOT NULL,
    transformation_type TEXT NOT NULL
        CHECK(transformation_type IN ('TRIM','UPPER','LOWER','REPLACE','DATE_FORMAT','NUMBER_FORMAT','LOOKUP','CONDITIONAL','CONCAT','SPLIT','REGEX_EXTRACT')),
    parameters TEXT NOT NULL,               -- JSON: type-specific parameters
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, transformation_name)
);
```

### 2.8 Reference Data Lookups
```sql
CREATE TABLE reference_lookups (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lookup_name TEXT NOT NULL,
    source_entity TEXT NOT NULL,            -- 'suppliers', 'items', 'accounts'
    source_code_field TEXT NOT NULL,        -- 'supplier_number', 'item_code', 'account_code'
    target_id_field TEXT NOT NULL,          -- 'id'
    additional_filters TEXT,               -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, lookup_name)
);
```

---

## 3. REST API Endpoints

```
# Import Templates
GET/POST      /api/v1/etl/import-templates              Permission: etl.templates.read/create
GET/PUT       /api/v1/etl/import-templates/{id}           Permission: etl.templates.read/update
GET/POST      /api/v1/etl/import-templates/{id}/columns   Permission: etl.templates.read/create
GET           /api/v1/etl/import-templates/{id}/sample    Permission: etl.templates.read

# Import Jobs
POST          /api/v1/etl/import/upload                  Permission: etl.import.create
GET           /api/v1/etl/import/jobs                      Permission: etl.import.read
GET           /api/v1/etl/import/jobs/{id}                 Permission: etl.import.read
POST          /api/v1/etl/import/jobs/{id}/validate       Permission: etl.import.create
POST          /api/v1/etl/import/jobs/{id}/execute        Permission: etl.import.execute
POST          /api/v1/etl/import/jobs/{id}/cancel         Permission: etl.import.update
GET           /api/v1/etl/import/jobs/{id}/errors         Permission: etl.import.read

# Export Definitions
GET/POST      /api/v1/etl/export-definitions              Permission: etl.definitions.read/create
GET/PUT       /api/v1/etl/export-definitions/{id}         Permission: etl.definitions.read/update

# Export Jobs
POST          /api/v1/etl/export/execute                  Permission: etl.export.execute
GET           /api/v1/etl/export/jobs                      Permission: etl.export.read
GET           /api/v1/etl/export/jobs/{id}                 Permission: etl.export.read
GET           /api/v1/etl/export/jobs/{id}/download       Permission: etl.export.read

# Ad-hoc Export
POST          /api/v1/etl/export/adhoc                    Permission: etl.export.execute

# Reference Lookups
GET/POST      /api/v1/etl/lookups                          Permission: etl.lookups.read/create
```

---

## 4. Business Rules

### 4.1 Import Flow
```
1. Upload → Parse file (CSV/XLSX/JSON) into raw rows
2. Validate → Check each row against template column rules
3. Transform → Apply transformations (trim, date format, lookup, etc.)
4. Map → Map source columns to target fields
5. Validate Business → Check business rules (duplicates, references)
6. Load → Call target service REST API to create/update records
7. Report → Generate summary with success/failure counts
```

### 4.2 Validation Pipeline
- **Data type:** Check column values match expected types
- **Required:** Verify all required fields have values
- **Format:** Validate against regex patterns (email, date format, codes)
- **Length:** Check string lengths against max_length
- **Range:** Validate numeric values are within range
- **Lookup:** Verify reference codes exist (supplier_number, item_code, etc.)
- **Duplicate:** Check for duplicate records within import and against existing data

### 4.3 Transformation Rules
- **TRIM:** Remove leading/trailing whitespace
- **UPPER/LOWER:** Convert case
- **DATE_FORMAT:** Parse date from source format to ISO 8601
- **NUMBER_FORMAT:** Parse numbers with locale-specific formatting
- **LOOKUP:** Replace code with ID via reference data lookup
- **CONDITIONAL:** Apply value based on condition
- **CONCAT:** Combine multiple columns
- **DEFAULT:** Apply default value when empty

### 4.4 Import Modes
- **CREATE_ONLY:** Only insert new records; fail if record exists
- **UPDATE_ONLY:** Only update existing records; skip if not found
- **UPSERT:** Create if not exists, update if exists (by natural key)
- **REPLACE:** Delete existing and create new

### 4.5 Error Handling
- Collect all errors per row before reporting
- Error categories: VALIDATION, TRANSFORMATION, BUSINESS_RULE, DUPLICATE, SYSTEM
- Maximum error threshold (default 100); abort import if exceeded
- Failed rows can be downloaded as error report CSV

### 4.6 Export Processing
- Stream data from source service via gRPC
- Apply field selection and formatting
- Generate output file (CSV/XLSX/JSON)
- Support large datasets with chunked processing
- Auto-delete export files after configurable retention period

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `etl.import.completed` | Import job finished | Report |
| `etl.import.failed` | Import job failed | Report |
| `etl.export.completed` | Export file ready | — |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.etl.v1;

service EtlService {
    rpc GetImportTemplate(GetImportTemplateRequest) returns (GetImportTemplateResponse);
    rpc UploadImport(UploadImportRequest) returns (UploadImportResponse);
    rpc GetImportStatus(GetImportStatusRequest) returns (GetImportStatusResponse);
    rpc ExecuteExport(ExecuteExportRequest) returns (ExecuteExportResponse);
    rpc GetExportStatus(GetExportStatusRequest) returns (GetExportStatusResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Calls To (for data loading)
- **All services:** Load data via their REST API endpoints
- **All services:** Query data for export via their gRPC endpoints

### 6.2 Standard Import Templates (Pre-seeded)
- GL Accounts, Journal Entries
- Suppliers, Customers
- Items, Item Categories
- Purchase Orders, Sales Orders
- AP Invoices, AR Invoices
- Inventory balances, Stock Movements
- Users, Roles

---

## 7. Migrations

1. V001: `import_templates`
2. V002: `import_template_columns`
3. V003: `import_jobs`
4. V004: `import_job_rows`
5. V005: `export_definitions`
6. V006: `export_jobs`
7. V007: `data_transformations`
8. V008: `reference_lookups`
9. V009: Triggers for `updated_at`
