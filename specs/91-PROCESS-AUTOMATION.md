# 91 - Process Automation Service Specification

## 1. Domain Overview

Process Automation provides RPA (Robotic Process Automation) and intelligent automation capabilities for automating repetitive business tasks. Manages automation bot definitions with recorded or designed step sequences (click, type, read, decide), cron-based scheduling for unattended execution, real-time execution monitoring with detailed logs, process mining to discover automation opportunities from system event data, work queues for prioritized bot task assignment, exception handling rules with configurable recovery strategies, execution analytics and dashboards, and a secure credential vault for bot authentication. Integrates with Integration Cloud for system connectivity and AI Service for intelligent document processing and decision support.

**Bounded Context:** RPA and Intelligent Process Automation
**Service Name:** `automation-service`
**Database:** `data/automation.db`
**HTTP Port:** 8123 | **gRPC Port:** 9123

---

## 2. Database Schema

### 2.1 Automation Bots
```sql
CREATE TABLE automation_bots (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bot_name TEXT NOT NULL,
    bot_code TEXT NOT NULL,
    description TEXT,
    bot_type TEXT NOT NULL DEFAULT 'ATTENDED'
        CHECK(bot_type IN ('ATTENDED','UNATTENDED','HYBRID')),
    category TEXT NOT NULL DEFAULT 'GENERAL'
        CHECK(category IN ('GENERAL','FINANCE','HR','PROCUREMENT','OPERATIONS','IT','CUSTOM')),
    target_system TEXT NOT NULL,
    target_application TEXT,                         -- Specific application name
    execution_mode TEXT NOT NULL DEFAULT 'SEQUENTIAL'
        CHECK(execution_mode IN ('SEQUENTIAL','PARALLEL','PIPELINE')),
    max_concurrent_instances INTEGER NOT NULL DEFAULT 1,
    timeout_ms INTEGER NOT NULL DEFAULT 300000,
    retry_on_failure INTEGER NOT NULL DEFAULT 0,
    max_retries INTEGER NOT NULL DEFAULT 3,
    credential_vault_id TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','READY','ACTIVE','PAUSED','DEPRECATED','ARCHIVED')),
    total_runs INTEGER NOT NULL DEFAULT 0,
    successful_runs INTEGER NOT NULL DEFAULT 0,
    failed_runs INTEGER NOT NULL DEFAULT 0,
    avg_duration_ms INTEGER NOT NULL DEFAULT 0,
    last_run_at TEXT,
    tags TEXT,                                       -- JSON array of tags
    metadata TEXT,                                   -- JSON: extensible bot metadata

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, bot_code)
);

CREATE INDEX idx_bots_tenant_status ON automation_bots(tenant_id, status);
CREATE INDEX idx_bots_tenant_type ON automation_bots(tenant_id, bot_type);
CREATE INDEX idx_bots_tenant_category ON automation_bots(tenant_id, category);
CREATE INDEX idx_bots_tenant_active ON automation_bots(tenant_id, is_active);
```

### 2.2 Bot Steps
```sql
CREATE TABLE bot_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bot_id TEXT NOT NULL,
    step_name TEXT NOT NULL,
    step_type TEXT NOT NULL
        CHECK(step_type IN ('CLICK','TYPE','READ','DECIDE','WAIT','NAVIGATE','EXTRACT','LOOP','SCRIPT','API_CALL','SCREENSHOT','COMPARE','DRAG_DROP','SCROLL','CUSTOM')),
    step_order INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    config TEXT NOT NULL,                             -- JSON: step-specific configuration
    selector TEXT,                                   -- UI element selector (CSS/XPath)
    target_value TEXT,                               -- Value to type, assert, or compare
    timeout_ms INTEGER NOT NULL DEFAULT 10000,
    continue_on_error INTEGER NOT NULL DEFAULT 0,
    condition_expression TEXT,                       -- Expression for DECIDE step
    on_success_step_id TEXT,                         -- Next step on success
    on_failure_step_id TEXT,                         -- Next step on failure
    loop_variable TEXT,                              -- Variable name for LOOP iteration
    loop_collection TEXT,                            -- JSON path or expression for collection
    screenshot_on_failure INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (bot_id) REFERENCES automation_bots(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, bot_id, step_order)
);

CREATE INDEX idx_steps_tenant_bot ON bot_steps(tenant_id, bot_id);
CREATE INDEX idx_steps_tenant_type ON bot_steps(tenant_id, step_type);
CREATE INDEX idx_steps_tenant_order ON bot_steps(tenant_id, bot_id, step_order);
CREATE INDEX idx_steps_tenant_active ON bot_steps(tenant_id, is_active);
```

