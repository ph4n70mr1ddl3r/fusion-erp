# 179 - CX Sales Engagement Service Specification

## 1. Domain Overview

CX Sales Engagement provides sales sequence management, multi-step outreach cadences, engagement tracking, and sales activity automation for sales representatives. Supports configurable sales sequences with automated email, call, and task steps, A/B testing of outreach templates, engagement analytics (open/click/reply rates), and AI-powered optimal timing recommendations. Enables sales teams to execute structured outreach programs at scale while personalizing individual interactions. Integrates with Sales Automation for CRM data, Marketing for content, and Customer Data Platform for prospect intelligence.

**Bounded Context:** Sales Sequences & Outreach Automation
**Service Name:** `sales-engagement-service`
**Database:** `data/sales_engagement.db`
**HTTP Port:** 8197 | **gRPC Port:** 9197

---

## 2. Database Schema

### 2.1 Sales Sequences
```sql
CREATE TABLE se_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_code TEXT NOT NULL,
    sequence_name TEXT NOT NULL,
    sequence_type TEXT NOT NULL CHECK(sequence_type IN ('PROSPECTING','NURTURE','RENEWAL','UPSELL','EVENT_FOLLOWUP','CUSTOM')),
    description TEXT,
    target_persona TEXT,
    target_segment TEXT,
    total_steps INTEGER NOT NULL DEFAULT 0,
    estimated_duration_days INTEGER NOT NULL DEFAULT 14,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','ARCHIVED')),
    owner_id TEXT NOT NULL,
    created_from_template TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, sequence_code)
);

CREATE INDEX idx_se_seq_tenant ON se_sequences(tenant_id, status);
CREATE INDEX idx_se_seq_owner ON se_sequences(owner_id);
```

### 2.2 Sequence Steps
```sql
CREATE TABLE se_sequence_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    step_type TEXT NOT NULL CHECK(step_type IN ('EMAIL','CALL','TASK','SOCIAL','SMS','LINKEDIN','CUSTOM')),
    subject TEXT,                           -- Email subject
    body_template TEXT NOT NULL,            -- Email/task body with {{variables}}
    call_script TEXT,                       -- For call steps
    delay_days INTEGER NOT NULL DEFAULT 1,  -- Days after previous step
    delay_time TEXT,                        -- Time of day to execute
    auto_execute INTEGER NOT NULL DEFAULT 0,
    required INTEGER NOT NULL DEFAULT 1,
    conditions TEXT,                        -- JSON: conditional execution rules
    template_variant TEXT,                  -- For A/B testing: "A" or "B"

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (sequence_id) REFERENCES se_sequences(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, sequence_id, step_number)
);
```

### 2.3 Sequence Enrollments
```sql
CREATE TABLE se_enrollments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_id TEXT NOT NULL,
    contact_id TEXT NOT NULL,
    contact_email TEXT NOT NULL,
    contact_name TEXT NOT NULL,
    sales_rep_id TEXT NOT NULL,
    opportunity_id TEXT,
    current_step INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','COMPLETED','BOUNCED','OPTED_OUT','CONVERTED','FAILED')),
    enrolled_at TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at TEXT,
    last_step_executed_at TEXT,
    next_step_at TEXT,
    bounce_reason TEXT,
    opt_out_reason TEXT,
    conversion_details TEXT,                -- JSON: what they converted to

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (sequence_id) REFERENCES se_sequences(id) ON DELETE RESTRICT
);

CREATE INDEX idx_se_enroll_seq ON se_enrollments(sequence_id, status);
CREATE INDEX idx_se_enroll_rep ON se_enrollments(sales_rep_id, status);
CREATE INDEX idx_se_enroll_contact ON se_enrollments(contact_id);
CREATE INDEX idx_se_enroll_next ON se_enrollments(next_step_at, status);
```

### 2.4 Engagement Events
```sql
CREATE TABLE se_engagement_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    enrollment_id TEXT NOT NULL,
    step_id TEXT NOT NULL,
    contact_id TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN (
        'EMAIL_SENT','EMAIL_OPENED','EMAIL_CLICKED','EMAIL_REPLIED',
        'EMAIL_BOUNCED','CALL_MADE','CALL_CONNECTED','CALL_VOICEMAIL',
        'TASK_COMPLETED','LINK_SHARED','MEETING_BOOKED','OPT_OUT','UNSUBSCRIBE'
    )),
    event_data TEXT,                        -- JSON: event-specific data
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_se_event_enroll ON se_engagement_events(enrollment_id, timestamp DESC);
CREATE INDEX idx_se_event_type ON se_engagement_events(tenant_id, event_type);
CREATE INDEX idx_se_event_contact ON se_engagement_events(contact_id, timestamp DESC);
```

