# 137 - Public Sector Specification

## 1. Domain Overview

Public Sector extends the Fusion ERP platform for government agencies and public institutions, providing constituent services, program management, fund accounting, regulatory compliance, and transparency reporting. It supports federal, state, and local government operations with features specific to public finance, citizen engagement, and government-specific reporting.

**Bounded Context:** Government & Public Sector Operations
**Service Name:** `publicsector-service`
**Database:** `data/publicsector.db`
**HTTP Port:** 8217 | **gRPC Port:** 9217

---

## 2. Database Schema

### 2.1 Government Entities
```sql
CREATE TABLE ps_govt_entities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    entity_code TEXT NOT NULL,
    entity_type TEXT NOT NULL CHECK(entity_type IN ('FEDERAL','STATE','COUNTY','MUNICIPAL','SPECIAL_DISTRICT','AUTHORITY','AGENCY','DEPARTMENT')),
    parent_entity_id TEXT,
    jurisdiction TEXT,
    fiscal_year_start_month INTEGER NOT NULL DEFAULT 10,
    fund_accounting_enabled INTEGER NOT NULL DEFAULT 1,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, entity_code)
);
```

### 2.2 Funds
```sql
CREATE TABLE ps_funds (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    fund_code TEXT NOT NULL,
    fund_name TEXT NOT NULL,
    fund_type TEXT NOT NULL CHECK(fund_type IN ('GENERAL','SPECIAL_REVENUE','CAPITAL_PROJECT','DEBT_SERVICE','ENTERPRISE','INTERNAL_SERVICE','TRUST','AGENCY','PERMANENT')),
    fund_category TEXT NOT NULL CHECK(fund_category IN ('GOVERNMENTAL','PROPRIETARY','FIDUCIARY')),
    fund_group TEXT,
    beginning_balance_cents INTEGER NOT NULL DEFAULT 0,
    current_budget_cents INTEGER NOT NULL DEFAULT 0,
    year_to_date_actual_cents INTEGER NOT NULL DEFAULT 0,
    encumbrance_cents INTEGER NOT NULL DEFAULT 0,
    available_balance_cents INTEGER NOT NULL DEFAULT 0,
    fiscal_year TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','CLOSED','ELIMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, fund_code, fiscal_year)
);

CREATE INDEX idx_ps_funds_tenant_year ON ps_funds(tenant_id, fiscal_year, fund_type);
```

### 2.3 Programs
```sql
CREATE TABLE ps_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_name TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_type TEXT NOT NULL CHECK(program_type IN ('SOCIAL_SERVICES','INFRASTRUCTURE','PUBLIC_SAFETY','EDUCATION','HEALTH','ENVIRONMENT','TRANSPORTATION','HOUSING','ECONOMIC_DEVELOPMENT','OTHER')),
    department_id TEXT,
    fund_id TEXT,
    description TEXT,
    objectives TEXT,
    target_population TEXT,
    performance_metrics TEXT,            -- JSON: [{ "name": "...", "target": 100, "unit": "..." }]
    start_date TEXT NOT NULL,
    end_date TEXT,
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    total_spent_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ACTIVE','SUSPENDED','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (fund_id) REFERENCES ps_funds(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, program_code)
);
```

### 2.4 Constituent Records
```sql
CREATE TABLE ps_constituents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    constituent_type TEXT NOT NULL CHECK(constituent_type IN ('CITIZEN','BUSINESS','CONTRACTOR','NONPROFIT','OTHER_AGENCY')),
    first_name TEXT,
    last_name TEXT,
    business_name TEXT,
    email TEXT,
    phone TEXT,
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT NOT NULL DEFAULT 'US',
    district TEXT,                       -- Electoral/jurisdictional district
    external_id TEXT,                    -- SSN last 4, EIN, etc. (encrypted)
    registration_date TEXT,
    preferred_language TEXT DEFAULT 'en',
    communication_preferences TEXT,      -- JSON: { "email": true, "sms": false, "mail": true }
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','DECEASED','MERGED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, id)
);

CREATE INDEX idx_ps_constituents_tenant_type ON ps_constituents(tenant_id, constituent_type);
CREATE INDEX idx_ps_constituents_tenant_district ON ps_constituents(tenant_id, district);
```

