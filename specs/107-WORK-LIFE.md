# 107 - Work Life Service Specification

## 1. Domain Overview

The Work Life service provides employee well-being and engagement capabilities including physical, mental, and financial wellness resources, community volunteering opportunities, flexible benefits selection, wellness challenges, and work-life balance tracking. It supports configurable wellness programs with activity tracking, point-based rewards, and integration with benefits administration to promote holistic employee well-being.

**Bounded Context:** Employee Well-Being & Wellness
**Service Name:** `worklife-service`
**Database:** `data/worklife.db`
**HTTP Port:** 8144 | **gRPC Port:** 9144

---

## 2. Database Schema

### 2.1 Wellness Programs
```sql
CREATE TABLE wellness_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_name TEXT NOT NULL,
    program_type TEXT NOT NULL CHECK(program_type IN ('PHYSICAL','MENTAL','FINANCIAL','NUTRITIONAL','COMMUNITY','HOLISTIC')),
    description TEXT,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    enrollment_type TEXT NOT NULL DEFAULT 'OPEN' CHECK(enrollment_type IN ('OPEN','INVITATION','QUALIFYING')),
    max_participants INTEGER,
    enrolled_count INTEGER NOT NULL DEFAULT 0,
    points_per_activity INTEGER NOT NULL DEFAULT 100,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','PAUSED','COMPLETED','CANCELLED')),
    eligibility_criteria TEXT,  -- JSON: eligibility rules

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_name)
);

CREATE INDEX idx_programs_tenant_status ON wellness_programs(tenant_id, status);
CREATE INDEX idx_programs_tenant_type ON wellness_programs(tenant_id, program_type);
CREATE INDEX idx_programs_tenant_dates ON wellness_programs(tenant_id, start_date, end_date);
```

### 2.2 Wellness Enrollments
```sql
CREATE TABLE wellness_enrollments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    enrollment_date TEXT NOT NULL,
    opt_out_date TEXT,
    status TEXT NOT NULL DEFAULT 'ENROLLED' CHECK(status IN ('ENROLLED','ACTIVE','COMPLETED','OPTED_OUT')),
    total_points INTEGER NOT NULL DEFAULT 0,
    activities_completed INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES wellness_programs(id),
    UNIQUE(tenant_id, program_id, employee_id)
);

CREATE INDEX idx_enrollments_tenant_employee ON wellness_enrollments(tenant_id, employee_id);
CREATE INDEX idx_enrollments_tenant_program ON wellness_enrollments(tenant_id, program_id, status);
```

### 2.3 Wellness Activities
```sql
CREATE TABLE wellness_activities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    activity_name TEXT NOT NULL,
    activity_type TEXT NOT NULL CHECK(activity_type IN ('EXERCISE','MEDITATION','HEALTH_SCREENING','WORKSHOP','WEBINAR','VOLUNTEERING','STEP_CHALLENGE','JOURNALING','CUSTOM')),
    description TEXT,
    points_reward INTEGER NOT NULL DEFAULT 0,
    duration_minutes INTEGER,
    frequency TEXT NOT NULL DEFAULT 'ONE_TIME' CHECK(frequency IN ('ONE_TIME','DAILY','WEEKLY','MONTHLY')),
    proof_required INTEGER NOT NULL DEFAULT 0,
    external_link TEXT,
    start_date TEXT,
    end_date TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES wellness_programs(id) ON DELETE CASCADE
);

CREATE INDEX idx_activities_tenant_program ON wellness_activities(tenant_id, program_id);
CREATE INDEX idx_activities_tenant_type ON wellness_activities(tenant_id, activity_type, is_active);
```

