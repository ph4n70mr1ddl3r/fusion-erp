# 135 - Document IO Agent Specification

## 1. Domain Overview

Document IO Agent provides AI-powered document capture, processing, generation, and transformation across multiple formats, standards, and languages. It automates the ingestion of invoices, receipts, contracts, purchase orders, and other business documents using OCR, intelligent extraction, and classification, then routes extracted data to the appropriate services.

**Bounded Context:** AI Document Processing & Intelligent Capture
**Service Name:** `docio-service`
**Database:** `data/docio.db`
**HTTP Port:** 8155 | **gRPC Port:** 9155

---

## 2. Database Schema

### 2.1 Document Templates
```sql
CREATE TABLE dio_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_code TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN ('INVOICE','PURCHASE_ORDER','RECEIPT','CONTRACT','BILL_OF_LADING','PACKING_SLIP','CUSTOMS_DECLARATION','TAX_FORM','EXPENSE_RECEIPT','BANK_STATEMENT','EMPLOYEE_FORM','OTHER')),
    description TEXT,
    extraction_schema TEXT NOT NULL,     -- JSON Schema: fields to extract with types
    classification_rules TEXT,           -- JSON: rules for auto-classifying documents
    validation_rules TEXT,               -- JSON: post-extraction validation logic
    supported_formats TEXT NOT NULL,     -- JSON: ["PDF","PNG","JPG","TIFF","DOCX","XML","CSV","EDI"]
    language TEXT NOT NULL DEFAULT 'en',
    country TEXT NOT NULL DEFAULT 'US',
    processing_priority INTEGER NOT NULL DEFAULT 5,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);
```

### 2.2 Incoming Documents
```sql
CREATE TABLE dio_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_number TEXT NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('UPLOAD','EMAIL','FAX','API','SCANNER','EDI','SFTP')),
    source_reference TEXT,               -- Email message ID, SFTP filename, etc.
    file_name TEXT NOT NULL,
    file_url TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    file_type TEXT NOT NULL,             -- MIME type
    file_hash TEXT NOT NULL,             -- SHA-256 for deduplication
    document_type TEXT,                  -- Auto-classified or manually set
    template_id TEXT,                    -- Matched template
    language TEXT,
    page_count INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'RECEIVED'
        CHECK(status IN ('RECEIVED','CLASSIFYING','EXTRACTING','REVIEWING','APPROVED','REJECTED','ERROR','DUPLICATE')),
    extracted_data TEXT,                 -- JSON: extracted field values
    extraction_confidence REAL,          -- Overall confidence 0.0-1.0
    classified_by TEXT,                  -- "AI", "RULE", "MANUAL"
    reviewed_by TEXT,
    reviewed_at TEXT,
    error_message TEXT,
    reference_type TEXT,                 -- "AP_INVOICE", "EXPENSE_RECEIPT", etc.
    reference_id TEXT,                   -- ID of created target record
    processing_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, document_number)
);

CREATE INDEX idx_dio_docs_tenant_status ON dio_documents(tenant_id, status);
CREATE INDEX idx_dio_docs_tenant_type ON dio_documents(tenant_id, document_type);
CREATE INDEX idx_dio_docs_tenant_hash ON dio_documents(tenant_id, file_hash);
```

### 2.3 Extraction Results
```sql
CREATE TABLE dio_extraction_fields (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    document_id TEXT NOT NULL,
    field_name TEXT NOT NULL,
    field_value TEXT,
    field_type TEXT NOT NULL CHECK(field_type IN ('STRING','NUMBER','DATE','AMOUNT','EMAIL','PHONE','ADDRESS','TABLE','ARRAY')),
    confidence REAL NOT NULL DEFAULT 0.0,
    bounding_box TEXT,                   -- JSON: { "page": 1, "x": 100, "y": 200, "w": 300, "h": 50 }
    source_page INTEGER,
    is_edited INTEGER NOT NULL DEFAULT 0,
    original_value TEXT,                 -- Before human correction
    edited_by TEXT,
    edited_at TEXT,
    validation_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(validation_status IN ('PENDING','VALID','INVALID','CORRECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (document_id) REFERENCES dio_documents(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, document_id, field_name)
);

CREATE INDEX idx_dio_fields_tenant_doc ON dio_extraction_fields(tenant_id, document_id);
CREATE INDEX idx_dio_fields_tenant_validation ON dio_extraction_fields(tenant_id, validation_status);
```

