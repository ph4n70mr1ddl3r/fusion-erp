# 112 - Digital Customer Service Service Specification

## 1. Domain Overview

The Digital Customer Service service provides AI-powered digital-first customer support through chatbot-driven resolution, conversational IVR, self-service portals, intelligent escalation, omnichannel digital engagement, and automated case deflection. Distinct from the general Customer Service module, this service focuses on automated and digital-first interactions rather than agent-assisted support, enabling high-volume self-service resolution across web, mobile, SMS, social, and messaging channels.

**Bounded Context:** Digital-First Customer Support & Self-Service
**Service Name:** `digitalcs-service`
**Database:** `data/digitalcs.db`
**HTTP Port:** 8149 | **gRPC Port:** 9149

---

## 2. Database Schema

### 2.1 Digital Channels
```sql
CREATE TABLE digital_channels (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    channel_name TEXT NOT NULL,
    channel_type TEXT NOT NULL CHECK(channel_type IN ('WEB_CHAT','MOBILE_APP','SMS','EMAIL','SOCIAL','WHATSAPP','MESSENGER','VOICE_BOT','ALEXA','WECHAT')),
    configuration TEXT NOT NULL,        -- JSON: channel-specific settings
    bot_personality_config TEXT,        -- JSON: tone, language style, name
    welcome_message TEXT,
    operating_hours TEXT,               -- JSON: schedule definition
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','MAINTENANCE')),
    avg_response_time_ms INTEGER,
    resolution_rate DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, channel_name)
);

CREATE INDEX idx_channels_tenant_type ON digital_channels(tenant_id, channel_type);
CREATE INDEX idx_channels_tenant_status ON digital_channels(tenant_id, status);
```

### 2.2 Chat Sessions
```sql
CREATE TABLE chat_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    channel_id TEXT NOT NULL,
    customer_id TEXT,
    session_token TEXT NOT NULL,
    start_time TEXT NOT NULL,
    end_time TEXT,
    duration_seconds INTEGER,
    message_count INTEGER NOT NULL DEFAULT 0,
    language TEXT NOT NULL DEFAULT 'en',
    initial_intent TEXT,
    resolved_intent TEXT,
    resolution_status TEXT CHECK(resolution_status IN ('SELF_SERVED','ESCALATED','ABANDONED','TRANSFERRED')),
    satisfaction_rating INTEGER CHECK(satisfaction_rating BETWEEN 1 AND 5),
    agent_id TEXT,
    bot_version TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (channel_id) REFERENCES digital_channels(id)
);

CREATE INDEX idx_sessions_tenant_channel ON chat_sessions(tenant_id, channel_id);
CREATE INDEX idx_sessions_tenant_customer ON chat_sessions(tenant_id, customer_id);
CREATE INDEX idx_sessions_tenant_status ON chat_sessions(tenant_id, resolution_status);
CREATE INDEX idx_sessions_tenant_start ON chat_sessions(tenant_id, start_time);
```

### 2.3 Chat Messages
```sql
CREATE TABLE chat_messages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    sender_type TEXT NOT NULL CHECK(sender_type IN ('CUSTOMER','BOT','AGENT','SYSTEM')),
    message_content TEXT NOT NULL,
    message_type TEXT NOT NULL DEFAULT 'TEXT' CHECK(message_type IN ('TEXT','IMAGE','FILE','CARD','CAROUSEL','QUICK_REPLY','LINK')),
    intent_detected TEXT,
    entities_extracted TEXT,            -- JSON: extracted entities
    confidence_score DECIMAL(5,2),
    response_time_ms INTEGER,
    attachments TEXT,                   -- JSON: attachment references

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (session_id) REFERENCES chat_sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_messages_tenant_session ON chat_messages(tenant_id, session_id);
CREATE INDEX idx_messages_tenant_sender ON chat_messages(tenant_id, sender_type);
CREATE INDEX idx_messages_tenant_intent ON chat_messages(tenant_id, intent_detected);
```

