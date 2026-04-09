# 122 - Production Scheduling Specification

## 1. Domain Overview

Production Scheduling provides advanced shop floor scheduling capabilities, optimizing resource allocation across work centers, machines, and labor. It generates feasible production schedules using constraint-based planning, supports Gantt chart visualization, and enables real-time schedule adjustments based on capacity, material availability, and priority changes.

**Bounded Context:** Production Scheduling & Shop Floor Optimization
**Service Name:** `scheduling-service` (Note: distinct from workforce scheduling service `scheduling-service` in spec 73 — this uses `prodsched-service`)
**Database:** `data/prodsched.db`
**HTTP Port:** 8202 | **gRPC Port:** 9202

---

## 2. Database Schema

### 2.1 Resources
```sql
CREATE TABLE ps_resources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_code TEXT NOT NULL,
    resource_name TEXT NOT NULL,
    resource_type TEXT NOT NULL CHECK(resource_type IN ('MACHINE','LABOR','WORK_CENTER','PRODUCTION_LINE','TOOL')),
    department TEXT,
    description TEXT,
    capacity_per_hour REAL NOT NULL DEFAULT 1.0,
    capacity_uom TEXT NOT NULL,          -- Units produced per hour
    efficiency_percentage REAL NOT NULL DEFAULT 100.0,
    utilization_target_pct REAL NOT NULL DEFAULT 85.0,
    cost_per_hour_cents INTEGER NOT NULL DEFAULT 0,
    is_bottleneck INTEGER NOT NULL DEFAULT 0,
    schedule_group TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, resource_code)
);

CREATE INDEX idx_ps_resources_tenant_type ON ps_resources(tenant_id, resource_type);
```

### 2.2 Resource Shifts
```sql
CREATE TABLE ps_resource_shifts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    shift_name TEXT NOT NULL,            -- "Morning", "Night", "Weekend"
    day_of_week INTEGER NOT NULL CHECK(day_of_week BETWEEN 1 AND 7),
    start_time TEXT NOT NULL,            -- "08:00"
    end_time TEXT NOT NULL,              -- "16:00"
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (resource_id) REFERENCES ps_resources(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, resource_id, shift_name, day_of_week)
);
```

### 2.3 Resource Downtime
```sql
CREATE TABLE ps_resource_downtime (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    downtime_type TEXT NOT NULL CHECK(downtime_type IN ('PLANNED_MAINTENANCE','BREAKDOWN','SETUP','CHANGEOVER','CALIBRATION','HOLIDAY')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    reason TEXT,
    is_recurring INTEGER NOT NULL DEFAULT 0,
    recurrence_rule TEXT,                -- iCal RRULE format

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (resource_id) REFERENCES ps_resources(id) ON DELETE CASCADE
);
```

### 2.4 Scheduling Parameters
```sql
CREATE TABLE ps_scheduling_params (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    param_name TEXT NOT NULL,
    param_value TEXT NOT NULL,
    param_type TEXT NOT NULL DEFAULT 'STRING' CHECK(param_type IN ('STRING','INTEGER','REAL','BOOLEAN','JSON')),
    description TEXT,
    category TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, param_name)
);
```

### 2.5 Schedule Orders
```sql
CREATE TABLE ps_schedule_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_type TEXT NOT NULL CHECK(order_type IN ('WORK_ORDER','BATCH','MAINTENANCE')),
    order_id TEXT NOT NULL,              -- FK to work order or batch
    item_id TEXT NOT NULL,
    quantity REAL NOT NULL,
    uom TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 5, -- 1 (highest) to 10 (lowest)
    due_date TEXT NOT NULL,
    routing_id TEXT,
    estimated_duration_minutes INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'UNSCHEDULED'
        CHECK(status IN ('UNSCHEDULED','SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_type, order_id)
);

CREATE INDEX idx_ps_orders_tenant_status ON ps_schedule_orders(tenant_id, status);
CREATE INDEX idx_ps_orders_tenant_due ON ps_schedule_orders(tenant_id, due_date, priority);
```

