# 52 - Supply Chain Orchestration Service Specification

## 1. Domain Overview

Supply Chain Orchestration provides a central coordination layer that manages end-to-end supply chain processes across multiple fulfillment systems. It orchestrates tasks like supply assignment, demand fulfillment, transfer order creation, and intercompany supply using the Saga pattern for distributed transaction management. Provides a unified process layer over disparate fulfillment systems, ensuring that a single demand signal triggers the correct sequence of supply chain events across procurement, manufacturing, inventory, and logistics.

**Bounded Context:** Supply Chain Process Orchestration & Saga Coordination
**Service Name:** `orchestration-service`
**Database:** `data/orchestration.db`
**HTTP Port:** 8084 | **gRPC Port:** 9084

---

## 2. Database Schema

### 2.1 Orchestration Processes
```sql
CREATE TABLE orchestration_processes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    process_code TEXT NOT NULL,
    process_name TEXT NOT NULL,
    process_type TEXT NOT NULL
        CHECK(process_type IN ('ORDER_FULFILLMENT','PROCUREMENT','TRANSFER','RETURN','INTERCOMPANY','BACK_TO_BACK','DROP_SHIP')),
    description TEXT,
    version INTEGER NOT NULL DEFAULT 1,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, process_code)
);

CREATE INDEX idx_orch_processes_tenant ON orchestration_processes(tenant_id, process_type, is_active);
```

### 2.2 Orchestration Steps
```sql
CREATE TABLE orchestration_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    process_id TEXT NOT NULL,
    step_code TEXT NOT NULL,
    step_name TEXT NOT NULL,
    step_type TEXT NOT NULL
        CHECK(step_type IN ('TASK','DECISION','PARALLEL_SPLIT','PARALLEL_JOIN','TIMER','NOTIFICATION','SUB_PROCESS')),
    target_service TEXT NOT NULL,
    target_operation TEXT NOT NULL,
    input_mapping TEXT,
    output_mapping TEXT,
    timeout_seconds INTEGER NOT NULL DEFAULT 300,
    max_retries INTEGER NOT NULL DEFAULT 3,
    retry_delay_seconds INTEGER NOT NULL DEFAULT 30,
    sequence INTEGER NOT NULL,
    parent_step_id TEXT,
    branch_condition TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (process_id) REFERENCES orchestration_processes(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_step_id) REFERENCES orchestration_steps(id),
    UNIQUE(tenant_id, process_id, step_code)
);

CREATE INDEX idx_orch_steps_process ON orchestration_steps(process_id, sequence);
```

### 2.3 Step Dependencies
```sql
CREATE TABLE step_dependencies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    step_id TEXT NOT NULL,
    depends_on_step_id TEXT NOT NULL,
    condition_expression TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    FOREIGN KEY (step_id) REFERENCES orchestration_steps(id) ON DELETE CASCADE,
    FOREIGN KEY (depends_on_step_id) REFERENCES orchestration_steps(id) ON DELETE CASCADE,
    UNIQUE(step_id, depends_on_step_id)
);
```

### 2.4 Process Instances
```sql
CREATE TABLE process_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    instance_number TEXT NOT NULL,
    process_id TEXT NOT NULL,
    source_type TEXT NOT NULL,
    source_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'STARTED'
        CHECK(status IN ('STARTED','IN_PROGRESS','COMPLETED','FAILED','CANCELLED','COMPENSATING','COMPENSATED')),
    started_at TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at TEXT,
    error_message TEXT,
    total_steps INTEGER NOT NULL DEFAULT 0,
    completed_steps INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (process_id) REFERENCES orchestration_processes(id),
    UNIQUE(tenant_id, instance_number)
);

CREATE INDEX idx_process_instances_tenant ON process_instances(tenant_id, status, started_at);
CREATE INDEX idx_process_instances_source ON process_instances(tenant_id, source_type, source_id);
```

### 2.5 Step Executions
```sql
CREATE TABLE step_executions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    instance_id TEXT NOT NULL,
    step_id TEXT NOT NULL,
    execution_sequence INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','SKIPPED','COMPENSATING','COMPENSATED','TIMED_OUT')),
    input_data TEXT,
    output_data TEXT,
    error_message TEXT,
    started_at TEXT,
    completed_at TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    target_service_response TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (instance_id) REFERENCES process_instances(id),
    FOREIGN KEY (step_id) REFERENCES orchestration_steps(id)
);

CREATE INDEX idx_step_exec_instance ON step_executions(instance_id, execution_sequence);
CREATE INDEX idx_step_exec_status ON step_executions(tenant_id, status);
```

### 2.6 Task Assignments
```sql
CREATE TABLE task_assignments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    step_execution_id TEXT NOT NULL,
    assigned_service TEXT NOT NULL,
    assigned_handler TEXT,
    priority INTEGER NOT NULL DEFAULT 100,
    deadline TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (step_execution_id) REFERENCES step_executions(id)
);

CREATE INDEX idx_task_assign_exec ON task_assignments(step_execution_id);
CREATE INDEX idx_task_assign_service ON task_assignments(assigned_service, priority);
```

### 2.7 Compensation Actions
```sql
CREATE TABLE compensation_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    instance_id TEXT NOT NULL,
    step_execution_id TEXT NOT NULL,
    compensation_type TEXT NOT NULL
        CHECK(compensation_type IN ('UNDO','CANCEL','REVERSE','NOTIFY')),
    target_service TEXT NOT NULL,
    target_operation TEXT NOT NULL,
    input_data TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','EXECUTING','COMPLETED','FAILED')),
    executed_at TEXT,
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (instance_id) REFERENCES process_instances(id),
    FOREIGN KEY (step_execution_id) REFERENCES step_executions(id)
);

CREATE INDEX idx_compensation_instance ON compensation_actions(instance_id, status);
```

