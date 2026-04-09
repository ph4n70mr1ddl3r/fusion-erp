# 25 - Expense Management Service Specification

## 1. Domain Overview

Expense Management handles employee expense reporting, policy compliance, receipt processing, per diem calculations, mileage reimbursement, corporate card transaction matching, and expense reimbursement. It integrates with AP for reimbursement payments, GL for expense journal entries, PM for project expense allocation, and Workflow for approval routing.

**Bounded Context:** Expense Reporting & Reimbursement
**Service Name:** `expense-service`
**Database:** `data/expense.db`
**HTTP Port:** 8024 | **gRPC Port:** 9024

---

## 2. Database Schema

### 2.1 Expense Policies
```sql
CREATE TABLE expense_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_name TEXT NOT NULL,
    description TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    -- Spending limits
    daily_limit_cents INTEGER,
    weekly_limit_cents INTEGER,
    monthly_limit_cents INTEGER,
    report_max_lines INTEGER,

    -- Receipt rules
    receipt_threshold_cents INTEGER NOT NULL DEFAULT 2500,   -- Receipt required above $25.00

    -- Approval rules
    auto_approve_below_cents INTEGER,       -- Auto-approve reports below this amount
    manager_approval_required INTEGER NOT NULL DEFAULT 1,
    finance_approval_threshold_cents INTEGER,   -- Finance approval above this amount

    -- General rules
    require_purpose INTEGER NOT NULL DEFAULT 1,
    require_project INTEGER NOT NULL DEFAULT 0,
    allow_split_expenses INTEGER NOT NULL DEFAULT 1,
    allow_cash_advances INTEGER NOT NULL DEFAULT 0,
    duplicate_check_window_days INTEGER NOT NULL DEFAULT 3,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_name)
);

CREATE INDEX idx_expense_policies_tenant_status ON expense_policies(tenant_id, status);
```

### 2.2 Expense Policy Rules (per-category)
```sql
CREATE TABLE expense_policy_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_id TEXT NOT NULL,
    category_id TEXT NOT NULL,

    -- Spending limits per category
    max_amount_per_transaction_cents INTEGER,
    max_daily_amount_cents INTEGER,
    max_monthly_amount_cents INTEGER,

    -- Receipt requirements
    receipt_required INTEGER NOT NULL DEFAULT 0,
    receipt_threshold_cents INTEGER,            -- Override policy-level threshold

    -- Approval thresholds
    approval_required_above_cents INTEGER,

    -- Time restrictions
    allowed_start_time TEXT,                    -- "06:00"
    allowed_end_time TEXT,                      -- "22:00"
    allowed_days TEXT,                          -- JSON array: [1,2,3,4,5] (Mon-Fri)

    -- Restrictions
    max_transactions_per_day INTEGER,
    requires_attendees INTEGER NOT NULL DEFAULT 0,  -- For entertainment
    requires_location INTEGER NOT NULL DEFAULT 0,
    requires_project INTEGER NOT NULL DEFAULT 0,

    -- Violation action
    violation_action TEXT NOT NULL DEFAULT 'FLAG'
        CHECK(violation_action IN ('FLAG','BLOCK','WARN')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (policy_id) REFERENCES expense_policies(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, policy_id, category_id)
);

CREATE INDEX idx_expense_policy_rules_tenant_policy ON expense_policy_rules(tenant_id, policy_id);
```

### 2.3 Expense Categories
```sql
CREATE TABLE expense_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL,
    category_name TEXT NOT NULL,
    description TEXT,
    category_type TEXT NOT NULL DEFAULT 'GENERAL'
        CHECK(category_type IN ('GENERAL','TRAVEL','MEALS','LODGING','ENTERTAINMENT','OFFICE','MILEAGE','PER_DIEM','COMMUNICATION','TRAINING','OTHER')),
    requires_receipt INTEGER NOT NULL DEFAULT 0,
    is_billable INTEGER NOT NULL DEFAULT 1,
    default_expense_account_id TEXT,            -- GL account for this category
    parent_category_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, category_code)
);

CREATE INDEX idx_expense_categories_tenant_type ON expense_categories(tenant_id, category_type);
```

