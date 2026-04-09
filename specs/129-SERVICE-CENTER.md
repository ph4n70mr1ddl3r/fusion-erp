# 129 - Service Center Specification

## 1. Domain Overview

Service Center provides a unified agent workspace for omnichannel case management. It consolidates customer interactions across phone, email, chat, social media, and self-service into a single console with 360-degree customer view, intelligent routing, real-time collaboration, and supervisor monitoring capabilities.

**Bounded Context:** Agent Workspace & Omnichannel Console
**Service Name:** `svccenter-service`
**Database:** `data/svccenter.db`
**HTTP Port:** 8209 | **gRPC Port:** 9209

---

## 2. Database Schema

### 2.1 Agent Workspaces
```sql
CREATE TABLE sc_workspaces (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    workspace_name TEXT NOT NULL,
    workspace_type TEXT NOT NULL CHECK(workspace_type IN ('GENERAL','VIP','TECHNICAL','BILLING','RETENTION','SALES')),
    description TEXT,
    layout_config TEXT NOT NULL,         -- JSON: widget layout, panel positions, column config
    default_channels TEXT NOT NULL,      -- JSON array: ["CHAT","EMAIL","PHONE"]
    skill_requirements TEXT,             -- JSON array of required skills
    max_concurrent_cases INTEGER NOT NULL DEFAULT 5,
    auto_accept INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, workspace_name)
);
```

### 2.2 Agent Profiles
```sql
CREATE TABLE sc_agent_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    display_name TEXT NOT NULL,
    workspace_id TEXT,
    skills TEXT NOT NULL,                -- JSON array: ["billing", "technical", "vip"]
    proficiency_levels TEXT,             -- JSON: { "billing": "expert", "technical": "intermediate" }
    language_skills TEXT,                -- JSON array: ["en", "es", "fr"]
    status TEXT NOT NULL DEFAULT 'OFFLINE'
        CHECK(status IN ('OFFLINE','AVAILABLE','BUSY','AWAY','WRAP_UP','ON_BREAK')),
    status_since TEXT NOT NULL DEFAULT (datetime('now')),
    current_case_count INTEGER NOT NULL DEFAULT 0,
    max_case_capacity INTEGER NOT NULL DEFAULT 5,
    average_handle_time_seconds INTEGER NOT NULL DEFAULT 0,
    channels_active TEXT,                -- JSON array of active channels

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, user_id)
);

CREATE INDEX idx_sc_agents_tenant_status ON sc_agent_profiles(tenant_id, status);
```

### 2.3 Queue Management
```sql
CREATE TABLE sc_queues (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    queue_name TEXT NOT NULL,
    queue_type TEXT NOT NULL CHECK(queue_type IN ('CHAT','EMAIL','PHONE','SOCIAL','GENERAL')),
    description TEXT,
    routing_strategy TEXT NOT NULL DEFAULT 'ROUND_ROBIN'
        CHECK(routing_strategy IN ('ROUND_ROBIN','SKILL_BASED','LEAST_BUSY','PRIORITY','WEIGHTED')),
    priority INTEGER NOT NULL DEFAULT 5,
    max_wait_seconds INTEGER NOT NULL DEFAULT 300,
    overflow_queue_id TEXT,
    sla_response_seconds INTEGER,
    sla_resolution_seconds INTEGER,
    skill_requirements TEXT,             -- JSON array of required skills
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, queue_name)
);
```

### 2.4 Queue Items
```sql
CREATE TABLE sc_queue_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    queue_id TEXT NOT NULL,
    item_type TEXT NOT NULL CHECK(item_type IN ('CASE','CHAT','CALL','EMAIL','SOCIAL')),
    reference_id TEXT NOT NULL,          -- Case or interaction ID
    customer_id TEXT,
    customer_priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(customer_priority IN ('NORMAL','VIP','PREMIUM','URGENT')),
    skill_requirements TEXT,
    enqueued_at TEXT NOT NULL DEFAULT (datetime('now')),
    assigned_at TEXT,
    assigned_agent_id TEXT,
    wait_time_seconds INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'WAITING'
        CHECK(status IN ('WAITING','ASSIGNED','TIMEOUT','OVERFLOW','CANCELLED')),
    priority_score REAL NOT NULL DEFAULT 0.0,
    retry_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (queue_id) REFERENCES sc_queues(id) ON DELETE CASCADE
);

CREATE INDEX idx_sc_queue_items_tenant_queue ON sc_queue_items(tenant_id, queue_id, status, priority_score DESC);
CREATE INDEX idx_sc_queue_items_tenant_wait ON sc_queue_items(tenant_id, status, enqueued_at);
```

