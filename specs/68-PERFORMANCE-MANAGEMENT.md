# 68 - Performance Management Service Specification

## 1. Domain Overview

The Performance Management service manages employee performance evaluation cycles including goal setting, goal alignment, review creation, multi-source feedback, performance rating, calibration sessions, and performance improvement plans (PIPs). It supports continuous performance management with ongoing feedback and periodic formal assessments.

**Bounded Context:** Performance & Talent Assessment
**Service Name:** `performance-service`
**Database:** `data/performance.db`
**HTTP Port:** 8100 | **gRPC Port:** 9100

---

## 2. Database Schema

### 2.1 Performance Cycles
```sql
CREATE TABLE performance_cycles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_name TEXT NOT NULL,
    cycle_code TEXT NOT NULL,
    cycle_type TEXT NOT NULL CHECK(cycle_type IN ('ANNUAL','SEMI_ANNUAL','QUARTERLY','MONTHLY','AD_HOC')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    goal_setting_start TEXT,
    goal_setting_end TEXT,
    self_assessment_start TEXT,
    self_assessment_end TEXT,
    manager_assessment_start TEXT,
    manager_assessment_end TEXT,
    calibration_start TEXT,
    calibration_end TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','GOAL_SETTING','SELF_ASSESSMENT','MANAGER_ASSESSMENT','CALIBRATION','FINALIZED','CLOSED')),
    rating_model_id TEXT,
    target_population TEXT,  -- JSON: eligibility criteria
    description TEXT,
    auto_advance INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, cycle_code)
);

CREATE INDEX idx_perf_cycles_tenant_status ON performance_cycles(tenant_id, status);
CREATE INDEX idx_perf_cycles_tenant_dates ON performance_cycles(tenant_id, start_date, end_date);
```

### 2.2 Performance Goals
```sql
CREATE TABLE performance_goals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    goal_name TEXT NOT NULL,
    goal_description TEXT NOT NULL,
    goal_category TEXT NOT NULL CHECK(goal_category IN ('PERFORMANCE','DEVELOPMENT','BEHAVIORAL','STRATEGIC','OPERATIONAL','CUSTOM')),
    target_metric TEXT NOT NULL,
    target_value TEXT NOT NULL,
    actual_value TEXT,
    progress_percentage DECIMAL(5,2) DEFAULT 0,
    weight DECIMAL(5,2) DEFAULT 0,
    priority TEXT DEFAULT 'MEDIUM' CHECK(priority IN ('HIGH','MEDIUM','LOW')),
    start_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    completion_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','PENDING_APPROVAL','APPROVED','IN_PROGRESS','COMPLETED','CANCELLED','OVERDUE')),
    parent_goal_id TEXT,
    source TEXT DEFAULT 'SELF' CHECK(source IN ('SELF','MANAGER','ORGANIZATION','CASCADE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES performance_cycles(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_goal_id) REFERENCES performance_goals(id) ON DELETE SET NULL
);

CREATE INDEX idx_perf_goals_tenant_cycle ON performance_goals(tenant_id, cycle_id);
CREATE INDEX idx_perf_goals_tenant_employee ON performance_goals(tenant_id, employee_id, cycle_id);
CREATE INDEX idx_perf_goals_tenant_status ON performance_goals(tenant_id, status);
CREATE INDEX idx_perf_goals_tenant_parent ON performance_goals(tenant_id, parent_goal_id);
```

### 2.3 Goal Alignments
```sql
CREATE TABLE goal_alignments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_goal_id TEXT NOT NULL,
    target_goal_id TEXT NOT NULL,
    alignment_type TEXT NOT NULL CHECK(alignment_type IN ('CASCADE','ALIGN','CONTRIBUTE')),
    alignment_strength TEXT DEFAULT 'FULL' CHECK(alignment_strength IN ('FULL','PARTIAL','INDIRECT')),
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_goal_id) REFERENCES performance_goals(id) ON DELETE CASCADE,
    FOREIGN KEY (target_goal_id) REFERENCES performance_goals(id) ON DELETE CASCADE
);

CREATE INDEX idx_goal_align_tenant_source ON goal_alignments(tenant_id, source_goal_id);
CREATE INDEX idx_goal_align_tenant_target ON goal_alignments(tenant_id, target_goal_id);
```

