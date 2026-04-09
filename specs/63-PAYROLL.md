# 63 - Payroll Service Specification

## 1. Domain Overview

The Payroll service manages end-to-end payroll processing including payroll definition, element setup, formula configuration, payroll relationship management, balance tracking, payroll run execution, payslip generation, and accounting integration. It supports multiple payroll frequencies, legislative data groups, and complex earnings/deductions calculations.

**Bounded Context:** Payroll Processing & Administration
**Service Name:** `payroll-service`
**Database:** `data/payroll.db`
**HTTP Port:** 8095 | **gRPC Port:** 9095

---

## 2. Database Schema

### 2.1 Payroll Definitions
```sql
CREATE TABLE payroll_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    payroll_name TEXT NOT NULL,
    payroll_code TEXT NOT NULL,
    payroll_frequency TEXT NOT NULL CHECK(payroll_frequency IN ('WEEKLY','BIWEEKLY','SEMI_MONTHLY','MONTHLY','QUARTERLY')),
    default_currency_code TEXT NOT NULL DEFAULT 'USD',
    legislative_data_group_id TEXT NOT NULL,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    period_type TEXT NOT NULL CHECK(period_type IN ('CALENDAR_MONTH','LUNAR_MONTH','445_WEEKS','CUSTOM')),
    processing_status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(processing_status IN ('ACTIVE','INACTIVE','ARCHIVED')),
    payment_method TEXT DEFAULT 'BANK_TRANSFER' CHECK(payment_method IN ('BANK_TRANSFER','CHECK','CASH','MIXED')),
    description TEXT,
    default_gl_account_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, payroll_code)
);

CREATE INDEX idx_payroll_defs_tenant_status ON payroll_definitions(tenant_id, processing_status);
CREATE INDEX idx_payroll_defs_tenant_freq ON payroll_definitions(tenant_id, payroll_frequency);
```

### 2.2 Payroll Elements
```sql
CREATE TABLE payroll_elements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    element_code TEXT NOT NULL,
    element_name TEXT NOT NULL,
    element_type TEXT NOT NULL CHECK(element_type IN ('EARNING','DEDUCTION','TAX','EMPLOYER_CHARGE','INFORMATION','ABSENCE')),
    element_classification TEXT NOT NULL CHECK(element_classification IN ('PRIMARY_EARNING','SUPPLEMENTAL_EARNING','PRE_TAX_DEDUCTION','TAX_DEDUCTION','INVOLUNTARY_DEDUCTION','EMPLOYER_TAX','EMPLOYER_LIABILITY')),
    category TEXT,
    reporting_name TEXT,
    description TEXT,
    processing_type TEXT NOT NULL DEFAULT 'RECURRING' CHECK(processing_type IN ('RECURRING','NON_RECURRING')),
    calculation_rule TEXT NOT NULL DEFAULT 'FLAT_AMOUNT' CHECK(calculation_rule IN ('FLAT_AMOUNT','HOURLY_RATE','PERCENTAGE','FORMULA','TABLE_LOOKUP')),
    default_value INTEGER,  -- cents
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    retroactive_flag INTEGER NOT NULL DEFAULT 0,
    adjustment_flag INTEGER NOT NULL DEFAULT 0,
    proration_rule TEXT DEFAULT 'NO_PRORATION' CHECK(proration_rule IN ('NO_PRORATION','CALENDAR_DAYS','WORKING_DAYS','HOURS')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, element_code)
);

CREATE INDEX idx_payroll_elements_tenant_type ON payroll_elements(tenant_id, element_type);
CREATE INDEX idx_payroll_elements_tenant_class ON payroll_elements(tenant_id, element_classification);
```

### 2.3 Payroll Element Formulas
```sql
CREATE TABLE payroll_element_formulas (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    element_id TEXT NOT NULL,
    formula_name TEXT NOT NULL,
    formula_text TEXT NOT NULL,
    formula_language TEXT NOT NULL DEFAULT 'FASTFORMULA' CHECK(formula_language IN ('FASTFORMULA','JAVASCRIPT','SQL')),
    description TEXT,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','COMPILED','ACTIVE','INACTIVE','ERROR')),
    compilation_output TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (element_id) REFERENCES payroll_elements(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, element_id, formula_name)
);

CREATE INDEX idx_payroll_formulas_tenant_element ON payroll_element_formulas(tenant_id, element_id);
```

