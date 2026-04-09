# 104 - U.S. Federal Financials Service Specification

## 1. Domain Overview

U.S. Federal Financials provides government-specific financial management supporting federal accounting standards (USSGL), fund control, appropriations management, allotments, obligations, expenditures, federal reporting (SF-133, GTAS, FACTS I/II), CFDA management, and OMB compliance. It enforces fiscal law constraints including the Anti-Deficiency Act and supports the complete federal budget execution lifecycle.

**Bounded Context:** Federal Financial Management & Compliance
**Service Name:** `federal-service`
**Database:** `data/federal.db`
**HTTP Port:** 8141 | **gRPC Port:** 9141

---

## 2. Database Schema

### 2.1 Federal Funds
```sql
CREATE TABLE federal_funds (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    fund_code TEXT NOT NULL,
    fund_name TEXT NOT NULL,
    fund_type TEXT NOT NULL
        CHECK(fund_type IN ('APPROPRIATION','REVOLVING','TRUST','GENERAL_FUND','SPECIAL')),
    treasury_symbol TEXT NOT NULL,
    budget_fiscal_year INTEGER NOT NULL,
    authority_type TEXT NOT NULL
        CHECK(authority_type IN ('DIRECT','BORROWING','CONTRACT_AUTHORITY')),
    available_balance_cents INTEGER NOT NULL DEFAULT 0,
    obligated_balance_cents INTEGER NOT NULL DEFAULT 0,
    expended_balance_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','CLOSED','CANCELLED')),
    expiration_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, fund_code, budget_fiscal_year)
);

CREATE INDEX idx_fed_funds_tenant_type ON federal_funds(tenant_id, fund_type);
CREATE INDEX idx_fed_funds_tenant_status ON federal_funds(tenant_id, status);
CREATE INDEX idx_fed_funds_tenant_fy ON federal_funds(tenant_id, budget_fiscal_year);
```

### 2.2 Appropriations
```sql
CREATE TABLE appropriations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    fund_id TEXT NOT NULL,
    appropriation_code TEXT NOT NULL,
    appropriation_title TEXT NOT NULL,
    fiscal_year INTEGER NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    enactment_date TEXT NOT NULL,
    public_law_reference TEXT,
    purpose TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','CANCELLED')),
    available_cents INTEGER NOT NULL DEFAULT 0,
    committed_cents INTEGER NOT NULL DEFAULT 0,
    obligated_cents INTEGER NOT NULL DEFAULT 0,
    expended_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (fund_id) REFERENCES federal_funds(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, appropriation_code, fiscal_year)
);

CREATE INDEX idx_approps_tenant_fund ON appropriations(tenant_id, fund_id);
CREATE INDEX idx_approps_tenant_fy ON appropriations(tenant_id, fiscal_year);
CREATE INDEX idx_approps_tenant_status ON appropriations(tenant_id, status);
```

### 2.3 Allotments
```sql
CREATE TABLE allotments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    appropriation_id TEXT NOT NULL,
    allotment_code TEXT NOT NULL,
    allottee_organization_id TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    issued_date TEXT NOT NULL,
    issued_by TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ISSUED'
        CHECK(status IN ('ISSUED','PARTIALLY_OBLIGATED','FULLY_OBLIGATED','CANCELLED')),
    available_cents INTEGER NOT NULL DEFAULT 0,
    obligated_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (appropriation_id) REFERENCES appropriations(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, allotment_code)
);

CREATE INDEX idx_allotments_tenant_approp ON allotments(tenant_id, appropriation_id);
CREATE INDEX idx_allotments_tenant_org ON allotments(tenant_id, allottee_organization_id);
CREATE INDEX idx_allotments_tenant_status ON allotments(tenant_id, status);
```

