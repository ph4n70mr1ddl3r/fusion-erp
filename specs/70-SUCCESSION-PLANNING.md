# 70 - Succession Planning Service Specification

## 1. Domain Overview

The Succession Planning service manages talent pipeline identification and development for critical positions. It supports succession plan creation, candidate assessment, readiness evaluation, talent pool management, development action tracking, scenario modeling, and talent review processes to ensure organizational continuity.

**Bounded Context:** Succession & Talent Pipeline Management
**Service Name:** `succession-service`
**Database:** `data/succession.db`
**HTTP Port:** 8102 | **gRPC Port:** 9102

---

## 2. Database Schema

### 2.1 Succession Plans
```sql
CREATE TABLE succession_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('POSITION','ROLE','KEY_PERSON','EMERGENCY','STRATEGIC')),
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','UNDER_REVIEW','APPROVED','ARCHIVED')),
    owner_id TEXT NOT NULL,
    business_unit_id TEXT,
    review_frequency TEXT DEFAULT 'ANNUAL' CHECK(review_frequency IN ('QUARTERLY','SEMI_ANNUAL','ANNUAL','BIENNIAL')),
    last_review_date TEXT,
    next_review_date TEXT,
    description TEXT,
    risk_level TEXT DEFAULT 'MEDIUM' CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    time_sensitivity TEXT DEFAULT 'MEDIUM' CHECK(time_sensitivity IN ('LOW','MEDIUM','HIGH','IMMEDIATE')),
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_succession_plans_tenant_status ON succession_plans(tenant_id, status);
CREATE INDEX idx_succession_plans_tenant_owner ON succession_plans(tenant_id, owner_id);
```

### 2.2 Critical Positions
```sql
CREATE TABLE critical_positions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    position_id TEXT,
    job_id TEXT NOT NULL,
    position_title TEXT NOT NULL,
    department_id TEXT,
    business_unit_id TEXT,
    current_incumbent_id TEXT,
    criticality_level TEXT NOT NULL CHECK(criticality_level IN ('CRITICAL','HIGH','MEDIUM','LOW')),
    vacancy_risk TEXT DEFAULT 'MEDIUM' CHECK(vacancy_risk IN ('LOW','MEDIUM','HIGH','IMMINENT')),
    retirement_risk TEXT DEFAULT 'LOW' CHECK(retirement_risk IN ('LOW','MEDIUM','HIGH','WITHIN_2_YEARS')),
    time_to_fill_days INTEGER,
    replacement_cost INTEGER,  -- cents estimated
    impact_if_vacant TEXT,
    minimum_candidates INTEGER DEFAULT 2,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES succession_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_critical_positions_tenant_plan ON critical_positions(tenant_id, plan_id);
CREATE INDEX idx_critical_positions_tenant_incumbent ON critical_positions(tenant_id, current_incumbent_id);
CREATE INDEX idx_critical_positions_tenant_risk ON critical_positions(tenant_id, criticality_level, vacancy_risk);
```

### 2.3 Succession Candidates
```sql
CREATE TABLE succession_candidates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    critical_position_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    readiness_level TEXT NOT NULL CHECK(readiness_level IN ('READY_NOW','READY_IN_1_YEAR','READY_IN_2_YEARS','READY_IN_3_YEARS','NOT_READY')),
    candidate_rank INTEGER NOT NULL DEFAULT 1,
    nomination_source TEXT DEFAULT 'MANAGER' CHECK(nomination_source IN ('MANAGER','SELF','HR','EXECUTIVE','COMMITTEE')),
    performance_rating TEXT,
    potential_rating TEXT,
    flight_risk TEXT DEFAULT 'LOW' CHECK(flight_risk IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    development_needs TEXT,
    strengths TEXT,
    concerns TEXT,
    notes TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','UNDER_DEVELOPMENT','PROMOTED','REMOVED','ON_HOLD')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES succession_plans(id) ON DELETE CASCADE,
    FOREIGN KEY (critical_position_id) REFERENCES critical_positions(id) ON DELETE CASCADE
);

CREATE INDEX idx_succession_cands_tenant_plan ON succession_candidates(tenant_id, plan_id);
CREATE INDEX idx_succession_cands_tenant_employee ON succession_candidates(tenant_id, employee_id);
CREATE INDEX idx_succession_cands_tenant_readiness ON succession_candidates(tenant_id, readiness_level);
```

