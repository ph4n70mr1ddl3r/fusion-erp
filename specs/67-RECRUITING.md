# 67 - Recruiting Service Specification

## 1. Domain Overview

The Recruiting service manages the complete talent acquisition lifecycle from job requisition creation through candidate sourcing, application management, interview scheduling, offer management, and hiring decisions. It supports internal and external recruiting, career site management, talent pooling, and recruiting analytics.

**Bounded Context:** Talent Acquisition & Recruiting
**Service Name:** `recruiting-service`
**Database:** `data/recruiting.db`
**HTTP Port:** 8099 | **gRPC Port:** 9099

---

## 2. Database Schema

### 2.1 Job Requisitions
```sql
CREATE TABLE job_requisitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requisition_number TEXT NOT NULL,
    job_title TEXT NOT NULL,
    job_id TEXT,
    department_id TEXT,
    location_id TEXT,
    business_unit_id TEXT,
    legal_employer_id TEXT,
    grade_id TEXT,
    position_id TEXT,
    employment_type TEXT NOT NULL CHECK(employment_type IN ('FULL_TIME','PART_TIME','CONTRACT','TEMPORARY','INTERNSHIP')),
    headcount INTEGER NOT NULL DEFAULT 1,
    positions_filled INTEGER NOT NULL DEFAULT 0,
    salary_min INTEGER,   -- cents
    salary_max INTEGER,   -- cents
    currency_code TEXT DEFAULT 'USD',
    description TEXT NOT NULL,
    requirements TEXT NOT NULL,
    responsibilities TEXT,
    qualifications TEXT,
    benefits_summary TEXT,
    priority TEXT DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    reason TEXT NOT NULL CHECK(reason IN ('NEW_POSITION','REPLACEMENT','ADDITIONAL_HEADCOUNT','TEMPORARY')),
    target_start_date TEXT,
    posting_start_date TEXT,
    posting_end_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','PENDING_APPROVAL','APPROVED','OPEN','ON_HOLD','FILLED','CANCELLED','CLOSED')),
    hiring_manager_id TEXT NOT NULL,
    recruiter_id TEXT,
    assigned_recruiter_id TEXT,
    internal_only INTEGER DEFAULT 0,
    source_type TEXT DEFAULT 'INTERNAL' CHECK(source_type IN ('INTERNAL','EXTERNAL','BOTH')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, requisition_number)
);

CREATE INDEX idx_requisitions_tenant_status ON job_requisitions(tenant_id, status);
CREATE INDEX idx_requisitions_tenant_dept ON job_requisitions(tenant_id, department_id);
CREATE INDEX idx_requisitions_tenant_recruiter ON job_requisitions(tenant_id, recruiter_id);
CREATE INDEX idx_requisitions_tenant_dates ON job_requisitions(tenant_id, posting_start_date, posting_end_date);
```

### 2.2 Requisition Approvals
```sql
CREATE TABLE requisition_approvals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requisition_id TEXT NOT NULL,
    approver_id TEXT NOT NULL,
    approval_level INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','APPROVED','REJECTED','SKIPPED')),
    comments TEXT,
    approved_at TEXT,
    delegated_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (requisition_id) REFERENCES job_requisitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_req_approvals_tenant_req ON requisition_approvals(tenant_id, requisition_id);
CREATE INDEX idx_req_approvals_tenant_approver ON requisition_approvals(tenant_id, approver_id, status);
```