### 2.4 Performance Reviews
```sql
CREATE TABLE performance_reviews (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    reviewer_id TEXT NOT NULL,
    review_type TEXT NOT NULL CHECK(review_type IN ('SELF','MANAGER','PEER','DIRECT_REPORT','SKIP_LEVEL','EXTERNAL')),
    review_number TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED' CHECK(status IN ('NOT_STARTED','IN_PROGRESS','SUBMITTED','ACKNOWLEDGED','CALIBRATED','FINALIZED','DISPUTED')),
    overall_rating_id TEXT,
    overall_comments TEXT,
    submitted_at TEXT,
    acknowledged_at TEXT,
    disputed_at TEXT,
    dispute_reason TEXT,
    due_date TEXT,
    late_flag INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES performance_cycles(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, review_number)
);

CREATE INDEX idx_perf_reviews_tenant_cycle ON performance_reviews(tenant_id, cycle_id);
CREATE INDEX idx_perf_reviews_tenant_employee ON performance_reviews(tenant_id, employee_id);
CREATE INDEX idx_perf_reviews_tenant_reviewer ON performance_reviews(tenant_id, reviewer_id, status);
CREATE INDEX idx_perf_reviews_tenant_status ON performance_reviews(tenant_id, status);
```

### 2.5 Review Sections
```sql
CREATE TABLE review_sections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    review_id TEXT NOT NULL,
    section_type TEXT NOT NULL CHECK(section_type IN ('GOAL_REVIEW','COMPETENCY','SKILL','BEHAVIOR','OVERALL','CUSTOM')),
    section_title TEXT NOT NULL,
    section_description TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    weight DECIMAL(5,2) DEFAULT 0,
    response_text TEXT,
    rating_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (review_id) REFERENCES performance_reviews(id) ON DELETE CASCADE
);

CREATE INDEX idx_review_sections_tenant_review ON review_sections(tenant_id, review_id);
```

### 2.6 Performance Ratings
```sql
CREATE TABLE performance_ratings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rating_model_id TEXT NOT NULL,
    rating_level INTEGER NOT NULL,
    rating_label TEXT NOT NULL,
    rating_description TEXT,
    numeric_value DECIMAL(5,2) NOT NULL,
    color_code TEXT,
    is_default INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rating_model_id, rating_level)
);

CREATE INDEX idx_perf_ratings_tenant_model ON performance_ratings(tenant_id, rating_model_id);
```

### 2.7 Calibration Sessions
```sql
CREATE TABLE calibration_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    session_name TEXT NOT NULL,
    facilitator_id TEXT NOT NULL,
    department_id TEXT,
    business_unit_id TEXT,
    session_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK(status IN ('SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED')),
    participant_ids TEXT NOT NULL,  -- JSON array
    ratings_adjusted INTEGER DEFAULT 0,
    ratings_confirmed INTEGER DEFAULT 0,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES performance_cycles(id) ON DELETE CASCADE
);

CREATE INDEX idx_calib_sessions_tenant_cycle ON calibration_sessions(tenant_id, cycle_id);
CREATE INDEX idx_calib_sessions_tenant_status ON calibration_sessions(tenant_id, status);
```

