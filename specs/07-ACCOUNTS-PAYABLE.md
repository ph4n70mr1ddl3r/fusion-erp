# 07 - Accounts Payable Service Specification

## 1. Domain Overview

Accounts Payable (AP) manages supplier invoices, payment processing, and outgoing cash flows. It integrates with GL for journal posting, Procurement for PO matching, and Cash Management for disbursements.

**Bounded Context:** Accounts Payable & Supplier Management
**Service Name:** `ap-service`
**Database:** `data/ap.db`
**HTTP Port:** 8011 | **gRPC Port:** 9011

---

## 2. Database Schema

### 2.1 Suppliers
```sql
CREATE TABLE suppliers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_number TEXT NOT NULL,
    supplier_name TEXT NOT NULL,
    legal_name TEXT,
    supplier_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(supplier_type IN ('STANDARD','EMPLOYEE','ONE_TIME')),
    tax_id TEXT,
    tax_id_encrypted TEXT,                -- AES-256-GCM encrypted
    payment_terms_id TEXT,
    payment_method TEXT NOT NULL DEFAULT 'CHECK'
        CHECK(payment_method IN ('CHECK','WIRE','ACH','EFT','INTERNAL')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    credit_limit_cents INTEGER,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ON_HOLD','BLACKLISTED')),
    on_hold_reason TEXT,
    bank_name TEXT,
    bank_account_encrypted TEXT,
    bank_routing_number TEXT,
    address_line1 TEXT, address_line2 TEXT, city TEXT, state TEXT,
    postal_code TEXT, country TEXT,
    email TEXT, phone TEXT, fax TEXT, website TEXT,
    contact_name TEXT, contact_email TEXT, contact_phone TEXT,

    -- Standard columns
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_number)
);

CREATE INDEX idx_suppliers_tenant_name ON suppliers(tenant_id, supplier_name);
CREATE INDEX idx_suppliers_tenant_status ON suppliers(tenant_id, status);
```

### 2.2 Payment Terms
```sql
CREATE TABLE payment_terms (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    name TEXT NOT NULL,                   -- "Net 30", "2/10 Net 30", "Due on Receipt"
    description TEXT,
    due_days INTEGER NOT NULL DEFAULT 30,
    discount_days INTEGER,                -- Days within which discount applies
    discount_percent REAL,                -- Early payment discount percentage
    is_system INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, name)
);
```

### 2.3 AP Invoices (Header)
```sql
CREATE TABLE ap_invoices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    invoice_number TEXT NOT NULL,         -- Our internal number: AP-INV-2024-00001
    supplier_invoice_number TEXT NOT NULL, -- Supplier's invoice number
    supplier_id TEXT NOT NULL,
    invoice_date TEXT NOT NULL,
    invoice_received_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    accounting_date TEXT NOT NULL,
    period_name TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    payment_terms_id TEXT,
    payment_method TEXT,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','POSTED','PAID','PARTIALLY_PAID','CANCELLED','VOID')),
    gl_journal_id TEXT,                   -- Link to GL journal entry

    -- Matching status (3-way match)
    match_status TEXT DEFAULT 'UNMATCHED'
        CHECK(match_status IN ('UNMATCHED','2_WAY','3_WAY','DISCREPANCY')),

    -- Totals
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    withholding_tax_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    amount_paid_cents INTEGER NOT NULL DEFAULT 0,
    amount_remaining_cents INTEGER NOT NULL DEFAULT 0,
    discount_available_cents INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,

    -- References
    purchase_order_id TEXT,
    goods_receipt_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_ap_invoices_tenant_supplier ON ap_invoices(tenant_id, supplier_id);
CREATE INDEX idx_ap_invoices_tenant_status ON ap_invoices(tenant_id, status);
CREATE INDEX idx_ap_invoices_tenant_due ON ap_invoices(tenant_id, due_date);
CREATE INDEX idx_ap_invoices_tenant_period ON ap_invoices(tenant_id, period_name);
CREATE INDEX idx_ap_invoices_tenant_po ON ap_invoices(tenant_id, purchase_order_id);
```

