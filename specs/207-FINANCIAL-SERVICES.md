# 207 - Financial Services Industry Solution Specification

## 1. Domain Overview

Financial Services provides industry-specific capabilities for banking, insurance, and capital markets operations including regulatory compliance reporting, loan lifecycle management, KYC/AML screening, treasury operations, and risk-weighted asset calculations. Supports bank account and deposit management, loan origination and servicing, insurance policy administration, claims processing, regulatory reporting (Basel III/IV, Solvency II, IFRS 9), anti-money laundering screening, and treasury and liquidity management. Enables financial institutions to manage complex regulatory requirements, streamline lending operations, and maintain compliance across jurisdictions. Integrates with General Ledger, Risk & Compliance, and Financial Reporting.

**Bounded Context:** Banking, Insurance & Capital Markets Operations
**Service Name:** `financial-services-service`
**Database:** `data/financial_services.db`
**HTTP Port:** 8225 | **gRPC Port:** 9225

---

## 2. Database Schema

### 2.1 Loan Management
```sql
CREATE TABLE fs_loans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    loan_number TEXT NOT NULL,
    loan_type TEXT NOT NULL CHECK(loan_type IN ('MORTGAGE','COMMERCIAL','CONSUMER','LINE_OF_CREDIT','SYNDICATED','SBA')),
    borrower_id TEXT NOT NULL,
    borrower_name TEXT NOT NULL,
    principal_cents INTEGER NOT NULL DEFAULT 0,
    interest_rate_pct REAL NOT NULL DEFAULT 0,
    term_months INTEGER NOT NULL,
    origination_date TEXT NOT NULL,
    maturity_date TEXT NOT NULL,
    collateral_type TEXT,
    collateral_value_cents INTEGER,
    payment_frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(payment_frequency IN ('WEEKLY','BIWEEKLY','MONTHLY','QUARTERLY')),
    outstanding_balance_cents INTEGER NOT NULL DEFAULT 0,
    next_payment_date TEXT,
    next_payment_amount_cents INTEGER NOT NULL DEFAULT 0,
    days_past_due INTEGER NOT NULL DEFAULT 0,
    risk_rating TEXT NOT NULL DEFAULT 'PASS'
        CHECK(risk_rating IN ('PASS','SPECIAL_MENTION','SUBSTANDARD','DOUBTFUL','LOSS')),
    provision_cents INTEGER NOT NULL DEFAULT 0,
    officer_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ORIGINATED'
        CHECK(status IN ('ORIGINATED','ACTIVE','DELINQUENT','DEFAULT','RESTRUCTURED','PAID_OFF','CHARGED_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, loan_number)
);

CREATE INDEX idx_fs_loan_tenant ON fs_loans(tenant_id, status);
CREATE INDEX idx_fs_loan_borrower ON fs_loans(borrower_id);
CREATE INDEX idx_fs_loan_type ON fs_loans(loan_type);
CREATE INDEX idx_fs_loan_risk ON fs_loans(risk_rating);
CREATE INDEX idx_fs_loan_maturity ON fs_loans(maturity_date);
```

### 2.2 KYC/AML Screening
```sql
CREATE TABLE fs_kyc_screenings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_type TEXT NOT NULL CHECK(entity_type IN ('INDIVIDUAL','CORPORATE','PARTNERSHIP','TRUST')),
    entity_id TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    screening_type TEXT NOT NULL CHECK(screening_type IN ('KYC_ONBOARDING','PERIODIC_REVIEW','TRANSACTION_TRIGGERED','PEP_CHECK','SANCTION_CHECK')),
    country_of_residence TEXT NOT NULL,
    country_of_incorporation TEXT,
    identification_documents TEXT NOT NULL,        -- JSON: document details
    beneficial_owners TEXT,                        -- JSON: UBO details
    screening_results TEXT NOT NULL,               -- JSON: matches found
    risk_score REAL NOT NULL DEFAULT 0,
    risk_level TEXT NOT NULL DEFAULT 'LOW'
        CHECK(risk_level IN ('LOW','MEDIUM','HIGH','PROHIBITED')),
    pep_flag INTEGER NOT NULL DEFAULT 0,
    sanction_flag INTEGER NOT NULL DEFAULT 0,
    adverse_media_flag INTEGER NOT NULL DEFAULT 0,
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_notes TEXT,
    next_review_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','CLEARED','ESCALATED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(id)
);

CREATE INDEX idx_fs_kyc_entity ON fs_kyc_screenings(entity_id, status);
CREATE INDEX idx_fs_kyc_risk ON fs_kyc_screenings(risk_level);
CREATE INDEX idx_fs_kyc_review ON fs_kyc_screenings(next_review_date);
```