---

## 3. REST API Endpoints

### 3.1 Process Definitions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/orchestration/processes` | List process definitions |
| POST | `/api/v1/orchestration/processes` | Create process definition |
| GET | `/api/v1/orchestration/processes/{id}` | Get process with steps |
| PUT | `/api/v1/orchestration/processes/{id}` | Update process definition |
| DELETE | `/api/v1/orchestration/processes/{id}` | Deactivate process |

### 3.2 Step Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/orchestration/processes/{id}/steps` | Add step to process |
| PUT | `/api/v1/orchestration/processes/{id}/steps/{stepId}` | Update step |
| DELETE | `/api/v1/orchestration/processes/{id}/steps/{stepId}` | Remove step |
| POST | `/api/v1/orchestration/processes/{id}/steps/{stepId}/dependencies` | Add step dependency |

### 3.3 Process Execution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/orchestration/instances` | Start process instance |
| GET | `/api/v1/orchestration/instances` | List process instances |
| GET | `/api/v1/orchestration/instances/{id}` | Get instance with step status |
| POST | `/api/v1/orchestration/instances/{id}/cancel` | Cancel running instance |
| POST | `/api/v1/orchestration/instances/{id}/retry` | Retry failed instance |
| POST | `/api/v1/orchestration/instances/{id}/compensate` | Trigger compensation |

### 3.4 Step Executions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/orchestration/instances/{id}/steps` | Get step execution status |
| POST | `/api/v1/orchestration/instances/{id}/steps/{stepExecId}/complete` | Complete a step |
| POST | `/api/v1/orchestration/instances/{id}/steps/{stepExecId}/fail` | Report step failure |
| POST | `/api/v1/orchestration/instances/{id}/steps/{stepExecId}/skip` | Skip a step |

### 3.5 Monitoring
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/orchestration/monitoring/dashboard` | Get orchestration dashboard |
| GET | `/api/v1/orchestration/monitoring/failed` | List failed instances |
| GET | `/api/v1/orchestration/monitoring/stats` | Get execution statistics |

---

## 4. Business Rules

1. Process definitions MUST have at least one step before activation.
2. Steps without dependencies are starting steps and execute immediately when the process starts.
3. A step executes only when ALL its dependencies are completed successfully.
4. PARALLEL_SPLIT MUST be paired with a corresponding PARALLEL_JOIN step.
5. Step execution timeout MUST trigger a failure and initiate retry logic.
6. After max retries exceeded, the step MUST be marked as FAILED and the process enters compensation mode.
7. Compensation actions MUST execute in reverse order of completed steps (Saga pattern).
8. Process instances MUST be uniquely identifiable by source_type and source_id.
9. Cancellation of a running process MUST trigger compensation for all completed steps.
10. Step input/output data MUST be serialized as JSON for audit and recovery.
11. Process definitions MAY be versioned — running instances use the version active at start time.
12. Each tenant MUST be isolated — process definitions and instances are tenant-scoped.

---

## 5. gRPC Service Definition

```protobuf
service OrchestrationService {
    rpc CreateProcess(CreateProcessRequest) returns (ProcessResponse);
    rpc GetProcess(GetProcessRequest) returns (ProcessResponse);
    rpc ListProcesses(ListProcessesRequest) returns (ListProcessesResponse);

    rpc StartInstance(StartInstanceRequest) returns (InstanceResponse);
    rpc GetInstance(GetInstanceRequest) returns (InstanceResponse);
    rpc ListInstances(ListInstancesRequest) returns (ListInstancesResponse);
    rpc CancelInstance(CancelInstanceRequest) returns (InstanceResponse);
    rpc RetryInstance(RetryInstanceRequest) returns (InstanceResponse);
    rpc CompensateInstance(CompensateInstanceRequest) returns (InstanceResponse);

    rpc CompleteStepExecution(CompleteStepRequest) returns (StepExecutionResponse);
    rpc FailStepExecution(FailStepRequest) returns (StepExecutionResponse);

    rpc GetMonitoringDashboard(GetDashboardRequest) returns (DashboardResponse);
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `om-service` | Sales order creation triggers fulfillment orchestration |
| `proc-service` | Purchase requisitions trigger procurement orchestration |
| `inv-service` | Transfer requirements trigger transfer orchestration |
| All services | Step completion callbacks |

### Published To
| Service | Data |
|---------|------|
| `om-service` | Fulfillment status updates |
| `proc-service` | Purchase order creation, goods receipt |
| `inv-service` | Inventory movements, reservations, transfers |
| `mfg-service` | Work order creation, material issues |
| `wms-service` | Pick, pack, ship instructions |
| `tms-service` | Shipment planning, carrier assignment |
| `workflow-service` | Approval requests for decision steps |
| `report-service` | Process execution metrics |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `orchestration.process-defined` | `{process_id, code, type}` | Process definition created |
| `orchestration.instance-started` | `{instance_id, process_id, source}` | Process instance started |
| `orchestration.step-completed` | `{instance_id, step_code, output}` | Step execution completed |
| `orchestration.step-failed` | `{instance_id, step_code, error}` | Step execution failed |
| `orchestration.instance-completed` | `{instance_id, duration_ms}` | Process instance completed |
| `orchestration.instance-failed` | `{instance_id, failed_step, error}` | Process instance failed |
| `orchestration.compensation-started` | `{instance_id}` | Compensation initiated |
| `orchestration.compensation-completed` | `{instance_id}` | Compensation finished |
| `orchestration.instance-cancelled` | `{instance_id, reason}` | Instance cancelled |
