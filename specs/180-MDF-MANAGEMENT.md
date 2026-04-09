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
| POST | `/api/v1/mdf/programs` | Create MDF program |
| GET | `/api/v1/mdf/programs` | List programs |
| GET | `/api/v1/mdf/programs/{id}` | Get program details |
| PUT | `/api/v1/mdf/programs/{id}` | Update program |
| POST | `/api/v1/mdf/programs/{id}/activate` | Activate program |

### 3.2 Allocations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/mdf/allocations` | Allocate funds to partner |
| GET | `/api/v1/mdf/allocations` | List allocations |
| PUT | `/api/v1/mdf/allocations/{id}` | Update allocation |

### 3.3 Requests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/mdf/requests` | Submit fund request |
| GET | `/api/v1/mdf/requests` | List requests |
| POST | `/api/v1/mdf/requests/{id}/approve` | Approve request |
| POST | `/api/v1/mdf/requests/{id}/reject` | Reject request |

### 3.4 Claims
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/mdf/claims` | Submit claim |
| GET | `/api/v1/mdf/claims` | List claims |
| GET | `/api/v1/mdf/claims/{id}` | Get claim details |
| POST | `/api/v1/mdf/claims/{id}/approve` | Approve claim |
| POST | `/api/v1/mdf/claims/{id}/reject` | Reject claim |
| POST | `/api/v1/mdf/claims/{id}/pay` | Process payment |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mdf/dashboard` | MDF dashboard |
| GET | `/api/v1/mdf/roi` | ROI analysis |
| GET | `/api/v1/mdf/utilization` | Fund utilization |

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

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.mdf.v1;

service MdfManagementService {
    rpc GetProgram(GetProgramRequest) returns (GetProgramResponse);
    rpc CreateProgram(CreateProgramRequest) returns (CreateProgramResponse);
    rpc SubmitFundRequest(SubmitFundRequestRequest) returns (SubmitFundRequestResponse);
    rpc SubmitClaim(SubmitClaimRequest) returns (SubmitClaimResponse);
    rpc ApproveClaim(ApproveClaimRequest) returns (ApproveClaimResponse);
    rpc GetRoiAnalysis(GetRoiAnalysisRequest) returns (GetRoiAnalysisResponse);
}

// Program messages
message GetProgramRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetProgramResponse {
    MdfProgram data = 1;
}

message CreateProgramRequest {
    string tenant_id = 1;
    string program_code = 2;
    string program_name = 3;
    string program_type = 4;
    string description = 5;
    int64 total_budget_cents = 6;
    string currency_code = 7;
    string start_date = 8;
    string end_date = 9;
    string eligibility_criteria = 10;
    string allowed_activities = 11;
    double max_claim_pct = 12;
    double co_fund_pct = 13;
}

message CreateProgramResponse {
    MdfProgram data = 1;
}

message MdfProgram {
    string id = 1;
    string tenant_id = 2;
    string program_code = 3;
    string program_name = 4;
    string program_type = 5;
    string description = 6;
    int64 total_budget_cents = 7;
    int64 allocated_cents = 8;
    int64 claimed_cents = 9;
    int64 paid_cents = 10;
    int64 remaining_cents = 11;
    string currency_code = 12;
    string start_date = 13;
    string end_date = 14;
    string eligibility_criteria = 15;
    string allowed_activities = 16;
    double max_claim_pct = 17;
    double co_fund_pct = 18;
    string status = 19;
    string created_at = 20;
    string updated_at = 21;
}

// Fund request messages
message SubmitFundRequestRequest {
    string tenant_id = 1;
    string program_id = 2;
    string allocation_id = 3;
    string partner_id = 4;
    string activity_type = 5;
    string activity_name = 6;
    string activity_description = 7;
    string planned_start_date = 8;
    string planned_end_date = 9;
    int64 requested_amount_cents = 10;
    int64 partner_contribution_cents = 11;
    int64 total_budget_cents = 12;
    string expected_outcomes = 13;
}

message SubmitFundRequestResponse {
    MdfFundRequest data = 1;
}

