# 139 - Supplier Management Service Specification

## 1. Domain Overview

Supplier Management provides comprehensive supplier lifecycle management including supplier qualification, onboarding, evaluation, risk assessment, and performance scoring. Supports qualification programs with configurable criteria, supplier segmentation, risk tiering, real-time compliance monitoring, and corrective action workflows. Integrates with Procurement for approved supplier lists, Accounts Payable for financial scoring, Quality Management for defect tracking, and Supplier Portal for self-service registration.

**Bounded Context:** Supplier Lifecycle & Performance Management
**Service Name:** `supplier-mgmt-service`
**Database:** `data/supplier_mgmt.db`
**HTTP Port:** 8219 | **gRPC Port:** 9219

---

## 2. Database Schema

### 2.1 Supplier Qualification Programs
```sql
CREATE TABLE sm_qualification_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_name TEXT NOT NULL,
    description TEXT,
    program_type TEXT NOT NULL CHECK(program_type IN ('INITIAL_QUALIFICATION','REQUALIFICATION','CATEGORY_SPECIFIC','COMPLIANCE','CERTIFICATION')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','SUSPENDED','ARCHIVED')),
    evaluation_criteria TEXT NOT NULL,       -- JSON: [{ "criterion": "...", "weight": 20, "scoring_model": "RANGE" }]
    pass_threshold_score REAL NOT NULL DEFAULT 70.0,
    validity_period_days INTEGER NOT NULL DEFAULT 365,
    requires_documentation INTEGER NOT NULL DEFAULT 1,
    requires_site_visit INTEGER NOT NULL DEFAULT 0,
    auto_approve_on_pass INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_code)
);

CREATE INDEX idx_sm_qual_prog_tenant ON sm_qualification_programs(tenant_id, status);
```

### 2.2 Supplier Assessments
```sql
CREATE TABLE sm_supplier_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    assessment_number TEXT NOT NULL,         -- SA-2024-00001
    assessment_type TEXT NOT NULL CHECK(assessment_type IN ('INITIAL','PERIODIC','TRIGGERED','SELF_ASSESSMENT')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED','EXPIRED')),
    assessor_id TEXT,
    submitted_at TEXT,
    reviewed_at TEXT,
    overall_score REAL NOT NULL DEFAULT 0,
    max_possible_score REAL NOT NULL DEFAULT 100,
    risk_level TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    qualification_decision TEXT CHECK(qualification_decision IN ('QUALIFIED','CONDITIONAL','DISQUALIFIED','PENDING_ACTION')),
    validity_expiry TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES sm_qualification_programs(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, assessment_number)
);

CREATE INDEX idx_sm_assess_supplier ON sm_supplier_assessments(tenant_id, supplier_id, created_at DESC);
CREATE INDEX idx_sm_assess_status ON sm_supplier_assessments(tenant_id, status);
CREATE INDEX idx_sm_assess_expiry ON sm_supplier_assessments(tenant_id, validity_expiry);
```

### 2.3 Assessment Responses
```sql
CREATE TABLE sm_assessment_responses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    assessment_id TEXT NOT NULL,
    criterion_code TEXT NOT NULL,
    criterion_name TEXT NOT NULL,
    response_value TEXT,
    score REAL NOT NULL DEFAULT 0,
    max_score REAL NOT NULL,
    evidence_document_ids TEXT,             -- JSON array of document IDs
    comments TEXT,
    assessor_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (assessment_id) REFERENCES sm_supplier_assessments(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, assessment_id, criterion_code)
);
```

### 2.4 Supplier Performance Scorecards
```sql
CREATE TABLE sm_performance_scorecards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    evaluation_period TEXT NOT NULL,         -- "2024-Q1"
    overall_rating REAL NOT NULL DEFAULT 0, -- 0-100
    quality_score REAL NOT NULL DEFAULT 0,
    delivery_score REAL NOT NULL DEFAULT 0,
    cost_score REAL NOT NULL DEFAULT 0,
    responsiveness_score REAL NOT NULL DEFAULT 0,
    compliance_score REAL NOT NULL DEFAULT 0,
    innovation_score REAL NOT NULL DEFAULT 0,
    tier TEXT NOT NULL CHECK(tier IN ('STRATEGIC','PREFERRED','APPROVED','PROVISIONAL','WATCH','RESTRICTED')),
    total_po_value_cents INTEGER NOT NULL DEFAULT 0,
    on_time_delivery_pct REAL NOT NULL DEFAULT 0,
    quality_rejection_rate REAL NOT NULL DEFAULT 0,
    avg_lead_time_days REAL NOT NULL DEFAULT 0,
    invoice_accuracy_pct REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, evaluation_period)
);

CREATE INDEX idx_sm_scorecard_tenant ON sm_performance_scorecards(tenant_id, evaluation_period DESC);
CREATE INDEX idx_sm_scorecard_supplier ON sm_performance_scorecards(tenant_id, supplier_id);
```

