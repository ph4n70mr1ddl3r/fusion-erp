# 76 - Workforce Safety Service Specification

## 1. Domain Overview
**Bounded Context:** Workforce Safety - Manages workplace safety incidents, investigations, corrective actions, safety inspections, permits, hazard assessments, OSHA compliance, return-to-work programs, and safety training records to ensure a safe working environment.

**Service Name:** safety-service
**Database:** safety_db
**HTTP Port:** 8108 | **gRPC Port:** 9108

## 2. Database Schema

```sql
CREATE TABLE safety_incidents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    incident_number TEXT NOT NULL,
    incident_type TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'LOW',
    status TEXT NOT NULL DEFAULT 'REPORTED',
    incident_date TEXT NOT NULL,
    incident_time TEXT,
    reported_date TEXT NOT NULL,
    location_id TEXT,
    department_id TEXT NOT NULL,
    description TEXT NOT NULL,
    immediate_actions_taken TEXT,
    involved_employee_ids TEXT NOT NULL,
    witness_ids TEXT,
    supervisor_id TEXT NOT NULL,
    reporter_id TEXT NOT NULL,
    is_injury INTEGER NOT NULL DEFAULT 0,
    is_illness INTEGER NOT NULL DEFAULT 0,
    is_near_miss INTEGER NOT NULL DEFAULT 0,
    is_property_damage INTEGER NOT NULL DEFAULT 0,
    is_environmental INTEGER NOT NULL DEFAULT 0,
    body_parts_affected TEXT,
    injury_type TEXT,
    days_away_from_work INTEGER DEFAULT 0,
    days_restricted_work INTEGER DEFAULT 0,
    property_damage_estimate_cents INTEGER DEFAULT 0,
    root_cause TEXT,
    osha_recordable INTEGER NOT NULL DEFAULT 0,
    osha_classification TEXT,
    regulatory_body TEXT,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    attachments TEXT
);

CREATE INDEX idx_incidents_tenant ON safety_incidents(tenant_id);
CREATE INDEX idx_incidents_number ON safety_incidents(incident_number);
CREATE INDEX idx_incidents_date ON safety_incidents(incident_date);
CREATE INDEX idx_incidents_status ON safety_incidents(status);
CREATE INDEX idx_incidents_severity ON safety_incidents(severity);
CREATE INDEX idx_incidents_department ON safety_incidents(department_id);
CREATE INDEX idx_incidents_osha ON safety_incidents(osha_recordable);

CREATE TABLE incident_investigations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    incident_id TEXT NOT NULL,
    investigation_type TEXT NOT NULL,
    lead_investigator_id TEXT NOT NULL,
    team_member_ids TEXT,
    start_date TEXT NOT NULL,
    completion_date TEXT,
    status TEXT NOT NULL DEFAULT 'IN_PROGRESS',
    methodology TEXT,
    findings TEXT,
    root_cause_analysis TEXT,
    contributing_factors TEXT,
    timeline_events TEXT,
    evidence_collected TEXT,
    recommendations TEXT,
    lessons_learned TEXT,
    review_date TEXT,
    reviewed_by TEXT,
    FOREIGN KEY (incident_id) REFERENCES safety_incidents(id)
);

CREATE INDEX idx_investigations_tenant ON incident_investigations(tenant_id);
CREATE INDEX idx_investigations_incident ON incident_investigations(incident_id);
CREATE INDEX idx_investigations_status ON incident_investigations(status);

CREATE TABLE corrective_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    incident_id TEXT NOT NULL,
    investigation_id TEXT,
    action_type TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    priority TEXT NOT NULL DEFAULT 'MEDIUM',
    status TEXT NOT NULL DEFAULT 'PLANNED',
    assigned_to TEXT NOT NULL,
    assigned_date TEXT NOT NULL,
    target_completion_date TEXT NOT NULL,
    actual_completion_date TEXT,
    verification_method TEXT,
    verified_by TEXT,
    verified_date TEXT,
    effectiveness_rating INTEGER,
    cost_cents INTEGER DEFAULT 0,
    progress_percentage INTEGER DEFAULT 0,
    barriers TEXT,
    outcome_notes TEXT,
    FOREIGN KEY (incident_id) REFERENCES safety_incidents(id)
);

CREATE INDEX idx_corrective_tenant ON corrective_actions(tenant_id);
CREATE INDEX idx_corrective_incident ON corrective_actions(incident_id);
CREATE INDEX idx_corrective_status ON corrective_actions(status);
CREATE INDEX idx_corrective_assignee ON corrective_actions(assigned_to);

CREATE TABLE safety_inspections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    inspection_name TEXT NOT NULL,
    inspection_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'SCHEDULED',
    scheduled_date TEXT NOT NULL,
    completed_date TEXT,
    inspector_id TEXT NOT NULL,
    inspector_name TEXT,
    location_id TEXT,
    department_id TEXT,
    checklist_id TEXT,
    total_items_checked INTEGER DEFAULT 0,
    items_passed INTEGER DEFAULT 0,
    items_failed INTEGER DEFAULT 0,
    items_not_applicable INTEGER DEFAULT 0,
    overall_score INTEGER,
    follow_up_required INTEGER NOT NULL DEFAULT 0,
    follow_up_date TEXT,
    notes TEXT,
    attachments TEXT
);

CREATE INDEX idx_inspections_tenant ON safety_inspections(tenant_id);
CREATE INDEX idx_inspections_date ON safety_inspections(scheduled_date);
CREATE INDEX idx_inspections_status ON safety_inspections(status);
CREATE INDEX idx_inspections_inspector ON safety_inspections(inspector_id);

CREATE TABLE inspection_findings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    inspection_id TEXT NOT NULL,
    checklist_item_id TEXT,
    category TEXT NOT NULL,
    finding_text TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'LOW',
    compliance_status TEXT NOT NULL DEFAULT 'NON_COMPLIANT',
    evidence TEXT,
    location_detail TEXT,
    corrective_action_required INTEGER NOT NULL DEFAULT 0,
    corrective_action_id TEXT,
    resolved INTEGER NOT NULL DEFAULT 0,
    resolved_date TEXT,
    FOREIGN KEY (inspection_id) REFERENCES safety_inspections(id)
);

CREATE INDEX idx_findings_tenant ON inspection_findings(tenant_id);
CREATE INDEX idx_findings_inspection ON inspection_findings(inspection_id);
CREATE INDEX idx_findings_severity ON inspection_findings(severity);

CREATE TABLE safety_permits (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    permit_number TEXT NOT NULL,
    permit_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE',
    issued_date TEXT NOT NULL,
    expiry_date TEXT NOT NULL,
    issuing_authority TEXT,
    location_id TEXT,
    department_id TEXT,
    authorized_activities TEXT,
    conditions TEXT,
    restrictions TEXT,
    responsible_person_id TEXT NOT NULL,
    renewal_required INTEGER NOT NULL DEFAULT 1,
    renewal_date TEXT,
    inspection_required INTEGER NOT NULL DEFAULT 0,
    attachments TEXT,
    notes TEXT
);

CREATE INDEX idx_permits_tenant ON safety_permits(tenant_id);
CREATE INDEX idx_permits_expiry ON safety_permits(expiry_date);
CREATE INDEX idx_permits_status ON safety_permits(status);
CREATE INDEX idx_permits_type ON safety_permits(permit_type);

CREATE TABLE hazard_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    assessment_name TEXT NOT NULL,
    assessment_type TEXT NOT NULL,
    location_id TEXT,
    department_id TEXT,
    job_id TEXT,
    assessor_id TEXT NOT NULL,
    assessment_date TEXT NOT NULL,
    review_date TEXT,
    hazard_category TEXT NOT NULL,
    risk_probability TEXT NOT NULL,
    risk_impact TEXT NOT NULL,
    risk_score INTEGER NOT NULL,
    risk_level TEXT NOT NULL,
    existing_controls TEXT,
    additional_controls_needed TEXT,
    residual_risk_score INTEGER,
    residual_risk_level TEXT,
    affected_employee_count INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    approved_by TEXT,
    approved_at TEXT
);

CREATE INDEX idx_hazards_tenant ON hazard_assessments(tenant_id);
CREATE INDEX idx_hazards_risk_level ON hazard_assessments(risk_level);
CREATE INDEX idx_hazards_department ON hazard_assessments(department_id);

CREATE TABLE osha_log_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    log_year INTEGER NOT NULL,
    entry_number INTEGER NOT NULL,
    incident_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    employee_name TEXT,
    job_title TEXT,
    event_date TEXT NOT NULL,
    case_type TEXT NOT NULL,
    description TEXT NOT NULL,
    classification TEXT,
    body_part TEXT,
    source_of_injury TEXT,
    days_away INTEGER DEFAULT 0,
    days_restricted INTEGER DEFAULT 0,
    is_fatality INTEGER NOT NULL DEFAULT 0,
    is_hospitalization INTEGER NOT NULL DEFAULT 0,
    is_amputation INTEGER NOT NULL DEFAULT 0,
    is_loss_of_eye INTEGER NOT NULL DEFAULT 0,
    physician_name TEXT,
    facility_name TEXT,
    treated_in_er INTEGER NOT NULL DEFAULT 0,
    hospitalized_overnight INTEGER NOT NULL DEFAULT 0,
    establishment_id TEXT,
    naics_code TEXT,
    FOREIGN KEY (incident_id) REFERENCES safety_incidents(id)
);

CREATE INDEX idx_osha_tenant ON osha_log_entries(tenant_id);
CREATE INDEX idx_osha_year ON osha_log_entries(log_year);
CREATE INDEX idx_osha_incident ON osha_log_entries(incident_id);
CREATE INDEX idx_osha_employee ON osha_log_entries(employee_id);

CREATE TABLE return_to_work_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    incident_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE',
    start_date TEXT NOT NULL,
    target_full_duty_date TEXT,
    actual_return_date TEXT,
    restrictions TEXT,
    modified_duties TEXT,
    max_hours_per_day INTEGER,
    restricted_activities TEXT,
    physician_id TEXT,
    physician_clearance_date TEXT,
    supervisor_id TEXT NOT NULL,
    hr_coordinator_id TEXT,
    check_in_frequency_days INTEGER DEFAULT 7,
    last_check_in_date TEXT,
    next_check_in_date TEXT,
    progress_notes TEXT,
    accommodations TEXT,
    FOREIGN KEY (incident_id) REFERENCES safety_incidents(id)
);

CREATE INDEX idx_rtw_tenant ON return_to_work_plans(tenant_id);
CREATE INDEX idx_rtw_incident ON return_to_work_plans(incident_id);
CREATE INDEX idx_rtw_employee ON return_to_work_plans(employee_id);
CREATE INDEX idx_rtw_status ON return_to_work_plans(status);

CREATE TABLE safety_training_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    training_name TEXT NOT NULL,
    training_code TEXT NOT NULL,
    training_type TEXT NOT NULL,
    description TEXT,
    duration_hours DECIMAL(5,2) NOT NULL,
    provider TEXT,
    completion_date TEXT NOT NULL,
    expiry_date TEXT,
    employee_id TEXT NOT NULL,
    employee_name TEXT,
    status TEXT NOT NULL DEFAULT 'COMPLETED',
    score DECIMAL(5,2),
    max_score DECIMAL(5,2),
    passed INTEGER NOT NULL DEFAULT 1,
    certificate_number TEXT,
    certificate_url TEXT,
    department_id TEXT,
    location_id TEXT,
    recertification_required INTEGER NOT NULL DEFAULT 0,
    recertification_date TEXT,
    delivery_method TEXT,
    instructor_id TEXT
);

CREATE INDEX idx_training_tenant ON safety_training_records(tenant_id);
CREATE INDEX idx_training_employee ON safety_training_records(employee_id);
CREATE INDEX idx_training_type ON safety_training_records(training_type);
CREATE INDEX idx_training_expiry ON safety_training_records(expiry_date);
CREATE INDEX idx_training_code ON safety_training_records(training_code);
```

