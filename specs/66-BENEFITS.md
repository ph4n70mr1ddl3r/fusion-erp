# 66 - Benefits Service Specification

## 1. Domain Overview

The Benefits service manages employee benefit plan administration including plan definitions, eligibility rules, enrollment processing, life event management, open enrollment periods, cost calculations, provider integration, and beneficiary designations. It supports medical, dental, vision, life insurance, disability, retirement, and flexible spending benefit types.

**Bounded Context:** Benefits Administration & Enrollment
**Service Name:** `benefits-service`
**Database:** `data/benefits.db`
**HTTP Port:** 8098 | **gRPC Port:** 9098

---

## 2. Database Schema

### 2.1 Benefit Plans
```sql
CREATE TABLE benefit_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('MEDICAL','DENTAL','VISION','LIFE_INSURANCE','DISABILITY','RETIREMENT_401K','RETIREMENT_PENSION','HSA','FSA','HRA','EAP','WELLNESS','TRANSPORTATION','TUITION_ASSISTANCE','LEGAL')),
    plan_category TEXT NOT NULL CHECK(plan_category IN ('HEALTH','INSURANCE','RETIREMENT','SPENDING_ACCOUNT','LIFESTYLE')),
    carrier_name TEXT,
    carrier_plan_id TEXT,
    coverage_level TEXT NOT NULL CHECK(coverage_level IN ('EMPLOYEE_ONLY','EMPLOYEE_SPOUSE','EMPLOYEE_CHILD','FAMILY')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    enrollment_type TEXT NOT NULL DEFAULT 'OPEN_ENROLLMENT' CHECK(enrollment_type IN ('OPEN_ENROLLMENT','NEW_HIRE','LIFE_EVENT','ALWAYS_OPEN')),
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','PENDING','TERMINATED')),
    description TEXT,
    plan_document_url TEXT,
    minimum_participation_hours INTEGER DEFAULT 30,
    waiting_period_days INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_benefit_plans_tenant_type ON benefit_plans(tenant_id, plan_type);
CREATE INDEX idx_benefit_plans_tenant_status ON benefit_plans(tenant_id, status);
```

### 2.2 Plan Options
```sql
CREATE TABLE plan_options (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    option_name TEXT NOT NULL,
    option_code TEXT NOT NULL,
    coverage_level TEXT NOT NULL CHECK(coverage_level IN ('EMPLOYEE_ONLY','EMPLOYEE_SPOUSE','EMPLOYEE_CHILD','FAMILY')),
    deductable_amount INTEGER NOT NULL DEFAULT 0,  -- cents
    out_of_pocket_max INTEGER NOT NULL DEFAULT 0,
    copay_amount INTEGER DEFAULT 0,
    coinsurance_percentage DECIMAL(5,2) DEFAULT 0,
    in_network_flag INTEGER DEFAULT 1,
    is_default INTEGER DEFAULT 0,
    display_order INTEGER DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES benefit_plans(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, plan_id, option_code)
);

CREATE INDEX idx_plan_options_tenant_plan ON plan_options(tenant_id, plan_id);
```

### 2.3 Plan Rates
```sql
CREATE TABLE plan_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    option_id TEXT NOT NULL,
    coverage_level TEXT NOT NULL,
    employee_contribution INTEGER NOT NULL,   -- cents per period
    employer_contribution INTEGER NOT NULL,   -- cents per period
    total_cost INTEGER NOT NULL,              -- cents per period
    pay_period_type TEXT NOT NULL DEFAULT 'MONTHLY' CHECK(pay_period_type IN ('WEEKLY','BIWEEKLY','SEMI_MONTHLY','MONTHLY','ANNUAL')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    age_based INTEGER DEFAULT 0,
    min_age INTEGER,
    max_age INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES benefit_plans(id) ON DELETE CASCADE,
    FOREIGN KEY (option_id) REFERENCES plan_options(id) ON DELETE CASCADE
);

CREATE INDEX idx_plan_rates_tenant_plan ON plan_rates(tenant_id, plan_id, effective_from);
CREATE INDEX idx_plan_rates_tenant_option ON plan_rates(tenant_id, option_id, coverage_level);
```