### 2.4 Obligations
```sql
CREATE TABLE obligations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    allotment_id TEXT NOT NULL,
    obligation_number TEXT NOT NULL,
    obligation_type TEXT NOT NULL DEFAULT 'INITIAL'
        CHECK(obligation_type IN ('INITIAL','MODIFICATION','INCREASE','DECREASE')),
    vendor_id TEXT,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    obligation_date TEXT NOT NULL,
    commitment_id TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','CERTIFIED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (allotment_id) REFERENCES allotments(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, obligation_number)
);

CREATE INDEX idx_obligations_tenant_allotment ON obligations(tenant_id, allotment_id);
CREATE INDEX idx_obligations_tenant_status ON obligations(tenant_id, status);
CREATE INDEX idx_obligations_tenant_vendor ON obligations(tenant_id, vendor_id);
CREATE INDEX idx_obligations_tenant_date ON obligations(tenant_id, obligation_date);
```

### 2.5 Federal Expenditures
```sql
CREATE TABLE federal_expenditures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    obligation_id TEXT NOT NULL,
    expenditure_type TEXT NOT NULL
        CHECK(expenditure_type IN ('ACCRUAL','CASH','DISBURSEMENT','TRANSFER')),
    amount_cents INTEGER NOT NULL DEFAULT 0,
    expenditure_date TEXT NOT NULL,
    vendor_id TEXT,
    payment_voucher_reference TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','PAID','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (obligation_id) REFERENCES obligations(id) ON DELETE RESTRICT
);

CREATE INDEX idx_fed_exp_tenant_obligation ON federal_expenditures(tenant_id, obligation_id);
CREATE INDEX idx_fed_exp_tenant_status ON federal_expenditures(tenant_id, status);
CREATE INDEX idx_fed_exp_tenant_date ON federal_expenditures(tenant_id, expenditure_date);
```

### 2.6 Federal Accounts
```sql
CREATE TABLE federal_accounts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    account_code TEXT NOT NULL,
    account_title TEXT NOT NULL,
    treasury_account_symbol TEXT NOT NULL,
    agency_identifier TEXT NOT NULL,
    account_type TEXT NOT NULL
        CHECK(account_type IN ('APPROPRIATION','RECEIPT','EXPENDITURE')),
    parent_account_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, account_code)
);

CREATE INDEX idx_fed_accts_tenant_type ON federal_accounts(tenant_id, account_type);
CREATE INDEX idx_fed_accts_tenant_agency ON federal_accounts(tenant_id, agency_identifier);
CREATE INDEX idx_fed_accts_tenant_parent ON federal_accounts(tenant_id, parent_account_id);
```

### 2.7 SF-133 Reports
```sql
CREATE TABLE sf133_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_period TEXT NOT NULL,
    fiscal_year INTEGER NOT NULL,
    quarter TEXT NOT NULL
        CHECK(quarter IN ('Q1','Q2','Q3','Q4')),
    line_number TEXT NOT NULL,
    line_description TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED')),
    prepared_by TEXT,
    approved_by TEXT,
    submitted_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, fiscal_year, quarter, line_number)
);

CREATE INDEX idx_sf133_tenant_fy ON sf133_reports(tenant_id, fiscal_year);
CREATE INDEX idx_sf133_tenant_status ON sf133_reports(tenant_id, status);
CREATE INDEX idx_sf133_tenant_quarter ON sf133_reports(tenant_id, fiscal_year, quarter);
```

### 2.8 CFDA Programs
```sql
CREATE TABLE cfda_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cfda_number TEXT NOT NULL,
    program_title TEXT NOT NULL,
    agency_code TEXT NOT NULL,
    assistance_type TEXT NOT NULL
        CHECK(assistance_type IN ('GRANT','LOAN','INSURANCE','OTHER')),
    federal_outlay_cents INTEGER NOT NULL DEFAULT 0,
    recipient_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, cfda_number)
);

CREATE INDEX idx_cfda_tenant_agency ON cfda_programs(tenant_id, agency_code);
CREATE INDEX idx_cfda_tenant_type ON cfda_programs(tenant_id, assistance_type);
CREATE INDEX idx_cfda_tenant_status ON cfda_programs(tenant_id, status);
```