### 2.4 Expense Reports (Header)
```sql
CREATE TABLE expense_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_number TEXT NOT NULL,                -- EXP-2024-00001
    employee_id TEXT NOT NULL,
    purpose TEXT NOT NULL,
    description TEXT,

    -- Trip details
    trip_from TEXT,
    trip_to TEXT,
    trip_start_date TEXT,
    trip_end_date TEXT,

    -- Status
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','MANAGER_APPROVED','FINANCE_APPROVED','REIMBURSED','POSTED','REJECTED','RETURNED','CANCELLED')),

    -- Policy
    policy_id TEXT NOT NULL,
    policy_compliance_status TEXT NOT NULL DEFAULT 'COMPLIANT'
        CHECK(policy_compliance_status IN ('COMPLIANT','WARNING','VIOLATION')),

    -- Totals
    total_cents INTEGER NOT NULL DEFAULT 0,
    reimbursable_cents INTEGER NOT NULL DEFAULT 0,
    non_reimbursable_cents INTEGER NOT NULL DEFAULT 0,
    cash_advance_cents INTEGER NOT NULL DEFAULT 0,
    amount_due_employee_cents INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,

    -- Currency
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Approval
    submitted_at TEXT,
    manager_approved_by TEXT,
    manager_approved_at TEXT,
    finance_approved_by TEXT,
    finance_approved_at TEXT,
    reimbursed_at TEXT,
    posted_at TEXT,
    rejection_reason TEXT,

    -- Integration references
    gl_journal_id TEXT,
    ap_invoice_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (policy_id) REFERENCES expense_policies(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, report_number)
);

CREATE INDEX idx_expense_reports_tenant_employee ON expense_reports(tenant_id, employee_id);
CREATE INDEX idx_expense_reports_tenant_status ON expense_reports(tenant_id, status);
CREATE INDEX idx_expense_reports_tenant_dates ON expense_reports(tenant_id, trip_start_date, trip_end_date);
CREATE INDEX idx_expense_reports_tenant_submitted ON expense_reports(tenant_id, submitted_at);
```

### 2.5 Expense Report Lines
```sql
CREATE TABLE expense_report_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    category_id TEXT NOT NULL,
    expense_type TEXT NOT NULL DEFAULT 'ACTUAL'
        CHECK(expense_type IN ('ACTUAL','PER_DIEM','MILEAGE','CASH_ADVANCE','CORPORATE_CARD')),

    -- Expense detail
    expense_date TEXT NOT NULL,
    description TEXT NOT NULL,
    merchant_name TEXT,
    merchant_location TEXT,

    -- Amounts
    amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    reimbursable_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    converted_amount_cents INTEGER NOT NULL DEFAULT 0,  -- In report currency

    -- Receipt
    receipt_id TEXT,
    receipt_required INTEGER NOT NULL DEFAULT 0,
    receipt_status TEXT DEFAULT 'NOT_REQUIRED'
        CHECK(receipt_status IN ('NOT_REQUIRED','PENDING','ATTACHED','MISSING','REJECTED')),

    -- Per diem details
    per_diem_rate_id TEXT,
    per_diem_location TEXT,
    per_diem_hours INTEGER,
    per_diem_rate_cents INTEGER,
    per_diem_amount_cents INTEGER,

    -- Mileage details
    mileage_from_location TEXT,
    mileage_to_location TEXT,
    mileage_distance_miles REAL,
    mileage_round_trip INTEGER NOT NULL DEFAULT 0,
    mileage_rate_id TEXT,
    mileage_rate_per_mile_cents INTEGER,
    mileage_amount_cents INTEGER,

    -- Corporate card matching
    corporate_card_transaction_id TEXT,

    -- Project/cost center allocation
    project_id TEXT,
    project_task_id TEXT,
    cost_center TEXT,
    department TEXT,

    -- Attendees (for entertainment/meals)
    attendees TEXT,                             -- JSON array of attendee objects
    number_of_attendees INTEGER,

    -- Policy compliance
    policy_compliance_status TEXT NOT NULL DEFAULT 'COMPLIANT'
        CHECK(policy_compliance_status IN ('COMPLIANT','WARNING','VIOLATION')),
    policy_violation_reason TEXT,

    -- Tax recovery
    tax_recoverable INTEGER NOT NULL DEFAULT 0,
    tax_recovery_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_type TEXT,                              -- VAT, GST, etc.

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (report_id) REFERENCES expense_reports(id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES expense_categories(id) ON DELETE RESTRICT,
    -- receipt_id is a logical reference to expense_receipts(id), validated via gRPC
    UNIQUE(tenant_id, report_id, line_number)
);

CREATE INDEX idx_expense_report_lines_tenant_report ON expense_report_lines(tenant_id, report_id);
CREATE INDEX idx_expense_report_lines_tenant_category ON expense_report_lines(tenant_id, category_id);
CREATE INDEX idx_expense_report_lines_tenant_date ON expense_report_lines(tenant_id, expense_date);
CREATE INDEX idx_expense_report_lines_tenant_project ON expense_report_lines(tenant_id, project_id);
CREATE INDEX idx_expense_report_lines_tenant_card_txn ON expense_report_lines(tenant_id, corporate_card_transaction_id);
```

