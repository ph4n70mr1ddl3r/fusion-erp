# 43 - Digital Assistant / Conversational AI Service Specification

## 1. Domain Overview

The Digital Assistant provides a conversational AI interface and proactive notification system across the entire ERP suite. It enables users to interact with ERP modules through natural language, execute transactions via voice or chat, receive intelligent notifications and approval shortcuts, and configure personalized alert preferences. The assistant supports multiple channels (web, mobile, Slack, Teams, API), offers quick-action shortcuts for common workflows, and surfaces approval tasks for on-the-go decision making. It integrates with every ERP service for query routing and transaction execution, with the AI/ML platform for intent detection, and with the workflow engine for approval orchestration.

**Bounded Context:** Conversational AI, Natural Language Interface & Proactive Notifications
**Service Name:** `assistant-service`
**Database:** `data/assistant.db`
**HTTP Port:** 8070 | **gRPC Port:** 9070

---

## 2. Database Schema

### 2.1 Assistant Skills
```sql
CREATE TABLE assistant_skills (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    skill_name TEXT NOT NULL,
    skill_type TEXT NOT NULL
        CHECK(skill_type IN ('QUERY','TRANSACTION','NAVIGATION','NOTIFICATION')),
    description TEXT,
    intent_patterns TEXT NOT NULL,              -- JSON array of regex/patterns
    required_entities TEXT,                     -- JSON: entity definitions and validation rules
    response_template TEXT NOT NULL,
    confirmation_required INTEGER NOT NULL DEFAULT 0,
    target_service TEXT NOT NULL,
    target_endpoint TEXT NOT NULL,
    http_method TEXT NOT NULL DEFAULT 'GET'
        CHECK(http_method IN ('GET','POST','PUT','PATCH','DELETE')),
    is_active INTEGER NOT NULL DEFAULT 1,
    priority INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, skill_name)
);

CREATE INDEX idx_skills_tenant ON assistant_skills(tenant_id);
CREATE INDEX idx_skills_tenant_type ON assistant_skills(tenant_id, skill_type);
CREATE INDEX idx_skills_tenant_active ON assistant_skills(tenant_id, is_active);
CREATE INDEX idx_skills_tenant_priority ON assistant_skills(tenant_id, priority);
```

### 2.2 Assistant Conversations
```sql
CREATE TABLE assistant_conversations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    channel TEXT NOT NULL
        CHECK(channel IN ('WEB','MOBILE','SLACK','TEAMS','API')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','COMPLETED','ABANDONED')),
    started_at TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at TEXT,
    context_data TEXT,                          -- JSON: accumulated conversation context
    satisfaction_rating INTEGER
        CHECK(satisfaction_rating BETWEEN 1 AND 5),
    feedback_text TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_conversations_tenant ON assistant_conversations(tenant_id);
CREATE INDEX idx_conversations_tenant_user ON assistant_conversations(tenant_id, user_id);
CREATE INDEX idx_conversations_tenant_status ON assistant_conversations(tenant_id, status);
CREATE INDEX idx_conversations_tenant_channel ON assistant_conversations(tenant_id, channel);
CREATE INDEX idx_conversations_tenant_started ON assistant_conversations(tenant_id, started_at);
```

### 2.3 Assistant Messages
```sql
CREATE TABLE assistant_messages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    conversation_id TEXT NOT NULL,
    message_type TEXT NOT NULL
        CHECK(message_type IN ('USER','BOT','SYSTEM')),
    content TEXT NOT NULL,
    intent_detected TEXT,
    entities_extracted TEXT,                    -- JSON: extracted entity values
    confidence_score REAL,
    suggested_actions TEXT,                     -- JSON array of action suggestions
    response_data TEXT,                         -- JSON: structured response payload
    processing_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (conversation_id) REFERENCES assistant_conversations(id) ON DELETE CASCADE
);

CREATE INDEX idx_messages_tenant ON assistant_messages(tenant_id);
CREATE INDEX idx_messages_tenant_conversation ON assistant_messages(tenant_id, conversation_id);
CREATE INDEX idx_messages_tenant_type ON assistant_messages(tenant_id, message_type);
CREATE INDEX idx_messages_tenant_intent ON assistant_messages(tenant_id, intent_detected);
```

