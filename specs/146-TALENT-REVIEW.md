# 146 - Talent Review Service Specification

## 1. Domain Overview

Talent Review provides structured talent assessment processes including calibration sessions, 9-box talent matrix, talent pool management, and succession readiness evaluation. Supports configurable review models, multi-rater assessments, talent categorization, and action planning for development, retention, and mobility. Integrates with Performance Management for ratings, Career Development for growth plans, Succession Planning for readiness, and Recruiting for internal mobility.

**Bounded Context:** Talent Assessment & Calibration
**Service Name:** `talent-review-service`
**Database:** `data/talent_review.db`
**HTTP Port:** 8164 | **gRPC Port:** 9164

---

## 2. Database Schema

### 2.1 Review Models
```sql
CREATE TABLE tr_review_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL CHECK(model_type IN ('NINE_BOX','FOUR_QUADRANT','THREE_BY_THREE','CUSTOM_GRID')),
    performance_axis TEXT NOT NULL,         -- JSON: { "low": "Below", "mid": "Meets", "high": "Exceeds" }
    potential_axis TEXT NOT NULL,           -- JSON: { "low": "Limited", "mid": "Moderate", "high": "High" }
    grid_dimensions TEXT NOT NULL,          -- JSON: { "rows": 3, "cols": 3 }
    category_definitions TEXT NOT NULL,     -- JSON: per-cell label, description, color
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_tr_model_tenant ON tr_review_models(tenant_id, status);
```

### 2.2 Review Sessions
```sql
CREATE TABLE tr_review_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_name TEXT NOT NULL,
    session_type TEXT NOT NULL CHECK(session_type IN ('ANNUAL','MID_YEAR','QUARTERLY','ADHOC')),
    review_model_id TEXT NOT NULL,
    organization_id TEXT NOT NULL,          -- Scope: department, division, etc.
    review_period TEXT NOT NULL,            -- "2024"
    facilitator_id TEXT NOT NULL,
    participants TEXT NOT NULL,             -- JSON array of reviewer user IDs
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','PREPARATION','IN_REVIEW','CALIBRATION','COMPLETED','CANCELLED')),
    total_population INTEGER NOT NULL DEFAULT 0,
    reviewed_count INTEGER NOT NULL DEFAULT 0,
    calibration_date TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (review_model_id) REFERENCES tr_review_models(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, session_name, review_period)
);

CREATE INDEX idx_tr_session_tenant ON tr_review_sessions(tenant_id, status);
CREATE INDEX idx_tr_session_org ON tr_review_sessions(tenant_id, organization_id);
```

### 2.3 Talent Assessments
```sql
CREATE TABLE tr_talent_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    job_id TEXT,
    department_id TEXT,
    performance_rating TEXT NOT NULL CHECK(performance_rating IN ('BELOW_EXPECTATIONS','MEETS_EXPECTATIONS','EXCEEDS_EXPECTATIONS','OUTSTANDING')),
    potential_rating TEXT NOT NULL CHECK(potential_rating IN ('LIMITED','MODERATE','HIGH','VERY_HIGH')),
    grid_position TEXT NOT NULL,            -- JSON: { "row": 1, "col": 2, "label": "Work Horse" }
    talent_category TEXT NOT NULL CHECK(talent_category IN (
        'STAR','HIGH_PERFORMER','WORK_HORSE','KEY_CONTRIBUTOR',
        'SOLID_CITIZEN','UNDERPERFORMER_HIGH_POTENTIAL','INCONSISTENT','UNDERPERFORMER'
    )),
    readiness_level TEXT CHECK(readiness_level IN ('READY_NOW','READY_IN_1_YEAR','READY_IN_2_3_YEARS','NOT_READY')),
    flight_risk TEXT CHECK(flight_risk IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    retention_risk TEXT CHECK(retention_risk IN ('LOW','MEDIUM','HIGH')),
    assessed_by TEXT NOT NULL,
    assessment_notes TEXT,
    strengths TEXT,                         -- JSON array
    development_areas TEXT,                 -- JSON array
    recommended_actions TEXT,               -- JSON array: [{ "action": "promote", "timeline": "6mo" }]

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (session_id) REFERENCES tr_review_sessions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, session_id, person_id)
);

CREATE INDEX idx_tr_assess_session ON tr_talent_assessments(session_id);
CREATE INDEX idx_tr_assess_person ON tr_talent_assessments(tenant_id, person_id);
CREATE INDEX idx_tr_assess_category ON tr_talent_assessments(tenant_id, talent_category);
CREATE INDEX idx_tr_assess_flight ON tr_talent_assessments(tenant_id, flight_risk);
```

### 2.4 Calibration Notes
```sql
CREATE TABLE tr_calibration_notes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    reviewer_id TEXT NOT NULL,
    note_type TEXT NOT NULL CHECK(note_type IN ('RATING_JUSTIFICATION','ADJUSTMENT','DISCUSSION_POINT','ACTION_ITEM')),
    content TEXT NOT NULL,
    rating_before TEXT,
    rating_after TEXT,
    adjustment_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (session_id) REFERENCES tr_review_sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_tr_calib_session ON tr_calibration_notes(session_id, person_id);
```

