# 133 - Payroll Interface & Connect Specification

## 1. Domain Overview

Payroll Interface & Connect provides third-party payroll integration capabilities, enabling bi-directional data exchange between Fusion HCM and external payroll providers (ADP, Workday, SAP, local payroll systems). It manages payroll data extracts, tax filing data, garnishment orders, and payment confirmations through configurable integration templates and scheduled data transfers.

**Bounded Context:** Third-Party Payroll Integration & Data Exchange
**Service Name:** `payrollint-service`
**Database:** `data/payrollint.db`
**HTTP Port:** 8213 | **gRPC Port:** 9213

---

## 2. Database Schema

### 2.1 Payroll Providers
```sql
CREATE TABLE pi_providers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    provider_name TEXT NOT NULL,
    provider_code TEXT NOT NULL,
    provider_type TEXT NOT NULL CHECK(provider_type IN ('ADP','WORKDAY','SAP','CERIDIAN','UKG','LOCAL_GOVERNMENT','CUSTOM')),
    country TEXT NOT NULL DEFAULT 'US',
    api_endpoint TEXT,
    api_version TEXT,
    authentication_type TEXT NOT NULL CHECK(authentication_type IN ('OAUTH2','API_KEY','CERTIFICATE','BASIC','SAML')),
    auth_config TEXT,                    -- JSON: encrypted credentials reference
    status TEXT NOT NULL DEFAULT 'CONFIGURED'
        CHECK(status IN ('CONFIGURED','TESTED','ACTIVE','SUSPENDED','DEPRECATED')),
    last_connection_test TEXT,
    connection_status TEXT DEFAULT 'UNKNOWN'
        CHECK(connection_status IN ('UNKNOWN','HEALTHY','DEGRADED','FAILED')),
    support_contact TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, provider_code)
);
```

### 2.2 Integration Templates
```sql
CREATE TABLE pi_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_code TEXT NOT NULL,
    provider_id TEXT NOT NULL,
    template_type TEXT NOT NULL CHECK(template_type IN ('PAYROLL_EXTRACT','TAX_FILING','GARNISHMENT','BENEFITS','DIRECT_DEPOSIT','NEW_HIRE','TERMINATION','TIME_IMPORT','PAY_STATEMENT','CUSTOM')),
    data_format TEXT NOT NULL CHECK(data_format IN ('CSV','XML','JSON','FIXED_WIDTH','EDI','FLAT_FILE')),
    field_mapping TEXT NOT NULL,         -- JSON: { "source_field": "target_field", "transform": "..." }
    transformation_rules TEXT,           -- JSON: data transformation expressions
    file_naming_pattern TEXT,            -- "PAYROLL_{TENANT}_{DATE}_{SEQ}.csv"
    delimiter TEXT DEFAULT ',',
    encoding TEXT NOT NULL DEFAULT 'UTF-8',
    include_header INTEGER NOT NULL DEFAULT 1,
    batch_size INTEGER NOT NULL DEFAULT 1000,
    schedule_cron TEXT,                  -- Cron expression for scheduled runs
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (provider_id) REFERENCES pi_providers(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, template_code)
);
```

### 2.3 Data Exchanges
```sql
CREATE TABLE pi_exchanges (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    provider_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    exchange_type TEXT NOT NULL CHECK(exchange_type IN ('EXPORT','IMPORT')),
    direction TEXT NOT NULL CHECK(direction IN ('OUTBOUND','INBOUND')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PROCESSING','COMPLETED','PARTIALLY_COMPLETED','FAILED','CANCELLED')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    record_count INTEGER NOT NULL DEFAULT 0,
    success_count INTEGER NOT NULL DEFAULT 0,
    error_count INTEGER NOT NULL DEFAULT 0,
    file_url TEXT,
    file_size_bytes INTEGER,
    checksum TEXT,
    initiated_by TEXT NOT NULL,
    started_at TEXT,
    completed_at TEXT,
    error_details TEXT,                  -- JSON: array of { record, error_message }
    processing_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (provider_id) REFERENCES pi_providers(id) ON DELETE RESTRICT,
    FOREIGN KEY (template_id) REFERENCES pi_templates(id) ON DELETE RESTRICT
);

CREATE INDEX idx_pi_exchanges_tenant_status ON pi_exchanges(tenant_id, status);
CREATE INDEX idx_pi_exchanges_tenant_dates ON pi_exchanges(tenant_id, period_start, period_end);
```

