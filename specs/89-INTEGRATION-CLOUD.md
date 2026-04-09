# 89 - Integration Cloud Service Specification

## 1. Domain Overview

Integration Cloud provides a comprehensive iPaaS (Integration Platform as a Service) for connecting enterprise systems, orchestrating data flows, transforming messages, and managing API gateways. Manages registered system connections with credential vaulting, connection adapters supporting multiple protocols (REST, SOAP, FTP, Database, File), visual integration flow design with sequential steps (trigger, transform, route, deliver), data transformation mapping rules, cron-based scheduling, execution logging with full audit trails, failure alerting with escalation, API gateway routing with rate limiting and throttling, and connection pooling for high-throughput scenarios. Integrates with ALL services as the central integration backbone.

**Bounded Context:** Integration Platform as a Service (iPaaS)
**Service Name:** `integration-service`
**Database:** `data/integration.db`
**HTTP Port:** 8121 | **gRPC Port:** 9121

---

## 2. Database Schema

### 2.1 Integration Connections
```sql
CREATE TABLE integration_connections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    connection_name TEXT NOT NULL,
    connection_code TEXT NOT NULL,
    description TEXT,
    system_type TEXT NOT NULL
        CHECK(system_type IN ('ERP','CRM','SCM','HCM','BI','EPM','DATABASE','FILE_SYSTEM','FTP','SFTP','EMAIL','CUSTOM')),
    endpoint_url TEXT,
    auth_type TEXT NOT NULL DEFAULT 'API_KEY'
        CHECK(auth_type IN ('NONE','BASIC','API_KEY','OAUTH2','JWT','CERTIFICATE','CUSTOM')),
    credentials_encrypted TEXT NOT NULL,              -- JSON: encrypted credential blob
    credential_vault_key TEXT NOT NULL,
    health_check_url TEXT,
    timeout_ms INTEGER NOT NULL DEFAULT 30000,
    retry_count INTEGER NOT NULL DEFAULT 3,
    retry_delay_ms INTEGER NOT NULL DEFAULT 1000,
    status TEXT NOT NULL DEFAULT 'CONFIGURED'
        CHECK(status IN ('CONFIGURED','TESTING','CONNECTED','DISCONNECTED','ERROR','DEPRECATED')),
    last_tested_at TEXT,
    last_error_message TEXT,
    metadata TEXT,                                   -- JSON: extensible connection metadata

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, connection_code)
);

CREATE INDEX idx_connections_tenant_status ON integration_connections(tenant_id, status);
CREATE INDEX idx_connections_tenant_type ON integration_connections(tenant_id, system_type);
CREATE INDEX idx_connections_tenant_active ON integration_connections(tenant_id, is_active);
CREATE INDEX idx_connections_tenant_auth ON integration_connections(tenant_id, auth_type);
```

### 2.2 Connection Adapters
```sql
CREATE TABLE connection_adapters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    adapter_name TEXT NOT NULL,
    adapter_type TEXT NOT NULL
        CHECK(adapter_type IN ('REST','SOAP','FTP','SFTP','DATABASE','FILE','MQTT','KAFKA','JMS','EMAIL','CUSTOM')),
    version TEXT NOT NULL DEFAULT '1.0',
    description TEXT,
    config_schema TEXT NOT NULL,                     -- JSON: adapter configuration schema
    default_config TEXT,                             -- JSON: default configuration values
    supported_auth_types TEXT NOT NULL,              -- JSON array of supported auth types
    capabilities TEXT NOT NULL,                      -- JSON: read, write, stream, batch, etc.
    is_builtin INTEGER NOT NULL DEFAULT 0,
    adapter_library_path TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, adapter_name, adapter_type)
);

CREATE INDEX idx_adapters_tenant_type ON connection_adapters(tenant_id, adapter_type);
CREATE INDEX idx_adapters_tenant_active ON connection_adapters(tenant_id, is_active);
CREATE INDEX idx_adapters_tenant_builtin ON connection_adapters(tenant_id, is_builtin);
```