### 2.6 Expense Per Diems
```sql
CREATE TABLE expense_per_diems (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    location_code TEXT NOT NULL,                -- City/region/country code
    location_name TEXT NOT NULL,
    country TEXT NOT NULL,
    state_province TEXT,

    -- Rate tiers (by employee grade/band)
    grade_band TEXT NOT NULL,                   -- "EXECUTIVE", "MANAGER", "STAFF", etc.

    -- Daily rates (cents)
    lodging_rate_cents INTEGER NOT NULL DEFAULT 0,
    meals_rate_cents INTEGER NOT NULL DEFAULT 0,
    incidental_rate_cents INTEGER NOT NULL DEFAULT 0,
    total_rate_cents INTEGER NOT NULL DEFAULT 0,

    -- Effective dates
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    -- Partial day rules
    first_day_rate_percent REAL NOT NULL DEFAULT 75.0,
    last_day_rate_percent REAL NOT NULL DEFAULT 75.0,
    partial_day_threshold_hours INTEGER NOT NULL DEFAULT 6,

    -- Meal deductions (if meals provided)
    breakfast_deduction_cents INTEGER NOT NULL DEFAULT 0,
    lunch_deduction_cents INTEGER NOT NULL DEFAULT 0,
    dinner_deduction_cents INTEGER NOT NULL DEFAULT 0,

    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, location_code, grade_band, effective_from)
);

CREATE INDEX idx_expense_per_diems_tenant_location ON expense_per_diems(tenant_id, location_code);
CREATE INDEX idx_expense_per_diems_tenant_country ON expense_per_diems(tenant_id, country);
```

### 2.7 Corporate Cards
```sql
CREATE TABLE corporate_cards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    card_number_encrypted TEXT NOT NULL,        -- AES-256-GCM encrypted
    card_number_masked TEXT NOT NULL,           -- Display: ****1234
    card_type TEXT NOT NULL
        CHECK(card_type IN ('VISA','MASTERCARD','AMEX','DISCOVER','OTHER')),
    cardholder_name TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    issuing_bank TEXT,
    expiry_month INTEGER,
    expiry_year INTEGER,

    -- Limits
    monthly_limit_cents INTEGER,
    single_transaction_limit_cents INTEGER,
    allowed_categories TEXT,                    -- JSON array of category IDs

    -- Status
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUSPENDED','CANCELLED','EXPIRED','LOST_STOLEN')),
    activation_date TEXT,
    cancellation_date TEXT,
    cancellation_reason TEXT,

    -- Billing cycle
    billing_cycle_day INTEGER NOT NULL DEFAULT 1,
    last_statement_date TEXT,
    current_balance_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    -- employee_id is a logical reference to users(id), validated via gRPC
    UNIQUE(tenant_id, card_number_encrypted)
);

CREATE INDEX idx_corporate_cards_tenant_employee ON corporate_cards(tenant_id, employee_id);
CREATE INDEX idx_corporate_cards_tenant_status ON corporate_cards(tenant_id, status);
```

### 2.8 Corporate Card Transactions
```sql
CREATE TABLE corporate_card_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    corporate_card_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,

    -- Transaction details
    transaction_date TEXT NOT NULL,
    posting_date TEXT,
    merchant_name TEXT NOT NULL,
    merchant_category TEXT,
    merchant_location TEXT,
    description TEXT,

    -- Amounts
    amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    converted_amount_cents INTEGER NOT NULL DEFAULT 0,  -- In functional currency

    -- Matching
    match_status TEXT NOT NULL DEFAULT 'UNMATCHED'
        CHECK(match_status IN ('UNMATCHED','MATCHED','DISPUTED','EXCLUDED')),
    matched_report_line_id TEXT,
    matched_at TEXT,

    -- Import reference
    import_batch_id TEXT,
    statement_date TEXT,
    reference_number TEXT,

    -- Dispute
    dispute_status TEXT CHECK(dispute_status IN ('NONE','PENDING','RESOLVED','REJECTED')),
    dispute_reason TEXT,
    dispute_resolved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (corporate_card_id) REFERENCES corporate_cards(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, corporate_card_id, transaction_date, reference_number)
);

CREATE INDEX idx_card_txns_tenant_card ON corporate_card_transactions(tenant_id, corporate_card_id);
CREATE INDEX idx_card_txns_tenant_employee ON corporate_card_transactions(tenant_id, employee_id);
CREATE INDEX idx_card_txns_tenant_status ON corporate_card_transactions(tenant_id, match_status);
CREATE INDEX idx_card_txns_tenant_date ON corporate_card_transactions(tenant_id, transaction_date);
CREATE INDEX idx_card_txns_tenant_import ON corporate_card_transactions(tenant_id, import_batch_id);
```