### 2.4 AP Invoice Lines
```sql
CREATE TABLE ap_invoice_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    invoice_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    line_type TEXT NOT NULL DEFAULT 'ITEM'
        CHECK(line_type IN ('ITEM','TAX','FREIGHT','MISC','CHARGE')),

    -- Line detail
    description TEXT NOT NULL,
    quantity INTEGER DEFAULT 1000,
    unit_of_measure TEXT,
    unit_price_cents INTEGER NOT NULL DEFAULT 0,
    line_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_code TEXT,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    withholding_tax_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Account distribution
    expense_account_id TEXT NOT NULL,

    -- Matching references
    purchase_order_line_id TEXT,
    goods_receipt_line_id TEXT,

    -- Asset tracking (if this line creates a fixed asset)
    create_asset INTEGER NOT NULL DEFAULT 0,
    asset_category_id TEXT,

    -- Project accounting
    project_id TEXT,
    project_task_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (invoice_id) REFERENCES ap_invoices(id) ON DELETE RESTRICT,
    -- expense_account_id is a logical reference to gl_accounts(id), validated via gRPC
    UNIQUE(tenant_id, invoice_id, line_number)
);

CREATE INDEX idx_ap_invoice_lines_tenant_invoice ON ap_invoice_lines(tenant_id, invoice_id);
CREATE INDEX idx_ap_invoice_lines_tenant_account ON ap_invoice_lines(tenant_id, expense_account_id);
```

### 2.5 AP Payments
```sql
CREATE TABLE ap_payments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    payment_number TEXT NOT NULL,          -- AP-PAY-2024-00001
    payment_date TEXT NOT NULL,
    payment_method TEXT NOT NULL
        CHECK(payment_method IN ('CHECK','WIRE','ACH','EFT','CASH','INTERNAL')),
    bank_account_id TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','PAID','CANCELLED','VOID')),
    gl_journal_id TEXT,

    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    check_number TEXT,
    reference TEXT,

    payment_run_id TEXT,                  -- Grouped by payment run

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, payment_number)
);

CREATE TABLE ap_payment_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    payment_id TEXT NOT NULL,
    invoice_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    amount_applied_cents INTEGER NOT NULL DEFAULT 0,
    discount_taken_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (payment_id) REFERENCES ap_payments(id) ON DELETE RESTRICT,
    FOREIGN KEY (invoice_id) REFERENCES ap_invoices(id) ON DELETE RESTRICT
);

CREATE INDEX idx_ap_payment_apps_tenant_payment ON ap_payment_applications(tenant_id, payment_id);
CREATE INDEX idx_ap_payment_apps_tenant_invoice ON ap_payment_applications(tenant_id, invoice_id);
```