### 2.4 Payroll Relationships
```sql
CREATE TABLE payroll_relationships (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    payroll_id TEXT NOT NULL,
    relationship_number TEXT NOT NULL,
    tax_group TEXT,
    payment_method TEXT DEFAULT 'BANK_TRANSFER' CHECK(payment_method IN ('BANK_TRANSFER','CHECK','CASH')),
    bank_account_id TEXT,
    pay_group TEXT,
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT NOT NULL DEFAULT '4712-12-31',
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','SUSPENDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, relationship_number)
);

CREATE INDEX idx_payroll_rels_tenant_employee ON payroll_relationships(tenant_id, employee_id);
CREATE INDEX idx_payroll_rels_tenant_payroll ON payroll_relationships(tenant_id, payroll_id);
```

### 2.5 Payroll Balance Dimensions
```sql
CREATE TABLE payroll_balance_dimensions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_name TEXT NOT NULL,
    dimension_code TEXT NOT NULL,
    dimension_type TEXT NOT NULL CHECK(dimension_type IN ('PERIOD','QUARTER_TO_DATE','YEAR_TO_DATE','LIFETIME_TO_DATE','RUN')),
    element_id TEXT,
    description TEXT,
    unit_of_measure TEXT DEFAULT 'MONEY' CHECK(unit_of_measure IN ('MONEY','HOURS','DAYS','UNITS')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dimension_code)
);

CREATE INDEX idx_payroll_bal_dims_tenant_type ON payroll_balance_dimensions(tenant_id, dimension_type);
```

### 2.6 Payroll Process Runs
```sql
CREATE TABLE payroll_process_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    payroll_id TEXT NOT NULL,
    run_name TEXT NOT NULL,
    run_type TEXT NOT NULL CHECK(run_type IN ('REGULAR','OFF_CYCLE','RETROACTIVE','ADJUSTMENT','PREPAYMENT','PAYMENT')),
    period_start_date TEXT NOT NULL,
    period_end_date TEXT NOT NULL,
    pay_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','PROCESSING','COMPLETED','CONFIRMED','REVERSED','ERROR')),
    started_at TEXT,
    completed_at TEXT,
    total_gross_pay INTEGER DEFAULT 0,   -- cents
    total_net_pay INTEGER DEFAULT 0,     -- cents
    total_deductions INTEGER DEFAULT 0,  -- cents
    total_taxes INTEGER DEFAULT 0,       -- cents
    total_employer_charges INTEGER DEFAULT 0,  -- cents
    employee_count INTEGER DEFAULT 0,
    error_count INTEGER DEFAULT 0,
    error_details TEXT,  -- JSON
    confirmation_date TEXT,
    confirmed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (payroll_id) REFERENCES payroll_definitions(id) ON DELETE RESTRICT
);

CREATE INDEX idx_payroll_runs_tenant_payroll ON payroll_process_runs(tenant_id, payroll_id);
CREATE INDEX idx_payroll_runs_tenant_status ON payroll_process_runs(tenant_id, status);
CREATE INDEX idx_payroll_runs_tenant_period ON payroll_process_runs(tenant_id, period_start_date, period_end_date);
```

### 2.7 Payroll Run Results
```sql
CREATE TABLE payroll_run_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    process_run_id TEXT NOT NULL,
    payroll_relationship_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    assignment_id TEXT NOT NULL,
    gross_pay INTEGER NOT NULL DEFAULT 0,   -- cents
    net_pay INTEGER NOT NULL DEFAULT 0,     -- cents
    total_deductions INTEGER NOT NULL DEFAULT 0,
    total_taxes INTEGER NOT NULL DEFAULT 0,
    total_employer_charges INTEGER NOT NULL DEFAULT 0,
    payment_status TEXT DEFAULT 'UNPAID' CHECK(payment_status IN ('UNPAID','PAID','HELD','REVERSED')),
    payment_date TEXT,
    processing_status TEXT DEFAULT 'SUCCESS' CHECK(processing_status IN ('SUCCESS','WARNING','ERROR')),
    error_messages TEXT,  -- JSON array

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (process_run_id) REFERENCES payroll_process_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_payroll_results_tenant_run ON payroll_run_results(tenant_id, process_run_id);
CREATE INDEX idx_payroll_results_tenant_employee ON payroll_run_results(tenant_id, employee_id);
```

