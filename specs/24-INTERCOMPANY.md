# 24 - Intercompany Accounting Service Specification

## 1. Domain Overview

Intercompany Accounting (IC) manages financial transactions between legal entities within the same tenant. It handles intercompany sales, purchases, transfers, and loans — generating paired journal entries (IC receivable and IC payable) that net to zero at the consolidated level. The service supports transfer pricing, multi-currency transactions, automatic reconciliation, and elimination entries for consolidated financial reporting.

**Bounded Context:** Intercompany Accounting & Consolidation Eliminations
**Service Name:** `ic-service`
**Database:** `data/ic.db`
**HTTP Port:** 8016 | **gRPC Port:** 9016

---

## 2. Database Schema

### 2.1 Intercompany Entities
```sql
CREATE TABLE intercompany_entities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_code TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    legal_entity_id TEXT NOT NULL,            -- Reference to external legal entity register
    gl_company_segment TEXT NOT NULL,         -- Company segment value in GL chart of accounts
    base_currency_code TEXT NOT NULL DEFAULT 'USD',
    tax_id TEXT,
    description TEXT,

    -- IC accounting configuration
    ic_receivable_account_id TEXT NOT NULL,   -- Default IC receivable account
    ic_payable_account_id TEXT NOT NULL,      -- Default IC payable account
    ic_revenue_account_id TEXT,               -- Default IC revenue account
    ic_expense_account_id TEXT,               -- Default IC expense account
    ic_auto_match INTEGER NOT NULL DEFAULT 0, -- Auto-match IC transactions

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, entity_code)
);

CREATE INDEX idx_ic_entities_tenant_status ON intercompany_entities(tenant_id, status);
```

### 2.2 Intercompany Relationships
```sql
CREATE TABLE intercompany_relationships (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    from_entity_id TEXT NOT NULL,
    to_entity_id TEXT NOT NULL,
    relationship_type TEXT NOT NULL DEFAULT 'BILATERAL'
        CHECK(relationship_type IN ('BILATERAL','PARENT_SUBSIDIARY','SIBLING')),

    -- Transfer pricing
    pricing_method TEXT NOT NULL DEFAULT 'ARM_LENGTH'
        CHECK(pricing_method IN ('ARM_LENGTH','COST_PLUS','MARKET_BASED','FIXED_MARKUP','NEGOTIATED')),
    markup_percent REAL DEFAULT 0,
    cost_plus_percent REAL DEFAULT 0,
    default_currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Matching tolerances
    match_tolerance_amount_cents INTEGER NOT NULL DEFAULT 0,
    match_tolerance_percent REAL DEFAULT 0,

    -- Override accounts (if different from entity defaults)
    from_ic_receivable_account_id TEXT,
    from_ic_revenue_account_id TEXT,
    to_ic_payable_account_id TEXT,
    to_ic_expense_account_id TEXT,

    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (from_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    FOREIGN KEY (to_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, from_entity_id, to_entity_id)
);

CREATE INDEX idx_ic_rels_tenant_from ON intercompany_relationships(tenant_id, from_entity_id);
CREATE INDEX idx_ic_rels_tenant_to ON intercompany_relationships(tenant_id, to_entity_id);
```

