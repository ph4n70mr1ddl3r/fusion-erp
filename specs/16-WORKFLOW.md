# 16 - Workflow & Approval Engine Specification

## 1. Domain Overview

The Workflow service manages configurable approval flows, business rule routing, notifications, and delegation. All services delegate approval decisions to this centralized engine.

**Bounded Context:** Workflow, Approval & Notification Management
**Service Name:** `workflow-service`
**Database:** `data/workflow.db`
**HTTP Port:** 8040 | **gRPC Port:** 9040

---

## 2. Database Schema

### 2.1 Workflow Definitions
```sql
CREATE TABLE workflow_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    workflow_name TEXT NOT NULL,
    description TEXT,
    document_type TEXT NOT NULL,           -- 'JOURNAL_ENTRY', 'AP_INVOICE', 'AR_INVOICE', 'PURCHASE_ORDER', 'SALES_ORDER', 'EXPENSE_REPORT', 'TIME_SHEET'
    is_active INTEGER NOT NULL DEFAULT 1,
    priority INTEGER NOT NULL DEFAULT 5,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, workflow_name)
);

CREATE INDEX idx_workflow_defs_tenant_doc ON workflow_definitions(tenant_id, document_type);
```

### 2.2 Workflow Steps
```sql
CREATE TABLE workflow_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    workflow_definition_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    step_name TEXT NOT NULL,
    step_type TEXT NOT NULL
        CHECK(step_type IN ('APPROVAL','REVIEW','NOTIFICATION','CONDITION','PARALLEL_SPLIT','PARALLEL_JOIN')),
    description TEXT,

    -- Approver determination
    approver_type TEXT NOT NULL
        CHECK(approver_type IN ('FIXED_USER','ROLE_BASED','AMOUNT_BASED','HIERARCHY','SELF')),
    approver_user_id TEXT,                 -- For FIXED_USER
    approver_role TEXT,                    -- For ROLE_BASED

    -- Amount-based rules
    amount_threshold_cents INTEGER,        -- Trigger rule if amount >= threshold
    amount_field TEXT,                     -- Which field to check: "total_cents", "amount_cents"

    -- Auto-approve rules
    auto_approve_below_cents INTEGER,      -- Auto-approve if amount below threshold

    -- Timeout
    timeout_hours INTEGER DEFAULT 72,
    timeout_action TEXT DEFAULT 'ESCALATE'
        CHECK(timeout_action IN ('ESCALATE','AUTO_APPROVE','AUTO_REJECT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (workflow_definition_id) REFERENCES workflow_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, workflow_definition_id, step_number)
);
```

### 2.3 Workflow Instances (Runtime)
```sql
CREATE TABLE workflow_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    workflow_definition_id TEXT NOT NULL,
    document_type TEXT NOT NULL,
    document_id TEXT NOT NULL,
    document_number TEXT,
    initiator_id TEXT NOT NULL,
    current_step_number INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','CANCELLED','ESCALATED','ERROR')),
    submitted_at TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (workflow_definition_id) REFERENCES workflow_definitions(id) ON DELETE RESTRICT
);

CREATE INDEX idx_workflow_instances_tenant_doc ON workflow_instances(tenant_id, document_type, document_id);
CREATE INDEX idx_workflow_instances_tenant_status ON workflow_instances(tenant_id, status);
CREATE INDEX idx_workflow_instances_tenant_initiator ON workflow_instances(tenant_id, initiator_id);
```

### 2.4 Approval Actions
```sql
CREATE TABLE approval_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    workflow_instance_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    approver_id TEXT NOT NULL,
    action TEXT NOT NULL
        CHECK(action IN ('APPROVED','REJECTED','RETURNED','DELEGATED','ESCALATED','COMMENTED')),
    comments TEXT,
    action_date TEXT NOT NULL DEFAULT (datetime('now')),

    -- Delegation
    delegated_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (workflow_instance_id) REFERENCES workflow_instances(id) ON DELETE CASCADE
);

CREATE INDEX idx_approval_actions_tenant_instance ON approval_actions(tenant_id, workflow_instance_id);
CREATE INDEX idx_approval_actions_tenant_approver ON approval_actions(tenant_id, approver_id);
```

