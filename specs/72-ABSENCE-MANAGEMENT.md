# 72 - Absence Management Service Specification

## 1. Domain Overview

The Absence Management service handles employee leave and absence tracking including absence plan definition, accrual rule configuration, absence request processing, approval workflows, balance management, holiday schedule administration, team calendar visibility, and leave liability calculations. It integrates with payroll for deductions and time/labor for schedule enforcement.

**Bounded Context:** Absence & Leave Administration
**Service Name:** `absence-service`
**Database:** `data/absence.db`
**HTTP Port:** 8104 | **gRPC Port:** 9104

---

## 2. Database Schema

### 2.1 Absence Plans
```sql
CREATE TABLE absence_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('VACATION','SICK','PERSONAL','MATERNITY','PATERNITY','BEREAVEMENT','JURY_DUTY','MILITARY','SABBATICAL','UNPAID','COMPASSIONATE','MEDICAL','STUDY','OTHER')),
    plan_category TEXT NOT NULL CHECK(plan_category IN ('PAID','UNPAID','PARTIALLY_PAID')),
    uom TEXT NOT NULL DEFAULT 'DAYS' CHECK(uom IN ('DAYS','HOURS','HALF_DAYS')),
    eligible_population TEXT,  -- JSON: eligibility criteria
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    carry_over_rule TEXT DEFAULT 'NONE' CHECK(carry_over_rule IN ('NONE','UNLIMITED','CAPPED','USE_IT_OR_LOSE_IT')),
    carry_over_max DECIMAL(10,2),
    carry_over_expiry_months INTEGER,
    advance_availability INTEGER DEFAULT 0,
    advance_limit DECIMAL(10,2),
    negative_balance_allowed INTEGER DEFAULT 0,
    negative_balance_limit DECIMAL(10,2),
    waiting_period_days INTEGER DEFAULT 0,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_absence_plans_tenant_type ON absence_plans(tenant_id, plan_type);
CREATE INDEX idx_absence_plans_tenant_status ON absence_plans(tenant_id, status);
```

### 2.2 Absence Plan Rules
```sql
CREATE TABLE absence_plan_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('ACCRUAL','PRORATION','ELIGIBILITY','ROUNDING','WAITING_PERIOD','CERTIFICATION','CONSECUTIVE_LIMIT','ANNUAL_LIMIT','CARRY_FORWARD')),
    accrual_frequency TEXT CHECK(accrual_frequency IN ('DAILY','WEEKLY','BIWEEKLY','SEMI_MONTHLY','MONTHLY','QUARTERLY','SEMI_ANNUALLY','ANNUALLY','ON_HIRE_ANNIVERSARY')),
    accrual_rate DECIMAL(10,4),
    accrual_basis TEXT DEFAULT 'FLAT' CHECK(accrual_basis IN ('FLAT','PER_HOUR_WORKED','PER_PAY_PERIOD','GRADE_BASED','LENGTH_OF_SERVICE')),
    service_based_rates TEXT,  -- JSON: { years_of_service: rate }
    grade_based_rates TEXT,    -- JSON: { grade_id: rate }
    max_accrual DECIMAL(10,2),
    max_balance DECIMAL(10,2),
    proration_rule TEXT DEFAULT 'CALENDAR_DAYS' CHECK(proration_rule IN ('CALENDAR_DAYS','WORKING_DAYS','NONE')),
    rounding_rule TEXT DEFAULT 'NO_ROUNDING' CHECK(rounding_rule IN ('NO_ROUNDING','UP_HALF','DOWN_HALF','NEAREST_HALF','UP_WHOLE','DOWN_WHOLE')),
    consecutive_day_limit INTEGER,
    annual_usage_limit DECIMAL(10,2),
    certification_required_after_days INTEGER,
    description TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES absence_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_absence_rules_tenant_plan ON absence_plan_rules(tenant_id, plan_id);
CREATE INDEX idx_absence_rules_tenant_type ON absence_plan_rules(tenant_id, rule_type);
```