### 2.4 Activity Logs
```sql
CREATE TABLE activity_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    enrollment_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    activity_id TEXT NOT NULL,
    log_date TEXT NOT NULL,
    duration_minutes INTEGER,
    notes TEXT,
    proof_reference TEXT,
    points_earned INTEGER NOT NULL DEFAULT 0,
    verified INTEGER NOT NULL DEFAULT 0,
    verified_by TEXT,
    verified_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','VERIFIED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (enrollment_id) REFERENCES wellness_enrollments(id),
    FOREIGN KEY (activity_id) REFERENCES wellness_activities(id)
);

CREATE INDEX idx_activitylogs_tenant_employee ON activity_logs(tenant_id, employee_id, log_date);
CREATE INDEX idx_activitylogs_tenant_program ON activity_logs(tenant_id, program_id);
CREATE INDEX idx_activitylogs_tenant_status ON activity_logs(tenant_id, status);
```

### 2.5 Wellness Challenges
```sql
CREATE TABLE wellness_challenges (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    challenge_name TEXT NOT NULL,
    challenge_type TEXT NOT NULL CHECK(challenge_type IN ('INDIVIDUAL','TEAM','DEPARTMENT')),
    metric_type TEXT NOT NULL CHECK(metric_type IN ('STEPS','MINUTES','ACTIVITIES','POINTS')),
    target_value INTEGER NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    participants_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','COMPLETED','CANCELLED')),
    rewards_config TEXT,  -- JSON: reward tiers

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, challenge_name)
);

CREATE INDEX idx_challenges_tenant_status ON wellness_challenges(tenant_id, status);
CREATE INDEX idx_challenges_tenant_type ON wellness_challenges(tenant_id, challenge_type);
CREATE INDEX idx_challenges_tenant_dates ON wellness_challenges(tenant_id, start_date, end_date);
```

### 2.6 Challenge Participants
```sql
CREATE TABLE challenge_participants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    challenge_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    team_name TEXT,
    current_value INTEGER NOT NULL DEFAULT 0,
    target_achieved INTEGER NOT NULL DEFAULT 0,
    rank INTEGER,
    joined_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (challenge_id) REFERENCES wellness_challenges(id),
    UNIQUE(tenant_id, challenge_id, employee_id)
);

CREATE INDEX idx_participants_tenant_challenge ON challenge_participants(tenant_id, challenge_id);
CREATE INDEX idx_participants_tenant_employee ON challenge_participants(tenant_id, employee_id);
CREATE INDEX idx_participants_tenant_rank ON challenge_participants(tenant_id, challenge_id, rank);
```

### 2.7 Work-Life Balance Assessments
```sql
CREATE TABLE work_life_balance_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    assessment_date TEXT NOT NULL,
    work_hours_per_week DECIMAL(4,1),
    overtime_hours DECIMAL(4,1),
    pto_taken_days INTEGER,
    pto_accrued_days INTEGER,
    flexibility_score DECIMAL(3,2) CHECK(flexibility_score BETWEEN 0 AND 10),
    satisfaction_score DECIMAL(3,2) CHECK(satisfaction_score BETWEEN 0 AND 10),
    burnout_risk TEXT NOT NULL DEFAULT 'LOW' CHECK(burnout_risk IN ('LOW','MODERATE','HIGH','CRITICAL')),
    recommendations TEXT,  -- JSON array of recommendations
    is_anonymous INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_assessments_tenant_employee ON work_life_balance_assessments(tenant_id, employee_id, assessment_date);
CREATE INDEX idx_assessments_tenant_risk ON work_life_balance_assessments(tenant_id, burnout_risk);
CREATE INDEX idx_assessments_tenant_date ON work_life_balance_assessments(tenant_id, assessment_date);
```

### 2.8 Wellness Resources
```sql
CREATE TABLE wellness_resources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_name TEXT NOT NULL,
    resource_type TEXT NOT NULL CHECK(resource_type IN ('ARTICLE','VIDEO','PODCAST','TOOL','EAP','HELPLINE','WORKSHOP')),
    category TEXT,
    description TEXT,
    external_url TEXT,
    is_featured INTEGER NOT NULL DEFAULT 0,
    view_count INTEGER NOT NULL DEFAULT 0,
    target_audience TEXT,  -- JSON: audience filter criteria

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_resources_tenant_type ON wellness_resources(tenant_id, resource_type);
CREATE INDEX idx_resources_tenant_featured ON wellness_resources(tenant_id, is_featured);
```