### 2.3 Integration Flows
```sql
CREATE TABLE integration_flows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    flow_name TEXT NOT NULL,
    flow_code TEXT NOT NULL,
    description TEXT,
    flow_type TEXT NOT NULL DEFAULT 'REALTIME'
        CHECK(flow_type IN ('REALTIME','BATCH','SCHEDULED','EVENT_DRIVEN','MANUAL')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','TESTING','ACTIVE','PAUSED','DEPRECATED','ARCHIVED')),
    source_connection_id TEXT NOT NULL,
    target_connection_id TEXT NOT NULL,
    source_adapter_id TEXT,
    target_adapter_id TEXT,
    trigger_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(trigger_type IN ('MANUAL','SCHEDULED','EVENT','WEBHOOK','POLLING')),
    trigger_config TEXT,                             -- JSON: trigger configuration
    error_handling_strategy TEXT NOT NULL DEFAULT 'STOP'
        CHECK(error_handling_strategy IN ('STOP','SKIP','RETRY','DEAD_LETTER','CUSTOM')),
    max_retries INTEGER NOT NULL DEFAULT 3,
    retry_interval_ms INTEGER NOT NULL DEFAULT 5000,
    timeout_ms INTEGER NOT NULL DEFAULT 60000,
    batch_size INTEGER NOT NULL DEFAULT 100,
    parallelism INTEGER NOT NULL DEFAULT 1,
    tags TEXT,                                       -- JSON array of tags

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_connection_id) REFERENCES integration_connections(id),
    FOREIGN KEY (target_connection_id) REFERENCES integration_connections(id),
    UNIQUE(tenant_id, flow_code)
);

CREATE INDEX idx_flows_tenant_status ON integration_flows(tenant_id, status);
CREATE INDEX idx_flows_tenant_type ON integration_flows(tenant_id, flow_type);
CREATE INDEX idx_flows_tenant_source ON integration_flows(tenant_id, source_connection_id);
CREATE INDEX idx_flows_tenant_target ON integration_flows(tenant_id, target_connection_id);
CREATE INDEX idx_flows_tenant_active ON integration_flows(tenant_id, is_active);
```

### 2.4 Flow Steps
```sql
CREATE TABLE flow_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    flow_id TEXT NOT NULL,
    step_name TEXT NOT NULL,
    step_type TEXT NOT NULL
        CHECK(step_type IN ('TRIGGER','TRANSFORM','ROUTE','FILTER','ENRICH','DELIVER','VALIDATE','DELAY','CONDITION','LOOP','SCRIPT','SUBFLOW')),
    step_order INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    config TEXT NOT NULL,                             -- JSON: step-specific configuration
    input_schema TEXT,                               -- JSON: expected input schema
    output_schema TEXT,                              -- JSON: expected output schema
    error_step_id TEXT,                              -- Step to execute on error
    condition_expression TEXT,                       -- Expression for CONDITION step
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (flow_id) REFERENCES integration_flows(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, flow_id, step_order)
);

CREATE INDEX idx_steps_tenant_flow ON flow_steps(tenant_id, flow_id);
CREATE INDEX idx_steps_tenant_type ON flow_steps(tenant_id, step_type);
CREATE INDEX idx_steps_tenant_order ON flow_steps(tenant_id, flow_id, step_order);
CREATE INDEX idx_steps_tenant_active ON flow_steps(tenant_id, is_active);
```

### 2.5 Data Transformations
```sql
CREATE TABLE data_transformations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transformation_name TEXT NOT NULL,
    transformation_code TEXT NOT NULL,
    description TEXT,
    source_format TEXT NOT NULL DEFAULT 'JSON'
        CHECK(source_format IN ('JSON','XML','CSV','FIXED_WIDTH','CUSTOM')),
    target_format TEXT NOT NULL DEFAULT 'JSON'
        CHECK(target_format IN ('JSON','XML','CSV','FIXED_WIDTH','CUSTOM')),
    mapping_rules TEXT NOT NULL,                     -- JSON: field mapping definitions
    transformation_script TEXT,                      -- Custom transformation script
    source_schema TEXT,                              -- JSON: source data schema
    target_schema TEXT,                              -- JSON: target data schema
    lookup_tables TEXT,                              -- JSON: lookup reference data
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, transformation_code)
);

CREATE INDEX idx_transformations_tenant_active ON data_transformations(tenant_id, is_active);
CREATE INDEX idx_transformations_tenant_source ON data_transformations(tenant_id, source_format);
CREATE INDEX idx_transformations_tenant_target ON data_transformations(tenant_id, target_format);
```

### 2.6 Integration Schedules
```sql
CREATE TABLE integration_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    flow_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    cron_expression TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active INTEGER NOT NULL DEFAULT 1,
    start_date TEXT,
    end_date TEXT,
    max_concurrent_runs INTEGER NOT NULL DEFAULT 1,
    misfire_policy TEXT NOT NULL DEFAULT 'FIRE_NOW'
        CHECK(misfire_policy IN ('FIRE_NOW','SKIP','FIRE_ALL','QUEUE')),
    last_triggered_at TEXT,
    next_trigger_at TEXT,
    total_executions INTEGER NOT NULL DEFAULT 0,
    last_execution_status TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (flow_id) REFERENCES integration_flows(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, flow_id, schedule_name)
);

CREATE INDEX idx_schedules_tenant_flow ON integration_schedules(tenant_id, flow_id);
CREATE INDEX idx_schedules_tenant_active ON integration_schedules(tenant_id, is_active);
CREATE INDEX idx_schedules_tenant_next ON integration_schedules(tenant_id, next_trigger_at);
```