### 2.6 Schedule Operations
```sql
CREATE TABLE ps_schedule_operations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_order_id TEXT NOT NULL,
    operation_number INTEGER NOT NULL,
    operation_name TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    planned_start TEXT NOT NULL,
    planned_end TEXT NOT NULL,
    actual_start TEXT,
    actual_end TEXT,
    setup_minutes INTEGER NOT NULL DEFAULT 0,
    run_time_minutes INTEGER NOT NULL,
    queue_minutes INTEGER NOT NULL DEFAULT 0,
    move_minutes INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','SETUP','RUNNING','COMPLETED','SKIPPED')),
    predecessor_operation_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (schedule_order_id) REFERENCES ps_schedule_orders(id) ON DELETE CASCADE,
    FOREIGN KEY (resource_id) REFERENCES ps_resources(id) ON DELETE RESTRICT,
    FOREIGN KEY (predecessor_operation_id) REFERENCES ps_schedule_operations(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, schedule_order_id, operation_number)
);

CREATE INDEX idx_ps_ops_tenant_resource ON ps_schedule_operations(tenant_id, resource_id, planned_start);
CREATE INDEX idx_ps_ops_tenant_dates ON ps_schedule_operations(tenant_id, planned_start, planned_end);
```

### 2.7 Scheduling Runs
```sql
CREATE TABLE ps_schedule_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_name TEXT NOT NULL,
    scheduling_method TEXT NOT NULL CHECK(scheduling_method IN ('FORWARD','BACKWARD','FINITE_CAPACITY','INFINITE_CAPACITY','OPTIMIZED')),
    planning_horizon_start TEXT NOT NULL,
    planning_horizon_end TEXT NOT NULL,
    total_orders INTEGER NOT NULL DEFAULT 0,
    scheduled_orders INTEGER NOT NULL DEFAULT 0,
    unscheduled_orders INTEGER NOT NULL DEFAULT 0,
    total_tardiness_minutes INTEGER NOT NULL DEFAULT 0,
    resource_utilization_pct REAL NOT NULL DEFAULT 0.0,
    makespan_minutes INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED','SUPERSEDED')),
    schedule_version INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, run_name, schedule_version)
);
```

---

## 3. REST API Endpoints

### 3.1 Resources
```
GET    /api/v1/prodsched/resources                     Permission: ps.resources.read
GET    /api/v1/prodsched/resources/{id}                Permission: ps.resources.read
POST   /api/v1/prodsched/resources                     Permission: ps.resources.create
PUT    /api/v1/prodsched/resources/{id}                Permission: ps.resources.update
GET    /api/v1/prodsched/resources/{id}/capacity
  ?date_from=2024-01-01&date_to=2024-01-31             Permission: ps.resources.read
GET    /api/v1/prodsched/resources/{id}/shifts          Permission: ps.resources.read
POST   /api/v1/prodsched/resources/{id}/downtime        Permission: ps.resources.update
GET    /api/v1/prodsched/resources/{id}/gantt
  ?date_from=2024-01-01&date_to=2024-01-31             Permission: ps.resources.read
```

### 3.2 Scheduling
```
POST   /api/v1/prodsched/schedule/run                  Permission: ps.schedule.run
  Request: {
    "method": "FINITE_CAPACITY",
    "horizon_start": "2024-01-01",
    "horizon_end": "2024-01-31",
    "resource_ids": ["r1", "r2"],
    "consider_downtime": true,
    "optimize_for": "MINIMIZE_TARDINESS"
  }
  Response 201: { "data": { "run_id": "...", "status": "RUNNING" } }

GET    /api/v1/prodsched/schedule/runs                  Permission: ps.schedule.read
GET    /api/v1/prodsched/schedule/runs/{id}             Permission: ps.schedule.read
GET    /api/v1/prodsched/schedule/runs/{id}/result      Permission: ps.schedule.read
POST   /api/v1/prodsched/schedule/runs/{id}/publish     Permission: ps.schedule.publish
```

