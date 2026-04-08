# 23 - Tax Management Service Specification

## 1. Domain Overview

Tax Management (TAX) manages tax registrations, tax determination, tax calculation, withholding tax, tax reporting, and tax filing. It serves as the central tax engine for the ERP, providing automatic tax calculation for AP invoices (purchase/recoverable tax), AR invoices (sales/collectible tax), and payment withholding tax. It integrates with GL for tax journal entries, AP for purchase tax, AR for sales tax, and provides comprehensive tax reporting for statutory filing.

**Bounded Context:** Tax Management & Compliance
**Service Name:** `tax-service`
**Database:** `data/tax.db`
**HTTP Port:** 8015 | **gRPC Port:** 9015

---

## 2. Database Schema

### 2.1 Tax Registrations
```sql
CREATE TABLE tax_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('LEGAL_ENTITY','BUSINESS_UNIT','SUPPLIER','CUSTOMER')),
    entity_id TEXT NOT NULL,
    tax_authority TEXT NOT NULL,              -- e.g., "IRS", "HMRC", "GST_COUNCIL"
    registration_number TEXT NOT NULL,        -- Tax ID / VAT number / GST number
    registration_number_encrypted TEXT,       -- AES-256-GCM encrypted
    tax_type TEXT NOT NULL
        CHECK(tax_type IN ('VAT','GST','SALES_TAX','USE_TAX','WITHHOLDING','EXCISE','CUSTOMS')),
    country_code TEXT NOT NULL,               -- ISO 3166-1 alpha-2
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','DEREGISTERED','PENDING')),
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, entity_type, entity_id, tax_type, country_code)
);

CREATE INDEX idx_tax_registrations_tenant_entity ON tax_registrations(tenant_id, entity_type, entity_id);
CREATE INDEX idx_tax_registrations_tenant_country ON tax_registrations(tenant_id, country_code);
CREATE INDEX idx_tax_registrations_tenant_status ON tax_registrations(tenant_id, status);
```

### 2.2 Tax Jurisdictions
```sql
CREATE TABLE tax_jurisdictions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    jurisdiction_code TEXT NOT NULL,          -- e.g., "US-CA", "EU-DE", "IN-KA"
    jurisdiction_name TEXT NOT NULL,
    jurisdiction_level TEXT NOT NULL
        CHECK(jurisdiction_level IN ('FEDERAL','STATE','COUNTY','CITY','REGIONAL','NATIONAL')),
    country_code TEXT NOT NULL,
    state_code TEXT,                          -- State/province
    county TEXT,
    city TEXT,
    description TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, jurisdiction_code)
);

CREATE INDEX idx_tax_jurisdictions_tenant_country ON tax_jurisdictions(tenant_id, country_code);
CREATE INDEX idx_tax_jurisdictions_tenant_level ON tax_jurisdictions(tenant_id, jurisdiction_level);
```

### 2.3 Tax Rates
```sql
CREATE TABLE tax_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    tax_rate_code TEXT NOT NULL,             -- e.g., "CA-STATE-SALES", "DE-VAT-STANDARD"
    tax_rate_name TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    tax_type TEXT NOT NULL
        CHECK(tax_type IN ('VAT','GST','SALES_TAX','USE_TAX','EXCISE','CUSTOMS')),
    rate_type TEXT NOT NULL
        CHECK(rate_type IN ('PERCENTAGE','FLAT','COMPOUND')),
    rate_value REAL NOT NULL,                -- e.g., 8.25 for 8.25%, or flat amount in cents
    rate_value_cents INTEGER,                -- For FLAT rate type, amount in cents
    compound_base_tax_rate_id TEXT,          -- For COMPOUND: the tax rate this is compounded on
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    currency_code TEXT DEFAULT 'USD',
    description TEXT,
    is_recoverable INTEGER NOT NULL DEFAULT 0,  -- Whether input tax can be recovered
    recoverable_percent REAL DEFAULT 0,      -- Percentage of tax recoverable (0-100)
    gl_tax_account_id TEXT,                  -- GL account for tax payable/receivable
    gl_recovery_account_id TEXT,             -- GL account for recoverable tax
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','EXPIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jurisdiction_id) REFERENCES tax_jurisdictions(id) ON DELETE RESTRICT,
    FOREIGN KEY (compound_base_tax_rate_id) REFERENCES tax_rates(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, tax_rate_code)
);

CREATE INDEX idx_tax_rates_tenant_jurisdiction ON tax_rates(tenant_id, jurisdiction_id);
CREATE INDEX idx_tax_rates_tenant_type ON tax_rates(tenant_id, tax_type);
CREATE INDEX idx_tax_rates_tenant_status ON tax_rates(tenant_id, status);
CREATE INDEX idx_tax_rates_tenant_dates ON tax_rates(tenant_id, effective_from, effective_to);
```

### 2.4 Tax Codes
```sql
CREATE TABLE tax_codes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    tax_code TEXT NOT NULL,                  -- e.g., "STANDARD", "REDUCED", "ZERO", "EXEMPT"
    tax_code_name TEXT NOT NULL,
    description TEXT,
    tax_rate_id TEXT NOT NULL,               -- Default tax rate for this code
    tax_category TEXT NOT NULL
        CHECK(tax_category IN ('STANDARD','REDUCED','ZERO_RATED','EXEMPT','REVERSE_CHARGE','DEFERRED')),
    applicable_to TEXT NOT NULL DEFAULT 'BOTH'
        CHECK(applicable_to IN ('SALES','PURCHASES','BOTH')),
    is_default INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (tax_rate_id) REFERENCES tax_rates(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, tax_code)
);

CREATE INDEX idx_tax_codes_tenant_category ON tax_codes(tenant_id, tax_category);
CREATE INDEX idx_tax_codes_tenant_applicable ON tax_codes(tenant_id, applicable_to);
```

