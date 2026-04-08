# 41 - Quality Management Service Specification

## 1. Domain Overview

Quality Management provides comprehensive quality assurance, inspection execution, non-conformance tracking, corrective and preventive action (CAPA) management, statistical process control (SPC), quality certificate generation, and supplier quality scoring. The service manages the full quality lifecycle from inspection plan definition through characteristic measurement, result recording, disposition decisions, and continuous improvement. Integrates with INV for incoming inspection on receipts, MFG for in-process and final inspections, Procurement for supplier quality scoring, and GL for cost-of-quality accounting.

**Bounded Context:** Quality Assurance, Inspection & Continuous Improvement
**Service Name:** `quality-service`
**Database:** `data/quality.db`
**HTTP Port:** 8068 | **gRPC Port:** 9068

---

## 2. Database Schema

### 2.1 Inspection Plans
```sql
CREATE TABLE quality_inspection_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_type TEXT NOT NULL
        CHECK(plan_type IN ('INCOMING','IN_PROCESS','FINAL','AUDIT','GAU')),
    item_id TEXT,                              -- Nullable for general plans
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),
    description TEXT,
    sampling_plan_type TEXT NOT NULL DEFAULT 'AQL'
        CHECK(sampling_plan_type IN ('100_PERCENT','AQL','INSPECTION_LEVEL')),
    aql_level TEXT,
    frequency_type TEXT NOT NULL DEFAULT 'EVERY_LOT'
        CHECK(frequency_type IN ('EVERY_LOT','EVERY_N_LOTS','PERIODIC','TRIGGERED')),
    frequency_interval INTEGER NOT NULL DEFAULT 1,
    auto_create INTEGER NOT NULL DEFAULT 0,
    estimated_duration_minutes INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_name)
);

CREATE INDEX idx_inspection_plans_tenant_status ON quality_inspection_plans(tenant_id, status);
CREATE INDEX idx_inspection_plans_tenant_type ON quality_inspection_plans(tenant_id, plan_type);
CREATE INDEX idx_inspection_plans_tenant_item ON quality_inspection_plans(tenant_id, item_id);
CREATE INDEX idx_inspection_plans_tenant_active ON quality_inspection_plans(tenant_id, is_active);
```

### 2.2 Inspection Characteristics
```sql
CREATE TABLE quality_inspection_characteristics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    characteristic_name TEXT NOT NULL,
    characteristic_type TEXT NOT NULL
        CHECK(characteristic_type IN ('VARIABLE','ATTRIBUTE')),
    measurement_unit TEXT,
    target_value REAL,
    lower_spec_limit REAL,
    upper_spec_limit REAL,
    criticality TEXT NOT NULL DEFAULT 'MAJOR'
        CHECK(criticality IN ('CRITICAL','MAJOR','MINOR')),
    inspection_method TEXT,
    is_required INTEGER NOT NULL DEFAULT 1,
    sequence INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (plan_id) REFERENCES quality_inspection_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_characteristics_tenant_plan ON quality_inspection_characteristics(tenant_id, plan_id);
CREATE INDEX idx_characteristics_tenant_criticality ON quality_inspection_characteristics(tenant_id, criticality);
```

### 2.3 Inspections
```sql
CREATE TABLE quality_inspections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    inspection_number TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('RECEIPT','WORK_ORDER','TRANSFER','INVENTORY','RMA')),
    source_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    lot_number TEXT,
    quantity_inspected REAL NOT NULL DEFAULT 0,
    quantity_sampled REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','PASSED','FAILED','CONDITIONAL')),
    inspector_id TEXT,
    inspection_date TEXT,
    start_time TEXT,
    end_time TEXT,
    overall_result TEXT
        CHECK(overall_result IN ('PASS','FAIL','CONDITIONAL_PASS')),
    notes TEXT,
    disposition TEXT
        CHECK(disposition IN ('ACCEPT','REJECT','CONDITIONAL_ACCEPT','HOLD','SCRAP')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES quality_inspection_plans(id),

    UNIQUE(tenant_id, inspection_number)
);

CREATE INDEX idx_inspections_tenant_status ON quality_inspections(tenant_id, status);
CREATE INDEX idx_inspections_tenant_source ON quality_inspections(tenant_id, source_type, source_id);
CREATE INDEX idx_inspections_tenant_item ON quality_inspections(tenant_id, item_id);
CREATE INDEX idx_inspections_tenant_result ON quality_inspections(tenant_id, overall_result);
CREATE INDEX idx_inspections_tenant_date ON quality_inspections(tenant_id, inspection_date);
CREATE INDEX idx_inspections_tenant_inspector ON quality_inspections(tenant_id, inspector_id);
```