### 2.8 Payroll Run Result Lines
```sql
CREATE TABLE payroll_run_result_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_result_id TEXT NOT NULL,
    element_id TEXT NOT NULL,
    element_type TEXT NOT NULL,
    element_name TEXT NOT NULL,
    input_value_id TEXT,
    input_value_name TEXT,
    calculated_value INTEGER NOT NULL DEFAULT 0,  -- cents
    rate INTEGER,                                  -- cents per unit
    hours_or_units DECIMAL(10,2),
    effective_date TEXT NOT NULL,
    proration_type TEXT,
    retroactive_flag INTEGER DEFAULT 0,
    source_type TEXT DEFAULT 'CALCULATION' CHECK(source_type IN ('CALCULATION','MANUAL_ENTRY','FORMULA','IMPORT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (run_result_id) REFERENCES payroll_run_results(id) ON DELETE CASCADE
);

CREATE INDEX idx_payroll_lines_tenant_result ON payroll_run_result_lines(tenant_id, run_result_id);
CREATE INDEX idx_payroll_lines_tenant_element ON payroll_run_result_lines(tenant_id, element_id);
```

### 2.9 Payroll Payslips
```sql
CREATE TABLE payroll_payslips (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_result_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    payslip_number TEXT NOT NULL,
    pay_period_start TEXT NOT NULL,
    pay_period_end TEXT NOT NULL,
    pay_date TEXT NOT NULL,
    gross_pay INTEGER NOT NULL DEFAULT 0,
    net_pay INTEGER NOT NULL DEFAULT 0,
    total_earnings INTEGER NOT NULL DEFAULT 0,
    total_deductions INTEGER NOT NULL DEFAULT 0,
    total_taxes INTEGER NOT NULL DEFAULT 0,
    employer_charges INTEGER NOT NULL DEFAULT 0,
    year_to_date_gross INTEGER NOT NULL DEFAULT 0,
    year_to_date_net INTEGER NOT NULL DEFAULT 0,
    year_to_date_tax INTEGER NOT NULL DEFAULT 0,
    distribution_method TEXT,
    document_reference TEXT,
    generated_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (run_result_id) REFERENCES payroll_run_results(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, payslip_number)
);

CREATE INDEX idx_payroll_payslips_tenant_employee ON payroll_payslips(tenant_id, employee_id);
CREATE INDEX idx_payroll_payslips_tenant_period ON payroll_payslips(tenant_id, pay_period_start, pay_period_end);
```

### 2.10 Payroll Accounting
```sql
CREATE TABLE payroll_accounting (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    process_run_id TEXT NOT NULL,
    run_result_id TEXT NOT NULL,
    element_id TEXT NOT NULL,
    debit_account_code TEXT NOT NULL,
    credit_account_code TEXT NOT NULL,
    amount INTEGER NOT NULL,  -- cents
    accounting_date TEXT NOT NULL,
    accounting_type TEXT NOT NULL CHECK(accounting_type IN ('COSTING','LIABILITY','PAYMENT','REVERSAL','ACCRUAL')),
    journal_batch_id TEXT,
    journal_status TEXT DEFAULT 'PENDING' CHECK(journal_status IN ('PENDING','POSTED','REVERSED','ERROR')),
    cost_center TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (process_run_id) REFERENCES payroll_process_runs(id) ON DELETE CASCADE,
    FOREIGN KEY (run_result_id) REFERENCES payroll_run_results(id) ON DELETE CASCADE
);

CREATE INDEX idx_payroll_acct_tenant_run ON payroll_accounting(tenant_id, process_run_id);
CREATE INDEX idx_payroll_acct_tenant_status ON payroll_accounting(tenant_id, journal_status);
CREATE INDEX idx_payroll_acct_tenant_date ON payroll_accounting(tenant_id, accounting_date);
```

---

## 3. REST API Endpoints

### Payroll Definitions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/payrolls` | Create a new payroll definition |
| GET | `/api/v1/payrolls` | List payroll definitions |
| GET | `/api/v1/payrolls/{id}` | Get payroll definition details |
| PUT | `/api/v1/payrolls/{id}` | Update payroll definition |

