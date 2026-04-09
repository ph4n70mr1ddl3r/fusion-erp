# 126 - B2C Service Specification

## 1. Domain Overview

B2C Service provides high-volume consumer-facing customer support capabilities optimized for retail, e-commerce, and consumer goods industries. It handles millions of service requests with knowledge-guided resolution, intelligent routing, self-service portals, and omnichannel engagement across chat, email, phone, social media, and messaging platforms.

**Bounded Context:** B2C Customer Service & Consumer Support
**Service Name:** `b2cservice-service`
**Database:** `data/b2cservice.db`
**HTTP Port:** 8206 | **gRPC Port:** 9206

---

## 2. Database Schema

### 2.1 Service Requests
```sql
CREATE TABLE b2c_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_number TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    channel TEXT NOT NULL CHECK(channel IN ('WEB','CHAT','EMAIL','PHONE','SOCIAL','SMS','WHATSAPP','MESSENGER','IN_APP')),
    request_type TEXT NOT NULL CHECK(request_type IN ('INQUIRY','COMPLAINT','RETURN','EXCHANGE','WARRANTY','FEEDBACK','ORDER_ISSUE')),
    subject TEXT NOT NULL,
    description TEXT NOT NULL,
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','OPEN','IN_PROGRESS','WAITING_CUSTOMER','WAITING_INTERNAL','RESOLVED','CLOSED','CANCELLED')),
    category_id TEXT,
    subcategory_id TEXT,
    product_id TEXT,
    order_id TEXT,
    order_line_id TEXT,
    assigned_agent_id TEXT,
    assigned_team TEXT,
    sla_policy_id TEXT,
    sla_due_date TEXT,
    first_response_date TEXT,
    resolution_date TEXT,
    close_date TEXT,
    customer_satisfaction_score INTEGER CHECK(customer_satisfaction_score BETWEEN 1 AND 5),
    resolution_method TEXT CHECK(resolution_method IN ('SELF_SERVICE','AGENT','KNOWLEDGE_BASE','AUTOMATED','ESCALATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, request_number)
);

CREATE INDEX idx_b2c_requests_tenant_status ON b2c_requests(tenant_id, status);
CREATE INDEX idx_b2c_requests_tenant_customer ON b2c_requests(tenant_id, customer_id);
CREATE INDEX idx_b2c_requests_tenant_agent ON b2c_requests(tenant_id, assigned_agent_id, status);
CREATE INDEX idx_b2c_requests_tenant_sla ON b2c_requests(tenant_id, sla_due_date);
```

### 2.2 Request Threads (Conversation History)
```sql
CREATE TABLE b2c_request_threads (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_id TEXT NOT NULL,
    message_type TEXT NOT NULL CHECK(message_type IN ('CUSTOMER','AGENT','SYSTEM','BOT','NOTE')),
    sender_id TEXT,
    sender_name TEXT,
    content TEXT NOT NULL,
    content_type TEXT NOT NULL DEFAULT 'TEXT'
        CHECK(content_type IN ('TEXT','HTML','RICH_TEXT','ATTACHMENT')),
    attachments TEXT,                    -- JSON array of attachment IDs
    is_internal_note INTEGER NOT NULL DEFAULT 0,
    channel TEXT,
    metadata TEXT,                       -- JSON: platform-specific data

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (request_id) REFERENCES b2c_requests(id) ON DELETE CASCADE
);

CREATE INDEX idx_b2c_threads_tenant_request ON b2c_request_threads(tenant_id, request_id, created_at);
```

### 2.3 Knowledge Guides
```sql
CREATE TABLE b2c_knowledge_guides (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    guide_title TEXT NOT NULL,
    guide_type TEXT NOT NULL CHECK(guide_type IN ('DECISION_TREE','FAQ','TROUBLESHOOTING','HOW_TO','POLICY')),
    category_id TEXT,
    trigger_keywords TEXT,               -- Comma-separated keywords for auto-suggestion
    content TEXT NOT NULL,               -- JSON: structured guide content / decision tree
    applicable_channels TEXT,            -- JSON array: ["WEB","CHAT","EMAIL"]
    applicable_request_types TEXT,       -- JSON array: ["RETURN","COMPLAINT"]
    is_customer_visible INTEGER NOT NULL DEFAULT 1,
    is_agent_visible INTEGER NOT NULL DEFAULT 1,
    view_count INTEGER NOT NULL DEFAULT 0,
    helpful_count INTEGER NOT NULL DEFAULT 0,
    not_helpful_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, guide_title)
);

CREATE INDEX idx_b2c_guides_tenant_category ON b2c_knowledge_guides(tenant_id, category_id, status);
```

