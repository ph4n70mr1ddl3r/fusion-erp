# 06 - General Ledger Service Specification

## 1. Domain Overview

The General Ledger (GL) service is the central financial record-keeping system. All financial transactions from subledgers (AP, AR, FA, CM, etc.) post to the GL. It manages the chart of accounts, journal entries, accounting periods, balances, and budgets.

**Bounded Context:** General Ledger & Financial Reporting
**Service Name:** `gl-service`
**Database:** `data/gl.db`
**HTTP Port:** 8010 | **gRPC Port:** 9010

---

## 2. Database Schema

### 2.1 Chart of Accounts Segments
```sql
CREATE TABLE gl_segment_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_name TEXT NOT NULL,         -- e.g., "Company", "Department", "Account", "Location"
    segment_code TEXT NOT NULL,         -- e.g., "COMP", "DEPT", "ACCT", "LOC"
    segment_sequence INTEGER NOT NULL,  -- Order in the account code
    segment_length INTEGER NOT NULL,    -- Number of characters
    separator TEXT NOT NULL DEFAULT '-',
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, segment_code)
);

CREATE INDEX idx_gl_segment_defs_tenant ON gl_segment_definitions(tenant_id, segment_sequence);
```

### 2.2 Segment Values
```sql
CREATE TABLE gl_segment_values (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_definition_id TEXT NOT NULL,
    value_code TEXT NOT NULL,           -- e.g., "1000", "010", "0001"
    value_name TEXT NOT NULL,           -- e.g., "Assets", "Operating", "Cash"
    parent_value_id TEXT,               -- For hierarchical segments
    description TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    is_enabled INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (segment_definition_id) REFERENCES gl_segment_definitions(id) ON DELETE RESTRICT,
    FOREIGN KEY (parent_value_id) REFERENCES gl_segment_values(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, segment_definition_id, value_code)
);
```

### 2.3 GL Accounts
```sql
CREATE TABLE gl_accounts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    account_code TEXT NOT NULL,             -- Full concatenated code: "1000-010-0001-0000"
    account_name TEXT NOT NULL,
    account_type TEXT NOT NULL CHECK(account_type IN ('ASSET','LIABILITY','EQUITY','REVENUE','EXPENSE')),
    account_category TEXT,                  -- Sub-classification: CURRENT_ASSET, FIXED_ASSET, etc.
    parent_account_id TEXT,                 -- Parent account for hierarchy
    description TEXT,
    is_postable INTEGER NOT NULL DEFAULT 1, -- Only postable accounts accept journal lines
    is_reconcile INTEGER NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    default_currency_code TEXT DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_account_id) REFERENCES gl_accounts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, account_code)
);

CREATE INDEX idx_gl_accounts_tenant_type ON gl_accounts(tenant_id, account_type);
CREATE INDEX idx_gl_accounts_tenant_parent ON gl_accounts(tenant_id, parent_account_id);
CREATE INDEX idx_gl_accounts_tenant_active ON gl_accounts(tenant_id, is_active);
```

### 2.4 Accounting Periods
```sql
CREATE TABLE gl_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,              -- "Jan-2024", "Q1-2024", "FY-2024"
    period_year INTEGER NOT NULL,
    period_number INTEGER NOT NULL,         -- 1-12 for months, 1-4 for quarters, 1 for year
    period_type TEXT NOT NULL CHECK(period_type IN ('MONTH','QUARTER','YEAR')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'FUTURE'
        CHECK(status IN ('FUTURE','OPEN','CLOSED','NEVER_OPENED')),
    closing_date TEXT,
    closed_by TEXT,
    quarter_name TEXT,                      -- Denormalized: "Q1-2024"
    year_name TEXT,                         -- Denormalized: "FY-2024"

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_name)
);

CREATE INDEX idx_gl_periods_tenant_dates ON gl_periods(tenant_id, start_date, end_date);
CREATE INDEX idx_gl_periods_tenant_status ON gl_periods(tenant_id, status);
CREATE INDEX idx_gl_periods_tenant_year ON gl_periods(tenant_id, period_year, period_number);
```