## 3. REST API Endpoints

### Safety Incidents
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/safety-incidents` | Report a new safety incident |
| GET | `/api/v1/safety-incidents` | List incidents with filters |
| GET | `/api/v1/safety-incidents/{id}` | Get incident details |
| PUT | `/api/v1/safety-incidents/{id}` | Update incident |
| PATCH | `/api/v1/safety-incidents/{id}/status` | Update incident status |
| GET | `/api/v1/safety-incidents/{id}/timeline` | Get incident timeline |

### Investigations
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/incident-investigations` | Start an investigation |
| GET | `/api/v1/incident-investigations` | List investigations |
| GET | `/api/v1/incident-investigations/{id}` | Get investigation details |
| PUT | `/api/v1/incident-investigations/{id}` | Update investigation |
| POST | `/api/v1/incident-investigations/{id}/complete` | Complete investigation |

### Corrective Actions
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/corrective-actions` | Create a corrective action |
| GET | `/api/v1/corrective-actions` | List corrective actions |
| GET | `/api/v1/corrective-actions/{id}` | Get corrective action details |
| PUT | `/api/v1/corrective-actions/{id}` | Update corrective action |
| POST | `/api/v1/corrective-actions/{id}/progress` | Update progress |
| POST | `/api/v1/corrective-actions/{id}/verify` | Verify corrective action |
| POST | `/api/v1/corrective-actions/{id}/complete` | Mark corrective action complete |

### Safety Inspections
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/safety-inspections` | Schedule a safety inspection |
| GET | `/api/v1/safety-inspections` | List inspections |
| GET | `/api/v1/safety-inspections/{id}` | Get inspection details |
| PUT | `/api/v1/safety-inspections/{id}` | Update inspection |
| POST | `/api/v1/safety-inspections/{id}/conduct` | Record inspection findings |
| POST | `/api/v1/safety-inspections/{id}/complete` | Complete inspection |
| GET | `/api/v1/safety-inspections/{id}/findings` | Get inspection findings |

