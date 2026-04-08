# 15 - Project Management Service Specification

## 1. Domain Overview

Project Management handles project planning, task breakdown, resource allocation, time/expense entry, project costing, and billing. Integrates with GL for project accounting, AP for project expenses, and AR for project billing.

**Bounded Context:** Project & Resource Management
**Service Name:** `pm-service`
**Database:** `data/pm.db`
**HTTP Port:** 8030 | **gRPC Port:** 9030

---

## 2. Database Schema

### 2.1 Projects
```sql
CREATE TABLE projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_number TEXT NOT NULL,          -- PRJ-2024-00001
    project_name TEXT NOT NULL,
    description TEXT,
    project_type TEXT NOT NULL DEFAULT 'INTERNAL'
        CHECK(project_type IN ('INTERNAL','BILLABLE','CAPITAL','R_AND_D','MAINTENANCE')),
    project_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(project_status IN ('DRAFT','PLANNED','IN_PROGRESS','ON_HOLD','COMPLETED','CANCELLED')),

    -- Dates
    planned_start_date TEXT,
    planned_end_date TEXT,
    actual_start_date TEXT,
    actual_end_date TEXT,

    -- Hierarchy
    parent_project_id TEXT,
    program_id TEXT,

    -- Customer (for billable projects)
    customer_id TEXT,
    contract_id TEXT,
    billing_method TEXT CHECK(billing_method IN ('FIXED_PRICE','TIME_AND_MATERIALS','MILESTONE','RETAINER')),

    -- Budget
    budget_revenue_cents INTEGER NOT NULL DEFAULT 0,
    budget_cost_cents INTEGER NOT NULL DEFAULT 0,
    budget_margin_percent REAL,

    -- Actuals
    actual_revenue_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    committed_cost_cents INTEGER NOT NULL DEFAULT 0,
    remaining_budget_cents INTEGER NOT NULL DEFAULT 0,

    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Ownership
    project_manager_id TEXT NOT NULL,
    department TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_project_id) REFERENCES projects(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, project_number)
);

CREATE INDEX idx_projects_tenant_customer ON projects(tenant_id, customer_id);
CREATE INDEX idx_projects_tenant_status ON projects(tenant_id, project_status);
CREATE INDEX idx_projects_tenant_manager ON projects(tenant_id, project_manager_id);
CREATE INDEX idx_projects_tenant_dates ON projects(tenant_id, planned_start_date, planned_end_date);
```

### 2.2 Project Tasks (WBS)
```sql
CREATE TABLE project_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    parent_task_id TEXT,
    task_number TEXT NOT NULL,             -- "1.2.3" (WBS numbering)
    task_name TEXT NOT NULL,
    description TEXT,
    task_type TEXT DEFAULT 'TASK'
        CHECK(task_type IN ('TASK','MILESTONE','SUMMARY','DELIVERABLE')),

    -- Schedule
    planned_start_date TEXT,
    planned_end_date TEXT,
    actual_start_date TEXT,
    actual_end_date TEXT,
    planned_duration_days INTEGER,
    percent_complete REAL DEFAULT 0,

    -- Effort
    planned_effort_hours REAL DEFAULT 0,
    actual_effort_hours REAL DEFAULT 0,

    -- Cost
    planned_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,

    -- Assignment
    assigned_to TEXT,
    priority TEXT DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','ON_HOLD','CANCELLED')),

    -- Dependencies
    dependency_type TEXT CHECK(dependency_type IN ('FINISH_TO_START','START_TO_START','FINISH_TO_FINISH','START_TO_FINISH')),
    predecessor_task_id TEXT,

    -- Billing
    is_billable INTEGER NOT NULL DEFAULT 1,
    billing_rate_cents INTEGER,
    milestone_amount_cents INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE RESTRICT,
    FOREIGN KEY (parent_task_id) REFERENCES project_tasks(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, project_id, task_number)
);

CREATE INDEX idx_project_tasks_tenant_project ON project_tasks(tenant_id, project_id);
CREATE INDEX idx_project_tasks_tenant_assigned ON project_tasks(tenant_id, assigned_to);
```

### 2.3 Time Entries
```sql
CREATE TABLE time_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    entry_date TEXT NOT NULL,
    hours DECIMAL(8,2) NOT NULL,
    description TEXT,
    activity_type TEXT DEFAULT 'WORK'
        CHECK(activity_type IN ('WORK','OVERTIME','TRAVEL','TRAINING','MEETING')),

    -- Cost & billing
    cost_rate_cents INTEGER NOT NULL,
    cost_amount_cents INTEGER NOT NULL DEFAULT 0,
    billing_rate_cents INTEGER,
    billing_amount_cents INTEGER,
    is_billable INTEGER NOT NULL DEFAULT 1,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED','INVOICED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE RESTRICT,
    FOREIGN KEY (task_id) REFERENCES project_tasks(id) ON DELETE RESTRICT
);

CREATE INDEX idx_time_entries_tenant_project ON time_entries(tenant_id, project_id);
CREATE INDEX idx_time_entries_tenant_employee ON time_entries(tenant_id, employee_id);
CREATE INDEX idx_time_entries_tenant_date ON time_entries(tenant_id, entry_date);
CREATE INDEX idx_time_entries_tenant_status ON time_entries(tenant_id, status);
```