### 2.5 Journal Entries (Header)
```sql
CREATE TABLE journal_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journal_number TEXT NOT NULL,           -- Auto: "JE-2024-00001"
    batch_id TEXT,
    journal_source TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(journal_source IN ('MANUAL','AP','AR','FA','CM','INV','MFG','PM','REVAL','ALLOC')),
    description TEXT NOT NULL,
    journal_date TEXT NOT NULL,
    period_name TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','POSTED','REVERSED','ERROR')),
    posting_date TEXT,
    posted_by TEXT,
    reversal_of_id TEXT,
    reversal_reason TEXT,
    reference TEXT,                         -- External reference
    reference_type TEXT,                    -- "AP_INVOICE", "AR_INVOICE", etc.
    reference_id TEXT,                      -- ID of the source document

    -- Denormalized totals
    total_entered_debit_cents INTEGER NOT NULL DEFAULT 0,
    total_entered_credit_cents INTEGER NOT NULL DEFAULT 0,
    total_accounted_debit_cents INTEGER NOT NULL DEFAULT 0,
    total_accounted_credit_cents INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (reversal_of_id) REFERENCES journal_entries(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, journal_number)
);

CREATE INDEX idx_journal_entries_tenant_date ON journal_entries(tenant_id, journal_date);
CREATE INDEX idx_journal_entries_tenant_status ON journal_entries(tenant_id, status);
CREATE INDEX idx_journal_entries_tenant_period ON journal_entries(tenant_id, period_name);
CREATE INDEX idx_journal_entries_tenant_source ON journal_entries(tenant_id, journal_source);
CREATE INDEX idx_journal_entries_tenant_reference ON journal_entries(tenant_id, reference_type, reference_id);
```

### 2.6 Journal Lines
```sql
CREATE TABLE journal_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journal_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,

    account_id TEXT NOT NULL,
    description TEXT,

    -- Entered (foreign) currency amounts
    entered_debit_cents INTEGER NOT NULL DEFAULT 0,
    entered_credit_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Accounting (functional) currency amounts
    accounted_debit_cents INTEGER NOT NULL DEFAULT 0,
    accounted_credit_cents INTEGER NOT NULL DEFAULT 0,
    exchange_rate REAL,

    -- Reference to source document line
    reference_type TEXT,
    reference_id TEXT,

    -- Statistical amounts (optional)
    statistical_amount REAL,
    statistical_unit TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (journal_id) REFERENCES journal_entries(id) ON DELETE RESTRICT,
    FOREIGN KEY (account_id) REFERENCES gl_accounts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, journal_id, line_number),
    CHECK(entered_debit_cents >= 0),
    CHECK(entered_credit_cents >= 0),
    CHECK(accounted_debit_cents >= 0),
    CHECK(accounted_credit_cents >= 0)
);

CREATE INDEX idx_journal_lines_tenant_journal ON journal_lines(tenant_id, journal_id);
CREATE INDEX idx_journal_lines_tenant_account ON journal_lines(tenant_id, account_id);
```

### 2.7 Account Balances
```sql
CREATE TABLE gl_balances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    balance_type TEXT NOT NULL DEFAULT 'ACTUAL'
        CHECK(balance_type IN ('ACTUAL','BUDGET','ENCUMBRANCE')),
    currency_code TEXT NOT NULL DEFAULT 'USD',

    beginning_balance_cents INTEGER NOT NULL DEFAULT 0,
    period_debits_cents INTEGER NOT NULL DEFAULT 0,
    period_credits_cents INTEGER NOT NULL DEFAULT 0,
    ending_balance_cents INTEGER NOT NULL DEFAULT 0,

    net_change_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (account_id) REFERENCES gl_accounts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, account_id, period_name, balance_type, currency_code)
);

CREATE INDEX idx_gl_balances_tenant_account ON gl_balances(tenant_id, account_id);
CREATE INDEX idx_gl_balances_tenant_period ON gl_balances(tenant_id, period_name);
```

### 2.8 Budgets
```sql
CREATE TABLE gl_budgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    budget_name TEXT NOT NULL,
    fiscal_year INTEGER NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','CLOSED')),
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, budget_name, fiscal_year)
);

CREATE TABLE gl_budget_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    budget_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (budget_id) REFERENCES gl_budgets(id) ON DELETE RESTRICT,
    FOREIGN KEY (account_id) REFERENCES gl_accounts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, budget_id, account_id, period_name)
);

CREATE INDEX idx_gl_budget_lines_tenant_budget ON gl_budget_lines(tenant_id, budget_id);
CREATE INDEX idx_gl_budget_lines_tenant_account ON gl_budget_lines(tenant_id, account_id);
```