### 2.9 Mileage Rates
```sql
CREATE TABLE mileage_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    vehicle_type TEXT NOT NULL DEFAULT 'CAR'
        CHECK(vehicle_type IN ('CAR','MOTORCYCLE','VAN','TRUCK','ELECTRIC_VEHICLE')),
    rate_per_mile_cents INTEGER NOT NULL,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    country TEXT NOT NULL DEFAULT 'US',
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, vehicle_type, country, effective_from)
);

CREATE INDEX idx_mileage_rates_tenant_vehicle ON mileage_rates(tenant_id, vehicle_type);
```

### 2.10 Expense Receipts
```sql
CREATE TABLE expense_receipts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_line_id TEXT,

    -- File reference
    file_name TEXT NOT NULL,
    file_path TEXT NOT NULL,                    -- Storage path or object key
    file_size_bytes INTEGER NOT NULL,
    file_type TEXT NOT NULL                     -- 'PDF', 'JPEG', 'PNG', 'TIFF', 'HTML'
        CHECK(file_type IN ('PDF','JPEG','PNG','TIFF','HTML','OTHER')),
    content_hash TEXT NOT NULL,                 -- SHA-256 for deduplication

    -- OCR data
    ocr_status TEXT DEFAULT 'PENDING'
        CHECK(ocr_status IN ('PENDING','PROCESSING','COMPLETED','FAILED')),
    ocr_vendor_name TEXT,
    ocr_total_amount_cents INTEGER,
    ocr_transaction_date TEXT,
    ocr_confidence_score REAL,                 -- 0.0 to 1.0
    ocr_raw_data TEXT,                          -- JSON blob of full OCR output

    -- Upload metadata
    uploaded_at TEXT NOT NULL DEFAULT (datetime('now')),
    uploaded_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (report_line_id) REFERENCES expense_report_lines(id) ON DELETE SET NULL
);

CREATE INDEX idx_expense_receipts_tenant_line ON expense_receipts(tenant_id, report_line_id);
CREATE INDEX idx_expense_receipts_tenant_hash ON expense_receipts(tenant_id, content_hash);
CREATE INDEX idx_expense_receipts_tenant_ocr ON expense_receipts(tenant_id, ocr_status);
```

### 2.11 Expense Allocations
```sql
CREATE TABLE expense_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_line_id TEXT NOT NULL,
    allocation_percent REAL NOT NULL DEFAULT 100.0,

    -- Allocation targets
    project_id TEXT,
    project_task_id TEXT,
    cost_center TEXT,
    department TEXT,
    expense_account_id TEXT,                   -- GL account override

    -- Allocated amount
    allocated_amount_cents INTEGER NOT NULL DEFAULT 0,
    allocated_tax_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (report_line_id) REFERENCES expense_report_lines(id) ON DELETE CASCADE
);

CREATE INDEX idx_expense_allocations_tenant_line ON expense_allocations(tenant_id, report_line_id);
CREATE INDEX idx_expense_allocations_tenant_project ON expense_allocations(tenant_id, project_id);
CREATE INDEX idx_expense_allocations_tenant_cost_center ON expense_allocations(tenant_id, cost_center);
```

### 2.12 Expense Number Sequences
```sql
CREATE TABLE expense_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_type TEXT NOT NULL,               -- 'REPORT', 'RECEIPT'
    prefix TEXT NOT NULL,
    current_value INTEGER NOT NULL DEFAULT 0,
    fiscal_year INTEGER NOT NULL,
    UNIQUE(tenant_id, sequence_type, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Expense Policies
```
GET    /api/v1/expense/policies                        Permission: expense.policies.read
POST   /api/v1/expense/policies                        Permission: expense.policies.create
GET    /api/v1/expense/policies/{id}                    Permission: expense.policies.read
PUT    /api/v1/expense/policies/{id}                    Permission: expense.policies.update
DELETE /api/v1/expense/policies/{id}                    Permission: expense.policies.delete

