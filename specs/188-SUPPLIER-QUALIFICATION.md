# 188 - Supplier Qualification Service Specification

## 1. Domain Overview

Supplier Qualification provides end-to-end supplier onboarding and compliance management covering qualification program definition with questionnaire and certification requirements, structured questionnaire management with sections and scoring methodologies, supplier response capture with automated scoring and approval workflows, certification tracking across ISO, safety, quality, and regulatory categories, and disqualification management with root cause analysis and corrective action tracking. The system ensures that only qualified, compliant suppliers participate in procurement activities and enables continuous monitoring of supplier certifications and performance standards. Integrates with Purchasing for approved supplier lists, HCM for internal evaluators, and Risk for supplier risk scoring.

**Bounded Context:** Supplier Onboarding & Compliance Qualification
**Service Name:** `supplier-qualification-service`
**Database:** `data/supplier_qualification.db`
**HTTP Port:** 8206 | **gRPC Port:** 9206

---

## 2. Database Schema

### 2.1 Qualification Programs
```sql
CREATE TABLE sq_qualification_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_name TEXT NOT NULL,
    description TEXT,
    program_type TEXT NOT NULL
        CHECK(program_type IN ('NEW_SUPPLIER','PERIODIC_REVIEW','CATEGORY_SPECIFIC','REGULATORY')),
    questionnaire_ids TEXT,                          -- JSON: array of required questionnaire IDs
    required_certifications TEXT,                    -- JSON: array of required certification types and levels
    evaluation_criteria TEXT NOT NULL,               -- JSON: scoring thresholds for pass/conditional/fail
    min_score_threshold REAL NOT NULL DEFAULT 70.0
        CHECK(min_score_threshold >= 0.0 AND min_score_threshold <= 100.0),
    validity_period_days INTEGER NOT NULL DEFAULT 365
        CHECK(validity_period_days > 0),
    auto_approve INTEGER NOT NULL DEFAULT 0,
    assigned_evaluator_ids TEXT,                     -- JSON: array of internal evaluator user IDs
    program_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(program_status IN ('DRAFT','ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_code)
);

CREATE INDEX idx_qual_programs_tenant_type ON sq_qualification_programs(tenant_id, program_type);
CREATE INDEX idx_qual_programs_tenant_status ON sq_qualification_programs(tenant_id, program_status);
CREATE INDEX idx_qual_programs_tenant_active ON sq_qualification_programs(tenant_id, is_active);
```

### 2.2 Questionnaires
```sql
CREATE TABLE sq_questionnaires (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    questionnaire_code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    category TEXT NOT NULL
        CHECK(category IN ('GENERAL','FINANCIAL','TECHNICAL','REGULATORY','SUSTAINABILITY','SAFETY')),
    sections TEXT NOT NULL,                          -- JSON: ordered array of sections with question groups
    scoring_model TEXT NOT NULL DEFAULT 'WEIGHTED'
        CHECK(scoring_model IN ('WEIGHTED','SIMPLE','PASS_FAIL','RUBRIC')),
    total_points REAL NOT NULL DEFAULT 0.0
        CHECK(total_points >= 0.0),
    passing_score REAL NOT NULL DEFAULT 70.0
        CHECK(passing_score >= 0.0 AND passing_score <= 100.0),
    estimated_duration_minutes INTEGER NOT NULL DEFAULT 30
        CHECK(estimated_duration_minutes > 0),
    questionnaire_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(questionnaire_status IN ('DRAFT','PUBLISHED','RETIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, questionnaire_code)
);

CREATE INDEX idx_questionnaires_tenant_category ON sq_questionnaires(tenant_id, category);
CREATE INDEX idx_questionnaires_tenant_status ON sq_questionnaires(tenant_id, questionnaire_status);
CREATE INDEX idx_questionnaires_tenant_active ON sq_questionnaires(tenant_id, is_active);
```