### 2.5 Tax Determinants
```sql
CREATE TABLE tax_determinants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_priority INTEGER NOT NULL DEFAULT 100,  -- Lower number = higher priority
    determinant_type TEXT NOT NULL
        CHECK(determinant_type IN ('SHIP_FROM','SHIP_TO','ORIGIN','DESTINATION','PRODUCT_TYPE','CUSTOMER_TYPE','SUPPLIER_TYPE','AMOUNT_THRESHOLD')),
    ship_from_country TEXT,
    ship_from_state TEXT,
    ship_to_country TEXT,
    ship_to_state TEXT,
    product_type TEXT,                       -- 'GOODS', 'SERVICES', 'DIGITAL', 'EXEMPT_GOODS'
    customer_type TEXT,                      -- 'B2B', 'B2C', 'GOVERNMENT', 'EXEMPT'
    supplier_type TEXT,                      -- 'DOMESTIC', 'FOREIGN', 'EXEMPT'
    min_amount_cents INTEGER,                -- Minimum amount threshold
    max_amount_cents INTEGER,                -- Maximum amount threshold
    tax_code_id TEXT NOT NULL,               -- Resulting tax code
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (tax_code_id) REFERENCES tax_codes(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_tax_determinants_tenant_type ON tax_determinants(tenant_id, determinant_type);
CREATE INDEX idx_tax_determinants_tenant_priority ON tax_determinants(tenant_id, rule_priority);
CREATE INDEX idx_tax_determinants_tenant_status ON tax_determinants(tenant_id, status);
```

### 2.6 Tax Transactions
```sql
CREATE TABLE tax_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,        -- TAX-TXN-2024-00001
    source_type TEXT NOT NULL
        CHECK(source_type IN ('AP_INVOICE','AP_PAYMENT','AR_INVOICE','AR_RECEIPT','AR_CREDIT_MEMO','MANUAL')),
    source_id TEXT NOT NULL,                 -- ID of the invoice/payment
    source_line_id TEXT,                     -- ID of the specific line
    transaction_date TEXT NOT NULL,
    tax_date TEXT NOT NULL,                  -- Tax point date
    period_name TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,

    -- Parties
    entity_type TEXT,                        -- 'SUPPLIER' or 'CUSTOMER'
    entity_id TEXT,
    ship_from_country TEXT,
    ship_from_state TEXT,
    ship_to_country TEXT,
    ship_to_state TEXT,

    -- Tax determination
    tax_code_id TEXT NOT NULL,
    tax_rate_id TEXT NOT NULL,
    tax_jurisdiction_id TEXT NOT NULL,

    -- Amounts
    taxable_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_rate_percent REAL NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code_tax TEXT NOT NULL DEFAULT 'USD',

    -- Recovery (for purchase tax)
    is_recoverable INTEGER NOT NULL DEFAULT 0,
    recoverable_amount_cents INTEGER NOT NULL DEFAULT 0,
    non_recoverable_amount_cents INTEGER NOT NULL DEFAULT 0,
    recovery_percent REAL DEFAULT 0,

    -- Withholding
    is_withholding INTEGER NOT NULL DEFAULT 0,

    -- GL integration
    gl_journal_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('DRAFT','ACTIVE','REVERSED','ADJUSTED')),

    -- Reversal
    reversal_of_id TEXT,
    reversal_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (tax_code_id) REFERENCES tax_codes(id) ON DELETE RESTRICT,
    FOREIGN KEY (tax_rate_id) REFERENCES tax_rates(id) ON DELETE RESTRICT,
    FOREIGN KEY (tax_jurisdiction_id) REFERENCES tax_jurisdictions(id) ON DELETE RESTRICT,
    FOREIGN KEY (reversal_of_id) REFERENCES tax_transactions(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_tax_transactions_tenant_source ON tax_transactions(tenant_id, source_type, source_id);
CREATE INDEX idx_tax_transactions_tenant_date ON tax_transactions(tenant_id, transaction_date);
CREATE INDEX idx_tax_transactions_tenant_period ON tax_transactions(tenant_id, period_name);
CREATE INDEX idx_tax_transactions_tenant_entity ON tax_transactions(tenant_id, entity_type, entity_id);
CREATE INDEX idx_tax_transactions_tenant_status ON tax_transactions(tenant_id, status);
CREATE INDEX idx_tax_transactions_tenant_jurisdiction ON tax_transactions(tenant_id, tax_jurisdiction_id);
CREATE INDEX idx_tax_transactions_tenant_journal ON tax_transactions(tenant_id, gl_journal_id);
```

### 2.7 Tax Reporting Periods
```sql
CREATE TABLE tax_reporting_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,               -- "VAT-Q1-2024", "SALES-TAX-JAN-2024"
    period_type TEXT NOT NULL
        CHECK(period_type IN ('MONTHLY','QUARTERLY','SEMI_ANNUAL','ANNUAL')),
    tax_type TEXT NOT NULL
        CHECK(tax_type IN ('VAT','GST','SALES_TAX','USE_TAX','WITHHOLDING','EXCISE')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    filing_due_date TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','CLOSED','FILED')),
    closed_date TEXT,
    closed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (jurisdiction_id) REFERENCES tax_jurisdictions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, period_name)
);

CREATE INDEX idx_tax_reporting_tenant_dates ON tax_reporting_periods(tenant_id, start_date, end_date);
CREATE INDEX idx_tax_reporting_tenant_status ON tax_reporting_periods(tenant_id, status);
CREATE INDEX idx_tax_reporting_tenant_jurisdiction ON tax_reporting_periods(tenant_id, jurisdiction_id);
```

