# 106 - Manager Edge Service Specification

## 1. Domain Overview

The Manager Edge service provides an AI-powered role-specific experience layer for people managers. It aggregates actionable team insights, pending approvals, recommended actions, team analytics dashboards, AI-driven coaching suggestions, and manager effectiveness scoring into a unified command center. The service pulls data from across the HCM suite to give managers a single pane of glass for all people-management responsibilities.

**Bounded Context:** Manager Experience & Insights
**Service Name:** `manageredge-service`
**Database:** `data/manageredge.db`
**HTTP Port:** 8143 | **gRPC Port:** 9143

---

## 2. Database Schema

### 2.1 Manager Dashboards
```sql
CREATE TABLE manager_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    layout_config TEXT NOT NULL,  -- JSON: widget layout grid
    widget_preferences TEXT,      -- JSON: per-widget settings
    is_default INTEGER NOT NULL DEFAULT 0,
    refresh_interval_seconds INTEGER NOT NULL DEFAULT 300,
    last_viewed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, manager_id, dashboard_name)
);

CREATE INDEX idx_dashboards_tenant_manager ON manager_dashboards(tenant_id, manager_id);
CREATE INDEX idx_dashboards_tenant_default ON manager_dashboards(tenant_id, manager_id, is_default);
```

### 2.2 Team Insights
```sql
CREATE TABLE team_insights (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    team_member_id TEXT NOT NULL,
    insight_type TEXT NOT NULL CHECK(insight_type IN ('ENGAGEMENT','PERFORMANCE','RISK','RETENTION','DEVELOPMENT','ABSENCE')),
    insight_title TEXT NOT NULL,
    insight_description TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'INFO' CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    confidence_score DECIMAL(5,2),
    recommended_actions TEXT,  -- JSON array of action objects
    source_data TEXT,          -- JSON: originating data references
    expires_at TEXT,
    is_actioned INTEGER NOT NULL DEFAULT 0,
    actioned_at TEXT,
    actioned_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_insights_tenant_manager ON team_insights(tenant_id, manager_id);
CREATE INDEX idx_insights_tenant_member ON team_insights(tenant_id, team_member_id);
CREATE INDEX idx_insights_tenant_type ON team_insights(tenant_id, insight_type, severity);
CREATE INDEX idx_insights_tenant_expires ON team_insights(tenant_id, expires_at);
```

### 2.3 Manager Action Items
```sql
CREATE TABLE manager_action_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN ('APPROVAL','REVIEW','FEEDBACK','CHECKIN','GOAL_REVIEW','COMPENSATION','DEVELOPMENT','COMPLIANCE')),
    title TEXT NOT NULL,
    description TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    source_module TEXT NOT NULL,
    source_reference TEXT,
    due_date TEXT,
    completed_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','OVERDUE','WAIVED')),
    delegation_eligible INTEGER NOT NULL DEFAULT 0,
    delegated_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_actions_tenant_manager ON manager_action_items(tenant_id, manager_id, status);
CREATE INDEX idx_actions_tenant_due ON manager_action_items(tenant_id, manager_id, due_date);
CREATE INDEX idx_actions_tenant_priority ON manager_action_items(tenant_id, priority, status);
CREATE INDEX idx_actions_tenant_source ON manager_action_items(tenant_id, source_module);
```

### 2.4 Team Analytics Snapshots
```sql
CREATE TABLE team_analytics_snapshots (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    snapshot_date TEXT NOT NULL,
    team_size INTEGER NOT NULL,
    avg_performance_rating DECIMAL(3,2),
    avg_engagement_score DECIMAL(5,2),
    attrition_risk_count INTEGER NOT NULL DEFAULT 0,
    open_positions INTEGER NOT NULL DEFAULT 0,
    pending_promotions_count INTEGER NOT NULL DEFAULT 0,
    avg_tenure_months DECIMAL(7,1),
    headcount_change_pct DECIMAL(5,2),
    diversity_metrics TEXT,  -- JSON: diversity breakdown

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, manager_id, snapshot_date)
);

CREATE INDEX idx_snapshots_tenant_manager ON team_analytics_snapshots(tenant_id, manager_id, snapshot_date);
CREATE INDEX idx_snapshots_tenant_date ON team_analytics_snapshots(tenant_id, snapshot_date);
```