### 2.4 Field Mapping Cache
```sql
CREATE TABLE pi_field_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    source_entity TEXT NOT NULL,         -- "employee", "time_entry", "tax_withholding"
    source_field TEXT NOT NULL,
    target_field TEXT NOT NULL,
    data_type TEXT NOT NULL CHECK(data_type IN ('STRING','INTEGER','DECIMAL','DATE','BOOLEAN','ENUM')),
    transform_expression TEXT,           -- Optional: "date_format(source, 'MMDDYYYY')"
    default_value TEXT,
    is_required INTEGER NOT NULL DEFAULT 0,
    validation_rule TEXT,                -- Regex or range validation
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES pi_templates(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, template_id, source_field)
);
```

### 2.5 Exchange Errors
```sql
CREATE TABLE pi_exchange_errors (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    exchange_id TEXT NOT NULL,
    record_number INTEGER NOT NULL,
    record_data TEXT NOT NULL,           -- Original record data
    error_type TEXT NOT NULL CHECK(error_type IN ('VALIDATION','TRANSFORMATION','CONNECTION','FORMAT','DUPLICATE','REFERENTIAL')),
    error_message TEXT NOT NULL,
    field_name TEXT,
    resolution_status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(resolution_status IN ('OPEN','RESOLVED','IGNORED','ESCALATED')),
    resolved_by TEXT,
    resolved_at TEXT,
    resolution_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (exchange_id) REFERENCES pi_exchanges(id) ON DELETE CASCADE
);

CREATE INDEX idx_pi_errors_tenant_status ON pi_exchange_errors(tenant_id, resolution_status);
```

### 2.6 Payroll Confirmation
```sql
CREATE TABLE pi_confirmations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    provider_id TEXT NOT NULL,
    exchange_id TEXT,
    confirmation_type TEXT NOT NULL CHECK(confirmation_type IN ('PAYROLL_PROCESSED','PAYMENT_CONFIRMED','TAX_FILED','GARNISHMENT_PROCESSED','DIRECT_DEPOSIT_SET')),
    confirmation_date TEXT NOT NULL,
    pay_period_start TEXT NOT NULL,
    pay_period_end TEXT NOT NULL,
    total_pay_cents INTEGER NOT NULL,
    total_employees INTEGER NOT NULL,
    confirmation_reference TEXT,         -- Provider's confirmation number
    status TEXT NOT NULL DEFAULT 'RECEIVED'
        CHECK(status IN ('RECEIVED','VERIFIED','DISCREPANCY','REJECTED')),
    discrepancy_notes TEXT,
    verified_by TEXT,
    verified_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (exchange_id) REFERENCES pi_exchanges(id) ON DELETE SET NULL
);
```

---

## 3. REST API Endpoints

### 3.1 Providers
```
GET    /api/v1/payrollint/providers                      Permission: pi.providers.read
GET    /api/v1/payrollint/providers/{id}                 Permission: pi.providers.read
POST   /api/v1/payrollint/providers                      Permission: pi.providers.create
PUT    /api/v1/payrollint/providers/{id}                 Permission: pi.providers.update
POST   /api/v1/payrollint/providers/{id}/test            Permission: pi.providers.test
GET    /api/v1/payrollint/providers/{id}/status           Permission: pi.providers.read
```

### 3.2 Templates
```
GET    /api/v1/payrollint/templates                      Permission: pi.templates.read
GET    /api/v1/payrollint/templates/{id}                 Permission: pi.templates.read
POST   /api/v1/payrollint/templates                      Permission: pi.templates.create
PUT    /api/v1/payrollint/templates/{id}                 Permission: pi.templates.update
GET    /api/v1/payrollint/templates/{id}/mappings        Permission: pi.templates.read
POST   /api/v1/payrollint/templates/{id}/mappings        Permission: pi.templates.update
POST   /api/v1/payrollint/templates/{id}/preview
  ?period_start=...&period_end=...                       Permission: pi.templates.preview
```

