# 86 - Narrative Reporting Service Specification

## 1. Domain Overview

Narrative Reporting provides management and regulatory report authoring with structured report packages (SEC filings, board reports, annual reports), section-based content management with embedded financial data, review and approval workflows (draft-review-approve), XBRL taxonomy tagging for regulatory compliance, embedded visualization support, reusable report templates, frozen snapshot versioning, and disclosure checklist management. Supports collaborative authoring with role-based section assignment, variable interpolation for dynamic financial data, and automated XBRL instance document generation. Integrates with GL for financial data, EPM for planning data, and REPORTING for chart and visualization rendering.

**Bounded Context:** Narrative & Regulatory Reporting
**Service Name:** `narrative-service`
**Database:** `data/narrative.db`
**HTTP Port:** 8118 | **gRPC Port:** 9118

---

## 2. Database Schema

### 2.1 Report Packages
```sql
CREATE TABLE narrative_packages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_name TEXT NOT NULL,
    package_type TEXT NOT NULL
        CHECK(package_type IN ('10K','10Q','8K','BOARD_REPORT','ANNUAL_REPORT','QUARTERLY_REPORT','INTERNAL_MEMO','REGULATORY_FILING','OTHER')),
    fiscal_year TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('Q1','Q2','Q3','Q4','ANNUAL','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','APPROVED','PUBLISHED','ARCHIVED')),
    owner_id TEXT NOT NULL,
    due_date TEXT,
    published_at TEXT,
    description TEXT,
    tags TEXT,                                     -- JSON array of tags

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, package_name, fiscal_year, period_type)
);

CREATE INDEX idx_packages_tenant_type ON narrative_packages(tenant_id, package_type);
CREATE INDEX idx_packages_tenant_status ON narrative_packages(tenant_id, status);
CREATE INDEX idx_packages_tenant_year ON narrative_packages(tenant_id, fiscal_year);
CREATE INDEX idx_packages_tenant_owner ON narrative_packages(tenant_id, owner_id);
CREATE INDEX idx_packages_tenant_due ON narrative_packages(tenant_id, due_date);
```

### 2.2 Report Sections
```sql
CREATE TABLE narrative_sections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_id TEXT NOT NULL,
    section_title TEXT NOT NULL,
    section_number TEXT NOT NULL,
    parent_section_id TEXT,
    section_type TEXT NOT NULL
        CHECK(section_type IN ('COVER','TABLE_OF_CONTENTS','EXECUTIVE_SUMMARY','FINANCIAL_STATEMENTS','NOTES','MDNA','RISK_FACTORS','NARRATIVE','APPENDIX','CUSTOM')),
    content_html TEXT NOT NULL DEFAULT '',
    content_plain TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','CHANGES_REQUESTED','APPROVED','LOCKED')),
    assigned_author_id TEXT,
    assigned_reviewer_id TEXT,
    word_count INTEGER NOT NULL DEFAULT 0,
    last_edited_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (package_id) REFERENCES narrative_packages(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, package_id, section_number)
);

CREATE INDEX idx_sections_tenant_package ON narrative_sections(tenant_id, package_id);
CREATE INDEX idx_sections_tenant_status ON narrative_sections(tenant_id, status);
CREATE INDEX idx_sections_tenant_type ON narrative_sections(tenant_id, section_type);
CREATE INDEX idx_sections_tenant_author ON narrative_sections(tenant_id, assigned_author_id);
CREATE INDEX idx_sections_tenant_parent ON narrative_sections(tenant_id, parent_section_id);
```

### 2.3 Report Data Sources
```sql
CREATE TABLE narrative_data_sources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    section_id TEXT NOT NULL,
    source_name TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('GL_BALANCE','GL_TRIAL','GL_JOURNAL','EPM_PLAN','EPM_FORECAST','KPI_METRIC','EXTERNAL_API','MANUAL_INPUT')),
    source_config TEXT NOT NULL,                   -- JSON: connection and query configuration
    refresh_frequency TEXT NOT NULL DEFAULT 'ON_DEMAND'
        CHECK(refresh_frequency IN ('ON_DEMAND','REAL_TIME','DAILY','WEEKLY','PERIOD_END')),
    last_refreshed_at TEXT,
    cache_ttl_seconds INTEGER NOT NULL DEFAULT 300,
    is_live INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (section_id) REFERENCES narrative_sections(id) ON DELETE CASCADE
);

CREATE INDEX idx_data_sources_tenant_section ON narrative_data_sources(tenant_id, section_id);
CREATE INDEX idx_data_sources_tenant_type ON narrative_data_sources(tenant_id, source_type);
```