### 2.4 Assistant Notification Rules
```sql
CREATE TABLE assistant_notification_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    trigger_event TEXT NOT NULL,
    trigger_condition TEXT,                     -- JSON filter: conditions for firing
    channel TEXT NOT NULL
        CHECK(channel IN ('PUSH','EMAIL','IN_APP','SLACK')),
    message_template TEXT NOT NULL,
    recipient_type TEXT NOT NULL
        CHECK(recipient_type IN ('USER','ROLE','DEPARTMENT')),
    recipient_id TEXT NOT NULL,
    frequency TEXT NOT NULL DEFAULT 'IMMEDIATE'
        CHECK(frequency IN ('IMMEDIATE','DIGEST_DAILY','DIGEST_WEEKLY')),
    is_active INTEGER NOT NULL DEFAULT 1,
    last_triggered_at TEXT,
    trigger_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_notif_rules_tenant ON assistant_notification_rules(tenant_id);
CREATE INDEX idx_notif_rules_tenant_active ON assistant_notification_rules(tenant_id, is_active);
CREATE INDEX idx_notif_rules_tenant_event ON assistant_notification_rules(tenant_id, trigger_event);
CREATE INDEX idx_notif_rules_tenant_recipient ON assistant_notification_rules(tenant_id, recipient_type, recipient_id);
```

### 2.5 Assistant Notifications
```sql
CREATE TABLE assistant_notifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_id TEXT,
    user_id TEXT NOT NULL,
    notification_type TEXT NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    channel TEXT NOT NULL
        CHECK(channel IN ('PUSH','EMAIL','IN_APP','SLACK')),
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','SENT','DELIVERED','READ','DISMISSED')),
    sent_at TEXT,
    delivered_at TEXT,
    read_at TEXT,
    action_taken TEXT,
    related_entity_type TEXT,
    related_entity_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (rule_id) REFERENCES assistant_notification_rules(id) ON DELETE SET NULL
);

CREATE INDEX idx_notifications_tenant ON assistant_notifications(tenant_id);
CREATE INDEX idx_notifications_tenant_user ON assistant_notifications(tenant_id, user_id);
CREATE INDEX idx_notifications_tenant_status ON assistant_notifications(tenant_id, status);
CREATE INDEX idx_notifications_tenant_priority ON assistant_notifications(tenant_id, priority);
CREATE INDEX idx_notifications_tenant_type ON assistant_notifications(tenant_id, notification_type);
CREATE INDEX idx_notifications_tenant_rule ON assistant_notifications(tenant_id, rule_id);
CREATE INDEX idx_notifications_tenant_entity ON assistant_notifications(tenant_id, related_entity_type, related_entity_id);
```

### 2.6 Assistant Quick Actions
```sql
CREATE TABLE assistant_quick_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    action_name TEXT NOT NULL,
    action_type TEXT NOT NULL
        CHECK(action_type IN ('APPROVE','REJECT','SUBMIT','VIEW','NAVIGATE')),
    description TEXT,
    icon TEXT,
    target_url TEXT NOT NULL,
    target_service TEXT NOT NULL,
    required_permission TEXT NOT NULL,
    display_condition TEXT,                     -- JSON: conditions for showing action
    sequence INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    usage_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_quick_actions_tenant ON assistant_quick_actions(tenant_id);
CREATE INDEX idx_quick_actions_tenant_type ON assistant_quick_actions(tenant_id, action_type);
CREATE INDEX idx_quick_actions_tenant_active ON assistant_quick_actions(tenant_id, is_active);
CREATE INDEX idx_quick_actions_tenant_sequence ON assistant_quick_actions(tenant_id, sequence);
```

