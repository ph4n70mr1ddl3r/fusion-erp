# 54 - Grant Management Service Specification

## 1. Domain Overview

Grant Management provides full lifecycle management of grants from application through award, budgeting, expenditure tracking, billing (draw-down), compliance reporting, and closeout. Supports federal grant compliance including OMB Uniform Guidance, CFDA tracking, cost sharing requirements, indirect cost rate management, and grant-specific financial reporting (SF-425, Federal Audit Clearinghouse). Designed for universities, research institutions, nonprofits, and government contractors receiving federal and non-federal funding.

**Bounded Context:** Grant Lifecycle, Compliance & Federal Award Management
**Service Name:** `grant-service`
**Database:** `data/grant.db`
**HTTP Port:** 8086 | **gRPC Port:** 9086

---

## 2. Database Schema

### 2.1 Grant Programs
```sql
CREATE TABLE grant_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_code TEXT NOT NULL,
    program_name TEXT NOT NULL,
    funding_agency TEXT NOT NULL,
    cfda_number TEXT,
    program_type TEXT NOT NULL
        CHECK(program_type IN ('FEDERAL','STATE','PRIVATE','FOUNDATION','CORPORATE','INTERNAL')),
    description TEXT,
    total_funding_available INTEGER NOT NULL DEFAULT 0,
    application_deadline TEXT,
    award_start_date TEXT,
    award_end_date TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_code)
);

CREATE INDEX idx_grant_programs_tenant ON grant_programs(tenant_id, program_type, is_active);
```

### 2.2 Grants
```sql
CREATE TABLE grants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_number TEXT NOT NULL,
    program_id TEXT NOT NULL,
    grant_title TEXT NOT NULL,
    principal_investigator_id TEXT NOT NULL,
    department_id TEXT,
    status TEXT NOT NULL DEFAULT 'APPLICATION'
        CHECK(status IN ('APPLICATION','SUBMITTED','UNDER_REVIEW','AWARDED','ACTIVE','SUSPENDED','CLOSED_OUT','REJECTED')),
    award_amount INTEGER NOT NULL DEFAULT 0,
    award_date TEXT,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    cost_sharing_required INTEGER NOT NULL DEFAULT 0,
    cost_sharing_amount INTEGER NOT NULL DEFAULT 0,
    indirect_cost_rate REAL NOT NULL DEFAULT 0.0,
    reporting_frequency TEXT NOT NULL DEFAULT 'QUARTERLY'
        CHECK(reporting_frequency IN ('MONTHLY','QUARTERLY','SEMI_ANNUALLY','ANNUALLY')),
    compliance_framework TEXT DEFAULT 'OMB_UNIFORM_GUIDANCE',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES grant_programs(id),
    UNIQUE(tenant_id, grant_number)
);

CREATE INDEX idx_grants_tenant ON grants(tenant_id, status, end_date);
CREATE INDEX idx_grants_pi ON grants(tenant_id, principal_investigator_id);
```

### 2.3 Grant Budgets
```sql
CREATE TABLE grant_budgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_id TEXT NOT NULL,
    budget_period TEXT NOT NULL,
    budget_category TEXT NOT NULL
        CHECK(budget_category IN ('PERSONNEL','FRINGE_BENEFITS','TRAVEL','EQUIPMENT','SUPPLIES',
                                  'CONTRACTUAL','CONSORTIUM','OTHER_DIRECT','INDIRECT_COSTS')),
    budget_amount INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    gl_account_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (grant_id) REFERENCES grants(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, grant_id, budget_period, budget_category)
);

CREATE INDEX idx_grant_budgets_grant ON grant_budgets(grant_id, budget_period);
```

### 2.4 Grant Expenditures
```sql
CREATE TABLE grant_expenditures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_id TEXT NOT NULL,
    expenditure_number TEXT NOT NULL,
    expenditure_date TEXT NOT NULL,
    budget_category TEXT NOT NULL,
    description TEXT NOT NULL,
    amount INTEGER NOT NULL,
    vendor_id TEXT,
    invoice_reference TEXT,
    is_cost_sharing INTEGER NOT NULL DEFAULT 0,
    project_id TEXT,
    task_id TEXT,
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (grant_id) REFERENCES grants(id),
    UNIQUE(tenant_id, expenditure_number)
);

CREATE INDEX idx_grant_exp_grant ON grant_expenditures(grant_id, budget_category, expenditure_date);
CREATE INDEX idx_grant_exp_date ON grant_expenditures(tenant_id, expenditure_date);
```

### 2.5 Grant Billing
```sql
CREATE TABLE grant_billing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_id TEXT NOT NULL,
    billing_number TEXT NOT NULL,
    billing_period_start TEXT NOT NULL,
    billing_period_end TEXT NOT NULL,
    total_expenditures INTEGER NOT NULL DEFAULT 0,
    total_indirect_costs INTEGER NOT NULL DEFAULT 0,
    total_billed INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','PAID','REJECTED')),
    submitted_date TEXT,
    paid_date TEXT,
    payment_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (grant_id) REFERENCES grants(id),
    UNIQUE(tenant_id, billing_number)
);

CREATE INDEX idx_grant_billing_grant ON grant_billing(grant_id, status);
```

