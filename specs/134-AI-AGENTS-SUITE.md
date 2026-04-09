# 134 - AI Agents Suite Specification

## 1. Domain Overview

AI Agents Suite provides a collection of pre-built, cross-application AI agents that automate complex business processes across ERP, SCM, HCM, and CX domains. These agents combine large language models with business context to perform tasks like invoice matching, demand forecasting explanation, expense auditing, candidate screening, and customer resolution — going beyond analytics to take autonomous actions with human oversight.

**Bounded Context:** Cross-Application AI Agents & Autonomous Workflows
**Service Name:** `aiagents-service`
**Database:** `data/aiagents.db`
**HTTP Port:** 8154 | **gRPC Port:** 9154

---

## 2. Database Schema

### 2.1 Agent Definitions
```sql
CREATE TABLE aa_agent_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agent_name TEXT NOT NULL,
    agent_code TEXT NOT NULL,
    agent_type TEXT NOT NULL CHECK(agent_type IN ('ERP','SCM','HCM','CX','PLATFORM','CUSTOM')),
    description TEXT NOT NULL,
    capabilities TEXT NOT NULL,          -- JSON array: ["invoice_matching", "anomaly_detection"]
    domain_context TEXT NOT NULL,        -- JSON: domain-specific knowledge and rules
    system_prompt TEXT NOT NULL,         -- LLM system prompt for this agent
    model_config TEXT NOT NULL,          -- JSON: { "model": "...", "temperature": 0.2, "max_tokens": 4096 }
    tool_definitions TEXT NOT NULL,      -- JSON: tools the agent can use (API calls)
    autonomy_level TEXT NOT NULL DEFAULT 'ASSIST'
        CHECK(autonomy_level IN ('ASSIST','SUGGEST','ACT_WITH_APPROVAL','AUTONOMOUS')),
    max_actions_per_run INTEGER NOT NULL DEFAULT 10,
    timeout_seconds INTEGER NOT NULL DEFAULT 300,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, agent_code)
);
```

### 2.2 Agent Runs
```sql
CREATE TABLE aa_agent_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    run_type TEXT NOT NULL CHECK(run_type IN ('SCHEDULED','TRIGGERED','MANUAL','EVENT_DRIVEN')),
    trigger_event TEXT,                  -- Event that triggered this run
    trigger_data TEXT,                   -- JSON: event payload
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','AWAITING_APPROVAL','COMPLETED','FAILED','CANCELLED')),
    started_at TEXT,
    completed_at TEXT,
    total_tokens_used INTEGER NOT NULL DEFAULT 0,
    total_actions INTEGER NOT NULL DEFAULT 0,
    approved_actions INTEGER NOT NULL DEFAULT 0,
    rejected_actions INTEGER NOT NULL DEFAULT 0,
    autonomous_actions INTEGER NOT NULL DEFAULT 0,
    summary TEXT,                        -- LLM-generated run summary
    error_message TEXT,
    execution_cost_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (agent_id) REFERENCES aa_agent_definitions(id) ON DELETE RESTRICT
);

CREATE INDEX idx_aa_runs_tenant_agent ON aa_agent_runs(tenant_id, agent_id, created_at DESC);
CREATE INDEX idx_aa_runs_tenant_status ON aa_agent_runs(tenant_id, status);
```

### 2.3 Agent Actions
```sql
CREATE TABLE aa_agent_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_id TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN ('API_CALL','DATA_QUERY','NOTIFICATION','UPDATE_RECORD','CREATE_RECORD','SEND_EMAIL','APPROVE','REJECT','ESCALATE','GENERATE_REPORT')),
    target_service TEXT NOT NULL,        -- Service to call
    target_endpoint TEXT NOT NULL,       -- API endpoint or gRPC method
    action_payload TEXT NOT NULL,        -- JSON: request body
    action_reasoning TEXT NOT NULL,      -- LLM explanation of why this action
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','EXECUTED','FAILED','SKIPPED')),
    approved_by TEXT,
    approved_at TEXT,
    rejection_reason TEXT,
    execution_result TEXT,               -- JSON: response from target service
    execution_time_ms INTEGER,
    sequence_number INTEGER NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (run_id) REFERENCES aa_agent_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_aa_actions_tenant_run ON aa_agent_actions(tenant_id, run_id, sequence_number);
CREATE INDEX idx_aa_actions_tenant_status ON aa_agent_actions(tenant_id, status);
```

