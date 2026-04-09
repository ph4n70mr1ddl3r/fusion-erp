# 165 - Notification Center Service Specification

## 1. Domain Overview

Notification Center provides a centralized notification framework for delivering alerts, reminders, action items, and informational messages across all Fusion modules. Supports multi-channel delivery (in-app, email, SMS, push, webhook), user preference management, notification grouping and digesting, priority escalation, and read/acknowledge tracking. Provides a unified notification inbox for users with filtering, archiving, and batch operations. Integrates with all modules as the central notification delivery engine.

**Bounded Context:** Centralized Notification & Alert Management
**Service Name:** `notification-service`
**Database:** `data/notification.db`
**HTTP Port:** 8183 | **gRPC Port:** 9183

---

## 2. Database Schema

### 2.1 Notification Definitions
```sql
CREATE TABLE nc_notification_types (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    type_code TEXT NOT NULL,
    type_name TEXT NOT NULL,
    source_module TEXT NOT NULL,
    category TEXT NOT NULL CHECK(category IN (
        'ACTION_REQUIRED','APPROVAL','ALERT','REMINDER','INFORMATION','SYSTEM','SOCIAL'
    )),
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    default_channels TEXT NOT NULL,         -- JSON: ["IN_APP","EMAIL"]
    template_id TEXT,
    description TEXT,
    expiry_hours INTEGER NOT NULL DEFAULT 720,
    actionable INTEGER NOT NULL DEFAULT 0,
    action_url_template TEXT,
    grouping_key TEXT,                      -- Key for grouping similar notifications
    max_per_group INTEGER NOT NULL DEFAULT 50,
    digestible INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, type_code)
);

CREATE INDEX idx_nc_type_module ON nc_notification_types(tenant_id, source_module);
CREATE INDEX idx_nc_type_category ON nc_notification_types(tenant_id, category);
```

### 2.2 User Preferences
```sql
CREATE TABLE nc_user_preferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    type_code TEXT,                         -- NULL = global preferences
    channel_in_app INTEGER NOT NULL DEFAULT 1,
    channel_email INTEGER NOT NULL DEFAULT 1,
    channel_sms INTEGER NOT NULL DEFAULT 0,
    channel_push INTEGER NOT NULL DEFAULT 0,
    channel_webhook_url TEXT,
    digest_enabled INTEGER NOT NULL DEFAULT 0,
    digest_frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(digest_frequency IN ('IMMEDIATE','HOURLY','DAILY','WEEKLY')),
    quiet_hours_start TEXT,                 -- e.g., "22:00"
    quiet_hours_end TEXT,                   -- e.g., "07:00"
    quiet_hours_timezone TEXT DEFAULT 'UTC',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, user_id, type_code)
);

CREATE INDEX idx_nc_pref_user ON nc_user_preferences(user_id);
```

### 2.3 Notifications
```sql
CREATE TABLE nc_notifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    type_code TEXT NOT NULL,
    recipient_id TEXT NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    rich_body TEXT,                         -- HTML body for email
    source_module TEXT NOT NULL,
    source_id TEXT,                         -- Reference to source record
    action_url TEXT,
    action_label TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'UNREAD'
        CHECK(status IN ('UNREAD','READ','ACKNOWLEDGED','ACTED_UPON','ARCHIVED','DISMISSED')),
    channels_delivered TEXT NOT NULL,       -- JSON: channels used for delivery
    group_key TEXT,
    expires_at TEXT,
    read_at TEXT,
    acknowledged_at TEXT,
    acted_upon_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(id)
);

CREATE INDEX idx_nc_notif_recipient ON nc_notifications(recipient_id, status, created_at DESC);
CREATE INDEX idx_nc_notif_type ON nc_notifications(tenant_id, type_code);
CREATE INDEX nc_notif_group ON nc_notifications(recipient_id, group_key);
CREATE INDEX nc_notif_expiry ON nc_notifications(expires_at, status);
```

### 2.4 Delivery Log
```sql
CREATE TABLE nc_delivery_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    notification_id TEXT NOT NULL,
    recipient_id TEXT NOT NULL,
    channel TEXT NOT NULL CHECK(channel IN ('IN_APP','EMAIL','SMS','PUSH','WEBHOOK')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','SENT','DELIVERED','FAILED','BOUNCED','BLOCKED')),
    external_message_id TEXT,               -- Email/message service ID
    sent_at TEXT,
    delivered_at TEXT,
    failure_reason TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (notification_id) REFERENCES nc_notifications(id) ON DELETE CASCADE
);

CREATE INDEX idx_nc_delivery_notif ON nc_delivery_log(notification_id);
CREATE INDEX idx_nc_delivery_status ON nc_delivery_log(tenant_id, channel, status);
```

