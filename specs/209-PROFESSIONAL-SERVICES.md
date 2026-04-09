# 209 - Professional Services Industry Solution Specification

## 1. Domain Overview

Professional Services provides industry-specific capabilities for client engagement management, resource utilization optimization, time-and-expense billing, project profitability tracking, and practice management. Supports client opportunity pipeline with win probability, resource demand forecasting with skill matching, utilization tracking at individual and practice level, multiple billing methods (hourly, fixed fee, retainer, milestone), work-in-progress management, and project profitability analysis with margin tracking. Enables consulting, legal, accounting, and IT services firms to optimize resource allocation, maximize billable utilization, and track profitability across engagements. Integrates with Project Management, Resource Management, and Financials.

**Bounded Context:** Professional Services Automation & Practice Management
**Service Name:** `professional-services-service`
**Database:** `data/professional_services.db`
**HTTP Port:** 8227 | **gRPC Port:** 9227

---

## 2. Database Schema

### 2.1 Client Engagements
```sql
CREATE TABLE ps_engagements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    engagement_number TEXT NOT NULL,
    engagement_name TEXT NOT NULL,
    client_id TEXT NOT NULL,
    client_name TEXT NOT NULL,
    practice_area TEXT NOT NULL,
    engagement_type TEXT NOT NULL CHECK(engagement_type IN ('CONSULTING','ADVISORY','IMPLEMENTATION','MANAGED_SERVICE','AUDIT','LEGAL')),
    billing_method TEXT NOT NULL CHECK(billing_method IN ('HOURLY','FIXED_FEE','RETAINER','MILESTONE','BLENDED')),
    contract_value_cents INTEGER NOT NULL DEFAULT 0,
    budget_hours REAL NOT NULL DEFAULT 0,
    bill_rate_cents INTEGER NOT NULL DEFAULT 0,
    cost_rate_cents INTEGER NOT NULL DEFAULT 0,
    engagement_start TEXT NOT NULL,
    engagement_end TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    win_probability_pct REAL NOT NULL DEFAULT 50,
    stage TEXT NOT NULL DEFAULT 'PIPELINE'
        CHECK(stage IN ('PIPELINE','PROPOSAL','NEGOTIATION','WON','ACTIVE','COMPLETED','LOST')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, engagement_number)
);

CREATE INDEX idx_ps_eng_client ON ps_engagements(client_id, stage);
CREATE INDEX idx_ps_eng_practice ON ps_engagements(practice_area, stage);
CREATE INDEX idx_ps_eng_partner ON ps_engagements(partner_id);
CREATE INDEX idx_ps_eng_dates ON ps_engagements(engagement_start, engagement_end);
```

### 2.2 Time & Expense Entries
```sql
CREATE TABLE ps_time_expense (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    engagement_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    entry_date TEXT NOT NULL,
    entry_type TEXT NOT NULL CHECK(entry_type IN ('BILLABLE','NON_BILLABLE','ADMIN','BUSINESS_DEV','TRAINING','PTO')),
    hours REAL NOT NULL DEFAULT 0,
    bill_rate_cents INTEGER NOT NULL DEFAULT 0,
    cost_rate_cents INTEGER NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    expense_type TEXT CHECK(expense_type IN ('TRAVEL','MEALS','ACCOMMODATION','MATERIALS','OTHER')),
    expense_amount_cents INTEGER NOT NULL DEFAULT 0,
    expense_billable INTEGER NOT NULL DEFAULT 1,
    description TEXT NOT NULL,
    approved_by TEXT,
    approved_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED','INVOICED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(id)
);

CREATE INDEX idx_ps_te_engagement ON ps_time_expense(engagement_id, entry_date DESC);
CREATE INDEX idx_ps_te_resource ON ps_time_expense(resource_id, entry_date DESC);
CREATE INDEX idx_ps_te_type ON ps_time_expense(entry_type);
CREATE INDEX idx_ps_te_status ON ps_time_expense(tenant_id, status);
```

### 2.3 Resource Utilization
```sql
CREATE TABLE ps_utilization (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    period TEXT NOT NULL,
    total_available_hours REAL NOT NULL DEFAULT 0,
    billable_hours REAL NOT NULL DEFAULT 0,
    non_billable_hours REAL NOT NULL DEFAULT 0,
    admin_hours REAL NOT NULL DEFAULT 0,
    biz_dev_hours REAL NOT NULL DEFAULT 0,
    training_hours REAL NOT NULL DEFAULT 0,
    pto_hours REAL NOT NULL DEFAULT 0,
    billable_utilization_pct REAL NOT NULL DEFAULT 0,
    total_utilization_pct REAL NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    margin_pct REAL NOT NULL DEFAULT 0,
    target_utilization_pct REAL NOT NULL DEFAULT 80,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, resource_id, period)
);

CREATE INDEX idx_ps_util_resource ON ps_utilization(resource_id, period DESC);
CREATE INDEX idx_ps_util_period ON ps_utilization(period DESC);
```