### 2.4 Agent Memory
```sql
CREATE TABLE aa_agent_memory (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    memory_type TEXT NOT NULL CHECK(memory_type IN ('CONTEXT','LEARNING','PREFERENCE','FEEDBACK','PATTERN')),
    key TEXT NOT NULL,
    value TEXT NOT NULL,
    source_run_id TEXT,
    relevance_score REAL NOT NULL DEFAULT 1.0,
    access_count INTEGER NOT NULL DEFAULT 0,
    last_accessed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, agent_id, memory_type, key)
);

CREATE INDEX idx_aa_memory_tenant_agent ON aa_agent_memory(tenant_id, agent_id, memory_type);
```

### 2.5 Agent Schedules
```sql
CREATE TABLE aa_agent_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    cron_expression TEXT NOT NULL,
    run_parameters TEXT,                 -- JSON: parameters to pass on each run
    is_active INTEGER NOT NULL DEFAULT 1,
    last_run_at TEXT,
    next_run_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (agent_id) REFERENCES aa_agent_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, agent_id, schedule_name)
);
```

### 2.6 Human Approvals
```sql
CREATE TABLE aa_approval_queue (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_id TEXT NOT NULL,
    action_id TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    approval_type TEXT NOT NULL CHECK(approval_type IN ('SINGLE_ACTION','BATCH','DELEGATE')),
    assigned_approver_id TEXT NOT NULL,
    summary TEXT NOT NULL,               -- Human-readable action summary
    risk_level TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','EXPIRED','ESCALATED')),
    decided_by TEXT,
    decided_at TEXT,
    decision_notes TEXT,
    expires_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (action_id) REFERENCES aa_agent_actions(id) ON DELETE CASCADE
);

CREATE INDEX idx_aa_approvals_tenant_approver ON aa_approval_queue(tenant_id, assigned_approver_id, status);
CREATE INDEX idx_aa_approvals_tenant_expires ON aa_approval_queue(tenant_id, expires_at, status);
```

---

## 3. REST API Endpoints

### 3.1 Agent Management
```
GET    /api/v1/aiagents/agents                          Permission: aa.agents.read
GET    /api/v1/aiagents/agents/{id}                     Permission: aa.agents.read
POST   /api/v1/aiagents/agents                          Permission: aa.agents.create
PUT    /api/v1/aiagents/agents/{id}                     Permission: aa.agents.update
POST   /api/v1/aiagents/agents/{id}/activate            Permission: aa.agents.activate
POST   /api/v1/aiagents/agents/{id}/pause               Permission: aa.agents.pause
GET    /api/v1/aiagents/agents/{id}/memory               Permission: aa.agents.read
```

### 3.2 Agent Execution
```
POST   /api/v1/aiagents/run                             Permission: aa.run.execute
  Request: { "agent_code": "INVOICE_MATCHER", "trigger_data": { "invoice_id": "..." } }
  Response 201: { "data": { "run_id": "...", "status": "RUNNING" } }

GET    /api/v1/aiagents/runs                            Permission: aa.run.read
GET    /api/v1/aiagents/runs/{id}                       Permission: aa.run.read
GET    /api/v1/aiagents/runs/{id}/actions               Permission: aa.run.read
POST   /api/v1/aiagents/runs/{id}/cancel                Permission: aa.run.cancel
GET    /api/v1/aiagents/runs/{id}/trace                  Permission: aa.run.read
  Response: Full execution trace with reasoning chain
```

### 3.3 Approval Queue
```
GET    /api/v1/aiagents/approvals                       Permission: aa.approvals.read
GET    /api/v1/aiagents/approvals/{id}                  Permission: aa.approvals.read
POST   /api/v1/aiagents/approvals/{id}/approve          Permission: aa.approvals.decide
POST   /api/v1/aiagents/approvals/{id}/reject           Permission: aa.approvals.decide
POST   /api/v1/aiagents/approvals/batch-approve         Permission: aa.approvals.decide
GET    /api/v1/aiagents/approvals/pending               Permission: aa.approvals.read
```

### 3.4 Schedules
```
GET    /api/v1/aiagents/schedules                       Permission: aa.schedules.read
POST   /api/v1/aiagents/schedules                       Permission: aa.schedules.create
PUT    /api/v1/aiagents/schedules/{id}                  Permission: aa.schedules.update
DELETE /api/v1/aiagents/schedules/{id}                  Permission: aa.schedules.delete
```