GET    /api/v1/expense/policies/{id}/rules              Permission: expense.policies.read
POST   /api/v1/expense/policies/{id}/rules              Permission: expense.policies.create
PUT    /api/v1/expense/policies/{id}/rules/{rule_id}    Permission: expense.policies.update
DELETE /api/v1/expense/policies/{id}/rules/{rule_id}    Permission: expense.policies.delete
```

### 3.2 Expense Categories
```
GET    /api/v1/expense/categories                       Permission: expense.categories.read
POST   /api/v1/expense/categories                       Permission: expense.categories.create
GET    /api/v1/expense/categories/{id}                   Permission: expense.categories.read
PUT    /api/v1/expense/categories/{id}                   Permission: expense.categories.update
DELETE /api/v1/expense/categories/{id}                   Permission: expense.categories.delete
```

### 3.3 Expense Reports
```
GET    /api/v1/expense/reports                          Permission: expense.reports.read
POST   /api/v1/expense/reports                          Permission: expense.reports.create
GET    /api/v1/expense/reports/{id}                     Permission: expense.reports.read
PUT    /api/v1/expense/reports/{id}                     Permission: expense.reports.update
POST   /api/v1/expense/reports/{id}/submit              Permission: expense.reports.create
POST   /api/v1/expense/reports/{id}/approve             Permission: expense.reports.approve
POST   /api/v1/expense/reports/{id}/reject              Permission: expense.reports.approve
POST   /api/v1/expense/reports/{id}/return              Permission: expense.reports.approve
POST   /api/v1/expense/reports/{id}/cancel              Permission: expense.reports.update
POST   /api/v1/expense/reports/{id}/reimburse           Permission: expense.reports.reimburse
POST   /api/v1/expense/reports/{id}/post                Permission: expense.reports.post
GET    /api/v1/expense/reports/{id}/audit-trail         Permission: expense.reports.read
```

### 3.4 Expense Report Lines
```
GET    /api/v1/expense/reports/{id}/lines               Permission: expense.reports.read
POST   /api/v1/expense/reports/{id}/lines               Permission: expense.reports.create
GET    /api/v1/expense/lines/{line_id}                   Permission: expense.reports.read
PUT    /api/v1/expense/lines/{line_id}                   Permission: expense.reports.update
DELETE /api/v1/expense/lines/{line_id}                   Permission: expense.reports.delete
POST   /api/v1/expense/lines/{line_id}/allocations       Permission: expense.reports.create
GET    /api/v1/expense/lines/{line_id}/allocations       Permission: expense.reports.read
```

### 3.5 Per Diems
```
GET    /api/v1/expense/per-diems                        Permission: expense.per-diems.read
POST   /api/v1/expense/per-diems                        Permission: expense.per-diems.create
GET    /api/v1/expense/per-diems/{id}                    Permission: expense.per-diems.read
PUT    /api/v1/expense/per-diems/{id}                    Permission: expense.per-diems.update
DELETE /api/v1/expense/per-diems/{id}                    Permission: expense.per-diems.delete
POST   /api/v1/expense/per-diems/calculate              Permission: expense.per-diems.read
```

### 3.6 Corporate Cards
```
GET    /api/v1/expense/corporate-cards                  Permission: expense.corporate-cards.read
POST   /api/v1/expense/corporate-cards                  Permission: expense.corporate-cards.create
GET    /api/v1/expense/corporate-cards/{id}              Permission: expense.corporate-cards.read
PUT    /api/v1/expense/corporate-cards/{id}              Permission: expense.corporate-cards.update
PATCH  /api/v1/expense/corporate-cards/{id}/status       Permission: expense.corporate-cards.update

