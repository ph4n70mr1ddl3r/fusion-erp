# 96 - Onboarding & Transitions Service Specification

## 1. Domain Overview

The Onboarding & Transitions service manages employee onboarding journeys, offboarding checklists, transfer transitions, and role change orientations. It supports configurable journey templates, task assignments, automated triggers, progress tracking, and new hire experience management to ensure smooth employee lifecycle transitions.

**Bounded Context:** Employee Lifecycle Transitions
**Service Name:** `onboarding-service`
**Database:** `data/onboarding.db`
**HTTP Port:** 8133 | **gRPC Port:** 9133

---

## 2. Database Schema

### 2.1 Journey Templates
```sql
CREATE TABLE journey_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    journey_type TEXT NOT NULL CHECK(journey_type IN ('ONBOARDING','OFFBOARDING','TRANSFER','ROLE_CHANGE','PROMOTION')),
    department_id TEXT,
    location_id TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    estimated_duration_days INTEGER,
    trigger_event TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, name)
);

CREATE INDEX idx_journey_templates_tenant_type ON journey_templates(tenant_id, journey_type);
CREATE INDEX idx_journey_templates_tenant_status ON journey_templates(tenant_id, status);
CREATE INDEX idx_journey_templates_tenant_dept ON journey_templates(tenant_id, department_id);
```

### 2.2 Journey Tasks
```sql
CREATE TABLE journey_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    task_name TEXT NOT NULL,
    task_description TEXT,
    task_category TEXT NOT NULL CHECK(task_category IN ('ADMIN','IT_SETUP','TRAINING','SOCIAL','COMPLIANCE','BENEFITS','EQUIPMENT')),
    assignee_type TEXT NOT NULL CHECK(assignee_type IN ('MANAGER','HR','IT','EMPLOYEE','BUDDY','ONBOARDEE')),
    due_offset_days INTEGER NOT NULL DEFAULT 0,
    is_mandatory INTEGER NOT NULL DEFAULT 1,
    completion_type TEXT NOT NULL DEFAULT 'MANUAL' CHECK(completion_type IN ('MANUAL','AUTOMATIC','APPROVAL')),
    depends_on_task_id TEXT,
    order_sequence INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES journey_templates(id) ON DELETE CASCADE
);

CREATE INDEX idx_journey_tasks_tenant_template ON journey_tasks(tenant_id, template_id);
CREATE INDEX idx_journey_tasks_tenant_category ON journey_tasks(tenant_id, task_category);
CREATE INDEX idx_journey_tasks_tenant_assignee ON journey_tasks(tenant_id, assignee_type);
```

### 2.3 Employee Journeys
```sql
CREATE TABLE employee_journeys (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    template_id TEXT,
    journey_type TEXT NOT NULL CHECK(journey_type IN ('ONBOARDING','OFFBOARDING','TRANSFER','ROLE_CHANGE','PROMOTION')),
    trigger_event TEXT,
    start_date TEXT NOT NULL,
    target_completion_date TEXT,
    completed_date TEXT,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED' CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','OVERDUE','CANCELLED')),
    overall_progress DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    assigned_buddy_id TEXT,
    assigned_hr_id TEXT,
    manager_id TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES journey_templates(id)
);

CREATE INDEX idx_emp_journeys_tenant_emp ON employee_journeys(tenant_id, employee_id);
CREATE INDEX idx_emp_journeys_tenant_type ON employee_journeys(tenant_id, journey_type);
CREATE INDEX idx_emp_journeys_tenant_status ON employee_journeys(tenant_id, status);
CREATE INDEX idx_emp_journeys_tenant_dates ON employee_journeys(tenant_id, start_date, target_completion_date);
CREATE INDEX idx_emp_journeys_tenant_manager ON employee_journeys(tenant_id, manager_id);
```

