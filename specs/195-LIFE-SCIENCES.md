# 195 - Life Sciences Industry Solution Service Specification

## 1. Domain Overview

Life Sciences Industry Solution provides comprehensive pharmaceutical and biotechnology operations management supporting compound lifecycle tracking from preclinical through approved phases with regulatory status management, clinical trial management with protocol tracking, site management, enrollment monitoring, and regulatory submissions, compliance document management for IND, NDA, ANDA, DMF, and GMP certifications with submission status and agency tracking, pharmacovigilance with adverse event reporting, causality assessment, severity classification, and regulatory reporting compliance, and full lot traceability from raw materials through manufacturing to distribution with genealogy chain tracking. Enables regulated industry compliance with FDA, EMA, and other global health authority requirements. Integrates with Quality for deviations, External systems for regulatory updates, and Inventory for expiry management.

**Bounded Context:** Life Sciences Manufacturing, Compliance & Pharmacovigilance
**Service Name:** `life-sciences-service`
**Database:** `data/life_sciences.db`
**HTTP Port:** 8213 | **gRPC Port:** 9213

---

## 2. Database Schema

### 2.1 Compounds
```sql
CREATE TABLE ls_compounds (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    compound_id TEXT NOT NULL,
    compound_name TEXT NOT NULL,
    therapeutic_area TEXT NOT NULL,
    indication TEXT NOT NULL,
    development_phase TEXT NOT NULL
        CHECK(development_phase IN ('PRECLINICAL','PHASE_I','PHASE_II','PHASE_III','APPROVED')),
    molecule_type TEXT NOT NULL
        CHECK(molecule_type IN ('SMALL_MOLECULE','BIologic','VACCINE','GENE_THERAPY','CELL_THERAPY')),
    regulatory_status TEXT NOT NULL DEFAULT 'DISCOVERY'
        CHECK(regulatory_status IN ('DISCOVERY','PRECLINICAL','IND_SUBMITTED','IND_APPROVED','NDA_SUBMITTED','NDA_APPROVED','MARKETED','WITHDRAWN')),
    patent_number TEXT,
    patent_expiry_date TEXT,
    originator TEXT,
    development_partner TEXT,
    target_mechanism TEXT,
    formulation TEXT,                                 -- JSON: formulation details
    development_milestones TEXT,                      -- JSON: milestone tracking

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, compound_id)
);

CREATE INDEX idx_ls_compound_tenant_phase ON ls_compounds(tenant_id, development_phase);
CREATE INDEX idx_ls_compound_tenant_area ON ls_compounds(tenant_id, therapeutic_area);
CREATE INDEX idx_ls_compound_tenant_reg ON ls_compounds(tenant_id, regulatory_status);
CREATE INDEX idx_ls_compound_tenant_molecule ON ls_compounds(tenant_id, molecule_type);
CREATE INDEX idx_ls_compound_tenant_patent ON ls_compounds(tenant_id, patent_expiry_date);
CREATE INDEX idx_ls_compound_tenant_active ON ls_compounds(tenant_id, is_active);
```