### 2.3 Automation Schedules
```sql
CREATE TABLE automation_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bot_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    cron_expression TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active INTEGER NOT NULL DEFAULT 1,
    start_date TEXT,
    end_date TEXT,
    priority INTEGER NOT NULL DEFAULT 5,
    max_concurrent_runs INTEGER NOT NULL DEFAULT 1,
    misfire_policy TEXT NOT NULL DEFAULT 'FIRE_NOW'
        CHECK(misfire_policy IN ('FIRE_NOW','SKIP','FIRE_ALL','QUEUE')),
    input_parameters TEXT,                           -- JSON: bot input parameters
    queue_id TEXT,                                   -- Optional: pull items from queue
    last_triggered_at TEXT,
    next_trigger_at TEXT,
    total_executions INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (bot_id) REFERENCES automation_bots(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, bot_id, schedule_name)
);

CREATE INDEX idx_schedules_tenant_bot ON automation_schedules(tenant_id, bot_id);
CREATE INDEX idx_schedules_tenant_active ON automation_schedules(tenant_id, is_active);
CREATE INDEX idx_schedules_tenant_next ON automation_schedules(tenant_id, next_trigger_at);
CREATE INDEX idx_schedules_tenant_priority ON automation_schedules(tenant_id, priority);
```

### 2.4 Bot Executions
```sql
CREATE TABLE bot_executions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bot_id TEXT NOT NULL,
    schedule_id TEXT,
    queue_item_id TEXT,
    execution_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(execution_type IN ('MANUAL','SCHEDULED','API','TRIGGERED','QUEUED')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','CANCELLED','TIMED_OUT','RETRYING')),
    priority INTEGER NOT NULL DEFAULT 5,
    input_parameters TEXT,                           -- JSON: execution input values
    output_data TEXT,                                -- JSON: execution output values
    error_code TEXT,
    error_message TEXT,
    error_screenshot_url TEXT,
    current_step_id TEXT,
    current_step_order INTEGER NOT NULL DEFAULT 0,
    total_steps INTEGER NOT NULL DEFAULT 0,
    completed_steps INTEGER NOT NULL DEFAULT 0,
    started_at TEXT,
    completed_at TEXT,
    duration_ms INTEGER NOT NULL DEFAULT 0,
    triggered_by TEXT NOT NULL,
    worker_id TEXT,                                  -- Execution worker/machine identifier
    retry_count INTEGER NOT NULL DEFAULT 0,
    correlation_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (bot_id) REFERENCES automation_bots(id),
    FOREIGN KEY (schedule_id) REFERENCES automation_schedules(id)
);

CREATE INDEX idx_executions_tenant_bot ON bot_executions(tenant_id, bot_id);
CREATE INDEX idx_executions_tenant_status ON bot_executions(tenant_id, status);
CREATE INDEX idx_executions_tenant_type ON bot_executions(tenant_id, execution_type);
CREATE INDEX idx_executions_tenant_dates ON bot_executions(tenant_id, started_at, completed_at);
CREATE INDEX idx_executions_tenant_correlation ON bot_executions(tenant_id, correlation_id);
CREATE INDEX idx_executions_tenant_worker ON bot_executions(tenant_id, worker_id);
```

### 2.5 Execution Logs
```sql
CREATE TABLE execution_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    execution_id TEXT NOT NULL,
    step_id TEXT,
    step_name TEXT,
    step_type TEXT,
    log_level TEXT NOT NULL DEFAULT 'INFO'
        CHECK(log_level IN ('TRACE','DEBUG','INFO','WARN','ERROR','FATAL')),
    message TEXT NOT NULL,
    screenshot_url TEXT,
    element_snapshot TEXT,                           -- JSON: captured UI element state
    data_before TEXT,                                -- JSON: data before step execution
    data_after TEXT,                                 -- JSON: data after step execution
    duration_ms INTEGER NOT NULL DEFAULT 0,
    is_successful INTEGER NOT NULL DEFAULT 1,
    error_details TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (execution_id) REFERENCES bot_executions(id) ON DELETE CASCADE
);

CREATE INDEX idx_execlogs_tenant_execution ON execution_logs(tenant_id, execution_id);
CREATE INDEX idx_execlogs_tenant_step ON execution_logs(tenant_id, step_id);
CREATE INDEX idx_execlogs_tenant_level ON execution_logs(tenant_id, log_level);
CREATE INDEX idx_execlogs_tenant_dates ON execution_logs(tenant_id, created_at);
```

