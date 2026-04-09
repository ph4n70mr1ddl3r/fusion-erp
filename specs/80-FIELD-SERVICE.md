# 80 - Field Service Service Specification

## 1. Domain Overview

Field Service manages field service operations including appointment scheduling, technician dispatch, resource management, shift scheduling, route optimization, service task tracking, parts logistics, service completion documentation, customer feedback collection, and service contract management. Appointments are created from customer requests, work orders, or preventive maintenance schedules and assigned to qualified technicians based on skills, availability, and proximity. Dispatch rules automate technician assignment using configurable criteria. Route optimization minimizes travel time across daily appointments. Service tasks track work performed, time spent, and parts consumed. Parts needed for service are ordered from inventory. Service completions capture work summaries, resolution codes, and customer sign-off. Customer feedback is collected post-service for quality tracking. Service contracts define coverage terms, SLAs, and entitled service types. Integrates with Inventory for parts availability, Transportation Management for fleet tracking, Enterprise Asset Management for asset service history, and Order Management for parts ordering.

**Bounded Context:** Field Service Operations
**Service Name:** `fieldservice-service`
**Database:** `data/fieldservice.db`
**HTTP Port:** 8112 | **gRPC Port:** 9112

---

## 2. Database Schema

### 2.1 Service Appointments
```sql
CREATE TABLE fieldservice_appointments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appointment_number TEXT NOT NULL,
    account_id TEXT NOT NULL,
    contact_id TEXT,
    asset_id TEXT,
    contract_id TEXT,
    work_order_id TEXT,
    service_type TEXT NOT NULL
        CHECK(service_type IN ('INSTALLATION','REPAIR','MAINTENANCE','INSPECTION','EMERGENCY','UPGRADE','DECOMMISSION')),
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('CRITICAL','HIGH','MEDIUM','LOW')),
    status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(status IN ('SCHEDULED','DISPATCHED','EN_ROUTE','ON_SITE','IN_PROGRESS','COMPLETED','CANCELLED','RESCHEDULED','NO_SHOW')),
    description TEXT NOT NULL,
    resolution_notes TEXT,
    resolution_code TEXT
        CHECK(resolution_code IN ('RESOLVED','PARTIAL','ESCALATED','DEFERRED','NOT_REPRODUCIBLE','REPLACED')),
    scheduled_start TEXT NOT NULL,
    scheduled_end TEXT NOT NULL,
    actual_start TEXT,
    actual_end TEXT,
    estimated_duration_minutes INTEGER NOT NULL,
    actual_duration_minutes INTEGER,
    travel_time_minutes INTEGER,
    assigned_resource_id TEXT,
    dispatched_at TEXT,
    arrived_at TEXT,
    location_address TEXT,
    location_city TEXT,
    location_state TEXT,
    location_postal_code TEXT,
    location_country TEXT,
    latitude REAL,
    longitude REAL,
    customer_sign_off TEXT,
    internal_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, appointment_number)
);

CREATE INDEX idx_appointments_tenant_status ON fieldservice_appointments(tenant_id, status);
CREATE INDEX idx_appointments_tenant_account ON fieldservice_appointments(tenant_id, account_id);
CREATE INDEX idx_appointments_tenant_resource ON fieldservice_appointments(tenant_id, assigned_resource_id);
CREATE INDEX idx_appointments_tenant_scheduled ON fieldservice_appointments(tenant_id, scheduled_start);
CREATE INDEX idx_appointments_tenant_priority ON fieldservice_appointments(tenant_id, priority);
CREATE INDEX idx_appointments_tenant_contract ON fieldservice_appointments(tenant_id, contract_id);
CREATE INDEX idx_appointments_tenant_active ON fieldservice_appointments(tenant_id, is_active);
```

### 2.2 Service Resources (Technicians)
```sql
CREATE TABLE fieldservice_resources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_code TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    resource_type TEXT NOT NULL DEFAULT 'TECHNICIAN'
        CHECK(resource_type IN ('TECHNICIAN','SUPERVISOR','CONTRACTOR','VEHICLE','EQUIPMENT')),
    status TEXT NOT NULL DEFAULT 'AVAILABLE'
        CHECK(status IN ('AVAILABLE','BUSY','ON_BREAK','OFF_DUTY','ON_LEAVE','UNAVAILABLE')),
    skills TEXT NOT NULL,
    certifications TEXT,
    home_base_latitude REAL,
    home_base_longitude REAL,
    home_base_address TEXT,
    max_daily_appointments INTEGER NOT NULL DEFAULT 8,
    max_travel_distance_km INTEGER NOT NULL DEFAULT 200,
    hourly_rate INTEGER NOT NULL DEFAULT 0,
    overtime_rate INTEGER NOT NULL DEFAULT 0,
    vehicle_id TEXT,
    region TEXT,
    hire_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, resource_code)
);

CREATE INDEX idx_resources_tenant_status ON fieldservice_resources(tenant_id, status);
CREATE INDEX idx_resources_tenant_type ON fieldservice_resources(tenant_id, resource_type);
CREATE INDEX idx_resources_tenant_skills ON fieldservice_resources(tenant_id, skills);
CREATE INDEX idx_resources_tenant_region ON fieldservice_resources(tenant_id, region);
CREATE INDEX idx_resources_tenant_active ON fieldservice_resources(tenant_id, is_active);
```

### 2.3 Resource Shifts
```sql
CREATE TABLE fieldservice_resource_shifts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    shift_date TEXT NOT NULL,
    shift_start TEXT NOT NULL,
    shift_end TEXT NOT NULL,
    break_start TEXT,
    break_end TEXT,
    shift_type TEXT NOT NULL DEFAULT 'REGULAR'
        CHECK(shift_type IN ('REGULAR','OVERTIME','ON_CALL','HOLIDAY','TRAINING')),
    availability_status TEXT NOT NULL DEFAULT 'AVAILABLE'
        CHECK(availability_status IN ('AVAILABLE','PARTIALLY_AVAILABLE','UNAVAILABLE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, resource_id, shift_date)
);

CREATE INDEX idx_shifts_tenant_resource ON fieldservice_resource_shifts(tenant_id, resource_id);
CREATE INDEX idx_shifts_tenant_date ON fieldservice_resource_shifts(tenant_id, shift_date);
CREATE INDEX idx_shifts_tenant_status ON fieldservice_resource_shifts(tenant_id, availability_status);
CREATE INDEX idx_shifts_tenant_active ON fieldservice_resource_shifts(tenant_id, is_active);
```

