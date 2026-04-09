# 62 - Core HR Service Specification

## 1. Domain Overview

The Core HR service is the foundational human resources module managing employee lifecycle, organizational structures, employment records, assignments, and workforce data. It serves as the system of record for all people-related data consumed by downstream HCM modules including Payroll, Benefits, Compensation, and Performance.

**Bounded Context:** Core Human Resources & Workforce Management
**Service Name:** `hr-service`
**Database:** `data/hr.db`
**HTTP Port:** 8094 | **gRPC Port:** 9094

---

## 2. Database Schema

### 2.1 Employees
```sql
CREATE TABLE employees (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_number TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    middle_name TEXT,
    preferred_name TEXT,
    display_name TEXT NOT NULL,
    email TEXT NOT NULL,
    work_phone TEXT,
    mobile_phone TEXT,
    date_of_birth TEXT,
    gender TEXT CHECK(gender IN ('MALE','FEMALE','NON_BINARY','PREFER_NOT_TO_SAY')),
    nationality TEXT,
    country_of_birth TEXT,
    national_identifier TEXT,
    national_identifier_type TEXT,
    marital_status TEXT CHECK(marital_status IN ('SINGLE','MARRIED','DIVORCED','WIDOWED','DOMESTIC_PARTNER','SEPARATED')),
    hire_date TEXT NOT NULL,
    original_hire_date TEXT NOT NULL,
    termination_date TEXT,
    termination_reason TEXT,
    last_working_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','TERMINATED','ON_LEAVE','PENDING','RETIRED')),
    worker_type TEXT NOT NULL DEFAULT 'EMPLOYEE' CHECK(worker_type IN ('EMPLOYEE','CONTRACTOR','CONTINGENT','INTERN','VOLUNTEER')),
    photo_url TEXT,
    honorific TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_number)
);

CREATE INDEX idx_employees_tenant_status ON employees(tenant_id, status);
CREATE INDEX idx_employees_tenant_hire_date ON employees(tenant_id, hire_date);
CREATE INDEX idx_employees_tenant_name ON employees(tenant_id, last_name, first_name);
```

### 2.2 Employment Records
```sql
CREATE TABLE employment_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    record_type TEXT NOT NULL CHECK(record_type IN ('HIRE','REHIRE','TRANSFER','PROMOTION','DEMOSSION','CHANGE','TERMINATION','LEAVE','RETURN_FROM_LEAVE')),
    effective_date TEXT NOT NULL,
    legal_employer_id TEXT NOT NULL,
    business_unit_id TEXT,
    department_id TEXT,
    job_id TEXT,
    position_id TEXT,
    location_id TEXT,
    grade_id TEXT,
    grade_step TEXT,
    manager_id TEXT,
    assignment_status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(assignment_status IN ('ACTIVE','INACTIVE','SUSPENDED')),
    reason_code TEXT,
    comments TEXT,
    approval_status TEXT NOT NULL DEFAULT 'PENDING' CHECK(approval_status IN ('PENDING','APPROVED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);

CREATE INDEX idx_emp_records_tenant_employee ON employment_records(tenant_id, employee_id, effective_date);
CREATE INDEX idx_emp_records_tenant_type ON employment_records(tenant_id, record_type);
```

### 2.3 Work Structures
```sql
CREATE TABLE work_structures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    structure_type TEXT NOT NULL CHECK(structure_type IN ('DEPARTMENT','BUSINESS_UNIT','DIVISION','LOCATION','JOB','POSITION','GRADE','GRADE_STEP','GRADE_LADDER')),
    structure_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    parent_id TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','PLANNED')),
    additional_attributes TEXT,  -- JSON for type-specific fields

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, structure_type, structure_code)
);

CREATE INDEX idx_work_structures_tenant_type ON work_structures(tenant_id, structure_type);
CREATE INDEX idx_work_structures_tenant_parent ON work_structures(tenant_id, parent_id);
```