### 2.7 Assistant Approval Shortcuts
```sql
CREATE TABLE assistant_approval_shortcuts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    approval_type TEXT NOT NULL
        CHECK(approval_type IN ('AP_INVOICE','PO','EXPENSE','LEAVE','JOURNAL','TIMECARD')),
    approval_id TEXT NOT NULL,
    source_service TEXT NOT NULL,
    summary_data TEXT,                          -- JSON: summary for approval display
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','DELEGATED')),
    delegated_to TEXT,
    action_via TEXT NOT NULL DEFAULT 'WEB'
        CHECK(action_via IN ('WEB','MOBILE','ASSISTANT','EMAIL')),
    actioned_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, approval_type, approval_id)
);

CREATE INDEX idx_approvals_tenant ON assistant_approval_shortcuts(tenant_id);
CREATE INDEX idx_approvals_tenant_user ON assistant_approval_shortcuts(tenant_id, user_id);
CREATE INDEX idx_approvals_tenant_status ON assistant_approval_shortcuts(tenant_id, status);
CREATE INDEX idx_approvals_tenant_type ON assistant_approval_shortcuts(tenant_id, approval_type);
CREATE INDEX idx_approvals_tenant_action_via ON assistant_approval_shortcuts(tenant_id, action_via);
```

### 2.8 Assistant User Preferences
```sql
CREATE TABLE assistant_user_preferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    notification_channel TEXT,                  -- Preferred notification channel
    category TEXT NOT NULL
        CHECK(category IN ('APPROVAL','ALERT','REMINDER','REPORT','WORKFLOW')),
    is_enabled INTEGER NOT NULL DEFAULT 1,
    quiet_hours_start TEXT,                     -- HH:MM format
    quiet_hours_end TEXT,                       -- HH:MM format
    digest_frequency TEXT NOT NULL DEFAULT 'IMMEDIATE'
        CHECK(digest_frequency IN ('NONE','IMMEDIATE','HOURLY','DAILY','WEEKLY')),
    language TEXT NOT NULL DEFAULT 'en',
    timezone TEXT NOT NULL DEFAULT 'UTC',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, user_id, category)
);

CREATE INDEX idx_user_prefs_tenant ON assistant_user_preferences(tenant_id);
CREATE INDEX idx_user_prefs_tenant_user ON assistant_user_preferences(tenant_id, user_id);
CREATE INDEX idx_user_prefs_tenant_category ON assistant_user_preferences(tenant_id, category);
```

---

## 3. REST API Endpoints

```
# Conversations
POST          /api/v1/assistant/conversations              Permission: assistant.conversations.create
GET           /api/v1/assistant/conversations/{id}          Permission: assistant.conversations.read
GET           /api/v1/assistant/conversations/{id}/messages Permission: assistant.conversations.read
POST          /api/v1/assistant/conversations/{id}/message  Permission: assistant.conversations.message
POST          /api/v1/assistant/conversations/{id}/feedback Permission: assistant.conversations.feedback
DELETE        /api/v1/assistant/conversations/{id}          Permission: assistant.conversations.delete

# Skills
GET/POST      /api/v1/assistant/skills                     Permission: assistant.skills.manage
GET/PUT       /api/v1/assistant/skills/{id}                 Permission: assistant.skills.manage
POST          /api/v1/assistant/skills/{id}/test            Permission: assistant.skills.manage

# Natural Language Query
POST          /api/v1/assistant/query                      Permission: assistant.query
GET           /api/v1/assistant/query/suggestions           Permission: assistant.query

# Notifications
GET/POST      /api/v1/assistant/notifications/rules         Permission: assistant.notifications.manage
GET/PUT       /api/v1/assistant/notifications/rules/{id}    Permission: assistant.notifications.manage
GET           /api/v1/assistant/notifications               Permission: assistant.notifications.read
POST          /api/v1/assistant/notifications/{id}/read     Permission: assistant.notifications.read
POST          /api/v1/assistant/notifications/{id}/dismiss  Permission: assistant.notifications.read
POST          /api/v1/assistant/notifications/mark-all-read Permission: assistant.notifications.read
GET           /api/v1/assistant/notifications/unread-count  Permission: assistant.notifications.read

# Quick Actions
GET/POST      /api/v1/assistant/quick-actions              Permission: assistant.quick-actions.read/create
GET/PUT       /api/v1/assistant/quick-actions/{id}          Permission: assistant.quick-actions.read/update
POST          /api/v1/assistant/quick-actions/{id}/execute  Permission: assistant.quick-actions.execute

# Approvals
GET           /api/v1/assistant/approvals/pending           Permission: assistant.approvals.read
POST          /api/v1/assistant/approvals/{id}/approve      Permission: assistant.approvals.action
POST          /api/v1/assistant/approvals/{id}/reject       Permission: assistant.approvals.action
POST          /api/v1/assistant/approvals/{id}/delegate     Permission: assistant.approvals.delegate

# User Preferences
GET           /api/v1/assistant/preferences                Permission: assistant.preferences.read
PUT           /api/v1/assistant/preferences                 Permission: assistant.preferences.update
PUT           /api/v1/assistant/preferences/{category}      Permission: assistant.preferences.update

# Dashboard
GET           /api/v1/assistant/dashboard                  Permission: assistant.dashboard.read
```