### Safety Permits
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/safety-permits` | Issue a safety permit |
| GET | `/api/v1/safety-permits` | List permits |
| GET | `/api/v1/safety-permits/{id}` | Get permit details |
| PUT | `/api/v1/safety-permits/{id}` | Update permit |
| POST | `/api/v1/safety-permits/{id}/renew` | Renew permit |
| POST | `/api/v1/safety-permits/{id}/revoke` | Revoke permit |
| GET | `/api/v1/safety-permits/expiring` | Get expiring permits |

### Hazard Assessments
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/hazard-assessments` | Create hazard assessment |
| GET | `/api/v1/hazard-assessments` | List hazard assessments |
| GET | `/api/v1/hazard-assessments/{id}` | Get assessment details |
| PUT | `/api/v1/hazard-assessments/{id}` | Update assessment |
| POST | `/api/v1/hazard-assessments/{id}/approve` | Approve assessment |
| GET | `/api/v1/hazard-assessments/risk-matrix` | Get risk matrix |

### OSHA Reports
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/osha-log-entries` | Create OSHA log entry |
| GET | `/api/v1/osha-log-entries` | List OSHA log entries |
| GET | `/api/v1/osha-log-entries/{id}` | Get OSHA entry details |
| PUT | `/api/v1/osha-log-entries/{id}` | Update OSHA entry |
| GET | `/api/v1/osha-reports/300-log` | Generate OSHA 300 Log |
| GET | `/api/v1/osha-reports/300a-summary` | Generate OSHA 300A Summary |
| GET | `/api/v1/osha-reports/301-report` | Generate OSHA 301 Report |

### Return to Work & Training
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/return-to-work-plans` | Create return-to-work plan |
| GET | `/api/v1/return-to-work-plans` | List return-to-work plans |
| GET | `/api/v1/return-to-work-plans/{id}` | Get plan details |
| PUT | `/api/v1/return-to-work-plans/{id}` | Update plan |
| POST | `/api/v1/return-to-work-plans/{id}/check-in` | Record check-in |
| POST | `/api/v1/return-to-work-plans/{id}/complete` | Complete return-to-work |
| POST | `/api/v1/safety-training-records` | Record safety training |
| GET | `/api/v1/safety-training-records` | List training records |
| GET | `/api/v1/safety-training-records/{id}` | Get training record |
| GET | `/api/v1/safety-analytics/dashboard` | Get safety analytics dashboard |