### 2.5 Omnichannel Interactions
```sql
CREATE TABLE sc_interactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    case_id TEXT,
    customer_id TEXT,
    agent_id TEXT,
    channel TEXT NOT NULL CHECK(channel IN ('CHAT','EMAIL','PHONE','SOCIAL','SMS','WHATSAPP','WEB_FORM','IN_PERSON')),
    direction TEXT NOT NULL CHECK(direction IN ('INBOUND','OUTBOUND')),
    subject TEXT,
    content TEXT NOT NULL,
    content_type TEXT NOT NULL DEFAULT 'TEXT' CHECK(content_type IN ('TEXT','HTML','RICH_TEXT')),
    attachments TEXT,                    -- JSON array
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','OPEN','IN_PROGRESS','WAITING','RESOLVED','CLOSED')),
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','URGENT')),
    handling_agent_id TEXT,
    start_time TEXT NOT NULL,
    end_time TEXT,
    handle_time_seconds INTEGER,
    wrap_up_notes TEXT,
    wrap_up_codes TEXT,                  -- JSON array of disposition codes
    sentiment_score REAL,
    tags TEXT,                           -- JSON array

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_sc_interactions_tenant_case ON sc_interactions(tenant_id, case_id);
CREATE INDEX idx_sc_interactions_tenant_agent ON sc_interactions(tenant_id, agent_id, start_time DESC);
CREATE INDEX idx_sc_interactions_tenant_customer ON sc_interactions(tenant_id, customer_id);
```

### 2.6 Supervisor Actions
```sql
CREATE TABLE sc_supervisor_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supervisor_id TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN ('LISTEN','WHISPER','BARGE_IN','TRANSFER','REASSIGN','CLOSE','ESCALATE','COACH')),
    interaction_id TEXT,
    agent_id TEXT,
    queue_item_id TEXT,
    notes TEXT,
    action_timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (interaction_id) REFERENCES sc_interactions(id) ON DELETE SET NULL
);

CREATE INDEX idx_sc_super_actions_tenant ON sc_supervisor_actions(tenant_id, supervisor_id, action_timestamp DESC);
```

---

## 3. REST API Endpoints

### 3.1 Agent Console
```
GET    /api/v1/svccenter/agent/profile                   Permission: sc.agent.read
PUT    /api/v1/svccenter/agent/status                    Permission: sc.agent.status
  Request: { "status": "AVAILABLE", "channels": ["CHAT","EMAIL"] }
GET    /api/v1/svccenter/agent/cases                     Permission: sc.agent.read
GET    /api/v1/svccenter/agent/dashboard                 Permission: sc.agent.read
POST   /api/v1/svccenter/agent/next-item                 Permission: sc.agent.dequeue
  Response: Next queued item assigned to agent
```

### 3.2 Queue Management
```
GET    /api/v1/svccenter/queues                          Permission: sc.queues.read
POST   /api/v1/svccenter/queues                          Permission: sc.queues.create
PUT    /api/v1/svccenter/queues/{id}                     Permission: sc.queues.update
GET    /api/v1/svccenter/queues/{id}/items                Permission: sc.queues.read
GET    /api/v1/svccenter/queues/{id}/stats                Permission: sc.queues.read
  Response: { "data": { "waiting": 12, "avg_wait_seconds": 45, "agents_available": 3 } }
POST   /api/v1/svccenter/queues/{id}/enqueue              Permission: sc.queues.enqueue
POST   /api/v1/svccenter/queues/{id}/dequeue              Permission: sc.queues.dequeue
```

### 3.3 Interactions
```
GET    /api/v1/svccenter/interactions                     Permission: sc.interactions.read
GET    /api/v1/svccenter/interactions/{id}                Permission: sc.interactions.read
POST   /api/v1/svccenter/interactions                     Permission: sc.interactions.create
PUT    /api/v1/svccenter/interactions/{id}                Permission: sc.interactions.update
POST   /api/v1/svccenter/interactions/{id}/transfer       Permission: sc.interactions.transfer
POST   /api/v1/svccenter/interactions/{id}/wrap-up        Permission: sc.interactions.wrapup
```

