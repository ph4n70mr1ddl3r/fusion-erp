# 10 - Cash Management Service Specification

## 1. Domain Overview

Cash Management (CM) manages bank accounts, cash positions, bank statement reconciliation, and cash forecasting. It serves as the bridge between AP/AR payment/receipt processing and the GL.

**Bounded Context:** Cash & Treasury Management
**Service Name:** `cm-service`
**Database:** `data/cm.db`
**HTTP Port:** 8014 | **gRPC Port:** 9014

---

## 2. Database Schema

### 2.1 Bank Accounts
```sql
CREATE TABLE bank_accounts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    account_number TEXT NOT NULL,           -- Masked: ****1234
    account_number_encrypted TEXT NOT NULL, -- AES-256-GCM encrypted full number
    account_name TEXT NOT NULL,
    bank_name TEXT NOT NULL,
    bank_code TEXT,                         -- Routing/SWIFT/sort code
    branch_code TEXT,
    account_type TEXT NOT NULL
        CHECK(account_type IN ('CHECKING','SAVINGS','ESCROW','PETTY_CASH')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    gl_account_id TEXT NOT NULL,            -- GL cash account
    opening_balance_cents INTEGER NOT NULL DEFAULT 0,
    current_balance_cents INTEGER NOT NULL DEFAULT 0,
    available_balance_cents INTEGER NOT NULL DEFAULT 0,
    last_reconciled_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','CLOSED','FROZEN')),
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, account_number)
);

CREATE INDEX idx_bank_accounts_tenant_status ON bank_accounts(tenant_id, status);
```

### 2.2 Cash Transactions
```sql
CREATE TABLE cash_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,       -- CM-TXN-2024-00001
    bank_account_id TEXT NOT NULL,
    transaction_date TEXT NOT NULL,
    value_date TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('DEPOSIT','WITHDRAWAL','TRANSFER','INTEREST','FEE','ADJUSTMENT')),
    direction TEXT NOT NULL
        CHECK(direction IN ('INFLOW','OUTFLOW')),
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    running_balance_cents INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    reference TEXT,
    counterparty TEXT,
    counterparty_account TEXT,

    -- Source reference
    source_type TEXT,                       -- 'AP_PAYMENT', 'AR_RECEIPT', 'MANUAL', 'BANK_STATEMENT'
    source_id TEXT,

    -- Reconciliation
    is_reconciled INTEGER NOT NULL DEFAULT 0,
    reconciled_with_statement_id TEXT,
    reconciled_at TEXT,
    reconciled_by TEXT,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','VOID','REVERSED')),
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (bank_account_id) REFERENCES bank_accounts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_cash_trans_tenant_bank ON cash_transactions(tenant_id, bank_account_id);
CREATE INDEX idx_cash_trans_tenant_date ON cash_transactions(tenant_id, transaction_date);
CREATE INDEX idx_cash_trans_tenant_recon ON cash_transactions(tenant_id, is_reconciled);
```

### 2.3 Bank Statements
```sql
CREATE TABLE bank_statements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bank_account_id TEXT NOT NULL,
    statement_date TEXT NOT NULL,
    statement_number TEXT,
    opening_balance_cents INTEGER NOT NULL,
    closing_balance_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    total_deposits_cents INTEGER NOT NULL DEFAULT 0,
    total_withdrawals_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'IMPORTED'
        CHECK(status IN ('IMPORTED','IN_REVIEW','RECONCILED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (bank_account_id) REFERENCES bank_accounts(id) ON DELETE RESTRICT
);

CREATE TABLE bank_statement_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    statement_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    transaction_date TEXT NOT NULL,
    value_date TEXT,
    description TEXT NOT NULL,
    amount_cents INTEGER NOT NULL,
    direction TEXT NOT NULL CHECK(direction IN ('DEBIT','CREDIT')),
    reference TEXT,
    counterparty TEXT,

    -- Matching
    is_matched INTEGER NOT NULL DEFAULT 0,
    matched_transaction_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (statement_id) REFERENCES bank_statements(id) ON DELETE CASCADE,
    FOREIGN KEY (matched_transaction_id) REFERENCES cash_transactions(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, statement_id, line_number)
);
```

### 2.4 Cash Forecasts
```sql
CREATE TABLE cash_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    forecast_name TEXT NOT NULL,
    forecast_date TEXT NOT NULL,
    period_from TEXT NOT NULL,
    period_to TEXT NOT NULL,
    opening_balance_cents INTEGER NOT NULL,
    projected_inflows_cents INTEGER NOT NULL DEFAULT 0,
    projected_outflows_cents INTEGER NOT NULL DEFAULT 0,
    projected_closing_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, forecast_name)
);
```

---

## 3. REST API Endpoints

```
# Bank Accounts
GET/POST      /api/v1/cm/bank-accounts              Permission: cm.bank-accounts.read/create
GET/PUT       /api/v1/cm/bank-accounts/{id}          Permission: cm.bank-accounts.read/update
GET           /api/v1/cm/bank-accounts/{id}/balance  Permission: cm.bank-accounts.read

# Cash Transactions
GET/POST      /api/v1/cm/transactions                Permission: cm.transactions.read/create
GET           /api/v1/cm/transactions/{id}            Permission: cm.transactions.read
POST          /api/v1/cm/transactions/{id}/void       Permission: cm.transactions.update
GET           /api/v1/cm/transactions/unreconciled    Permission: cm.transactions.read

# Bank Statements
POST          /api/v1/cm/statements/import           Permission: cm.statements.create
GET           /api/v1/cm/statements                   Permission: cm.statements.read
GET           /api/v1/cm/statements/{id}              Permission: cm.statements.read
GET           /api/v1/cm/statements/{id}/lines        Permission: cm.statements.read

# Reconciliation
POST          /api/v1/cm/reconcile                   Permission: cm.reconciliation.create
POST          /api/v1/cm/reconcile/auto              Permission: cm.reconciliation.create
GET           /api/v1/cm/reconcile/status/{id}       Permission: cm.reconciliation.read

# Cash Position & Forecast
GET           /api/v1/cm/cash-position                Permission: cm.forecasts.read
POST          /api/v1/cm/forecasts                    Permission: cm.forecasts.create
GET           /api/v1/cm/forecasts                    Permission: cm.forecasts.read
```

---

## 4. Business Rules

### 4.1 Balance Management
- Current balance updated on every cash transaction
- Available balance = current balance - pending (unreconciled outflows)
- Running balance maintained on each transaction for audit

### 4.2 Bank Reconciliation
1. Import bank statement (CSV, OFX, or manual entry)
2. System auto-matches statement lines to cash transactions (by amount, date, reference)
3. Manual matching for unmatched items
4. Reconciliation complete when: matched items = all statement lines AND closing balance = system balance

### 4.3 GL Integration
Each cash transaction creates a GL journal:
```
Inflow:  Debit Cash account, Credit source account
Outflow: Debit destination account, Credit Cash account
```

### 4.4 Events Published
| Event | Trigger |
|-------|---------|
| `cm.transaction.created` | Cash transaction recorded |
| `cm.transaction.reconciled` | Transaction matched with bank statement |
| `cm.statement.reconciled` | Bank statement fully reconciled |
| `cm.balance.updated` | Bank account balance changed |
