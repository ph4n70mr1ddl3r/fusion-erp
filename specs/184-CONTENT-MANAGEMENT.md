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
| POST | `/content-management/v1/items` | Create content item |
| GET | `/content-management/v1/items` | List content items |
| GET | `/content-management/v1/items/{id}` | Get content item details |
| PUT | `/content-management/v1/items/{id}` | Update content item |
| DELETE | `/content-management/v1/items/{id}` | Delete content item |
| POST | `/content-management/v1/items/{id}/publish` | Publish content |
| POST | `/content-management/v1/items/{id}/archive` | Archive content |
| GET | `/content-management/v1/items/search` | Search content |

### 3.2 Folders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/content-management/v1/folders` | Create folder |
| GET | `/content-management/v1/folders` | List folders |
| GET | `/content-management/v1/folders/{id}` | Get folder details |
| PUT | `/content-management/v1/folders/{id}` | Update folder |
| DELETE | `/content-management/v1/folders/{id}` | Delete folder |
| PUT | `/content-management/v1/folders/{id}/permissions` | Update permissions |
| GET | `/content-management/v1/folders/{id}/contents` | List folder contents |

### 3.3 Versioning
| Method | Path | Description |
|--------|------|-------------|
| POST | `/content-management/v1/items/{id}/versions` | Create new version |
| GET | `/content-management/v1/items/{id}/versions` | List versions |
| GET | `/content-management/v1/versions/{id}` | Get version details |
| POST | `/content-management/v1/versions/{id}/restore` | Restore version |
| GET | `/content-management/v1/versions/compare` | Compare two versions |

### 3.4 Workflows
| Method | Path | Description |
|--------|------|-------------|
| POST | `/content-management/v1/workflows` | Create approval workflow |
| GET | `/content-management/v1/workflows` | List workflows |
| GET | `/content-management/v1/workflows/{id}` | Get workflow details |
| POST | `/content-management/v1/workflows/{id}/approve` | Approve current stage |
| POST | `/content-management/v1/workflows/{id}/reject` | Reject workflow |
| GET | `/content-management/v1/workflows/pending` | List pending approvals |

### 3.5 Translations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/content-management/v1/translations` | Create translation request |
| GET | `/content-management/v1/translations` | List translations |
| GET | `/content-management/v1/translations/{id}` | Get translation details |
| PUT | `/content-management/v1/translations/{id}` | Update translation |
| POST | `/content-management/v1/translations/{id}/complete` | Complete translation |
| GET | `/content-management/v1/items/{id}/translations` | Get all translations for item |

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

## 5. Business Rules

1. **Version Control**: Every content change creates a new version; previous versions are never deleted
2. **Permission Inheritance**: Folders inherit permissions from parent unless explicitly overridden
3. **Approval Required**: Content must pass approval workflow before publishing
4. **Translation Sync**: Source content updates trigger re-translation notifications for dependent languages
5. **Checksum Validation**: File integrity verified via checksums on upload and retrieval
6. **Expiry Management**: Content with expiry dates auto-archived after expiration
7. **Concurrent Editing**: Optimistic locking via version field prevents conflicting updates

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Identity (05) | User authentication and role-based permissions |
| Workflow (16) | Approval workflow orchestration |
| Document Management (29) | Document storage and retrieval |
| Eloqua (181) | Email template content management |
| Responsys (182) | Cross-channel message content |
| CX Advertising (183) | Creative asset management |
| CX Analytics (131) | Content engagement analytics |
| Notification Center (165) | Approval and review notifications |
