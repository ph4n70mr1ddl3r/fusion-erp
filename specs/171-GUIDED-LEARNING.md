# 171 - Guided Learning Service Specification

## 1. Domain Overview

Guided Learning provides in-app contextual help, step-by-step guided tours, tooltips, walkthroughs, and smart tips embedded within the Fusion application UI. Supports guided journeys for new feature adoption, process walkthroughs, contextual help triggered by user actions, and adaptive guidance based on user role and proficiency. Enables administrators to create and manage guidance content without code changes. Integrates with Frontend for UI overlay injection, Oracle Journeys for guided process steps, and Learning & Development for linked training content.

**Bounded Context:** In-App Guidance & Contextual Help
**Service Name:** `guided-learning-service`
**Database:** `data/guided_learning.db`
**HTTP Port:** 8189 | **gRPC Port:** 9189

---

## 2. Database Schema

### 2.1 Guidance Topics
```sql
CREATE TABLE gl_topics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    topic_code TEXT NOT NULL,
    topic_name TEXT NOT NULL,
    topic_type TEXT NOT NULL CHECK(topic_type IN ('TOUR','WALKTHROUGH','TOOLTIP','SMART_TIP','BEACON','CHECKLIST','VIDEO','CUSTOM')),
    description TEXT,
    target_page TEXT NOT NULL,              -- URL pattern where guidance appears
    target_element TEXT,                    -- CSS selector for element-specific guidance
    target_roles TEXT NOT NULL,             -- JSON: roles that see this guidance
    trigger_type TEXT NOT NULL CHECK(trigger_type IN (
        'PAGE_LOAD','FIRST_VISIT','ELEMENT_HOVER','ELEMENT_CLICK','IDLE','MANUAL','EVENT'
    )),
    trigger_config TEXT,                    -- JSON: trigger-specific parameters
    priority INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','ARCHIVED')),
    version_number INTEGER NOT NULL DEFAULT 1,
    published_at TEXT,
    start_date TEXT,
    end_date TEXT,
    display_frequency TEXT NOT NULL DEFAULT 'ONCE'
        CHECK(display_frequency IN ('ONCE','EVERY_VISIT','DAILY','UNTIL_DISMISSED','ALWAYS')),
    dismissible INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, topic_code)
);

CREATE INDEX idx_gl_topic_page ON gl_topics(tenant_id, target_page, status);
```

### 2.2 Guidance Steps
```sql
CREATE TABLE gl_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    topic_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    step_title TEXT NOT NULL,
    step_content TEXT NOT NULL,             -- Rich text / HTML content
    content_type TEXT NOT NULL DEFAULT 'TEXT'
        CHECK(content_type IN ('TEXT','HTML','VIDEO_URL','IMAGE_URL','MARKDOWN')),
    target_element TEXT,                    -- CSS selector to highlight
    placement TEXT NOT NULL DEFAULT 'RIGHT'
        CHECK(placement IN ('TOP','RIGHT','BOTTOM','LEFT','CENTER','MODAL')),
    action_type TEXT CHECK(action_type IN ('NEXT','CLICK_TARGET','COMPLETE_FORM','WATCH_VIDEO','EXTERNAL_LINK','CUSTOM')),
    action_config TEXT,                     -- JSON: action-specific parameters
    is_optional INTEGER NOT NULL DEFAULT 0,
    wait_for_action INTEGER NOT NULL DEFAULT 0, -- Block until user performs action
    advance_on_complete INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (topic_id) REFERENCES gl_topics(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, topic_id, step_number)
);

CREATE INDEX idx_gl_step_topic ON gl_steps(topic_id, step_number);
```

### 2.3 User Progress
```sql
CREATE TABLE gl_user_progress (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    topic_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','DISMISSED','SKIPPED')),
    current_step INTEGER NOT NULL DEFAULT 0,
    total_steps INTEGER NOT NULL DEFAULT 0,
    completed_steps INTEGER NOT NULL DEFAULT 0,
    started_at TEXT,
    completed_at TEXT,
    dismissed_at TEXT,
    time_spent_seconds INTEGER NOT NULL DEFAULT 0,
    feedback_rating INTEGER CHECK(feedback_rating BETWEEN 1 AND 5),
    feedback_comment TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, user_id, topic_id)
);

CREATE INDEX idx_gl_progress_user ON gl_user_progress(user_id, status);
CREATE INDEX idx_gl_progress_topic ON gl_user_progress(topic_id, status);
```