### Element Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/elements` | Create a payroll element |
| GET | `/api/v1/elements` | List payroll elements |
| GET | `/api/v1/elements/{id}` | Get element details |
| PUT | `/api/v1/elements/{id}` | Update element |
| POST | `/api/v1/elements/{id}/formulas` | Create element formula |
| GET | `/api/v1/elements/{id}/formulas` | List element formulas |
| PUT | `/api/v1/formulas/{formulaId}` | Update formula |
| POST | `/api/v1/formulas/{formulaId}/compile` | Compile and validate formula |

### Payroll Relationships
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/payroll-relationships` | Create payroll relationship |
| GET | `/api/v1/payroll-relationships` | List payroll relationships |
| GET | `/api/v1/payroll-relationships/{id}` | Get relationship details |
| PUT | `/api/v1/payroll-relationships/{id}` | Update relationship |

### Balance Dimensions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/balance-dimensions` | Create balance dimension |
| GET | `/api/v1/balance-dimensions` | List balance dimensions |
| GET | `/api/v1/employees/{employeeId}/balances` | Get employee balance summary |

### Payroll Processing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/payrolls/{id}/run` | Start a payroll process run |
| GET | `/api/v1/process-runs` | List payroll process runs |
| GET | `/api/v1/process-runs/{id}` | Get run details and status |
| POST | `/api/v1/process-runs/{id}/confirm` | Confirm a completed payroll run |
| POST | `/api/v1/process-runs/{id}/reverse` | Reverse a confirmed payroll run |
| POST | `/api/v1/process-runs/{id}/rollback` | Rollback an in-progress run |

### Run Results
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/process-runs/{id}/results` | Get all run results for a process |
| GET | `/api/v1/process-runs/{id}/results/{resultId}` | Get individual run result |
| GET | `/api/v1/process-runs/{id}/results/{resultId}/lines` | Get result line details |

### Payslips
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/payslips` | List payslips with filters |
| GET | `/api/v1/payslips/{id}` | Get payslip detail |
| GET | `/api/v1/employees/{employeeId}/payslips` | Get employee payslip history |
| GET | `/api/v1/payslips/{id}/download` | Download payslip PDF |