### 2.8 Tax Returns
```sql
CREATE TABLE tax_returns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    return_number TEXT NOT NULL,             -- TAX-RET-2024-00001
    reporting_period_id TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    tax_type TEXT NOT NULL,
    return_type TEXT NOT NULL
        CHECK(return_type IN ('VAT_RETURN','GST_RETURN','SALES_TAX_RETURN','USE_TAX_RETURN','WITHHOLDING_RETURN','EXCISE_RETURN')),
    filing_date TEXT,
    period_start_date TEXT NOT NULL,
    period_end_date TEXT NOT NULL,

    -- Tax collected / output tax (sales)
    total_output_tax_cents INTEGER NOT NULL DEFAULT 0,
    taxable_sales_cents INTEGER NOT NULL DEFAULT 0,
    exempt_sales_cents INTEGER NOT NULL DEFAULT 0,
    zero_rated_sales_cents INTEGER NOT NULL DEFAULT 0,

    -- Tax paid / input tax (purchases)
    total_input_tax_cents INTEGER NOT NULL DEFAULT 0,
    taxable_purchases_cents INTEGER NOT NULL DEFAULT 0,
    exempt_purchases_cents INTEGER NOT NULL DEFAULT 0,

    -- Adjustments
    adjustments_cents INTEGER NOT NULL DEFAULT 0,
    adjustment_reason TEXT,

    -- Net tax
    tax_payable_cents INTEGER NOT NULL DEFAULT 0,     -- Output - Input (if positive)
    tax_refund_cents INTEGER NOT NULL DEFAULT 0,      -- Input - Output (if positive)

    -- Withholding totals
    total_withheld_cents INTEGER NOT NULL DEFAULT 0,

    -- Filing
    filing_method TEXT DEFAULT 'ELECTRONIC'
        CHECK(filing_method IN ('ELECTRONIC','PAPER','AGENT')),
    filing_reference TEXT,                   -- Confirmation number from tax authority
    filed_by TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','FILED','ACCEPTED','REJECTED','AMENDED')),
    rejection_reason TEXT,
    notes TEXT,

    -- GL integration
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (reporting_period_id) REFERENCES tax_reporting_periods(id) ON DELETE RESTRICT,
    FOREIGN KEY (jurisdiction_id) REFERENCES tax_jurisdictions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, return_number)
);

CREATE INDEX idx_tax_returns_tenant_period ON tax_returns(tenant_id, reporting_period_id);
CREATE INDEX idx_tax_returns_tenant_status ON tax_returns(tenant_id, status);
CREATE INDEX idx_tax_returns_tenant_jurisdiction ON tax_returns(tenant_id, jurisdiction_id);
CREATE INDEX idx_tax_returns_tenant_type ON tax_returns(tenant_id, tax_type);
```

### 2.9 Withholding Tax Codes
```sql
CREATE TABLE withholding_tax_codes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    wht_code TEXT NOT NULL,                  -- e.g., "WHT-10", "WHT-20-RENT"
    wht_name TEXT NOT NULL,
    description TEXT,
    wht_type TEXT NOT NULL
        CHECK(wht_type IN ('STANDARD','REDUCED','TREATY','EXEMPT')),
    rate_percent REAL NOT NULL,              -- Withholding rate as percentage
    threshold_cents INTEGER,                 -- Minimum amount before WHT applies
    cumulative_threshold_cents INTEGER,      -- Annual cumulative threshold
    tax_authority TEXT NOT NULL,
    country_code TEXT NOT NULL,
    gl_account_id TEXT NOT NULL,             -- GL account for WHT payable
    applicable_to TEXT NOT NULL DEFAULT 'BOTH'
        CHECK(applicable_to IN ('SERVICES','RENT','ROYALTIES','DIVIDENDS','INTEREST','CONTRACTS','BOTH')),
    certificate_required INTEGER NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, wht_code)
);

CREATE INDEX idx_wht_codes_tenant_country ON withholding_tax_codes(tenant_id, country_code);
CREATE INDEX idx_wht_codes_tenant_status ON withholding_tax_codes(tenant_id, status);
```

### 2.10 Tax Exemptions
```sql
CREATE TABLE tax_exemptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    exemption_code TEXT NOT NULL,            -- e.g., "EXEMPT-RESALE-001"
    exemption_type TEXT NOT NULL
        CHECK(exemption_type IN ('ENTITY','PRODUCT','TRANSACTION','CERTIFICATE')),
    entity_type TEXT,                        -- 'SUPPLIER', 'CUSTOMER'
    entity_id TEXT,
    product_type TEXT,                       -- 'GOODS', 'SERVICES', 'DIGITAL'
    tax_rate_id TEXT,                        -- Specific tax rate exempted (optional)
    jurisdiction_id TEXT,                    -- Specific jurisdiction (optional)
    certificate_number TEXT,                 -- Exemption certificate number
    certificate_expiry_date TEXT,
    reason TEXT NOT NULL,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','REVOKED','PENDING')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (tax_rate_id) REFERENCES tax_rates(id) ON DELETE SET NULL,
    FOREIGN KEY (jurisdiction_id) REFERENCES tax_jurisdictions(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, exemption_code)
);

CREATE INDEX idx_tax_exemptions_tenant_entity ON tax_exemptions(tenant_id, entity_type, entity_id);
CREATE INDEX idx_tax_exemptions_tenant_status ON tax_exemptions(tenant_id, status);
CREATE INDEX idx_tax_exemptions_tenant_jurisdiction ON tax_exemptions(tenant_id, jurisdiction_id);
```