### 2.7 Integration Logs
```sql
CREATE TABLE integration_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    flow_id TEXT NOT NULL,
    schedule_id TEXT,
    execution_id TEXT NOT NULL,
    step_id TEXT,
    log_level TEXT NOT NULL DEFAULT 'INFO'
        CHECK(log_level IN ('TRACE','DEBUG','INFO','WARN','ERROR','FATAL')),
    message TEXT NOT NULL,
    payload_before TEXT,                             -- JSON: data before step
    payload_after TEXT,                              -- JSON: data after step
    error_code TEXT,
    error_message TEXT,
    error_stack_trace TEXT,
    records_processed INTEGER NOT NULL DEFAULT 0,
    records_succeeded INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    duration_ms INTEGER NOT NULL DEFAULT 0,
    source_ip TEXT,
    correlation_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (flow_id) REFERENCES integration_flows(id)
);

CREATE INDEX idx_logs_tenant_flow ON integration_logs(tenant_id, flow_id);
CREATE INDEX idx_logs_tenant_execution ON integration_logs(tenant_id, execution_id);
CREATE INDEX idx_logs_tenant_level ON integration_logs(tenant_id, log_level);
CREATE INDEX idx_logs_tenant_dates ON integration_logs(tenant_id, created_at);
CREATE INDEX idx_logs_tenant_step ON integration_logs(tenant_id, step_id);
CREATE INDEX idx_logs_tenant_correlation ON integration_logs(tenant_id, correlation_id);
```

### 2.8 Integration Alerts
```sql
CREATE TABLE integration_alerts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    flow_id TEXT,
    connection_id TEXT,
    execution_id TEXT,
    alert_type TEXT NOT NULL
        CHECK(alert_type IN ('FLOW_FAILED','CONNECTION_LOST','TIMEOUT','THRESHOLD_BREACH','AUTH_FAILURE','DATA_QUALITY','CUSTOM')),
    severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(severity IN ('INFO','WARNING','ERROR','CRITICAL')),
    title TEXT NOT NULL,
    message TEXT NOT NULL,
    alert_details TEXT,                              -- JSON: additional context
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','ACKNOWLEDGED','RESOLVED','ESCALATED','DISMISSED')),
    assigned_to TEXT,
    acknowledged_at TEXT,
    resolved_at TEXT,
    resolution_notes TEXT,
    notification_channels TEXT,                      -- JSON array: email, slack, webhook
    escalation_rules TEXT,                           -- JSON: escalation configuration

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (flow_id) REFERENCES integration_flows(id),
    FOREIGN KEY (connection_id) REFERENCES integration_connections(id)
);

CREATE INDEX idx_alerts_tenant_flow ON integration_alerts(tenant_id, flow_id);
CREATE INDEX idx_alerts_tenant_status ON integration_alerts(tenant_id, status);
CREATE INDEX idx_alerts_tenant_severity ON integration_alerts(tenant_id, severity);
CREATE INDEX idx_alerts_tenant_type ON integration_alerts(tenant_id, alert_type);
CREATE INDEX idx_alerts_tenant_dates ON integration_alerts(tenant_id, created_at);
CREATE INDEX idx_alerts_tenant_assigned ON integration_alerts(tenant_id, assigned_to);
```

### 2.9 API Gateway Routes
```sql
CREATE TABLE api_gateway_routes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    route_name TEXT NOT NULL,
    route_code TEXT NOT NULL,
    description TEXT,
    path_pattern TEXT NOT NULL,                      -- e.g., /api/v1/invoices/*
    http_methods TEXT NOT NULL,                      -- JSON array: GET, POST, PUT, DELETE
    target_connection_id TEXT NOT NULL,
    target_path TEXT NOT NULL,                       -- Upstream target path
    rate_limit_per_minute INTEGER NOT NULL DEFAULT 100,
    rate_limit_per_hour INTEGER NOT NULL DEFAULT 1000,
    burst_limit INTEGER NOT NULL DEFAULT 50,
    timeout_ms INTEGER NOT NULL DEFAULT 30000,
    requires_auth INTEGER NOT NULL DEFAULT 1,
    allowed_scopes TEXT,                             -- JSON array of OAuth scopes
    request_transform_id TEXT,
    response_transform_id TEXT,
    cache_ttl_seconds INTEGER NOT NULL DEFAULT 0,
    cors_enabled INTEGER NOT NULL DEFAULT 0,
    cors_origins TEXT,                               -- JSON array of allowed origins
    is_active INTEGER NOT NULL DEFAULT 1,
    priority INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (target_connection_id) REFERENCES integration_connections(id),
    UNIQUE(tenant_id, route_code)
);

CREATE INDEX idx_routes_tenant_path ON api_gateway_routes(tenant_id, path_pattern);
CREATE INDEX idx_routes_tenant_active ON api_gateway_routes(tenant_id, is_active);
CREATE INDEX idx_routes_tenant_connection ON api_gateway_routes(tenant_id, target_connection_id);
CREATE INDEX idx_routes_tenant_priority ON api_gateway_routes(tenant_id, priority);
```