### 2.3 Intercompany Transactions (Header)
```sql
CREATE TABLE intercompany_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,         -- IC-TXN-2024-00001
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('SALE','PURCHASE','TRANSFER','LOAN','SERVICE','ROYALTY','MANAGEMENT_FEE','EXPENSE_ALLOCATION')),

    -- Source entity (seller/lender)
    source_entity_id TEXT NOT NULL,
    source_entity_code TEXT NOT NULL,

    -- Target entity (buyer/borrower)
    target_entity_id TEXT NOT NULL,
    target_entity_code TEXT NOT NULL,

    -- Transaction detail
    description TEXT NOT NULL,
    transaction_date TEXT NOT NULL,
    accounting_date TEXT NOT NULL,
    period_name TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,

    -- Transfer pricing
    pricing_method TEXT NOT NULL DEFAULT 'ARM_LENGTH'
        CHECK(pricing_method IN ('ARM_LENGTH','COST_PLUS','MARKET_BASED','FIXED_MARKUP','NEGOTIATED')),
    markup_percent REAL DEFAULT 0,
    cost_basis_cents INTEGER NOT NULL DEFAULT 0,

    -- Amounts (in transaction currency)
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,

    -- Amounts in source entity functional currency
    source_total_cents INTEGER NOT NULL DEFAULT 0,
    source_currency_code TEXT NOT NULL DEFAULT 'USD',
    source_exchange_rate REAL,

    -- Amounts in target entity functional currency
    target_total_cents INTEGER NOT NULL DEFAULT 0,
    target_currency_code TEXT NOT NULL DEFAULT 'USD',
    target_exchange_rate REAL,

    -- Status
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','POSTED','SETTLED','CANCELLED')),

    -- Settlement
    settlement_date TEXT,
    settlement_method TEXT
        CHECK(settlement_method IN ('NETTING','PAYMENT','CLEARING','AUTO')),

    -- GL references
    source_gl_journal_id TEXT,
    target_gl_journal_id TEXT,
    elimination_gl_journal_id TEXT,

    -- Source document reference
    source_reference_type TEXT,               -- 'AR_INVOICE', 'AP_INVOICE', 'MANUAL', etc.
    source_reference_id TEXT,

    line_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    FOREIGN KEY (target_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_ic_trans_tenant_source ON intercompany_transactions(tenant_id, source_entity_id);
CREATE INDEX idx_ic_trans_tenant_target ON intercompany_transactions(tenant_id, target_entity_id);
CREATE INDEX idx_ic_trans_tenant_status ON intercompany_transactions(tenant_id, status);
CREATE INDEX idx_ic_trans_tenant_date ON intercompany_transactions(tenant_id, transaction_date);
CREATE INDEX idx_ic_trans_tenant_period ON intercompany_transactions(tenant_id, period_name);
CREATE INDEX idx_ic_trans_tenant_type ON intercompany_transactions(tenant_id, transaction_type);
```

### 2.4 Intercompany Transaction Lines
```sql
CREATE TABLE intercompany_transaction_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,

    -- Line detail
    description TEXT NOT NULL,
    line_type TEXT NOT NULL DEFAULT 'ITEM'
        CHECK(line_type IN ('ITEM','SERVICE','FREIGHT','TAX','CHARGE','ALLOCATION')),
    quantity DECIMAL(18,4) DEFAULT 1,
    unit_of_measure TEXT,
    unit_price_cents INTEGER NOT NULL DEFAULT 0,
    line_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Source entity accounting
    source_account_id TEXT NOT NULL,          -- Revenue/asset account for source entity
    source_department TEXT,
    source_project_id TEXT,

    -- Target entity accounting
    target_account_id TEXT NOT NULL,          -- Expense/asset account for target entity
    target_department TEXT,
    target_project_id TEXT,

    -- Item reference (if goods transfer)
    item_id TEXT,
    item_code TEXT,

    -- Cost basis for transfer pricing
    cost_cents INTEGER NOT NULL DEFAULT 0,
    markup_applied_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (transaction_id) REFERENCES intercompany_transactions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, transaction_id, line_number)
);

CREATE INDEX idx_ic_trans_lines_tenant_txn ON intercompany_transaction_lines(tenant_id, transaction_id);
CREATE INDEX idx_ic_trans_lines_tenant_source ON intercompany_transaction_lines(tenant_id, source_account_id);
CREATE INDEX idx_ic_trans_lines_tenant_target ON intercompany_transaction_lines(tenant_id, target_account_id);
```

