# 85 - Tax Reporting Service Specification

## 1. Domain Overview

Tax Reporting provides tax provision calculations compliant with ASC 740, deferred tax asset and liability tracking, effective tax rate management by jurisdiction, tax return data preparation, book-to-tax adjustment management, tax credit tracking (R&D, foreign tax credits, and other incentives), and tax audit examination management. Supports quarterly and annual provision runs, multi-jurisdiction rate tables (federal, state, local, international), and automated book-to-tax difference identification. Integrates with GL for book income data, TAX for tax transaction details, and EPM for tax planning and forecasting.

**Bounded Context:** Tax Provision & Reporting
**Service Name:** `taxreport-service`
**Database:** `data/taxreport.db`
**HTTP Port:** 8117 | **gRPC Port:** 9117

---

## 2. Database Schema

### 2.1 Tax Jurisdictions
```sql
CREATE TABLE taxreport_jurisdictions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jurisdiction_name TEXT NOT NULL,
    jurisdiction_type TEXT NOT NULL
        CHECK(jurisdiction_type IN ('FEDERAL','STATE','LOCAL','INTERNATIONAL')),
    country_code TEXT NOT NULL,
    region_code TEXT,
    statutory_rate_real REAL NOT NULL DEFAULT 0.0,
    effective_rate_real REAL NOT NULL DEFAULT 0.0,
    filing_frequency TEXT NOT NULL DEFAULT 'ANNUAL'
        CHECK(filing_frequency IN ('MONTHLY','QUARTERLY','ANNUAL')),
    return_due_date TEXT,
    extension_due_date TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, jurisdiction_name, country_code)
);

CREATE INDEX idx_jurisdictions_tenant_type ON taxreport_jurisdictions(tenant_id, jurisdiction_type);
CREATE INDEX idx_jurisdictions_tenant_country ON taxreport_jurisdictions(tenant_id, country_code);
CREATE INDEX idx_jurisdictions_tenant_active ON taxreport_jurisdictions(tenant_id, is_active);
```

### 2.2 Tax Entities
```sql
CREATE TABLE taxreport_entities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    entity_code TEXT NOT NULL,
    legal_entity_id TEXT,
    tax_id_number TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('CORPORATION','PARTNERSHIP','S_CORP','LLC','FOREIGN','DISREGARDED')),
    filing_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(filing_status IN ('ACTIVE','INACTIVE','MERGED','CONSOLIDATED')),
    consolidation_group TEXT,
    parent_entity_id TEXT,
    fiscal_year_end TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jurisdiction_id) REFERENCES taxreport_jurisdictions(id),
    UNIQUE(tenant_id, entity_code)
);

CREATE INDEX idx_entities_tenant_jurisdiction ON taxreport_entities(tenant_id, jurisdiction_id);
CREATE INDEX idx_entities_tenant_type ON taxreport_entities(tenant_id, entity_type);
CREATE INDEX idx_entities_tenant_status ON taxreport_entities(tenant_id, filing_status);
CREATE INDEX idx_entities_tenant_group ON taxreport_entities(tenant_id, consolidation_group);
```

### 2.3 Tax Provision Runs
```sql
CREATE TABLE taxreport_provision_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_name TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('Q1','Q2','Q3','Q4','ANNUAL')),
    fiscal_year TEXT NOT NULL,
    book_income_cents INTEGER NOT NULL DEFAULT 0,
    tax_income_cents INTEGER NOT NULL DEFAULT 0,
    current_tax_expense_cents INTEGER NOT NULL DEFAULT 0,
    deferred_tax_expense_cents INTEGER NOT NULL DEFAULT 0,
    total_tax_expense_cents INTEGER NOT NULL DEFAULT 0,
    effective_rate_real REAL NOT NULL DEFAULT 0.0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','APPROVED','POSTED','ARCHIVED')),
    started_at TEXT,
    completed_at TEXT,
    approved_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id),
    UNIQUE(tenant_id, entity_id, period_type, fiscal_year)
);

CREATE INDEX idx_provision_runs_tenant_entity ON taxreport_provision_runs(tenant_id, entity_id);
CREATE INDEX idx_provision_runs_tenant_status ON taxreport_provision_runs(tenant_id, status);
CREATE INDEX idx_provision_runs_tenant_year ON taxreport_provision_runs(tenant_id, fiscal_year);
CREATE INDEX idx_provision_runs_tenant_period ON taxreport_provision_runs(tenant_id, period_type);
```

