# 186 - Oracle Health Service Specification

## 1. Domain Overview

Oracle Health provides comprehensive clinical data management covering patient record lifecycle management with demographic and insurance tracking, clinical encounter documentation across inpatient, outpatient, emergency, telehealth, and surgical settings, care plan orchestration with conditions, goals, interventions, and medication management, clinical workflow automation for admission, discharge, transfer, and referral processes, and health analytics with quality measures and readmission rate monitoring. The system serves as the clinical backbone for healthcare operations, enabling longitudinal patient records and evidence-based care delivery. Integrates with HCM for provider credentialing, Security for PHI access controls, and Reporting for regulatory submissions.

**Bounded Context:** Clinical Data Management & Healthcare Operations
**Service Name:** `oracle-health-service`
**Database:** `data/oracle_health.db`
**HTTP Port:** 8204 | **gRPC Port:** 9204

---

## 2. Database Schema

### 2.1 Patient Records
```sql
CREATE TABLE oh_patient_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mrn TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    date_of_birth TEXT NOT NULL,
    gender TEXT NOT NULL
        CHECK(gender IN ('MALE','FEMALE','NON_BINARY','OTHER','UNKNOWN')),
    demographics TEXT NOT NULL,                      -- JSON: address, phone, email, emergency contacts, language, ethnicity
    insurance TEXT,                                  -- JSON: payer name, policy number, group number, effective dates
    consent TEXT,                                    -- JSON: consent types with dates (treatment, data sharing, research)
    patient_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(patient_status IN ('ACTIVE','INACTIVE','DECEASED','TRANSFERRED')),
    primary_provider_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, mrn)
);

CREATE INDEX idx_patient_records_tenant_name ON oh_patient_records(tenant_id, last_name, first_name);
CREATE INDEX idx_patient_records_tenant_dob ON oh_patient_records(tenant_id, date_of_birth);
CREATE INDEX idx_patient_records_tenant_status ON oh_patient_records(tenant_id, patient_status);
CREATE INDEX idx_patient_records_tenant_provider ON oh_patient_records(tenant_id, primary_provider_id);
CREATE INDEX idx_patient_records_tenant_active ON oh_patient_records(tenant_id, is_active);
```

### 2.2 Clinical Encounters
```sql
CREATE TABLE oh_clinical_encounters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    patient_id TEXT NOT NULL,
    encounter_id TEXT NOT NULL,
    encounter_type TEXT NOT NULL
        CHECK(encounter_type IN ('INPATIENT','OUTPATIENT','EMERGENCY','TELEHEALTH','SURGICAL')),
    encounter_date TEXT NOT NULL,
    admitting_provider_id TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    department_id TEXT,
    chief_complaint TEXT NOT NULL,
    diagnosis_codes TEXT,                            -- JSON: array of ICD-10 codes
    procedure_codes TEXT,                            -- JSON: array of CPT codes
    encounter_notes TEXT,
    discharge_disposition TEXT
        CHECK(discharge_disposition IS NULL OR discharge_disposition IN ('HOME','SNF','REHAB','AMA','EXPIRED','TRANSFERRED')),
    encounter_status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(encounter_status IN ('SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED','NO_SHOW')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (patient_id) REFERENCES oh_patient_records(id),
    UNIQUE(tenant_id, encounter_id)
);

CREATE INDEX idx_encounters_tenant_patient ON oh_clinical_encounters(tenant_id, patient_id);
CREATE INDEX idx_encounters_tenant_type ON oh_clinical_encounters(tenant_id, encounter_type);
CREATE INDEX idx_encounters_tenant_date ON oh_clinical_encounters(tenant_id, encounter_date);
CREATE INDEX idx_encounters_tenant_status ON oh_clinical_encounters(tenant_id, encounter_status);
CREATE INDEX idx_encounters_tenant_provider ON oh_clinical_encounters(tenant_id, admitting_provider_id);
CREATE INDEX idx_encounters_tenant_facility ON oh_clinical_encounters(tenant_id, facility_id);
CREATE INDEX idx_encounters_tenant_active ON oh_clinical_encounters(tenant_id, is_active);
```

