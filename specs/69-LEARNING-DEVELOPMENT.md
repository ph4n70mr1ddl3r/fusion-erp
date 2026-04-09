# 69 - Learning & Development Service Specification

## 1. Domain Overview

The Learning & Development service manages corporate training programs including course catalog management, session scheduling, learning path configuration, learner enrollment, completion tracking, certification management, compliance training rules, instructor profiles, and learning materials. It ensures workforce skills development and regulatory compliance training requirements are met.

**Bounded Context:** Learning & Training Administration
**Service Name:** `learning-service`
**Database:** `data/learning.db`
**HTTP Port:** 8101 | **gRPC Port:** 9101

---

## 2. Database Schema

### 2.1 Course Catalog
```sql
CREATE TABLE course_catalog (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    course_code TEXT NOT NULL,
    course_title TEXT NOT NULL,
    course_description TEXT NOT NULL,
    course_type TEXT NOT NULL CHECK(course_type IN ('INSTRUCTOR_LED','SELF_PACED','VIRTUAL_ILT','BLENDED','WEBINAR','ON_THE_JOB','ASSESSMENT')),
    category TEXT NOT NULL,
    sub_category TEXT,
    duration_hours DECIMAL(5,2) NOT NULL,
    difficulty_level TEXT DEFAULT 'BEGINNER' CHECK(difficulty_level IN ('BEGINNER','INTERMEDIATE','ADVANCED','EXPERT')),
    delivery_method TEXT DEFAULT 'ONLINE' CHECK(delivery_method IN ('ONLINE','IN_PERSON','HYBRID')),
    credits DECIMAL(5,2) DEFAULT 0,
    ceu_credits DECIMAL(5,2) DEFAULT 0,
    pdus DECIMAL(5,2) DEFAULT 0,
    max_enrollment INTEGER,
    waitlist_enabled INTEGER DEFAULT 0,
    approval_required INTEGER DEFAULT 0,
    cost_per_learner INTEGER DEFAULT 0,  -- cents
    currency_code TEXT DEFAULT 'USD',
    provider TEXT,
    tags TEXT,  -- JSON array of tags
    thumbnail_url TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','UNDER_DEVELOPMENT','RETIRED')),
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    version_number INTEGER DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, course_code)
);

CREATE INDEX idx_course_catalog_tenant_category ON course_catalog(tenant_id, category);
CREATE INDEX idx_course_catalog_tenant_type ON course_catalog(tenant_id, course_type);
CREATE INDEX idx_course_catalog_tenant_status ON course_catalog(tenant_id, status);
```

### 2.2 Course Sessions
```sql
CREATE TABLE course_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    course_id TEXT NOT NULL,
    session_code TEXT NOT NULL,
    session_name TEXT NOT NULL,
    instructor_id TEXT,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    start_time TEXT,
    end_time TEXT,
    timezone TEXT DEFAULT 'UTC',
    location TEXT,
    virtual_meeting_url TEXT,
    capacity INTEGER NOT NULL,
    enrolled_count INTEGER DEFAULT 0,
    waitlist_count INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK(status IN ('SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED','FULL')),
    enrollment_deadline TEXT,
    cancellation_deadline TEXT,
    cost_per_learner INTEGER DEFAULT 0,
    room_id TEXT,
    materials_sent INTEGER DEFAULT 0,
    evaluation_sent INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (course_id) REFERENCES course_catalog(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, session_code)
);

CREATE INDEX idx_course_sessions_tenant_course ON course_sessions(tenant_id, course_id);
CREATE INDEX idx_course_sessions_tenant_dates ON course_sessions(tenant_id, start_date, end_date);
CREATE INDEX idx_course_sessions_tenant_status ON course_sessions(tenant_id, status);
```

### 2.3 Learning Paths
```sql
CREATE TABLE learning_paths (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    path_name TEXT NOT NULL,
    path_code TEXT NOT NULL,
    description TEXT,
    target_audience TEXT,  -- JSON: job IDs, grade IDs, department IDs
    estimated_duration_hours DECIMAL(7,2),
    course_sequence TEXT NOT NULL,  -- JSON array of { course_id, order, required }
    total_courses INTEGER DEFAULT 0,
    total_credits DECIMAL(5,2) DEFAULT 0,
    completion_criteria TEXT DEFAULT 'ALL_REQUIRED' CHECK(completion_criteria IN ('ALL_REQUIRED','MINIMUM_CREDITS','PERCENTAGE')),
    minimum_credits DECIMAL(5,2),
    completion_percentage DECIMAL(5,2) DEFAULT 100,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, path_code)
);

CREATE INDEX idx_learning_paths_tenant_status ON learning_paths(tenant_id, status);
```

