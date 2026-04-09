# 169 - Goals Management Service Specification

## 1. Domain Overview

Goals Management provides enterprise goal setting, cascading alignment, progress tracking, and OKR/KPI management across the organization. Supports top-down goal cascading from company objectives to department to individual goals, weight-based alignment, quantitative and qualitative goal types, milestone tracking, and goal-based performance evaluation integration. Enables organizations to align workforce efforts with strategic objectives, track OKR progress, and link goals to performance reviews and compensation.

**Bounded Context:** Goal Setting, Alignment & OKR Management
**Service Name:** `goals-service`
**Database:** `data/goals.db`
**HTTP Port:** 8187 | **gRPC Port:** 9187

---

## 2. Database Schema

### 2.1 Goal Plans
```sql
CREATE TABLE gl_goal_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('ANNUAL','QUARTERLY','OKR_CYCLE','PROJECT','CUSTOM')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    review_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','REVIEW','CLOSED','ARCHIVED')),
    goal_count INTEGER NOT NULL DEFAULT 0,
    avg_completion_pct REAL NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_gl_plan_tenant ON gl_goal_plans(tenant_id, status);
```

### 2.2 Goals
```sql
CREATE TABLE gl_goals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    parent_goal_id TEXT,                   -- For cascaded goals
    goal_name TEXT NOT NULL,
    goal_type TEXT NOT NULL CHECK(goal_type IN ('COMPANY','DEPARTMENT','TEAM','INDIVIDUAL','OKR_OBJECTIVE','KEY_RESULT')),
    description TEXT NOT NULL,
    owner_type TEXT NOT NULL CHECK(owner_type IN ('COMPANY','DEPARTMENT','TEAM','INDIVIDUAL')),
    owner_id TEXT NOT NULL,                -- Person, team, or org unit ID
    assigned_by TEXT,
    weight REAL NOT NULL DEFAULT 1.0,      -- Weight in parent goal or review
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    start_date TEXT NOT NULL,
    target_date TEXT NOT NULL,
    completion_pct REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','ON_TRACK','AT_RISK','BEHIND','COMPLETED','CANCELLED')),
    alignment_type TEXT CHECK(alignment_type IN ('DIRECTLY_ALIGNED','INDIRECTLY_ALIGNED','NONE')),
    linked_goal_ids TEXT,                  -- JSON: goals this aligns with
    category TEXT,
    tags TEXT,                             -- JSON array

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES gl_goal_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_gl_goal_owner ON gl_goals(owner_type, owner_id, status);
CREATE INDEX idx_gl_goal_parent ON gl_goals(parent_goal_id);
CREATE INDEX idx_gl_goal_plan ON gl_goals(plan_id, status);
CREATE INDEX idx_gl_goal_status ON gl_goals(tenant_id, status);
```

### 2.3 Goal Metrics
```sql
CREATE TABLE gl_goal_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    goal_id TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    metric_type TEXT NOT NULL CHECK(metric_type IN ('QUANTITATIVE','QUALITATIVE','PERCENTAGE','CURRENCY','COUNT','BOOLEAN')),
    target_value REAL NOT NULL,
    current_value REAL NOT NULL DEFAULT 0,
    start_value REAL NOT NULL DEFAULT 0,
    unit TEXT NOT NULL,                     -- "$", "%", "units", "score"
    measurement_method TEXT NOT NULL CHECK(measurement_method IN ('MANUAL','AUTOMATIC','FORMULA','INTEGRATION')),
    data_source TEXT,                       -- For automatic metrics
    formula TEXT,                           -- For formula-based metrics
    frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(frequency IN ('WEEKLY','BIWEEKLY','MONTHLY','QUARTERLY','MANUAL')),
    last_updated TEXT,
    last_updated_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (goal_id) REFERENCES gl_goals(id) ON DELETE CASCADE
);

CREATE INDEX idx_gl_metric_goal ON gl_goal_metrics(goal_id);
```

