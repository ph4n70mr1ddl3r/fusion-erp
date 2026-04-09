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

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.responsys.v1;

service ResponsysService {
    rpc GetProgram(GetProgramRequest) returns (GetProgramResponse);
    rpc CreateProgram(CreateProgramRequest) returns (CreateProgramResponse);
    rpc GetMessage(GetMessageRequest) returns (GetMessageResponse);
    rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
    rpc GetCustomerProfile(GetCustomerProfileRequest) returns (GetCustomerProfileResponse);
    rpc RecordInteraction(RecordInteractionRequest) returns (RecordInteractionResponse);
}

// Program messages
message GetProgramRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetProgramResponse {
    RsysProgram data = 1;
}

message CreateProgramRequest {
    string tenant_id = 1;
    string program_code = 2;
    string program_name = 3;
    string description = 4;
    string channel_types = 5;
    string orchestration_rules = 6;
    string entry_criteria = 7;
    string exit_criteria = 8;
    int32 max_duration_days = 9;
    int32 priority = 10;
    string owner_id = 11;
}

message CreateProgramResponse {
    RsysProgram data = 1;
}

message RsysProgram {
    string id = 1;
    string tenant_id = 2;
    string program_code = 3;
    string program_name = 4;
    string description = 5;
    string channel_types = 6;
    string orchestration_rules = 7;
    string entry_criteria = 8;
    string exit_criteria = 9;
    int32 max_duration_days = 10;
    int32 priority = 11;
    string frequency_capping = 12;
    int32 estimated_audience_size = 13;
    string owner_id = 14;
    string status = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Message messages
message GetMessageRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetMessageResponse {
    RsysMessage data = 1;
}

message SendMessageRequest {
    string tenant_id = 1;
    string message_id = 2;
    string segment_id = 3;
    string send_schedule = 4;
}

message SendMessageResponse {
    string message_id = 1;
    string status = 2;
    int32 recipients = 3;
}

message RsysMessage {
    string id = 1;
    string tenant_id = 2;
    string program_id = 3;
    string message_code = 4;
    string message_name = 5;
    string channel = 6;
    string content_template_id = 7;
    string subject_line = 8;
    string preview_text = 9;
    string personalization_rules = 10;
    string send_schedule = 11;
    string send_window_start = 12;
    string send_window_end = 13;
    int32 priority = 14;
    string status = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Customer profile messages
message GetCustomerProfileRequest {
    string tenant_id = 1;
    string customer_id = 2;
}

message GetCustomerProfileResponse {
    RsysCustomerProfile data = 1;
}

message RsysCustomerProfile {
    string id = 1;
    string tenant_id = 2;
    string customer_id = 3;
    string email = 4;
    string phone = 5;
    string first_name = 6;
    string last_name = 7;
    string preferences = 8;
    string channel_opt_ins = 9;
    double engagement_score = 10;
    string lifecycle_stage = 11;
    string last_engagement_at = 12;
    int32 total_messages_received = 13;
    int32 total_opens = 14;
    int32 total_clicks = 15;
    int32 total_conversions = 16;
    int64 total_revenue_cents = 17;
    string preferred_channel = 18;
    string timezone = 19;
    string language = 20;
    string status = 21;
    string created_at = 22;
    string updated_at = 23;
}

// Interaction messages
message RecordInteractionRequest {
    string tenant_id = 1;
    string message_id = 2;
    string customer_id = 3;
    string interaction_type = 4;
    string channel = 5;
    string url_clicked = 6;
    int64 conversion_value_cents = 7;
    string conversion_order_id = 8;
    string interaction_data = 9;
}

message RecordInteractionResponse {
    string id = 1;
    string interaction_type = 2;
    string timestamp = 3;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | rsys_programs | -- |
| V002 | rsys_messages | V001 |
| V003 | rsys_customer_profiles | -- |
| V004 | rsys_message_interactions | V002, V003 |
| V005 | rsys_audience_segments | -- |

---

## 7. Business Rules

1. **Frequency Capping**: Messages respect per-channel and per-customer frequency limits within rolling windows
2. **Channel Preference**: Only send via channels the customer has opted into; respect global unsubscribe
3. **Lifecycle Triggers**: Lifecycle stage transitions automatically trigger relevant program enrollment
4. **Suppression Lists**: Globally suppressed contacts excluded from all sends regardless of program
5. **Send Windows**: Messages delivered only within configured send windows in recipient timezone
6. **Lookalike Limits**: Lookalike segments capped at configured expansion percentage of seed audience
7. **Conversion Attribution**: Conversions attributed to last message within 7-day attribution window

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| cdp-service | `GetProfile` / `GetSegment` | Unified customer profiles and identity resolution |
| order-service | `GetOrder` | Purchase-triggered messages and conversion tracking |
| loyalty-service | `GetMember` | Tier-based personalization and rewards messaging |
| workflow-service | `SubmitApproval` | Program approval workflows |
| notification-service | `SendAlert` | Campaign alerts and delivery reports |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| cx-analytics-service | `GetInteractionData` | Cross-channel marketing performance analytics |
| eloqua-service | `GetCustomerProfile` | B2B/B2C campaign coordination |
| cx-advertising-service | `GetSegment` | Audience activation for paid media |