### 2.3 Absence Entitlements
```sql
CREATE TABLE absence_entitlements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    entitlement_year INTEGER NOT NULL,
    starting_balance DECIMAL(10,2) DEFAULT 0,
    accrued DECIMAL(10,2) DEFAULT 0,
    carried_over DECIMAL(10,2) DEFAULT 0,
    advanced DECIMAL(10,2) DEFAULT 0,
    adjusted DECIMAL(10,2) DEFAULT 0,
    total_entitlement DECIMAL(10,2) DEFAULT 0,
    used DECIMAL(10,2) DEFAULT 0,
    pending DECIMAL(10,2) DEFAULT 0,
    remaining DECIMAL(10,2) DEFAULT 0,
    carry_over_to_next_year DECIMAL(10,2) DEFAULT 0,
    last_accrual_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES absence_plans(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, employee_id, plan_id, entitlement_year)
);

CREATE INDEX idx_entitlements_tenant_employee ON absence_entitlements(tenant_id, employee_id, entitlement_year);
CREATE INDEX idx_entitlements_tenant_plan ON absence_entitlements(tenant_id, plan_id, entitlement_year);
```

### 2.4 Absence Requests
```sql
CREATE TABLE absence_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    request_number TEXT NOT NULL,
    absence_type_id TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    start_time TEXT,
    end_time TEXT,
    duration_days DECIMAL(10,2) NOT NULL,
    duration_hours DECIMAL(10,2),
    reason TEXT,
    comments TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','APPROVED','REJECTED','CANCELLED','WITHDRAWN','MODIFIED')),
    submitted_at TEXT,
    approved_by TEXT,
    approved_at TEXT,
    rejected_reason TEXT,
    contact_during_absence TEXT,
    handover_notes TEXT,
    certification_provided INTEGER DEFAULT 0,
    certification_due_date TEXT,
    source TEXT DEFAULT 'SELF_SERVICE' CHECK(source IN ('SELF_SERVICE','MANAGER','HR','SYSTEM','IMPORT')),
    modified_from_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, request_number)
);

CREATE INDEX idx_absence_requests_tenant_employee ON absence_requests(tenant_id, employee_id, start_date);
CREATE INDEX idx_absence_requests_tenant_status ON absence_requests(tenant_id, status);
CREATE INDEX idx_absence_requests_tenant_dates ON absence_requests(tenant_id, start_date, end_date);
CREATE INDEX idx_absence_requests_tenant_plan ON absence_requests(tenant_id, plan_id);
```

### 2.5 Absence Approvals
```sql
CREATE TABLE absence_approvals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_id TEXT NOT NULL,
    approver_id TEXT NOT NULL,
    approval_level INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','APPROVED','REJECTED','DELEGATED')),
    comments TEXT,
    approved_at TEXT,
    delegated_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (request_id) REFERENCES absence_requests(id) ON DELETE CASCADE
);

CREATE INDEX idx_absence_approvals_tenant_request ON absence_approvals(tenant_id, request_id);
CREATE INDEX idx_absence_approvals_tenant_approver ON absence_approvals(tenant_id, approver_id, status);
```

### 2.6 Absence Balances
```sql
CREATE TABLE absence_balances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    balance_date TEXT NOT NULL,
    opening_balance DECIMAL(10,2) NOT NULL DEFAULT 0,
    accrued DECIMAL(10,2) DEFAULT 0,
    used DECIMAL(10,2) DEFAULT 0,
    adjusted DECIMAL(10,2) DEFAULT 0,
    closing_balance DECIMAL(10,2) NOT NULL DEFAULT 0,
    pending_deduction DECIMAL(10,2) DEFAULT 0,
    available_balance DECIMAL(10,2) NOT NULL DEFAULT 0,
    balance_type TEXT DEFAULT 'CURRENT' CHECK(balance_type IN ('CURRENT','PROJECTED','SNAPSHOT')),
    calculation_source TEXT DEFAULT 'SYSTEM' CHECK(calculation_source IN ('SYSTEM','MANUAL','IMPORT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_id, plan_id, balance_date, balance_type)
);

CREATE INDEX idx_absence_balances_tenant_employee ON absence_balances(tenant_id, employee_id, plan_id);
CREATE INDEX idx_absence_balances_tenant_date ON absence_balances(tenant_id, balance_date);
```

