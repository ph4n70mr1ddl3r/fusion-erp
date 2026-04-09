# 194 - Oil & Gas Industry Solution Service Specification

## 1. Domain Overview

Oil & Gas Industry Solution provides comprehensive upstream and midstream operations management supporting well lifecycle tracking from exploration through production with working interest ownership, joint venture accounting with partner ownership percentages, cost sharing rules, and operator designation, production recording by well and field with oil, gas, and water volume tracking, lifting costs, royalties, and severance taxes, lease agreement management with mineral rights, royalty rates, and renewal options, and cost allocation with AFE (Authorization for Expenditure) tracking, actual cost capture, and partner share calculations. Enables operator and non-operator joint venture participation with transparent cost allocation and regulatory compliance. Integrates with Accounts Payable for invoices, General Ledger for postings, and Fixed Assets for asset capitalization.

**Bounded Context:** Oil & Gas Operations & Joint Venture Accounting
**Service Name:** `oil-gas-service`
**Database:** `data/oil_gas.db`
**HTTP Port:** 8212 | **gRPC Port:** 9212

---

## 2. Database Schema

### 2.1 Wells
```sql
CREATE TABLE og_wells (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    well_id TEXT NOT NULL,
    well_name TEXT NOT NULL,
    field_name TEXT NOT NULL,
    basin TEXT NOT NULL,
    well_type TEXT NOT NULL
        CHECK(well_type IN ('EXPLORATION','PRODUCTION','INJECTION','DISPOSAL','OBSERVATION')),
    well_status TEXT NOT NULL DEFAULT 'PERMITTED'
        CHECK(well_status IN ('PERMITTED','DRILLING','COMPLETING','ACTIVE','SHUT_IN','PLUGGED','ABANDONED')),
    spud_date TEXT,
    completion_date TEXT,
    operator_id TEXT NOT NULL,
    working_interest_pct REAL NOT NULL DEFAULT 100.0,
    surface_location_lat REAL,
    surface_location_long REAL,
    bottomhole_location_lat REAL,
    bottomhole_location_long REAL,
    total_depth_feet REAL,
    production_zone TEXT,
    api_well_number TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, well_id)
);

CREATE INDEX idx_og_wells_tenant_field ON og_wells(tenant_id, field_name);
CREATE INDEX idx_og_wells_tenant_basin ON og_wells(tenant_id, basin);
CREATE INDEX idx_og_wells_tenant_type ON og_wells(tenant_id, well_type);
CREATE INDEX idx_og_wells_tenant_status ON og_wells(tenant_id, well_status);
CREATE INDEX idx_og_wells_tenant_operator ON og_wells(tenant_id, operator_id);
CREATE INDEX idx_og_wells_tenant_dates ON og_wells(tenant_id, spud_date);
CREATE INDEX idx_og_wells_tenant_active ON og_wells(tenant_id, is_active);
```

### 2.2 Joint Ventures
```sql
CREATE TABLE og_joint_ventures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jv_code TEXT NOT NULL,
    jv_name TEXT NOT NULL,
    jv_type TEXT NOT NULL
        CHECK(jv_type IN ('EXPLORATION','DEVELOPMENT','PRODUCTION','PROCESSING','PIPELINE')),
    partners TEXT NOT NULL,                           -- JSON: array of partner IDs and ownership percentages
    operator_partner_id TEXT NOT NULL,
    operator_designation_date TEXT NOT NULL,
    cost_sharing_rules TEXT NOT NULL,                 -- JSON: cost category sharing percentages
    revenue_sharing_rules TEXT NOT NULL,              -- JSON: revenue sharing percentages
    agreement_date TEXT NOT NULL,
    agreement_expiry_date TEXT,
    accounting_method TEXT NOT NULL DEFAULT 'SUCCESSFUL_EFFORTS'
        CHECK(accounting_method IN ('SUCCESSFUL_EFFORTS','FULL_COST')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUSPENDED','TERMINATED','WOUND_UP')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, jv_code)
);

CREATE INDEX idx_og_jv_tenant_type ON og_joint_ventures(tenant_id, jv_type);
CREATE INDEX idx_og_jv_tenant_status ON og_joint_ventures(tenant_id, status);
CREATE INDEX idx_og_jv_tenant_operator ON og_joint_ventures(tenant_id, operator_partner_id);
CREATE INDEX idx_og_jv_tenant_dates ON og_joint_ventures(tenant_id, agreement_date, agreement_expiry_date);
CREATE INDEX idx_og_jv_tenant_active ON og_joint_ventures(tenant_id, is_active);
```