### 2.3 Candidate Profiles
```sql
CREATE TABLE candidate_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    candidate_number TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT NOT NULL,
    phone TEXT,
    mobile TEXT,
    address_line1 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT,
    linkedin_url TEXT,
    portfolio_url TEXT,
    source TEXT DEFAULT 'DIRECT' CHECK(source IN ('DIRECT','REFERRAL','JOB_BOARD','SOCIAL_MEDIA','AGENCY','CAREER_SITE','INTERNAL','CAMPUS','OTHER')),
    source_detail TEXT,
    referred_by_employee_id TEXT,
    current_employer TEXT,
    current_job_title TEXT,
    current_salary INTEGER,  -- cents
    expected_salary INTEGER,
    currency_code TEXT DEFAULT 'USD',
    availability_date TEXT,
    notice_period_days INTEGER,
    willing_to_relocate INTEGER DEFAULT 0,
    willing_to_travel INTEGER DEFAULT 0,
    resume_file_reference TEXT,
    cover_letter_reference TEXT,
    status TEXT DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','HIRED','REJECTED','ON_HOLD','BLACKLISTED')),
    consent_given INTEGER DEFAULT 0,
    gdpr_consent_date TEXT,
    tags TEXT,  -- JSON array of tags
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, candidate_number)
);

CREATE INDEX idx_candidates_tenant_name ON candidate_profiles(tenant_id, last_name, first_name);
CREATE INDEX idx_candidates_tenant_source ON candidate_profiles(tenant_id, source);
CREATE INDEX idx_candidates_tenant_status ON candidate_profiles(tenant_id, status);
CREATE INDEX idx_candidates_tenant_email ON candidate_profiles(tenant_id, email);
```

### 2.4 Candidate Applications
```sql
CREATE TABLE candidate_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    candidate_id TEXT NOT NULL,
    requisition_id TEXT NOT NULL,
    application_number TEXT NOT NULL,
    application_date TEXT NOT NULL,
    application_source TEXT,
    status TEXT NOT NULL DEFAULT 'APPLIED' CHECK(status IN ('APPLIED','SCREENING','PHONE_SCREEN','INTERVIEW','ASSESSMENT','REFERENCE_CHECK','OFFER','OFFER_ACCEPTED','OFFER_REJECTED','HIRED','REJECTED','WITHDRAWN','ON_HOLD')),
    current_step TEXT,
    screening_score DECIMAL(5,2),
    screening_notes TEXT,
    fit_score DECIMAL(5,2),
    disposition_reason TEXT,
    last_activity_date TEXT,
    days_in_pipeline INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (candidate_id) REFERENCES candidate_profiles(id) ON DELETE CASCADE,
    FOREIGN KEY (requisition_id) REFERENCES job_requisitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, application_number)
);

CREATE INDEX idx_applications_tenant_candidate ON candidate_applications(tenant_id, candidate_id);
CREATE INDEX idx_applications_tenant_requisition ON candidate_applications(tenant_id, requisition_id, status);
CREATE INDEX idx_applications_tenant_status ON candidate_applications(tenant_id, status);
CREATE INDEX idx_applications_tenant_date ON candidate_applications(tenant_id, application_date);
```

### 2.5 Interview Schedules
```sql
CREATE TABLE interview_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    interview_type TEXT NOT NULL CHECK(interview_type IN ('PHONE_SCREEN','PHONE_INTERVIEW','VIDEO_INTERVIEW','ON_SITE','PANEL','TECHNICAL','BEHAVIORAL','CASE_STUDY','PRESENTATION','FINAL')),
    interview_date TEXT NOT NULL,
    start_time TEXT NOT NULL,
    end_time TEXT NOT NULL,
    location TEXT,
    meeting_link TEXT,
    interviewer_ids TEXT NOT NULL,  -- JSON array of employee IDs
    panel_lead_id TEXT,
    status TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK(status IN ('SCHEDULED','CONFIRMED','IN_PROGRESS','COMPLETED','CANCELLED','RESCHEDULED','NO_SHOW')),
    feedback_deadline TEXT,
    reminder_sent INTEGER DEFAULT 0,
    reschedule_count INTEGER DEFAULT 0,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES candidate_applications(id) ON DELETE CASCADE
);

CREATE INDEX idx_interviews_tenant_application ON interview_schedules(tenant_id, application_id);
CREATE INDEX idx_interviews_tenant_date ON interview_schedules(tenant_id, interview_date);
CREATE INDEX idx_interviews_tenant_status ON interview_schedules(tenant_id, status);
```