### 2.4 Deferred Tax Assets
```sql
CREATE TABLE taxreport_deferred_tax_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    provision_run_id TEXT NOT NULL,
    dta_category TEXT NOT NULL
        CHECK(dta_category IN ('NET_OPERATING_LOSS','TAX_CREDIT_CARRYFORWARD','ACCRUAL_RESERVE','DEPRECIATION','LEASE_LIABILITY','OTHER_TEMPORARY')),
    description TEXT NOT NULL,
    beginning_balance_cents INTEGER NOT NULL DEFAULT 0,
    current_period_change_cents INTEGER NOT NULL DEFAULT 0,
    ending_balance_cents INTEGER NOT NULL DEFAULT 0,
    tax_rate_real REAL NOT NULL DEFAULT 0.0,
    gross_dta_cents INTEGER NOT NULL DEFAULT 0,
    valuation_allowance_cents INTEGER NOT NULL DEFAULT 0,
    net_dta_cents INTEGER NOT NULL DEFAULT 0,
    expiration_date TEXT,
    carryforward_years INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id),
    FOREIGN KEY (provision_run_id) REFERENCES taxreport_provision_runs(id)
);

CREATE INDEX idx_dta_tenant_entity ON taxreport_deferred_tax_assets(tenant_id, entity_id);
CREATE INDEX idx_dta_tenant_category ON taxreport_deferred_tax_assets(tenant_id, dta_category);
CREATE INDEX idx_dta_tenant_run ON taxreport_deferred_tax_assets(tenant_id, provision_run_id);
```

### 2.5 Deferred Tax Liabilities
```sql
CREATE TABLE taxreport_deferred_tax_liabilities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    provision_run_id TEXT NOT NULL,
    dtl_category TEXT NOT NULL
        CHECK(dtl_category IN ('ACCELERATED_DEPRECIATION','INSTALLMENT_SALE','PREPAID_EXPENSE','LEASE_ASSET','GOODWILL','OTHER_TEMPORARY')),
    description TEXT NOT NULL,
    beginning_balance_cents INTEGER NOT NULL DEFAULT 0,
    current_period_change_cents INTEGER NOT NULL DEFAULT 0,
    ending_balance_cents INTEGER NOT NULL DEFAULT 0,
    tax_rate_real REAL NOT NULL DEFAULT 0.0,
    gross_dtl_cents INTEGER NOT NULL DEFAULT 0,
    reversal_period TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id),
    FOREIGN KEY (provision_run_id) REFERENCES taxreport_provision_runs(id)
);

CREATE INDEX idx_dtl_tenant_entity ON taxreport_deferred_tax_liabilities(tenant_id, entity_id);
CREATE INDEX idx_dtl_tenant_category ON taxreport_deferred_tax_liabilities(tenant_id, dtl_category);
CREATE INDEX idx_dtl_tenant_run ON taxreport_deferred_tax_liabilities(tenant_id, provision_run_id);
```

### 2.6 Tax Rate Tables
```sql
CREATE TABLE taxreport_rate_tables (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    rate_type TEXT NOT NULL
        CHECK(rate_type IN ('STATUTORY','EFFECTIVE','MARGINAL','BLENDED','ALTERNATIVE_MINIMUM')),
    rate_value_real REAL NOT NULL DEFAULT 0.0,
    income_bracket_min_cents INTEGER NOT NULL DEFAULT 0,
    income_bracket_max_cents INTEGER,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jurisdiction_id) REFERENCES taxreport_jurisdictions(id),
    UNIQUE(tenant_id, jurisdiction_id, rate_type, effective_from)
);

CREATE INDEX idx_rate_tables_tenant_jurisdiction ON taxreport_rate_tables(tenant_id, jurisdiction_id);
CREATE INDEX idx_rate_tables_tenant_type ON taxreport_rate_tables(tenant_id, rate_type);
CREATE INDEX idx_rate_tables_tenant_active ON taxreport_rate_tables(tenant_id, is_active);
CREATE INDEX idx_rate_tables_tenant_effective ON taxreport_rate_tables(tenant_id, effective_from, effective_to);
```