### 2.4 Journey Task Instances
```sql
CREATE TABLE journey_task_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journey_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    task_name TEXT NOT NULL,
    assignee_id TEXT NOT NULL,
    due_date TEXT,
    completed_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','OVERDUE','WAIVED')),
    completion_notes TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL' CHECK(priority IN ('URGENT','HIGH','NORMAL','LOW')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (journey_id) REFERENCES employee_journeys(id) ON DELETE CASCADE
);

CREATE INDEX idx_task_instances_tenant_journey ON journey_task_instances(tenant_id, journey_id);
CREATE INDEX idx_task_instances_tenant_assignee ON journey_task_instances(tenant_id, assignee_id);
CREATE INDEX idx_task_instances_tenant_status ON journey_task_instances(tenant_id, status);
CREATE INDEX idx_task_instances_tenant_due ON journey_task_instances(tenant_id, due_date);
```

### 2.5 Journey Documents
```sql
CREATE TABLE journey_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journey_id TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN ('OFFER_LETTER','NDA','POLICY','TAX_FORM','BENEFITS_ENROLLMENT','HANDBOOK')),
    document_reference TEXT NOT NULL,
    employee_acknowledged INTEGER NOT NULL DEFAULT 0,
    acknowledged_date TEXT,
    is_mandatory INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (journey_id) REFERENCES employee_journeys(id) ON DELETE CASCADE
);

CREATE INDEX idx_journey_docs_tenant_journey ON journey_documents(tenant_id, journey_id);
CREATE INDEX idx_journey_docs_tenant_type ON journey_documents(tenant_id, document_type);
CREATE INDEX idx_journey_docs_tenant_ack ON journey_documents(tenant_id, employee_acknowledged);
```

### 2.6 Onboarding Feedback
```sql
CREATE TABLE onboarding_feedback (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journey_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    feedback_type TEXT NOT NULL CHECK(feedback_type IN ('WEEK_1','WEEK_30','DAY_60','DAY_90','OFFBOARDING')),
    rating INTEGER CHECK(rating BETWEEN 1 AND 5),
    comments TEXT,
    would_recommend INTEGER DEFAULT 0,
    improvement_suggestions TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (journey_id) REFERENCES employee_journeys(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, journey_id, feedback_type)
);

CREATE INDEX idx_onboarding_feedback_tenant_journey ON onboarding_feedback(tenant_id, journey_id);
CREATE INDEX idx_onboarding_feedback_tenant_emp ON onboarding_feedback(tenant_id, employee_id);
CREATE INDEX idx_onboarding_feedback_tenant_type ON onboarding_feedback(tenant_id, feedback_type);
```

### 2.7 Transition Checklists
```sql
CREATE TABLE transition_checklists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    transition_type TEXT NOT NULL CHECK(transition_type IN ('OFFBOARDING','TRANSFER','LEAVE')),
    checklist_items TEXT NOT NULL,  -- JSON array of checklist item objects
    completed_items TEXT NOT NULL DEFAULT '[]',  -- JSON array of completed item IDs
    target_completion_date TEXT,
    responsible_manager_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_transition_checklists_tenant_emp ON transition_checklists(tenant_id, employee_id);
CREATE INDEX idx_transition_checklists_tenant_type ON transition_checklists(tenant_id, transition_type);
CREATE INDEX idx_transition_checklists_tenant_status ON transition_checklists(tenant_id, status);
CREATE INDEX idx_transition_checklists_tenant_mgr ON transition_checklists(tenant_id, responsible_manager_id);
```

---

## 3. REST API Endpoints

### Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/templates` | Create a journey template |
| GET | `/api/v1/templates` | List journey templates |
| GET | `/api/v1/templates/{id}` | Get template details |
| PUT | `/api/v1/templates/{id}` | Update template |
| POST | `/api/v1/templates/{id}/publish` | Publish/activate template |
| POST | `/api/v1/templates/{id}/duplicate` | Duplicate a template |