### 2.4 Report Narratives
```sql
CREATE TABLE narrative_narratives (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    section_id TEXT NOT NULL,
    narrative_key TEXT NOT NULL,
    content_template TEXT NOT NULL,                -- Template with {{variable}} placeholders
    content_rendered TEXT,
    language_code TEXT NOT NULL DEFAULT 'en-US',
    variables TEXT,                                -- JSON: variable name to value mapping
    is_conditional INTEGER NOT NULL DEFAULT 0,
    condition_expression TEXT,                     -- Expression for conditional display
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (section_id) REFERENCES narrative_sections(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, section_id, narrative_key)
);

CREATE INDEX idx_narratives_tenant_section ON narrative_narratives(tenant_id, section_id);
CREATE INDEX idx_narratives_tenant_key ON narrative_narratives(tenant_id, narrative_key);
```

### 2.5 Report Charts
```sql
CREATE TABLE narrative_charts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    section_id TEXT NOT NULL,
    chart_title TEXT NOT NULL,
    chart_type TEXT NOT NULL
        CHECK(chart_type IN ('BAR','LINE','PIE','SCATTER','TABLE','WATERFALL','TREEMAP','HEATMAP','COMBO')),
    data_source_id TEXT,
    chart_config TEXT NOT NULL,                    -- JSON: chart rendering configuration
    data_payload TEXT,                             -- JSON: cached chart data
    width INTEGER NOT NULL DEFAULT 800,
    height INTEGER NOT NULL DEFAULT 400,
    alt_text TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (section_id) REFERENCES narrative_sections(id) ON DELETE CASCADE
);

CREATE INDEX idx_charts_tenant_section ON narrative_charts(tenant_id, section_id);
CREATE INDEX idx_charts_tenant_type ON narrative_charts(tenant_id, chart_type);
```

### 2.6 Review Workflows
```sql
CREATE TABLE narrative_review_workflows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_id TEXT NOT NULL,
    section_id TEXT,
    workflow_type TEXT NOT NULL
        CHECK(workflow_type IN ('SECTION_REVIEW','PACKAGE_REVIEW','LEGAL_REVIEW','AUDITOR_REVIEW','FINAL_APPROVAL')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','CHANGES_REQUESTED','APPROVED','REJECTED')),
    requested_by TEXT NOT NULL,
    assigned_to TEXT NOT NULL,
    requested_at TEXT NOT NULL,
    completed_at TEXT,
    comments TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    due_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (package_id) REFERENCES narrative_packages(id),
    FOREIGN KEY (section_id) REFERENCES narrative_sections(id)
);

CREATE INDEX idx_review_tenant_package ON narrative_review_workflows(tenant_id, package_id);
CREATE INDEX idx_review_tenant_status ON narrative_review_workflows(tenant_id, status);
CREATE INDEX idx_review_tenant_assigned ON narrative_review_workflows(tenant_id, assigned_to);
CREATE INDEX idx_review_tenant_type ON narrative_review_workflows(tenant_id, workflow_type);
CREATE INDEX idx_review_tenant_due ON narrative_review_workflows(tenant_id, due_date);
```

### 2.7 XBRL Tags
```sql
CREATE TABLE narrative_xbrl_tags (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    taxonomy_year TEXT NOT NULL,
    taxonomy_type TEXT NOT NULL
        CHECK(taxonomy_type IN ('US_GAAP','IFRS','DEI','CUSTOM')),
    tag_name TEXT NOT NULL,
    tag_label TEXT NOT NULL,
    balance_type TEXT
        CHECK(balance_type IN ('DEBIT','CREDIT','NONE')),
    period_type TEXT NOT NULL
        CHECK(period_type IN ('DURATION','INSTANT')),
    abstract_tag INTEGER NOT NULL DEFAULT 0,
    parent_tag_id TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, taxonomy_year, taxonomy_type, tag_name)
);

CREATE INDEX idx_xbrl_tenant_taxonomy ON narrative_xbrl_tags(tenant_id, taxonomy_year, taxonomy_type);
CREATE INDEX idx_xbrl_tenant_name ON narrative_xbrl_tags(tenant_id, tag_name);
CREATE INDEX idx_xbrl_tenant_parent ON narrative_xbrl_tags(tenant_id, parent_tag_id);
CREATE INDEX idx_xbrl_tenant_active ON narrative_xbrl_tags(tenant_id, is_active);
```

