# 49 - Accounting Hub Service Specification

## 1. Domain Overview

Accounting Hub provides a centralized accounting transformation engine that captures transactional data from third-party systems (banking, insurance, hospitality, retail, legacy ERPs) and generates compliant journal entries in the General Ledger. It offers a rules-based accounting transformation engine that maps source transactions to accounting entries using configurable templates with debit/credit patterns. Supports multiple source systems with different chart of accounts and maps them to a unified GL structure.

**Bounded Context:** Centralized Accounting Transformation & Multi-System Journal Generation
**Service Name:** `accounthub-service`
**Database:** `data/accounthub.db`
**HTTP Port:** 8081 | **gRPC Port:** 9081

---

## 2. Database Schema

### 2.1 Source Systems
```sql
CREATE TABLE source_systems (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    system_code TEXT NOT NULL,
    system_name TEXT NOT NULL,
    system_type TEXT NOT NULL
        CHECK(system_type IN ('ERP','BANKING','RETAIL','INSURANCE','HOSPITALITY','CUSTOM','LEGACY')),
    connection_config TEXT NOT NULL,
    authentication_type TEXT NOT NULL
        CHECK(authentication_type IN ('API_KEY','OAUTH2','BASIC','CERTIFICATE','NONE')),
    is_active INTEGER NOT NULL DEFAULT 1,
    last_sync_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, system_code)
);

CREATE INDEX idx_source_systems_tenant ON source_systems(tenant_id, system_type);
```

### 2.2 Source Event Types
```sql
CREATE TABLE source_event_types (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_system_id TEXT NOT NULL,
    event_type_code TEXT NOT NULL,
    event_type_name TEXT NOT NULL,
    event_category TEXT NOT NULL
        CHECK(event_category IN ('SALES','PURCHASE','PAYMENT','RECEIPT','TRANSFER','ADJUSTMENT','DEPRECIATION','OTHER')),
    description TEXT,
    payload_schema TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_system_id) REFERENCES source_systems(id),
    UNIQUE(tenant_id, source_system_id, event_type_code)
);

CREATE INDEX idx_source_events_tenant ON source_event_types(tenant_id, source_system_id);
```

### 2.3 Accounting Rules
```sql
CREATE TABLE accounting_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    source_event_type_id TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 100,
    condition_expression TEXT,
    journal_batch_description TEXT,
    gl_ledger_id TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_event_type_id) REFERENCES source_event_types(id),
    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_accounting_rules_tenant ON accounting_rules(tenant_id, source_event_type_id, priority);
CREATE INDEX idx_accounting_rules_active ON accounting_rules(tenant_id, is_active, priority);
```

### 2.4 Accounting Rule Lines
```sql
CREATE TABLE accounting_rule_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    debit_credit TEXT NOT NULL CHECK(debit_credit IN ('DEBIT','CREDIT')),
    account_segment_mapping TEXT NOT NULL,
    amount_source TEXT NOT NULL,
    amount_expression TEXT,
    description_template TEXT,
    dimension_mapping TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (rule_id) REFERENCES accounting_rules(id) ON DELETE CASCADE
);

CREATE INDEX idx_rule_lines_rule ON accounting_rule_lines(rule_id, line_number);
```

### 2.5 Source Transactions
```sql
CREATE TABLE source_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_system_id TEXT NOT NULL,
    source_event_type_id TEXT NOT NULL,
    transaction_ref TEXT NOT NULL,
    transaction_date TEXT NOT NULL,
    raw_payload TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'RECEIVED'
        CHECK(status IN ('RECEIVED','VALIDATING','VALIDATED','PROCESSING','COMPLETED','ERROR')),
    error_message TEXT,
    processing_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_system_id) REFERENCES source_systems(id),
    FOREIGN KEY (source_event_type_id) REFERENCES source_event_types(id),
    UNIQUE(tenant_id, source_system_id, transaction_ref)
);

CREATE INDEX idx_source_trans_tenant ON source_transactions(tenant_id, status, transaction_date);
CREATE INDEX idx_source_trans_system ON source_transactions(source_system_id, transaction_date);
```