### 2.4 Intent Definitions
```sql
CREATE TABLE intent_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    intent_name TEXT NOT NULL,
    training_phrases TEXT NOT NULL,     -- JSON: array of training phrases
    response_templates TEXT NOT NULL,   -- JSON: array of response templates
    escalation_rules TEXT,              -- JSON: escalation trigger conditions
    required_entities TEXT,             -- JSON: required entity types
    fulfillment_type TEXT NOT NULL CHECK(fulfillment_type IN ('TEXT_RESPONSE','API_CALL','WORKFLOW','TRANSFER')),
    parent_intent_id TEXT,
    avg_resolution_time_seconds INTEGER,
    success_rate DECIMAL(5,2),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, intent_name)
);

CREATE INDEX idx_intents_tenant_active ON intent_definitions(tenant_id, is_active);
CREATE INDEX idx_intents_tenant_fulfillment ON intent_definitions(tenant_id, fulfillment_type);
CREATE INDEX idx_intents_tenant_parent ON intent_definitions(tenant_id, parent_intent_id);
```

### 2.5 Knowledge Articles Digital
```sql
CREATE TABLE knowledge_articles_digital (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    article_title TEXT NOT NULL,
    article_content TEXT NOT NULL,
    article_type TEXT NOT NULL CHECK(article_type IN ('FAQ','TROUBLESHOOTING','HOW_TO','POLICY','PRODUCT_INFO')),
    tags TEXT,                          -- JSON: array of tags
    view_count INTEGER NOT NULL DEFAULT 0,
    helpful_count INTEGER NOT NULL DEFAULT 0,
    not_helpful_count INTEGER NOT NULL DEFAULT 0,
    language TEXT NOT NULL DEFAULT 'en',
    intent_mapping TEXT,                -- JSON: linked intent IDs
    is_published INTEGER NOT NULL DEFAULT 0,
    published_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_articles_tenant_type ON knowledge_articles_digital(tenant_id, article_type);
CREATE INDEX idx_articles_tenant_published ON knowledge_articles_digital(tenant_id, is_published);
CREATE INDEX idx_articles_tenant_tags ON knowledge_articles_digital(tenant_id, tags);
CREATE INDEX idx_articles_tenant_language ON knowledge_articles_digital(tenant_id, language);
```

### 2.6 Escalation Rules
```sql
CREATE TABLE escalation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    trigger_condition TEXT NOT NULL,    -- JSON: condition expression
    escalation_type TEXT NOT NULL CHECK(escalation_type IN ('AGENT_TRANSFER','PRIORITY_BOOST','NOTIFICATION','WORKFLOW')),
    target_queue_id TEXT,
    target_agent_id TEXT,
    priority_boost INTEGER,
    sla_minutes INTEGER,
    notification_config TEXT,           -- JSON: notification settings

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_escalation_tenant_active ON escalation_rules(tenant_id, is_active);
CREATE INDEX idx_escalation_tenant_type ON escalation_rules(tenant_id, escalation_type);
```

### 2.7 Self-Service Analytics
```sql
CREATE TABLE self_service_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_date TEXT NOT NULL,
    channel_id TEXT,
    total_sessions INTEGER NOT NULL DEFAULT 0,
    self_served_count INTEGER NOT NULL DEFAULT 0,
    escalated_count INTEGER NOT NULL DEFAULT 0,
    abandoned_count INTEGER NOT NULL DEFAULT 0,
    avg_resolution_time_seconds INTEGER,
    containment_rate DECIMAL(5,2),
    avg_satisfaction DECIMAL(3,2),
    top_intents TEXT,  -- JSON: top resolved intents with counts

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_date, channel_id)
);

CREATE INDEX idx_analytics_tenant_date ON self_service_analytics(tenant_id, period_date);
CREATE INDEX idx_analytics_tenant_channel ON self_service_analytics(tenant_id, channel_id);
```