### 2.5 Intercompany Reconciliation
```sql
CREATE TABLE intercompany_reconciliations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    reconciliation_number TEXT NOT NULL,      -- IC-REC-2024-00001
    from_entity_id TEXT NOT NULL,
    to_entity_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    reconciliation_date TEXT NOT NULL,

    -- Receivable balance (from_entity perspective)
    receivable_opening_cents INTEGER NOT NULL DEFAULT 0,
    receivable_transactions_cents INTEGER NOT NULL DEFAULT 0,
    receivable_closing_cents INTEGER NOT NULL DEFAULT 0,

    -- Payable balance (to_entity perspective)
    payable_opening_cents INTEGER NOT NULL DEFAULT 0,
    payable_transactions_cents INTEGER NOT NULL DEFAULT 0,
    payable_closing_cents INTEGER NOT NULL DEFAULT 0,

    -- Variance
    variance_cents INTEGER NOT NULL DEFAULT 0,
    variance_cause TEXT,
    is_balanced INTEGER NOT NULL DEFAULT 0,

    -- Resolution
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','MATCHED','VARIANCE','RESOLVED','POSTED')),
    resolution_notes TEXT,
    resolved_by TEXT,
    resolved_at TEXT,

    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (from_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    FOREIGN KEY (to_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, reconciliation_number)
);

CREATE INDEX idx_ic_recon_tenant_entities ON intercompany_reconciliations(tenant_id, from_entity_id, to_entity_id);
CREATE INDEX idx_ic_recon_tenant_period ON intercompany_reconciliations(tenant_id, period_name);
CREATE INDEX idx_ic_recon_tenant_status ON intercompany_reconciliations(tenant_id, status);

CREATE TABLE intercompany_reconciliation_matches (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    reconciliation_id TEXT NOT NULL,
    receivable_transaction_id TEXT,           -- IC transaction showing as receivable
    payable_transaction_id TEXT,              -- IC transaction showing as payable
    match_type TEXT NOT NULL DEFAULT 'AUTO'
        CHECK(match_type IN ('AUTO','MANUAL','ONE_TO_MANY','MANY_TO_ONE')),
    match_amount_cents INTEGER NOT NULL,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    is_full_match INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (reconciliation_id) REFERENCES intercompany_reconciliations(id) ON DELETE CASCADE,
    FOREIGN KEY (receivable_transaction_id) REFERENCES intercompany_transactions(id) ON DELETE SET NULL,
    FOREIGN KEY (payable_transaction_id) REFERENCES intercompany_transactions(id) ON DELETE SET NULL
);

CREATE INDEX idx_ic_recon_matches_tenant ON intercompany_reconciliation_matches(tenant_id, reconciliation_id);
```

### 2.6 Intercompany Elimination Rules
```sql
CREATE TABLE intercompany_elimination_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    description TEXT,

    -- Which entities to eliminate
    from_entity_id TEXT,
    to_entity_id TEXT,
    elimination_scope TEXT NOT NULL DEFAULT 'ALL_PAIRS'
        CHECK(elimination_scope IN ('ALL_PAIRS','SPECIFIC_PAIR','BY_SEGMENT')),

    -- Accounts to eliminate
    from_account_pattern TEXT,                -- GL account pattern or segment filter
    to_account_pattern TEXT,
    from_account_id TEXT,                     -- Specific account override
    to_account_id TEXT,                       -- Specific account override

    -- Elimination posting
    elimination_debit_account_id TEXT NOT NULL,  -- Offset account for elimination
    elimination_credit_account_id TEXT NOT NULL,
    elimination_method TEXT NOT NULL DEFAULT 'NET'
        CHECK(elimination_method IN ('NET','GROSS')),

    -- Trigger
    auto_generate INTEGER NOT NULL DEFAULT 1, -- Auto-generate on period close
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_ic_elim_rules_tenant_status ON intercompany_elimination_rules(tenant_id, status);
```

### 2.7 Intercompany Agreements
```sql
CREATE TABLE intercompany_agreements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_number TEXT NOT NULL,           -- IC-AGR-2024-00001
    agreement_name TEXT NOT NULL,
    agreement_type TEXT NOT NULL
        CHECK(agreement_type IN ('SALE','SERVICE','LOAN','ROYALTY','MANAGEMENT_FEE','COST_SHARING','TRANSFER_PRICING')),
    description TEXT,

    -- Parties
    from_entity_id TEXT NOT NULL,
    to_entity_id TEXT NOT NULL,

    -- Transfer pricing terms
    pricing_method TEXT NOT NULL DEFAULT 'ARM_LENGTH'
        CHECK(pricing_method IN ('ARM_LENGTH','COST_PLUS','MARKET_BASED','FIXED_MARKUP','NEGOTIATED')),
    markup_percent REAL DEFAULT 0,
    cost_plus_percent REAL DEFAULT 0,
    fixed_price_cents INTEGER DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Schedule (recurring agreements)
    billing_cycle TEXT
        CHECK(billing_cycle IN ('ONE_TIME','MONTHLY','QUARTERLY','ANNUALLY','AS_NEEDED')),

    -- Amount limits
    minimum_amount_cents INTEGER,
    maximum_amount_cents INTEGER,

    -- Dates
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    auto_renew INTEGER NOT NULL DEFAULT 0,

    -- Approval
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','EXPIRED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (from_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    FOREIGN KEY (to_entity_id) REFERENCES intercompany_entities(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, agreement_number)
);

CREATE INDEX idx_ic_agreements_tenant_entities ON intercompany_agreements(tenant_id, from_entity_id, to_entity_id);
CREATE INDEX idx_ic_agreements_tenant_status ON intercompany_agreements(tenant_id, status);
CREATE INDEX idx_ic_agreements_tenant_type ON intercompany_agreements(tenant_id, agreement_type);
```