### 2.3 Production Records
```sql
CREATE TABLE og_production_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    well_id TEXT NOT NULL,
    field_name TEXT NOT NULL,
    production_date TEXT NOT NULL,
    oil_volume_bbl REAL NOT NULL DEFAULT 0.0,
    gas_volume_mcf REAL NOT NULL DEFAULT 0.0,
    water_volume_bbl REAL NOT NULL DEFAULT 0.0,
    condensate_volume_bbl REAL NOT NULL DEFAULT 0.0,
    lifting_cost_per_bbl_cents INTEGER NOT NULL DEFAULT 0,
    royalty_amount_cents INTEGER NOT NULL DEFAULT 0,
    severance_tax_cents INTEGER NOT NULL DEFAULT 0,
    gross_revenue_cents INTEGER NOT NULL DEFAULT 0,
    net_revenue_cents INTEGER NOT NULL DEFAULT 0,
    downtime_hours REAL NOT NULL DEFAULT 0.0,
    choke_size_inches REAL,
    tubing_pressure_psi REAL,
    casing_pressure_psi REAL,
    measurement_method TEXT NOT NULL DEFAULT 'METER'
        CHECK(measurement_method IN ('METER','TANK','ESTIMATED','ALLOCATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (well_id) REFERENCES og_wells(id),
    UNIQUE(tenant_id, well_id, production_date)
);

CREATE INDEX idx_og_prod_tenant_well ON og_production_records(tenant_id, well_id);
CREATE INDEX idx_og_prod_tenant_field ON og_production_records(tenant_id, field_name);
CREATE INDEX idx_og_prod_tenant_date ON og_production_records(tenant_id, production_date);
CREATE INDEX idx_og_prod_tenant_method ON og_production_records(tenant_id, measurement_method);
```

### 2.4 Lease Agreements
```sql
CREATE TABLE og_lease_agreements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lease_number TEXT NOT NULL,
    lease_name TEXT NOT NULL,
    lessor_name TEXT NOT NULL,
    lessor_type TEXT NOT NULL
        CHECK(lessor_type IN ('PRIVATE','STATE','FEDERAL','TRIBAL','FOREIGN_GOVERNMENT')),
    mineral_rights TEXT NOT NULL,                     -- JSON: mineral types covered
    royalty_rate_pct REAL NOT NULL DEFAULT 0.0,
    bonus_payment_cents INTEGER NOT NULL DEFAULT 0,
    annual_rental_cents INTEGER NOT NULL DEFAULT 0,
    primary_term_start TEXT NOT NULL,
    primary_term_end TEXT NOT NULL,
    renewal_option_years INTEGER,
    surface_area_acres REAL NOT NULL DEFAULT 0.0,
    depth_rights TEXT,                                -- JSON: depth restrictions
    assignment_restrictions TEXT,                     -- JSON: transfer limitations
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PENDING','EXPIRED','SURRENDERED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, lease_number)
);

CREATE INDEX idx_og_lease_tenant_lessor ON og_lease_agreements(tenant_id, lessor_type);
CREATE INDEX idx_og_lease_tenant_status ON og_lease_agreements(tenant_id, status);
CREATE INDEX idx_og_lease_tenant_dates ON og_lease_agreements(tenant_id, primary_term_start, primary_term_end);
CREATE INDEX idx_og_lease_tenant_active ON og_lease_agreements(tenant_id, is_active);
```