### 2.4 Learner Enrollments
```sql
CREATE TABLE learner_enrollments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    course_id TEXT NOT NULL,
    session_id TEXT,
    learning_path_id TEXT,
    enrollment_type TEXT NOT NULL CHECK(enrollment_type IN ('SELF','MANAGER','SYSTEM','COMPLIANCE','PATH')),
    enrollment_date TEXT NOT NULL,
    completion_deadline TEXT,
    status TEXT NOT NULL DEFAULT 'ENROLLED' CHECK(status IN ('ENROLLED','IN_PROGRESS','COMPLETED','FAILED','CANCELLED','WITHDREW','NO_SHOW','EXPIRED')),
    completion_date TEXT,
    score DECIMAL(5,2),
    pass_fail TEXT CHECK(pass_fail IN ('PASS','FAIL','INCOMPLETE')),
    time_spent_minutes INTEGER DEFAULT 0,
    progress_percentage DECIMAL(5,2) DEFAULT 0,
    waitlist_position INTEGER,
    approval_status TEXT DEFAULT 'APPROVED' CHECK(approval_status IN ('PENDING','APPROVED','REJECTED')),
    approved_by TEXT,
    certificate_id TEXT,
    evaluation_completed INTEGER DEFAULT 0,
    evaluation_score DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (course_id) REFERENCES course_catalog(id) ON DELETE CASCADE
);

CREATE INDEX idx_enrollments_tenant_employee ON learner_enrollments(tenant_id, employee_id, status);
CREATE INDEX idx_enrollments_tenant_course ON learner_enrollments(tenant_id, course_id);
CREATE INDEX idx_enrollments_tenant_session ON learner_enrollments(tenant_id, session_id);
CREATE INDEX idx_enrollments_tenant_deadline ON learner_enrollments(tenant_id, completion_deadline);
```

### 2.5 Learner Completions
```sql
CREATE TABLE learner_completions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    enrollment_id TEXT NOT NULL,
    course_id TEXT NOT NULL,
    session_id TEXT,
    completion_date TEXT NOT NULL,
    score DECIMAL(5,2),
    pass_fail TEXT NOT NULL CHECK(pass_fail IN ('PASS','FAIL')),
    credits_earned DECIMAL(5,2) DEFAULT 0,
    ceu_credits_earned DECIMAL(5,2) DEFAULT 0,
    pdus_earned DECIMAL(5,2) DEFAULT 0,
    time_spent_minutes INTEGER DEFAULT 0,
    completion_method TEXT DEFAULT 'ONLINE' CHECK(completion_method IN ('ONLINE','IN_PERSON','ASSESSMENT','EQUIVALENCY','TRANSFER')),
    verification_status TEXT DEFAULT 'PENDING' CHECK(verification_status IN ('PENDING','VERIFIED','REJECTED')),
    verified_by TEXT,
    verified_date TEXT,
    certificate_number TEXT,
    certificate_url TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (enrollment_id) REFERENCES learner_enrollments(id) ON DELETE CASCADE
);

CREATE INDEX idx_completions_tenant_employee ON learner_completions(tenant_id, employee_id);
CREATE INDEX idx_completions_tenant_course ON learner_completions(tenant_id, course_id);
CREATE INDEX idx_completions_tenant_date ON learner_completions(tenant_id, completion_date);
```

### 2.6 Certifications
```sql
CREATE TABLE certifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    certification_name TEXT NOT NULL,
    certification_code TEXT NOT NULL,
    description TEXT,
    issuing_body TEXT,
    certification_type TEXT NOT NULL CHECK(certification_type IN ('INTERNAL','EXTERNAL','LICENSE','REGULATORY','PROFESSIONAL')),
    validity_period_months INTEGER,
    renewal_required INTEGER DEFAULT 0,
    renewal_requirements TEXT,  -- JSON: courses, hours, exam
    required_courses TEXT,  -- JSON array of course IDs
    exam_required INTEGER DEFAULT 0,
    exam_passing_score DECIMAL(5,2),
    total_credits_required DECIMAL(5,2),
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','RETIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, certification_code)
);

CREATE INDEX idx_certifications_tenant_type ON certifications(tenant_id, certification_type);
```