### 2.8 Intercompany Number Sequences
```sql
CREATE TABLE ic_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_type TEXT NOT NULL,              -- 'TRANSACTION', 'RECONCILIATION', 'AGREEMENT'
    prefix TEXT NOT NULL,                     -- 'IC-TXN-', 'IC-REC-', 'IC-AGR-'
    current_value INTEGER NOT NULL DEFAULT 0,
    fiscal_year INTEGER NOT NULL,
    UNIQUE(tenant_id, sequence_type, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Intercompany Entities
```
GET    /api/v1/ic/entities                        Permission: ic.entities.read
POST   /api/v1/ic/entities                        Permission: ic.entities.create
GET    /api/v1/ic/entities/{id}                   Permission: ic.entities.read
PUT    /api/v1/ic/entities/{id}                   Permission: ic.entities.update
PATCH  /api/v1/ic/entities/{id}/status            Permission: ic.entities.update
DELETE /api/v1/ic/entities/{id}                   Permission: ic.entities.delete
GET    /api/v1/ic/entities/{id}/balances          Permission: ic.entities.read
```

### 3.2 Intercompany Relationships
```
GET    /api/v1/ic/relationships                   Permission: ic.relationships.read
POST   /api/v1/ic/relationships                   Permission: ic.relationships.create
GET    /api/v1/ic/relationships/{id}              Permission: ic.relationships.read
PUT    /api/v1/ic/relationships/{id}              Permission: ic.relationships.update
DELETE /api/v1/ic/relationships/{id}              Permission: ic.relationships.delete
```

### 3.3 Intercompany Transactions
```
GET    /api/v1/ic/transactions                    Permission: ic.transactions.read
POST   /api/v1/ic/transactions                    Permission: ic.transactions.create
GET    /api/v1/ic/transactions/{id}               Permission: ic.transactions.read
PUT    /api/v1/ic/transactions/{id}               Permission: ic.transactions.update
POST   /api/v1/ic/transactions/{id}/submit        Permission: ic.transactions.create
POST   /api/v1/ic/transactions/{id}/approve       Permission: ic.transactions.approve
POST   /api/v1/ic/transactions/{id}/post          Permission: ic.transactions.post
POST   /api/v1/ic/transactions/{id}/cancel        Permission: ic.transactions.update
POST   /api/v1/ic/transactions/{id}/settle        Permission: ic.transactions.settle
GET    /api/v1/ic/transactions/{id}/lines         Permission: ic.transactions.read
POST   /api/v1/ic/transactions/{id}/lines         Permission: ic.transactions.create
```

### 3.4 Create Transaction Request
```json
{
  "transaction_type": "SALE",
  "source_entity_id": "0192...",
  "target_entity_id": "0192...",
  "description": "Intercompany sale of raw materials",
  "transaction_date": "2024-03-15",
  "currency_code": "USD",
  "pricing_method": "COST_PLUS",
  "markup_percent": 15.0,
  "lines": [
    {
      "description": "Steel coils - Grade A",
      "line_type": "ITEM",
      "quantity": 100,
      "unit_of_measure": "KG",
      "unit_price_cents": 5000,
      "cost_cents": 4000,
      "source_account_id": "0192...revenue",
      "target_account_id": "0192...expense",
      "item_id": "0192...item",
      "item_code": "RAW-STEEL-A"
    }
  ]
}
```

### 3.5 Intercompany Reconciliation
```
GET    /api/v1/ic/reconciliations                 Permission: ic.reconciliations.read
POST   /api/v1/ic/reconciliations                 Permission: ic.reconciliations.create
GET    /api/v1/ic/reconciliations/{id}            Permission: ic.reconciliations.read
POST   /api/v1/ic/reconciliations/{id}/match      Permission: ic.reconciliations.create
POST   /api/v1/ic/reconciliations/{id}/auto-match Permission: ic.reconciliations.create
POST   /api/v1/ic/reconciliations/{id}/resolve    Permission: ic.reconciliations.update
POST   /api/v1/ic/reconciliations/{id}/post       Permission: ic.reconciliations.post
GET    /api/v1/ic/reconciliations/{id}/matches    Permission: ic.reconciliations.read
```

### 3.6 Elimination Rules
```
GET    /api/v1/ic/elimination-rules               Permission: ic.eliminations.read
POST   /api/v1/ic/elimination-rules               Permission: ic.eliminations.create
GET    /api/v1/ic/elimination-rules/{id}          Permission: ic.eliminations.read
PUT    /api/v1/ic/elimination-rules/{id}          Permission: ic.eliminations.update
DELETE /api/v1/ic/elimination-rules/{id}          Permission: ic.eliminations.delete
POST   /api/v1/ic/elimination-rules/{id}/generate Permission: ic.eliminations.create
```

### 3.7 Intercompany Agreements
```
GET    /api/v1/ic/agreements                      Permission: ic.agreements.read
POST   /api/v1/ic/agreements                      Permission: ic.agreements.create
GET    /api/v1/ic/agreements/{id}                 Permission: ic.agreements.read
PUT    /api/v1/ic/agreements/{id}                 Permission: ic.agreements.update
POST   /api/v1/ic/agreements/{id}/activate        Permission: ic.agreements.update
POST   /api/v1/ic/agreements/{id}/terminate       Permission: ic.agreements.update
```

### 3.8 Reports
```
GET /api/v1/ic/reports/balances                    Permission: ic.reports.view
  ?filter[from_entity_id]={id}
  ?filter[to_entity_id]={id}
  ?filter[period_name]=Mar-2024
  ?filter[status]=POSTED