### 2.6 Process Mining Results
```sql
CREATE TABLE process_mining_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    analysis_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    process_name TEXT NOT NULL,
    analysis_type TEXT NOT NULL DEFAULT 'DISCOVERY'
        CHECK(analysis_type IN ('DISCOVERY','CONFORMANCE','ENHANCEMENT','BOTTLENECK','VARIANT')),
    date_from TEXT NOT NULL,
    date_to TEXT NOT NULL,
    total_cases INTEGER NOT NULL DEFAULT 0,
    total_events INTEGER NOT NULL DEFAULT 0,
    unique_activities INTEGER NOT NULL DEFAULT 0,
    variants_found INTEGER NOT NULL DEFAULT 0,
    avg_case_duration_ms INTEGER NOT NULL DEFAULT 0,
    median_case_duration_ms INTEGER NOT NULL DEFAULT 0,
    bottleneck_activities TEXT,                      -- JSON: activities with highest wait times
    process_graph TEXT,                              -- JSON: discovered process flow graph
    variant_distribution TEXT,                       -- JSON: case counts per variant
    automation_candidates TEXT,                      -- JSON: ranked automation opportunities
    conformance_rate_real REAL NOT NULL DEFAULT 0.0,
    analysis_config TEXT,                            -- JSON: analysis parameters
    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED')),
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, analysis_name)
);

CREATE INDEX idx_mining_tenant_status ON process_mining_results(tenant_id, status);
CREATE INDEX idx_mining_tenant_system ON process_mining_results(tenant_id, source_system);
CREATE INDEX idx_mining_tenant_process ON process_mining_results(tenant_id, process_name);
CREATE INDEX idx_mining_tenant_dates ON process_mining_results(tenant_id, date_from, date_to);
```

### 2.7 Automation Queues
```sql
CREATE TABLE automation_queues (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    queue_name TEXT NOT NULL,
    queue_code TEXT NOT NULL,
    description TEXT,
    bot_id TEXT,                                     -- Bot assigned to process this queue
    priority INTEGER NOT NULL DEFAULT 5,
    max_retries INTEGER NOT NULL DEFAULT 3,
    retry_delay_ms INTEGER NOT NULL DEFAULT 60000,
    item_ttl_ms INTEGER NOT NULL DEFAULT 86400000,
    deadlock_timeout_ms INTEGER NOT NULL DEFAULT 3600000,
    total_items INTEGER NOT NULL DEFAULT 0,
    pending_items INTEGER NOT NULL DEFAULT 0,
    in_progress_items INTEGER NOT NULL DEFAULT 0,
    completed_items INTEGER NOT NULL DEFAULT 0,
    failed_items INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (bot_id) REFERENCES automation_bots(id),
    UNIQUE(tenant_id, queue_code)
);

CREATE INDEX idx_queues_tenant_active ON automation_queues(tenant_id, is_active);
CREATE INDEX idx_queues_tenant_bot ON automation_queues(tenant_id, bot_id);
```

### 2.8 Queue Items
```sql
CREATE TABLE automation_queue_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    queue_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','FAILED','DEADLOCKED','RETRYING','CANCELLED')),
    priority INTEGER NOT NULL DEFAULT 5,
    payload TEXT NOT NULL,                           -- JSON: item data for bot processing
    result TEXT,                                     -- JSON: processing result
    error_message TEXT,
    execution_id TEXT,
    assigned_worker TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    max_retries INTEGER NOT NULL DEFAULT 3,
    due_date TEXT,
    deadline TEXT,
    started_at TEXT,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (queue_id) REFERENCES automation_queues(id) ON DELETE CASCADE
);

CREATE INDEX idx_queueitems_tenant_queue ON automation_queue_items(tenant_id, queue_id);
CREATE INDEX idx_queueitems_tenant_status ON automation_queue_items(tenant_id, status);
CREATE INDEX idx_queueitems_tenant_priority ON automation_queue_items(tenant_id, priority);
CREATE INDEX idx_queueitems_tenant_due ON automation_queue_items(tenant_id, due_date);
```

### 2.9 Exception Handling Rules
```sql
CREATE TABLE exception_handling_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    description TEXT,
    applies_to TEXT NOT NULL DEFAULT 'ALL'
        CHECK(applies_to IN ('ALL','BOT','CATEGORY','SYSTEM')),
    target_id TEXT,                                  -- Bot ID or category when applies_to is specific
    exception_type TEXT NOT NULL
        CHECK(exception_type IN ('ELEMENT_NOT_FOUND','TIMEOUT','AUTH_FAILURE','DATA_ERROR','NETWORK_ERROR','APPLICATION_ERROR','CUSTOM')),
    error_pattern TEXT,                              -- Regex to match error messages
    action TEXT NOT NULL DEFAULT 'RETRY'
        CHECK(action IN ('RETRY','SKIP','STOP','ESCALATE','CUSTOM_SCRIPT','SWITCH_BOT')),
    max_retries INTEGER NOT NULL DEFAULT 3,
    retry_delay_ms INTEGER NOT NULL DEFAULT 10000,
    retry_backoff_multiplier REAL NOT NULL DEFAULT 1.0,
    escalation_channel TEXT,                         -- Notification channel for escalate action
    custom_script TEXT,                              -- Script for CUSTOM_SCRIPT action
    fallback_bot_id TEXT,                            -- Bot to invoke for SWITCH_BOT action
    is_active INTEGER NOT NULL DEFAULT 1,
    priority INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_exceptions_tenant_type ON exception_handling_rules(tenant_id, exception_type);
CREATE INDEX idx_exceptions_tenant_applies ON exception_handling_rules(tenant_id, applies_to);
CREATE INDEX idx_exceptions_tenant_active ON exception_handling_rules(tenant_id, is_active);
CREATE INDEX idx_exceptions_tenant_priority ON exception_handling_rules(tenant_id, priority);
```