### 2.4 Service Tasks
```sql
CREATE TABLE fieldservice_service_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appointment_id TEXT NOT NULL,
    task_number TEXT NOT NULL,
    task_name TEXT NOT NULL,
    task_type TEXT NOT NULL
        CHECK(task_type IN ('DIAGNOSTIC','REPAIR','REPLACEMENT','INSTALLATION','INSPECTION','TESTING','CLEANING','CALIBRATION','DOCUMENTATION')),
    description TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','SKIPPED','DEFERRED')),
    estimated_minutes INTEGER NOT NULL DEFAULT 0,
    actual_minutes INTEGER,
    task_order INTEGER NOT NULL DEFAULT 0,
    is_mandatory INTEGER NOT NULL DEFAULT 1,
    completion_notes TEXT,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, appointment_id, task_number)
);

CREATE INDEX idx_tasks_tenant_appointment ON fieldservice_service_tasks(tenant_id, appointment_id);
CREATE INDEX idx_tasks_tenant_status ON fieldservice_service_tasks(tenant_id, status);
CREATE INDEX idx_tasks_tenant_type ON fieldservice_service_tasks(tenant_id, task_type);
CREATE INDEX idx_tasks_tenant_active ON fieldservice_service_tasks(tenant_id, is_active);
```

### 2.5 Service Parts Needed
```sql
CREATE TABLE fieldservice_service_parts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appointment_id TEXT NOT NULL,
    task_id TEXT,
    item_id TEXT NOT NULL,
    item_code TEXT NOT NULL,
    item_name TEXT NOT NULL,
    quantity_needed INTEGER NOT NULL DEFAULT 1,
    quantity_used INTEGER NOT NULL DEFAULT 0,
    unit_cost INTEGER NOT NULL DEFAULT 0,
    total_cost INTEGER NOT NULL DEFAULT 0,
    source_warehouse_id TEXT,
    order_id TEXT,
    order_status TEXT
        CHECK(order_status IN ('NOT_ORDERED','ORDERED','SHIPPED','DELIVERED','CANCELLED')),
    is_returned INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_parts_tenant_appointment ON fieldservice_service_parts(tenant_id, appointment_id);
CREATE INDEX idx_parts_tenant_item ON fieldservice_service_parts(tenant_id, item_id);
CREATE INDEX idx_parts_tenant_order_status ON fieldservice_service_parts(tenant_id, order_status);
CREATE INDEX idx_parts_tenant_warehouse ON fieldservice_service_parts(tenant_id, source_warehouse_id);
CREATE INDEX idx_parts_tenant_active ON fieldservice_service_parts(tenant_id, is_active);
```

### 2.6 Dispatch Rules
```sql
CREATE TABLE fieldservice_dispatch_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('SKILL_MATCH','PROXIMITY','AVAILABILITY','WORKLOAD_BALANCE','PRIORITY_BASED','CONTRACT_BASED','CUSTOM')),
    rule_condition TEXT NOT NULL,
    rule_priority INTEGER NOT NULL DEFAULT 0,
    weight REAL NOT NULL DEFAULT 1.0,
    service_type_filter TEXT,
    region_filter TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_dispatch_rules_tenant_type ON fieldservice_dispatch_rules(tenant_id, rule_type);
CREATE INDEX idx_dispatch_rules_tenant_priority ON fieldservice_dispatch_rules(tenant_id, rule_priority);
CREATE INDEX idx_dispatch_rules_tenant_active ON fieldservice_dispatch_rules(tenant_id, is_active);
```

### 2.7 Route Optimizations
```sql
CREATE TABLE fieldservice_route_optimizations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    route_date TEXT NOT NULL,
    optimization_type TEXT NOT NULL DEFAULT 'MINIMIZE_TRAVEL'
        CHECK(optimization_type IN ('MINIMIZE_TRAVEL','MAXIMIZE_APPOINTMENTS','BALANCE_WORKLOAD','PRIORITY_FIRST','CUSTOM')),
    total_distance_km REAL NOT NULL DEFAULT 0.0,
    total_travel_time_minutes INTEGER NOT NULL DEFAULT 0,
    total_service_time_minutes INTEGER NOT NULL DEFAULT 0,
    appointment_count INTEGER NOT NULL DEFAULT 0,
    optimization_score REAL NOT NULL DEFAULT 0.0,
    route_data TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ACCEPTED','IN_PROGRESS','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, resource_id, route_date)
);

CREATE INDEX idx_route_opt_tenant_resource ON fieldservice_route_optimizations(tenant_id, resource_id);
CREATE INDEX idx_route_opt_tenant_date ON fieldservice_route_optimizations(tenant_id, route_date);
CREATE INDEX idx_route_opt_tenant_status ON fieldservice_route_optimizations(tenant_id, status);
CREATE INDEX idx_route_opt_tenant_active ON fieldservice_route_optimizations(tenant_id, is_active);
```

### 2.8 Service Completions
```sql
CREATE TABLE fieldservice_completions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appointment_id TEXT NOT NULL,
    completion_date TEXT NOT NULL,
    resolution_code TEXT NOT NULL
        CHECK(resolution_code IN ('RESOLVED','PARTIAL','ESCALATED','DEFERRED','NOT_REPRODUCIBLE','REPLACED')),
    resolution_description TEXT NOT NULL,
    root_cause TEXT,
    corrective_action TEXT,
    time_on_site_minutes INTEGER NOT NULL DEFAULT 0,
    travel_time_minutes INTEGER NOT NULL DEFAULT 0,
    parts_total_cost INTEGER NOT NULL DEFAULT 0,
    labor_total_cost INTEGER NOT NULL DEFAULT 0,
    total_cost INTEGER NOT NULL DEFAULT 0,
    customer_signature_name TEXT,
    customer_signature_date TEXT,
    customer_accepted INTEGER NOT NULL DEFAULT 0,
    follow_up_required INTEGER NOT NULL DEFAULT 0,
    follow_up_notes TEXT,
    photos TEXT,
    documents TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, appointment_id)
);

CREATE INDEX idx_completions_tenant_date ON fieldservice_completions(tenant_id, completion_date);
CREATE INDEX idx_completions_tenant_resolution ON fieldservice_completions(tenant_id, resolution_code);
CREATE INDEX idx_completions_tenant_appointment ON fieldservice_completions(tenant_id, appointment_id);
CREATE INDEX idx_completions_tenant_active ON fieldservice_completions(tenant_id, is_active);
```