### 3.3 Exchanges
```
POST   /api/v1/payrollint/exchanges                      Permission: pi.exchanges.create
  Request: { "provider_id": "...", "template_id": "...", "period_start": "...", "period_end": "..." }
  Response 201: { "data": { "exchange_id": "...", "status": "PENDING" } }
GET    /api/v1/payrollint/exchanges                      Permission: pi.exchanges.read
GET    /api/v1/payrollint/exchanges/{id}                 Permission: pi.exchanges.read
GET    /api/v1/payrollint/exchanges/{id}/download        Permission: pi.exchanges.download
GET    /api/v1/payrollint/exchanges/{id}/errors          Permission: pi.exchanges.read
PUT    /api/v1/payrollint/exchanges/{id}/cancel          Permission: pi.exchanges.cancel
POST   /api/v1/payrollint/exchanges/{id}/retry           Permission: pi.exchanges.retry
```

### 3.4 Confirmations
```
GET    /api/v1/payrollint/confirmations                  Permission: pi.confirmations.read
POST   /api/v1/payrollint/confirmations                  Permission: pi.confirmations.create
GET    /api/v1/payrollint/confirmations/{id}             Permission: pi.confirmations.read
PUT    /api/v1/payrollint/confirmations/{id}/verify      Permission: pi.confirmations.verify
```

---

## 4. Business Rules

### 4.1 Data Exchange
- Exchanges MUST use templates with validated field mappings
- Outbound data is extracted from HCM tables, transformed, and formatted per template
- Inbound data is validated against expected schema before importing
- Duplicate detection on import based on configurable key fields
- Large exchanges processed in configurable batch sizes
- Failed records don't block entire exchange (partial completion allowed)

### 4.2 Transformation
- Field mappings support: direct mapping, type conversion, date formatting, conditional logic
- Null handling: default values or skip record (configurable per mapping)
- Enum mapping: source values mapped to target provider's enum values
- Currency conversion applied when multi-country payroll
- Sensitive data (SSN, bank accounts) encrypted in transit and at rest

### 4.3 Scheduling
- Scheduled exchanges execute per cron expression
- Missed schedules detected and flagged for manual trigger
- Overlap prevention: same template cannot run concurrently
- Notification on completion (success or failure)

### 4.4 Confirmation Reconciliation
- Payroll confirmations matched against original exchanges
- Total pay and employee counts compared for discrepancies
- Discrepancy threshold: configurable tolerance percentage
- Unconfirmed exchanges escalated after configurable timeout

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.payrollint.v1;

service PayrollInterfaceService {
    rpc InitiateExchange(InitiateExchangeRequest) returns (InitiateExchangeResponse);
    rpc GetExchangeStatus(GetExchangeStatusRequest) returns (GetExchangeStatusResponse);
    rpc ProcessConfirmation(ProcessConfirmationRequest) returns (ProcessConfirmationResponse);
    rpc TestProviderConnection(TestProviderConnectionRequest) returns (TestProviderConnectionResponse);
    rpc GetFieldMappings(GetFieldMappingsRequest) returns (GetFieldMappingsResponse);
}

// Entity messages
message PayrollProvider {
    string id = 1;
    string tenant_id = 2;
    string provider_name = 3;
    string provider_code = 4;
    string provider_type = 5;
    string country = 6;
    string api_endpoint = 7;
    string api_version = 8;
    string authentication_type = 9;
    string auth_config = 10;
    string status = 11;
    string last_connection_test = 12;
    string connection_status = 13;
    string support_contact = 14;
    string created_at = 15;
    string updated_at = 16;
}

message IntegrationTemplate {
    string id = 1;
    string tenant_id = 2;
    string template_name = 3;
    string template_code = 4;
    string provider_id = 5;
    string template_type = 6;
    string data_format = 7;
    string field_mapping = 8;
    string transformation_rules = 9;
    string file_naming_pattern = 10;
    string delimiter = 11;
    string encoding = 12;
    int32 include_header = 13;
    int32 batch_size = 14;
    string schedule_cron = 15;
    string status = 16;
    string created_at = 17;
    string updated_at = 18;
}

message DataExchange {
    string id = 1;
    string tenant_id = 2;
    string provider_id = 3;
    string template_id = 4;
    string exchange_type = 5;
    string direction = 6;
    string status = 7;
    string period_start = 8;
    string period_end = 9;
    int32 record_count = 10;
    int32 success_count = 11;
    int32 error_count = 12;
    string file_url = 13;
    int64 file_size_bytes = 14;
    string checksum = 15;
    string initiated_by = 16;
    string started_at = 17;
    string completed_at = 18;
    string error_details = 19;
    int64 processing_time_ms = 20;
    string created_at = 21;
    string updated_at = 22;
}

