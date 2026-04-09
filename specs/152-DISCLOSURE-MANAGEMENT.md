# 152 - Disclosure Management Service Specification

## 1. Domain Overview

Disclosure Management streamlines the creation, review, approval, and filing of regulatory disclosures including SEC filings (10-K, 10-Q, 8-K), statutory reports, management discussion & analysis (MD&A), and other regulatory documents. Provides collaborative document authoring with XBRL tagging, audit trail, version control, role-based workflows, and electronic filing integration. Supports narrative financial reporting with embedded data links to source financial data for automatic updates. Integrates with Financial Consolidation, EPM, Narrative Reporting, and Account Reconciliation.

**Bounded Context:** Regulatory Disclosure & Filing Management
**Service Name:** `disclosure-mgmt-service`
**Database:** `data/disclosure_mgmt.db`
**HTTP Port:** 8170 | **gRPC Port:** 9170

---

## 2. Database Schema

### 2.1 Disclosure Packages
```sql
CREATE TABLE dm_disclosure_packages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_number TEXT NOT NULL,           -- DIS-2024-00001
    package_name TEXT NOT NULL,
    disclosure_type TEXT NOT NULL CHECK(disclosure_type IN (
        '10K','10Q','8K','ANNUAL_REPORT','QUARTERLY_REPORT',
        'STATUTORY','REGULATORY','INTERNAL_MD&A','BOARD_REPORT','CUSTOM'
    )),
    fiscal_period TEXT NOT NULL,            -- "2024-Q4", "2024"
    filing_deadline TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','REVISION','APPROVED','FILED','ARCHIVED')),
    lead_author_id TEXT NOT NULL,
    reviewers TEXT NOT NULL,                -- JSON array of reviewer user IDs
    approver_id TEXT NOT NULL,
    regulatory_body TEXT,                   -- SEC, FCA, etc.
    filing_format TEXT CHECK(filing_format IN ('XBRL','HTML','PDF','EDGAR','ESEF')),
    contains_xbrl INTEGER NOT NULL DEFAULT 0,
    confidential INTEGER NOT NULL DEFAULT 0,
    version_number INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, package_number)
);

CREATE INDEX idx_dm_disc_tenant ON dm_disclosure_packages(tenant_id, disclosure_type, status);
CREATE INDEX idx_dm_disc_deadline ON dm_disclosure_packages(tenant_id, filing_deadline);
```

### 2.2 Disclosure Sections
```sql
CREATE TABLE dm_disclosure_sections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_id TEXT NOT NULL,
    section_number TEXT NOT NULL,           -- "1.1", "1.2.3"
    section_title TEXT NOT NULL,
    section_type TEXT NOT NULL CHECK(section_type IN (
        'NARRATIVE','FINANCIAL_STATEMENT','TABLE','NOTE','XHTML','XHTML_TABLE','IMAGE','REFERENCE'
    )),
    content TEXT,                           -- Rich text / XHTML content
    data_links TEXT,                        -- JSON: links to financial data sources
    xbrl_tags TEXT,                         -- JSON: XBRL taxonomy mappings
    assigned_author_id TEXT,
    review_status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(review_status IN ('NOT_STARTED','DRAFT','SUBMITTED_FOR_REVIEW','REVIEWED','APPROVED','NEEDS_REVISION')),
    reviewer_id TEXT,
    review_comments TEXT,
    last_reviewed_at TEXT,
    word_count INTEGER NOT NULL DEFAULT 0,
    has_changes_since_review INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (package_id) REFERENCES dm_disclosure_packages(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, package_id, section_number)
);

CREATE INDEX idx_dm_section_package ON dm_disclosure_sections(package_id, section_number);
CREATE INDEX idx_dm_section_review ON dm_disclosure_sections(tenant_id, review_status);
```

### 2.3 XBRL Tag Mappings
```sql
CREATE TABLE dm_xbrl_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    taxonomy TEXT NOT NULL,                 -- e.g., "us-gaap/2024"
    element_name TEXT NOT NULL,
    element_label TEXT NOT NULL,
    data_type TEXT NOT NULL,
    balance_type TEXT CHECK(balance_type IN ('DEBIT','CREDIT','NONE')),
    period_type TEXT NOT NULL CHECK(period_type IN ('DURATION','INSTANT')),
    gl_account_mapping TEXT,               -- JSON: mapped GL account codes
    custom_mapping INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, taxonomy, element_name)
);

CREATE INDEX idx_dm_xbrl_taxonomy ON dm_xbrl_mappings(tenant_id, taxonomy);
```