### 2.8 Report Snapshots
```sql
CREATE TABLE narrative_snapshots (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_id TEXT NOT NULL,
    snapshot_name TEXT NOT NULL,
    snapshot_type TEXT NOT NULL
        CHECK(snapshot_type IN ('AUTO_SAVE','MANUAL','PRE_REVIEW','POST_APPROVAL','PUBLISHED')),
    content_html TEXT NOT NULL,                    -- Full rendered HTML snapshot
    content_xbrl TEXT,                             -- XBRL instance document (if applicable)
    section_versions TEXT NOT NULL,                -- JSON: section_id -> version at snapshot time
    data_values TEXT,                              -- JSON: frozen data values at snapshot time
    total_word_count INTEGER NOT NULL DEFAULT 0,
    snapshot_hash TEXT NOT NULL,                   -- SHA-256 hash for integrity

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (package_id) REFERENCES narrative_packages(id),
    UNIQUE(tenant_id, package_id, snapshot_name)
);

CREATE INDEX idx_snapshots_tenant_package ON narrative_snapshots(tenant_id, package_id);
CREATE INDEX idx_snapshots_tenant_type ON narrative_snapshots(tenant_id, snapshot_type);
CREATE INDEX idx_snapshots_tenant_dates ON narrative_snapshots(tenant_id, created_at);
```

### 2.9 Report Templates
```sql
CREATE TABLE narrative_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_type TEXT NOT NULL
        CHECK(template_type IN ('10K','10Q','BOARD_REPORT','ANNUAL_REPORT','INTERNAL_MEMO','CUSTOM')),
    description TEXT,
    section_structure TEXT NOT NULL,               -- JSON: template section hierarchy
    default_narratives TEXT,                       -- JSON: default narrative templates per section
    default_styles TEXT,                           -- JSON: CSS/style configuration
    is_system INTEGER NOT NULL DEFAULT 0,
    usage_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_name)
);

CREATE INDEX idx_templates_tenant_type ON narrative_templates(tenant_id, template_type);
CREATE INDEX idx_templates_tenant_active ON narrative_templates(tenant_id, is_active);
CREATE INDEX idx_templates_tenant_system ON narrative_templates(tenant_id, is_system);
```

### 2.10 Disclosure Checklists
```sql
CREATE TABLE narrative_disclosure_checklists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NULL,
    checklist_name TEXT NOT NULL,
    regulation_type TEXT NOT NULL
        CHECK(regulation_type IN ('SEC','GAAP','IFRS','SOX','INDUSTRY_SPECIFIC','INTERNAL')),
    fiscal_year TEXT NOT NULL,
    package_id TEXT,
    is_complete INTEGER NOT NULL DEFAULT 0,
    completion_percent_real REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (package_id) REFERENCES narrative_packages(id)
);

CREATE INDEX idx_checklists_tenant_regulation ON narrative_disclosure_checklists(tenant_id, regulation_type);
CREATE INDEX idx_checklists_tenant_package ON narrative_disclosure_checklists(tenant_id, package_id);
CREATE INDEX idx_checklists_tenant_year ON narrative_disclosure_checklists(tenant_id, fiscal_year);
```

### 2.11 Disclosure Checklist Items
```sql
CREATE TABLE narrative_disclosure_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    checklist_id TEXT NOT NULL,
    item_number TEXT NOT NULL,
    requirement TEXT NOT NULL,
    regulation_reference TEXT,
    section_id TEXT,
    is_applicable INTEGER NOT NULL DEFAULT 1,
    is_addressed INTEGER NOT NULL DEFAULT 0,
    evidence_text TEXT,
    reviewed_by TEXT,
    reviewed_at TEXT,
    notes TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (checklist_id) REFERENCES narrative_disclosure_checklists(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, checklist_id, item_number)
);

CREATE INDEX idx_disclosure_items_tenant_checklist ON narrative_disclosure_items(tenant_id, checklist_id);
CREATE INDEX idx_disclosure_items_tenant_addressed ON narrative_disclosure_items(tenant_id, is_addressed);
```

---

## 3. REST API Endpoints