### 2.7 Tax Adjustments
```sql
CREATE TABLE taxreport_adjustments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    provision_run_id TEXT NOT NULL,
    adjustment_type TEXT NOT NULL
        CHECK(adjustment_type IN ('PERMANENT','TEMPORARY','RETURN_TO_PROVISION','UNCERTAIN_TAX_POSITION','OTHER')),
    adjustment_category TEXT NOT NULL
        CHECK(adjustment_category IN ('MEALS_ENTERTAINMENT','DEPRECIATION','GOODWILL_AMORT','STATE_TAX','NONTAXABLE_INCOME','ACCRUAL_TO_CASH','OTHER')),
    description TEXT NOT NULL,
    book_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    difference_cents INTEGER NOT NULL DEFAULT 0,
    is_recurring INTEGER NOT NULL DEFAULT 0,
    gl_account_id TEXT,
    reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id),
    FOREIGN KEY (provision_run_id) REFERENCES taxreport_provision_runs(id)
);

CREATE INDEX idx_adjustments_tenant_entity ON taxreport_adjustments(tenant_id, entity_id);
CREATE INDEX idx_adjustments_tenant_type ON taxreport_adjustments(tenant_id, adjustment_type);
CREATE INDEX idx_adjustments_tenant_run ON taxreport_adjustments(tenant_id, provision_run_id);
CREATE INDEX idx_adjustments_tenant_category ON taxreport_adjustments(tenant_id, adjustment_category);
```

### 2.8 Tax Credits
```sql
CREATE TABLE taxreport_credits (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    provision_run_id TEXT,
    credit_type TEXT NOT NULL
        CHECK(credit_type IN ('RD_CREDIT','FOREIGN_TAX_CREDIT','GENERAL_BUSINESS','INVESTMENT_TAX','LOW_INCOME_HOUSING','ENERGY','WORK OPPORTUNITY','OTHER')),
    credit_name TEXT NOT NULL,
    credit_amount_cents INTEGER NOT NULL DEFAULT 0,
    utilized_amount_cents INTEGER NOT NULL DEFAULT 0,
    carryforward_amount_cents INTEGER NOT NULL DEFAULT 0,
    carryback_amount_cents INTEGER NOT NULL DEFAULT 0,
    expiration_date TEXT,
    source_jurisdiction_id TEXT,
    supporting_document_id TEXT,
    status TEXT NOT NULL DEFAULT 'CLAIMED'
        CHECK(status IN ('CLAIMED','VERIFIED','DENIED','CARRIED_FORWARD')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id)
);

CREATE INDEX idx_credits_tenant_entity ON taxreport_credits(tenant_id, entity_id);
CREATE INDEX idx_credits_tenant_type ON taxreport_credits(tenant_id, credit_type);
CREATE INDEX idx_credits_tenant_status ON taxreport_credits(tenant_id, status);
CREATE INDEX idx_credits_tenant_expiration ON taxreport_credits(tenant_id, expiration_date);
```