### 2.11 Tax Reconciliation
```sql
CREATE TABLE tax_reconciliation (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    reconciliation_number TEXT NOT NULL,     -- TAX-RECON-2024-00001
    reporting_period_id TEXT NOT NULL,
    jurisdiction_id TEXT NOT NULL,
    tax_type TEXT NOT NULL,

    -- Tax collected (from tax_transactions for sales)
    tax_collected_cents INTEGER NOT NULL DEFAULT 0,
    tax_collected_count INTEGER NOT NULL DEFAULT 0,

    -- Tax reported (from filed tax returns)
    tax_reported_cents INTEGER NOT NULL DEFAULT 0,

    -- Tax paid (recoverable input tax)
    tax_paid_cents INTEGER NOT NULL DEFAULT 0,
    tax_paid_count INTEGER NOT NULL DEFAULT 0,

    -- Tax recovered (input tax credit claimed)
    tax_recovered_cents INTEGER NOT NULL DEFAULT 0,

    -- Variance
    variance_cents INTEGER NOT NULL DEFAULT 0,
    variance_reason TEXT,
    adjustments_cents INTEGER NOT NULL DEFAULT 0,
    adjustment_reason TEXT,

    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','RECONCILED','ADJUSTED')),
    reconciled_date TEXT,
    reconciled_by TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (reporting_period_id) REFERENCES tax_reporting_periods(id) ON DELETE RESTRICT,
    FOREIGN KEY (jurisdiction_id) REFERENCES tax_jurisdictions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, reconciliation_number)
);

CREATE INDEX idx_tax_recon_tenant_period ON tax_reconciliation(tenant_id, reporting_period_id);
CREATE INDEX idx_tax_recon_tenant_status ON tax_reconciliation(tenant_id, status);
CREATE INDEX idx_tax_recon_tenant_jurisdiction ON tax_reconciliation(tenant_id, jurisdiction_id);
```

### 2.12 Tax Number Sequences
```sql
CREATE TABLE tax_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_type TEXT NOT NULL,             -- 'TRANSACTION', 'RETURN', 'RECONCILIATION'
    prefix TEXT NOT NULL,                    -- 'TAX-TXN-', 'TAX-RET-', 'TAX-RECON-'
    current_value INTEGER NOT NULL DEFAULT 0,
    fiscal_year INTEGER NOT NULL,
    UNIQUE(tenant_id, sequence_type, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Tax Registrations
```
GET    /api/v1/tax/registrations                       Permission: tax.registrations.read
POST   /api/v1/tax/registrations                       Permission: tax.registrations.create
GET    /api/v1/tax/registrations/{id}                  Permission: tax.registrations.read
PUT    /api/v1/tax/registrations/{id}                  Permission: tax.registrations.update
DELETE /api/v1/tax/registrations/{id}                  Permission: tax.registrations.delete
```

### 3.2 Tax Jurisdictions
```
GET    /api/v1/tax/jurisdictions                       Permission: tax.jurisdictions.read
POST   /api/v1/tax/jurisdictions                       Permission: tax.jurisdictions.create
GET    /api/v1/tax/jurisdictions/{id}                  Permission: tax.jurisdictions.read
PUT    /api/v1/tax/jurisdictions/{id}                  Permission: tax.jurisdictions.update
DELETE /api/v1/tax/jurisdictions/{id}                  Permission: tax.jurisdictions.delete
```

### 3.3 Tax Rates
```
GET    /api/v1/tax/rates                               Permission: tax.rates.read
POST   /api/v1/tax/rates                               Permission: tax.rates.create
GET    /api/v1/tax/rates/{id}                          Permission: tax.rates.read
PUT    /api/v1/tax/rates/{id}                          Permission: tax.rates.update
DELETE /api/v1/tax/rates/{id}                          Permission: tax.rates.delete
GET    /api/v1/tax/rates/{id}/history                  Permission: tax.rates.read
```

### 3.4 Tax Codes
```
GET    /api/v1/tax/codes                               Permission: tax.codes.read
POST   /api/v1/tax/codes                               Permission: tax.codes.create
GET    /api/v1/tax/codes/{id}                          Permission: tax.codes.read
PUT    /api/v1/tax/codes/{id}                          Permission: tax.codes.update
DELETE /api/v1/tax/codes/{id}                          Permission: tax.codes.delete
```

### 3.5 Tax Determinants
```
GET    /api/v1/tax/determinants                        Permission: tax.determinants.read
POST   /api/v1/tax/determinants                        Permission: tax.determinants.create
GET    /api/v1/tax/determinants/{id}                   Permission: tax.determinants.read
PUT    /api/v1/tax/determinants/{id}                   Permission: tax.determinants.update
DELETE /api/v1/tax/determinants/{id}                   Permission: tax.determinants.delete
POST   /api/v1/tax/determinants/reorder                Permission: tax.determinants.update
```

### 3.6 Tax Transactions
```
GET    /api/v1/tax/transactions                        Permission: tax.transactions.read
POST   /api/v1/tax/transactions                        Permission: tax.transactions.create
GET    /api/v1/tax/transactions/{id}                   Permission: tax.transactions.read
POST   /api/v1/tax/transactions/{id}/reverse           Permission: tax.transactions.update
GET    /api/v1/tax/transactions/by-source              Permission: tax.transactions.read
  ?source_type=AP_INVOICE&source_id=xxx