### 2.6 Journal Batches
```sql
CREATE TABLE journal_batches (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    batch_number TEXT NOT NULL,
    source_transaction_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    ledger_id TEXT,
    batch_status TEXT NOT NULL DEFAULT 'GENERATED'
        CHECK(batch_status IN ('GENERATED','REVIEW','APPROVED','POSTED','REJECTED')),
    total_debits INTEGER NOT NULL DEFAULT 0,
    total_credits INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,
    posted_to_gl INTEGER NOT NULL DEFAULT 0,
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_transaction_id) REFERENCES source_transactions(id),
    FOREIGN KEY (rule_id) REFERENCES accounting_rules(id),
    UNIQUE(tenant_id, batch_number)
);

CREATE INDEX idx_journal_batches_tenant ON journal_batches(tenant_id, batch_status);
CREATE INDEX idx_journal_batches_source ON journal_batches(source_transaction_id);
```

### 2.7 Journal Batch Lines
```sql
CREATE TABLE journal_batch_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    batch_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    account_code TEXT NOT NULL,
    debit_credit TEXT NOT NULL CHECK(debit_credit IN ('DEBIT','CREDIT')),
    amount INTEGER NOT NULL,
    description TEXT,
    dimension1 TEXT,
    dimension2 TEXT,
    dimension3 TEXT,
    dimension4 TEXT,
    source_line_ref TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (batch_id) REFERENCES journal_batches(id) ON DELETE CASCADE
);

CREATE INDEX idx_batch_lines_batch ON journal_batch_lines(batch_id, line_number);
```

### 2.8 Transformation Logs
```sql
CREATE TABLE transformation_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_transaction_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    batch_id TEXT,
    log_level TEXT NOT NULL CHECK(log_level IN ('INFO','WARN','ERROR')),
    message TEXT NOT NULL,
    field_name TEXT,
    field_value TEXT,
    transformed_value TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_transaction_id) REFERENCES source_transactions(id)
);

CREATE INDEX idx_trans_logs_tenant ON transformation_logs(tenant_id, source_transaction_id);
CREATE INDEX idx_trans_logs_level ON transformation_logs(tenant_id, log_level);
```

---

## 3. REST API Endpoints

### 3.1 Source Systems
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/accounthub/source-systems` | List registered source systems |
| POST | `/api/v1/accounthub/source-systems` | Register source system |
| GET | `/api/v1/accounthub/source-systems/{id}` | Get source system |
| PUT | `/api/v1/accounthub/source-systems/{id}` | Update source system |
| DELETE | `/api/v1/accounthub/source-systems/{id}` | Deactivate source system |
| POST | `/api/v1/accounthub/source-systems/{id}/test` | Test source system connection |

### 3.2 Source Event Types
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/accounthub/event-types` | List source event types |
| POST | `/api/v1/accounthub/event-types` | Define source event type |
| GET | `/api/v1/accounthub/event-types/{id}` | Get event type |
| PUT | `/api/v1/accounthub/event-types/{id}` | Update event type |

### 3.3 Accounting Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/accounthub/rules` | List accounting rules |
| POST | `/api/v1/accounthub/rules` | Create accounting rule |
| GET | `/api/v1/accounthub/rules/{id}` | Get rule with lines |
| PUT | `/api/v1/accounthub/rules/{id}` | Update rule |
| DELETE | `/api/v1/accounthub/rules/{id}` | Deactivate rule |
| POST | `/api/v1/accounthub/rules/{id}/lines` | Add rule line |
| PUT | `/api/v1/accounthub/rules/{id}/lines/{lineId}` | Update rule line |

### 3.4 Source Transactions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/accounthub/transactions` | Submit source transaction |
| POST | `/api/v1/accounthub/transactions/batch` | Submit batch of transactions |
| GET | `/api/v1/accounthub/transactions` | List source transactions |
| GET | `/api/v1/accounthub/transactions/{id}` | Get transaction detail |
| POST | `/api/v1/accounthub/transactions/{id}/reprocess` | Reprocess failed transaction |

### 3.5 Journal Batches
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/accounthub/journal-batches` | List generated journal batches |
| GET | `/api/v1/accounthub/journal-batches/{id}` | Get batch with lines |
| POST | `/api/v1/accounthub/journal-batches/{id}/approve` | Approve batch |
| POST | `/api/v1/accounthub/journal-batches/{id}/reject` | Reject batch |
| POST | `/api/v1/accounthub/journal-batches/{id}/post` | Post to GL |
| POST | `/api/v1/accounthub/journal-batches/bulk-post` | Bulk post approved batches |

### 3.6 Transformation Logs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/accounthub/transformation-logs` | List transformation logs |
| GET | `/api/v1/accounthub/transformation-logs/errors` | Get error summary |

---

## 4. Business Rules

