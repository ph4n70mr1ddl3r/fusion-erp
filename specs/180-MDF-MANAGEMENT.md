# 180 - MDF Management Service Specification

## 1. Domain Overview

MDF (Market Development Funds) Management provides fund allocation, claim management, accrual tracking, and ROI measurement for channel partner marketing programs. Supports MDF budget allocation to partners, fund request/approval workflows, proof-of-execution documentation, claim submission and validation, payment processing, and program ROI analysis. Enables vendors to manage co-marketing investments with channel partners, track fund utilization, and measure marketing program effectiveness. Integrates with Channel Management, Accounts Payable, Marketing, and Procurement.

**Bounded Context:** Market Development Funds & Partner Co-Marketing
**Service Name:** `mdf-service`
**Database:** `data/mdf.db`
**HTTP Port:** 8198 | **gRPC Port:** 9198

---

## 2. Database Schema

### 2.1 MDF Programs
```sql
CREATE TABLE mdf_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_name TEXT NOT NULL,
    program_type TEXT NOT NULL CHECK(program_type IN ('ANNUAL','QUARTERLY','CAMPAIGN','LAUNCH','REBATE','CUSTOM')),
    description TEXT,
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    allocated_cents INTEGER NOT NULL DEFAULT 0,
    claimed_cents INTEGER NOT NULL DEFAULT 0,
    paid_cents INTEGER NOT NULL DEFAULT 0,
    remaining_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    eligibility_criteria TEXT NOT NULL,       -- JSON: partner eligibility rules
    allowed_activities TEXT NOT NULL,        -- JSON: eligible marketing activities
    max_claim_pct REAL NOT NULL DEFAULT 100,
    co_fund_pct REAL NOT NULL DEFAULT 50,   -- Vendor contribution percentage
    approval_workflow TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','CLOSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_code)
);

CREATE INDEX idx_mdf_prog_tenant ON mdf_programs(tenant_id, status);
CREATE INDEX idx_mdf_prog_date ON mdf_programs(start_date, end_date);
```

### 2.2 Partner Allocations
```sql
CREATE TABLE mdf_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    partner_name TEXT NOT NULL,
    allocated_amount_cents INTEGER NOT NULL DEFAULT 0,
    claimed_amount_cents INTEGER NOT NULL DEFAULT 0,
    paid_amount_cents INTEGER NOT NULL DEFAULT 0,
    remaining_amount_cents INTEGER NOT NULL DEFAULT 0,
    allocation_date TEXT NOT NULL,
    earned_pct REAL NOT NULL DEFAULT 0,     -- % earned based on performance
    performance_metrics TEXT,                -- JSON: metrics determining allocation
    special_conditions TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXHAUSTED','EXPIRED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (program_id) REFERENCES mdf_programs(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, program_id, partner_id)
);

CREATE INDEX idx_mdf_alloc_partner ON mdf_allocations(partner_id, status);
CREATE INDEX idx_mdf_alloc_program ON mdf_allocations(program_id);
```

### 2.3 Fund Requests
```sql
CREATE TABLE mdf_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    allocation_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    request_number TEXT NOT NULL,            -- MDF-REQ-2024-00001
    activity_type TEXT NOT NULL CHECK(activity_type IN (
        'EVENT','DIGITAL_MARKETING','PRINT_ADVERTISING','TRAINING',
        'DEMO_EQUIPMENT','CO_BRANDED_MATERIAL','WEBINAR','SOCIAL_MEDIA','OTHER'
    )),
    activity_name TEXT NOT NULL,
    activity_description TEXT NOT NULL,
    planned_start_date TEXT NOT NULL,
    planned_end_date TEXT NOT NULL,
    requested_amount_cents INTEGER NOT NULL,
    partner_contribution_cents INTEGER NOT NULL DEFAULT 0,
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    expected_outcomes TEXT NOT NULL,         -- JSON: expected leads, pipeline, etc.
    target_audience TEXT,
    geography TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','APPROVED','PARTIALLY_APPROVED','REJECTED','CANCELLED')),
    approved_amount_cents INTEGER,
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_comments TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, request_number)
);

CREATE INDEX idx_mdf_req_program ON mdf_requests(program_id, status);
CREATE INDEX idx_mdf_req_partner ON mdf_requests(partner_id, status);
```