## 4. Business Rules

1. An incident number MUST be auto-generated with format SI-YYYYMMDD-NNNNN on creation.
2. All incidents with severity HIGH or CRITICAL MUST trigger an automatic investigation within 24 hours.
3. The system MUST classify incidents as OSHA recordable when they meet OSHA criteria (days away, restricted work, medical treatment beyond first aid, loss of consciousness, or diagnosis by healthcare professional).
4. A corrective action MUST be linked to an incident and optionally to an investigation.
5. The system MUST NOT allow closure of an incident with incomplete corrective actions.
6. OSHA 300 log entries MUST be created within 7 calendar days of the incident being reported.
7. Safety permits approaching expiry within 30 days SHOULD trigger renewal notifications.
8. Risk scores in hazard assessments MUST be calculated as probability (1-5) multiplied by impact (1-5).
9. Risk levels MUST be categorized: 1-4 LOW, 5-9 MEDIUM, 10-15 HIGH, 16-25 CRITICAL.
10. Return-to-work plans MUST include physician clearance before full duty restoration.
11. Safety inspections MUST follow the assigned checklist and record all findings.
12. The system MUST maintain a complete audit trail of all incident status changes.
13. Employees working in hazardous areas MUST have valid safety training records; expired training MUST be flagged.
14. OSHA fatality reports MUST be filed within 8 hours; hospitalization, amputation, or eye loss within 24 hours.
15. The system SHOULD automatically generate OSHA 300A summary from 300 log entries at year end.

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package safety;