### 2.3 Insurance Policies
```sql
CREATE TABLE fs_insurance_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_number TEXT NOT NULL,
    policy_type TEXT NOT NULL CHECK(policy_type IN ('LIFE','HEALTH','PROPERTY','CASUALTY','AUTO','TRAVEL','COMMERCIAL')),
    product_name TEXT NOT NULL,
    insured_id TEXT NOT NULL,
    insured_name TEXT NOT NULL,
    premium_cents INTEGER NOT NULL DEFAULT 0,
    premium_frequency TEXT NOT NULL DEFAULT 'ANNUAL'
        CHECK(premium_frequency IN ('MONTHLY','QUARTERLY','SEMI_ANNUAL','ANNUAL')),
    coverage_amount_cents INTEGER NOT NULL DEFAULT 0,
    deductible_cents INTEGER NOT NULL DEFAULT 0,
    effective_date TEXT NOT NULL,
    expiry_date TEXT NOT NULL,
    renewal_date TEXT,
    beneficiaries TEXT,                            -- JSON: beneficiary details
    underwriter_id TEXT NOT NULL,
    agent_id TEXT,
    claims_count INTEGER NOT NULL DEFAULT 0,
    total_claims_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'QUOTED'
        CHECK(status IN ('QUOTED','ACTIVE','LAPSED','CANCELLED','EXPIRED','CLAIMED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_number)
);

CREATE INDEX idx_fs_policy_tenant ON fs_insurance_policies(tenant_id, status);
CREATE INDEX idx_fs_policy_type ON fs_insurance_policies(policy_type);
CREATE INDEX idx_fs_policy_expiry ON fs_insurance_policies(expiry_date);
CREATE INDEX idx_fs_policy_insured ON fs_insurance_policies(insured_id);
```

### 2.4 Regulatory Reports
```sql
CREATE TABLE fs_regulatory_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_code TEXT NOT NULL,
    report_name TEXT NOT NULL,
    reporting_framework TEXT NOT NULL CHECK(reporting_framework IN ('BASEL_III','BASEL_IV','SOLVENCY_II','IFRS_9','LOCAL_REGULATORY')),
    report_type TEXT NOT NULL CHECK(report_type IN ('CAPITAL_ADEQUACY','LIQUIDITY','LARGE_EXPOSURES','LEVERAGE','NPL','PROVISIONING')),
    period TEXT NOT NULL,
    submission_deadline TEXT NOT NULL,
    data_payload TEXT NOT NULL,                    -- JSON: report data
    calculated_ratios TEXT,                        -- JSON: regulatory ratios
    thresholds TEXT,                              -- JSON: regulatory thresholds
    breach_flag INTEGER NOT NULL DEFAULT 0,
    prepared_by TEXT NOT NULL,
    reviewed_by TEXT,
    approved_by TEXT,
    submitted_at TEXT,
    status TEXT NOT NULL DEFAULT 'PREPARATION'
        CHECK(status IN ('PREPARATION','REVIEW','APPROVED','SUBMITTED','ACKNOWLEDGED','RESUBMISSION')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, report_code, period)
);

CREATE INDEX idx_fs_reg_framework ON fs_regulatory_reports(reporting_framework, period);
CREATE INDEX idx_fs_reg_status ON fs_regulatory_reports(tenant_id, status);
CREATE INDEX idx_fs_reg_deadline ON fs_regulatory_reports(submission_deadline);
```