### 2.3 Care Plans
```sql
CREATE TABLE oh_care_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    patient_id TEXT NOT NULL,
    encounter_id TEXT,
    care_plan_id TEXT NOT NULL,
    conditions TEXT NOT NULL,                        -- JSON: array of diagnosed conditions with ICD codes
    goals TEXT NOT NULL,                             -- JSON: array of care goals with target dates and measures
    interventions TEXT NOT NULL,                     -- JSON: array of planned interventions with frequencies
    medications TEXT,                                -- JSON: array of prescribed medications with dosages
    care_team TEXT,                                  -- JSON: array of provider IDs and roles
    plan_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(plan_status IN ('DRAFT','ACTIVE','COMPLETED','SUSPENDED','CANCELLED')),
    effective_start TEXT NOT NULL,
    effective_end TEXT,
    review_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (patient_id) REFERENCES oh_patient_records(id),
    UNIQUE(tenant_id, care_plan_id)
);

CREATE INDEX idx_care_plans_tenant_patient ON oh_care_plans(tenant_id, patient_id);
CREATE INDEX idx_care_plans_tenant_status ON oh_care_plans(tenant_id, plan_status);
CREATE INDEX idx_care_plans_tenant_dates ON oh_care_plans(tenant_id, effective_start, effective_end);
CREATE INDEX idx_care_plans_tenant_review ON oh_care_plans(tenant_id, review_date);
CREATE INDEX idx_care_plans_tenant_active ON oh_care_plans(tenant_id, is_active);
```

### 2.4 Clinical Workflows
```sql
CREATE TABLE oh_clinical_workflows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    workflow_type TEXT NOT NULL
        CHECK(workflow_type IN ('ADMISSION','DISCHARGE','TRANSFER','REFERRAL')),
    workflow_name TEXT NOT NULL,
    patient_id TEXT NOT NULL,
    encounter_id TEXT,
    steps TEXT NOT NULL,                             -- JSON: ordered array of steps with status, assignee, due_date
    current_step_index INTEGER NOT NULL DEFAULT 0,
    total_steps INTEGER NOT NULL DEFAULT 0,
    assigned_provider_id TEXT,
    priority TEXT NOT NULL DEFAULT 'ROUTINE'
        CHECK(priority IN ('ROUTINE','URGENT','STAT')),
    workflow_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(workflow_status IN ('PENDING','IN_PROGRESS','COMPLETED','CANCELLED','ON_HOLD')),
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (patient_id) REFERENCES oh_patient_records(id)
);

CREATE INDEX idx_workflows_tenant_patient ON oh_clinical_workflows(tenant_id, patient_id);
CREATE INDEX idx_workflows_tenant_type ON oh_clinical_workflows(tenant_id, workflow_type);
CREATE INDEX idx_workflows_tenant_status ON oh_clinical_workflows(tenant_id, workflow_status);
CREATE INDEX idx_workflows_tenant_priority ON oh_clinical_workflows(tenant_id, priority);
CREATE INDEX idx_workflows_tenant_provider ON oh_clinical_workflows(tenant_id, assigned_provider_id);
CREATE INDEX idx_workflows_tenant_active ON oh_clinical_workflows(tenant_id, is_active);
```

### 2.5 Health Analytics
```sql
CREATE TABLE oh_health_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,
    quality_measure_type TEXT NOT NULL
        CHECK(quality_measure_type IN ('READMISSION_RATE','MORTALITY_RATE','PATIENT_SAFETY','CLINICAL_OUTCOME','PATIENT_EXPERIENCE','EFFICIENCY')),
    measure_name TEXT NOT NULL,
    measure_value REAL NOT NULL DEFAULT 0.0,
    target_value REAL NOT NULL DEFAULT 0.0,
    numerator INTEGER NOT NULL DEFAULT 0,
    denominator INTEGER NOT NULL DEFAULT 0,
    unit TEXT NOT NULL DEFAULT 'PERCENTAGE',
    reporting_period TEXT NOT NULL
        CHECK(reporting_period IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY','ANNUALLY')),
    benchmark_value REAL,
    percentile_rank REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, facility_id, metric_date, quality_measure_type, measure_name, reporting_period)
);

CREATE INDEX idx_analytics_tenant_facility ON oh_health_analytics(tenant_id, facility_id);
CREATE INDEX idx_analytics_tenant_date ON oh_health_analytics(tenant_id, metric_date);
CREATE INDEX idx_analytics_tenant_type ON oh_health_analytics(tenant_id, quality_measure_type);
CREATE INDEX idx_analytics_tenant_period ON oh_health_analytics(tenant_id, reporting_period);
CREATE INDEX idx_analytics_tenant_active ON oh_health_analytics(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Patient Records
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health/patients` | List patient records |
| POST | `/api/v1/health/patients` | Create patient record |
| GET | `/api/v1/health/patients/{id}` | Get patient details |
| PUT | `/api/v1/health/patients/{id}` | Update patient record |
| PATCH | `/api/v1/health/patients/{id}/status` | Update patient status |
| GET | `/api/v1/health/patients/search` | Search patients by name, MRN, or DOB |