---

## 3. REST API Endpoints

### Funds
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/funds` | Create a federal fund |
| GET | `/api/v1/federal/funds` | List federal funds |
| GET | `/api/v1/federal/funds/{id}` | Get fund details |
| PUT | `/api/v1/federal/funds/{id}` | Update fund |
| GET | `/api/v1/federal/funds/{id}/balance` | Get fund balance |

### Appropriations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/appropriations` | Create an appropriation |
| GET | `/api/v1/federal/appropriations` | List appropriations |
| GET | `/api/v1/federal/appropriations/{id}` | Get appropriation details |
| PUT | `/api/v1/federal/appropriations/{id}` | Update appropriation |
| POST | `/api/v1/federal/appropriations/{id}/allot` | Create an allotment from appropriation |

### Allotments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/allotments` | Create an allotment |
| GET | `/api/v1/federal/allotments` | List allotments |
| GET | `/api/v1/federal/allotments/{id}` | Get allotment details |
| PUT | `/api/v1/federal/allotments/{id}` | Update allotment |
| POST | `/api/v1/federal/allotments/{id}/obligate` | Create an obligation from allotment |

### Obligations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/obligations` | Create an obligation |
| GET | `/api/v1/federal/obligations` | List obligations |
| GET | `/api/v1/federal/obligations/{id}` | Get obligation details |
| PUT | `/api/v1/federal/obligations/{id}` | Update obligation |
| POST | `/api/v1/federal/obligations/{id}/certify` | Certify an obligation |
| POST | `/api/v1/federal/obligations/{id}/expend` | Record an expenditure |

### Expenditures
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/expenditures` | Create an expenditure |
| GET | `/api/v1/federal/expenditures` | List expenditures |
| GET | `/api/v1/federal/expenditures/{id}` | Get expenditure details |
| PUT | `/api/v1/federal/expenditures/{id}` | Update expenditure |

### Accounts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/accounts` | Create a federal account |
| GET | `/api/v1/federal/accounts` | List federal accounts |
| GET | `/api/v1/federal/accounts/{id}` | Get account details |
| PUT | `/api/v1/federal/accounts/{id}` | Update account |
| GET | `/api/v1/federal/accounts/{id}/hierarchy` | Get account hierarchy |

### Reports
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/reports/generate-sf133` | Generate SF-133 report |
| POST | `/api/v1/federal/reports/generate-gtas` | Generate GTAS report |
| POST | `/api/v1/federal/reports/generate-facts` | Generate FACTS I/II report |
| GET | `/api/v1/federal/reports/sf133` | List SF-133 reports |
| GET | `/api/v1/federal/reports/sf133/{id}` | Get SF-133 report details |

### CFDA
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/federal/cfda` | Create a CFDA program |
| GET | `/api/v1/federal/cfda` | List CFDA programs |
| GET | `/api/v1/federal/cfda/{id}` | Get CFDA program details |
| PUT | `/api/v1/federal/cfda/{id}` | Update CFDA program |
| GET | `/api/v1/federal/cfda/outlay-summary` | Get CFDA outlay summary |

---

## 4. Business Rules

1. An obligation MUST NOT exceed the available balance of its parent allotment; violations constitute Anti-Deficiency Act breaches.
2. An allotment MUST NOT exceed the available balance of its parent appropriation.
3. Total expended amount across all expenditures for an obligation MUST NOT exceed the obligated amount.
4. An obligation in `PENDING` status MUST NOT have expenditures recorded against it; the obligation MUST be `CERTIFIED` first.
5. A federal fund with `CANCELLED` status MUST NOT allow new obligations or expenditures; only existing obligated balances MAY be expended.
6. SF-133 report generation MUST validate that all line items balance according to OMB Circular A-11 requirements.
7. Appropriation available balance MUST equal total appropriation amount minus committed minus obligated minus expended amounts.
8. The system MUST enforce the fund expiration date; obligations MUST NOT be created against expired funds unless within the allowable obligation period.
9. CFDA program federal outlay totals MUST be reconciled with expenditure records at least quarterly.
10. Each obligation MUST reference a valid commitment or purchase order before it MAY be certified.
11. The system SHOULD provide a warning when fund utilization exceeds 80% of the appropriation amount.
12. Federal account hierarchy MUST support at least 5 levels of parent-child relationships.
13. An appropriation MUST NOT be created without a valid public law reference.
14. Expenditure records MUST be reconciled with disbursement records from the payment system before being marked as `PAID`.
15. The system MUST maintain a complete audit trail of all balance changes at the fund, appropriation, allotment, and obligation levels.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package federal.v1;