### 2.10 Connection Pools
```sql
CREATE TABLE connection_pools (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    connection_id TEXT NOT NULL,
    pool_name TEXT NOT NULL,
    min_size INTEGER NOT NULL DEFAULT 1,
    max_size INTEGER NOT NULL DEFAULT 10,
    idle_timeout_ms INTEGER NOT NULL DEFAULT 300000,
    max_lifetime_ms INTEGER NOT NULL DEFAULT 1800000,
    acquire_timeout_ms INTEGER NOT NULL DEFAULT 30000,
    validation_query TEXT,
    current_active INTEGER NOT NULL DEFAULT 0,
    current_idle INTEGER NOT NULL DEFAULT 0,
    total_acquired INTEGER NOT NULL DEFAULT 0,
    total_released INTEGER NOT NULL DEFAULT 0,
    total_failed INTEGER NOT NULL DEFAULT 0,
    avg_acquire_time_ms INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (connection_id) REFERENCES integration_connections(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, connection_id, pool_name)
);

CREATE INDEX idx_pools_tenant_connection ON connection_pools(tenant_id, connection_id);
CREATE INDEX idx_pools_tenant_active ON connection_pools(tenant_id, is_active);
```

---

## 3. REST API Endpoints

```
# Connections
GET/POST      /api/v1/integration/connections                 Permission: integration.connections.read/create
GET/PUT       /api/v1/integration/connections/{id}            Permission: integration.connections.read/update
DELETE        /api/v1/integration/connections/{id}            Permission: integration.connections.delete
POST          /api/v1/integration/connections/{id}/test       Permission: integration.connections.test
GET           /api/v1/integration/connections/{id}/status     Permission: integration.connections.read

# Connection Adapters
GET/POST      /api/v1/integration/adapters                    Permission: integration.adapters.read/create
GET/PUT       /api/v1/integration/adapters/{id}               Permission: integration.adapters.read/update
DELETE        /api/v1/integration/adapters/{id}               Permission: integration.adapters.delete
GET           /api/v1/integration/adapters/{id}/schema        Permission: integration.adapters.read

# Integration Flows
GET/POST      /api/v1/integration/flows                       Permission: integration.flows.read/create
GET/PUT       /api/v1/integration/flows/{id}                  Permission: integration.flows.read/update
DELETE        /api/v1/integration/flows/{id}                  Permission: integration.flows.delete
POST          /api/v1/integration/flows/{id}/test             Permission: integration.flows.test
POST          /api/v1/integration/flows/{id}/activate         Permission: integration.flows.activate
POST          /api/v1/integration/flows/{id}/deactivate       Permission: integration.flows.deactivate
POST          /api/v1/integration/flows/{id}/execute          Permission: integration.flows.execute
GET           /api/v1/integration/flows/{id}/steps            Permission: integration.flows.read

# Flow Steps
GET/POST      /api/v1/integration/flows/{id}/steps            Permission: integration.steps.read/create
GET/PUT       /api/v1/integration/steps/{id}                  Permission: integration.steps.read/update
DELETE        /api/v1/integration/steps/{id}                  Permission: integration.steps.delete
PATCH         /api/v1/integration/steps/{id}/reorder          Permission: integration.steps.update

# Data Transformations
GET/POST      /api/v1/integration/transformations             Permission: integration.transformations.read/create
GET/PUT       /api/v1/integration/transformations/{id}        Permission: integration.transformations.read/update
DELETE        /api/v1/integration/transformations/{id}        Permission: integration.transformations.delete
POST          /api/v1/integration/transformations/{id}/test   Permission: integration.transformations.test
POST          /api/v1/integration/transformations/generate    Permission: integration.transformations.create

# Schedules
GET/POST      /api/v1/integration/schedules                   Permission: integration.schedules.read/create
GET/PUT       /api/v1/integration/schedules/{id}              Permission: integration.schedules.read/update
DELETE        /api/v1/integration/schedules/{id}              Permission: integration.schedules.delete
PATCH         /api/v1/integration/schedules/{id}/toggle       Permission: integration.schedules.update

# Execution Logs
GET           /api/v1/integration/executions                  Permission: integration.logs.read
GET           /api/v1/integration/executions/{id}             Permission: integration.logs.read
GET           /api/v1/integration/executions/{id}/steps       Permission: integration.logs.read
GET           /api/v1/integration/flows/{id}/executions       Permission: integration.logs.read
GET           /api/v1/integration/executions/{id}/payloads    Permission: integration.logs.read
POST          /api/v1/integration/executions/{id}/retry       Permission: integration.logs.execute

# Alerts
GET/POST      /api/v1/integration/alerts                      Permission: integration.alerts.read/create
GET           /api/v1/integration/alerts/{id}                 Permission: integration.alerts.read
PATCH         /api/v1/integration/alerts/{id}/acknowledge     Permission: integration.alerts.update
PATCH         /api/v1/integration/alerts/{id}/resolve         Permission: integration.alerts.update
GET           /api/v1/integration/alerts/summary              Permission: integration.alerts.read

# API Gateway
GET/POST      /api/v1/integration/gateway/routes              Permission: integration.gateway.read/create
GET/PUT       /api/v1/integration/gateway/routes/{id}         Permission: integration.gateway.read/update
DELETE        /api/v1/integration/gateway/routes/{id}         Permission: integration.gateway.delete
GET           /api/v1/integration/gateway/metrics             Permission: integration.gateway.read

# Connection Pools
GET/POST      /api/v1/integration/pools                       Permission: integration.pools.read/create
GET/PUT       /api/v1/integration/pools/{id}                  Permission: integration.pools.read/update
DELETE        /api/v1/integration/pools/{id}                  Permission: integration.pools.delete
GET           /api/v1/integration/pools/{id}/stats            Permission: integration.pools.read

# Import/Export
POST          /api/v1/integration/import                      Permission: integration.import.execute
GET           /api/v1/integration/export/{id}                 Permission: integration.export.execute
POST          /api/v1/integration/export/bulk                 Permission: integration.export.execute
```