service SafetyService {
    // Safety Incidents
    rpc ReportIncident(ReportIncidentRequest) returns (IncidentResponse);
    rpc GetIncident(GetByIdRequest) returns (IncidentResponse);
    rpc ListIncidents(ListIncidentsRequest) returns (IncidentListResponse);
    rpc UpdateIncident(UpdateIncidentRequest) returns (IncidentResponse);
    rpc UpdateIncidentStatus(UpdateIncidentStatusRequest) returns (IncidentResponse);

    // Investigations
    rpc StartInvestigation(StartInvestigationRequest) returns (InvestigationResponse);
    rpc GetInvestigation(GetByIdRequest) returns (InvestigationResponse);
    rpc ListInvestigations(ListInvestigationsRequest) returns (InvestigationListResponse);
    rpc CompleteInvestigation(CompleteInvestigationRequest) returns (InvestigationResponse);

    // Corrective Actions
    rpc CreateCorrectiveAction(CreateCorrectiveActionRequest) returns (CorrectiveActionResponse);
    rpc GetCorrectiveAction(GetByIdRequest) returns (CorrectiveActionResponse);
    rpc ListCorrectiveActions(ListCorrectiveActionsRequest) returns (CorrectiveActionListResponse);
    rpc UpdateCorrectiveActionProgress(UpdateCAProgressRequest) returns (CorrectiveActionResponse);
    rpc VerifyCorrectiveAction(VerifyCorrectiveActionRequest) returns (CorrectiveActionResponse);

    // Safety Inspections
    rpc ScheduleInspection(ScheduleInspectionRequest) returns (InspectionResponse);
    rpc GetInspection(GetByIdRequest) returns (InspectionResponse);
    rpc ListInspections(ListInspectionsRequest) returns (InspectionListResponse);
    rpc CompleteInspection(CompleteInspectionRequest) returns (InspectionResponse);
    rpc GetInspectionFindings(GetByIdRequest) returns (InspectionFindingsResponse);

    // Safety Permits
    rpc IssuePermit(IssuePermitRequest) returns (PermitResponse);
    rpc GetPermit(GetByIdRequest) returns (PermitResponse);
    rpc ListPermits(ListPermitsRequest) returns (PermitListResponse);
    rpc RenewPermit(RenewPermitRequest) returns (PermitResponse);
    rpc RevokePermit(RevokePermitRequest) returns (PermitResponse);

    // Hazard Assessments
    rpc CreateHazardAssessment(CreateHazardAssessmentRequest) returns (HazardAssessmentResponse);
    rpc GetHazardAssessment(GetByIdRequest) returns (HazardAssessmentResponse);
    rpc ApproveHazardAssessment(ApproveHazardAssessmentRequest) returns (HazardAssessmentResponse);

    // OSHA Reports
    rpc CreateOSHALogEntry(CreateOSHALogEntryRequest) returns (OSHALogEntryResponse);
    rpc GenerateOSHA300Log(GenerateOSHAReportRequest) returns (OSHAReportResponse);
    rpc GenerateOSHA300ASummary(GenerateOSHAReportRequest) returns (OSHAReportResponse);
    rpc GenerateOSHA301Report(GenerateOSHAReportRequest) returns (OSHAReportResponse);

    // Return to Work
    rpc CreateReturnToWorkPlan(CreateReturnToWorkPlanRequest) returns (ReturnToWorkResponse);
    rpc GetReturnToWorkPlan(GetByIdRequest) returns (ReturnToWorkResponse);
    rpc RecordCheckIn(RecordCheckInRequest) returns (ReturnToWorkResponse);
    rpc CompleteReturnToWork(CompleteReturnToWorkRequest) returns (ReturnToWorkResponse);

    // Training Records
    rpc RecordSafetyTraining(RecordSafetyTrainingRequest) returns (SafetyTrainingResponse);
    rpc ListSafetyTraining(ListSafetyTrainingRequest) returns (SafetyTrainingListResponse);
}