### 2.9 Customer Feedback
```sql
CREATE TABLE fieldservice_customer_feedback (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appointment_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    contact_id TEXT,
    resource_id TEXT NOT NULL,
    overall_rating INTEGER NOT NULL
        CHECK(overall_rating >= 1 AND overall_rating <= 5),
    punctuality_rating INTEGER CHECK(punctuality_rating >= 1 AND punctuality_rating <= 5),
    professionalism_rating INTEGER CHECK(professionalism_rating >= 1 AND professionalism_rating <= 5),
    quality_rating INTEGER CHECK(quality_rating >= 1 AND quality_rating <= 5),
    communication_rating INTEGER CHECK(communication_rating >= 1 AND communication_rating <= 5),
    would_recommend INTEGER,
    comments TEXT,
    feedback_date TEXT NOT NULL,
    feedback_source TEXT NOT NULL DEFAULT 'POST_SERVICE'
        CHECK(feedback_source IN ('POST_SERVICE','EMAIL','PHONE','WEB','IN_APP')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_feedback_tenant_appointment ON fieldservice_customer_feedback(tenant_id, appointment_id);
CREATE INDEX idx_feedback_tenant_account ON fieldservice_customer_feedback(tenant_id, account_id);
CREATE INDEX idx_feedback_tenant_resource ON fieldservice_customer_feedback(tenant_id, resource_id);
CREATE INDEX idx_feedback_tenant_date ON fieldservice_customer_feedback(tenant_id, feedback_date);
CREATE INDEX idx_feedback_tenant_rating ON fieldservice_customer_feedback(tenant_id, overall_rating);
CREATE INDEX idx_feedback_tenant_active ON fieldservice_customer_feedback(tenant_id, is_active);
```

### 2.10 Service Contracts
```sql
CREATE TABLE fieldservice_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,
    contract_name TEXT NOT NULL,
    account_id TEXT NOT NULL,
    contract_type TEXT NOT NULL
        CHECK(contract_type IN ('WARRANTY','MAINTENANCE','SLA','PREVENTIVE','COMPREHENSIVE','ON_DEMAND')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','EXPIRED','TERMINATED','RENEWED')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    auto_renew INTEGER NOT NULL DEFAULT 0,
    renewal_period_days INTEGER NOT NULL DEFAULT 30,
    response_time_sla_minutes INTEGER,
    resolution_time_sla_minutes INTEGER,
    covered_assets TEXT,
    covered_service_types TEXT,
    max_visits_per_year INTEGER,
    max_parts_value INTEGER,
    annual_contract_value INTEGER NOT NULL DEFAULT 0,
    billing_frequency TEXT NOT NULL DEFAULT 'ANNUALLY'
        CHECK(billing_frequency IN ('MONTHLY','QUARTERLY','ANNUALLY','ONE_TIME')),
    terms_and_conditions TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_contracts_tenant_account ON fieldservice_contracts(tenant_id, account_id);
CREATE INDEX idx_contracts_tenant_status ON fieldservice_contracts(tenant_id, status);
CREATE INDEX idx_contracts_tenant_type ON fieldservice_contracts(tenant_id, contract_type);
CREATE INDEX idx_contracts_tenant_dates ON fieldservice_contracts(tenant_id, start_date, end_date);
CREATE INDEX idx_contracts_tenant_active ON fieldservice_contracts(tenant_id, is_active);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/appointments` | Create a service appointment |
| GET | `/api/v1/appointments` | List appointments with filtering and pagination |
| GET | `/api/v1/appointments/{id}` | Get appointment by ID with tasks and parts |
| PUT | `/api/v1/appointments/{id}` | Update appointment details |
| DELETE | `/api/v1/appointments/{id}` | Cancel an appointment |
| PATCH | `/api/v1/appointments/{id}/dispatch` | Dispatch appointment to a resource |
| PATCH | `/api/v1/appointments/{id}/arrive` | Record resource arrival on site |
| PATCH | `/api/v1/appointments/{id}/start` | Start service work on appointment |
| PATCH | `/api/v1/appointments/{id}/complete` | Complete an appointment |
| PATCH | `/api/v1/appointments/{id}/reschedule` | Reschedule an appointment |
| GET | `/api/v1/appointments/by-resource/{resourceId}` | List appointments for a resource |
| GET | `/api/v1/appointments/by-account/{accountId}` | List appointments for an account |
| GET | `/api/v1/appointments/by-date-range` | List appointments within a date range |
| POST | `/api/v1/resources` | Create a service resource (technician) |
| GET | `/api/v1/resources` | List resources with filtering |
| GET | `/api/v1/resources/{id}` | Get resource by ID with skills and availability |
| PUT | `/api/v1/resources/{id}` | Update resource details |
| DELETE | `/api/v1/resources/{id}` | Deactivate a resource |
| GET | `/api/v1/resources/{id}/availability` | Check resource availability for a date range |
| GET | `/api/v1/resources/nearby` | Find nearby resources by coordinates |
| POST | `/api/v1/resources/{id}/shifts` | Create a resource shift |
| GET | `/api/v1/resources/{id}/shifts` | List shifts for a resource |
| PUT | `/api/v1/shifts/{id}` | Update a shift |
| DELETE | `/api/v1/shifts/{id}` | Delete a shift |
| GET | `/api/v1/shifts/calendar` | Get shift calendar view |
| POST | `/api/v1/appointments/{appointmentId}/tasks` | Add a task to an appointment |
| GET | `/api/v1/appointments/{appointmentId}/tasks` | List tasks for an appointment |
| GET | `/api/v1/tasks/{id}` | Get task by ID |
| PUT | `/api/v1/tasks/{id}` | Update a task |
| PATCH | `/api/v1/tasks/{id}/complete` | Mark a task as completed |
| DELETE | `/api/v1/tasks/{id}` | Remove a task |
| POST | `/api/v1/appointments/{appointmentId}/parts` | Add a part requirement to an appointment |
| GET | `/api/v1/appointments/{appointmentId}/parts` | List parts needed for an appointment |
| PUT | `/api/v1/parts/{id}` | Update a parts entry |
| DELETE | `/api/v1/parts/{id}` | Remove a parts entry |
| POST | `/api/v1/parts/{id}/order` | Order a part from inventory |
| POST | `/api/v1/dispatch-rules` | Create a dispatch rule |
| GET | `/api/v1/dispatch-rules` | List dispatch rules |
| PUT | `/api/v1/dispatch-rules/{id}` | Update a dispatch rule |
| DELETE | `/api/v1/dispatch-rules/{id}` | Remove a dispatch rule |
| POST | `/api/v1/routes/optimize` | Optimize routes for a date and resource set |
| GET | `/api/v1/routes/{id}` | Get optimized route details |
| GET | `/api/v1/routes/by-resource/{resourceId}` | Get routes for a resource |
| POST | `/api/v1/routes/{id}/accept` | Accept an optimized route plan |
| GET | `/api/v1/completions/{id}` | Get service completion details |
| GET | `/api/v1/completions/by-appointment/{appointmentId}` | Get completion for an appointment |
| POST | `/api/v1/feedback` | Submit customer feedback |
| GET | `/api/v1/feedback` | List feedback with filtering |
| GET | `/api/v1/feedback/{id}` | Get feedback by ID |
| GET | `/api/v1/feedback/summary` | Get feedback summary statistics |
| POST | `/api/v1/contracts` | Create a service contract |
| GET | `/api/v1/contracts` | List contracts with filtering |
| GET | `/api/v1/contracts/{id}` | Get contract with coverage details |
| PUT | `/api/v1/contracts/{id}` | Update contract details |
| DELETE | `/api/v1/contracts/{id}` | Terminate a contract |
| POST | `/api/v1/contracts/{id}/renew` | Renew a contract |
| GET | `/api/v1/contracts/{id}/entitlements` | Check entitlements for a contract |
| GET | `/api/v1/contracts/expiring` | List contracts nearing expiration |
| GET | `/api/v1/dashboard/schedule` | Get scheduling dashboard data |
| GET | `/api/v1/dashboard/utilization` | Get resource utilization metrics |