---

## 4. Business Rules

### 4.1 Connection Management
1. Each connection MUST store credentials in an encrypted vault; plaintext credentials MUST NEVER be persisted
2. Connections MUST be tested successfully before being used in any active flow
3. Connection health checks SHOULD be performed at configurable intervals (default: 60 seconds)
4. Connections in ERROR status MUST NOT be used for new flow executions
5. Deprecated connections MUST be migrated to replacement connections before removal
6. Connection credentials MUST be rotatable without modifying dependent flows

### 4.2 Flow Design
1. Every integration flow MUST have exactly one TRIGGER step as its first step
2. Flow step ordering MUST be enforced; steps execute sequentially unless parallelism is configured
3. A flow with status DRAFT MAY be modified freely; ACTIVE flows MUST be paused before modification
4. Flow validation MUST verify all referenced connections and transformations exist and are reachable
5. Each flow SHOULD define an error handling strategy appropriate to its criticality
6. Flows with batch_size greater than 0 MUST process records in batches for memory efficiency
7. Flow execution MUST respect the configured timeout; exceeded timeouts MUST abort and alert

### 4.3 Data Transformation
1. Transformation mapping rules MUST define a source field path and target field path for each mapping
2. Transformations SHOULD support default values for missing source fields
3. Schema validation MUST be performed before and after transformation when schemas are defined
4. Custom transformation scripts MUST run in a sandboxed environment with resource limits
5. Transformation test execution MUST provide before/after data comparison
6. Lookup tables used in transformations MUST be versioned and cacheable

### 4.4 Scheduling
1. Cron expressions MUST be validated against standard cron syntax before saving
2. Scheduled flows MUST NOT execute concurrently beyond max_concurrent_runs
3. Misfire policy MUST be applied when a scheduled trigger is missed due to system downtime
4. Schedule end dates MUST be enforced; expired schedules MUST be automatically deactivated
5. Next trigger times MUST be computed and stored for monitoring and dashboards
6. Timezone-aware scheduling MUST be supported for global deployments

### 4.5 Logging and Monitoring
1. Every flow execution MUST generate a unique execution_id for correlation
2. Execution logs MUST capture payload snapshots at each step (configurable for PII redaction)
3. Log retention MUST be configurable per tenant; default retention is 90 days
4. Failed executions MUST capture error code, message, and stack trace
5. Records processed, succeeded, and failed counts MUST be aggregated per execution
6. Execution duration MUST be tracked at both flow and individual step levels