### Accounting
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/process-runs/{id}/accounting` | Get accounting entries for a run |
| POST | `/api/v1/process-runs/{id}/post-accounting` | Post payroll accounting to GL |
| GET | `/api/v1/accounting-summary` | Get payroll accounting summary |

---

## 4. Business Rules

1. A payroll run MUST NOT be started if a previous run for the same payroll and period is in `PROCESSING` status.
2. An element MUST belong to exactly one classification and the classification determines its processing order.
3. Payroll formula compilation MUST succeed before the formula can be activated for use in payroll calculations.
4. An employee MUST have exactly one active payroll relationship per payroll definition at any given time.
5. A confirmed payroll run MUST NOT have its results modified; a reversal MUST be performed instead.
6. Net pay for each employee MUST equal gross pay minus total deductions minus total taxes.
7. Payroll accounting entries MUST balance (total debits MUST equal total credits).
8. Retroactive payroll calculations MUST preserve the original run results and create adjustment entries.
9. Payslips MUST be generated only after a payroll run has reached `COMPLETED` status.
10. Elements marked as `INFORMATION` type MUST NOT affect net pay calculations.
11. The system SHOULD support proration of earnings when an employee has a mid-period change.
12. A payroll relationship effective dates MUST NOT overlap for the same employee and payroll.
13. Employer charges MUST be calculated separately from employee deductions and reported distinctly.
14. Payment status on run results MUST be updated to `PAID` only after bank file generation.
15. The system MUST support at least 5 levels of element priority for processing order.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package payroll.v1;

service PayrollService {
    // Payroll definitions
    rpc CreatePayroll(CreatePayrollRequest) returns (CreatePayrollResponse);
    rpc GetPayroll(GetPayrollRequest) returns (GetPayrollResponse);
    rpc ListPayrolls(ListPayrollsRequest) returns (ListPayrollsResponse);
    rpc UpdatePayroll(UpdatePayrollRequest) returns (UpdatePayrollResponse);

    // Elements
    rpc CreateElement(CreateElementRequest) returns (CreateElementResponse);
    rpc GetElement(GetElementRequest) returns (GetElementResponse);
    rpc ListElements(ListElementsRequest) returns (ListElementsResponse);
    rpc CompileFormula(CompileFormulaRequest) returns (CompileFormulaResponse);

    // Relationships
    rpc CreatePayrollRelationship(CreatePayrollRelationshipRequest) returns (CreatePayrollRelationshipResponse);
    rpc GetPayrollRelationship(GetPayrollRelationshipRequest) returns (GetPayrollRelationshipResponse);

    // Processing
    rpc StartPayrollRun(StartPayrollRunRequest) returns (StartPayrollRunResponse);
    rpc GetPayrollRunStatus(GetPayrollRunStatusRequest) returns (GetPayrollRunStatusResponse);
    rpc ConfirmPayrollRun(ConfirmPayrollRunRequest) returns (ConfirmPayrollRunResponse);
    rpc ReversePayrollRun(ReversePayrollRunRequest) returns (ReversePayrollRunResponse);

    // Results
    rpc GetRunResults(GetRunResultsRequest) returns (GetRunResultsResponse);
    rpc GetEmployeeBalances(GetEmployeeBalancesRequest) returns (GetEmployeeBalancesResponse);

    // Payslips
    rpc GetPayslip(GetPayslipRequest) returns (GetPayslipResponse);
    rpc ListEmployeePayslips(ListEmployeePayslipsRequest) returns (ListEmployeePayslipsResponse);
}

message PayrollDefinition {
    string id = 1;
    string tenant_id = 2;
    string payroll_name = 3;
    string payroll_code = 4;
    string payroll_frequency = 5;
    string default_currency_code = 6;
    string processing_status = 7;
}

message CreatePayrollRequest {
    string tenant_id = 1;
    string payroll_name = 2;
    string payroll_code = 3;
    string payroll_frequency = 4;
    string default_currency_code = 5;
    string legislative_data_group_id = 6;
    string created_by = 7;
}

message CreatePayrollResponse {
    PayrollDefinition payroll = 1;
}

message GetPayrollRequest {
    string tenant_id = 1;
    string payroll_id = 2;
}

message GetPayrollResponse {
    PayrollDefinition payroll = 1;
}

message ListPayrollsRequest {
    string tenant_id = 1;
    string status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListPayrollsResponse {
    repeated PayrollDefinition payrolls = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdatePayrollRequest {
    string tenant_id = 1;
    string payroll_id = 2;
    string payroll_name = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdatePayrollResponse {
    PayrollDefinition payroll = 1;
}

message PayrollElement {
    string id = 1;
    string tenant_id = 2;
    string element_code = 3;
    string element_name = 4;
    string element_type = 5;
    string element_classification = 6;
    string calculation_rule = 7;
    int64 default_value = 8;
}

message CreateElementRequest {
    string tenant_id = 1;
    string element_code = 2;
    string element_name = 3;
    string element_type = 4;
    string element_classification = 5;
    string calculation_rule = 6;
    int64 default_value = 7;
    string created_by = 8;
}

message CreateElementResponse {
    PayrollElement element = 1;
}

message GetElementRequest {
    string tenant_id = 1;
    string element_id = 2;
}

message GetElementResponse {
    PayrollElement element = 1;
}

message ListElementsRequest {
    string tenant_id = 1;
    string element_type = 2;
    string element_classification = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListElementsResponse {
    repeated PayrollElement elements = 1;
    string next_page_token = 2;
}

message CompileFormulaRequest {
    string tenant_id = 1;
    string formula_id = 2;
}

message CompileFormulaResponse {
    bool success = 1;
    string compilation_output = 2;
    repeated string errors = 3;
}

message CreatePayrollRelationshipRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string payroll_id = 3;
    string effective_start_date = 4;
    string created_by = 5;
}

message CreatePayrollRelationshipResponse {
    string relationship_id = 1;
    string relationship_number = 2;
}

message GetPayrollRelationshipRequest {
    string tenant_id = 1;
    string relationship_id = 2;
}

message GetPayrollRelationshipResponse {
    string id = 1;
    string employee_id = 2;
    string payroll_id = 3;
    string status = 4;
}

message StartPayrollRunRequest {
    string tenant_id = 1;
    string payroll_id = 2;
    string run_type = 3;
    string period_start_date = 4;
    string period_end_date = 5;
    string pay_date = 6;
    string created_by = 7;
}

message StartPayrollRunResponse {
    string process_run_id = 1;
    string status = 2;
}

message GetPayrollRunStatusRequest {
    string tenant_id = 1;
    string process_run_id = 2;
}

message GetPayrollRunStatusResponse {
    string id = 1;
    string status = 2;
    int32 employee_count = 3;
    int64 total_gross_pay = 4;
    int64 total_net_pay = 5;
    int32 error_count = 6;
}

message ConfirmPayrollRunRequest {
    string tenant_id = 1;
    string process_run_id = 2;
    string confirmed_by = 3;
}

message ConfirmPayrollRunResponse {
    string process_run_id = 1;
    string status = 2;
    string confirmation_date = 3;
}

message ReversePayrollRunRequest {
    string tenant_id = 1;
    string process_run_id = 2;
    string reason = 3;
    string reversed_by = 4;
}

message ReversePayrollRunResponse {
    string reversal_run_id = 1;
    string status = 2;
}

message RunResult {
    string id = 1;
    string employee_id = 2;
    int64 gross_pay = 3;
    int64 net_pay = 4;
    int64 total_deductions = 5;
    int64 total_taxes = 6;
    string processing_status = 7;
}

message GetRunResultsRequest {
    string tenant_id = 1;
    string process_run_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetRunResultsResponse {
    repeated RunResult results = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message BalanceEntry {
    string dimension_name = 1;
    string dimension_type = 2;
    int64 amount = 3;
}

message GetEmployeeBalancesRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string effective_date = 3;
}

message GetEmployeeBalancesResponse {
    repeated BalanceEntry balances = 1;
}

message Payslip {
    string id = 1;
    string payslip_number = 2;
    string employee_id = 3;
    string pay_period_start = 4;
    string pay_period_end = 5;
    int64 gross_pay = 6;
    int64 net_pay = 7;
    int64 total_deductions = 8;
    int64 total_taxes = 9;
}

message GetPayslipRequest {
    string tenant_id = 1;
    string payslip_id = 2;
}

message GetPayslipResponse {
    Payslip payslip = 1;
}

message ListEmployeePayslipsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string year = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListEmployeePayslipsResponse {
    repeated Payslip payslips = 1;
    string next_page_token = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, grade, job, department | Determine payroll population and assignment data |
| `compensation-service` | Salary, bonus, stock option data | Calculate earnings elements |
| `timelabor-service` | Time cards, hours worked | Calculate hourly earnings and overtime |
| `absence-service` | Absence entries, leave balances | Process absence deductions and accruals |
| `benefits-service` | Benefit enrollment, deductions | Calculate benefit deduction elements |
| `tax-management` | Tax codes, withholding rules | Calculate tax elements |
| `gl-service` | Account codes, posting periods | Post payroll accounting entries |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | Payroll accounting journal entries | Record payroll costs in GL |
| `cash-management` | Payment batches, bank files | Execute payroll payments |
| `reporting-service` | Payroll run summaries | Payroll analytics and reports |
| `document-service` | Payslip PDF storage | Archive payslips |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `PayrollRunStarted` | `payroll.run.started` | `{ tenant_id, process_run_id, payroll_id, run_type, period_start, period_end }` | Published when a payroll run begins processing |
| `PayrollRunCompleted` | `payroll.run.completed` | `{ tenant_id, process_run_id, payroll_id, employee_count, total_gross_pay, total_net_pay }` | Published when payroll run finishes calculation |
| `PayslipGenerated` | `payroll.payslip.generated` | `{ tenant_id, payslip_id, employee_id, pay_period_start, pay_period_end, net_pay }` | Published when a payslip is generated |
| `PayrollAccountingCreated` | `payroll.accounting.created` | `{ tenant_id, process_run_id, total_debits, total_credits, journal_batch_id }` | Published when accounting entries are ready |
| `PayrollRunConfirmed` | `payroll.run.confirmed` | `{ tenant_id, process_run_id, confirmed_by, confirmation_date }` | Published when a payroll run is confirmed |
| `PayrollRunReversed` | `payroll.run.reversed` | `{ tenant_id, original_run_id, reversal_run_id, reversed_by }` | Published when a payroll run is reversed |
| `PayrollRunError` | `payroll.run.error` | `{ tenant_id, process_run_id, error_count, error_details }` | Published when a payroll run encounters errors |