### 2.6 Interview Feedback
```sql
CREATE TABLE interview_feedback (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    interview_id TEXT NOT NULL,
    interviewer_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    overall_rating INTEGER NOT NULL CHECK(overall_rating BETWEEN 1 AND 5),
    technical_skills INTEGER CHECK(technical_skills BETWEEN 1 AND 5),
    communication_skills INTEGER CHECK(communication_skills BETWEEN 1 AND 5),
    cultural_fit INTEGER CHECK(cultural_fit BETWEEN 1 AND 5),
    experience_relevance INTEGER CHECK(experience_relevance BETWEEN 1 AND 5),
    recommendation TEXT NOT NULL CHECK(recommendation IN ('STRONG_HIRE','HIRE','NEUTRAL','NO_HIRE','STRONG_NO_HIRE')),
    strengths TEXT,
    concerns TEXT,
    detailed_feedback TEXT NOT NULL,
    submitted_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','SUBMITTED','LATE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (interview_id) REFERENCES interview_schedules(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, interview_id, interviewer_id)
);

CREATE INDEX idx_interview_feedback_tenant_interview ON interview_feedback(tenant_id, interview_id);
CREATE INDEX idx_interview_feedback_tenant_interviewer ON interview_feedback(tenant_id, interviewer_id, status);
```

### 2.7 Offer Letters
```sql
CREATE TABLE offer_letters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    candidate_id TEXT NOT NULL,
    requisition_id TEXT NOT NULL,
    offer_number TEXT NOT NULL,
    job_title TEXT NOT NULL,
    department_id TEXT,
    location_id TEXT,
    grade_id TEXT,
    employment_type TEXT NOT NULL,
    start_date TEXT NOT NULL,
    base_salary INTEGER NOT NULL,   -- cents
    currency_code TEXT DEFAULT 'USD',
    signing_bonus INTEGER DEFAULT 0,
    annual_bonus_target INTEGER DEFAULT 0,
    stock_grant_details TEXT,
    benefits_summary TEXT,
    relocation_package INTEGER DEFAULT 0,
    special_conditions TEXT,
    expiration_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','PENDING_APPROVAL','APPROVED','SENT','ACCEPTED','REJECTED','WITHDRAWN','EXPIRED','COUNTER_OFFERED')),
    sent_date TEXT,
    response_date TEXT,
    approved_by TEXT,
    approved_at TEXT,
    document_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, offer_number)
);

CREATE INDEX idx_offers_tenant_candidate ON offer_letters(tenant_id, candidate_id);
CREATE INDEX idx_offers_tenant_requisition ON offer_letters(tenant_id, requisition_id);
CREATE INDEX idx_offers_tenant_status ON offer_letters(tenant_id, status);
```

### 2.8 Hiring Decisions
```sql
CREATE TABLE hiring_decisions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    candidate_id TEXT NOT NULL,
    requisition_id TEXT NOT NULL,
    offer_id TEXT,
    decision TEXT NOT NULL CHECK(decision IN ('HIRE','REJECT','HOLD','REFER_TO_OTHER')),
    decision_maker_id TEXT NOT NULL,
    decision_date TEXT NOT NULL,
    effective_hire_date TEXT,
    employee_id TEXT,  -- Set after HR creates employee
    reason TEXT,
    onboarding_status TEXT DEFAULT 'PENDING' CHECK(onboarding_status IN ('PENDING','INITIATED','COMPLETED','NOT_APPLICABLE')),
    onboarding_initiated_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES candidate_applications(id) ON DELETE CASCADE
);

CREATE INDEX idx_hiring_decisions_tenant_req ON hiring_decisions(tenant_id, requisition_id);
CREATE INDEX idx_hiring_decisions_tenant_candidate ON hiring_decisions(tenant_id, candidate_id);
```