### 2.4 Goal Progress Updates
```sql
CREATE TABLE gl_progress_updates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    goal_id TEXT NOT NULL,
    metric_id TEXT,
    update_type TEXT NOT NULL CHECK(update_type IN ('STATUS_CHANGE','METRIC_UPDATE','COMMENT','MILESTONE','CHECK_IN')),
    previous_value TEXT,
    new_value TEXT,
    comment TEXT NOT NULL,
    confidence_level TEXT CHECK(confidence_level IN ('HIGH','MEDIUM','LOW')),
    blockers TEXT,
    next_steps TEXT,
    updated_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (goal_id) REFERENCES gl_goals(id) ON DELETE CASCADE
);

CREATE INDEX idx_gl_progress_goal ON gl_progress_updates(goal_id, created_at DESC);
CREATE INDEX idx_gl_progress_user ON gl_progress_updates(tenant_id, updated_by, created_at DESC);
```

### 2.5 Goal Library (Templates)
```sql
CREATE TABLE gl_goal_library (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    category TEXT NOT NULL,
    description TEXT NOT NULL,
    suggested_metrics TEXT NOT NULL,        -- JSON: metric templates
    applicable_roles TEXT,                 -- JSON: role/job family applicability
    default_weight REAL NOT NULL DEFAULT 1.0,
    is_system INTEGER NOT NULL DEFAULT 0,
    usage_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_name)
);
```

---

## 3. API Endpoints

### 3.1 Goal Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/goals/plans` | Create goal plan |
| GET | `/api/v1/goals/plans` | List plans |
| GET | `/api/v1/goals/plans/{id}` | Get plan details |
| PUT | `/api/v1/goals/plans/{id}` | Update plan |
| POST | `/api/v1/goals/plans/{id}/activate` | Activate plan |
| POST | `/api/v1/goals/plans/{id}/close` | Close plan |

### 3.2 Goals
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/goals/goals` | Create goal |
| GET | `/api/v1/goals/goals` | List goals |
| GET | `/api/v1/goals/goals/{id}` | Get goal with metrics |
| PUT | `/api/v1/goals/goals/{id}` | Update goal |
| POST | `/api/v1/goals/goals/{id}/cascade` | Cascade goal to reports |
| POST | `/api/v1/goals/goals/{id}/align` | Align to parent goal |
| DELETE | `/api/v1/goals/goals/{id}` | Cancel goal |
| GET | `/api/v1/goals/goals/{id}/tree` | Get cascaded goal tree |

### 3.3 Metrics & Progress
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/goals/goals/{id}/metrics` | Add metric |
| PUT | `/api/v1/goals/metrics/{id}` | Update metric |
| POST | `/api/v1/goals/goals/{id}/update` | Submit progress update |
| GET | `/api/v1/goals/goals/{id}/progress` | Get progress history |

