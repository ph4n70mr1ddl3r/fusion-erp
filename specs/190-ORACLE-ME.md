# 190 - Oracle ME Platform Service Specification

## 1. Domain Overview

Oracle ME (Mobilization Engine) provides a conversational AI and digital assistant platform covering skill definitions with configurable invocation phrases and channel enablement, multi-step conversation flows with message, question, API call, condition, and transfer node types, intent mapping with training phrases and machine learning-based confidence scoring, cross-channel chat session management (web, mobile, Slack, Teams, SMS) with resolution and satisfaction tracking, and comprehensive analytics on resolution rates, popular skills, escalation patterns, and user satisfaction. The system enables users to interact with Fusion modules through natural conversation, automating common tasks and providing instant answers. Integrates with Auth for user context, Enterprise Search for knowledge retrieval, and all Fusion modules for skill execution.

**Bounded Context:** Conversational AI & Digital Assistant Platform
**Service Name:** `oracle-me-service`
**Database:** `data/oracle_me.db`
**HTTP Port:** 8208 | **gRPC Port:** 9208

---

## 2. Database Schema

### 2.1 Skill Definitions
```sql
CREATE TABLE me_skill_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    skill_code TEXT NOT NULL,
    skill_name TEXT NOT NULL,
    description TEXT,
    invocation_phrases TEXT NOT NULL,                 -- JSON: array of trigger phrases ["show my expenses", "create report"]
    enabled_channels TEXT NOT NULL,                   -- JSON: array of channels ["WEB","MOBILE","SLACK","TEAMS","SMS"]
    module_ref TEXT NOT NULL,                         -- Reference to the Fusion module this skill targets
    owner_id TEXT NOT NULL,
    fallback_flow_id TEXT,
    skill_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(skill_status IN ('DRAFT','ACTIVE','INACTIVE','DEPRECATED')),
    invocation_count INTEGER NOT NULL DEFAULT 0,
    last_invoked_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, skill_code)
);

CREATE INDEX idx_skill_defs_tenant_status ON me_skill_definitions(tenant_id, skill_status);
CREATE INDEX idx_skill_defs_tenant_module ON me_skill_definitions(tenant_id, module_ref);
CREATE INDEX idx_skill_defs_tenant_owner ON me_skill_definitions(tenant_id, owner_id);
CREATE INDEX idx_skill_defs_tenant_invocations ON me_skill_definitions(tenant_id, invocation_count);
CREATE INDEX idx_skill_defs_tenant_active ON me_skill_definitions(tenant_id, is_active);
```

### 2.2 Conversation Flows
```sql
CREATE TABLE me_conversation_flows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    flow_code TEXT NOT NULL,
    flow_name TEXT NOT NULL,
    description TEXT,
    skill_id TEXT NOT NULL,
    nodes TEXT NOT NULL,                              -- JSON: ordered array of nodes [{type, config, next_node}]
                                                     -- Types: MESSAGE, QUESTION, API_CALL, CONDITION, TRANSFER
    entry_node_id TEXT,
    error_handling_config TEXT,                      -- JSON: fallback messages and retry logic
    timeout_seconds INTEGER NOT NULL DEFAULT 300
        CHECK(timeout_seconds > 0),
    parent_flow_id TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    flow_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(flow_status IN ('DRAFT','ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES me_skill_definitions(id),
    UNIQUE(tenant_id, flow_code)
);

CREATE INDEX idx_conv_flows_tenant_skill ON me_conversation_flows(tenant_id, skill_id);
CREATE INDEX idx_conv_flows_tenant_status ON me_conversation_flows(tenant_id, flow_status);
CREATE INDEX idx_conv_flows_tenant_parent ON me_conversation_flows(tenant_id, parent_flow_id);
CREATE INDEX idx_conv_flows_tenant_active ON me_conversation_flows(tenant_id, is_active);
```

