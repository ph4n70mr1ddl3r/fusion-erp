# 95 - Employee Experience Service Specification

## 1. Domain Overview

The Employee Experience service (My Experience/Oracle ME) provides an employee engagement platform with pulse surveys, check-ins, well-being tracking, sentiment analysis, personalized recommendations, and experience analytics. It supports continuous listening strategies, action planning, and manager dashboards to drive organizational engagement and retention.

**Bounded Context:** Employee Engagement & Experience
**Service Name:** `experience-service`
**Database:** `data/experience.db`
**HTTP Port:** 8132 | **gRPC Port:** 9132

---

## 2. Database Schema

### 2.1 Engagement Surveys
```sql
CREATE TABLE engagement_surveys (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    survey_type TEXT NOT NULL CHECK(survey_type IN ('PULSE','ANNUAL','ONBOARDING','EXIT','CUSTOM')),
    frequency TEXT NOT NULL CHECK(frequency IN ('WEEKLY','MONTHLY','QUARTERLY','ANNUAL','ONE_TIME')),
    questions TEXT NOT NULL,  -- JSON array of question objects
    start_date TEXT NOT NULL,
    end_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','PAUSED','CLOSED')),
    anonymity_level TEXT NOT NULL DEFAULT 'FULL_ANONYMOUS' CHECK(anonymity_level IN ('FULL_ANONYMOUS','PARTIALLY_ANONYMOUS','NAMED')),
    target_audience TEXT,  -- JSON: department_ids, location_ids, employee_ids
    response_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_surveys_tenant_status ON engagement_surveys(tenant_id, status);
CREATE INDEX idx_surveys_tenant_type ON engagement_surveys(tenant_id, survey_type);
CREATE INDEX idx_surveys_tenant_dates ON engagement_surveys(tenant_id, start_date, end_date);
```

### 2.2 Survey Responses
```sql
CREATE TABLE survey_responses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    survey_id TEXT NOT NULL,
    respondent_id TEXT NOT NULL,
    responses TEXT NOT NULL,  -- JSON: array of { question_id, answer, answer_type }
    submitted_at TEXT NOT NULL,
    is_anonymous INTEGER NOT NULL DEFAULT 0,
    sentiment_score DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (survey_id) REFERENCES engagement_surveys(id) ON DELETE CASCADE
);

CREATE INDEX idx_survey_responses_tenant_survey ON survey_responses(tenant_id, survey_id);
CREATE INDEX idx_survey_responses_tenant_respondent ON survey_responses(tenant_id, respondent_id);
CREATE INDEX idx_survey_responses_tenant_date ON survey_responses(tenant_id, submitted_at);
```

### 2.3 Employee Check-ins
```sql
CREATE TABLE employee_checkins (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    checkin_type TEXT NOT NULL CHECK(checkin_type IN ('WEEKLY','MONTHLY','QUARTERLY','ADHOC')),
    discussion_topics TEXT,  -- JSON array of topic strings
    employee_mood INTEGER CHECK(employee_mood BETWEEN 1 AND 5),
    notes TEXT,
    action_items TEXT,  -- JSON array of action item objects
    follow_up_date TEXT,
    status TEXT NOT NULL DEFAULT 'SCHEDULED' CHECK(status IN ('SCHEDULED','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_checkins_tenant_emp ON employee_checkins(tenant_id, employee_id);
CREATE INDEX idx_checkins_tenant_mgr ON employee_checkins(tenant_id, manager_id);
CREATE INDEX idx_checkins_tenant_status ON employee_checkins(tenant_id, status);
CREATE INDEX idx_checkins_tenant_date ON employee_checkins(tenant_id, follow_up_date);
```

### 2.4 Wellbeing Assessments
```sql
CREATE TABLE wellbeing_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    assessment_date TEXT NOT NULL,
    work_life_balance_score DECIMAL(5,2),
    stress_level TEXT CHECK(stress_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    engagement_score DECIMAL(5,2),
    burnout_risk TEXT CHECK(burnout_risk IN ('LOW','MEDIUM','HIGH')),
    recommendations TEXT,  -- JSON array of recommendation strings
    is_anonymous INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_wellbeing_tenant_emp ON wellbeing_assessments(tenant_id, employee_id);
CREATE INDEX idx_wellbeing_tenant_date ON wellbeing_assessments(tenant_id, assessment_date);
CREATE INDEX idx_wellbeing_tenant_stress ON wellbeing_assessments(tenant_id, stress_level);
CREATE INDEX idx_wellbeing_tenant_burnout ON wellbeing_assessments(tenant_id, burnout_risk);
```