### 2.10 Automation Analytics
```sql
CREATE TABLE automation_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,
    bot_id TEXT,
    category TEXT,
    total_executions INTEGER NOT NULL DEFAULT 0,
    successful_executions INTEGER NOT NULL DEFAULT 0,
    failed_executions INTEGER NOT NULL DEFAULT 0,
    avg_duration_ms INTEGER NOT NULL DEFAULT 0,
    total_duration_ms INTEGER NOT NULL DEFAULT 0,
    time_saved_ms INTEGER NOT NULL DEFAULT 0,
    cost_saved_cents INTEGER NOT NULL DEFAULT 0,
    manual_effort_hours_real REAL NOT NULL DEFAULT 0.0,
    automated_effort_hours_real REAL NOT NULL DEFAULT 0.0,
    roi_pct REAL NOT NULL DEFAULT 0.0,
    error_rate_pct REAL NOT NULL DEFAULT 0.0,
    throughput_per_hour REAL NOT NULL DEFAULT 0.0,
    queue_backlog INTEGER NOT NULL DEFAULT 0,
    queue_avg_wait_ms INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, metric_date, bot_id)
);

CREATE INDEX idx_analytics_tenant_date ON automation_analytics(tenant_id, metric_date);
CREATE INDEX idx_analytics_tenant_bot ON automation_analytics(tenant_id, bot_id);
CREATE INDEX idx_analytics_tenant_category ON automation_analytics(tenant_id, category);
```

### 2.11 Credential Vaults
```sql
CREATE TABLE credential_vaults (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    vault_name TEXT NOT NULL,
    vault_code TEXT NOT NULL,
    description TEXT,
    credential_type TEXT NOT NULL
        CHECK(credential_type IN ('USERNAME_PASSWORD','API_KEY','OAUTH2_TOKEN','CERTIFICATE','WINDOWS_CREDENTIAL','CUSTOM')),
    credentials_encrypted TEXT NOT NULL,              -- JSON: encrypted credential blob
    encryption_key_id TEXT NOT NULL,
    target_system TEXT NOT NULL,
    target_application TEXT,
    rotation_enabled INTEGER NOT NULL DEFAULT 0,
    rotation_interval_days INTEGER NOT NULL DEFAULT 90,
    last_rotated_at TEXT,
    expires_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, vault_code)
);

CREATE INDEX idx_vaults_tenant_type ON credential_vaults(tenant_id, credential_type);
CREATE INDEX idx_vaults_tenant_active ON credential_vaults(tenant_id, is_active);
CREATE INDEX idx_vaults_tenant_expiry ON credential_vaults(tenant_id, expires_at);
```

---

## 3. REST API Endpoints

```
# Bots
GET/POST      /api/v1/automation/bots                       Permission: automation.bots.read/create
GET/PUT       /api/v1/automation/bots/{id}                  Permission: automation.bots.read/update
DELETE        /api/v1/automation/bots/{id}                  Permission: automation.bots.delete
PATCH         /api/v1/automation/bots/{id}/status           Permission: automation.bots.update
GET           /api/v1/automation/bots/{id}/statistics       Permission: automation.bots.read

# Bot Steps
GET/POST      /api/v1/automation/bots/{id}/steps            Permission: automation.steps.read/create
GET/PUT       /api/v1/automation/steps/{id}                 Permission: automation.steps.read/update
DELETE        /api/v1/automation/steps/{id}                 Permission: automation.steps.delete
PATCH         /api/v1/automation/steps/{id}/reorder         Permission: automation.steps.update
POST          /api/v1/automation/bots/{id}/steps/record     Permission: automation.steps.create

# Schedules
GET/POST      /api/v1/automation/schedules                  Permission: automation.schedules.read/create
GET/PUT       /api/v1/automation/schedules/{id}             Permission: automation.schedules.read/update
DELETE        /api/v1/automation/schedules/{id}             Permission: automation.schedules.delete
PATCH         /api/v1/automation/schedules/{id}/toggle      Permission: automation.schedules.update

# Executions
POST          /api/v1/automation/bots/{id}/execute          Permission: automation.execute.run
GET           /api/v1/automation/executions                 Permission: automation.executions.read
GET           /api/v1/automation/executions/{id}            Permission: automation.executions.read
POST          /api/v1/automation/executions/{id}/cancel     Permission: automation.executions.update
POST          /api/v1/automation/executions/{id}/retry      Permission: automation.execute.run
GET           /api/v1/automation/bots/{id}/executions       Permission: automation.executions.read

# Execution Logs
GET           /api/v1/automation/executions/{id}/logs       Permission: automation.logs.read
GET           /api/v1/automation/executions/{id}/steps      Permission: automation.logs.read
GET           /api/v1/automation/executions/{id}/screenshots Permission: automation.logs.read

# Process Mining
POST          /api/v1/automation/process-mining/analyze     Permission: automation.mining.execute
GET           /api/v1/automation/process-mining/results     Permission: automation.mining.read
GET           /api/v1/automation/process-mining/results/{id} Permission: automation.mining.read
GET           /api/v1/automation/process-mining/candidates  Permission: automation.mining.read

# Queues
GET/POST      /api/v1/automation/queues                     Permission: automation.queues.read/create
GET/PUT       /api/v1/automation/queues/{id}                Permission: automation.queues.read/update
DELETE        /api/v1/automation/queues/{id}                Permission: automation.queues.delete
GET           /api/v1/automation/queues/{id}/items          Permission: automation.queues.read
POST          /api/v1/automation/queues/{id}/items          Permission: automation.queues.create
PATCH         /api/v1/automation/queue-items/{id}/priority  Permission: automation.queues.update
POST          /api/v1/automation/queue-items/{id}/cancel    Permission: automation.queues.update

# Exception Handling
GET/POST      /api/v1/automation/exception-rules            Permission: automation.exceptions.read/create
GET/PUT       /api/v1/automation/exception-rules/{id}       Permission: automation.exceptions.read/update
DELETE        /api/v1/automation/exception-rules/{id}       Permission: automation.exceptions.delete

# Analytics
GET           /api/v1/automation/analytics/summary          Permission: automation.analytics.read
GET           /api/v1/automation/analytics/trends            Permission: automation.analytics.read
GET           /api/v1/automation/analytics/roi               Permission: automation.analytics.read
GET           /api/v1/automation/analytics/queue-metrics     Permission: automation.analytics.read

# Credential Vaults
GET/POST      /api/v1/automation/vaults                     Permission: automation.vaults.read/create
GET/PUT       /api/v1/automation/vaults/{id}                Permission: automation.vaults.read/update
DELETE        /api/v1/automation/vaults/{id}                Permission: automation.vaults.delete
POST          /api/v1/automation/vaults/{id}/rotate         Permission: automation.vaults.execute
POST          /api/v1/automation/vaults/{id}/test           Permission: automation.vaults.test
```