### Tasks
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/templates/{id}/tasks` | Add task to template |
| GET | `/api/v1/templates/{id}/tasks` | List template tasks |
| PUT | `/api/v1/tasks/{id}` | Update task |
| DELETE | `/api/v1/tasks/{id}` | Remove task from template |
| POST | `/api/v1/templates/{id}/tasks/reorder` | Reorder tasks in template |

### Journeys
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/journeys/start` | Start a journey for an employee |
| GET | `/api/v1/journeys` | List journeys with filters |
| GET | `/api/v1/journeys/{id}` | Get journey details |
| GET | `/api/v1/journeys/{id}/progress` | Get journey progress |
| POST | `/api/v1/journeys/{id}/tasks/{taskId}/complete` | Complete a journey task |
| POST | `/api/v1/journeys/{id}/tasks/{taskId}/waive` | Waive a journey task |
| POST | `/api/v1/journeys/{id}/cancel` | Cancel a journey |

### Documents
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/journeys/{id}/documents` | List journey documents |
| POST | `/api/v1/journeys/{id}/documents` | Add document to journey |
| POST | `/api/v1/documents/{id}/acknowledge` | Acknowledge a document |

### Feedback
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/journeys/{id}/feedback` | Submit journey feedback |
| GET | `/api/v1/journeys/{id}/feedback` | Get journey feedback |
| GET | `/api/v1/feedback` | List feedback with filters |

### Transitions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/transitions/start-offboarding` | Initiate offboarding |
| POST | `/api/v1/transitions/start-transfer` | Initiate transfer |
| GET | `/api/v1/transitions/{id}/checklist` | Get transition checklist progress |
| POST | `/api/v1/transitions/{id}/checklist/{itemId}/complete` | Complete checklist item |
| GET | `/api/v1/transitions` | List transitions with filters |

---

## 4. Business Rules

1. All mandatory tasks in a journey MUST be completed or explicitly waived before the journey can be marked as COMPLETED.
2. The system MUST automatically trigger an onboarding journey when a new employee is created in the HR system.
3. Task instances with `depends_on_task_id` MUST NOT become available for completion until the prerequisite task is completed or waived.
4. Mandatory documents MUST be acknowledged by the employee before the journey can be completed.
5. A buddy MUST be assigned to every onboarding journey and MUST be a different employee from the new hire's manager.
6. Task instances that pass their due date without completion MUST be automatically marked as OVERDUE and trigger a notification to the assignee and the journey manager.
7. The `overall_progress` percentage MUST be calculated as the ratio of completed and waived tasks to total tasks in the journey.
8. Only one active journey of the same type MUST be allowed per employee at any given time.
9. Waived mandatory tasks MUST require a justification in `completion_notes` and MUST be logged for audit purposes.
10. Offboarding checklists MUST include system access revocation as a mandatory item that MUST NOT be waived.
11. The system SHOULD auto-assign the default template for a journey type and department when no specific template is specified.
12. Journey feedback at the WEEK_1 milestone SHOULD be triggered automatically 7 days after the journey start date.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package onboarding.v1;