### 2.3 Supplier Responses
```sql
CREATE TABLE sq_supplier_responses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    response_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    questionnaire_id TEXT NOT NULL,
    responses TEXT NOT NULL,                         -- JSON: array of question-answer pairs with scores
    total_score REAL NOT NULL DEFAULT 0.0,
    max_possible_score REAL NOT NULL DEFAULT 0.0,
    score_percentage REAL NOT NULL DEFAULT 0.0,
    evaluator_id TEXT,
    evaluator_comments TEXT,
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','UNDER_REVIEW','APPROVED','CONDITIONALLY_APPROVED','REJECTED')),
    submitted_at TEXT,
    reviewed_at TEXT,
    qualification_valid_until TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES sq_qualification_programs(id),
    UNIQUE(tenant_id, response_id)
);

CREATE INDEX idx_supplier_responses_tenant_supplier ON sq_supplier_responses(tenant_id, supplier_id);
CREATE INDEX idx_supplier_responses_tenant_program ON sq_supplier_responses(tenant_id, program_id);
CREATE INDEX idx_supplier_responses_tenant_status ON sq_supplier_responses(tenant_id, approval_status);
CREATE INDEX idx_supplier_responses_tenant_evaluator ON sq_supplier_responses(tenant_id, evaluator_id);
CREATE INDEX idx_supplier_responses_tenant_score ON sq_supplier_responses(tenant_id, score_percentage);
CREATE INDEX idx_supplier_responses_tenant_dates ON sq_supplier_responses(tenant_id, submitted_at, reviewed_at);
CREATE INDEX idx_supplier_responses_tenant_active ON sq_supplier_responses(tenant_id, is_active);
```

### 2.4 Certifications
```sql
CREATE TABLE sq_certifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    certification_type TEXT NOT NULL
        CHECK(certification_type IN ('ISO','SAFETY','QUALITY','REGULATORY','ENVIRONMENTAL','FINANCIAL','CUSTOM')),
    certification_name TEXT NOT NULL,
    certifying_body TEXT NOT NULL,
    certificate_number TEXT NOT NULL,
    issue_date TEXT NOT NULL,
    expiry_date TEXT NOT NULL,
    scope_description TEXT,
    attachment_ids TEXT,                             -- JSON: array of document attachment references
    certification_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(certification_status IN ('ACTIVE','EXPIRED','SUSPENDED','REVOKED','PENDING_RENEWAL')),
    renewal_reminder_days INTEGER NOT NULL DEFAULT 90
        CHECK(renewal_reminder_days > 0),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, certification_type, certificate_number)
);

CREATE INDEX idx_certifications_tenant_supplier ON sq_certifications(tenant_id, supplier_id);
CREATE INDEX idx_certifications_tenant_type ON sq_certifications(tenant_id, certification_type);
CREATE INDEX idx_certifications_tenant_status ON sq_certifications(tenant_id, certification_status);
CREATE INDEX idx_certifications_tenant_expiry ON sq_certifications(tenant_id, expiry_date);
CREATE INDEX idx_certifications_tenant_active ON sq_certifications(tenant_id, is_active);
```

### 2.5 Disqualification Records
```sql
CREATE TABLE sq_disqualification_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    disqualification_id TEXT NOT NULL,
    disqualification_type TEXT NOT NULL
        CHECK(disqualification_type IN ('QUALITY_FAILURE','COMPLIANCE_VIOLATION','CERTIFICATION_LAPSE','PERFORMANCE','ETHICAL_VIOLATION','FINANCIAL_INSTABILITY')),
    reason TEXT NOT NULL,
    detailed_findings TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'MAJOR'
        CHECK(severity IN ('MINOR','MAJOR','CRITICAL')),
    corrective_actions TEXT,                         -- JSON: array of required corrective actions with deadlines
    remediation_deadline TEXT,
    blocked_from_date TEXT NOT NULL,
    blocked_to_date TEXT,
    issued_by TEXT NOT NULL,
    reinstatement_eligible INTEGER NOT NULL DEFAULT 1,
    reinstatement_conditions TEXT,                   -- JSON: conditions that must be met for reinstatement
    disqualification_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(disqualification_status IN ('ACTIVE','RESOLVED','REINSTATED','APPEALED')),
    resolved_at TEXT,
    resolved_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, disqualification_id)
);

CREATE INDEX idx_disqual_records_tenant_supplier ON sq_disqualification_records(tenant_id, supplier_id);
CREATE INDEX idx_disqual_records_tenant_type ON sq_disqualification_records(tenant_id, disqualification_type);
CREATE INDEX idx_disqual_records_tenant_severity ON sq_disqualification_records(tenant_id, severity);
CREATE INDEX idx_disqual_records_tenant_status ON sq_disqualification_records(tenant_id, disqualification_status);
CREATE INDEX idx_disqual_records_tenant_dates ON sq_disqualification_records(tenant_id, blocked_from_date, blocked_to_date);
CREATE INDEX idx_disqual_records_tenant_active ON sq_disqualification_records(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Qualification Programs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sq/programs` | List qualification programs |
| POST | `/api/v1/sq/programs` | Create qualification program |
| GET | `/api/v1/sq/programs/{id}` | Get program details |
| PUT | `/api/v1/sq/programs/{id}` | Update program |
| PATCH | `/api/v1/sq/programs/{id}/status` | Update program status |