---

## 4. Business Rules

1. **Appointment Scheduling**: An appointment MUST have a scheduled start and end time. The scheduled end MUST be after the scheduled start. The scheduled duration MUST match the difference between start and end in minutes within a tolerance of 1 minute.

2. **Resource Assignment**: A resource MUST be assigned to an appointment before it can be dispatched. The assigned resource MUST have `AVAILABLE` or `BUSY` status. Resources on `OFF_DUTY`, `ON_LEAVE`, or `UNAVAILABLE` status MUST NOT be assigned new appointments.

3. **Skill Matching**: When a dispatch rule of type `SKILL_MATCH` is active, the system MUST verify that the assigned resource possesses the required skills for the appointment's service type. A warning SHOULD be generated when assigning a resource without the recommended certifications.

4. **Dispatch Workflow**: An appointment MUST follow the status progression: `SCHEDULED` -> `DISPATCHED` -> `EN_ROUTE` -> `ON_SITE` -> `IN_PROGRESS` -> `COMPLETED`. Status transitions MAY skip intermediate states (e.g., `DISPATCHED` directly to `IN_PROGRESS`) but MUST NOT go backward except to `RESCHEDULED` or `CANCELLED`.

5. **SLA Compliance**: For appointments linked to a contract with SLA terms, the system MUST track the time from appointment creation to dispatch (`response_time`) and from dispatch to completion (`resolution_time`). SLA breaches MUST be flagged and SHOULD trigger escalation notifications.

6. **Route Optimization**: Route optimization MUST consider all confirmed appointments for the day, resource shift schedules, appointment time windows, and travel time estimates. The optimized route SHOULD minimize total travel time while respecting all appointment scheduled start times.

7. **Parts Ordering**: Parts needed for an appointment MUST be ordered from the nearest available warehouse. If the required quantity is not available in any warehouse, the system MUST create a backorder and SHOULD suggest alternative parts or reschedule the appointment.

8. **Service Completion**: A service completion record MUST be created for every appointment that reaches `COMPLETED` status. The completion MUST include a `resolution_code`, `resolution_description`, and time tracking. If parts were used, the `parts_total_cost` MUST reflect actual parts consumed.

9. **Contract Entitlement Check**: Before creating an appointment linked to a contract, the system MUST verify that the contract is `ACTIVE`, the current date falls within the contract period, the service type is covered, and the visit limit (if any) has not been exceeded.

10. **Resource Workload Balance**: A resource MUST NOT be assigned more appointments in a day than their `max_daily_appointments` limit. The system SHOULD distribute appointments evenly across available resources with equivalent skills.

11. **Shift Validation**: A resource shift MUST NOT overlap with another active shift for the same resource on the same date. Shifts MUST fall within valid time ranges (00:00 to 23:59). Resources without a shift for a given date SHOULD be treated as unavailable.

12. **Task Completion Order**: Tasks with `is_mandatory = 1` MUST be completed before an appointment can be marked as `COMPLETED`. Tasks SHOULD be completed in ascending `task_order`, but technicians MAY complete them out of order unless the task explicitly requires sequential completion.

13. **Customer Feedback Timing**: Customer feedback SHOULD be collected within 7 days of appointment completion. The system MUST prevent duplicate feedback submissions for the same appointment from the same contact.