### 2.4 Benefit Eligibility Rules
```sql
CREATE TABLE benefit_eligibility_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('EMPLOYMENT_STATUS','HOURS_WORKED','LENGTH_OF_SERVICE','LOCATION','JOB_GRADE','AGE','CUSTOM')),
    condition_operator TEXT NOT NULL DEFAULT 'AND' CHECK(condition_operator IN ('AND','OR')),
    condition_expression TEXT NOT NULL,  -- JSON: field, operator, value
    priority INTEGER DEFAULT 100,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES benefit_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_eligibility_rules_tenant_plan ON benefit_eligibility_rules(tenant_id, plan_id);
```

### 2.5 Benefit Enrollments
```sql
CREATE TABLE benefit_enrollments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    option_id TEXT NOT NULL,
    coverage_level TEXT NOT NULL,
    enrollment_type TEXT NOT NULL CHECK(enrollment_type IN ('OPEN_ENROLLMENT','NEW_HIRE','LIFE_EVENT','QUALIFYING_EVENT','TRANSFER')),
    enrollment_status TEXT NOT NULL DEFAULT 'PENDING' CHECK(enrollment_status IN ('PENDING','ACTIVE','WAIVED','CANCELLED','TERMINATED','SUSPENDED')),
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT,
    employee_cost_per_period INTEGER NOT NULL DEFAULT 0,  -- cents
    employer_cost_per_period INTEGER NOT NULL DEFAULT 0,
    total_cost_per_period INTEGER NOT NULL DEFAULT 0,
    payroll_deduction_element_id TEXT,
    waive_reason TEXT,
    coverage_elected_date TEXT,
    enrollment_submitted_date TEXT,
    approval_status TEXT DEFAULT 'APPROVED' CHECK(approval_status IN ('PENDING','APPROVED','REJECTED')),
    approved_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES benefit_plans(id) ON DELETE RESTRICT,
    FOREIGN KEY (option_id) REFERENCES plan_options(id) ON DELETE RESTRICT
);

CREATE INDEX idx_benefit_enroll_tenant_employee ON benefit_enrollments(tenant_id, employee_id, enrollment_status);
CREATE INDEX idx_benefit_enroll_tenant_plan ON benefit_enrollments(tenant_id, plan_id);
CREATE INDEX idx_benefit_enroll_tenant_dates ON benefit_enrollments(tenant_id, effective_start_date, effective_end_date);
```

### 2.6 Benefit Beneficiaries
```sql
CREATE TABLE benefit_beneficiaries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    enrollment_id TEXT NOT NULL,
    beneficiary_name TEXT NOT NULL,
    relationship TEXT NOT NULL CHECK(relationship IN ('SPOUSE','DOMESTIC_PARTNER','CHILD','PARENT','SIBLING','OTHER')),
    date_of_birth TEXT,
    gender TEXT,
    ssn_last_four TEXT,
    address_line1 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT,
    phone TEXT,
    email TEXT,
    allocation_percentage DECIMAL(5,2) NOT NULL DEFAULT 100.00,
    is_contingent INTEGER DEFAULT 0,
    is_primary INTEGER DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (enrollment_id) REFERENCES benefit_enrollments(id) ON DELETE CASCADE
);

CREATE INDEX idx_benefit_benef_tenant_employee ON benefit_beneficiaries(tenant_id, employee_id);
```

### 2.7 Life Events
```sql
CREATE TABLE life_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN ('MARRIAGE','DIVORCE','BIRTH','ADOPTION','DEATH_OF_DEPENDENT','SPOUSE_JOB_CHANGE','DEPENDENT_AGE_OUT','ADDRESS_CHANGE','EMPLOYMENT_STATUS_CHANGE','COURT_ORDER','OTHER')),
    event_date TEXT NOT NULL,
    reported_date TEXT NOT NULL,
    description TEXT,
    documentation_provided INTEGER DEFAULT 0,
    documentation_urls TEXT,  -- JSON array
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','VERIFIED','REJECTED','EXPIRED')),
    verified_by TEXT,
    verified_date TEXT,
    enrollment_window_start TEXT NOT NULL,
    enrollment_window_end TEXT NOT NULL,
    enrollment_window_used INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);

CREATE INDEX idx_life_events_tenant_employee ON life_events(tenant_id, employee_id, event_date);
CREATE INDEX idx_life_events_tenant_status ON life_events(tenant_id, status);
CREATE INDEX idx_life_events_tenant_window ON life_events(tenant_id, enrollment_window_start, enrollment_window_end);
```