### 2.2 Clinical Trials
```sql
CREATE TABLE ls_clinical_trials (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    trial_id TEXT NOT NULL,
    trial_name TEXT NOT NULL,
    protocol_number TEXT NOT NULL,
    compound_id TEXT NOT NULL,
    trial_phase TEXT NOT NULL
        CHECK(trial_phase IN ('PHASE_I','PHASE_II','PHASE_III','PHASE_IV')),
    trial_type TEXT NOT NULL DEFAULT 'INTERVENTIONAL'
        CHECK(trial_type IN ('INTERVENTIONAL','OBSERVATIONAL','EXPANDED_ACCESS')),
    study_design TEXT,                                -- JSON: randomization, blinding, control
    sites TEXT,                                       -- JSON: array of site IDs and locations
    enrollment_target INTEGER NOT NULL DEFAULT 0,
    enrollment_current INTEGER NOT NULL DEFAULT 0,
    planned_start_date TEXT NOT NULL,
    planned_end_date TEXT NOT NULL,
    actual_start_date TEXT,
    actual_end_date TEXT,
    primary_endpoints TEXT,                           -- JSON: primary endpoint definitions
    regulatory_submissions TEXT,                      -- JSON: submission tracking
    data_management_system TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','RECRUITING','ACTIVE','COMPLETED','TERMINATED','SUSPENDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (compound_id) REFERENCES ls_compounds(id),
    UNIQUE(tenant_id, trial_id)
);

CREATE INDEX idx_ls_trial_tenant_compound ON ls_clinical_trials(tenant_id, compound_id);
CREATE INDEX idx_ls_trial_tenant_phase ON ls_clinical_trials(tenant_id, trial_phase);
CREATE INDEX idx_ls_trial_tenant_status ON ls_clinical_trials(tenant_id, status);
CREATE INDEX idx_ls_trial_tenant_dates ON ls_clinical_trials(tenant_id, planned_start_date, planned_end_date);
CREATE INDEX idx_ls_trial_tenant_active ON ls_clinical_trials(tenant_id, is_active);
```

### 2.3 Compliance Documents
```sql
CREATE TABLE ls_compliance_docs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_number TEXT NOT NULL,
    compound_id TEXT NOT NULL,
    document_type TEXT NOT NULL
        CHECK(document_type IN ('IND','NDA','ANDA','DMF','GMP_CERT','CTD','IDE','PMA','510K')),
    document_title TEXT NOT NULL,
    submission_type TEXT NOT NULL DEFAULT 'INITIAL'
        CHECK(submission_type IN ('INITIAL','AMENDMENT','SUPPLEMENT','ANNUAL_REPORT','RESPONSE')),
    regulatory_agency TEXT NOT NULL
        CHECK(regulatory_agency IN ('FDA','EMA','PMDA','TGA','HEALTH_CANADA','MHRA','NMPA','OTHER')),
    submission_date TEXT,
    acceptance_date TEXT,
    approval_date TEXT,
    rejection_date TEXT,
    submission_status TEXT NOT NULL DEFAULT 'IN_PREPARATION'
        CHECK(submission_status IN ('IN_PREPARATION','SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED','WITHDRAWN')),
    review_comments TEXT,
    conditions TEXT,                                  -- JSON: approval conditions if applicable
    next_submission_date TEXT,
    document_reference_url TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (compound_id) REFERENCES ls_compounds(id),
    UNIQUE(tenant_id, document_number)
);

CREATE INDEX idx_ls_comp_tenant_type ON ls_compliance_docs(tenant_id, document_type);
CREATE INDEX idx_ls_comp_tenant_agency ON ls_compliance_docs(tenant_id, regulatory_agency);
CREATE INDEX idx_ls_comp_tenant_status ON ls_compliance_docs(tenant_id, submission_status);
CREATE INDEX idx_ls_comp_tenant_compound ON ls_compliance_docs(tenant_id, compound_id);
CREATE INDEX idx_ls_comp_tenant_dates ON ls_compliance_docs(tenant_id, submission_date);
CREATE INDEX idx_ls_comp_tenant_active ON ls_compliance_docs(tenant_id, is_active);
```