14. **Contract Auto-Renewal**: Contracts with `auto_renew = 1` SHOULD generate a renewal notification at least `renewal_period_days` before the contract end date. The renewed contract MUST inherit all terms from the expiring contract with updated start and end dates.

15. **Emergency Priority Handling**: Appointments with `CRITICAL` priority MUST bypass standard scheduling constraints and be dispatched to the nearest available qualified resource immediately. The system SHOULD automatically reassign lower-priority appointments if necessary to accommodate critical cases.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fieldservice;

option go_package = "github.com/fusion/fieldservicepb";

service FieldServiceService {
    // Appointments
    rpc CreateAppointment(CreateAppointmentRequest) returns (AppointmentResponse);
    rpc GetAppointment(GetAppointmentRequest) returns (AppointmentResponse);
    rpc ListAppointments(ListAppointmentsRequest) returns (ListAppointmentsResponse);
    rpc UpdateAppointment(UpdateAppointmentRequest) returns (AppointmentResponse);
    rpc CancelAppointment(CancelAppointmentRequest) returns (AppointmentResponse);
    rpc DispatchAppointment(DispatchAppointmentRequest) returns (AppointmentResponse);
    rpc ArriveOnSite(ArriveOnSiteRequest) returns (AppointmentResponse);
    rpc StartService(StartServiceRequest) returns (AppointmentResponse);
    rpc CompleteAppointment(CompleteAppointmentRequest) returns (AppointmentResponse);
    rpc RescheduleAppointment(RescheduleAppointmentRequest) returns (AppointmentResponse);
    rpc GetAppointmentsByResource(GetAppointmentsByResourceRequest) returns (ListAppointmentsResponse);
    rpc GetAppointmentsByAccount(GetAppointmentsByAccountRequest) returns (ListAppointmentsResponse);

    // Resources
    rpc CreateResource(CreateResourceRequest) returns (ResourceResponse);
    rpc GetResource(GetResourceRequest) returns (ResourceResponse);
    rpc ListResources(ListResourcesRequest) returns (ListResourcesResponse);
    rpc UpdateResource(UpdateResourceRequest) returns (ResourceResponse);
    rpc DeleteResource(DeleteResourceRequest) returns (DeleteResponse);
    rpc CheckAvailability(CheckAvailabilityRequest) returns (AvailabilityResponse);
    rpc FindNearbyResources(FindNearbyResourcesRequest) returns (ListResourcesResponse);

    // Shifts
    rpc CreateShift(CreateShiftRequest) returns (ShiftResponse);
    rpc ListShifts(ListShiftsRequest) returns (ListShiftsResponse);
    rpc UpdateShift(UpdateShiftRequest) returns (ShiftResponse);
    rpc DeleteShift(DeleteShiftRequest) returns (DeleteResponse);
    rpc GetShiftCalendar(GetShiftCalendarRequest) returns (ShiftCalendarResponse);

    // Tasks
    rpc CreateTask(CreateTaskRequest) returns (TaskResponse);
    rpc ListTasks(ListTasksRequest) returns (ListTasksResponse);
    rpc GetTask(GetTaskRequest) returns (TaskResponse);
    rpc UpdateTask(UpdateTaskRequest) returns (TaskResponse);
    rpc CompleteTask(CompleteTaskRequest) returns (TaskResponse);
    rpc DeleteTask(DeleteTaskRequest) returns (DeleteResponse);

    // Parts
    rpc AddServicePart(AddServicePartRequest) returns (ServicePartResponse);
    rpc ListServiceParts(ListServicePartsRequest) returns (ListServicePartsResponse);
    rpc UpdateServicePart(UpdateServicePartRequest) returns (ServicePartResponse);
    rpc RemoveServicePart(RemoveServicePartRequest) returns (DeleteResponse);
    rpc OrderPart(OrderPartRequest) returns (ServicePartResponse);

    // Dispatch Rules
    rpc CreateDispatchRule(CreateDispatchRuleRequest) returns (DispatchRuleResponse);
    rpc ListDispatchRules(ListDispatchRulesRequest) returns (ListDispatchRulesResponse);
    rpc UpdateDispatchRule(UpdateDispatchRuleRequest) returns (DispatchRuleResponse);
    rpc DeleteDispatchRule(DeleteDispatchRuleRequest) returns (DeleteResponse);

    // Route Optimization
    rpc OptimizeRoutes(OptimizeRoutesRequest) returns (OptimizeRoutesResponse);
    rpc GetRoute(GetRouteRequest) returns (RouteResponse);
    rpc GetRoutesByResource(GetRoutesByResourceRequest) returns (ListRoutesResponse);
    rpc AcceptRoute(AcceptRouteRequest) returns (RouteResponse);

    // Completions
    rpc GetCompletion(GetCompletionRequest) returns (CompletionResponse);
    rpc GetCompletionByAppointment(GetCompletionByAppointmentRequest) returns (CompletionResponse);

    // Customer Feedback
    rpc SubmitFeedback(SubmitFeedbackRequest) returns (FeedbackResponse);
    rpc ListFeedback(ListFeedbackRequest) returns (ListFeedbackResponse);
    rpc GetFeedback(GetFeedbackRequest) returns (FeedbackResponse);
    rpc GetFeedbackSummary(GetFeedbackSummaryRequest) returns (FeedbackSummaryResponse);

    // Contracts
    rpc CreateContract(CreateContractRequest) returns (ContractResponse);
    rpc GetContract(GetContractRequest) returns (ContractResponse);
    rpc ListContracts(ListContractsRequest) returns (ListContractsResponse);
    rpc UpdateContract(UpdateContractRequest) returns (ContractResponse);
    rpc TerminateContract(TerminateContractRequest) returns (ContractResponse);
    rpc RenewContract(RenewContractRequest) returns (ContractResponse);
    rpc CheckEntitlements(CheckEntitlementsRequest) returns (EntitlementsResponse);
    rpc GetExpiringContracts(GetExpiringContractsRequest) returns (ListContractsResponse);
}