message GetByIdRequest {
    string id = 1;
    string tenant_id = 2;
}

message ReportIncidentRequest {
    string tenant_id = 1;
    string incident_type = 2;
    string severity = 3;
    string incident_date = 4;
    string incident_time = 5;
    string location_id = 6;
    string department_id = 7;
    string description = 8;
    string immediate_actions_taken = 9;
    repeated string involved_employee_ids = 10;
    string supervisor_id = 11;
    string reporter_id = 12;
    bool is_injury = 13;
    bool is_illness = 14;
    bool is_near_miss = 15;
}

message IncidentResponse {
    string id = 1;
    string incident_number = 2;
    string incident_type = 3;
    string severity = 4;
    string status = 5;
    string incident_date = 6;
    string department_id = 7;
    bool osha_recordable = 8;
    string created_at = 9;
    int32 version = 10;
}

message ListIncidentsRequest {
    string tenant_id = 1;
    string status = 2;
    string severity = 3;
    string department_id = 4;
    string start_date = 5;
    string end_date = 6;
    int32 page = 7;
    int32 page_size = 8;
}

message IncidentListResponse {
    repeated IncidentResponse items = 1;
    int32 total_count = 2;
}

message UpdateIncidentRequest {
    string id = 1;
    string tenant_id = 2;
    string severity = 3;
    string description = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateIncidentStatusRequest {
    string id = 1;
    string tenant_id = 2;
    string status = 3;
    string updated_by = 4;
}

message StartInvestigationRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string investigation_type = 3;
    string lead_investigator_id = 4;
    string methodology = 5;
    string created_by = 6;
}

message InvestigationResponse {
    string id = 1;
    string incident_id = 2;
    string investigation_type = 3;
    string lead_investigator_id = 4;
    string status = 5;
    string start_date = 6;
    int32 version = 7;
}

message ListInvestigationsRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string status = 3;
    int32 page = 4;
    int32 page_size = 5;
}

message InvestigationListResponse {
    repeated InvestigationResponse items = 1;
    int32 total_count = 2;
}

message CompleteInvestigationRequest {
    string id = 1;
    string tenant_id = 2;
    string findings = 3;
    string root_cause_analysis = 4;
    string recommendations = 5;
    string completed_by = 6;
}

message CreateCorrectiveActionRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string investigation_id = 3;
    string action_type = 4;
    string title = 5;
    string description = 6;
    string priority = 7;
    string assigned_to = 8;
    string target_completion_date = 9;
    int64 cost_cents = 10;
    string created_by = 11;
}

message CorrectiveActionResponse {
    string id = 1;
    string incident_id = 2;
    string title = 3;
    string status = 4;
    string assigned_to = 5;
    string target_completion_date = 6;
    int32 progress_percentage = 7;
    int32 version = 8;
}

message ListCorrectiveActionsRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string status = 3;
    string assigned_to = 4;
    int32 page = 5;
    int32 page_size = 6;
}