### 2.4 Pharmacovigilance
```sql
CREATE TABLE ls_pharmacovigilance (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    compound_id TEXT NOT NULL,
    report_type TEXT NOT NULL DEFAULT 'SPONTANEOUS'
        CHECK(report_type IN ('SPONTANEOUS','CLINICAL_TRIAL','LITERATURE','REGULATORY','CONSUMER')),
    patient_age INTEGER,
    patient_gender TEXT
        CHECK(patient_gender IN ('MALE','FEMALE','UNKNOWN')),
    patient_weight_kg REAL,
    suspect_drug TEXT NOT NULL,
    suspect_drug_batch TEXT,
    suspect_drug_dosage TEXT,
    event_description TEXT NOT NULL,
    event_onset_date TEXT NOT NULL,
    event_resolution_date TEXT,
    severity TEXT NOT NULL DEFAULT 'MILD'
        CHECK(severity IN ('MILD','MODERATE','SEVERE','LIFE_THREATENING','FATAL')),
    seriousness_criteria TEXT,                        -- JSON: CIOMS seriousness criteria
    causality_assessment TEXT NOT NULL DEFAULT 'UNCLASSIFIED'
        CHECK(causality_assessment IN ('UNCLASSIFIED','UNLIKELY','POSSIBLE','PROBABLE','CERTAIN')),
    outcome TEXT
        CHECK(outcome IN ('RECOVERED','RECOVERING','NOT_RECOVERED','FATAL','UNKNOWN')),
    reporter_name TEXT NOT NULL,
    reporter_type TEXT NOT NULL DEFAULT 'HEALTHCARE_PROFESSIONAL'
        CHECK(reporter_type IN ('HEALTHCARE_PROFESSIONAL','CONSUMER','REGULATORY_AUTHORITY','OTHER')),
    country_of_event TEXT NOT NULL,
    regulatory_reporting_required INTEGER NOT NULL DEFAULT 0,
    regulatory_report_due_date TEXT,
    regulatory_report_submitted_date TEXT,
    regulatory_reporting_status TEXT NOT NULL DEFAULT 'NOT_REQUIRED'
        CHECK(regulatory_reporting_status IN ('NOT_REQUIRED','PENDING','SUBMITTED','OVERDUE','FILED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (compound_id) REFERENCES ls_compounds(id),
    UNIQUE(tenant_id, event_id)
);

CREATE INDEX idx_pv_tenant_compound ON ls_pharmacovigilance(tenant_id, compound_id);
CREATE INDEX idx_pv_tenant_severity ON ls_pharmacovigilance(tenant_id, severity);
CREATE INDEX idx_pv_tenant_causality ON ls_pharmacovigilance(tenant_id, causality_assessment);
CREATE INDEX idx_pv_tenant_reporting ON ls_pharmacovigilance(tenant_id, regulatory_reporting_status);
CREATE INDEX idx_pv_tenant_dates ON ls_pharmacovigilance(tenant_id, event_onset_date);
CREATE INDEX idx_pv_tenant_country ON ls_pharmacovigilance(tenant_id, country_of_event);
CREATE INDEX idx_pv_tenant_due ON ls_pharmacovigilance(tenant_id, regulatory_report_due_date);
CREATE INDEX idx_pv_tenant_active ON ls_pharmacovigilance(tenant_id, is_active);
```