### 2.4 Smart Tips (Contextual)
```sql
CREATE TABLE gl_smart_tips (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    tip_code TEXT NOT NULL,
    tip_title TEXT NOT NULL,
    tip_content TEXT NOT NULL,
    target_page TEXT NOT NULL,
    target_element TEXT NOT NULL,
    trigger_condition TEXT NOT NULL,         -- JSON: when to show (e.g., field focus, form state)
    audience TEXT NOT NULL,                 -- JSON: role/segment criteria
    priority INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, tip_code)
);
```

### 2.5 Guidance Analytics
```sql
CREATE TABLE gl_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    topic_id TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-W01"
    impressions INTEGER NOT NULL DEFAULT 0,
    starts INTEGER NOT NULL DEFAULT 0,
    completions INTEGER NOT NULL DEFAULT 0,
    dismissals INTEGER NOT NULL DEFAULT 0,
    avg_completion_pct REAL NOT NULL DEFAULT 0,
    avg_time_seconds REAL NOT NULL DEFAULT 0,
    avg_rating REAL,
    step_drop_offs TEXT,                    -- JSON: { step_number: drop_off_count }

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, topic_id, period)
);
```

---

## 3. API Endpoints

### 3.1 Guidance Topics
| Method | Path | Description |
|--------|------|-------------|
| POST | `/guided-learning/v1/topics` | Create topic |
| GET | `/guided-learning/v1/topics` | List topics |
| GET | `/guided-learning/v1/topics/{id}` | Get topic with steps |
| PUT | `/guided-learning/v1/topics/{id}` | Update topic |
| POST | `/guided-learning/v1/topics/{id}/publish` | Publish topic |
| DELETE | `/guided-learning/v1/topics/{id}` | Archive topic |

### 3.2 Steps
| Method | Path | Description |
|--------|------|-------------|
| POST | `/guided-learning/v1/topics/{id}/steps` | Add step |
| PUT | `/guided-learning/v1/steps/{id}` | Update step |
| DELETE | `/guided-learning/v1/steps/{id}` | Remove step |

### 3.3 Runtime (Client API)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/guided-learning/v1/guidance` | Get active guidance for current page/user |
| POST | `/guided-learning/v1/progress` | Record step completion |
| POST | `/guided-learning/v1/feedback` | Submit guidance feedback |
| GET | `/guided-learning/v1/user/{userId}/pending` | Pending guidance for user |

### 3.4 Smart Tips & Analytics
| Method | Path | Description |
|--------|------|-------------|
| POST | `/guided-learning/v1/smart-tips` | Create smart tip |
| GET | `/guided-learning/v1/smart-tips` | List tips |
| GET | `/guided-learning/v1/analytics` | Guidance analytics |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `glearn.topic.published` | `{ topic_id, type, target_page }` | Guidance published |
| `glearn.guidance.started` | `{ topic_id, user_id }` | User started guidance |
| `glearn.guidance.completed` | `{ topic_id, user_id, duration }` | User completed guidance |
| `glearn.guidance.dismissed` | `{ topic_id, user_id, step }` | User dismissed guidance |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `feature.new` | Deployment | Trigger new feature walkthrough |
| `user.role.changed` | Core HR | Show role-specific guidance |
| `journey.step.reached` | Journeys | Trigger contextual guidance |

---

## 5. Business Rules

1. **Role Targeting**: Guidance shown only to users with matching roles
2. **Display Frequency**: ONCE shows only on first visit; every visit shows each session
3. **Priority Ordering**: Higher priority guidance shown first when multiple match
4. **Progress Persistence**: Step progress saved; users resume where they left off
5. **Auto-Dismiss**: Guidance auto-dismissed after configurable timeout
6. **A/B Testing**: Optional A/B variant testing for guidance effectiveness

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Frontend (20) | UI overlay injection |
| Oracle Journeys (147) | Journey step guidance |
| Learning & Development (69) | Training content links |
| Experience Design Studio (109) | Guidance visual customization |
| Redwood UX (151) | Component-aware targeting |
| Core HR (62) | User role data |
| Onboarding (96) | New hire walkthroughs |