### 2.9 Recruiting Sources
```sql
CREATE TABLE recruiting_sources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_name TEXT NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('JOB_BOARD','SOCIAL_MEDIA','AGENCY','REFERRAL','CAREER_SITE','CAMPUS','EVENT','INTERNAL','OTHER')),
    source_url TEXT,
    cost_per_posting INTEGER DEFAULT 0,  -- cents
    annual_contract_cost INTEGER DEFAULT 0,
    effectiveness_rating INTEGER CHECK(effectiveness_rating BETWEEN 1 AND 5),
    is_active INTEGER DEFAULT 1,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, source_name)
);

CREATE INDEX idx_recruiting_sources_tenant_type ON recruiting_sources(tenant_id, source_type);
```

### 2.10 Career Sites
```sql
CREATE TABLE career_sites (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    site_name TEXT NOT NULL,
    site_url TEXT NOT NULL,
    site_type TEXT NOT NULL DEFAULT 'EXTERNAL' CHECK(site_type IN ('EXTERNAL','INTERNAL','BOTH')),
    branding_config TEXT,  -- JSON: colors, logo, fonts, layout
    welcome_message TEXT,
    is_default INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','MAINTENANCE')),
    application_form_config TEXT,  -- JSON: custom fields, required fields
    seo_keywords TEXT,
    analytics_tracking_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, site_url)
);

CREATE INDEX idx_career_sites_tenant ON career_sites(tenant_id, status);
```

### 2.11 Talent Pools
```sql
CREATE TABLE talent_pools (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pool_name TEXT NOT NULL,
    pool_type TEXT NOT NULL CHECK(pool_type IN ('SKILL_BASED','DEPARTMENT','EXPERIENCE_LEVEL','CAMPUS','REFERRAL','GENERAL','CUSTOM')),
    description TEXT,
    criteria TEXT,  -- JSON: filter criteria
    candidate_count INTEGER DEFAULT 0,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, pool_name)
);

CREATE INDEX idx_talent_pools_tenant ON talent_pools(tenant_id, pool_type, status);
```

---

## 3. REST API Endpoints

### Requisitions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/requisitions` | Create a job requisition |
| GET | `/api/v1/requisitions` | List requisitions with filters |
| GET | `/api/v1/requisitions/{id}` | Get requisition details |
| PUT | `/api/v1/requisitions/{id}` | Update requisition |
| POST | `/api/v1/requisitions/{id}/submit` | Submit requisition for approval |
| POST | `/api/v1/requisitions/{id}/approve` | Approve requisition |
| POST | `/api/v1/requisitions/{id}/reject` | Reject requisition |
| POST | `/api/v1/requisitions/{id}/close` | Close requisition |
| POST | `/api/v1/requisitions/{id}/post` | Post requisition to career site |

### Candidates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/candidates` | Create candidate profile |
| GET | `/api/v1/candidates` | Search/list candidates |
| GET | `/api/v1/candidates/{id}` | Get candidate profile |
| PUT | `/api/v1/candidates/{id}` | Update candidate profile |
| GET | `/api/v1/candidates/{id}/applications` | List candidate's applications |
| POST | `/api/v1/candidates/search` | Advanced candidate search |

### Applications
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/applications` | Submit a job application |
| GET | `/api/v1/applications` | List applications with filters |
| GET | `/api/v1/applications/{id}` | Get application details |
| PUT | `/api/v1/applications/{id}/status` | Update application status |
| POST | `/api/v1/applications/{id}/withdraw` | Withdraw application |
| GET | `/api/v1/requisitions/{id}/applications` | List requisition applications |

### Interviews
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/interviews` | Schedule an interview |
| GET | `/api/v1/interviews` | List interviews |
| GET | `/api/v1/interviews/{id}` | Get interview details |
| PUT | `/api/v1/interviews/{id}` | Update interview schedule |
| POST | `/api/v1/interviews/{id}/cancel` | Cancel interview |
| POST | `/api/v1/interviews/{id}/reschedule` | Reschedule interview |