### 2.5 Supplier Risk Assessments
```sql
CREATE TABLE sm_supplier_risks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    risk_category TEXT NOT NULL CHECK(risk_category IN (
        'FINANCIAL','OPERATIONAL','COMPLIANCE','GEOPOLITICAL','CYBER','ESG','SUPPLY_DISRUPTION','CONCENTRATION','OTHER'
    )),
    risk_level TEXT NOT NULL CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    risk_score REAL NOT NULL DEFAULT 0,     -- 0-100
    description TEXT NOT NULL,
    likelihood TEXT NOT NULL CHECK(likelihood IN ('RARE','UNLIKELY','POSSIBLE','LIKELY','ALMOST_CERTAIN')),
    impact TEXT NOT NULL CHECK(impact IN ('NEGLIGIBLE','MINOR','MODERATE','MAJOR','CATASTROPHIC')),
    mitigation_plan TEXT,
    owner_id TEXT,
    status TEXT NOT NULL DEFAULT 'IDENTIFIED'
        CHECK(status IN ('IDENTIFIED','ASSESSING','MITIGATING','MONITORING','RESOLVED','ACCEPTED')),
    source TEXT CHECK(source IN ('AUTOMATED','MANUAL','EXTERNAL_FEED','THIRD_PARTY')),
    external_risk_id TEXT,                  -- Reference to external risk monitoring service

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (supplier_id) REFERENCES sm_performance_scorecards(supplier_id) ON DELETE CASCADE
);

CREATE INDEX idx_sm_risk_tenant_supplier ON sm_supplier_risks(tenant_id, supplier_id);
CREATE INDEX idx_sm_risk_level ON sm_supplier_risks(tenant_id, risk_level);
CREATE INDEX idx_sm_risk_category ON sm_supplier_risks(tenant_id, risk_category);
```

### 2.6 Corrective Actions
```sql
CREATE TABLE sm_corrective_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    assessment_id TEXT,
    risk_id TEXT,
    action_number TEXT NOT NULL,            -- CA-2024-00001
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    severity TEXT NOT NULL CHECK(severity IN ('MINOR','MAJOR','CRITICAL')),
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','ACKNOWLEDGED','IN_PROGRESS','SUBMITTED_FOR_REVIEW','VERIFIED','CLOSED','ESCALATED')),
    assigned_to TEXT NOT NULL,
    due_date TEXT NOT NULL,
    completed_date TEXT,
    resolution_notes TEXT,
    evidence_document_ids TEXT,             -- JSON array
    follow_up_required INTEGER NOT NULL DEFAULT 0,
    follow_up_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (assessment_id) REFERENCES sm_supplier_assessments(id) ON DELETE SET NULL,
    FOREIGN KEY (risk_id) REFERENCES sm_supplier_risks(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, action_number)
);

CREATE INDEX idx_sm_ca_tenant_supplier ON sm_corrective_actions(tenant_id, supplier_id, status);
CREATE INDEX idx_sm_ca_due ON sm_corrective_actions(tenant_id, due_date, status);
```

### 2.7 Supplier Segmentation
```sql
CREATE TABLE sm_supplier_segments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    segment TEXT NOT NULL CHECK(segment IN ('STRATEGIC','KEY','PREFERRED','APPROVED','TRANSACTIONAL','NEW')),
    sub_segment TEXT,
    category_ids TEXT,                      -- JSON array of procurement category IDs
    spend_category TEXT CHECK(spend_category IN ('DIRECT','INDIRECT','SERVICES','MIXED')),
    total_annual_spend_cents INTEGER NOT NULL DEFAULT 0,
    contract_count INTEGER NOT NULL DEFAULT 0,
    relationship_start_date TEXT,
    last_review_date TEXT,
    next_review_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id)
);

CREATE INDEX idx_sm_segment_tenant ON sm_supplier_segments(tenant_id, segment);
```

---

## 3. API Endpoints

### 3.1 Qualification Programs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/supplier-mgmt/programs` | Create qualification program |
| GET | `/api/v1/supplier-mgmt/programs` | List programs (filterable) |
| GET | `/api/v1/supplier-mgmt/programs/{id}` | Get program details |
| PUT | `/api/v1/supplier-mgmt/programs/{id}` | Update program |
| DELETE | `/api/v1/supplier-mgmt/programs/{id}` | Deactivate program |
| POST | `/api/v1/supplier-mgmt/programs/{id}/activate` | Activate program |