### 2.5 Engagement Analytics
```sql
CREATE TABLE se_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_id TEXT NOT NULL,
    period TEXT NOT NULL,
    total_enrolled INTEGER NOT NULL DEFAULT 0,
    total_completed INTEGER NOT NULL DEFAULT 0,
    total_converted INTEGER NOT NULL DEFAULT 0,
    conversion_rate REAL NOT NULL DEFAULT 0,
    emails_sent INTEGER NOT NULL DEFAULT 0,
    email_open_rate REAL NOT NULL DEFAULT 0,
    email_click_rate REAL NOT NULL DEFAULT 0,
    email_reply_rate REAL NOT NULL DEFAULT 0,
    email_bounce_rate REAL NOT NULL DEFAULT 0,
    calls_made INTEGER NOT NULL DEFAULT 0,
    call_connect_rate REAL NOT NULL DEFAULT 0,
    meetings_booked INTEGER NOT NULL DEFAULT 0,
    avg_steps_to_convert REAL NOT NULL DEFAULT 0,
    avg_days_to_convert REAL NOT NULL DEFAULT 0,
    opt_out_rate REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, sequence_id, period)
);

CREATE INDEX idx_se_analytics_seq ON se_analytics(sequence_id, period DESC);
```

---

## 3. API Endpoints

### 3.1 Sequences
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sales-engagement/sequences` | Create sequence |
| GET | `/api/v1/sales-engagement/sequences` | List sequences |
| GET | `/api/v1/sales-engagement/sequences/{id}` | Get sequence with steps |
| PUT | `/api/v1/sales-engagement/sequences/{id}` | Update sequence |
| POST | `/api/v1/sales-engagement/sequences/{id}/activate` | Activate sequence |

### 3.2 Sequence Steps
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sales-engagement/sequences/{id}/steps` | Add step |
| PUT | `/api/v1/sales-engagement/steps/{id}` | Update step |
| DELETE | `/api/v1/sales-engagement/steps/{id}` | Remove step |

### 3.3 Enrollments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sales-engagement/enroll` | Enroll contacts |
| POST | `/api/v1/sales-engagement/enroll/bulk` | Bulk enroll |
| GET | `/api/v1/sales-engagement/enrollments` | List enrollments |
| POST | `/api/v1/sales-engagement/enrollments/{id}/pause` | Pause enrollment |
| POST | `/api/v1/sales-engagement/enrollments/{id}/resume` | Resume enrollment |
| POST | `/api/v1/sales-engagement/enrollments/{id}/opt-out` | Process opt-out |

### 3.4 Engagement & Analytics
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sales-engagement/events` | Track engagement event |
| GET | `/api/v1/sales-engagement/sequences/{id}/analytics` | Sequence analytics |
| GET | `/api/v1/sales-engagement/analytics/team` | Team performance |
| GET | `/api/v1/sales-engagement/contacts/{id}/timeline` | Contact engagement timeline |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sengage.enrollment.created` | `{ enrollment_id, contact_id, sequence_id }` | Contact enrolled |
| `sengage.enrollment.converted` | `{ enrollment_id, opportunity_id }` | Enrollment converted |
| `sengage.email.opened` | `{ enrollment_id, step_id }` | Email opened |
| `sengage.email.replied` | `{ enrollment_id, step_id }` | Email replied |
| `sengage.opt_out` | `{ contact_id, sequence_id }` | Contact opted out |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `opportunity.created` | Sales Automation | Auto-enroll prospect sequence |
| `lead.assigned` | Sales Automation | Auto-enroll nurture sequence |
| `email.bounced` | Marketing | Mark enrollment as bounced |

---

## 5. Business Rules

1. **Duplicate Prevention**: Contacts can only be in one active sequence at a time
2. **Opt-Out Compliance**: Opt-out immediately stops all sequence steps for contact
3. **Bounce Handling**: Hard bounces auto-stop sequence; soft bounces retry once
4. **Auto-Completion**: Sequence completed when all steps executed or contact converts
5. **Business Hours**: Emails and calls sent only during recipient's business hours
6. **Step Timing**: Configurable delay between steps; AI suggests optimal send times
7. **A/B Testing**: Step templates can have variants; auto-select winner by engagement

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Sales Automation (77) | Contact and opportunity data |
| Marketing (61) | Email templates and content |
| Customer Data Platform (60) | Contact intelligence |
| Email Gateway | Email delivery and tracking |
| Telephony Integration | Call tracking and logging |
| Calendar Integration | Meeting scheduling |
| CX Analytics (131) | Unified CX performance |
| Notification Center (165) | Step reminders for reps |