### 2.3 Intent Mappings
```sql
CREATE TABLE me_intent_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    intent_name TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    flow_id TEXT,
    training_phrases TEXT NOT NULL,                   -- JSON: array of training phrase strings
    entity_extraction TEXT,                           -- JSON: array of entities [{name, type, prompt, required}]
    confidence_threshold REAL NOT NULL DEFAULT 0.70
        CHECK(confidence_threshold >= 0.0 AND confidence_threshold <= 1.0),
    fallback_flow_id TEXT,
    priority INTEGER NOT NULL DEFAULT 5
        CHECK(priority >= 1 AND priority <= 10),
    intent_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(intent_status IN ('ACTIVE','INACTIVE','TRAINING')),
    model_version TEXT,
    last_trained_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES me_skill_definitions(id),
    FOREIGN KEY (flow_id) REFERENCES me_conversation_flows(id),
    UNIQUE(tenant_id, intent_name)
);

CREATE INDEX idx_intent_maps_tenant_skill ON me_intent_mappings(tenant_id, skill_id);
CREATE INDEX idx_intent_maps_tenant_flow ON me_intent_mappings(tenant_id, flow_id);
CREATE INDEX idx_intent_maps_tenant_status ON me_intent_mappings(tenant_id, intent_status);
CREATE INDEX idx_intent_maps_tenant_priority ON me_intent_mappings(tenant_id, priority);
CREATE INDEX idx_intent_maps_tenant_active ON me_intent_mappings(tenant_id, is_active);
```

### 2.4 Chat Sessions
```sql
CREATE TABLE me_chat_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    channel TEXT NOT NULL
        CHECK(channel IN ('WEB','MOBILE','SLACK','TEAMS','SMS')),
    skill_invoked TEXT,
    intent_matched TEXT,
    flow_id TEXT,
    start_time TEXT NOT NULL,
    end_time TEXT,
    message_count INTEGER NOT NULL DEFAULT 0,
    resolution_status TEXT NOT NULL DEFAULT 'IN_PROGRESS'
        CHECK(resolution_status IN ('IN_PROGRESS','RESOLVED','ESCALATED','ABANDONED','TIMEOUT')),
    escalation_reason TEXT,
    escalated_to TEXT,
    escalation_channel TEXT,
    satisfaction_rating REAL
        CHECK(satisfaction_rating IS NULL OR (satisfaction_rating >= 1.0 AND satisfaction_rating <= 5.0)),
    feedback_text TEXT,
    conversation_log TEXT,                            -- JSON: array of message objects {role, content, timestamp}
    session_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(session_status IN ('ACTIVE','COMPLETED','ESCALATED','ABANDONED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, session_id)
);

CREATE INDEX idx_chat_sessions_tenant_user ON me_chat_sessions(tenant_id, user_id);
CREATE INDEX idx_chat_sessions_tenant_channel ON me_chat_sessions(tenant_id, channel);
CREATE INDEX idx_chat_sessions_tenant_resolution ON me_chat_sessions(tenant_id, resolution_status);
CREATE INDEX idx_chat_sessions_tenant_status ON me_chat_sessions(tenant_id, session_status);
CREATE INDEX idx_chat_sessions_tenant_skill ON me_chat_sessions(tenant_id, skill_invoked);
CREATE INDEX idx_chat_sessions_tenant_start ON me_chat_sessions(tenant_id, start_time);
CREATE INDEX idx_chat_sessions_tenant_satisfaction ON me_chat_sessions(tenant_id, satisfaction_rating);
```

### 2.5 Analytics
```sql
CREATE TABLE me_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('DAILY','WEEKLY','MONTHLY')),
    total_sessions INTEGER NOT NULL DEFAULT 0,
    resolved_sessions INTEGER NOT NULL DEFAULT 0,
    escalated_sessions INTEGER NOT NULL DEFAULT 0,
    abandoned_sessions INTEGER NOT NULL DEFAULT 0,
    resolution_rate REAL NOT NULL DEFAULT 0.0
        CHECK(resolution_rate >= 0.0 AND resolution_rate <= 100.0),
    escalation_rate REAL NOT NULL DEFAULT 0.0,
    avg_session_duration_seconds REAL NOT NULL DEFAULT 0.0,
    avg_response_time_ms REAL NOT NULL DEFAULT 0.0,
    avg_messages_per_session REAL NOT NULL DEFAULT 0.0,
    popular_skills TEXT,                              -- JSON: array of {skill_code, skill_name, invocation_count}
    channel_distribution TEXT,                        -- JSON: {channel: session_count}
    satisfaction_score_avg REAL
        CHECK(satisfaction_score_avg IS NULL OR (satisfaction_score_avg >= 1.0 AND satisfaction_score_avg <= 5.0)),
    unique_users INTEGER NOT NULL DEFAULT 0,
    peak_concurrent_sessions INTEGER NOT NULL DEFAULT 0,
    top_escalation_reasons TEXT,                      -- JSON: array of {reason, count}

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_start, period_type)
);

CREATE INDEX idx_analytics_tenant_period ON me_analytics(tenant_id, period_start);
CREATE INDEX idx_analytics_tenant_type ON me_analytics(tenant_id, period_type);
CREATE INDEX idx_analytics_tenant_resolution ON me_analytics(tenant_id, resolution_rate);
CREATE INDEX idx_analytics_tenant_active ON me_analytics(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Skill Definitions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/me/skills` | List skill definitions |
| POST | `/api/v1/me/skills` | Create skill definition |
| GET | `/api/v1/me/skills/{id}` | Get skill details |
| PUT | `/api/v1/me/skills/{id}` | Update skill |
| PATCH | `/api/v1/me/skills/{id}/status` | Update skill status |
| GET | `/api/v1/me/skills/search` | Search skills by invocation phrase |