### 2.8 Open Enrollment Periods
```sql
CREATE TABLE open_enrollment_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    period_start_date TEXT NOT NULL,
    period_end_date TEXT NOT NULL,
    coverage_effective_date TEXT NOT NULL,
    plan_ids TEXT,  -- JSON array of plan IDs included
    target_population TEXT,  -- JSON: eligibility criteria
    status TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK(status IN ('SCHEDULED','ACTIVE','CLOSED','CANCELLED')),
    auto_enroll_flag INTEGER DEFAULT 0,
    auto_enroll_plan_id TEXT,
    communication_template_id TEXT,
    total_eligible INTEGER DEFAULT 0,
    total_enrolled INTEGER DEFAULT 0,
    total_waived INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_name)
);

CREATE INDEX idx_open_enrollment_tenant_dates ON open_enrollment_periods(tenant_id, period_start_date, period_end_date);
CREATE INDEX idx_open_enrollment_tenant_status ON open_enrollment_periods(tenant_id, status);
```

### 2.9 Benefit Cost Calculations
```sql
CREATE TABLE benefit_cost_calculations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    calculation_date TEXT NOT NULL,
    calculation_type TEXT NOT NULL CHECK(calculation_type IN ('MONTHLY','ANNUAL','PAY_PERIOD','LIFETIME')),
    total_employee_cost INTEGER NOT NULL DEFAULT 0,  -- cents
    total_employer_cost INTEGER NOT NULL DEFAULT 0,
    total_combined_cost INTEGER NOT NULL DEFAULT 0,
    pre_tax_deductions INTEGER NOT NULL DEFAULT 0,
    post_tax_deductions INTEGER NOT NULL DEFAULT 0,
    employer_tax_savings INTEGER NOT NULL DEFAULT 0,
    plan_count INTEGER DEFAULT 0,
    calculation_details TEXT,  -- JSON breakdown by plan

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_id, calculation_date, calculation_type)
);

CREATE INDEX idx_benefit_costs_tenant_employee ON benefit_cost_calculations(tenant_id, employee_id);
```

### 2.10 Benefit Provider Integrations
```sql
CREATE TABLE benefit_provider_integrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    provider_name TEXT NOT NULL,
    provider_code TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    integration_type TEXT NOT NULL CHECK(integration_type IN ('EDI_834','API','FILE_EXPORT','FILE_IMPORT','SFTP')),
    endpoint_url TEXT,
    authentication_type TEXT CHECK(authentication_type IN ('API_KEY','OAUTH2','BASIC','CERTIFICATE')),
    credentials_reference TEXT,
    file_format TEXT DEFAULT 'CSV' CHECK(file_format IN ('CSV','XML','JSON','EDI','FIXED_WIDTH')),
    transmission_frequency TEXT DEFAULT 'DAILY' CHECK(transmission_frequency IN ('REAL_TIME','DAILY','WEEKLY','MONTHLY','ON_DEMAND')),
    last_transmission_date TEXT,
    last_transmission_status TEXT CHECK(last_transmission_status IN ('SUCCESS','PARTIAL','FAILURE','UNKNOWN')),
    last_transmission_record_count INTEGER,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','ERROR')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, provider_code)
);

CREATE INDEX idx_provider_integrations_tenant ON benefit_provider_integrations(tenant_id, plan_id);
```

---

## 3. REST API Endpoints

### Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans` | Create a benefit plan |
| GET | `/api/v1/plans` | List benefit plans with filters |
| GET | `/api/v1/plans/{id}` | Get plan details with options and rates |
| PUT | `/api/v1/plans/{id}` | Update benefit plan |

### Plan Options & Rates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans/{planId}/options` | Add option to plan |
| GET | `/api/v1/plans/{planId}/options` | List plan options |
| PUT | `/api/v1/options/{id}` | Update plan option |
| POST | `/api/v1/plans/{planId}/rates` | Set plan rate |
| GET | `/api/v1/plans/{planId}/rates` | List plan rates |
| PUT | `/api/v1/rates/{id}` | Update plan rate |