1. Source systems MUST be registered before submitting transactions.
2. Each source event type MUST belong to exactly one source system.
3. Accounting rules MUST have at least one debit and one credit line.
4. Total debits MUST equal total credits for each generated journal batch (double-entry bookkeeping).
5. Rules are evaluated in priority order — the first matching rule MUST be applied.
6. Condition expressions MUST be validated at rule creation time.
7. Source transactions with status ERROR MUST be reviewable and reprocessable.
8. Journal batches MUST NOT be posted to GL until approved by an authorized user.
9. Account segment mappings MUST reference valid GL account codes.
10. Transformation logs MUST be retained for audit purposes for the period defined by tenant policy.
11. Batch posting to GL MUST be idempotent — retrying a successful post MUST NOT create duplicate entries.
12. Source transaction references MUST be unique per source system.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.accounthub.v1;

service AccountingHubService {
    rpc RegisterSourceSystem(RegisterSourceSystemRequest) returns (SourceSystemResponse);
    rpc ListSourceSystems(ListSourceSystemsRequest) returns (ListSourceSystemsResponse);

    rpc DefineEventType(DefineEventTypeRequest) returns (EventTypeResponse);
    rpc ListEventTypes(ListEventTypesRequest) returns (ListEventTypesResponse);

    rpc CreateAccountingRule(CreateAccountingRuleRequest) returns (AccountingRuleResponse);
    rpc GetAccountingRule(GetAccountingRuleRequest) returns (AccountingRuleResponse);

    rpc SubmitSourceTransaction(SubmitTransactionRequest) returns (TransactionResponse);
    rpc SubmitBatchTransactions(SubmitBatchTransactionsRequest) returns (BatchSubmitResponse);
    rpc ReprocessTransaction(ReprocessTransactionRequest) returns (TransactionResponse);

    rpc GetJournalBatch(GetJournalBatchRequest) returns (JournalBatchResponse);
    rpc ApproveJournalBatch(ApproveJournalBatchRequest) returns (JournalBatchResponse);
    rpc PostToGl(PostToGlRequest) returns (PostToGlResponse);

    rpc GetTransformationLogs(GetTransformationLogsRequest) returns (TransformationLogsResponse);
}

// Entity messages

message SourceSystem {
    string id = 1;
    string tenant_id = 2;
    string system_code = 3;
    string system_name = 4;
    string system_type = 5;
    string connection_config = 6;
    string authentication_type = 7;
    string last_sync_at = 8;
    string created_at = 9;
    string updated_at = 10;
}

message SourceEventType {
    string id = 1;
    string tenant_id = 2;
    string source_system_id = 3;
    string event_type_code = 4;
    string event_type_name = 5;
    string event_category = 6;
    string description = 7;
    string payload_schema = 8;
    string created_at = 9;
    string updated_at = 10;
}

message AccountingRule {
    string id = 1;
    string tenant_id = 2;
    string rule_code = 3;
    string rule_name = 4;
    string source_event_type_id = 5;
    int32 priority = 6;
    string condition_expression = 7;
    string journal_batch_description = 8;
    string gl_ledger_id = 9;
    string created_at = 10;
    string updated_at = 11;
}

message AccountingRuleLine {
    string id = 1;
    string tenant_id = 2;
    string rule_id = 3;
    int32 line_number = 4;
    string debit_credit = 5;
    string account_segment_mapping = 6;
    string amount_source = 7;
    string amount_expression = 8;
    string description_template = 9;
    string dimension_mapping = 10;
    string created_at = 11;
    string updated_at = 12;
}

message SourceTransaction {
    string id = 1;
    string tenant_id = 2;
    string source_system_id = 3;
    string source_event_type_id = 4;
    string transaction_ref = 5;
    string transaction_date = 6;
    string raw_payload = 7;
    string status = 8;
    string error_message = 9;
    int32 processing_time_ms = 10;
    string created_at = 11;
    string updated_at = 12;
}

message JournalBatch {
    string id = 1;
    string tenant_id = 2;
    string batch_number = 3;
    string source_transaction_id = 4;
    string rule_id = 5;
    string ledger_id = 6;
    string batch_status = 7;
    int64 total_debits = 8;
    int64 total_credits = 9;
    int32 line_count = 10;
    bool posted_to_gl = 11;
    string gl_journal_id = 12;
    string created_at = 13;
    string updated_at = 14;
}

message JournalBatchLine {
    string id = 1;
    string tenant_id = 2;
    string batch_id = 3;
    int32 line_number = 4;
    string account_code = 5;
    string debit_credit = 6;
    int64 amount = 7;
    string description = 8;
    string dimension1 = 9;
    string dimension2 = 10;
    string dimension3 = 11;
    string dimension4 = 12;
    string source_line_ref = 13;
    string created_at = 14;
    string updated_at = 15;
}

