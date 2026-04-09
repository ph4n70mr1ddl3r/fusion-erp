# 206 - Construction & Engineering Industry Solution Specification

## 1. Domain Overview

Construction & Engineering provides industry-specific capabilities for construction project delivery, progress billing with retainage, BIM coordination, change order management, and compliance documentation. Supports construction project lifecycle from bidding through close-out, unit price and lump sum contract management, progress claim management with retainage tracking, construction change directives and variation orders, daily log and site reporting, safety compliance and incident management, and BIM model coordination. Enables construction and engineering firms to manage complex projects with multiple stakeholders, maintain regulatory compliance, and control costs throughout the project lifecycle. Integrates with Project Management, Financials, and Procurement.

**Bounded Context:** Construction Project Delivery & Engineering Management
**Service Name:** `construction-engineering-service`
**Database:** `data/construction_engineering.db`
**HTTP Port:** 8224 | **gRPC Port:** 9224

---

## 2. Database Schema

### 2.1 Construction Projects
```sql
CREATE TABLE ce_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_number TEXT NOT NULL,
    project_name TEXT NOT NULL,
    project_type TEXT NOT NULL CHECK(project_type IN ('GENERAL_CONSTRUCTION','DESIGN_BUILD','EPC','RENOVATION','INFRASTRUCTURE','RESIDENTIAL','COMMERCIAL')),
    client_id TEXT NOT NULL,
    contract_type TEXT NOT NULL CHECK(contract_type IN ('LUMP_SUM','UNIT_PRICE','COST_PLUS','GMP','TIME_MATERIALS')),
    contract_value_cents INTEGER NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    location TEXT NOT NULL,
    site_address TEXT,
    start_date TEXT NOT NULL,
    planned_end_date TEXT NOT NULL,
    actual_end_date TEXT,
    project_manager_id TEXT NOT NULL,
    superintendent_id TEXT,
    prime_contractor_id TEXT,
    permit_number TEXT,
    permit_expiry TEXT,
    safety_plan_id TEXT,
    bim_model_reference TEXT,
    weather_delay_days INTEGER NOT NULL DEFAULT 0,
    completion_pct REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNING'
        CHECK(status IN ('PLANNING','BIDDING','IN_PROGRESS','SUBSTANTIALLY_COMPLETE','COMPLETED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_number)
);

CREATE INDEX idx_ce_proj_tenant ON ce_projects(tenant_id, status);
CREATE INDEX idx_ce_proj_client ON ce_projects(client_id);
CREATE INDEX idx_ce_proj_dates ON ce_projects(start_date, planned_end_date);
CREATE INDEX idx_ce_proj_pm ON ce_projects(project_manager_id);
```

### 2.2 Progress Claims
```sql
CREATE TABLE ce_progress_claims (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    claim_number TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    work_completed_pct REAL NOT NULL DEFAULT 0,
    work_completed_value_cents INTEGER NOT NULL DEFAULT 0,
    materials_stored_cents INTEGER NOT NULL DEFAULT 0,
    total_claimed_cents INTEGER NOT NULL DEFAULT 0,
    retainage_pct REAL NOT NULL DEFAULT 10,
    retainage_held_cents INTEGER NOT NULL DEFAULT 0,
    previous_certified_cents INTEGER NOT NULL DEFAULT 0,
    net_certified_cents INTEGER NOT NULL DEFAULT 0,
    change_orders_included TEXT,                  -- JSON: change order IDs
    supporting_documents TEXT,                    -- JSON: document references
    certified_by TEXT,
    certified_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','CERTIFIED','REJECTED','PAID')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES ce_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, claim_number)
);

CREATE INDEX idx_ce_claim_project ON ce_progress_claims(project_id, status);
CREATE INDEX idx_ce_claim_period ON ce_progress_claims(period_start, period_end);
CREATE INDEX idx_ce_claim_status ON ce_progress_claims(tenant_id, status);
```

### 2.3 Change Orders
```sql
CREATE TABLE ce_change_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    co_number TEXT NOT NULL,
    co_type TEXT NOT NULL CHECK(co_type IN ('OWNER_DIRECTED','CONSTRUCTION_CHANGE_DIRECTIVE','VARIATION','FIELD_ORDER','CLAIM')),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    reason TEXT NOT NULL,
    impact_schedule_days INTEGER NOT NULL DEFAULT 0,
    impact_cost_cents INTEGER NOT NULL DEFAULT 0,
    proposed_cost_cents INTEGER NOT NULL DEFAULT 0,
    approved_cost_cents INTEGER,
    affected_work TEXT NOT NULL,                  -- JSON: work items affected
    documents TEXT,                               -- JSON: supporting documents
    requested_by TEXT NOT NULL,
    approved_by TEXT,
    approved_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED','EXECUTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES ce_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, co_number)
);

CREATE INDEX idx_ce_co_project ON ce_change_orders(project_id, status);
CREATE INDEX idx_ce_co_type ON ce_change_orders(co_type);
```