### 2.4 Document Generation
```sql
CREATE TABLE dio_generated_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    generation_template_id TEXT NOT NULL,
    target_document_type TEXT NOT NULL,
    target_format TEXT NOT NULL DEFAULT 'PDF'
        CHECK(target_format IN ('PDF','DOCX','HTML','XML','CSV','JSON','EDI')),
    source_data TEXT NOT NULL,           -- JSON: data used to populate template
    source_reference_type TEXT,
    source_reference_id TEXT,
    file_url TEXT,
    file_size_bytes INTEGER,
    language TEXT NOT NULL DEFAULT 'en',
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','GENERATING','COMPLETED','FAILED')),
    generated_at TEXT,
    delivery_method TEXT CHECK(delivery_method IN ('DOWNLOAD','EMAIL','SFTP','API')),
    delivery_status TEXT CHECK(delivery_status IN ('PENDING','SENT','DELIVERED','FAILED')),
    delivered_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_dio_gen_tenant_status ON dio_generated_documents(tenant_id, status);
CREATE INDEX idx_dio_gen_tenant_source ON dio_generated_documents(tenant_id, source_reference_type, source_reference_id);
```

### 2.5 Processing Queue
```sql
CREATE TABLE dio_processing_queue (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    processing_step TEXT NOT NULL CHECK(processing_step IN ('CLASSIFY','OCR','EXTRACT','VALIDATE','TRANSFORM','ROUTE')),
    priority INTEGER NOT NULL DEFAULT 5,
    status TEXT NOT NULL DEFAULT 'QUEUED'
        CHECK(status IN ('QUEUED','PROCESSING','COMPLETED','FAILED')),
    retry_count INTEGER NOT NULL DEFAULT 0,
    max_retries INTEGER NOT NULL DEFAULT 3,
    error_message TEXT,
    started_at TEXT,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (document_id) REFERENCES dio_documents(id) ON DELETE CASCADE
);

CREATE INDEX idx_dio_queue_tenant_status ON dio_processing_queue(tenant_id, status, priority);
```

---

## 3. REST API Endpoints

### 3.1 Document Ingestion
```
POST   /api/v1/docio/documents/upload                   Permission: dio.documents.upload
  Content-Type: multipart/form-data
  Response 201: { "data": { "document_id": "...", "document_number": "DOC-2024-001" } }
POST   /api/v1/docio/documents/email-ingest             Permission: dio.documents.ingest
  Request: { "from": "...", "subject": "...", "attachments": [...] }
POST   /api/v1/docio/documents/batch                    Permission: dio.documents.upload
  Request: { "documents": [{ "file_url": "...", "source_type": "SFTP" }] }
GET    /api/v1/docio/documents                           Permission: dio.documents.read
GET    /api/v1/docio/documents/{id}                      Permission: dio.documents.read
GET    /api/v1/docio/documents/{id}/file                 Permission: dio.documents.download
GET    /api/v1/docio/documents/{id}/extracted            Permission: dio.documents.read
PUT    /api/v1/docio/documents/{id}/extracted            Permission: dio.documents.correct
POST   /api/v1/docio/documents/{id}/approve              Permission: dio.documents.approve
POST   /api/v1/docio/documents/{id}/reject               Permission: dio.documents.reject
```

### 3.2 Templates
```
GET    /api/v1/docio/templates                           Permission: dio.templates.read
GET    /api/v1/docio/templates/{id}                      Permission: dio.templates.read
POST   /api/v1/docio/templates                           Permission: dio.templates.create
PUT    /api/v1/docio/templates/{id}                      Permission: dio.templates.update
POST   /api/v1/docio/templates/{id}/test                 Permission: dio.templates.test
  Request: { "document_url": "..." }
  Response: Classification + extraction preview
```