### 2.4 Talent Pools
```sql
CREATE TABLE talent_pools (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pool_name TEXT NOT NULL,
    pool_code TEXT NOT NULL,
    pool_type TEXT NOT NULL CHECK(pool_type IN ('HIGH_POTENTIAL','EMERGING_LEADERS','TECHNICAL_EXPERTS','EXECUTIVE','DIVERSITY','GENERAL','CUSTOM')),
    description TEXT,
    criteria TEXT,  -- JSON: eligibility criteria
    candidate_count INTEGER DEFAULT 0,
    owner_id TEXT NOT NULL,
    max_pool_size INTEGER,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),
    review_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, pool_code)
);

CREATE INDEX idx_talent_pools_tenant_type ON talent_pools(tenant_id, pool_type);
CREATE INDEX idx_talent_pools_tenant_status ON talent_pools(tenant_id, status);
```

### 2.5 Readiness Assessments
```sql
CREATE TABLE readiness_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    candidate_id TEXT NOT NULL,
    assessor_id TEXT NOT NULL,
    assessment_type TEXT NOT NULL CHECK(assessment_type IN ('COMPETENCY','POTENTIAL','PERFORMANCE','BEHAVIORAL','LEADERSHIP','CUSTOM')),
    assessment_date TEXT NOT NULL,
    overall_score DECIMAL(5,2),
    readiness_recommendation TEXT NOT NULL CHECK(readiness_recommendation IN ('READY_NOW','READY_IN_1_YEAR','READY_IN_2_YEARS','READY_IN_3_YEARS','NOT_READY')),
    competency_scores TEXT,  -- JSON: { competency: score }
    comments TEXT,
    development_recommendations TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','SUBMITTED','REVIEWED','APPROVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (candidate_id) REFERENCES succession_candidates(id) ON DELETE CASCADE
);

CREATE INDEX idx_readiness_tenant_candidate ON readiness_assessments(tenant_id, candidate_id);
CREATE INDEX idx_readiness_tenant_assessor ON readiness_assessments(tenant_id, assessor_id);
CREATE INDEX idx_readiness_tenant_date ON readiness_assessments(tenant_id, assessment_date);
```

### 2.6 Development Actions
```sql
CREATE TABLE development_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    candidate_id TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN ('TRAINING','MENTORING','COACHING','JOB_ROTATION','STRETCH_ASSIGNMENT','PROJECT_LEADERSHIP','FORMAL_EDUCATION','CERTIFICATION','NETWORKING','FEEDBACK')),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    target_competency TEXT,
    start_date TEXT NOT NULL,
    target_completion_date TEXT NOT NULL,
    actual_completion_date TEXT,
    responsible_person_id TEXT,
    priority TEXT DEFAULT 'MEDIUM' CHECK(priority IN ('HIGH','MEDIUM','LOW')),
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','CANCELLED','OVERDUE')),
    progress_percentage DECIMAL(5,2) DEFAULT 0,
    outcome TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (candidate_id) REFERENCES succession_candidates(id) ON DELETE CASCADE
);

CREATE INDEX idx_dev_actions_tenant_candidate ON development_actions(tenant_id, candidate_id);
CREATE INDEX idx_dev_actions_tenant_status ON development_actions(tenant_id, status);
CREATE INDEX idx_dev_actions_tenant_type ON development_actions(tenant_id, action_type);
```