### 2.7 Absence Calendars
```sql
CREATE TABLE absence_calendars (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    calendar_name TEXT NOT NULL,
    calendar_code TEXT NOT NULL,
    calendar_type TEXT NOT NULL DEFAULT 'TEAM' CHECK(calendar_type IN ('TEAM','DEPARTMENT','ORGANIZATION','LOCATION','CUSTOM')),
    owner_id TEXT,
    department_id TEXT,
    location_id TEXT,
    viewer_ids TEXT,  -- JSON array of employee IDs with access
    is_default INTEGER DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, calendar_code)
);

CREATE INDEX idx_absence_calendars_tenant_type ON absence_calendars(tenant_id, calendar_type);
```

### 2.8 Holiday Schedules
```sql
CREATE TABLE holiday_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    schedule_code TEXT NOT NULL,
    year INTEGER NOT NULL,
    country TEXT NOT NULL,
    region TEXT,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, schedule_code, year)
);

CREATE INDEX idx_holiday_schedules_tenant_year ON holiday_schedules(tenant_id, year);
CREATE INDEX idx_holiday_schedules_tenant_country ON holiday_schedules(tenant_id, country);
```

### 2.9 Leave Liability Calculations
```sql
CREATE TABLE leave_liability_calculations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    calculation_date TEXT NOT NULL,
    calculation_type TEXT NOT NULL DEFAULT 'QUARTERLY' CHECK(calculation_type IN ('MONTHLY','QUARTERLY','ANNUAL','ON_DEMAND')),
    employee_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    accrued_balance DECIMAL(10,2) NOT NULL,
    daily_rate INTEGER NOT NULL,          -- cents per day
    total_liability INTEGER NOT NULL,     -- cents
    currency_code TEXT DEFAULT 'USD',
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    department_id TEXT,
    business_unit_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_id, plan_id, calculation_date, calculation_type)
);

CREATE INDEX idx_liability_tenant_date ON leave_liability_calculations(tenant_id, calculation_date);
CREATE INDEX idx_liability_tenant_employee ON leave_liability_calculations(tenant_id, employee_id);
CREATE INDEX idx_liability_tenant_dept ON leave_liability_calculations(tenant_id, department_id);
```

---

## 3. REST API Endpoints

### Absence Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans` | Create absence plan |
| GET | `/api/v1/plans` | List absence plans |
| GET | `/api/v1/plans/{id}` | Get plan details |
| PUT | `/api/v1/plans/{id}` | Update plan |

### Accrual Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans/{planId}/rules` | Add accrual/rule to plan |
| GET | `/api/v1/plans/{planId}/rules` | List plan rules |
| PUT | `/api/v1/rules/{id}` | Update rule |
| POST | `/api/v1/plans/{planId}/simulate-accrual` | Simulate accrual calculation |

### Absence Requests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/absence-requests` | Submit absence request |
| GET | `/api/v1/absence-requests` | List absence requests |
| GET | `/api/v1/absence-requests/{id}` | Get request details |
| PUT | `/api/v1/absence-requests/{id}` | Update request |
| POST | `/api/v1/absence-requests/{id}/withdraw` | Withdraw request |
| POST | `/api/v1/absence-requests/{id}/modify` | Modify approved request |
| GET | `/api/v1/employees/{employeeId}/absence-requests` | Get employee requests |

### Approvals
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/approvals/pending` | List pending approvals |
| POST | `/api/v1/absence-requests/{id}/approve` | Approve absence request |
| POST | `/api/v1/absence-requests/{id}/reject` | Reject absence request |
| POST | `/api/v1/approvals/bulk-approve` | Bulk approve requests |

### Balances
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/employees/{employeeId}/balances` | Get employee absence balances |
| GET | `/api/v1/employees/{employeeId}/balances/{planId}` | Get plan-specific balance |
| GET | `/api/v1/employees/{employeeId}/projected-balances` | Get projected balances |
| POST | `/api/v1/balances/adjust` | Manually adjust balance |