### 2.5 Delegation Rules
```sql
CREATE TABLE delegation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    delegator_id TEXT NOT NULL,
    delegate_id TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    reason TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, delegator_id, start_date, end_date)
);
```

### 2.6 Notifications
```sql
CREATE TABLE notifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    notification_type TEXT NOT NULL
        CHECK(notification_type IN ('APPROVAL_REQUIRED','APPROVED','REJECTED','ESCALATED','REMINDER','SYSTEM')),
    title TEXT NOT NULL,
    message TEXT NOT NULL,
    document_type TEXT,
    document_id TEXT,
    action_url TEXT,
    is_read INTEGER NOT NULL DEFAULT 0,
    read_at TEXT,
    priority TEXT DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_notifications_tenant_user ON notifications(tenant_id, user_id, is_read);
CREATE INDEX idx_notifications_tenant_created ON notifications(tenant_id, created_at);
```

---

## 3. REST API Endpoints

```
# Workflow Definitions
GET/POST      /api/v1/workflow/definitions            Permission: workflow.definitions.read/create
GET/PUT       /api/v1/workflow/definitions/{id}        Permission: workflow.definitions.read/update
GET/POST      /api/v1/workflow/definitions/{id}/steps  Permission: workflow.definitions.read/create

# Workflow Instances
POST          /api/v1/workflow/submit                  Permission: workflow.submit
GET           /api/v1/workflow/pending                  Permission: workflow.pending.read
GET           /api/v1/workflow/history/{doc_type}/{id}  Permission: workflow.history.read
GET           /api/v1/workflow/instances/{id}           Permission: workflow.instances.read

# Approval Actions
POST          /api/v1/workflow/approve/{instance_id}    Permission: workflow.approve
POST          /api/v1/workflow/reject/{instance_id}     Permission: workflow.approve
POST          /api/v1/workflow/return/{instance_id}     Permission: workflow.approve
POST          /api/v1/workflow/delegate/{instance_id}   Permission: workflow.delegate

# Delegation
GET/POST      /api/v1/workflow/delegations             Permission: workflow.delegations.read/create
DELETE        /api/v1/workflow/delegations/{id}         Permission: workflow.delegations.delete

# Notifications
GET           /api/v1/workflow/notifications           Permission: workflow.notifications.read
PATCH         /api/v1/workflow/notifications/{id}/read  Permission: workflow.notifications.update
POST          /api/v1/workflow/notifications/read-all   Permission: workflow.notifications.update
GET           /api/v1/workflow/notifications/unread-count Permission: workflow.notifications.read
```

---

## 4. Business Rules

### 4.1 Approval Routing Logic
1. Service submits document for approval via gRPC: `SubmitApproval(doc_type, doc_id, amount, initiator_id)`
2. Workflow engine looks up active workflow definition for `doc_type`
3. Evaluates step rules:
   - **FIXED_USER:** Route to specified user
   - **ROLE_BASED:** Route to anyone with the specified role
   - **AMOUNT_BASED:** Compare document amount field against thresholds
   - **HIERARCHY:** Route to initiator's manager
4. Create workflow_instance and notification(s)
5. On approval action, advance to next step or complete

### 4.2 Amount-Based Rules
```json
{
  "rules": [
    { "below": 100000, "approver_type": "SELF", "auto_approve": true },
    { "from": 100000, "to": 1000000, "approver_role": "FinanceManager" },
    { "above": 1000000, "approver_role": "TenantAdmin", "multi_step": true }
  ]
}
```

### 4.3 Parallel Approvals
- PARALLEL_SPLIT creates multiple approval tasks simultaneously
- PARALLEL_JOIN waits for all parallel tasks to complete
- Any rejection in parallel flow cancels the entire branch