---

## 4. Business Rules

### 4.1 Intent Detection and Skill Matching
```
1. Intent detection uses pattern matching and NLP to identify user requests
2. Process each user message through active skill intent_patterns in priority order
3. Confidence score threshold: minimum 0.6 to auto-execute; below 0.6 asks for clarification
4. Skill priority determines order when multiple skills match an intent
5. Entity extraction validates required_entities defined on the matched skill
6. Missing required entities trigger follow-up prompts to the user
7. All intent detection results are stored with confidence scores for analytics
```

### 4.2 Transaction Confirmation
- Transaction-type skills MUST require explicit user confirmation before execution
- Confirmation message includes a summary of the action to be performed
- User must respond with an explicit affirmative (yes/approve/confirm)
- Negative or ambiguous responses cancel the transaction
- System messages log the confirmation exchange for audit trail
- Timeout on confirmation (5 minutes) cancels the pending action

### 4.3 Conversation Lifecycle
- Conversations expire after 30 minutes of inactivity
- Expired conversations are marked ABANDONED by background job
- Active conversations are limited to 100 messages per session
- Context data accumulates across messages within a conversation
- Completed conversations collect optional satisfaction_rating and feedback_text
- Conversation history is retained per tenant retention policy

### 4.4 Notification Rules and Delivery
- Notification rules evaluate in real-time against published events
- Rules match trigger_event and trigger_condition filter against event payload
- Multiple rules can fire for a single event; each generates a notification
- Quiet hours suppress non-urgent notifications
  - Notifications during quiet hours are queued and delivered at quiet_hours_end
  - URGENT priority notifications bypass quiet hours
- Digest notifications aggregate multiple alerts per configured frequency
  - DIGEST_DAILY: sent at configured time with all pending notifications
  - DIGEST_WEEKLY: sent on configured day with weekly summary
- Duplicate suppression: same rule + same entity within 1 hour counts as duplicate

### 4.5 Approval Shortcuts
- Approval shortcuts validate user permissions before allowing action
- User must have the required_permission defined on the source service
- Approval action routes to the source_service via gRPC or REST
- Delegate action requires the delegate user to have equivalent permissions
- All approval actions are audit-logged with conversation context and action_via
- Amount thresholds enforced: high-value approvals may require additional confirmation

### 4.6 Quick Actions
- Quick actions are filtered by user permissions and display conditions
- Actions without matching required_permission are hidden from the user
- display_condition JSON evaluates against current user context
- Usage count tracks popularity for sorting and analytics
- Inactive actions are excluded from display but retained for history