### 2.8 Feedback Items
```sql
CREATE TABLE feedback_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT,
    from_employee_id TEXT NOT NULL,
    to_employee_id TEXT NOT NULL,
    feedback_type TEXT NOT NULL CHECK(feedback_type IN ('RECOGNITION','CONSTRUCTIVE','GENERAL','REQUESTED','ANONYMOUS')),
    feedback_text TEXT NOT NULL,
    competency_tags TEXT,  -- JSON array of competency tags
    visibility TEXT DEFAULT 'MANAGER' CHECK(visibility IN ('PRIVATE','MANAGER','RECIPIENT','PUBLIC')),
    is_anonymous INTEGER DEFAULT 0,
    status TEXT DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','HIDDEN','FLAGGED','ARCHIVED')),
    feedback_date TEXT NOT NULL,
    related_goal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES performance_cycles(id) ON DELETE SET NULL
);

CREATE INDEX idx_feedback_tenant_to ON feedback_items(tenant_id, to_employee_id, feedback_date);
CREATE INDEX idx_feedback_tenant_from ON feedback_items(tenant_id, from_employee_id);
CREATE INDEX idx_feedback_tenant_cycle ON feedback_items(tenant_id, cycle_id);
```

### 2.9 Performance Plans
```sql
CREATE TABLE performance_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('PIP','DEVELOPMENT','COACHING','RETENTION','SUCCESSION')),
    plan_name TEXT NOT NULL,
    description TEXT NOT NULL,
    objectives TEXT NOT NULL,
    success_criteria TEXT NOT NULL,
    support_resources TEXT,
    start_date TEXT NOT NULL,
    review_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    hr_representative_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','IN_PROGRESS','UNDER_REVIEW','COMPLETED_SUCCESS','COMPLETED_UNSUCCESSFUL','CANCELLED','ESCALATED')),
    outcome TEXT,
    milestone_count INTEGER DEFAULT 0,
    completed_milestones INTEGER DEFAULT 0,
    escalation_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_perf_plans_tenant_employee ON performance_plans(tenant_id, employee_id);
CREATE INDEX idx_perf_plans_tenant_status ON performance_plans(tenant_id, status);
CREATE INDEX idx_perf_plans_tenant_review ON performance_plans(tenant_id, review_date);
```

---

## 3. REST API Endpoints

### Cycles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/cycles` | Create a performance cycle |
| GET | `/api/v1/cycles` | List performance cycles |
| GET | `/api/v1/cycles/{id}` | Get cycle details |
| PUT | `/api/v1/cycles/{id}` | Update cycle configuration |
| POST | `/api/v1/cycles/{id}/activate` | Activate cycle |
| PATCH | `/api/v1/cycles/{id}/advance` | Advance cycle to next phase |
| POST | `/api/v1/cycles/{id}/finalize` | Finalize cycle |

### Goals
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/goals` | Create a performance goal |
| GET | `/api/v1/goals` | List goals with filters |
| GET | `/api/v1/goals/{id}` | Get goal details |
| PUT | `/api/v1/goals/{id}` | Update goal |
| POST | `/api/v1/goals/{id}/submit` | Submit goal for approval |
| POST | `/api/v1/goals/{id}/approve` | Approve goal |
| PUT | `/api/v1/goals/{id}/progress` | Update goal progress |
| GET | `/api/v1/employees/{employeeId}/goals` | Get employee goals |
| GET | `/api/v1/goals/{id}/alignment` | Get goal alignment tree |

### Reviews
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/reviews` | Create a performance review |
| GET | `/api/v1/reviews` | List reviews with filters |
| GET | `/api/v1/reviews/{id}` | Get review with sections |
| PUT | `/api/v1/reviews/{id}` | Update review content |
| POST | `/api/v1/reviews/{id}/submit` | Submit review |
| POST | `/api/v1/reviews/{id}/acknowledge` | Acknowledge review |
| POST | `/api/v1/reviews/{id}/dispute` | Dispute review |
| GET | `/api/v1/employees/{employeeId}/reviews` | Get employee review history |

### Feedback
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/feedback` | Give feedback |
| GET | `/api/v1/feedback` | List feedback with filters |
| GET | `/api/v1/feedback/{id}` | Get feedback details |
| GET | `/api/v1/employees/{employeeId}/feedback` | Get employee feedback |

### Calibration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/calibration-sessions` | Create calibration session |
| GET | `/api/v1/calibration-sessions` | List calibration sessions |
| GET | `/api/v1/calibration-sessions/{id}` | Get session details |
| POST | `/api/v1/calibration-sessions/{id}/adjust` | Adjust rating in session |
| POST | `/api/v1/calibration-sessions/{id}/confirm` | Confirm session ratings |
| GET | `/api/v1/calibration-sessions/{id}/distribution` | Get rating distribution |