### 3.2 Clinical Encounters
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health/encounters` | List clinical encounters |
| POST | `/api/v1/health/encounters` | Create encounter |
| GET | `/api/v1/health/encounters/{id}` | Get encounter details |
| PUT | `/api/v1/health/encounters/{id}` | Update encounter |
| POST | `/api/v1/health/encounters/{id}/start` | Start encounter |
| POST | `/api/v1/health/encounters/{id}/complete` | Complete encounter |
| POST | `/api/v1/health/encounters/{id}/cancel` | Cancel encounter |

### 3.3 Care Plans
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health/care-plans` | List care plans |
| POST | `/api/v1/health/care-plans` | Create care plan |
| GET | `/api/v1/health/care-plans/{id}` | Get care plan details |
| PUT | `/api/v1/health/care-plans/{id}` | Update care plan |
| POST | `/api/v1/health/care-plans/{id}/activate` | Activate care plan |
| POST | `/api/v1/health/care-plans/{id}/complete` | Complete care plan |

### 3.4 Clinical Workflows
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health/workflows` | List clinical workflows |
| POST | `/api/v1/health/workflows` | Create workflow |
| GET | `/api/v1/health/workflows/{id}` | Get workflow details |
| PUT | `/api/v1/health/workflows/{id}` | Update workflow |
| POST | `/api/v1/health/workflows/{id}/advance` | Advance to next step |
| POST | `/api/v1/health/workflows/{id}/hold` | Put workflow on hold |
| POST | `/api/v1/health/workflows/{id}/cancel` | Cancel workflow |

### 3.5 Health Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health/analytics/dashboard` | Get analytics dashboard |
| GET | `/api/v1/health/analytics/quality-measures` | Get quality measure results |
| GET | `/api/v1/health/analytics/readmission-rates` | Get readmission rate trends |
| GET | `/api/v1/health/analytics/benchmarks` | Get benchmark comparisons |

---

## 4. Business Rules

### 4.1 Patient Records
1. Each patient MUST have a unique MRN within a tenant.
2. Patient demographics MUST include at minimum: address, primary phone, and preferred language.
3. Insurance data MUST be validated for required fields (payer, policy number) before encounter creation.
4. Consent records MUST include a consent date and the type of consent granted.
5. A patient with status DECEASED MUST NOT be assigned new encounters or care plans.

### 4.2 Clinical Encounters
6. Encounter status transitions MUST follow: SCHEDULED -> IN_PROGRESS -> COMPLETED or SCHEDULED -> CANCELLED or SCHEDULED -> NO_SHOW.
7. An encounter MUST NOT be set to IN_PROGRESS unless the patient record status is ACTIVE.
8. Diagnosis codes MUST be valid ICD-10 codes; procedure codes MUST be valid CPT codes.
9. Each encounter MUST be associated with exactly one admitting provider and one facility.
10. Discharge disposition MUST be set before an encounter can be marked COMPLETED.

### 4.3 Care Plans
11. Care plan status transitions MUST follow: DRAFT -> ACTIVE -> COMPLETED or DRAFT -> CANCELLED or ACTIVE -> SUSPENDED -> ACTIVE.
12. A care plan MUST have at least one condition, one goal, and one intervention before activation.
13. Medication entries MUST include dosage, frequency, and route of administration.
14. The review_date MUST be within the effective_start and effective_end range for active plans.
15. Care team changes MUST be logged in the care plan version history.

