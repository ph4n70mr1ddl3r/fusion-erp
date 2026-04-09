# 173 - Enterprise Scheduler Service Specification

## 1. Domain Overview

Enterprise Scheduler provides centralized job scheduling, execution management, and monitoring for background processes across all Fusion modules. Supports cron-based scheduling, event-driven triggers, job dependencies, priority queuing, retry logic, and execution monitoring. Enables administrators to schedule recurring jobs (report generation, data loads, batch processing), manage job lifecycles, monitor execution health, and handle failures with alerting.

**Bounded Context:** Centralized Job Scheduling & Execution
**Service Name:** `scheduler-service`
**Database:** `data/scheduler.db`
**HTTP Port:** 8191 | **gRPC Port:** 9191

---

## 2. Database Schema

### 2.1 Job Definitions
```sql
CREATE TABLE sched_job_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    job_code TEXT NOT NULL,
    job_name TEXT NOT NULL,
    job_type TEXT NOT NULL CHECK(job_type IN ('REPORT','DATA_LOAD','DATA_EXPORT','BATCH_PROCESS','CLEANUP','NOTIFICATION','INTEGRATION','CUSTOM')),
    description TEXT,
    source_module TEXT NOT NULL,
    execution_endpoint TEXT NOT NULL,        -- REST/gRPC endpoint to invoke
    execution_method TEXT NOT NULL DEFAULT 'REST'
        CHECK(execution_method IN ('REST','GRPC','INTERNAL','SCRIPT')),
    parameters TEXT,                         -- JSON: job parameter definitions
    default_parameters TEXT,                 -- JSON: default values
    timeout_seconds INTEGER NOT NULL DEFAULT 3600,
    max_retries INTEGER NOT NULL DEFAULT 0,
    retry_delay_seconds INTEGER NOT NULL DEFAULT 300,
    retry_backoff_multiplier REAL NOT NULL DEFAULT 2.0,
    priority INTEGER NOT NULL DEFAULT 5,    -- 1 (highest) to 10 (lowest)
    estimated_duration_seconds INTEGER,
    singleton INTEGER NOT NULL DEFAULT 0,   -- Only one instance at a time
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, job_code)
);

CREATE INDEX idx_sched_job_module ON sched_job_definitions(source_module);
```

### 2.2 Schedules
```sql
CREATE TABLE sched_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    job_id TEXT NOT NULL,
    cron_expression TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    parameters TEXT,                         -- JSON: runtime parameters
    active_from TEXT,
    active_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','EXPIRED','CANCELLED')),
    last_run_at TEXT,
    last_run_status TEXT CHECK(last_run_status IN ('SUCCESS','FAILURE','TIMEOUT')),
    next_run_at TEXT,
    total_runs INTEGER NOT NULL DEFAULT 0,
    consecutive_failures INTEGER NOT NULL DEFAULT 0,
    max_consecutive_failures INTEGER NOT NULL DEFAULT 3,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (job_id) REFERENCES sched_job_definitions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, schedule_name)
);

CREATE INDEX idx_sched_next ON sched_schedules(next_run_at, status);
CREATE INDEX idx_sched_job ON sched_schedules(job_id, status);
```

### 2.3 Job Executions
```sql
CREATE TABLE sched_executions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    job_id TEXT NOT NULL,
    schedule_id TEXT,
    execution_number TEXT NOT NULL,          -- EXEC-2024-00001
    trigger_type TEXT NOT NULL CHECK(trigger_type IN ('SCHEDULED','MANUAL','EVENT','API','DEPENDENCY','RETRY')),
    status TEXT NOT NULL DEFAULT 'QUEUED'
        CHECK(status IN ('QUEUED','RUNNING','COMPLETED','FAILED','TIMEOUT','CANCELLED','RETRYING')),
    priority INTEGER NOT NULL DEFAULT 5,
    parameters TEXT,
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,
    result_summary TEXT,
    error_message TEXT,
    output_artifacts TEXT,                   -- JSON: files/reports generated
    records_processed INTEGER NOT NULL DEFAULT 0,
    records_succeeded INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    retry_attempt INTEGER NOT NULL DEFAULT 0,
    worker_node TEXT,
    progress_pct REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (job_id) REFERENCES sched_job_definitions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, execution_number)
);

CREATE INDEX idx_sched_exec_job ON sched_executions(job_id, started_at DESC);
CREATE INDEX idx_sched_exec_status ON sched_executions(tenant_id, status);
CREATE INDEX idx_sched_exec_time ON sched_executions(tenant_id, started_at DESC);
```