GET /api/v1/ic/reports/outstanding                 Permission: ic.reports.view
  ?filter[as_of_date]=2024-03-31
  ?filter[entity_id]={id}

GET /api/v1/ic/reports/reconciliation-summary      Permission: ic.reports.view
GET /api/v1/ic/reports/elimination-preview          Permission: ic.reports.view
  ?filter[period_name]=Mar-2024

GET /api/v1/ic/reports/netting-summary              Permission: ic.reports.view
```

---

## 4. Business Rules

### 4.1 Intercompany Transaction Creation
- Source and target entity MUST be different within the same tenant
- An active intercompany relationship MUST exist between the two entities
- Transaction number auto-generated: `IC-TXN-{YYYY}-{NNNNN}`
- At least one line is required to create a transaction
- Total = SUM of line total_cents across all lines
- If `pricing_method = COST_PLUS`: `unit_price = cost_basis * (1 + cost_plus_percent / 100)`
- If `pricing_method = FIXED_MARKUP`: `unit_price = cost_basis + (cost_basis * markup_percent / 100)`

### 4.2 Automatic Paired Journal Entries
On posting, IC creates two paired GL journal entries:

**Source Entity Journal (seller/lender):**
```
Debit:  IC Receivable (source_entity.ic_receivable_account_id) → total_cents
Credit: IC Revenue/Asset (line.source_account_id) → line.total_cents per line
```

**Target Entity Journal (buyer/borrower):**
```
Debit:  IC Expense/Asset (line.target_account_id) → line.total_cents per line
Credit: IC Payable (target_entity.ic_payable_account_id) → total_cents
```

- Both journals are created atomically via GL gRPC calls
- Journal source is set to `'IC'` in the GL
- If either journal creation fails, the entire posting is rolled back
- Journals reference each other via GL reference fields

### 4.3 Intercompany Reconciliation
1. Reconciliation compares IC receivable (source entity) vs IC payable (target entity) for a period
2. `receivable_closing = receivable_opening + receivable_transactions`
3. `payable_closing = payable_opening + payable_transactions`
4. `variance = receivable_closing - payable_closing` (in common currency)
5. Auto-match attempts to pair transactions by:
   - Exact amount match within tolerance
   - Matching transaction dates (within configurable days)
   - Same source/target entity pair
6. Status flow: `DRAFT → IN_REVIEW → MATCHED → RESOLVED → POSTED`
7. `MATCHED` when variance = 0; `VARIANCE` when variance != 0
8. Unresolved variances require manual journal adjustment before posting

### 4.4 Elimination Entries for Consolidated Reporting
- Elimination rules define how to zero out IC balances for consolidation
- On elimination journal generation:
  ```
  Debit:  elimination_credit_account_id (reverse IC receivable)
  Credit: elimination_debit_account_id (reverse IC payable)
  ```
- Elimination entries are created as separate GL journals with source `'IC_ELIM'`
- Net method: single elimination entry per pair netting receivable vs payable
- Gross method: separate entries for each side (receivable reversal and payable reversal)
- Revenue/expense eliminations zero out IC revenue and corresponding IC expense
- Auto-generation triggered on period close or on-demand via API

### 4.5 Transfer Pricing Support
- **Arm's Length:** Price determined as if entities were independent parties (manual entry)
- **Cost Plus:** `selling_price = cost * (1 + cost_plus_percent / 100)`
- **Market Based:** Price pulled from external market data reference
- **Fixed Markup:** `selling_price = cost + (cost * markup_percent / 100)`
- **Negotiated:** Manually agreed price, audit trail required
- Transfer pricing method is set at the relationship or agreement level
- Method can be overridden per transaction (with approval)
- All transfer pricing decisions are audited for tax compliance

### 4.6 Multi-Currency Intercompany
- Transaction can be in any currency different from both entities' base currencies
- Three currency layers are tracked:
  - Transaction currency (the deal currency)
  - Source entity functional currency
  - Target entity functional currency
- Exchange rates stored on each transaction
- If source and target have different functional currencies, both conversions are tracked
- Reconciliation converts both sides to a common currency for matching
- Currency gains/losses on IC balances are tracked separately
- On settlement, any exchange rate difference creates a realized gain/loss entry

### 4.7 Intercompany Matching and Settlement
- **Netting:** Offset IC receivable against IC payable between entity pairs
- **Payment:** Create AP invoice in target entity and AR invoice in source entity
- **Clearing:** Post through a central clearing account
- **Auto:** System selects settlement method based on configured rules
- Settlement steps:
  1. Identify matched IC transactions ready for settlement
  2. Calculate net amount between entity pair
  3. Generate settlement entries (AP/AR or netting journal)
  4. Update transaction status to SETTLED
  5. Create corresponding AP/AR invoices via gRPC calls

### 4.8 Status Flow
```
DRAFT → SUBMITTED → APPROVED → POSTED → SETTLED
                                       ↓
                                   CANCELLED