### 4.7 Multi-Channel Support
- Channel-specific formatting applied to responses (e.g., Slack markdown vs web HTML)
- Mobile channel supports push notification integration
- API channel returns structured JSON without formatting
- Cross-channel conversation continuity is NOT supported; each channel is independent
- Channel authentication uses tenant API keys for programmatic access

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `assistant.notification.sent` | Notification delivered to user | Reporting, Audit |
| `assistant.approval.actioned` | Approval shortcut executed | AP, Proc, Expense, WF |
| `assistant.conversation.completed` | Conversation ended | AI Service, Reporting |
| `assistant.skill.executed` | Skill transaction completed | Audit, Target Service |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.assistant.v1;

service AssistantService {
    rpc ProcessMessage(ProcessMessageRequest) returns (ProcessMessageResponse);
    rpc GetPendingApprovals(GetPendingApprovalsRequest) returns (GetPendingApprovalsResponse);
    rpc SendNotification(SendNotificationRequest) returns (SendNotificationResponse);
    rpc GetNotificationPreferences(GetNotificationPreferencesRequest) returns (GetNotificationPreferencesResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Auth:** User identity, permissions, and role assignments for access control
- **GL:** General ledger data for natural language financial queries
- **AP:** Invoice data for approval shortcuts and query responses
- **AR:** Receivable data for customer balance queries
- **Procurement:** Purchase order data for approval shortcuts and status queries
- **Expense:** Expense report data for approval shortcuts
- **Inventory:** Inventory levels and item data for stock queries
- **OM:** Order status and fulfillment data for order queries
- **PM:** Project data for project status and timesheet queries
- **AI Service:** NLP intent detection, entity extraction, and query generation
- **Workflow:** Approval workflow definitions and current approval states
- **Reporting:** Report definitions and cached report data for query responses
- **EAM:** Asset status and work order data for maintenance queries

### 6.2 Data Published To
- **AP:** Invoice approval/rejection actions from approval shortcuts
- **Procurement:** Purchase order approval/rejection actions from approval shortcuts
- **Expense:** Expense report approval/rejection actions from approval shortcuts
- **GL:** Journal approval actions from approval shortcuts
- **Workflow:** Approval delegation and action events
- **AI Service:** Conversation transcripts for model improvement
- **Reporting:** Notification metrics, approval analytics, usage statistics

### 6.3 Events Consumed
| Event | Action |
|-------|--------|
| `gl.journal.posted` | Trigger notification rules for journal events |
| `ap.invoice.created` | Create approval shortcut for authorized approvers |
| `ap.invoice.approved` | Remove pending approval shortcut, notify requester |
| `ap.payment.completed` | Trigger notification to vendor contact |
| `proc.po.created` | Create approval shortcut for PO approvers |
| `proc.po.approved` | Remove pending approval shortcut, notify requester |
| `expense.report.submitted` | Create approval shortcut for expense approvers |
| `expense.report.approved` | Remove pending approval shortcut, notify submitter |
| `wf.approval.required` | Create approval shortcut for assigned approver |
| `wf.approval.completed` | Update approval shortcut status |
| `iot.alert.triggered` | Trigger notification rule for IoT alert events |

### 6.4 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `assistant.notification.sent` | Notification delivered to user | Reporting, Audit |
| `assistant.approval.actioned` | Approval shortcut executed | AP, Proc, Expense, WF |
| `assistant.conversation.completed` | Conversation ended with feedback | AI Service, Reporting |
| `assistant.skill.executed` | Skill transaction completed | Audit, Target Service |

---

## 7. Migrations

1. V001: `assistant_skills`
2. V002: `assistant_conversations`
3. V003: `assistant_messages`
4. V004: `assistant_notification_rules`
5. V005: `assistant_notifications`
6. V006: `assistant_quick_actions`
7. V007: `assistant_approval_shortcuts`
8. V008: `assistant_user_preferences`
9. V009: Indexes for all tables
10. V010: Triggers for `updated_at`