### Team Calendar
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/team-calendar` | Get team absence calendar |
| GET | `/api/v1/team-calendar/{departmentId}` | Get department calendar |
| GET | `/api/v1/absence-calendars` | List absence calendars |
| POST | `/api/v1/absence-calendars` | Create custom calendar |
| GET | `/api/v1/absence-calendars/{id}/view` | View calendar absences |

### Holiday Schedules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/holiday-schedules` | Create holiday schedule |
| GET | `/api/v1/holiday-schedules` | List holiday schedules |
| GET | `/api/v1/holiday-schedules/{id}` | Get schedule with holidays |
| PUT | `/api/v1/holiday-schedules/{id}` | Update schedule |
| POST | `/api/v1/holiday-schedules/{id}/holidays` | Add holiday to schedule |
| GET | `/api/v1/holidays` | List holidays for date range |

### Leave Liability
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/leave-liability/calculate` | Calculate leave liability |
| GET | `/api/v1/leave-liability` | Get liability calculations |
| GET | `/api/v1/leave-liability/summary` | Get liability summary |
| GET | `/api/v1/leave-liability/by-department` | Get liability by department |

---

## 4. Business Rules

1. An absence request MUST NOT have a start date in the past unless configured to allow retroactive requests.
2. Accrual balances MUST be updated on the configured frequency schedule.
3. An employee MUST NOT submit an absence request that exceeds their available balance unless negative balances are allowed.
4. Carry-over rules MUST be applied at the end of each entitlement year.
5. Holiday dates MUST NOT be counted as absence days.
6. An absence request MUST be approved before the balance is deducted.
7. Certification requirements MUST be enforced for absences exceeding the configured day threshold.
8. Accrual proration MUST be applied for mid-year hires based on the configured rule.
9. Consecutive day limits on plans MUST be validated at submission time.
10. The system SHOULD auto-detect conflicts between approved absences and critical work schedules.
11. Leave liability MUST be calculated using the employee's current daily rate and accrued balance.
12. Waiting period rules MUST prevent usage before the eligibility date.
13. Withdrawn requests MUST restore the deducted balance immediately.
14. Absence calendars MUST show only employees visible to the requesting user.
15. Annual usage limits MUST be enforced across all approved and pending requests.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package absence.v1;

service AbsenceService {
    // Plans
    rpc CreatePlan(CreatePlanRequest) returns (CreatePlanResponse);
    rpc GetPlan(GetPlanRequest) returns (GetPlanResponse);
    rpc ListPlans(ListPlansRequest) returns (ListPlansResponse);

    // Rules
    rpc CreatePlanRule(CreatePlanRuleRequest) returns (CreatePlanRuleResponse);
    rpc ListPlanRules(ListPlanRulesRequest) returns (ListPlanRulesResponse);

    // Requests
    rpc SubmitAbsenceRequest(SubmitAbsenceRequestRequest) returns (SubmitAbsenceRequestResponse);
    rpc GetAbsenceRequest(GetAbsenceRequestRequest) returns (GetAbsenceRequestResponse);
    rpc ListAbsenceRequests(ListAbsenceRequestsRequest) returns (ListAbsenceRequestsResponse);
    rpc WithdrawAbsenceRequest(WithdrawAbsenceRequestRequest) returns (WithdrawAbsenceRequestResponse);

    // Approvals
    rpc ApproveAbsence(ApproveAbsenceRequest) returns (ApproveAbsenceResponse);
    rpc RejectAbsence(RejectAbsenceRequest) returns (RejectAbsenceResponse);
    rpc GetPendingApprovals(GetPendingApprovalsRequest) returns (GetPendingApprovalsResponse);

    // Balances
    rpc GetEmployeeBalances(GetEmployeeBalancesRequest) returns (GetEmployeeBalancesResponse);
    rpc GetProjectedBalance(GetProjectedBalanceRequest) returns (GetProjectedBalanceResponse);
    rpc AdjustBalance(AdjustBalanceRequest) returns (AdjustBalanceResponse);

    // Calendar
    rpc GetTeamCalendar(GetTeamCalendarRequest) returns (GetTeamCalendarResponse);

    // Holidays
    rpc GetHolidays(GetHolidaysRequest) returns (GetHolidaysResponse);

    // Liability
    rpc CalculateLiability(CalculateLiabilityRequest) returns (CalculateLiabilityResponse);
    rpc GetLiabilitySummary(GetLiabilitySummaryRequest) returns (GetLiabilitySummaryResponse);
}

message AbsencePlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_code = 4;
    string plan_type = 5;
    string plan_category = 6;
    string uom = 7;
    string status = 8;
}

message CreatePlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    string plan_code = 3;
    string plan_type = 4;
    string plan_category = 5;
    string uom = 6;
    string effective_from = 7;
    string created_by = 8;
}

message CreatePlanResponse {
    AbsencePlan plan = 1;
}

message GetPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetPlanResponse {
    AbsencePlan plan = 1;
}

message ListPlansRequest {
    string tenant_id = 1;
    string plan_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPlansResponse {
    repeated AbsencePlan plans = 1;
    string next_page_token = 2;
}

message PlanRule {
    string id = 1;
    string plan_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string accrual_frequency = 5;
    double accrual_rate = 6;
    string accrual_basis = 7;
}

message CreatePlanRuleRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string accrual_frequency = 5;
    double accrual_rate = 6;
    string accrual_basis = 7;
    string created_by = 8;
}

message CreatePlanRuleResponse {
    PlanRule rule = 1;
}

message ListPlanRulesRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string rule_type = 3;
}

message ListPlanRulesResponse {
    repeated PlanRule rules = 1;
}

message AbsenceRequestInfo {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string plan_id = 4;
    string request_number = 5;
    string start_date = 6;
    string end_date = 7;
    double duration_days = 8;
    string status = 9;
    string reason = 10;
}

message SubmitAbsenceRequestRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
    string start_date = 4;
    string end_date = 5;
    string start_time = 6;
    string end_time = 7;
    double duration_days = 8;
    string reason = 9;
    string created_by = 10;
}

message SubmitAbsenceRequestResponse {
    AbsenceRequestInfo request = 1;
}

message GetAbsenceRequestRequest {
    string tenant_id = 1;
    string request_id = 2;
}

message GetAbsenceRequestResponse {
    AbsenceRequestInfo request = 1;
}

message ListAbsenceRequestsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string status = 3;
    string date_from = 4;
    string date_to = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListAbsenceRequestsResponse {
    repeated AbsenceRequestInfo requests = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message WithdrawAbsenceRequestRequest {
    string tenant_id = 1;
    string request_id = 2;
    string withdrawn_by = 3;
}

message WithdrawAbsenceRequestResponse {
    AbsenceRequestInfo request = 1;
}

message ApproveAbsenceRequest {
    string tenant_id = 1;
    string request_id = 2;
    string approved_by = 3;
    string comments = 4;
}

message ApproveAbsenceResponse {
    AbsenceRequestInfo request = 1;
}

message RejectAbsenceRequest {
    string tenant_id = 1;
    string request_id = 2;
    string rejected_by = 3;
    string reason = 4;
}

message RejectAbsenceResponse {
    AbsenceRequestInfo request = 1;
}

message GetPendingApprovalsRequest {
    string tenant_id = 1;
    string approver_id = 2;
}

message GetPendingApprovalsResponse {
    repeated AbsenceRequestInfo pending_requests = 1;
}

message BalanceEntry {
    string plan_id = 1;
    string plan_name = 2;
    string plan_type = 3;
    double total_entitlement = 4;
    double used = 5;
    double pending = 6;
    double remaining = 7;
    double available = 8;
    double carried_over = 9;
}

message GetEmployeeBalancesRequest {
    string tenant_id = 1;
    string employee_id = 2;
    int32 year = 3;
}

message GetEmployeeBalancesResponse {
    repeated BalanceEntry balances = 1;
    int32 year = 2;
}

message ProjectedBalance {
    string plan_id = 1;
    string plan_name = 2;
    double current_balance = 3;
    double projected_accrual = 4;
    double projected_balance = 5;
    string projection_date = 6;
}

message GetProjectedBalanceRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
    string projection_date = 4;
}

message GetProjectedBalanceResponse {
    repeated ProjectedBalance projections = 1;
}

message AdjustBalanceRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_id = 3;
    double adjustment_amount = 4;
    string reason = 5;
    string adjusted_by = 6;
}

message AdjustBalanceResponse {
    string entitlement_id = 1;
    double new_balance = 2;
}

message CalendarAbsenceEntry {
    string employee_id = 1;
    string employee_name = 2;
    string plan_type = 3;
    string start_date = 4;
    string end_date = 5;
    double duration_days = 6;
    string status = 7;
}

message GetTeamCalendarRequest {
    string tenant_id = 1;
    string department_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetTeamCalendarResponse {
    repeated CalendarAbsenceEntry absences = 1;
    repeated string holidays = 2;
}

message Holiday {
    string date = 1;
    string name = 2;
    string type = 3;
}

message GetHolidaysRequest {
    string tenant_id = 1;
    string schedule_id = 2;
    int32 year = 3;
}

message GetHolidaysResponse {
    repeated Holiday holidays = 1;
    int32 total_count = 2;
}

message LiabilityEntry {
    string employee_id = 1;
    string plan_id = 2;
    double accrued_balance = 3;
    int64 daily_rate = 4;
    int64 total_liability = 5;
    string currency_code = 6;
}

message CalculateLiabilityRequest {
    string tenant_id = 1;
    string calculation_type = 2;
    string calculation_date = 3;
    string department_id = 4;
    string calculated_by = 5;
}

message CalculateLiabilityResponse {
    repeated LiabilityEntry entries = 1;
    int64 total_liability = 2;
    int32 employee_count = 3;
}

message GetLiabilitySummaryRequest {
    string tenant_id = 1;
    string calculation_date = 2;
}

message GetLiabilitySummaryResponse {
    int64 total_liability = 1;
    int32 employee_count = 2;
    string currency_code = 3;
    repeated LiabilityEntry top_entries = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, status, org hierarchy | Determine eligible population |
| `payroll-service` | Salary data, pay periods | Calculate daily rates for liability |
| `timelabor-service` | Work schedules, time entries | Validate absence against schedule |
| `workflow-service` | Approval workflows | Route absence approvals |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `payroll-service` | Approved absences, deductions | Process absence payroll elements |
| `timelabor-service` | Approved absences | Populate absence time entries |
| `reporting-service` | Absence analytics | Absence trends and liability reports |
| `hr-service` | Absence status changes | Update employee availability |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `AbsenceRequested` | `absence.requested` | `{ tenant_id, request_id, employee_id, plan_id, start_date, end_date, duration_days }` | Published when an absence request is submitted |
| `AbsenceApproved` | `absence.approved` | `{ tenant_id, request_id, employee_id, plan_id, start_date, end_date, approved_by }` | Published when an absence request is approved |
| `AbsenceRejected` | `absence.rejected` | `{ tenant_id, request_id, employee_id, rejected_by, reason }` | Published when an absence request is rejected |
| `AbsenceBalanceUpdated` | `absence.balance.updated` | `{ tenant_id, employee_id, plan_id, previous_balance, new_balance, change_amount, change_type }` | Published when balance changes |
| `LiabilityCalculated` | `absence.liability.calculated` | `{ tenant_id, calculation_date, total_liability, employee_count, currency_code }` | Published when liability calculation completes |
| `AbsenceWithdrawn` | `absence.withdrawn` | `{ tenant_id, request_id, employee_id, plan_id, restored_days }` | Published when an approved absence is withdrawn |
| `AccrualProcessed` | `absence.accrual.processed` | `{ tenant_id, plan_id, employee_count, total_accrued }` | Published when accrual batch runs |
| `CarryOverProcessed` | `absence.carryover.processed` | `{ tenant_id, plan_id, year, employee_count, total_carried }` | Published when year-end carry-over runs |
