# 08 - Accounts Receivable Service Specification

## 1. Domain Overview

Accounts Receivable (AR) manages customer invoices, receipt processing, incoming cash flows, credit management, and collections. Integrates with GL for journal posting, OM for invoice generation, and CM for cash receipts.

**Bounded Context:** Accounts Receivable & Customer Management
**Service Name:** `ar-service`
**Database:** `data/ar.db`
**HTTP Port:** 8012 | **gRPC Port:** 9012

---

## 2. Database Schema

### 2.1 Customers
```sql
CREATE TABLE customers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_number TEXT NOT NULL,         -- CUST-00001
    customer_name TEXT NOT NULL,
    legal_name TEXT,
    customer_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(customer_type IN ('STANDARD','EMPLOYEE','INTERNAL','ONE_TIME')),
    tax_id TEXT,
    tax_id_encrypted TEXT,
    payment_terms_id TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    credit_limit_cents INTEGER,
    credit_used_cents INTEGER NOT NULL DEFAULT 0,
    credit_available_cents INTEGER,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ON_HOLD','WRITE_OFF','BANKRUPT')),
    on_hold_reason TEXT,
    customer_group TEXT,
    sales_rep_id TEXT,
    price_list_id TEXT,

    -- Bill-to address
    bill_address_line1 TEXT, bill_address_line2 TEXT,
    bill_city TEXT, bill_state TEXT, bill_postal_code TEXT, bill_country TEXT,
    -- Ship-to address
    ship_address_line1 TEXT, ship_address_line2 TEXT,
    ship_city TEXT, ship_state TEXT, ship_postal_code TEXT, ship_country TEXT,

    -- Contact
    email TEXT, phone TEXT, fax TEXT, website TEXT,
    contact_name TEXT, contact_email TEXT, contact_phone TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, customer_number)
);

CREATE INDEX idx_customers_tenant_name ON customers(tenant_id, customer_name);
CREATE INDEX idx_customers_tenant_status ON customers(tenant_id, status);
CREATE INDEX idx_customers_tenant_group ON customers(tenant_id, customer_group);
```

### 2.2 AR Invoices (Header)
```sql
CREATE TABLE ar_invoices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    invoice_number TEXT NOT NULL,          -- AR-INV-2024-00001
    customer_id TEXT NOT NULL,
    invoice_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    accounting_date TEXT NOT NULL,
    period_name TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    payment_terms_id TEXT,
    billing_address_id TEXT,
    shipping_address_id TEXT,
    description TEXT,
    notes TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','POSTED','PARTIALLY_PAID','PAID','CANCELLED','WRITE_OFF')),
    gl_journal_id TEXT,

    -- Totals
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    freight_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    amount_paid_cents INTEGER NOT NULL DEFAULT 0,
    amount_remaining_cents INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,

    -- Source reference
    source_type TEXT DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','ORDER','PROJECT','RECURRING')),
    source_id TEXT,                        -- Order ID or project ID

    -- Dunning/collection
    dunning_level INTEGER NOT NULL DEFAULT 0,
    last_dunning_date TEXT,
    days_overdue INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_ar_invoices_tenant_customer ON ar_invoices(tenant_id, customer_id);
CREATE INDEX idx_ar_invoices_tenant_status ON ar_invoices(tenant_id, status);
CREATE INDEX idx_ar_invoices_tenant_due ON ar_invoices(tenant_id, due_date);
CREATE INDEX idx_ar_invoices_tenant_overdue ON ar_invoices(tenant_id, days_overdue) WHERE days_overdue > 0;
CREATE INDEX idx_ar_invoices_tenant_source ON ar_invoices(tenant_id, source_type, source_id);
```