### 2.4 Self-Service Portal Config
```sql
CREATE TABLE b2c_portal_config (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    config_key TEXT NOT NULL,
    config_value TEXT NOT NULL,
    config_type TEXT NOT NULL DEFAULT 'STRING' CHECK(config_type IN ('STRING','INTEGER','BOOLEAN','JSON')),
    description TEXT,
    category TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, config_key)
);
```

### 2.5 Customer Interaction Log
```sql
CREATE TABLE b2c_interactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    interaction_type TEXT NOT NULL CHECK(interaction_type IN ('PAGE_VIEW','SEARCH','CHAT','CALL','EMAIL','PURCHASE','RETURN','FEEDBACK','SOCIAL_POST')),
    channel TEXT NOT NULL,
    request_id TEXT,
    session_id TEXT,
    content_summary TEXT,
    sentiment_score REAL CHECK(sentiment_score BETWEEN -1.0 AND 1.0),
    intent TEXT,
    products_viewed TEXT,                -- JSON array of product IDs
    interaction_timestamp TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (request_id) REFERENCES b2c_requests(id) ON DELETE SET NULL
);

CREATE INDEX idx_b2c_interactions_tenant_customer ON b2c_interactions(tenant_id, customer_id, interaction_timestamp DESC);
CREATE INDEX idx_b2c_interactions_tenant_session ON b2c_interactions(tenant_id, session_id);
```

### 2.6 Chat Sessions
```sql
CREATE TABLE b2c_chat_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    customer_id TEXT,
    customer_name TEXT,
    customer_email TEXT,
    channel TEXT NOT NULL DEFAULT 'WEB',
    agent_id TEXT,
    bot_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','QUEUED','TRANSFERRED','ENDED')),
    queue_name TEXT,
    wait_time_seconds INTEGER,
    handle_time_seconds INTEGER,
    resolution_status TEXT CHECK(resolution_status IN ('RESOLVED','UNRESOLVED','ESCALATED')),
    satisfaction_score INTEGER,
    tags TEXT,                           -- JSON array

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, session_id)
);
```

---

## 3. REST API Endpoints

### 3.1 Service Requests
```
GET    /api/v1/b2cservice/requests                       Permission: b2c.requests.read
GET    /api/v1/b2cservice/requests/{id}                  Permission: b2c.requests.read
POST   /api/v1/b2cservice/requests                       Permission: b2c.requests.create
PUT    /api/v1/b2cservice/requests/{id}                  Permission: b2c.requests.update
POST   /api/v1/b2cservice/requests/{id}/assign           Permission: b2c.requests.assign
POST   /api/v1/b2cservice/requests/{id}/resolve          Permission: b2c.requests.resolve
POST   /api/v1/b2cservice/requests/{id}/close            Permission: b2c.requests.close
POST   /api/v1/b2cservice/requests/{id}/escalate         Permission: b2c.requests.escalate
GET    /api/v1/b2cservice/requests/{id}/thread           Permission: b2c.requests.read
POST   /api/v1/b2cservice/requests/{id}/messages         Permission: b2c.requests.respond
```

### 3.2 Knowledge Guides
```
GET    /api/v1/b2cservice/guides                         Permission: b2c.guides.read
GET    /api/v1/b2cservice/guides/{id}                    Permission: b2c.guides.read
POST   /api/v1/b2cservice/guides                         Permission: b2c.guides.create
PUT    /api/v1/b2cservice/guides/{id}                    Permission: b2c.guides.update
GET    /api/v1/b2cservice/guides/suggest
  ?keywords=return+damaged&request_type=RETURN            Permission: b2c.guides.read
POST   /api/v1/b2cservice/guides/{id}/feedback           Permission: b2c.guides.feedback
```