service OnboardingService {
    // Templates
    rpc CreateTemplate(CreateTemplateRequest) returns (CreateTemplateResponse);
    rpc GetTemplate(GetTemplateRequest) returns (GetTemplateResponse);
    rpc ListTemplates(ListTemplatesRequest) returns (ListTemplatesResponse);
    rpc PublishTemplate(PublishTemplateRequest) returns (PublishTemplateResponse);
    rpc DuplicateTemplate(DuplicateTemplateRequest) returns (DuplicateTemplateResponse);

    // Tasks
    rpc AddTask(AddTaskRequest) returns (AddTaskResponse);
    rpc ListTasks(ListTasksRequest) returns (ListTasksResponse);
    rpc UpdateTask(UpdateTaskRequest) returns (UpdateTaskResponse);
    rpc ReorderTasks(ReorderTasksRequest) returns (ReorderTasksResponse);

    // Journeys
    rpc StartJourney(StartJourneyRequest) returns (StartJourneyResponse);
    rpc GetJourney(GetJourneyRequest) returns (GetJourneyResponse);
    rpc ListJourneys(ListJourneysRequest) returns (ListJourneysResponse);
    rpc GetJourneyProgress(GetJourneyProgressRequest) returns (GetJourneyProgressResponse);
    rpc CompleteTask(CompleteTaskRequest) returns (CompleteTaskResponse);
    rpc WaiveTask(WaiveTaskRequest) returns (WaiveTaskResponse);
    rpc CancelJourney(CancelJourneyRequest) returns (CancelJourneyResponse);

    // Documents
    rpc ListDocuments(ListDocumentsRequest) returns (ListDocumentsResponse);
    rpc AddDocument(AddDocumentRequest) returns (AddDocumentResponse);
    rpc AcknowledgeDocument(AcknowledgeDocumentRequest) returns (AcknowledgeDocumentResponse);

    // Feedback
    rpc SubmitFeedback(SubmitFeedbackRequest) returns (SubmitFeedbackResponse);
    rpc GetFeedback(GetFeedbackRequest) returns (GetFeedbackResponse);

    // Transitions
    rpc StartOffboarding(StartOffboardingRequest) returns (StartOffboardingResponse);
    rpc StartTransfer(StartTransferRequest) returns (StartTransferResponse);
    rpc GetTransitionChecklist(GetTransitionChecklistRequest) returns (GetTransitionChecklistResponse);
    rpc CompleteChecklistItem(CompleteChecklistItemRequest) returns (CompleteChecklistItemResponse);
}

message JourneyTemplate {
    string id = 1;
    string tenant_id = 2;
    string name = 3;
    string description = 4;
    string journey_type = 5;
    string department_id = 6;
    string location_id = 7;
    bool is_default = 8;
    int32 estimated_duration_days = 9;
    string trigger_event = 10;
    string status = 11;
}

message CreateTemplateRequest {
    string tenant_id = 1;
    string name = 2;
    string description = 3;
    string journey_type = 4;
    string department_id = 5;
    string location_id = 6;
    bool is_default = 7;
    int32 estimated_duration_days = 8;
    string trigger_event = 9;
    string created_by = 10;
}

message CreateTemplateResponse {
    JourneyTemplate template = 1;
}

message GetTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
}

message GetTemplateResponse {
    JourneyTemplate template = 1;
}

message ListTemplatesRequest {
    string tenant_id = 1;
    string journey_type = 2;
    string department_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListTemplatesResponse {
    repeated JourneyTemplate templates = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
    string updated_by = 3;
}

message PublishTemplateResponse {
    JourneyTemplate template = 1;
}

message DuplicateTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
    string new_name = 3;
    string created_by = 4;
}

message DuplicateTemplateResponse {
    JourneyTemplate template = 1;
}

message JourneyTask {
    string id = 1;
    string tenant_id = 2;
    string template_id = 3;
    string task_name = 4;
    string task_description = 5;
    string task_category = 6;
    string assignee_type = 7;
    int32 due_offset_days = 8;
    bool is_mandatory = 9;
    string completion_type = 10;
    string depends_on_task_id = 11;
    int32 order_sequence = 12;
}

message AddTaskRequest {
    string tenant_id = 1;
    string template_id = 2;
    string task_name = 3;
    string task_description = 4;
    string task_category = 5;
    string assignee_type = 6;
    int32 due_offset_days = 7;
    bool is_mandatory = 8;
    string completion_type = 9;
    string depends_on_task_id = 10;
    int32 order_sequence = 11;
    string created_by = 12;
}

message AddTaskResponse {
    JourneyTask task = 1;
}