### 2.5 Coaching Recommendations
```sql
CREATE TABLE coaching_recommendations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    team_member_id TEXT NOT NULL,
    recommendation_type TEXT NOT NULL CHECK(recommendation_type IN ('ONE_ON_ONE','FEEDBACK','RECOGNITION','CAREER_DISCUSSION','SKILL_DEVELOPMENT','WORKLOAD_BALANCE')),
    context TEXT,
    suggested_talking_points TEXT,  -- JSON array of strings
    priority TEXT NOT NULL DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    based_on_insights TEXT,         -- JSON array of insight IDs
    dismissed INTEGER NOT NULL DEFAULT 0,
    dismissed_reason TEXT,
    acted_on INTEGER NOT NULL DEFAULT 0,
    action_taken TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_coaching_tenant_manager ON coaching_recommendations(tenant_id, manager_id);
CREATE INDEX idx_coaching_tenant_member ON coaching_recommendations(tenant_id, team_member_id);
CREATE INDEX idx_coaching_tenant_type ON coaching_recommendations(tenant_id, recommendation_type, dismissed);
```

### 2.6 Manager Effectiveness Scores
```sql
CREATE TABLE manager_effectiveness_scores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    feedback_response_rate DECIMAL(5,2),
    checkin_frequency_score DECIMAL(5,2),
    team_satisfaction_score DECIMAL(5,2),
    goal_completion_rate DECIMAL(5,2),
    retention_rate DECIMAL(5,2),
    overall_score DECIMAL(5,2),
    benchmark_percentile DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, manager_id, period_start, period_end)
);

CREATE INDEX idx_effectiveness_tenant_manager ON manager_effectiveness_scores(tenant_id, manager_id);
CREATE INDEX idx_effectiveness_tenant_period ON manager_effectiveness_scores(tenant_id, period_start, period_end);
```

---

## 3. REST API Endpoints

### Dashboards
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/dashboards` | List manager dashboards |
| GET | `/api/v1/dashboards/{id}` | Get dashboard configuration |
| PUT | `/api/v1/dashboards/{id}` | Update dashboard layout and preferences |
| GET | `/api/v1/dashboards/{id}/widgets` | Get widget data for dashboard |

### Insights
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/insights/team-insights` | List team insights for manager |
| POST | `/api/v1/insights/{id}/dismiss` | Dismiss an insight |
| POST | `/api/v1/insights/{id}/action` | Mark insight as actioned |

### Action Items
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/action-items/pending` | List pending action items |
| POST | `/api/v1/action-items/{id}/complete` | Complete an action item |
| POST | `/api/v1/action-items/{id}/delegate` | Delegate action item to another user |
| GET | `/api/v1/action-items/overdue` | List overdue action items |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/analytics/team-summary` | Get team analytics summary |
| GET | `/api/v1/analytics/trends` | Get team metric trends over time |
| GET | `/api/v1/analytics/headcount` | Get headcount change data |

### Coaching
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/coaching/recommendations` | List coaching recommendations |
| POST | `/api/v1/coaching/{id}/dismiss` | Dismiss a coaching recommendation |
| POST | `/api/v1/coaching/{id}/acted-on` | Mark recommendation as acted upon |

### Effectiveness
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/effectiveness/score` | Get manager effectiveness score |
| GET | `/api/v1/effectiveness/benchmark` | Get benchmark comparison data |

---

## 4. Business Rules

1. Insights with severity CRITICAL MUST be surfaced to the manager within 5 minutes of generation and MUST trigger a push notification.
2. Action items past their due date MUST be automatically escalated to OVERDUE status and the manager's supervisor SHOULD be notified.
3. A manager MUST only see insights and action items for direct and indirect reports within their reporting chain.
4. Coaching recommendations MUST be generated at most once per team member per week to avoid recommendation fatigue.
5. The manager effectiveness score MUST be calculated using a weighted composite of all sub-scores, with retention_rate weighted at least 25%.
6. Dashboard configuration changes MUST be persisted per manager and MUST NOT affect other managers' views.
7. Insights that have expired (past `expires_at`) MUST be automatically excluded from active insight listings but MUST be retained for audit purposes.
8. Delegation of action items MUST only be permitted when `delegation_eligible` is set to 1 and the delegate is within the same reporting hierarchy.
9. Team analytics snapshots MUST be captured at least daily for active managers and MUST retain 24 months of historical data.
10. The confidence_score on insights MUST be recalculated when underlying source data changes and SHOULD trigger re-evaluation of severity.
11. Managers MUST NOT be able to dismiss CRITICAL insights without providing an action taken description.
12. The system SHOULD aggregate pending approvals from all HCM modules and present them as unified action items on the dashboard.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package manageredge.v1;