### 2.9 Tax Audit Tracking
```sql
CREATE TABLE taxreport_audit_tracking (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    audit_type TEXT NOT NULL
        CHECK(audit_type IN ('FEDERAL_EXAM','STATE_EXAM','LOCAL_EXAM','INTERNATIONAL_REVIEW','INFORMATION_REQUEST')),
    tax_years TEXT NOT NULL,                       -- JSON array of fiscal years under exam
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_PROGRESS','PROPOSED_ADJUSTMENT','AGREED','LITIGATION','CLOSED')),
    examining_agent TEXT,
    assigned_counsel TEXT,
    proposed_adjustment_cents INTEGER NOT NULL DEFAULT 0,
    potential_penalty_cents INTEGER NOT NULL DEFAULT 0,
    potential_interest_cents INTEGER NOT NULL DEFAULT 0,
    resolution_amount_cents INTEGER NOT NULL DEFAULT 0,
    open_date TEXT NOT NULL,
    close_date TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id),
    FOREIGN KEY (jurisdiction_id) REFERENCES taxreport_jurisdictions(id)
);

CREATE INDEX idx_audit_tenant_entity ON taxreport_audit_tracking(tenant_id, entity_id);
CREATE INDEX idx_audit_tenant_status ON taxreport_audit_tracking(tenant_id, status);
CREATE INDEX idx_audit_tenant_jurisdiction ON taxreport_audit_tracking(tenant_id, jurisdiction_id);
CREATE INDEX idx_audit_tenant_dates ON taxreport_audit_tracking(tenant_id, open_date, close_date);
```

### 2.10 Tax Return Data
```sql
CREATE TABLE taxreport_return_data (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    fiscal_year TEXT NOT NULL,
    return_type TEXT NOT NULL
        CHECK(return_type IN ('1120','1120S','1065','1040','INTERNATIONAL','STATE_RETURN','LOCAL_RETURN')),
    taxable_income_cents INTEGER NOT NULL DEFAULT 0,
    tax_liability_cents INTEGER NOT NULL DEFAULT 0,
    estimated_payments_cents INTEGER NOT NULL DEFAULT 0,
    balance_due_cents INTEGER NOT NULL DEFAULT 0,
    refund_amount_cents INTEGER NOT NULL DEFAULT 0,
    filing_date TEXT,
    filing_status TEXT NOT NULL DEFAULT 'PREPARATION'
        CHECK(filing_status IN ('PREPARATION','REVIEW','READY_TO_FILE','FILED','EXTENDED','AMENDED')),
    e_file_confirmation TEXT,
    preparer_id TEXT,
    reviewer_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (entity_id) REFERENCES taxreport_entities(id),
    FOREIGN KEY (jurisdiction_id) REFERENCES taxreport_jurisdictions(id),
    UNIQUE(tenant_id, entity_id, jurisdiction_id, fiscal_year, return_type)
);

CREATE INDEX idx_return_data_tenant_entity ON taxreport_return_data(tenant_id, entity_id);
CREATE INDEX idx_return_data_tenant_status ON taxreport_return_data(tenant_id, filing_status);
CREATE INDEX idx_return_data_tenant_year ON taxreport_return_data(tenant_id, fiscal_year);
CREATE INDEX idx_return_data_tenant_type ON taxreport_return_data(tenant_id, return_type);
```

---

## 3. REST API Endpoints

