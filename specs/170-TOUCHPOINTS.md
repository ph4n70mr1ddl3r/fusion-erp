# 170 - Touchpoints (Continuous Check-ins) Service Specification

## 1. Domain Overview

Touchpoints provides continuous feedback, 1-on-1 meeting management, and check-in workflows between employees and managers. Supports structured and ad-hoc check-ins, meeting agenda templates, discussion topics, feedback exchange, action item tracking, and sentiment tracking. Enables organizations to move from annual reviews to continuous performance conversations with documented touchpoints. Integrates with Performance Management, Goals, Career Development, and Notification Center.

**Bounded Context:** Continuous Feedback & Check-in Management
**Service Name:** `touchpoints-service`
**Database:** `data/touchpoints.db`
**HTTP Port:** 8188 | **gRPC Port:** 9188

---

## 2. Database Schema

### 2.1 Check-in Templates
```sql
CREATE TABLE tp_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_type TEXT NOT NULL CHECK(template_type IN ('WEEKLY_1ON1','MONTHLY_CHECKIN','QUARTERLY_REVIEW','ADHOC_FEEDBACK','ONBOARDING','PEER_FEEDBACK','CUSTOM')),
    description TEXT,
    frequency TEXT NOT NULL CHECK(frequency IN ('WEEKLY','BIWEEKLY','MONTHLY','QUARTERLY','ADHOC')),
    agenda_items TEXT NOT NULL,             -- JSON: [{ "topic": "...", "required": true }]
    discussion_prompts TEXT,                -- JSON: guided questions
    duration_minutes INTEGER NOT NULL DEFAULT 30,
    participants TEXT NOT NULL CHECK(participants IN ('MANAGER_EMPLOYEE','PEER','SKIP_LEVEL','TEAM','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_name)
);
```

### 2.2 Scheduled Check-ins
```sql
CREATE TABLE tp_scheduled_checkins (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    checkin_title TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    scheduled_date TEXT NOT NULL,
    scheduled_time TEXT NOT NULL,
    duration_minutes INTEGER NOT NULL DEFAULT 30,
    recurrence TEXT CHECK(recurrence IN ('NONE','WEEKLY','BIWEEKLY','MONTHLY')),
    status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(status IN ('SCHEDULED','CONFIRMED','IN_PROGRESS','COMPLETED','CANCELLED','MISSED','RESCHEDULED')),
    actual_date TEXT,
    actual_duration_minutes INTEGER,
    location TEXT,
    meeting_url TEXT,
    notes TEXT,
    sentiment TEXT CHECK(sentiment IN ('POSITIVE','NEUTRAL','CONCERNED','NEGATIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (template_id) REFERENCES tp_templates(id) ON DELETE SET NULL
);

CREATE INDEX idx_tp_sched_employee ON tp_scheduled_checkins(employee_id, scheduled_date DESC);
CREATE INDEX idx_tp_sched_manager ON tp_scheduled_checkins(manager_id, status);
CREATE INDEX idx_tp_sched_date ON tp_scheduled_checkins(tenant_id, scheduled_date, status);
```

### 2.3 Check-in Agendas
```sql
CREATE TABLE tp_agenda_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    checkin_id TEXT NOT NULL,
    topic TEXT NOT NULL,
    topic_type TEXT NOT NULL CHECK(topic_type IN ('GOALS','PROJECT','CAREER','WELLBEING','TEAM','FEEDBACK','BLOCKER','CUSTOM')),
    added_by TEXT NOT NULL,
    notes TEXT,
    discussion_notes TEXT,
    is_discussed INTEGER NOT NULL DEFAULT 0,
    priority INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (checkin_id) REFERENCES tp_scheduled_checkins(id) ON DELETE CASCADE
);

CREATE INDEX idx_tp_agenda_checkin ON tp_agenda_items(checkin_id);
```

### 2.4 Feedback Exchange
```sql
CREATE TABLE tp_feedback (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    checkin_id TEXT,
    from_person_id TEXT NOT NULL,
    to_person_id TEXT NOT NULL,
    feedback_type TEXT NOT NULL CHECK(feedback_type IN ('MANAGER_TO_EMPLOYEE','EMPLOYEE_TO_MANAGER','PEER','UPWARD','SELF','KUDOS')),
    category TEXT NOT NULL CHECK(category IN ('RECOGNITION','CONSTRUCTIVE','COACHING','SUGGESTION','GENERAL')),
    content TEXT NOT NULL,
    linked_goal_id TEXT,
    visibility TEXT NOT NULL DEFAULT 'PRIVATE'
        CHECK(visibility IN ('PRIVATE','SHARED_WITH_MANAGER','PUBLIC','HR_VISIBLE')),
    sentiment TEXT CHECK(sentiment IN ('POSITIVE','NEUTRAL','CONSTRUCTIVE','NEGATIVE')),
    tags TEXT,                              -- JSON array
    acknowledged INTEGER NOT NULL DEFAULT 0,
    acknowledged_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (checkin_id) REFERENCES tp_scheduled_checkins(id) ON DELETE SET NULL
);

CREATE INDEX idx_tp_feedback_to ON tp_feedback(to_person_id, created_at DESC);
CREATE INDEX idx_tp_feedback_from ON tp_feedback(from_person_id, created_at DESC);
CREATE INDEX idx_tp_feedback_type ON tp_feedback(tenant_id, feedback_type);
```