message ListTasksRequest {
    string tenant_id = 1;
    string template_id = 2;
    string task_category = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListTasksResponse {
    repeated JourneyTask tasks = 1;
    string next_page_token = 2;
}

message UpdateTaskRequest {
    string tenant_id = 1;
    string task_id = 2;
    string task_name = 3;
    string task_description = 4;
    string task_category = 5;
    string assignee_type = 6;
    int32 due_offset_days = 7;
    bool is_mandatory = 8;
    string updated_by = 9;
}

message UpdateTaskResponse {
    JourneyTask task = 1;
}

message ReorderTasksRequest {
    string tenant_id = 1;
    string template_id = 2;
    repeated string task_ids_in_order = 3;
    string updated_by = 4;
}

message ReorderTasksResponse {
    repeated JourneyTask tasks = 1;
}

message EmployeeJourney {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string template_id = 4;
    string journey_type = 5;
    string trigger_event = 6;
    string start_date = 7;
    string target_completion_date = 8;
    string completed_date = 9;
    string status = 10;
    double overall_progress = 11;
    string assigned_buddy_id = 12;
    string assigned_hr_id = 13;
    string manager_id = 14;
}

message StartJourneyRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string template_id = 3;
    string journey_type = 4;
    string trigger_event = 5;
    string start_date = 6;
    string assigned_buddy_id = 7;
    string assigned_hr_id = 8;
    string manager_id = 9;
    string created_by = 10;
}

message StartJourneyResponse {
    EmployeeJourney journey = 1;
}

message GetJourneyRequest {
    string tenant_id = 1;
    string journey_id = 2;
}

message GetJourneyResponse {
    EmployeeJourney journey = 1;
}

message ListJourneysRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string journey_type = 3;
    string status = 4;
    string manager_id = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListJourneysResponse {
    repeated EmployeeJourney journeys = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message TaskInstanceProgress {
    string task_instance_id = 1;
    string task_name = 2;
    string task_category = 3;
    string assignee_id = 4;
    string due_date = 5;
    string status = 6;
    string completion_notes = 7;
}

message JourneyProgress {
    string journey_id = 1;
    double overall_progress = 2;
    int32 total_tasks = 3;
    int32 completed_tasks = 4;
    int32 pending_tasks = 5;
    int32 overdue_tasks = 6;
    int32 waived_tasks = 7;
    repeated TaskInstanceProgress tasks = 8;
}

message GetJourneyProgressRequest {
    string tenant_id = 1;
    string journey_id = 2;
}

message GetJourneyProgressResponse {
    JourneyProgress progress = 1;
}

message CompleteTaskRequest {
    string tenant_id = 1;
    string journey_id = 2;
    string task_instance_id = 3;
    string completion_notes = 4;
    string completed_by = 5;
}

message CompleteTaskResponse {
    string task_instance_id = 1;
    string status = 2;
    double updated_progress = 3;
}

message WaiveTaskRequest {
    string tenant_id = 1;
    string journey_id = 2;
    string task_instance_id = 3;
    string completion_notes = 4;
    string waived_by = 5;
}

message WaiveTaskResponse {
    string task_instance_id = 1;
    string status = 2;
    double updated_progress = 3;
}

message CancelJourneyRequest {
    string tenant_id = 1;
    string journey_id = 2;
    string updated_by = 3;
}

message CancelJourneyResponse {
    EmployeeJourney journey = 1;
}

message JourneyDocument {
    string id = 1;
    string tenant_id = 2;
    string journey_id = 3;
    string document_type = 4;
    string document_reference = 5;
    bool employee_acknowledged = 6;
    string acknowledged_date = 7;
    bool is_mandatory = 8;
}

message ListDocumentsRequest {
    string tenant_id = 1;
    string journey_id = 2;
}

message ListDocumentsResponse {
    repeated JourneyDocument documents = 1;
}

message AddDocumentRequest {
    string tenant_id = 1;
    string journey_id = 2;
    string document_type = 3;
    string document_reference = 4;
    bool is_mandatory = 5;
    string created_by = 6;
}

message AddDocumentResponse {
    JourneyDocument document = 1;
}

message AcknowledgeDocumentRequest {
    string tenant_id = 1;
    string document_id = 2;
    string acknowledged_by = 3;
}

message AcknowledgeDocumentResponse {
    JourneyDocument document = 1;
}

message OnboardingFeedback {
    string id = 1;
    string tenant_id = 2;
    string journey_id = 3;
    string employee_id = 4;
    string feedback_type = 5;
    int32 rating = 6;
    string comments = 7;
    bool would_recommend = 8;
    string improvement_suggestions = 9;
}

message SubmitFeedbackRequest {
    string tenant_id = 1;
    string journey_id = 2;
    string employee_id = 3;
    string feedback_type = 4;
    int32 rating = 5;
    string comments = 6;
    bool would_recommend = 7;
    string improvement_suggestions = 8;
    string created_by = 9;
}

message SubmitFeedbackResponse {
    OnboardingFeedback feedback = 1;
}

message GetFeedbackRequest {
    string tenant_id = 1;
    string journey_id = 2;
}

message GetFeedbackResponse {
    repeated OnboardingFeedback feedback = 1;
}

message TransitionChecklist {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string transition_type = 4;
    string checklist_items = 5;
    string completed_items = 6;
    string target_completion_date = 7;
    string responsible_manager_id = 8;
    string status = 9;
}

message StartOffboardingRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string responsible_manager_id = 3;
    string target_completion_date = 4;
    string checklist_items = 5;
    string created_by = 6;
}

