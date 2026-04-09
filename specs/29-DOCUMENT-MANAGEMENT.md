# 29 - Document Management Service Specification

## 1. Domain Overview

Document Management provides universal file attachment capabilities across all ERP entities. Supports file upload/download, version control, document categorization, access control, thumbnail generation, and storage management. Every service uses DMS for attaching files to invoices, POs, receipts, contracts, etc.

**Bounded Context:** Document & Attachment Management
**Service Name:** `dms-service`
**Database:** `data/dms.db`
**HTTP Port:** 8028 | **gRPC Port:** 9028

---

## 2. Database Schema

### 2.1 Document Categories
```sql
CREATE TABLE document_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL,
    category_name TEXT NOT NULL,
    description TEXT,
    allowed_content_types TEXT,             -- JSON array: ["application/pdf","image/png",...]
    max_file_size_bytes INTEGER DEFAULT 26214400, -- 25MB
    require_approval INTEGER NOT NULL DEFAULT 0,
    retention_days INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, category_code)
);
```

### 2.2 Documents
```sql
CREATE TABLE documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_name TEXT NOT NULL,
    category_id TEXT,
    description TEXT,

    -- File information
    original_filename TEXT NOT NULL,
    content_type TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    file_hash_sha256 TEXT NOT NULL,         -- Integrity check

    -- Storage
    storage_provider TEXT NOT NULL DEFAULT 'LOCAL'
        CHECK(storage_provider IN ('LOCAL','S3')),
    storage_path TEXT NOT NULL,             -- Relative path or S3 key
    storage_bucket TEXT,                    -- S3 bucket name

    -- Thumbnails
    thumbnail_path TEXT,
    thumbnail_generated INTEGER NOT NULL DEFAULT 0,

    -- Version control
    current_version INTEGER NOT NULL DEFAULT 1,
    is_latest INTEGER NOT NULL DEFAULT 1,

    -- Security
    is_confidential INTEGER NOT NULL DEFAULT 0,
    access_level TEXT NOT NULL DEFAULT 'TENANT'
        CHECK(access_level IN ('PUBLIC','TENANT','DEPARTMENT','CONFIDENTIAL')),

    -- Metadata
    metadata TEXT,                          -- JSON: custom key-value pairs
    ocr_text TEXT,                          -- Extracted text for search
    tags TEXT,                              -- JSON array of tags

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ARCHIVED','DELETED','QUARANTINED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (category_id) REFERENCES document_categories(id) ON DELETE SET NULL
);

CREATE INDEX idx_documents_tenant_category ON documents(tenant_id, category_id);
CREATE INDEX idx_documents_tenant_status ON documents(tenant_id, status);
CREATE INDEX idx_documents_tenant_created ON documents(tenant_id, created_at);
CREATE INDEX idx_documents_tenant_hash ON documents(tenant_id, file_hash_sha256);
```

### 2.3 Document Versions
```sql
CREATE TABLE document_versions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    version_number INTEGER NOT NULL,
    storage_path TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    content_type TEXT NOT NULL,
    file_hash_sha256 TEXT NOT NULL,
    change_description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, document_id, version_number)
);
```

### 2.4 Document Links (Polymorphic)
```sql
CREATE TABLE document_links (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    entity_type TEXT NOT NULL,              -- 'AP_INVOICE', 'AR_INVOICE', 'PURCHASE_ORDER', 'JOURNAL_ENTRY', etc.
    entity_id TEXT NOT NULL,                -- ID of the linked entity
    link_type TEXT NOT NULL DEFAULT 'ATTACHMENT'
        CHECK(link_type IN ('ATTACHMENT','REFERENCE','RECEIPT','CONTRACT','EVIDENCE')),
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, document_id, entity_type, entity_id, link_type)
);

CREATE INDEX idx_doc_links_tenant_entity ON document_links(tenant_id, entity_type, entity_id);
CREATE INDEX idx_doc_links_tenant_document ON document_links(tenant_id, document_id);
```

### 2.5 Document Access Log
```sql
CREATE TABLE document_access_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    action TEXT NOT NULL
        CHECK(action IN ('VIEW','DOWNLOAD','UPLOAD','DELETE','SHARE','PRINT')),
    ip_address TEXT,
    user_agent TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_doc_access_tenant_document ON document_access_log(tenant_id, document_id);
CREATE INDEX idx_doc_access_tenant_user ON document_access_log(tenant_id, user_id);
CREATE INDEX idx_doc_access_tenant_date ON document_access_log(tenant_id, created_at);
```