### 3.4 Library & Alignment
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/goals/library` | List goal templates |
| POST | `/api/v1/goals/library` | Add to library |
| GET | `/api/v1/goals/alignment-map` | Organization alignment visualization |
| GET | `/api/v1/goals/dashboard` | Goals dashboard |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `goal.plan.activated` | `{ plan_id, period }` | Goal plan activated |
| `goal.created` | `{ goal_id, owner_id, parent_goal_id }` | Goal created |
| `goal.progress.updated` | `{ goal_id, completion_pct, updated_by }` | Progress update submitted |
| `goal.status.changed` | `{ goal_id, old_status, new_status }` | Goal status changed |
| `goal.completed` | `{ goal_id, owner_id, completion_pct }` | Goal completed |
| `goal.at_risk` | `{ goal_id, owner_id, completion_pct, target_date }` | Goal flagged at risk |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `performance.cycle.started` | Performance Management | Activate goal plan |
| `metric.data.available` | ERP/SCM Analytics | Auto-update quantitative metrics |
| `project.milestone.completed` | Project Management | Update linked project goals |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.goals.v1;

service GoalsService {
    rpc GetGoalPlan(GetGoalPlanRequest) returns (GetGoalPlanResponse);
    rpc CreateGoalPlan(CreateGoalPlanRequest) returns (CreateGoalPlanResponse);
    rpc GetGoal(GetGoalRequest) returns (GetGoalResponse);
    rpc CreateGoal(CreateGoalRequest) returns (CreateGoalResponse);
    rpc UpdateProgress(UpdateProgressRequest) returns (UpdateProgressResponse);
    rpc GetGoalTree(GetGoalTreeRequest) returns (GetGoalTreeResponse);
}

// Goal Plan messages
message GetGoalPlanRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetGoalPlanResponse {
    GoalPlan data = 1;
}

message GoalPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_code = 3;
    string plan_name = 4;
    string plan_type = 5;
    string period_start = 6;
    string period_end = 7;
    string status = 8;
    int32 goal_count = 9;
    double avg_completion_pct = 10;
    string description = 11;
    string created_at = 12;
    string updated_at = 13;
}

message CreateGoalPlanRequest {
    string tenant_id = 1;
    string plan_code = 2;
    string plan_name = 3;
    string plan_type = 4;
    string period_start = 5;
    string period_end = 6;
    string description = 7;
}

message CreateGoalPlanResponse {
    GoalPlan data = 1;
}

// Goal messages
message GetGoalRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetGoalResponse {
    Goal data = 1;
}

message Goal {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string parent_goal_id = 4;
    string goal_name = 5;
    string goal_type = 6;
    string description = 7;
    string owner_type = 8;
    string owner_id = 9;
    double weight = 10;
    string priority = 11;
    string start_date = 12;
    string target_date = 13;
    double completion_pct = 14;
    string status = 15;
    string alignment_type = 16;
    string linked_goal_ids = 17;
    string category = 18;
    string tags = 19;
    string created_at = 20;
    string updated_at = 21;
}

message CreateGoalRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string parent_goal_id = 3;
    string goal_name = 4;
    string goal_type = 5;
    string description = 6;
    string owner_type = 7;
    string owner_id = 8;
    double weight = 9;
    string priority = 10;
    string start_date = 11;
    string target_date = 12;
    string category = 13;
}

message CreateGoalResponse {
    Goal data = 1;
}

// Progress messages
message UpdateProgressRequest {
    string tenant_id = 1;
    string goal_id = 2;
    string metric_id = 3;
    string update_type = 4;
    string new_value = 5;
    string comment = 6;
    string confidence_level = 7;
    string updated_by = 8;
}

message UpdateProgressResponse {
    string id = 1;
    double completion_pct = 2;
    string status = 3;
}

// Goal Tree messages
message GetGoalTreeRequest {
    string tenant_id = 1;
    string goal_id = 2;
}

message GetGoalTreeResponse {
    repeated GoalNode nodes = 1;
}

message GoalNode {
    Goal goal = 1;
    repeated GoalNode children = 2;
    double alignment_weight = 3;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | gl_goal_plans | -- |
| V002 | gl_goals | V001 |
| V003 | gl_goal_metrics | V002 |
| V004 | gl_progress_updates | V002 |
| V005 | gl_goal_library | -- |

---

## 7. Business Rules

1. **Cascading**: Company goals cascade to departments → teams → individuals; alignment tracked
2. **Weight Validation**: Sum of individual goal weights within a plan must equal 100%
3. **Metric Ownership**: Goal owner updates metrics; manager can override with justification
4. **Status Automation**: Goals auto-flagged AT_RISK when completion < 50% with < 30% time remaining
5. **Goal Library**: Reusable templates for common goal patterns; auto-suggest during creation
6. **Alignment Scoring**: Organization alignment score = % of individual goals linked to company goals
7. **Close Process**: Plan close freezes all goals; final completion recorded for review

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Performance Management (68) | Goals feed into performance reviews |
| Compensation (65) | Goal completion influences merit/variable pay |
| Career Development (71) | Development goals |
| Talent Review (146) | Goal completion as talent assessment input |
| Manager Edge (106) | Manager goal visibility and coaching |
| Employee Experience (95) | Employee goal dashboard |
| HCM Analytics (148) | Organization-wide goal metrics |
| OKR/Project Management (15) | Project milestone-linked goals |