### 2.5 Treasury Operations
```sql
CREATE TABLE fs_treasury_operations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    operation_type TEXT NOT NULL CHECK(operation_type IN ('FX_TRADE','MONEY_MARKET','BOND','DEPOSIT','LOAN_TRANSFER','SWAP')),
    trade_number TEXT NOT NULL,
    counterparty TEXT NOT NULL,
    buy_currency TEXT,
    sell_currency TEXT,
    buy_amount_cents INTEGER NOT NULL DEFAULT 0,
    sell_amount_cents INTEGER NOT NULL DEFAULT 0,
    exchange_rate REAL,
    trade_date TEXT NOT NULL,
    value_date TEXT NOT NULL,
    maturity_date TEXT,
    portfolio_id TEXT,
    book_id TEXT,
    dealer_id TEXT NOT NULL,
    settlement_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(settlement_status IN ('PENDING','SETTLED','FAILED','CANCELLED')),
    status TEXT NOT NULL DEFAULT 'EXECUTED'
        CHECK(status IN ('DRAFT','EXECUTED','CONFIRMED','SETTLED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, trade_number)
);

CREATE INDEX idx_fs_treasury_type ON fs_treasury_operations(operation_type, trade_date DESC);
CREATE INDEX idx_fs_treasury_counterparty ON fs_treasury_operations(counterparty);
CREATE INDEX idx_fs_treasury_status ON fs_treasury_operations(tenant_id, status);
CREATE INDEX idx_fs_treasury_dates ON fs_treasury_operations(value_date);
```

---

## 3. API Endpoints

### 3.1 Loans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-services/v1/loans` | Originate loan |
| GET | `/financial-services/v1/loans` | List loans |
| GET | `/financial-services/v1/loans/{id}` | Get loan details |
| PUT | `/financial-services/v1/loans/{id}` | Update loan |
| POST | `/financial-services/v1/loans/{id}/disburse` | Disburse loan |
| GET | `/financial-services/v1/loans/portfolio` | Portfolio summary |

### 3.2 KYC/AML
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-services/v1/kyc/screen` | Run screening |
| GET | `/financial-services/v1/kyc/screenings` | List screenings |
| POST | `/financial-services/v1/kyc/{id}/clear` | Clear screening |
| GET | `/financial-services/v1/kyc/due-reviews` | List due reviews |

### 3.3 Insurance
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-services/v1/policies` | Create policy |
| GET | `/financial-services/v1/policies` | List policies |
| GET | `/financial-services/v1/policies/{id}` | Get policy |
| PUT | `/financial-services/v1/policies/{id}` | Update policy |
| POST | `/financial-services/v1/policies/{id}/renew` | Renew policy |

### 3.4 Regulatory Reporting
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-services/v1/regulatory-reports` | Generate report |
| GET | `/financial-services/v1/regulatory-reports` | List reports |
| GET | `/financial-services/v1/regulatory-reports/{id}` | Get report |
| POST | `/financial-services/v1/regulatory-reports/{id}/submit` | Submit to regulator |

### 3.5 Treasury
| Method | Path | Description |
|--------|------|-------------|
| POST | `/financial-services/v1/treasury/trades` | Execute trade |
| GET | `/financial-services/v1/treasury/trades` | List trades |
| GET | `/financial-services/v1/treasury/positions` | Current positions |
| GET | `/financial-services/v1/treasury/exposure` | FX exposure |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `finsvc.loan.disbursed` | `{ loan_id, amount, borrower }` | Loan disbursed |
| `finsvc.kyc.flagged` | `{ screening_id, risk_level, flags }` | KYC screening flagged |
| `finsvc.policy.issued` | `{ policy_id, type, premium }` | Policy issued |
| `finsvc.regulatory.breach` | `{ report_id, ratio, threshold }` | Regulatory breach detected |
| `finsvc.trade.executed` | `{ trade_id, type, amount }` | Treasury trade executed |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `payment.received` | Payment (14) | Apply loan payment |
| `gl.journal.posted` | GL (06) | Update positions |
| `compliance.rule.triggered` | Compliance (34) | Escalate KYC/AML alert |

---

## 5. Business Rules

1. **KYC Renewal**: Customer screenings renewed based on risk level (annual for low, quarterly for high)
2. **Loan Provisioning**: Provisions calculated per IFRS 9 expected credit loss model
3. **Regulatory Deadlines**: Reports must be submitted before regulatory deadline; breaches trigger alerts
4. **Sanction Screening**: All transactions screened against sanctions lists in real-time
5. **Capital Adequacy**: CET1, Tier 1, and total capital ratios monitored against Basel requirements
6. **Insurance Reserves**: Claim reserves calculated using actuarial methods per policy type

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| General Ledger (06) | Journal entries and posting |
| Cash Management (10) | Treasury and liquidity |
| Risk & Compliance (34) | Risk assessment and screening |
| Financial Reporting (154) | Regulatory report generation |
| Financial Consolidation (100) | Group-level reporting |
| BI Publisher (150) | Report templates and bursting |