```

### 3.7 Tax Calculation
```
POST   /api/v1/tax/calculate                          Permission: tax.transactions.create
  Request:
  {
    "source_type": "AP_INVOICE",
    "source_id": "0192...",
    "transaction_date": "2024-01-15",
    "lines": [
      {
        "source_line_id": "line-001",
        "amount_cents": 100000,
        "ship_from_country": "US",
        "ship_from_state": "CA",
        "ship_to_country": "US",
        "ship_to_state": "TX",
        "product_type": "GOODS",
        "entity_type": "SUPPLIER",
        "entity_id": "0192...",
        "tax_code_override": null
      }
    ]
  }

  Response 200:
  {
    "data": {
      "total_taxable_amount_cents": 100000,
      "total_tax_amount_cents": 8250,
      "lines": [
        {
          "source_line_id": "line-001",
          "tax_code": "STANDARD",
          "tax_rate_id": "0192...",
          "tax_rate_percent": 8.25,
          "taxable_amount_cents": 100000,
          "tax_amount_cents": 8250,
          "jurisdiction_code": "US-CA",
          "is_recoverable": true,
          "recoverable_amount_cents": 8250,
          "non_recoverable_amount_cents": 0
        }
      ]
    }
  }
```

### 3.8 Withholding Tax Codes
```
GET    /api/v1/tax/withholding-codes                   Permission: tax.withholding.read
POST   /api/v1/tax/withholding-codes                   Permission: tax.withholding.create
GET    /api/v1/tax/withholding-codes/{id}              Permission: tax.withholding.read
PUT    /api/v1/tax/withholding-codes/{id}              Permission: tax.withholding.update
DELETE /api/v1/tax/withholding-codes/{id}              Permission: tax.withholding.delete
POST   /api/v1/tax/withholding-codes/calculate         Permission: tax.withholding.read
```

### 3.9 Tax Exemptions
```
GET    /api/v1/tax/exemptions                          Permission: tax.exemptions.read
POST   /api/v1/tax/exemptions                          Permission: tax.exemptions.create
GET    /api/v1/tax/exemptions/{id}                     Permission: tax.exemptions.read
PUT    /api/v1/tax/exemptions/{id}                     Permission: tax.exemptions.update
DELETE /api/v1/tax/exemptions/{id}                     Permission: tax.exemptions.delete
GET    /api/v1/tax/exemptions/lookup                   Permission: tax.exemptions.read
  ?entity_type=CUSTOMER&entity_id=xxx&jurisdiction_id=xxx
```

### 3.10 Tax Reporting Periods
```
GET    /api/v1/tax/reporting-periods                   Permission: tax.reporting.read
POST   /api/v1/tax/reporting-periods                   Permission: tax.reporting.create
POST   /api/v1/tax/reporting-periods/generate          Permission: tax.reporting.create
GET    /api/v1/tax/reporting-periods/{id}              Permission: tax.reporting.read
PUT    /api/v1/tax/reporting-periods/{id}              Permission: tax.reporting.update
POST   /api/v1/tax/reporting-periods/{id}/close        Permission: tax.reporting.update
```

### 3.11 Tax Returns
```
GET    /api/v1/tax/returns                             Permission: tax.returns.read
POST   /api/v1/tax/returns                             Permission: tax.returns.create
GET    /api/v1/tax/returns/{id}                        Permission: tax.returns.read
PUT    /api/v1/tax/returns/{id}                        Permission: tax.returns.update
POST   /api/v1/tax/returns/{id}/submit                 Permission: tax.returns.update
POST   /api/v1/tax/returns/{id}/file                   Permission: tax.returns.file
POST   /api/v1/tax/returns/{id}/amend                  Permission: tax.returns.update
POST   /api/v1/tax/returns/generate                    Permission: tax.returns.create
  Request:
  {
    "reporting_period_id": "0192...",
    "jurisdiction_id": "0192...",
    "tax_type": "VAT",
    "return_type": "VAT_RETURN"
  }
```

### 3.12 Tax Reconciliation
```
GET    /api/v1/tax/reconciliation                      Permission: tax.reconciliation.read
POST   /api/v1/tax/reconciliation                      Permission: tax.reconciliation.create
GET    /api/v1/tax/reconciliation/{id}                 Permission: tax.reconciliation.read
PUT    /api/v1/tax/reconciliation/{id}                 Permission: tax.reconciliation.update
POST   /api/v1/tax/reconciliation/{id}/reconcile       Permission: tax.reconciliation.update
POST   /api/v1/tax/reconciliation/auto                 Permission: tax.reconciliation.create
```

### 3.13 Reports
```
GET    /api/v1/tax/reports/tax-summary                 Permission: tax.reports.view
  ?filter[period_from]=2024-01-01&filter[period_to]=2024-03-31
  ?filter[jurisdiction_id]=xxx