### 2.5 Talent Pools
```sql
CREATE TABLE tr_talent_pools (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pool_name TEXT NOT NULL,
    pool_type TEXT NOT NULL CHECK(pool_type IN ('LEADERSHIP','TECHNICAL','HIGH_POTENTIAL','DIVERSITY','SUCCESSION','CUSTOM')),
    description TEXT,
    criteria TEXT NOT NULL,                 -- JSON: eligibility criteria
    max_members INTEGER,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, pool_name)
);

CREATE TABLE tr_talent_pool_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pool_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    added_reason TEXT NOT NULL,
    added_date TEXT NOT NULL,
    review_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','GRADUATED','REMOVED','ON_HOLD')),
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (pool_id) REFERENCES tr_talent_pools(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, pool_id, person_id)
);

CREATE INDEX idx_tr_pool_tenant ON tr_talent_pools(tenant_id, pool_type);
CREATE INDEX idx_tr_pool_member ON tr_talent_pool_members(pool_id, status);
```

### 2.6 Talent Action Plans
```sql
CREATE TABLE tr_action_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN (
        'PROMOTE','DEVELOP','ROTATE','MENTOR','RETAIN','COACH','PERFORMANCE_PLAN','SUCCESSION_PLAN','OTHER'
    )),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    assigned_to TEXT NOT NULL,
    target_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','CANCELLED')),
    completion_notes TEXT,
    linked_development_plan_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (session_id) REFERENCES tr_review_sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_tr_action_session ON tr_action_plans(session_id, status);
CREATE INDEX idx_tr_action_person ON tr_action_plans(tenant_id, person_id);
```

---

## 3. API Endpoints

### 3.1 Review Models
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-review/models` | Create review model |
| GET | `/api/v1/talent-review/models` | List models |
| GET | `/api/v1/talent-review/models/{id}` | Get model details |
| PUT | `/api/v1/talent-review/models/{id}` | Update model |

### 3.2 Review Sessions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-review/sessions` | Create review session |
| GET | `/api/v1/talent-review/sessions` | List sessions |
| GET | `/api/v1/talent-review/sessions/{id}` | Get session details |
| PUT | `/api/v1/talent-review/sessions/{id}` | Update session |
| POST | `/api/v1/talent-review/sessions/{id}/start` | Start review process |
| POST | `/api/v1/talent-review/sessions/{id}/calibrate` | Start calibration |
| POST | `/api/v1/talent-review/sessions/{id}/complete` | Complete session |

### 3.3 Assessments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-review/assessments` | Submit assessment |
| GET | `/api/v1/talent-review/assessments` | List assessments |
| GET | `/api/v1/talent-review/assessments/{id}` | Get assessment details |
| PUT | `/api/v1/talent-review/assessments/{id}` | Update assessment |
| GET | `/api/v1/talent-review/sessions/{id}/matrix` | Get 9-box matrix view |
| GET | `/api/v1/talent-review/sessions/{id}/distribution` | Get rating distribution |

### 3.4 Calibration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-review/calibration-notes` | Add calibration note |
| GET | `/api/v1/talent-review/sessions/{id}/calibration-notes` | List notes |
| POST | `/api/v1/talent-review/assessments/{id}/adjust` | Adjust rating during calibration |

### 3.5 Talent Pools & Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-review/pools` | Create talent pool |
| GET | `/api/v1/talent-review/pools` | List pools |
| POST | `/api/v1/talent-review/pools/{id}/members` | Add member to pool |
| POST | `/api/v1/talent-review/action-plans` | Create action plan |
| GET | `/api/v1/talent-review/action-plans` | List action plans |
| PUT | `/api/v1/talent-review/action-plans/{id}` | Update action plan |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `talent.session.started` | `{ session_id, population_count }` | Review session started |
| `talent.assessment.submitted` | `{ session_id, person_id, category }` | Assessment submitted |
| `talent.session.calibrated` | `{ session_id, adjustments_count }` | Calibration completed |
| `talent.pool.member_added` | `{ pool_id, person_id }` | Person added to talent pool |
| `talent.action.created` | `{ action_id, person_id, action_type }` | Talent action plan created |
| `talent.flight_risk.escalated` | `{ person_id, risk_level, department }` | Flight risk escalation |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `performance.review.completed` | Performance Mgmt | Load performance rating |
| `succession.candidate.evaluated` | Succession Planning | Update readiness level |
| `career.plan.updated` | Career Development | Update development areas |
| `employee.terminated` | Core HR | Remove from active reviews |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.talent_review.v1;

service TalentReviewService {
    rpc GetReviewModel(GetReviewModelRequest) returns (GetReviewModelResponse);
    rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);
    rpc GetSession(GetSessionRequest) returns (GetSessionResponse);
    rpc SubmitAssessment(SubmitAssessmentRequest) returns (SubmitAssessmentResponse);
    rpc GetTalentPool(GetTalentPoolRequest) returns (GetTalentPoolResponse);
    rpc CreateActionPlan(CreateActionPlanRequest) returns (CreateActionPlanResponse);
}

