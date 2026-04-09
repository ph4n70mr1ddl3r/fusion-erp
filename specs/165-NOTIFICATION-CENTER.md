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

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.notification.v1;

service NotificationService {
    rpc GetNotification(GetNotificationRequest) returns (GetNotificationResponse);
    rpc SendNotification(SendNotificationRequest) returns (SendNotificationResponse);
    rpc SendBatchNotification(SendBatchNotificationRequest) returns (SendBatchNotificationResponse);
    rpc GetUserPreferences(GetUserPreferencesRequest) returns (GetUserPreferencesResponse);
    rpc UpdateUserPreferences(UpdateUserPreferencesRequest) returns (UpdateUserPreferencesResponse);
    rpc GetUnreadCount(GetUnreadCountRequest) returns (GetUnreadCountResponse);
}

// Notification messages
message GetNotificationRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetNotificationResponse {
    Notification data = 1;
}

message Notification {
    string id = 1;
    string tenant_id = 2;
    string type_code = 3;
    string recipient_id = 4;
    string title = 5;
    string body = 6;
    string rich_body = 7;
    string source_module = 8;
    string source_id = 9;
    string action_url = 10;
    string action_label = 11;
    string priority = 12;
    string status = 13;
    string channels_delivered = 14;
    string group_key = 15;
    string expires_at = 16;
    string read_at = 17;
    string acknowledged_at = 18;
    string created_at = 19;
    string updated_at = 20;
}

message SendNotificationRequest {
    string tenant_id = 1;
    string type_code = 2;
    string recipient_id = 3;
    string title = 4;
    string body = 5;
    string rich_body = 6;
    string source_module = 7;
    string source_id = 8;
    string action_url = 9;
    string action_label = 10;
    string priority = 11;
    string group_key = 12;
}

message SendNotificationResponse {
    Notification data = 1;
}

message SendBatchNotificationRequest {
    string tenant_id = 1;
    string type_code = 2;
    repeated string recipient_ids = 3;
    string title = 4;
    string body = 5;
    string source_module = 6;
    string priority = 7;
}

message SendBatchNotificationResponse {
    int32 sent_count = 1;
    repeated string notification_ids = 2;
}

// User Preference messages
message GetUserPreferencesRequest {
    string tenant_id = 1;
    string user_id = 2;
    string type_code = 3;
}

message GetUserPreferencesResponse {
    UserPreferences data = 1;
}

message UserPreferences {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string type_code = 4;
    int32 channel_in_app = 5;
    int32 channel_email = 6;
    int32 channel_sms = 7;
    int32 channel_push = 8;
    int32 digest_enabled = 9;
    string digest_frequency = 10;
    string quiet_hours_start = 11;
    string quiet_hours_end = 12;
}

message UpdateUserPreferencesRequest {
    string tenant_id = 1;
    string user_id = 2;
    string type_code = 3;
    int32 channel_in_app = 4;
    int32 channel_email = 5;
    int32 channel_sms = 6;
    int32 channel_push = 7;
    int32 digest_enabled = 8;
    string digest_frequency = 9;
    string quiet_hours_start = 10;
    string quiet_hours_end = 11;
}

message UpdateUserPreferencesResponse {
    UserPreferences data = 1;
}

// Unread Count messages
message GetUnreadCountRequest {
    string tenant_id = 1;
    string user_id = 2;
}

message GetUnreadCountResponse {
    int32 unread_count = 1;
    int32 urgent_count = 2;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | nc_notification_types | -- |
| V002 | nc_templates | -- |
| V003 | nc_user_preferences | -- |
| V004 | nc_notifications | V001 |
| V005 | nc_delivery_log | V004 |

---

## 7. Business Rules

1. **Preference Respect**: Notifications delivered only via channels enabled in user preferences
2. **Quiet Hours**: No push/SMS during quiet hours; queued for delivery after quiet period
3. **Digest Mode**: When enabled, notifications batched into single digest per frequency
4. **Auto-Expiry**: Notifications not acted upon within expiry period auto-archived
5. **Escalation**: URGENT notifications not acknowledged within 4 hours escalated to manager
6. **Rate Limiting**: Max 100 notifications per user per day (configurable); digest above threshold
7. **Grouping**: Notifications with same group_key combined in inbox display
8. **Delivery Guarantee**: At-least-once delivery for all channels; deduplication on client side

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| auth-service | `GetUserPreferences` | Resolve user identity and delivery preferences |
| email-gateway | `SendEmail` | Deliver email notifications |
| push-service | `SendPush` | Deliver mobile push notifications |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| All modules | `SendNotification` | Send notifications to users |
| All modules | `SendBatchNotification` | Batch send to multiple recipients |
| workflow-service | `SendApprovalNotification` | Deliver approval/task notifications |
| digital-assistant-service | `GetNotifications` | Chat-based notification retrieval |