### 2.4 Employee Assignments
```sql
CREATE TABLE employee_assignments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    assignment_number TEXT NOT NULL,
    assignment_type TEXT NOT NULL DEFAULT 'PRIMARY' CHECK(assignment_type IN ('PRIMARY','SECONDARY','PROJECT')),
    primary_flag INTEGER NOT NULL DEFAULT 1,
    legal_employer_id TEXT NOT NULL,
    business_unit_id TEXT,
    department_id TEXT,
    job_id TEXT,
    position_id TEXT,
    location_id TEXT,
    grade_id TEXT,
    grade_step TEXT,
    manager_id TEXT,
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT NOT NULL DEFAULT '4712-12-31',
    assignment_status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(assignment_status IN ('ACTIVE','INACTIVE','SUSPENDED','FUTURE')),
    work_hours_per_week INTEGER,
    full_time_flag INTEGER NOT NULL DEFAULT 1,
    probation_end_date TEXT,
    payroll_record_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, assignment_number)
);

CREATE INDEX idx_emp_assignments_tenant_employee ON employee_assignments(tenant_id, employee_id);
CREATE INDEX idx_emp_assignments_tenant_dept ON employee_assignments(tenant_id, department_id);
CREATE INDEX idx_emp_assignments_tenant_manager ON employee_assignments(tenant_id, manager_id);
CREATE INDEX idx_emp_assignments_tenant_effective ON employee_assignments(tenant_id, effective_start_date, effective_end_date);
```

### 2.5 Employee Contacts
```sql
CREATE TABLE employee_contacts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    contact_type TEXT NOT NULL CHECK(contact_type IN ('EMERGENCY','PERSONAL','WORK','REFERENCE')),
    contact_name TEXT NOT NULL,
    relationship TEXT,
    phone_primary TEXT,
    phone_secondary TEXT,
    email TEXT,
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT,
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);

CREATE INDEX idx_emp_contacts_tenant_employee ON employee_contacts(tenant_id, employee_id, contact_type);
```

### 2.6 Employee Qualifications
```sql
CREATE TABLE employee_qualifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    qualification_type TEXT NOT NULL CHECK(qualification_type IN ('EDUCATION','CERTIFICATION','LICENSE','SKILL','LANGUAGE','COMPETENCY','AWARD')),
    title TEXT NOT NULL,
    issuing_body TEXT,
    description TEXT,
    start_date TEXT,
    end_date TEXT,
    expiry_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','EXPIRED','REVOKED','PENDING_RENEWAL')),
    grade_score TEXT,
    verification_status TEXT DEFAULT 'UNVERIFIED' CHECK(verification_status IN ('UNVERIFIED','VERIFIED','REJECTED')),
    verified_by TEXT,
    verified_date TEXT,
    attachment_ids TEXT,  -- JSON array of document IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);

CREATE INDEX idx_emp_quals_tenant_employee ON employee_qualifications(tenant_id, employee_id, qualification_type);
CREATE INDEX idx_emp_quals_tenant_expiry ON employee_qualifications(tenant_id, expiry_date);
```

### 2.7 Employee Documents
```sql
CREATE TABLE employee_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN ('IDENTIFICATION','TAX','VISA','WORK_PERMIT','CONTRACT','CONFIDENTIAL','MEDICAL','OTHER')),
    document_name TEXT NOT NULL,
    file_reference TEXT NOT NULL,
    file_size_bytes INTEGER,
    mime_type TEXT,
    category TEXT,
    description TEXT,
    issue_date TEXT,
    expiry_date TEXT,
    confidentiality_level TEXT DEFAULT 'STANDARD' CHECK(confidentiality_level IN ('STANDARD','CONFIDENTIAL','RESTRICTED')),
    retention_until TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);

CREATE INDEX idx_emp_docs_tenant_employee ON employee_documents(tenant_id, employee_id, document_type);
```

### 2.8 Organizational Units
```sql
CREATE TABLE organizational_units (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    unit_code TEXT NOT NULL,
    unit_name TEXT NOT NULL,
    unit_type TEXT NOT NULL CHECK(unit_type IN ('ENTERPRISE','LEGAL_ENTITY','BUSINESS_UNIT','DIVISION','DEPARTMENT','TEAM','COST_CENTER')),
    parent_unit_id TEXT,
    head_employee_id TEXT,
    location_id TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','PLANNED','RESTRUCTURED')),
    headcount_budget INTEGER,
    description TEXT,
    external_code TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_unit_id) REFERENCES organizational_units(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, unit_code)
);

CREATE INDEX idx_org_units_tenant_type ON organizational_units(tenant_id, unit_type);
CREATE INDEX idx_org_units_tenant_parent ON organizational_units(tenant_id, parent_unit_id);
CREATE INDEX idx_org_units_tenant_status ON organizational_units(tenant_id, status);
```