message FieldMapping {
    string id = 1;
    string tenant_id = 2;
    string template_id = 3;
    string source_entity = 4;
    string source_field = 5;
    string target_field = 6;
    string data_type = 7;
    string transform_expression = 8;
    string default_value = 9;
    int32 is_required = 10;
    string validation_rule = 11;
    int32 sort_order = 12;
    string created_at = 13;
    string updated_at = 14;
}

message ExchangeError {
    string id = 1;
    string tenant_id = 2;
    string exchange_id = 3;
    int32 record_number = 4;
    string record_data = 5;
    string error_type = 6;
    string error_message = 7;
    string field_name = 8;
    string resolution_status = 9;
    string resolved_by = 10;
    string resolved_at = 11;
    string resolution_notes = 12;
    string created_at = 13;
    string updated_at = 14;
}

message PayrollConfirmation {
    string id = 1;
    string tenant_id = 2;
    string provider_id = 3;
    string exchange_id = 4;
    string confirmation_type = 5;
    string confirmation_date = 6;
    string pay_period_start = 7;
    string pay_period_end = 8;
    int64 total_pay_cents = 9;
    int32 total_employees = 10;
    string confirmation_reference = 11;
    string status = 12;
    string discrepancy_notes = 13;
    string verified_by = 14;
    string verified_at = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Request/Response messages
message InitiateExchangeRequest {
    string tenant_id = 1;
    string provider_id = 2;
    string template_id = 3;
    string period_start = 4;
    string period_end = 5;
}

message InitiateExchangeResponse {
    DataExchange data = 1;
}

message GetExchangeStatusRequest {
    string tenant_id = 1;
    string exchange_id = 2;
}

message GetExchangeStatusResponse {
    DataExchange data = 1;
    repeated ExchangeError errors = 2;
}

message ProcessConfirmationRequest {
    string tenant_id = 1;
    string provider_id = 2;
    string confirmation_type = 3;
    string confirmation_date = 4;
    string pay_period_start = 5;
    string pay_period_end = 6;
    int64 total_pay_cents = 7;
    int32 total_employees = 8;
    string confirmation_reference = 9;
    string exchange_id = 10;
}

message ProcessConfirmationResponse {
    PayrollConfirmation data = 1;
}

message TestProviderConnectionRequest {
    string tenant_id = 1;
    string provider_id = 2;
}

message TestProviderConnectionResponse {
    string provider_id = 1;
    string connection_status = 2;
    string tested_at = 3;
    string error_message = 4;
}

message GetFieldMappingsRequest {
    string tenant_id = 1;
    string template_id = 2;
}

message GetFieldMappingsResponse {
    repeated FieldMapping mappings = 1;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **payroll-service**: Core payroll data extraction
- **hr-service**: Employee master data for extracts
- **timelabor-service**: Time and attendance data
- **benefits-service**: Benefits deduction data
- **tax-service**: Tax withholding data
- **compensation-service**: Salary and compensation data
- **absence-service**: Leave accrual data
- **integration-service**: iPaaS connectivity, API gateway

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `pi.exchange.created` | Exchange initiated | exchange_id, provider_id, type |
| `pi.exchange.completed` | Exchange finished successfully | exchange_id, record_count |
| `pi.exchange.failed` | Exchange failed | exchange_id, error_count |
| `pi.confirmation.received` | Payroll confirmation received | confirmation_id, provider_id |
| `pi.confirmation.discrepancy` | Confirmation doesn't match exchange | confirmation_id, discrepancy_details |
| `pi.provider.connection.failed` | Provider connectivity issue | provider_id, error_message |

---

## 7. Migrations

### Migration Order for payrollint-service:
1. V001: `pi_providers`
2. V002: `pi_templates`
3. V003: `pi_field_mappings`
4. V004: `pi_exchanges`
5. V005: `pi_exchange_errors`
6. V006: `pi_confirmations`
7. V007: Triggers for `updated_at`
8. V008: Seed data (ADP/Workday template defaults, common field mappings)
