# 93 - Opportunity Marketplace Service Specification

## 1. Domain Overview

The Opportunity Marketplace service provides an internal talent marketplace for employees to discover and apply for internal gigs, projects, short-term assignments, stretch assignments, and mentorship opportunities. It supports skill-based matching, manager approvals, and workforce mobility tracking to help organizations maximize talent utilization and foster career growth.

**Bounded Context:** Internal Talent Marketplace & Workforce Mobility
**Service Name:** `marketplace-service`
**Database:** `data/marketplace.db`
**HTTP Port:** 8130 | **gRPC Port:** 9130

---

## 2. Database Schema

### 2.1 Internal Opportunities
```sql
CREATE TABLE internal_opportunities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    type TEXT NOT NULL CHECK(type IN ('PROJECT','GIG','STRETCH_ASSIGNMENT','MENTORSHIP','COMMITTEE','ACTING_ROLE')),
    business_unit_id TEXT,
    department_id TEXT,
    owner_id TEXT NOT NULL,
    skills_required TEXT,  -- JSON array of skill descriptors
    duration_weeks INTEGER,
    start_date TEXT,
    end_date TEXT,
    headcount INTEGER NOT NULL DEFAULT 1,
    positions_filled INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','PUBLISHED','FILLED','CLOSED','CANCELLED')),
    visibility TEXT NOT NULL DEFAULT 'ALL' CHECK(visibility IN ('ALL','DEPARTMENT','BUSINESS_UNIT','SELECTED')),
    compensation_override_cents INTEGER,
    priority TEXT DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    application_deadline TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_opportunities_tenant_status ON internal_opportunities(tenant_id, status);
CREATE INDEX idx_opportunities_tenant_dept ON internal_opportunities(tenant_id, department_id);
CREATE INDEX idx_opportunities_tenant_owner ON internal_opportunities(tenant_id, owner_id);
CREATE INDEX idx_opportunities_tenant_type ON internal_opportunities(tenant_id, type);
CREATE INDEX idx_opportunities_tenant_dates ON internal_opportunities(tenant_id, start_date, end_date);
```

### 2.2 Opportunity Applications
```sql
CREATE TABLE opportunity_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    opportunity_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    application_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'APPLIED' CHECK(status IN ('APPLIED','MANAGER_APPROVED','MANAGER_REJECTED','SELECTED','COMPLETED','WITHDRAWN')),
    cover_statement TEXT,
    skills_match_score DECIMAL(5,2),
    manager_approval_status TEXT DEFAULT 'PENDING' CHECK(manager_approval_status IN ('PENDING','APPROVED','REJECTED')),
    approved_by_manager_id TEXT,
    manager_comments TEXT,
    start_date TEXT,
    end_date TEXT,
    feedback_rating INTEGER CHECK(feedback_rating BETWEEN 1 AND 5),
    feedback_text TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (opportunity_id) REFERENCES internal_opportunities(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, opportunity_id, employee_id)
);

CREATE INDEX idx_opp_applications_tenant_opp ON opportunity_applications(tenant_id, opportunity_id);
CREATE INDEX idx_opp_applications_tenant_emp ON opportunity_applications(tenant_id, employee_id);
CREATE INDEX idx_opp_applications_tenant_status ON opportunity_applications(tenant_id, status);
CREATE INDEX idx_opp_applications_tenant_date ON opportunity_applications(tenant_id, application_date);
```

### 2.3 Opportunity Skills
```sql
CREATE TABLE opportunity_skills (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    opportunity_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    skill_name TEXT NOT NULL,
    importance TEXT NOT NULL DEFAULT 'REQUIRED' CHECK(importance IN ('REQUIRED','PREFERRED','NICE_TO_HAVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (opportunity_id) REFERENCES internal_opportunities(id) ON DELETE CASCADE
);

CREATE INDEX idx_opp_skills_tenant_opp ON opportunity_skills(tenant_id, opportunity_id);
CREATE INDEX idx_opp_skills_tenant_skill ON opportunity_skills(tenant_id, skill_id);
```