### 2.5 Action Plans
```sql
CREATE TABLE action_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    title TEXT NOT NULL,
    target_metric TEXT NOT NULL,
    current_value DECIMAL(5,2),
    target_value DECIMAL(5,2),
    owner_id TEXT NOT NULL,
    department_id TEXT,
    start_date TEXT NOT NULL,
    end_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_action_plans_tenant_owner ON action_plans(tenant_id, owner_id);
CREATE INDEX idx_action_plans_tenant_dept ON action_plans(tenant_id, department_id);
CREATE INDEX idx_action_plans_tenant_status ON action_plans(tenant_id, status);
```

### 2.6 Experience Actions
```sql
CREATE TABLE experience_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    action_plan_id TEXT NOT NULL,
    assignee_id TEXT NOT NULL,
    action_type TEXT NOT NULL,
    description TEXT NOT NULL,
    priority TEXT NOT NULL DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    due_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','OVERDUE','CANCELLED')),
    impact_score DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (action_plan_id) REFERENCES action_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_exp_actions_tenant_plan ON experience_actions(tenant_id, action_plan_id);
CREATE INDEX idx_exp_actions_tenant_assignee ON experience_actions(tenant_id, assignee_id);
CREATE INDEX idx_exp_actions_tenant_status ON experience_actions(tenant_id, status);
CREATE INDEX idx_exp_actions_tenant_due ON experience_actions(tenant_id, due_date);
```

### 2.7 Experience Recommendations
```sql
CREATE TABLE experience_recommendations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    recommendation_type TEXT NOT NULL CHECK(recommendation_type IN ('WELLBEING','DEVELOPMENT','ENGAGEMENT','RECOGNITION')),
    title TEXT NOT NULL,
    description TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    source TEXT NOT NULL CHECK(source IN ('AI_INFERRED','MANAGER_SUGGESTED','SYSTEM')),
    dismissed INTEGER NOT NULL DEFAULT 0,
    acted_on INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_exp_recs_tenant_emp ON experience_recommendations(tenant_id, employee_id);
CREATE INDEX idx_exp_recs_tenant_type ON experience_recommendations(tenant_id, recommendation_type);
CREATE INDEX idx_exp_recs_tenant_source ON experience_recommendations(tenant_id, source);
```

---

## 3. REST API Endpoints

### Surveys
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/surveys` | Create an engagement survey |
| GET | `/api/v1/surveys` | List surveys with filters |
| GET | `/api/v1/surveys/{id}` | Get survey details |
| PUT | `/api/v1/surveys/{id}` | Update survey |
| POST | `/api/v1/surveys/{id}/publish` | Publish/activate survey |
| POST | `/api/v1/surveys/{id}/pause` | Pause active survey |
| POST | `/api/v1/surveys/{id}/close` | Close survey |
| POST | `/api/v1/surveys/{id}/respond` | Submit survey response |
| GET | `/api/v1/surveys/{id}/results` | Get aggregated survey results |

### Check-ins
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/checkins` | Create a check-in |
| GET | `/api/v1/checkins` | List check-ins with filters |
| GET | `/api/v1/checkins/{id}` | Get check-in details |
| PUT | `/api/v1/checkins/{id}` | Update check-in |
| POST | `/api/v1/checkins/{id}/complete` | Complete a check-in |
| POST | `/api/v1/checkins/{id}/cancel` | Cancel a scheduled check-in |
| POST | `/api/v1/checkins/schedule` | Schedule recurring check-ins |

### Wellbeing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/wellbeing/assess` | Submit wellbeing assessment |
| GET | `/api/v1/wellbeing/employees/{id}` | Get employee wellbeing history |
| GET | `/api/v1/wellbeing/trends` | Get organizational wellbeing trends |
| GET | `/api/v1/wellbeing/alerts` | Get active wellbeing alerts |

### Action Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/action-plans` | Create action plan |
| GET | `/api/v1/action-plans` | List action plans |
| GET | `/api/v1/action-plans/{id}` | Get action plan details |
| PUT | `/api/v1/action-plans/{id}` | Update action plan |
| POST | `/api/v1/action-plans/{id}/actions` | Add action to plan |
| PUT | `/api/v1/action-plans/{id}/actions/{actionId}` | Update action status |
| POST | `/api/v1/action-plans/{id}/activate` | Activate action plan |

