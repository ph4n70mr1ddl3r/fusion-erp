# 110 - Activity Centers Service Specification

## 1. Domain Overview

The Activity Centers service provides role-based workspaces that aggregate relevant transactions, analytics, and actions for specific business roles such as Benefits Activity Center, Payroll Activity Center, Compensation Activity Center, and Recruiting Activity Center. It delivers configurable activity feeds, task-driven workflows, embedded analytics, and quick-action shortcuts tailored to each role, giving users a single entry point for all their responsibilities.

**Bounded Context:** Role-Based Workspaces & Activity Aggregation
**Service Name:** `activity-service`
**Database:** `data/activity.db`
**HTTP Port:** 8147 | **gRPC Port:** 9147

---

## 2. Database Schema

### 2.1 Activity Center Definitions
```sql
CREATE TABLE activity_center_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    center_name TEXT NOT NULL,
    center_key TEXT NOT NULL,
    target_role_id TEXT NOT NULL,
    target_application TEXT NOT NULL CHECK(target_application IN ('HCM','ERP','SCM','CX','EPM')),
    description TEXT,
    icon_reference TEXT,
    layout_config TEXT NOT NULL,  -- JSON: widget grid layout
    is_default INTEGER NOT NULL DEFAULT 0,
    display_order INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, center_key)
);

CREATE INDEX idx_centers_tenant_role ON activity_center_definitions(tenant_id, target_role_id);
CREATE INDEX idx_centers_tenant_app ON activity_center_definitions(tenant_id, target_application);
CREATE INDEX idx_centers_tenant_status ON activity_center_definitions(tenant_id, status);
```

### 2.2 Center Widgets
```sql
CREATE TABLE center_widgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    center_id TEXT NOT NULL,
    widget_name TEXT NOT NULL,
    widget_type TEXT NOT NULL CHECK(widget_type IN ('TASK_LIST','KPI','CHART','FEED','QUICK_ACTION','EMBEDDED_PAGE','LINK_LIST')),
    data_source_config TEXT NOT NULL,  -- JSON: data binding and query config
    display_config TEXT,               -- JSON: visual settings
    position_row INTEGER NOT NULL DEFAULT 0,
    position_column INTEGER NOT NULL DEFAULT 0,
    span_rows INTEGER NOT NULL DEFAULT 1,
    span_columns INTEGER NOT NULL DEFAULT 1,
    refresh_interval_seconds INTEGER,
    is_collapsible INTEGER NOT NULL DEFAULT 1,
    default_collapsed INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (center_id) REFERENCES activity_center_definitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_widgets_tenant_center ON center_widgets(tenant_id, center_id);
CREATE INDEX idx_widgets_tenant_type ON center_widgets(tenant_id, widget_type);
```

### 2.3 Activity Feeds
```sql
CREATE TABLE activity_feeds (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    center_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    feed_type TEXT NOT NULL CHECK(feed_type IN ('TRANSACTION','NOTIFICATION','ALERT','RECOMMENDATION','SYSTEM')),
    source_module TEXT NOT NULL,
    source_event_type TEXT,
    title TEXT NOT NULL,
    description TEXT,
    entity_type TEXT,
    entity_id TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),
    action_url TEXT,
    is_read INTEGER NOT NULL DEFAULT 0,
    read_at TEXT,
    dismissed INTEGER NOT NULL DEFAULT 0,
    dismissed_at TEXT,
    expires_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (center_id) REFERENCES activity_center_definitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_feeds_tenant_center ON activity_feeds(tenant_id, center_id);
CREATE INDEX idx_feeds_tenant_user ON activity_feeds(tenant_id, user_id, is_read);
CREATE INDEX idx_feeds_tenant_priority ON activity_feeds(tenant_id, priority, created_at);
CREATE INDEX idx_feeds_tenant_expires ON activity_feeds(tenant_id, expires_at);
```