GET    /api/v1/expense/corporate-cards/{id}/transactions  Permission: expense.corporate-cards.read
POST   /api/v1/expense/corporate-cards/import            Permission: expense.corporate-cards.create
POST   /api/v1/expense/corporate-cards/{id}/transactions/{txn_id}/dispute Permission: expense.corporate-cards.update
```

### 3.7 Card Transaction Matching
```
GET    /api/v1/expense/card-transactions/unmatched      Permission: expense.corporate-cards.read
POST   /api/v1/expense/card-transactions/match          Permission: expense.corporate-cards.update
POST   /api/v1/expense/card-transactions/auto-match     Permission: expense.corporate-cards.update
POST   /api/v1/expense/card-transactions/{id}/unmatch   Permission: expense.corporate-cards.update
POST   /api/v1/expense/card-transactions/{id}/exclude   Permission: expense.corporate-cards.update
```

### 3.8 Mileage Rates
```
GET    /api/v1/expense/mileage-rates                    Permission: expense.mileage-rates.read
POST   /api/v1/expense/mileage-rates                    Permission: expense.mileage-rates.create
GET    /api/v1/expense/mileage-rates/{id}                Permission: expense.mileage-rates.read
PUT    /api/v1/expense/mileage-rates/{id}                Permission: expense.mileage-rates.update
DELETE /api/v1/expense/mileage-rates/{id}                Permission: expense.mileage-rates.delete
POST   /api/v1/expense/mileage-rates/calculate          Permission: expense.mileage-rates.read
```

### 3.9 Receipts
```
POST   /api/v1/expense/receipts/upload                  Permission: expense.receipts.create
GET    /api/v1/expense/receipts/{id}                     Permission: expense.receipts.read
GET    /api/v1/expense/receipts/{id}/download            Permission: expense.receipts.read
DELETE /api/v1/expense/receipts/{id}                     Permission: expense.receipts.delete
POST   /api/v1/expense/receipts/{id}/ocr                 Permission: expense.receipts.create
GET    /api/v1/expense/receipts/{id}/ocr-status          Permission: expense.receipts.read
```

### 3.10 Reports & Analytics
```
GET    /api/v1/expense/reports-analytics/by-employee     Permission: expense.reports.read
GET    /api/v1/expense/reports-analytics/by-category     Permission: expense.reports.read
GET    /api/v1/expense/reports-analytics/by-department   Permission: expense.reports.read
GET    /api/v1/expense/reports-analytics/by-project      Permission: expense.reports.read
GET    /api/v1/expense/reports-analytics/policy-violations Permission: expense.reports.read
GET    /api/v1/expense/reports-analytics/aging           Permission: expense.reports.read
GET    /api/v1/expense/reports-analytics/corporate-card-summary Permission: expense.corporate-cards.read
```

### 3.11 Per Diem Calculation Request
```json
{
  "employee_id": "0192...",
  "grade_band": "MANAGER",
  "location_code": "NYC",
  "trip_start_date": "2024-03-10T08:00:00Z",
  "trip_end_date": "2024-03-13T18:00:00Z",
  "meals_provided": ["2024-03-11:lunch", "2024-03-12:breakfast,dinner"]
}
```

### 3.12 Mileage Calculation Request
```json
{
  "from_location": "New York, NY",
  "to_location": "Boston, MA",
  "round_trip": true,
  "vehicle_type": "CAR",
  "expense_date": "2024-03-15"
}
```

---

## 4. Business Rules

### 4.1 Expense Report Creation and Submission
1. Employee creates report in DRAFT status with a mandatory purpose
2. Employee adds expense lines (actual, per diem, or mileage)
3. On submit: validate all required fields, run policy compliance check
4. Report number auto-generated: `EXP-{YYYY}-{NNNNN}`
5. Only reports with all required receipts attached can be submitted
6. Cannot submit report with zero total
7. Duplicate detection: warn if similar amount + merchant + date found within `duplicate_check_window_days`

### 4.2 Policy Compliance Checking
On report submission and line save, evaluate against policy rules:
1. Check report-level limits: daily, weekly, monthly
2. Check category-level rules from `expense_policy_rules`
3. Compare each line amount against `max_amount_per_transaction_cents`
4. Check aggregate daily/monthly totals per category
5. Validate time-of-day and day-of-week restrictions
6. Auto-flag violations: set `policy_compliance_status` to WARNING or VIOLATION
7. VIOLATION with action=BLOCK prevents submission entirely
8. VIOLATION with action=FLAG allows submission but adds review notes for approver
9. All violations recorded in `policy_violation_reason` on each line

### 4.3 Receipt Requirement Rules
1. Global threshold: `expense_policies.receipt_threshold_cents`
2. Category override: `expense_policy_rules.receipt_threshold_cents`
3. If line `total_cents >= threshold`, receipt is required
4. Categories with `requires_receipt = 1` always require receipt regardless of amount
5. Line `receipt_status` transitions: NOT_REQUIRED -> PENDING -> ATTACHED
6. Report cannot be submitted if any line has receipt_status = PENDING or MISSING
7. Maximum receipt file size: 10 MB per file, 50 MB per report

### 4.4 Per Diem Calculation
1. Look up rate by `location_code`, `grade_band`, and effective date range
2. Calculate full days between trip_start and trip_end (exclusive of first and last day)
3. First day: `total_rate_cents * (first_day_rate_percent / 100)`
4. Last day: `total_rate_cents * (last_day_rate_percent / 100)`
5. Partial day: if hours at destination < `partial_day_threshold_hours`, apply partial rate
6. Meal deductions: subtract provided meal amounts from daily rate
7. Per diem lines are always non-taxable (no tax recovery)
8. Per diem cannot exceed total daily rate regardless of actual spending

### 4.5 Mileage Calculation
1. Look up rate by `vehicle_type`, `country`, and effective date
2. `mileage_amount_cents = ROUND(mileage_distance_miles * mileage_rate_per_mile_cents)`
3. Round trip doubles the distance
4. Distance source: manual entry or integrated map service
5. Only one mileage rate effective at a time per vehicle type + country
6. Mileage is reimbursable; not subject to receipts

### 4.6 Corporate Card Transaction Matching
1. Import card transactions via file upload (CSV, OFX) or API feed
2. Auto-match algorithm compares: `employee_id`, `transaction_date` (within 3 days), `amount_cents` (exact or within tolerance), `merchant_name` (fuzzy)
3. Matching priority: exact amount + same date > exact amount + close date > close amount + same date
4. Matched transactions auto-populate expense line fields
5. Unmatched transactions listed for employee to manually match or create new expense lines
6. Disputed transactions excluded from report totals
7. Import batch tracked via `import_batch_id` for audit

### 4.7 Expense Allocation
1. A single expense line MAY be split across multiple cost centers, projects, or departments
2. Allocation percentages MUST sum to 100% per line
3. `allocated_amount_cents = ROUND(line.total_cents * (allocation_percent / 100))`
4. Tax allocated proportionally with the base amount
5. GL account override per allocation for fine-grained posting
6. When no allocations exist, the full line posts to the default accounts

### 4.8 Approval Routing
1. On submit: if `auto_approve_below_cents` and report total <= threshold, auto-approve to MANAGER_APPROVED
2. Otherwise: route to employee's direct manager via Workflow gRPC call
3. Manager approves: status transitions to MANAGER_APPROVED
4. If report total >= `finance_approval_threshold_cents`, route to finance team
5. Finance approves: status transitions to FINANCE_APPROVED
6. Any rejection returns report to employee with reason; status = RETURNED or REJECTED
7. Workflow delegation rules apply (managed by workflow-service)

### 4.9 Currency Conversion
1. Each line MAY have a different `currency_code` from the report currency
2. `exchange_rate` captured at time of entry or auto-fetched from rate service
3. `converted_amount_cents = ROUND(amount_cents * exchange_rate)`
4. Report totals always in the report `currency_code`
5. GL posting uses functional currency (from tenant settings)
6. Exchange rate source tracked for audit

### 4.10 Tax Recovery (VAT/GST)
1. Lines may specify `tax_type` (VAT, GST, etc.) and `tax_recoverable` flag
2. `tax_recovery_amount_cents` calculated based on jurisdiction rules
3. Recoverable tax is excluded from reimbursement (employer claims directly)
4. Non-recoverable tax included in reimbursement amount
5. Tax recovery GL entries posted separately on report posting

### 4.11 Expense Reimbursement via AP Integration
1. After FINANCE_APPROVED, expense-service calls AP gRPC to create an employee invoice
2. AP creates supplier invoice with type=EMPLOYEE referencing `ap_invoice_id`
3. On AP payment completion, AP publishes `ap.invoice.paid` event
4. Expense-service receives event and updates report status to REIMBURSED
5. `amount_due_employee_cents = reimbursable_cents - cash_advance_cents`

### 4.12 GL Posting
On report posting, expense-service calls GL gRPC to create journal entry:
```
For each expense line (net of tax recovery):
  Debit: expense_account_id (from category or allocation) → line.reimbursable_amount_cents
  Debit: tax_receivable_account (if tax recoverable) → line.tax_recovery_amount_cents