### 3.2 Questionnaires
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sq/questionnaires` | List questionnaires |
| POST | `/api/v1/sq/questionnaires` | Create questionnaire |
| GET | `/api/v1/sq/questionnaires/{id}` | Get questionnaire details |
| PUT | `/api/v1/sq/questionnaires/{id}` | Update questionnaire |
| POST | `/api/v1/sq/questionnaires/{id}/publish` | Publish questionnaire |
| POST | `/api/v1/sq/questionnaires/{id}/score` | Calculate questionnaire score |

### 3.3 Supplier Responses
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sq/responses` | List supplier responses |
| POST | `/api/v1/sq/responses` | Submit supplier response |
| GET | `/api/v1/sq/responses/{id}` | Get response details |
| POST | `/api/v1/sq/responses/{id}/review` | Submit evaluation review |
| POST | `/api/v1/sq/responses/{id}/approve` | Approve response |
| POST | `/api/v1/sq/responses/{id}/reject` | Reject response |

### 3.4 Certifications
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sq/certifications` | List certifications |
| POST | `/api/v1/sq/certifications` | Record certification |
| GET | `/api/v1/sq/certifications/{id}` | Get certification details |
| PUT | `/api/v1/sq/certifications/{id}` | Update certification |
| GET | `/api/v1/sq/certifications/expiring` | Get certifications nearing expiry |

### 3.5 Disqualification Records
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sq/disqualifications` | List disqualification records |
| POST | `/api/v1/sq/disqualifications` | Issue disqualification |
| GET | `/api/v1/sq/disqualifications/{id}` | Get disqualification details |
| PUT | `/api/v1/sq/disqualifications/{id}` | Update disqualification |
| POST | `/api/v1/sq/disqualifications/{id}/resolve` | Resolve disqualification |
| POST | `/api/v1/sq/disqualifications/{id}/reinstate` | Reinstate supplier |

---

## 4. Business Rules

### 4.1 Qualification Programs
1. Each program MUST have a unique program_code within a tenant.
2. Programs of type REGULATORY MUST include at least one required certification in required_certifications.
3. min_score_threshold MUST be between 0 and 100 inclusive.
4. A program in DRAFT status MUST NOT be assigned to suppliers; it MUST be set to ACTIVE first.
5. validity_period_days MUST be a positive integer representing the qualification validity in days.
6. When auto_approve is 0, all supplier responses MUST go through manual evaluation before approval.

### 4.2 Questionnaires
7. Each questionnaire MUST have at least one section defined in the sections JSON array.
8. Sections MUST contain at least one question with a defined scoring weight.
9. total_points MUST equal the sum of all question point values across all sections.
10. A questionnaire in PUBLISHED status MUST NOT have its sections or scoring model modified; a new version MUST be created.
11. passing_score MUST be between 0 and 100 inclusive.

### 4.3 Supplier Responses
12. A supplier response MUST be linked to exactly one program and one questionnaire.
13. score_percentage MUST be calculated as (total_score / max_possible_score) * 100 when max_possible_score > 0.
14. Responses with score_percentage below the program's min_score_threshold MUST be flagged for rejection.
15. approval_status transitions MUST follow: PENDING -> UNDER_REVIEW -> APPROVED/CONDITIONALLY_APPROVED/REJECTED.
16. CONDITIONALLY_APPROVED responses MUST document specific conditions the supplier must fulfill.