message StartOffboardingResponse {
    TransitionChecklist checklist = 1;
    string journey_id = 2;
}

message StartTransferRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string target_department_id = 3;
    string target_role_id = 4;
    string responsible_manager_id = 5;
    string target_completion_date = 6;
    string created_by = 7;
}

message StartTransferResponse {
    TransitionChecklist checklist = 1;
    string journey_id = 2;
}

message GetTransitionChecklistRequest {
    string tenant_id = 1;
    string transition_id = 2;
}

message GetTransitionChecklistResponse {
    TransitionChecklist checklist = 1;
}

message CompleteChecklistItemRequest {
    string tenant_id = 1;
    string transition_id = 2;
    string item_id = 3;
    string completed_by = 4;
}

message CompleteChecklistItemResponse {
    TransitionChecklist checklist = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee created/updated events, org structure, manager assignments | Trigger journeys and resolve task assignees |
| `document-service` | Document storage and retrieval | Manage journey documents and acknowledgments |
| `notification-service` | Notification delivery | Send task reminders, deadline alerts, and feedback prompts |
| `workflow-service` | Approval workflows | Process task approvals and journey escalations |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Journey completion status, onboarding progress | Update employee records with onboarding state |
| `document-service` | Document references, acknowledgment records | Store and track document acknowledgments |
| `notification-service` | Task assignments, overdue alerts, feedback reminders | Notify assignees, managers, and HR |
| `reporting-service` | Onboarding metrics, time-to-productivity data | HR analytics and onboarding dashboards |
| `experience-service` | Onboarding feedback, new hire engagement data | Feed employee experience analytics |
| `it-service` | Equipment provisioning, access requests | Fulfill IT setup tasks in journeys |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `JourneyStarted` | `onboarding.journey.started` | `{ tenant_id, journey_id, employee_id, journey_type, template_id, start_date, target_completion_date }` | Published when a new journey is initiated for an employee |
| `TaskCompleted` | `onboarding.task.completed` | `{ tenant_id, journey_id, task_instance_id, task_name, assignee_id, completed_date }` | Published when a journey task is completed |
| `JourneyCompleted` | `onboarding.journey.completed` | `{ tenant_id, journey_id, employee_id, journey_type, completed_date, overall_progress }` | Published when all mandatory tasks in a journey are finished |
| `OffboardingInitiated` | `onboarding.offboarding.initiated` | `{ tenant_id, employee_id, transition_id, target_completion_date, responsible_manager_id }` | Published when offboarding is started for an employee |
| `DocumentAcknowledged` | `onboarding.document.acknowledged` | `{ tenant_id, document_id, journey_id, employee_id, document_type, acknowledged_date }` | Published when an employee acknowledges a document |