### 2.4 Daily Logs
```sql
CREATE TABLE ce_daily_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    log_date TEXT NOT NULL,
    weather_conditions TEXT,                      -- JSON: temp, precipitation, wind
    weather_delay_hours REAL NOT NULL DEFAULT 0,
    workers_on_site INTEGER NOT NULL DEFAULT 0,
    subcontractors_on_site TEXT,                  -- JSON: [{company, workers, trade}]
    equipment_on_site TEXT,                       -- JSON: [{equipment, hours}]
    work_performed TEXT NOT NULL,
    areas_worked TEXT,
    deliveries TEXT,                              -- JSON: deliveries received
    inspections TEXT,                             -- JSON: inspections conducted
    safety_incidents INTEGER NOT NULL DEFAULT 0,
    visitors TEXT,                                -- JSON: site visitors
    issues TEXT,                                  -- JSON: issues noted
    photos TEXT,                                  -- JSON: photo references
    reported_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, project_id, log_date)
);

CREATE INDEX idx_ce_log_project ON ce_daily_logs(project_id, log_date DESC);
CREATE INDEX idx_ce_log_date ON ce_daily_logs(log_date DESC);
```

### 2.5 Submittals & RFIs
```sql
CREATE TABLE ce_submittals_rfis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    item_type TEXT NOT NULL CHECK(item_type IN ('SUBMITTAL','RFI','SHOP_DRAWING','SAMPLE')),
    item_number TEXT NOT NULL,
    subject TEXT NOT NULL,
    description TEXT NOT NULL,
    specification_reference TEXT,
    drawing_reference TEXT,
    due_date TEXT,
    response_required_date TEXT,
    assigned_to TEXT NOT NULL,
    response TEXT,
    response_by TEXT,
    response_at TEXT,
    attachments TEXT,                             -- JSON: document references
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    cost_impact INTEGER NOT NULL DEFAULT 0,
    schedule_impact_days INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_REVIEW','RESPONDED','CLOSED','VOID')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (project_id) REFERENCES ce_projects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, item_number)
);

CREATE INDEX idx_ce_rfi_project ON ce_submittals_rfis(project_id, status);
CREATE INDEX idx_ce_rfi_type ON ce_submittals_rfis(item_type, status);
CREATE INDEX idx_ce_rfi_due ON ce_submittals_rfis(due_date);
```

---

## 3. API Endpoints

### 3.1 Projects
| Method | Path | Description |
|--------|------|-------------|
| POST | `/construction/v1/projects` | Create project |
| GET | `/construction/v1/projects` | List projects |
| GET | `/construction/v1/projects/{id}` | Get project |
| PUT | `/construction/v1/projects/{id}` | Update project |
| GET | `/construction/v1/projects/{id}/dashboard` | Project dashboard |

### 3.2 Progress Claims
| Method | Path | Description |
|--------|------|-------------|
| POST | `/construction/v1/claims` | Create progress claim |
| GET | `/construction/v1/claims` | List claims |
| GET | `/construction/v1/claims/{id}` | Get claim |
| POST | `/construction/v1/claims/{id}/certify` | Certify claim |

### 3.3 Change Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/construction/v1/change-orders` | Create change order |
| GET | `/construction/v1/change-orders` | List change orders |
| GET | `/construction/v1/change-orders/{id}` | Get change order |
| POST | `/construction/v1/change-orders/{id}/approve` | Approve change order |

### 3.4 Daily Logs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/construction/v1/daily-logs` | Submit daily log |
| GET | `/construction/v1/daily-logs` | List daily logs |
| GET | `/construction/v1/daily-logs/{id}` | Get log details |

### 3.5 Submittals & RFIs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/construction/v1/rfis` | Create RFI/submittal |
| GET | `/construction/v1/rfis` | List RFIs |
| GET | `/construction/v1/rfis/{id}` | Get RFI |
| POST | `/construction/v1/rfis/{id}/respond` | Respond to RFI |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `construct.project.started` | `{ project_id, contract_value, start_date }` | Construction started |
| `construct.claim.certified` | `{ claim_id, amount, retainage }` | Progress claim certified |
| `construct.co.approved` | `{ co_id, impact_cost, impact_days }` | Change order approved |
| `construct.rfi.responded` | `{ rfi_id, schedule_impact }` | RFI responded |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `project.milestone.completed` | Project Management (15) | Update completion % |
| `invoice.approved` | AP (07) | Process progress payment |
| `permit.approved` | External | Update project status |

---

## 5. Business Rules

1. **Retainage**: Default 10% retainage held until substantial completion; released per contract terms
2. **Change Order Impact**: All change orders require cost and schedule impact assessment
3. **Daily Log Required**: Active projects require daily log submission by end of business day
4. **RFI Response SLA**: RFIs must be responded to within contractually specified timeframe
5. **Weather Delays**: Weather delays tracked and applied to schedule extension calculations
6. **Progress Verification**: Progress claims verified against on-site inspection before certification

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Project Management (15) | Project structure and milestones |
| Accounts Payable (07) | Progress payment processing |
| Project Costing (158) | Cost tracking and budgeting |
| Document Management (29) | Drawing and submittal storage |
| Procurement (11) | Material and subcontractor ordering |
| Enterprise Scheduler (173) | Scheduled reporting |
