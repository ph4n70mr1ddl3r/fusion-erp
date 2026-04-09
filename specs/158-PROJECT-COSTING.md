# 158 - Project Costing Service Specification

## 1. Domain Overview

Project Costing provides comprehensive cost tracking, allocation, capitalization, and analysis for projects. Supports direct and indirect cost capture, burdening calculations, cost-to-completion forecasting, work-in-progress (WIP) accounting, and capitalization rules. Enables project managers and controllers to track actual vs. budgeted costs, manage cost transfers, apply overhead rates, and generate capitalization events for Fixed Assets. Integrates with Project Management, General Ledger, Accounts Payable, Payroll, Inventory, and Fixed Assets.

**Bounded Context:** Project Cost Management & Capitalization
**Service Name:** `project-costing-service`
**Database:** `data/project_costing.db`
**HTTP Port:** 8176 | **gRPC Port:** 9176

---

## 2. Database Schema

### 2.1 Project Cost Budgets
```sql
CREATE TABLE pc_cost_budgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    task_id TEXT,                           -- NULL = project-level
    budget_type TEXT NOT NULL CHECK(budget_type IN ('ORIGINAL','REVISED','FORECAST','AT_COMPLETION')),
    cost_type TEXT NOT NULL CHECK(cost_type IN ('LABOR','MATERIAL','EQUIPMENT','EXPENSE','OVERHEAD','SUBCONTRACT','ALL')),
    budgeted_cost_cents INTEGER NOT NULL DEFAULT 0,
    committed_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    remaining_budget_cents INTEGER NOT NULL DEFAULT 0,
    forecast_to_complete_cents INTEGER NOT NULL DEFAULT 0,
    estimate_at_completion_cents INTEGER NOT NULL DEFAULT 0,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    variance_pct REAL NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    period TEXT NOT NULL,                   -- "2024-01"

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_id, task_id, budget_type, cost_type, period)
);

CREATE INDEX idx_pc_budget_project ON pc_cost_budgets(project_id, period DESC);
CREATE INDEX idx_pc_budget_type ON pc_cost_budgets(tenant_id, budget_type);
```

### 2.2 Cost Transactions
```sql
CREATE TABLE pc_cost_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,       -- PCT-2024-00001
    project_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    cost_type TEXT NOT NULL CHECK(cost_type IN ('LABOR','MATERIAL','EQUIPMENT','EXPENSE','OVERHEAD','SUBCONTRACT')),
    source_type TEXT NOT NULL CHECK(source_type IN (
        'TIMESHEET','EXPENSE_REPORT','INVOICE','INVENTORY_ISSUE','JOURNAL_ENTRY',
        'BURDEN_ALLOCATION','COST_TRANSFER','SYSTEM_GENERATED'
    )),
    source_document_id TEXT NOT NULL,
    source_line_id TEXT NOT NULL,
    description TEXT NOT NULL,
    quantity DECIMAL(18,4),
    uom TEXT,
    raw_cost_cents INTEGER NOT NULL DEFAULT 0,
    burden_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL NOT NULL DEFAULT 1.0,
    transaction_date TEXT NOT NULL,
    accounting_date TEXT NOT NULL,
    gl_account_id TEXT,
    cost_center_id TEXT,
    capitalized INTEGER NOT NULL DEFAULT 0,
    capitalization_event_id TEXT,
    reversal_of_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_pc_cost_project ON pc_cost_transactions(project_id, transaction_date DESC);
CREATE INDEX idx_pc_cost_task ON pc_cost_transactions(task_id);
CREATE INDEX idx_pc_cost_type ON pc_cost_transactions(tenant_id, cost_type);
CREATE INDEX idx_pc_cost_source ON pc_cost_transactions(source_type, source_document_id);
```

### 2.3 Burden Schedules
```sql
CREATE TABLE pc_burden_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    cost_type TEXT NOT NULL,
    burden_cost_code TEXT NOT NULL,
    burden_rate REAL NOT NULL DEFAULT 0,
    rate_type TEXT NOT NULL CHECK(rate_type IN ('PERCENTAGE','FIXED_PER_UNIT','TIERED')),
    tier_config TEXT,                       -- JSON: tiered rates
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, schedule_name)
);
```

### 2.4 Capitalization Rules
```sql
CREATE TABLE pc_capitalization_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    project_type TEXT NOT NULL,             -- CAPEX, INTERNAL, R&D
    cost_types TEXT NOT NULL,               -- JSON: eligible cost types
    minimum_amount_cents INTEGER NOT NULL DEFAULT 0,
    capitalization_threshold_pct REAL NOT NULL DEFAULT 100,
    asset_category_id TEXT,
    asset_book TEXT,
    depreciation_method TEXT,
    useful_life_months INTEGER,
    start_capitalization TEXT NOT NULL CHECK(start_capitalization IN ('IMMEDIATE','ON_PROJECT_COMPLETION','PERIODIC')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, rule_name)
);
```