### 2.4 Inspection Results
```sql
CREATE TABLE quality_inspection_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    inspection_id TEXT NOT NULL,
    characteristic_id TEXT NOT NULL,
    result_type TEXT NOT NULL
        CHECK(result_type IN ('NUMERIC','PASS_FAIL','TEXT')),
    numeric_value REAL,
    pass_fail_result TEXT
        CHECK(pass_fail_result IN ('PASS','FAIL')),
    text_result TEXT,
    is_out_of_spec INTEGER NOT NULL DEFAULT 0,
    deviation_amount REAL,
    notes TEXT,
    measured_by TEXT NOT NULL,
    measured_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (inspection_id) REFERENCES quality_inspections(id) ON DELETE CASCADE,
    FOREIGN KEY (characteristic_id) REFERENCES quality_inspection_characteristics(id)
);

CREATE INDEX idx_results_tenant_inspection ON quality_inspection_results(tenant_id, inspection_id);
CREATE INDEX idx_results_tenant_characteristic ON quality_inspection_results(tenant_id, characteristic_id);
CREATE INDEX idx_results_tenant_out_of_spec ON quality_inspection_results(tenant_id, is_out_of_spec);
```

### 2.5 Non-Conformance Reports
```sql
CREATE TABLE quality_ncr (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    ncr_number TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('INSPECTION','CUSTOMER_COMPLAINT','INTERNAL','AUDIT')),
    source_id TEXT,
    item_id TEXT,
    lot_number TEXT,
    quantity_affected REAL NOT NULL DEFAULT 0,
    defect_type TEXT,
    defect_code TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'MAJOR'
        CHECK(severity IN ('MINOR','MAJOR','CRITICAL','SAFETY')),
    description TEXT NOT NULL,
    root_cause_category TEXT
        CHECK(root_cause_category IN ('MATERIAL','METHOD','MACHINE','MANPOWER','MEASUREMENT','ENVIRONMENT')),
    root_cause_description TEXT,
    containment_actions TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','INVESTIGATING','CAPA_CREATED','RESOLVED','CLOSED')),
    reported_by TEXT NOT NULL,
    assigned_to TEXT,
    due_date TEXT,
    resolution_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, ncr_number)
);

CREATE INDEX idx_ncr_tenant_status ON quality_ncr(tenant_id, status);
CREATE INDEX idx_ncr_tenant_severity ON quality_ncr(tenant_id, severity);
CREATE INDEX idx_ncr_tenant_item ON quality_ncr(tenant_id, item_id);
CREATE INDEX idx_ncr_tenant_assigned ON quality_ncr(tenant_id, assigned_to);
CREATE INDEX idx_ncr_tenant_due ON quality_ncr(tenant_id, due_date);
```

### 2.6 Corrective and Preventive Actions
```sql
CREATE TABLE quality_capa (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    capa_number TEXT NOT NULL,
    capa_type TEXT NOT NULL
        CHECK(capa_type IN ('CORRECTIVE','PREVENTIVE')),
    ncr_id TEXT,                               -- Nullable for standalone CAPAs
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    owner_id TEXT NOT NULL,
    team_members TEXT,                         -- JSON array of user IDs
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'IDENTIFIED'
        CHECK(status IN ('IDENTIFIED','INVESTIGATING','ACTION_PLANNED','IMPLEMENTING','VERIFYING','CLOSED')),
    investigation_findings TEXT,
    root_cause TEXT,
    corrective_actions TEXT,
    preventive_actions TEXT,
    effectiveness_criteria TEXT,
    effectiveness_verified_by TEXT,
    effectiveness_verified_at TEXT,
    target_date TEXT NOT NULL,
    completed_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (ncr_id) REFERENCES quality_ncr(id) ON DELETE SET NULL,

    UNIQUE(tenant_id, capa_number)
);

CREATE INDEX idx_capa_tenant_status ON quality_capa(tenant_id, status);
CREATE INDEX idx_capa_tenant_priority ON quality_capa(tenant_id, priority);
CREATE INDEX idx_capa_tenant_ncr ON quality_capa(tenant_id, ncr_id);
CREATE INDEX idx_capa_tenant_owner ON quality_capa(tenant_id, owner_id);
CREATE INDEX idx_capa_tenant_target ON quality_capa(tenant_id, target_date);
```