### 2.3 AR Invoice Lines
```sql
CREATE TABLE ar_invoice_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    invoice_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    line_type TEXT NOT NULL DEFAULT 'ITEM'
        CHECK(line_type IN ('ITEM','TAX','FREIGHT','DISCOUNT','MISC','CHARGE')),

    description TEXT NOT NULL,
    item_id TEXT,                          -- Optional link to inventory item
    quantity INTEGER DEFAULT 1000,
    unit_of_measure TEXT,
    unit_price_cents INTEGER NOT NULL DEFAULT 0,
    line_amount_cents INTEGER NOT NULL DEFAULT 0,
    discount_percent REAL DEFAULT 0,
    discount_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_code TEXT,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Revenue account
    revenue_account_id TEXT NOT NULL,

    -- References
    sales_order_line_id TEXT,
    project_id TEXT,
    project_task_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (invoice_id) REFERENCES ar_invoices(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, invoice_id, line_number)
);

CREATE INDEX idx_ar_invoice_lines_tenant_invoice ON ar_invoice_lines(tenant_id, invoice_id);
```

### 2.4 AR Receipts
```sql
CREATE TABLE ar_receipts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    receipt_number TEXT NOT NULL,          -- AR-RCPT-2024-00001
    receipt_date TEXT NOT NULL,
    receipt_method TEXT NOT NULL
        CHECK(receipt_method IN ('CASH','CHECK','WIRE','ACH','CREDIT_CARD','OTHER')),
    customer_id TEXT NOT NULL,
    bank_account_id TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    unapplied_amount_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'UNAPPLIED'
        CHECK(status IN ('UNAPPLIED','PARTIALLY_APPLIED','APPLIED','VOID','REFUNDED')),
    reference TEXT,
    check_number TEXT,
    description TEXT,
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, receipt_number)
);

CREATE TABLE ar_receipt_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    receipt_id TEXT NOT NULL,
    invoice_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    amount_applied_cents INTEGER NOT NULL DEFAULT 0,
    discount_taken_cents INTEGER NOT NULL DEFAULT 0,
    application_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (receipt_id) REFERENCES ar_receipts(id) ON DELETE RESTRICT,
    FOREIGN KEY (invoice_id) REFERENCES ar_invoices(id) ON DELETE RESTRICT
);
```

### 2.5 Credit Memos
```sql
CREATE TABLE ar_credit_memos (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    credit_memo_number TEXT NOT NULL,      -- AR-CM-2024-00001
    customer_id TEXT NOT NULL,
    invoice_id TEXT,                       -- Optional link to original invoice
    credit_memo_date TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    reason TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','APPROVED','POSTED','APPLIED','CANCELLED')),
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT,
    FOREIGN KEY (invoice_id) REFERENCES ar_invoices(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, credit_memo_number)
);
```

---

## 3. REST API Endpoints

### 3.1 Customers
```
GET    /api/v1/ar/customers                      Permission: ar.customers.read
POST   /api/v1/ar/customers                      Permission: ar.customers.create
GET    /api/v1/ar/customers/{id}                  Permission: ar.customers.read
PUT    /api/v1/ar/customers/{id}                  Permission: ar.customers.update
PATCH  /api/v1/ar/customers/{id}/credit-limit     Permission: ar.customers.update
GET    /api/v1/ar/customers/{id}/balance          Permission: ar.customers.read
GET    /api/v1/ar/customers/{id}/statements       Permission: ar.customers.read
```

### 3.2 Invoices
```
GET    /api/v1/ar/invoices                        Permission: ar.invoices.read
POST   /api/v1/ar/invoices                        Permission: ar.invoices.create
GET    /api/v1/ar/invoices/{id}                   Permission: ar.invoices.read
PUT    /api/v1/ar/invoices/{id}                   Permission: ar.invoices.update
POST   /api/v1/ar/invoices/{id}/submit            Permission: ar.invoices.create
POST   /api/v1/ar/invoices/{id}/approve           Permission: ar.invoices.approve
POST   /api/v1/ar/invoices/{id}/post              Permission: ar.invoices.post
POST   /api/v1/ar/invoices/{id}/cancel            Permission: ar.invoices.update
GET    /api/v1/ar/invoices/{id}/lines              Permission: ar.invoices.read
POST   /api/v1/ar/invoices/{id}/lines              Permission: ar.invoices.create
```