service FederalFinancialsService {
    // Funds
    rpc CreateFund(CreateFundRequest) returns (CreateFundResponse);
    rpc GetFund(GetFundRequest) returns (GetFundResponse);
    rpc ListFunds(ListFundsRequest) returns (ListFundsResponse);
    rpc UpdateFund(UpdateFundRequest) returns (UpdateFundResponse);
    rpc GetFundBalance(GetFundBalanceRequest) returns (GetFundBalanceResponse);

    // Appropriations
    rpc CreateAppropriation(CreateAppropriationRequest) returns (CreateAppropriationResponse);
    rpc GetAppropriation(GetAppropriationRequest) returns (GetAppropriationResponse);
    rpc ListAppropriations(ListAppropriationsRequest) returns (ListAppropriationsResponse);
    rpc UpdateAppropriation(UpdateAppropriationRequest) returns (UpdateAppropriationResponse);
    rpc AllotAppropriation(AllotAppropriationRequest) returns (AllotAppropriationResponse);

    // Allotments
    rpc CreateAllotment(CreateAllotmentRequest) returns (CreateAllotmentResponse);
    rpc GetAllotment(GetAllotmentRequest) returns (GetAllotmentResponse);
    rpc ListAllotments(ListAllotmentsRequest) returns (ListAllotmentsResponse);
    rpc UpdateAllotment(UpdateAllotmentRequest) returns (UpdateAllotmentResponse);
    rpc ObligateAllotment(ObligateAllotmentRequest) returns (ObligateAllotmentResponse);

    // Obligations
    rpc CreateObligation(CreateObligationRequest) returns (CreateObligationResponse);
    rpc GetObligation(GetObligationRequest) returns (GetObligationResponse);
    rpc ListObligations(ListObligationsRequest) returns (ListObligationsResponse);
    rpc UpdateObligation(UpdateObligationRequest) returns (UpdateObligationResponse);
    rpc CertifyObligation(CertifyObligationRequest) returns (CertifyObligationResponse);
    rpc ExpendObligation(ExpendObligationRequest) returns (ExpendObligationResponse);

    // Expenditures
    rpc CreateExpenditure(CreateExpenditureRequest) returns (CreateExpenditureResponse);
    rpc GetExpenditure(GetExpenditureRequest) returns (GetExpenditureResponse);
    rpc ListExpenditures(ListExpendituresRequest) returns (ListExpendituresResponse);
    rpc UpdateExpenditure(UpdateExpenditureRequest) returns (UpdateExpenditureResponse);

    // Accounts
    rpc CreateAccount(CreateAccountRequest) returns (CreateAccountResponse);
    rpc GetAccount(GetAccountRequest) returns (GetAccountResponse);
    rpc ListAccounts(ListAccountsRequest) returns (ListAccountsResponse);
    rpc UpdateAccount(UpdateAccountRequest) returns (UpdateAccountResponse);
    rpc GetAccountHierarchy(GetAccountHierarchyRequest) returns (GetAccountHierarchyResponse);

    // Reports
    rpc GenerateSF133(GenerateSF133Request) returns (GenerateSF133Response);
    rpc GenerateGTAS(GenerateGTASRequest) returns (GenerateGTASResponse);
    rpc GenerateFACTS(GenerateFACTSRequest) returns (GenerateFACTSResponse);
    rpc ListSF133Reports(ListSF133ReportsRequest) returns (ListSF133ReportsResponse);
    rpc GetSF133Report(GetSF133ReportRequest) returns (GetSF133ReportResponse);

    // CFDA
    rpc CreateCFDAProgram(CreateCFDAProgramRequest) returns (CreateCFDAProgramResponse);
    rpc GetCFDAProgram(GetCFDAProgramRequest) returns (GetCFDAProgramResponse);
    rpc ListCFDAPrograms(ListCFDAProgramsRequest) returns (ListCFDAProgramsResponse);
    rpc UpdateCFDAProgram(UpdateCFDAProgramRequest) returns (UpdateCFDAProgramResponse);
    rpc GetCFDAOutlaySummary(GetCFDAOutlaySummaryRequest) returns (GetCFDAOutlaySummaryResponse);
}