### 2.6 AP Number Sequences
```sql
CREATE TABLE ap_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_type TEXT NOT NULL,           -- 'INVOICE', 'PAYMENT', 'CREDIT_MEMO'
    prefix TEXT NOT NULL,
    current_value INTEGER NOT NULL DEFAULT 0,
    fiscal_year INTEGER NOT NULL,
    UNIQUE(tenant_id, sequence_type, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Suppliers
```
GET    /api/v1/ap/suppliers                     Permission: ap.suppliers.read
POST   /api/v1/ap/suppliers                     Permission: ap.suppliers.create
GET    /api/v1/ap/suppliers/{id}                 Permission: ap.suppliers.read
PUT    /api/v1/ap/suppliers/{id}                 Permission: ap.suppliers.update
PATCH  /api/v1/ap/suppliers/{id}/status          Permission: ap.suppliers.update
DELETE /api/v1/ap/suppliers/{id}                 Permission: ap.suppliers.delete
GET    /api/v1/ap/suppliers/{id}/balance         Permission: ap.suppliers.read
```

### 3.2 Invoices
```
GET    /api/v1/ap/invoices                       Permission: ap.invoices.read
POST   /api/v1/ap/invoices                       Permission: ap.invoices.create
GET    /api/v1/ap/invoices/{id}                  Permission: ap.invoices.read
PUT    /api/v1/ap/invoices/{id}                  Permission: ap.invoices.update
POST   /api/v1/ap/invoices/{id}/submit           Permission: ap.invoices.create
POST   /api/v1/ap/invoices/{id}/approve          Permission: ap.invoices.approve
POST   /api/v1/ap/invoices/{id}/post             Permission: ap.invoices.post
POST   /api/v1/ap/invoices/{id}/cancel           Permission: ap.invoices.update
GET    /api/v1/ap/invoices/{id}/lines             Permission: ap.invoices.read
POST   /api/v1/ap/invoices/{id}/lines             Permission: ap.invoices.create
```

### 3.3 Payments
```
GET    /api/v1/ap/payments                       Permission: ap.payments.read
POST   /api/v1/ap/payments                       Permission: ap.payments.create
GET    /api/v1/ap/payments/{id}                  Permission: ap.payments.read
POST   /api/v1/ap/payments/{id}/submit           Permission: ap.payments.create
POST   /api/v1/ap/payments/{id}/approve          Permission: ap.payments.approve
POST   /api/v1/ap/payments/{id}/pay              Permission: ap.payments.create
POST   /api/v1/ap/payments/{id}/void             Permission: ap.payments.update
POST   /api/v1/ap/payments/run                   Permission: ap.payments.create
GET    /api/v1/ap/payments/run/status/{run_id}   Permission: ap.payments.read
```

### 3.4 Payment Run Request
```json
{
  "payment_date": "2024-01-31",
  "bank_account_id": "0192...",
  "payment_method": "ACH",
  "criteria": {
    "due_date_to": "2024-02-15",
    "supplier_ids": ["id1", "id2"],
    "minimum_amount_cents": 10000,
    "take_discounts": true
  }
}
```

### 3.5 Reports
```
GET /api/v1/ap/reports/aging                     Permission: ap.reports.view
  ?filter[as_of_date]=2024-01-31
  ?filter[aging_buckets]=30,60,90,120

GET /api/v1/ap/reports/supplier-ledger           Permission: ap.reports.view
GET /api/v1/ap/reports/payment-history           Permission: ap.reports.view
```

---

## 4. Business Rules

### 4.1 Invoice Processing
- Invoice number MUST be unique per tenant (internal) and per supplier (supplier_invoice_number)
- Due date calculated from invoice_date + payment_terms.due_days
- Early payment discount calculated from payment_terms.discount_days/discount_percent
- Total = subtotal + tax - withholding_tax
- amount_remaining = total - amount_paid

### 4.2 Three-Way Matching
When invoice references a PO:
1. **2-Way Match:** Invoice total matches PO total (within tolerance)
2. **3-Way Match:** Invoice matches PO AND goods receipt
3. Tolerance: configurable per tenant (default: ±$0.05 or 1%)
4. Discrepancies routed for manual approval
5. Over-receipts flagged; under-receipts auto-matched

### 4.3 Payment Processing
- Payment run selects eligible invoices based on criteria
- Payment number auto-generated: `AP-PAY-{YYYY}-{NNNNN}`
- On payment: update invoice amount_paid, status transitions to PAID or PARTIALLY_PAID
- Payment creates GL journal entry: Debit AP, Credit Cash

### 4.4 GL Integration
On invoice posting, AP calls GL gRPC to create journal:
```
For each invoice line:
  Debit: line.expense_account_id → line.total_cents