---

## 3. REST API Endpoints

### Programs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/programs` | Create a wellness program |
| GET | `/api/v1/programs` | List wellness programs |
| GET | `/api/v1/programs/{id}` | Get program details |
| PUT | `/api/v1/programs/{id}` | Update program |
| POST | `/api/v1/programs/{id}/enroll` | Enroll employee in program |
| POST | `/api/v1/programs/{id}/opt-out` | Opt out of program |

### Activities
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/activities` | Create a wellness activity |
| GET | `/api/v1/activities` | List activities |
| GET | `/api/v1/activities/{id}` | Get activity details |
| PUT | `/api/v1/activities/{id}` | Update activity |
| GET | `/api/v1/programs/{id}/activities` | List activities by program |

### Activity Logs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/activity-logs` | Log a completed activity |
| GET | `/api/v1/activity-logs` | Get activity log history |
| POST | `/api/v1/activity-logs/{id}/verify` | Verify an activity log |

### Challenges
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/challenges` | Create a wellness challenge |
| GET | `/api/v1/challenges` | List challenges |
| GET | `/api/v1/challenges/{id}` | Get challenge details |
| PUT | `/api/v1/challenges/{id}` | Update challenge |
| POST | `/api/v1/challenges/{id}/join` | Join a challenge |
| GET | `/api/v1/challenges/{id}/leaderboard` | Get challenge leaderboard |
| GET | `/api/v1/challenges/{id}/progress` | Get challenge progress |

### Balance Assessments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/assessments` | Submit a work-life balance assessment |
| GET | `/api/v1/assessments/trends` | Get assessment trends |
| GET | `/api/v1/assessments/recommendations` | Get personalized recommendations |

### Resources
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/resources` | Create a wellness resource |
| GET | `/api/v1/resources` | List wellness resources |
| GET | `/api/v1/resources/{id}` | Get resource details |
| PUT | `/api/v1/resources/{id}` | Update resource |
| POST | `/api/v1/resources/{id}/track-view` | Track resource view |
| GET | `/api/v1/resources/featured` | Get featured resources |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/analytics/program-metrics` | Get program participation metrics |
| GET | `/api/v1/analytics/participation-stats` | Get overall participation statistics |
| GET | `/api/v1/analytics/wellness-dashboard` | Get wellness dashboard data |

---

## 4. Business Rules

1. An employee MUST NOT be enrolled in a program that has reached its `max_participants` limit.
2. Activity log points MUST NOT be awarded until the log is verified when `proof_required` is set to 1 on the activity.
3. Programs with enrollment_type QUALIFYING MUST validate employee eligibility against `eligibility_criteria` before enrollment is permitted.
4. An employee MAY opt out of a program at any time, but earned points MUST be retained in their total.
5. Challenge participant rankings MUST be recalculated whenever a participant's `current_value` is updated.
6. Work-life balance assessments with `is_anonymous` set to 1 MUST strip employee identity before any aggregated reporting.
7. The system MUST trigger a BalanceAlertTriggered event when burnout_risk reaches CRITICAL level.
8. Wellness resources MUST be filtered by `target_audience` criteria so that employees only see resources relevant to their profile.
9. Activity logs for ONE_TIME activities MUST be rejected if the employee has already logged completion for that activity.
10. The `enrolled_count` on wellness programs MUST be maintained as a denormalized counter and MUST be reconciled against actual enrollment records daily.
11. Challenge rewards MUST be distributed only after the challenge status transitions to COMPLETED and all activity logs have been verified.
12. Assessment scores MUST be calculated using a consistent methodology and SHOULD benchmark against organizational averages.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package worklife.v1;