### 3.3 Receipts
```
GET    /api/v1/ar/receipts                        Permission: ar.receipts.read
POST   /api/v1/ar/receipts                        Permission: ar.receipts.create
GET    /api/v1/ar/receipts/{id}                   Permission: ar.receipts.read
POST   /api/v1/ar/receipts/{id}/apply             Permission: ar.receipts.create
POST   /api/v1/ar/receipts/{id}/unapply           Permission: ar.receipts.update
POST   /api/v1/ar/receipts/{id}/void              Permission: ar.receipts.update
```

### 3.4 Credit Memos
```
GET    /api/v1/ar/credit-memos                    Permission: ar.credit-memos.read
POST   /api/v1/ar/credit-memos                    Permission: ar.credit-memos.create
GET    /api/v1/ar/credit-memos/{id}               Permission: ar.credit-memos.read
POST   /api/v1/ar/credit-memos/{id}/apply         Permission: ar.credit-memos.create
```

### 3.5 Reports
```
GET /api/v1/ar/reports/aging                      Permission: ar.reports.view
GET /api/v1/ar/reports/customer-ledger            Permission: ar.reports.view
GET /api/v1/ar/reports/receipts-history           Permission: ar.reports.view
```

---

## 4. Business Rules

### 4.1 Credit Management
- Before posting invoice, check customer credit limit
- If `credit_used + invoice_total > credit_limit`, block or require approval
- Credit used updated on invoice posting and receipt application

### 4.2 Invoice → GL Journal
On invoice posting:
```
Debit: AR control account → invoice.total_cents
Credit: Revenue accounts → line.total_cents (per line)
Credit: Tax payable → invoice.tax_cents
```

### 4.3 Receipt → GL Journal
On receipt application:
```
Debit: Cash/Bank account → receipt.amount_cents
Credit: AR control account → receipt.amount_cents
```

### 4.4 Days Overdue Calculation
- Updated daily or on query: `days_overdue = MAX(0, today - due_date)`
- Used for aging reports and dunning level escalation

### 4.5 Events Published
| Event | Trigger |
|-------|---------|
| `ar.invoice.created` | Invoice created |
| `ar.invoice.posted` | Invoice posted to GL |
| `ar.invoice.paid` | Invoice fully paid |
| `ar.receipt.created` | Receipt recorded |
| `ar.receipt.applied` | Receipt applied to invoice |
| `ar.credit_memo.created` | Credit memo created |
| `ar.customer.credit_exceeded` | Credit limit exceeded |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.ar.v1;

service AccountsReceivableService {
    rpc GetCustomer(GetCustomerRequest) returns (GetCustomerResponse);
    rpc GetInvoice(GetInvoiceRequest) returns (GetInvoiceResponse);
    rpc CreateInvoice(CreateInvoiceRequest) returns (CreateInvoiceResponse);
    rpc PostInvoice(PostInvoiceRequest) returns (PostInvoiceResponse);
    rpc ApplyReceipt(ApplyReceiptRequest) returns (ApplyReceiptResponse);
    rpc GetCustomerBalance(GetCustomerBalanceRequest) returns (GetCustomerBalanceResponse);
    rpc GetAging(GetAgingRequest) returns (GetAgingResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| gl-service | `PostJournal` | Post AR invoice/receipt journal entries |
| tax-service | `CalculateTax` | Calculate tax on invoice lines |
| cm-service | `RecordReceipt` | Record cash receipt in cash management |
| workflow-service | `SubmitApproval` | Submit credit memo approvals |

### 6.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| om-service | `CreateInvoice` | Generate AR invoice from sales order |
| collections-service | `GetInvoice` / `GetAging` | Read invoice data for collections |
| ic-service | `CreateInvoice` | Create intercompany AR invoice |

---

## 7. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | ar_sequences | — |
| V002 | customers | V001 |
| V003 | customer_groups | V002 |
| V004 | ar_invoices | V002 |
| V005 | ar_invoice_lines | V004 |
| V006 | ar_receipts | V002 |
| V007 | ar_receipt_applications | V004, V006 |
| V008 | ar_credit_memos | V002, V004 |
