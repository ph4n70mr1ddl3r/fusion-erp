# 147 - Oracle Journeys Service Specification

## 1. Domain Overview

Oracle Journeys provides configurable, guided process templates for employee and manager life events and transitions. Supports personalization based on role, location, and event type. Enables HR to design step-by-step journeys for onboarding, offboarding, leave of absence, promotion, relocation, and other life events with tasks, content, integrations, and deadlines. Includes journey analytics, completion tracking, and dynamic task adjustment based on employee responses. Integrates with Core HR, Onboarding, Learning, Benefits, Payroll, and Workflow.

**Bounded Context:** Guided Employee Journeys & Transitions
**Service Name:** `journeys-service`
**Database:** `data/journeys.db`
**HTTP Port:** 8165 | **gRPC Port:** 9165

---

## 2. Database Schema

### 2.1 Journey Templates
```sql
CREATE TABLE j_journey_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    journey_type TEXT NOT NULL CHECK(journey_type IN (
        'ONBOARDING','OFFBOARDING','LEAVE_OF_ABSENCE','RETURN_FROM_LEAVE',
        'PROMOTION','TRANSFER','RELOCATION','ROLE_CHANGE','MERGER_ACQUISITION',
        'COMPLIANCE','ANNUAL_RENEWAL','CUSTOM'
    )),
    target_audience TEXT NOT NULL CHECK(target_audience IN ('EMPLOYEE','MANAGER','HR_ADMIN','ALL')),
    estimated_duration_days INTEGER NOT NULL DEFAULT 30,
    personalization_rules TEXT,             -- JSON: rules for dynamic content
    eligibility_criteria TEXT,              -- JSON: who qualifies for this journey
    auto_assign INTEGER NOT NULL DEFAULT 0,
    auto_assign_trigger TEXT,              -- JSON: event that triggers auto-assignment
    default_language TEXT NOT NULL DEFAULT 'en',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),
    version_number INTEGER NOT NULL DEFAULT 1,
    published_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_j_template_tenant ON j_journey_templates(tenant_id, journey_type, status);
```

### 2.2 Journey Steps
```sql
CREATE TABLE j_journey_steps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    step_number INTEGER NOT NULL,
    step_code TEXT NOT NULL,
    step_title TEXT NOT NULL,
    step_description TEXT,
    step_type TEXT NOT NULL CHECK(step_type IN (
        'TASK','FORM','DOCUMENT','LEARNING','APPROVAL','NOTIFICATION',
        'INTEGRATION','WAIT','CONDITIONAL_BRANCH','CHECKLIST','EXTERNAL_LINK'
    )),
    assigned_to TEXT NOT NULL CHECK(assigned_to IN ('EMPLOYEE','MANAGER','HR_ADMIN','SYSTEM','EXTERNAL')),
    due_offset_days INTEGER NOT NULL,       -- Days relative to journey start
    due_offset_from TEXT NOT NULL DEFAULT 'JOURNEY_START'
        CHECK(due_offset_from IN ('JOURNEY_START','PREVIOUS_STEP','CUSTOM_DATE')),
    mandatory INTEGER NOT NULL DEFAULT 1,
    prerequisites TEXT,                     -- JSON array of step codes that must be completed first
    conditional_logic TEXT,                 -- JSON: show/hide based on employee attributes
    content_items TEXT,                     -- JSON array: documents, videos, links
    integration_config TEXT,                -- JSON: external system call config
    form_template_id TEXT,
    estimated_minutes INTEGER NOT NULL DEFAULT 15,
    category TEXT,                          -- Grouping for UI display

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES j_journey_templates(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, template_id, step_code)
);

CREATE INDEX idx_j_step_template ON j_journey_steps(template_id, step_number);
```

### 2.3 Journey Instances
```sql
CREATE TABLE j_journey_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    manager_id TEXT,
    journey_type TEXT NOT NULL,
    trigger_event TEXT,                     -- What triggered this journey
    trigger_event_id TEXT,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','CANCELLED','PAUSED')),
    started_at TEXT,
    completed_at TEXT,
    target_completion TEXT,
    completion_pct REAL NOT NULL DEFAULT 0,
    total_steps INTEGER NOT NULL DEFAULT 0,
    completed_steps INTEGER NOT NULL DEFAULT 0,
    overdue_steps INTEGER NOT NULL DEFAULT 0,
    personalization_applied TEXT,           -- JSON: applied personalization context
    assigned_by TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES j_journey_templates(id) ON DELETE RESTRICT
);

CREATE INDEX idx_j_instance_person ON j_journey_instances(tenant_id, person_id, status);
CREATE INDEX idx_j_instance_template ON j_journey_instances(template_id);
CREATE INDEX idx_j_instance_status ON j_journey_instances(tenant_id, status);
```

### 2.4 Journey Step Instances
```sql
CREATE TABLE j_step_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journey_instance_id TEXT NOT NULL,
    step_id TEXT NOT NULL,
    step_code TEXT NOT NULL,
    step_title TEXT NOT NULL,
    step_type TEXT NOT NULL,
    assigned_to TEXT NOT NULL,
    assignee_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','SKIPPED','OVERDUE','CANCELLED')),
    due_date TEXT NOT NULL,
    started_at TEXT,
    completed_at TEXT,
    skipped_reason TEXT,
    form_data TEXT,                         -- JSON: submitted form responses
    integration_response TEXT,              -- JSON: external system response
    notes TEXT,
    attachments TEXT,                       -- JSON array of document IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (journey_instance_id) REFERENCES j_journey_instances(id) ON DELETE CASCADE,
    FOREIGN KEY (step_id) REFERENCES j_journey_steps(id) ON DELETE RESTRICT
);

CREATE INDEX idx_j_stepinstance_journey ON j_step_instances(journey_instance_id);
CREATE INDEX idx_j_stepinstance_assignee ON j_step_instances(tenant_id, assignee_id, status);
CREATE INDEX idx_j_stepinstance_due ON j_step_instances(tenant_id, due_date, status);
```