### Performance Plans (PIP)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/performance-plans` | Create a performance plan |
| GET | `/api/v1/performance-plans` | List performance plans |
| GET | `/api/v1/performance-plans/{id}` | Get plan details |
| PUT | `/api/v1/performance-plans/{id}` | Update plan |
| POST | `/api/v1/performance-plans/{id}/review` | Conduct plan review |
| POST | `/api/v1/performance-plans/{id}/complete` | Complete plan |

---

## 4. Business Rules

1. A performance cycle MUST define all phase dates before activation.
2. An employee MUST NOT have more than one active review of the same type in a single cycle.
3. Goal weights for an employee in a cycle SHOULD sum to 100%.
4. Self-assessment MUST be submitted before the manager assessment can be viewed by the manager.
5. Calibration session participants MUST include at least one HR representative.
6. Performance ratings adjusted during calibration MUST preserve the original rating for audit.
7. A PIP MUST have at least one review date between start and end dates.
8. Feedback visibility MUST be enforced; anonymous feedback MUST NOT reveal the source employee.
9. Goal alignment MUST NOT create circular references.
10. An employee MUST be able to dispute a review within a configurable window after submission.
11. The system MUST auto-flag overdue reviews and goals.
12. Rating distribution in calibration SHOULD follow a predefined curve guideline.
13. Performance plans with outcome `COMPLETED_UNSUCCESSFUL` MUST trigger an escalation workflow.
14. The system SHOULD support at least 3 levels of goal cascading.
15. Cycle finalization MUST lock all ratings and prevent further modifications.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package performance.v1;

service PerformanceService {
    // Cycles
    rpc CreateCycle(CreateCycleRequest) returns (CreateCycleResponse);
    rpc GetCycle(GetCycleRequest) returns (GetCycleResponse);
    rpc ListCycles(ListCyclesRequest) returns (ListCyclesResponse);
    rpc AdvanceCycle(AdvanceCycleRequest) returns (AdvanceCycleResponse);
    rpc FinalizeCycle(FinalizeCycleRequest) returns (FinalizeCycleResponse);

    // Goals
    rpc CreateGoal(CreateGoalRequest) returns (CreateGoalResponse);
    rpc GetGoal(GetGoalRequest) returns (GetGoalResponse);
    rpc ListEmployeeGoals(ListEmployeeGoalsRequest) returns (ListEmployeeGoalsResponse);
    rpc UpdateGoalProgress(UpdateGoalProgressRequest) returns (UpdateGoalProgressResponse);
    rpc ApproveGoal(ApproveGoalRequest) returns (ApproveGoalResponse);

    // Reviews
    rpc CreateReview(CreateReviewRequest) returns (CreateReviewResponse);
    rpc GetReview(GetReviewRequest) returns (GetReviewResponse);
    rpc SubmitReview(SubmitReviewRequest) returns (SubmitReviewResponse);
    rpc ListEmployeeReviews(ListEmployeeReviewsRequest) returns (ListEmployeeReviewsResponse);

    // Feedback
    rpc GiveFeedback(GiveFeedbackRequest) returns (GiveFeedbackResponse);
    rpc ListFeedback(ListFeedbackRequest) returns (ListFeedbackResponse);

    // Calibration
    rpc CreateCalibrationSession(CreateCalibrationSessionRequest) returns (CreateCalibrationSessionResponse);
    rpc AdjustRating(AdjustRatingRequest) returns (AdjustRatingResponse);
    rpc GetRatingDistribution(GetRatingDistributionRequest) returns (GetRatingDistributionResponse);

    // Performance plans
    rpc CreatePerformancePlan(CreatePerformancePlanRequest) returns (CreatePerformancePlanResponse);
    rpc GetPerformancePlan(GetPerformancePlanRequest) returns (GetPerformancePlanResponse);
}