### 2.9 Legal Employers
```sql
CREATE TABLE legal_employers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employer_code TEXT NOT NULL,
    legal_name TEXT NOT NULL,
    trading_name TEXT,
    registration_number TEXT NOT NULL,
    tax_identification_number TEXT,
    country TEXT NOT NULL,
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    employer_type TEXT NOT NULL CHECK(employer_type IN ('CORPORATION','PARTNERSHIP','SOLE_PROPRIETORSHIP','GOVERNMENT','NON_PROFIT','LLC')),
    industry_code TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employer_code)
);

CREATE INDEX idx_legal_employers_tenant_country ON legal_employers(tenant_id, country);
```

---

## 3. REST API Endpoints

### Employee Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees` | Create a new employee with primary assignment |
| GET | `/api/v1/employees` | List employees with filtering and pagination |
| GET | `/api/v1/employees/{id}` | Get employee full profile |
| PUT | `/api/v1/employees/{id}` | Update employee personal details |
| PATCH | `/api/v1/employees/{id}/status` | Change employee status (activate, terminate) |
| GET | `/api/v1/employees/search` | Full-text search across employee records |
| POST | `/api/v1/employees/bulk` | Bulk create employees |
| GET | `/api/v1/employees/{id}/history` | Get employee change history |

### Assignment Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees/{id}/assignments` | Create a new assignment |
| GET | `/api/v1/employees/{id}/assignments` | List all assignments for employee |
| PUT | `/api/v1/assignments/{assignmentId}` | Update assignment details |
| GET | `/api/v1/assignments/{assignmentId}/history` | Get assignment change history |

### Transfer and Promotion
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees/{id}/transfer` | Initiate employee transfer |
| POST | `/api/v1/employees/{id}/promote` | Initiate employee promotion |
| POST | `/api/v1/employees/{id}/demote` | Initiate employee demotion |
| GET | `/api/v1/transfers` | List pending/completed transfers |
| GET | `/api/v1/promotions` | List pending/completed promotions |

### Organization
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/org-units` | Create organizational unit |
| GET | `/api/v1/org-units` | List organizational units |
| GET | `/api/v1/org-units/{id}` | Get organizational unit details |
| PUT | `/api/v1/org-units/{id}` | Update organizational unit |
| GET | `/api/v1/org-units/{id}/hierarchy` | Get org unit with children |
| GET | `/api/v1/org-chart` | Get full org chart tree |
| GET | `/api/v1/org-units/{id}/headcount` | Get headcount for unit and sub-units |
| POST | `/api/v1/legal-employers` | Create legal employer |
| GET | `/api/v1/legal-employers` | List legal employers |

### Work Structures
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/work-structures` | Create work structure (job, position, grade, etc.) |
| GET | `/api/v1/work-structures?type={type}` | List structures by type |
| PUT | `/api/v1/work-structures/{id}` | Update work structure |
| GET | `/api/v1/work-structures/{id}/tree` | Get hierarchy tree for structure |

### Contacts, Qualifications, Documents
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees/{id}/contacts` | Add employee contact |
| GET | `/api/v1/employees/{id}/contacts` | List employee contacts |
| PUT | `/api/v1/contacts/{contactId}` | Update contact |
| DELETE | `/api/v1/contacts/{contactId}` | Delete contact |
| POST | `/api/v1/employees/{id}/qualifications` | Add qualification |
| GET | `/api/v1/employees/{id}/qualifications` | List qualifications |
| PUT | `/api/v1/qualifications/{qualId}` | Update qualification |
| POST | `/api/v1/employees/{id}/documents` | Upload employee document |
| GET | `/api/v1/employees/{id}/documents` | List employee documents |
| GET | `/api/v1/documents/{docId}/download` | Download document file |

---

## 4. Business Rules

1. An employee MUST have exactly one primary assignment at any given time.
2. An employee number MUST be unique within a tenant and SHOULD be auto-generated if not provided.
3. A transfer MUST create a new employment record with type `TRANSFER` and update the primary assignment effective on the transfer date.
4. A promotion MUST create a new employment record with type `PROMOTION` and update grade, job, or position on the effective date.
5. Employee status changes MUST be auditable with reason codes and approval workflows.
6. Organizational units MUST support hierarchical nesting up to 10 levels deep.
7. An assignment's effective dates MUST NOT overlap for the same assignment number.
8. A legal employer MUST be referenced by at least one active assignment before it can be deactivated.
9. Termination of an employee MUST set the termination date on the employee record and end-date all active assignments.
10. Work structures (jobs, positions, grades) MUST support date-effective changes with no gaps in validity.
11. Employee documents marked as `CONFIDENTIAL` or `RESTRICTED` MUST require elevated access roles for viewing.
12. The system SHOULD validate national identifier formats based on the employee's country.
13. Secondary assignments MUST reference a different legal employer or department than the primary assignment.
14. A manager assignment change MUST trigger re-evaluation of the reporting hierarchy.
15. Bulk employee creation MUST be limited to 500 records per request.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package hr.v1;