message MdfFundRequest {
    string id = 1;
    string tenant_id = 2;
    string program_id = 3;
    string allocation_id = 4;
    string partner_id = 5;
    string request_number = 6;
    string activity_type = 7;
    string activity_name = 8;
    string activity_description = 9;
    string planned_start_date = 10;
    string planned_end_date = 11;
    int64 requested_amount_cents = 12;
    int64 partner_contribution_cents = 13;
    int64 total_budget_cents = 14;
    string expected_outcomes = 15;
    string status = 16;
    int64 approved_amount_cents = 17;
    string reviewed_by = 18;
    string reviewed_at = 19;
    string created_at = 20;
    string updated_at = 21;
}

// Claim messages
message SubmitClaimRequest {
    string tenant_id = 1;
    string request_id = 2;
    string allocation_id = 3;
    string partner_id = 4;
    string activity_completion_date = 5;
    int64 claimed_amount_cents = 6;
    int64 actual_spent_cents = 7;
    string invoice_reference = 8;
    string proof_of_execution = 9;
    string actual_outcomes = 10;
}

message SubmitClaimResponse {
    MdfClaim data = 1;
}

message ApproveClaimRequest {
    string tenant_id = 1;
    string id = 2;
    int64 approved_amount_cents = 3;
    string review_comments = 4;
}

message ApproveClaimResponse {
    string id = 1;
    string status = 2;
    int64 approved_amount_cents = 3;
}

message MdfClaim {
    string id = 1;
    string tenant_id = 2;
    string request_id = 3;
    string allocation_id = 4;
    string partner_id = 5;
    string claim_number = 6;
    string activity_completion_date = 7;
    int64 claimed_amount_cents = 8;
    int64 actual_spent_cents = 9;
    string invoice_reference = 10;
    string proof_of_execution = 11;
    string actual_outcomes = 12;
    string roi_analysis = 13;
    string status = 14;
    int64 approved_amount_cents = 15;
    string payment_reference = 16;
    string paid_at = 17;
    string created_at = 18;
    string updated_at = 19;
}

// ROI messages
message GetRoiAnalysisRequest {
    string tenant_id = 1;
    string program_id = 2;
    string partner_id = 3;
    string period = 4;
}

message GetRoiAnalysisResponse {
    MdfRoiAnalysis data = 1;
}

message MdfRoiAnalysis {
    string id = 1;
    string tenant_id = 2;
    string program_id = 3;
    string partner_id = 4;
    string period = 5;
    int64 total_investment_cents = 6;
    int64 pipeline_generated_cents = 7;
    int64 revenue_attributed_cents = 8;
    int32 leads_generated = 9;
    int32 opportunities_created = 10;
    int32 deals_closed = 11;
    double roi_pct = 12;
    int64 cost_per_lead_cents = 13;
    int64 cost_per_opportunity_cents = 14;
    string notes = 15;
    string created_at = 16;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | mdf_programs | -- |
| V002 | mdf_allocations | V001 |
| V003 | mdf_requests | V001, V002 |
| V004 | mdf_claims | V003, V002 |
| V005 | mdf_roi_analysis | V001 |

---

## 7. Business Rules

1. **Eligibility**: Partners must meet program eligibility criteria before allocation
2. **Co-Funding**: Partner must contribute minimum co-fund percentage
3. **Pre-Approval**: Fund requests must be approved before activity execution
4. **Proof Required**: Claims require proof of execution documentation
5. **Claim Deadline**: Claims must be submitted within 60 days of activity completion
6. **Budget Control**: Claims cannot exceed allocated amount
7. **Payment**: Approved claims paid through AP integration

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| channel-service | `GetPartner` | Partner data and qualification |
| ap-service | `CreateInvoice` | Claim payment processing |
| marketing-service | `GetTemplate` | Activity templates and content |
| sales-service | `GetOpportunity` | Pipeline and revenue attribution |
| workflow-service | `SubmitApproval` | Approval workflows |
| procurement-service | `GetVendor` | Vendor setup for payments |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| document-service | `GetDocument` | Proof of execution storage |
| sales-service | `GetRoiAnalysis` | Pipeline revenue attribution |