### 2.6 Cost Sharing Commitments
```sql
CREATE TABLE cost_sharing_commitments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_id TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('CASH','IN_KIND','VOLUNTEER_TIME','EQUIPMENT_USE','FACILITY_USE')),
    committed_amount INTEGER NOT NULL DEFAULT 0,
    contributed_amount INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    contributed_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (grant_id) REFERENCES grants(id) ON DELETE CASCADE
);

CREATE INDEX idx_cost_sharing_grant ON cost_sharing_commitments(grant_id);
```

### 2.7 Indirect Cost Rates
```sql
CREATE TABLE indirect_cost_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rate_name TEXT NOT NULL,
    rate_type TEXT NOT NULL
        CHECK(rate_type IN ('PREDETERMINED','PROVISIONAL','FINAL','NEGOTIATED')),
    fiscal_year TEXT NOT NULL,
    rate_percentage REAL NOT NULL,
    base_type TEXT NOT NULL DEFAULT 'MTDC'
        CHECK(base_type IN ('MTDC','TDC','TOTAL_COST','SALARIES_WAGES')),
    negotiated_with TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rate_name, fiscal_year)
);

CREATE INDEX idx_indirect_rates_tenant ON indirect_cost_rates(tenant_id, fiscal_year);
```

### 2.8 Grant Reports
```sql
CREATE TABLE grant_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_id TEXT NOT NULL,
    report_type TEXT NOT NULL
        CHECK(report_type IN ('SF425_FEDERAL','SF425_CASH','PROGRESS_REPORT','INVENTION_REPORT',
                              'FINANCIAL_STATUS','FINAL_REPORT','AUDIT_FINDING','CUSTOM')),
    reporting_period_start TEXT NOT NULL,
    reporting_period_end TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','REVIEW','SUBMITTED','ACKNOWLEDGED')),
    report_data TEXT,
    submitted_date TEXT,
    due_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (grant_id) REFERENCES grants(id)
);

CREATE INDEX idx_grant_reports_grant ON grant_reports(grant_id, report_type, status);
CREATE INDEX idx_grant_reports_due ON grant_reports(tenant_id, due_date, status);
```

### 2.9 Grant Closeout
```sql
CREATE TABLE grant_closeout (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    grant_id TEXT NOT NULL,
    closeout_type TEXT NOT NULL
        CHECK(closeout_type IN ('NORMAL','EARLY','NO_COST_EXTENSION','TRANSFER')),
    closeout_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(closeout_status IN ('PENDING','IN_PROGRESS','FINAL_BILLING','AWAITING_APPROVAL','COMPLETED')),
    final_expenditure_total INTEGER NOT NULL DEFAULT 0,
    final_drawdown_amount INTEGER NOT NULL DEFAULT 0,
    unobligated_balance INTEGER NOT NULL DEFAULT 0,
    property_disposition TEXT,
    final_report_submitted INTEGER NOT NULL DEFAULT 0,
    closeout_date TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (grant_id) REFERENCES grants(id),
    UNIQUE(tenant_id, grant_id)
);

CREATE INDEX idx_grant_closeout_status ON grant_closeout(tenant_id, closeout_status);
```

---

## 3. REST API Endpoints

### 3.1 Grant Programs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/programs` | List grant programs |
| POST | `/api/v1/grants/programs` | Create grant program |
| GET | `/api/v1/grants/programs/{id}` | Get program details |
| PUT | `/api/v1/grants/programs/{id}` | Update program |

### 3.2 Grants
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants` | List grants (filter by status, PI, program) |
| POST | `/api/v1/grants` | Create grant application |
| GET | `/api/v1/grants/{id}` | Get grant detail with budget summary |
| PUT | `/api/v1/grants/{id}` | Update grant |
| POST | `/api/v1/grants/{id}/submit` | Submit application |
| POST | `/api/v1/grants/{id}/award` | Record award |
| POST | `/api/v1/grants/{id}/activate` | Activate grant |
| POST | `/api/v1/grants/{id}/closeout` | Initiate closeout |

### 3.3 Budgets
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/{id}/budgets` | Get grant budget |
| POST | `/api/v1/grants/{id}/budgets` | Create/update budget line |
| PUT | `/api/v1/grants/{id}/budgets/{budgetId}` | Update budget line |
| GET | `/api/v1/grants/{id}/budgets/summary` | Budget vs actual summary |

### 3.4 Expenditures
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/{id}/expenditures` | List expenditures |
| POST | `/api/v1/grants/{id}/expenditures` | Record expenditure |
| GET | `/api/v1/grants/{id}/expenditures/{expId}` | Get expenditure detail |

### 3.5 Billing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/grants/{id}/billing/generate` | Generate billing/draw-down |
| GET | `/api/v1/grants/{id}/billing` | List billings |
| POST | `/api/v1/grants/{id}/billing/{billId}/submit` | Submit billing |
| GET | `/api/v1/grants/{id}/billing/{billId}` | Get billing detail |