```
# Tax Jurisdictions
GET/POST      /api/v1/taxreport/jurisdictions                Permission: taxreport.jurisdictions.read/create
GET/PUT       /api/v1/taxreport/jurisdictions/{id}            Permission: taxreport.jurisdictions.read/update
PATCH         /api/v1/taxreport/jurisdictions/{id}/deactivate Permission: taxreport.jurisdictions.update

# Tax Entities
GET/POST      /api/v1/taxreport/entities                      Permission: taxreport.entities.read/create
GET/PUT       /api/v1/taxreport/entities/{id}                  Permission: taxreport.entities.read/update
GET           /api/v1/taxreport/entities/{id}/consolidated     Permission: taxreport.entities.read

# Tax Provision Runs
GET/POST      /api/v1/taxreport/provisions                    Permission: taxreport.provisions.read/create
GET           /api/v1/taxreport/provisions/{id}                Permission: taxreport.provisions.read
POST          /api/v1/taxreport/provisions/{id}/run            Permission: taxreport.provisions.execute
POST          /api/v1/taxreport/provisions/{id}/approve        Permission: taxreport.provisions.approve
GET           /api/v1/taxreport/provisions/{id}/summary        Permission: taxreport.provisions.read
GET           /api/v1/taxreport/provisions/{id}/rate-reconciliation Permission: taxreport.provisions.read

# Deferred Tax
GET/POST      /api/v1/taxreport/deferred-tax/assets            Permission: taxreport.deferred.read/create
GET/PUT       /api/v1/taxreport/deferred-tax/assets/{id}       Permission: taxreport.deferred.read/update
GET/POST      /api/v1/taxreport/deferred-tax/liabilities       Permission: taxreport.deferred.read/create
GET/PUT       /api/v1/taxreport/deferred-tax/liabilities/{id}  Permission: taxreport.deferred.read/update
GET           /api/v1/taxreport/deferred-tax/rollforward       Permission: taxreport.deferred.read
POST          /api/v1/taxreport/deferred-tax/calculate          Permission: taxreport.deferred.execute

# Tax Rate Tables
GET/POST      /api/v1/taxreport/rate-tables                    Permission: taxreport.rates.read/create
GET/PUT       /api/v1/taxreport/rate-tables/{id}               Permission: taxreport.rates.read/update
GET           /api/v1/taxreport/rate-tables/jurisdiction/{id}  Permission: taxreport.rates.read
GET           /api/v1/taxreport/rate-tables/effective-rates    Permission: taxreport.rates.read

# Tax Adjustments
GET/POST      /api/v1/taxreport/adjustments                    Permission: taxreport.adjustments.read/create
GET/PUT       /api/v1/taxreport/adjustments/{id}               Permission: taxreport.adjustments.read/update
GET           /api/v1/taxreport/adjustments/provision/{id}     Permission: taxreport.adjustments.read
POST          /api/v1/taxreport/adjustments/auto-detect         Permission: taxreport.adjustments.execute

# Tax Credits
GET/POST      /api/v1/taxreport/credits                        Permission: taxreport.credits.read/create
GET/PUT       /api/v1/taxreport/credits/{id}                   Permission: taxreport.credits.read/update
GET           /api/v1/taxreport/credits/utilization             Permission: taxreport.credits.read
GET           /api/v1/taxreport/credits/carryforward-summary    Permission: taxreport.credits.read

# Tax Return Data
GET/POST      /api/v1/taxreport/returns                        Permission: taxreport.returns.read/create
GET/PUT       /api/v1/taxreport/returns/{id}                   Permission: taxreport.returns.read/update
POST          /api/v1/taxreport/returns/{id}/prepare            Permission: taxreport.returns.execute
POST          /api/v1/taxreport/returns/{id}/file               Permission: taxreport.returns.file
GET           /api/v1/taxreport/returns/{id}/workpapers         Permission: taxreport.returns.read

# Tax Audit Tracking
GET/POST      /api/v1/taxreport/audits                         Permission: taxreport.audits.read/create
GET/PUT       /api/v1/taxreport/audits/{id}                    Permission: taxreport.audits.read/update
POST          /api/v1/taxreport/audits/{id}/propose-adjustment Permission: taxreport.audits.update
POST          /api/v1/taxreport/audits/{id}/close              Permission: taxreport.audits.close
GET           /api/v1/taxreport/audits/{id}/timeline           Permission: taxreport.audits.read

# Reports
GET           /api/v1/taxreport/reports/provision-summary      Permission: taxreport.reports.view
GET           /api/v1/taxreport/reports/effective-rate          Permission: taxreport.reports.view
GET           /api/v1/taxreport/reports/deferred-tax-rollforward Permission: taxreport.reports.view
GET           /api/v1/taxreport/reports/uncertain-tax-positions Permission: taxreport.reports.view
GET           /api/v1/taxreport/reports/credit-utilization      Permission: taxreport.reports.view
```

---

## 4. Business Rules

### 4.1 Tax Provision Processing
1. A provision run MUST calculate current tax expense using the applicable statutory rate for each jurisdiction
2. Quarterly provisions MUST use the year-to-date approach (cumulative) for effective rate computation
3. Annual provisions MUST reconcile total tax expense to the effective tax rate
4. Provision calculations MUST incorporate all permanent and temporary differences
5. The effective tax rate reconciliation MUST explain differences between statutory and effective rates
6. Provision runs are immutable once approved; corrections require a new run