message TransformationLog {
    string id = 1;
    string tenant_id = 2;
    string source_transaction_id = 3;
    string rule_id = 4;
    string batch_id = 5;
    string log_level = 6;
    string message = 7;
    string field_name = 8;
    string field_value = 9;
    string transformed_value = 10;
    string created_at = 11;
    string updated_at = 12;
}

// Request/Response messages

message RegisterSourceSystemRequest {
    string tenant_id = 1;
    string system_code = 2;
    string system_name = 3;
    string system_type = 4;
    string connection_config = 5;
    string authentication_type = 6;
}

message SourceSystemResponse {
    SourceSystem data = 1;
}

message ListSourceSystemsRequest {
    string tenant_id = 1;
    string system_type = 2;
}

message ListSourceSystemsResponse {
    repeated SourceSystem items = 1;
}

message DefineEventTypeRequest {
    string tenant_id = 1;
    string source_system_id = 2;
    string event_type_code = 3;
    string event_type_name = 4;
    string event_category = 5;
    string description = 6;
    string payload_schema = 7;
}

message EventTypeResponse {
    SourceEventType data = 1;
}

message ListEventTypesRequest {
    string tenant_id = 1;
    string source_system_id = 2;
}

message ListEventTypesResponse {
    repeated SourceEventType items = 1;
}

message CreateAccountingRuleRequest {
    string tenant_id = 1;
    string rule_code = 2;
    string rule_name = 3;
    string source_event_type_id = 4;
    int32 priority = 5;
    string condition_expression = 6;
    string journal_batch_description = 7;
    string gl_ledger_id = 8;
    repeated AccountingRuleLine lines = 9;
}

message AccountingRuleResponse {
    AccountingRule data = 1;
    repeated AccountingRuleLine lines = 2;
}

message GetAccountingRuleRequest {
    string tenant_id = 1;
    string id = 2;
}

message SubmitTransactionRequest {
    string tenant_id = 1;
    string source_system_id = 2;
    string source_event_type_id = 3;
    string transaction_ref = 4;
    string transaction_date = 5;
    string raw_payload = 6;
}

message TransactionResponse {
    SourceTransaction data = 1;
    JournalBatch batch = 2;
}

message SubmitBatchTransactionsRequest {
    string tenant_id = 1;
    repeated SubmitTransactionRequest transactions = 2;
}

message BatchSubmitResponse {
    repeated TransactionResponse results = 1;
    int32 succeeded = 2;
    int32 failed = 3;
}

message ReprocessTransactionRequest {
    string tenant_id = 1;
    string transaction_id = 2;
}

message GetJournalBatchRequest {
    string tenant_id = 1;
    string id = 2;
}

message JournalBatchResponse {
    JournalBatch data = 1;
    repeated JournalBatchLine lines = 2;
}

message ApproveJournalBatchRequest {
    string tenant_id = 1;
    string id = 2;
}

message PostToGlRequest {
    string tenant_id = 1;
    string batch_id = 2;
}

message PostToGlResponse {
    string batch_id = 1;
    string gl_journal_id = 2;
    string status = 3;
}

message GetTransformationLogsRequest {
    string tenant_id = 1;
    string source_transaction_id = 2;
    string log_level = 3;
}

message TransformationLogsResponse {
    repeated TransformationLog items = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `gl-service` | Chart of accounts, ledger info, journal entry validation |
| `auth-service` | User permissions for approval workflows |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | Approved journal entries for posting |
| `workflow-service` | Approval routing for journal batches |
| `report-service` | Transformation statistics, error reports |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `accounthub.source-system.registered` | `{system_id, system_code, type}` | Source system registered |
| `accounthub.transaction.received` | `{transaction_id, source_system, event_type}` | Source transaction received |
| `accounthub.transaction.validated` | `{transaction_id, status}` | Source transaction validated |
| `accounthub.transaction.error` | `{transaction_id, error_message}` | Processing error occurred |
| `accounthub.journal-batch.generated` | `{batch_id, total_debits, total_credits}` | Journal batch generated |
| `accounthub.journal-batch.approved` | `{batch_id, approved_by}` | Journal batch approved |
| `accounthub.journal-batch.posted` | `{batch_id, gl_journal_id}` | Journal batch posted to GL |
| `accounthub.rule.validation-error` | `{rule_id, error_message}` | Rule validation failed |