### Recommendations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/employees/{id}/recommendations` | Get recommendations for employee |
| POST | `/api/v1/recommendations/{id}/dismiss` | Dismiss a recommendation |
| POST | `/api/v1/recommendations/{id}/act` | Mark recommendation as acted on |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/analytics/engagement-scores` | Get engagement score trends |
| GET | `/api/v1/analytics/sentiment-trends` | Get sentiment analysis trends |
| GET | `/api/v1/analytics/wellbeing-dashboard` | Get wellbeing dashboard data |
| GET | `/api/v1/analytics/checkin-summary` | Get check-in activity summary |
| GET | `/api/v1/analytics/survey-response-rates` | Get survey response rate metrics |

---

## 4. Business Rules

1. Surveys with `FULL_ANONYMOUS` anonymity level MUST NOT store or expose respondent identity in any analytics output.
2. Pulse surveys MUST NOT be scheduled more frequently than the tenant-configured minimum interval (default: weekly).
3. An employee MUST NOT submit more than one response to the same survey instance.
4. Wellbeing assessments with stress_level CRITICAL or burnout_risk HIGH MUST trigger an alert to the designated HR representative within 24 hours.
5. Action plan actions MUST have a due date no later than the parent action plan's end date.
6. Sentiment scores on survey responses MUST be calculated using the configured sentiment analysis engine and MUST NOT be manually overridden.
7. AI-inferred recommendations MUST be clearly labeled with their source to distinguish them from manager-suggested items.
8. Check-ins with employee_mood values of 1 or 2 (out of 5) MUST trigger a follow-up notification to the manager within 48 hours.
9. Survey results MUST NOT be made available for aggregation until a minimum response threshold (configurable per tenant, default 5 responses) is met to protect anonymity.
10. Experience actions that pass their due date without completion MUST be automatically marked as OVERDUE.
11. The system MUST NOT allow wellbeing assessments for the same employee more than once per calendar day.
12. Managers MUST NOT view individual responses for anonymous surveys, even for their direct reports.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package experience.v1;

service EmployeeExperienceService {
    // Surveys
    rpc CreateSurvey(CreateSurveyRequest) returns (CreateSurveyResponse);
    rpc GetSurvey(GetSurveyRequest) returns (GetSurveyResponse);
    rpc ListSurveys(ListSurveysRequest) returns (ListSurveysResponse);
    rpc PublishSurvey(PublishSurveyRequest) returns (PublishSurveyResponse);
    rpc CloseSurvey(CloseSurveyRequest) returns (CloseSurveyResponse);
    rpc SubmitSurveyResponse(SubmitSurveyResponseRequest) returns (SubmitSurveyResponseResponse);
    rpc GetSurveyResults(GetSurveyResultsRequest) returns (GetSurveyResultsResponse);

    // Check-ins
    rpc CreateCheckin(CreateCheckinRequest) returns (CreateCheckinResponse);
    rpc GetCheckin(GetCheckinRequest) returns (GetCheckinResponse);
    rpc ListCheckins(ListCheckinsRequest) returns (ListCheckinsResponse);
    rpc CompleteCheckin(CompleteCheckinRequest) returns (CompleteCheckinResponse);

    // Wellbeing
    rpc AssessWellbeing(AssessWellbeingRequest) returns (AssessWellbeingResponse);
    rpc GetWellbeingHistory(GetWellbeingHistoryRequest) returns (GetWellbeingHistoryResponse);
    rpc GetWellbeingTrends(GetWellbeingTrendsRequest) returns (GetWellbeingTrendsResponse);

    // Action Plans
    rpc CreateActionPlan(CreateActionPlanRequest) returns (CreateActionPlanResponse);
    rpc GetActionPlan(GetActionPlanRequest) returns (GetActionPlanResponse);
    rpc ListActionPlans(ListActionPlansRequest) returns (ListActionPlansResponse);
    rpc UpdateAction(UpdateActionRequest) returns (UpdateActionResponse);

    // Recommendations
    rpc GetRecommendations(GetRecommendationsRequest) returns (GetRecommendationsResponse);
    rpc DismissRecommendation(DismissRecommendationRequest) returns (DismissRecommendationResponse);
    rpc ActOnRecommendation(ActOnRecommendationRequest) returns (ActOnRecommendationResponse);

    // Analytics
    rpc GetEngagementScores(GetEngagementScoresRequest) returns (GetEngagementScoresResponse);
    rpc GetSentimentTrends(GetSentimentTrendsRequest) returns (GetSentimentTrendsResponse);
    rpc GetWellbeingDashboard(GetWellbeingDashboardRequest) returns (GetWellbeingDashboardResponse);
}

message EngagementSurvey {
    string id = 1;
    string tenant_id = 2;
    string title = 3;
    string description = 4;
    string survey_type = 5;
    string frequency = 6;
    string questions = 7;
    string start_date = 8;
    string end_date = 9;
    string status = 10;
    string anonymity_level = 11;
    string target_audience = 12;
    int32 response_count = 13;
}

message CreateSurveyRequest {
    string tenant_id = 1;
    string title = 2;
    string description = 3;
    string survey_type = 4;
    string frequency = 5;
    string questions = 6;
    string start_date = 7;
    string end_date = 8;
    string anonymity_level = 9;
    string target_audience = 10;
    string created_by = 11;
}

message CreateSurveyResponse {
    EngagementSurvey survey = 1;
}

message GetSurveyRequest {
    string tenant_id = 1;
    string survey_id = 2;
}

message GetSurveyResponse {
    EngagementSurvey survey = 1;
}

message ListSurveysRequest {
    string tenant_id = 1;
    string status = 2;
    string survey_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListSurveysResponse {
    repeated EngagementSurvey surveys = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishSurveyRequest {
    string tenant_id = 1;
    string survey_id = 2;
    string updated_by = 3;
}

message PublishSurveyResponse {
    EngagementSurvey survey = 1;
}

message CloseSurveyRequest {
    string tenant_id = 1;
    string survey_id = 2;
    string updated_by = 3;
}

message CloseSurveyResponse {
    EngagementSurvey survey = 1;
}

message SubmitSurveyResponseRequest {
    string tenant_id = 1;
    string survey_id = 2;
    string respondent_id = 3;
    string responses = 4;
    bool is_anonymous = 5;
}

message SubmitSurveyResponseResponse {
    string response_id = 1;
    double sentiment_score = 2;
}

message SurveyResultAggregate {
    string question_id = 1;
    double average_score = 2;
    int32 response_count = 3;
    double sentiment_average = 4;
}

message GetSurveyResultsRequest {
    string tenant_id = 1;
    string survey_id = 2;
}

message GetSurveyResultsResponse {
    repeated SurveyResultAggregate results = 1;
    int32 total_responses = 2;
    double overall_sentiment = 3;
}

message Checkin {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string manager_id = 4;
    string checkin_type = 5;
    string discussion_topics = 6;
    int32 employee_mood = 7;
    string notes = 8;
    string action_items = 9;
    string follow_up_date = 10;
    string status = 11;
}

message CreateCheckinRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string manager_id = 3;
    string checkin_type = 4;
    string discussion_topics = 5;
    string follow_up_date = 6;
    string created_by = 7;
}

message CreateCheckinResponse {
    Checkin checkin = 1;
}

message GetCheckinRequest {
    string tenant_id = 1;
    string checkin_id = 2;
}

message GetCheckinResponse {
    Checkin checkin = 1;
}

message ListCheckinsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string manager_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListCheckinsResponse {
    repeated Checkin checkins = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CompleteCheckinRequest {
    string tenant_id = 1;
    string checkin_id = 2;
    int32 employee_mood = 3;
    string notes = 4;
    string action_items = 5;
    string updated_by = 6;
}

message CompleteCheckinResponse {
    Checkin checkin = 1;
}

message WellbeingAssessment {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string assessment_date = 4;
    double work_life_balance_score = 5;
    string stress_level = 6;
    double engagement_score = 7;
    string burnout_risk = 8;
    string recommendations = 9;
}

message AssessWellbeingRequest {
    string tenant_id = 1;
    string employee_id = 2;
    double work_life_balance_score = 3;
    string stress_level = 4;
    double engagement_score = 5;
    string burnout_risk = 6;
    string created_by = 7;
}

message AssessWellbeingResponse {
    WellbeingAssessment assessment = 1;
}

message GetWellbeingHistoryRequest {
    string tenant_id = 1;
    string employee_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetWellbeingHistoryResponse {
    repeated WellbeingAssessment assessments = 1;
    string next_page_token = 2;
}

message WellbeingTrend {
    string period = 1;
    double avg_work_life_balance = 2;
    double avg_engagement = 3;
    string dominant_stress_level = 4;
}

message GetWellbeingTrendsRequest {
    string tenant_id = 1;
    string department_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetWellbeingTrendsResponse {
    repeated WellbeingTrend trends = 1;
}

message ActionPlan {
    string id = 1;
    string tenant_id = 2;
    string title = 3;
    string target_metric = 4;
    double current_value = 5;
    double target_value = 6;
    string owner_id = 7;
    string department_id = 8;
    string status = 9;
    string start_date = 10;
    string end_date = 11;
}

message CreateActionPlanRequest {
    string tenant_id = 1;
    string title = 2;
    string target_metric = 3;
    double current_value = 4;
    double target_value = 5;
    string owner_id = 6;
    string department_id = 7;
    string start_date = 8;
    string end_date = 9;
    string created_by = 10;
}

message CreateActionPlanResponse {
    ActionPlan action_plan = 1;
}

message GetActionPlanRequest {
    string tenant_id = 1;
    string action_plan_id = 2;
}

message GetActionPlanResponse {
    ActionPlan action_plan = 1;
}

message ListActionPlansRequest {
    string tenant_id = 1;
    string owner_id = 2;
    string department_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListActionPlansResponse {
    repeated ActionPlan action_plans = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateActionRequest {
    string tenant_id = 1;
    string action_id = 2;
    string status = 3;
    double impact_score = 4;
    string updated_by = 5;
}

message UpdateActionResponse {
    string action_id = 1;
    string status = 2;
}

message ExperienceRecommendation {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string recommendation_type = 4;
    string title = 5;
    string description = 6;
    string priority = 7;
    string source = 8;
    bool dismissed = 9;
    bool acted_on = 10;
}

message GetRecommendationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string recommendation_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetRecommendationsResponse {
    repeated ExperienceRecommendation recommendations = 1;
    string next_page_token = 2;
}

message DismissRecommendationRequest {
    string tenant_id = 1;
    string recommendation_id = 2;
    string updated_by = 3;
}

message DismissRecommendationResponse {
    ExperienceRecommendation recommendation = 1;
}

message ActOnRecommendationRequest {
    string tenant_id = 1;
    string recommendation_id = 2;
    string updated_by = 3;
}

message ActOnRecommendationResponse {
    ExperienceRecommendation recommendation = 1;
}

message EngagementScore {
    string period = 1;
    double overall_score = 2;
    double survey_participation = 3;
    double checkin_frequency = 4;
    double wellbeing_average = 5;
}

message GetEngagementScoresRequest {
    string tenant_id = 1;
    string department_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetEngagementScoresResponse {
    repeated EngagementScore scores = 1;
}

message SentimentTrend {
    string period = 1;
    double positive_percentage = 2;
    double neutral_percentage = 3;
    double negative_percentage = 4;
    double overall_sentiment = 5;
}

message GetSentimentTrendsRequest {
    string tenant_id = 1;
    string department_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetSentimentTrendsResponse {
    repeated SentimentTrend trends = 1;
}

message WellbeingDashboardData {
    double avg_engagement_score = 1;
    double avg_work_life_balance = 2;
    int32 critical_stress_count = 3;
    int32 high_burnout_count = 4;
    int32 total_assessments = 5;
}

message GetWellbeingDashboardRequest {
    string tenant_id = 1;
    string department_id = 2;
}

message GetWellbeingDashboardResponse {
    WellbeingDashboardData dashboard = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee profiles, manager relationships, org structure | Identify survey target audiences and check-in participants |
| `skills-service` | Employee skill profiles, skill gaps | Generate development recommendations |
| `notification-service` | Notification delivery | Send survey invitations, check-in reminders, and wellbeing alerts |
| `learning-service` | Learning activity data | Correlate learning with engagement and wellbeing trends |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `notification-service` | Survey invitations, wellbeing alerts, check-in reminders | Deliver timely notifications to employees and managers |
| `reporting-service` | Engagement metrics, sentiment data, wellbeing scores | Executive and HR dashboards |
| `hr-service` | Engagement scores, wellbeing flags | Enrich employee profiles with experience data |
| `learning-service` | Development recommendations | Trigger personalized learning paths |
| `marketplace-service` | Engagement indicators | Factor engagement into opportunity matching |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `SurveyPublished` | `experience.survey.published` | `{ tenant_id, survey_id, title, survey_type, target_audience, deadline }` | Published when a survey is activated and made available to respondents |
| `SurveyResponseSubmitted` | `experience.survey.response.submitted` | `{ tenant_id, survey_id, response_id, sentiment_score, is_anonymous }` | Published when a survey response is submitted |
| `CheckinCompleted` | `experience.checkin.completed` | `{ tenant_id, checkin_id, employee_id, manager_id, employee_mood, action_items_count }` | Published when a check-in is marked as completed |
| `WellbeingAlertTriggered` | `experience.wellbeing.alert` | `{ tenant_id, employee_id, assessment_id, stress_level, burnout_risk, alert_type }` | Published when a wellbeing assessment triggers a critical alert |
| `ActionPlanCreated` | `experience.action-plan.created` | `{ tenant_id, action_plan_id, title, owner_id, target_metric, target_value }` | Published when a new action plan is created |
