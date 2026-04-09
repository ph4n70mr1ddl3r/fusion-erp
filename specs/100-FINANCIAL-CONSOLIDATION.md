# 100 - Financial Consolidation & Close Service Specification

## 1. Domain Overview

Financial Consolidation & Close provides multi-entity financial consolidation supporting legal entity registration with ownership structures, multiple consolidation methods (full, equity, proportional, cost), currency translation using closing/average/historical/spot rates, automated and manual intercompany eliminations, minority interest calculations, ownership structure management with effective-dated changes, consolidation period lifecycle management with close task orchestration, close task dependency tracking, configurable consolidation rules for eliminations and reclassifications, intercompany balance matching and reconciliation, and consolidated financial reporting including trial balance, balance sheet, income statement, and cash flow statement generation. Integrates with GL for entity trial balances, TAX for tax provisions, and IC for intercompany transaction details.

**Bounded Context:** Multi-Entity Financial Consolidation & Close Process Management
**Service Name:** `consolidation-service`
**Database:** `data/consolidation.db`
**HTTP Port:** 8137 | **gRPC Port:** 9137

---

## 2. Database Schema

### 2.1 Consolidation Entities
```sql
CREATE TABLE consolidation_entities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    entity_code TEXT NOT NULL,
    legal_entity_id TEXT NOT NULL,
    country TEXT NOT NULL,
    currency_code TEXT NOT NULL,
    ownership_percentage REAL NOT NULL DEFAULT 100.0,
    consolidation_method TEXT NOT NULL DEFAULT 'FULL'
        CHECK(consolidation_method IN ('FULL','EQUITY','PROPORTIONAL','COST')),
    is_sub_consolidating INTEGER NOT NULL DEFAULT 0,
    parent_entity_id TEXT,
    consolidation_chart_of_accounts_id TEXT,
    fiscal_calendar_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, entity_code)
);

CREATE INDEX idx_consolidation_entities_tenant_status ON consolidation_entities(tenant_id, status);
CREATE INDEX idx_consolidation_entities_tenant_parent ON consolidation_entities(tenant_id, parent_entity_id);
CREATE INDEX idx_consolidation_entities_tenant_country ON consolidation_entities(tenant_id, country);
CREATE INDEX idx_consolidation_entities_tenant_method ON consolidation_entities(tenant_id, consolidation_method);
CREATE INDEX idx_consolidation_entities_tenant_active ON consolidation_entities(tenant_id, is_active);
```

### 2.2 Ownership Structures
```sql
CREATE TABLE ownership_structures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    parent_entity_id TEXT NOT NULL,
    child_entity_id TEXT NOT NULL,
    ownership_percentage REAL NOT NULL,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    consolidation_method TEXT NOT NULL DEFAULT 'FULL'
        CHECK(consolidation_method IN ('FULL','EQUITY','PROPORTIONAL','COST')),
    voting_interest_pct REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_entity_id) REFERENCES consolidation_entities(id),
    FOREIGN KEY (child_entity_id) REFERENCES consolidation_entities(id)
);

CREATE INDEX idx_ownership_tenant_parent ON ownership_structures(tenant_id, parent_entity_id);
CREATE INDEX idx_ownership_tenant_child ON ownership_structures(tenant_id, child_entity_id);
CREATE INDEX idx_ownership_tenant_effective ON ownership_structures(tenant_id, effective_from, effective_to);
CREATE INDEX idx_ownership_tenant_active ON ownership_structures(tenant_id, is_active);
```

### 2.3 Consolidation Periods
```sql
CREATE TABLE consolidation_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('MONTHLY','QUARTERLY','ANNUAL')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_PROGRESS','SUBMITTED','APPROVED','CLOSED')),
    close_deadline TEXT NOT NULL,
    responsible_user_id TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_name)
);

CREATE INDEX idx_consolidation_periods_tenant_status ON consolidation_periods(tenant_id, status);
CREATE INDEX idx_consolidation_periods_tenant_dates ON consolidation_periods(tenant_id, start_date, end_date);
CREATE INDEX idx_consolidation_periods_tenant_type ON consolidation_periods(tenant_id, period_type);
CREATE INDEX idx_consolidation_periods_tenant_responsible ON consolidation_periods(tenant_id, responsible_user_id);
CREATE INDEX idx_consolidation_periods_tenant_active ON consolidation_periods(tenant_id, is_active);
```