### 2.5 Lot Traceability
```sql
CREATE TABLE ls_lot_traceability (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lot_number TEXT NOT NULL,
    compound_id TEXT NOT NULL,
    product_name TEXT NOT NULL,
    batch_number TEXT NOT NULL,
    manufacturing_site TEXT NOT NULL,
    manufacturing_date TEXT NOT NULL,
    expiry_date TEXT NOT NULL,
    quantity_produced INTEGER NOT NULL DEFAULT 0,
    quantity_released INTEGER NOT NULL DEFAULT 0,
    quantity_quarantined INTEGER NOT NULL DEFAULT 0,
    genealogy_chain TEXT NOT NULL,                    -- JSON: full upstream raw material genealogy
    raw_material_lots TEXT,                           -- JSON: array of input lot numbers
    parent_lot_id TEXT,                               -- Reference to parent lot for sub-batches
    manufacturing_order TEXT,
    quality_status TEXT NOT NULL DEFAULT 'IN_PROCESS'
        CHECK(quality_status IN ('IN_PROCESS','QUARANTINE','RELEASED','REJECTED','RECALLED')),
    stability_study_id TEXT,
    storage_conditions TEXT,                          -- JSON: temperature, humidity requirements
    distribution_records TEXT,                        -- JSON: distributed-to tracking
    recall_status TEXT
        CHECK(recall_status IN ('NOT_RECALLED','RECALL_PENDING','RECALLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (compound_id) REFERENCES ls_compounds(id),
    FOREIGN KEY (parent_lot_id) REFERENCES ls_lot_traceability(id),
    UNIQUE(tenant_id, lot_number)
);

CREATE INDEX idx_ls_lot_tenant_compound ON ls_lot_traceability(tenant_id, compound_id);
CREATE INDEX idx_ls_lot_tenant_batch ON ls_lot_traceability(tenant_id, batch_number);
CREATE INDEX idx_ls_lot_tenant_quality ON ls_lot_traceability(tenant_id, quality_status);
CREATE INDEX idx_ls_lot_tenant_expiry ON ls_lot_traceability(tenant_id, expiry_date);
CREATE INDEX idx_ls_lot_tenant_site ON ls_lot_traceability(tenant_id, manufacturing_site);
CREATE INDEX idx_ls_lot_tenant_recall ON ls_lot_traceability(tenant_id, recall_status);
CREATE INDEX idx_ls_lot_tenant_parent ON ls_lot_traceability(tenant_id, parent_lot_id);
CREATE INDEX idx_ls_lot_tenant_active ON ls_lot_traceability(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Compounds
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/lifesci/compounds` | List compounds |
| POST | `/api/v1/lifesci/compounds` | Register compound |
| GET | `/api/v1/lifesci/compounds/{id}` | Get compound details |
| PUT | `/api/v1/lifesci/compounds/{id}` | Update compound |
| PATCH | `/api/v1/lifesci/compounds/{id}/phase` | Update development phase |
| GET | `/api/v1/lifesci/compounds/{id}/pipeline` | Get compound pipeline view |

### 3.2 Clinical Trials
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/lifesci/trials` | List clinical trials |
| POST | `/api/v1/lifesci/trials` | Create clinical trial |
| GET | `/api/v1/lifesci/trials/{id}` | Get trial details |
| PUT | `/api/v1/lifesci/trials/{id}` | Update trial |
| PATCH | `/api/v1/lifesci/trials/{id}/enrollment` | Update enrollment count |
| PATCH | `/api/v1/lifesci/trials/{id}/status` | Update trial status |

### 3.3 Compliance
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/lifesci/compliance` | List compliance documents |
| POST | `/api/v1/lifesci/compliance` | Create compliance document |
| GET | `/api/v1/lifesci/compliance/{id}` | Get document details |
| PUT | `/api/v1/lifesci/compliance/{id}` | Update document |
| POST | `/api/v1/lifesci/compliance/{id}/submit` | Submit to regulatory agency |
| GET | `/api/v1/lifesci/compliance/submission-calendar` | Get upcoming submission deadlines |

### 3.4 Pharmacovigilance
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/lifesci/pharmacovigilance` | Report adverse event |
| GET | `/api/v1/lifesci/pharmacovigilance` | List adverse event reports |
| GET | `/api/v1/lifesci/pharmacovigilance/{id}` | Get report details |
| PUT | `/api/v1/lifesci/pharmacovigilance/{id}` | Update adverse event report |
| POST | `/api/v1/lifesci/pharmacovigilance/{id}/submit-regulatory` | Submit regulatory report |
| GET | `/api/v1/lifesci/pharmacovigilance/overdue` | Get overdue regulatory reports |

### 3.5 Lot Traceability
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/lifesci/lots` | List lots |
| POST | `/api/v1/lifesci/lots` | Register lot |
| GET | `/api/v1/lifesci/lots/{id}` | Get lot details |
| PUT | `/api/v1/lifesci/lots/{id}` | Update lot |
| GET | `/api/v1/lifesci/lots/{id}/genealogy` | Get full lot genealogy chain |
| PATCH | `/api/v1/lifesci/lots/{id}/quality-status` | Update lot quality status |
| POST | `/api/v1/lifesci/lots/{id}/recall` | Initiate lot recall |

---

## 4. Business Rules