---

## 4. Business Rules

### 4.1 Bot Management
1. Each bot MUST have a unique code within a tenant
2. Bots in DRAFT status MAY be modified freely; ACTIVE bots MUST be paused before modification
3. Bot deletion MUST soft-delete and check for active executions before removal
4. Each bot MUST have at least one step before it can be moved to READY status
5. Bots referencing a credential vault MUST validate the credential is active before execution
6. Bot type determines execution context: attended bots require a user session; unattended bots run on designated workers
7. Max concurrent instances MUST be enforced; excess execution requests MUST be queued

### 4.2 Step Configuration
1. Step ordering MUST be sequential unless the step type is DECIDE with branching paths
2. DECIDE steps MUST define a condition_expression that evaluates to true/false for branching
3. LOOP steps MUST define a termination condition to prevent infinite loops
4. Steps with continue_on_error set to 0 MUST halt the bot execution on failure
5. Screenshot on failure MUST be captured when screenshot_on_failure is enabled and the step fails
6. Selectors for CLICK, TYPE, and READ steps MUST be validated against the target application
7. Custom SCRIPT steps MUST execute in a sandboxed environment with resource and time limits

### 4.3 Scheduling
1. Cron expressions MUST be validated before saving
2. Scheduled executions MUST NOT exceed max_concurrent_runs for the associated bot
3. Schedules with end dates in the past MUST be automatically deactivated
4. Misfire policy determines behavior when a trigger is missed: fire immediately, skip, fire all missed, or queue
5. Input parameters from the schedule MUST be merged with default bot parameters at execution time
6. Schedule priority MUST influence execution queue ordering when workers are contested

### 4.4 Execution Management
1. Each execution MUST generate a unique execution_id and optional correlation_id for tracing
2. Executions in RUNNING status MUST track current_step_id and current_step_order
3. Timed-out executions MUST be marked as TIMED_OUT and trigger exception handling
4. Execution cancellation MUST be graceful: complete the current step, then stop
5. Retry logic MUST respect the configured max_retries and exponential backoff
6. Execution output data MUST be captured in the output_data JSON field upon completion

### 4.5 Process Mining
1. Process mining analysis MUST accept system event logs as input data
2. Discovery analysis MUST identify all process variants and their frequency
3. Bottleneck analysis MUST rank activities by average wait time
4. Automation candidate ranking MUST consider volume, repetitiveness, and rule-based nature
5. Conformance analysis MUST compare actual executions against the expected process model
6. Analysis results MUST be retained for historical comparison and trend analysis

### 4.6 Queue Management
1. Queue items MUST be processed in priority order (higher priority first)
2. Items in IN_PROGRESS status for longer than deadlock_timeout_ms MUST be marked as DEADLOCKED
3. Failed items MUST be retried up to max_retries with configurable delay
4. Items exceeding their TTL MUST be marked as expired and removed from active processing
5. Queue statistics (pending, in_progress, completed, failed) MUST be updated atomically
6. Queue assignment to a bot MUST allow only one bot to consume from the queue unless explicitly configured