### Eligibility
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans/{planId}/eligibility-rules` | Add eligibility rule |
| GET | `/api/v1/plans/{planId}/eligibility-rules` | List eligibility rules |
| GET | `/api/v1/employees/{employeeId}/eligible-plans` | Check employee eligibility |

### Enrollment
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/enrollments` | Enroll employee in benefit |
| GET | `/api/v1/enrollments` | List enrollments with filters |
| GET | `/api/v1/enrollments/{id}` | Get enrollment details |
| PUT | `/api/v1/enrollments/{id}` | Update enrollment |
| POST | `/api/v1/enrollments/{id}/cancel` | Cancel enrollment |
| POST | `/api/v1/enrollments/{id}/waive` | Waive benefit |
| GET | `/api/v1/employees/{employeeId}/enrollments` | List employee enrollments |

### Beneficiaries
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/enrollments/{enrollmentId}/beneficiaries` | Add beneficiary |
| GET | `/api/v1/enrollments/{enrollmentId}/beneficiaries` | List beneficiaries |
| PUT | `/api/v1/beneficiaries/{id}` | Update beneficiary |
| DELETE | `/api/v1/beneficiaries/{id}` | Remove beneficiary |

### Life Events
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/life-events` | Report a life event |
| GET | `/api/v1/life-events` | List life events |
| GET | `/api/v1/life-events/{id}` | Get life event details |
| POST | `/api/v1/life-events/{id}/verify` | Verify a life event |
| GET | `/api/v1/employees/{employeeId}/life-events` | List employee life events |

### Open Enrollment
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/open-enrollment` | Create open enrollment period |
| GET | `/api/v1/open-enrollment` | List open enrollment periods |
| GET | `/api/v1/open-enrollment/{id}` | Get enrollment period details |
| PATCH | `/api/v1/open-enrollment/{id}/activate` | Activate open enrollment |
| PATCH | `/api/v1/open-enrollment/{id}/close` | Close open enrollment |
| GET | `/api/v1/open-enrollment/{id}/summary` | Get enrollment summary |

### Cost Calculations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/cost-calculations` | Calculate benefit costs |
| GET | `/api/v1/employees/{employeeId}/costs` | Get employee benefit costs |
| GET | `/api/v1/cost-summary` | Get organization benefit cost summary |

### Provider Integration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/provider-integrations` | Configure provider integration |
| GET | `/api/v1/provider-integrations` | List provider integrations |
| POST | `/api/v1/provider-integrations/{id}/transmit` | Transmit enrollment data to provider |
| GET | `/api/v1/provider-integrations/{id}/status` | Get last transmission status |

---

## 4. Business Rules

1. An employee MUST be eligible for a plan based on configured eligibility rules before enrollment.
2. Beneficiary allocation percentages for a single enrollment MUST sum to 100% for primary beneficiaries.
3. Open enrollment period dates MUST NOT overlap for the same target population.
4. Life event enrollment window MUST default to 30 days from the event date unless otherwise configured.
5. An employee MUST NOT be enrolled in the same plan with overlapping coverage periods.
6. Plan rates MUST be configured for all supported coverage levels before enrollment is allowed.
7. Waived enrollments MUST record a waive reason for audit purposes.
8. Auto-enrollment MUST only apply during open enrollment periods where the flag is set.
9. Provider integration transmissions MUST log record count and status for each transmission.
10. Employer contribution rates MUST NOT exceed the total cost of the plan option.
11. Waiting period rules MUST be validated against the employee hire date before enrollment activation.
12. The system SHOULD support imputed income calculation for domestic partner coverage.
13. Dependent age-out rules MUST be evaluated at the start of each plan year.
14. Enrollment changes during open enrollment MUST be allowed multiple times until the period closes.
15. Cost calculations MUST distinguish between pre-tax and post-tax deductions.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package benefits.v1;