### 3.2 Supplier Assessments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/supplier-mgmt/assessments` | Initiate supplier assessment |
| GET | `/api/v1/supplier-mgmt/assessments` | List assessments |
| GET | `/api/v1/supplier-mgmt/assessments/{id}` | Get assessment details |
| PUT | `/api/v1/supplier-mgmt/assessments/{id}/responses` | Submit assessment responses |
| POST | `/api/v1/supplier-mgmt/assessments/{id}/submit` | Submit for review |
| POST | `/api/v1/supplier-mgmt/assessments/{id}/review` | Review and decide |
| POST | `/api/v1/supplier-mgmt/assessments/{id}/reopen` | Reopen assessment |

### 3.3 Performance Scorecards
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/supplier-mgmt/scorecards` | List scorecards |
| GET | `/api/v1/supplier-mgmt/scorecards/{supplierId}` | Get supplier scorecard |
| POST | `/api/v1/supplier-mgmt/scorecards/calculate` | Trigger scorecard calculation |
| GET | `/api/v1/supplier-mgmt/scorecards/compare` | Compare supplier scorecards |

### 3.4 Risk Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/supplier-mgmt/risks` | Create risk assessment |
| GET | `/api/v1/supplier-mgmt/risks` | List risks (filterable) |
| GET | `/api/v1/supplier-mgmt/risks/{id}` | Get risk details |
| PUT | `/api/v1/supplier-mgmt/risks/{id}` | Update risk |
| POST | `/api/v1/supplier-mgmt/risks/bulk-assess` | Trigger bulk risk assessment |
| GET | `/api/v1/supplier-mgmt/risks/dashboard` | Risk dashboard data |

### 3.5 Corrective Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/supplier-mgmt/corrective-actions` | Create corrective action |
| GET | `/api/v1/supplier-mgmt/corrective-actions` | List corrective actions |
| PUT | `/api/v1/supplier-mgmt/corrective-actions/{id}` | Update corrective action |
| POST | `/api/v1/supplier-mgmt/corrective-actions/{id}/verify` | Verify completion |

### 3.6 Segmentation
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/supplier-mgmt/segments` | List segment assignments |
| PUT | `/api/v1/supplier-mgmt/segments/{supplierId}` | Update supplier segment |
| POST | `/api/v1/supplier-mgmt/segments/auto-classify` | Auto-classify suppliers |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `supplier.assessment.completed` | `{ assessment_id, supplier_id, score, decision }` | Assessment completed |
| `supplier.qualification.changed` | `{ supplier_id, old_decision, new_decision }` | Qualification status changed |
| `supplier.risk.level_changed` | `{ supplier_id, risk_category, old_level, new_level }` | Risk level changed |
| `supplier.performance.scored` | `{ supplier_id, period, overall_rating, tier }` | Performance scorecard updated |
| `supplier.segment.changed` | `{ supplier_id, old_segment, new_segment }` | Segment assignment changed |
| `supplier.corrective_action.overdue` | `{ action_id, supplier_id, due_date }` | Corrective action past due |
| `supplier.corrective_action.closed` | `{ action_id, supplier_id, resolution }` | Corrective action resolved |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `po.goods_receipt.completed` | Procurement | Update delivery score metrics |
| `invoice.payment.completed` | Accounts Payable | Update financial metrics |
| `quality.inspection.failed` | Quality Management | Trigger corrective action |
| `supplier.registered` | Supplier Portal | Initiate qualification assessment |

---

## 5. Business Rules

1. **Qualification Required**: Suppliers must pass qualification before being added to approved supplier lists
2. **Auto-Requalification**: Trigger requalification when validity expires or risk level changes
3. **Score Thresholds**: STRATEGIC tier requires minimum 85 score; PREFERRED requires 70; APPROVED requires 55
4. **Risk Escalation**: CRITICAL risk automatically escalates to procurement manager and restricts new POs
5. **Corrective Action Timelines**: MINOR actions due in 30 days, MAJOR in 14 days, CRITICAL in 7 days
6. **Performance Calculation**: Weighted average of quality (30%), delivery (25%), cost (20%), responsiveness (15%), compliance (10%)
7. **Auto-Segmentation**: Suppliers auto-classified based on spend volume, strategic importance, and performance scores

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Procurement (11) | Approved supplier list, qualification status checks |
| Accounts Payable (07) | Invoice accuracy metrics, financial health data |
| Supplier Portal (45) | Self-registration, document submission, self-assessment |
| Quality Management (41) | Defect rates, quality inspection results |
| Sourcing (124) | Supplier eligibility for sourcing events |
| Risk & Compliance (34) | Compliance checks, regulatory alerts |
| Workflow (16) | Approval workflows for qualification decisions |
| AI Agents (134) | AI-powered risk prediction and scoring |