### 2.4 Job Dependencies
```sql
CREATE TABLE sched_dependencies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    parent_job_id TEXT NOT NULL,
    child_job_id TEXT NOT NULL,
    condition TEXT NOT NULL DEFAULT 'ON_SUCCESS'
        CHECK(condition IN ('ON_SUCCESS','ON_FAILURE','ON_COMPLETION')),
    delay_seconds INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, parent_job_id, child_job_id, condition)
);
```

### 2.5 Execution Log
```sql
CREATE TABLE sched_execution_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    execution_id TEXT NOT NULL,
    log_level TEXT NOT NULL CHECK(log_level IN ('DEBUG','INFO','WARN','ERROR')),
    message TEXT NOT NULL,
    details TEXT,
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (execution_id) REFERENCES sched_executions(id) ON DELETE CASCADE
);

CREATE INDEX idx_sched_log_exec ON sched_execution_log(execution_id, timestamp);
```

---

## 3. API Endpoints

### 3.1 Job Definitions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/scheduler/v1/jobs` | Register job |
| GET | `/scheduler/v1/jobs` | List jobs |
| GET | `/scheduler/v1/jobs/{id}` | Get job details |
| PUT | `/scheduler/v1/jobs/{id}` | Update job |

### 3.2 Schedules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/scheduler/v1/schedules` | Create schedule |
| GET | `/scheduler/v1/schedules` | List schedules |
| PUT | `/scheduler/v1/schedules/{id}` | Update schedule |
| POST | `/scheduler/v1/schedules/{id}/pause` | Pause schedule |
| POST | `/scheduler/v1/schedules/{id}/resume` | Resume schedule |
| DELETE | `/scheduler/v1/schedules/{id}` | Cancel schedule |

### 3.3 Execution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/scheduler/v1/jobs/{id}/run` | Run job now |
| GET | `/scheduler/v1/executions` | List executions |
| GET | `/scheduler/v1/executions/{id}` | Get execution details |
| GET | `/scheduler/v1/executions/{id}/log` | Get execution log |
| POST | `/scheduler/v1/executions/{id}/cancel` | Cancel execution |
| POST | `/scheduler/v1/executions/{id}/retry` | Retry execution |