### 4.4 Clinical Workflows
16. Admission workflows MUST verify insurance eligibility and consent before proceeding past the initial step.
17. Discharge workflows MUST confirm all care plan goals addressed, prescriptions ordered, and follow-up scheduled.
18. Transfer workflows MUST have both source and target facility identifiers documented.
19. Referral workflows MUST include the referring provider, target specialty, and clinical justification.
20. current_step_index MUST NOT exceed total_steps minus one.
21. A workflow with priority STAT MUST complete all steps within 24 hours of creation.

### 4.5 Health Analytics
22. Quality measure values MUST be calculated as numerator divided by denominator when denominator > 0.
23. Readmission rate MUST track 30-day unplanned readmissions for the same or related condition.
24. Benchmark values SHOULD be sourced from national or regional healthcare databases.
25. Analytics records MUST NOT be modified after creation; corrections require a new record.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.health.v1;

service OracleHealthService {
    // Patient records
    rpc CreatePatient(CreatePatientRequest) returns (CreatePatientResponse);
    rpc GetPatient(GetPatientRequest) returns (GetPatientResponse);
    rpc ListPatients(ListPatientsRequest) returns (ListPatientsResponse);
    rpc UpdatePatient(UpdatePatientRequest) returns (UpdatePatientResponse);
    rpc SearchPatients(SearchPatientsRequest) returns (SearchPatientsResponse);

    // Clinical encounters
    rpc CreateEncounter(CreateEncounterRequest) returns (CreateEncounterResponse);
    rpc GetEncounter(GetEncounterRequest) returns (GetEncounterResponse);
    rpc ListEncounters(ListEncountersRequest) returns (ListEncountersResponse);
    rpc StartEncounter(StartEncounterRequest) returns (StartEncounterResponse);
    rpc CompleteEncounter(CompleteEncounterRequest) returns (CompleteEncounterResponse);

    // Care plans
    rpc CreateCarePlan(CreateCarePlanRequest) returns (CreateCarePlanResponse);
    rpc GetCarePlan(GetCarePlanRequest) returns (GetCarePlanResponse);
    rpc ListCarePlans(ListCarePlansRequest) returns (ListCarePlansResponse);
    rpc ActivateCarePlan(ActivateCarePlanRequest) returns (ActivateCarePlanResponse);
    rpc UpdateCarePlan(UpdateCarePlanRequest) returns (UpdateCarePlanResponse);

    // Clinical workflows
    rpc CreateWorkflow(CreateWorkflowRequest) returns (CreateWorkflowResponse);
    rpc GetWorkflow(GetWorkflowRequest) returns (GetWorkflowResponse);
    rpc ListWorkflows(ListWorkflowsRequest) returns (ListWorkflowsResponse);
    rpc AdvanceWorkflow(AdvanceWorkflowRequest) returns (AdvanceWorkflowResponse);

    // Health analytics
    rpc GetAnalyticsDashboard(GetAnalyticsDashboardRequest) returns (GetAnalyticsDashboardResponse);
    rpc GetQualityMeasures(GetQualityMeasuresRequest) returns (GetQualityMeasuresResponse);
    rpc GetReadmissionRates(GetReadmissionRatesRequest) returns (GetReadmissionRatesResponse);
}

message CreatePatientRequest {
    string tenant_id = 1;
    string mrn = 2;
    string first_name = 3;
    string last_name = 4;
    string date_of_birth = 5;
    string gender = 6;
    string demographics = 7;
    string insurance = 8;
    string consent = 9;
    string primary_provider_id = 10;
}

message CreatePatientResponse {
    string patient_id = 1;
    string mrn = 2;
    string patient_status = 3;
}

message GetPatientRequest {
    string tenant_id = 1;
    string patient_id = 2;
}

message GetPatientResponse {
    string patient_id = 1;
    string mrn = 2;
    string first_name = 3;
    string last_name = 4;
    string date_of_birth = 5;
    string gender = 6;
    string demographics = 7;
    string insurance = 8;
    string consent = 9;
    string patient_status = 10;
    string primary_provider_id = 11;
}