### 4.1 Compound Management
1. Each compound MUST have a unique compound_id within a tenant.
2. Development phase transitions MUST follow the sequential order: PRECLINICAL -> PHASE_I -> PHASE_II -> PHASE_III -> APPROVED.
3. A compound MUST have regulatory_status IND_APPROVED or higher before entering PHASE_II clinical trials.
4. patent_expiry_date, if provided, MUST be a future date.
5. development_milestones JSON MUST be updated whenever development_phase changes.

### 4.2 Clinical Trial Management
6. Each trial MUST have a unique trial_id within a tenant.
7. Trial phase MUST be compatible with the compound's development_phase; a trial CANNOT be in a higher phase than the compound's current phase.
8. enrollment_current MUST NOT exceed enrollment_target.
9. Trial status transitions MUST follow: PLANNED -> RECRUITING -> ACTIVE -> COMPLETED; TERMINATED and SUSPENDED are valid from any active status.
10. actual_start_date MUST be recorded when status transitions to RECRUITING or ACTIVE.

### 4.3 Compliance Document Management
11. Document numbers MUST be unique within a tenant.
12. submission_status transitions MUST follow: IN_PREPARATION -> SUBMITTED -> UNDER_REVIEW -> APPROVED/REJECTED; WITHDRAWN is valid from SUBMITTED or UNDER_REVIEW.
13. Regulatory submissions with submission_status APPROVED MUST record the approval_date.
14. Each document MUST reference a valid compound_id and specify the target regulatory_agency.
15. Annual report submissions SHOULD be tracked with next_submission_date for compliance calendaring.

### 4.4 Pharmacovigilance
16. Each adverse event MUST have a unique event_id within a tenant.
17. Serious adverse events (severity SEVERE, LIFE_THREATENING, or FATAL) MUST set regulatory_reporting_required to 1 and calculate regulatory_report_due_date per agency timelines.
18. regulatory_reporting_status MUST transition to OVERDUE if regulatory_report_due_date passes without submission.
19. causality_assessment MUST be performed by a qualified safety reviewer; UNCLASSIFIED is only valid as an initial state.
20. Fatal events MUST trigger immediate notification to the relevant regulatory authority and internal safety committee.

### 4.5 Lot Traceability
21. Lot numbers MUST be unique within a tenant.
22. expiry_date MUST be later than manufacturing_date.
23. quantity_released + quantity_quarantined MUST NOT exceed quantity_produced.
24. genealogy_chain JSON MUST trace all upstream raw material lots for full backward traceability.
25. A lot with quality_status RELEASED MUST NOT be changed to IN_PROCESS; only QUARANTINE or REJECTED transitions are valid.
26. Recall initiation MUST update recall_status to RECALL_PENDING and MUST trigger notification to all distribution recipients in distribution_records.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.lifesci.v1;