### 2.7 Certification Assignments
```sql
CREATE TABLE certification_assignments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    certification_id TEXT NOT NULL,
    assignment_type TEXT NOT NULL CHECK(assignment_type IN ('REQUIRED','RECOMMENDED','VOLUNTARY')),
    assigned_date TEXT NOT NULL,
    assigned_by TEXT,
    due_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','IN_PROGRESS','ACHIEVED','EXPIRED','REVOKED','WAIVED')),
    achieved_date TEXT,
    expiry_date TEXT,
    renewal_due_date TEXT,
    certificate_number TEXT,
    score DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (certification_id) REFERENCES certifications(id) ON DELETE CASCADE
);

CREATE INDEX idx_cert_assign_tenant_employee ON certification_assignments(tenant_id, employee_id, status);
CREATE INDEX idx_cert_assign_tenant_cert ON certification_assignments(tenant_id, certification_id);
CREATE INDEX idx_cert_assign_tenant_expiry ON certification_assignments(tenant_id, expiry_date);
```

### 2.8 Compliance Training Rules
```sql
CREATE TABLE compliance_training_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    course_id TEXT NOT NULL,
    regulation_reference TEXT,
    target_population TEXT NOT NULL,  -- JSON: job, department, location criteria
    frequency TEXT NOT NULL CHECK(frequency IN ('ONE_TIME','ANNUAL','BIENNIAL','QUARTERLY','MONTHLY')),
    due_within_days INTEGER,
    grace_period_days INTEGER DEFAULT 0,
    auto_enroll INTEGER DEFAULT 1,
    escalation_contact TEXT,
    non_compliance_action TEXT DEFAULT 'NOTIFY' CHECK(non_compliance_action IN ('NOTIFY','BLOCK_ACCESS','ESCALATE','SUSPEND')),
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_compliance_rules_tenant_course ON compliance_training_rules(tenant_id, course_id);
CREATE INDEX idx_compliance_rules_tenant_status ON compliance_training_rules(tenant_id, status);
```

### 2.9 Instructor Profiles
```sql
CREATE TABLE instructor_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT,
    external_instructor_name TEXT,
    instructor_type TEXT NOT NULL CHECK(instructor_type IN ('INTERNAL','EXTERNAL')),
    bio TEXT,
    expertise_areas TEXT,  -- JSON array
    certifications TEXT,   -- JSON array
    rating_average DECIMAL(3,2),
    total_sessions INTEGER DEFAULT 0,
    hourly_rate INTEGER,  -- cents
    currency_code TEXT DEFAULT 'USD',
    availability TEXT,  -- JSON: schedule
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','ON_LEAVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_instructor_tenant_status ON instructor_profiles(tenant_id, status);
CREATE INDEX idx_instructor_tenant_type ON instructor_profiles(tenant_id, instructor_type);
```

### 2.10 Learning Materials
```sql
CREATE TABLE learning_materials (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    course_id TEXT NOT NULL,
    material_title TEXT NOT NULL,
    material_type TEXT NOT NULL CHECK(material_type IN ('DOCUMENT','VIDEO','PRESENTATION','SCORM','XAPI','LINK','ASSESSMENT','QUIZ','SIMULATION','EBOOK')),
    file_reference TEXT,
    file_size_bytes INTEGER,
    mime_type TEXT,
    url TEXT,
    duration_minutes INTEGER,
    display_order INTEGER DEFAULT 0,
    is_required INTEGER DEFAULT 1,
    access_level TEXT DEFAULT 'ENROLLED' CHECK(access_level IN ('PUBLIC','ENROLLED','INSTRUCTOR','ADMIN')),
    description TEXT,
    scorm_version TEXT,
    completion_criteria TEXT,  -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (course_id) REFERENCES course_catalog(id) ON DELETE CASCADE
);

CREATE INDEX idx_materials_tenant_course ON learning_materials(tenant_id, course_id);
CREATE INDEX idx_materials_tenant_type ON learning_materials(tenant_id, material_type);
```