Credit: employee payable / AP control account → report.amount_due_employee_cents
```

### 4.13 Status Flow
```
DRAFT → SUBMITTED → MANAGER_APPROVED → FINANCE_APPROVED → REIMBURSED → POSTED
  ↓        ↓              ↓                  ↓
CANCELLED  REJECTED      RETURNED           REJECTED
```

- DRAFT: Editable, lines can be added/removed
- SUBMITTED: Locked for editing, pending manager approval
- MANAGER_APPROVED: Manager approved, awaiting finance (if threshold met)
- FINANCE_APPROVED: All approvals complete, awaiting reimbursement
- REIMBURSED: Payment issued to employee via AP
- POSTED: GL journal entry created, immutable
- RETURNED: Sent back to employee for corrections (returns to DRAFT)
- REJECTED: Permanently rejected
- CANCELLED: Cancelled by employee before submission

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.expense.v1;

service ExpenseManagementService {
    // Expense report operations
    rpc GetExpenseReport(GetExpenseReportRequest) returns (GetExpenseReportResponse);
    rpc CreateExpenseReport(CreateExpenseReportRequest) returns (CreateExpenseReportResponse);
    rpc SubmitExpenseReport(SubmitExpenseReportRequest) returns (SubmitExpenseReportResponse);
    rpc ApproveExpenseReport(ApproveExpenseReportRequest) returns (ApproveExpenseReportResponse);
    rpc RejectExpenseReport(RejectExpenseReportRequest) returns (RejectExpenseReportResponse);
    rpc GetExpenseReportByNumber(GetExpenseReportByNumberRequest) returns (GetExpenseReportByNumberResponse);

    // Expense line operations
    rpc AddExpenseLine(AddExpenseLineRequest) returns (AddExpenseLineResponse);
    rpc GetExpenseLinesByReport(GetExpenseLinesByReportRequest) returns (GetExpenseLinesByReportResponse);

    // Policy compliance
    rpc CheckPolicyCompliance(CheckPolicyComplianceRequest) returns (CheckPolicyComplianceResponse);

    // Per diem & mileage calculations
    rpc CalculatePerDiem(CalculatePerDiemRequest) returns (CalculatePerDiemResponse);
    rpc CalculateMileage(CalculateMileageRequest) returns (CalculateMileageResponse);

    // Corporate card operations
    rpc ImportCardTransactions(ImportCardTransactionsRequest) returns (ImportCardTransactionsResponse);
    rpc GetUnmatchedTransactions(GetUnmatchedTransactionsRequest) returns (GetUnmatchedTransactionsResponse);
    rpc MatchTransaction(MatchTransactionRequest) returns (MatchTransactionResponse);

    // Integration operations
    rpc GetEmployeeExpenseTotal(GetEmployeeExpenseTotalRequest) returns (GetEmployeeExpenseTotalResponse);
    rpc GetProjectExpenses(GetProjectExpensesRequest) returns (GetProjectExpensesResponse);

    // Receipt operations
    rpc UploadReceipt(UploadReceiptRequest) returns (UploadReceiptResponse);
    rpc TriggerOcr(TriggerOcrRequest) returns (TriggerOcrResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 GL Integration
On expense report posting, call GL gRPC `CreateJournal`:
```
For each expense line (or allocation):
  Debit: expense_account_id → allocated_amount_cents
  Debit: tax_receivable_account (if tax_recoverable) → tax_recovery_amount_cents