```
# Report Packages
GET/POST      /api/v1/narrative/packages                     Permission: narrative.packages.read/create
GET/PUT       /api/v1/narrative/packages/{id}                Permission: narrative.packages.read/update
PATCH         /api/v1/narrative/packages/{id}/archive        Permission: narrative.packages.update
DELETE        /api/v1/narrative/packages/{id}                Permission: narrative.packages.delete

# Report Sections
GET/POST      /api/v1/narrative/packages/{id}/sections       Permission: narrative.sections.read/create
GET/PUT       /api/v1/narrative/sections/{id}                Permission: narrative.sections.read/update
PATCH         /api/v1/narrative/sections/{id}/reorder        Permission: narrative.sections.update
DELETE        /api/v1/narrative/sections/{id}                Permission: narrative.sections.delete

# Data Sources
GET/POST      /api/v1/narrative/sections/{id}/data-sources   Permission: narrative.datasources.read/create
GET/PUT       /api/v1/narrative/data-sources/{id}            Permission: narrative.datasources.read/update
POST          /api/v1/narrative/data-sources/{id}/refresh    Permission: narrative.datasources.execute
DELETE        /api/v1/narrative/data-sources/{id}            Permission: narrative.datasources.delete

# Narratives
GET/POST      /api/v1/narrative/sections/{id}/narratives     Permission: narrative.narratives.read/create
GET/PUT       /api/v1/narrative/narratives/{id}              Permission: narrative.narratives.read/update
POST          /api/v1/narrative/narratives/{id}/render       Permission: narrative.narratives.execute
DELETE        /api/v1/narrative/narratives/{id}              Permission: narrative.narratives.delete

# Charts
GET/POST      /api/v1/narrative/sections/{id}/charts         Permission: narrative.charts.read/create
GET/PUT       /api/v1/narrative/charts/{id}                  Permission: narrative.charts.read/update
POST          /api/v1/narrative/charts/{id}/render           Permission: narrative.charts.execute
DELETE        /api/v1/narrative/charts/{id}                  Permission: narrative.charts.delete

# Review Workflows
POST          /api/v1/narrative/packages/{id}/submit-review  Permission: narrative.reviews.create
POST          /api/v1/narrative/sections/{id}/submit-review  Permission: narrative.reviews.create
GET           /api/v1/narrative/reviews/pending              Permission: narrative.reviews.read
GET/PUT       /api/v1/narrative/reviews/{id}                 Permission: narrative.reviews.read/update
POST          /api/v1/narrative/reviews/{id}/approve         Permission: narrative.reviews.approve
POST          /api/v1/narrative/reviews/{id}/reject          Permission: narrative.reviews.reject
POST          /api/v1/narrative/reviews/{id}/request-changes Permission: narrative.reviews.update

# XBRL
GET/POST      /api/v1/narrative/xbrl/tags                    Permission: narrative.xbrl.read/create
GET           /api/v1/narrative/xbrl/tags/search             Permission: narrative.xbrl.read
POST          /api/v1/narrative/packages/{id}/xbrl/generate  Permission: narrative.xbrl.execute
GET           /api/v1/narrative/packages/{id}/xbrl/instance  Permission: narrative.xbrl.read
POST          /api/v1/narrative/xbrl/validate                Permission: narrative.xbrl.execute

# Snapshots
POST          /api/v1/narrative/packages/{id}/snapshot       Permission: narrative.snapshots.create
GET           /api/v1/narrative/packages/{id}/snapshots      Permission: narrative.snapshots.read
GET           /api/v1/narrative/snapshots/{id}               Permission: narrative.snapshots.read
POST          /api/v1/narrative/snapshots/{id}/restore       Permission: narrative.snapshots.update

# Templates
GET/POST      /api/v1/narrative/templates                    Permission: narrative.templates.read/create
GET/PUT       /api/v1/narrative/templates/{id}               Permission: narrative.templates.read/update
POST          /api/v1/narrative/templates/{id}/instantiate   Permission: narrative.templates.execute
DELETE        /api/v1/narrative/templates/{id}               Permission: narrative.templates.delete

# Disclosure Checklists
GET/POST      /api/v1/narrative/checklists                   Permission: narrative.checklists.read/create
GET/PUT       /api/v1/narrative/checklists/{id}              Permission: narrative.checklists.read/update
GET/POST      /api/v1/narrative/checklists/{id}/items        Permission: narrative.checklists.read/create
PUT           /api/v1/narrative/checklists/items/{id}        Permission: narrative.checklists.update
GET           /api/v1/narrative/checklists/{id}/progress     Permission: narrative.checklists.read

# Publish
POST          /api/v1/narrative/packages/{id}/publish        Permission: narrative.packages.publish
GET           /api/v1/narrative/packages/{id}/export         Permission: narrative.packages.read
```

