# 184 - Content Management Service Specification

## 1. Domain Overview

Content Management provides enterprise content management and digital asset management for creating, organizing, versioning, and publishing content across the organization. Supports multiple content types including documents, images, videos, webpages, templates, and forms with full metadata management. Enables hierarchical folder structures with permission inheritance, comprehensive version control with content deltas, multi-stage approval workflows, and multi-language translation management. Integrates with Identity for user permissions, Workflow for approval orchestration, and across all services for unified content delivery.

**Bounded Context:** Enterprise Content Management & Digital Asset Management
**Service Name:** `content-management-service`
**Database:** `data/content_management.db`
**HTTP Port:** 8202 | **gRPC Port:** 9202

---

## 2. Database Schema

### 2.1 Content Items
```sql
CREATE TABLE cm_content_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    content_code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    content_type TEXT NOT NULL CHECK(content_type IN ('DOCUMENT','IMAGE','VIDEO','WEBPAGE','TEMPLATE','FORM')),
    mime_type TEXT,
    file_url TEXT,
    file_size_kb INTEGER,
    folder_id TEXT,
    tags TEXT,                                   -- JSON: array of tags
    metadata TEXT,                               -- JSON: extensible metadata key-value pairs
    language TEXT NOT NULL DEFAULT 'en',
    is_translation_source INTEGER NOT NULL DEFAULT 0,
    source_content_id TEXT,                      -- For translated content, points to source
    current_version INTEGER NOT NULL DEFAULT 1,
    thumbnail_url TEXT,
    checksum TEXT,                               -- File integrity checksum
    owner_id TEXT NOT NULL,
    published_at TEXT,
    expires_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','APPROVED','PUBLISHED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (folder_id) REFERENCES cm_folders(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, content_code)
);

CREATE INDEX idx_cm_ci_tenant ON cm_content_items(tenant_id, status);
CREATE INDEX idx_cm_ci_type ON cm_content_items(content_type);
CREATE INDEX idx_cm_ci_folder ON cm_content_items(folder_id);
CREATE INDEX idx_cm_ci_owner ON cm_content_items(owner_id);
CREATE INDEX idx_cm_ci_lang ON cm_content_items(language);
```

### 2.2 Folders
```sql
CREATE TABLE cm_folders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    folder_path TEXT NOT NULL,                   -- Materialized path: /root/marketing/campaigns
    folder_name TEXT NOT NULL,
    parent_folder_id TEXT,
    description TEXT,
    permissions TEXT NOT NULL,                   -- JSON: permission rules (read/write/admin per role)
    inherit_permissions INTEGER NOT NULL DEFAULT 1,
    effective_permissions TEXT,                  -- JSON: computed permissions after inheritance
    content_count INTEGER NOT NULL DEFAULT 0,
    total_size_kb INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, folder_path)
);

CREATE INDEX idx_cm_folder_tenant ON cm_folders(tenant_id, is_active);
CREATE INDEX idx_cm_folder_parent ON cm_folders(parent_folder_id);
CREATE INDEX idx_cm_folder_path ON cm_folders(folder_path);
```

### 2.3 Content Versions
```sql
CREATE TABLE cm_content_versions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    content_id TEXT NOT NULL,
    version_number INTEGER NOT NULL,
    title TEXT NOT NULL,
    content_delta TEXT,                          -- JSON: diff from previous version
    file_url TEXT,
    file_size_kb INTEGER,
    checksum TEXT,
    change_notes TEXT NOT NULL,
    change_type TEXT NOT NULL CHECK(change_type IN ('CREATE','UPDATE','MINOR_EDIT','MAJOR_REVISION','RESTORE')),
    author_id TEXT NOT NULL,
    reviewed_by TEXT,
    reviewed_at TEXT,
    is_current INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (content_id) REFERENCES cm_content_items(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, content_id, version_number)
);

CREATE INDEX idx_cm_cv_content ON cm_content_versions(content_id, version_number DESC);
CREATE INDEX idx_cm_cv_author ON cm_content_versions(author_id);
CREATE INDEX idx_cm_cv_date ON cm_content_versions(created_at DESC);
```

### 2.4 Approval Workflows
```sql
CREATE TABLE cm_approval_workflows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    content_id TEXT NOT NULL,
    workflow_type TEXT NOT NULL CHECK(workflow_type IN ('PUBLISH','UPDATE','ARCHIVE','TRANSLATION')),
    stages TEXT NOT NULL,                        -- JSON: ordered approval stages
    current_stage INTEGER NOT NULL DEFAULT 0,
    total_stages INTEGER NOT NULL DEFAULT 1,
    current_approvers TEXT NOT NULL,             -- JSON: array of approver IDs for current stage
    approval_comments TEXT,                      -- JSON: array of { approver, comment, action, timestamp }
    deadline TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    requested_by TEXT NOT NULL,
    completed_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_REVIEW','APPROVED','REJECTED','CANCELLED','EXPIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (content_id) REFERENCES cm_content_items(id) ON DELETE CASCADE
);

CREATE INDEX idx_cm_aw_content ON cm_approval_workflows(content_id, status);
CREATE INDEX idx_cm_aw_approver ON cm_approval_workflows(current_approvers);
CREATE INDEX idx_cm_aw_status ON cm_approval_workflows(tenant_id, status);
CREATE INDEX idx_cm_aw_deadline ON cm_approval_workflows(deadline);
```