---

## 3. REST API Endpoints

### Course Catalog
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/courses` | Create a course |
| GET | `/api/v1/courses` | List courses with filters |
| GET | `/api/v1/courses/{id}` | Get course details |
| PUT | `/api/v1/courses/{id}` | Update course |
| GET | `/api/v1/courses/search` | Search courses |

### Sessions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sessions` | Create a course session |
| GET | `/api/v1/sessions` | List sessions |
| GET | `/api/v1/sessions/{id}` | Get session details |
| PUT | `/api/v1/sessions/{id}` | Update session |
| POST | `/api/v1/sessions/{id}/cancel` | Cancel session |

### Learning Paths
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/learning-paths` | Create learning path |
| GET | `/api/v1/learning-paths` | List learning paths |
| GET | `/api/v1/learning-paths/{id}` | Get path details |
| PUT | `/api/v1/learning-paths/{id}` | Update learning path |
| POST | `/api/v1/learning-paths/{id}/assign` | Assign path to employees |

### Enrollments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/enrollments` | Enroll a learner |
| GET | `/api/v1/enrollments` | List enrollments |
| GET | `/api/v1/enrollments/{id}` | Get enrollment details |
| POST | `/api/v1/enrollments/{id}/cancel` | Cancel enrollment |
| GET | `/api/v1/employees/{employeeId}/learning-history` | Get employee learning history |
| GET | `/api/v1/employees/{employeeId}/current-courses` | Get employee active enrollments |

### Completions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/completions` | Record a course completion |
| GET | `/api/v1/completions` | List completions |
| GET | `/api/v1/completions/{id}` | Get completion details |

### Certifications
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/certifications` | Create certification |
| GET | `/api/v1/certifications` | List certifications |
| GET | `/api/v1/certifications/{id}` | Get certification details |
| PUT | `/api/v1/certifications/{id}` | Update certification |
| POST | `/api/v1/certifications/{id}/assign` | Assign certification to employee |
| GET | `/api/v1/employees/{employeeId}/certifications` | Get employee certifications |

### Compliance
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/compliance-rules` | Create compliance training rule |
| GET | `/api/v1/compliance-rules` | List compliance rules |
| GET | `/api/v1/compliance/status` | Get compliance status overview |
| GET | `/api/v1/employees/{employeeId}/compliance` | Check employee compliance |
| POST | `/api/v1/compliance/check` | Run compliance check across workforce |

### Materials
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/courses/{courseId}/materials` | Add learning material |
| GET | `/api/v1/courses/{courseId}/materials` | List course materials |
| PUT | `/api/v1/materials/{id}` | Update material |
| DELETE | `/api/v1/materials/{id}` | Remove material |

### Instructors
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/instructors` | Create instructor profile |
| GET | `/api/v1/instructors` | List instructors |
| PUT | `/api/v1/instructors/{id}` | Update instructor profile |

---

## 4. Business Rules

1. A course session MUST NOT exceed its configured capacity; excess enrollments MUST go to waitlist if enabled.
2. Completion recording MUST require a passing score if the course has an exam component.
3. Compliance training rules MUST auto-enroll eligible employees within the configured due date window.
4. Certification assignments marked as `REQUIRED` MUST trigger alerts when overdue.
5. Learning path completion MUST be calculated based on the configured completion criteria.
6. An employee MUST NOT be enrolled in the same course session twice simultaneously.
7. Course materials marked as required MUST be accessed before completion can be recorded.
8. Instructors MUST NOT be assigned to overlapping sessions.
9. The system SHOULD send notifications at configurable intervals before enrollment deadlines.
10. Certification renewals MUST be tracked and reminders sent before expiry.
11. Non-compliance escalation actions MUST be executed within 24 hours of the grace period ending.
12. External instructor profiles MUST NOT reference an employee ID.
13. The system SHOULD support SCORM 1.2 and 2004 content tracking.
14. Learning history MUST be preserved even after course retirement.
15. Course evaluations SHOULD be anonymous and linked to session quality metrics.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package learning.v1;