### 2.4 Claims
```sql
CREATE TABLE mdf_claims (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_id TEXT NOT NULL,
    allocation_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    claim_number TEXT NOT NULL,              -- MDF-CLM-2024-00001
    activity_completion_date TEXT NOT NULL,
    claimed_amount_cents INTEGER NOT NULL,
    actual_spent_cents INTEGER NOT NULL,
    invoice_reference TEXT,
    proof_of_execution TEXT NOT NULL,        -- JSON: document IDs (photos, receipts, reports)
    actual_outcomes TEXT NOT NULL,           -- JSON: actual results vs planned
    roi_analysis TEXT,                       -- JSON: ROI calculation
    status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(status IN ('SUBMITTED','UNDER_REVIEW','APPROVED','PARTIALLY_APPROVED','REJECTED','PAID','DISPUTED')),
    approved_amount_cents INTEGER,
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_comments TEXT,
    payment_reference TEXT,
    paid_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, claim_number)
);

CREATE INDEX idx_mdf_claim_partner ON mdf_claims(partner_id, status);
CREATE INDEX idx_mdf_claim_request ON mdf_claims(request_id);
CREATE INDEX idx_mdf_claim_status ON mdf_claims(tenant_id, status);
```

### 2.5 ROI Analysis
```sql
CREATE TABLE mdf_roi_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    period TEXT NOT NULL,
    total_investment_cents INTEGER NOT NULL DEFAULT 0,
    pipeline_generated_cents INTEGER NOT NULL DEFAULT 0,
    revenue_attributed_cents INTEGER NOT NULL DEFAULT 0,
    leads_generated INTEGER NOT NULL DEFAULT 0,
    opportunities_created INTEGER NOT NULL DEFAULT 0,
    deals_closed INTEGER NOT NULL DEFAULT 0,
    roi_pct REAL NOT NULL DEFAULT 0,
    cost_per_lead_cents INTEGER NOT NULL DEFAULT 0,
    cost_per_opportunity_cents INTEGER NOT NULL DEFAULT 0,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, program_id, partner_id, period)
);

CREATE INDEX idx_mdf_roi_program ON mdf_roi_analysis(program_id, period DESC);
```

---

## 3. API Endpoints

### 3.1 Programs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mdf/v1/programs` | Create MDF program |
| GET | `/mdf/v1/programs` | List programs |
| GET | `/mdf/v1/programs/{id}` | Get program details |
| PUT | `/mdf/v1/programs/{id}` | Update program |
| POST | `/mdf/v1/programs/{id}/activate` | Activate program |

### 3.2 Allocations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mdf/v1/allocations` | Allocate funds to partner |
| GET | `/mdf/v1/allocations` | List allocations |
| PUT | `/mdf/v1/allocations/{id}` | Update allocation |

### 3.3 Requests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mdf/v1/requests` | Submit fund request |
| GET | `/mdf/v1/requests` | List requests |
| POST | `/mdf/v1/requests/{id}/approve` | Approve request |
| POST | `/mdf/v1/requests/{id}/reject` | Reject request |

### 3.4 Claims
| Method | Path | Description |
|--------|------|-------------|
| POST | `/mdf/v1/claims` | Submit claim |
| GET | `/mdf/v1/claims` | List claims |
| GET | `/mdf/v1/claims/{id}` | Get claim details |
| POST | `/mdf/v1/claims/{id}/approve` | Approve claim |
| POST | `/mdf/v1/claims/{id}/reject` | Reject claim |
| POST | `/mdf/v1/claims/{id}/pay` | Process payment |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/mdf/v1/dashboard` | MDF dashboard |
| GET | `/mdf/v1/roi` | ROI analysis |
| GET | `/mdf/v1/utilization` | Fund utilization |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `mdf.allocation.created` | `{ allocation_id, partner_id, amount }` | Funds allocated |
| `mdf.request.approved` | `{ request_id, approved_amount }` | Fund request approved |
| `mdf.claim.submitted` | `{ claim_id, partner_id, amount }` | Claim submitted |
| `mdf.claim.paid` | `{ claim_id, payment_ref }` | Claim paid |
| `mdf.program.closing` | `{ program_id, days_remaining }` | Program closing soon |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `opportunity.closed_won` | Sales Automation | Attribute revenue to MDF program |
| `partner.qualified` | Channel Management | Make partner eligible for MDF |

---

## 5. Business Rules

1. **Eligibility**: Partners must meet program eligibility criteria before allocation
2. **Co-Funding**: Partner must contribute minimum co-fund percentage
3. **Pre-Approval**: Fund requests must be approved before activity execution
4. **Proof Required**: Claims require proof of execution documentation
5. **Claim Deadline**: Claims must be submitted within 60 days of activity completion
6. **Budget Control**: Claims cannot exceed allocated amount
7. **Payment**: Approved claims paid through AP integration

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Channel Management (84) | Partner data and qualification |
| Accounts Payable (07) | Claim payment processing |
| Marketing (61) | Activity templates and content |
| Sales Automation (77) | Pipeline and revenue attribution |
| Workflow (16) | Approval workflows |
| Procurement (11) | Vendor setup for payments |
| Document Management (29) | Proof of execution storage |