### 2.4 Quick Actions
```sql
CREATE TABLE quick_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    center_id TEXT NOT NULL,
    action_name TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN ('NAVIGATE','API_CALL','WORKFLOW_TRIGGER','MODAL','EXTERNAL')),
    target_config TEXT NOT NULL,  -- JSON: action target configuration
    icon TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    required_permission TEXT,
    usage_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (center_id) REFERENCES activity_center_definitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_actions_tenant_center ON quick_actions(tenant_id, center_id);
CREATE INDEX idx_actions_tenant_active ON quick_actions(tenant_id, is_active);
```

### 2.5 Center KPI Configs
```sql
CREATE TABLE center_kpi_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    center_id TEXT NOT NULL,
    kpi_name TEXT NOT NULL,
    data_source_query TEXT NOT NULL,
    display_type TEXT NOT NULL CHECK(display_type IN ('NUMBER','PERCENTAGE','CURRENCY','SPARKLINE')),
    target_value INTEGER,
    warning_threshold INTEGER,
    critical_threshold INTEGER,
    trend_direction TEXT CHECK(trend_direction IN ('UP','DOWN','FLAT')),
    refresh_frequency TEXT NOT NULL DEFAULT 'HOURLY',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (center_id) REFERENCES activity_center_definitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_kpis_tenant_center ON center_kpi_configs(tenant_id, center_id);
```

### 2.6 Center Preferences
```sql
CREATE TABLE center_preferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    center_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    personalized_layout TEXT,       -- JSON: custom widget positions
    widget_states TEXT,             -- JSON: collapsed/expanded states
    notification_preferences TEXT,  -- JSON: feed notification settings
    pinned_actions TEXT,            -- JSON: pinned quick action IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (center_id) REFERENCES activity_center_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, center_id, user_id)
);

CREATE INDEX idx_prefs_tenant_user ON center_preferences(tenant_id, user_id);
CREATE INDEX idx_prefs_tenant_center ON center_preferences(tenant_id, center_id);
```

---

## 3. REST API Endpoints

### Centers
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/centers` | Create an activity center definition |
| GET | `/api/v1/centers` | List activity center definitions |
| GET | `/api/v1/centers/{id}` | Get center details |
| PUT | `/api/v1/centers/{id}` | Update center definition |
| GET | `/api/v1/centers/by-role/{roleId}` | Get centers by target role |
| GET | `/api/v1/centers/by-user/{userId}` | Get centers for a specific user |

### Widgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/widgets` | Create a center widget |
| GET | `/api/v1/widgets` | List widgets for a center |
| GET | `/api/v1/widgets/{id}` | Get widget details |
| PUT | `/api/v1/widgets/{id}` | Update widget |
| POST | `/api/v1/widgets/reorder` | Reorder widgets within a center |

### Activity Feed
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/feed` | Get activity feed items |
| POST | `/api/v1/feed/{id}/mark-read` | Mark feed item as read |
| POST | `/api/v1/feed/{id}/dismiss` | Dismiss a feed item |
| POST | `/api/v1/feed/bulk-read` | Mark multiple items as read |

### Quick Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/quick-actions` | Create a quick action |
| GET | `/api/v1/quick-actions` | List quick actions |
| GET | `/api/v1/quick-actions/{id}` | Get quick action details |
| PUT | `/api/v1/quick-actions/{id}` | Update quick action |
| POST | `/api/v1/quick-actions/{id}/execute` | Execute a quick action |
| GET | `/api/v1/quick-actions/usage-stats` | Get quick action usage statistics |

### KPI Configs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/kpi-configs` | Create a KPI configuration |
| GET | `/api/v1/kpi-configs` | List KPI configurations |
| GET | `/api/v1/kpi-configs/{id}` | Get KPI config details |
| PUT | `/api/v1/kpi-configs/{id}` | Update KPI configuration |
| GET | `/api/v1/kpi-configs/{id}/values` | Get current KPI values |