### 4.2 Deferred Tax Accounting
1. Deferred tax assets and liabilities MUST be classified as current or non-current on the balance sheet
2. A valuation allowance MUST be assessed for DTAs where "more likely than not" standard is not met
3. Deferred tax balances MUST be recalculated each period using enacted tax rates
4. DTA carryforward expiration dates MUST be tracked and flagged when approaching expiration
5. Net DTA/DTL presentation MUST follow the netting rules by jurisdiction
6. Rollforward schedules MUST show beginning balance, changes, and ending balance for each DTA/DTL category

### 4.3 Tax Adjustments
1. Permanent differences (non-deductible items) MUST be identified and excluded from deferred tax calculations
2. Temporary differences MUST generate corresponding deferred tax entries
3. Book-to-tax adjustments MUST reference GL account IDs for traceability
4. Uncertain tax positions MUST be evaluated under ASC 740-10 (FIN 48) recognition threshold
5. Auto-detection of adjustments SHOULD compare GL trial balance to prior year return data
6. Recurring adjustments SHOULD be flagged for automatic carry-forward to future periods

### 4.4 Tax Credits
1. Tax credits MUST be utilized in the order required by applicable tax law
2. Foreign tax credits MUST be computed separately for each basket category
3. R&D credit calculations MUST follow IRC Section 41 rules (regular or ASC methodology)
4. Carryforward credits MUST be tracked with expiration dates and utilization history
5. Credit utilization MUST not exceed the applicable limitation (e.g., foreign tax credit limitation)
6. Unused credits from expired periods MUST be removed from carryforward schedules

### 4.5 Tax Audit Management
1. Open audits MUST be assessed for potential exposure and accrued if probable and estimable
2. Proposed adjustments MUST be evaluated for reasonableness before agreement
3. Audit settlements MUST be reflected in the provision as either current or deferred tax expense
4. Interest and penalties on proposed adjustments MUST be computed using applicable rates
5. Closed audits MUST retain all documentation for the required retention period
6. Litigation reserves MUST be established when audit disputes proceed to litigation

### 4.6 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `taxreport.provision.started` | Provision run initiated | GL, EPM |
| `taxreport.provision.calculated` | Provision calculation completes | GL, EPM, Reporting |
| `taxreport.deferred.updated` | Deferred tax balances recalculated | GL, Reporting |
| `taxreport.return.prepared` | Tax return data finalized | GL, Workflow |
| `taxreport.credit.utilized` | Tax credit applied to liability | GL, Reporting |
| `taxreport.audit.status.changed` | Audit status transitions | Workflow, Notification |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.taxreport.v1;

service TaxReportService {
    rpc RunProvision(RunProvisionRequest) returns (RunProvisionResponse);
    rpc CalculateDeferredTax(CalculateDeferredTaxRequest) returns (CalculateDeferredTaxResponse);
    rpc GetProvisionSummary(GetProvisionSummaryRequest) returns (GetProvisionSummaryResponse);
    rpc GetEffectiveRateReconciliation(GetEffectiveRateReconciliationRequest) returns (GetEffectiveRateReconciliationResponse);
    rpc GetDeferredTaxRollforward(GetDeferredTaxRollforwardRequest) returns (GetDeferredTaxRollforwardResponse);
    rpc PrepareReturnData(PrepareReturnDataRequest) returns (PrepareReturnDataResponse);
}

// --- Tax Jurisdiction ---
message TaxJurisdiction {
    string id = 1; string tenant_id = 2; string jurisdiction_name = 3; string jurisdiction_type = 4;
    string country_code = 5; string region_code = 6; double statutory_rate_real = 7;
    double effective_rate_real = 8; string filing_frequency = 9; string return_due_date = 10;
    string extension_due_date = 11; string description = 12;
    string created_at = 13; string updated_at = 14; string created_by = 15; string updated_by = 16;
    int32 version = 17; bool is_active = 18;
}

// --- Tax Entity ---
message TaxEntity {
    string id = 1; string tenant_id = 2; string entity_name = 3; string entity_code = 4;
    string legal_entity_id = 5; string tax_id_number = 6; string jurisdiction_id = 7;
    string entity_type = 8; string filing_status = 9; string consolidation_group = 10;
    string parent_entity_id = 11; string fiscal_year_end = 12;
    string created_at = 13; string updated_at = 14; string created_by = 15; string updated_by = 16;
    int32 version = 17; bool is_active = 18;
}