### 3.2 Conversation Flows
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/me/flows` | List conversation flows |
| POST | `/api/v1/me/flows` | Create conversation flow |
| GET | `/api/v1/me/flows/{id}` | Get flow details |
| PUT | `/api/v1/me/flows/{id}` | Update flow |
| POST | `/api/v1/me/flows/{id}/validate` | Validate flow node structure |
| POST | `/api/v1/me/flows/{id}/publish` | Publish flow |

### 3.3 Intent Mappings
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/me/intents` | List intent mappings |
| POST | `/api/v1/me/intents` | Create intent mapping |
| GET | `/api/v1/me/intents/{id}` | Get intent details |
| PUT | `/api/v1/me/intents/{id}` | Update intent mapping |
| POST | `/api/v1/me/intents/train` | Trigger model retraining |
| POST | `/api/v1/me/intents/test` | Test intent matching |

### 3.4 Chat Sessions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/me/sessions` | Start new chat session |
| POST | `/api/v1/me/sessions/{id}/message` | Send message in session |
| GET | `/api/v1/me/sessions/{id}` | Get session details and history |
| POST | `/api/v1/me/sessions/{id}/end` | End chat session |
| POST | `/api/v1/me/sessions/{id}/escalate` | Escalate to human agent |
| POST | `/api/v1/me/sessions/{id}/feedback` | Submit session feedback |
| GET | `/api/v1/me/sessions` | List chat sessions |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/me/analytics/dashboard` | Get analytics dashboard |
| GET | `/api/v1/me/analytics/skills` | Get skill usage report |
| GET | `/api/v1/me/analytics/resolution` | Get resolution rate trends |
| GET | `/api/v1/me/analytics/satisfaction` | Get satisfaction trends |
| GET | `/api/v1/me/analytics/channels` | Get channel distribution |

---

## 4. Business Rules

### 4.1 Skill Definitions
1. Each skill MUST have a unique skill_code within a tenant.
2. invocation_phrases MUST contain at least one phrase string.
3. enabled_channels MUST contain at least one valid channel identifier.
4. A skill in DRAFT status MUST NOT be invocable by end users; it MUST be set to ACTIVE first.
5. The fallback_flow_id, when specified, MUST reference a valid flow within the same tenant.
6. invocation_count MUST be incremented atomically each time the skill is invoked.

### 4.2 Conversation Flows
7. Each flow MUST have at least one node in the nodes JSON array.
8. Node types MUST be one of: MESSAGE, QUESTION, API_CALL, CONDITION, TRANSFER.
9. QUESTION nodes MUST define at least one expected response or entity extraction rule.
10. API_CALL nodes MUST specify a target endpoint URL and HTTP method.
11. CONDITION nodes MUST define branching logic with at least two possible next node paths.
12. TRANSFER nodes MUST specify the escalation target (agent queue, phone number, or channel).
13. A flow in ACTIVE status MUST NOT have nodes removed or reordered; a new version MUST be created.
14. timeout_seconds MUST be a positive integer; sessions exceeding this duration MUST be auto-closed.

### 4.3 Intent Mappings
15. training_phrases MUST contain at least three distinct phrase variations per intent.
16. confidence_threshold MUST be between 0.0 and 1.0 inclusive.
17. Intents matching below the confidence_threshold MUST route to the fallback_flow_id.
18. Priority values MUST be between 1 (highest) and 10 (lowest); higher priority intents are preferred on ties.
19. Retraining MUST NOT be triggered more than once per hour per tenant.
20. Entity extraction rules with required=true MUST be validated during intent testing.

### 4.4 Chat Sessions
21. Each session MUST be associated with exactly one channel and one user.
22. Session status transitions MUST follow: ACTIVE -> COMPLETED or ACTIVE -> ESCALATED or ACTIVE -> ABANDONED.
23. A session in COMPLETED or ESCALATED status MUST NOT accept new messages.
24. satisfaction_rating, when provided, MUST be between 1.0 and 5.0 inclusive.
25. Inactive sessions MUST auto-close after a configurable timeout (default 15 minutes) with resolution_status TIMEOUT.
26. conversation_log MUST capture the full message history including timestamps and roles.

### 4.5 Analytics
27. Analytics MUST be aggregated per period_type (DAILY, WEEKLY, MONTHLY) per tenant.
28. resolution_rate MUST be calculated as (resolved_sessions / total_sessions) * 100 when total_sessions > 0.
29. Analytics records for a given period MUST NOT be modified after creation; corrections require a new record.
30. popular_skills MUST list skills ranked by invocation count in descending order.
31. peak_concurrent_sessions MUST track the maximum number of simultaneously active sessions.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.me.v1;

service OracleMEService {
    // Skill definitions
    rpc CreateSkill(CreateSkillRequest) returns (CreateSkillResponse);
    rpc GetSkill(GetSkillRequest) returns (GetSkillResponse);
    rpc ListSkills(ListSkillsRequest) returns (ListSkillsResponse);
    rpc UpdateSkill(UpdateSkillRequest) returns (UpdateSkillResponse);

    // Conversation flows
    rpc CreateFlow(CreateFlowRequest) returns (CreateFlowResponse);
    rpc GetFlow(GetFlowRequest) returns (GetFlowResponse);
    rpc ListFlows(ListFlowsRequest) returns (ListFlowsResponse);
    rpc ValidateFlow(ValidateFlowRequest) returns (ValidateFlowResponse);
    rpc PublishFlow(PublishFlowRequest) returns (PublishFlowResponse);

    // Intent mappings
    rpc CreateIntent(CreateIntentRequest) returns (CreateIntentResponse);
    rpc GetIntent(GetIntentRequest) returns (GetIntentResponse);
    rpc ListIntents(ListIntentsRequest) returns (ListIntentsResponse);
    rpc TrainIntents(TrainIntentsRequest) returns (TrainIntentsResponse);
    rpc TestIntent(TestIntentRequest) returns (TestIntentResponse);

    // Chat sessions
    rpc StartSession(StartSessionRequest) returns (StartSessionResponse);
    rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
    rpc GetSession(GetSessionRequest) returns (GetSessionResponse);
    rpc EndSession(EndSessionRequest) returns (EndSessionResponse);
    rpc EscalateSession(EscalateSessionRequest) returns (EscalateSessionResponse);
    rpc SubmitFeedback(SubmitFeedbackRequest) returns (SubmitFeedbackResponse);
    rpc ListSessions(ListSessionsRequest) returns (ListSessionsResponse);

    // Analytics
    rpc GetAnalyticsDashboard(GetAnalyticsDashboardRequest) returns (GetAnalyticsDashboardResponse);
    rpc GetSkillUsage(GetSkillUsageRequest) returns (GetSkillUsageResponse);
    rpc GetResolutionTrends(GetResolutionTrendsRequest) returns (GetResolutionTrendsResponse);
    rpc GetSatisfactionTrends(GetSatisfactionTrendsRequest) returns (GetSatisfactionTrendsResponse);
}

message CreateSkillRequest {
    string tenant_id = 1;
    string skill_code = 2;
    string skill_name = 3;
    string description = 4;
    string invocation_phrases = 5;
    string enabled_channels = 6;
    string module_ref = 7;
    string owner_id = 8;
    string fallback_flow_id = 9;
}

message CreateSkillResponse {
    string skill_id = 1;
    string skill_code = 2;
    string skill_status = 3;
}

message GetSkillRequest {
    string tenant_id = 1;
    string skill_id = 2;
}

message GetSkillResponse {
    string skill_id = 1;
    string skill_code = 2;
    string skill_name = 3;
    string description = 4;
    string invocation_phrases = 5;
    string enabled_channels = 6;
    string module_ref = 7;
    string owner_id = 8;
    string fallback_flow_id = 9;
    string skill_status = 10;
    int32 invocation_count = 11;
    string last_invoked_at = 12;
}

message ListSkillsRequest {
    string tenant_id = 1;
    string skill_status = 2;
    string module_ref = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListSkillsResponse {
    repeated GetSkillResponse skills = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateSkillRequest {
    string tenant_id = 1;
    string skill_id = 2;
    string skill_name = 3;
    string invocation_phrases = 4;
    string enabled_channels = 5;
    string skill_status = 6;
}

message UpdateSkillResponse {
    string skill_id = 1;
    string skill_status = 2;
    int32 version = 3;
}

message CreateFlowRequest {
    string tenant_id = 1;
    string flow_code = 2;
    string flow_name = 3;
    string description = 4;
    string skill_id = 5;
    string nodes = 6;
    string entry_node_id = 7;
    string error_handling_config = 8;
    int32 timeout_seconds = 9;
    string parent_flow_id = 10;
    bool is_default = 11;
}

message CreateFlowResponse {
    string flow_id = 1;
    string flow_code = 2;
    string flow_status = 3;
}

message GetFlowRequest {
    string tenant_id = 1;
    string flow_id = 2;
}

message GetFlowResponse {
    string flow_id = 1;
    string flow_code = 2;
    string flow_name = 3;
    string description = 4;
    string skill_id = 5;
    string nodes = 6;
    string entry_node_id = 7;
    string error_handling_config = 8;
    int32 timeout_seconds = 9;
    string parent_flow_id = 10;
    bool is_default = 11;
    string flow_status = 12;
    int32 version = 13;
}

message ListFlowsRequest {
    string tenant_id = 1;
    string skill_id = 2;
    string flow_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListFlowsResponse {
    repeated GetFlowResponse flows = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ValidateFlowRequest {
    string tenant_id = 1;
    string flow_id = 2;
}

message ValidateFlowResponse {
    bool is_valid = 1;
    repeated string errors = 2;
    repeated string warnings = 3;
}

message PublishFlowRequest {
    string tenant_id = 1;
    string flow_id = 2;
}

message PublishFlowResponse {
    string flow_id = 1;
    string flow_status = 2;
}

message CreateIntentRequest {
    string tenant_id = 1;
    string intent_name = 2;
    string skill_id = 3;
    string flow_id = 4;
    string training_phrases = 5;
    string entity_extraction = 6;
    double confidence_threshold = 7;
    string fallback_flow_id = 8;
    int32 priority = 9;
}

message CreateIntentResponse {
    string intent_id = 1;
    string intent_name = 2;
    string intent_status = 3;
}

message GetIntentRequest {
    string tenant_id = 1;
    string intent_id = 2;
}

message GetIntentResponse {
    string intent_id = 1;
    string intent_name = 2;
    string skill_id = 3;
    string flow_id = 4;
    string training_phrases = 5;
    string entity_extraction = 6;
    double confidence_threshold = 7;
    string fallback_flow_id = 8;
    int32 priority = 9;
    string intent_status = 10;
    string model_version = 11;
    string last_trained_at = 12;
}

message ListIntentsRequest {
    string tenant_id = 1;
    string skill_id = 2;
    string intent_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListIntentsResponse {
    repeated GetIntentResponse intents = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TrainIntentsRequest {
    string tenant_id = 1;
    repeated string intent_ids = 2;
}

message TrainIntentsResponse {
    string model_version = 1;
    int32 intents_trained = 2;
    string completed_at = 3;
}

message TestIntentRequest {
    string tenant_id = 1;
    string test_phrase = 2;
}

message TestIntentResponse {
    string matched_intent = 1;
    double confidence = 2;
    string matched_skill = 3;
    string matched_flow = 4;
}

message StartSessionRequest {
    string tenant_id = 1;
    string user_id = 2;
    string channel = 3;
    string initial_message = 4;
}

message StartSessionResponse {
    string session_id = 1;
    string matched_skill = 2;
    string matched_intent = 3;
    string initial_response = 4;
}

message SendMessageRequest {
    string tenant_id = 1;
    string session_id = 2;
    string message = 3;
}

message SendMessageResponse {
    string session_id = 1;
    string response = 2;
    string response_type = 3;
    string next_node_id = 4;
}

message GetSessionRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message GetSessionResponse {
    string id = 1;
    string session_id = 2;
    string user_id = 3;
    string channel = 4;
    string skill_invoked = 5;
    string intent_matched = 6;
    string flow_id = 7;
    string start_time = 8;
    string end_time = 9;
    int32 message_count = 10;
    string resolution_status = 11;
    string session_status = 12;
    string conversation_log = 13;
}

message ListSessionsRequest {
    string tenant_id = 1;
    string user_id = 2;
    string channel = 3;
    string resolution_status = 4;
    string session_status = 5;
    string from_date = 6;
    string to_date = 7;
    int32 page_size = 8;
    string page_token = 9;
}

message ListSessionsResponse {
    repeated GetSessionResponse sessions = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message EndSessionRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message EndSessionResponse {
    string session_id = 1;
    string session_status = 2;
    string end_time = 3;
}

message EscalateSessionRequest {
    string tenant_id = 1;
    string session_id = 2;
    string escalation_reason = 3;
    string escalated_to = 4;
}

message EscalateSessionResponse {
    string session_id = 1;
    string session_status = 2;
    string resolution_status = 3;
    string escalation_channel = 4;
}

message SubmitFeedbackRequest {
    string tenant_id = 1;
    string session_id = 2;
    double satisfaction_rating = 3;
    string feedback_text = 4;
}

message SubmitFeedbackResponse {
    string session_id = 1;
    double satisfaction_rating = 2;
}

message GetAnalyticsDashboardRequest {
    string tenant_id = 1;
    string period_type = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetAnalyticsDashboardResponse {
    double overall_resolution_rate = 1;
    double overall_satisfaction_score = 2;
    int32 total_sessions = 3;
    int32 total_resolved = 4;
    int32 total_escalated = 5;
    double avg_session_duration_seconds = 6;
    int32 unique_users = 7;
    repeated SkillUsageEntry popular_skills = 8;
}

message SkillUsageEntry {
    string skill_code = 1;
    string skill_name = 2;
    int32 invocation_count = 3;
    double resolution_rate = 4;
}

message GetSkillUsageRequest {
    string tenant_id = 1;
    string from_date = 2;
    string to_date = 3;
}

message GetSkillUsageResponse {
    repeated SkillUsageEntry skills = 1;
}

message GetResolutionTrendsRequest {
    string tenant_id = 1;
    string from_date = 2;
    string to_date = 3;
    string period_type = 4;
}

message GetResolutionTrendsResponse {
    repeated ResolutionDataPoint data_points = 1;
}

message ResolutionDataPoint {
    string period = 1;
    double resolution_rate = 2;
    double escalation_rate = 3;
    int32 total_sessions = 4;
}

message GetSatisfactionTrendsRequest {
    string tenant_id = 1;
    string from_date = 2;
    string to_date = 3;
    string period_type = 4;
}

message GetSatisfactionTrendsResponse {
    repeated SatisfactionDataPoint data_points = 1;
}

message SatisfactionDataPoint {
    string period = 1;
    double satisfaction_score_avg = 2;
    int32 feedback_count = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `auth-service` | User authentication context, role resolution, permission checks for skill access |
| `enterprise-search-service` | Search indices for knowledge-based skill responses |
| `notification-service` | Proactive notification delivery through chat channels |
| `fusion-data-intelligence-service` | ML model hosting for intent classification and entity extraction |
| `redwood-ux-service` | Chat widget UI components and design system integration |

### Published To
| Service | Data |
|---------|------|
| `notification-service` | Chat-triggered notifications, escalation alerts to agents |
| `reporting-service` | Chat analytics, resolution metrics, satisfaction dashboards |
| `audit-service` | Conversation logs for compliance, data access audit trail |
| `all-fusion-modules` | Skill execution API calls (CRUD operations) |
| `workflow-service` | Escalation routing, approval workflows initiated from chat |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `me.chat.started` | `me.events` | `{ session_id, user_id, channel, skill_invoked, intent_matched, start_time }` | New chat session started |
| `me.chat.resolved` | `me.events` | `{ session_id, skill_invoked, resolution_status, message_count, duration_seconds }` | Chat session resolved |
| `me.chat.escalated` | `me.events` | `{ session_id, escalation_reason, escalated_to, escalation_channel, conversation_log }` | Chat escalated to human agent |
| `me.skill.invoked` | `me.events` | `{ skill_code, skill_name, user_id, channel, invocation_count }` | Skill invoked by user |
| `me.feedback.received` | `me.events` | `{ session_id, skill_invoked, satisfaction_rating, feedback_text }` | User feedback submitted |

---

## 8. Migrations

1. V001: `me_skill_definitions`
2. V002: `me_conversation_flows`
3. V003: `me_intent_mappings`
4. V004: `me_chat_sessions`
5. V005: `me_analytics`
6. V006: Triggers for `updated_at`