### Preferences
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/preferences/{centerId}/{userId}` | Get user preferences for center |
| PUT | `/api/v1/preferences/{centerId}/{userId}` | Update user preferences |

---

## 4. Business Rules

1. A user MUST only access activity centers whose target_role_id matches one of their assigned roles.
2. Activity feed items MUST be ordered by priority first (URGENT > HIGH > NORMAL > LOW) and then by creation timestamp descending.
3. Widget data MUST be scoped to the user's accessible data based on their role and department assignment.
4. Quick actions that require a permission MUST validate the user has that permission before execution.
5. Expired feed items (past `expires_at`) MUST be automatically excluded from active feeds but retained for historical queries.
6. Each user MAY have only one set of preferences per center; creating preferences for an existing center-user pair MUST update the existing record.
7. KPI values MUST be cached according to the configured `refresh_frequency` and MUST NOT be recalculated on every request.
8. The system SHOULD support at least 50 activity centers per tenant without performance degradation.
9. Widget reorder operations MUST be atomic; all position changes within a single reorder request MUST be applied together.
10. Feed items with type ALERT and priority URGENT MUST trigger a real-time push notification to the user.
11. The usage_count on quick actions MUST be incremented atomically on each execution to maintain accurate statistics.
12. Personalized layout preferences MUST fall back to the center's default layout_config when a user has not customized their view.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package activity.v1;

service ActivityCentersService {
    // Centers
    rpc CreateCenter(CreateCenterRequest) returns (CreateCenterResponse);
    rpc GetCenter(GetCenterRequest) returns (GetCenterResponse);
    rpc ListCenters(ListCentersRequest) returns (ListCentersResponse);
    rpc GetCentersByRole(GetCentersByRoleRequest) returns (GetCentersByRoleResponse);
    rpc GetCentersByUser(GetCentersByUserRequest) returns (GetCentersByUserResponse);

    // Widgets
    rpc CreateWidget(CreateWidgetRequest) returns (CreateWidgetResponse);
    rpc ListWidgets(ListWidgetsRequest) returns (ListWidgetsResponse);
    rpc ReorderWidgets(ReorderWidgetsRequest) returns (ReorderWidgetsResponse);

    // Activity Feed
    rpc GetFeed(GetFeedRequest) returns (GetFeedResponse);
    rpc MarkFeedRead(MarkFeedReadRequest) returns (MarkFeedReadResponse);
    rpc DismissFeedItem(DismissFeedItemRequest) returns (DismissFeedItemResponse);
    rpc BulkMarkRead(BulkMarkReadRequest) returns (BulkMarkReadResponse);

    // Quick Actions
    rpc CreateQuickAction(CreateQuickActionRequest) returns (CreateQuickActionResponse);
    rpc ListQuickActions(ListQuickActionsRequest) returns (ListQuickActionsResponse);
    rpc ExecuteQuickAction(ExecuteQuickActionRequest) returns (ExecuteQuickActionResponse);
    rpc GetQuickActionUsageStats(GetQuickActionUsageStatsRequest) returns (GetQuickActionUsageStatsResponse);

    // KPIs
    rpc CreateKpiConfig(CreateKpiConfigRequest) returns (CreateKpiConfigResponse);
    rpc ListKpiConfigs(ListKpiConfigsRequest) returns (ListKpiConfigsResponse);
    rpc GetKpiValues(GetKpiValuesRequest) returns (GetKpiValuesResponse);

    // Preferences
    rpc GetPreferences(GetPreferencesRequest) returns (GetPreferencesResponse);
    rpc UpdatePreferences(UpdatePreferencesRequest) returns (UpdatePreferencesResponse);
}

message ActivityCenter {
    string id = 1;
    string tenant_id = 2;
    string center_name = 3;
    string center_key = 4;
    string target_role_id = 5;
    string target_application = 6;
    string description = 7;
    string icon_reference = 8;
    string layout_config = 9;
    bool is_default = 10;
    int32 display_order = 11;
    string status = 12;
}

message CreateCenterRequest {
    string tenant_id = 1;
    string center_name = 2;
    string center_key = 3;
    string target_role_id = 4;
    string target_application = 5;
    string description = 6;
    string layout_config = 7;
    string created_by = 8;
}

message CreateCenterResponse {
    ActivityCenter center = 1;
}

message GetCenterRequest {
    string tenant_id = 1;
    string center_id = 2;
}

message GetCenterResponse {
    ActivityCenter center = 1;
}

message ListCentersRequest {
    string tenant_id = 1;
    string target_application = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCentersResponse {
    repeated ActivityCenter centers = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetCentersByRoleRequest {
    string tenant_id = 1;
    string role_id = 2;
}

message GetCentersByRoleResponse {
    repeated ActivityCenter centers = 1;
}

message GetCentersByUserRequest {
    string tenant_id = 1;
    string user_id = 2;
}

message GetCentersByUserResponse {
    repeated ActivityCenter centers = 1;
}

message CenterWidget {
    string id = 1;
    string tenant_id = 2;
    string center_id = 3;
    string widget_name = 4;
    string widget_type = 5;
    string data_source_config = 6;
    string display_config = 7;
    int32 position_row = 8;
    int32 position_column = 9;
    int32 span_rows = 10;
    int32 span_columns = 11;
    int32 refresh_interval_seconds = 12;
    bool is_collapsible = 13;
    bool default_collapsed = 14;
}

message CreateWidgetRequest {
    string tenant_id = 1;
    string center_id = 2;
    string widget_name = 3;
    string widget_type = 4;
    string data_source_config = 5;
    string display_config = 6;
    int32 position_row = 7;
    int32 position_column = 8;
    string created_by = 9;
}

message CreateWidgetResponse {
    CenterWidget widget = 1;
}

message ListWidgetsRequest {
    string tenant_id = 1;
    string center_id = 2;
}

message ListWidgetsResponse {
    repeated CenterWidget widgets = 1;
}

message WidgetPosition {
    string widget_id = 1;
    int32 position_row = 2;
    int32 position_column = 3;
    int32 span_rows = 4;
    int32 span_columns = 5;
}

message ReorderWidgetsRequest {
    string tenant_id = 1;
    string center_id = 2;
    repeated WidgetPosition positions = 3;
    string updated_by = 4;
}

message ReorderWidgetsResponse {
    repeated CenterWidget widgets = 1;
}

message FeedItem {
    string id = 1;
    string tenant_id = 2;
    string center_id = 3;
    string user_id = 4;
    string feed_type = 5;
    string source_module = 6;
    string title = 7;
    string description = 8;
    string entity_type = 9;
    string entity_id = 10;
    string priority = 11;
    string action_url = 12;
    bool is_read = 13;
    bool dismissed = 14;
    string created_at = 15;
}

message GetFeedRequest {
    string tenant_id = 1;
    string center_id = 2;
    string user_id = 3;
    string feed_type = 4;
    bool unread_only = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message GetFeedResponse {
    repeated FeedItem items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
    int32 unread_count = 4;
}

message MarkFeedReadRequest {
    string tenant_id = 1;
    string feed_item_id = 2;
    string user_id = 3;
}

message MarkFeedReadResponse {
    FeedItem item = 1;
}

message DismissFeedItemRequest {
    string tenant_id = 1;
    string feed_item_id = 2;
    string user_id = 3;
}

message DismissFeedItemResponse {
    FeedItem item = 1;
}

message BulkMarkReadRequest {
    string tenant_id = 1;
    repeated string feed_item_ids = 2;
    string user_id = 3;
}

message BulkMarkReadResponse {
    int32 updated_count = 1;
}

message QuickAction {
    string id = 1;
    string tenant_id = 2;
    string center_id = 3;
    string action_name = 4;
    string action_type = 5;
    string target_config = 6;
    string icon = 7;
    int32 display_order = 8;
    string required_permission = 9;
    bool is_active = 10;
    int32 usage_count = 11;
}

message CreateQuickActionRequest {
    string tenant_id = 1;
    string center_id = 2;
    string action_name = 3;
    string action_type = 4;
    string target_config = 5;
    string icon = 6;
    int32 display_order = 7;
    string required_permission = 8;
    string created_by = 9;
}

message CreateQuickActionResponse {
    QuickAction action = 1;
}

message ListQuickActionsRequest {
    string tenant_id = 1;
    string center_id = 2;
}

message ListQuickActionsResponse {
    repeated QuickAction actions = 1;
}

message ExecuteQuickActionRequest {
    string tenant_id = 1;
    string action_id = 2;
    string user_id = 3;
    string parameters = 4;
}

message ExecuteQuickActionResponse {
    bool success = 1;
    string result_data = 2;
    string redirect_url = 3;
}

message QuickActionUsageStat {
    string action_id = 1;
    string action_name = 2;
    int32 usage_count = 3;
    string last_used_at = 4;
}

message GetQuickActionUsageStatsRequest {
    string tenant_id = 1;
    string center_id = 2;
}

message GetQuickActionUsageStatsResponse {
    repeated QuickActionUsageStat stats = 1;
}

message KpiConfig {
    string id = 1;
    string tenant_id = 2;
    string center_id = 3;
    string kpi_name = 4;
    string data_source_query = 5;
    string display_type = 6;
    int64 target_value = 7;
    int64 warning_threshold = 8;
    int64 critical_threshold = 9;
    string trend_direction = 10;
    string refresh_frequency = 11;
}

message CreateKpiConfigRequest {
    string tenant_id = 1;
    string center_id = 2;
    string kpi_name = 3;
    string data_source_query = 4;
    string display_type = 5;
    int64 target_value = 6;
    int64 warning_threshold = 7;
    int64 critical_threshold = 8;
    string created_by = 9;
}

message CreateKpiConfigResponse {
    KpiConfig config = 1;
}

message ListKpiConfigsRequest {
    string tenant_id = 1;
    string center_id = 2;
}

message ListKpiConfigsResponse {
    repeated KpiConfig configs = 1;
}

message KpiValue {
    string kpi_id = 1;
    string kpi_name = 2;
    int64 current_value = 3;
    int64 target_value = 4;
    string display_type = 5;
    string status = 6;
    double trend_pct = 7;
}

message GetKpiValuesRequest {
    string tenant_id = 1;
    string center_id = 2;
    string user_id = 3;
}

message GetKpiValuesResponse {
    repeated KpiValue values = 1;
}

message CenterPreferences {
    string id = 1;
    string tenant_id = 2;
    string center_id = 3;
    string user_id = 4;
    string personalized_layout = 5;
    string widget_states = 6;
    string notification_preferences = 7;
    string pinned_actions = 8;
}

message GetPreferencesRequest {
    string tenant_id = 1;
    string center_id = 2;
    string user_id = 3;
}

message GetPreferencesResponse {
    CenterPreferences preferences = 1;
}

message UpdatePreferencesRequest {
    string tenant_id = 1;
    string center_id = 2;
    string user_id = 3;
    string personalized_layout = 4;
    string widget_states = 5;
    string notification_preferences = 6;
    string pinned_actions = 7;
    string updated_by = 8;
}

message UpdatePreferencesResponse {
    CenterPreferences preferences = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee roles, department assignments, org structure | Resolve user roles for center access and data scoping |
| `security-service` | User permissions | Validate quick action execution permissions |
| `notification-service` | Event subscriptions | Generate activity feed items from system events |
| `analytics-service` | KPI data, chart datasets | Populate KPI and chart widget values |
| `workflow-service` | Pending tasks, approvals | Feed task list widgets with pending items |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `notification-service` | Unread feed counts, urgent alerts | Real-time user notifications |
| `reporting-service` | Center usage metrics, quick action stats | Workspace adoption analytics |
| `gateway-service` | Center configurations, user preferences | Route users to correct activity center |
| `workflow-service` | Quick action triggers | Initiate workflows from quick actions |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ActivityFeedItemCreated` | `activity.feed.created` | `{ tenant_id, center_id, user_id, feed_item_id, feed_type, priority, title }` | Published when a new activity feed item is created |
| `WidgetDataRefreshed` | `activity.widget.refreshed` | `{ tenant_id, center_id, widget_id, widget_type, refreshed_at }` | Published when widget data is refreshed |
| `QuickActionExecuted` | `activity.action.executed` | `{ tenant_id, center_id, action_id, action_name, user_id, success }` | Published when a quick action is executed |
| `CenterAccessed` | `activity.center.accessed` | `{ tenant_id, center_id, center_key, user_id, accessed_at }` | Published when a user accesses an activity center |