service LifeSciencesService {
    // Compound management
    rpc CreateCompound(CreateCompoundRequest) returns (CreateCompoundResponse);
    rpc GetCompound(GetCompoundRequest) returns (GetCompoundResponse);
    rpc ListCompounds(ListCompoundsRequest) returns (ListCompoundsResponse);
    rpc UpdateDevelopmentPhase(UpdateDevelopmentPhaseRequest) returns (UpdateDevelopmentPhaseResponse);

    // Clinical trials
    rpc CreateTrial(CreateTrialRequest) returns (CreateTrialResponse);
    rpc GetTrial(GetTrialRequest) returns (GetTrialResponse);
    rpc ListTrials(ListTrialsRequest) returns (ListTrialsResponse);
    rpc UpdateEnrollment(UpdateEnrollmentRequest) returns (UpdateEnrollmentResponse);
    rpc UpdateTrialStatus(UpdateTrialStatusRequest) returns (UpdateTrialStatusResponse);

    // Compliance documents
    rpc CreateComplianceDocument(CreateComplianceDocumentRequest) returns (CreateComplianceDocumentResponse);
    rpc GetComplianceDocument(GetComplianceDocumentRequest) returns (GetComplianceDocumentResponse);
    rpc ListComplianceDocuments(ListComplianceDocumentsRequest) returns (ListComplianceDocumentsResponse);
    rpc SubmitToAgency(SubmitToAgencyRequest) returns (SubmitToAgencyResponse);

    // Pharmacovigilance
    rpc ReportAdverseEvent(ReportAdverseEventRequest) returns (ReportAdverseEventResponse);
    rpc GetAdverseEvent(GetAdverseEventRequest) returns (GetAdverseEventResponse);
    rpc ListAdverseEvents(ListAdverseEventsRequest) returns (ListAdverseEventsResponse);
    rpc SubmitRegulatoryReport(SubmitRegulatoryReportRequest) returns (SubmitRegulatoryReportResponse);
    rpc GetOverdueReports(GetOverdueReportsRequest) returns (GetOverdueReportsResponse);

    // Lot traceability
    rpc CreateLot(CreateLotRequest) returns (CreateLotResponse);
    rpc GetLot(GetLotRequest) returns (GetLotResponse);
    rpc ListLots(ListLotsRequest) returns (ListLotsResponse);
    rpc GetLotGenealogy(GetLotGenealogyRequest) returns (GetLotGenealogyResponse);
    rpc UpdateLotQualityStatus(UpdateLotQualityStatusRequest) returns (UpdateLotQualityStatusResponse);
    rpc InitiateRecall(InitiateRecallRequest) returns (InitiateRecallResponse);
}

message CreateCompoundRequest {
    string tenant_id = 1;
    string compound_id = 2;
    string compound_name = 3;
    string therapeutic_area = 4;
    string indication = 5;
    string development_phase = 6;
    string molecule_type = 7;
    string patent_number = 8;
    string patent_expiry_date = 9;
    string originator = 10;
    string target_mechanism = 11;
    string formulation = 12;
}

message CreateCompoundResponse {
    string id = 1;
    string compound_id = 2;
    string development_phase = 3;
    string regulatory_status = 4;
}

message GetCompoundRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetCompoundResponse {
    string id = 1;
    string compound_id = 2;
    string compound_name = 3;
    string therapeutic_area = 4;
    string indication = 5;
    string development_phase = 6;
    string molecule_type = 7;
    string regulatory_status = 8;
    string patent_number = 9;
    string patent_expiry_date = 10;
}