### 2.5 Service Requests (Constituent Cases)
```sql
CREATE TABLE ps_service_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_number TEXT NOT NULL,
    constituent_id TEXT NOT NULL,
    service_type TEXT NOT NULL CHECK(service_type IN ('PERMIT','LICENSE','COMPLAINT','INQUIRY','INSPECTION','MAINTENANCE','BENEFITS_APPLICATION','RECORDS_REQUEST','OTHER')),
    subject TEXT NOT NULL,
    description TEXT NOT NULL,
    location TEXT,
    geo_coordinates TEXT,
    assigned_department TEXT,
    assigned_to TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','EMERGENCY')),
    status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(status IN ('SUBMITTED','IN_REVIEW','IN_PROGRESS','AWAITING_INFO','APPROVED','DENIED','COMPLETED','CANCELLED')),
    submission_date TEXT NOT NULL DEFAULT (datetime('now')),
    target_completion_date TEXT,
    actual_completion_date TEXT,
    resolution_notes TEXT,
    satisfaction_rating INTEGER CHECK(satisfaction_rating BETWEEN 1 AND 5),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (constituent_id) REFERENCES ps_constituents(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, request_number)
);

CREATE INDEX idx_ps_requests_tenant_status ON ps_service_requests(tenant_id, status);
CREATE INDEX idx_ps_requests_tenant_type ON ps_service_requests(tenant_id, service_type);
CREATE INDEX idx_ps_requests_tenant_constituent ON ps_service_requests(tenant_id, constituent_id);
```

### 2.6 Appropriations
```sql
CREATE TABLE ps_appropriations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appropriation_code TEXT NOT NULL,
    description TEXT NOT NULL,
    fund_id TEXT NOT NULL,
    program_id TEXT,
    department_id TEXT,
    fiscal_year TEXT NOT NULL,
    original_amount_cents INTEGER NOT NULL,
    current_amount_cents INTEGER NOT NULL,
    spent_cents INTEGER NOT NULL DEFAULT 0,
    encumbered_cents INTEGER NOT NULL DEFAULT 0,
    available_cents INTEGER NOT NULL DEFAULT 0,
    authority_type TEXT NOT NULL CHECK(authority_type IN ('ANNUAL','CONTINUING','SUPPLEMENTAL','EMERGENCY')),
    legislation_reference TEXT,
    effective_date TEXT NOT NULL,
    expiration_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','SUPERSEDED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (fund_id) REFERENCES ps_funds(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, appropriation_code, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Government Structure
```
GET    /api/v1/publicsector/entities                    Permission: pubsec.entities.read
POST   /api/v1/publicsector/entities                    Permission: pubsec.entities.create
GET    /api/v1/publicsector/funds                        Permission: pubsec.funds.read
POST   /api/v1/publicsector/funds                        Permission: pubsec.funds.create
PUT    /api/v1/publicsector/funds/{id}                   Permission: pubsec.funds.update
GET    /api/v1/publicsector/programs                     Permission: pubsec.programs.read
POST   /api/v1/publicsector/programs                     Permission: pubsec.programs.create
```

### 3.2 Appropriations
```
GET    /api/v1/publicsector/appropriations               Permission: pubsec.appropriations.read
POST   /api/v1/publicsector/appropriations               Permission: pubsec.appropriations.create
PUT    /api/v1/publicsector/appropriations/{id}          Permission: pubsec.appropriations.update
GET    /api/v1/publicsector/appropriations/{id}/balance  Permission: pubsec.appropriations.read
```

### 3.3 Constituent Services
```
GET    /api/v1/publicsector/constituents                 Permission: pubsec.constituents.read
POST   /api/v1/publicsector/constituents                 Permission: pubsec.constituents.create
GET    /api/v1/publicsector/constituents/{id}            Permission: pubsec.constituents.read
PUT    /api/v1/publicsector/constituents/{id}            Permission: pubsec.constituents.update
GET    /api/v1/publicsector/constituents/{id}/cases      Permission: pubsec.constituents.read
```

### 3.4 Service Requests
```
GET    /api/v1/publicsector/requests                     Permission: pubsec.requests.read
GET    /api/v1/publicsector/requests/{id}                Permission: pubsec.requests.read
POST   /api/v1/publicsector/requests                     Permission: pubsec.requests.create
PUT    /api/v1/publicsector/requests/{id}                Permission: pubsec.requests.update
POST   /api/v1/publicsector/requests/{id}/assign         Permission: pubsec.requests.assign
POST   /api/v1/publicsector/requests/{id}/complete       Permission: pubsec.requests.complete
```

### 3.5 Transparency Reporting
```
GET    /api/v1/publicsector/transparency/budget
  ?fiscal_year={year}                                    Permission: pubsec.transparency.read
GET    /api/v1/publicsector/transparency/spending
  ?department={id}&fiscal_year={year}                    Permission: pubsec.transparency.read