### 2.6 Storage Configuration
```sql
CREATE TABLE storage_config (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    provider TEXT NOT NULL CHECK(provider IN ('LOCAL','S3')),
    base_path TEXT NOT NULL,
    bucket_name TEXT,
    region TEXT,
    endpoint_url TEXT,
    access_key_encrypted TEXT,
    secret_key_encrypted TEXT,
    max_total_storage_mb INTEGER DEFAULT 5120,
    current_usage_mb REAL NOT NULL DEFAULT 0,
    is_default INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, provider)
);
```

### 2.7 Document Approvals
```sql
CREATE TABLE document_approvals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    requested_by TEXT NOT NULL,
    approver_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED')),
    comments TEXT,
    action_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE
);
```

---

## 3. REST API Endpoints

```
# Documents
POST          /api/v1/dms/documents/upload             Permission: dms.documents.create
GET           /api/v1/dms/documents                      Permission: dms.documents.read
GET           /api/v1/dms/documents/{id}                  Permission: dms.documents.read
GET           /api/v1/dms/documents/{id}/download        Permission: dms.documents.read
PUT           /api/v1/dms/documents/{id}                  Permission: dms.documents.update
DELETE        /api/v1/dms/documents/{id}                  Permission: dms.documents.delete
GET           /api/v1/dms/documents/{id}/thumbnail       Permission: dms.documents.read
POST          /api/v1/dms/documents/{id}/new-version     Permission: dms.documents.update
GET           /api/v1/dms/documents/{id}/versions        Permission: dms.documents.read

# Document Links
POST          /api/v1/dms/links                          Permission: dms.links.create
GET           /api/v1/dms/links?entity_type=X&entity_id=Y Permission: dms.links.read
DELETE        /api/v1/dms/links/{id}                      Permission: dms.links.delete

# Categories
GET/POST      /api/v1/dms/categories                     Permission: dms.categories.read/create
GET/PUT       /api/v1/dms/categories/{id}                 Permission: dms.categories.read/update

# Search
POST          /api/v1/dms/search                         Permission: dms.documents.read
GET           /api/v1/dms/search?query=X&tags=A,B        Permission: dms.documents.read

# Storage
GET           /api/v1/dms/storage/usage                  Permission: dms.storage.read
GET/PUT       /api/v1/dms/storage/config                  Permission: dms.storage.admin

# Approvals
GET           /api/v1/dms/approvals/pending              Permission: dms.approvals.read
POST          /api/v1/dms/approvals/{id}/approve         Permission: dms.approvals.update
POST          /api/v1/dms/approvals/{id}/reject          Permission: dms.approvals.update
```

---

## 4. Business Rules

### 4.1 File Upload
- Accept multipart/form-data with file + metadata
- Validate content type against category allowed types
- Validate file size against category max and tenant storage quota
- Generate SHA-256 hash for deduplication and integrity
- Store file with generated filename (UUID-based) to prevent collisions
- Generate thumbnail for images (PNG, JPG) and PDFs
- Extract OCR text for searchable documents

### 4.2 Supported File Formats
- **Documents:** PDF, DOCX, XLSX, DOC, XLS
- **Images:** PNG, JPG, JPEG, GIF, BMP, TIFF
- **Data:** CSV, JSON, XML
- **Max file size:** 25MB default, configurable per category

### 4.3 Version Control
- Each upload of same document creates new version
- Only latest version is "active" by default
- Previous versions retained and accessible
- Version number incremented sequentially

### 4.4 Document Linking
- Polymorphic links: any entity can have documents attached
- Entity types: AP_INVOICE, AR_INVOICE, PO, SO, JOURNAL_ENTRY, RECEIPT, SUPPLIER, CUSTOMER, ASSET, CONTRACT, etc.
- One document can be linked to multiple entities
- One entity can have multiple documents

### 4.5 Access Control
- Permission-based: `dms.documents.read`, `dms.documents.create`, etc.
- Document-level: confidential documents require elevated permissions
- Access logged for audit trail
- Tenant isolation enforced

### 4.6 Storage Management
- Track usage per tenant
- Quota enforcement on upload
- Storage path: `data/dms/{tenant_id}/{yyyy}/{mm}/{uuid}.{ext}`
- Cleanup of orphaned documents (no links, older than retention period)

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `dms.document.uploaded` | File uploaded | — |
| `dms.document.linked` | Document linked to entity | — |
| `dms.document.deleted` | Document deleted | Audit |
| `dms.storage.quota_warning` | Storage > 80% quota | — |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.dms.v1;