service HRService {
    // Employee operations
    rpc CreateEmployee(CreateEmployeeRequest) returns (CreateEmployeeResponse);
    rpc GetEmployee(GetEmployeeRequest) returns (GetEmployeeResponse);
    rpc UpdateEmployee(UpdateEmployeeRequest) returns (UpdateEmployeeResponse);
    rpc ListEmployees(ListEmployeesRequest) returns (ListEmployeesResponse);
    rpc SearchEmployees(SearchEmployeesRequest) returns (SearchEmployeesResponse);
    rpc TerminateEmployee(TerminateEmployeeRequest) returns (TerminateEmployeeResponse);

    // Assignment operations
    rpc CreateAssignment(CreateAssignmentRequest) returns (CreateAssignmentResponse);
    rpc GetAssignment(GetAssignmentRequest) returns (GetAssignmentResponse);
    rpc ListAssignments(ListAssignmentsRequest) returns (ListAssignmentsResponse);
    rpc UpdateAssignment(UpdateAssignmentRequest) returns (UpdateAssignmentResponse);

    // Transfer and promotion
    rpc TransferEmployee(TransferEmployeeRequest) returns (TransferEmployeeResponse);
    rpc PromoteEmployee(PromoteEmployeeRequest) returns (PromoteEmployeeResponse);

    // Organization
    rpc GetOrgChart(GetOrgChartRequest) returns (GetOrgChartResponse);
    rpc GetOrgUnit(GetOrgUnitRequest) returns (GetOrgUnitResponse);
    rpc ListOrgUnits(ListOrgUnitsRequest) returns (ListOrgUnitsResponse);
    rpc GetHeadcount(GetHeadcountRequest) returns (GetHeadcountResponse);

    // Direct reports
    rpc GetDirectReports(GetDirectReportsRequest) returns (GetDirectReportsResponse);
}

message Employee {
    string id = 1;
    string tenant_id = 2;
    string employee_number = 3;
    string first_name = 4;
    string last_name = 5;
    string email = 6;
    string hire_date = 7;
    string status = 8;
    string worker_type = 9;
    string primary_assignment_id = 10;
}

message CreateEmployeeRequest {
    string tenant_id = 1;
    string employee_number = 2;
    string first_name = 3;
    string last_name = 4;
    string email = 5;
    string hire_date = 6;
    string worker_type = 7;
    string legal_employer_id = 8;
    string job_id = 9;
    string department_id = 10;
    string location_id = 11;
    string created_by = 12;
}

message CreateEmployeeResponse {
    Employee employee = 1;
    string assignment_id = 2;
}

message GetEmployeeRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message GetEmployeeResponse {
    Employee employee = 1;
}

message UpdateEmployeeRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string first_name = 3;
    string last_name = 4;
    string email = 5;
    string updated_by = 6;
    int32 version = 7;
}

message UpdateEmployeeResponse {
    Employee employee = 1;
}

message ListEmployeesRequest {
    string tenant_id = 1;
    string status = 2;
    string department_id = 3;
    string manager_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListEmployeesResponse {
    repeated Employee employees = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SearchEmployeesRequest {
    string tenant_id = 1;
    string query = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message SearchEmployeesResponse {
    repeated Employee employees = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TerminateEmployeeRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string termination_date = 3;
    string termination_reason = 4;
    string last_working_date = 5;
    string updated_by = 6;
}

message TerminateEmployeeResponse {
    Employee employee = 1;
}

message Assignment {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string assignment_number = 4;
    string assignment_type = 5;
    string legal_employer_id = 6;
    string department_id = 7;
    string job_id = 8;
    string position_id = 9;
    string grade_id = 10;
    string manager_id = 11;
    string effective_start_date = 12;
    string effective_end_date = 13;
    string assignment_status = 14;
}

message CreateAssignmentRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string assignment_type = 3;
    string legal_employer_id = 4;
    string department_id = 5;
    string job_id = 6;
    string position_id = 7;
    string grade_id = 8;
    string manager_id = 9;
    string effective_start_date = 10;
    string created_by = 11;
}

message CreateAssignmentResponse {
    Assignment assignment = 1;
}

message GetAssignmentRequest {
    string tenant_id = 1;
    string assignment_id = 2;
}

message GetAssignmentResponse {
    Assignment assignment = 1;
}

message ListAssignmentsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string effective_date = 3;
}

message ListAssignmentsResponse {
    repeated Assignment assignments = 1;
}

message UpdateAssignmentRequest {
    string tenant_id = 1;
    string assignment_id = 2;
    string department_id = 3;
    string job_id = 4;
    string grade_id = 5;
    string manager_id = 6;
    string effective_start_date = 7;
    string updated_by = 8;
    int32 version = 9;
}

message UpdateAssignmentResponse {
    Assignment assignment = 1;
}

message TransferEmployeeRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string to_department_id = 3;
    string to_job_id = 4;
    string to_location_id = 5;
    string to_manager_id = 6;
    string effective_date = 7;
    string reason_code = 8;
    string updated_by = 9;
}

