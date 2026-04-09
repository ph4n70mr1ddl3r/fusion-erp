# 182 - Responsys Service Specification

## 1. Domain Overview

Responsys provides B2C cross-channel marketing orchestration, enabling personalized customer engagement across email, SMS, push notifications, web, display, and social channels. Supports program-based campaign orchestration with branching logic, dynamic message personalization, customer lifecycle management, and real-time interaction tracking. Enables marketers to build sophisticated cross-channel customer journeys, manage opt-in/opt-out preferences per channel, and leverage audience segmentation with dynamic, static, and lookalike capabilities. Integrates with Order Management for purchase-triggered messages, Customer Data Platform for unified profiles, and Loyalty for tier-based personalization.

**Bounded Context:** B2C Cross-Channel Marketing & Customer Engagement
**Service Name:** `responsys-service`
**Database:** `data/responsys.db`
**HTTP Port:** 8200 | **gRPC Port:** 9200

---

## 2. Database Schema

### 2.1 Programs
```sql
CREATE TABLE rsys_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_name TEXT NOT NULL,
    description TEXT,
    channel_types TEXT NOT NULL,                 -- JSON: array of enabled channels
    orchestration_rules TEXT NOT NULL,           -- JSON: journey steps, branches, conditions
    entry_criteria TEXT,                         -- JSON: criteria to enter program
    exit_criteria TEXT,                          -- JSON: criteria to exit program
    max_duration_days INTEGER NOT NULL DEFAULT 30,
    priority INTEGER NOT NULL DEFAULT 5,
    frequency_capping TEXT,                      -- JSON: per-channel send limits
    estimated_audience_size INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SCHEDULED','ACTIVE','PAUSED','COMPLETED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_code)
);

CREATE INDEX idx_rsys_prog_tenant ON rsys_programs(tenant_id, status);
CREATE INDEX idx_rsys_prog_owner ON rsys_programs(owner_id);
CREATE INDEX idx_rsys_prog_dates ON rsys_programs(created_at DESC);
```

### 2.2 Messages
```sql
CREATE TABLE rsys_messages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT,
    message_code TEXT NOT NULL,
    message_name TEXT NOT NULL,
    channel TEXT NOT NULL CHECK(channel IN ('EMAIL','SMS','PUSH','WEB','DISPLAY','SOCIAL')),
    content_template_id TEXT NOT NULL,
    subject_line TEXT,
    preview_text TEXT,
    html_body TEXT,
    text_body TEXT,
    push_title TEXT,
    push_body TEXT,
    sms_text TEXT,
    personalization_rules TEXT NOT NULL,         -- JSON: dynamic content and merge fields
    send_schedule TEXT,                          -- JSON: send timing configuration
    send_window_start TEXT,                      -- Time window for sending
    send_window_end TEXT,
    timezone_aware INTEGER NOT NULL DEFAULT 1,
    suppress_if_recent_send INTEGER NOT NULL DEFAULT 0,
    suppress_window_hours INTEGER NOT NULL DEFAULT 24,
    priority INTEGER NOT NULL DEFAULT 5,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','APPROVED','SCHEDULED','SENT','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES rsys_programs(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, message_code)
);

CREATE INDEX idx_rsys_msg_tenant ON rsys_messages(tenant_id, channel, status);
CREATE INDEX idx_rsys_msg_program ON rsys_messages(program_id);
CREATE INDEX idx_rsys_msg_schedule ON rsys_messages(send_schedule);
```

### 2.3 Customer Profiles
```sql
CREATE TABLE rsys_customer_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    first_name TEXT,
    last_name TEXT,
    preferences TEXT,                            -- JSON: communication preferences
    channel_opt_ins TEXT NOT NULL,               -- JSON: opt-in status per channel
    engagement_score REAL NOT NULL DEFAULT 0,
    lifecycle_stage TEXT NOT NULL DEFAULT 'PROSPECT'
        CHECK(lifecycle_stage IN ('PROSPECT','NEW','ACTIVE','LOYAL','AT_RISK','CHURNED','REACTIVATED')),
    last_engagement_at TEXT,
    total_messages_received INTEGER NOT NULL DEFAULT 0,
    total_opens INTEGER NOT NULL DEFAULT 0,
    total_clicks INTEGER NOT NULL DEFAULT 0,
    total_conversions INTEGER NOT NULL DEFAULT 0,
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    preferred_channel TEXT,
    timezone TEXT,
    language TEXT DEFAULT 'en',
    tags TEXT,                                   -- JSON: array of tags
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','UNSUBSCRIBED','SUPPRESSED','INVALID')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, customer_id)
);

CREATE INDEX idx_rsys_cust_tenant ON rsys_customer_profiles(tenant_id, status);
CREATE INDEX idx_rsys_cust_lifecycle ON rsys_customer_profiles(lifecycle_stage);
CREATE INDEX idx_rsys_cust_engagement ON rsys_customer_profiles(engagement_score DESC);
CREATE INDEX idx_rsys_cust_email ON rsys_customer_profiles(email);
```

### 2.4 Message Interactions
```sql
CREATE TABLE rsys_message_interactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    message_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    interaction_type TEXT NOT NULL CHECK(interaction_type IN (
        'SENT','DELIVERED','OPENED','CLICKED','CONVERTED','UNSUBSCRIBED','BOUNCED','COMPLAINED'
    )),
    channel TEXT NOT NULL,
    url_clicked TEXT,
    conversion_value_cents INTEGER,
    conversion_order_id TEXT,
    device_type TEXT,
    os_type TEXT,
    browser TEXT,
    ip_address TEXT,
    geolocation TEXT,                            -- JSON: lat/long/city/country
    interaction_data TEXT,                       -- JSON: event-specific details
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (message_id) REFERENCES rsys_messages(id) ON DELETE CASCADE
);

CREATE INDEX idx_rsys_int_message ON rsys_message_interactions(message_id, interaction_type);
CREATE INDEX idx_rsys_int_customer ON rsys_message_interactions(customer_id, timestamp DESC);
CREATE INDEX idx_rsys_int_type ON rsys_message_interactions(tenant_id, interaction_type);
CREATE INDEX idx_rsys_int_time ON rsys_message_interactions(timestamp DESC);
```