### 4.7 Exception Handling
1. Exception rules MUST be evaluated in priority order when an error occurs
2. Rules with more specific applies_to targets MUST take precedence over ALL rules
3. RETRY action MUST respect the configured retry_delay_ms and backoff_multiplier
4. ESCALATE action MUST send a notification through the configured escalation channel
5. SWITCH_BOT action MUST invoke the fallback_bot_id with the same input parameters
6. Custom scripts for exception handling MUST run in a sandboxed environment

### 4.8 Credential Security
1. Credentials MUST be stored encrypted using the specified encryption_key_id
2. Credential access MUST be logged for audit purposes
3. Expired credentials MUST prevent bot execution and trigger an alert
4. Credential rotation MUST update the encrypted blob without requiring bot reconfiguration
5. Vaults with rotation_enabled MUST automatically rotate at the configured interval
6. Credential values MUST NEVER appear in execution logs or API responses

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.automation.v1;

service AutomationService {
    rpc GetBot(GetBotRequest) returns (GetBotResponse);
    rpc ExecuteBot(ExecuteBotRequest) returns (ExecuteBotResponse);
    rpc GetExecutionStatus(GetExecutionStatusRequest) returns (GetExecutionStatusResponse);
    rpc CancelExecution(CancelExecutionRequest) returns (CancelExecutionResponse);
    rpc GetExecutionLogs(GetExecutionLogsRequest) returns (GetExecutionLogsResponse);
    rpc EnqueueItem(EnqueueItemRequest) returns (EnqueueItemResponse);
    rpc GetQueueStatus(GetQueueStatusRequest) returns (GetQueueStatusResponse);
    rpc RunProcessMining(RunProcessMiningRequest) returns (RunProcessMiningResponse);
    rpc GetAnalytics(GetAnalyticsRequest) returns (GetAnalyticsResponse);
}

// --- Automation Bot ---
message AutomationBot {
    string id = 1; string tenant_id = 2; string bot_name = 3; string bot_code = 4;
    string description = 5; string bot_type = 6; string category = 7; string target_system = 8;
    string target_application = 9; string execution_mode = 10; int32 max_concurrent_instances = 11;
    int32 timeout_ms = 12; bool retry_on_failure = 13; int32 max_retries = 14;
    string credential_vault_id = 15; string status = 16; int32 total_runs = 17;
    int32 successful_runs = 18; int32 failed_runs = 19; int32 avg_duration_ms = 20;
    string last_run_at = 21; string tags = 22; string metadata = 23;
    string created_at = 24; string updated_at = 25; string created_by = 26; string updated_by = 27;
    int32 version = 28; bool is_active = 29;
}

// --- Bot Step ---
message BotStep {
    string id = 1; string tenant_id = 2; string bot_id = 3; string step_name = 4;
    string step_type = 5; int32 step_order = 6; string description = 7; string config = 8;
    string selector = 9; string target_value = 10; int32 timeout_ms = 11;
    bool continue_on_error = 12; string condition_expression = 13; string on_success_step_id = 14;
    string on_failure_step_id = 15; string loop_variable = 16; string loop_collection = 17;
    bool screenshot_on_failure = 18; bool is_active = 19;
    string created_at = 20; string updated_at = 21; string created_by = 22; string updated_by = 23;
    int32 version = 24;
}

// --- Automation Schedule ---
message AutomationSchedule {
    string id = 1; string tenant_id = 2; string bot_id = 3; string schedule_name = 4;
    string cron_expression = 5; string timezone = 6; bool is_active = 7;
    string start_date = 8; string end_date = 9; int32 priority = 10;
    int32 max_concurrent_runs = 11; string misfire_policy = 12; string input_parameters = 13;
    string queue_id = 14; string last_triggered_at = 15; string next_trigger_at = 16;
    int32 total_executions = 17;
    string created_at = 18; string updated_at = 19; string created_by = 20; string updated_by = 21;
    int32 version = 22;
}

// --- Bot Execution ---
message BotExecution {
    string id = 1; string tenant_id = 2; string bot_id = 3; string schedule_id = 4;
    string queue_item_id = 5; string execution_type = 6; string status = 7;
    int32 priority = 8; string input_parameters = 9; string output_data = 10;
    string error_code = 11; string error_message = 12; string error_screenshot_url = 13;
    string current_step_id = 14; int32 current_step_order = 15; int32 total_steps = 16;
    int32 completed_steps = 17; string started_at = 18; string completed_at = 19;
    int32 duration_ms = 20; string triggered_by = 21; string worker_id = 22;
    int32 retry_count = 23; string correlation_id = 24;
    string created_at = 25; string updated_at = 26; string created_by = 27; string updated_by = 28;
}

// --- Execution Log ---
message ExecutionLog {
    string id = 1; string tenant_id = 2; string execution_id = 3; string step_id = 4;
    string step_name = 5; string step_type = 6; string log_level = 7; string message = 8;
    string screenshot_url = 9; string element_snapshot = 10; string data_before = 11;
    string data_after = 12; int32 duration_ms = 13; bool is_successful = 14;
    string error_details = 15;
    string created_at = 16; string updated_at = 17; string created_by = 18; string updated_by = 19;
}

