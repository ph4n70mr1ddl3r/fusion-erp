# 04 - Database Design Specification

## 1. SQLite Configuration

### 1.1 Required PRAGMAs (Applied on Every Connection)
```sql
PRAGMA journal_mode = WAL;           -- Write-Ahead Logging for concurrent reads
PRAGMA foreign_keys = ON;            -- Enforce referential integrity
PRAGMA busy_timeout = 5000;          -- Wait up to 5s for locked database
PRAGMA synchronous = NORMAL;         -- Balance safety/performance with WAL
PRAGMA mmap_size = 268435456;        -- 256MB memory-mapped I/O
PRAGMA cache_size = -64000;          -- 64MB page cache
PRAGMA temp_store = MEMORY;          -- In-memory temp tables
PRAGMA locking_mode = NORMAL;        -- Normal locking (allows shared access)
```

### 1.2 Connection Pool Configuration
- **Pool library:** r2d2 with r2d2_sqlite
- **Max pool size:** 8 connections per service
- **Min idle:** 2 connections
- **Connection timeout:** 30 seconds
- **Idle timeout:** 600 seconds (10 minutes)
- **Write operations** MUST use `tokio::task::spawn_blocking` to avoid blocking the async runtime

### 1.3 Database File Layout
```
data/
├── auth.db
├── gl.db
├── ap.db
├── ar.db
├── fa.db
├── cm.db
├── proc.db
├── inv.db
├── om.db
├── mfg.db
├── pm.db
├── workflow.db
└── report.db
```

Each service owns exactly one `.db` file. No cross-service database access.

---

## 2. Naming Conventions

### 2.1 Table Naming
- snake_case, plural: `gl_accounts`, `journal_entries`, `invoice_lines`
- Service prefix not needed (each service has its own database)
- Junction tables: `{table_a}_{table_b}`: `invoice_line_tags`, `order_item_attributes`

### 2.2 Column Naming
- snake_case: `tenant_id`, `account_code`, `created_at`
- Primary key: `id`
- Foreign keys: `{referenced_table_singular}_id`: `account_id`, `journal_id`
- Boolean columns: `is_{adjective}`: `is_active`, `is_postable`, `is_reconciled`
- Date columns: `{name}_date`: `journal_date`, `due_date`, `posting_date`
- Timestamp columns: `{name}_at`: `created_at`, `updated_at`, `posted_at`
- Monetary columns: `{name}_cents`: `amount_cents`, `tax_cents`, `debit_cents`
- Count columns: `{name}_count`: `line_count`, `quantity`

### 2.3 Index Naming
- Format: `idx_{table}_{column1}_{column2}`
- Example: `idx_journal_entries_tenant_id_status`, `idx_gl_balances_account_id_period`

### 2.4 Constraint Naming
- Unique: `uq_{table}_{columns}`: `uq_gl_accounts_tenant_code`
- Check: `ck_{table}_{condition}`: `ck_journal_lines_debit_credit`

---

## 3. Standard Columns

### 3.1 Required Columns (Every Table MUST Have)
| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY NOT NULL | UUIDv7 as string |
| `tenant_id` | TEXT | NOT NULL | Tenant identifier |
| `created_at` | TEXT | NOT NULL DEFAULT (datetime('now')) | ISO 8601 UTC timestamp |
| `updated_at` | TEXT | NOT NULL DEFAULT (datetime('now')) | ISO 8601 UTC timestamp |
| `created_by` | TEXT | NOT NULL | User ID who created |
| `updated_by` | TEXT | NOT NULL | User ID who last updated |
| `version` | INTEGER | NOT NULL DEFAULT 1 | Optimistic locking version |
| `is_active` | INTEGER | NOT NULL DEFAULT 1 | Soft delete flag (1=active, 0=deleted) |

### 3.2 Standard Columns Rust Mapping
```rust
pub struct StandardColumns {
    pub id: String,
    pub tenant_id: String,
    pub created_at: String,
    pub updated_at: String,
    pub created_by: String,
    pub updated_by: String,
    pub version: i64,
    pub is_active: bool,
}
```

---

## 4. Multi-Tenancy Data Isolation

### 4.1 Strategy: Shared Database, Shared Schema, Discriminator Column
- Every table includes `tenant_id` column
- Every query MUST include `WHERE tenant_id = ?` as the first condition
- Composite indexes MUST include `tenant_id` as the leftmost column
- No cross-tenant data access is permitted at the application layer

### 4.2 Tenant Enforcement
- Auth middleware extracts `tenant_id` from JWT and injects into request context
- Repository layer MUST accept `tenant_id` as the first parameter for all queries
- Write operations MUST set `tenant_id` from the authenticated context
- Repository trait design enforces tenant scoping:

```rust
pub trait Repository<T> {
    fn find_by_id(&self, tenant_id: &str, id: &str) -> FusionResult<Option<T>>;
    fn find_all(&self, tenant_id: &str, filter: &Filter) -> FusionResult<Vec<T>>;
    fn create(&self, tenant_id: &str, entity: &NewT) -> FusionResult<T>;
    fn update(&self, tenant_id: &str, entity: &T) -> FusionResult<T>;
    fn soft_delete(&self, tenant_id: &str, id: &str) -> FusionResult<()>;
}
```