### 2.4 Consolidation Journals
```sql
CREATE TABLE consolidation_journals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    journal_type TEXT NOT NULL
        CHECK(journal_type IN ('ELIMINATION','CURRENCY_ADJUSTMENT','RECLASS','MINORITY_INTEREST','EQUITY_PICKUP')),
    debit_entity_id TEXT NOT NULL,
    credit_entity_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL,
    translated_amount_cents INTEGER NOT NULL DEFAULT 0,
    exchange_rate REAL NOT NULL DEFAULT 1.0,
    description TEXT,
    reference TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','POSTED','REVERSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES consolidation_periods(id),
    FOREIGN KEY (debit_entity_id) REFERENCES consolidation_entities(id),
    FOREIGN KEY (credit_entity_id) REFERENCES consolidation_entities(id)
);

CREATE INDEX idx_consolidation_journals_tenant_period ON consolidation_journals(tenant_id, period_id);
CREATE INDEX idx_consolidation_journals_tenant_type ON consolidation_journals(tenant_id, journal_type);
CREATE INDEX idx_consolidation_journals_tenant_status ON consolidation_journals(tenant_id, status);
CREATE INDEX idx_consolidation_journals_tenant_entity ON consolidation_journals(tenant_id, debit_entity_id);
CREATE INDEX idx_consolidation_journals_tenant_account ON consolidation_journals(tenant_id, account_id);
```

### 2.5 Currency Translations
```sql
CREATE TABLE currency_translations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    from_currency TEXT NOT NULL,
    to_currency TEXT NOT NULL,
    exchange_rate_type TEXT NOT NULL
        CHECK(exchange_rate_type IN ('CLOSING','AVERAGE','HISTORICAL','SPOT')),
    exchange_rate REAL NOT NULL,
    effective_date TEXT NOT NULL,
    rate_source TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES consolidation_periods(id),
    UNIQUE(tenant_id, period_id, from_currency, to_currency, exchange_rate_type, effective_date)
);

CREATE INDEX idx_currency_trans_tenant_period ON currency_translations(tenant_id, period_id);
CREATE INDEX idx_currency_trans_tenant_pair ON currency_translations(tenant_id, from_currency, to_currency);
CREATE INDEX idx_currency_trans_tenant_type ON currency_translations(tenant_id, exchange_rate_type);
CREATE INDEX idx_currency_trans_tenant_date ON currency_translations(tenant_id, effective_date);
CREATE INDEX idx_currency_trans_tenant_active ON currency_translations(tenant_id, is_active);
```

### 2.6 Close Tasks
```sql
CREATE TABLE close_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    task_name TEXT NOT NULL,
    task_description TEXT,
    task_category TEXT NOT NULL
        CHECK(task_category IN ('DATA_COLLECTION','VALIDATION','ELIMINATION','REPORTING','REVIEW','APPROVAL')),
    assignee_id TEXT NOT NULL,
    entity_id TEXT,
    due_date TEXT NOT NULL,
    completed_date TEXT,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED','BLOCKED','WAIVED')),
    dependency_task_ids TEXT,                        -- JSON: array of prerequisite task IDs
    estimated_hours REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES consolidation_periods(id),
    FOREIGN KEY (entity_id) REFERENCES consolidation_entities(id)
);

CREATE INDEX idx_close_tasks_tenant_period ON close_tasks(tenant_id, period_id);
CREATE INDEX idx_close_tasks_tenant_assignee ON close_tasks(tenant_id, assignee_id);
CREATE INDEX idx_close_tasks_tenant_status ON close_tasks(tenant_id, status);
CREATE INDEX idx_close_tasks_tenant_category ON close_tasks(tenant_id, task_category);
CREATE INDEX idx_close_tasks_tenant_entity ON close_tasks(tenant_id, entity_id);
CREATE INDEX idx_close_tasks_tenant_due ON close_tasks(tenant_id, due_date);
```