### 2.7 CAPA Action Items
```sql
CREATE TABLE quality_capa_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    capa_id TEXT NOT NULL,
    action_type TEXT NOT NULL
        CHECK(action_type IN ('CORRECTIVE','PREVENTIVE')),
    action_description TEXT NOT NULL,
    assigned_to TEXT NOT NULL,
    due_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','OVERDUE')),
    completed_date TEXT,
    verification_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (capa_id) REFERENCES quality_capa(id) ON DELETE CASCADE
);

CREATE INDEX idx_capa_actions_tenant_capa ON quality_capa_actions(tenant_id, capa_id);
CREATE INDEX idx_capa_actions_tenant_status ON quality_capa_actions(tenant_id, status);
CREATE INDEX idx_capa_actions_tenant_assigned ON quality_capa_actions(tenant_id, assigned_to);
CREATE INDEX idx_capa_actions_tenant_due ON quality_capa_actions(tenant_id, due_date);
```

### 2.8 Statistical Process Control Data
```sql
CREATE TABLE quality_spc_data (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    process_id TEXT NOT NULL,
    characteristic_id TEXT NOT NULL,
    measurement_date TEXT NOT NULL,
    measurement_value REAL NOT NULL,
    sample_group TEXT NOT NULL,
    subgroup_size INTEGER NOT NULL DEFAULT 1,
    machine_id TEXT,
    operator_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_spc_data_tenant_process ON quality_spc_data(tenant_id, process_id);
CREATE INDEX idx_spc_data_tenant_characteristic ON quality_spc_data(tenant_id, characteristic_id);
CREATE INDEX idx_spc_data_tenant_date ON quality_spc_data(tenant_id, measurement_date);
CREATE INDEX idx_spc_data_tenant_group ON quality_spc_data(tenant_id, sample_group);
CREATE INDEX idx_spc_data_tenant_machine ON quality_spc_data(tenant_id, machine_id);
```

### 2.9 Quality Certificates
```sql
CREATE TABLE quality_certificates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    certificate_type TEXT NOT NULL
        CHECK(certificate_type IN ('COA','COC','COMPLIANCE','TEST_REPORT')),
    certificate_number TEXT NOT NULL,
    item_id TEXT NOT NULL,
    lot_number TEXT,
    customer_id TEXT,
    inspection_id TEXT,
    issue_date TEXT,
    expiry_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ISSUED','VOIDED')),
    issued_by TEXT,
    certificate_data TEXT,                     -- JSON: certificate field values
    file_path TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (inspection_id) REFERENCES quality_inspections(id) ON DELETE SET NULL,

    UNIQUE(tenant_id, certificate_number)
);

CREATE INDEX idx_certificates_tenant_type ON quality_certificates(tenant_id, certificate_type);
CREATE INDEX idx_certificates_tenant_status ON quality_certificates(tenant_id, status);
CREATE INDEX idx_certificates_tenant_item ON quality_certificates(tenant_id, item_id);
CREATE INDEX idx_certificates_tenant_customer ON quality_certificates(tenant_id, customer_id);
CREATE INDEX idx_certificates_tenant_expiry ON quality_certificates(tenant_id, expiry_date);
```

### 2.10 Supplier Quality Scores
```sql
CREATE TABLE quality_supplier_scores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    period TEXT NOT NULL,                      -- Format: YYYY-MM
    total_inspections INTEGER NOT NULL DEFAULT 0,
    passed_inspections INTEGER NOT NULL DEFAULT 0,
    failed_inspections INTEGER NOT NULL DEFAULT 0,
    defect_rate_ppm REAL NOT NULL DEFAULT 0,
    on_time_delivery_percent REAL NOT NULL DEFAULT 0,
    quality_score REAL NOT NULL DEFAULT 0,
    delivery_score REAL NOT NULL DEFAULT 0,
    overall_score REAL NOT NULL DEFAULT 0,
    ncr_count INTEGER NOT NULL DEFAULT 0,
    capa_count INTEGER NOT NULL DEFAULT 0,
    rating TEXT NOT NULL DEFAULT 'B'
        CHECK(rating IN ('A','B','C','D','F')),
    review_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, supplier_id, period)
);

CREATE INDEX idx_supplier_scores_tenant_supplier ON quality_supplier_scores(tenant_id, supplier_id);
CREATE INDEX idx_supplier_scores_tenant_period ON quality_supplier_scores(tenant_id, period);
CREATE INDEX idx_supplier_scores_tenant_rating ON quality_supplier_scores(tenant_id, rating);
```

---

## 3. REST API Endpoints