message Appointment {
    string id = 1;
    string tenant_id = 2;
    string appointment_number = 3;
    string account_id = 4;
    string contact_id = 5;
    string asset_id = 6;
    string contract_id = 7;
    string work_order_id = 8;
    string service_type = 9;
    string priority = 10;
    string status = 11;
    string description = 12;
    string scheduled_start = 13;
    string scheduled_end = 14;
    string actual_start = 15;
    string actual_end = 16;
    int32 estimated_duration_minutes = 17;
    int32 actual_duration_minutes = 18;
    int32 travel_time_minutes = 19;
    string assigned_resource_id = 20;
    string location_address = 21;
    string location_city = 22;
    string location_state = 23;
    double latitude = 24;
    double longitude = 25;
    string created_at = 26;
    string updated_at = 27;
    int32 version = 28;
}

message Resource {
    string id = 1;
    string tenant_id = 2;
    string resource_code = 3;
    string first_name = 4;
    string last_name = 5;
    string email = 6;
    string phone = 7;
    string resource_type = 8;
    string status = 9;
    string skills = 10;
    string certifications = 11;
    double home_base_latitude = 12;
    double home_base_longitude = 13;
    int32 max_daily_appointments = 14;
    int32 max_travel_distance_km = 15;
    int64 hourly_rate = 16;
    string region = 17;
    string created_at = 18;
    string updated_at = 19;
    int32 version = 20;
}

message Shift {
    string id = 1;
    string tenant_id = 2;
    string resource_id = 3;
    string shift_date = 4;
    string shift_start = 5;
    string shift_end = 6;
    string shift_type = 7;
    string availability_status = 8;
    string created_at = 9;
    string updated_at = 10;
    int32 version = 11;
}

message ServiceTask {
    string id = 1;
    string tenant_id = 2;
    string appointment_id = 3;
    string task_number = 4;
    string task_name = 5;
    string task_type = 6;
    string description = 7;
    string status = 8;
    int32 estimated_minutes = 9;
    int32 actual_minutes = 10;
    int32 task_order = 11;
    bool is_mandatory = 12;
    string completion_notes = 13;
    string completed_at = 14;
    string created_at = 15;
    string updated_at = 16;
    int32 version = 17;
}

message ServicePart {
    string id = 1;
    string tenant_id = 2;
    string appointment_id = 3;
    string task_id = 4;
    string item_id = 5;
    string item_code = 6;
    string item_name = 7;
    int32 quantity_needed = 8;
    int32 quantity_used = 9;
    int64 unit_cost = 10;
    int64 total_cost = 11;
    string source_warehouse_id = 12;
    string order_id = 13;
    string order_status = 14;
    string created_at = 15;
    string updated_at = 16;
    int32 version = 17;
}

message DispatchRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string rule_condition = 5;
    int32 rule_priority = 6;
    double weight = 7;
    string service_type_filter = 8;
    string region_filter = 9;
    bool is_default = 10;
    string created_at = 11;
    string updated_at = 12;
    int32 version = 13;
}

message RouteOptimization {
    string id = 1;
    string tenant_id = 2;
    string resource_id = 3;
    string route_date = 4;
    string optimization_type = 5;
    double total_distance_km = 6;
    int32 total_travel_time_minutes = 7;
    int32 total_service_time_minutes = 8;
    int32 appointment_count = 9;
    double optimization_score = 10;
    string route_data = 11;
    string status = 12;
    string created_at = 13;
    string updated_at = 14;
    int32 version = 15;
}

message Completion {
    string id = 1;
    string tenant_id = 2;
    string appointment_id = 3;
    string completion_date = 4;
    string resolution_code = 5;
    string resolution_description = 6;
    string root_cause = 7;
    string corrective_action = 8;
    int32 time_on_site_minutes = 9;
    int32 travel_time_minutes = 10;
    int64 parts_total_cost = 11;
    int64 labor_total_cost = 12;
    int64 total_cost = 13;
    bool customer_accepted = 14;
    bool follow_up_required = 15;
    string created_at = 16;
    string updated_at = 17;
    int32 version = 18;
}

message CustomerFeedback {
    string id = 1;
    string tenant_id = 2;
    string appointment_id = 3;
    string account_id = 4;
    string contact_id = 5;
    string resource_id = 6;
    int32 overall_rating = 7;
    int32 punctuality_rating = 8;
    int32 professionalism_rating = 9;
    int32 quality_rating = 10;
    int32 communication_rating = 11;
    bool would_recommend = 12;
    string comments = 13;
    string feedback_date = 14;
    string created_at = 15;
    string updated_at = 16;
    int32 version = 17;
}

message ServiceContract {
    string id = 1;
    string tenant_id = 2;
    string contract_number = 3;
    string contract_name = 4;
    string account_id = 5;
    string contract_type = 6;
    string status = 7;
    string start_date = 8;
    string end_date = 9;
    bool auto_renew = 10;
    int32 renewal_period_days = 11;
    int32 response_time_sla_minutes = 12;
    int32 resolution_time_sla_minutes = 13;
    string covered_service_types = 14;
    int32 max_visits_per_year = 15;
    int64 annual_contract_value = 16;
    string billing_frequency = 17;
    string created_at = 18;
    string updated_at = 19;
    int32 version = 20;
}

// Request/Response messages - Appointments
message CreateAppointmentRequest { Appointment appointment = 1; }
message GetAppointmentRequest { string tenant_id = 1; string id = 2; }
message ListAppointmentsRequest {
    string tenant_id = 1;
    string status = 2;
    string account_id = 3;
    string resource_id = 4;
    string service_type = 5;
    string priority = 6;
    string date_from = 7;
    string date_to = 8;
    int32 page = 9;
    int32 page_size = 10;
}
message UpdateAppointmentRequest { Appointment appointment = 1; }
message CancelAppointmentRequest { string tenant_id = 1; string id = 2; string reason = 3; }
message DispatchAppointmentRequest { string tenant_id = 1; string id = 2; string resource_id = 3; }
message ArriveOnSiteRequest { string tenant_id = 1; string id = 2; double latitude = 3; double longitude = 4; }
message StartServiceRequest { string tenant_id = 1; string id = 2; }
message CompleteAppointmentRequest {
    string tenant_id = 1;
    string id = 2;
    string resolution_code = 3;
    string resolution_description = 4;
    int32 time_on_site_minutes = 5;
}
message RescheduleAppointmentRequest {
    string tenant_id = 1;
    string id = 2;
    string new_scheduled_start = 3;
    string new_scheduled_end = 4;
    string reason = 5;
}
message GetAppointmentsByResourceRequest { string tenant_id = 1; string resource_id = 2; string date = 3; }
message GetAppointmentsByAccountRequest { string tenant_id = 1; string account_id = 2; string status = 3; }
message AppointmentResponse { Appointment appointment = 1; }
message ListAppointmentsResponse { repeated Appointment appointments = 1; int32 total = 2; }