### 2.5 Journey Analytics
```sql
CREATE TABLE j_journey_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_id TEXT NOT NULL,
    journey_type TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-Q1"
    instances_started INTEGER NOT NULL DEFAULT 0,
    instances_completed INTEGER NOT NULL DEFAULT 0,
    instances_cancelled INTEGER NOT NULL DEFAULT 0,
    instances_in_progress INTEGER NOT NULL DEFAULT 0,
    avg_completion_days REAL NOT NULL DEFAULT 0,
    avg_completion_pct REAL NOT NULL DEFAULT 0,
    step_completion_rates TEXT NOT NULL,    -- JSON: { step_code: completion_pct }
    most_overdue_steps TEXT NOT NULL,       -- JSON: top overdue steps
    nps_score REAL,
    feedback_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, template_id, period)
);

CREATE INDEX idx_j_analytics_tenant ON j_journey_analytics(tenant_id, journey_type, period DESC);
```

---

## 3. API Endpoints

### 3.1 Journey Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/journeys/templates` | Create journey template |
| GET | `/api/v1/journeys/templates` | List templates |
| GET | `/api/v1/journeys/templates/{id}` | Get template details |
| PUT | `/api/v1/journeys/templates/{id}` | Update template |
| POST | `/api/v1/journeys/templates/{id}/publish` | Publish template |
| POST | `/api/v1/journeys/templates/{id}/clone` | Clone template |
| DELETE | `/api/v1/journeys/templates/{id}` | Archive template |

### 3.2 Journey Steps
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/journeys/templates/{id}/steps` | Add step to template |
| GET | `/api/v1/journeys/templates/{id}/steps` | List steps |
| PUT | `/api/v1/journeys/steps/{id}` | Update step |
| DELETE | `/api/v1/journeys/steps/{id}` | Remove step |
| POST | `/api/v1/journeys/templates/{id}/steps/reorder` | Reorder steps |

### 3.3 Journey Instances
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/journeys/instances` | Assign journey to person |
| POST | `/api/v1/journeys/instances/auto-assign` | Auto-assign based on event |
| GET | `/api/v1/journeys/instances` | List instances |
| GET | `/api/v1/journeys/instances/{id}` | Get instance details with steps |
| PUT | `/api/v1/journeys/instances/{id}` | Update instance |
| POST | `/api/v1/journeys/instances/{id}/pause` | Pause journey |
| POST | `/api/v1/journeys/instances/{id}/resume` | Resume journey |
| POST | `/api/v1/journeys/instances/{id}/cancel` | Cancel journey |

### 3.4 Step Execution
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/journeys/instances/{id}/steps` | Get step statuses |
| POST | `/api/v1/journeys/step-instances/{id}/start` | Start step |
| POST | `/api/v1/journeys/step-instances/{id}/complete` | Complete step |
| POST | `/api/v1/journeys/step-instances/{id}/skip` | Skip step |
| PUT | `/api/v1/journeys/step-instances/{id}/form` | Submit form data |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/journeys/analytics/summary` | Journey analytics summary |
| GET | `/api/v1/journeys/analytics/template/{id}` | Template-specific analytics |
| GET | `/api/v1/journeys/analytics/person/{personId}` | Person's journey history |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `journey.assigned` | `{ instance_id, person_id, template_id, type }` | Journey assigned to person |
| `journey.started` | `{ instance_id, person_id }` | Journey started |
| `journey.step.completed` | `{ instance_id, step_code, assignee_id }` | Step completed |
| `journey.step.overdue` | `{ instance_id, step_code, due_date }` | Step overdue |
| `journey.completed` | `{ instance_id, person_id, completion_pct }` | Journey completed |
| `journey.integration.triggered` | `{ instance_id, step_code, system }` | External integration triggered |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `employee.hired` | Core HR | Auto-assign onboarding journey |
| `employee.terminated` | Core HR | Auto-assign offboarding journey |
| `employee.transferred` | Core HR | Auto-assign transfer journey |
| `employee.leave.started` | Absence Mgmt | Auto-assign LOA journey |
| `employee.promoted` | Core HR | Auto-assign promotion journey |
| `learning.course.completed` | Learning | Complete learning step |
| `benefits.enrollment.completed` | Benefits | Complete benefits step |

---

## 5. Business Rules

1. **Auto-Assignment**: Journeys auto-assigned based on configured trigger events matching employee attributes
2. **Personalization**: Steps dynamically adjusted based on employee role, location, department
3. **Conditional Steps**: Steps may be skipped based on employee attributes (e.g., location-specific compliance)
4. **Overdue Escalation**: Steps overdue by 3 days escalate to manager; 7 days to HR admin
5. **Mandatory Completion**: All mandatory steps must be completed before journey marked complete
6. **Template Versioning**: Published templates are versioned; active instances continue on original version
7. **Parallel Steps**: Multiple steps can be assigned simultaneously for faster completion
8. **Completion Percentage**: Calculated as completed_mandatory_steps / total_mandatory_steps

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Core HR (62) | Employee events trigger journey assignment |
| Onboarding (96) | Onboarding journey orchestration |
| Learning & Development (69) | Learning course assignments as steps |
| Benefits (66) | Benefits enrollment steps |
| Payroll (63) | Payroll setup steps for new hires |
| Workflow (16) | Approval steps within journeys |
| Document Management (29) | Document collection and storage |
| Manager Edge (106) | Manager task notifications |
| Digital Assistant (43) | Conversational journey guidance |