Credit: AP control account / Employee Payable → report.amount_due_employee_cents
```

### 6.2 AP Integration
On FINANCE_APPROVED, call AP gRPC `CreateInvoice`:
- Create employee-type supplier invoice
- One line per expense report line (or allocation)
- Reference `expense_report_id` in invoice
- On `ap.invoice.paid` event: update report status to REIMBURSED

### 6.3 PM Integration
- Expense lines with `project_id` are reported to PM via gRPC `GetProjectExpenses`
- PM queries expense-service for project cost rollup
- Expense allocations propagate project costing

### 6.4 Workflow Integration
- On report submission, call Workflow gRPC `SubmitApproval` with `doc_type = EXPENSE_REPORT`
- Workflow engine routes to manager (HIERARCHY) and/or finance (ROLE_BASED)
- On `workflow.approved` event for EXPENSE_REPORT: advance report status
- On `workflow.rejected` event: set report to REJECTED

### 6.5 Event Consumption
| Event | Source | Action |
|-------|--------|--------|
| `workflow.approved` | workflow-service | Advance report approval status |
| `workflow.rejected` | workflow-service | Reject expense report |
| `ap.invoice.paid` | ap-service | Mark report as REIMBURSED |

---

## 7. Events Published

| Event | Trigger | Consumers |
|-------|---------|-----------|
| `expense.report.created` | Report created | Workflow, Report |
| `expense.report.submitted` | Report submitted for approval | Workflow |
| `expense.report.approved` | Report fully approved | AP, Report |
| `expense.report.rejected` | Report rejected | Report |
| `expense.report.reimbursed` | Payment issued to employee | GL, Report |
| `expense.report.posted` | GL journal created | GL, PM, Report |
| `expense.report.returned` | Report returned for corrections | Report |
| `expense.line.violation` | Policy violation detected | Report |
| `expense.card_transaction.imported` | Card transactions imported | Report |
| `expense.card_transaction.matched` | Card transaction matched to line | Report |

---

## 8. Migrations

### Migration Order for expense-service:
1. V001: `expense_categories`
2. V002: `expense_policies`
3. V003: `expense_policy_rules`
4. V004: `expense_per_diems`
5. V005: `mileage_rates`
6. V006: `corporate_cards`
7. V007: `corporate_card_transactions`
8. V008: `expense_receipts`
9. V009: `expense_reports`
10. V010: `expense_report_lines`
11. V011: `expense_allocations`
12. V012: `expense_sequences`
13. V013: Triggers for `updated_at`
14. V014: Seed data (default categories: Travel, Meals, Lodging, Entertainment, Office Supplies, Mileage, Per Diem, Communication, Training, Other)
15. V015: Seed data (default expense policy with standard rules per category)