### 3.4 Monitoring
| Method | Path | Description |
|--------|------|-------------|
| GET | `/scheduler/v1/dashboard` | Scheduler dashboard |
| GET | `/scheduler/v1/health` | Service health |
| GET | `/scheduler/v1/queue` | Current job queue |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sched.job.started` | `{ exec_id, job_id, trigger_type }` | Job execution started |
| `sched.job.completed` | `{ exec_id, job_id, duration_ms }` | Job completed |
| `sched.job.failed` | `{ exec_id, job_id, error }` | Job failed |
| `sched.schedule.paused` | `{ schedule_id, reason }` | Schedule auto-paused |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| Any business event | All modules | Trigger event-driven jobs |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.scheduler.v1;

service SchedulerService {
    rpc GetJobDefinition(GetJobDefinitionRequest) returns (GetJobDefinitionResponse);
    rpc RegisterJob(RegisterJobRequest) returns (RegisterJobResponse);
    rpc CreateSchedule(CreateScheduleRequest) returns (CreateScheduleResponse);
    rpc RunJob(RunJobRequest) returns (RunJobResponse);
    rpc GetExecution(GetExecutionRequest) returns (GetExecutionResponse);
    rpc CancelExecution(CancelExecutionRequest) returns (CancelExecutionResponse);
}

// Job Definition messages
message GetJobDefinitionRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetJobDefinitionResponse {
    JobDefinition data = 1;
}

message JobDefinition {
    string id = 1;
    string tenant_id = 2;
    string job_code = 3;
    string job_name = 4;
    string job_type = 5;
    string source_module = 6;
    string execution_endpoint = 7;
    string execution_method = 8;
    string parameters = 9;
    string default_parameters = 10;
    int32 timeout_seconds = 11;
    int32 max_retries = 12;
    int32 retry_delay_seconds = 13;
    double retry_backoff_multiplier = 14;
    int32 priority = 15;
    int32 singleton = 16;
    string status = 17;
    string created_at = 18;
    string updated_at = 19;
}

message RegisterJobRequest {
    string tenant_id = 1;
    string job_code = 2;
    string job_name = 3;
    string job_type = 4;
    string source_module = 5;
    string execution_endpoint = 6;
    string execution_method = 7;
    string parameters = 8;
    int32 timeout_seconds = 9;
    int32 max_retries = 10;
    int32 retry_delay_seconds = 11;
    int32 priority = 12;
    int32 singleton = 13;
}

message RegisterJobResponse {
    JobDefinition data = 1;
}

// Schedule messages
message CreateScheduleRequest {
    string tenant_id = 1;
    string schedule_name = 2;
    string job_id = 3;
    string cron_expression = 4;
    string timezone = 5;
    string parameters = 6;
    string active_from = 7;
    string active_to = 8;
    int32 max_consecutive_failures = 9;
}

message CreateScheduleResponse {
    Schedule data = 1;
}

message Schedule {
    string id = 1;
    string tenant_id = 2;
    string schedule_name = 3;
    string job_id = 4;
    string cron_expression = 5;
    string timezone = 6;
    string status = 7;
    string last_run_at = 8;
    string last_run_status = 9;
    string next_run_at = 10;
    int32 total_runs = 11;
    int32 consecutive_failures = 12;
    string created_at = 13;
    string updated_at = 14;
}

// Execution messages
message RunJobRequest {
    string tenant_id = 1;
    string job_id = 2;
    string trigger_type = 3;
    string parameters = 4;
    int32 priority = 5;
}

message RunJobResponse {
    Execution data = 1;
}

message GetExecutionRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetExecutionResponse {
    Execution data = 1;
}

message Execution {
    string id = 1;
    string tenant_id = 2;
    string job_id = 3;
    string schedule_id = 4;
    string execution_number = 5;
    string trigger_type = 6;
    string status = 7;
    int32 priority = 8;
    string parameters = 9;
    string started_at = 10;
    string completed_at = 11;
    int32 execution_time_ms = 12;
    string result_summary = 13;
    string error_message = 14;
    int32 records_processed = 15;
    int32 records_succeeded = 16;
    int32 records_failed = 17;
    int32 retry_attempt = 18;
    double progress_pct = 19;
    string created_at = 20;
}

message CancelExecutionRequest {
    string tenant_id = 1;
    string id = 2;
}

message CancelExecutionResponse {
    Execution data = 1;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | sched_job_definitions | -- |
| V002 | sched_schedules | V001 |
| V003 | sched_executions | V001 |
| V004 | sched_dependencies | V001 |
| V005 | sched_execution_log | V003 |

---

## 7. Business Rules

1. **Singleton Enforcement**: Singleton jobs reject new execution if already running
2. **Auto-Pause**: Schedules auto-paused after N consecutive failures (configurable)
3. **Priority Queue**: Higher priority jobs executed first within tenant queue
4. **Timeout**: Jobs exceeding timeout are killed and marked TIMEOUT
5. **Retry Logic**: Retries with exponential backoff up to max retries
6. **Dependency Chains**: Child jobs queued only after parent satisfies condition

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| All Modules | Job registration and execution |
| BI Publisher (150) | Report generation scheduling |
| EPM Automate (156) | EPM batch operations |
| Data Import/Export (30) | Scheduled data loads |
| Notification Center (165) | Job failure alerts |
| Integration Cloud (89) | Integration job scheduling |