---

## 3. REST API Endpoints

### Channels
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/channels` | Create a digital channel |
| GET | `/api/v1/channels` | List digital channels |
| GET | `/api/v1/channels/{id}` | Get channel details |
| PUT | `/api/v1/channels/{id}` | Update channel |
| POST | `/api/v1/channels/{id}/test` | Test channel connectivity |
| GET | `/api/v1/channels/{id}/metrics` | Get channel performance metrics |

### Sessions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sessions` | Start a new chat session |
| POST | `/api/v1/sessions/{id}/end` | End a chat session |
| GET | `/api/v1/sessions/{id}/history` | Get session message history |
| GET | `/api/v1/sessions/active` | List active sessions |

### Messages
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/messages` | Send a message in a session |
| GET | `/api/v1/messages` | Get message history for session |
| POST | `/api/v1/messages/upload-attachment` | Upload a message attachment |

### Intents
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/intents` | Create an intent definition |
| GET | `/api/v1/intents` | List intent definitions |
| GET | `/api/v1/intents/{id}` | Get intent details |
| PUT | `/api/v1/intents/{id}` | Update intent |
| POST | `/api/v1/intents/train` | Trigger intent model training |
| POST | `/api/v1/intents/test-match` | Test intent matching |
| GET | `/api/v1/intents/{id}/performance` | Get intent performance metrics |

### Knowledge Articles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/knowledge-articles` | Create a knowledge article |
| GET | `/api/v1/knowledge-articles` | List knowledge articles |
| GET | `/api/v1/knowledge-articles/{id}` | Get article details |
| PUT | `/api/v1/knowledge-articles/{id}` | Update article |
| POST | `/api/v1/knowledge-articles/{id}/publish` | Publish an article |
| POST | `/api/v1/knowledge-articles/{id}/track-view` | Track article view |
| POST | `/api/v1/knowledge-articles/{id}/rate` | Rate article helpfulness |