### 3.4 Supervisor
```
GET    /api/v1/svccenter/supervisor/agents                Permission: sc.supervisor.read
GET    /api/v1/svccenter/supervisor/queues                Permission: sc.supervisor.read
GET    /api/v1/svccenter/supervisor/realtime              Permission: sc.supervisor.read
  Response: Live dashboard data - agent states, queue depths, SLA metrics
POST   /api/v1/svccenter/supervisor/action                Permission: sc.supervisor.action
  Request: { "action_type": "WHISPER", "interaction_id": "...", "message": "..." }
POST   /api/v1/svccenter/supervisor/reassign              Permission: sc.supervisor.action
GET    /api/v1/svccenter/supervisor/alerts                Permission: sc.supervisor.read
```

### 3.5 Workspaces
```
GET    /api/v1/svccenter/workspaces                       Permission: sc.workspaces.read
POST   /api/v1/svccenter/workspaces                       Permission: sc.workspaces.create
PUT    /api/v1/svccenter/workspaces/{id}                  Permission: sc.workspaces.update
GET    /api/v1/svccenter/workspaces/{id}/agents           Permission: sc.workspaces.read
```

---

## 4. Business Rules

### 4.1 Routing Engine
- Skill-based routing matches interaction requirements to agent skills
- Priority routing: VIP customers and urgent cases ahead of normal queue
- Least busy: agent with fewest active cases gets next item
- Weighted: balances workload across agents considering proficiency
- Overflow: items exceeding max_wait_seconds route to overflow queue

### 4.2 Agent Capacity
- Agent cannot exceed max_case_capacity concurrent cases
- Auto-accept pulls next item immediately when agent becomes available
- Wrap-up time: configurable post-interaction period before next assignment
- Break/away status prevents new assignments

### 4.3 Omnichannel Unification
- Customer interactions across channels linked to single case
- Channel switching preserves full context (e.g., chat → phone → email)
- Agent sees full interaction history regardless of channel
- Response channel can differ from incoming channel

### 4.4 Supervisor Controls
- Listen: silently monitor agent-customer interaction
- Whisper: speak to agent without customer hearing
- Barge-in: join the interaction as active participant
- Transfer: move interaction to another agent or queue
- Real-time alerts for SLA breaches, long handle times, negative sentiment

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.svccenter.v1;

service ServiceCenterService {
    rpc EnqueueItem(EnqueueItemRequest) returns (EnqueueItemResponse);
    rpc DequeueItem(DequeueItemRequest) returns (DequeueItemResponse);
    rpc UpdateAgentStatus(UpdateAgentStatusRequest) returns (UpdateAgentStatusResponse);
    rpc GetRealtimeDashboard(GetRealtimeDashboardRequest) returns (GetRealtimeDashboardResponse);
    rpc SupervisorAction(SupervisorActionRequest) returns (SupervisorActionResponse);
    rpc StreamAgentEvents(StreamAgentEventsRequest) returns (stream AgentEvent);
}

// Entity messages
message ScWorkspace {
    string id = 1;
    string tenant_id = 2;
    string workspace_name = 3;
    string workspace_type = 4;
    string description = 5;
    string layout_config = 6;
    string default_channels = 7;
    string skill_requirements = 8;
    int32 max_concurrent_cases = 9;
    bool auto_accept = 10;
    string status = 11;
    string created_at = 12;
    string updated_at = 13;
}

message ScAgentProfile {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string display_name = 4;
    string workspace_id = 5;
    string skills = 6;
    string proficiency_levels = 7;
    string language_skills = 8;
    string status = 9;
    string status_since = 10;
    int32 current_case_count = 11;
    int32 max_case_capacity = 12;
    int32 average_handle_time_seconds = 13;
    string channels_active = 14;
    string created_at = 15;
    string updated_at = 16;
}

message ScQueue {
    string id = 1;
    string tenant_id = 2;
    string queue_name = 3;
    string queue_type = 4;
    string description = 5;
    string routing_strategy = 6;
    int32 priority = 7;
    int32 max_wait_seconds = 8;
    string overflow_queue_id = 9;
    int32 sla_response_seconds = 10;
    int32 sla_resolution_seconds = 11;
    string skill_requirements = 12;
    bool is_active = 13;
    string created_at = 14;
    string updated_at = 15;
}

message ScQueueItem {
    string id = 1;
    string tenant_id = 2;
    string queue_id = 3;
    string item_type = 4;
    string reference_id = 5;
    string customer_id = 6;
    string customer_priority = 7;
    string skill_requirements = 8;
    string enqueued_at = 9;
    string assigned_at = 10;
    string assigned_agent_id = 11;
    int32 wait_time_seconds = 12;
    string status = 13;
    double priority_score = 14;
    int32 retry_count = 15;
    string created_at = 16;
    string updated_at = 17;
}