// --- Process Mining Result ---
message ProcessMiningResult {
    string id = 1; string tenant_id = 2; string analysis_name = 3; string source_system = 4;
    string process_name = 5; string analysis_type = 6; string date_from = 7; string date_to = 8;
    int32 total_cases = 9; int32 total_events = 10; int32 unique_activities = 11;
    int32 variants_found = 12; int32 avg_case_duration_ms = 13; int32 median_case_duration_ms = 14;
    string bottleneck_activities = 15; string process_graph = 16; string variant_distribution = 17;
    string automation_candidates = 18; double conformance_rate_real = 19;
    string analysis_config = 20; string status = 21; string completed_at = 22;
    string created_at = 23; string updated_at = 24; string created_by = 25; string updated_by = 26;
    int32 version = 27;
}

// --- Automation Queue ---
message AutomationQueue {
    string id = 1; string tenant_id = 2; string queue_name = 3; string queue_code = 4;
    string description = 5; string bot_id = 6; int32 priority = 7; int32 max_retries = 8;
    int32 retry_delay_ms = 9; int32 item_ttl_ms = 10; int32 deadlock_timeout_ms = 11;
    int32 total_items = 12; int32 pending_items = 13; int32 in_progress_items = 14;
    int32 completed_items = 15; int32 failed_items = 16; bool is_active = 17;
    string created_at = 18; string updated_at = 19; string created_by = 20; string updated_by = 21;
    int32 version = 22;
}

// --- Queue Item ---
message AutomationQueueItem {
    string id = 1; string tenant_id = 2; string queue_id = 3; string status = 4;
    int32 priority = 5; string payload = 6; string result = 7; string error_message = 8;
    string execution_id = 9; string assigned_worker = 10; int32 retry_count = 11;
    int32 max_retries = 12; string due_date = 13; string deadline = 14;
    string started_at = 15; string completed_at = 16;
    string created_at = 17; string updated_at = 18; string created_by = 19; string updated_by = 20;
}

// --- Exception Handling Rule ---
message ExceptionHandlingRule {
    string id = 1; string tenant_id = 2; string rule_name = 3; string rule_code = 4;
    string description = 5; string applies_to = 6; string target_id = 7;
    string exception_type = 8; string error_pattern = 9; string action = 10;
    int32 max_retries = 11; int32 retry_delay_ms = 12; double retry_backoff_multiplier = 13;
    string escalation_channel = 14; string custom_script = 15; string fallback_bot_id = 16;
    bool is_active = 17; int32 priority = 18;
    string created_at = 19; string updated_at = 20; string created_by = 21; string updated_by = 22;
    int32 version = 23;
}

// --- Automation Analytics ---
message AutomationAnalytics {
    string id = 1; string tenant_id = 2; string metric_date = 3; string bot_id = 4;
    string category = 5; int32 total_executions = 6; int32 successful_executions = 7;
    int32 failed_executions = 8; int32 avg_duration_ms = 9; int32 total_duration_ms = 10;
    int32 time_saved_ms = 11; int64 cost_saved_cents = 12;
    double manual_effort_hours_real = 13; double automated_effort_hours_real = 14;
    double roi_pct = 15; double error_rate_pct = 16; double throughput_per_hour = 17;
    int32 queue_backlog = 18; int32 queue_avg_wait_ms = 19;
    string created_at = 20; string updated_at = 21; string created_by = 22; string updated_by = 23;
}

// --- Credential Vault ---
message CredentialVault {
    string id = 1; string tenant_id = 2; string vault_name = 3; string vault_code = 4;
    string description = 5; string credential_type = 6; string credentials_encrypted = 7;
    string encryption_key_id = 8; string target_system = 9; string target_application = 10;
    bool rotation_enabled = 11; int32 rotation_interval_days = 12; string last_rotated_at = 13;
    string expires_at = 14; bool is_active = 15;
    string created_at = 16; string updated_at = 17; string created_by = 18; string updated_by = 19;
    int32 version = 20;
}

// --- RPC Request/Response Messages ---
message GetBotRequest { string tenant_id = 1; string id = 2; }
message GetBotResponse { AutomationBot data = 1; repeated BotStep steps = 2; }

message ExecuteBotRequest { string tenant_id = 1; string bot_id = 2; string input_parameters = 3; string correlation_id = 4; }
message ExecuteBotResponse { BotExecution execution = 1; }

message GetExecutionStatusRequest { string tenant_id = 1; string execution_id = 2; }
message GetExecutionStatusResponse { BotExecution execution = 1; }

message CancelExecutionRequest { string tenant_id = 1; string execution_id = 2; }
message CancelExecutionResponse { BotExecution execution = 1; }

message GetExecutionLogsRequest { string tenant_id = 1; string execution_id = 2; int32 page_size = 3; string page_token = 4; }
message GetExecutionLogsResponse { repeated ExecutionLog items = 1; int32 total_count = 2; string next_page_token = 3; }

