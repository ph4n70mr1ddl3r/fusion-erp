# 172 - B2B Gateway / EDI Service Specification

## 1. Domain Overview

B2B Gateway provides electronic data interchange (EDI), B2B messaging, and trading partner integration for automated document exchange. Supports standard EDI formats (ANSI X12, EDIFACT, EANCOM), custom flat files, and XML/JSON document exchange with trading partners. Covers partner onboarding, document translation, validation, correlation, acknowledgment processing, and exception handling. Integrates with Integration Cloud for orchestration, Procurement for PO/ASN exchange, Order Management for order confirmations, and Accounts Payable for invoice processing.

**Bounded Context:** B2B/EDI Document Exchange & Partner Integration
**Service Name:** `b2b-gateway-service`
**Database:** `data/b2b_gateway.db`
**HTTP Port:** 8190 | **gRPC Port:** 9190

---

## 2. Database Schema

### 2.1 Trading Partners
```sql
CREATE TABLE b2b_partners (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    partner_code TEXT NOT NULL,
    partner_name TEXT NOT NULL,
    partner_type TEXT NOT NULL CHECK(partner_type IN ('CUSTOMER','SUPPLIER','LOGISTICS','BANK','CUSTOM')),
    partner_id TEXT,                        -- Reference to internal customer/supplier record
    communications TEXT NOT NULL,           -- JSON: AS2, FTP, SFTP, API, email configs
    certificate_id TEXT,                    -- Digital certificate for AS2/SFTP
    timezone TEXT NOT NULL DEFAULT 'UTC',
    document_types TEXT NOT NULL,           -- JSON: supported EDI document types
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','ACTIVE','SUSPENDED','TERMINATED')),
    onboarding_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, partner_code)
);

CREATE INDEX idx_b2b_partner_tenant ON b2b_partners(tenant_id, partner_type, status);
```

### 2.2 Document Definitions
```sql
CREATE TABLE b2b_document_types (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_standard TEXT NOT NULL CHECK(document_standard IN ('X12','EDIFACT','EANCOM','XML','JSON','FLAT_FILE','CUSTOM')),
    document_type TEXT NOT NULL,            -- e.g., "850" (PO), "855" (PO Ack), "856" (ASN)
    document_name TEXT NOT NULL,            -- e.g., "Purchase Order"
    direction TEXT NOT NULL CHECK(direction IN ('INBOUND','OUTBOUND','BIDIRECTIONAL')),
    version TEXT NOT NULL DEFAULT '004010',
    description TEXT,
    translation_map_id TEXT,                -- Map to internal format
    validation_rules TEXT,                  -- JSON: document validation rules
    acknowledgment_required INTEGER NOT NULL DEFAULT 1,
    ack_document_type TEXT,                 -- e.g., "997" for X12 functional ack
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, document_standard, document_type, direction)
);
```

### 2.3 B2B Transactions
```sql
CREATE TABLE b2b_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,        -- B2B-2024-00001
    partner_id TEXT NOT NULL,
    document_type TEXT NOT NULL,
    document_standard TEXT NOT NULL,
    direction TEXT NOT NULL CHECK(direction IN ('INBOUND','OUTBOUND')),
    interchange_control_id TEXT,
    group_control_id TEXT,
    transaction_control_id TEXT,
    status TEXT NOT NULL DEFAULT 'RECEIVED'
        CHECK(status IN ('RECEIVED','VALIDATING','TRANSLATING','TRANSLATED','PROCESSING','COMPLETED','FAILED','ACK_SENT','ACK_RECEIVED')),
    raw_payload_path TEXT NOT NULL,
    translated_payload_path TEXT,
    internal_document_id TEXT,               -- Reference to PO, Invoice, etc.
    internal_document_type TEXT,
    validation_errors TEXT,                 -- JSON: validation error details
    processing_errors TEXT,
    acknowledgment_status TEXT CHECK(acknowledgment_status IN ('PENDING','SENT','RECEIVED','FAILED','NOT_REQUIRED')),
    ack_control_id TEXT,
    file_size_bytes INTEGER NOT NULL DEFAULT 0,
    retry_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_b2b_txn_partner ON b2b_transactions(partner_id, created_at DESC);
CREATE INDEX idx_b2b_txn_status ON b2b_transactions(tenant_id, status);
CREATE INDEX idx_b2b_txn_type ON b2b_transactions(tenant_id, document_type, direction);
CREATE INDEX idx_b2b_txn_internal ON b2b_transactions(internal_document_type, internal_document_id);
```

### 2.4 Translation Maps
```sql
CREATE TABLE b2b_translation_maps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    map_name TEXT NOT NULL,
    source_format TEXT NOT NULL,
    target_format TEXT NOT NULL,
    document_type TEXT NOT NULL,
    field_mappings TEXT NOT NULL,            -- JSON: source_field → target_field transforms
    default_values TEXT,                    -- JSON: default values for missing fields
    code_mappings TEXT,                     -- JSON: external codes → internal codes
    custom_transformations TEXT,            -- JSON: custom transformation functions
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, map_name)
);
```