### 2.4 Work-in-Progress
```sql
CREATE TABLE ps_wip (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    engagement_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    period TEXT NOT NULL,
    unbilled_revenue_cents INTEGER NOT NULL DEFAULT 0,
    unbilled_cost_cents INTEGER NOT NULL DEFAULT 0,
    recognized_revenue_cents INTEGER NOT NULL DEFAULT 0,
    billed_amount_cents INTEGER NOT NULL DEFAULT 0,
    written_off_cents INTEGER NOT NULL DEFAULT 0,
    aging_days INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','BILLED','WRITTEN_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, engagement_id, resource_id, period)
);

CREATE INDEX idx_ps_wip_engagement ON ps_wip(engagement_id, status);
CREATE INDEX idx_ps_wip_aging ON ps_wip(aging_days DESC);
```

### 2.5 Practice Analytics
```sql
CREATE TABLE ps_practice_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    practice_area TEXT NOT NULL,
    period TEXT NOT NULL,
    total_headcount INTEGER NOT NULL DEFAULT 0,
    avg_billable_utilization_pct REAL NOT NULL DEFAULT 0,
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    gross_margin_pct REAL NOT NULL DEFAULT 0,
    avg_realization_rate_pct REAL NOT NULL DEFAULT 0,
    avg_bill_rate_cents INTEGER NOT NULL DEFAULT 0,
    new_engagements INTEGER NOT NULL DEFAULT 0,
    completed_engagements INTEGER NOT NULL DEFAULT 0,
    active_engagements INTEGER NOT NULL DEFAULT 0,
    pipeline_value_cents INTEGER NOT NULL DEFAULT 0,
    win_rate_pct REAL NOT NULL DEFAULT 0,
    client_satisfaction_score REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, practice_area, period)
);

CREATE INDEX idx_ps_pa_practice ON ps_practice_analytics(practice_area, period DESC);
```

---

## 3. API Endpoints

### 3.1 Engagements
| Method | Path | Description |
|--------|------|-------------|
| POST | `/professional-services/v1/engagements` | Create engagement |
| GET | `/professional-services/v1/engagements` | List engagements |
| GET | `/professional-services/v1/engagements/{id}` | Get engagement |
| PUT | `/professional-services/v1/engagements/{id}` | Update engagement |
| GET | `/professional-services/v1/engagements/{id}/profitability` | Profitability analysis |

### 3.2 Time & Expense
| Method | Path | Description |
|--------|------|-------------|
| POST | `/professional-services/v1/time-entries` | Submit time entry |
| POST | `/professional-services/v1/expenses` | Submit expense |
| GET | `/professional-services/v1/time-entries` | List entries |
| POST | `/professional-services/v1/time-entries/{id}/approve` | Approve entry |

### 3.3 Utilization
| Method | Path | Description |
|--------|------|-------------|
| GET | `/professional-services/v1/utilization` | Utilization dashboard |
| GET | `/professional-services/v1/utilization/{resourceId}` | Resource utilization |
| GET | `/professional-services/v1/utilization/team` | Team utilization |

### 3.4 WIP Management
| Method | Path | Description |
|--------|------|-------------|
| GET | `/professional-services/v1/wip` | WIP summary |
| GET | `/professional-services/v1/wip/{engagementId}` | Engagement WIP |
| POST | `/professional-services/v1/wip/invoice` | Generate invoice from WIP |

### 3.5 Practice Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/professional-services/v1/analytics/dashboard` | Practice dashboard |
| GET | `/professional-services/v1/analytics/pipeline` | Pipeline analysis |
| GET | `/professional-services/v1/analytics/margins` | Margin trends |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `psvc.engagement.won` | `{ engagement_id, client, value }` | Engagement won |
| `psvc.time.submitted` | `{ entry_id, resource, hours, billable }` | Time entry submitted |
| `psvc.wip.threshold` | `{ engagement_id, unbilled_amount, days }` | WIP aging threshold |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `opportunity.closed_won` | Sales (77) | Create engagement |
| `invoice.sent` | AR (08) | Update WIP status |
| `resource.assigned` | Resource Mgmt (160) | Update utilization |

---

## 5. Business Rules

1. **Utilization Targets**: Individual utilization targets tracked; underutilization flagged weekly
2. **WIP Aging**: Unbilled WIP exceeding 60 days flagged for review
3. **Billing Realization**: Actual billed vs standard rates tracked; write-downs require approval
4. **Engagement Margins**: Gross margin monitored against practice benchmarks
5. **Time Entry Compliance**: Time entries required daily; missing entries flagged
6. **Fixed Fee Revenue**: Revenue recognized based on completion percentage for fixed-fee engagements

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Project Management (15) | Project structure and tasks |
| Resource Management (160) | Resource allocation |
| Accounts Receivable (08) | Invoicing |
| Project Costing (158) | Cost tracking |
| Sales Automation (77) | Opportunity pipeline |
| HCM Analytics (148) | Workforce metrics |