### 2.5 Notification Templates
```sql
CREATE TABLE nc_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    channel TEXT NOT NULL,
    subject_template TEXT,                  -- For email
    body_template TEXT NOT NULL,            -- Template with {{variables}}
    locale TEXT NOT NULL DEFAULT 'en',
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, template_code, channel, locale)
);
```

---

## 3. API Endpoints

### 3.1 Notification Inbox
| Method | Path | Description |
|--------|------|-------------|
| GET | `/notifications/v1/inbox` | Get user's notifications |
| GET | `/notifications/v1/inbox/unread-count` | Get unread count |
| GET | `/notifications/v1/inbox/{id}` | Get notification details |
| POST | `/notifications/v1/inbox/{id}/read` | Mark as read |
| POST | `/notifications/v1/inbox/{id}/acknowledge` | Acknowledge |
| POST | `/notifications/v1/inbox/{id}/dismiss` | Dismiss |
| POST | `/notifications/v1/inbox/{id}/act` | Take action |
| POST | `/notifications/v1/inbox/batch-read` | Batch mark as read |
| POST | `/notifications/v1/inbox/batch-archive` | Batch archive |

### 3.2 Send Notifications
| Method | Path | Description |
|--------|------|-------------|
| POST | `/notifications/v1/send` | Send notification |
| POST | `/notifications/v1/send-batch` | Send to multiple recipients |
| POST | `/notifications/v1/send-template` | Send using template |

### 3.3 User Preferences
| Method | Path | Description |
|--------|------|-------------|
| GET | `/notifications/v1/preferences` | Get user preferences |
| PUT | `/notifications/v1/preferences` | Update preferences |
| POST | `/notifications/v1/preferences/reset` | Reset to defaults |

### 3.4 Admin - Types & Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/notifications/v1/types` | Create notification type |
| GET | `/notifications/v1/types` | List types |
| PUT | `/notifications/v1/types/{id}` | Update type |
| POST | `/notifications/v1/templates` | Create template |
| GET | `/notifications/v1/templates` | List templates |
| PUT | `/notifications/v1/templates/{id}` | Update template |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `notif.created` | `{ notification_id, recipient_id, type }` | Notification created |
| `notif.read` | `{ notification_id, recipient_id }` | Notification read |
| `notif.acknowledged` | `{ notification_id, recipient_id }` | Notification acknowledged |
| `notif.action.taken` | `{ notification_id, action, recipient_id }` | Action taken |
| `notif.delivery.failed` | `{ notification_id, channel, reason }` | Delivery failed |
| `notif.escalated` | `{ notification_id, original_recipient, escalated_to }` | Escalated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `workflow.pending_approval` | Workflow | Send approval notification |
| `*.deadline.approaching` | All modules | Send reminder notification |
| `*.exception.detected` | All modules | Send alert notification |
| `*.action.required` | All modules | Send action notification |

---

## 5. Business Rules

1. **Preference Respect**: Notifications delivered only via channels enabled in user preferences
2. **Quiet Hours**: No push/SMS during quiet hours; queued for delivery after quiet period
3. **Digest Mode**: When enabled, notifications batched into single digest per frequency
4. **Auto-Expiry**: Notifications not acted upon within expiry period auto-archived
5. **Escalation**: URGENT notifications not acknowledged within 4 hours escalated to manager
6. **Rate Limiting**: Max 100 notifications per user per day (configurable); digest above threshold
7. **Grouping**: Notifications with same group_key combined in inbox display
8. **Delivery Guarantee**: At-least-once delivery for all channels; deduplication on client side

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| All Modules | Notification event source |
| Workflow (16) | Approval and task notifications |
| Digital Assistant (43) | Chat-based notifications |
| Email/SMS Gateways | External delivery channels |
| Push Notification Service | Mobile push delivery |
| Mobile (46) | Mobile notification display |
| Enterprise Search (164) | Notification search |
| Auth & Security (05) | User identity for delivery |