// --- Provision Run ---
message ProvisionRun {
    string id = 1; string tenant_id = 2; string run_name = 3; string entity_id = 4;
    string period_type = 5; string fiscal_year = 6; int64 book_income_cents = 7;
    int64 tax_income_cents = 8; int64 current_tax_expense_cents = 9;
    int64 deferred_tax_expense_cents = 10; int64 total_tax_expense_cents = 11;
    double effective_rate_real = 12; string status = 13; string started_at = 14;
    string completed_at = 15; string approved_by = 16;
    string created_at = 17; string updated_at = 18; string created_by = 19; string updated_by = 20;
    int32 version = 21; bool is_active = 22;
}

// --- Deferred Tax Asset ---
message DeferredTaxAsset {
    string id = 1; string tenant_id = 2; string entity_id = 3; string provision_run_id = 4;
    string dta_category = 5; string description = 6; int64 beginning_balance_cents = 7;
    int64 current_period_change_cents = 8; int64 ending_balance_cents = 9;
    double tax_rate_real = 10; int64 gross_dta_cents = 11; int64 valuation_allowance_cents = 12;
    int64 net_dta_cents = 13; string expiration_date = 14; int32 carryforward_years = 15;
    string created_at = 16; string updated_at = 17; string created_by = 18; string updated_by = 19;
    int32 version = 20;
}

// --- Deferred Tax Liability ---
message DeferredTaxLiability {
    string id = 1; string tenant_id = 2; string entity_id = 3; string provision_run_id = 4;
    string dtl_category = 5; string description = 6; int64 beginning_balance_cents = 7;
    int64 current_period_change_cents = 8; int64 ending_balance_cents = 9;
    double tax_rate_real = 10; int64 gross_dtl_cents = 11; string reversal_period = 12;
    string created_at = 13; string updated_at = 14; string created_by = 15; string updated_by = 16;
    int32 version = 17;
}

// --- Tax Rate Table ---
message TaxRateTable {
    string id = 1; string tenant_id = 2; string jurisdiction_id = 3; string rate_type = 4;
    double rate_value_real = 5; int64 income_bracket_min_cents = 6;
    int64 income_bracket_max_cents = 7; string effective_from = 8; string effective_to = 9;
    string description = 10;
    string created_at = 11; string updated_at = 12; string created_by = 13; string updated_by = 14;
    int32 version = 15; bool is_active = 16;
}