service LearningService {
    // Courses
    rpc CreateCourse(CreateCourseRequest) returns (CreateCourseResponse);
    rpc GetCourse(GetCourseRequest) returns (GetCourseResponse);
    rpc ListCourses(ListCoursesRequest) returns (ListCoursesResponse);

    // Sessions
    rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);
    rpc GetSession(GetSessionRequest) returns (GetSessionResponse);

    // Learning paths
    rpc CreateLearningPath(CreateLearningPathRequest) returns (CreateLearningPathResponse);
    rpc AssignLearningPath(AssignLearningPathRequest) returns (AssignLearningPathResponse);

    // Enrollments
    rpc EnrollLearner(EnrollLearnerRequest) returns (EnrollLearnerResponse);
    rpc GetEnrollment(GetEnrollmentRequest) returns (GetEnrollmentResponse);
    rpc ListEmployeeEnrollments(ListEmployeeEnrollmentsRequest) returns (ListEmployeeEnrollmentsResponse);

    // Completions
    rpc RecordCompletion(RecordCompletionRequest) returns (RecordCompletionResponse);
    rpc GetLearningHistory(GetLearningHistoryRequest) returns (GetLearningHistoryResponse);

    // Certifications
    rpc CreateCertification(CreateCertificationRequest) returns (CreateCertificationResponse);
    rpc AssignCertification(AssignCertificationRequest) returns (AssignCertificationResponse);
    rpc GetEmployeeCertifications(GetEmployeeCertificationsRequest) returns (GetEmployeeCertificationsResponse);

    // Compliance
    rpc CheckCompliance(CheckComplianceRequest) returns (CheckComplianceResponse);
    rpc GetComplianceStatus(GetComplianceStatusRequest) returns (GetComplianceStatusResponse);
}

message Course {
    string id = 1;
    string tenant_id = 2;
    string course_code = 3;
    string course_title = 4;
    string course_type = 5;
    string category = 6;
    double duration_hours = 7;
    string difficulty_level = 8;
    string status = 9;
}

message CreateCourseRequest {
    string tenant_id = 1;
    string course_code = 2;
    string course_title = 3;
    string course_description = 4;
    string course_type = 5;
    string category = 6;
    double duration_hours = 7;
    string created_by = 8;
}

message CreateCourseResponse {
    Course course = 1;
}

message GetCourseRequest {
    string tenant_id = 1;
    string course_id = 2;
}

message GetCourseResponse {
    Course course = 1;
}