```
# Inspection Plans
GET/POST      /api/v1/quality/inspection-plans                        Permission: quality.plans.read/create
GET/PUT       /api/v1/quality/inspection-plans/{id}                   Permission: quality.plans.read/update
GET/POST      /api/v1/quality/inspection-plans/{id}/characteristics   Permission: quality.plans.read/create

# Inspections
GET/POST      /api/v1/quality/inspections                             Permission: quality.inspections.read/create
GET/PUT       /api/v1/quality/inspections/{id}                        Permission: quality.inspections.read/update
POST          /api/v1/quality/inspections/{id}/start                  Permission: quality.inspections.execute
POST          /api/v1/quality/inspections/{id}/complete               Permission: quality.inspections.execute
GET/POST      /api/v1/quality/inspections/{id}/results                Permission: quality.inspections.read/create
POST          /api/v1/quality/inspections/{id}/disposition            Permission: quality.inspections.disposition

# NCR
GET/POST      /api/v1/quality/ncr                                     Permission: quality.ncr.read/create
GET/PUT       /api/v1/quality/ncr/{id}                                Permission: quality.ncr.read/update
POST          /api/v1/quality/ncr/{id}/create-capa                    Permission: quality.ncr.create-capa
POST          /api/v1/quality/ncr/{id}/resolve                        Permission: quality.ncr.resolve
POST          /api/v1/quality/ncr/{id}/close                          Permission: quality.ncr.close

# CAPA
GET/POST      /api/v1/quality/capa                                    Permission: quality.capa.read/create
GET/PUT       /api/v1/quality/capa/{id}                               Permission: quality.capa.read/update
GET/POST      /api/v1/quality/capa/{id}/actions                       Permission: quality.capa.read/create
POST          /api/v1/quality/capa/{id}/verify-effectiveness          Permission: quality.capa.verify
POST          /api/v1/quality/capa/{id}/close                         Permission: quality.capa.close

# SPC
POST          /api/v1/quality/spc/data                                Permission: quality.spc.create
GET           /api/v1/quality/spc/control-chart                       Permission: quality.spc.read
GET           /api/v1/quality/spc/capability-analysis                  Permission: quality.spc.read

# Certificates
GET/POST      /api/v1/quality/certificates                            Permission: quality.certificates.read/create
GET           /api/v1/quality/certificates/{id}                       Permission: quality.certificates.read
POST          /api/v1/quality/certificates/{id}/issue                 Permission: quality.certificates.issue
POST          /api/v1/quality/certificates/{id}/void                  Permission: quality.certificates.void

# Supplier Quality
GET           /api/v1/quality/supplier-scores                         Permission: quality.supplier.read
GET           /api/v1/quality/supplier-scores/{supplier_id}           Permission: quality.supplier.read

# Reports
GET           /api/v1/quality/reports/defect-trend                    Permission: quality.reports.view
GET           /api/v1/quality/reports/inspection-yield                 Permission: quality.reports.view
GET           /api/v1/quality/reports/capa-aging                       Permission: quality.reports.view
GET           /api/v1/quality/reports/supplier-quality                 Permission: quality.reports.view
GET           /api/v1/quality/reports/cost-of-quality                  Permission: quality.reports.view
```

---

## 4. Business Rules

### 4.1 Inspection Plan Activation
```
Before activating an inspection plan:
  1. Plan MUST have at least one characteristic defined
  2. All required characteristics must have inspection_method populated
  3. VARIABLE characteristics must have target_value and at least one spec limit
  4. AQL-based plans must have aql_level set
  5. Plans with frequency_type EVERY_N_LOTS must have frequency_interval > 0
  6. DRAFT plans with no characteristics cannot be set to ACTIVE
```

### 4.2 AQL Sampling
- AQL sampling determines sample size based on lot size and inspection level
- Standard AQL levels: 0.065, 0.10, 0.15, 0.25, 0.40, 0.65, 1.0, 1.5, 2.5, 4.0, 6.5
- Sample size determined by lot size range and AQL level lookup table
- 100_PERCENT sampling requires inspection of entire lot; quantity_sampled = quantity_inspected
- INSPECTION_LEVEL sampling uses ISO 2859-1 tables for sample derivation

### 4.3 Inspection Execution and Results
- Out-of-spec results (value outside lower_spec_limit/upper_spec_limit) automatically flag inspection as FAILED
- CRITICAL characteristic failures immediately set overall_result to FAIL regardless of other results
- MAJOR characteristic failures contribute to failure based on acceptance number
- MINOR characteristic failures tracked but do not independently fail inspection
- Failed inspections require disposition decision: REJECT, HOLD, or CONDITIONAL_ACCEPT
- CONDITIONAL_ACCEPT disposition requires documented justification and approval