### 2.7 Succession Scenarios
```sql
CREATE TABLE succession_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    scenario_type TEXT NOT NULL CHECK(scenario_type IN ('PLANNED_DEPARTURE','UNPLANNED_DEPARTURE','PROMOTION','REORGANIZATION','RETIREMENT','WHAT_IF')),
    description TEXT,
    trigger_event TEXT,
    affected_position_ids TEXT NOT NULL,  -- JSON array
    proposed_succession_map TEXT NOT NULL, -- JSON: { position_id: candidate_id }
    impact_analysis TEXT,
    transition_plan TEXT,
    estimated_transition_days INTEGER,
    risk_assessment TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','SIMULATED','APPROVED','EXECUTING','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES succession_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_scenarios_tenant_plan ON succession_scenarios(tenant_id, plan_id);
CREATE INDEX idx_scenarios_tenant_type ON succession_scenarios(tenant_id, scenario_type);
```

### 2.8 Talent Reviews
```sql
CREATE TABLE talent_reviews (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    review_name TEXT NOT NULL,
    review_date TEXT NOT NULL,
    facilitator_id TEXT NOT NULL,
    department_id TEXT,
    business_unit_id TEXT,
    participant_ids TEXT NOT NULL,  -- JSON array of reviewer IDs
    status TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK(status IN ('SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED')),
    review_criteria TEXT,  -- JSON: criteria definition
    ratings_discussed INTEGER DEFAULT 0,
    actions_agreed INTEGER DEFAULT 0,
    outcomes TEXT,
    next_review_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_talent_reviews_tenant_date ON talent_reviews(tenant_id, review_date);
CREATE INDEX idx_talent_reviews_tenant_status ON talent_reviews(tenant_id, status);
```

---

## 3. REST API Endpoints

### Succession Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans` | Create a succession plan |
| GET | `/api/v1/plans` | List succession plans |
| GET | `/api/v1/plans/{id}` | Get plan details |
| PUT | `/api/v1/plans/{id}` | Update plan |
| POST | `/api/v1/plans/{id}/activate` | Activate a plan |

### Critical Positions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans/{planId}/positions` | Identify a critical position |
| GET | `/api/v1/plans/{planId}/positions` | List critical positions |
| GET | `/api/v1/positions/{id}` | Get position details |
| PUT | `/api/v1/positions/{id}` | Update position |

### Candidates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/positions/{positionId}/candidates` | Add candidate to succession |
| GET | `/api/v1/positions/{positionId}/candidates` | List position candidates |
| PUT | `/api/v1/candidates/{id}` | Update candidate details |
| DELETE | `/api/v1/candidates/{id}` | Remove candidate |
| GET | `/api/v1/employees/{employeeId}/succession-candidates` | Get employee's succession candidacies |

### Readiness Assessment
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/candidates/{candidateId}/assessments` | Create readiness assessment |
| GET | `/api/v1/candidates/{candidateId}/assessments` | List candidate assessments |
| GET | `/api/v1/assessments/{id}` | Get assessment details |
| POST | `/api/v1/assessments/{id}/submit` | Submit assessment |

### Development Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/candidates/{candidateId}/development-actions` | Create development action |
| GET | `/api/v1/candidates/{candidateId}/development-actions` | List development actions |
| PUT | `/api/v1/development-actions/{id}` | Update action |
| POST | `/api/v1/development-actions/{id}/complete` | Mark action complete |

### Talent Pools
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-pools` | Create talent pool |
| GET | `/api/v1/talent-pools` | List talent pools |
| GET | `/api/v1/talent-pools/{id}` | Get pool details |
| POST | `/api/v1/talent-pools/{id}/members` | Add member to pool |
| GET | `/api/v1/talent-pools/{id}/members` | List pool members |
| DELETE | `/api/v1/talent-pools/{poolId}/members/{memberId}` | Remove member |

### Scenarios
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans/{planId}/scenarios` | Create succession scenario |
| GET | `/api/v1/plans/{planId}/scenarios` | List scenarios |
| GET | `/api/v1/scenarios/{id}` | Get scenario details |
| POST | `/api/v1/scenarios/{id}/simulate` | Simulate scenario impact |