GET    /api/v1/tax/reports/tax-detail                  Permission: tax.reports.view
GET    /api/v1/tax/reports/tax-liability                Permission: tax.reports.view
GET    /api/v1/tax/reports/recoverable-tax              Permission: tax.reports.view
GET    /api/v1/tax/reports/withholding-summary          Permission: tax.reports.view
GET    /api/v1/tax/reports/tax-jurisdiction-comparison  Permission: tax.reports.view
```

---

## 4. Business Rules

### 4.1 Automatic Tax Determination
When AP or AR submits a line for tax calculation, the system determines the applicable tax:
1. Check for explicit tax code override on the line
2. Check for tax exemptions on the entity (supplier/customer) for the jurisdiction
3. Evaluate tax_determinants rules in priority order (lowest priority number first)
4. Match rules by: ship_from/ship_to country/state, product_type, customer_type, supplier_type
5. First matching rule determines the tax_code
6. Look up the tax_rate for the code effective on the transaction date
7. If no rule matches, apply the default tax code for the jurisdiction

### 4.2 Tax Calculation on AP Invoices (Purchase Tax / Recoverable)
When an AP invoice is posted:
1. For each invoice line, call tax calculation
2. Calculate tax: `tax_amount_cents = ROUND(taxable_amount_cents * rate_percent / 100)`
3. Determine recoverability based on `tax_rates.is_recoverable` and `recoverable_percent`
4. `recoverable_amount_cents = ROUND(tax_amount_cents * recoverable_percent / 100)`
5. `non_recoverable_amount_cents = tax_amount_cents - recoverable_amount_cents`
6. Create tax_transactions records for each line
7. GL journal entries:
   - Tax payable: Debit Tax Expense (or Tax Recoverable), Credit Tax Payable
   - Non-recoverable portion: added to the expense line cost

```
For each tax line (recoverable):
  Debit: Tax Recoverable (asset account) → recoverable_amount_cents
  Debit: Expense account → non_recoverable_amount_cents
  Credit: Tax Payable (liability account) → tax_amount_cents
```

### 4.3 Tax Calculation on AR Invoices (Sales Tax / Collectible)
When an AR invoice is posted:
1. For each invoice line, call tax calculation
2. Calculate tax: `tax_amount_cents = ROUND(taxable_amount_cents * rate_percent / 100)`
3. Create tax_transactions records for each line
4. GL journal entries:
```
For each tax line:
  Debit: Tax Expense (or AR if unpaid) → tax_amount_cents
  Credit: Tax Payable (liability account) → tax_amount_cents

Invoice posting summary:
  Debit: Accounts Receivable → subtotal + tax_amount
  Credit: Revenue → subtotal
  Credit: Tax Payable → tax_amount
```

### 4.4 Withholding Tax on AP Payments
When an AP payment is made:
1. Check supplier withholding_tax_codes configuration
2. Calculate withholding: `wht_amount_cents = ROUND(payment_amount_cents * wht_rate_percent / 100)`
3. Check threshold: only apply if `payment_amount_cents >= threshold_cents`
4. Reduce payment amount to supplier by WHT amount
5. Create tax_transaction with `is_withholding = 1`
6. GL journal entries:
```
Debit: Accounts Payable → full invoice amount
  Credit: Cash/Bank → (invoice_amount - wht_amount)
  Credit: WHT Payable (liability) → wht_amount
```

### 4.5 Reverse Charge Mechanism
When the reverse charge applies (determined by tax_determinants rule):
1. Both the output tax and input tax are recorded
2. They net to zero in the tax return, so no cash payment to tax authority
3. GL journal entries:
```
Debit: Tax Recoverable → reverse_charge_amount
  Credit: Tax Payable → reverse_charge_amount
```
4. On the tax return, the reverse charge appears in both output and input columns
5. Applicable for: cross-border B2B services, imports, specific domestic transactions

### 4.6 Tax Reporting and Filing
1. Tax reporting periods are generated based on jurisdiction filing frequency
2. Tax return generation:
   - Sum all tax_transactions for the period, grouped by jurisdiction and tax_type
   - `total_output_tax_cents = SUM(tax_amount) WHERE source_type IN ('AR_INVOICE','AR_CREDIT_MEMO')`
   - `total_input_tax_cents = SUM(tax_amount) WHERE source_type = 'AP_INVOICE'`
   - `tax_payable_cents = MAX(0, total_output_tax_cents - total_input_tax_cents)`
   - `tax_refund_cents = MAX(0, total_input_tax_cents - total_output_tax_cents)`
3. Tax return status flow:
```
DRAFT → SUBMITTED → FILED → ACCEPTED
                                    ↓
                              REJECTED (back to DRAFT)
                              AMENDED (creates amendment)
```
4. Filing triggers GL journal for tax payment:
```
Debit: Tax Payable → tax_payable_cents
  Credit: Cash/Bank → tax_payable_cents
```

### 4.7 Tax Recovery (Input Tax Credit)
1. Input tax on purchases can be recovered (claimed as credit against output tax)
2. Recovery eligibility determined by:
   - `tax_rates.is_recoverable = 1`
   - Business purpose (not for exempt supplies)
   - Within filing deadline
3. Recovery process:
   - Input tax credit claimed on the tax return for the period
   - Reduces net tax payable to the authority
   - GL journal for recovery:
```
Debit: Tax Recoverable (asset) → recoverable_amount
  Credit: Tax Expense → recoverable_amount