### 2.4 Review Audit Trail
```sql
CREATE TABLE dm_review_audit (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_id TEXT NOT NULL,
    section_id TEXT,
    action TEXT NOT NULL CHECK(action IN (
        'SECTION_CREATED','CONTENT_EDITED','DATA_LINKED','REVIEW_REQUESTED',
        'COMMENT_ADDED','REVISION_REQUESTED','SECTION_APPROVED','SECTION_REJECTED',
        'PACKAGE_SUBMITTED','PACKAGE_APPROVED','PACKAGE_REJECTED','PACKAGE_FILED',
        'XBRL_TAGGED','VERSION_CREATED'
    )),
    user_id TEXT NOT NULL,
    user_name TEXT NOT NULL,
    old_value TEXT,                         -- JSON: previous state
    new_value TEXT,                         -- JSON: new state
    comment TEXT,
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (package_id) REFERENCES dm_disclosure_packages(id) ON DELETE CASCADE
);

CREATE INDEX idx_dm_audit_package ON dm_review_audit(package_id, timestamp DESC);
CREATE INDEX idx_dm_audit_user ON dm_review_audit(tenant_id, user_id, timestamp DESC);
```

### 2.5 Filing Submissions
```sql
CREATE TABLE dm_filing_submissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    package_id TEXT NOT NULL,
    submission_number TEXT NOT NULL,        -- FIL-2024-00001
    filing_type TEXT NOT NULL,
    regulatory_body TEXT NOT NULL,
    filing_format TEXT NOT NULL,
    generated_document_path TEXT NOT NULL,
    xbrl_instance_path TEXT,
    filing_status TEXT NOT NULL DEFAULT 'PREPARED'
        CHECK(filing_status IN ('PREPARED','SUBMITTED','ACCEPTED','REJECTED','AMENDED')),
    submission_timestamp TEXT,
    acceptance_timestamp TEXT,
    confirmation_number TEXT,
    rejection_reason TEXT,
    filed_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (package_id) REFERENCES dm_disclosure_packages(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, submission_number)
);

CREATE INDEX idx_dm_filing_tenant ON dm_filing_submissions(tenant_id, filing_status);
CREATE INDEX idx_dm_filing_package ON dm_filing_submissions(package_id);
```

---

## 3. API Endpoints

### 3.1 Disclosure Packages
| Method | Path | Description |
|--------|------|-------------|
| POST | `/disclosure-mgmt/v1/packages` | Create disclosure package |
| GET | `/disclosure-mgmt/v1/packages` | List packages |
| GET | `/disclosure-mgmt/v1/packages/{id}` | Get package details |
| PUT | `/disclosure-mgmt/v1/packages/{id}` | Update package |
| POST | `/disclosure-mgmt/v1/packages/{id}/submit` | Submit for review |
| POST | `/disclosure-mgmt/v1/packages/{id}/approve` | Approve package |
| POST | `/disclosure-mgmt/v1/packages/{id}/reject` | Request revision |

### 3.2 Sections & Content
| Method | Path | Description |
|--------|------|-------------|
| POST | `/disclosure-mgmt/v1/packages/{id}/sections` | Add section |
| GET | `/disclosure-mgmt/v1/packages/{id}/sections` | List sections |
| PUT | `/disclosure-mgmt/v1/sections/{id}` | Update section content |
| POST | `/disclosure-mgmt/v1/sections/{id}/submit-review` | Submit for review |
| POST | `/disclosure-mgmt/v1/sections/{id}/review` | Review section |
| POST | `/disclosure-mgmt/v1/sections/{id}/refresh-data` | Refresh linked data |

### 3.3 XBRL
| Method | Path | Description |
|--------|------|-------------|
| GET | `/disclosure-mgmt/v1/xbrl/taxonomies` | List available taxonomies |
| GET | `/disclosure-mgmt/v1/xbrl/elements` | Search XBRL elements |
| POST | `/disclosure-mgmt/v1/sections/{id}/xbrl-tag` | Apply XBRL tag |
| POST | `/disclosure-mgmt/v1/packages/{id}/xbrl-validate` | Validate XBRL tagging |