### Talent Reviews
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-reviews` | Schedule a talent review |
| GET | `/api/v1/talent-reviews` | List talent reviews |
| GET | `/api/v1/talent-reviews/{id}` | Get review details |
| POST | `/api/v1/talent-reviews/{id}/complete` | Complete talent review |

---

## 4. Business Rules

1. A critical position MUST have at least the configured minimum number of candidates.
2. A candidate's readiness level MUST be determined by the most recent assessment.
3. An employee MUST NOT be nominated as their own successor.
4. Succession plans MUST be reviewed at the configured review frequency.
5. Flight risk assessment for candidates MUST be refreshed at least annually.
6. Development actions for a candidate MUST target identified competency gaps.
7. Scenario simulation MUST calculate the cascading impact of multiple position vacancies.
8. Talent pool membership MUST be validated against the configured criteria.
9. Readiness assessments MUST be conducted by someone other than the candidate.
10. Critical positions without any candidates MUST generate alerts to the plan owner.
11. The system SHOULD recommend candidates based on performance, potential, and competency scores.
12. Development actions marked as overdue MUST trigger escalation notifications.
13. Succession plans marked as `CRITICAL` risk MUST have executive visibility.
14. A talent review MUST include at least one HR representative as participant.
15. Candidate ranking within a position MUST be unique (no ties).

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package succession.v1;

service SuccessionService {
    // Plans
    rpc CreatePlan(CreatePlanRequest) returns (CreatePlanResponse);
    rpc GetPlan(GetPlanRequest) returns (GetPlanResponse);
    rpc ListPlans(ListPlansRequest) returns (ListPlansResponse);

    // Positions
    rpc AddCriticalPosition(AddCriticalPositionRequest) returns (AddCriticalPositionResponse);
    rpc GetCriticalPosition(GetCriticalPositionRequest) returns (GetCriticalPositionResponse);
    rpc ListCriticalPositions(ListCriticalPositionsRequest) returns (ListCriticalPositionsResponse);

    // Candidates
    rpc AddCandidate(AddCandidateRequest) returns (AddCandidateResponse);
    rpc GetCandidate(GetCandidateRequest) returns (GetCandidateResponse);
    rpc ListCandidates(ListCandidatesRequest) returns (ListCandidatesResponse);
    rpc RemoveCandidate(RemoveCandidateRequest) returns (RemoveCandidateResponse);

    // Readiness
    rpc AssessReadiness(AssessReadinessRequest) returns (AssessReadinessResponse);
    rpc GetReadinessHistory(GetReadinessHistoryRequest) returns (GetReadinessHistoryResponse);

    // Development
    rpc CreateDevelopmentAction(CreateDevelopmentActionRequest) returns (CreateDevelopmentActionResponse);
    rpc ListDevelopmentActions(ListDevelopmentActionsRequest) returns (ListDevelopmentActionsResponse);

    // Talent pools
    rpc CreateTalentPool(CreateTalentPoolRequest) returns (CreateTalentPoolResponse);
    rpc AddPoolMember(AddPoolMemberRequest) returns (AddPoolMemberResponse);

    // Scenarios
    rpc CreateScenario(CreateScenarioRequest) returns (CreateScenarioResponse);
    rpc SimulateScenario(SimulateScenarioRequest) returns (SimulateScenarioResponse);

    // Talent reviews
    rpc CreateTalentReview(CreateTalentReviewRequest) returns (CreateTalentReviewResponse);
    rpc CompleteTalentReview(CompleteTalentReviewRequest) returns (CompleteTalentReviewResponse);
}

message SuccessionPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_code = 4;
    string plan_type = 5;
    string status = 6;
    string owner_id = 7;
    string risk_level = 8;
}

message CreatePlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    string plan_code = 3;
    string plan_type = 4;
    string owner_id = 5;
    string created_by = 6;
}

message CreatePlanResponse {
    SuccessionPlan plan = 1;
}

message GetPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetPlanResponse {
    SuccessionPlan plan = 1;
}

message ListPlansRequest {
    string tenant_id = 1;
    string status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListPlansResponse {
    repeated SuccessionPlan plans = 1;
    string next_page_token = 2;
}

message CriticalPosition {
    string id = 1;
    string plan_id = 2;
    string position_title = 3;
    string current_incumbent_id = 4;
    string criticality_level = 5;
    string vacancy_risk = 6;
}

message AddCriticalPositionRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string job_id = 3;
    string position_title = 4;
    string current_incumbent_id = 5;
    string criticality_level = 6;
    string created_by = 7;
}

message AddCriticalPositionResponse {
    CriticalPosition position = 1;
}

message GetCriticalPositionRequest {
    string tenant_id = 1;
    string position_id = 2;
}

message GetCriticalPositionResponse {
    CriticalPosition position = 1;
}

message ListCriticalPositionsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string criticality_level = 3;
}

message ListCriticalPositionsResponse {
    repeated CriticalPosition positions = 1;
}

message SuccessionCandidate {
    string id = 1;
    string plan_id = 2;
    string critical_position_id = 3;
    string employee_id = 4;
    string readiness_level = 5;
    int32 candidate_rank = 6;
    string status = 7;
}

message AddCandidateRequest {
    string tenant_id = 1;
    string critical_position_id = 2;
    string employee_id = 3;
    string readiness_level = 4;
    string nomination_source = 5;
    string created_by = 6;
}

message AddCandidateResponse {
    SuccessionCandidate candidate = 1;
}

message GetCandidateRequest {
    string tenant_id = 1;
    string candidate_id = 2;
}

message GetCandidateResponse {
    SuccessionCandidate candidate = 1;
}

message ListCandidatesRequest {
    string tenant_id = 1;
    string critical_position_id = 2;
    string readiness_level = 3;
}

message ListCandidatesResponse {
    repeated SuccessionCandidate candidates = 1;
}

message RemoveCandidateRequest {
    string tenant_id = 1;
    string candidate_id = 2;
    string reason = 3;
    string removed_by = 4;
}

message RemoveCandidateResponse {
    string candidate_id = 1;
    string status = 2;
}

message ReadinessAssessment {
    string id = 1;
    string candidate_id = 2;
    string assessment_type = 3;
    double overall_score = 4;
    string readiness_recommendation = 5;
    string assessment_date = 6;
}

message AssessReadinessRequest {
    string tenant_id = 1;
    string candidate_id = 2;
    string assessor_id = 3;
    string assessment_type = 4;
    double overall_score = 5;
    string readiness_recommendation = 6;
    string comments = 7;
}

message AssessReadinessResponse {
    ReadinessAssessment assessment = 1;
}

message GetReadinessHistoryRequest {
    string tenant_id = 1;
    string candidate_id = 2;
}

message GetReadinessHistoryResponse {
    repeated ReadinessAssessment assessments = 1;
}

message DevelopmentAction {
    string id = 1;
    string candidate_id = 2;
    string action_type = 3;
    string title = 4;
    string status = 5;
    string target_completion_date = 6;
    double progress_percentage = 7;
}

message CreateDevelopmentActionRequest {
    string tenant_id = 1;
    string candidate_id = 2;
    string action_type = 3;
    string title = 4;
    string description = 5;
    string start_date = 6;
    string target_completion_date = 7;
    string created_by = 8;
}

message CreateDevelopmentActionResponse {
    DevelopmentAction action = 1;
}

message ListDevelopmentActionsRequest {
    string tenant_id = 1;
    string candidate_id = 2;
    string status = 3;
}

message ListDevelopmentActionsResponse {
    repeated DevelopmentAction actions = 1;
}

message TalentPoolInfo {
    string id = 1;
    string pool_name = 2;
    string pool_type = 3;
    int32 candidate_count = 4;
    string status = 5;
}

message CreateTalentPoolRequest {
    string tenant_id = 1;
    string pool_name = 2;
    string pool_code = 3;
    string pool_type = 4;
    string owner_id = 5;
    string created_by = 6;
}

message CreateTalentPoolResponse {
    TalentPoolInfo pool = 1;
}

message AddPoolMemberRequest {
    string tenant_id = 1;
    string pool_id = 2;
    string employee_id = 3;
    string added_by = 4;
}

message AddPoolMemberResponse {
    string member_id = 1;
    int32 pool_count = 2;
}

message Scenario {
    string id = 1;
    string plan_id = 2;
    string scenario_name = 3;
    string scenario_type = 4;
    string status = 5;
}

message CreateScenarioRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string scenario_name = 3;
    string scenario_type = 4;
    string proposed_succession_map = 5;
    string created_by = 6;
}

message CreateScenarioResponse {
    Scenario scenario = 1;
}

message SimulateScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
}

message SimulateScenarioResponse {
    string scenario_id = 1;
    int32 positions_affected = 2;
    int32 candidates_promoted = 3;
    int32 positions_needing_backfill = 4;
    double readiness_score = 5;
}

message TalentReview {
    string id = 1;
    string review_name = 2;
    string review_date = 3;
    string facilitator_id = 4;
    string status = 5;
}

message CreateTalentReviewRequest {
    string tenant_id = 1;
    string review_name = 2;
    string review_date = 3;
    string facilitator_id = 4;
    string created_by = 5;
}

message CreateTalentReviewResponse {
    TalentReview review = 1;
}

message CompleteTalentReviewRequest {
    string tenant_id = 1;
    string review_id = 2;
    string outcomes = 3;
    string completed_by = 4;
}

message CompleteTalentReviewResponse {
    TalentReview review = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, org hierarchy, positions | Identify incumbents and positions |
| `performance-service` | Performance ratings, review outcomes | Assess candidate performance |
| `learning-service` | Development completions, certifications | Track readiness progress |
| `career-service` | Skill assessments, career preferences | Match candidates to roles |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `learning-service` | Development action requirements | Create learning assignments |
| `career-service` | Succession candidacy data | Career path alignment |
| `reporting-service` | Succession analytics | Talent pipeline reports |
| `hr-service` | Position vacancy risk updates | Workforce planning |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `CriticalPositionIdentified` | `succession.position.identified` | `{ tenant_id, position_id, plan_id, position_title, criticality_level, incumbent_id }` | Published when a position is marked as critical |
| `CandidateAssessed` | `succession.candidate.assessed` | `{ tenant_id, candidate_id, employee_id, readiness_level, overall_score, assessor_id }` | Published when a readiness assessment is completed |
| `SuccessionPlanUpdated` | `succession.plan.updated` | `{ tenant_id, plan_id, plan_name, status, position_count, candidate_count }` | Published when a plan is modified |
| `TalentReviewCompleted` | `succession.review.completed` | `{ tenant_id, review_id, review_name, ratings_discussed, actions_agreed }` | Published when a talent review session ends |
| `CandidatePromoted` | `succession.candidate.promoted` | `{ tenant_id, candidate_id, employee_id, position_id, new_position }` | Published when a succession candidate fills the role |
| `DevelopmentActionCompleted` | `succession.development.completed` | `{ tenant_id, action_id, candidate_id, action_type, outcome }` | Published when a development action is finished |
| `FlightRiskAlert` | `succession.flight_risk.alert` | `{ tenant_id, candidate_id, employee_id, flight_risk_level, position_title }` | Published when flight risk is elevated |