```

### 4.8 Compound Tax Calculation (Tax on Tax)
When `rate_type = 'COMPOUND'`:
1. First calculate the base tax: `base_tax = taxable_amount * base_rate / 100`
2. Then calculate compound tax: `compound_tax = (taxable_amount + base_tax) * compound_rate / 100`
3. Total tax = base_tax + compound_tax
4. Example: Provincial sales tax compounded on federal GST
   - GST 5% on $100 = $5.00
   - PST 7% on ($100 + $5) = $7.35
   - Total tax = $12.35
5. The `compound_base_tax_rate_id` links to the base tax rate

### 4.9 Tax Exemption Handling
1. Entity exemptions: Check tax_exemptions for the entity in the jurisdiction
   - If active exemption found, tax_amount = 0
   - Certificate number and expiry date tracked
   - Expired certificates generate alerts
2. Product exemptions: Check for product_type exemptions
   - Certain goods (food, medicine, education) may be exempt
   - Zero-rated: tax calculated at 0% (still reported, allows input recovery)
   - Exempt: no tax calculated at all (no input recovery on related purchases)
3. Transaction exemptions: Based on specific transaction characteristics
4. Exempt transactions are still recorded in tax_transactions with tax_amount_cents = 0

### 4.10 Tax Date Rules
- Tax date defaults to the transaction date (invoice date)
- For goods: tax date = delivery/shipment date (if different from invoice date)
- For services: tax date = date services are completed
- For prepayments: tax date = payment receipt date
- Tax date determines which tax rate is applicable (based on effective dates)

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.tax.v1;

service TaxManagementService {
    // Tax calculation (called by AP, AR)
    rpc CalculateTax(CalculateTaxRequest) returns (CalculateTaxResponse);
    rpc CalculateWithholdingTax(CalculateWithholdingTaxRequest) returns (CalculateWithholdingTaxResponse);

    // Tax transaction management
    rpc GetTaxTransaction(GetTaxTransactionRequest) returns (GetTaxTransactionResponse);
    rpc GetTaxTransactionsBySource(GetTaxTransactionsBySourceRequest) returns (GetTaxTransactionsBySourceResponse);
    rpc ReverseTaxTransaction(ReverseTaxTransactionRequest) returns (ReverseTaxTransactionResponse);

    // Tax registration lookup
    rpc GetTaxRegistration(GetTaxRegistrationRequest) returns (GetTaxRegistrationResponse);

    // Tax rate lookup
    rpc GetEffectiveTaxRate(GetEffectiveTaxRateRequest) returns (GetEffectiveTaxRateResponse);

    // Tax reporting
    rpc GenerateTaxReturn(GenerateTaxReturnRequest) returns (GenerateTaxReturnResponse);
    rpc GetTaxSummary(GetTaxSummaryRequest) returns (GetTaxSummaryResponse);

    // Exemption check
    rpc CheckExemption(CheckExemptionRequest) returns (CheckExemptionResponse);
}

message TaxLineRequest {
    string source_line_id = 1;
    int64 amount_cents = 2;
    string ship_from_country = 3;
    string ship_from_state = 4;
    string ship_to_country = 5;
    string ship_to_state = 6;
    string product_type = 7;
    string entity_type = 8;
    string entity_id = 9;
    string tax_code_override = 10;
}

message TaxLineResult {
    string source_line_id = 1;
    string tax_code = 2;
    string tax_rate_id = 3;
    double tax_rate_percent = 4;
    int64 taxable_amount_cents = 5;
    int64 tax_amount_cents = 6;
    string jurisdiction_code = 7;
    bool is_recoverable = 8;
    int64 recoverable_amount_cents = 9;
    int64 non_recoverable_amount_cents = 10;
}

message CalculateTaxRequest {
    string tenant_id = 1;
    string source_type = 2;
    string source_id = 3;
    string transaction_date = 4;
    string currency_code = 5;
    repeated TaxLineRequest lines = 6;
}

message CalculateTaxResponse {
    int64 total_taxable_amount_cents = 1;
    int64 total_tax_amount_cents = 2;
    repeated TaxLineResult lines = 3;
}

message CalculateWithholdingTaxRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    int64 payment_amount_cents = 3;
    string payment_date = 4;
    string country_code = 5;
    string applicable_type = 6;
}

message CalculateWithholdingTaxResponse {
    string wht_code = 1;
    double wht_rate_percent = 2;
    int64 wht_amount_cents = 3;
    int64 net_payment_cents = 4;
}

message GetTaxTransactionRequest {
    string tenant_id = 1;
    string transaction_id = 2;
}

message GetTaxTransactionResponse {
    string id = 1;
    string transaction_number = 2;
    string source_type = 3;
    string source_id = 4;
    int64 taxable_amount_cents = 5;
    int64 tax_amount_cents = 6;
    string tax_code = 7;
    double tax_rate_percent = 8;
}

message GetTaxTransactionsBySourceRequest {
    string tenant_id = 1;
    string source_type = 2;
    string source_id = 3;
}

message GetTaxTransactionsBySourceResponse {
    repeated GetTaxTransactionResponse transactions = 1;
}

message ReverseTaxTransactionRequest {
    string tenant_id = 1;
    string transaction_id = 2;
    string reason = 3;
}

message ReverseTaxTransactionResponse {
    string reversal_transaction_id = 1;
    string transaction_number = 2;
}

message GetTaxRegistrationRequest {
    string tenant_id = 1;
    string entity_type = 2;
    string entity_id = 3;
    string tax_type = 4;
    string country_code = 5;
}

message GetTaxRegistrationResponse {
    string id = 1;
    string registration_number = 2;
    string tax_authority = 3;
    string status = 4;
}

message GetEffectiveTaxRateRequest {
    string tenant_id = 1;
    string tax_rate_code = 2;
    string date = 3;
}

message GetEffectiveTaxRateResponse {
    string tax_rate_id = 1;
    double rate_value = 2;
    string rate_type = 3;
    bool is_recoverable = 4;
    double recoverable_percent = 5;
}

message GenerateTaxReturnRequest {
    string tenant_id = 1;
    string reporting_period_id = 2;
    string jurisdiction_id = 3;
    string tax_type = 4;
    string return_type = 5;
}

message GenerateTaxReturnResponse {
    string return_id = 1;
    string return_number = 2;
    int64 total_output_tax_cents = 3;
    int64 total_input_tax_cents = 4;
    int64 tax_payable_cents = 5;
    int64 tax_refund_cents = 6;
}

message GetTaxSummaryRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
    string jurisdiction_id = 4;
    string tax_type = 5;
}

message GetTaxSummaryResponse {
    int64 total_output_tax_cents = 1;
    int64 total_input_tax_cents = 2;
    int64 tax_payable_cents = 3;
    int64 tax_refund_cents = 4;
    int32 transaction_count = 5;
}

message CheckExemptionRequest {
    string tenant_id = 1;
    string entity_type = 2;
    string entity_id = 3;
    string jurisdiction_id = 4;
    string product_type = 5;
}

message CheckExemptionResponse {
    bool is_exempt = 1;
    string exemption_code = 2;
    string reason = 3;
    string certificate_number = 4;
    string certificate_expiry = 5;
}
```