message ScInteraction {
    string id = 1;
    string tenant_id = 2;
    string case_id = 3;
    string customer_id = 4;
    string agent_id = 5;
    string channel = 6;
    string direction = 7;
    string subject = 8;
    string content = 9;
    string content_type = 10;
    string attachments = 11;
    string status = 12;
    string priority = 13;
    string handling_agent_id = 14;
    string start_time = 15;
    string end_time = 16;
    int32 handle_time_seconds = 17;
    string wrap_up_notes = 18;
    string wrap_up_codes = 19;
    double sentiment_score = 20;
    string tags = 21;
    string created_at = 22;
    string updated_at = 23;
}

message AgentEvent {
    string event_id = 1;
    string tenant_id = 2;
    string agent_id = 3;
    string event_type = 4;
    string timestamp = 5;
    string payload = 6;
}

// Request/Response messages
message EnqueueItemRequest {
    string tenant_id = 1;
    string queue_id = 2;
    string item_type = 3;
    string reference_id = 4;
    string customer_id = 5;
    string customer_priority = 6;
    string skill_requirements = 7;
}

message EnqueueItemResponse {
    ScQueueItem data = 1;
    int32 estimated_wait_seconds = 2;
    int32 queue_position = 3;
}

message DequeueItemRequest {
    string tenant_id = 1;
    string agent_id = 2;
    string queue_id = 3;
}

message DequeueItemResponse {
    ScQueueItem data = 1;
    ScInteraction interaction = 2;
}

message UpdateAgentStatusRequest {
    string tenant_id = 1;
    string agent_id = 2;
    string status = 3;
    string channels_active = 4;
}

message UpdateAgentStatusResponse {
    ScAgentProfile data = 1;
}

message GetRealtimeDashboardRequest {
    string tenant_id = 1;
    string workspace_id = 2;
    repeated string queue_ids = 3;
}

message QueueStats {
    string queue_id = 1;
    string queue_name = 2;
    int32 waiting_count = 3;
    int32 avg_wait_seconds = 4;
    int32 agents_available = 5;
    int32 agents_busy = 6;
    int32 sla_breach_count = 7;
}

message AgentState {
    string agent_id = 1;
    string display_name = 2;
    string status = 3;
    int32 active_cases = 4;
    int32 avg_handle_time_seconds = 5;
    string current_channel = 6;
}

message GetRealtimeDashboardResponse {
    repeated QueueStats queue_stats = 1;
    repeated AgentState agent_states = 2;
    int32 total_waiting = 3;
    int32 total_agents_available = 4;
    double overall_sla_pct = 5;
}

message SupervisorActionRequest {
    string tenant_id = 1;
    string supervisor_id = 2;
    string action_type = 3;
    string interaction_id = 4;
    string agent_id = 5;
    string queue_item_id = 6;
    string notes = 7;
}

message SupervisorActionResponse {
    bool success = 1;
    string message = 2;
    ScInteraction interaction = 3;
}

message StreamAgentEventsRequest {
    string tenant_id = 1;
    string agent_id = 2;
    string event_types = 3;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **customerservice-service**: Core case data model
- **b2cservice-service**: B2C request handling
- **knowledge-service**: Knowledge base integration in agent console
- **cdp-service**: Customer 360 view, interaction history
- **ai-service**: Sentiment analysis, skill inference, auto-classification
- **workflow-service**: Escalation workflows

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `sc.agent.status.changed` | Agent status updated | agent_id, old_status, new_status |
| `sc.item.enqueued` | Item added to queue | queue_id, item_id, wait_time_est |
| `sc.item.assigned` | Item assigned to agent | item_id, agent_id |
| `sc.interaction.started` | Interaction begun | interaction_id, channel, agent_id |
| `sc.interaction.ended` | Interaction concluded | interaction_id, handle_time_seconds |
| `sc.sla.breach.warning` | SLA approaching breach | queue_id, item_id, seconds_remaining |
| `sc.supervisor.action` | Supervisor intervention | supervisor_id, action_type, target |

---

## 7. Migrations

### Migration Order for svccenter-service:
1. V001: `sc_workspaces`
2. V002: `sc_agent_profiles`
3. V003: `sc_queues`
4. V004: `sc_queue_items`
5. V005: `sc_interactions`
6. V006: `sc_supervisor_actions`
7. V007: Triggers for `updated_at`
8. V008: Seed data (default workspace, sample queues)