message PerformanceCycle {
    string id = 1;
    string tenant_id = 2;
    string cycle_name = 3;
    string cycle_type = 4;
    string start_date = 5;
    string end_date = 6;
    string status = 7;
}

message CreateCycleRequest {
    string tenant_id = 1;
    string cycle_name = 2;
    string cycle_type = 3;
    string start_date = 4;
    string end_date = 5;
    string created_by = 6;
}

message CreateCycleResponse {
    PerformanceCycle cycle = 1;
}

message GetCycleRequest {
    string tenant_id = 1;
    string cycle_id = 2;
}

message GetCycleResponse {
    PerformanceCycle cycle = 1;
}

message ListCyclesRequest {
    string tenant_id = 1;
    string status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCyclesResponse {
    repeated PerformanceCycle cycles = 1;
    string next_page_token = 2;
}

message AdvanceCycleRequest {
    string tenant_id = 1;
    string cycle_id = 2;
    string advanced_by = 3;
}

message AdvanceCycleResponse {
    PerformanceCycle cycle = 1;
}

message FinalizeCycleRequest {
    string tenant_id = 1;
    string cycle_id = 2;
    string finalized_by = 3;
}

message FinalizeCycleResponse {
    PerformanceCycle cycle = 1;
}

message PerformanceGoal {
    string id = 1;
    string tenant_id = 2;
    string cycle_id = 3;
    string employee_id = 4;
    string goal_name = 5;
    string goal_category = 6;
    double progress_percentage = 7;
    double weight = 8;
    string status = 9;
}

message CreateGoalRequest {
    string tenant_id = 1;
    string cycle_id = 2;
    string employee_id = 3;
    string goal_name = 4;
    string goal_description = 5;
    string target_metric = 6;
    string target_value = 7;
    double weight = 8;
    string due_date = 9;
    string created_by = 10;
}

message CreateGoalResponse {
    PerformanceGoal goal = 1;
}

message GetGoalRequest {
    string tenant_id = 1;
    string goal_id = 2;
}

message GetGoalResponse {
    PerformanceGoal goal = 1;
}

message ListEmployeeGoalsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string cycle_id = 3;
    string status = 4;
}

message ListEmployeeGoalsResponse {
    repeated PerformanceGoal goals = 1;
}

message UpdateGoalProgressRequest {
    string tenant_id = 1;
    string goal_id = 2;
    double progress_percentage = 3;
    string actual_value = 4;
    string updated_by = 5;
}

message UpdateGoalProgressResponse {
    PerformanceGoal goal = 1;
}

message ApproveGoalRequest {
    string tenant_id = 1;
    string goal_id = 2;
    string approved_by = 3;
}

message ApproveGoalResponse {
    PerformanceGoal goal = 1;
}

message Review {
    string id = 1;
    string tenant_id = 2;
    string cycle_id = 3;
    string employee_id = 4;
    string reviewer_id = 5;
    string review_type = 6;
    string status = 7;
    string overall_rating_id = 8;
}

message CreateReviewRequest {
    string tenant_id = 1;
    string cycle_id = 2;
    string employee_id = 3;
    string reviewer_id = 4;
    string review_type = 5;
    string created_by = 6;
}

message CreateReviewResponse {
    Review review = 1;
}

message GetReviewRequest {
    string tenant_id = 1;
    string review_id = 2;
}

message GetReviewResponse {
    Review review = 1;
}

message SubmitReviewRequest {
    string tenant_id = 1;
    string review_id = 2;
    string overall_comments = 3;
    string submitted_by = 4;
}

message SubmitReviewResponse {
    Review review = 1;
}

message ListEmployeeReviewsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string cycle_id = 3;
}

message ListEmployeeReviewsResponse {
    repeated Review reviews = 1;
}

message FeedbackItem {
    string id = 1;
    string tenant_id = 2;
    string from_employee_id = 3;
    string to_employee_id = 4;
    string feedback_type = 5;
    string feedback_text = 6;
    string feedback_date = 7;
}