### 4.6 Alert Management
1. Alerts of severity CRITICAL MUST trigger immediate notification to assigned responders
2. Unacknowledged alerts of severity ERROR or higher MUST escalate after a configurable timeout
3. Alert notifications MUST be sent through all configured channels (email, slack, webhook)
4. Resolved alerts MUST include resolution notes for audit purposes
5. Alert deduplication MUST prevent duplicate alerts for the same failure within a cooldown window
6. Connection lost alerts MUST be generated automatically when health checks fail consecutively

### 4.7 API Gateway
1. Gateway routes MUST enforce rate limiting per tenant and per route
2. Routes requiring authentication MUST validate tokens before proxying to upstream
3. CORS headers MUST be applied only when cors_enabled is set and origins are configured
4. Route matching MUST follow the most specific path pattern when multiple routes match
5. Cache TTL MUST be respected; cached responses MUST be invalidated on TTL expiry
6. Request and response transformations MUST be applied in order when both are configured

### 4.8 Connection Pooling
1. Connection pools MUST maintain at least min_size connections at all times when active
2. Pool size MUST NOT exceed max_size; excess acquire requests MUST queue until timeout
3. Idle connections MUST be evicted after idle_timeout_ms to free resources
4. Pool statistics (active, idle, acquired, released, failed) MUST be tracked in real-time
5. Failed connections MUST be removed from the pool and replaced automatically
6. Pool health MUST be monitored; degraded pools MUST trigger alerts

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.integration.v1;

service IntegrationService {
    rpc GetConnection(GetConnectionRequest) returns (GetConnectionResponse);
    rpc TestConnection(TestConnectionRequest) returns (TestConnectionResponse);
    rpc CreateFlow(CreateFlowRequest) returns (CreateFlowResponse);
    rpc ExecuteFlow(ExecuteFlowRequest) returns (ExecuteFlowResponse);
    rpc GetExecutionStatus(GetExecutionStatusRequest) returns (GetExecutionStatusResponse);
    rpc TransformData(TransformDataRequest) returns (TransformDataResponse);
    rpc RegisterRoute(RegisterRouteRequest) returns (RegisterRouteResponse);
    rpc GetFlowMetrics(GetFlowMetricsRequest) returns (GetFlowMetricsResponse);
}

// --- Integration Connection ---
message IntegrationConnection {
    string id = 1; string tenant_id = 2; string connection_name = 3; string connection_code = 4;
    string description = 5; string system_type = 6; string endpoint_url = 7;
    string auth_type = 8; string credentials_encrypted = 9; string credential_vault_key = 10;
    string health_check_url = 11; int32 timeout_ms = 12; int32 retry_count = 13;
    int32 retry_delay_ms = 14; string status = 15; string last_tested_at = 16;
    string last_error_message = 17; string metadata = 18;
    string created_at = 19; string updated_at = 20; string created_by = 21; string updated_by = 22;
    int32 version = 23; bool is_active = 24;
}