### 3.3 Chat
```
POST   /api/v1/b2cservice/chat/sessions                  Permission: b2c.chat.create
GET    /api/v1/b2cservice/chat/sessions/{id}             Permission: b2c.chat.read
POST   /api/v1/b2cservice/chat/sessions/{id}/messages    Permission: b2c.chat.message
POST   /api/v1/b2cservice/chat/sessions/{id}/transfer    Permission: b2c.chat.transfer
POST   /api/v1/b2cservice/chat/sessions/{id}/end         Permission: b2c.chat.end
GET    /api/v1/b2cservice/chat/queue                      Permission: b2c.chat.queue
```

### 3.4 Customer Portal (Public)
```
POST   /api/v1/b2cservice/portal/requests                Permission: b2c.portal.create
GET    /api/v1/b2cservice/portal/requests
  ?customer_id={id}                                       Permission: b2c.portal.read
GET    /api/v1/b2cservice/portal/guides
  ?search={keyword}                                       Permission: b2c.portal.read
GET    /api/v1/b2cservice/portal/requests/{id}           Permission: b2c.portal.read
POST   /api/v1/b2cservice/portal/requests/{id}/messages  Permission: b2c.portal.respond
```

### 3.5 Analytics
```
GET    /api/v1/b2cservice/analytics/volume
  ?period=day|week|month&from=...&to=...                  Permission: b2c.analytics.read
GET    /api/v1/b2cservice/analytics/sla                   Permission: b2c.analytics.read
GET    /api/v1/b2cservice/analytics/satisfaction          Permission: b2c.analytics.read
GET    /api/v1/b2cservice/analytics/agent-performance     Permission: b2c.analytics.read
GET    /api/v1/b2cservice/analytics/channel-distribution  Permission: b2c.analytics.read
```

---

## 4. Business Rules

### 4.1 Request Routing
- New requests auto-classified by type based on keywords and channel
- Intelligent routing assigns to agent with matching skill set and lowest queue
- VIP customers identified by loyalty tier and routed to priority agents
- Overflow routing when queue exceeds configurable threshold

### 4.2 SLA Management
- SLA policies define response and resolution time targets by priority and type
- SLA timers pause during customer wait states
- Escalation triggered when SLA due date approaches (configurable warning threshold)
- SLA breach events published for reporting and alerts

### 4.3 Knowledge-Guided Resolution
- Guides auto-suggested based on request type, keywords, and product
- Decision trees walk agents through structured troubleshooting
- Agent can push guide content directly to customer in chat
- Guide effectiveness tracked (helpful/not helpful ratio)

### 4.4 Omnichannel Rules
- Customer conversations unified across channels (web → chat → email)
- Channel switching preserves full conversation context
- Social media mentions auto-create requests based on sentiment
- WhatsApp/SMS messages threaded as request messages

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.b2cservice.v1;

service B2CService {
    rpc CreateRequest(CreateRequestRequest) returns (CreateRequestResponse);
    rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
    rpc GetSuggestedGuides(GetSuggestedGuidesRequest) returns (GetSuggestedGuidesResponse);
    rpc ClassifyRequest(ClassifyRequestRequest) returns (ClassifyRequestResponse);
    rpc CheckSLA(CheckSLARequest) returns (CheckSLAResponse);
}

// Entity messages
message B2cRequest {
    string id = 1;
    string tenant_id = 2;
    string request_number = 3;
    string customer_id = 4;
    string channel = 5;
    string request_type = 6;
    string subject = 7;
    string description = 8;
    string priority = 9;
    string status = 10;
    string category_id = 11;
    string subcategory_id = 12;
    string product_id = 13;
    string order_id = 14;
    string order_line_id = 15;
    string assigned_agent_id = 16;
    string assigned_team = 17;
    string sla_policy_id = 18;
    string sla_due_date = 19;
    string first_response_date = 20;
    string resolution_date = 21;
    string close_date = 22;
    int32 customer_satisfaction_score = 23;
    string resolution_method = 24;
    string created_at = 25;
    string updated_at = 26;
}

message B2cRequestThread {
    string id = 1;
    string tenant_id = 2;
    string request_id = 3;
    string message_type = 4;
    string sender_id = 5;
    string sender_name = 6;
    string content = 7;
    string content_type = 8;
    string attachments = 9;
    bool is_internal_note = 10;
    string channel = 11;
    string metadata = 12;
    string created_at = 13;
    string updated_at = 14;
}