### 2.5 Action Items
```sql
CREATE TABLE tp_action_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    checkin_id TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    assigned_to TEXT NOT NULL,
    due_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_PROGRESS','COMPLETED','CANCELLED')),
    completed_date TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (checkin_id) REFERENCES tp_scheduled_checkins(id) ON DELETE CASCADE
);

CREATE INDEX idx_tp_action_assignee ON tp_action_items(assigned_to, status);
CREATE INDEX idx_tp_action_due ON tp_action_items(tenant_id, due_date, status);
```

### 2.6 Touchpoint Analytics
```sql
CREATE TABLE tp_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-Q1"
    manager_id TEXT,
    department_id TEXT,
    total_checkins INTEGER NOT NULL DEFAULT 0,
    completed_checkins INTEGER NOT NULL DEFAULT 0,
    missed_checkins INTEGER NOT NULL DEFAULT 0,
    avg_duration_minutes REAL NOT NULL DEFAULT 0,
    feedback_given_count INTEGER NOT NULL DEFAULT 0,
    feedback_received_count INTEGER NOT NULL DEFAULT 0,
    action_items_created INTEGER NOT NULL DEFAULT 0,
    action_items_completed INTEGER NOT NULL DEFAULT 0,
    avg_sentiment_score REAL NOT NULL DEFAULT 0,
    checkin_frequency_score REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, period, manager_id, department_id)
);

CREATE INDEX idx_tp_analytics_period ON tp_analytics(tenant_id, period DESC);
```

---

## 3. API Endpoints

### 3.1 Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/touchpoints/v1/templates` | Create template |
| GET | `/touchpoints/v1/templates` | List templates |
| PUT | `/touchpoints/v1/templates/{id}` | Update template |

### 3.2 Check-ins
| Method | Path | Description |
|--------|------|-------------|
| POST | `/touchpoints/v1/checkins` | Schedule check-in |
| GET | `/touchpoints/v1/checkins` | List check-ins |
| GET | `/touchpoints/v1/checkins/{id}` | Get check-in details |
| PUT | `/touchpoints/v1/checkins/{id}` | Update check-in |
| POST | `/touchpoints/v1/checkins/{id}/start` | Start check-in |
| POST | `/touchpoints/v1/checkins/{id}/complete` | Complete check-in |
| POST | `/touchpoints/v1/checkins/{id}/cancel` | Cancel check-in |

### 3.3 Agenda & Feedback
| Method | Path | Description |
|--------|------|-------------|
| POST | `/touchpoints/v1/checkins/{id}/agenda` | Add agenda item |
| PUT | `/touchpoints/v1/agenda/{id}` | Update agenda item |
| POST | `/touchpoints/v1/feedback` | Submit feedback |
| GET | `/touchpoints/v1/feedback` | List feedback |
| POST | `/touchpoints/v1/feedback/{id}/acknowledge` | Acknowledge feedback |

### 3.4 Action Items & Analytics
| Method | Path | Description |
|--------|------|-------------|
| POST | `/touchpoints/v1/actions` | Create action item |
| PUT | `/touchpoints/v1/actions/{id}` | Update action item |
| GET | `/touchpoints/v1/analytics` | Touchpoint analytics |
| GET | `/touchpoints/v1/dashboard` | Manager dashboard |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `touch.checkin.scheduled` | `{ checkin_id, employee_id, date }` | Check-in scheduled |
| `touch.checkin.completed` | `{ checkin_id, duration, sentiment }` | Check-in completed |
| `touch.checkin.missed` | `{ checkin_id, manager_id, employee_id }` | Check-in missed |
| `touch.feedback.given` | `{ feedback_id, from, to, type }` | Feedback submitted |
| `touch.action.overdue` | `{ action_id, assigned_to, due_date }` | Action item overdue |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `goal.at_risk` | Goals Management | Add goal discussion to agenda |
| `performance.cycle.started` | Performance Mgmt | Schedule review check-ins |
| `employee.hired` | Core HR | Schedule onboarding check-ins |

---

## 5. Business Rules

1. **Check-in Frequency**: Minimum monthly check-in enforced; managers alerted for missed check-ins
2. **Agenda Collaboration**: Both manager and employee can add agenda items before the meeting
3. **Feedback Privacy**: Private feedback visible only to giver and receiver; HR_VISIBLE includes HR admin
4. **Action Tracking**: All action items tracked with due dates; overdue items escalated
5. **Sentiment Analysis**: Manager sentiment auto-assessed from discussion notes (opt-in)
6. **Missed Check-in Escalation**: 3+ consecutive missed check-ins escalated to skip-level manager
7. **Template Enforcement**: Structured check-ins follow template agenda; free-form allowed

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Performance Management (68) | Check-in data feeds reviews |
| Goals Management (169) | Goal progress discussions |
| Career Development (71) | Career conversation topics |
| Manager Edge (106) | Manager coaching prompts |
| Notification Center (165) | Check-in reminders |
| HCM Analytics (148) | Check-in frequency and quality metrics |
| Employee Experience (95) | Employee check-in portal |
| Core HR (62) | Employee/manager relationships |