### 2.5 Content Translations
```sql
CREATE TABLE cm_content_translations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_content_id TEXT NOT NULL,
    translated_content_id TEXT,                  -- Link to the translated content item
    target_language TEXT NOT NULL,
    translation_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(translation_status IN ('PENDING','IN_PROGRESS','IN_REVIEW','APPROVED','PUBLISHED','CANCELLED')),
    translator_id TEXT,
    translation_vendor TEXT,                     -- External translation service
    assigned_at TEXT,
    due_date TEXT,
    completed_at TEXT,
    quality_score REAL,                         -- Translation quality score 0-100
    review_notes TEXT,
    word_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_content_id) REFERENCES cm_content_items(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, source_content_id, target_language)
);

CREATE INDEX idx_cm_ct_source ON cm_content_translations(source_content_id);
CREATE INDEX idx_cm_ct_lang ON cm_content_translations(target_language, translation_status);
CREATE INDEX idx_cm_ct_translator ON cm_content_translations(translator_id, translation_status);
CREATE INDEX idx_cm_ct_status ON cm_content_translations(tenant_id, translation_status);
```

---

## 3. API Endpoints

### 3.1 Content Items
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/content-management/items` | Create content item |
| GET | `/api/v1/content-management/items` | List content items |
| GET | `/api/v1/content-management/items/{id}` | Get content item details |
| PUT | `/api/v1/content-management/items/{id}` | Update content item |
| DELETE | `/api/v1/content-management/items/{id}` | Delete content item |
| POST | `/api/v1/content-management/items/{id}/publish` | Publish content |
| POST | `/api/v1/content-management/items/{id}/archive` | Archive content |
| GET | `/api/v1/content-management/items/search` | Search content |

### 3.2 Folders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/content-management/folders` | Create folder |
| GET | `/api/v1/content-management/folders` | List folders |
| GET | `/api/v1/content-management/folders/{id}` | Get folder details |
| PUT | `/api/v1/content-management/folders/{id}` | Update folder |
| DELETE | `/api/v1/content-management/folders/{id}` | Delete folder |
| PUT | `/api/v1/content-management/folders/{id}/permissions` | Update permissions |
| GET | `/api/v1/content-management/folders/{id}/contents` | List folder contents |

### 3.3 Versioning
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/content-management/items/{id}/versions` | Create new version |
| GET | `/api/v1/content-management/items/{id}/versions` | List versions |
| GET | `/api/v1/content-management/versions/{id}` | Get version details |
| POST | `/api/v1/content-management/versions/{id}/restore` | Restore version |
| GET | `/api/v1/content-management/versions/compare` | Compare two versions |

### 3.4 Workflows
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/content-management/workflows` | Create approval workflow |
| GET | `/api/v1/content-management/workflows` | List workflows |
| GET | `/api/v1/content-management/workflows/{id}` | Get workflow details |
| POST | `/api/v1/content-management/workflows/{id}/approve` | Approve current stage |
| POST | `/api/v1/content-management/workflows/{id}/reject` | Reject workflow |
| GET | `/api/v1/content-management/workflows/pending` | List pending approvals |

### 3.5 Translations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/content-management/translations` | Create translation request |
| GET | `/api/v1/content-management/translations` | List translations |
| GET | `/api/v1/content-management/translations/{id}` | Get translation details |
| PUT | `/api/v1/content-management/translations/{id}` | Update translation |
| POST | `/api/v1/content-management/translations/{id}/complete` | Complete translation |
| GET | `/api/v1/content-management/items/{id}/translations` | Get all translations for item |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `content.published` | `{ content_id, content_type, title, version }` | Content published |
| `content.archived` | `{ content_id, content_type, archived_by }` | Content archived |
| `content.translation.completed` | `{ translation_id, source_id, target_language }` | Translation completed |
| `content.approval.requested` | `{ workflow_id, content_id, approvers }` | Approval requested |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `user.updated` | Identity | Sync content owner and approver data |
| `workflow.completed` | Workflow | Update approval workflow status |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.content_management.v1;