### 4.4 Certifications
17. issue_date MUST be on or before the current date; expiry_date MUST be after issue_date.
18. Certification status MUST automatically transition to EXPIRED when the current date exceeds expiry_date.
19. A certification in EXPIRED, SUSPENDED, or REVOKED status MUST NOT count toward qualification program requirements.
20. renewal_reminder_days MUST trigger a notification the specified number of days before expiry_date.
21. Certificate numbers MUST be unique per supplier and certification type within a tenant.

### 4.5 Disqualification Records
22. A disqualification of severity CRITICAL MUST block the supplier from all procurement activities immediately.
23. Corrective actions MUST include a deadline date and a description of required remediation steps.
24. A disqualification MAY be resolved only if all corrective actions are marked complete or the remediation_deadline has passed.
25. Reinstatement is permitted only if reinstatement_eligible is 1 and all reinstatement_conditions are satisfied.
26. blocked_to_date MUST be null or after blocked_from_date.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.sq.v1;

service SupplierQualificationService {
    // Qualification programs
    rpc CreateProgram(CreateProgramRequest) returns (CreateProgramResponse);
    rpc GetProgram(GetProgramRequest) returns (GetProgramResponse);
    rpc ListPrograms(ListProgramsRequest) returns (ListProgramsResponse);
    rpc UpdateProgram(UpdateProgramRequest) returns (UpdateProgramResponse);

    // Questionnaires
    rpc CreateQuestionnaire(CreateQuestionnaireRequest) returns (CreateQuestionnaireResponse);
    rpc GetQuestionnaire(GetQuestionnaireRequest) returns (GetQuestionnaireResponse);
    rpc ListQuestionnaires(ListQuestionnairesRequest) returns (ListQuestionnairesResponse);
    rpc PublishQuestionnaire(PublishQuestionnaireRequest) returns (PublishQuestionnaireResponse);

    // Supplier responses
    rpc SubmitResponse(SubmitResponseRequest) returns (SubmitResponseResponse);
    rpc GetResponse(GetResponseRequest) returns (GetResponseResponse);
    rpc ListResponses(ListResponsesRequest) returns (ListResponsesResponse);
    rpc ReviewResponse(ReviewResponseRequest) returns (ReviewResponseResponse);
    rpc ApproveResponse(ApproveRejectRequest) returns (ApproveRejectResponse);

    // Certifications
    rpc RecordCertification(RecordCertificationRequest) returns (RecordCertificationResponse);
    rpc GetCertification(GetCertificationRequest) returns (GetCertificationResponse);
    rpc ListCertifications(ListCertificationsRequest) returns (ListCertificationsResponse);
    rpc GetExpiringCertifications(GetExpiringCertificationsRequest) returns (GetExpiringCertificationsResponse);

    // Disqualification records
    rpc IssueDisqualification(IssueDisqualificationRequest) returns (IssueDisqualificationResponse);
    rpc GetDisqualification(GetDisqualificationRequest) returns (GetDisqualificationResponse);
    rpc ListDisqualifications(ListDisqualificationsRequest) returns (ListDisqualificationsResponse);
    rpc ResolveDisqualification(ResolveDisqualificationRequest) returns (ResolveDisqualificationResponse);
    rpc ReinstateSupplier(ReinstateSupplierRequest) returns (ReinstateSupplierResponse);
}

message CreateProgramRequest {
    string tenant_id = 1;
    string program_code = 2;
    string program_name = 3;
    string description = 4;
    string program_type = 5;
    string questionnaire_ids = 6;
    string required_certifications = 7;
    string evaluation_criteria = 8;
    double min_score_threshold = 9;
    int32 validity_period_days = 10;
    bool auto_approve = 11;
    string assigned_evaluator_ids = 12;
}

message CreateProgramResponse {
    string program_id = 1;
    string program_code = 2;
    string program_status = 3;
}

message GetProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
}

message GetProgramResponse {
    string program_id = 1;
    string program_code = 2;
    string program_name = 3;
    string program_type = 4;
    string questionnaire_ids = 5;
    string required_certifications = 6;
    string evaluation_criteria = 7;
    double min_score_threshold = 8;
    int32 validity_period_days = 9;
    bool auto_approve = 10;
    string assigned_evaluator_ids = 11;
    string program_status = 12;
}