message B2cKnowledgeGuide {
    string id = 1;
    string tenant_id = 2;
    string guide_title = 3;
    string guide_type = 4;
    string category_id = 5;
    string trigger_keywords = 6;
    string content = 7;
    string applicable_channels = 8;
    string applicable_request_types = 9;
    bool is_customer_visible = 10;
    bool is_agent_visible = 11;
    int32 view_count = 12;
    int32 helpful_count = 13;
    int32 not_helpful_count = 14;
    string status = 15;
    string created_at = 16;
    string updated_at = 17;
}

message B2cChatSession {
    string id = 1;
    string tenant_id = 2;
    string session_id = 3;
    string customer_id = 4;
    string customer_name = 5;
    string customer_email = 6;
    string channel = 7;
    string agent_id = 8;
    string bot_id = 9;
    string status = 10;
    string queue_name = 11;
    int32 wait_time_seconds = 12;
    int32 handle_time_seconds = 13;
    string resolution_status = 14;
    int32 satisfaction_score = 15;
    string tags = 16;
    string created_at = 17;
    string updated_at = 18;
}

// Request/Response messages
message CreateRequestRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string channel = 3;
    string request_type = 4;
    string subject = 5;
    string description = 6;
    string priority = 7;
    string category_id = 8;
    string product_id = 9;
    string order_id = 10;
    string order_line_id = 11;
}

message CreateRequestResponse {
    B2cRequest data = 1;
}

message SendMessageRequest {
    string tenant_id = 1;
    string request_id = 2;
    string message_type = 3;
    string content = 4;
    string content_type = 5;
    string attachments = 6;
    bool is_internal_note = 7;
}

message SendMessageResponse {
    B2cRequestThread data = 1;
}

message GetSuggestedGuidesRequest {
    string tenant_id = 1;
    string request_id = 2;
    string request_type = 3;
    string keywords = 4;
    int32 limit = 5;
}

message GetSuggestedGuidesResponse {
    repeated B2cKnowledgeGuide guides = 1;
    repeated double confidence_scores = 2;
}

message ClassifyRequestRequest {
    string tenant_id = 1;
    string request_id = 2;
    string subject = 3;
    string description = 4;
    string channel = 5;
}

message ClassificationResult {
    string request_type = 1;
    string category_id = 2;
    string subcategory_id = 3;
    string priority = 4;
    double confidence = 5;
}

message ClassifyRequestResponse {
    repeated ClassificationResult classifications = 1;
}

message CheckSLARequest {
    string tenant_id = 1;
    string request_id = 2;
    string priority = 3;
    string request_type = 4;
}

message SLAStatus {
    string sla_policy_id = 1;
    string response_due_date = 2;
    string resolution_due_date = 3;
    bool is_response_breached = 4;
    bool is_resolution_breached = 5;
    int32 response_remaining_seconds = 6;
    int32 resolution_remaining_seconds = 7;
}

message CheckSLAResponse {
    SLAStatus sla_status = 1;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **customerservice-service**: Core case management integration
- **knowledge-service**: Knowledge base articles and guides
- **commerce-service**: Order details for order-related requests
- **cdp-service**: Customer profiles, interaction history
- **ai-service**: Sentiment analysis, request classification, chatbot
- **dms-service**: Attachments and document management
- **workflow-service**: Escalation and approval workflows

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `b2c.request.created` | New service request | request_id, customer_id, channel, type |
| `b2c.request.resolved` | Request resolved | request_id, resolution_method, satisfaction |
| `b2c.request.sla.breach` | SLA deadline breached | request_id, sla_type, breach_minutes |
| `b2c.chat.session.started` | Chat session initiated | session_id, channel |
| `b2c.chat.session.ended` | Chat session concluded | session_id, handle_time_seconds |
| `b2c.guide.suggested` | Guide auto-suggested | guide_id, request_id, confidence |
| `b2c.interaction.logged` | Customer interaction recorded | customer_id, type, channel |

---

## 7. Migrations

### Migration Order for b2cservice-service:
1. V001: `b2c_portal_config`
2. V002: `b2c_knowledge_guides`
3. V003: `b2c_requests`
4. V004: `b2c_request_threads`
5. V005: `b2c_interactions`
6. V006: `b2c_chat_sessions`
7. V007: Triggers for `updated_at`
8. V008: Seed data (default portal config, SLA policies, sample guides)