// Review Model messages
message GetReviewModelRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetReviewModelResponse {
    ReviewModel data = 1;
}

message ReviewModel {
    string id = 1;
    string tenant_id = 2;
    string model_code = 3;
    string model_name = 4;
    string model_type = 5;
    string performance_axis = 6;
    string potential_axis = 7;
    string grid_dimensions = 8;
    string category_definitions = 9;
    string description = 10;
    string status = 11;
    string created_at = 12;
    string updated_at = 13;
}

// Session messages
message CreateSessionRequest {
    string tenant_id = 1;
    string session_name = 2;
    string session_type = 3;
    string review_model_id = 4;
    string organization_id = 5;
    string review_period = 6;
    string facilitator_id = 7;
    string participants = 8;
}

message CreateSessionResponse {
    ReviewSession data = 1;
}

message GetSessionRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetSessionResponse {
    ReviewSession data = 1;
}

message ReviewSession {
    string id = 1;
    string tenant_id = 2;
    string session_name = 3;
    string session_type = 4;
    string review_model_id = 5;
    string organization_id = 6;
    string review_period = 7;
    string facilitator_id = 8;
    string participants = 9;
    string status = 10;
    int32 total_population = 11;
    int32 reviewed_count = 12;
    string calibration_date = 13;
    string notes = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Assessment messages
message SubmitAssessmentRequest {
    string tenant_id = 1;
    string session_id = 2;
    string person_id = 3;
    string job_id = 4;
    string department_id = 5;
    string performance_rating = 6;
    string potential_rating = 7;
    string grid_position = 8;
    string talent_category = 9;
    string readiness_level = 10;
    string flight_risk = 11;
    string retention_risk = 12;
    string assessed_by = 13;
    string assessment_notes = 14;
    string strengths = 15;
    string development_areas = 16;
    string recommended_actions = 17;
}

message SubmitAssessmentResponse {
    TalentAssessment data = 1;
}

message TalentAssessment {
    string id = 1;
    string tenant_id = 2;
    string session_id = 3;
    string person_id = 4;
    string job_id = 5;
    string department_id = 6;
    string performance_rating = 7;
    string potential_rating = 8;
    string grid_position = 9;
    string talent_category = 10;
    string readiness_level = 11;
    string flight_risk = 12;
    string retention_risk = 13;
    string assessed_by = 14;
    string assessment_notes = 15;
    string strengths = 16;
    string development_areas = 17;
    string recommended_actions = 18;
    string created_at = 19;
    string updated_at = 20;
}

// Talent Pool messages
message GetTalentPoolRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetTalentPoolResponse {
    TalentPool data = 1;
}

message TalentPool {
    string id = 1;
    string tenant_id = 2;
    string pool_name = 3;
    string pool_type = 4;
    string description = 5;
    string criteria = 6;
    int32 max_members = 7;
    string owner_id = 8;
    string status = 9;
    string created_at = 10;
    string updated_at = 11;
}

// Action Plan messages
message CreateActionPlanRequest {
    string tenant_id = 1;
    string session_id = 2;
    string person_id = 3;
    string action_type = 4;
    string title = 5;
    string description = 6;
    string assigned_to = 7;
    string target_date = 8;
}

message CreateActionPlanResponse {
    ActionPlan data = 1;
}

message ActionPlan {
    string id = 1;
    string tenant_id = 2;
    string session_id = 3;
    string person_id = 4;
    string action_type = 5;
    string title = 6;
    string description = 7;
    string assigned_to = 8;
    string target_date = 9;
    string status = 10;
    string completion_notes = 11;
    string linked_development_plan_id = 12;
    string created_at = 13;
    string updated_at = 14;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | tr_review_models | — |
| V002 | tr_review_sessions | V001 |
| V003 | tr_talent_assessments | V002 |
| V004 | tr_calibration_notes | V002 |
| V005 | tr_talent_pools | — |
| V006 | tr_talent_pool_members | V005 |
| V007 | tr_action_plans | V002 |

---

## 7. Business Rules

1. **Calibration Required**: All assessments must go through calibration before finalization
2. **Dual Review**: Each person assessed by at least 2 reviewers (manager + skip-level)
3. **Forced Distribution**: Optional forced distribution curve (configurable percentages per category)
4. **Flight Risk Triggers**: Auto-escalate HIGH and CRITICAL flight risk to HR business partner
5. **Confidentiality**: Assessment data restricted to session participants and HR admins
6. **Talent Pool Auto-Populate**: Auto-suggest pool membership based on assessment results
7. **Action Plan Tracking**: All HIGH_PERFORMER and STAR categories must have action plans

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| performance-service | `GetReviewRating` | Load performance rating for assessment |
| succession-service | `GetCandidateReadiness` | Read succession readiness level |
| career-service | `GetDevelopmentPlan` | Get development areas |
| core-hr-service | `GetEmployee` | Get employee data and org hierarchy |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| recruiting-service | `GetTalentCategory` | Read talent category for mobility |
| compensation-service | `GetFlightRisk` | Get flight/retention risk for merit |
| learning-service | `GetActionPlans` | Get development action plans |
| dynamic-skills-service | `GetAssessmentSkills` | Read skill assessment data |