message FederalFund {
    string id = 1;
    string tenant_id = 2;
    string fund_code = 3;
    string fund_name = 4;
    string fund_type = 5;
    string treasury_symbol = 6;
    int32 budget_fiscal_year = 7;
    string authority_type = 8;
    int64 available_balance_cents = 9;
    int64 obligated_balance_cents = 10;
    int64 expended_balance_cents = 11;
    string status = 12;
    string expiration_date = 13;
}

message CreateFundRequest {
    string tenant_id = 1;
    string fund_code = 2;
    string fund_name = 3;
    string fund_type = 4;
    string treasury_symbol = 5;
    int32 budget_fiscal_year = 6;
    string authority_type = 7;
    int64 available_balance_cents = 8;
    string expiration_date = 9;
    string created_by = 10;
}

message CreateFundResponse {
    FederalFund fund = 1;
}

message GetFundRequest {
    string tenant_id = 1;
    string fund_id = 2;
}

message GetFundResponse {
    FederalFund fund = 1;
}

message ListFundsRequest {
    string tenant_id = 1;
    string fund_type = 2;
    string status = 3;
    int32 budget_fiscal_year = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListFundsResponse {
    repeated FederalFund funds = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateFundRequest {
    string tenant_id = 1;
    string fund_id = 2;
    string fund_name = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateFundResponse {
    FederalFund fund = 1;
}

message FundBalance {
    string fund_id = 1;
    int64 total_appropriated_cents = 2;
    int64 committed_cents = 3;
    int64 obligated_cents = 4;
    int64 expended_cents = 5;
    int64 available_cents = 6;
}

message GetFundBalanceRequest {
    string tenant_id = 1;
    string fund_id = 2;
}

message GetFundBalanceResponse {
    FundBalance balance = 1;
}

message Appropriation {
    string id = 1;
    string tenant_id = 2;
    string fund_id = 3;
    string appropriation_code = 4;
    string appropriation_title = 5;
    int32 fiscal_year = 6;
    int64 amount_cents = 7;
    string enactment_date = 8;
    string public_law_reference = 9;
    string status = 10;
    int64 available_cents = 11;
    int64 obligated_cents = 12;
    int64 expended_cents = 13;
}

message CreateAppropriationRequest {
    string tenant_id = 1;
    string fund_id = 2;
    string appropriation_code = 3;
    string appropriation_title = 4;
    int32 fiscal_year = 5;
    int64 amount_cents = 6;
    string enactment_date = 7;
    string public_law_reference = 8;
    string purpose = 9;
    string created_by = 10;
}

message CreateAppropriationResponse {
    Appropriation appropriation = 1;
}

message GetAppropriationRequest {
    string tenant_id = 1;
    string appropriation_id = 2;
}

message GetAppropriationResponse {
    Appropriation appropriation = 1;
}

message ListAppropriationsRequest {
    string tenant_id = 1;
    string fund_id = 2;
    int32 fiscal_year = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListAppropriationsResponse {
    repeated Appropriation appropriations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateAppropriationRequest {
    string tenant_id = 1;
    string appropriation_id = 2;
    string appropriation_title = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateAppropriationResponse {
    Appropriation appropriation = 1;
}

message AllotAppropriationRequest {
    string tenant_id = 1;
    string appropriation_id = 2;
    string allotment_code = 3;
    string allottee_organization_id = 4;
    int64 amount_cents = 5;
    string issued_by = 6;
}

message AllotAppropriationResponse {
    string allotment_id = 1;
    string allotment_code = 2;
    int64 amount_cents = 3;
}

message Allotment {
    string id = 1;
    string tenant_id = 2;
    string appropriation_id = 3;
    string allotment_code = 4;
    string allottee_organization_id = 5;
    int64 amount_cents = 6;
    string issued_date = 7;
    string status = 8;
    int64 available_cents = 9;
    int64 obligated_cents = 10;
}

message CreateAllotmentRequest {
    string tenant_id = 1;
    string appropriation_id = 2;
    string allotment_code = 3;
    string allottee_organization_id = 4;
    int64 amount_cents = 5;
    string issued_by = 6;
    string created_by = 7;
}

message CreateAllotmentResponse {
    Allotment allotment = 1;
}

message GetAllotmentRequest {
    string tenant_id = 1;
    string allotment_id = 2;
}

message GetAllotmentResponse {
    Allotment allotment = 1;
}

message ListAllotmentsRequest {
    string tenant_id = 1;
    string appropriation_id = 2;
    string allottee_organization_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListAllotmentsResponse {
    repeated Allotment allotments = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateAllotmentRequest {
    string tenant_id = 1;
    string allotment_id = 2;
    string updated_by = 3;
    int32 version = 4;
}

message UpdateAllotmentResponse {
    Allotment allotment = 1;
}

message ObligateAllotmentRequest {
    string tenant_id = 1;
    string allotment_id = 2;
    string obligation_number = 3;
    string vendor_id = 4;
    int64 amount_cents = 5;
    string description = 6;
    string commitment_id = 7;
    string created_by = 8;
}

message ObligateAllotmentResponse {
    string obligation_id = 1;
    string obligation_number = 2;
}

message Obligation {
    string id = 1;
    string tenant_id = 2;
    string allotment_id = 3;
    string obligation_number = 4;
    string obligation_type = 5;
    string vendor_id = 6;
    int64 amount_cents = 7;
    string description = 8;
    string obligation_date = 9;
    string status = 10;
}

message CreateObligationRequest {
    string tenant_id = 1;
    string allotment_id = 2;
    string obligation_number = 3;
    string obligation_type = 4;
    string vendor_id = 5;
    int64 amount_cents = 6;
    string description = 7;
    string obligation_date = 8;
    string commitment_id = 9;
    string created_by = 10;
}

message CreateObligationResponse {
    Obligation obligation = 1;
}

message GetObligationRequest {
    string tenant_id = 1;
    string obligation_id = 2;
}

message GetObligationResponse {
    Obligation obligation = 1;
}

message ListObligationsRequest {
    string tenant_id = 1;
    string allotment_id = 2;
    string vendor_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListObligationsResponse {
    repeated Obligation obligations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateObligationRequest {
    string tenant_id = 1;
    string obligation_id = 2;
    string description = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateObligationResponse {
    Obligation obligation = 1;
}

message CertifyObligationRequest {
    string tenant_id = 1;
    string obligation_id = 2;
    string certified_by = 3;
}

message CertifyObligationResponse {
    Obligation obligation = 1;
    string certified_at = 2;
}

message ExpendObligationRequest {
    string tenant_id = 1;
    string obligation_id = 2;
    string expenditure_type = 3;
    int64 amount_cents = 4;
    string expenditure_date = 5;
    string vendor_id = 6;
    string created_by = 7;
}

message ExpendObligationResponse {
    string expenditure_id = 1;
    int64 remaining_obligation_cents = 2;
}

message Expenditure {
    string id = 1;
    string tenant_id = 2;
    string obligation_id = 3;
    string expenditure_type = 4;
    int64 amount_cents = 5;
    string expenditure_date = 6;
    string vendor_id = 7;
    string status = 8;
}

message CreateExpenditureRequest {
    string tenant_id = 1;
    string obligation_id = 2;
    string expenditure_type = 3;
    int64 amount_cents = 4;
    string expenditure_date = 5;
    string vendor_id = 6;
    string payment_voucher_reference = 7;
    string created_by = 8;
}

message CreateExpenditureResponse {
    Expenditure expenditure = 1;
}

message GetExpenditureRequest {
    string tenant_id = 1;
    string expenditure_id = 2;
}

message GetExpenditureResponse {
    Expenditure expenditure = 1;
}

message ListExpendituresRequest {
    string tenant_id = 1;
    string obligation_id = 2;
    string expenditure_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListExpendituresResponse {
    repeated Expenditure expenditures = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateExpenditureRequest {
    string tenant_id = 1;
    string expenditure_id = 2;
    string payment_voucher_reference = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateExpenditureResponse {
    Expenditure expenditure = 1;
}

message FederalAccount {
    string id = 1;
    string tenant_id = 2;
    string account_code = 3;
    string account_title = 4;
    string treasury_account_symbol = 5;
    string agency_identifier = 6;
    string account_type = 7;
    string parent_account_id = 8;
    string status = 9;
}

message CreateAccountRequest {
    string tenant_id = 1;
    string account_code = 2;
    string account_title = 3;
    string treasury_account_symbol = 4;
    string agency_identifier = 5;
    string account_type = 6;
    string parent_account_id = 7;
    string created_by = 8;
}

message CreateAccountResponse {
    FederalAccount account = 1;
}

message GetAccountRequest {
    string tenant_id = 1;
    string account_id = 2;
}

message GetAccountResponse {
    FederalAccount account = 1;
}

message ListAccountsRequest {
    string tenant_id = 1;
    string account_type = 2;
    string agency_identifier = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListAccountsResponse {
    repeated FederalAccount accounts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateAccountRequest {
    string tenant_id = 1;
    string account_id = 2;
    string account_title = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateAccountResponse {
    FederalAccount account = 1;
}

message AccountHierarchyNode {
    FederalAccount account = 1;
    repeated AccountHierarchyNode children = 2;
}

message GetAccountHierarchyRequest {
    string tenant_id = 1;
    string account_id = 2;
}

message GetAccountHierarchyResponse {
    AccountHierarchyNode root = 1;
}

message SF133Line {
    string id = 1;
    string line_number = 2;
    string line_description = 3;
    int64 amount_cents = 4;
}

message GenerateSF133Request {
    string tenant_id = 1;
    int32 fiscal_year = 2;
    string quarter = 3;
    string generated_by = 4;
}

message GenerateSF133Response {
    string report_id = 1;
    repeated SF133Line lines = 2;
    string status = 3;
}

message GenerateGTASRequest {
    string tenant_id = 1;
    int32 fiscal_year = 2;
    string quarter = 3;
    string generated_by = 4;
}

message GenerateGTASResponse {
    string report_id = 1;
    string report_data = 2;  -- JSON
    string status = 3;
}

message GenerateFACTSRequest {
    string tenant_id = 1;
    int32 fiscal_year = 2;
    string facts_type = 3;  -- FACTS_I or FACTS_II
    string generated_by = 4;
}

message GenerateFACTSResponse {
    string report_id = 1;
    string report_data = 2;  -- JSON
    string status = 3;
}

message SF133Report {
    string id = 1;
    string tenant_id = 2;
    int32 fiscal_year = 3;
    string quarter = 4;
    string status = 5;
    string prepared_by = 6;
    string submitted_date = 7;
}

message ListSF133ReportsRequest {
    string tenant_id = 1;
    int32 fiscal_year = 2;
    string quarter = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListSF133ReportsResponse {
    repeated SF133Report reports = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetSF133ReportRequest {
    string tenant_id = 1;
    string report_id = 2;
}

message GetSF133ReportResponse {
    SF133Report report = 1;
    repeated SF133Line lines = 2;
}

message CFDAProgram {
    string id = 1;
    string tenant_id = 2;
    string cfda_number = 3;
    string program_title = 4;
    string agency_code = 5;
    string assistance_type = 6;
    int64 federal_outlay_cents = 7;
    int32 recipient_count = 8;
    string status = 9;
}

message CreateCFDAProgramRequest {
    string tenant_id = 1;
    string cfda_number = 2;
    string program_title = 3;
    string agency_code = 4;
    string assistance_type = 5;
    string created_by = 6;
}

message CreateCFDAProgramResponse {
    CFDAProgram program = 1;
}

message GetCFDAProgramRequest {
    string tenant_id = 1;
    string cfda_id = 2;
}

message GetCFDAProgramResponse {
    CFDAProgram program = 1;
}

message ListCFDAProgramsRequest {
    string tenant_id = 1;
    string agency_code = 2;
    string assistance_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCFDAProgramsResponse {
    repeated CFDAProgram programs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateCFDAProgramRequest {
    string tenant_id = 1;
    string cfda_id = 2;
    string program_title = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateCFDAProgramResponse {
    CFDAProgram program = 1;
}

message CFDAOutlaySummary {
    string assistance_type = 1;
    int32 program_count = 2;
    int64 total_outlay_cents = 3;
    int32 total_recipients = 4;
}

message GetCFDAOutlaySummaryRequest {
    string tenant_id = 1;
    int32 fiscal_year = 2;
}

message GetCFDAOutlaySummaryResponse {
    repeated CFDAOutlaySummary summaries = 1;
    int64 grand_total_outlay_cents = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | USSGL account balances, journal entries | Federal account reconciliation |
| `ap-service` | Vendor invoices, payment vouchers | Obligation and expenditure processing |
| `procurement-service` | Purchase orders, commitments | Commitment validation for obligations |
| `cash-management` | Disbursement records | Expenditure reconciliation |
| `budget-service` | Budget allocations, spending limits | Appropriation and fund validation |
| `user-service` | Agency organization hierarchy, certifiers | Allotment issuance and obligation certification |
| `reporting-service` | Report templates, filing formats | SF-133, GTAS, FACTS report generation |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | Federal accounting entries (USSGL) | Record federal financial transactions |
| `cash-management` | Payment requests for expenditures | Execute federal disbursements |
| `reporting-service` | SF-133, GTAS, FACTS report data | Federal compliance reporting |
| `audit-service` | Fund balance changes, obligation certifications | Federal audit trail |
| `grant-management` | CFDA outlay data | Grant program financial reporting |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `AppropriationEnacted` | `federal.appropriation.enacted` | `{ tenant_id, appropriation_id, appropriation_code, fiscal_year, amount_cents, enactment_date }` | Published when a new appropriation is recorded |
| `AllotmentIssued` | `federal.allotment.issued` | `{ tenant_id, allotment_id, allotment_code, appropriation_id, amount_cents, allottee_organization_id }` | Published when an allotment is issued |
| `ObligationCreated` | `federal.obligation.created` | `{ tenant_id, obligation_id, obligation_number, allotment_id, amount_cents, vendor_id }` | Published when a new obligation is recorded |
| `ExpenditureRecorded` | `federal.expenditure.recorded` | `{ tenant_id, expenditure_id, obligation_id, amount_cents, expenditure_type, expenditure_date }` | Published when an expenditure is recorded |
| `SF133Generated` | `federal.sf133.generated` | `{ tenant_id, report_id, fiscal_year, quarter, line_count, status }` | Published when an SF-133 report is generated |