### 4.3 Index Pattern for Multi-Tenancy
All indexes MUST start with `tenant_id`:
```sql
CREATE INDEX idx_gl_accounts_tenant_code ON gl_accounts(tenant_id, account_code);
CREATE INDEX idx_journal_entries_tenant_date ON journal_entries(tenant_id, journal_date);
```

---

## 5. Monetary Values

### 5.1 Storage Rules
- MUST be stored as INTEGER representing minor units (cents)
- Column name MUST end with `_cents`: `amount_cents`, `tax_cents`, `discount_cents`
- Currency code stored in adjacent column: `currency_code` TEXT (ISO 4217, e.g., "USD", "EUR")
- MUST NOT use REAL, FLOAT, or DOUBLE for monetary values
- Application layer handles formatting for display

### 5.2 Example
```sql
amount_cents INTEGER NOT NULL,          -- $123.45 stored as 12345
currency_code TEXT NOT NULL DEFAULT 'USD',
```

### 5.3 Arithmetic Rules
- All arithmetic done in cents (integers) — no rounding errors
- Currency conversion: multiply then round to minor units
- Tax calculation: apply rate to cents, round at line level
- Totals: sum of line-level cent values

---

## 6. Migration Framework

### 6.1 Migration File Naming
```
migrations/{service}/
├── V001_create_gl_accounts.up.sql
├── V001_create_gl_accounts.down.sql
├── V002_create_gl_journals.up.sql
├── V002_create_gl_journals.down.sql
├── V003_create_gl_periods.up.sql
├── V003_create_gl_periods.down.sql
└── ...
```

### 6.2 Migration Tracking Table
```sql
CREATE TABLE IF NOT EXISTS _migrations (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    applied_at TEXT NOT NULL DEFAULT (datetime('now')),
    checksum TEXT NOT NULL,
    execution_time_ms INTEGER
);
```

### 6.3 Migration Rules
- Migrations MUST be numbered sequentially (V001, V002, V003, ...)
- Each migration MUST have both `.up.sql` and `.down.sql` files
- Up migrations MUST be idempotent where possible (use `IF NOT EXISTS`)
- Down migrations MUST reverse the up migration completely
- Migrations MUST NOT reference data from other services
- Migrations run automatically on service startup
- Migration files embedded in binary via `include_str!`

### 6.4 Migration Template
```sql
-- V{NNN}_{description}.up.sql
-- Description: Brief description of what this migration does
-- Author: AI Agent
-- Date: 2024-01-15

CREATE TABLE IF NOT EXISTS example_table (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    -- ... columns ...
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_example_table_tenant_id ON example_table(tenant_id, id);
```

---

## 7. Index Strategy

### 7.1 Mandatory Indexes
1. Every foreign key column MUST have an index
2. Every column used frequently in WHERE clauses MUST have an index
3. Every unique natural key MUST have a UNIQUE index (composite with tenant_id)
4. All indexes MUST include tenant_id as the first column (multi-tenancy)

### 7.2 Index Types
```sql
-- Single-column index
CREATE INDEX idx_journal_lines_journal_id ON journal_lines(tenant_id, journal_id);

-- Composite index for common queries
CREATE INDEX idx_journal_entries_tenant_status_date
    ON journal_entries(tenant_id, status, journal_date);

-- Unique constraint (natural key)
CREATE UNIQUE INDEX uq_gl_accounts_tenant_code
    ON gl_accounts(tenant_id, account_code) WHERE is_active = 1;

-- Partial index (only active records)
CREATE INDEX idx_active_suppliers
    ON suppliers(tenant_id, supplier_name) WHERE is_active = 1;
```

### 7.3 Covering Indexes
For frequently accessed column sets, use covering indexes:
```sql
CREATE INDEX idx_gl_balances_covering ON gl_balances(tenant_id, account_id, period_name, balance_type, beginning_balance_cents, ending_balance_cents);
```

---

## 8. Common Table Patterns

### 8.1 Master Data Table (e.g., gl_accounts)
```sql
CREATE TABLE gl_accounts (
    -- Identity
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,

    -- Business fields
    account_code TEXT NOT NULL,
    account_name TEXT NOT NULL,
    account_type TEXT NOT NULL CHECK(account_type IN ('ASSET','LIABILITY','EQUITY','REVENUE','EXPENSE')),
    account_category TEXT,
    parent_account_id TEXT,
    description TEXT,
    is_postable INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    -- Standard columns
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    -- Constraints
    FOREIGN KEY (parent_account_id) REFERENCES gl_accounts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, account_code)
);

CREATE INDEX idx_gl_accounts_tenant_type ON gl_accounts(tenant_id, account_type);
CREATE INDEX idx_gl_accounts_tenant_parent ON gl_accounts(tenant_id, parent_account_id);
```