### Feedback
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/interviews/{id}/feedback` | Submit interview feedback |
| GET | `/api/v1/interviews/{id}/feedback` | Get all feedback for interview |
| GET | `/api/v1/applications/{id}/feedback-summary` | Get aggregated feedback |

### Offers
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/offers` | Create an offer letter |
| GET | `/api/v1/offers` | List offers |
| GET | `/api/v1/offers/{id}` | Get offer details |
| PUT | `/api/v1/offers/{id}` | Update offer |
| POST | `/api/v1/offers/{id}/submit` | Submit offer for approval |
| POST | `/api/v1/offers/{id}/send` | Send offer to candidate |
| POST | `/api/v1/offers/{id}/accept` | Accept offer |
| POST | `/api/v1/offers/{id}/reject` | Reject offer |

### Hiring
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/hiring-decisions` | Make a hiring decision |
| GET | `/api/v1/hiring-decisions` | List hiring decisions |
| POST | `/api/v1/hiring-decisions/{id}/initiate-onboarding` | Trigger onboarding |

### Talent Pools
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-pools` | Create talent pool |
| GET | `/api/v1/talent-pools` | List talent pools |
| POST | `/api/v1/talent-pools/{id}/candidates` | Add candidate to pool |
| GET | `/api/v1/talent-pools/{id}/candidates` | List pool candidates |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/analytics/pipeline` | Get pipeline funnel metrics |
| GET | `/api/v1/analytics/time-to-fill` | Get time-to-fill statistics |
| GET | `/api/v1/analytics/source-effectiveness` | Get source ROI analysis |
| GET | `/api/v1/analytics/recruiter-performance` | Get recruiter activity metrics |

---

## 4. Business Rules

1. A requisition MUST be approved before applications can be accepted against it.
2. A candidate MUST have a unique email address within a tenant.
3. An application MUST be linked to exactly one candidate and one requisition.
4. Offer salary amount MUST fall within the requisition salary range unless an override is approved.
5. Hiring decisions MUST reference an accepted offer before employee creation.
6. Interview feedback MUST be submitted within the configured feedback deadline.
7. A requisition's `positions_filled` MUST NOT exceed its `headcount` value.
8. Candidate data MUST be retained per GDPR consent and MUST support right-to-erasure requests.
9. The system MUST calculate `days_in_pipeline` for each application automatically.
10. Offer letters MUST have an expiration date and MUST auto-expire if not responded to.
11. Interviewers MUST NOT submit feedback for interviews they did not attend.
12. The system SHOULD auto-screen applications based on minimum qualifications.
13. Talent pool criteria SHOULD auto-match existing candidate profiles.
14. Requisition approval workflows MUST support at least 3 levels.
15. Career site applications MUST be linked to candidate profiles automatically on creation.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package recruiting.v1;

service RecruitingService {
    // Requisitions
    rpc CreateRequisition(CreateRequisitionRequest) returns (CreateRequisitionResponse);
    rpc GetRequisition(GetRequisitionRequest) returns (GetRequisitionResponse);
    rpc ListRequisitions(ListRequisitionsRequest) returns (ListRequisitionsResponse);
    rpc SubmitRequisition(SubmitRequisitionRequest) returns (SubmitRequisitionResponse);
    rpc ApproveRequisition(ApproveRequisitionRequest) returns (ApproveRequisitionResponse);

    // Candidates
    rpc CreateCandidate(CreateCandidateRequest) returns (CreateCandidateResponse);
    rpc GetCandidate(GetCandidateRequest) returns (GetCandidateResponse);
    rpc SearchCandidates(SearchCandidatesRequest) returns (SearchCandidatesResponse);

    // Applications
    rpc SubmitApplication(SubmitApplicationRequest) returns (SubmitApplicationResponse);
    rpc GetApplication(GetApplicationRequest) returns (GetApplicationResponse);
    rpc ListApplications(ListApplicationsRequest) returns (ListApplicationsResponse);
    rpc UpdateApplicationStatus(UpdateApplicationStatusRequest) returns (UpdateApplicationStatusResponse);

    // Interviews
    rpc ScheduleInterview(ScheduleInterviewRequest) returns (ScheduleInterviewResponse);
    rpc SubmitFeedback(SubmitFeedbackRequest) returns (SubmitFeedbackResponse);

    // Offers
    rpc CreateOffer(CreateOfferRequest) returns (CreateOfferResponse);
    rpc SendOffer(SendOfferRequest) returns (SendOfferResponse);
    rpc AcceptOffer(AcceptOfferRequest) returns (AcceptOfferResponse);

    // Hiring
    rpc MakeHiringDecision(MakeHiringDecisionRequest) returns (MakeHiringDecisionResponse);

    // Analytics
    rpc GetPipelineMetrics(GetPipelineMetricsRequest) returns (GetPipelineMetricsResponse);
}

message Requisition {
    string id = 1;
    string tenant_id = 2;
    string requisition_number = 3;
    string job_title = 4;
    string department_id = 5;
    string employment_type = 6;
    int32 headcount = 7;
    string status = 8;
    string hiring_manager_id = 9;
}

message CreateRequisitionRequest {
    string tenant_id = 1;
    string job_title = 2;
    string department_id = 3;
    string employment_type = 4;
    int32 headcount = 5;
    string description = 6;
    string requirements = 7;
    string hiring_manager_id = 8;
    string created_by = 9;
}

message CreateRequisitionResponse {
    Requisition requisition = 1;
}

message GetRequisitionRequest {
    string tenant_id = 1;
    string requisition_id = 2;
}

message GetRequisitionResponse {
    Requisition requisition = 1;
}

message ListRequisitionsRequest {
    string tenant_id = 1;
    string status = 2;
    string department_id = 3;
    string recruiter_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListRequisitionsResponse {
    repeated Requisition requisitions = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitRequisitionRequest {
    string tenant_id = 1;
    string requisition_id = 2;
    string submitted_by = 3;
}

message SubmitRequisitionResponse {
    Requisition requisition = 1;
}

message ApproveRequisitionRequest {
    string tenant_id = 1;
    string requisition_id = 2;
    string approved_by = 3;
    string comments = 4;
}

message ApproveRequisitionResponse {
    Requisition requisition = 1;
}

message Candidate {
    string id = 1;
    string tenant_id = 2;
    string candidate_number = 3;
    string first_name = 4;
    string last_name = 5;
    string email = 6;
    string phone = 7;
    string source = 8;
    string status = 9;
}

message CreateCandidateRequest {
    string tenant_id = 1;
    string first_name = 2;
    string last_name = 3;
    string email = 4;
    string phone = 5;
    string source = 6;
    string created_by = 7;
}

message CreateCandidateResponse {
    Candidate candidate = 1;
}

message GetCandidateRequest {
    string tenant_id = 1;
    string candidate_id = 2;
}

message GetCandidateResponse {
    Candidate candidate = 1;
}

message SearchCandidatesRequest {
    string tenant_id = 1;
    string query = 2;
    string source = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message SearchCandidatesResponse {
    repeated Candidate candidates = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message Application {
    string id = 1;
    string candidate_id = 2;
    string requisition_id = 3;
    string application_number = 4;
    string status = 5;
    string application_date = 6;
}

message SubmitApplicationRequest {
    string tenant_id = 1;
    string candidate_id = 2;
    string requisition_id = 3;
    string application_source = 4;
    string created_by = 5;
}

message SubmitApplicationResponse {
    Application application = 1;
}

message GetApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
}

message GetApplicationResponse {
    Application application = 1;
}

message ListApplicationsRequest {
    string tenant_id = 1;
    string requisition_id = 2;
    string candidate_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListApplicationsResponse {
    repeated Application applications = 1;
    string next_page_token = 2;
}

message UpdateApplicationStatusRequest {
    string tenant_id = 1;
    string application_id = 2;
    string new_status = 3;
    string disposition_reason = 4;
    string updated_by = 5;
}

message UpdateApplicationStatusResponse {
    Application application = 1;
}

message ScheduleInterviewRequest {
    string tenant_id = 1;
    string application_id = 2;
    string interview_type = 3;
    string interview_date = 4;
    string start_time = 5;
    string end_time = 6;
    repeated string interviewer_ids = 7;
    string created_by = 8;
}

message ScheduleInterviewResponse {
    string interview_id = 1;
    string status = 2;
}

message SubmitFeedbackRequest {
    string tenant_id = 1;
    string interview_id = 2;
    string interviewer_id = 3;
    int32 overall_rating = 4;
    string recommendation = 5;
    string detailed_feedback = 6;
    string submitted_by = 7;
}

message SubmitFeedbackResponse {
    string feedback_id = 1;
    string status = 2;
}

message CreateOfferRequest {
    string tenant_id = 1;
    string application_id = 2;
    string job_title = 3;
    string start_date = 4;
    int64 base_salary = 5;
    string created_by = 6;
}

message CreateOfferResponse {
    string offer_id = 1;
    string offer_number = 2;
    string status = 3;
}

message SendOfferRequest {
    string tenant_id = 1;
    string offer_id = 2;
    string sent_by = 3;
}

message SendOfferResponse {
    string offer_id = 1;
    string status = 2;
    string sent_date = 3;
}

message AcceptOfferRequest {
    string tenant_id = 1;
    string offer_id = 2;
}

message AcceptOfferResponse {
    string offer_id = 1;
    string status = 2;
    string response_date = 3;
}

message MakeHiringDecisionRequest {
    string tenant_id = 1;
    string application_id = 2;
    string decision = 3;
    string offer_id = 4;
    string effective_hire_date = 5;
    string decision_maker_id = 6;
}

message MakeHiringDecisionResponse {
    string decision_id = 1;
    string employee_id = 2;
}

message GetPipelineMetricsRequest {
    string tenant_id = 1;
    string requisition_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetPipelineMetricsResponse {
    int32 total_applications = 1;
    int32 total_interviews = 2;
    int32 total_offers = 3;
    int32 total_hires = 4;
    double avg_days_to_fill = 5;
    double offer_acceptance_rate = 6;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Job definitions, org structure, hiring manager list | Populate requisition details |
| `workflow-service` | Approval workflows | Process requisition and offer approvals |
| `document-service` | Resume and offer letter storage | Manage candidate documents |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | New hire details | Create employee record on hire |
| `document-service` | Offer letter documents | Store and send offer letters |
| `reporting-service` | Recruiting analytics | Talent acquisition dashboards |
| `career-service` | Internal job postings | Internal career mobility |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `RequisitionCreated` | `recruiting.requisition.created` | `{ tenant_id, requisition_id, requisition_number, job_title, department_id, headcount }` | Published when a new requisition is created |
| `RequisitionApproved` | `recruiting.requisition.approved` | `{ tenant_id, requisition_id, approved_by }` | Published when requisition is approved |
| `ApplicationReceived` | `recruiting.application.received` | `{ tenant_id, application_id, candidate_id, requisition_id, application_date, source }` | Published when a new application is submitted |
| `InterviewScheduled` | `recruiting.interview.scheduled` | `{ tenant_id, interview_id, application_id, interview_date, interviewer_ids }` | Published when interview is scheduled |
| `OfferExtended` | `recruiting.offer.extended` | `{ tenant_id, offer_id, candidate_id, requisition_id, base_salary, expiration_date }` | Published when an offer is sent to a candidate |
| `OfferAccepted` | `recruiting.offer.accepted` | `{ tenant_id, offer_id, candidate_id, start_date }` | Published when candidate accepts offer |
| `CandidateHired` | `recruiting.candidate.hired` | `{ tenant_id, decision_id, candidate_id, employee_id, requisition_id, hire_date }` | Published when hiring decision is finalized |
| `ApplicationRejected` | `recruiting.application.rejected` | `{ tenant_id, application_id, candidate_id, disposition_reason }` | Published when an application is rejected |
