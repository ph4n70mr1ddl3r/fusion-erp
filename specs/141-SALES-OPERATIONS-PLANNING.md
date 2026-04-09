# 141 - Sales & Operations Planning (S&OP) Service Specification

## 1. Domain Overview

Sales & Operations Planning (S&OP) provides cross-functional consensus planning that aligns demand, supply, financial, and strategic objectives. Supports multi-tier planning hierarchies, scenario modeling, gap analysis, and executive review workflows. Enables monthly S&OP cycles with demand review, supply review, financial integration, and executive approval phases. Integrates with Demand Management for demand inputs, Supply Chain Planning for supply constraints, EPM for financial targets, and all functional planning modules.

**Bounded Context:** Cross-Functional Consensus Planning
**Service Name:** `sop-service`
**Database:** `data/sop.db`
**HTTP Port:** 8159 | **gRPC Port:** 9159

---

## 2. Database Schema

### 2.1 S&OP Cycles
```sql
CREATE TABLE sop_cycles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_name TEXT NOT NULL,               -- "S&OP 2024-Q1"
    planning_horizon_months INTEGER NOT NULL DEFAULT 18,
    granularity TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(granularity IN ('WEEKLY','MONTHLY','QUARTERLY')),
    status TEXT NOT NULL DEFAULT 'INITIATED'
        CHECK(status IN ('INITIATED','DATA_GATHERING','DEMAND_REVIEW','SUPPLY_REVIEW',
                         'PRE_SOP','EXECUTIVE_SOP','APPROVED','COMPLETED','ARCHIVED')),
    fiscal_year TEXT NOT NULL,
    planning_period_start TEXT NOT NULL,
    planning_period_end TEXT NOT NULL,
    current_phase TEXT NOT NULL DEFAULT 'DATA_GATHERING',
    phase_deadlines TEXT NOT NULL,          -- JSON: { "DEMAND_REVIEW": "2024-01-15", ... }
    demand_plan_id TEXT,
    supply_plan_id TEXT,
    financial_plan_id TEXT,
    executive_summary TEXT,
    meeting_date TEXT,
    meeting_participants TEXT,              -- JSON array of user IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, cycle_name)
);

CREATE INDEX idx_sop_cycle_tenant ON sop_cycles(tenant_id, fiscal_year, status);
```

### 2.2 S&OP Scenarios
```sql
CREATE TABLE sop_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    scenario_type TEXT NOT NULL CHECK(scenario_type IN ('BASELINE','BEST_CASE','WORST_CASE','WHATIF','CONSENSUS')),
    description TEXT,
    demand_plan_ref TEXT,                   -- Reference to demand plan
    supply_plan_ref TEXT,                   -- Reference to supply plan
    financial_plan_ref TEXT,                -- Reference to financial plan
    assumptions TEXT NOT NULL,              -- JSON: scenario-specific assumptions
    constraints TEXT,                       -- JSON: capacity, material, financial constraints
    is_approved INTEGER NOT NULL DEFAULT 0,
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES sop_cycles(id) ON DELETE CASCADE
);

CREATE INDEX idx_sop_scenario_cycle ON sop_scenarios(cycle_id);
CREATE INDEX idx_sop_scenario_tenant ON sop_scenarios(tenant_id, scenario_type);
```

### 2.3 S&OP Plan Lines
```sql
CREATE TABLE sop_plan_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    scenario_id TEXT NOT NULL,
    dimension_type TEXT NOT NULL CHECK(dimension_type IN (
        'PRODUCT_FAMILY','PRODUCT','CUSTOMER','REGION','CHANNEL','BUSINESS_UNIT'
    )),
    dimension_id TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-01"
    demand_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    supply_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    gap_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    margin_cents INTEGER NOT NULL DEFAULT 0,
    headcount INTEGER NOT NULL DEFAULT 0,
    capacity_utilization_pct REAL NOT NULL DEFAULT 0,
    inventory_days_supply REAL NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (scenario_id) REFERENCES sop_scenarios(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, scenario_id, dimension_type, dimension_id, period)
);

CREATE INDEX idx_sop_line_scenario ON sop_plan_lines(scenario_id, period);
CREATE INDEX idx_sop_line_dimension ON sop_plan_lines(tenant_id, dimension_type, dimension_id);
```

### 2.4 S&OP Gap Analysis
```sql
CREATE TABLE sop_gap_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    scenario_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    period TEXT NOT NULL,
    gap_type TEXT NOT NULL CHECK(gap_type IN ('DEMAND_SUPPLY','REVENUE_TARGET','MARGIN_TARGET','CAPACITY','INVENTORY','HEADCOUNT')),
    gap_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    gap_amount_cents INTEGER NOT NULL DEFAULT 0,
    gap_direction TEXT NOT NULL CHECK(gap_direction IN ('SHORTFALL','SURPLUS')),
    severity TEXT NOT NULL CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    recommended_actions TEXT,               -- JSON array of action recommendations
    resolution_status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(resolution_status IN ('OPEN','IN_PROGRESS','RESOLVED','ACCEPTED','ESCALATED')),
    resolved_by TEXT,
    resolution_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (cycle_id) REFERENCES sop_cycles(id) ON DELETE CASCADE,
    FOREIGN KEY (scenario_id) REFERENCES sop_scenarios(id) ON DELETE CASCADE
);

CREATE INDEX idx_sop_gap_cycle ON sop_gap_analysis(cycle_id, gap_type);
CREATE INDEX idx_sop_gap_severity ON sop_gap_analysis(tenant_id, severity, resolution_status);
```