### 2.9 Journal Number Sequence
```sql
CREATE TABLE gl_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_type TEXT NOT NULL,            -- 'JOURNAL', 'BATCH'
    prefix TEXT NOT NULL,                   -- 'JE-', 'JB-'
    current_value INTEGER NOT NULL DEFAULT 0,
    fiscal_year INTEGER NOT NULL,

    UNIQUE(tenant_id, sequence_type, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Chart of Accounts

#### List Accounts
```
GET /api/v1/gl/accounts
  ?filter[account_type]=ASSET
  ?filter[is_postable]=true
  ?sort=+account_code
  ?fields=id,account_code,account_name,account_type
  ?cursor=xxx&limit=50

Response 200:
{
  "data": [
    {
      "id": "0192...",
      "account_code": "1000-010-0001-0000",
      "account_name": "Cash - Operating",
      "account_type": "ASSET",
      "account_category": "CURRENT_ASSET",
      "is_postable": true,
      "parent_account_id": "0192...",
      "effective_from": "2020-01-01",
      "effective_to": null
    }
  ],
  "meta": { "pagination": { "cursor": "...", "has_more": true, "total_count": 150 } }
}
```

#### Create Account
```
POST /api/v1/gl/accounts
  Permission: gl.accounts.create

Request:
{
  "account_code": "1000-010-0001-0000",
  "account_name": "Cash - Operating",
  "account_type": "ASSET",
  "account_category": "CURRENT_ASSET",
  "parent_account_id": "0192...",
  "is_postable": true,
  "description": "Operating cash account",
  "effective_from": "2024-01-01"
}

Response 201: { "data": { "id": "0192...", ... } }
```

#### Get Account
```
GET /api/v1/gl/accounts/{id}
  Permission: gl.accounts.read

Response 200: { "data": { ... full account object } }
```

#### Update Account
```
PUT /api/v1/gl/accounts/{id}
  Permission: gl.accounts.update
  Headers: If-Match: {version}

Response 200: { "data": { ... updated account } }
```

#### Get Account Hierarchy (Tree)
```
GET /api/v1/gl/accounts/tree
  Permission: gl.accounts.read

Response 200:
{
  "data": {
    "accounts": [
      {
        "id": "...", "account_code": "1000", "account_name": "Assets",
        "children": [
          { "id": "...", "account_code": "1000-010", "account_name": "Current Assets", "children": [...] }
        ]
      }
    ]
  }
}
```

#### Get Account Segments
```
GET /api/v1/gl/segments
GET /api/v1/gl/segments/{id}/values
POST /api/v1/gl/segments
POST /api/v1/gl/segments/{id}/values
```

### 3.2 Journal Entries

#### List Journals
```
GET /api/v1/gl/journals
  ?filter[status]=POSTED
  ?filter[date_from]=2024-01-01&filter[date_to]=2024-01-31
  ?filter[source]=AP
  ?sort=-journal_date,+journal_number
```

#### Create Journal
```
POST /api/v1/gl/journals
  Permission: gl.journals.create
  Headers: X-Idempotency-Key: {uuid}

Request:
{
  "description": "Monthly rent expense",
  "journal_date": "2024-01-15",
  "currency_code": "USD",
  "reference": "LEASE-2024-001",
  "lines": [
    {
      "account_id": "0192...expense",
      "description": "Rent expense - January",
      "entered_debit_cents": 500000,
      "entered_credit_cents": 0
    },
    {
      "account_id": "0192...cash",
      "description": "Cash payment for rent",
      "entered_debit_cents": 0,
      "entered_credit_cents": 500000
    }
  ]
}

Response 201:
{
  "data": {
    "id": "0192...",
    "journal_number": "JE-2024-00001",
    "status": "DRAFT",
    "total_entered_debit_cents": 500000,
    "total_entered_credit_cents": 500000,
    "line_count": 2,
    "lines": [ ... ]
  }
}
```

#### Get Journal (with lines)
```
GET /api/v1/gl/journals/{id}
  Permission: gl.journals.read
  Query: ?include=lines

Response 200: { "data": { ... journal with lines array } }
```

#### Post Journal
```
POST /api/v1/gl/journals/{id}/post
  Permission: gl.journals.post

Response 200:
{
  "data": {
    "id": "0192...",
    "status": "POSTED",
    "posting_date": "2024-01-15T10:30:00Z",
    "posted_by": "user-id"
  }
}
```

#### Batch Post
```
POST /api/v1/gl/journals/batch-post
  Permission: gl.journals.post

Request: { "journal_ids": ["id1", "id2", "id3"] }
Response 200: { "data": { "posted": 3, "failed": 0, "results": [...] } }
```

#### Reverse Journal
```
POST /api/v1/gl/journals/{id}/reverse
  Permission: gl.journals.reverse