### 8.2 Transaction Header + Lines Pattern (e.g., journal_entries + journal_lines)
```sql
CREATE TABLE journal_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journal_number TEXT NOT NULL,       -- Auto-generated: JE-2024-00001
    journal_batch_id TEXT,
    journal_source TEXT NOT NULL DEFAULT 'MANUAL',
    description TEXT NOT NULL,
    journal_date TEXT NOT NULL,
    period_name TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','POSTED','REVERSED')),
    posting_date TEXT,
    reversal_of_id TEXT,
    reference TEXT,
    reference_type TEXT,
    reference_id TEXT,

    -- Totals (denormalized for performance)
    total_debit_cents INTEGER NOT NULL DEFAULT 0,
    total_credit_cents INTEGER NOT NULL DEFAULT 0,
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

CREATE TABLE journal_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journal_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,

    -- Account reference
    account_id TEXT NOT NULL,

    -- Amounts
    description TEXT,
    entered_debit_cents INTEGER NOT NULL DEFAULT 0,
    entered_credit_cents INTEGER NOT NULL DEFAULT 0,
    accounted_debit_cents INTEGER NOT NULL DEFAULT 0,
    accounted_credit_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,  -- Only place REAL is acceptable (exchange rates)

    -- Reference (polymorphic)
    reference_type TEXT,
    reference_id TEXT,

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

### 8.3 Reference/Lookup Table (e.g., currencies)
```sql
CREATE TABLE currencies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    code TEXT NOT NULL,                 -- ISO 4217: USD, EUR, GBP
    name TEXT NOT NULL,
    symbol TEXT NOT NULL,
    minor_units INTEGER NOT NULL DEFAULT 2,  -- Cents, pence, etc.
    is_active INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, code)
);
```

### 8.4 Party Table (suppliers, customers)
```sql
CREATE TABLE suppliers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_number TEXT NOT NULL,      -- Auto-generated: SUP-00001
    supplier_name TEXT NOT NULL,
    legal_name TEXT,
    tax_identification_number TEXT,
    supplier_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(supplier_type IN ('STANDARD','EMPLOYEE','ONE_TIME')),
    payment_terms_id TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    payment_method TEXT DEFAULT 'CHECK'
        CHECK(payment_method IN ('CHECK','WIRE','ACH','EFT')),

    -- Address
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state TEXT,
    postal_code TEXT,
    country TEXT,

    -- Contact
    email TEXT,
    phone TEXT,
    fax TEXT,

    -- Status
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ON_HOLD')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_number)
);

CREATE INDEX idx_suppliers_tenant_name ON suppliers(tenant_id, supplier_name);
CREATE INDEX idx_suppliers_tenant_status ON suppliers(tenant_id, status);
```

---

## 9. Data Integrity

### 9.1 Foreign Key Rules
- All foreign keys MUST use `ON DELETE RESTRICT` (prevent orphaning)
- Soft delete preferred over hard delete
- Application layer handles cascade logic for soft deletes

### 9.2 CHECK Constraints
- Use for simple business rules enforced at the database level
- Examples:
  - `CHECK(entered_debit_cents >= 0)`
  - `CHECK(status IN ('DRAFT','APPROVED','POSTED','REVERSED'))`
  - `CHECK(start_date <= end_date)`
  - `CHECK(quantity >= 0)`

### 9.3 UNIQUE Constraints
- Natural keys MUST have unique constraints (composite with tenant_id)
- Examples:
  - `UNIQUE(tenant_id, account_code)`
  - `UNIQUE(tenant_id, journal_number)`
  - `UNIQUE(tenant_id, period_name)`

### 9.4 Triggers
```sql
-- Auto-update updated_at timestamp
CREATE TRIGGER trg_{table}_updated_at
AFTER UPDATE ON {table}
FOR EACH ROW
BEGIN
    UPDATE {table} SET updated_at = datetime('now') WHERE id = NEW.id;
END;
```

### 9.5 Optimistic Locking
- Every UPDATE MUST include a version check:
```sql
UPDATE journal_entries
SET description = ?, updated_by = ?, version = version + 1
WHERE tenant_id = ? AND id = ? AND version = ?
```
- If affected rows = 0, return a conflict error (409)

---

## 10. Transaction Patterns

### 10.1 Read Transaction
```rust
let conn = pool.get()?;
let results = conn.query_row(
    "SELECT ... FROM table WHERE tenant_id = ?1 AND id = ?2 AND is_active = 1",
    params![tenant_id, id],
    |row| { /* map row */ }
)?;
```

### 10.2 Write Transaction
```rust
let mut conn = pool.get()?;
let tx = conn.transaction()?;

// Insert header
tx.execute("INSERT INTO journal_entries ...", params![...])?;

// Insert lines
for line in lines {
    tx.execute("INSERT INTO journal_lines ...", params![...])?;
}

// Update header totals
tx.execute("UPDATE journal_entries SET total_debit_cents = ?, ...", params![...])?;

tx.commit()?;
```

### 10.3 Transaction Scope Rules
- Keep transactions as short as possible
- MUST NOT make external API/gRPC calls within a transaction
- MUST NOT perform expensive computations within a transaction
- All validation MUST happen before the transaction starts
