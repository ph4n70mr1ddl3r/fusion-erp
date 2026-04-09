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
| POST | `/talent-review/v1/models` | Create review model |
| GET | `/talent-review/v1/models` | List models |
| GET | `/talent-review/v1/models/{id}` | Get model details |
| PUT | `/talent-review/v1/models/{id}` | Update model |

### 3.2 Review Sessions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/talent-review/v1/sessions` | Create review session |
| GET | `/talent-review/v1/sessions` | List sessions |
| GET | `/talent-review/v1/sessions/{id}` | Get session details |
| PUT | `/talent-review/v1/sessions/{id}` | Update session |
| POST | `/talent-review/v1/sessions/{id}/start` | Start review process |
| POST | `/talent-review/v1/sessions/{id}/calibrate` | Start calibration |
| POST | `/talent-review/v1/sessions/{id}/complete` | Complete session |

### 3.3 Assessments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/talent-review/v1/assessments` | Submit assessment |
| GET | `/talent-review/v1/assessments` | List assessments |
| GET | `/talent-review/v1/assessments/{id}` | Get assessment details |
| PUT | `/talent-review/v1/assessments/{id}` | Update assessment |
| GET | `/talent-review/v1/sessions/{id}/matrix` | Get 9-box matrix view |
| GET | `/talent-review/v1/sessions/{id}/distribution` | Get rating distribution |

### 3.4 Calibration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/talent-review/v1/calibration-notes` | Add calibration note |
| GET | `/talent-review/v1/sessions/{id}/calibration-notes` | List notes |
| POST | `/talent-review/v1/assessments/{id}/adjust` | Adjust rating during calibration |

### 3.5 Talent Pools & Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/talent-review/v1/pools` | Create talent pool |
| GET | `/talent-review/v1/pools` | List pools |
| POST | `/talent-review/v1/pools/{id}/members` | Add member to pool |
| POST | `/talent-review/v1/action-plans` | Create action plan |
| GET | `/talent-review/v1/action-plans` | List action plans |
| PUT | `/talent-review/v1/action-plans/{id}` | Update action plan |

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

## 5. Business Rules

1. **Calibration Required**: All assessments must go through calibration before finalization
2. **Dual Review**: Each person assessed by at least 2 reviewers (manager + skip-level)
3. **Forced Distribution**: Optional forced distribution curve (configurable percentages per category)
4. **Flight Risk Triggers**: Auto-escalate HIGH and CRITICAL flight risk to HR business partner
5. **Confidentiality**: Assessment data restricted to session participants and HR admins
6. **Talent Pool Auto-Populate**: Auto-suggest pool membership based on assessment results
7. **Action Plan Tracking**: All HIGH_PERFORMER and STAR categories must have action plans

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Performance Management (68) | Performance ratings input |
| Succession Planning (70) | Succession readiness data |
| Career Development (71) | Development plans, career paths |
| Recruiting (67) | Internal mobility candidates |
| Core HR (62) | Employee data, organization hierarchy |
| Learning & Development (69) | Development action tracking |
| Dynamic Skills (94) | Skills assessment data |
| Compensation (65) | Retention and merit recommendations |