Request:
{
  "reason": "Correcting entry - wrong amount",
  "reversal_date": "2024-02-01"
}

Response 201: { "data": { "id": "0192...reversal", "journal_number": "JE-2024-00042", "reversal_of_id": "0192..." } }
```

### 3.3 Periods

#### List Periods
```
GET /api/v1/gl/periods
  ?filter[period_year]=2024
  ?filter[status]=OPEN
```

#### Create Fiscal Year Periods (Auto-generate)
```
POST /api/v1/gl/periods/generate
  Permission: gl.periods.create

Request:
{
  "fiscal_year": 2024,
  "start_month": 1,
  "periods_per_year": 12,
  "period_type": "MONTH"
}

Response 201: Generates 12 monthly periods + 4 quarterly + 1 yearly
```

#### Close Period
```
POST /api/v1/gl/periods/{id}/close
  Permission: gl.periods.close

Response 200: { "data": { "id": "...", "status": "CLOSED", "closing_date": "..." } }
```

#### Reopen Period
```
POST /api/v1/gl/periods/{id}/reopen
  Permission: gl.periods.reopen
```

### 3.4 Balances & Reporting

#### Trial Balance
```
GET /api/v1/gl/trial-balance
  ?period_name=Jan-2024
  ?balance_type=ACTUAL
  ?include_zero_balances=false

Response 200:
{
  "data": {
    "period_name": "Jan-2024",
    "as_of_date": "2024-01-31",
    "currency_code": "USD",
    "total_debit_cents": 15000000,
    "total_credit_cents": 15000000,
    "lines": [
      {
        "account_code": "1000-010-0001-0000",
        "account_name": "Cash - Operating",
        "account_type": "ASSET",
        "beginning_balance_cents": 5000000,
        "period_debits_cents": 1000000,
        "period_credits_cents": 500000,
        "ending_balance_cents": 5500000
      }
    ]
  }
}
```

#### Account Balances
```
GET /api/v1/gl/balances
  ?filter[account_id]={id}
  ?filter[period_name]=Jan-2024
  ?filter[balance_type]=ACTUAL
```

#### Balance Sheet
```
GET /api/v1/gl/balance-sheet?period_name=Mar-2024
```

#### Income Statement
```
GET /api/v1/gl/income-statement?period_from=Jan-2024&period_to=Mar-2024
```

### 3.5 Budgets

```
GET    /api/v1/gl/budgets
POST   /api/v1/gl/budgets                     Permission: gl.budgets.create
GET    /api/v1/gl/budgets/{id}
PUT    /api/v1/gl/budgets/{id}                 Permission: gl.budgets.update
POST   /api/v1/gl/budgets/{id}/activate       Permission: gl.budgets.update
GET    /api/v1/gl/budgets/{id}/lines
POST   /api/v1/gl/budgets/{id}/lines           Permission: gl.budgets.create
PUT    /api/v1/gl/budgets/{id}/lines/bulk      Permission: gl.budgets.update
GET    /api/v1/gl/budgets/{id}/vs-actual       Permission: gl.budgets.read
```

---

## 4. Business Rules

### 4.1 Double-Entry Bookkeeping
- Every journal entry MUST have at least 2 lines
- Total debits MUST equal total credits (per journal, per currency)
- Debit and credit on the same line are mutually exclusive (one must be 0)
- Validation: `SUM(entered_debit_cents) = SUM(entered_credit_cents)`

### 4.2 Account Rules
- Only accounts with `is_postable = 1` can be used on journal lines
- Account codes are unique per tenant
- Account type determines normal balance direction:
  - ASSET, EXPENSE: Normal debit balance
  - LIABILITY, EQUITY, REVENUE: Normal credit balance
- Parent accounts (non-postable) aggregate child balances for reporting

### 4.3 Period Rules
- Cannot post to a CLOSED or FUTURE period
- Period MUST be OPEN to post
- Closing a period:
  1. Validate no DRAFT/SUBMITTED journals exist for the period
  2. Calculate final period balances
  3. Set period status to CLOSED
  4. Roll ending balance forward to next period's beginning balance
- Reopening a period requires permission and creates audit trail

### 4.4 Journal Numbering
- Format: `JE-{YYYY}-{NNNNN}` (e.g., JE-2024-00001)
- Sequential per tenant per fiscal year
- Gap-free: number is reserved on creation, not reused
- Source-specific prefix optional: `AP-2024-00001`, `AR-2024-00001`

### 4.5 Journal Status Flow
```
DRAFT → SUBMITTED → APPROVED → POSTED
                                    ↓
                              REVERSED (creates reversal journal)