### 2.7 Consolidation Rules
```sql
CREATE TABLE consolidation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('ELIMINATION','RECLASSIFICATION','CURRENCY','MINORITY_INTEREST')),
    source_criteria TEXT NOT NULL,                   -- JSON: source account/entity filters
    target_criteria TEXT NOT NULL,                   -- JSON: target account/entity mappings
    conditions TEXT,                                 -- JSON: additional rule conditions
    priority INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    last_execution_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_consolidation_rules_tenant_type ON consolidation_rules(tenant_id, rule_type);
CREATE INDEX idx_consolidation_rules_tenant_active ON consolidation_rules(tenant_id, is_active);
CREATE INDEX idx_consolidation_rules_tenant_priority ON consolidation_rules(tenant_id, priority);
```

### 2.8 Intercompany Balances
```sql
CREATE TABLE intercompany_balances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    from_entity_id TEXT NOT NULL,
    to_entity_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    from_entity_amount_cents INTEGER NOT NULL,
    to_entity_amount_cents INTEGER NOT NULL,
    difference_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'UNMATCHED'
        CHECK(status IN ('MATCHED','UNMATCHED','PARTIALLY_MATCHED')),
    resolution TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES consolidation_periods(id),
    FOREIGN KEY (from_entity_id) REFERENCES consolidation_entities(id),
    FOREIGN KEY (to_entity_id) REFERENCES consolidation_entities(id)
);

CREATE INDEX idx_ic_balances_tenant_period ON intercompany_balances(tenant_id, period_id);
CREATE INDEX idx_ic_balances_tenant_from ON intercompany_balances(tenant_id, from_entity_id);
CREATE INDEX idx_ic_balances_tenant_to ON intercompany_balances(tenant_id, to_entity_id);
CREATE INDEX idx_ic_balances_tenant_status ON intercompany_balances(tenant_id, status);
CREATE INDEX idx_ic_balances_tenant_account ON intercompany_balances(tenant_id, account_id);
```

---

## 3. REST API Endpoints

### 3.1 Entities
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/entities` | List consolidation entities |
| POST | `/api/v1/consolidation/entities` | Create consolidation entity |
| GET | `/api/v1/consolidation/entities/{id}` | Get entity details |
| PUT | `/api/v1/consolidation/entities/{id}` | Update entity |

### 3.2 Ownership Structures
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/ownership` | List ownership structures |
| POST | `/api/v1/consolidation/ownership` | Create ownership structure |
| GET | `/api/v1/consolidation/ownership/{id}` | Get ownership details |
| PUT | `/api/v1/consolidation/ownership/{id}` | Update ownership structure |
| GET | `/api/v1/consolidation/ownership/structure-tree` | Get ownership hierarchy tree |

### 3.3 Periods
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/periods` | List consolidation periods |
| POST | `/api/v1/consolidation/periods` | Create consolidation period |
| GET | `/api/v1/consolidation/periods/{id}` | Get period details |
| PUT | `/api/v1/consolidation/periods/{id}` | Update period |
| POST | `/api/v1/consolidation/periods/{id}/open` | Open period for consolidation |
| POST | `/api/v1/consolidation/periods/{id}/close` | Close consolidation period |
| POST | `/api/v1/consolidation/periods/{id}/lock` | Lock period from changes |

### 3.4 Journals
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/periods/{id}/journals` | List consolidation journals |
| POST | `/api/v1/consolidation/periods/{id}/journals` | Create consolidation journal |
| GET | `/api/v1/consolidation/journals/{id}` | Get journal details |
| PUT | `/api/v1/consolidation/journals/{id}` | Update journal |
| POST | `/api/v1/consolidation/journals/{id}/post` | Post consolidation journal |
| POST | `/api/v1/consolidation/journals/{id}/reverse` | Reverse posted journal |
| POST | `/api/v1/consolidation/periods/{id}/auto-eliminate` | Run automatic eliminations |