---

## 6. Inter-Service Integration

### 6.1 AP calls Tax for Invoice Tax Calculation
When AP creates or posts an invoice:
```
AP → tax.CalculateTax(source_type="AP_INVOICE", source_id=invoice_id, lines=[...])
Tax → returns tax amounts per line
AP → stores tax_amount on invoice lines, creates tax_transactions
AP → posts to GL including tax journal entries
```

### 6.2 AR calls Tax for Sales Tax
When AR creates or posts an invoice:
```
AR → tax.CalculateTax(source_type="AR_INVOICE", source_id=invoice_id, lines=[...])
Tax → returns tax amounts per line
AR → stores tax_amount on invoice lines, creates tax_transactions
AR → posts to GL including tax journal entries
```

### 6.3 AP calls Tax for Withholding Tax on Payments
When AP processes a payment:
```
AP → tax.CalculateWithholdingTax(supplier_id, payment_amount, ...)
Tax → returns WHT amount
AP → reduces payment to supplier, routes WHT to liability account
```

### 6.4 GL Integration for Tax Journal Entries
Tax service calls GL gRPC to create tax-related journal entries:

**Tax payable journal (on AP invoice with purchase tax):**
```
Debit: Tax Recoverable (asset) → recoverable_amount_cents
Debit: Expense account → non_recoverable_amount_cents
Credit: Tax Payable (liability) → tax_amount_cents
```

**Tax payable journal (on AR invoice with sales tax):**
```
Debit: Tax Expense or Accounts Receivable → tax_amount_cents
Credit: Tax Payable (liability) → tax_amount_cents
```

**Tax recovery journal (input tax credit claimed):**
```
Debit: Tax Recoverable (asset) → recoverable_amount_cents
Credit: Tax Expense → recoverable_amount_cents
```

**Tax payment journal (tax return filed and tax paid):**
```
Debit: Tax Payable (liability) → tax_payable_cents
Credit: Cash/Bank → tax_payable_cents
```

**Withholding tax journal (on AP payment):**
```
Debit: Accounts Payable → full_amount
  Credit: Cash/Bank → (full_amount - wht_amount)
  Credit: WHT Payable → wht_amount
```

**Reverse charge journal:**
```
Debit: Tax Recoverable → reverse_charge_amount
Credit: Tax Payable → reverse_charge_amount
```

### 6.5 Event Publishing
Tax service publishes these events for other services to consume:

| Event | Trigger | Payload |
|-------|---------|---------|
| `tax.transaction.created` | Tax transaction calculated | transaction_id, source_type, source_id, tax_amount_cents, tenant_id |
| `tax.transaction.reversed` | Tax transaction reversed | transaction_id, reversal_id, reason |
| `tax.return.generated` | Tax return generated | return_id, period, jurisdiction, tax_payable_cents |
| `tax.return.filed` | Tax return filed with authority | return_id, filing_reference, tax_type |
| `tax.return.accepted` | Tax return accepted by authority | return_id, acceptance_date |
| `tax.return.rejected` | Tax return rejected | return_id, rejection_reason |
| `tax.period.closed` | Tax reporting period closed | period_id, period_name, jurisdiction_id |
| `tax.reconciliation.completed` | Tax reconciliation completed | reconciliation_id, variance_cents |
| `tax.exemption.expiring` | Tax exemption certificate nearing expiry | exemption_id, entity_type, entity_id, expiry_date |
| `tax.rate.changed` | Tax rate effective date reached | rate_id, old_rate, new_rate, effective_date |
| `tax.withholding.calculated` | WHT calculated on payment | transaction_id, supplier_id, wht_amount_cents |

---

## 7. Migrations

### Migration Order for tax-service:
1. V001: `tax_registrations`
2. V002: `tax_jurisdictions`
3. V003: `tax_rates`
4. V004: `tax_codes`
5. V005: `tax_determinants`
6. V006: `tax_transactions`
7. V007: `tax_reporting_periods`
8. V008: `tax_returns`
9. V009: `withholding_tax_codes`
10. V010: `tax_exemptions`
11. V011: `tax_reconciliation`
12. V012: `tax_sequences`
13. V013: Triggers for `updated_at`
14. V014: Seed data (standard tax jurisdictions, common tax rate templates, default tax codes)