// --- Connection Adapter ---
message ConnectionAdapter {
    string id = 1; string tenant_id = 2; string adapter_name = 3; string adapter_type = 4;
    string version_label = 5; string description = 6; string config_schema = 7;
    string default_config = 8; string supported_auth_types = 9; string capabilities = 10;
    bool is_builtin = 11; string adapter_library_path = 12; bool is_active = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- Integration Flow ---
message IntegrationFlow {
    string id = 1; string tenant_id = 2; string flow_name = 3; string flow_code = 4;
    string description = 5; string flow_type = 6; string status = 7;
    string source_connection_id = 8; string target_connection_id = 9;
    string source_adapter_id = 10; string target_adapter_id = 11;
    string trigger_type = 12; string trigger_config = 13; string error_handling_strategy = 14;
    int32 max_retries = 15; int32 retry_interval_ms = 16; int32 timeout_ms = 17;
    int32 batch_size = 18; int32 parallelism = 19; string tags = 20;
    string created_at = 21; string updated_at = 22; string created_by = 23; string updated_by = 24;
    int32 version = 25; bool is_active = 26;
}

// --- Flow Step ---
message FlowStep {
    string id = 1; string tenant_id = 2; string flow_id = 3; string step_name = 4;
    string step_type = 5; int32 step_order = 6; string description = 7; string config = 8;
    string input_schema = 9; string output_schema = 10; string error_step_id = 11;
    string condition_expression = 12; bool is_active = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- Data Transformation ---
message DataTransformation {
    string id = 1; string tenant_id = 2; string transformation_name = 3;
    string transformation_code = 4; string description = 5; string source_format = 6;
    string target_format = 7; string mapping_rules = 8; string transformation_script = 9;
    string source_schema = 10; string target_schema = 11; string lookup_tables = 12;
    bool is_active = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- Integration Schedule ---
message IntegrationSchedule {
    string id = 1; string tenant_id = 2; string flow_id = 3; string schedule_name = 4;
    string cron_expression = 5; string timezone = 6; bool is_active = 7;
    string start_date = 8; string end_date = 9; int32 max_concurrent_runs = 10;
    string misfire_policy = 11; string last_triggered_at = 12; string next_trigger_at = 13;
    int32 total_executions = 14; string last_execution_status = 15;
    string created_at = 16; string updated_at = 17; string created_by = 18; string updated_by = 19;
    int32 version = 20;
}

// --- Integration Log ---
message IntegrationLog {
    string id = 1; string tenant_id = 2; string flow_id = 3; string schedule_id = 4;
    string execution_id = 5; string step_id = 6; string log_level = 7; string message = 8;
    string payload_before = 9; string payload_after = 10; string error_code = 11;
    string error_message = 12; string error_stack_trace = 13; int32 records_processed = 14;
    int32 records_succeeded = 15; int32 records_failed = 16; int32 duration_ms = 17;
    string source_ip = 18; string correlation_id = 19;
    string created_at = 20; string updated_at = 21; string created_by = 22; string updated_by = 23;
}

// --- Integration Alert ---
message IntegrationAlert {
    string id = 1; string tenant_id = 2; string flow_id = 3; string connection_id = 4;
    string execution_id = 5; string alert_type = 6; string severity = 7; string title = 8;
    string message = 9; string alert_details = 10; string status = 11; string assigned_to = 12;
    string acknowledged_at = 13; string resolved_at = 14; string resolution_notes = 15;
    string notification_channels = 16; string escalation_rules = 17;
    string created_at = 18; string updated_at = 19; string created_by = 20; string updated_by = 21;
}

// --- API Gateway Route ---
message ApiGatewayRoute {
    string id = 1; string tenant_id = 2; string route_name = 3; string route_code = 4;
    string description = 5; string path_pattern = 6; string http_methods = 7;
    string target_connection_id = 8; string target_path = 9;
    int32 rate_limit_per_minute = 10; int32 rate_limit_per_hour = 11; int32 burst_limit = 12;
    int32 timeout_ms = 13; bool requires_auth = 14; string allowed_scopes = 15;
    string request_transform_id = 16; string response_transform_id = 17;
    int32 cache_ttl_seconds = 18; bool cors_enabled = 19; string cors_origins = 20;
    bool is_active = 21; int32 priority = 22;
    string created_at = 23; string updated_at = 24; string created_by = 25; string updated_by = 26;
    int32 version = 27;
}

// --- Connection Pool ---
message ConnectionPool {
    string id = 1; string tenant_id = 2; string connection_id = 3; string pool_name = 4;
    int32 min_size = 5; int32 max_size = 6; int32 idle_timeout_ms = 7;
    int32 max_lifetime_ms = 8; int32 acquire_timeout_ms = 9; string validation_query = 10;
    int32 current_active = 11; int32 current_idle = 12; int32 total_acquired = 13;
    int32 total_released = 14; int32 total_failed = 15; int32 avg_acquire_time_ms = 16;
    bool is_active = 17;
    string created_at = 18; string updated_at = 19; string created_by = 20; string updated_by = 21;
    int32 version = 22;
}

// --- RPC Request/Response Messages ---
message GetConnectionRequest { string tenant_id = 1; string id = 2; }
message GetConnectionResponse { IntegrationConnection data = 1; }

message TestConnectionRequest { string tenant_id = 1; string connection_id = 2; }
message TestConnectionResponse { bool success = 1; string message = 2; string tested_at = 3; }

message CreateFlowRequest { string tenant_id = 1; string flow_name = 2; string flow_code = 3; string flow_type = 4; string source_connection_id = 5; string target_connection_id = 6; }
message CreateFlowResponse { IntegrationFlow data = 1; }

message ExecuteFlowRequest { string tenant_id = 1; string flow_id = 2; string correlation_id = 3; }
message ExecuteFlowResponse { string execution_id = 1; string status = 2; }

message GetExecutionStatusRequest { string tenant_id = 1; string execution_id = 2; }
message GetExecutionStatusResponse { string status = 1; repeated IntegrationLog logs = 2; }

message TransformDataRequest { string tenant_id = 1; string transformation_id = 2; string input_data = 3; }
message TransformDataResponse { string output_data = 1; DataTransformation transformation = 2; }

message RegisterRouteRequest { string tenant_id = 1; string route_name = 2; string route_code = 3; string path_pattern = 4; string target_connection_id = 5; string target_path = 6; }
message RegisterRouteResponse { ApiGatewayRoute data = 1; }

message GetFlowMetricsRequest { string tenant_id = 1; string flow_id = 2; string date_from = 3; string date_to = 4; }
message GetFlowMetricsResponse { int32 total_executions = 1; int32 successful = 2; int32 failed = 3; int64 avg_duration_ms = 4; int32 total_records_processed = 5; }

message ListConnectionsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListConnectionsResponse { repeated IntegrationConnection items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListFlowsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListFlowsResponse { repeated IntegrationFlow items = 1; int32 total_count = 2; string next_page_token = 3; }
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Auth:** User identity for connection ownership, flow execution permissions, and audit attribution
- **Workflow:** Approval workflows for flow activation, change management processes
- **Notification:** Alert delivery, escalation notifications, execution summary reports
- **ALL Services:** Service endpoints, API contracts, authentication requirements for connection registration

### 6.2 Data Published To
- **ALL Services:** Inbound data from external systems via integration flows (GL journal entries, AP invoices, AR receipts, etc.)
- **Workflow:** Flow execution events for process orchestration and monitoring
- **Notification:** Connection status alerts, flow execution alerts, threshold breach notifications
- **Reporting:** Execution metrics, connection health, throughput analytics, error rates
- **Document-Management:** Generated log files, execution reports, transformation metadata

### 6.3 Integration Matrix

| Source Service | Integration Type | Data Exchanged |
|----------------|------------------|----------------|
| GL | Flow (inbound/outbound) | Journal entries, account balances, trial balances |
| AP | Flow (inbound/outbound) | Supplier invoices, payment confirmations |
| AR | Flow (inbound/outbound) | Customer invoices, receipt acknowledgements |
| Procurement | Flow (inbound/outbound) | Purchase orders, supplier catalogs |
| Inventory | Flow (inbound/outbound) | Stock levels, movement transactions |
| Order-Management | Flow (inbound/outbound) | Sales orders, fulfillment confirmations |
| HCM (Core-HR) | Flow (inbound/outbound) | Employee data, organizational structures |
| Product-Hub | Flow (inbound/outbound) | Product master data, pricing catalogs |
| PLM | Flow (inbound) | Engineering change orders, BOM data |
| EPM | Flow (inbound/outbound) | Budget data, actuals, forecasts |
| Auth | API call | Token validation, user permissions |
| Workflow | Event-driven | Flow approval tasks, status changes |
| Notification | Event-driven | Alert routing, notification delivery |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `integration.flow.triggered` | `{ flow_id, execution_id, trigger_type, triggered_by, timestamp }` | Flow execution has been initiated |
| `integration.flow.completed` | `{ flow_id, execution_id, records_processed, records_succeeded, records_failed, duration_ms, timestamp }` | Flow execution completed successfully |
| `integration.flow.failed` | `{ flow_id, execution_id, step_id, error_code, error_message, timestamp }` | Flow execution failed |
| `integration.connection.lost` | `{ connection_id, connection_name, error_message, timestamp }` | Connection health check failed |
| `integration.connection.restored` | `{ connection_id, connection_name, timestamp }` | Connection health check recovered |
| `integration.alert.triggered` | `{ alert_id, alert_type, severity, flow_id, connection_id, message, timestamp }` | Alert condition detected |
| `integration.alert.escalated` | `{ alert_id, severity, assigned_to, escalation_level, timestamp }` | Alert escalated due to no acknowledgement |
| `integration.route.registered` | `{ route_id, path_pattern, target_connection_id, timestamp }` | API gateway route registered |
| `integration.schedule.misfired` | `{ schedule_id, flow_id, expected_trigger_at, timestamp }` | Scheduled trigger was missed |
| `integration.step.completed` | `{ flow_id, execution_id, step_id, step_type, duration_ms, records_processed, timestamp }` | Individual flow step completed |
| `integration.transformation.applied` | `{ transformation_id, source_format, target_format, records_transformed, timestamp }` | Data transformation executed |
| `integration.pool.exhausted` | `{ pool_id, connection_id, current_active, max_size, timestamp }` | Connection pool reached capacity |

---

## 8. Migrations

1. V001: `integration_connections`
2. V002: `connection_adapters`
3. V003: `integration_flows`
4. V004: `flow_steps`
5. V005: `data_transformations`
6. V006: `integration_schedules`
7. V007: `integration_logs`
8. V008: `integration_alerts`
9. V009: `api_gateway_routes`
10. V010: `connection_pools`
11. V011: Triggers for `updated_at`