### 3.3 Document Generation
```
POST   /api/v1/docio/generate                            Permission: dio.generate.create
  Request: { "template_id": "...", "data": {...}, "format": "PDF", "language": "en" }
  Response 201: { "data": { "document_id": "...", "file_url": "..." } }
GET    /api/v1/docio/generate/{id}                       Permission: dio.generate.read
GET    /api/v1/docio/generate/{id}/download              Permission: dio.generate.download
POST   /api/v1/docio/generate/{id}/deliver               Permission: dio.generate.deliver
```

### 3.4 Processing
```
GET    /api/v1/docio/processing/queue                    Permission: dio.processing.read
POST   /api/v1/docio/processing/reprocess/{document_id}  Permission: dio.processing.reprocess
GET    /api/v1/docio/processing/stats                    Permission: dio.processing.read
GET    /api/v1/docio/dashboard                           Permission: dio.dashboard.read
```

---

## 4. Business Rules

### 4.1 Document Classification
- Auto-classification using AI model trained on template patterns
- Classification considers: file type, layout, keywords, header/footer patterns
- Confidence threshold (default 0.85): below threshold flags for manual review
- Learning: manual corrections improve future classification accuracy
- Multi-page documents classified as single document type

### 4.2 Extraction Rules
- Field extraction uses template-defined schema
- Key-value extraction: label-value pairs from document layout
- Table extraction: line-item tables from invoices, POs
- Handwriting recognition for manually completed forms
- Multi-language OCR with automatic language detection
- Confidence scoring per extracted field

### 4.3 Validation & Routing
- Post-extraction validation: required fields, data types, business rules
- Cross-reference validation: vendor exists? PO number valid? Amount within tolerance?
- Documents passing validation auto-routed to target service
- Documents failing validation queued for human review
- Duplicate detection via file hash + document type + key fields

### 4.4 Document Generation
- Template-based generation with data merge fields
- Multi-format output: PDF, DOCX, XML, EDI
- Multi-language: same template, different language variants
- Batch generation for bulk document needs (e.g., 1000 invoices)
- Digital signature application for legal documents

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.docio.v1;

service DocumentIOService {
    rpc IngestDocument(IngestDocumentRequest) returns (IngestDocumentResponse);
    rpc GetExtractedData(GetExtractedDataRequest) returns (GetExtractedDataResponse);
    rpc ClassifyDocument(ClassifyDocumentRequest) returns (ClassifyDocumentResponse);
    rpc GenerateDocument(GenerateDocumentRequest) returns (GenerateDocumentResponse);
    rpc ApproveDocument(ApproveDocumentRequest) returns (ApproveDocumentResponse);
    rpc GetDocumentStatus(GetDocumentStatusRequest) returns (GetDocumentStatusResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **ai-service**: OCR engine, classification models, extraction models, NLP
- **dms-service**: Document storage, version management
- **ap-service**: Invoice document → AP invoice creation
- **expense-service**: Receipt → expense line creation
- **contracts-service**: Contract document ingestion
- **etl-service**: Bulk document processing pipelines
- **workflow-service**: Document review and approval workflows

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `dio.document.received` | Document uploaded/ingested | document_id, source_type |
| `dio.document.classified` | Document type identified | document_id, document_type, confidence |
| `dio.document.extracted` | Data extraction complete | document_id, field_count, confidence |
| `dio.document.approved` | Document review approved | document_id, reference_type, reference_id |
| `dio.document.rejected` | Document review rejected | document_id, rejection_reason |
| `dio.document.duplicate` | Duplicate document detected | document_id, original_document_id |
| `dio.document.generated` | Document generated | document_id, format, file_url |

---

## 7. Migrations

### Migration Order for docio-service:
1. V001: `dio_templates`
2. V002: `dio_documents`
3. V003: `dio_extraction_fields`
4. V004: `dio_generated_documents`
5. V005: `dio_processing_queue`
6. V006: Triggers for `updated_at`
7. V007: Seed data (invoice template, receipt template, PO template, expense receipt template)