### 3.5 Currency
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/currency/rates` | Get exchange rates |
| POST | `/api/v1/consolidation/currency/rates` | Set exchange rate |
| POST | `/api/v1/consolidation/currency/translate` | Translate amounts to group currency |
| POST | `/api/v1/consolidation/currency/calculate-ctd` | Calculate cumulative translation adjustment |

### 3.6 Close Tasks
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/periods/{id}/tasks` | List close tasks for period |
| POST | `/api/v1/consolidation/periods/{id}/tasks` | Create close task |
| GET | `/api/v1/consolidation/tasks/{id}` | Get task details |
| PUT | `/api/v1/consolidation/tasks/{id}` | Update task |
| POST | `/api/v1/consolidation/tasks/{id}/complete` | Mark task as completed |
| POST | `/api/v1/consolidation/tasks/{id}/reassign` | Reassign task to another user |
| GET | `/api/v1/consolidation/periods/{id}/progress-dashboard` | Get close progress dashboard |

### 3.7 Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/rules` | List consolidation rules |
| POST | `/api/v1/consolidation/rules` | Create consolidation rule |
| GET | `/api/v1/consolidation/rules/{id}` | Get rule details |
| PUT | `/api/v1/consolidation/rules/{id}` | Update rule |
| POST | `/api/v1/consolidation/rules/{id}/execute` | Execute a consolidation rule |

### 3.8 Intercompany
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/periods/{id}/intercompany-balances` | Get intercompany balances |
| POST | `/api/v1/consolidation/intercompany/match` | Match intercompany balances |
| POST | `/api/v1/consolidation/periods/{id}/auto-eliminate-ic` | Auto-eliminate matched intercompany |

### 3.9 Reports
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/consolidation/reports/consolidated-trial-balance` | Get consolidated trial balance |
| GET | `/api/v1/consolidation/reports/balance-sheet` | Get consolidated balance sheet |
| GET | `/api/v1/consolidation/reports/income-statement` | Get consolidated income statement |
| GET | `/api/v1/consolidation/reports/cash-flow` | Get consolidated cash flow statement |

---

## 4. Business Rules

### 4.1 Entity and Ownership Management
1. Each consolidation entity MUST have a unique entity_code within a tenant.
2. Ownership percentage MUST be between 0.00 and 100.00 inclusive.
3. Entities with ownership_percentage >= 50.01 MUST use the FULL consolidation method by default.
4. Entities with ownership_percentage between 20.00 and 50.00 SHOULD use the EQUITY method by default.
5. Entities with ownership_percentage below 20.00 SHOULD use the COST method by default.
6. Ownership structures MUST have valid effective_from dates; overlapping effective periods for the same parent-child pair MUST NOT be allowed.
7. A parent entity MUST NOT be its own descendant (circular ownership detection).

### 4.2 Period Management
8. Consolidation period status transitions MUST follow: OPEN -> IN_PROGRESS -> SUBMITTED -> APPROVED -> CLOSED.
9. A CLOSED period MUST NOT allow journal entries, rate changes, or task modifications.
10. Only one period per period_type MAY be in IN_PROGRESS status at any time within a tenant.
11. The close_deadline MUST be after the period end_date.

### 4.3 Journal Processing
12. Consolidation journals MUST balance: the sum of debits MUST equal the sum of credits across all entities for each journal entry.
13. Elimination journals MUST reference both a debit_entity_id and a credit_entity_id that are different entities within the same consolidation group.
14. Posted journals MUST NOT be modified; corrections MUST be made through reversal journals.
15. Currency adjustment journals MUST use exchange rates from the currency_translations table for the corresponding period.

### 4.4 Currency Translation
16. Balance sheet items (assets and liabilities) MUST be translated using the CLOSING exchange rate.
17. Income statement items MUST be translated using the AVERAGE exchange rate for the period.
18. Equity items MUST be translated using HISTORICAL exchange rates from the date of original transaction.
19. Cumulative translation adjustments (CTA) MUST be recorded in a dedicated equity account as part of other comprehensive income.