service WorkLifeService {
    // Programs
    rpc CreateProgram(CreateProgramRequest) returns (CreateProgramResponse);
    rpc GetProgram(GetProgramRequest) returns (GetProgramResponse);
    rpc ListPrograms(ListProgramsRequest) returns (ListProgramsResponse);
    rpc UpdateProgram(UpdateProgramRequest) returns (UpdateProgramResponse);
    rpc EnrollInProgram(EnrollInProgramRequest) returns (EnrollInProgramResponse);
    rpc OptOutOfProgram(OptOutOfProgramRequest) returns (OptOutOfProgramResponse);

    // Activities
    rpc CreateActivity(CreateActivityRequest) returns (CreateActivityResponse);
    rpc ListActivities(ListActivitiesRequest) returns (ListActivitiesResponse);

    // Activity Logs
    rpc LogActivity(LogActivityRequest) returns (LogActivityResponse);
    rpc ListActivityLogs(ListActivityLogsRequest) returns (ListActivityLogsResponse);
    rpc VerifyActivityLog(VerifyActivityLogRequest) returns (VerifyActivityLogResponse);

    // Challenges
    rpc CreateChallenge(CreateChallengeRequest) returns (CreateChallengeResponse);
    rpc GetChallenge(GetChallengeRequest) returns (GetChallengeResponse);
    rpc ListChallenges(ListChallengesRequest) returns (ListChallengesResponse);
    rpc JoinChallenge(JoinChallengeRequest) returns (JoinChallengeResponse);
    rpc GetLeaderboard(GetLeaderboardRequest) returns (GetLeaderboardResponse);
    rpc GetChallengeProgress(GetChallengeProgressRequest) returns (GetChallengeProgressResponse);

    // Assessments
    rpc SubmitAssessment(SubmitAssessmentRequest) returns (SubmitAssessmentResponse);
    rpc GetAssessmentTrends(GetAssessmentTrendsRequest) returns (GetAssessmentTrendsResponse);
    rpc GetRecommendations(GetRecommendationsRequest) returns (GetRecommendationsResponse);

    // Resources
    rpc CreateResource(CreateResourceRequest) returns (CreateResourceResponse);
    rpc ListResources(ListResourcesRequest) returns (ListResourcesResponse);
    rpc TrackResourceView(TrackResourceViewRequest) returns (TrackResourceViewResponse);

    // Analytics
    rpc GetProgramMetrics(GetProgramMetricsRequest) returns (GetProgramMetricsResponse);
    rpc GetWellnessDashboard(GetWellnessDashboardRequest) returns (GetWellnessDashboardResponse);
}

message WellnessProgram {
    string id = 1;
    string tenant_id = 2;
    string program_name = 3;
    string program_type = 4;
    string description = 5;
    string start_date = 6;
    string end_date = 7;
    string enrollment_type = 8;
    int32 max_participants = 9;
    int32 enrolled_count = 10;
    int32 points_per_activity = 11;
    string status = 12;
}

message CreateProgramRequest {
    string tenant_id = 1;
    string program_name = 2;
    string program_type = 3;
    string description = 4;
    string start_date = 5;
    string end_date = 6;
    string enrollment_type = 7;
    int32 max_participants = 8;
    int32 points_per_activity = 9;
    string created_by = 10;
}

message CreateProgramResponse {
    WellnessProgram program = 1;
}

message GetProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
}

message GetProgramResponse {
    WellnessProgram program = 1;
}

message ListProgramsRequest {
    string tenant_id = 1;
    string program_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListProgramsResponse {
    repeated WellnessProgram programs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
    string program_name = 3;
    string description = 4;
    string status = 5;
    string updated_by = 6;
}

message UpdateProgramResponse {
    WellnessProgram program = 1;
}

message EnrollInProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
    string employee_id = 3;
    string enrolled_by = 4;
}

message EnrollInProgramResponse {
    string enrollment_id = 1;
    string status = 2;
}

message OptOutOfProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
    string employee_id = 3;
}

message OptOutOfProgramResponse {
    string enrollment_id = 1;
    string status = 2;
}

message WellnessActivity {
    string id = 1;
    string tenant_id = 2;
    string program_id = 3;
    string activity_name = 4;
    string activity_type = 5;
    int32 points_reward = 6;
    int32 duration_minutes = 7;
    string frequency = 8;
    bool proof_required = 9;
    bool is_active = 10;
}