```
- **DRAFT:** Editable, lines can be added/removed
- **SUBMITTED:** Locked for editing, pending approval
- **APPROVED:** Approved by authorized user, ready for posting
- **POSTED:** Paired GL journals created, amounts are immutable
- **SETTLED:** Payment/netting completed between entities
- **CANCELLED:** Can only cancel from DRAFT or SUBMITTED status
- Posted transactions can be reversed (creates reversing IC transaction with paired journals)

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.ic.v1;

service IntercompanyService {
    // Entity operations
    rpc GetEntity(GetEntityRequest) returns (GetEntityResponse);
    rpc GetEntitiesByTenant(GetEntitiesByTenantRequest) returns (GetEntitiesByTenantResponse);

    // Transaction operations
    rpc CreateTransaction(CreateTransactionRequest) returns (CreateTransactionResponse);
    rpc GetTransaction(GetTransactionRequest) returns (GetTransactionResponse);
    rpc PostTransaction(PostTransactionRequest) returns (PostTransactionResponse);
    rpc SettleTransaction(SettleTransactionRequest) returns (SettleTransactionResponse);

    // Reconciliation operations
    rpc CreateReconciliation(CreateReconciliationRequest) returns (CreateReconciliationResponse);
    rpc AutoMatchReconciliation(AutoMatchReconciliationRequest) returns (AutoMatchReconciliationResponse);
    rpc GetReconciliationStatus(GetReconciliationStatusRequest) returns (GetReconciliationStatusResponse);

    // Elimination operations
    rpc GenerateEliminationEntries(GenerateEliminationEntriesRequest) returns (GenerateEliminationEntriesResponse);
    rpc GetEliminationPreview(GetEliminationPreviewRequest) returns (GetEliminationPreviewResponse);

    // Balance operations
    rpc GetIntercompanyBalance(GetIntercompanyBalanceRequest) returns (GetIntercompanyBalanceResponse);
    rpc GetOutstandingTransactions(GetOutstandingTransactionsRequest) returns (GetOutstandingTransactionsResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 GL Service (gl-service)
IC calls GL via gRPC to create paired journal entries:

**Transaction Posting → GL Journals:**
```
Source Journal (IC Receivable):
  Debit:  IC Receivable account → transaction.total_cents
  Credit: Revenue/Asset account → line.total_cents (per line)