### 2.5 S&OP Action Items
```sql
CREATE TABLE sop_action_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    scenario_id TEXT,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN (
        'CAPACITY_ADJUSTMENT','INVESTMENT','HIRING','PRICING_CHANGE',
        'PROMOTION','OUTSOURCE','DEFER','ACCELERATE','OTHER'
    )),
    owner_id TEXT NOT NULL,
    due_date TEXT NOT NULL,
    priority TEXT NOT NULL CHECK(priority IN ('LOW','MEDIUM','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_PROGRESS','COMPLETED','CANCELLED')),
    expected_impact TEXT,                   -- JSON: expected quantity/financial impact
    actual_impact TEXT,                     -- JSON: actual results after implementation

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cycle_id) REFERENCES sop_cycles(id) ON DELETE CASCADE
);

CREATE INDEX idx_sop_action_cycle ON sop_action_items(cycle_id, status);
CREATE INDEX idx_sop_action_owner ON sop_action_items(tenant_id, owner_id, status);
```

### 2.6 S&OP Meeting Minutes
```sql
CREATE TABLE sop_meetings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_id TEXT NOT NULL,
    meeting_type TEXT NOT NULL CHECK(meeting_type IN ('DEMAND_REVIEW','SUPPLY_REVIEW','PRE_SOP','EXECUTIVE_SOP')),
    meeting_date TEXT NOT NULL,
    facilitator_id TEXT NOT NULL,
    attendees TEXT NOT NULL,                -- JSON array of user IDs
    agenda TEXT,                            -- JSON array of agenda items
    key_decisions TEXT,                     -- JSON array of decisions
    open_issues TEXT,                       -- JSON array of issues
    action_items TEXT,                      -- JSON array of action item IDs
    recording_url TEXT,
    status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(status IN ('SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (cycle_id) REFERENCES sop_cycles(id) ON DELETE CASCADE
);

CREATE INDEX idx_sop_meeting_cycle ON sop_meetings(cycle_id, meeting_type);
```

---

## 3. API Endpoints

### 3.1 S&OP Cycles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sop/v1/cycles` | Create S&OP cycle |
| GET | `/sop/v1/cycles` | List cycles |
| GET | `/sop/v1/cycles/{id}` | Get cycle details |
| PUT | `/sop/v1/cycles/{id}` | Update cycle |
| POST | `/sop/v1/cycles/{id}/advance-phase` | Advance to next phase |
| POST | `/sop/v1/cycles/{id}/approve` | Executive approval |

### 3.2 Scenarios
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sop/v1/scenarios` | Create scenario |
| GET | `/sop/v1/scenarios` | List scenarios |
| GET | `/sop/v1/scenarios/{id}` | Get scenario with plan lines |
| PUT | `/sop/v1/scenarios/{id}` | Update scenario |
| POST | `/sop/v1/scenarios/{id}/copy` | Copy scenario |
| POST | `/sop/v1/scenarios/compare` | Compare scenarios side-by-side |

### 3.3 Plan Lines
| Method | Path | Description |
|--------|------|-------------|
| GET | `/sop/v1/scenarios/{id}/lines` | Get plan lines |
| PUT | `/sop/v1/scenarios/{id}/lines` | Update plan lines |
| POST | `/sop/v1/scenarios/{id}/lines/import` | Import from demand/supply plans |

### 3.4 Gap Analysis
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sop/v1/cycles/{id}/gap-analysis` | Run gap analysis |
| GET | `/sop/v1/cycles/{id}/gap-analysis` | Get gap analysis results |
| PUT | `/sop/v1/gaps/{id}/resolve` | Resolve gap |

### 3.5 Meetings & Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sop/v1/meetings` | Schedule S&OP meeting |
| GET | `/sop/v1/meetings` | List meetings |
| PUT | `/sop/v1/meetings/{id}` | Update meeting details |
| POST | `/sop/v1/action-items` | Create action item |
| PUT | `/sop/v1/action-items/{id}` | Update action item |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `sop.cycle.created` | `{ cycle_id, fiscal_year, horizon }` | New S&OP cycle initiated |
| `sop.phase.advanced` | `{ cycle_id, from_phase, to_phase }` | Cycle phase advanced |
| `sop.cycle.approved` | `{ cycle_id, approved_by, consensus_plan_id }` | Executive S&OP approved |
| `sop.gap.detected` | `{ cycle_id, gap_type, dimension, severity }` | Planning gap identified |
| `sop.gap.resolved` | `{ gap_id, resolution }` | Gap resolved |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `demand.consensus.published` | Demand Management | Load demand plan into cycle |
| `supply.plan.finalized` | Supply Chain Planning | Load supply plan into cycle |
| `epm.budget.approved` | EPM | Load financial targets |
| `capacity.constraint.detected` | Manufacturing | Update supply constraints |

---

## 5. Business Rules

1. **Phase Gate**: Each S&OP phase must be completed before advancing; no skipping phases
2. **Consensus Requirement**: At least one CONSENSUS scenario must exist before executive S&OP
3. **Gap Closure**: All CRITICAL gaps must have resolution plans before executive approval
4. **Scenario Versioning**: Each scenario modification creates a new version; history preserved
5. **Financial Alignment**: Revenue plan must be within 5% of EPM budget targets or escalation required
6. **Approval Authority**: Executive S&OP approval requires VP-level or above authorization
7. **Cycle Lockdown**: Approved cycles are locked; changes require new supplemental cycle

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Demand Management (140) | Demand plan input |
| Supply Chain Planning (28) | Supply plan input and constraints |
| EPM (33) | Financial targets and budget alignment |
| Manufacturing (14) | Capacity data and constraints |
| Inventory (12) | Current inventory positions |
| Order Management (13) | Actual order performance |
| Marketing (61) | Promotional calendar |
| Workflow (16) | Phase approval workflows |
| Reporting (17) | S&OP dashboards and analytics |