message CreateActivityRequest {
    string tenant_id = 1;
    string program_id = 2;
    string activity_name = 3;
    string activity_type = 4;
    int32 points_reward = 5;
    int32 duration_minutes = 6;
    string frequency = 7;
    bool proof_required = 8;
    string created_by = 9;
}

message CreateActivityResponse {
    WellnessActivity activity = 1;
}

message ListActivitiesRequest {
    string tenant_id = 1;
    string program_id = 2;
    string activity_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListActivitiesResponse {
    repeated WellnessActivity activities = 1;
    string next_page_token = 2;
}

message ActivityLog {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string program_id = 4;
    string activity_id = 5;
    string log_date = 6;
    int32 duration_minutes = 7;
    int32 points_earned = 8;
    string status = 9;
}

message LogActivityRequest {
    string tenant_id = 1;
    string program_id = 2;
    string activity_id = 3;
    string employee_id = 4;
    string log_date = 5;
    int32 duration_minutes = 6;
    string notes = 7;
    string proof_reference = 8;
}

message LogActivityResponse {
    ActivityLog log = 1;
}

message ListActivityLogsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string program_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListActivityLogsResponse {
    repeated ActivityLog logs = 1;
    string next_page_token = 2;
}

message VerifyActivityLogRequest {
    string tenant_id = 1;
    string log_id = 2;
    bool verified = 3;
    string verified_by = 4;
}

message VerifyActivityLogResponse {
    ActivityLog log = 1;
}

message Challenge {
    string id = 1;
    string tenant_id = 2;
    string challenge_name = 3;
    string challenge_type = 4;
    string metric_type = 5;
    int64 target_value = 6;
    string start_date = 7;
    string end_date = 8;
    int32 participants_count = 9;
    string status = 10;
}

message CreateChallengeRequest {
    string tenant_id = 1;
    string challenge_name = 2;
    string challenge_type = 3;
    string metric_type = 4;
    int64 target_value = 5;
    string start_date = 6;
    string end_date = 7;
    string rewards_config = 8;
    string created_by = 9;
}

message CreateChallengeResponse {
    Challenge challenge = 1;
}

message GetChallengeRequest {
    string tenant_id = 1;
    string challenge_id = 2;
}

message GetChallengeResponse {
    Challenge challenge = 1;
}

message ListChallengesRequest {
    string tenant_id = 1;
    string challenge_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListChallengesResponse {
    repeated Challenge challenges = 1;
    string next_page_token = 2;
}

message JoinChallengeRequest {
    string tenant_id = 1;
    string challenge_id = 2;
    string employee_id = 3;
    string team_name = 4;
}

message JoinChallengeResponse {
    string participant_id = 1;
}

message LeaderboardEntry {
    string employee_id = 1;
    string team_name = 2;
    int64 current_value = 3;
    int32 rank = 4;
    bool target_achieved = 5;
}

message GetLeaderboardRequest {
    string tenant_id = 1;
    string challenge_id = 2;
    int32 page_size = 3;
}

message GetLeaderboardResponse {
    repeated LeaderboardEntry entries = 1;
}

message ChallengeProgress {
    string challenge_id = 1;
    int64 current_value = 2;
    int64 target_value = 3;
    double completion_pct = 4;
    int32 rank = 5;
}

message GetChallengeProgressRequest {
    string tenant_id = 1;
    string challenge_id = 2;
    string employee_id = 3;
}

message GetChallengeProgressResponse {
    ChallengeProgress progress = 1;
}

message BalanceAssessment {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string assessment_date = 4;
    double work_hours_per_week = 5;
    double overtime_hours = 6;
    int32 pto_taken_days = 7;
    double flexibility_score = 8;
    double satisfaction_score = 9;
    string burnout_risk = 10;
}

message SubmitAssessmentRequest {
    string tenant_id = 1;
    string employee_id = 2;
    double work_hours_per_week = 3;
    double overtime_hours = 4;
    double flexibility_score = 5;
    double satisfaction_score = 6;
    bool is_anonymous = 7;
}

message SubmitAssessmentResponse {
    BalanceAssessment assessment = 1;
}

message AssessmentTrend {
    string assessment_date = 1;
    double avg_satisfaction_score = 2;
    double avg_flexibility_score = 3;
    int32 critical_count = 4;
}

message GetAssessmentTrendsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetAssessmentTrendsResponse {
    repeated AssessmentTrend trends = 1;
}

message Recommendation {
    string recommendation_id = 1;
    string title = 2;
    string description = 3;
    string category = 4;
}

message GetRecommendationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string burnout_risk = 3;
}

message GetRecommendationsResponse {
    repeated Recommendation recommendations = 1;
}

message WellnessResource {
    string id = 1;
    string tenant_id = 2;
    string resource_name = 3;
    string resource_type = 4;
    string category = 5;
    string external_url = 6;
    bool is_featured = 7;
    int32 view_count = 8;
}

message CreateResourceRequest {
    string tenant_id = 1;
    string resource_name = 2;
    string resource_type = 3;
    string category = 4;
    string description = 5;
    string external_url = 6;
    string target_audience = 7;
    string created_by = 8;
}

message CreateResourceResponse {
    WellnessResource resource = 1;
}

message ListResourcesRequest {
    string tenant_id = 1;
    string resource_type = 2;
    string category = 3;
    bool featured_only = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListResourcesResponse {
    repeated WellnessResource resources = 1;
    string next_page_token = 2;
}

message TrackResourceViewRequest {
    string tenant_id = 1;
    string resource_id = 2;
    string employee_id = 3;
}

message TrackResourceViewResponse {
    int32 view_count = 1;
}

message ProgramMetrics {
    string program_id = 1;
    string program_name = 2;
    int32 enrolled_count = 3;
    int32 activities_completed = 4;
    double avg_points_per_participant = 5;
    double completion_rate = 6;
}

message GetProgramMetricsRequest {
    string tenant_id = 1;
    string program_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetProgramMetricsResponse {
    repeated ProgramMetrics metrics = 1;
}

message WellnessDashboardData {
    int32 active_programs = 1;
    int32 total_participants = 2;
    int32 active_challenges = 3;
    double org_avg_satisfaction = 4;
    int32 critical_burnout_count = 5;
}

message GetWellnessDashboardRequest {
    string tenant_id = 1;
}

message GetWellnessDashboardResponse {
    WellnessDashboardData data = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee records, department assignments | Validate enrollment eligibility and audience targeting |
| `benefits-service` | Benefit plan elections, flexible spending info | Correlate wellness participation with benefit incentives |
| `time-service` | Time-off balances, work hours, overtime | Feed work-life balance assessments |
| `notification-service` | Communication preferences | Deliver wellness program communications |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `notification-service` | Enrollment confirmations, challenge updates, burnout alerts | Push wellness notifications |
| `benefits-service` | Wellness incentive completions, point totals | Apply wellness benefit credits |
| `reporting-service` | Program metrics, participation stats, assessment trends | Well-being analytics dashboards |
| `manageredge-service` | Team wellness summaries, burnout risk alerts | Manager awareness of team well-being |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ProgramEnrolled` | `worklife.program.enrolled` | `{ tenant_id, program_id, employee_id, enrollment_date, program_type }` | Published when an employee enrolls in a wellness program |
| `ActivityLogged` | `worklife.activity.logged` | `{ tenant_id, employee_id, program_id, activity_id, log_id, points_earned, log_date }` | Published when an activity completion is logged |
| `ChallengeJoined` | `worklife.challenge.joined` | `{ tenant_id, challenge_id, employee_id, team_name, challenge_type }` | Published when an employee joins a challenge |
| `WellnessAssessmentCompleted` | `worklife.assessment.completed` | `{ tenant_id, employee_id, assessment_id, burnout_risk, satisfaction_score, is_anonymous }` | Published when a work-life balance assessment is submitted |
| `BalanceAlertTriggered` | `worklife.balance.alert` | `{ tenant_id, employee_id, assessment_id, burnout_risk, flexibility_score, satisfaction_score }` | Published when burnout risk reaches HIGH or CRITICAL level |