---

## 4. Business Rules

### 4.1 Report Package Management
1. A report package MUST have at least one section before it can be submitted for review
2. Package status transitions MUST follow: DRAFT -> IN_REVIEW -> APPROVED -> PUBLISHED -> ARCHIVED
3. Only the package owner or admin MAY change the package owner
4. Published packages are immutable; changes require a new version or a new package
5. Each package MUST have a unique combination of name, fiscal year, and period type
6. Archived packages retain all section content and snapshots for audit purposes

### 4.2 Section Authoring
1. Sections MUST maintain a hierarchical structure with parent_section_id for nesting
2. Section numbers MUST be auto-generated based on sort_order and hierarchy depth
3. Each section MUST track word count and last-edited timestamp
4. Locked sections cannot be edited; approval or admin action is required to unlock
5. Section content is stored as HTML with sanitized input (no script tags permitted)
6. Sections assigned to an author SHOULD notify the assignee upon creation or reassignment

### 4.3 Data Source Integration
1. Data sources MUST validate connectivity before being saved
2. Live data sources MUST refresh according to their refresh_frequency setting
3. Cached data MUST respect the cache_ttl_seconds parameter
4. Data source configuration MUST be stored as encrypted JSON for credential safety
5. Manual input data sources SHOULD require approval before being used in published reports
6. Data refresh failures MUST trigger notifications to the section author and package owner

### 4.4 Narrative Variables
1. Variable placeholders MUST use the `{{variable_name}}` syntax
2. Variables MUST be resolved at render time using the current data source values
3. Conditional narratives MUST evaluate their condition_expression before rendering
4. Rendered content MUST be stored alongside the template for audit comparison
5. Missing variable values SHOULD produce a warning in the rendered output
6. Variables SHOULD support formatting directives (e.g., `{{amount|currency}}`)

### 4.5 Review Workflow
1. A review request MUST specify the assigned reviewer and workflow type
2. Reviewers MAY approve, reject, or request changes on sections or packages
3. Changes requested during review MUST reset the section status to DRAFT
4. All sections MUST be APPROVED before the package can transition to PUBLISHED
5. Review comments MUST be retained for audit trail purposes
6. Package-level approval requires all child sections to have APPROVED status

### 4.6 XBRL Generation
1. XBRL instance documents MUST conform to the selected taxonomy version
2. All financial figures in XBRL MUST reference a valid XBRL tag with correct balance type
3. XBRL validation MUST check for completeness (all required tags present) and accuracy (correct period types)
4. Generated XBRL MUST include context elements for reporting entity and period
5. XBRL tagging errors MUST be reported with specific section and data element references
6. DEI (Document and Entity Information) tags MUST be completed before XBRL generation

### 4.7 Snapshot Management
1. Snapshots MUST capture the full rendered HTML content at the time of creation
2. Snapshot hashes MUST be computed over the complete content for integrity verification
3. Published packages MUST automatically create a PUBLISHED snapshot
4. Pre-review snapshots SHOULD be auto-created when review is requested
5. Snapshot restoration MUST create a new version of affected sections
6. Snapshot content MUST NOT be modified after creation

### 4.8 Disclosure Checklists
1. Checklist items MUST be marked as applicable or not applicable for the current period
2. Applicable items MUST be addressed (evidence provided) before checklist completion
3. Completion percentage MUST be auto-calculated based on addressed/applicable ratio
4. Regulation references MUST link to specific paragraphs or sections of the standard
5. Unaddressed items at filing time MUST produce a warning and require sign-off
6. Checklist templates SHOULD be versioned to support changing regulatory requirements

### 4.9 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `narrative.package.created` | New report package created | Workflow, Notification |
| `narrative.section.drafted` | Section content saved | Notification |
| `narrative.review.requested` | Review workflow initiated | Workflow, Notification |
| `narrative.review.approved` | Section or package approved | Notification |
| `narrative.package.approved` | All sections approved | Workflow, Notification |
| `narrative.xbrl.generated` | XBRL instance document created | Document, Notification |
| `narrative.package.published` | Report package published | Notification, Document |
| `narrative.snapshot.created` | Snapshot taken | Document |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.narrative.v1;