### 2.5 Capitalization Events
```sql
CREATE TABLE pc_capitalization_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_number TEXT NOT NULL,             -- CAP-2024-00001
    project_id TEXT NOT NULL,
    task_id TEXT,
    capitalization_date TEXT NOT NULL,
    total_capitalized_cents INTEGER NOT NULL DEFAULT 0,
    cost_transaction_ids TEXT NOT NULL,     -- JSON array
    asset_id TEXT,                          -- Created fixed asset
    asset_number TEXT,
    gl_journal_id TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','POSTED','REVERSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, event_number)
);

CREATE INDEX idx_pc_cap_project ON pc_capitalization_events(project_id);
CREATE INDEX idx_pc_cap_status ON pc_capitalization_events(tenant_id, status);
```

---

## 3. API Endpoints

### 3.1 Cost Budgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-costing/v1/budgets` | Set cost budget |
| GET | `/project-costing/v1/budgets` | List budgets |
| GET | `/project-costing/v1/projects/{id}/budget-summary` | Project cost summary |
| PUT | `/project-costing/v1/budgets/{id}` | Update budget |
| POST | `/project-costing/v1/budgets/{id}/revise` | Create budget revision |

### 3.2 Cost Transactions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-costing/v1/transactions` | Record cost transaction |
| GET | `/project-costing/v1/transactions` | List transactions |
| GET | `/project-costing/v1/transactions/{id}` | Get transaction details |
| POST | `/project-costing/v1/transactions/{id}/reverse` | Reverse transaction |
| POST | `/project-costing/v1/transactions/bulk-import` | Bulk import costs |

### 3.3 Burden & Overhead
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-costing/v1/burden-schedules` | Create burden schedule |
| GET | `/project-costing/v1/burden-schedules` | List schedules |
| POST | `/project-costing/v1/projects/{id}/apply-burden` | Apply burden to costs |

### 3.4 Capitalization
| Method | Path | Description |
|--------|------|-------------|
| POST | `/project-costing/v1/capitalization-rules` | Create rule |
| GET | `/project-costing/v1/capitalization-rules` | List rules |
| POST | `/project-costing/v1/capitalization-events` | Create capitalization event |
| GET | `/project-costing/v1/capitalization-events` | List events |
| POST | `/project-costing/v1/capitalization-events/{id}/post` | Post to GL and FA |
| POST | `/project-costing/v1/capitalization-events/{id}/reverse` | Reverse capitalization |

### 3.5 Cost Analysis
| Method | Path | Description |
|--------|------|-------------|
| GET | `/project-costing/v1/projects/{id}/cost-analysis` | Cost analysis report |
| GET | `/project-costing/v1/projects/{id}/cost-trend` | Cost trend |
| GET | `/project-costing/v1/projects/{id}/forecast` | Cost forecast |
| GET | `/project-costing/v1/projects/{id}/wip` | WIP summary |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `pcost.transaction.created` | `{ transaction_id, project_id, amount }` | Cost transaction recorded |
| `pcost.transaction.reversed` | `{ transaction_id, reversal_id }` | Cost reversed |
| `pcost.budget.exceeded` | `{ project_id, task_id, variance_pct }` | Budget exceeded threshold |
| `pcost.capitalization.posted` | `{ event_id, asset_id, amount }` | Cost capitalized to asset |
| `pcost.burden.applied` | `{ project_id, burden_amount }` | Burden overhead applied |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `timesheet.approved` | Time & Labor | Create labor cost transaction |
| `expense.report.approved` | Expense Mgmt | Create expense cost transaction |
| `invoice.matched` | AP | Create material/subcontract cost |
| `inventory.issued` | Inventory | Create material cost transaction |
| `project.completed` | Project Mgmt | Trigger final capitalization |

---

## 5. Business Rules

1. **Budget Control**: Configurable budget tolerance; hard block or warning on overrun
2. **Burden Calculation**: Overhead applied automatically based on burden schedule rates
3. **Capitalization Threshold**: Costs below minimum amount remain expensed
4. **Cost Transfer Approval**: All cost transfers require approval with justification
5. **Reversal Tracking**: All cost reversals link to original transaction with audit trail
6. **Multi-Currency**: Foreign currency costs converted using transaction date exchange rate
7. **WIP Valuation**: Work-in-progress valued at actual cost + applied burden
8. **Forecast Method**: EAC = actuals + (remaining budget × cost performance index)

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Project Management (15) | Project/task structure, status |
| General Ledger (06) | GL posting for cost transactions |
| Accounts Payable (07) | Invoice-based cost capture |
| Time & Labor (64) | Labor cost from timesheets |
| Expense Management (25) | Expense report costs |
| Inventory (12) | Material issue costs |
| Fixed Assets (09) | Asset capitalization |
| Cost Management (48) | Standard cost data |
| Payroll (63) | Labor rate data |