Target Journal (IC Payable):
  Debit:  Expense/Asset account → line.total_cents (per line)
  Credit: IC Payable account → transaction.total_cents
```

**Elimination Entry → GL Journal:**
```
Debit:  Elimination credit account → eliminated amount
Credit: Elimination debit account → eliminated amount
```

**Settlement → GL Journal:**
```
Debit:  IC Payable → settled amount
Credit: IC Receivable → settled amount
```

### 6.2 AP Service (ap-service)
When settlement method is PAYMENT or AUTO resolves to payment:
- IC calls AP gRPC to create an IC AP invoice in the target entity
- AP invoice references the IC transaction as source

### 6.3 AR Service (ar-service)
When settlement method is PAYMENT or AUTO resolves to payment:
- IC calls AR gRPC to create an IC AR invoice in the source entity
- AR invoice references the IC transaction as source

### 6.4 Event Subscription
IC subscribes to:
| Event | Source | Action |
|-------|--------|--------|
| `gl.period.closed` | GL | Trigger auto-reconciliation and elimination entries |
| `ap.invoice.posted` | AP | Match to IC transaction if IC-related |
| `ar.invoice.posted` | AR | Match to IC transaction if IC-related |

---

## 7. Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `ic.transaction.created` | IC transaction created | transaction_id, transaction_number, source_entity_id, target_entity_id, tenant_id |
| `ic.transaction.submitted` | IC transaction submitted for approval | transaction_id, tenant_id |
| `ic.transaction.approved` | IC transaction approved | transaction_id, tenant_id |
| `ic.transaction.posted` | IC transaction posted (paired journals created) | transaction_id, source_gl_journal_id, target_gl_journal_id, totals |
| `ic.transaction.settled` | IC transaction settled between entities | transaction_id, settlement_method, settlement_date |
| `ic.transaction.cancelled` | IC transaction cancelled | transaction_id, tenant_id |
| `ic.reconciliation.created` | Reconciliation created | reconciliation_id, from_entity_id, to_entity_id, period_name |
| `ic.reconciliation.matched` | Reconciliation auto-matched | reconciliation_id, match_count, variance_cents |
| `ic.reconciliation.posted` | Reconciliation posted | reconciliation_id, period_name |
| `ic.elimination.generated` | Elimination entries generated | elimination_journal_id, period_name, entity_pair |
| `ic.agreement.created` | IC agreement created | agreement_id, agreement_number, from_entity_id, to_entity_id |
| `ic.agreement.activated` | IC agreement activated | agreement_id |
| `ic.agreement.expired` | IC agreement expired (effective_to reached) | agreement_id |
| `ic.settlement.completed` | Settlement completed between entity pair | source_entity_id, target_entity_id, net_amount_cents |

---

## 8. Migrations

### Migration Order for ic-service:
1. V001: `intercompany_entities`
2. V002: `intercompany_relationships`
3. V003: `ic_sequences`
4. V004: `intercompany_transactions`
5. V005: `intercompany_transaction_lines`
6. V006: `intercompany_reconciliations`
7. V007: `intercompany_reconciliation_matches`
8. V008: `intercompany_elimination_rules`
9. V009: `intercompany_agreements`
10. V010: Triggers for `updated_at`
11. V011: Seed data (default sequences, sample elimination templates)