message GiveFeedbackRequest {
    string tenant_id = 1;
    string from_employee_id = 2;
    string to_employee_id = 3;
    string feedback_type = 4;
    string feedback_text = 5;
    string created_by = 6;
}

message GiveFeedbackResponse {
    FeedbackItem feedback = 1;
}

message ListFeedbackRequest {
    string tenant_id = 1;
    string to_employee_id = 2;
    string cycle_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListFeedbackResponse {
    repeated FeedbackItem items = 1;
    string next_page_token = 2;
}

message CalibrationSession {
    string id = 1;
    string tenant_id = 2;
    string cycle_id = 3;
    string session_name = 4;
    string facilitator_id = 5;
    string status = 6;
}

message CreateCalibrationSessionRequest {
    string tenant_id = 1;
    string cycle_id = 2;
    string session_name = 3;
    string facilitator_id = 4;
    string session_date = 5;
    string created_by = 6;
}

message CreateCalibrationSessionResponse {
    CalibrationSession session = 1;
}

message AdjustRatingRequest {
    string tenant_id = 1;
    string session_id = 2;
    string review_id = 3;
    string new_rating_id = 4;
    string adjusted_by = 5;
    string reason = 6;
}

message AdjustRatingResponse {
    string review_id = 1;
    string new_rating_id = 2;
}

message RatingDistributionEntry {
    string rating_label = 1;
    int32 count = 2;
    double percentage = 3;
}

message GetRatingDistributionRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message GetRatingDistributionResponse {
    repeated RatingDistributionEntry distribution = 1;
    int32 total_reviews = 2;
}

message PerformancePlan {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string plan_type = 4;
    string plan_name = 5;
    string status = 6;
    string start_date = 7;
    string end_date = 8;
}

message CreatePerformancePlanRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_type = 3;
    string plan_name = 4;
    string description = 5;
    string objectives = 6;
    string success_criteria = 7;
    string start_date = 8;
    string review_date = 9;
    string end_date = 10;
    string manager_id = 11;
    string created_by = 12;
}

message CreatePerformancePlanResponse {
    PerformancePlan plan = 1;
}

message GetPerformancePlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetPerformancePlanResponse {
    PerformancePlan plan = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, org hierarchy, managers | Determine review participants |
| `workflow-service` | Approval workflows | Review and goal approval routing |
| `compensation-service` | Compensation plans | Link ratings to compensation decisions |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `compensation-service` | Finalized ratings | Factor into compensation awards |
| `succession-service` | Performance data | Talent review and readiness assessment |
| `learning-service` | Development goals | Assign training based on gaps |
| `reporting-service` | Performance analytics | Performance dashboards |
| `hr-service` | PIP status updates | Employee record updates |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `CycleStarted` | `performance.cycle.started` | `{ tenant_id, cycle_id, cycle_name, cycle_type, start_date }` | Published when a performance cycle is activated |
| `GoalsSet` | `performance.goals.set` | `{ tenant_id, goal_id, employee_id, cycle_id, goal_name, weight, due_date }` | Published when a goal is approved |
| `ReviewSubmitted` | `performance.review.submitted` | `{ tenant_id, review_id, employee_id, reviewer_id, review_type, cycle_id }` | Published when a review is submitted |
| `RatingFinalized` | `performance.rating.finalized` | `{ tenant_id, employee_id, cycle_id, rating_id, rating_label }` | Published when a performance rating is finalized |
| `PIPInitiated` | `performance.pip.initiated` | `{ tenant_id, plan_id, employee_id, manager_id, start_date, review_date, end_date }` | Published when a PIP is created |
| `PIPCompleted` | `performance.pip.completed` | `{ tenant_id, plan_id, employee_id, outcome }` | Published when a PIP is completed |
| `FeedbackGiven` | `performance.feedback.given` | `{ tenant_id, feedback_id, to_employee_id, feedback_type }` | Published when feedback is provided |
| `CycleFinalized` | `performance.cycle.finalized` | `{ tenant_id, cycle_id, total_reviews, average_rating }` | Published when cycle is closed |