// Resources
message CreateResourceRequest { Resource resource = 1; }
message GetResourceRequest { string tenant_id = 1; string id = 2; }
message ListResourcesRequest {
    string tenant_id = 1;
    string resource_type = 2;
    string status = 3;
    string skills = 4;
    string region = 5;
    int32 page = 6;
    int32 page_size = 7;
}
message UpdateResourceRequest { Resource resource = 1; }
message DeleteResourceRequest { string tenant_id = 1; string id = 2; }
message CheckAvailabilityRequest {
    string tenant_id = 1;
    string resource_id = 2;
    string date_from = 3;
    string date_to = 4;
}
message FindNearbyResourcesRequest {
    string tenant_id = 1;
    double latitude = 2;
    double longitude = 3;
    double radius_km = 4;
    string required_skills = 5;
}
message ResourceResponse { Resource resource = 1; }
message ListResourcesResponse { repeated Resource resources = 1; int32 total = 2; }
message AvailabilityResponse {
    string resource_id = 1;
    repeated string available_dates = 2;
    repeated string unavailable_dates = 3;
}
message NearbyResource { Resource resource = 1; double distance_km = 2; int64 eta_minutes = 3; }
message FindNearbyResourcesResponse { repeated NearbyResource resources = 1; }

// Shifts
message CreateShiftRequest { Shift shift = 1; }
message ListShiftsRequest { string tenant_id = 1; string resource_id = 2; string date_from = 3; string date_to = 4; }
message UpdateShiftRequest { Shift shift = 1; }
message DeleteShiftRequest { string tenant_id = 1; string id = 2; }
message GetShiftCalendarRequest { string tenant_id = 1; string resource_id = 2; int32 year = 3; int32 month = 4; }
message ShiftResponse { Shift shift = 1; }
message ListShiftsResponse { repeated Shift shifts = 1; int32 total = 2; }
message ShiftCalendarDay { string date = 1; string shift_id = 2; string shift_start = 3; string shift_end = 4; string availability_status = 5; }
message ShiftCalendarResponse { repeated ShiftCalendarDay days = 1; }

// Tasks
message CreateTaskRequest { ServiceTask task = 1; }
message ListTasksRequest { string tenant_id = 1; string appointment_id = 2; string status = 3; }
message GetTaskRequest { string tenant_id = 1; string id = 2; }
message UpdateTaskRequest { ServiceTask task = 1; }
message CompleteTaskRequest { string tenant_id = 1; string id = 2; string completion_notes = 3; int32 actual_minutes = 4; }
message DeleteTaskRequest { string tenant_id = 1; string id = 2; }
message TaskResponse { ServiceTask task = 1; }
message ListTasksResponse { repeated ServiceTask tasks = 1; int32 total = 2; }

// Parts
message AddServicePartRequest { ServicePart part = 1; }
message ListServicePartsRequest { string tenant_id = 1; string appointment_id = 2; string order_status = 3; }
message UpdateServicePartRequest { ServicePart part = 1; }
message RemoveServicePartRequest { string tenant_id = 1; string id = 2; }
message OrderPartRequest { string tenant_id = 1; string id = 2; string warehouse_id = 3; }
message ServicePartResponse { ServicePart part = 1; }
message ListServicePartsResponse { repeated ServicePart parts = 1; int32 total = 2; }

// Dispatch Rules
message CreateDispatchRuleRequest { DispatchRule rule = 1; }
message ListDispatchRulesRequest { string tenant_id = 1; string rule_type = 2; }
message UpdateDispatchRuleRequest { DispatchRule rule = 1; }
message DeleteDispatchRuleRequest { string tenant_id = 1; string id = 2; }
message DispatchRuleResponse { DispatchRule rule = 1; }
message ListDispatchRulesResponse { repeated DispatchRule rules = 1; int32 total = 2; }

// Route Optimization
message OptimizeRoutesRequest {
    string tenant_id = 1;
    string route_date = 2;
    repeated string resource_ids = 3;
    string optimization_type = 4;
}
message GetRouteRequest { string tenant_id = 1; string id = 2; }
message GetRoutesByResourceRequest { string tenant_id = 1; string resource_id = 2; string route_date = 3; }
message AcceptRouteRequest { string tenant_id = 1; string id = 2; }
message RouteResponse { RouteOptimization route = 1; }
message ListRoutesResponse { repeated RouteOptimization routes = 1; int32 total = 2; }
message OptimizeRoutesResponse { repeated RouteOptimization routes = 1; }

// Completions
message GetCompletionRequest { string tenant_id = 1; string id = 2; }
message GetCompletionByAppointmentRequest { string tenant_id = 1; string appointment_id = 2; }
message CompletionResponse { Completion completion = 1; }

// Customer Feedback
message SubmitFeedbackRequest { CustomerFeedback feedback = 1; }
message ListFeedbackRequest {
    string tenant_id = 1;
    string account_id = 2;
    string resource_id = 3;
    int32 min_rating = 4;
    int32 max_rating = 5;
    string date_from = 6;
    string date_to = 7;
    int32 page = 8;
    int32 page_size = 9;
}
message GetFeedbackRequest { string tenant_id = 1; string id = 2; }
message GetFeedbackSummaryRequest { string tenant_id = 1; string resource_id = 2; string date_from = 3; string date_to = 4; }
message FeedbackResponse { CustomerFeedback feedback = 1; }
message ListFeedbackResponse { repeated CustomerFeedback feedback = 1; int32 total = 2; }
message FeedbackSummary {
    double average_overall_rating = 1;
    double average_punctuality_rating = 2;
    double average_professionalism_rating = 3;
    double average_quality_rating = 4;
    double average_communication_rating = 5;
    int32 total_responses = 6;
    double recommend_percentage = 7;
}
message FeedbackSummaryResponse { FeedbackSummary summary = 1; }