### 2.5 Audience Segments
```sql
CREATE TABLE rsys_audience_segments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_code TEXT NOT NULL,
    segment_name TEXT NOT NULL,
    description TEXT,
    segment_type TEXT NOT NULL CHECK(segment_type IN ('DYNAMIC','STATIC','LOOKALIKE')),
    criteria TEXT NOT NULL,                      -- JSON: segment definition rules
    size INTEGER NOT NULL DEFAULT 0,
    data_source TEXT,                            -- Source system for segment data
    lookalike_seed_segment_id TEXT,              -- For lookalike segments
    lookalike_expansion_pct REAL,                -- Lookalike audience expansion percentage
    refresh_schedule TEXT,                       -- JSON: cron schedule for dynamic refresh
    last_refreshed_at TEXT,
    last_size_change INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, segment_code)
);

CREATE INDEX idx_rsys_seg_tenant ON rsys_audience_segments(tenant_id, status);
CREATE INDEX idx_rsys_seg_type ON rsys_audience_segments(tenant_id, segment_type);
CREATE INDEX idx_rsys_seg_owner ON rsys_audience_segments(owner_id);
```

---

## 3. API Endpoints

### 3.1 Programs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/responsys/programs` | Create program |
| GET | `/api/v1/responsys/programs` | List programs |
| GET | `/api/v1/responsys/programs/{id}` | Get program details |
| PUT | `/api/v1/responsys/programs/{id}` | Update program |
| POST | `/api/v1/responsys/programs/{id}/activate` | Activate program |
| POST | `/api/v1/responsys/programs/{id}/pause` | Pause program |

### 3.2 Messages
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/responsys/messages` | Create message |
| GET | `/api/v1/responsys/messages` | List messages |
| GET | `/api/v1/responsys/messages/{id}` | Get message details |
| PUT | `/api/v1/responsys/messages/{id}` | Update message |
| POST | `/api/v1/responsys/messages/{id}/send` | Send message |
| POST | `/api/v1/responsys/messages/{id}/schedule` | Schedule message |

### 3.3 Customer Profiles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/responsys/customers` | Create customer profile |
| GET | `/api/v1/responsys/customers` | List customer profiles |
| GET | `/api/v1/responsys/customers/{id}` | Get customer profile |
| PUT | `/api/v1/responsys/customers/{id}` | Update profile |
| PUT | `/api/v1/responsys/customers/{id}/preferences` | Update channel preferences |
| GET | `/api/v1/responsys/customers/{id}/history` | Get interaction history |

### 3.4 Interactions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/responsys/interactions` | Record interaction |
| GET | `/api/v1/responsys/interactions` | List interactions |
| GET | `/api/v1/responsys/messages/{id}/interactions` | Message interaction summary |
| GET | `/api/v1/responsys/customers/{id}/interactions` | Customer interaction timeline |

### 3.5 Segmentation
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/responsys/segments` | Create segment |
| GET | `/api/v1/responsys/segments` | List segments |
| GET | `/api/v1/responsys/segments/{id}` | Get segment details |
| PUT | `/api/v1/responsys/segments/{id}` | Update segment |
| POST | `/api/v1/responsys/segments/{id}/refresh` | Refresh segment membership |
| POST | `/api/v1/responsys/segments/{id}/lookalike` | Generate lookalike segment |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `responsys.message.delivered` | `{ message_id, customer_id, channel }` | Message delivered to customer |
| `responsys.customer.converted` | `{ customer_id, message_id, order_id, revenue }` | Customer converted from message |
| `responsys.segment.updated` | `{ segment_id, segment_type, size, size_change }` | Segment membership refreshed |
| `responsys.program.completed` | `{ program_id, customers_completed, conversion_rate }` | Program finished execution |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `order.placed` | Order Management | Trigger post-purchase messages and update profile |
| `customer.registered` | Customer Data Platform | Create Responsys customer profile |
| `loyalty.tier.changed` | Loyalty Management | Update lifecycle stage and trigger tier messaging |

---

## 5. Business Rules

1. **Frequency Capping**: Messages respect per-channel and per-customer frequency limits within rolling windows
2. **Channel Preference**: Only send via channels the customer has opted into; respect global unsubscribe
3. **Lifecycle Triggers**: Lifecycle stage transitions automatically trigger relevant program enrollment
4. **Suppression Lists**: Globally suppressed contacts excluded from all sends regardless of program
5. **Send Windows**: Messages delivered only within configured send windows in recipient timezone
6. **Lookalike Limits**: Lookalike segments capped at configured expansion percentage of seed audience
7. **Conversion Attribution**: Conversions attributed to last message within 7-day attribution window

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Customer Data Platform (60) | Unified customer profiles and identity resolution |
| Order Management (39) | Purchase-triggered messages and conversion tracking |
| Loyalty Management (97) | Tier-based personalization and rewards messaging |
| CX Analytics (131) | Cross-channel marketing performance analytics |
| Eloqua (181) | B2B/B2C campaign coordination |
| CX Advertising (183) | Audience activation for paid media |
| Workflow (16) | Program approval workflows |
| Notification Center (165) | Campaign alerts and delivery reports |