message CorrectiveActionListResponse {
    repeated CorrectiveActionResponse items = 1;
    int32 total_count = 2;
}

message UpdateCAProgressRequest {
    string id = 1;
    string tenant_id = 2;
    int32 progress_percentage = 3;
    string notes = 4;
    string updated_by = 5;
}

message VerifyCorrectiveActionRequest {
    string id = 1;
    string tenant_id = 2;
    string verification_method = 3;
    int32 effectiveness_rating = 4;
    string verified_by = 5;
}

message ScheduleInspectionRequest {
    string tenant_id = 1;
    string inspection_name = 2;
    string inspection_type = 3;
    string scheduled_date = 4;
    string inspector_id = 5;
    string location_id = 6;
    string department_id = 7;
    string created_by = 8;
}

message InspectionResponse {
    string id = 1;
    string inspection_name = 2;
    string inspection_type = 3;
    string status = 4;
    string scheduled_date = 5;
    int32 overall_score = 6;
    int32 version = 7;
}

message ListInspectionsRequest {
    string tenant_id = 1;
    string status = 2;
    string department_id = 3;
    int32 page = 4;
    int32 page_size = 5;
}

message InspectionListResponse {
    repeated InspectionResponse items = 1;
    int32 total_count = 2;
}

message CompleteInspectionRequest {
    string id = 1;
    string tenant_id = 2;
    int32 total_items_checked = 3;
    int32 items_passed = 4;
    int32 items_failed = 5;
    string notes = 6;
    string completed_by = 7;
}

message InspectionFindingsResponse {
    string inspection_id = 1;
    repeated FindingDetail findings = 2;
}

message FindingDetail {
    string id = 1;
    string category = 2;
    string finding_text = 3;
    string severity = 4;
    string compliance_status = 5;
}

message IssuePermitRequest {
    string tenant_id = 1;
    string permit_type = 2;
    string issued_date = 3;
    string expiry_date = 4;
    string location_id = 5;
    string responsible_person_id = 6;
    string authorized_activities = 7;
    string conditions = 8;
    string created_by = 9;
}

message PermitResponse {
    string id = 1;
    string permit_number = 2;
    string permit_type = 3;
    string status = 4;
    string issued_date = 5;
    string expiry_date = 6;
    int32 version = 7;
}

message ListPermitsRequest {
    string tenant_id = 1;
    string status = 2;
    string permit_type = 3;
    int32 page = 4;
    int32 page_size = 5;
}

message PermitListResponse {
    repeated PermitResponse items = 1;
    int32 total_count = 2;
}

message RenewPermitRequest {
    string id = 1;
    string tenant_id = 2;
    string new_expiry_date = 3;
    string renewed_by = 4;
}

message RevokePermitRequest {
    string id = 1;
    string tenant_id = 2;
    string reason = 3;
    string revoked_by = 4;
}

message CreateHazardAssessmentRequest {
    string tenant_id = 1;
    string assessment_name = 2;
    string assessment_type = 3;
    string location_id = 4;
    string department_id = 5;
    string assessor_id = 6;
    string hazard_category = 7;
    string risk_probability = 8;
    string risk_impact = 9;
    string existing_controls = 10;
    string created_by = 11;
}

message HazardAssessmentResponse {
    string id = 1;
    string assessment_name = 2;
    string hazard_category = 3;
    int32 risk_score = 4;
    string risk_level = 5;
    string status = 6;
    int32 version = 7;
}

message ApproveHazardAssessmentRequest {
    string id = 1;
    string tenant_id = 2;
    string approved_by = 3;
}

message CreateOSHALogEntryRequest {
    string tenant_id = 1;
    int32 log_year = 2;
    string incident_id = 3;
    string employee_id = 4;
    string case_type = 5;
    string description = 6;
    string created_by = 7;
}

message OSHALogEntryResponse {
    string id = 1;
    int32 log_year = 2;
    int32 entry_number = 3;
    string case_type = 4;
    string event_date = 5;
    int32 days_away = 6;
}

message GenerateOSHAReportRequest {
    string tenant_id = 1;
    int32 log_year = 2;
    string establishment_id = 3;
}