### 2.5 Cost Allocations
```sql
CREATE TABLE og_cost_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    afe_number TEXT NOT NULL,
    jv_id TEXT NOT NULL,
    well_id TEXT,
    cost_category TEXT NOT NULL
        CHECK(cost_category IN ('DRILLING','COMPLETION','FACILITIES','EQUIPMENT','OPERATIONS','MAINTENANCE','ABANDONMENT','OVERHEAD')),
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    partner_shares TEXT NOT NULL,                     -- JSON: partner ID -> share amount in cents
    allocation_date TEXT NOT NULL,
    authorization_date TEXT NOT NULL,
    authorized_by TEXT NOT NULL,
    cost_center TEXT,
    gl_account TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','ALLOCATED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jv_id) REFERENCES og_joint_ventures(id),
    UNIQUE(tenant_id, afe_number)
);

CREATE INDEX idx_og_cost_tenant_jv ON og_cost_allocations(tenant_id, jv_id);
CREATE INDEX idx_og_cost_tenant_well ON og_cost_allocations(tenant_id, well_id);
CREATE INDEX idx_og_cost_tenant_category ON og_cost_allocations(tenant_id, cost_category);
CREATE INDEX idx_og_cost_tenant_status ON og_cost_allocations(tenant_id, status);
CREATE INDEX idx_og_cost_tenant_dates ON og_cost_allocations(tenant_id, allocation_date);
CREATE INDEX idx_og_cost_tenant_active ON og_cost_allocations(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Wells
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/oil-gas/wells` | List wells |
| POST | `/api/v1/oil-gas/wells` | Register well |
| GET | `/api/v1/oil-gas/wells/{id}` | Get well details |
| PUT | `/api/v1/oil-gas/wells/{id}` | Update well |
| PATCH | `/api/v1/oil-gas/wells/{id}/status` | Update well status |
| GET | `/api/v1/oil-gas/wells/{id}/history` | Get well status history |

### 3.2 Joint Ventures
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/oil-gas/joint-ventures` | List joint ventures |
| POST | `/api/v1/oil-gas/joint-ventures` | Create joint venture |
| GET | `/api/v1/oil-gas/joint-ventures/{id}` | Get JV details |
| PUT | `/api/v1/oil-gas/joint-ventures/{id}` | Update joint venture |
| GET | `/api/v1/oil-gas/joint-ventures/{id}/partners` | Get JV partner breakdown |

### 3.3 Production
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/oil-gas/production` | Record production data |
| GET | `/api/v1/oil-gas/production` | List production records |
| GET | `/api/v1/oil-gas/production/summary` | Get production summary by well/field |
| GET | `/api/v1/oil-gas/production/daily` | Get daily production report |
| GET | `/api/v1/oil-gas/production/monthly` | Get monthly production report |

### 3.4 Leases
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/oil-gas/leases` | List lease agreements |
| POST | `/api/v1/oil-gas/leases` | Create lease agreement |
| GET | `/api/v1/oil-gas/leases/{id}` | Get lease details |
| PUT | `/api/v1/oil-gas/leases/{id}` | Update lease |
| POST | `/api/v1/oil-gas/leases/{id}/renew` | Renew lease agreement |
| GET | `/api/v1/oil-gas/leases/expiring` | Get leases nearing expiration |

### 3.5 Cost Allocations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/oil-gas/cost-allocations` | List cost allocations |
| POST | `/api/v1/oil-gas/cost-allocations` | Create AFE / cost allocation |
| GET | `/api/v1/oil-gas/cost-allocations/{id}` | Get allocation details |
| PUT | `/api/v1/oil-gas/cost-allocations/{id}` | Update allocation |
| POST | `/api/v1/oil-gas/cost-allocations/{id}/approve` | Approve AFE |
| POST | `/api/v1/oil-gas/cost-allocations/{id}/allocate` | Execute cost allocation |

---

## 4. Business Rules

### 4.1 Well Management
1. Each well MUST have a unique well_id within a tenant.
2. Well status transitions MUST follow the lifecycle: PERMITTED -> DRILLING -> COMPLETING -> ACTIVE -> SHUT_IN -> PLUGGED -> ABANDONED.
3. Only wells in ACTIVE status SHOULD accept production records.
4. working_interest_pct MUST be between 0.0 and 100.0.
5. spud_date MUST be recorded when a well transitions to DRILLING status.