service ContentManagementService {
    rpc GetContentItem(GetContentItemRequest) returns (GetContentItemResponse);
    rpc CreateContentItem(CreateContentItemRequest) returns (CreateContentItemResponse);
    rpc PublishContent(PublishContentRequest) returns (PublishContentResponse);
    rpc GetFolder(GetFolderRequest) returns (GetFolderResponse);
    rpc CreateTranslation(CreateTranslationRequest) returns (CreateTranslationResponse);
    rpc ApproveWorkflow(ApproveWorkflowRequest) returns (ApproveWorkflowResponse);
}

// Content item messages
message GetContentItemRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetContentItemResponse {
    CmContentItem data = 1;
}

message CreateContentItemRequest {
    string tenant_id = 1;
    string content_code = 2;
    string title = 3;
    string description = 4;
    string content_type = 5;
    string mime_type = 6;
    string file_url = 7;
    int32 file_size_kb = 8;
    string folder_id = 9;
    string tags = 10;
    string metadata = 11;
    string language = 12;
    string owner_id = 13;
}

message CreateContentItemResponse {
    CmContentItem data = 1;
}

message CmContentItem {
    string id = 1;
    string tenant_id = 2;
    string content_code = 3;
    string title = 4;
    string description = 5;
    string content_type = 6;
    string mime_type = 7;
    string file_url = 8;
    int32 file_size_kb = 9;
    string folder_id = 10;
    string tags = 11;
    string metadata = 12;
    string language = 13;
    bool is_translation_source = 14;
    string source_content_id = 15;
    int32 current_version = 16;
    string thumbnail_url = 17;
    string checksum = 18;
    string owner_id = 19;
    string published_at = 20;
    string expires_at = 21;
    string status = 22;
    string created_at = 23;
    string updated_at = 24;
}

// Publish messages
message PublishContentRequest {
    string tenant_id = 1;
    string id = 2;
    string change_notes = 3;
}

message PublishContentResponse {
    string id = 1;
    string status = 2;
    int32 version = 3;
    string published_at = 4;
}

// Folder messages
message GetFolderRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetFolderResponse {
    CmFolder data = 1;
}

message CmFolder {
    string id = 1;
    string tenant_id = 2;
    string folder_path = 3;
    string folder_name = 4;
    string parent_folder_id = 5;
    string description = 6;
    string permissions = 7;
    bool inherit_permissions = 8;
    int32 content_count = 9;
    int32 total_size_kb = 10;
    string owner_id = 11;
    string status = 12;
    string created_at = 13;
    string updated_at = 14;
}

// Translation messages
message CreateTranslationRequest {
    string tenant_id = 1;
    string source_content_id = 2;
    string target_language = 3;
    string translator_id = 4;
    string translation_vendor = 5;
    string due_date = 6;
}

message CreateTranslationResponse {
    CmTranslation data = 1;
}

message CmTranslation {
    string id = 1;
    string tenant_id = 2;
    string source_content_id = 3;
    string translated_content_id = 4;
    string target_language = 5;
    string translation_status = 6;
    string translator_id = 7;
    string translation_vendor = 8;
    string assigned_at = 9;
    string due_date = 10;
    string completed_at = 11;
    double quality_score = 12;
    int32 word_count = 13;
    string created_at = 14;
}

// Workflow messages
message ApproveWorkflowRequest {
    string tenant_id = 1;
    string id = 2;
    string comment = 3;
    bool approved = 4;
}

message ApproveWorkflowResponse {
    string id = 1;
    string status = 2;
    int32 current_stage = 3;
    int32 total_stages = 4;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | cm_folders | -- |
| V002 | cm_content_items | V001 |
| V003 | cm_content_versions | V002 |
| V004 | cm_approval_workflows | V002 |
| V005 | cm_content_translations | V002 |

---

## 7. Business Rules

1. **Version Control**: Every content change creates a new version; previous versions are never deleted
2. **Permission Inheritance**: Folders inherit permissions from parent unless explicitly overridden
3. **Approval Required**: Content must pass approval workflow before publishing
4. **Translation Sync**: Source content updates trigger re-translation notifications for dependent languages
5. **Checksum Validation**: File integrity verified via checksums on upload and retrieval
6. **Expiry Management**: Content with expiry dates auto-archived after expiration
7. **Concurrent Editing**: Optimistic locking via version field prevents conflicting updates

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| identity-service | `GetUser` / `CheckPermission` | User authentication and role-based permissions |
| workflow-service | `SubmitApproval` | Approval workflow orchestration |
| document-service | `StoreFile` / `RetrieveFile` | Document storage and retrieval |
| notification-service | `SendNotification` | Approval and review notifications |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| eloqua-service | `GetContentItem` | Email template content management |
| responsys-service | `GetContentItem` | Cross-channel message content |
| cx-advertising-service | `GetContentItem` | Creative asset management |
| cx-analytics-service | `GetContentMetrics` | Content engagement analytics |