message ListCoursesRequest {
    string tenant_id = 1;
    string category = 2;
    string course_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListCoursesResponse {
    repeated Course courses = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message Session {
    string id = 1;
    string course_id = 2;
    string session_name = 3;
    string start_date = 4;
    string end_date = 5;
    int32 capacity = 6;
    int32 enrolled_count = 7;
    string status = 8;
}

message CreateSessionRequest {
    string tenant_id = 1;
    string course_id = 2;
    string session_name = 3;
    string start_date = 4;
    string end_date = 5;
    int32 capacity = 6;
    string created_by = 7;
}

message CreateSessionResponse {
    Session session = 1;
}

message GetSessionRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message GetSessionResponse {
    Session session = 1;
}

message LearningPath {
    string id = 1;
    string tenant_id = 2;
    string path_name = 3;
    string path_code = 4;
    int32 total_courses = 5;
    string status = 6;
}

message CreateLearningPathRequest {
    string tenant_id = 1;
    string path_name = 2;
    string path_code = 3;
    string description = 4;
    string course_sequence = 5;
    string created_by = 6;
}

message CreateLearningPathResponse {
    LearningPath path = 1;
}

message AssignLearningPathRequest {
    string tenant_id = 1;
    string path_id = 2;
    repeated string employee_ids = 3;
    string assigned_by = 4;
}

message AssignLearningPathResponse {
    int32 enrolled_count = 1;
}

message Enrollment {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string course_id = 4;
    string session_id = 5;
    string status = 6;
    double progress_percentage = 7;
    string enrollment_date = 8;
}

message EnrollLearnerRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string course_id = 3;
    string session_id = 4;
    string enrollment_type = 5;
    string created_by = 6;
}

message EnrollLearnerResponse {
    Enrollment enrollment = 1;
}

message GetEnrollmentRequest {
    string tenant_id = 1;
    string enrollment_id = 2;
}

message GetEnrollmentResponse {
    Enrollment enrollment = 1;
}

message ListEmployeeEnrollmentsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListEmployeeEnrollmentsResponse {
    repeated Enrollment enrollments = 1;
    string next_page_token = 2;
}

message Completion {
    string id = 1;
    string employee_id = 2;
    string course_id = 3;
    string completion_date = 4;
    double score = 5;
    string pass_fail = 6;
    double credits_earned = 7;
}

message RecordCompletionRequest {
    string tenant_id = 1;
    string enrollment_id = 2;
    string employee_id = 3;
    string course_id = 4;
    double score = 5;
    string pass_fail = 6;
    int32 time_spent_minutes = 7;
    string recorded_by = 8;
}

message RecordCompletionResponse {
    Completion completion = 1;
    string certificate_number = 2;
}

message GetLearningHistoryRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetLearningHistoryResponse {
    repeated Completion completions = 1;
    double total_credits = 2;
    double total_hours = 3;
}

message Certification {
    string id = 1;
    string tenant_id = 2;
    string certification_name = 3;
    string certification_code = 4;
    string certification_type = 5;
    int32 validity_period_months = 6;
    string status = 7;
}

message CreateCertificationRequest {
    string tenant_id = 1;
    string certification_name = 2;
    string certification_code = 3;
    string certification_type = 4;
    string required_courses = 5;
    string created_by = 6;
}

message CreateCertificationResponse {
    Certification certification = 1;
}

message AssignCertificationRequest {
    string tenant_id = 1;
    string certification_id = 2;
    string employee_id = 3;
    string assignment_type = 4;
    string due_date = 5;
    string assigned_by = 6;
}

message AssignCertificationResponse {
    string assignment_id = 1;
    string status = 2;
}

message EmployeeCertification {
    string id = 1;
    string certification_name = 2;
    string status = 3;
    string achieved_date = 4;
    string expiry_date = 5;
}

message GetEmployeeCertificationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message GetEmployeeCertificationsResponse {
    repeated EmployeeCertification certifications = 1;
}

message ComplianceEntry {
    string rule_name = 1;
    string course_title = 2;
    string status = 3;
    string due_date = 4;
}

message CheckComplianceRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message CheckComplianceResponse {
    bool is_compliant = 1;
    repeated ComplianceEntry non_compliant_items = 2;
    int32 total_rules = 3;
    int32 compliant_count = 4;
}

message GetComplianceStatusRequest {
    string tenant_id = 1;
    string department_id = 2;
}

message GetComplianceStatusResponse {
    int32 total_employees = 1;
    int32 compliant_count = 2;
    int32 non_compliant_count = 3;
    double compliance_percentage = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, job, grade, department | Determine learning population and eligibility |
| `performance-service` | Development goals, skill gaps | Recommend courses based on reviews |
| `succession-service` | Development actions | Fulfill development plan requirements |
| `career-service` | Skill gap analysis, recommendations | Targeted skill development |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Qualification updates | Update employee qualification records |
| `career-service` | Completion data | Update skill proficiency |
| `reporting-service` | Learning analytics | Training effectiveness and compliance reports |
| `succession-service` | Development completion | Track readiness progress |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `CourseCreated` | `learning.course.created` | `{ tenant_id, course_id, course_code, course_title, category, course_type }` | Published when a new course is added |
| `LearnerEnrolled` | `learning.learner.enrolled` | `{ tenant_id, enrollment_id, employee_id, course_id, session_id, enrollment_date }` | Published when a learner enrolls |
| `CourseCompleted` | `learning.course.completed` | `{ tenant_id, completion_id, employee_id, course_id, score, pass_fail, credits_earned }` | Published when a course is completed |
| `CertificationExpiring` | `learning.certification.expiring` | `{ tenant_id, assignment_id, employee_id, certification_name, expiry_date }` | Published when certification nears expiry |
| `ComplianceViolation` | `learning.compliance.violation` | `{ tenant_id, rule_id, employee_id, course_id, due_date, days_overdue }` | Published when compliance deadline is missed |
| `SessionCancelled` | `learning.session.cancelled` | `{ tenant_id, session_id, course_id, enrolled_count }` | Published when a session is cancelled |
| `LearningPathCompleted` | `learning.path.completed` | `{ tenant_id, employee_id, path_id, path_name, total_courses }` | Published when all path courses are done |