// --- Tax Adjustment ---
message TaxAdjustment {
    string id = 1; string tenant_id = 2; string entity_id = 3; string provision_run_id = 4;
    string adjustment_type = 5; string adjustment_category = 6; string description = 7;
    int64 book_amount_cents = 8; int64 tax_amount_cents = 9; int64 difference_cents = 10;
    bool is_recurring = 11; string gl_account_id = 12; string reference = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- Tax Credit ---
message TaxCredit {
    string id = 1; string tenant_id = 2; string entity_id = 3; string provision_run_id = 4;
    string credit_type = 5; string credit_name = 6; int64 credit_amount_cents = 7;
    int64 utilized_amount_cents = 8; int64 carryforward_amount_cents = 9;
    int64 carryback_amount_cents = 10; string expiration_date = 11;
    string source_jurisdiction_id = 12; string supporting_document_id = 13; string status = 14;
    string created_at = 15; string updated_at = 16; string created_by = 17; string updated_by = 18;
    int32 version = 19; bool is_active = 20;
}

// --- Tax Audit Tracking ---
message TaxAuditTracking {
    string id = 1; string tenant_id = 2; string entity_id = 3; string jurisdiction_id = 4;
    string audit_type = 5; string tax_years = 6; string status = 7;
    string examining_agent = 8; string assigned_counsel = 9;
    int64 proposed_adjustment_cents = 10; int64 potential_penalty_cents = 11;
    int64 potential_interest_cents = 12; int64 resolution_amount_cents = 13;
    string open_date = 14; string close_date = 15; string notes = 16;
    string created_at = 17; string updated_at = 18; string created_by = 19; string updated_by = 20;
    int32 version = 21; bool is_active = 22;
}

// --- Tax Return Data ---
message TaxReturnData {
    string id = 1; string tenant_id = 2; string entity_id = 3; string jurisdiction_id = 4;
    string fiscal_year = 5; string return_type = 6; int64 taxable_income_cents = 7;
    int64 tax_liability_cents = 8; int64 estimated_payments_cents = 9;
    int64 balance_due_cents = 10; int64 refund_amount_cents = 11;
    string filing_date = 12; string filing_status = 13; string e_file_confirmation = 14;
    string preparer_id = 15; string reviewer_id = 16;
    string created_at = 17; string updated_at = 18; string created_by = 19; string updated_by = 20;
    int32 version = 21; bool is_active = 22;
}

// --- RPC Request/Response Messages ---
message RunProvisionRequest { string tenant_id = 1; string entity_id = 2; string period_type = 3; string fiscal_year = 4; string run_name = 5; }
message RunProvisionResponse { ProvisionRun data = 1; }

message CalculateDeferredTaxRequest { string tenant_id = 1; string entity_id = 2; string provision_run_id = 3; }
message CalculateDeferredTaxResponse { repeated DeferredTaxAsset assets = 1; repeated DeferredTaxLiability liabilities = 2; }

message GetProvisionSummaryRequest { string tenant_id = 1; string provision_run_id = 2; }
message GetProvisionSummaryResponse { ProvisionRun provision = 1; repeated TaxAdjustment adjustments = 2; repeated TaxCredit credits = 3; }

message GetEffectiveRateReconciliationRequest { string tenant_id = 1; string entity_id = 2; string fiscal_year = 3; string period_type = 4; }
message GetEffectiveRateReconciliationResponse { ProvisionRun provision = 1; repeated TaxAdjustment reconciling_items = 2; }

message GetDeferredTaxRollforwardRequest { string tenant_id = 1; string entity_id = 2; string fiscal_year = 3; }
message GetDeferredTaxRollforwardResponse { repeated DeferredTaxAsset assets = 1; repeated DeferredTaxLiability liabilities = 2; }

message PrepareReturnDataRequest { string tenant_id = 1; string entity_id = 2; string jurisdiction_id = 3; string fiscal_year = 4; string return_type = 5; }
message PrepareReturnDataResponse { TaxReturnData data = 1; }

message ListProvisionRunsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListProvisionRunsResponse { repeated ProvisionRun items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListTaxJurisdictionsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListTaxJurisdictionsResponse { repeated TaxJurisdiction items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListTaxEntitiesRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListTaxEntitiesResponse { repeated TaxEntity items = 1; int32 total_count = 2; string next_page_token = 3; }
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **GL:** Trial balance data, book income amounts, journal entries for provision calculations
- **TAX:** Tax transaction details, tax amounts from invoices and payments
- **EPM:** Tax planning scenarios, budgeted tax rates, forecasted provisions
- **Auth:** User identity for preparer, reviewer, and approver assignment
- **Workflow:** Approval routing for provision reviews and return filing authorization

### 6.2 Data Published To
- **GL:** Provision journal entries (current and deferred tax expense), true-up adjustments
- **EPM:** Actual tax expense data for planning variance analysis
- **Reporting:** Provision summaries, effective tax rate analysis, deferred tax rollforward data
- **Workflow:** Provision review and approval tasks, return filing workflows
- **Notification:** Provision completion alerts, audit status change notifications, filing deadline reminders

---

## 7. Migrations

1. V001: `taxreport_jurisdictions`
2. V002: `taxreport_entities`
3. V003: `taxreport_provision_runs`
4. V004: `taxreport_deferred_tax_assets`
5. V005: `taxreport_deferred_tax_liabilities`
6. V006: `taxreport_rate_tables`
7. V007: `taxreport_adjustments`
8. V008: `taxreport_credits`
9. V009: `taxreport_audit_tracking`
10. V010: `taxreport_return_data`
11. V011: Triggers for `updated_at`