message ListPatientsRequest {
    string tenant_id = 1;
    string patient_status = 2;
    string primary_provider_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPatientsResponse {
    repeated GetPatientResponse patients = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdatePatientRequest {
    string tenant_id = 1;
    string patient_id = 2;
    string demographics = 3;
    string insurance = 4;
    string consent = 5;
    string patient_status = 6;
}

message UpdatePatientResponse {
    string patient_id = 1;
    string patient_status = 2;
    int32 version = 3;
}

message SearchPatientsRequest {
    string tenant_id = 1;
    string query = 2;
    string search_field = 3;
    int32 page_size = 4;
}

message SearchPatientsResponse {
    repeated GetPatientResponse patients = 1;
    int32 total_count = 2;
}

message CreateEncounterRequest {
    string tenant_id = 1;
    string patient_id = 2;
    string encounter_type = 3;
    string encounter_date = 4;
    string admitting_provider_id = 5;
    string facility_id = 6;
    string department_id = 7;
    string chief_complaint = 8;
}

message CreateEncounterResponse {
    string encounter_id = 1;
    string encounter_db_id = 2;
    string encounter_status = 3;
}

message GetEncounterRequest {
    string tenant_id = 1;
    string encounter_id = 2;
}

message GetEncounterResponse {
    string encounter_db_id = 1;
    string encounter_id = 2;
    string patient_id = 3;
    string encounter_type = 4;
    string encounter_date = 5;
    string admitting_provider_id = 6;
    string facility_id = 7;
    string chief_complaint = 8;
    string diagnosis_codes = 9;
    string procedure_codes = 10;
    string discharge_disposition = 11;
    string encounter_status = 12;
}

message ListEncountersRequest {
    string tenant_id = 1;
    string patient_id = 2;
    string encounter_type = 3;
    string encounter_status = 4;
    string from_date = 5;
    string to_date = 6;
    int32 page_size = 7;
    string page_token = 8;
}

message ListEncountersResponse {
    repeated GetEncounterResponse encounters = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message StartEncounterRequest {
    string tenant_id = 1;
    string encounter_id = 2;
}

message StartEncounterResponse {
    string encounter_id = 1;
    string encounter_status = 2;
}

message CompleteEncounterRequest {
    string tenant_id = 1;
    string encounter_id = 2;
    string diagnosis_codes = 3;
    string procedure_codes = 4;
    string discharge_disposition = 5;
    string encounter_notes = 6;
}

message CompleteEncounterResponse {
    string encounter_id = 1;
    string encounter_status = 2;
    string completed_at = 3;
}

message CreateCarePlanRequest {
    string tenant_id = 1;
    string patient_id = 2;
    string encounter_id = 3;
    string conditions = 4;
    string goals = 5;
    string interventions = 6;
    string medications = 7;
    string care_team = 8;
    string effective_start = 9;
    string effective_end = 10;
    string review_date = 11;
}

message CreateCarePlanResponse {
    string care_plan_id = 1;
    string plan_status = 2;
}

message GetCarePlanRequest {
    string tenant_id = 1;
    string care_plan_id = 2;
}

message GetCarePlanResponse {
    string id = 1;
    string care_plan_id = 2;
    string patient_id = 3;
    string encounter_id = 4;
    string conditions = 5;
    string goals = 6;
    string interventions = 7;
    string medications = 8;
    string care_team = 9;
    string plan_status = 10;
    string effective_start = 11;
    string effective_end = 12;
    string review_date = 13;
}

message ListCarePlansRequest {
    string tenant_id = 1;
    string patient_id = 2;
    string plan_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCarePlansResponse {
    repeated GetCarePlanResponse care_plans = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ActivateCarePlanRequest {
    string tenant_id = 1;
    string care_plan_id = 2;
}

message ActivateCarePlanResponse {
    string care_plan_id = 1;
    string plan_status = 2;
}

message UpdateCarePlanRequest {
    string tenant_id = 1;
    string care_plan_id = 2;
    string goals = 3;
    string interventions = 4;
    string medications = 5;
    string care_team = 6;
    string review_date = 7;
}

message UpdateCarePlanResponse {
    string care_plan_id = 1;
    string plan_status = 2;
    int32 version = 3;
}

message CreateWorkflowRequest {
    string tenant_id = 1;
    string workflow_type = 2;
    string workflow_name = 3;
    string patient_id = 4;
    string encounter_id = 5;
    string steps = 6;
    string assigned_provider_id = 7;
    string priority = 8;
}

message CreateWorkflowResponse {
    string workflow_id = 1;
    string workflow_status = 2;
    int32 total_steps = 3;
}

message GetWorkflowRequest {
    string tenant_id = 1;
    string workflow_id = 2;
}

message GetWorkflowResponse {
    string id = 1;
    string workflow_type = 2;
    string workflow_name = 3;
    string patient_id = 4;
    string encounter_id = 5;
    string steps = 6;
    int32 current_step_index = 7;
    int32 total_steps = 8;
    string assigned_provider_id = 9;
    string priority = 10;
    string workflow_status = 11;
}

message ListWorkflowsRequest {
    string tenant_id = 1;
    string workflow_type = 2;
    string workflow_status = 3;
    string patient_id = 4;
    string priority = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListWorkflowsResponse {
    repeated GetWorkflowResponse workflows = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message AdvanceWorkflowRequest {
    string tenant_id = 1;
    string workflow_id = 2;
    string step_notes = 3;
}

message AdvanceWorkflowResponse {
    string workflow_id = 1;
    int32 current_step_index = 2;
    string workflow_status = 3;
}

message GetAnalyticsDashboardRequest {
    string tenant_id = 1;
    string facility_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetAnalyticsDashboardResponse {
    repeated QualityMeasureSummary measures = 1;
    ReadmissionSummary readmission = 2;
}

message QualityMeasureSummary {
    string measure_name = 1;
    double measure_value = 2;
    double target_value = 3;
    double benchmark_value = 4;
    string quality_measure_type = 5;
}

message ReadmissionSummary {
    double overall_rate = 1;
    double thirty_day_rate = 2;
    int32 total_readmissions = 3;
    int32 total_discharges = 4;
}

message GetQualityMeasuresRequest {
    string tenant_id = 1;
    string facility_id = 2;
    string quality_measure_type = 3;
    string from_date = 4;
    string to_date = 5;
}

message GetQualityMeasuresResponse {
    repeated QualityMeasureDetail measures = 1;
}

message QualityMeasureDetail {
    string id = 1;
    string measure_name = 2;
    double measure_value = 3;
    double target_value = 4;
    int32 numerator = 5;
    int32 denominator = 6;
    string reporting_period = 7;
    double benchmark_value = 8;
}

message GetReadmissionRatesRequest {
    string tenant_id = 1;
    string facility_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetReadmissionRatesResponse {
    repeated ReadmissionDataPoint data_points = 1;
}

message ReadmissionDataPoint {
    string date = 1;
    double rate = 2;
    int32 readmissions = 3;
    int32 discharges = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `auth-service` | User identity, provider credentials, role-based access for PHI |
| `hcm-service` | Provider roster, credentialing status, department assignments |
| `insurance-service` | Eligibility verification, payer rules, pre-authorization requirements |
| `facility-service` | Facility details, bed management, department configurations |
| `security-service` | PHI access audit, data encryption policies, consent enforcement |

### Published To
| Service | Data |
|---------|------|
| `billing-service` | Encounter data for claims generation, procedure and diagnosis codes |
| `reporting-service` | Quality measures, encounter volumes, care plan adherence dashboards |
| `notification-service` | Patient admission/discharge alerts, care plan review reminders |
| `analytics-service` | Aggregated health metrics, readmission analytics, population health data |
| `compliance-service` | Regulatory reporting data, consent audit trails, PHI access logs |
| `workflow-service` | Clinical workflow escalations, referral routing, care coordination |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `health.patient.admitted` | `health.events` | `{ patient_id, encounter_id, encounter_type, facility_id, admitting_provider_id, encounter_date }` | Patient admitted for a clinical encounter |
| `health.encounter.completed` | `health.events` | `{ encounter_id, patient_id, encounter_type, diagnosis_codes, procedure_codes, discharge_disposition }` | Clinical encounter completed |
| `health.care_plan.updated` | `health.events` | `{ care_plan_id, patient_id, plan_status, updated_fields, version }` | Care plan modified or status changed |
| `health.discharge.ready` | `health.events` | `{ patient_id, encounter_id, facility_id, discharge_disposition, pending_items }` | Patient cleared for discharge |

---

## 8. Migrations

1. V001: `oh_patient_records`
2. V002: `oh_clinical_encounters`
3. V003: `oh_care_plans`
4. V004: `oh_clinical_workflows`
5. V005: `oh_health_analytics`
6. V006: Triggers for `updated_at`