// Contracts
message CreateContractRequest { ServiceContract contract = 1; }
message GetContractRequest { string tenant_id = 1; string id = 2; }
message ListContractsRequest {
    string tenant_id = 1;
    string account_id = 2;
    string status = 3;
    string contract_type = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message UpdateContractRequest { ServiceContract contract = 1; }
message TerminateContractRequest { string tenant_id = 1; string id = 2; string reason = 3; }
message RenewContractRequest { string tenant_id = 1; string id = 2; string new_end_date = 3; int64 new_annual_value = 4; }
message CheckEntitlementsRequest { string tenant_id = 1; string contract_id = 2; string service_type = 3; string asset_id = 4; }
message GetExpiringContractsRequest { string tenant_id = 1; int32 days_until_expiry = 2; }
message ContractResponse { ServiceContract contract = 1; }
message ListContractsResponse { repeated ServiceContract contracts = 1; int32 total = 2; }
message EntitlementDetail { string service_type = 1; bool is_covered = 2; int32 visits_remaining = 3; int64 parts_value_remaining = 4; }
message EntitlementsResponse {
    bool is_entitled = 1;
    string contract_status = 2;
    repeated EntitlementDetail entitlements = 3;
}

message DeleteResponse { bool success = 1; }
```

---

## 6. Inter-Service Integration

| Integration | Service | Direction | Purpose |
|-------------|---------|-----------|---------|
| Inventory | `inv-service` | Outbound | Request parts availability check and reservation for service appointments |
| Inventory | `inv-service` | Inbound | Receive parts availability status, reservation confirmations, and shipment tracking updates |
| Transportation Management | `tms-service` | Outbound | Send technician dispatch details for fleet tracking and route planning |
| Transportation Management | `tms-service` | Inbound | Receive real-time vehicle location, ETA updates, and fleet availability data |
| Enterprise Asset Management | `eam-service` | Outbound | Send service completion details, parts used, and maintenance history updates |
| Enterprise Asset Management | `eam-service` | Inbound | Receive asset details, maintenance schedules, warranty status, and service history |
| Order Management | `om-service` | Outbound | Create parts orders for service appointments requiring inventory |
| Order Management | `om-service` | Inbound | Receive order confirmations, shipment status, and delivery tracking |
| Sales Automation | `sales-service` | Inbound | Receive account and contact details, service contract references, and customer data |
| Sales Automation | `sales-service` | Outbound | Publish service completion events for customer 360 and upsell opportunities |
| General Ledger | `gl-service` | Outbound | Post service revenue, parts cost, and labor cost journal entries |
| General Ledger | `gl-service` | Inbound | Receive GL period status to prevent modifications in closed periods |
| Identity Service | `identity-service` | Inbound | Validate resource (technician) credentials, certifications, and role permissions |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `AppointmentCreated` | `{ tenant_id, appointment_id, appointment_number, account_id, service_type, priority, scheduled_start, scheduled_end, created_at }` | Published when a new service appointment is created |
| `AppointmentDispatched` | `{ tenant_id, appointment_id, appointment_number, resource_id, resource_name, dispatched_at }` | Published when an appointment is dispatched to a technician |
| `AppointmentArrived` | `{ tenant_id, appointment_id, resource_id, latitude, longitude, arrived_at }` | Published when a technician arrives at the service location |
| `AppointmentStarted` | `{ tenant_id, appointment_id, resource_id, actual_start }` | Published when service work begins on an appointment |
| `AppointmentCompleted` | `{ tenant_id, appointment_id, appointment_number, resource_id, resolution_code, actual_duration_minutes, total_cost, completed_at }` | Published when a service appointment is completed |
| `AppointmentRescheduled` | `{ tenant_id, appointment_id, old_scheduled_start, new_scheduled_start, reason, rescheduled_at }` | Published when an appointment is rescheduled |
| `AppointmentCancelled` | `{ tenant_id, appointment_id, appointment_number, reason, cancelled_at }` | Published when an appointment is cancelled |
| `PartsNeeded` | `{ tenant_id, appointment_id, parts: [{ item_id, item_code, quantity_needed, unit_cost }], needed_at }` | Published when parts are identified as needed for an appointment |
| `PartsOrdered` | `{ tenant_id, appointment_id, order_id, parts: [{ item_id, quantity, warehouse_id }], ordered_at }` | Published when parts are ordered from inventory for a service appointment |
| `ServiceCompleted` | `{ tenant_id, appointment_id, completion_id, resolution_code, resolution_description, parts_total_cost, labor_total_cost, total_cost, completed_at }` | Published with full service completion details including cost breakdown |
| `FeedbackReceived` | `{ tenant_id, appointment_id, resource_id, overall_rating, would_recommend, feedback_date }` | Published when customer feedback is submitted post-service |
| `ContractCreated` | `{ tenant_id, contract_id, contract_number, account_id, contract_type, start_date, end_date, annual_contract_value, created_at }` | Published when a new service contract is created |
| `ContractRenewed` | `{ tenant_id, contract_id, contract_number, old_end_date, new_end_date, renewed_at }` | Published when a service contract is renewed |
| `ContractExpiring` | `{ tenant_id, contract_id, contract_number, account_id, end_date, days_until_expiry }` | Published when a contract is nearing its expiration date |
| `SLABreached` | `{ tenant_id, appointment_id, contract_id, breach_type, sla_target_minutes, actual_minutes, breached_at }` | Published when an SLA target is breached for a contract-covered appointment |
| `RouteOptimized` | `{ tenant_id, route_id, resource_id, route_date, appointment_count, total_distance_km, total_travel_time_minutes, optimized_at }` | Published when a route optimization is completed |