### 3.6 Cost Sharing
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/{id}/cost-sharing` | List cost sharing commitments |
| POST | `/api/v1/grants/{id}/cost-sharing` | Record cost sharing |
| PUT | `/api/v1/grants/{id}/cost-sharing/{csId}` | Update commitment |

### 3.7 Indirect Cost Rates
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/indirect-rates` | List indirect cost rates |
| POST | `/api/v1/grants/indirect-rates` | Create rate |
| PUT | `/api/v1/grants/indirect-rates/{id}` | Update rate |

### 3.8 Reports
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/{id}/reports` | List reports |
| POST | `/api/v1/grants/{id}/reports/generate` | Generate report |
| POST | `/api/v1/grants/{id}/reports/{reportId}/submit` | Submit report |
| GET | `/api/v1/grants/reports/upcoming` | List upcoming report deadlines |

### 3.9 Compliance
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grants/{id}/compliance` | Check compliance status |
| GET | `/api/v1/grants/compliance/dashboard` | Compliance dashboard |

---

## 4. Business Rules

1. Grants MUST have a defined budget before any expenditures can be recorded.
2. Total expenditures MUST NOT exceed the award amount plus approved amendments.
3. Expenditures MUST be categorized into valid budget categories.
4. Indirect costs MUST be calculated using the applicable negotiated rate for the period.
5. Cost sharing commitments MUST total at least the required cost sharing amount before grant closeout.
6. Billing (draw-down) amounts MUST NOT exceed cumulative expenditures minus prior billings.
7. SF-425 reports MUST be generated per the grant's reporting frequency.
8. Grant closeout MUST verify: all expenditures recorded, final billing submitted, final report filed, property disposition addressed.
9. Grant periods MAY be extended via no-cost extensions recorded as amendments.
10. Compliance violations MUST be flagged and reported to the principal investigator and compliance officer.
11. Budget categories MUST follow the sponsor's required budget structure.
12. Federal grants MUST track CFDA numbers for Federal Audit Clearinghouse reporting.

---

## 5. gRPC Service Definition

```protobuf
service GrantService {
    rpc CreateGrant(CreateGrantRequest) returns (GrantResponse);
    rpc GetGrant(GetGrantRequest) returns (GrantDetailResponse);
    rpc ListGrants(ListGrantsRequest) returns (ListGrantsResponse);

    rpc CreateBudgetLine(CreateBudgetLineRequest) returns (BudgetLineResponse);
    rpc GetBudgetSummary(GetBudgetSummaryRequest) returns (BudgetSummaryResponse);

    rpc RecordExpenditure(RecordExpenditureRequest) returns (ExpenditureResponse);
    rpc ListExpenditures(ListExpendituresRequest) returns (ListExpendituresResponse);

    rpc GenerateBilling(GenerateBillingRequest) returns (BillingResponse);
    rpc SubmitBilling(SubmitBillingRequest) returns (BillingResponse);

    rpc GenerateReport(GenerateReportRequest) returns (ReportResponse);
    rpc SubmitReport(SubmitReportRequest) returns (ReportResponse);

    rpc CheckCompliance(CheckComplianceRequest) returns (ComplianceResponse);
    rpc InitiateCloseout(InitiateCloseoutRequest) returns (CloseoutResponse);
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `gl-service` | GL accounts for budget categories, journal entries |
| `pm-service` | Project costs, task expenditures, labor charges |
| `tax-service` | Tax-exempt status for federal grants |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | Grant accounting entries, indirect cost entries |
| `pm-service` | Grant budget availability, expenditure validation |
| `ap-service` | Grant-related vendor invoices |
| `report-service` | Grant financial reports, compliance dashboards |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `grants.program.created` | `{program_id, code, agency}` | Grant program created |
| `grants.grant.awarded` | `{grant_id, number, award_amount}` | Grant awarded |
| `grants.grant.activated` | `{grant_id, start_date, end_date}` | Grant activated |
| `grants.expenditure.recorded` | `{grant_id, category, amount}` | Expenditure recorded |
| `grants.expenditure.over-budget` | `{grant_id, category, budget, actual}` | Budget exceeded |
| `grants.billing.generated` | `{grant_id, billing_id, amount}` | Billing/draw-down generated |
| `grants.billing.submitted` | `{grant_id, billing_id}` | Billing submitted |
| `grants.report.generated` | `{grant_id, report_type, period}` | Report generated |
| `grants.report.due` | `{grant_id, report_type, due_date}` | Report due reminder |
| `grants.compliance.violation` | `{grant_id, violation_type}` | Compliance violation |
| `grants.closeout.initiated` | `{grant_id, type}` | Closeout started |
| `grants.closeout.completed` | `{grant_id, final_total}` | Closeout completed |