### 2.4 Expense Entries
```sql
CREATE TABLE expense_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    task_id TEXT,
    employee_id TEXT NOT NULL,
    expense_date TEXT NOT NULL,
    expense_type TEXT NOT NULL
        CHECK(expense_type IN ('TRAVEL','MEALS','LODGING','MATERIALS','MISC','PER_DIEM')),
    description TEXT NOT NULL,
    amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    reimbursable_amount_cents INTEGER NOT NULL DEFAULT 0,
    is_billable INTEGER NOT NULL DEFAULT 1,
    billing_amount_cents INTEGER,

    receipt_url TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED','REIMBURSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE RESTRICT
);

CREATE INDEX idx_expense_entries_tenant_project ON expense_entries(tenant_id, project_id);
CREATE INDEX idx_expense_entries_tenant_employee ON expense_entries(tenant_id, employee_id);
```

### 2.5 Project Billing
```sql
CREATE TABLE project_billings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    billing_date TEXT NOT NULL,
    billing_type TEXT NOT NULL
        CHECK(billing_type IN ('TIME_AND_MATERIALS','MILESTONE','PERCENT_COMPLETE','FIXED')),
    amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','INVOICED','CANCELLED')),
    ar_invoice_id TEXT,                    -- Link to AR invoice

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE RESTRICT
);
```

---

## 3. REST API Endpoints

```
# Projects
GET/POST      /api/v1/pm/projects                    Permission: pm.projects.read/create
GET/PUT       /api/v1/pm/projects/{id}                Permission: pm.projects.read/update
POST          /api/v1/pm/projects/{id}/activate       Permission: pm.projects.update
POST          /api/v1/pm/projects/{id}/complete       Permission: pm.projects.update
POST          /api/v1/pm/projects/{id}/hold           Permission: pm.projects.update
GET           /api/v1/pm/projects/{id}/budget-vs-actual Permission: pm.projects.read

# Tasks
GET/POST      /api/v1/pm/projects/{id}/tasks          Permission: pm.tasks.read/create
GET/PUT       /api/v1/pm/tasks/{id}                    Permission: pm.tasks.read/update
POST          /api/v1/pm/tasks/{id}/complete           Permission: pm.tasks.update

# Time Entries
GET/POST      /api/v1/pm/time-entries                 Permission: pm.time.read/create
GET/PUT       /api/v1/pm/time-entries/{id}             Permission: pm.time.read/update
POST          /api/v1/pm/time-entries/{id}/submit      Permission: pm.time.create
POST          /api/v1/pm/time-entries/{id}/approve     Permission: pm.time.approve
POST          /api/v1/pm/time-entries/bulk              Permission: pm.time.create

# Expense Entries
GET/POST      /api/v1/pm/expenses                     Permission: pm.expenses.read/create
GET/PUT       /api/v1/pm/expenses/{id}                 Permission: pm.expenses.read/update
POST          /api/v1/pm/expenses/{id}/submit          Permission: pm.expenses.create
POST          /api/v1/pm/expenses/{id}/approve         Permission: pm.expenses.approve

# Billing
GET/POST      /api/v1/pm/projects/{id}/billings       Permission: pm.billing.read/create
POST          /api/v1/pm/billings/{id}/invoice        Permission: pm.billing.create

# Reports
GET           /api/v1/pm/reports/project-costs        Permission: pm.reports.view
GET           /api/v1/pm/reports/utilization           Permission: pm.reports.view
GET           /api/v1/pm/reports/revenue-by-project   Permission: pm.reports.view
```

---

## 4. Business Rules

### 4.1 Project Costing
- `actual_cost = SUM(time_entries.cost_amount) + SUM(expense_entries.amount) + overhead`
- `remaining_budget = budget_cost - actual_cost - committed_cost`
- Cost rates per employee (configurable), updated on time entry approval

### 4.2 Revenue Recognition
- **Fixed Price:** Recognize on milestone completion or % complete
- **T&M:** Recognize based on approved billable time and expenses
- On billing approval, create AR invoice via gRPC

### 4.3 GL Integration
- **Time entry approval:** Debit Project Cost (WIP), Credit Labor Cost
- **Expense approval:** Debit Project Cost (WIP), Credit AP/Cash
- **Billing:** Debit AR, Credit Project Revenue
- **Completion:** Debit COGS, Credit WIP

### 4.4 Events Published
| Event | Trigger |
|-------|---------|
| `pm.project.created` | Project created |
| `pm.project.status_changed` | Project status changed |
| `pm.time_entry.approved` | Time entry approved (GL cost entry) |
| `pm.expense.approved` | Expense approved |
| `pm.billing.invoiced` | Billing invoiced via AR |