### 2.5 B2B Error Log
```sql
CREATE TABLE b2b_error_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_id TEXT NOT NULL,
    error_type TEXT NOT NULL CHECK(error_type IN ('VALIDATION','TRANSLATION','DELIVERY','ACK_TIMEOUT','FORMAT','PARTNER','SYSTEM')),
    severity TEXT NOT NULL CHECK(severity IN ('WARNING','ERROR','CRITICAL')),
    error_code TEXT NOT NULL,
    error_message TEXT NOT NULL,
    segment TEXT,                           -- EDI segment where error occurred
    element TEXT,                           -- Specific element
    resolution TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','RESOLVED','IGNORED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (transaction_id) REFERENCES b2b_transactions(id) ON DELETE CASCADE
);

CREATE INDEX idx_b2b_error_txn ON b2b_error_log(transaction_id);
CREATE INDEX idx_b2b_error_type ON b2b_error_log(tenant_id, error_type, status);
```

---

## 3. API Endpoints

### 3.1 Trading Partners
| Method | Path | Description |
|--------|------|-------------|
| POST | `/b2b-gateway/v1/partners` | Register trading partner |
| GET | `/b2b-gateway/v1/partners` | List partners |
| GET | `/b2b-gateway/v1/partners/{id}` | Get partner details |
| PUT | `/b2b-gateway/v1/partners/{id}` | Update partner |
| POST | `/b2b-gateway/v1/partners/{id}/test-connection` | Test connectivity |

### 3.2 Document Types & Maps
| Method | Path | Description |
|--------|------|-------------|
| POST | `/b2b-gateway/v1/document-types` | Define document type |
| GET | `/b2b-gateway/v1/document-types` | List document types |
| POST | `/b2b-gateway/v1/translation-maps` | Create translation map |
| GET | `/b2b-gateway/v1/translation-maps` | List maps |
| PUT | `/b2b-gateway/v1/translation-maps/{id}` | Update map |

### 3.3 Transactions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/b2b-gateway/v1/send` | Send outbound document |
| POST | `/b2b-gateway/v1/receive` | Receive inbound document |
| GET | `/b2b-gateway/v1/transactions` | List transactions |
| GET | `/b2b-gateway/v1/transactions/{id}` | Get transaction details |
| POST | `/b2b-gateway/v1/transactions/{id}/resend` | Resend failed transaction |
| GET | `/b2b-gateway/v1/transactions/{id}/raw` | Download raw document |

### 3.4 Monitoring
| Method | Path | Description |
|--------|------|-------------|
| GET | `/b2b-gateway/v1/dashboard` | B2B dashboard |
| GET | `/b2b-gateway/v1/errors` | List errors |
| POST | `/b2b-gateway/v1/test-translate` | Test translation with sample document |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `b2b.document.received` | `{ txn_id, partner_id, doc_type }` | Inbound document received |
| `b2b.document.translated` | `{ txn_id, internal_doc_id }` | Document translated to internal format |
| `b2b.document.sent` | `{ txn_id, partner_id, doc_type }` | Outbound document sent |
| `b2b.document.failed` | `{ txn_id, error_type }` | Processing failed |
| `b2b.ack.received` | `{ txn_id, ack_status }` | Acknowledgment received |
| `b2b.ack.timeout` | `{ txn_id, partner_id }` | Acknowledgment timeout |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `po.approved` | Procurement | Send 850 (Purchase Order) |
| `goods_receipt.completed` | Warehouse | Send 856 (ASN) to customer |
| `invoice.created` | AP | Send 810 (Invoice) |
| `order.confirmed` | Order Mgmt | Send 855 (Order Acknowledgment) |

---

## 5. Business Rules

1. **Partner Validation**: Inbound documents validated against partner profile and document type
2. **Translation**: Documents translated from EDI format to internal JSON/XML before processing
3. **Acknowledgment**: Functional acknowledgments (997/CONTRL) auto-generated for received documents
4. **Error Handling**: Failed documents quarantined; retry up to 3 times with configurable delay
5. **Correlation**: Inbound documents correlated with outbound originals (e.g., ASN to PO)
6. **Audit Trail**: All documents stored with raw and translated versions for compliance
7. **Partner Certification**: New partners require test exchange before production activation

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Integration Cloud (89) | Orchestration and connectivity |
| Procurement (11) | PO (850), PO Ack (855) |
| Accounts Payable (07) | Invoice (810), Payment (820) |
| Order Management (13) | Order (850), Ack (855), ASN (856) |
| Warehouse Management (36) | ASN (856) processing |
| Shipping Execution (145) | Shipment confirmation |
| Inventory (12) | Inventory inquiry (846) |
| Supplier Portal (45) | Self-service partner management |
