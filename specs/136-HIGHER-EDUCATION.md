# 136 - Higher Education Specification

## 1. Domain Overview

Higher Education provides student management capabilities for colleges and universities, including academic structure management, student records, course catalog, enrollment, financial aid, advising, and degree audit. It adapts the Fusion platform's core financial and HCM capabilities to the unique needs of educational institutions.

**Bounded Context:** Higher Education & Student Management
**Service Name:** `highered-service`
**Database:** `data/highered.db`
**HTTP Port:** 8156 | **gRPC Port:** 9156

---

## 2. Database Schema

### 2.1 Academic Structure
```sql
CREATE TABLE he_institutions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    institution_name TEXT NOT NULL,
    institution_code TEXT NOT NULL,
    institution_type TEXT NOT NULL CHECK(institution_type IN ('UNIVERSITY','COLLEGE','COMMUNITY_COLLEGE','VOCATIONAL','GRADUATE_SCHOOL')),
    accreditation TEXT,
    address_line1 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, institution_code)
);

CREATE TABLE he_colleges (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    institution_id TEXT NOT NULL,
    college_name TEXT NOT NULL,
    college_code TEXT NOT NULL,
    dean_name TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (institution_id) REFERENCES he_institutions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, college_code)
);

CREATE TABLE he_departments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    college_id TEXT NOT NULL,
    department_name TEXT NOT NULL,
    department_code TEXT NOT NULL,
    chair_name TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (college_id) REFERENCES he_colleges(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, department_code)
);
```

### 2.2 Academic Periods
```sql
CREATE TABLE he_academic_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,            -- "Fall 2024", "Spring 2025"
    period_code TEXT NOT NULL,            -- "2024-FA", "2025-SP"
    period_type TEXT NOT NULL CHECK(period_type IN ('SEMESTER','QUARTER','TRIMESTER','SUMMER','INTERSESSION')),
    academic_year TEXT NOT NULL,          -- "2024-2025"
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    registration_start TEXT,
    registration_end TEXT,
    add_drop_deadline TEXT,
    withdrawal_deadline TEXT,
    final_exam_start TEXT,
    final_exam_end TEXT,
    grading_deadline TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','REGISTRATION_OPEN','IN_SESSION','GRADING','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_code)
);
```

### 2.3 Programs & Curricula
```sql
CREATE TABLE he_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_name TEXT NOT NULL,
    program_code TEXT NOT NULL,
    degree_level TEXT NOT NULL CHECK(degree_level IN ('ASSOCIATE','BACHELOR','MASTER','DOCTORAL','CERTIFICATE','DIPLOMA')),
    department_id TEXT NOT NULL,
    total_credits_required REAL NOT NULL,
    min_gpa_required REAL NOT NULL DEFAULT 2.0,
    description TEXT,
    learning_outcomes TEXT,               -- JSON array of learning outcomes
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','SUSPENDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (department_id) REFERENCES he_departments(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, program_code)
);
```

### 2.4 Course Catalog
```sql
CREATE TABLE he_courses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    course_code TEXT NOT NULL,            -- "CS101"
    course_name TEXT NOT NULL,
    description TEXT,
    department_id TEXT NOT NULL,
    credits REAL NOT NULL,
    contact_hours INTEGER NOT NULL,
    prerequisites TEXT,                   -- JSON array of course_ids
    corequisites TEXT,
    course_level TEXT NOT NULL CHECK(course_level IN ('INTRODUCTORY','INTERMEDIATE','ADVANCED','GRADUATE')),
    delivery_mode TEXT NOT NULL DEFAULT 'IN_PERSON'
        CHECK(delivery_mode IN ('IN_PERSON','ONLINE','HYBRID','HYFLEX')),
    max_enrollment INTEGER NOT NULL DEFAULT 30,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','UNDER_REVIEW')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, course_code)
);
```

### 2.5 Students
```sql
CREATE TABLE he_students (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    student_id_number TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT NOT NULL,
    phone TEXT,
    date_of_birth TEXT,
    gender TEXT,
    address_line1 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT NOT NULL,
    program_id TEXT,
    admission_date TEXT NOT NULL,
    expected_graduation TEXT,
    enrollment_status TEXT NOT NULL DEFAULT 'PROSPECTIVE'
        CHECK(enrollment_status IN ('PROSPECTIVE','APPLIED','ADMITTED','ENROLLED','ON_LEAVE','WITHDRAWN','GRADUATED','SUSPENDED')),
    academic_standing TEXT NOT NULL DEFAULT 'GOOD'
        CHECK(academic_standing IN ('GOOD','WARNING','PROBATION','SUSPENDED','DISMISSED')),
    cumulative_gpa REAL NOT NULL DEFAULT 0.0,
    cumulative_credits REAL NOT NOT NULL DEFAULT 0.0,
    advisor_id TEXT,
    cohort_year TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES he_programs(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, student_id_number)
);

CREATE INDEX idx_he_students_tenant_status ON he_students(tenant_id, enrollment_status);
CREATE INDEX idx_he_students_tenant_program ON he_students(tenant_id, program_id);
```