### 4.2 Joint Venture Management
6. Each joint venture MUST have a unique jv_code within a tenant.
7. Partner ownership percentages in the partners JSON MUST sum to 100.0.
8. The operator_partner_id MUST reference one of the partners listed in the partners JSON.
9. Cost sharing rules MUST define percentages for all cost categories and MUST sum to 100.0 per category.
10. A JV in TERMINATED or WOUND_UP status MUST NOT accept new cost allocations.

### 4.3 Production Recording
11. The combination of (tenant_id, well_id, production_date) MUST be unique; duplicate daily production entries for the same well are NOT allowed.
12. Production volumes (oil, gas, water, condensate) MUST be non-negative values.
13. gross_revenue_cents MUST be calculated from commodity volumes multiplied by prevailing prices.
14. net_revenue_cents MUST equal gross_revenue_cents minus royalty_amount_cents minus severance_tax_cents.
15. Wells with downtime_hours equal to 24.0 SHOULD be flagged for review as potential shut-in.

### 4.4 Lease Management
16. Lease numbers MUST be unique within a tenant.
17. royalty_rate_pct MUST be between 0.0 and 100.0.
18. primary_term_end MUST be later than primary_term_start.
19. Lease status transitions MUST follow: PENDING -> ACTIVE -> EXPIRED/SURRENDERED/TERMINATED.
20. Leases approaching primary_term_end within 90 days SHOULD trigger expiration warning notifications.

### 4.5 Cost Allocations
21. AFE numbers MUST be unique within a tenant.
22. estimated_cost_cents MUST be provided when creating an AFE; actual_cost_cents MUST be updated as costs are incurred.
23. variance_cents MUST be calculated as actual_cost_cents minus estimated_cost_cents.
24. partner_shares JSON amounts MUST sum to actual_cost_cents.
25. Allocation status transitions MUST follow: DRAFT -> SUBMITTED -> APPROVED -> ALLOCATED -> CLOSED.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.oilgas.v1;

service OilGasService {
    // Well management
    rpc CreateWell(CreateWellRequest) returns (CreateWellResponse);
    rpc GetWell(GetWellRequest) returns (GetWellResponse);
    rpc ListWells(ListWellsRequest) returns (ListWellsResponse);
    rpc UpdateWellStatus(UpdateWellStatusRequest) returns (UpdateWellStatusResponse);

    // Joint ventures
    rpc CreateJointVenture(CreateJointVentureRequest) returns (CreateJointVentureResponse);
    rpc GetJointVenture(GetJointVentureRequest) returns (GetJointVentureResponse);
    rpc ListJointVentures(ListJointVenturesRequest) returns (ListJointVenturesResponse);

    // Production
    rpc RecordProduction(RecordProductionRequest) returns (RecordProductionResponse);
    rpc GetProductionSummary(GetProductionSummaryRequest) returns (GetProductionSummaryResponse);
    rpc GetDailyProduction(GetDailyProductionRequest) returns (GetDailyProductionResponse);

    // Leases
    rpc CreateLease(CreateLeaseRequest) returns (CreateLeaseResponse);
    rpc GetLease(GetLeaseRequest) returns (GetLeaseResponse);
    rpc ListLeases(ListLeasesRequest) returns (ListLeasesResponse);
    rpc RenewLease(RenewLeaseRequest) returns (RenewLeaseResponse);

    // Cost allocations
    rpc CreateCostAllocation(CreateCostAllocationRequest) returns (CreateCostAllocationResponse);
    rpc GetCostAllocation(GetCostAllocationRequest) returns (GetCostAllocationResponse);
    rpc ApproveAFE(ApprovideAFERequest) returns (ApproveAFEResponse);
    rpc AllocateCosts(AllocateCostsRequest) returns (AllocateCostsResponse);
}

message CreateWellRequest {
    string tenant_id = 1;
    string well_id = 2;
    string well_name = 3;
    string field_name = 4;
    string basin = 5;
    string well_type = 6;
    string operator_id = 7;
    double working_interest_pct = 8;
    double surface_location_lat = 9;
    double surface_location_long = 10;
    double total_depth_feet = 11;
    string production_zone = 12;
    string api_well_number = 13;
}

message CreateWellResponse {
    string id = 1;
    string well_id = 2;
    string well_status = 3;
}

message GetWellRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetWellResponse {
    string id = 1;
    string well_id = 2;
    string well_name = 3;
    string field_name = 4;
    string basin = 5;
    string well_type = 6;
    string well_status = 7;
    string operator_id = 8;
    double working_interest_pct = 9;
    string spud_date = 10;
    string completion_date = 11;
}

message ListWellsRequest {
    string tenant_id = 1;
    string field_name = 2;
    string basin = 3;
    string well_type = 4;
    string well_status = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListWellsResponse {
    repeated GetWellResponse wells = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateWellStatusRequest {
    string tenant_id = 1;
    string id = 2;
    string new_status = 3;
    string effective_date = 4;
}

message UpdateWellStatusResponse {
    string id = 1;
    string well_status = 2;
    string effective_date = 3;
}

message CreateJointVentureRequest {
    string tenant_id = 1;
    string jv_code = 2;
    string jv_name = 3;
    string jv_type = 4;
    string partners = 5;
    string operator_partner_id = 6;
    string cost_sharing_rules = 7;
    string revenue_sharing_rules = 8;
    string agreement_date = 9;
    string agreement_expiry_date = 10;
    string accounting_method = 11;
}

message CreateJointVentureResponse {
    string jv_id = 1;
    string jv_code = 2;
    string status = 3;
}

message GetJointVentureRequest {
    string tenant_id = 1;
    string jv_id = 2;
}

message GetJointVentureResponse {
    string jv_id = 1;
    string jv_code = 2;
    string jv_name = 3;
    string jv_type = 4;
    string operator_partner_id = 5;
    string partners = 6;
    string status = 7;
}

message ListJointVenturesRequest {
    string tenant_id = 1;
    string jv_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListJointVenturesResponse {
    repeated GetJointVentureResponse joint_ventures = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RecordProductionRequest {
    string tenant_id = 1;
    string well_id = 2;
    string field_name = 3;
    string production_date = 4;
    double oil_volume_bbl = 5;
    double gas_volume_mcf = 6;
    double water_volume_bbl = 7;
    double condensate_volume_bbl = 8;
    int64 lifting_cost_per_bbl_cents = 9;
    int64 royalty_amount_cents = 10;
    int64 severance_tax_cents = 11;
    int64 gross_revenue_cents = 12;
    int64 net_revenue_cents = 13;
    double downtime_hours = 14;
    string measurement_method = 15;
}

message RecordProductionResponse {
    string record_id = 1;
    string well_id = 2;
    string production_date = 3;
}

message GetProductionSummaryRequest {
    string tenant_id = 1;
    string well_id = 2;
    string field_name = 3;
    string from_date = 4;
    string to_date = 5;
}

message GetProductionSummaryResponse {
    double total_oil_bbl = 1;
    double total_gas_mcf = 2;
    double total_water_bbl = 3;
    int64 total_gross_revenue_cents = 4;
    int64 total_net_revenue_cents = 5;
    int64 total_royalty_cents = 6;
    int64 total_severance_tax_cents = 7;
    int32 record_count = 8;
}

message GetDailyProductionRequest {
    string tenant_id = 1;
    string field_name = 2;
    string date = 3;
}

message GetDailyProductionResponse {
    repeated DailyProductionEntry entries = 1;
    int32 total_entries = 2;
}

message DailyProductionEntry {
    string well_id = 2;
    string well_name = 3;
    double oil_volume_bbl = 4;
    double gas_volume_mcf = 5;
    int64 gross_revenue_cents = 6;
}

message CreateLeaseRequest {
    string tenant_id = 1;
    string lease_number = 2;
    string lease_name = 3;
    string lessor_name = 4;
    string lessor_type = 5;
    string mineral_rights = 6;
    double royalty_rate_pct = 7;
    int64 bonus_payment_cents = 8;
    int64 annual_rental_cents = 9;
    string primary_term_start = 10;
    string primary_term_end = 11;
    int32 renewal_option_years = 12;
    double surface_area_acres = 13;
    string depth_rights = 14;
    string assignment_restrictions = 15;
}

message CreateLeaseResponse {
    string lease_id = 1;
    string lease_number = 2;
    string status = 3;
}

message GetLeaseRequest {
    string tenant_id = 1;
    string lease_id = 2;
}

message GetLeaseResponse {
    string lease_id = 1;
    string lease_number = 2;
    string lease_name = 3;
    string lessor_name = 4;
    string lessor_type = 5;
    double royalty_rate_pct = 6;
    string primary_term_start = 7;
    string primary_term_end = 8;
    string status = 9;
}

message ListLeasesRequest {
    string tenant_id = 1;
    string lessor_type = 2;
    string status = 3;
    bool expiring_soon = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListLeasesResponse {
    repeated GetLeaseResponse leases = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RenewLeaseRequest {
    string tenant_id = 1;
    string lease_id = 2;
    string new_term_end = 3;
    int64 new_annual_rental_cents = 4;
}

message RenewLeaseResponse {
    string lease_id = 1;
    string primary_term_end = 2;
    string status = 3;
}

message CreateCostAllocationRequest {
    string tenant_id = 1;
    string afe_number = 2;
    string jv_id = 3;
    string well_id = 4;
    string cost_category = 5;
    int64 estimated_cost_cents = 6;
    string partner_shares = 7;
    string allocation_date = 8;
    string cost_center = 9;
    string gl_account = 10;
}

message CreateCostAllocationResponse {
    string allocation_id = 1;
    string afe_number = 2;
    string status = 3;
}

message GetCostAllocationRequest {
    string tenant_id = 1;
    string allocation_id = 2;
}

message GetCostAllocationResponse {
    string allocation_id = 1;
    string afe_number = 2;
    string jv_id = 3;
    string well_id = 4;
    string cost_category = 5;
    int64 estimated_cost_cents = 6;
    int64 actual_cost_cents = 7;
    int64 variance_cents = 8;
    string partner_shares = 9;
    string status = 10;
}

message ApprovideAFERequest {
    string tenant_id = 1;
    string allocation_id = 2;
    string authorized_by = 3;
}

message ApproveAFEResponse {
    string allocation_id = 1;
    string afe_number = 2;
    string status = 3;
    string authorization_date = 4;
}

message AllocateCostsRequest {
    string tenant_id = 1;
    string allocation_id = 2;
    int64 actual_cost_cents = 3;
    string partner_shares = 4;
}

message AllocateCostsResponse {
    string allocation_id = 1;
    int64 actual_cost_cents = 2;
    int64 variance_cents = 3;
    string status = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `ap-service` | Vendor invoices for drilling, completion, and operational costs |
| `gl-service` | GL account structure for cost center and account mapping |
| `asset-service` | Fixed asset records for well capitalization and depreciation |
| `auth-service` | User identity for AFE authorization and operator role resolution |
| `workflow-service` | AFE approval workflows and lease renewal authorization |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | JV cost allocation journal entries, production revenue postings, royalty accruals |
| `ap-service` | Partner cost share payment requests, royalty payment obligations |
| `asset-service` | Well capitalization data, asset retirement obligation calculations |
| `reporting-service` | Production dashboards, JV cost reports, lease expiration analytics |
| `notification-service` | Lease expiration warnings, AFE variance alerts, well status change notifications |
| `audit-service` | Cost allocation audit trail, production measurement compliance logs |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `oilgas.production.reported` | `oilgas.events` | `{ record_id, well_id, field_name, production_date, oil_bbl, gas_mcf, gross_revenue_cents }` | Daily production data recorded |
| `oilgas.jv.cost.allocated` | `oilgas.events` | `{ allocation_id, afe_number, jv_id, cost_category, actual_cost_cents, partner_count }` | JV cost allocation executed |
| `oilgas.lease.expiring` | `oilgas.events` | `{ lease_id, lease_number, primary_term_end, days_remaining }` | Lease approaching expiration |
| `oilgas.well.completed` | `oilgas.events` | `{ well_id, well_name, field_name, completion_date, total_depth_feet }` | Well drilling and completion finished |

---

## 8. Migrations

1. V001: `og_wells`
2. V002: `og_joint_ventures`
3. V003: `og_production_records`
4. V004: `og_lease_agreements`
5. V005: `og_cost_allocations`
6. V006: Triggers for `updated_at`