### Escalation Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/escalation-rules` | Create an escalation rule |
| GET | `/api/v1/escalation-rules` | List escalation rules |
| GET | `/api/v1/escalation-rules/{id}` | Get rule details |
| PUT | `/api/v1/escalation-rules/{id}` | Update rule |
| POST | `/api/v1/escalation-rules/evaluate` | Evaluate escalation conditions |
| POST | `/api/v1/sessions/{id}/manual-escalate` | Manually escalate a session |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/analytics/containment-rate` | Get self-service containment rate |
| GET | `/api/v1/analytics/intent-performance` | Get intent resolution performance |
| GET | `/api/v1/analytics/channel-comparison` | Compare channel metrics |
| GET | `/api/v1/analytics/deflection-stats` | Get case deflection statistics |

---

## 4. Business Rules

1. Chat sessions with no activity for 30 minutes MUST be automatically ended with resolution_status set to ABANDONED.
2. Intent matching with a confidence_score below 0.65 MUST trigger a clarification prompt rather than proceeding with fulfillment.
3. Escalation rules MUST be evaluated in priority order and the first matching rule MUST determine the escalation action.
4. Knowledge articles that have not been reviewed or updated in 180 days MUST be flagged for review and SHOULD be deprioritized in search results.
5. Each tenant MUST have at least one active digital channel configured before chat sessions can be initiated.
6. Customer satisfaction ratings MUST be collected after session resolution and MUST NOT be prompted for sessions that were escalated to a live agent within the first 30 seconds.
7. Intent training MUST use at least 10 distinct training phrases per intent to ensure acceptable model accuracy.
8. The containment_rate metric MUST be calculated as (self_served_count / total_sessions) * 100 and MUST exclude abandoned sessions from the denominator.
9. Escalated sessions MUST respect the configured SLA and MUST trigger an alert if the SLA is breached before an agent responds.
10. Knowledge article view counts MUST be incremented atomically to prevent race conditions during concurrent access.
11. Messages with message_type FILE or IMAGE MUST require an attachment reference and MUST validate the attachment exists before delivery.
12. The system SHOULD auto-suggest knowledge articles to customers based on detected intent before offering agent escalation.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package digitalcs.v1;

service DigitalCustomerServiceService {
    // Channels
    rpc CreateChannel(CreateChannelRequest) returns (CreateChannelResponse);
    rpc GetChannel(GetChannelRequest) returns (GetChannelResponse);
    rpc ListChannels(ListChannelsRequest) returns (ListChannelsResponse);
    rpc TestChannel(TestChannelRequest) returns (TestChannelResponse);
    rpc GetChannelMetrics(GetChannelMetricsRequest) returns (GetChannelMetricsResponse);

    // Sessions
    rpc StartSession(StartSessionRequest) returns (StartSessionResponse);
    rpc EndSession(EndSessionRequest) returns (EndSessionResponse);
    rpc GetSessionHistory(GetSessionHistoryRequest) returns (GetSessionHistoryResponse);
    rpc ListActiveSessions(ListActiveSessionsRequest) returns (ListActiveSessionsResponse);

    // Messages
    rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
    rpc GetMessageHistory(GetMessageHistoryRequest) returns (GetMessageHistoryResponse);

    // Intents
    rpc CreateIntent(CreateIntentRequest) returns (CreateIntentResponse);
    rpc GetIntent(GetIntentRequest) returns (GetIntentResponse);
    rpc ListIntents(ListIntentsRequest) returns (ListIntentsResponse);
    rpc TrainIntents(TrainIntentsRequest) returns (TrainIntentsResponse);
    rpc TestIntentMatch(TestIntentMatchRequest) returns (TestIntentMatchResponse);
    rpc GetIntentPerformance(GetIntentPerformanceRequest) returns (GetIntentPerformanceResponse);

    // Knowledge Articles
    rpc CreateKnowledgeArticle(CreateKnowledgeArticleRequest) returns (CreateKnowledgeArticleResponse);
    rpc GetKnowledgeArticle(GetKnowledgeArticleRequest) returns (GetKnowledgeArticleResponse);
    rpc ListKnowledgeArticles(ListKnowledgeArticlesRequest) returns (ListKnowledgeArticlesResponse);
    rpc PublishKnowledgeArticle(PublishKnowledgeArticleRequest) returns (PublishKnowledgeArticleResponse);
    rpc TrackArticleView(TrackArticleViewRequest) returns (TrackArticleViewResponse);
    rpc RateArticle(RateArticleRequest) returns (RateArticleResponse);

    // Escalation
    rpc CreateEscalationRule(CreateEscalationRuleRequest) returns (CreateEscalationRuleResponse);
    rpc ListEscalationRules(ListEscalationRulesRequest) returns (ListEscalationRulesResponse);
    rpc EvaluateEscalation(EvaluateEscalationRequest) returns (EvaluateEscalationResponse);
    rpc ManualEscalate(ManualEscalateRequest) returns (ManualEscalateResponse);

    // Analytics
    rpc GetContainmentRate(GetContainmentRateRequest) returns (GetContainmentRateResponse);
    rpc GetIntentPerformanceReport(GetIntentPerformanceReportRequest) returns (GetIntentPerformanceReportResponse);
    rpc GetChannelComparison(GetChannelComparisonRequest) returns (GetChannelComparisonResponse);
    rpc GetDeflectionStats(GetDeflectionStatsRequest) returns (GetDeflectionStatsResponse);
}

message DigitalChannel {
    string id = 1;
    string tenant_id = 2;
    string channel_name = 3;
    string channel_type = 4;
    string configuration = 5;
    string bot_personality_config = 6;
    string welcome_message = 7;
    string operating_hours = 8;
    string status = 9;
    int32 avg_response_time_ms = 10;
    double resolution_rate = 11;
}

message CreateChannelRequest {
    string tenant_id = 1;
    string channel_name = 2;
    string channel_type = 3;
    string configuration = 4;
    string bot_personality_config = 5;
    string welcome_message = 6;
    string created_by = 7;
}

message CreateChannelResponse {
    DigitalChannel channel = 1;
}

message GetChannelRequest {
    string tenant_id = 1;
    string channel_id = 2;
}

message GetChannelResponse {
    DigitalChannel channel = 1;
}

message ListChannelsRequest {
    string tenant_id = 1;
    string channel_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListChannelsResponse {
    repeated DigitalChannel channels = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TestChannelRequest {
    string tenant_id = 1;
    string channel_id = 2;
}

message TestChannelResponse {
    bool connected = 1;
    string test_result = 2;
    int32 latency_ms = 3;
}

message ChannelMetrics {
    string channel_id = 1;
    int32 total_sessions = 2;
    double resolution_rate = 3;
    int32 avg_response_time_ms = 4;
    double avg_satisfaction = 5;
    double containment_rate = 6;
}

message GetChannelMetricsRequest {
    string tenant_id = 1;
    string channel_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetChannelMetricsResponse {
    ChannelMetrics metrics = 1;
}

message ChatSession {
    string id = 1;
    string tenant_id = 2;
    string channel_id = 3;
    string customer_id = 4;
    string session_token = 5;
    string start_time = 6;
    string end_time = 7;
    int32 duration_seconds = 8;
    int32 message_count = 9;
    string language = 10;
    string initial_intent = 11;
    string resolved_intent = 12;
    string resolution_status = 13;
    int32 satisfaction_rating = 14;
    string agent_id = 15;
}

message StartSessionRequest {
    string tenant_id = 1;
    string channel_id = 2;
    string customer_id = 3;
    string language = 4;
    string initial_message = 5;
}

message StartSessionResponse {
    ChatSession session = 1;
    string welcome_message = 2;
}

message EndSessionRequest {
    string tenant_id = 1;
    string session_id = 2;
    string resolution_status = 3;
    int32 satisfaction_rating = 4;
}

message EndSessionResponse {
    ChatSession session = 1;
}

message GetSessionHistoryRequest {
    string tenant_id = 1;
    string session_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetSessionHistoryResponse {
    repeated ChatMessage messages = 1;
    string next_page_token = 2;
}

message ListActiveSessionsRequest {
    string tenant_id = 1;
    string channel_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListActiveSessionsResponse {
    repeated ChatSession sessions = 1;
    string next_page_token = 2;
}

message ChatMessage {
    string id = 1;
    string tenant_id = 2;
    string session_id = 3;
    string sender_type = 4;
    string message_content = 5;
    string message_type = 6;
    string intent_detected = 7;
    string entities_extracted = 8;
    double confidence_score = 9;
    int32 response_time_ms = 10;
    string attachments = 11;
}

message SendMessageRequest {
    string tenant_id = 1;
    string session_id = 2;
    string sender_type = 3;
    string message_content = 4;
    string message_type = 5;
    string attachments = 6;
}

message SendMessageResponse {
    ChatMessage message = 1;
    repeated ChatMessage bot_responses = 2;
}

message GetMessageHistoryRequest {
    string tenant_id = 1;
    string session_id = 2;
    string sender_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetMessageHistoryResponse {
    repeated ChatMessage messages = 1;
    string next_page_token = 2;
}

message IntentDefinition {
    string id = 1;
    string tenant_id = 2;
    string intent_name = 3;
    string training_phrases = 4;
    string response_templates = 5;
    string escalation_rules = 6;
    string required_entities = 7;
    string fulfillment_type = 8;
    string parent_intent_id = 9;
    bool is_active = 10;
    int32 avg_resolution_time_seconds = 11;
    double success_rate = 12;
}

message CreateIntentRequest {
    string tenant_id = 1;
    string intent_name = 2;
    string training_phrases = 3;
    string response_templates = 4;
    string required_entities = 5;
    string fulfillment_type = 6;
    string created_by = 7;
}

message CreateIntentResponse {
    IntentDefinition intent = 1;
}

message GetIntentRequest {
    string tenant_id = 1;
    string intent_id = 2;
}

message GetIntentResponse {
    IntentDefinition intent = 1;
}

message ListIntentsRequest {
    string tenant_id = 1;
    bool active_only = 2;
    string fulfillment_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListIntentsResponse {
    repeated IntentDefinition intents = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TrainIntentsRequest {
    string tenant_id = 1;
    string trained_by = 2;
}

message TrainIntentsResponse {
    string training_job_id = 1;
    int32 intents_trained = 2;
    double accuracy_score = 3;
}

message TestIntentMatchRequest {
    string tenant_id = 1;
    string test_phrase = 2;
}

message TestIntentMatchResponse {
    string matched_intent = 1;
    double confidence_score = 2;
    string entities_extracted = 3;
}

message IntentPerformanceMetrics {
    string intent_id = 1;
    string intent_name = 2;
    int32 total_hits = 3;
    double success_rate = 4;
    int32 avg_resolution_time_seconds = 5;
    double escalation_rate = 6;
}

message GetIntentPerformanceRequest {
    string tenant_id = 1;
    string intent_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetIntentPerformanceResponse {
    IntentPerformanceMetrics metrics = 1;
}

message KnowledgeArticle {
    string id = 1;
    string tenant_id = 2;
    string article_title = 3;
    string article_content = 4;
    string article_type = 5;
    string tags = 6;
    int32 view_count = 7;
    int32 helpful_count = 8;
    int32 not_helpful_count = 9;
    string language = 10;
    string intent_mapping = 11;
    bool is_published = 12;
    string published_date = 13;
}

message CreateKnowledgeArticleRequest {
    string tenant_id = 1;
    string article_title = 2;
    string article_content = 3;
    string article_type = 4;
    string tags = 5;
    string language = 6;
    string intent_mapping = 7;
    string created_by = 8;
}

message CreateKnowledgeArticleResponse {
    KnowledgeArticle article = 1;
}

message GetKnowledgeArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message GetKnowledgeArticleResponse {
    KnowledgeArticle article = 1;
}

message ListKnowledgeArticlesRequest {
    string tenant_id = 1;
    string article_type = 2;
    bool published_only = 3;
    string language = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListKnowledgeArticlesResponse {
    repeated KnowledgeArticle articles = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishKnowledgeArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string published_by = 3;
}

message PublishKnowledgeArticleResponse {
    KnowledgeArticle article = 1;
}

message TrackArticleViewRequest {
    string tenant_id = 1;
    string article_id = 2;
    string customer_id = 3;
}

message TrackArticleViewResponse {
    int32 view_count = 1;
}

message RateArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    bool helpful = 3;
}

message RateArticleResponse {
    int32 helpful_count = 1;
    int32 not_helpful_count = 2;
}

message EscalationRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string trigger_condition = 4;
    string escalation_type = 5;
    string target_queue_id = 6;
    string target_agent_id = 7;
    int32 priority_boost = 8;
    int32 sla_minutes = 9;
    bool is_active = 10;
}

message CreateEscalationRuleRequest {
    string tenant_id = 1;
    string rule_name = 2;
    string trigger_condition = 3;
    string escalation_type = 4;
    string target_queue_id = 5;
    int32 sla_minutes = 6;
    string created_by = 7;
}

message CreateEscalationRuleResponse {
    EscalationRule rule = 1;
}

message ListEscalationRulesRequest {
    string tenant_id = 1;
    string escalation_type = 2;
    bool active_only = 3;
}

message ListEscalationRulesResponse {
    repeated EscalationRule rules = 1;
}

message EvaluateEscalationRequest {
    string tenant_id = 1;
    string session_id = 2;
    string context = 3;
}

message EvaluateEscalationResponse {
    bool should_escalate = 1;
    string escalation_type = 2;
    string target_queue_id = 3;
    string target_agent_id = 4;
}

message ManualEscalateRequest {
    string tenant_id = 1;
    string session_id = 2;
    string reason = 3;
    string target_queue_id = 4;
    string escalated_by = 5;
}

message ManualEscalateResponse {
    ChatSession session = 1;
}

message ContainmentRateData {
    string period_date = 1;
    int32 total_sessions = 2;
    int32 self_served_count = 3;
    int32 escalated_count = 4;
    int32 abandoned_count = 5;
    double containment_rate = 6;
}

message GetContainmentRateRequest {
    string tenant_id = 1;
    string channel_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message GetContainmentRateResponse {
    repeated ContainmentRateData data_points = 1;
    double overall_containment_rate = 2;
}

message IntentPerformanceReportItem {
    string intent_name = 1;
    int32 total_hits = 2;
    double success_rate = 3;
    double escalation_rate = 4;
    int32 avg_resolution_time_seconds = 5;
}

message GetIntentPerformanceReportRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
    int32 top_n = 4;
}

message GetIntentPerformanceReportResponse {
    repeated IntentPerformanceReportItem items = 1;
}

message ChannelComparisonItem {
    string channel_id = 1;
    string channel_name = 2;
    string channel_type = 3;
    int32 total_sessions = 4;
    double containment_rate = 5;
    double avg_satisfaction = 6;
    int32 avg_response_time_ms = 7;
}

message GetChannelComparisonRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message GetChannelComparisonResponse {
    repeated ChannelComparisonItem channels = 1;
}

message DeflectionStats {
    int32 total_interactions = 1;
    int32 deflected_to_self_service = 2;
    int32 escalated_to_agent = 3;
    double deflection_rate = 4;
    double cost_savings_cents = 5;
}

message GetDeflectionStatsRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message GetDeflectionStatsResponse {
    DeflectionStats stats = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `customer-service` | Customer profiles, case history, interaction logs | Resolve customer identity and context |
| `knowledge-service` | Knowledge base articles, FAQs | Serve article content in self-service |
| `workflow-service` | Workflow definitions, process automation | Fulfill intent-driven workflows |
| `auth-service` | Customer authentication tokens | Validate session identity |
| `notification-service` | Notification preferences | Deliver escalation alerts to agents |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `customer-service` | Session transcripts, resolution outcomes | Enrich customer interaction history |
| `notification-service` | Escalation alerts, SLA breach warnings | Notify agents and supervisors |
| `reporting-service` | Self-service analytics, containment rates, intent metrics | Contact center dashboards |
| `knowledge-service` | Article view counts, helpfulness ratings | Knowledge base optimization |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `SessionStarted` | `digitalcs.session.started` | `{ tenant_id, session_id, channel_id, channel_type, customer_id, language, initial_intent }` | Published when a new chat session is initiated |
| `IntentDetected` | `digitalcs.intent.detected` | `{ tenant_id, session_id, intent_name, confidence_score, entities_extracted }` | Published when an intent is matched during a session |
| `SessionEscalated` | `digitalcs.session.escalated` | `{ tenant_id, session_id, channel_id, escalation_type, target_queue_id, reason }` | Published when a session is escalated to a live agent |
| `SessionResolved` | `digitalcs.session.resolved` | `{ tenant_id, session_id, channel_id, resolution_status, resolved_intent, duration_seconds, satisfaction_rating }` | Published when a session is resolved or ended |
| `KnowledgeArticleViewed` | `digitalcs.article.viewed` | `{ tenant_id, article_id, session_id, customer_id, article_type }` | Published when a knowledge article is viewed during a session |