### 2.4 Employee Preferences
```sql
CREATE TABLE employee_preferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    preferred_types TEXT,  -- JSON array of opportunity types
    preferred_departments TEXT,  -- JSON array of department IDs
    preferred_locations TEXT,  -- JSON array of location IDs
    max_travel_percentage INTEGER DEFAULT 100,
    available_from TEXT,
    notification_enabled INTEGER DEFAULT 1,
    skills_interests TEXT,  -- JSON array of skill IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_id)
);

CREATE INDEX idx_emp_prefs_tenant_emp ON employee_preferences(tenant_id, employee_id);
```

### 2.5 Opportunity Match Scores
```sql
CREATE TABLE opportunity_match_scores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    opportunity_id TEXT NOT NULL,
    match_score DECIMAL(5,2) NOT NULL,
    skill_match DECIMAL(5,2),
    experience_match DECIMAL(5,2),
    interests_match DECIMAL(5,2),
    calculated_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (opportunity_id) REFERENCES internal_opportunities(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, employee_id, opportunity_id)
);

CREATE INDEX idx_match_scores_tenant_emp ON opportunity_match_scores(tenant_id, employee_id);
CREATE INDEX idx_opp_match_scores_tenant_opp ON opportunity_match_scores(tenant_id, opportunity_id);
CREATE INDEX idx_match_scores_tenant_score ON opportunity_match_scores(tenant_id, match_score DESC);
```

---

## 3. REST API Endpoints

### Opportunities
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/opportunities` | Create an internal opportunity |
| GET | `/api/v1/opportunities` | List opportunities with filters |
| GET | `/api/v1/opportunities/{id}` | Get opportunity details |
| PUT | `/api/v1/opportunities/{id}` | Update opportunity |
| POST | `/api/v1/opportunities/{id}/publish` | Publish opportunity to marketplace |
| POST | `/api/v1/opportunities/{id}/close` | Close opportunity |
| POST | `/api/v1/opportunities/{id}/cancel` | Cancel opportunity |

### Applications
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/opportunities/{id}/apply` | Apply for an opportunity |
| GET | `/api/v1/applications` | List applications with filters |
| GET | `/api/v1/applications/{id}` | Get application details |
| POST | `/api/v1/applications/{id}/withdraw` | Withdraw application |
| POST | `/api/v1/applications/{id}/approve` | Approve application (manager) |
| POST | `/api/v1/applications/{id}/reject` | Reject application (manager) |
| POST | `/api/v1/applications/{id}/select` | Select candidate for opportunity |
| POST | `/api/v1/applications/{id}/complete` | Mark assignment as completed |