```

- DRAFT: Editable, can add/remove lines
- SUBMITTED: Locked for editing, awaiting approval
- APPROVED: Approved, ready for posting
- POSTED: Immutable, balances updated
- REVERSED: Reversed by a reversal journal entry

### 4.6 Reversal Rules
- Reversal creates a new journal with inverted debits and credits
- Original journal status changes to REVERSED
- Reversal journal references original via `reversal_of_id`
- Reversal date can differ from original date (must be in an open period)
- Cannot reverse a reversal

### 4.7 Multi-Currency
- Journal entries can be in any currency
- `entered_*_cents`: amounts in the transaction currency
- `accounted_*_cents`: amounts converted to the accounting currency
- Exchange rate stored on each line
- Currency revaluation: periodic process to adjust foreign currency balances

### 4.8 Balance Calculation
- Balances updated when journal is POSTED (not before)
- `ending_balance = beginning_balance + period_debits - period_credits`
  - For ASSET/EXPENSE accounts
  - For LIABILITY/EQUITY/REVENUE: `ending = beginning + credits - debits`
- On period close: next period's beginning = current period's ending

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.gl.v1;

service GeneralLedgerService {
    // Account operations
    rpc GetAccount(GetAccountRequest) returns (GetAccountResponse);
    rpc GetAccountsByCode(GetAccountsByCodeRequest) returns (GetAccountsByCodeResponse);

    // Journal operations (used by subledgers)
    rpc CreateJournal(CreateJournalRequest) returns (CreateJournalResponse);
    rpc PostJournal(PostJournalRequest) returns (PostJournalResponse);
    rpc ReverseJournal(ReverseJournalRequest) returns (ReverseJournalResponse);

    // Balance operations
    rpc GetBalances(GetBalancesRequest) returns (GetBalancesResponse);
    rpc GetTrialBalance(GetTrialBalanceRequest) returns (GetTrialBalanceResponse);

    // Period operations
    rpc ClosePeriod(ClosePeriodRequest) returns (ClosePeriodResponse);
    rpc GetOpenPeriod(GetOpenPeriodRequest) returns (GetOpenPeriodResponse);
}

// Account messages
message GetAccountRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetAccountResponse {
    GlAccount data = 1;
}

message GetAccountsByCodeRequest {
    string tenant_id = 1;
    string account_code = 2;
}

message GetAccountsByCodeResponse {
    repeated GlAccount items = 1;
}

message GlAccount {
    string id = 1;
    string tenant_id = 2;
    string account_code = 3;
    string account_name = 4;
    string account_type = 5;
    string account_category = 6;
    string parent_account_id = 7;
    string description = 8;
    int32 is_postable = 9;
    int32 is_reconcile = 10;
    string effective_from = 11;
    string effective_to = 12;
    string default_currency_code = 13;
    string created_at = 14;
    string updated_at = 15;
}

// Journal messages
message CreateJournalRequest {
    string tenant_id = 1;
    string description = 2;
    string journal_date = 3;
    string currency_code = 4;
    string reference = 5;
    string reference_type = 6;
    string reference_id = 7;
    repeated JournalLineInput lines = 8;
}

message JournalLineInput {
    string account_id = 1;
    string description = 2;
    int64 entered_debit_cents = 3;
    int64 entered_credit_cents = 4;
    string currency_code = 5;
}

message CreateJournalResponse {
    JournalEntry data = 1;
}

message PostJournalRequest {
    string tenant_id = 1;
    string id = 2;
}

message PostJournalResponse {
    string id = 1;
    string status = 2;
    string posting_date = 3;
    string posted_by = 4;
}

message ReverseJournalRequest {
    string tenant_id = 1;
    string id = 2;
    string reason = 3;
    string reversal_date = 4;
}

message ReverseJournalResponse {
    JournalEntry data = 1;
}

message JournalEntry {
    string id = 1;
    string tenant_id = 2;
    string journal_number = 3;
    string batch_id = 4;
    string journal_source = 5;
    string description = 6;
    string journal_date = 7;
    string period_name = 8;
    string currency_code = 9;
    string status = 10;
    string posting_date = 11;
    string posted_by = 12;
    string reversal_of_id = 13;
    string reversal_reason = 14;
    string reference = 15;
    string reference_type = 16;
    string reference_id = 17;
    int64 total_entered_debit_cents = 18;
    int64 total_entered_credit_cents = 19;
    int64 total_accounted_debit_cents = 20;
    int64 total_accounted_credit_cents = 21;
    int32 line_count = 22;
    string created_at = 23;
    string updated_at = 24;
}

// Balance messages
message GetBalancesRequest {
    string tenant_id = 1;
    string account_id = 2;
    string period_name = 3;
    string balance_type = 4;
}

message GetBalancesResponse {
    repeated GlBalance items = 1;
}

message GlBalance {
    string id = 1;
    string tenant_id = 2;
    string account_id = 3;
    string period_name = 4;
    string balance_type = 5;
    string currency_code = 6;
    int64 beginning_balance_cents = 7;
    int64 period_debits_cents = 8;
    int64 period_credits_cents = 9;
    int64 ending_balance_cents = 10;
    int64 net_change_cents = 11;
}

message GetTrialBalanceRequest {
    string tenant_id = 1;
    string period_name = 2;
    string balance_type = 3;
    int32 include_zero_balances = 4;
}

message GetTrialBalanceResponse {
    string period_name = 1;
    string as_of_date = 2;
    string currency_code = 3;
    int64 total_debit_cents = 4;
    int64 total_credit_cents = 5;
    repeated TrialBalanceLine lines = 6;
}

message TrialBalanceLine {
    string account_code = 1;
    string account_name = 2;
    string account_type = 3;
    int64 beginning_balance_cents = 4;
    int64 period_debits_cents = 5;
    int64 period_credits_cents = 6;
    int64 ending_balance_cents = 7;
}

// Period messages
message ClosePeriodRequest {
    string tenant_id = 1;
    string id = 2;
}

message ClosePeriodResponse {
    string id = 1;
    string status = 2;
    string closing_date = 3;
}

message GetOpenPeriodRequest {
    string tenant_id = 1;
    string period_name = 2;
}

message GetOpenPeriodResponse {
    GlPeriod data = 1;
}

message GlPeriod {
    string id = 1;
    string tenant_id = 2;
    string period_name = 3;
    int32 period_year = 4;
    int32 period_number = 5;
    string period_type = 6;
    string start_date = 7;
    string end_date = 8;
    string status = 9;
    string closing_date = 10;
    string closed_by = 11;
    string quarter_name = 12;
    string year_name = 13;
    string created_at = 14;
    string updated_at = 15;
}
```