message ListCompoundsRequest {
    string tenant_id = 1;
    string therapeutic_area = 2;
    string development_phase = 3;
    string molecule_type = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListCompoundsResponse {
    repeated GetCompoundResponse compounds = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateDevelopmentPhaseRequest {
    string tenant_id = 1;
    string id = 2;
    string new_phase = 3;
    string milestone_notes = 4;
}

message UpdateDevelopmentPhaseResponse {
    string id = 1;
    string development_phase = 2;
    string regulatory_status = 3;
}

message CreateTrialRequest {
    string tenant_id = 1;
    string trial_id = 2;
    string trial_name = 3;
    string protocol_number = 4;
    string compound_id = 5;
    string trial_phase = 6;
    string trial_type = 7;
    string study_design = 8;
    string sites = 9;
    int32 enrollment_target = 10;
    string planned_start_date = 11;
    string planned_end_date = 12;
    string primary_endpoints = 13;
}

message CreateTrialResponse {
    string id = 1;
    string trial_id = 2;
    string status = 3;
}

message GetTrialRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetTrialResponse {
    string id = 1;
    string trial_id = 2;
    string trial_name = 3;
    string compound_id = 4;
    string trial_phase = 5;
    int32 enrollment_target = 6;
    int32 enrollment_current = 7;
    string planned_start_date = 8;
    string planned_end_date = 9;
    string status = 10;
}

message ListTrialsRequest {
    string tenant_id = 1;
    string compound_id = 2;
    string trial_phase = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListTrialsResponse {
    repeated GetTrialResponse trials = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateEnrollmentRequest {
    string tenant_id = 1;
    string id = 2;
    int32 enrollment_current = 3;
}

message UpdateEnrollmentResponse {
    string id = 1;
    int32 enrollment_current = 2;
    int32 enrollment_target = 3;
    double enrollment_pct = 4;
}

message UpdateTrialStatusRequest {
    string tenant_id = 1;
    string id = 2;
    string new_status = 3;
    string effective_date = 4;
}

message UpdateTrialStatusResponse {
    string id = 1;
    string status = 2;
    string effective_date = 3;
}

message CreateComplianceDocumentRequest {
    string tenant_id = 1;
    string document_number = 2;
    string compound_id = 3;
    string document_type = 4;
    string document_title = 5;
    string submission_type = 6;
    string regulatory_agency = 7;
    string conditions = 8;
}

message CreateComplianceDocumentResponse {
    string id = 1;
    string document_number = 2;
    string submission_status = 3;
}

message GetComplianceDocumentRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetComplianceDocumentResponse {
    string id = 1;
    string document_number = 2;
    string compound_id = 3;
    string document_type = 4;
    string document_title = 5;
    string regulatory_agency = 6;
    string submission_status = 7;
    string submission_date = 8;
    string approval_date = 9;
}

message ListComplianceDocumentsRequest {
    string tenant_id = 1;
    string compound_id = 2;
    string document_type = 3;
    string regulatory_agency = 4;
    string submission_status = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListComplianceDocumentsResponse {
    repeated GetComplianceDocumentResponse documents = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitToAgencyRequest {
    string tenant_id = 1;
    string id = 2;
    string submission_date = 3;
}

message SubmitToAgencyResponse {
    string id = 1;
    string submission_status = 2;
    string submission_date = 3;
}

message ReportAdverseEventRequest {
    string tenant_id = 1;
    string event_id = 2;
    string compound_id = 3;
    string report_type = 4;
    int32 patient_age = 5;
    string patient_gender = 6;
    double patient_weight_kg = 7;
    string suspect_drug = 8;
    string suspect_drug_batch = 9;
    string suspect_drug_dosage = 10;
    string event_description = 11;
    string event_onset_date = 12;
    string severity = 13;
    string causality_assessment = 14;
    string outcome = 15;
    string reporter_name = 16;
    string reporter_type = 17;
    string country_of_event = 18;
}

message ReportAdverseEventResponse {
    string id = 1;
    string event_id = 2;
    string severity = 3;
    bool regulatory_reporting_required = 4;
    string regulatory_report_due_date = 5;
}

message GetAdverseEventRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetAdverseEventResponse {
    string id = 1;
    string event_id = 2;
    string compound_id = 3;
    string suspect_drug = 4;
    string event_description = 5;
    string severity = 6;
    string causality_assessment = 7;
    string outcome = 8;
    string regulatory_reporting_status = 9;
    string regulatory_report_due_date = 10;
}

message ListAdverseEventsRequest {
    string tenant_id = 1;
    string compound_id = 2;
    string severity = 3;
    string regulatory_reporting_status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListAdverseEventsResponse {
    repeated GetAdverseEventResponse events = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitRegulatoryReportRequest {
    string tenant_id = 1;
    string id = 2;
    string submitted_date = 3;
}

message SubmitRegulatoryReportResponse {
    string id = 1;
    string regulatory_reporting_status = 2;
    string regulatory_report_submitted_date = 3;
}

message GetOverdueReportsRequest {
    string tenant_id = 1;
    int32 page_size = 2;
}

message GetOverdueReportsResponse {
    repeated GetAdverseEventResponse overdue_reports = 1;
    int32 total_count = 2;
}

message CreateLotRequest {
    string tenant_id = 1;
    string lot_number = 2;
    string compound_id = 3;
    string product_name = 4;
    string batch_number = 5;
    string manufacturing_site = 6;
    string manufacturing_date = 7;
    string expiry_date = 8;
    int32 quantity_produced = 9;
    string genealogy_chain = 10;
    string raw_material_lots = 11;
    string parent_lot_id = 12;
    string manufacturing_order = 13;
    string storage_conditions = 14;
}

message CreateLotResponse {
    string id = 1;
    string lot_number = 2;
    string quality_status = 3;
}

message GetLotRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetLotResponse {
    string id = 1;
    string lot_number = 2;
    string compound_id = 3;
    string product_name = 4;
    string batch_number = 5;
    string manufacturing_site = 6;
    string manufacturing_date = 7;
    string expiry_date = 8;
    int32 quantity_produced = 9;
    int32 quantity_released = 10;
    string quality_status = 11;
    string recall_status = 12;
}

message ListLotsRequest {
    string tenant_id = 1;
    string compound_id = 2;
    string quality_status = 3;
    string manufacturing_site = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListLotsResponse {
    repeated GetLotResponse lots = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetLotGenealogyRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetLotGenealogyResponse {
    string lot_number = 1;
    string product_name = 2;
    repeated GenealogyNode upstream_lots = 3;
}

message GenealogyNode {
    string lot_id = 1;
    string lot_number = 2;
    string material_type = 3;
    string supplier = 4;
    repeated GenealogyNode children = 5;
}

message UpdateLotQualityStatusRequest {
    string tenant_id = 1;
    string id = 2;
    string new_status = 3;
    int32 quantity_released = 4;
    int32 quantity_quarantined = 5;
    string reason = 6;
}

message UpdateLotQualityStatusResponse {
    string id = 1;
    string quality_status = 2;
    int32 quantity_released = 3;
}

message InitiateRecallRequest {
    string tenant_id = 1;
    string id = 2;
    string recall_reason = 3;
}

message InitiateRecallResponse {
    string id = 1;
    string lot_number = 2;
    string recall_status = 3;
    int32 distribution_recipients_notified = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `qual-service` | Quality deviations, non-conformance reports, out-of-specification results |
| `external-regulatory-service` | Regulatory update feeds, agency correspondence, guideline changes |
| `inv-service` | Inventory levels, expiry date warnings, stock location data |
| `auth-service` | User identity for safety reporter qualification, regulatory submitter authorization |
| `workflow-service` | Approval workflows for lot release, compliance submission authorization |

### Published To
| Service | Data |
|---------|------|
| `inv-service` | Lot release events for inventory availability, recall notifications for stock quarantine |
| `qual-service` | Adverse event data for signal detection, lot quality status changes for inspection triggers |
| `gl-service` | R&D expense allocations, clinical trial cost accruals, recall liability provisions |
| `reporting-service` | Pipeline analytics, trial enrollment dashboards, compliance submission tracking |
| `notification-service` | Adverse event alerts, compliance deadline warnings, lot recall notifications |
| `audit-service` | Pharmacovigilance audit trail, regulatory submission logs, lot disposition history |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `lifesci.trial.milestone` | `lifesci.events` | `{ trial_id, compound_id, trial_phase, enrollment_current, enrollment_target, status }` | Clinical trial milestone reached |
| `lifesci.adverse_event.reported` | `lifesci.events` | `{ event_id, compound_id, severity, causality_assessment, regulatory_reporting_required, regulatory_report_due_date }` | Adverse event reported |
| `lifesci.submission.approved` | `lifesci.events` | `{ document_id, document_number, compound_id, document_type, regulatory_agency, approval_date }` | Regulatory submission approved |
| `lifesci.lot.released` | `lifesci.events` | `{ lot_id, lot_number, compound_id, product_name, quantity_released, manufacturing_site }` | Lot released for distribution |

---

## 8. Migrations

1. V001: `ls_compounds`
2. V002: `ls_clinical_trials`
3. V003: `ls_compliance_docs`
4. V004: `ls_pharmacovigilance`
5. V005: `ls_lot_traceability`
6. V006: Triggers for `updated_at`