message ListProgramsRequest {
    string tenant_id = 1;
    string program_type = 2;
    string program_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListProgramsResponse {
    repeated GetProgramResponse programs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
    string program_name = 3;
    string description = 4;
    string questionnaire_ids = 5;
    string required_certifications = 6;
    string evaluation_criteria = 7;
    double min_score_threshold = 8;
    string program_status = 9;
}

message UpdateProgramResponse {
    string program_id = 1;
    string program_status = 2;
    int32 version = 3;
}

message CreateQuestionnaireRequest {
    string tenant_id = 1;
    string questionnaire_code = 2;
    string title = 3;
    string description = 4;
    string category = 5;
    string sections = 6;
    string scoring_model = 7;
    double total_points = 8;
    double passing_score = 9;
    int32 estimated_duration_minutes = 10;
}

message CreateQuestionnaireResponse {
    string questionnaire_id = 1;
    string questionnaire_code = 2;
    string questionnaire_status = 3;
}

message GetQuestionnaireRequest {
    string tenant_id = 1;
    string questionnaire_id = 2;
}

message GetQuestionnaireResponse {
    string questionnaire_id = 1;
    string questionnaire_code = 2;
    string title = 3;
    string category = 4;
    string sections = 5;
    string scoring_model = 6;
    double total_points = 7;
    double passing_score = 8;
    int32 estimated_duration_minutes = 9;
    string questionnaire_status = 10;
}

message ListQuestionnairesRequest {
    string tenant_id = 1;
    string category = 2;
    string questionnaire_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListQuestionnairesResponse {
    repeated GetQuestionnaireResponse questionnaires = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishQuestionnaireRequest {
    string tenant_id = 1;
    string questionnaire_id = 2;
}

message PublishQuestionnaireResponse {
    string questionnaire_id = 1;
    string questionnaire_status = 2;
}

message SubmitResponseRequest {
    string tenant_id = 1;
    string program_id = 2;
    string supplier_id = 3;
    string questionnaire_id = 4;
    string responses = 5;
}

message SubmitResponseResponse {
    string response_id = 1;
    double total_score = 2;
    double score_percentage = 3;
    string approval_status = 4;
}

message GetResponseRequest {
    string tenant_id = 1;
    string response_id = 2;
}

message GetResponseResponse {
    string id = 1;
    string response_id = 2;
    string program_id = 3;
    string supplier_id = 4;
    string questionnaire_id = 5;
    string responses = 6;
    double total_score = 7;
    double max_possible_score = 8;
    double score_percentage = 9;
    string evaluator_id = 10;
    string evaluator_comments = 11;
    string approval_status = 12;
    string submitted_at = 13;
    string reviewed_at = 14;
    string qualification_valid_until = 15;
}

message ListResponsesRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string program_id = 3;
    string approval_status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListResponsesResponse {
    repeated GetResponseResponse responses = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReviewResponseRequest {
    string tenant_id = 1;
    string response_id = 2;
    string evaluator_id = 3;
    string evaluator_comments = 4;
}

message ReviewResponseResponse {
    string response_id = 1;
    string approval_status = 2;
}

message ApproveRejectRequest {
    string tenant_id = 1;
    string response_id = 2;
    string approval_status = 3;
    string evaluator_comments = 4;
}

message ApproveRejectResponse {
    string response_id = 1;
    string approval_status = 2;
    string qualification_valid_until = 3;
}

message RecordCertificationRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string certification_type = 3;
    string certification_name = 4;
    string certifying_body = 5;
    string certificate_number = 6;
    string issue_date = 7;
    string expiry_date = 8;
    string scope_description = 9;
    string attachment_ids = 10;
    int32 renewal_reminder_days = 11;
}

message RecordCertificationResponse {
    string certification_id = 1;
    string certification_status = 2;
}

message GetCertificationRequest {
    string tenant_id = 1;
    string certification_id = 2;
}

message GetCertificationResponse {
    string certification_id = 1;
    string supplier_id = 2;
    string certification_type = 3;
    string certification_name = 4;
    string certifying_body = 5;
    string certificate_number = 6;
    string issue_date = 7;
    string expiry_date = 8;
    string scope_description = 9;
    string certification_status = 10;
    int32 renewal_reminder_days = 11;
}

message ListCertificationsRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string certification_type = 3;
    string certification_status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListCertificationsResponse {
    repeated GetCertificationResponse certifications = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetExpiringCertificationsRequest {
    string tenant_id = 1;
    int32 days_until_expiry = 2;
}

message GetExpiringCertificationsResponse {
    repeated GetCertificationResponse certifications = 1;
    int32 total_count = 2;
}

message IssueDisqualificationRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string disqualification_type = 3;
    string reason = 4;
    string detailed_findings = 5;
    string severity = 6;
    string corrective_actions = 7;
    string remediation_deadline = 8;
    string blocked_from_date = 9;
    string blocked_to_date = 10;
    bool reinstatement_eligible = 11;
    string reinstatement_conditions = 12;
}

message IssueDisqualificationResponse {
    string disqualification_id = 1;
    string disqualification_status = 2;
}

message GetDisqualificationRequest {
    string tenant_id = 1;
    string disqualification_id = 2;
}

message GetDisqualificationResponse {
    string id = 1;
    string disqualification_id = 2;
    string supplier_id = 3;
    string disqualification_type = 4;
    string reason = 5;
    string detailed_findings = 6;
    string severity = 7;
    string corrective_actions = 8;
    string remediation_deadline = 9;
    string blocked_from_date = 10;
    string blocked_to_date = 11;
    string issued_by = 12;
    bool reinstatement_eligible = 13;
    string reinstatement_conditions = 14;
    string disqualification_status = 15;
}

message ListDisqualificationsRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string disqualification_type = 3;
    string severity = 4;
    string disqualification_status = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListDisqualificationsResponse {
    repeated GetDisqualificationResponse disqualifications = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ResolveDisqualificationRequest {
    string tenant_id = 1;
    string disqualification_id = 2;
    string resolution_notes = 3;
}

message ResolveDisqualificationResponse {
    string disqualification_id = 1;
    string disqualification_status = 2;
    string resolved_at = 3;
}

message ReinstateSupplierRequest {
    string tenant_id = 1;
    string disqualification_id = 2;
    string reinstatement_evidence = 3;
}

message ReinstateSupplierResponse {
    string disqualification_id = 1;
    string disqualification_status = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `supplier-service` | Supplier master data, business classifications, contact information |
| `purchasing-service` | Approved supplier list status, procurement category requirements |
| `auth-service` | User identity for evaluator assignment, approval authority validation |
| `risk-service` | Supplier risk scores, financial stability indicators |
| `document-service` | Certificate attachments, supporting documentation storage |

### Published To
| Service | Data |
|---------|------|
| `supplier-service` | Qualification status updates, certification changes, disqualification flags |
| `purchasing-service` | Qualified supplier list updates, blocked supplier notifications |
| `notification-service` | Certification expiry reminders, disqualification alerts, review assignments |
| `reporting-service` | Qualification metrics, supplier compliance dashboards, scoring analytics |
| `audit-service` | Qualification approval trail, disqualification and reinstatement history |
| `workflow-service` | Evaluation routing, approval chains, corrective action tracking |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `sq.qualification.approved` | `sq.events` | `{ response_id, supplier_id, program_id, score_percentage, qualification_valid_until }` | Supplier qualification approved |
| `sq.qualification.rejected` | `sq.events` | `{ response_id, supplier_id, program_id, score_percentage, rejection_reason }` | Supplier qualification rejected |
| `sq.certification.expiring` | `sq.events` | `{ certification_id, supplier_id, certification_type, certification_name, expiry_date, days_remaining }` | Supplier certification nearing expiry |
| `sq.disqualification.issued` | `sq.events` | `{ disqualification_id, supplier_id, disqualification_type, severity, blocked_from_date, blocked_to_date }` | Supplier disqualification issued |

---

## 8. Migrations

1. V001: `sq_qualification_programs`
2. V002: `sq_questionnaires`
3. V003: `sq_supplier_responses`
4. V004: `sq_certifications`
5. V005: `sq_disqualification_records`
6. V006: Triggers for `updated_at`