message EnqueueItemRequest { string tenant_id = 1; string queue_id = 2; string payload = 3; int32 priority = 4; string due_date = 5; }
message EnqueueItemResponse { AutomationQueueItem item = 1; }

message GetQueueStatusRequest { string tenant_id = 1; string queue_id = 2; }
message GetQueueStatusResponse { AutomationQueue queue = 1; }

message RunProcessMiningRequest { string tenant_id = 1; string analysis_name = 2; string source_system = 3; string process_name = 4; string analysis_type = 5; string date_from = 6; string date_to = 7; }
message RunProcessMiningResponse { ProcessMiningResult result = 1; }

message GetAnalyticsRequest { string tenant_id = 1; string metric_date = 2; string bot_id = 3; }
message GetAnalyticsResponse { repeated AutomationAnalytics metrics = 1; }

message ListBotsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListBotsResponse { repeated AutomationBot items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListQueuesRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListQueuesResponse { repeated AutomationQueue items = 1; int32 total_count = 2; string next_page_token = 3; }
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Integration Cloud:** Connection configurations, adapter definitions, flow execution results for bot target systems
- **AI Service:** Intelligent document processing results, OCR data, classification results for decision steps
- **Auth:** User identity for bot ownership, execution permissions, credential vault access
- **Workflow:** Process definitions for coordination with human-in-the-loop workflows
- **Notification:** Alert delivery for execution failures, escalations, schedule triggers

### 6.2 Data Published To
- **Integration Cloud:** Bot execution requests for system interactions, API calls via integration flows
- **AI Service:** Document images and data for intelligent processing, classification requests
- **Workflow:** Human task creation for attended bot steps, exception escalation workflows
- **Reporting:** Execution metrics, ROI data, queue analytics, process mining results
- **Notification:** Execution completion alerts, failure notifications, credential expiry warnings

### 6.3 Integration Matrix

| Source Service | Integration Type | Data Exchanged |
|----------------|------------------|----------------|
| Integration Cloud | API call | System connections, adapter execution, flow triggering |
| AI Service | gRPC call | Document processing, OCR, classification, prediction |
| Auth | API call | User authentication, permission checks, credential vault access |
| Workflow | Event-driven | Task creation, approval routing, escalation handling |
| Notification | Event-driven | Alert routing, escalation delivery, summary reports |
| Reporting | Data push | Execution metrics, analytics aggregates, ROI calculations |
| GL | Via Integration Cloud | Journal entry automation, reconciliation data |
| AP | Via Integration Cloud | Invoice processing, payment execution |
| AR | Via Integration Cloud | Receipt processing, collection automation |
| HCM (Core-HR) | Via Integration Cloud | Employee onboarding, data entry automation |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `automation.bot.created` | `{ bot_id, bot_name, bot_type, category, created_by, timestamp }` | New bot definition created |
| `automation.bot.scheduled` | `{ bot_id, schedule_id, cron_expression, next_trigger_at, timestamp }` | Bot schedule configured or updated |
| `automation.execution.started` | `{ execution_id, bot_id, execution_type, triggered_by, timestamp }` | Bot execution initiated |
| `automation.step.completed` | `{ execution_id, step_id, step_name, step_type, duration_ms, is_successful, timestamp }` | Individual bot step completed |
| `automation.execution.completed` | `{ execution_id, bot_id, duration_ms, total_steps, completed_steps, timestamp }` | Bot execution finished successfully |
| `automation.execution.failed` | `{ execution_id, bot_id, step_id, error_code, error_message, retry_count, timestamp }` | Bot execution failed |
| `automation.exception.raised` | `{ execution_id, exception_type, rule_id, action_taken, timestamp }` | Exception detected and handling rule applied |
| `automation.queue.item.completed` | `{ queue_id, item_id, execution_id, duration_ms, timestamp }` | Queue item processed successfully |
| `automation.queue.deadlock` | `{ queue_id, item_id, assigned_worker, deadlock_duration_ms, timestamp }` | Queue item detected as deadlocked |
| `automation.credential.expiring` | `{ vault_id, vault_name, expires_at, days_remaining, timestamp }` | Credential approaching expiration |
| `automation.mining.completed` | `{ analysis_id, process_name, variants_found, automation_candidates_count, timestamp }` | Process mining analysis finished |
| `automation.analytics.updated` | `{ metric_date, total_executions, success_rate_pct, time_saved_ms, timestamp }` | Daily analytics aggregated |

---

## 8. Migrations

1. V001: `automation_bots`
2. V002: `bot_steps`
3. V003: `automation_schedules`
4. V004: `bot_executions`
5. V005: `execution_logs`
6. V006: `process_mining_results`
7. V007: `automation_queues`
8. V008: `automation_queue_items`
9. V009: `exception_handling_rules`
10. V010: `automation_analytics`
11. V011: `credential_vaults`
12. V012: Triggers for `updated_at`