service NarrativeService {
    rpc GetPackage(GetPackageRequest) returns (GetPackageResponse);
    rpc CreateSection(CreateSectionRequest) returns (CreateSectionResponse);
    rpc RenderNarrative(RenderNarrativeRequest) returns (RenderNarrativeResponse);
    rpc GenerateXBRL(GenerateXBRLRequest) returns (GenerateXBRLResponse);
    rpc GetSnapshot(GetSnapshotRequest) returns (GetSnapshotResponse);
    rpc GetDisclosureStatus(GetDisclosureStatusRequest) returns (GetDisclosureStatusResponse);
}

// --- Report Package ---
message ReportPackage {
    string id = 1; string tenant_id = 2; string package_name = 3; string package_type = 4;
    string fiscal_year = 5; string period_type = 6; string status = 7; string owner_id = 8;
    string due_date = 9; string published_at = 10; string description = 11; string tags = 12;
    string created_at = 13; string updated_at = 14; string created_by = 15; string updated_by = 16;
    int32 version = 17; bool is_active = 18;
}

// --- Report Section ---
message ReportSection {
    string id = 1; string tenant_id = 2; string package_id = 3; string section_title = 4;
    string section_number = 5; string parent_section_id = 6; string section_type = 7;
    string content_html = 8; string content_plain = 9; int32 sort_order = 10; string status = 11;
    string assigned_author_id = 12; string assigned_reviewer_id = 13; int32 word_count = 14;
    string last_edited_at = 15;
    string created_at = 16; string updated_at = 17; string created_by = 18; string updated_by = 19;
    int32 version = 20; bool is_active = 21;
}

// --- Data Source ---
message ReportDataSource {
    string id = 1; string tenant_id = 2; string section_id = 3; string source_name = 4;
    string source_type = 5; string source_config = 6; string refresh_frequency = 7;
    string last_refreshed_at = 8; int32 cache_ttl_seconds = 9; bool is_live = 10;
    string created_at = 11; string updated_at = 12; string created_by = 13; string updated_by = 14;
    int32 version = 15;
}

// --- Narrative ---
message ReportNarrative {
    string id = 1; string tenant_id = 2; string section_id = 3; string narrative_key = 4;
    string content_template = 5; string content_rendered = 6; string language_code = 7;
    string variables = 8; bool is_conditional = 9; string condition_expression = 10;
    int32 sort_order = 11;
    string created_at = 12; string updated_at = 13; string created_by = 14; string updated_by = 15;
    int32 version = 16;
}

// --- Report Chart ---
message ReportChart {
    string id = 1; string tenant_id = 2; string section_id = 3; string chart_title = 4;
    string chart_type = 5; string data_source_id = 6; string chart_config = 7;
    string data_payload = 8; int32 width = 9; int32 height = 10; string alt_text = 11;
    int32 sort_order = 12;
    string created_at = 13; string updated_at = 14; string created_by = 15; string updated_by = 16;
    int32 version = 17;
}