service ManagerEdgeService {
    // Dashboards
    rpc GetDashboard(GetDashboardRequest) returns (GetDashboardResponse);
    rpc UpdateDashboard(UpdateDashboardRequest) returns (UpdateDashboardResponse);
    rpc GetDashboardWidgets(GetDashboardWidgetsRequest) returns (GetDashboardWidgetsResponse);

    // Insights
    rpc ListTeamInsights(ListTeamInsightsRequest) returns (ListTeamInsightsResponse);
    rpc DismissInsight(DismissInsightRequest) returns (DismissInsightResponse);
    rpc ActionInsight(ActionInsightRequest) returns (ActionInsightResponse);

    // Action Items
    rpc ListPendingActions(ListPendingActionsRequest) returns (ListPendingActionsResponse);
    rpc CompleteAction(CompleteActionRequest) returns (CompleteActionResponse);
    rpc DelegateAction(DelegateActionRequest) returns (DelegateActionResponse);
    rpc ListOverdueActions(ListOverdueActionsRequest) returns (ListOverdueActionsResponse);

    // Analytics
    rpc GetTeamSummary(GetTeamSummaryRequest) returns (GetTeamSummaryResponse);
    rpc GetTeamTrends(GetTeamTrendsRequest) returns (GetTeamTrendsResponse);

    // Coaching
    rpc ListCoachingRecommendations(ListCoachingRecommendationsRequest) returns (ListCoachingRecommendationsResponse);
    rpc DismissRecommendation(DismissRecommendationRequest) returns (DismissRecommendationResponse);
    rpc MarkRecommendationActedOn(MarkRecommendationActedOnRequest) returns (MarkRecommendationActedOnResponse);

    // Effectiveness
    rpc GetEffectivenessScore(GetEffectivenessScoreRequest) returns (GetEffectivenessScoreResponse);
    rpc GetBenchmark(GetBenchmarkRequest) returns (GetBenchmarkResponse);
}

message Dashboard {
    string id = 1;
    string tenant_id = 2;
    string manager_id = 3;
    string dashboard_name = 4;
    string layout_config = 5;
    string widget_preferences = 6;
    bool is_default = 7;
    int32 refresh_interval_seconds = 8;
    string last_viewed_at = 9;
}

message GetDashboardRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string dashboard_id = 3;
}

message GetDashboardResponse {
    Dashboard dashboard = 1;
}

message UpdateDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    string layout_config = 3;
    string widget_preferences = 4;
    string updated_by = 5;
}

message UpdateDashboardResponse {
    Dashboard dashboard = 1;
}

message GetDashboardWidgetsRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
}

message GetDashboardWidgetsResponse {
    repeated WidgetData widgets = 1;
}

message WidgetData {
    string widget_id = 1;
    string widget_type = 2;
    string data = 3;
}

message TeamInsight {
    string id = 1;
    string tenant_id = 2;
    string manager_id = 3;
    string team_member_id = 4;
    string insight_type = 5;
    string insight_title = 6;
    string insight_description = 7;
    string severity = 8;
    double confidence_score = 9;
    string recommended_actions = 10;
    string expires_at = 11;
    bool is_actioned = 12;
}

message ListTeamInsightsRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string insight_type = 3;
    string severity = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListTeamInsightsResponse {
    repeated TeamInsight insights = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message DismissInsightRequest {
    string tenant_id = 1;
    string insight_id = 2;
    string dismissed_by = 3;
}

message DismissInsightResponse {
    TeamInsight insight = 1;
}

message ActionInsightRequest {
    string tenant_id = 1;
    string insight_id = 2;
    string actioned_by = 3;
}

message ActionInsightResponse {
    TeamInsight insight = 1;
}

message ActionItem {
    string id = 1;
    string tenant_id = 2;
    string manager_id = 3;
    string action_type = 4;
    string title = 5;
    string description = 6;
    string priority = 7;
    string source_module = 8;
    string due_date = 9;
    string status = 10;
}

message ListPendingActionsRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string action_type = 3;
    string priority = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListPendingActionsResponse {
    repeated ActionItem actions = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CompleteActionRequest {
    string tenant_id = 1;
    string action_id = 2;
    string completed_by = 3;
}

message CompleteActionResponse {
    ActionItem action = 1;
}

message DelegateActionRequest {
    string tenant_id = 1;
    string action_id = 2;
    string delegated_to = 3;
    string delegated_by = 4;
}

message DelegateActionResponse {
    ActionItem action = 1;
}

message ListOverdueActionsRequest {
    string tenant_id = 1;
    string manager_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListOverdueActionsResponse {
    repeated ActionItem actions = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TeamSummary {
    string id = 1;
    string manager_id = 2;
    string snapshot_date = 3;
    int32 team_size = 4;
    double avg_performance_rating = 5;
    double avg_engagement_score = 6;
    int32 attrition_risk_count = 7;
    int32 open_positions = 8;
    double headcount_change_pct = 9;
}

message GetTeamSummaryRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string snapshot_date = 3;
}

message GetTeamSummaryResponse {
    TeamSummary summary = 1;
}

message GetTeamTrendsRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetTeamTrendsResponse {
    repeated TeamSummary snapshots = 1;
}

message CoachingRecommendation {
    string id = 1;
    string tenant_id = 2;
    string manager_id = 3;
    string team_member_id = 4;
    string recommendation_type = 5;
    string context = 6;
    string suggested_talking_points = 7;
    string priority = 8;
    bool dismissed = 9;
    bool acted_on = 10;
}

message ListCoachingRecommendationsRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string recommendation_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCoachingRecommendationsResponse {
    repeated CoachingRecommendation recommendations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message DismissRecommendationRequest {
    string tenant_id = 1;
    string recommendation_id = 2;
    string dismissed_reason = 3;
    string dismissed_by = 4;
}

message DismissRecommendationResponse {
    CoachingRecommendation recommendation = 1;
}

message MarkRecommendationActedOnRequest {
    string tenant_id = 1;
    string recommendation_id = 2;
    string action_taken = 3;
    string acted_by = 4;
}

message MarkRecommendationActedOnResponse {
    CoachingRecommendation recommendation = 1;
}

message EffectivenessScore {
    string id = 1;
    string manager_id = 2;
    string period_start = 3;
    string period_end = 4;
    double feedback_response_rate = 5;
    double checkin_frequency_score = 6;
    double team_satisfaction_score = 7;
    double goal_completion_rate = 8;
    double retention_rate = 9;
    double overall_score = 10;
    double benchmark_percentile = 11;
}

message GetEffectivenessScoreRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetEffectivenessScoreResponse {
    EffectivenessScore score = 1;
}

message GetBenchmarkRequest {
    string tenant_id = 1;
    string manager_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetBenchmarkResponse {
    EffectivenessScore manager_score = 1;
    double org_average_score = 2;
    double org_median_score = 3;
    int32 total_managers_compared = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee records, reporting hierarchy, org structure | Resolve team membership and reporting chains |
| `performance-service` | Performance ratings, goal completion data | Feed team analytics and effectiveness scoring |
| `goals-service` | Goal progress and completion rates | Calculate goal_completion_rate for effectiveness |
| `compensation-service` | Compensation review pending actions | Aggregate approval action items |
| `time-service` | Absence and time-off data | Generate absence-related insights |
| `recruiting-service` | Open requisitions and hiring status | Populate open positions in team analytics |
| `survey-service` | Engagement survey results | Feed engagement scores into insights |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `notification-service` | Insight alerts, overdue reminders | Push notifications to managers |
| `reporting-service` | Team analytics, effectiveness scores | Executive and HR dashboards |
| `workflow-service` | Approval delegation requests | Route delegated action items |
| `coaching-service` | Coaching action outcomes | Track coaching program effectiveness |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `InsightGenerated` | `manageredge.insight.generated` | `{ tenant_id, manager_id, insight_id, insight_type, severity, team_member_id, confidence_score }` | Published when a new team insight is generated |
| `ActionItemCreated` | `manageredge.action.created` | `{ tenant_id, manager_id, action_id, action_type, priority, source_module, due_date }` | Published when a new action item is created |
| `CoachingRecommended` | `manageredge.coaching.recommended` | `{ tenant_id, manager_id, recommendation_id, team_member_id, recommendation_type, priority }` | Published when a coaching recommendation is generated |
| `EffectivenessScoreCalculated` | `manageredge.effectiveness.calculated` | `{ tenant_id, manager_id, period_start, period_end, overall_score, benchmark_percentile }` | Published when effectiveness score is recalculated |