### 4.5 Close Task Orchestration
20. A close task with unresolved dependencies (dependency_task_ids) MUST remain in NOT_STARTED or BLOCKED status until all dependencies are COMPLETED or WAIVED.
21. Task completion MUST update the period's overall close progress percentage.
22. Tasks past their due_date with status NOT_STARTED or IN_PROGRESS SHOULD trigger escalation alerts.
23. The close process MUST NOT be submitted (period status SUBMITTED) until all mandatory tasks are COMPLETED or WAIVED.

### 4.6 Intercompany Eliminations
24. Intercompany balances with status UNMATCHED MUST NOT be auto-eliminated; they MUST be manually matched or resolved first.
25. The difference_cents for matched intercompany balances MUST be zero; PARTIALLY_MATCHED balances MUST document the difference resolution.
26. Elimination rules MUST execute in priority order; lower priority numbers MUST execute first.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.consolidation.v1;

service FinancialConsolidationService {
    // Entity management
    rpc CreateEntity(CreateEntityRequest) returns (CreateEntityResponse);
    rpc GetEntity(GetEntityRequest) returns (GetEntityResponse);
    rpc ListEntities(ListEntitiesRequest) returns (ListEntitiesResponse);

    // Ownership structures
    rpc CreateOwnership(CreateOwnershipRequest) returns (CreateOwnershipResponse);
    rpc GetOwnershipTree(GetOwnershipTreeRequest) returns (GetOwnershipTreeResponse);

    // Period management
    rpc CreatePeriod(CreatePeriodRequest) returns (CreatePeriodResponse);
    rpc GetPeriod(GetPeriodRequest) returns (GetPeriodResponse);
    rpc ListPeriods(ListPeriodsRequest) returns (ListPeriodsResponse);
    rpc OpenPeriod(OpenPeriodRequest) returns (OpenPeriodResponse);
    rpc ClosePeriod(ClosePeriodRequest) returns (ClosePeriodResponse);

    // Journals
    rpc CreateJournal(CreateJournalRequest) returns (CreateJournalResponse);
    rpc PostJournal(PostJournalRequest) returns (PostJournalResponse);
    rpc ReverseJournal(ReverseJournalRequest) returns (ReverseJournalResponse);
    rpc AutoEliminate(AutoEliminateRequest) returns (AutoEliminateResponse);

    // Currency translation
    rpc SetExchangeRate(SetExchangeRateRequest) returns (SetExchangeRateResponse);
    rpc TranslateAmounts(TranslateAmountsRequest) returns (TranslateAmountsResponse);
    rpc CalculateCTD(CalculateCTDRequest) returns (CalculateCTDResponse);

    // Close tasks
    rpc CreateCloseTask(CreateCloseTaskRequest) returns (CreateCloseTaskResponse);
    rpc CompleteCloseTask(CompleteCloseTaskRequest) returns (CompleteCloseTaskResponse);
    rpc GetCloseProgress(GetCloseProgressRequest) returns (GetCloseProgressResponse);

    // Rules
    rpc CreateRule(CreateRuleRequest) returns (CreateRuleResponse);
    rpc ExecuteRule(ExecuteRuleRequest) returns (ExecuteRuleResponse);

    // Intercompany
    rpc GetIntercompanyBalances(GetIntercompanyBalancesRequest) returns (GetIntercompanyBalancesResponse);
    rpc MatchIntercompany(MatchIntercompanyRequest) returns (MatchIntercompanyResponse);

    // Reports
    rpc GetConsolidatedTrialBalance(GetConsolidatedTrialBalanceRequest) returns (GetConsolidatedTrialBalanceResponse);
    rpc GetConsolidatedBalanceSheet(GetConsolidatedBalanceSheetRequest) returns (GetConsolidatedBalanceSheetResponse);
    rpc GetConsolidatedIncomeStatement(GetConsolidatedIncomeStatementRequest) returns (GetConsolidatedIncomeStatementResponse);
}