GET    /api/v1/publicsector/transparency/contracts        Permission: pubsec.transparency.read
GET    /api/v1/publicsector/transparency/performance
  ?program_id={id}                                       Permission: pubsec.transparency.read
GET    /api/v1/publicsector/transparency/dashboard        Permission: pubsec.transparency.read
```

---

## 4. Business Rules

### 4.1 Fund Accounting
- Governmental funds use modified accrual accounting
- Proprietary funds use full accrual accounting
- Every transaction MUST be charged to a fund
- Interfund transactions tracked separately
- Fund balance = Beginning Balance + Revenues - Expenditures
- Encumbrances reduce available balance but don't affect fund balance until expended

### 4.2 Appropriation Controls
- Expenditures MUST NOT exceed appropriation available balance
- Multi-year appropriations carry forward unless expired
- Emergency appropriations require special authorization
- Supplemental appropriations require legislative approval
- Appropriation transfers between departments require approval workflow

### 4.3 Constituent Services
- Service requests tracked with SLA targets per service type
- Emergency requests (e.g., pothole, water main break) auto-escalated
- 311-style unified service request portal for constituents
- Service level reporting by department, type, and district
- Constituent notifications at each status change

### 4.4 Transparency
- Open data requirements: budget, spending, contracts publicly accessible
- Checkbook-level expenditure detail available
- Vendor/contractor payments published
- Program performance metrics with targets vs actuals
- Compliance with open records laws (FOIA/public records)

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.publicsector.v1;

service PublicSectorService {
    rpc CheckAppropriationBalance(CheckAppropriationBalanceRequest) returns (CheckAppropriationBalanceResponse);
    rpc CreateConstituentCase(CreateConstituentCaseRequest) returns (CreateConstituentCaseResponse);
    rpc GetFundBalance(GetFundBalanceRequest) returns (GetFundBalanceResponse);
    rpc GetProgramPerformance(GetProgramPerformanceRequest) returns (GetProgramPerformanceResponse);
    rpc GetTransparencyData(GetTransparencyDataRequest) returns (GetTransparencyDataResponse);
}

// Entity messages
message GovernmentEntity {
    string id = 1;
    string tenant_id = 2;
    string entity_name = 3;
    string entity_code = 4;
    string entity_type = 5;
    string parent_entity_id = 6;
    string jurisdiction = 7;
    int32 fiscal_year_start_month = 8;
    int32 fund_accounting_enabled = 9;
    string description = 10;
    string created_at = 11;
    string updated_at = 12;
}

message Fund {
    string id = 1;
    string tenant_id = 2;
    string fund_code = 3;
    string fund_name = 4;
    string fund_type = 5;
    string fund_category = 6;
    string fund_group = 7;
    int64 beginning_balance_cents = 8;
    int64 current_budget_cents = 9;
    int64 year_to_date_actual_cents = 10;
    int64 encumbrance_cents = 11;
    int64 available_balance_cents = 12;
    string fiscal_year = 13;
    string status = 14;
    string created_at = 15;
    string updated_at = 16;
}

message Program {
    string id = 1;
    string tenant_id = 2;
    string program_name = 3;
    string program_code = 4;
    string program_type = 5;
    string department_id = 6;
    string fund_id = 7;
    string description = 8;
    string objectives = 9;
    string target_population = 10;
    string performance_metrics = 11;
    string start_date = 12;
    string end_date = 13;
    int64 total_budget_cents = 14;
    int64 total_spent_cents = 15;
    string status = 16;
    string created_at = 17;
    string updated_at = 18;
}

message Constituent {
    string id = 1;
    string tenant_id = 2;
    string constituent_type = 3;
    string first_name = 4;
    string last_name = 5;
    string business_name = 6;
    string email = 7;
    string phone = 8;
    string address_line1 = 9;
    string address_line2 = 10;
    string city = 11;
    string state_province = 12;
    string postal_code = 13;
    string country = 14;
    string district = 15;
    string external_id = 16;
    string registration_date = 17;
    string preferred_language = 18;
    string communication_preferences = 19;
    string status = 20;
    string created_at = 21;
    string updated_at = 22;
}

message ServiceRequest {
    string id = 1;
    string tenant_id = 2;
    string request_number = 3;
    string constituent_id = 4;
    string service_type = 5;
    string subject = 6;
    string description = 7;
    string location = 8;
    string geo_coordinates = 9;
    string assigned_department = 10;
    string assigned_to = 11;
    string priority = 12;
    string status = 13;
    string submission_date = 14;
    string target_completion_date = 15;
    string actual_completion_date = 16;
    string resolution_notes = 17;
    int32 satisfaction_rating = 18;
    string created_at = 19;
    string updated_at = 20;
}

message Appropriation {
    string id = 1;
    string tenant_id = 2;
    string appropriation_code = 3;
    string description = 4;
    string fund_id = 5;
    string program_id = 6;
    string department_id = 7;
    string fiscal_year = 8;
    int64 original_amount_cents = 9;
    int64 current_amount_cents = 10;
    int64 spent_cents = 11;
    int64 encumbered_cents = 12;
    int64 available_cents = 13;
    string authority_type = 14;
    string legislation_reference = 15;
    string effective_date = 16;
    string expiration_date = 17;
    string status = 18;
    string created_at = 19;
    string updated_at = 20;
}

// Request/Response messages
message CheckAppropriationBalanceRequest {
    string tenant_id = 1;
    string appropriation_id = 2;
    int64 proposed_amount_cents = 3;
}

message CheckAppropriationBalanceResponse {
    string appropriation_id = 1;
    int64 current_amount_cents = 2;
    int64 spent_cents = 3;
    int64 encumbered_cents = 4;
    int64 available_cents = 5;
    bool sufficient_funds = 6;
}

message CreateConstituentCaseRequest {
    string tenant_id = 1;
    string constituent_id = 2;
    string service_type = 3;
    string subject = 4;
    string description = 5;
    string location = 6;
    string geo_coordinates = 7;
    string priority = 8;
    string target_completion_date = 9;
}

message CreateConstituentCaseResponse {
    ServiceRequest data = 1;
}

message GetFundBalanceRequest {
    string tenant_id = 1;
    string fund_id = 2;
    string fiscal_year = 3;
}

message GetFundBalanceResponse {
    string fund_id = 1;
    string fund_code = 2;
    string fund_name = 3;
    string fund_type = 4;
    int64 beginning_balance_cents = 5;
    int64 current_budget_cents = 6;
    int64 year_to_date_actual_cents = 7;
    int64 encumbrance_cents = 8;
    int64 available_balance_cents = 9;
    string fiscal_year = 10;
}

message GetProgramPerformanceRequest {
    string tenant_id = 1;
    string program_id = 2;
    string fiscal_year = 3;
}

message PerformanceMetric {
    string name = 1;
    double target = 2;
    double actual = 3;
    string unit = 4;
    double pct_achieved = 5;
}

message GetProgramPerformanceResponse {
    string program_id = 1;
    string program_name = 2;
    int64 total_budget_cents = 3;
    int64 total_spent_cents = 4;
    double budget_utilization_pct = 5;
    repeated PerformanceMetric metrics = 6;
}

message GetTransparencyDataRequest {
    string tenant_id = 1;
    string data_type = 2;
    string fiscal_year = 3;
    string department_id = 4;
    string program_id = 5;
}

message TransparencyEntry {
    string category = 1;
    string description = 2;
    int64 amount_cents = 3;
    string vendor_name = 4;
    string date = 5;
}

message GetTransparencyDataResponse {
    string fiscal_year = 1;
    int64 total_budget_cents = 2;
    int64 total_spent_cents = 3;
    int64 total_encumbered_cents = 4;
    repeated TransparencyEntry entries = 5;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **gl-service**: Fund-based journal entries, government accounting
- **ap-service**: Vendor payments with fund/appropriation charging
- **ar-service**: Revenue collection, fee processing
- **budget (gl-service)**: Budget preparation and appropriation management
- **federal-service**: Federal financials, USSGL integration
- **grant-service**: Federal/state grant management
- **workflow-service**: Approval routing, legislative workflows
- **report-service**: CAFR, SEFA, Schedule of Expenditures

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `pubsec.fund.created` | New fund established | fund_id, fund_type, fiscal_year |
| `pubsec.appropriation.created` | Appropriation enacted | appropriation_id, amount |
| `pubsec.constituent.registered` | New constituent record | constituent_id, type |
| `pubsec.request.submitted` | Service request filed | request_id, type, priority |
| `pubsec.request.completed` | Service request resolved | request_id, resolution_time |
| `pubsec.program.metrics.updated` | Performance data refreshed | program_id, metrics |

---

## 7. Migrations

### Migration Order for publicsector-service:
1. V001: `ps_govt_entities`
2. V002: `ps_funds`
3. V003: `ps_programs`
4. V004: `ps_constituents`
5. V005: `ps_service_requests`
6. V006: `ps_appropriations`
7. V007: Triggers for `updated_at`
8. V008: Seed data (sample entity, fund types, program categories)