message OSHAReportResponse {
    int32 log_year = 1;
    string report_type = 2;
    int32 total_entries = 3;
    string report_data = 4;
}

message CreateReturnToWorkPlanRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string employee_id = 3;
    string plan_type = 4;
    string start_date = 5;
    string target_full_duty_date = 6;
    string restrictions = 7;
    string modified_duties = 8;
    string supervisor_id = 9;
    string created_by = 10;
}

message ReturnToWorkResponse {
    string id = 1;
    string incident_id = 2;
    string employee_id = 3;
    string status = 4;
    string start_date = 5;
    string target_full_duty_date = 6;
    int32 version = 7;
}

message RecordCheckInRequest {
    string id = 1;
    string tenant_id = 2;
    string progress_notes = 3;
    string accommodations = 4;
    string recorded_by = 5;
}

message CompleteReturnToWorkRequest {
    string id = 1;
    string tenant_id = 2;
    string actual_return_date = 3;
    string physician_clearance_date = 4;
    string completed_by = 5;
}

message RecordSafetyTrainingRequest {
    string tenant_id = 1;
    string training_name = 2;
    string training_code = 3;
    string training_type = 4;
    double duration_hours = 5;
    string completion_date = 6;
    string employee_id = 7;
    double score = 8;
    bool passed = 9;
    string created_by = 10;
}

message SafetyTrainingResponse {
    string id = 1;
    string training_name = 2;
    string training_code = 3;
    string employee_id = 4;
    string completion_date = 5;
    string expiry_date = 6;
    bool passed = 7;
    int32 version = 8;
}

message ListSafetyTrainingRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string training_code = 3;
    bool expiring_soon = 4;
    int32 page = 5;
    int32 page_size = 6;
}

message SafetyTrainingListResponse {
    repeated SafetyTrainingResponse items = 1;
    int32 total_count = 2;
}
```

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data/Events | Purpose |
|---------------|-------------|---------|
| CORE-HR | Employee profiles, departments, locations, job assignments | Incident reporter/employee validation, org context |
| LEARNING | Training completions, certifications, course catalog | Safety training requirements, certification tracking |
| EAM | Asset information, maintenance schedules, equipment status | Equipment-related incidents, hazard assessments for assets |

### Published To
| Target Service | Data/Events | Purpose |
|---------------|-------------|---------|
| CORE-HR | Incident records, return-to-work status | Employee safety records, work restrictions |
| LEARNING | Training needs, recertification requirements | Trigger safety training assignments |
| NOTIFICATION | Incident alerts, permit expirations, training deadlines | Safety awareness notifications |
| EAM | Equipment incidents, maintenance triggers | Asset safety-related work orders |

## 7. Events

### Produced Events
| Event | Payload | Description |
|-------|---------|-------------|
| `IncidentReported` | `{ incident_id, incident_number, tenant_id, severity, department_id, incident_type, reporter_id }` | Emitted when a safety incident is reported |
| `InvestigationCompleted` | `{ investigation_id, incident_id, tenant_id, root_cause, recommendations }` | Emitted when an investigation is completed |
| `CorrectiveActionAssigned` | `{ corrective_action_id, incident_id, tenant_id, assigned_to, priority, target_date }` | Emitted when a corrective action is assigned |
| `OSHAReportGenerated` | `{ tenant_id, log_year, report_type, total_entries, establishment_id }` | Emitted when an OSHA report is generated |
| `InspectionCompleted` | `{ inspection_id, tenant_id, overall_score, items_failed, follow_up_required }` | Emitted when a safety inspection is completed |
| `PermitExpiring` | `{ permit_id, permit_number, tenant_id, expiry_date, responsible_person_id }` | Emitted when a safety permit is approaching expiry |
| `ReturnToWorkCompleted` | `{ rtw_plan_id, incident_id, employee_id, tenant_id, actual_return_date }` | Emitted when a return-to-work plan is completed |
| `TrainingExpired` | `{ training_record_id, employee_id, training_code, tenant_id, expiry_date }` | Emitted when a safety training certification expires |
| `HazardIdentified` | `{ assessment_id, tenant_id, risk_level, department_id, hazard_category }` | Emitted when a high or critical hazard is identified |