// --- Review Workflow ---
message ReviewWorkflow {
    string id = 1; string tenant_id = 2; string package_id = 3; string section_id = 4;
    string workflow_type = 5; string status = 6; string requested_by = 7; string assigned_to = 8;
    string requested_at = 9; string completed_at = 10; string comments = 11;
    string priority = 12; string due_date = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- XBRL Tag ---
message XbrlTag {
    string id = 1; string tenant_id = 2; string taxonomy_year = 3; string taxonomy_type = 4;
    string tag_name = 5; string tag_label = 6; string balance_type = 7; string period_type = 8;
    bool abstract_tag = 9; string parent_tag_id = 10; string description = 11;
    string created_at = 12; string updated_at = 13; string created_by = 14; string updated_by = 15;
    int32 version = 16; bool is_active = 17;
}

// --- Snapshot ---
message ReportSnapshot {
    string id = 1; string tenant_id = 2; string package_id = 3; string snapshot_name = 4;
    string snapshot_type = 5; string content_html = 6; string content_xbrl = 7;
    string section_versions = 8; string data_values = 9; int32 total_word_count = 10;
    string snapshot_hash = 11;
    string created_at = 12; string updated_at = 13; string created_by = 14; string updated_by = 15;
    int32 version = 16;
}

// --- Template ---
message ReportTemplate {
    string id = 1; string tenant_id = 2; string template_name = 3; string template_type = 4;
    string description = 5; string section_structure = 6; string default_narratives = 7;
    string default_styles = 8; bool is_system = 9; int32 usage_count = 10;
    string created_at = 11; string updated_at = 12; string created_by = 13; string updated_by = 14;
    int32 version = 15; bool is_active = 16;
}

// --- Disclosure Checklist ---
message DisclosureChecklist {
    string id = 1; string tenant_id = 2; string checklist_name = 3; string regulation_type = 4;
    string fiscal_year = 5; string package_id = 6; bool is_complete = 7;
    double completion_percent_real = 8;
    string created_at = 9; string updated_at = 10; string created_by = 11; string updated_by = 12;
    int32 version = 13; bool is_active = 14;
}

// --- Disclosure Checklist Item ---
message DisclosureChecklistItem {
    string id = 1; string tenant_id = 2; string checklist_id = 3; string item_number = 4;
    string requirement = 5; string regulation_reference = 6; string section_id = 7;
    bool is_applicable = 8; bool is_addressed = 9; string evidence_text = 10;
    string reviewed_by = 11; string reviewed_at = 12; string notes = 13; int32 sort_order = 14;
    string created_at = 15; string updated_at = 16; string created_by = 17; string updated_by = 18;
}

// --- RPC Request/Response Messages ---
message GetPackageRequest { string tenant_id = 1; string id = 2; }
message GetPackageResponse { ReportPackage data = 1; }

message CreateSectionRequest {
    string tenant_id = 1; string package_id = 2; string section_title = 3;
    string section_type = 4; string parent_section_id = 5; int32 sort_order = 6;
}
message CreateSectionResponse { ReportSection data = 1; }

message RenderNarrativeRequest { string tenant_id = 1; string narrative_id = 2; string section_id = 3; }
message RenderNarrativeResponse { ReportNarrative data = 1; }

message GenerateXBRLRequest { string tenant_id = 1; string package_id = 2; string taxonomy_year = 3; string taxonomy_type = 4; }
message GenerateXBRLResponse { string content_xbrl = 1; ReportSnapshot snapshot = 2; }

message GetSnapshotRequest { string tenant_id = 1; string snapshot_id = 2; }
message GetSnapshotResponse { ReportSnapshot data = 1; }

message GetDisclosureStatusRequest { string tenant_id = 1; string checklist_id = 2; }
message GetDisclosureStatusResponse { DisclosureChecklist checklist = 1; repeated DisclosureChecklistItem items = 2; }

message ListPackagesRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListPackagesResponse { repeated ReportPackage items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListSectionsRequest { string tenant_id = 1; string package_id = 2; int32 page_size = 3; string page_token = 4; }
message ListSectionsResponse { repeated ReportSection items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListSnapshotsRequest { string tenant_id = 1; string package_id = 2; int32 page_size = 3; string page_token = 4; }
message ListSnapshotsResponse { repeated ReportSnapshot items = 1; int32 total_count = 2; string next_page_token = 3; }
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **GL:** Financial statement data (trial balance, income statement, balance sheet) for data source bindings
- **EPM:** Planning and forecast data for variance analysis narratives
- **REPORTING:** Chart rendering engine for embedded visualizations
- **Auth:** User identity for author/reviewer/approver assignment and permissions
- **Workflow:** Approval routing, task assignment, and escalation rules
- **Document:** Supporting document storage and retrieval for disclosure evidence

### 6.2 Data Published To
- **Document:** Generated report exports (PDF, HTML, XBRL instance documents)
- **Workflow:** Review and approval task creation, deadline monitoring
- **Notification:** Review requests, approval notifications, deadline reminders, publishing alerts
- **REPORTING:** Report metadata for unified reporting dashboards

---

## 7. Migrations

1. V001: `narrative_packages`
2. V002: `narrative_sections`
3. V003: `narrative_data_sources`
4. V004: `narrative_narratives`
5. V005: `narrative_charts`
6. V006: `narrative_review_workflows`
7. V007: `narrative_xbrl_tags`
8. V008: `narrative_snapshots`
9. V009: `narrative_templates`
10. V010: `narrative_disclosure_checklists`
11. V011: `narrative_disclosure_items`
12. V012: Triggers for `updated_at`