### 2.6 Course Offerings & Enrollment
```sql
CREATE TABLE he_course_offerings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    course_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    section_number TEXT NOT NULL,         -- "001", "002"
    instructor_id TEXT,
    schedule_days TEXT,                   -- "MWF", "TR"
    start_time TEXT,
    end_time TEXT,
    location TEXT,
    current_enrollment INTEGER NOT NULL DEFAULT 0,
    max_enrollment INTEGER NOT NULL DEFAULT 30,
    waitlist_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','OPEN','CLOSED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (course_id) REFERENCES he_courses(id) ON DELETE RESTRICT,
    FOREIGN KEY (period_id) REFERENCES he_academic_periods(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, course_id, period_id, section_number)
);

CREATE TABLE he_enrollments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    student_id TEXT NOT NULL,
    offering_id TEXT NOT NULL,
    enrollment_date TEXT NOT NULL DEFAULT (datetime('now')),
    status TEXT NOT NULL DEFAULT 'ENROLLED'
        CHECK(status IN ('ENROLLED','DROPPED','WITHDRAWN','COMPLETED','AUDIT')),
    final_grade TEXT,
    grade_points REAL,
    credits_earned REAL NOT NULL DEFAULT 0.0,
    last_attendance_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    FOREIGN KEY (student_id) REFERENCES he_students(id) ON DELETE RESTRICT,
    FOREIGN KEY (offering_id) REFERENCES he_course_offerings(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, student_id, offering_id)
);

CREATE INDEX idx_he_enrollments_tenant_student ON he_enrollments(tenant_id, student_id);
CREATE INDEX idx_he_enrollments_tenant_offering ON he_enrollments(tenant_id, offering_id);
```

### 2.7 Financial Aid
```sql
CREATE TABLE he_financial_aid (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    student_id TEXT NOT NULL,
    aid_type TEXT NOT NULL CHECK(aid_type IN ('SCHOLARSHIP','GRANT','LOAN','WORK_STUDY','TUITION_WAIVER')),
    aid_name TEXT NOT NULL,
    award_amount_cents INTEGER NOT NULL,
    disbursed_amount_cents INTEGER NOT NULL DEFAULT 0,
    academic_year TEXT NOT NULL,
    period_id TEXT,
    status TEXT NOT NULL DEFAULT 'AWARDED'
        CHECK(status IN ('APPLIED','AWARDED','DISBURSED','CANCELLED','EXPIRED')),
    eligibility_criteria TEXT,
    renewal_requirements TEXT,
    funding_source TEXT,
    external_award_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (student_id) REFERENCES he_students(id) ON DELETE RESTRICT
);

CREATE INDEX idx_he_aid_tenant_student ON he_financial_aid(tenant_id, student_id);
CREATE INDEX idx_he_aid_tenant_year ON he_financial_aid(tenant_id, academic_year);
```

---

## 3. REST API Endpoints

### 3.1 Academic Structure
```
GET    /api/v1/highered/institutions                    Permission: he.structure.read
POST   /api/v1/highered/institutions                    Permission: he.structure.create
GET    /api/v1/highered/colleges                         Permission: he.structure.read
POST   /api/v1/highered/colleges                         Permission: he.structure.create
GET    /api/v1/highered/departments                      Permission: he.structure.read
POST   /api/v1/highered/departments                      Permission: he.structure.create
GET    /api/v1/highered/periods                          Permission: he.structure.read
POST   /api/v1/highered/periods                          Permission: he.structure.create
```

### 3.2 Programs & Courses
```
GET    /api/v1/highered/programs                        Permission: he.programs.read
POST   /api/v1/highered/programs                        Permission: he.programs.create
GET    /api/v1/highered/courses                         Permission: he.courses.read
POST   /api/v1/highered/courses                         Permission: he.courses.create
GET    /api/v1/highered/offerings
  ?period_id={id}&department_id={id}                    Permission: he.courses.read
POST   /api/v1/highered/offerings                        Permission: he.courses.create
```

### 3.3 Students
```
GET    /api/v1/highered/students                         Permission: he.students.read
GET    /api/v1/highered/students/{id}                    Permission: he.students.read
POST   /api/v1/highered/students                         Permission: he.students.create
PUT    /api/v1/highered/students/{id}                    Permission: he.students.update
GET    /api/v1/highered/students/{id}/transcript         Permission: he.students.read
GET    /api/v1/highered/students/{id}/degree-audit       Permission: he.students.read
  Response: Degree progress, requirements met, remaining requirements
GET    /api/v1/highered/students/{id}/schedule           Permission: he.students.read
GET    /api/v1/highered/students/{id}/financial-aid      Permission: he.students.read
```