### 3.4 Filing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/disclosure-mgmt/v1/packages/{id}/generate` | Generate filing document |
| POST | `/disclosure-mgmt/v1/packages/{id}/file` | Submit filing |
| GET | `/disclosure-mgmt/v1/filings` | List filings |
| GET | `/disclosure-mgmt/v1/filings/{id}` | Get filing status |

### 3.5 Audit Trail
| Method | Path | Description |
|--------|------|-------------|
| GET | `/disclosure-mgmt/v1/packages/{id}/audit-trail` | Get audit trail |
| GET | `/disclosure-mgmt/v1/packages/{id}/versions` | List versions |
| POST | `/disclosure-mgmt/v1/packages/{id}/versions/{version}/restore` | Restore version |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `disclosure.package.created` | `{ package_id, type, deadline }` | Disclosure package created |
| `disclosure.section.reviewed` | `{ section_id, decision, reviewer }` | Section reviewed |
| `disclosure.package.approved` | `{ package_id, approver }` | Package fully approved |
| `disclosure.filing.submitted` | `{ filing_id, regulatory_body }` | Filing submitted |
| `disclosure.filing.accepted` | `{ filing_id, confirmation_number }` | Filing accepted by regulator |
| `disclosure.filing.rejected` | `{ filing_id, reason }` | Filing rejected |
| `disclosure.deadline.approaching` | `{ package_id, days_remaining }` | Filing deadline approaching |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `consolidation.completed` | Financial Consolidation | Refresh financial data links |
| `reconciliation.completed` | Account Reconciliation | Update reconciliation data |
| `period.closed` | EPM | Trigger disclosure creation |

---


---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.disclosure.v1;

service DisclosureService {
    rpc GetPackage(GetPackageRequest) returns (GetPackageResponse);
    rpc CreatePackage(CreatePackageRequest) returns (CreatePackageResponse);
    rpc GenerateXBRL(GenerateXBRLRequest) returns (GenerateXBRLResponse);
    rpc SubmitFiling(SubmitFilingRequest) returns (SubmitFilingResponse);
}

message DisclosurePackage { string id = 1; string tenant_id = 2; string name = 3; string period = 4; string package_type = 5; string status = 6; string created_at = 7; string updated_at = 8; }
message DisclosureSection { string id = 1; string tenant_id = 2; string package_id = 3; string section_name = 4; string content = 5; string status = 6; string created_at = 7; string updated_at = 8; }

message GetPackageRequest { string tenant_id = 1; string id = 2; }
message GetPackageResponse { DisclosurePackage data = 1; }
message CreatePackageRequest { string tenant_id = 1; string name = 2; string period = 3; string package_type = 4; }
message CreatePackageResponse { DisclosurePackage data = 1; }
message GenerateXBRLRequest { string tenant_id = 1; string package_id = 2; }
message GenerateXBRLResponse { string xbrl_content = 1; string filing_url = 2; }
message SubmitFilingRequest { string tenant_id = 1; string package_id = 2; string filing_type = 3; }
message SubmitFilingResponse { string submission_id = 1; string status = 2; }
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | dm_disclosure_packages | — |
| V002 | dm_disclosure_sections | V001 |
| V003 | dm_xbrl_mappings | V002 |
| V004 | dm_review_audit | V003 |
| V005 | dm_filing_submissions | V004 |

---

## 7. Business Rules

1. **Review Workflow**: All sections must be reviewed and approved before package submission
2. **Data Linking**: Linked financial data auto-refreshes when source data changes
3. **XBRL Validation**: Mandatory XBRL validation before filing submission
4. **Deadline Monitoring**: Auto-escalate 30, 14, and 7 days before filing deadline
5. **Version Control**: Every content change creates audit entry; named versions for milestones
6. **Confidentiality**: Confidential packages require additional approval layer
7. **Filing Lockdown**: Approved packages locked for editing; changes require new version
8. **Amendment Support**: Filed documents can be amended with full audit trail

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Financial Consolidation (100) | Consolidated financial data |
| Account Reconciliation (101) | Reconciled account balances |
| EPM (33) | Period close status, actuals data |
| Narrative Reporting (86) | Narrative report content |
| General Ledger (06) | GL balances for financial statements |
| Tax Reporting (85) | Tax disclosure data |
| Document Management (29) | Document storage and versioning |
| Workflow (16) | Review and approval workflows |
| Reporting (17) | Report generation |