message CreateEntityRequest {
    string tenant_id = 1;
    string entity_name = 2;
    string entity_code = 3;
    string legal_entity_id = 4;
    string country = 5;
    string currency_code = 6;
    double ownership_percentage = 7;
    string consolidation_method = 8;
    bool is_sub_consolidating = 9;
    string parent_entity_id = 10;
    string consolidation_chart_of_accounts_id = 11;
    string fiscal_calendar_id = 12;
}

message CreateEntityResponse {
    string entity_id = 1;
    string entity_code = 2;
    string consolidation_method = 3;
}

message GetEntityRequest {
    string tenant_id = 1;
    string entity_id = 2;
}

message GetEntityResponse {
    string entity_id = 1;
    string entity_name = 2;
    string entity_code = 3;
    string country = 4;
    string currency_code = 5;
    double ownership_percentage = 6;
    string consolidation_method = 7;
    string parent_entity_id = 8;
    bool is_sub_consolidating = 9;
    string status = 10;
}

message ListEntitiesRequest {
    string tenant_id = 1;
    string status = 2;
    string country = 3;
    string consolidation_method = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListEntitiesResponse {
    repeated GetEntityResponse entities = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CreateOwnershipRequest {
    string tenant_id = 1;
    string parent_entity_id = 2;
    string child_entity_id = 3;
    double ownership_percentage = 4;
    string effective_from = 5;
    string effective_to = 6;
    string consolidation_method = 7;
    double voting_interest_pct = 8;
}

message CreateOwnershipResponse {
    string ownership_id = 1;
    string parent_entity_id = 2;
    string child_entity_id = 3;
    double ownership_percentage = 4;
}

message GetOwnershipTreeRequest {
    string tenant_id = 1;
    string root_entity_id = 2;
    string effective_date = 3;
}

message GetOwnershipTreeResponse {
    repeated OwnershipNode nodes = 1;
}

message OwnershipNode {
    string entity_id = 1;
    string entity_name = 2;
    string entity_code = 3;
    string parent_entity_id = 4;
    double ownership_percentage = 5;
    string consolidation_method = 6;
    repeated OwnershipNode children = 7;
}

message CreatePeriodRequest {
    string tenant_id = 1;
    string period_name = 2;
    string period_type = 3;
    string start_date = 4;
    string end_date = 5;
    string close_deadline = 6;
    string responsible_user_id = 7;
}

message CreatePeriodResponse {
    string period_id = 1;
    string period_name = 2;
    string status = 3;
}

message GetPeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetPeriodResponse {
    string period_id = 1;
    string period_name = 2;
    string period_type = 3;
    string start_date = 4;
    string end_date = 5;
    string status = 6;
    string close_deadline = 7;
    string responsible_user_id = 8;
    double close_progress_pct = 9;
}

message ListPeriodsRequest {
    string tenant_id = 1;
    string status = 2;
    string period_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPeriodsResponse {
    repeated GetPeriodResponse periods = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message OpenPeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message OpenPeriodResponse {
    string period_id = 1;
    string status = 2;
}

message ClosePeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message ClosePeriodResponse {
    string period_id = 1;
    string status = 2;
}

message CreateJournalRequest {
    string tenant_id = 1;
    string period_id = 2;
    string journal_type = 3;
    string debit_entity_id = 4;
    string credit_entity_id = 5;
    string account_id = 6;
    int64 amount_cents = 7;
    string currency_code = 8;
    double exchange_rate = 9;
    string description = 10;
    string reference = 11;
}

message CreateJournalResponse {
    string journal_id = 1;
    string journal_type = 2;
    int64 amount_cents = 3;
    int64 translated_amount_cents = 4;
    string status = 5;
}

message PostJournalRequest {
    string tenant_id = 1;
    string journal_id = 2;
}

message PostJournalResponse {
    string journal_id = 1;
    string status = 2;
}

message ReverseJournalRequest {
    string tenant_id = 1;
    string journal_id = 2;
    string reason = 3;
}

message ReverseJournalResponse {
    string reversal_journal_id = 1;
    string original_journal_id = 2;
    string status = 3;
}

message AutoEliminateRequest {
    string tenant_id = 1;
    string period_id = 2;
    repeated string rule_ids = 3;
}

message AutoEliminateResponse {
    string period_id = 1;
    int32 journals_created = 2;
    int64 total_elimination_amount_cents = 3;
}

message SetExchangeRateRequest {
    string tenant_id = 1;
    string period_id = 2;
    string from_currency = 3;
    string to_currency = 4;
    string exchange_rate_type = 5;
    double exchange_rate = 6;
    string effective_date = 7;
    string rate_source = 8;
}

message SetExchangeRateResponse {
    string rate_id = 1;
    double exchange_rate = 2;
}

message TranslateAmountsRequest {
    string tenant_id = 1;
    string period_id = 2;
    string entity_id = 3;
    string to_currency = 4;
    string exchange_rate_type = 5;
}

message TranslateAmountsResponse {
    string entity_id = 1;
    int32 lines_translated = 2;
    int64 total_translated_cents = 3;
}

message CalculateCTDRequest {
    string tenant_id = 1;
    string period_id = 2;
    string entity_id = 3;
}

message CalculateCTDResponse {
    string entity_id = 1;
    int64 ctd_amount_cents = 2;
    double closing_rate = 3;
    double average_rate = 4;
}

message CreateCloseTaskRequest {
    string tenant_id = 1;
    string period_id = 2;
    string task_name = 3;
    string task_description = 4;
    string task_category = 5;
    string assignee_id = 6;
    string entity_id = 7;
    string due_date = 8;
    string dependency_task_ids = 9;
    double estimated_hours = 10;
}

message CreateCloseTaskResponse {
    string task_id = 1;
    string task_name = 2;
    string status = 3;
}

message CompleteCloseTaskRequest {
    string tenant_id = 1;
    string task_id = 2;
    string completed_date = 3;
}

message CompleteCloseTaskResponse {
    string task_id = 1;
    string status = 2;
    string completed_date = 3;
}

message GetCloseProgressRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetCloseProgressResponse {
    string period_id = 1;
    int32 total_tasks = 2;
    int32 completed_tasks = 3;
    int32 blocked_tasks = 4;
    int32 overdue_tasks = 5;
    double progress_percentage = 6;
    repeated CloseTaskInfo tasks = 7;
}

message CloseTaskInfo {
    string task_id = 1;
    string task_name = 2;
    string task_category = 3;
    string assignee_id = 4;
    string status = 5;
    string due_date = 6;
    string completed_date = 7;
}

message CreateRuleRequest {
    string tenant_id = 1;
    string rule_name = 2;
    string rule_type = 3;
    string source_criteria = 4;
    string target_criteria = 5;
    string conditions = 6;
    int32 priority = 7;
}

message CreateRuleResponse {
    string rule_id = 1;
    string rule_name = 2;
    string rule_type = 3;
}

message ExecuteRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string period_id = 3;
}

message ExecuteRuleResponse {
    string rule_id = 1;
    string period_id = 2;
    int32 journals_created = 3;
    int64 total_amount_cents = 4;
}

message GetIntercompanyBalancesRequest {
    string tenant_id = 1;
    string period_id = 2;
    string from_entity_id = 3;
    string to_entity_id = 4;
    string status = 5;
}

message GetIntercompanyBalancesResponse {
    repeated IntercompanyBalanceInfo balances = 1;
    int32 matched_count = 2;
    int32 unmatched_count = 3;
    int64 total_difference_cents = 4;
}

message IntercompanyBalanceInfo {
    string balance_id = 1;
    string from_entity_id = 2;
    string to_entity_id = 3;
    string account_id = 4;
    int64 from_entity_amount_cents = 5;
    int64 to_entity_amount_cents = 6;
    int64 difference_cents = 7;
    string status = 8;
}

message MatchIntercompanyRequest {
    string tenant_id = 1;
    string balance_id = 2;
    string resolution = 3;
}

message MatchIntercompanyResponse {
    string balance_id = 1;
    string status = 2;
}

message GetConsolidatedTrialBalanceRequest {
    string tenant_id = 1;
    string period_id = 2;
    string entity_id = 3;
}

message GetConsolidatedTrialBalanceResponse {
    string period_id = 1;
    repeated TrialBalanceLine lines = 2;
    int64 total_debit_cents = 3;
    int64 total_credit_cents = 4;
}

message TrialBalanceLine {
    string account_id = 1;
    string account_name = 2;
    int64 debit_cents = 3;
    int64 credit_cents = 4;
    int64 net_balance_cents = 5;
    string currency_code = 6;
}

message GetConsolidatedBalanceSheetRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetConsolidatedBalanceSheetResponse {
    string period_id = 1;
    repeated BalanceSheetLine assets = 2;
    repeated BalanceSheetLine liabilities = 3;
    repeated BalanceSheetLine equity = 4;
    int64 total_assets_cents = 5;
    int64 total_liabilities_equity_cents = 6;
}

message BalanceSheetLine {
    string account_id = 1;
    string account_name = 2;
    int64 amount_cents = 3;
    string section = 4;
    string sub_section = 5;
}

message GetConsolidatedIncomeStatementRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetConsolidatedIncomeStatementResponse {
    string period_id = 1;
    repeated IncomeStatementLine revenue = 2;
    repeated IncomeStatementLine expenses = 3;
    repeated IncomeStatementLine other_income_expense = 4;
    int64 total_revenue_cents = 5;
    int64 total_expenses_cents = 6;
    int64 net_income_cents = 7;
    int64 minority_interest_cents = 8;
}

message IncomeStatementLine {
    string account_id = 1;
    string account_name = 2;
    int64 amount_cents = 3;
    string section = 4;
    string sub_section = 5;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `gl-service` | Entity trial balances, chart of accounts, posted journal details, account balances |
| `ic-service` | Intercompany transaction details, intercompany AR/AP balances |
| `tax-service` | Tax provision calculations, deferred tax balances, tax rate data |
| `auth-service` | User identity for task assignment, approval authority, audit trails |
| `inv-service` | Inventory valuation data for currency translation of balance sheet items |
| `ar-service` | Receivables aging data for intercompany reconciliation |
| `ap-service` | Payables data for intercompany matching and elimination |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | Consolidation journal entries (eliminations, CTA, minority interest) |
| `reporting-service` | Consolidated financial statements, close process dashboards, variance reports |
| `notification-service` | Close task reminders, overdue alerts, period status change notifications |
| `workflow-service` | Close task approval routing, period close approval workflows |
| `tax-service` | Consolidated pre-tax income for tax provision calculations |
| `audit-service` | Consolidation journal audit trail, period close audit log |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `consolidation.period.opened` | `consolidation.events` | `{ period_id, period_name, period_type, start_date, end_date, responsible_user_id }` | Consolidation period opened |
| `consolidation.eliminations.completed` | `consolidation.events` | `{ period_id, journals_created, total_elimination_amount_cents }` | Auto-eliminations completed |
| `consolidation.close-task.completed` | `consolidation.events` | `{ task_id, period_id, task_name, assignee_id, completed_date }` | Close task completed |
| `consolidation.period.closed` | `consolidation.events` | `{ period_id, period_name, closed_at, total_entities, total_journals }` | Consolidation period closed |
| `consolidation.intercompany.imbalance` | `consolidation.events` | `{ period_id, from_entity_id, to_entity_id, account_id, difference_cents }` | Intercompany imbalance detected |

---

## 8. Migrations

1. V001: `consolidation_entities`
2. V002: `ownership_structures`
3. V003: `consolidation_periods`
4. V004: `consolidation_journals`
5. V005: `currency_translations`
6. V006: `close_tasks`
7. V007: `consolidation_rules`
8. V008: `intercompany_balances`
9. V009: Triggers for `updated_at`