### 3.4 Enrollment
```
GET    /api/v1/highered/enrollment                       Permission: he.enrollment.read
POST   /api/v1/highered/enrollment/register              Permission: he.enrollment.register
  Request: { "student_id": "...", "offering_ids": [...] }
POST   /api/v1/highered/enrollment/drop                  Permission: he.enrollment.drop
GET    /api/v1/highered/offerings/{id}/roster            Permission: he.enrollment.read
POST   /api/v1/highered/offerings/{id}/grades            Permission: he.enrollment.grade
  Request: { "grades": [{ "student_id": "...", "grade": "A" }] }
GET    /api/v1/highered/enrollment/waitlist              Permission: he.enrollment.read
```

### 3.5 Financial Aid
```
GET    /api/v1/highered/financial-aid                    Permission: he.finaid.read
POST   /api/v1/highered/financial-aid                    Permission: he.finaid.create
PUT    /api/v1/highered/financial-aid/{id}               Permission: he.finaid.update
POST   /api/v1/highered/financial-aid/{id}/disburse      Permission: he.finaid.disburse
GET    /api/v1/highered/financial-aid/eligibility
  ?student_id={id}&academic_year={year}                  Permission: he.finaid.read
```

---

## 4. Business Rules

### 4.1 Enrollment
- Students can only enroll during open registration period
- Prerequisites checked before enrollment allowed
- Max enrollment enforced; overflow goes to waitlist
- Waitlist automatically promotes when seats open
- Add/drop deadline enforced
- Late withdrawal requires advisor approval and creates "W" grade

### 4.2 Grading
- Grade scale configurable per institution (letter, +/-, pass/fail, credit/no-credit)
- GPA calculated: SUM(grade_points × credits) / SUM(credits)
- Academic standing reviewed at end of each period
- Probation triggers mandatory advisor meeting
- Dean's list for students exceeding GPA threshold (default 3.5)

### 4.3 Degree Audit
- Degree requirements defined by program curriculum
- Audit checks: required courses, elective credits, GPA, total credits
- Transfer credits evaluated and applied toward requirements
- Substitution petitions workflow for course equivalencies
- Graduation candidacy check automated before commencement

### 4.4 Financial Aid
- Aid eligibility based on FAFSA data and institutional criteria
- Satisfactory Academic Progress (SAP) required for continued eligibility
- Disbursement split across periods per enrollment intensity
- Return of Title IV funds calculated for mid-term withdrawals
- Aggregate loan limits enforced per student

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.highered.v1;

service HigherEducationService {
    rpc GetStudentTranscript(GetStudentTranscriptRequest) returns (GetStudentTranscriptResponse);
    rpc RegisterForCourses(RegisterForCoursesRequest) returns (RegisterForCoursesResponse);
    rpc GetDegreeAudit(GetDegreeAuditRequest) returns (GetDegreeAuditResponse);
    rpc CheckPrerequisites(CheckPrerequisitesRequest) returns (CheckPrerequisitesResponse);
    rpc CalculateGPA(CalculateGPARequest) returns (CalculateGPAResponse);
    rpc DisburseFinancialAid(DisburseFinancialAidRequest) returns (DisburseFinancialAidResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **hr-service**: Faculty and staff records
- **payroll-service**: Faculty payroll, work-study payments
- **gl-service**: Tuition revenue, financial aid accounting
- **ar-service**: Student billing, tuition invoices
- **ap-service**: Vendor payments, research grants
- **pm-service**: Research project management
- **grant-service**: Research grant lifecycle management
- **workflow-service**: Registration approval, grade changes, petition workflows

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `he.student.admitted` | Student admission confirmed | student_id, program_id |
| `he.student.enrolled` | Student registered for courses | student_id, offering_ids |
| `he.student.dropped` | Course dropped | student_id, offering_id |
| `he.student.graduated` | Degree conferred | student_id, program_id, degree |
| `he.grade.submitted` | Final grade posted | student_id, course_id, grade |
| `he.aid.awarded` | Financial aid awarded | student_id, aid_type, amount |
| `he.aid.disbursed` | Financial aid disbursed | student_id, amount |
| `he.standing.changed` | Academic standing updated | student_id, old_standing, new_standing |

---

## 7. Migrations

### Migration Order for highered-service:
1. V001: `he_institutions`, `he_colleges`, `he_departments`
2. V002: `he_academic_periods`
3. V003: `he_programs`
4. V004: `he_courses`
5. V005: `he_students`
6. V006: `he_course_offerings`
7. V007: `he_enrollments`
8. V008: `he_financial_aid`
9. V009: Triggers for `updated_at`
10. V010: Seed data (sample institution, departments, grade scale, sample programs)