### Preferences
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/employees/{id}/preferences` | Get employee preferences |
| PUT | `/api/v1/employees/{id}/preferences` | Update employee preferences |

### Matching & Recommendations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/employees/{id}/recommendations` | Get personalized opportunity recommendations |
| POST | `/api/v1/calculate-matches` | Calculate match scores for an opportunity |
| POST | `/api/v1/opportunities/{id}/match-candidates` | Find best-matching candidates for opportunity |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/analytics/participation-stats` | Get marketplace participation statistics |
| GET | `/api/v1/analytics/mobility-trends` | Get workforce mobility trend data |
| GET | `/api/v1/opportunities/{id}/analytics` | Get opportunity-specific analytics |

---

## 4. Business Rules

1. An opportunity MUST be in PUBLISHED status before employees can apply to it.
2. An employee MUST have their current manager's approval before being selected for an opportunity.
3. The system MUST prevent an employee from applying to opportunities that create a dual employment conflict with their primary role schedule.
4. An employee MUST NOT submit more than one application for the same opportunity.
5. The `positions_filled` count MUST NOT exceed the `headcount` value for any opportunity.
6. Skill match scores MUST be recalculated whenever opportunity skills or employee skills are updated.
7. Opportunities with visibility set to DEPARTMENT MUST only be visible to employees within that department.
8. Opportunities with visibility set to BUSINESS_UNIT MUST only be visible to employees within that business unit.
9. Match scores below the tenant-configured minimum threshold SHOULD NOT appear in employee recommendations.
10. The system SHOULD automatically close opportunities once `positions_filled` equals `headcount`.
11. The system MUST respect the `application_deadline` and prevent applications after the deadline has passed.
12. Completed assignments MUST prompt for feedback from both the employee and the opportunity owner.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package marketplace.v1;

service OpportunityMarketplaceService {
    // Opportunities
    rpc CreateOpportunity(CreateOpportunityRequest) returns (CreateOpportunityResponse);
    rpc GetOpportunity(GetOpportunityRequest) returns (GetOpportunityResponse);
    rpc ListOpportunities(ListOpportunitiesRequest) returns (ListOpportunitiesResponse);
    rpc PublishOpportunity(PublishOpportunityRequest) returns (PublishOpportunityResponse);
    rpc CloseOpportunity(CloseOpportunityRequest) returns (CloseOpportunityResponse);

    // Applications
    rpc ApplyForOpportunity(ApplyForOpportunityRequest) returns (ApplyForOpportunityResponse);
    rpc GetApplication(GetApplicationRequest) returns (GetApplicationResponse);
    rpc ListApplications(ListApplicationsRequest) returns (ListApplicationsResponse);
    rpc ApproveApplication(ApproveApplicationRequest) returns (ApproveApplicationResponse);
    rpc RejectApplication(RejectApplicationRequest) returns (RejectApplicationResponse);
    rpc WithdrawApplication(WithdrawApplicationRequest) returns (WithdrawApplicationResponse);

    // Preferences
    rpc GetEmployeePreferences(GetEmployeePreferencesRequest) returns (GetEmployeePreferencesResponse);
    rpc UpdateEmployeePreferences(UpdateEmployeePreferencesRequest) returns (UpdateEmployeePreferencesResponse);

    // Matching
    rpc GetRecommendations(GetRecommendationsRequest) returns (GetRecommendationsResponse);
    rpc CalculateMatches(CalculateMatchesRequest) returns (CalculateMatchesResponse);

    // Analytics
    rpc GetParticipationStats(GetParticipationStatsRequest) returns (GetParticipationStatsResponse);
    rpc GetMobilityTrends(GetMobilityTrendsRequest) returns (GetMobilityTrendsResponse);
}

message Opportunity {
    string id = 1;
    string tenant_id = 2;
    string title = 3;
    string description = 4;
    string type = 5;
    string department_id = 6;
    string business_unit_id = 7;
    string owner_id = 8;
    int32 headcount = 9;
    int32 positions_filled = 10;
    string status = 11;
    string visibility = 12;
    int32 duration_weeks = 13;
    string start_date = 14;
    string end_date = 15;
    string application_deadline = 16;
}

message CreateOpportunityRequest {
    string tenant_id = 1;
    string title = 2;
    string description = 3;
    string type = 4;
    string department_id = 5;
    string business_unit_id = 6;
    string owner_id = 7;
    int32 headcount = 8;
    string visibility = 9;
    int32 duration_weeks = 10;
    string start_date = 11;
    string end_date = 12;
    string application_deadline = 13;
    string created_by = 14;
}

message CreateOpportunityResponse {
    Opportunity opportunity = 1;
}

message GetOpportunityRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
}

message GetOpportunityResponse {
    Opportunity opportunity = 1;
}

message ListOpportunitiesRequest {
    string tenant_id = 1;
    string status = 2;
    string type = 3;
    string department_id = 4;
    string employee_id = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListOpportunitiesResponse {
    repeated Opportunity opportunities = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishOpportunityRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string updated_by = 3;
}

message PublishOpportunityResponse {
    Opportunity opportunity = 1;
}

message CloseOpportunityRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string updated_by = 3;
}

message CloseOpportunityResponse {
    Opportunity opportunity = 1;
}

message OpportunityApplication {
    string id = 1;
    string tenant_id = 2;
    string opportunity_id = 3;
    string employee_id = 4;
    string application_date = 5;
    string status = 6;
    double skills_match_score = 7;
    string manager_approval_status = 8;
}

message ApplyForOpportunityRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string employee_id = 3;
    string cover_statement = 4;
    string created_by = 5;
}

message ApplyForOpportunityResponse {
    OpportunityApplication application = 1;
}

message GetApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
}

message GetApplicationResponse {
    OpportunityApplication application = 1;
}

message ListApplicationsRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string employee_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListApplicationsResponse {
    repeated OpportunityApplication applications = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ApproveApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
    string approved_by_manager_id = 3;
    string manager_comments = 4;
}

message ApproveApplicationResponse {
    OpportunityApplication application = 1;
}

message RejectApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
    string approved_by_manager_id = 3;
    string manager_comments = 4;
}

message RejectApplicationResponse {
    OpportunityApplication application = 1;
}

message WithdrawApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
    string updated_by = 3;
}

message WithdrawApplicationResponse {
    OpportunityApplication application = 1;
}

message EmployeePreferences {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string preferred_types = 4;
    string preferred_departments = 5;
    string preferred_locations = 6;
    int32 max_travel_percentage = 7;
    string available_from = 8;
    bool notification_enabled = 9;
}

message GetEmployeePreferencesRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message GetEmployeePreferencesResponse {
    EmployeePreferences preferences = 1;
}

message UpdateEmployeePreferencesRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string preferred_types = 3;
    string preferred_departments = 4;
    string preferred_locations = 5;
    int32 max_travel_percentage = 6;
    string available_from = 7;
    bool notification_enabled = 8;
    string updated_by = 9;
}

message UpdateEmployeePreferencesResponse {
    EmployeePreferences preferences = 1;
}

message MatchScore {
    string opportunity_id = 1;
    double match_score = 2;
    double skill_match = 3;
    double experience_match = 4;
    double interests_match = 5;
}

message GetRecommendationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetRecommendationsResponse {
    repeated MatchScore recommendations = 1;
    string next_page_token = 2;
}

message CalculateMatchesRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string calculated_by = 3;
}

message CalculateMatchesResponse {
    repeated MatchScore matches = 1;
    int32 total_matched = 2;
}

message ParticipationStats {
    int32 total_opportunities = 1;
    int32 total_applications = 2;
    int32 total_selected = 3;
    int32 total_completed = 4;
    double avg_fill_rate = 5;
    double avg_time_to_fill_days = 6;
}

message GetParticipationStatsRequest {
    string tenant_id = 1;
    string department_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetParticipationStatsResponse {
    ParticipationStats stats = 1;
}

message MobilityTrend {
    string period = 1;
    int32 internal_moves = 2;
    int32 cross_department = 3;
    int32 cross_business_unit = 4;
    int32 gigs_completed = 5;
}

message GetMobilityTrendsRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
    string granularity = 4;  // WEEKLY, MONTHLY, QUARTERLY
}

message GetMobilityTrendsResponse {
    repeated MobilityTrend trends = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee profiles, org structure, department/business unit hierarchy | Validate employee eligibility and apply visibility rules |
| `skills-service` | Employee skills, proficiency levels, skill definitions | Calculate skill match scores |
| `workflow-service` | Approval workflows | Process manager approval flows for applications |
| `notification-service` | Notification delivery | Alert employees about new matching opportunities |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Assignment records, mobility data | Update employee assignment history |
| `skills-service` | Completed assignment skills | Update employee skill profiles with gained experience |
| `reporting-service` | Marketplace analytics | Workforce mobility and participation dashboards |
| `notification-service` | Opportunity alerts, application status changes | Notify employees and managers |
| `career-service` | Internal mobility history | Feed career development records |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `OpportunityPublished` | `marketplace.opportunity.published` | `{ tenant_id, opportunity_id, title, type, department_id, application_deadline }` | Published when an opportunity is made available on the marketplace |
| `ApplicationSubmitted` | `marketplace.application.submitted` | `{ tenant_id, application_id, opportunity_id, employee_id, application_date }` | Published when an employee applies for an opportunity |
| `ApplicationSelected` | `marketplace.application.selected` | `{ tenant_id, application_id, opportunity_id, employee_id, start_date, end_date }` | Published when a candidate is selected for an opportunity |
| `AssignmentCompleted` | `marketplace.assignment.completed` | `{ tenant_id, application_id, opportunity_id, employee_id, feedback_rating }` | Published when an assignment is marked as completed |
| `ManagerApprovalRequested` | `marketplace.approval.requested` | `{ tenant_id, application_id, employee_id, manager_id, opportunity_id }` | Published when manager approval is needed for an application |