Credit: AP control account → invoice.total_cents
```

On payment:
```
Debit: AP control account → payment.total_amount_cents
Credit: Bank account → payment.total_amount_cents
```

### 4.5 Status Flow
```
Invoice: DRAFT → SUBMITTED → APPROVED → POSTED → PAID/PARTIALLY_PAID
Payment: DRAFT → SUBMITTED → APPROVED → PAID → (VOID)
```

### 4.6 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `ap.invoice.created` | Invoice created | Workflow |
| `ap.invoice.posted` | Invoice posted to GL | GL, Report |
| `ap.invoice.paid` | Payment applied | GL, CM, Report |
| `ap.payment.created` | Payment created | CM, Report |
| `ap.payment.voided` | Payment voided | GL, Report |
| `ap.supplier.created` | Supplier created | Report |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.ap.v1;

service AccountsPayableService {
    rpc GetSupplier(GetSupplierRequest) returns (GetSupplierResponse);
    rpc GetInvoice(GetInvoiceRequest) returns (GetInvoiceResponse);
    rpc CreateInvoice(CreateInvoiceRequest) returns (CreateInvoiceResponse);
    rpc PostInvoice(PostInvoiceRequest) returns (PostInvoiceResponse);
    rpc GetSupplierBalance(GetSupplierBalanceRequest) returns (GetSupplierBalanceResponse);
    rpc GetAging(GetAgingRequest) returns (GetAgingResponse);
}

// Supplier messages
message GetSupplierRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetSupplierResponse {
    Supplier data = 1;
}

message Supplier {
    string id = 1;
    string tenant_id = 2;
    string supplier_number = 3;
    string supplier_name = 4;
    string legal_name = 5;
    string supplier_type = 6;
    string tax_id = 7;
    string payment_terms_id = 8;
    string payment_method = 9;
    string currency_code = 10;
    int64 credit_limit_cents = 11;
    string status = 12;
    string on_hold_reason = 13;
    string bank_name = 14;
    string bank_routing_number = 15;
    string address_line1 = 16;
    string address_line2 = 17;
    string city = 18;
    string state = 19;
    string postal_code = 20;
    string country = 21;
    string email = 22;
    string phone = 23;
    string created_at = 24;
    string updated_at = 25;
}

// Invoice messages
message GetInvoiceRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetInvoiceResponse {
    ApInvoice data = 1;
}

message CreateInvoiceRequest {
    string tenant_id = 1;
    string supplier_invoice_number = 2;
    string supplier_id = 3;
    string invoice_date = 4;
    string invoice_received_date = 5;
    string due_date = 6;
    string accounting_date = 7;
    string period_name = 8;
    string currency_code = 9;
    double exchange_rate = 10;
    string payment_terms_id = 11;
    string payment_method = 12;
    string description = 13;
    string purchase_order_id = 14;
    string goods_receipt_id = 15;
}

message CreateInvoiceResponse {
    ApInvoice data = 1;
}

message PostInvoiceRequest {
    string tenant_id = 1;
    string id = 2;
}

message PostInvoiceResponse {
    string id = 1;
    string status = 2;
    string gl_journal_id = 3;
}

message ApInvoice {
    string id = 1;
    string tenant_id = 2;
    string invoice_number = 3;
    string supplier_invoice_number = 4;
    string supplier_id = 5;
    string invoice_date = 6;
    string invoice_received_date = 7;
    string due_date = 8;
    string accounting_date = 9;
    string period_name = 10;
    string currency_code = 11;
    double exchange_rate = 12;
    string payment_terms_id = 13;
    string payment_method = 14;
    string description = 15;
    string status = 16;
    string gl_journal_id = 17;
    string match_status = 18;
    int64 subtotal_cents = 19;
    int64 tax_cents = 20;
    int64 withholding_tax_cents = 21;
    int64 total_cents = 22;
    int64 amount_paid_cents = 23;
    int64 amount_remaining_cents = 24;
    int64 discount_available_cents = 25;
    int32 line_count = 26;
    string purchase_order_id = 27;
    string goods_receipt_id = 28;
    string created_at = 29;
    string updated_at = 30;
}

// Balance and aging messages
message GetSupplierBalanceRequest {
    string tenant_id = 1;
    string supplier_id = 2;
}

message GetSupplierBalanceResponse {
    string supplier_id = 1;
    int64 total_outstanding_cents = 2;
    int64 total_overdue_cents = 3;
    int64 current_cents = 4;
    int64 bucket_1_30_cents = 5;
    int64 bucket_31_60_cents = 6;
    int64 bucket_61_90_cents = 7;
    int64 bucket_90_plus_cents = 8;
}

message GetAgingRequest {
    string tenant_id = 1;
    string as_of_date = 2;
    repeated string aging_buckets = 3;
}

message GetAgingResponse {
    string as_of_date = 1;
    repeated AgingLine lines = 2;
}

message AgingLine {
    string supplier_id = 1;
    string supplier_name = 2;
    int64 current_cents = 3;
    int64 bucket_1_30_cents = 4;
    int64 bucket_31_60_cents = 5;
    int64 bucket_61_90_cents = 6;
    int64 bucket_90_plus_cents = 7;
    int64 total_cents = 8;
}
```