service BenefitsService {
    // Plans
    rpc CreatePlan(CreatePlanRequest) returns (CreatePlanResponse);
    rpc GetPlan(GetPlanRequest) returns (GetPlanResponse);
    rpc ListPlans(ListPlansRequest) returns (ListPlansResponse);

    // Eligibility
    rpc CheckEligibility(CheckEligibilityRequest) returns (CheckEligibilityResponse);
    rpc SetEligibilityRule(SetEligibilityRuleRequest) returns (SetEligibilityRuleResponse);

    // Enrollment
    rpc EnrollEmployee(EnrollEmployeeRequest) returns (EnrollEmployeeResponse);
    rpc GetEnrollment(GetEnrollmentRequest) returns (GetEnrollmentResponse);
    rpc ListEmployeeEnrollments(ListEmployeeEnrollmentsRequest) returns (ListEmployeeEnrollmentsResponse);
    rpc CancelEnrollment(CancelEnrollmentRequest) returns (CancelEnrollmentResponse);
    rpc WaiveBenefit(WaiveBenefitRequest) returns (WaiveBenefitResponse);

    // Beneficiaries
    rpc AddBeneficiary(AddBeneficiaryRequest) returns (AddBeneficiaryResponse);
    rpc ListBeneficiaries(ListBeneficiariesRequest) returns (ListBeneficiariesResponse);

    // Life events
    rpc ReportLifeEvent(ReportLifeEventRequest) returns (ReportLifeEventResponse);
    rpc VerifyLifeEvent(VerifyLifeEventRequest) returns (VerifyLifeEventResponse);

    // Open enrollment
    rpc CreateOpenEnrollment(CreateOpenEnrollmentRequest) returns (CreateOpenEnrollmentResponse);
    rpc GetOpenEnrollment(GetOpenEnrollmentRequest) returns (GetOpenEnrollmentResponse);

    // Costs
    rpc CalculateBenefitCosts(CalculateBenefitCostsRequest) returns (CalculateBenefitCostsResponse);

    // Provider integration
    rpc TransmitToProvider(TransmitToProviderRequest) returns (TransmitToProviderResponse);
}

message BenefitPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_code = 4;
    string plan_type = 5;
    string status = 6;
}

message CreatePlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    string plan_code = 3;
    string plan_type = 4;
    string plan_category = 5;
    string carrier_name = 6;
    string effective_from = 7;
    string created_by = 8;
}

message CreatePlanResponse {
    BenefitPlan plan = 1;
}

message GetPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetPlanResponse {
    BenefitPlan plan = 1;
    repeated PlanOption options = 2;
}

message ListPlansRequest {
    string tenant_id = 1;
    string plan_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPlansResponse {
    repeated BenefitPlan plans = 1;
    string next_page_token = 2;
}

message PlanOption {
    string id = 1;
    string option_name = 2;
    string coverage_level = 3;
    int64 deductable_amount = 4;
    int64 out_of_pocket_max = 5;
}

message EligiblePlan {
    string plan_id = 1;
    string plan_name = 2;
    string plan_type = 3;
    bool is_eligible = 4;
    string reason = 5;
}

message CheckEligibilityRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
}

message CheckEligibilityResponse {
    bool is_eligible = 1;
    repeated string reasons = 2;
}

message SetEligibilityRuleRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string condition_expression = 5;
    string created_by = 6;
}

message SetEligibilityRuleResponse {
    string rule_id = 1;
}

message Enrollment {
    string id = 1;
    string employee_id = 2;
    string plan_id = 3;
    string option_id = 4;
    string coverage_level = 5;
    string enrollment_status = 6;
    string effective_start_date = 7;
    int64 employee_cost_per_period = 8;
    int64 employer_cost_per_period = 9;
}

message EnrollEmployeeRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
    string option_id = 4;
    string coverage_level = 5;
    string enrollment_type = 6;
    string effective_start_date = 7;
    string created_by = 8;
}

message EnrollEmployeeResponse {
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
}

message ListEmployeeEnrollmentsResponse {
    repeated Enrollment enrollments = 1;
}

message CancelEnrollmentRequest {
    string tenant_id = 1;
    string enrollment_id = 2;
    string reason = 3;
    string cancelled_by = 4;
}

message CancelEnrollmentResponse {
    Enrollment enrollment = 1;
}

message WaiveBenefitRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
    string waive_reason = 4;
    string effective_date = 5;
    string created_by = 6;
}

message WaiveBenefitResponse {
    Enrollment enrollment = 1;
}

message Beneficiary {
    string id = 1;
    string beneficiary_name = 2;
    string relationship = 3;
    double allocation_percentage = 4;
    bool is_primary = 5;
}