service DocumentManagementService {
    rpc UploadDocument(UploadDocumentRequest) returns (UploadDocumentResponse);
    rpc DownloadDocument(DownloadDocumentRequest) returns (stream DocumentChunk);
    rpc GetDocument(GetDocumentRequest) returns (GetDocumentResponse);
    rpc LinkDocument(LinkDocumentRequest) returns (LinkDocumentResponse);
    rpc GetEntityDocuments(GetEntityDocumentsRequest) returns (GetEntityDocumentsResponse);
    rpc DeleteDocument(DeleteDocumentRequest) returns (DeleteDocumentResponse);
}
```

```protobuf
message UploadDocumentRequest {
    string tenant_id = 1;
    string document_name = 2;
    string category_id = 3;
    string description = 4;
    string original_filename = 5;
    string content_type = 6;
    int64 file_size_bytes = 7;
    string file_hash_sha256 = 8;
    string storage_provider = 9;
    string storage_path = 10;
    string storage_bucket = 11;
    int32 is_confidential = 12;
    string access_level = 13;
    string metadata = 14;
    string tags = 15;
    string created_by = 16;
}

message UploadDocumentResponse {
    Document document = 1;
}

message DownloadDocumentRequest {
    string tenant_id = 1;
    string document_id = 2;
    int32 version_number = 3;
}

message DocumentChunk {
    bytes data = 1;
    int64 offset = 2;
    int64 total_size = 3;
}

message GetDocumentRequest {
    string tenant_id = 1;
    string document_id = 2;
}

message GetDocumentResponse {
    Document document = 1;
}

message LinkDocumentRequest {
    string tenant_id = 1;
    string document_id = 2;
    string entity_type = 3;
    string entity_id = 4;
    string link_type = 5;
    string description = 6;
    string created_by = 7;
}

message LinkDocumentResponse {
    DocumentLink link = 1;
}

message GetEntityDocumentsRequest {
    string tenant_id = 1;
    string entity_type = 2;
    string entity_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetEntityDocumentsResponse {
    repeated DocumentLink links = 1;
    int32 total_count = 2;
    string next_page_token = 3;
}

message DeleteDocumentRequest {
    string tenant_id = 1;
    string document_id = 2;
    string deleted_by = 3;
}

message DeleteDocumentResponse {
    bool success = 1;
    string document_id = 2;
}

message DocumentCategory {
    string id = 1;
    string tenant_id = 2;
    string category_code = 3;
    string category_name = 4;
    string description = 5;
    string allowed_content_types = 6;
    int64 max_file_size_bytes = 7;
    int32 require_approval = 8;
    int32 retention_days = 9;
    string created_at = 10;
    string updated_at = 11;
    string created_by = 12;
    string updated_by = 13;
    int32 version = 14;
    int32 is_active = 15;
}

message Document {
    string id = 1;
    string tenant_id = 2;
    string document_name = 3;
    string category_id = 4;
    string description = 5;
    string original_filename = 6;
    string content_type = 7;
    int64 file_size_bytes = 8;
    string file_hash_sha256 = 9;
    string storage_provider = 10;
    string storage_path = 11;
    string storage_bucket = 12;
    string thumbnail_path = 13;
    int32 thumbnail_generated = 14;
    int32 current_version = 15;
    int32 is_latest = 16;
    int32 is_confidential = 17;
    string access_level = 18;
    string metadata = 19;
    string ocr_text = 20;
    string tags = 21;
    string status = 22;
    string created_at = 23;
    string updated_at = 24;
    string created_by = 25;
    string updated_by = 26;
    int32 version_number = 27;
    int32 is_active = 28;
}

message DocumentVersion {
    string id = 1;
    string tenant_id = 2;
    string document_id = 3;
    int32 version_number = 4;
    string storage_path = 5;
    int64 file_size_bytes = 6;
    string content_type = 7;
    string file_hash_sha256 = 8;
    string change_description = 9;
    string created_at = 10;
    string created_by = 11;
}

message DocumentLink {
    string id = 1;
    string tenant_id = 2;
    string document_id = 3;
    string entity_type = 4;
    string entity_id = 5;
    string link_type = 6;
    string description = 7;
    string created_at = 8;
    string created_by = 9;
}

message StorageConfig {
    string id = 1;
    string tenant_id = 2;
    string provider = 3;
    string base_path = 4;
    string bucket_name = 5;
    string region = 6;
    string endpoint_url = 7;
    int64 max_total_storage_mb = 8;
    double current_usage_mb = 9;
    int32 is_default = 10;
    string created_at = 11;
    string updated_at = 12;
}
```

---

## 6. Inter-Service Integration

### 6.1 Called By (ALL services)
- AP: Attach invoices, receipts to AP invoices
- AR: Attach customer documents to AR invoices
- Procurement: Attach quotes, specs to POs
- OM: Attach order confirmations, shipping docs
- FA: Attach asset photos, depreciation schedules
- PM: Attach project documents, receipts
- GL: Attach supporting documents to journal entries
- CM: Attach bank statements

---

## 7. Migrations

1. V001: `document_categories`
2. V002: `documents`
3. V003: `document_versions`
4. V004: `document_links`
5. V005: `document_access_log`
6. V006: `storage_config`
7. V007: `document_approvals`
8. V008: Triggers for `updated_at`