message TransferEmployeeResponse {
    Assignment new_assignment = 1;
    string employment_record_id = 2;
}

message PromoteEmployeeRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string new_grade_id = 3;
    string new_job_id = 4;
    string new_position_id = 5;
    string effective_date = 6;
    string reason_code = 7;
    string updated_by = 8;
}

message PromoteEmployeeResponse {
    Assignment new_assignment = 1;
    string employment_record_id = 2;
}

message OrgUnitNode {
    string id = 1;
    string unit_name = 2;
    string unit_type = 3;
    string head_employee_id = 4;
    repeated OrgUnitNode children = 5;
}

message GetOrgChartRequest {
    string tenant_id = 1;
    string root_unit_id = 2;
    int32 max_depth = 3;
}

message GetOrgChartResponse {
    OrgUnitNode root = 1;
}

message GetOrgUnitRequest {
    string tenant_id = 1;
    string unit_id = 2;
}

message GetOrgUnitResponse {
    OrgUnitNode unit = 1;
}

message ListOrgUnitsRequest {
    string tenant_id = 1;
    string unit_type = 2;
    string parent_unit_id = 3;
}

message ListOrgUnitsResponse {
    repeated OrgUnitNode units = 1;
}

message GetHeadcountRequest {
    string tenant_id = 1;
    string unit_id = 2;
    bool include_sub_units = 3;
    string effective_date = 4;
}

message GetHeadcountResponse {
    string unit_id = 1;
    int32 headcount = 2;
    int32 total_headcount_including_sub_units = 3;
}

message GetDirectReportsRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string effective_date = 3;
}

message GetDirectReportsResponse {
    repeated Employee direct_reports = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `auth-service` | User identity, roles | Link employees to user accounts |
| `workflow-service` | Approval status | Process transfer/promotion approvals |
| `document-service` | File storage | Store and retrieve employee documents |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `payroll-service` | Employee assignments, salary data | Process payroll |
| `compensation-service` | Employee grades, job data | Calculate compensation |
| `benefits-service` | Employee eligibility data | Determine benefit enrollment |
| `performance-service` | Employee and manager data | Manage reviews |
| `learning-service` | Employee job and grade data | Assign training |
| `absence-service` | Employee assignments | Manage leave balances |
| `timelabor-service` | Employee assignments | Track time and labor |
| `recruiting-service` | New hire data | Complete hiring process |
| `reporting-service` | Workforce analytics | Headcount and turnover reports |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `EmployeeHired` | `hr.employee.hired` | `{ tenant_id, employee_id, employee_number, hire_date, legal_employer_id, department_id, job_id }` | Published when a new employee is created |
| `EmployeeTransferred` | `hr.employee.transferred` | `{ tenant_id, employee_id, from_department_id, to_department_id, from_job_id, to_job_id, effective_date }` | Published when an employee transfer is completed |
| `EmployeePromoted` | `hr.employee.promoted` | `{ tenant_id, employee_id, old_grade_id, new_grade_id, old_job_id, new_job_id, effective_date }` | Published when a promotion is finalized |
| `EmployeeTerminated` | `hr.employee.terminated` | `{ tenant_id, employee_id, termination_date, termination_reason, last_working_date }` | Published when an employee is terminated |
| `AssignmentChanged` | `hr.assignment.changed` | `{ tenant_id, employee_id, assignment_id, change_type, old_values, new_values, effective_date }` | Published when any assignment is modified |
| `EmployeeUpdated` | `hr.employee.updated` | `{ tenant_id, employee_id, changed_fields, version }` | Published when employee personal data is updated |
| `OrgUnitChanged` | `hr.orgunit.changed` | `{ tenant_id, unit_id, unit_type, change_type }` | Published when organizational structure changes |