---

## 6. Inter-Service Integration

### 6.1 Subledger Journal Creation
When subledgers create financial transactions, they call GL via gRPC:

**AP Invoice Posting → GL Journal:**
```
Debit: Expense/Asset accounts (from invoice distribution lines)
Credit: Accounts Payable control account
```

**AP Payment → GL Journal:**
```
Debit: Accounts Payable control account
Credit: Cash/Bank account
```

**AR Invoice Posting → GL Journal:**
```
Debit: Accounts Receivable control account
Credit: Revenue accounts (from invoice lines)
Credit: Tax payable accounts
```

**FA Depreciation → GL Journal:**
```
Debit: Depreciation expense account
Credit: Accumulated depreciation account
```

**CM Bank Transaction → GL Journal:**
```
Debit/Credit: Bank account (depending on transaction type)
Credit/Debit: Corresponding account
```

### 6.2 Event Publishing
GL publishes these events for other services to consume:

| Event | Trigger | Payload |
|-------|---------|---------|
| `gl.journal.created` | Journal created | journal_id, journal_number, tenant_id |
| `gl.journal.posted` | Journal posted | journal_id, period_name, totals |
| `gl.journal.reversed` | Journal reversed | journal_id, reversal_journal_id |
| `gl.period.closed` | Period closed | period_name, period_year |
| `gl.period.opened` | Period opened/reopened | period_name |
| `gl.account.created` | Account created | account_id, account_code |
| `gl.budget.exceeded` | Actual exceeds budget | account_id, budget_id, period |

---

## 7. Migrations

### Migration Order for gl-service:
1. V001: `gl_segment_definitions`
2. V002: `gl_segment_values`
3. V003: `gl_accounts`
4. V004: `gl_periods`
5. V005: `gl_sequences`
6. V006: `journal_entries`
7. V007: `journal_lines`
8. V008: `gl_balances`
9. V009: `gl_budgets`
10. V010: `gl_budget_lines`
11. V011: Triggers for `updated_at`
12. V012: Seed data (default segments, standard account templates)