### 3.3 Schedule Orders & Operations
```
GET    /api/v1/prodsched/orders                         Permission: ps.orders.read
POST   /api/v1/prodsched/orders                         Permission: ps.orders.create
PUT    /api/v1/prodsched/orders/{id}/priority           Permission: ps.orders.update
GET    /api/v1/prodsched/operations                     Permission: ps.operations.read
PUT    /api/v1/prodsched/operations/{id}                Permission: ps.operations.update
POST   /api/v1/prodsched/operations/{id}/start          Permission: ps.operations.execute
POST   /api/v1/prodsched/operations/{id}/complete       Permission: ps.operations.execute
```

### 3.4 Gantt & Visualization
```
GET    /api/v1/prodsched/gantt
  ?date_from=2024-01-01&date_to=2024-01-31
  ?resource_id={id}
  ?view=resource|order                               Permission: ps.schedule.read
  Response: { "data": { "resources": [...], "operations": [...], "now_line": "..." } }

GET    /api/v1/prodsched/dashboard                      Permission: ps.dashboard.read
  Response: { "data": { "utilization": 87.5, "on_time_pct": 92.3, "tardiness_minutes": 120, "bottleneck_resource_id": "..." } }
```

---

## 4. Business Rules

### 4.1 Scheduling Constraints
- Operations MUST be scheduled within resource available shift times
- Resource downtime blocks MUST not be overlapped
- Predecessor operations MUST complete before successor starts
- Setup/Changeover time MUST be included between different product runs
- Finite capacity: total scheduled time per resource MUST NOT exceed available capacity
- Infinite capacity: overloads allowed but flagged as warnings

### 4.2 Priority Rules
- Orders with priority 1 are scheduled first (earliest due date tie-breaker)
- Emergency orders can interrupt existing schedule (manual override with approval)
- Late orders automatically escalated in priority
- Customer VIP classification can boost priority

### 4.3 Optimization Objectives
- Minimize tardiness: reduce total late minutes across all orders
- Maximize utilization: fill resource capacity as fully as possible
- Minimize makespan: complete all orders in shortest total time
- Minimize changeover: group similar products to reduce setup time
- Custom weighted combination of above objectives

### 4.4 Real-Time Updates
- Actual operation start/end times update the schedule in real time
- Deviations from planned times trigger automatic rescheduling of downstream operations
- Resource breakdown triggers immediate rescheduling of affected operations
- New rush orders inserted with constraint-aware scheduling

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.prodsched.v1;

service ProductionSchedulingService {
    rpc GetResourceCapacity(GetResourceCapacityRequest) returns (GetResourceCapacityResponse);
    rpc RunSchedule(RunScheduleRequest) returns (RunScheduleResponse);
    rpc GetScheduleResult(GetScheduleResultRequest) returns (GetScheduleResultResponse);
    rpc GetGanttData(GetGanttDataRequest) returns (GetGanttDataResponse);
    rpc UpdateOperationStatus(UpdateOperationStatusRequest) returns (UpdateOperationStatusResponse);
    rpc Reschedule(RescheduleRequest) returns (RescheduleResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **mfg-service**: Work orders, routings, operation definitions
- **processmfg-service**: Batch production orders
- **inv-service**: Material availability checks
- **planning-service**: Planned production orders from MRP
- **eam-service**: Resource maintenance schedules, equipment status
- **inv-service**: Material availability for scheduling feasibility

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `ps.schedule.run.started` | Scheduling run initiated | run_id, method, horizon |
| `ps.schedule.run.completed` | Scheduling run finished | run_id, scheduled_count, tardiness |
| `ps.schedule.published` | Schedule version published | run_id, version |
| `ps.operation.started` | Operation execution started | operation_id, resource_id, actual_start |
| `ps.operation.completed` | Operation execution finished | operation_id, actual_end, variance_minutes |
| `ps.resource.down` | Resource breakdown reported | resource_id, estimated_recovery |
| `ps.order.tardy` | Order predicted late | order_id, due_date, estimated_completion |

---

## 7. Migrations

### Migration Order for prodsched-service:
1. V001: `ps_resources`
2. V002: `ps_resource_shifts`
3. V003: `ps_resource_downtime`
4. V004: `ps_scheduling_params`
5. V005: `ps_schedule_orders`
6. V006: `ps_schedule_operations`
7. V007: `ps_schedule_runs`
8. V008: Triggers for `updated_at`
9. V009: Seed data (default scheduling parameters, sample shift patterns)