message AddBeneficiaryRequest {
    string tenant_id = 1;
    string enrollment_id = 2;
    string beneficiary_name = 3;
    string relationship = 4;
    double allocation_percentage = 5;
    string created_by = 6;
}

message AddBeneficiaryResponse {
    Beneficiary beneficiary = 1;
}

message ListBeneficiariesRequest {
    string tenant_id = 1;
    string enrollment_id = 2;
}

message ListBeneficiariesResponse {
    repeated Beneficiary beneficiaries = 1;
}

message ReportLifeEventRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string event_type = 3;
    string event_date = 4;
    string description = 5;
    string created_by = 6;
}

message ReportLifeEventResponse {
    string life_event_id = 1;
    string enrollment_window_start = 2;
    string enrollment_window_end = 3;
}

message VerifyLifeEventRequest {
    string tenant_id = 1;
    string life_event_id = 2;
    string verified_by = 3;
    bool approved = 4;
}

message VerifyLifeEventResponse {
    string status = 1;
}

message OpenEnrollmentPeriod {
    string id = 1;
    string period_name = 2;
    string period_start_date = 3;
    string period_end_date = 4;
    string coverage_effective_date = 5;
    string status = 6;
}

message CreateOpenEnrollmentRequest {
    string tenant_id = 1;
    string period_name = 2;
    string period_start_date = 3;
    string period_end_date = 4;
    string coverage_effective_date = 5;
    string created_by = 6;
}

message CreateOpenEnrollmentResponse {
    OpenEnrollmentPeriod period = 1;
}

message GetOpenEnrollmentRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetOpenEnrollmentResponse {
    OpenEnrollmentPeriod period = 1;
    int32 total_eligible = 2;
    int32 total_enrolled = 3;
    int32 total_waived = 4;
}

message CalculateBenefitCostsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string calculation_type = 3;
}

message CalculateBenefitCostsResponse {
    int64 total_employee_cost = 1;
    int64 total_employer_cost = 2;
    int64 total_combined_cost = 3;
    int64 pre_tax_deductions = 4;
    int64 post_tax_deductions = 5;
}

message TransmitToProviderRequest {
    string tenant_id = 1;
    string integration_id = 2;
    repeated string enrollment_ids = 3;
}

message TransmitToProviderResponse {
    bool success = 1;
    int32 record_count = 2;
    string transmission_status = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee demographics, assignments, status | Determine eligibility and population |
| `payroll-service` | Deduction elements, pay periods | Set up benefit payroll deductions |
| `compensation-service` | Salary data | Calculate benefit cost tiers |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `payroll-service` | Deduction amounts, enrollment changes | Process benefit payroll deductions |
| `compensation-service` | Employer contribution costs | Total compensation statements |
| `reporting-service` | Enrollment analytics | Benefits participation reports |
| `hr-service` | Life event status | Update employee records |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `EnrollmentCompleted` | `benefits.enrollment.completed` | `{ tenant_id, enrollment_id, employee_id, plan_id, option_id, coverage_level, effective_start_date }` | Published when enrollment is finalized |
| `LifeEventTriggered` | `benefits.life_event.triggered` | `{ tenant_id, life_event_id, employee_id, event_type, event_date, enrollment_window_end }` | Published when a life event is reported |
| `OpenEnrollmentStarted` | `benefits.open_enrollment.started` | `{ tenant_id, period_id, period_name, period_start_date, period_end_date, total_eligible }` | Published when open enrollment period opens |
| `BenefitCostChanged` | `benefits.cost.changed` | `{ tenant_id, plan_id, old_employee_cost, new_employee_cost, old_employer_cost, new_employer_cost }` | Published when plan rates are updated |
| `EnrollmentCancelled` | `benefits.enrollment.cancelled` | `{ tenant_id, enrollment_id, employee_id, plan_id, reason }` | Published when enrollment is cancelled |
| `ProviderTransmissionCompleted` | `benefits.provider.transmitted` | `{ tenant_id, integration_id, record_count, status }` | Published after provider data transmission |
| `BenefitWaived` | `benefits.enrollment.waived` | `{ tenant_id, employee_id, plan_id, waive_reason }` | Published when employee waives a benefit |