### 3.5 Agent Dashboard
```
GET    /api/v1/aiagents/dashboard                       Permission: aa.dashboard.read
  Response: { "data": { "active_agents": 5, "runs_today": 42, "pending_approvals": 3, "actions_taken": 156, "cost_today_cents": 2500 } }
GET    /api/v1/aiagents/agents/{id}/metrics             Permission: aa.dashboard.read
```

---

## 4. Business Rules

### 4.1 Autonomy Levels
- ASSIST: Agent provides information only, no actions
- SUGGEST: Agent recommends actions, human must initiate
- ACT_WITH_APPROVAL: Agent proposes and queues actions for human approval
- AUTONOMOUS: Agent executes actions directly (restricted to low-risk operations)
- Autonomy level per action type configurable by admin

### 4.2 Action Safety
- Actions classified by risk: LOW (read/query), MEDIUM (create/update), HIGH (delete/approve), CRITICAL (financial)
- HIGH and CRITICAL actions ALWAYS require human approval regardless of autonomy level
- Rate limiting: max N actions per run per agent
- Rollback capability: actions logged with before/after state for undo
- Cost guardrails: max LLM spend per day per agent

### 4.3 Approval Workflow
- Approvals assigned based on action type and affected domain
- Multi-level approval for CRITICAL actions
- Approval timeout: unapproved actions expire and auto-cancel
- Delegation: approvers can delegate to another user
- Batch approval for low-risk similar actions

### 4.4 Agent Memory
- Agents maintain contextual memory across runs
- Learning: successful action patterns reinforced
- Feedback: human approval/rejection trains future behavior
- Memory decays over time (configurable retention)
- Cross-agent memory sharing via shared memory namespace

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.aiagents.v1;

service AIAgentsService {
    rpc ExecuteAgent(ExecuteAgentRequest) returns (ExecuteAgentResponse);
    rpc GetRunStatus(GetRunStatusRequest) returns (GetRunStatusResponse);
    rpc ApproveAction(ApproveActionRequest) returns (ApproveActionResponse);
    rpc RejectAction(RejectActionRequest) returns (RejectActionResponse);
    rpc GetAgentMemory(GetAgentMemoryRequest) returns (GetAgentMemoryResponse);
    rpc TriggerAgent(TriggerAgentRequest) returns (TriggerAgentResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **ai-service**: LLM inference, embedding generation, model management
- **All services**: Agents can call any service API as tools
- **workflow-service**: Complex multi-step approval workflows
- **auth-service**: Permission checks for agent actions
- **dms-service**: Document access for document-processing agents
- **notification channels**: Email, Slack, SMS for approval requests

### 6.2 Pre-Built Agent Catalog

| Agent Code | Domain | Description |
|-----------|--------|-------------|
| `INVOICE_MATCHER` | ERP | Auto-match AP invoices to POs and receipts |
| `EXPENSE_AUDITOR` | ERP | Flag policy-violating expense reports |
| `CASH_FORECASTER` | ERP | Generate cash flow forecasts with narrative |
| `DEMAND_PLANNER` | SCM | Explain demand forecast anomalies |
| `PO_EXCEPTION` | SCM | Auto-resolve purchase order exceptions |
| `CANDIDATE_SCREENER` | HCM | Score and rank job applicants |
| `SCHEDULE_OPTIMIZER` | HCM | Suggest optimal shift assignments |
| `CASE_RESOLVER` | CX | Auto-suggest resolution for support cases |
| `SENTIMENT_WATCHER` | CX | Monitor and alert on sentiment trends |
| `COMPLIANCE_CHECKER` | Platform | Audit transactions for compliance violations |

### 6.3 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `aa.run.started` | Agent execution started | run_id, agent_id |
| `aa.run.completed` | Agent execution finished | run_id, actions_count, summary |
| `aa.run.failed` | Agent execution errored | run_id, error_message |
| `aa.action.pending_approval` | Action queued for approval | action_id, approver_id, risk_level |
| `aa.action.executed` | Action executed successfully | action_id, result_summary |
| `aa.action.rejected` | Action rejected by human | action_id, rejection_reason |
| `aa.approval.expired` | Approval request timed out | approval_id, action_id |

---

## 7. Migrations

### Migration Order for aiagents-service:
1. V001: `aa_agent_definitions`
2. V002: `aa_agent_runs`
3. V003: `aa_agent_actions`
4. V004: `aa_agent_memory`
5. V005: `aa_agent_schedules`
6. V006: `aa_approval_queue`
7. V007: Triggers for `updated_at`
8. V008: Seed data (pre-built agent definitions for all catalog agents)