### 4.4 Timeout Handling
- Background task checks for timed-out approvals periodically
- ESCALATE: route to next-level approver
- AUTO_APPROVE: automatically approve
- AUTO_REJECT: automatically reject

### 4.5 gRPC Service (called by other services)
```protobuf
service WorkflowService {
    rpc SubmitApproval(SubmitApprovalRequest) returns (SubmitApprovalResponse);
    rpc GetApprovalStatus(GetApprovalStatusRequest) returns (GetApprovalStatusResponse);
    rpc CancelApproval(CancelApprovalRequest) returns (CancelApprovalResponse);
}
```

```protobuf
message SubmitApprovalRequest {
    string tenant_id = 1;
    string document_type = 2;
    string document_id = 3;
    string document_number = 4;
    string initiator_id = 5;
    int64 amount_cents = 6;
    string amount_field = 7;
}

message SubmitApprovalResponse {
    WorkflowInstance instance = 1;
}

message GetApprovalStatusRequest {
    string tenant_id = 1;
    string document_type = 2;
    string document_id = 3;
}

message GetApprovalStatusResponse {
    WorkflowInstance instance = 1;
    repeated ApprovalAction actions = 2;
}

message CancelApprovalRequest {
    string tenant_id = 1;
    string instance_id = 2;
    string cancelled_by = 3;
    string reason = 4;
}

message CancelApprovalResponse {
    bool success = 1;
    string instance_id = 2;
}

message WorkflowDefinition {
    string id = 1;
    string tenant_id = 2;
    string workflow_name = 3;
    string description = 4;
    string document_type = 5;
    int32 is_active = 6;
    int32 priority = 7;
    string created_at = 8;
    string updated_at = 9;
    string created_by = 10;
    string updated_by = 11;
    int32 version = 12;
}

message WorkflowStep {
    string id = 1;
    string tenant_id = 2;
    string workflow_definition_id = 3;
    int32 step_number = 4;
    string step_name = 5;
    string step_type = 6;
    string description = 7;
    string approver_type = 8;
    string approver_user_id = 9;
    string approver_role = 10;
    int64 amount_threshold_cents = 11;
    string amount_field = 12;
    int64 auto_approve_below_cents = 13;
    int32 timeout_hours = 14;
    string timeout_action = 15;
    string created_at = 16;
    string updated_at = 17;
}

message WorkflowInstance {
    string id = 1;
    string tenant_id = 2;
    string workflow_definition_id = 3;
    string document_type = 4;
    string document_id = 5;
    string document_number = 6;
    string initiator_id = 7;
    int32 current_step_number = 8;
    string status = 9;
    string submitted_at = 10;
    string completed_at = 11;
    string created_at = 12;
    string updated_at = 13;
}

message ApprovalAction {
    string id = 1;
    string tenant_id = 2;
    string workflow_instance_id = 3;
    int32 step_number = 4;
    string approver_id = 5;
    string action = 6;
    string comments = 7;
    string action_date = 8;
    string delegated_to = 9;
    string created_at = 10;
}

message DelegationRule {
    string id = 1;
    string tenant_id = 2;
    string delegator_id = 3;
    string delegate_id = 4;
    string start_date = 5;
    string end_date = 6;
    string reason = 7;
    int32 is_active = 8;
    string created_at = 9;
}

message Notification {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string notification_type = 4;
    string title = 5;
    string message = 6;
    string document_type = 7;
    string document_id = 8;
    string action_url = 9;
    int32 is_read = 10;
    string read_at = 11;
    string priority = 12;
    string created_at = 13;
}
```

### 4.6 Events Published
| Event | Trigger |
|-------|---------|
| `workflow.approval_required` | New approval needed |
| `workflow.approved` | Document fully approved |
| `workflow.rejected` | Document rejected |
| `workflow.escalated` | Approval escalated |
| `workflow.notification.created` | New notification |