### 4.4 NCR and CAPA Workflow
- NCRs for CRITICAL or SAFETY severity MUST trigger CAPA creation
- CAPA lifecycle: IDENTIFIED -> INVESTIGATING -> ACTION_PLANNED -> IMPLEMENTING -> VERIFYING -> CLOSED
- CAPA actions must be verified for effectiveness before closure
- Overdue CAPA actions (past due_date) auto-set status to OVERDUE
- NCR cannot be CLOSED until linked CAPA is CLOSED (if CAPA was created)
- Root cause category is required before moving from INVESTIGATING to ACTION_PLANNED

### 4.5 Statistical Process Control
- SPC control charts calculate UCL/LCL from process data using X-bar and R chart methodology
- X-bar chart: plots subgroup means; UCL/LCL = X-double-bar +/- A2 * R-bar
- R chart: plots subgroup ranges; UCL = D4 * R-bar, LCL = D3 * R-bar
- Control constants (A2, D3, D4) derived from subgroup size
- Points outside control limits trigger out-of-control signals
- Process capability indices: Cp = (USL - LSL) / (6 * sigma), Cpk = min((USL - X-bar), (X-bar - LSL)) / (3 * sigma)
- Cp/Cpk < 1.33 triggers process improvement action
- Cp/Cpk < 1.0 indicates process is not capable

### 4.6 Supplier Quality Scoring
- Supplier quality scores are calculated monthly
- quality_score = (passed_inspections / total_inspections) * 100
- defect_rate_ppm = (failed_inspections / total_inspections) * 1,000,000
- overall_score = (quality_score * 0.5) + (delivery_score * 0.3) + ((100 - normalized_ncr_count) * 0.2)
- Rating thresholds: A >= 90, B >= 80, C >= 70, D >= 60, F < 60
- Ratings below C trigger supplier review notification to procurement

### 4.7 Quality Certificates
- Certificates of Analysis (CoA) auto-populate from inspection results
- CoC (Certificate of Conformance) confirms product meets specification requirements
- Certificate data includes: item details, lot number, test results, specifications, pass/fail status
- Issued certificates cannot be modified; must be voided and reissued
- Certificates may have expiry dates based on customer or regulatory requirements

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `quality.inspection.created` | New inspection record created | Workflow, INV |
| `quality.inspection.completed` | Inspection marked complete (pass/conditional) | INV, OM |
| `quality.inspection.failed` | Inspection result is FAIL | INV, Procurement |
| `quality.ncr.created` | Non-conformance report filed | Workflow, MFG |
| `quality.capa.created` | CAPA initiated | Workflow |
| `quality.capa.closed` | CAPA verified and closed | NCR, Reporting |
| `quality.certificate.issued` | Quality certificate issued | OM, DMS |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.quality.v1;

service QualityService {
    rpc CreateInspection(CreateInspectionRequest) returns (CreateInspectionResponse);
    rpc RecordResult(RecordResultRequest) returns (RecordResultResponse);
    rpc GetItemQualityProfile(GetItemQualityProfileRequest) returns (GetItemQualityProfileResponse);
    rpc GetSupplierQualityScore(GetSupplierQualityScoreRequest) returns (GetSupplierQualityScoreResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **INV:** Item master for inspection targets, lot tracking, receipt triggers for incoming inspection, inventory hold/release
- **MFG:** Work order details for in-process and final inspections, routing operations, production details
- **Procurement:** Supplier master for quality scoring, PO details for receipt-based inspection triggers
- **OM:** Customer details for certificate generation, RMA details for return inspections
- **DMS:** Document storage for certificates, evidence attachments, inspection reports
- **Workflow:** Approval workflows for disposition decisions, CAPA approvals

### 6.2 Data Published To
- **INV:** Inspection pass/fail results for lot disposition, hold/release signals, inventory status updates
- **Procurement:** Supplier quality score updates, NCR notifications for supplier-related defects
- **MFG:** In-process inspection results, SPC alerts for out-of-control conditions, quality hold signals
- **OM:** Quality certificate availability, inspection completion for shipment release
- **GL:** Cost-of-quality entries (scrap, rework, inspection costs)
- **Workflow:** Inspection approval routing, CAPA escalation, disposition approval workflows
- **DMS:** Generated certificate documents, inspection report documents

---

## 7. Migrations

1. V001: `quality_inspection_plans`
2. V002: `quality_inspection_characteristics`
3. V003: `quality_inspections`
4. V004: `quality_inspection_results`
5. V005: `quality_ncr`
6. V006: `quality_capa`
7. V007: `quality_capa_actions`
8. V008: `quality_spc_data`
9. V009: `quality_certificates`
10. V010: `quality_supplier_scores`
11. V011: Triggers for `updated_at`
