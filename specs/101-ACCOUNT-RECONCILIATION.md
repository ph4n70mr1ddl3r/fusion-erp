# 101 - Account Reconciliation Service Specification

## 1. Domain Overview

Account Reconciliation provides automated account reconciliation supporting balance comparisons, transaction matching, auto-certification, reconciliation templates, risk-based prioritization, and close integration. It delivers a matching rules engine, exception handling, and reconciliation analytics to ensure the accuracy and integrity of financial data across all accounts.

**Bounded Context:** Account Reconciliation & Close Management
**Service Name:** `recon-service`
**Database:** `data/reconciliation.db`
**HTTP Port:** 8138 | **gRPC Port:** 9138

---

## 2. Database Schema

### 2.1 Reconciliation Profiles
```sql
CREATE TABLE reconciliation_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_name TEXT NOT NULL,
    profile_type TEXT NOT NULL
        CHECK(profile_type IN ('BALANCE_COMPARISON','TRANSACTION_MATCHING','INTERCOMPANY','INVENTORY','PAYROLL','COMMITMENT')),
    source_system TEXT NOT NULL,
    target_system TEXT NOT NULL,
    frequency TEXT NOT NULL
        CHECK(frequency IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY')),
    auto_match_enabled INTEGER NOT NULL DEFAULT 1,
    tolerance_threshold_cents INTEGER NOT NULL DEFAULT 0,
    risk_level TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(risk_level IN ('HIGH','MEDIUM','LOW')),
    preparer_id TEXT NOT NULL,
    reviewer_id TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, profile_name)
);

CREATE INDEX idx_recon_profiles_tenant_type ON reconciliation_profiles(tenant_id, profile_type);
CREATE INDEX idx_recon_profiles_tenant_risk ON reconciliation_profiles(tenant_id, risk_level);
```

### 2.2 Reconciliation Periods
```sql
CREATE TABLE reconciliation_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('MONTHLY','QUARTERLY','ANNUAL')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'NOT_STARTED'
        CHECK(status IN ('NOT_STARTED','IN_PREPARATION','IN_REVIEW','APPROVED','CLOSED')),
    due_date TEXT NOT NULL,
    preparer_signed_date TEXT,
    reviewer_signed_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (profile_id) REFERENCES reconciliation_profiles(id) ON DELETE RESTRICT
);

CREATE INDEX idx_recon_periods_tenant_profile ON reconciliation_periods(tenant_id, profile_id);
CREATE INDEX idx_recon_periods_tenant_status ON reconciliation_periods(tenant_id, status);
CREATE INDEX idx_recon_periods_tenant_dates ON reconciliation_periods(tenant_id, start_date, end_date);
```

### 2.3 Reconciliation Items
```sql
CREATE TABLE reconciliation_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    source_balance_cents INTEGER NOT NULL DEFAULT 0,
    target_balance_cents INTEGER NOT NULL DEFAULT 0,
    difference_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    match_status TEXT NOT NULL DEFAULT 'UNMATCHED'
        CHECK(match_status IN ('MATCHED','PARTIALLY_MATCHED','UNMATCHED','EXCLUDED')),
    match_rule_id TEXT,
    explanation TEXT,
    is_reconciling_item INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES reconciliation_periods(id) ON DELETE CASCADE,
    FOREIGN KEY (profile_id) REFERENCES reconciliation_profiles(id) ON DELETE RESTRICT
);

CREATE INDEX idx_recon_items_tenant_period ON reconciliation_items(tenant_id, period_id);
CREATE INDEX idx_recon_items_tenant_status ON reconciliation_items(tenant_id, match_status);
CREATE INDEX idx_recon_items_tenant_account ON reconciliation_items(tenant_id, account_id);
```

### 2.4 Matching Rules
```sql
CREATE TABLE matching_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_priority INTEGER NOT NULL DEFAULT 1,
    match_criteria TEXT NOT NULL,  -- JSON
    match_type TEXT NOT NULL
        CHECK(match_type IN ('ONE_TO_ONE','ONE_TO_MANY','MANY_TO_ONE','MANY_TO_MANY')),
    tolerance_type TEXT NOT NULL DEFAULT 'EXACT'
        CHECK(tolerance_type IN ('AMOUNT','PERCENTAGE','EXACT')),
    tolerance_value INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    match_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (profile_id) REFERENCES reconciliation_profiles(id) ON DELETE CASCADE
);

CREATE INDEX idx_match_rules_tenant_profile ON matching_rules(tenant_id, profile_id);
CREATE INDEX idx_match_rules_tenant_active ON matching_rules(tenant_id, is_active);
```

### 2.5 Transaction Pairs
```sql
CREATE TABLE transaction_pairs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    source_transaction_id TEXT NOT NULL,
    target_transaction_id TEXT NOT NULL,
    source_amount_cents INTEGER NOT NULL DEFAULT 0,
    target_amount_cents INTEGER NOT NULL DEFAULT 0,
    match_confidence DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    match_date TEXT,
    status TEXT NOT NULL DEFAULT 'SUGGESTED'
        CHECK(status IN ('CONFIRMED','SUGGESTED','REJECTED')),
    confirmed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES reconciliation_periods(id) ON DELETE CASCADE,
    FOREIGN KEY (rule_id) REFERENCES matching_rules(id) ON DELETE RESTRICT
);

CREATE INDEX idx_trans_pairs_tenant_period ON transaction_pairs(tenant_id, period_id);
CREATE INDEX idx_trans_pairs_tenant_status ON transaction_pairs(tenant_id, status);
CREATE INDEX idx_trans_pairs_tenant_rule ON transaction_pairs(tenant_id, rule_id);
```

### 2.6 Reconciling Items
```sql
CREATE TABLE reconciling_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    item_type TEXT NOT NULL
        CHECK(item_type IN ('TIMING_DIFF','ERROR','OMISSION','MISCLASSIFICATION','ADJUSTMENT')),
    description TEXT NOT NULL,
    amount_cents INTEGER NOT NULL DEFAULT 0,
    aging_days INTEGER NOT NULL DEFAULT 0,
    auto_resolve INTEGER NOT NULL DEFAULT 0,
    resolution_date TEXT,
    resolution_notes TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','INVESTIGATING','RESOLVED','WRITE_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES reconciliation_periods(id) ON DELETE CASCADE,
    FOREIGN KEY (item_id) REFERENCES reconciliation_items(id) ON DELETE CASCADE
);

CREATE INDEX idx_recon_items_tenant_period ON reconciling_items(tenant_id, period_id);
CREATE INDEX idx_recon_items_tenant_status ON reconciling_items(tenant_id, status);
CREATE INDEX idx_recon_items_tenant_type ON reconciling_items(tenant_id, item_type);
```

### 2.7 Certification History
```sql
CREATE TABLE certification_history (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    certifier_id TEXT NOT NULL,
    certification_type TEXT NOT NULL
        CHECK(certification_type IN ('PREPARER','REVIEWER')),
    comments TEXT,
    signed_at TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','REVOKED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (period_id) REFERENCES reconciliation_periods(id) ON DELETE CASCADE
);

CREATE INDEX idx_cert_hist_tenant_period ON certification_history(tenant_id, period_id);
CREATE INDEX idx_cert_hist_tenant_certifier ON certification_history(tenant_id, certifier_id);
```

---

## 3. REST API Endpoints

### Profiles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recon/profiles` | Create a reconciliation profile |
| GET | `/api/v1/recon/profiles` | List reconciliation profiles |
| GET | `/api/v1/recon/profiles/{id}` | Get profile details |
| PUT | `/api/v1/recon/profiles/{id}` | Update profile |

### Periods
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recon/periods` | Create a reconciliation period |
| GET | `/api/v1/recon/periods` | List reconciliation periods |
| GET | `/api/v1/recon/periods/{id}` | Get period details |
| PUT | `/api/v1/recon/periods/{id}` | Update period |
| POST | `/api/v1/recon/periods/{id}/open` | Open a reconciliation period |
| POST | `/api/v1/recon/periods/{id}/close` | Close a reconciliation period |
| POST | `/api/v1/recon/periods/{id}/certify` | Certify a reconciliation period |

### Items
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/recon/periods/{periodId}/items` | List reconciliation items |
| GET | `/api/v1/recon/items/{id}` | Get item details |
| POST | `/api/v1/recon/items/{id}/match` | Match an item |
| POST | `/api/v1/recon/items/{id}/unmatch` | Unmatch an item |
| POST | `/api/v1/recon/items/{id}/explain` | Add explanation to item |

### Matching Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recon/rules` | Create a matching rule |
| GET | `/api/v1/recon/rules` | List matching rules |
| GET | `/api/v1/recon/rules/{id}` | Get rule details |
| PUT | `/api/v1/recon/rules/{id}` | Update rule |
| DELETE | `/api/v1/recon/rules/{id}` | Delete rule |
| POST | `/api/v1/recon/rules/{id}/execute` | Execute a matching rule |

### Transactions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/recon/periods/{periodId}/transactions/unmatched` | Get unmatched transactions |
| POST | `/api/v1/recon/periods/{periodId}/auto-match` | Run auto-match for a period |
| POST | `/api/v1/recon/periods/{periodId}/manual-match` | Create a manual match |

### Reconciling Items
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recon/reconciling-items` | Create a reconciling item |
| GET | `/api/v1/recon/reconciling-items` | List reconciling items |
| GET | `/api/v1/recon/reconciling-items/{id}` | Get reconciling item details |
| PUT | `/api/v1/recon/reconciling-items/{id}` | Update reconciling item |
| POST | `/api/v1/recon/reconciling-items/{id}/resolve` | Resolve a reconciling item |
| POST | `/api/v1/recon/reconciling-items/{id}/write-off` | Write off a reconciling item |

### Certification
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recon/periods/{periodId}/certify` | Certify a period |
| GET | `/api/v1/recon/periods/{periodId}/certification-status` | Get certification status |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/recon/analytics/completion-dashboard` | Get reconciliation completion dashboard |
| GET | `/api/v1/recon/analytics/aging-report` | Get reconciling items aging report |
| GET | `/api/v1/recon/analytics/exception-summary` | Get exception summary |

---

## 4. Business Rules

1. A reconciliation period MUST NOT be opened if another period for the same profile with overlapping dates is already in `IN_PREPARATION` or `IN_REVIEW` status.
2. Auto-match MUST only apply matching rules that are active and belong to the same profile as the reconciliation period.
3. A match MUST be considered valid only if the absolute difference between source and target amounts is within the tolerance threshold defined by the matching rule.
4. A reconciliation item with a difference exceeding the profile tolerance threshold MUST be flagged as an exception and a reconciling item MUST be created automatically.
5. A period MAY be certified by the preparer only when all reconciliation items have a match status of `MATCHED` or `EXCLUDED`, and all reconciling items are in `RESOLVED` or `WRITE_OFF` status.
6. The reviewer certification MUST occur after the preparer certification; the system MUST NOT allow reviewer certification before preparer sign-off.
7. Matching rules MUST be executed in priority order; lower priority numbers MUST be processed first.
8. A reconciling item flagged as `TIMING_DIFF` MAY be auto-resolved if the difference clears within the next reconciliation period.
9. The system SHOULD calculate aging days for all open reconciling items based on the period end date.
10. A closed reconciliation period MUST NOT have its items, matches, or reconciling items modified.
11. The system MUST track all certification actions in the certification history for audit purposes.
12. Reconciliation profiles with `HIGH` risk level SHOULD require reviewer certification before the period can be closed.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package reconciliation.v1;

service AccountReconciliationService {
    // Profiles
    rpc CreateProfile(CreateProfileRequest) returns (CreateProfileResponse);
    rpc GetProfile(GetProfileRequest) returns (GetProfileResponse);
    rpc ListProfiles(ListProfilesRequest) returns (ListProfilesResponse);
    rpc UpdateProfile(UpdateProfileRequest) returns (UpdateProfileResponse);

    // Periods
    rpc CreatePeriod(CreatePeriodRequest) returns (CreatePeriodResponse);
    rpc GetPeriod(GetPeriodRequest) returns (GetPeriodResponse);
    rpc ListPeriods(ListPeriodsRequest) returns (ListPeriodsResponse);
    rpc OpenPeriod(OpenPeriodRequest) returns (OpenPeriodResponse);
    rpc ClosePeriod(ClosePeriodRequest) returns (ClosePeriodResponse);
    rpc CertifyPeriod(CertifyPeriodRequest) returns (CertifyPeriodResponse);

    // Items
    rpc ListItems(ListItemsRequest) returns (ListItemsResponse);
    rpc MatchItem(MatchItemRequest) returns (MatchItemResponse);
    rpc UnmatchItem(UnmatchItemRequest) returns (UnmatchItemResponse);
    rpc ExplainItem(ExplainItemRequest) returns (ExplainItemResponse);

    // Matching Rules
    rpc CreateRule(CreateRuleRequest) returns (CreateRuleResponse);
    rpc GetRule(GetRuleRequest) returns (GetRuleResponse);
    rpc ListRules(ListRulesRequest) returns (ListRulesResponse);
    rpc UpdateRule(UpdateRuleRequest) returns (UpdateRuleResponse);
    rpc ExecuteRule(ExecuteRuleRequest) returns (ExecuteRuleResponse);

    // Auto-Match
    rpc AutoMatchPeriod(AutoMatchPeriodRequest) returns (AutoMatchPeriodResponse);
    rpc ManualMatch(ManualMatchRequest) returns (ManualMatchResponse);
    rpc GetUnmatchedTransactions(GetUnmatchedTransactionsRequest) returns (GetUnmatchedTransactionsResponse);

    // Reconciling Items
    rpc CreateReconcilingItem(CreateReconcilingItemRequest) returns (CreateReconcilingItemResponse);
    rpc GetReconcilingItem(GetReconcilingItemRequest) returns (GetReconcilingItemResponse);
    rpc ListReconcilingItems(ListReconcilingItemsRequest) returns (ListReconcilingItemsResponse);
    rpc ResolveReconcilingItem(ResolveReconcilingItemRequest) returns (ResolveReconcilingItemResponse);
    rpc WriteOffReconcilingItem(WriteOffReconcilingItemRequest) returns (WriteOffReconcilingItemResponse);

    // Certification
    rpc GetCertificationStatus(GetCertificationStatusRequest) returns (GetCertificationStatusResponse);

    // Analytics
    rpc GetCompletionDashboard(GetCompletionDashboardRequest) returns (GetCompletionDashboardResponse);
    rpc GetAgingReport(GetAgingReportRequest) returns (GetAgingReportResponse);
    rpc GetExceptionSummary(GetExceptionSummaryRequest) returns (GetExceptionSummaryResponse);
}

message ReconciliationProfile {
    string id = 1;
    string tenant_id = 2;
    string profile_name = 3;
    string profile_type = 4;
    string source_system = 5;
    string target_system = 6;
    string frequency = 7;
    bool auto_match_enabled = 8;
    int64 tolerance_threshold_cents = 9;
    string risk_level = 10;
    string preparer_id = 11;
    string reviewer_id = 12;
}

message CreateProfileRequest {
    string tenant_id = 1;
    string profile_name = 2;
    string profile_type = 3;
    string source_system = 4;
    string target_system = 5;
    string frequency = 6;
    bool auto_match_enabled = 7;
    int64 tolerance_threshold_cents = 8;
    string risk_level = 9;
    string preparer_id = 10;
    string reviewer_id = 11;
    string created_by = 12;
}

message CreateProfileResponse {
    ReconciliationProfile profile = 1;
}

message GetProfileRequest {
    string tenant_id = 1;
    string profile_id = 2;
}

message GetProfileResponse {
    ReconciliationProfile profile = 1;
}

message ListProfilesRequest {
    string tenant_id = 1;
    string profile_type = 2;
    string risk_level = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListProfilesResponse {
    repeated ReconciliationProfile profiles = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateProfileRequest {
    string tenant_id = 1;
    string profile_id = 2;
    string profile_name = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateProfileResponse {
    ReconciliationProfile profile = 1;
}

message ReconciliationPeriod {
    string id = 1;
    string tenant_id = 2;
    string profile_id = 3;
    string period_name = 4;
    string period_type = 5;
    string start_date = 6;
    string end_date = 7;
    string status = 8;
    string due_date = 9;
}

message CreatePeriodRequest {
    string tenant_id = 1;
    string profile_id = 2;
    string period_name = 3;
    string period_type = 4;
    string start_date = 5;
    string end_date = 6;
    string due_date = 7;
    string created_by = 8;
}

message CreatePeriodResponse {
    ReconciliationPeriod period = 1;
}

message GetPeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetPeriodResponse {
    ReconciliationPeriod period = 1;
}

message ListPeriodsRequest {
    string tenant_id = 1;
    string profile_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPeriodsResponse {
    repeated ReconciliationPeriod periods = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message OpenPeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
    string opened_by = 3;
}

message OpenPeriodResponse {
    ReconciliationPeriod period = 1;
}

message ClosePeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
    string closed_by = 3;
}

message ClosePeriodResponse {
    ReconciliationPeriod period = 1;
}

message CertifyPeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
    string certifier_id = 3;
    string certification_type = 4;
    string comments = 5;
}

message CertifyPeriodResponse {
    string certification_id = 1;
    string status = 2;
    string signed_at = 3;
}

message ReconciliationItem {
    string id = 1;
    string tenant_id = 2;
    string period_id = 3;
    string account_id = 4;
    int64 source_balance_cents = 5;
    int64 target_balance_cents = 6;
    int64 difference_cents = 7;
    string match_status = 8;
    string explanation = 9;
}

message ListItemsRequest {
    string tenant_id = 1;
    string period_id = 2;
    string match_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListItemsResponse {
    repeated ReconciliationItem items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message MatchItemRequest {
    string tenant_id = 1;
    string item_id = 2;
    string rule_id = 3;
    string matched_by = 4;
}

message MatchItemResponse {
    ReconciliationItem item = 1;
}

message UnmatchItemRequest {
    string tenant_id = 1;
    string item_id = 2;
    string unmatched_by = 3;
}

message UnmatchItemResponse {
    ReconciliationItem item = 1;
}

message ExplainItemRequest {
    string tenant_id = 1;
    string item_id = 2;
    string explanation = 3;
    string updated_by = 4;
}

message ExplainItemResponse {
    ReconciliationItem item = 1;
}

message MatchingRule {
    string id = 1;
    string tenant_id = 2;
    string profile_id = 3;
    string rule_name = 4;
    int32 rule_priority = 5;
    string match_criteria = 6;
    string match_type = 7;
    string tolerance_type = 8;
    int64 tolerance_value = 9;
    bool is_active = 10;
    int32 match_count = 11;
}

message CreateRuleRequest {
    string tenant_id = 1;
    string profile_id = 2;
    string rule_name = 3;
    int32 rule_priority = 4;
    string match_criteria = 5;
    string match_type = 6;
    string tolerance_type = 7;
    int64 tolerance_value = 8;
    string created_by = 9;
}

message CreateRuleResponse {
    MatchingRule rule = 1;
}

message GetRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message GetRuleResponse {
    MatchingRule rule = 1;
}

message ListRulesRequest {
    string tenant_id = 1;
    string profile_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListRulesResponse {
    repeated MatchingRule rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string rule_name = 3;
    int32 rule_priority = 4;
    string match_criteria = 5;
    string updated_by = 6;
    int32 version = 7;
}

message UpdateRuleResponse {
    MatchingRule rule = 1;
}

message ExecuteRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string period_id = 3;
}

message ExecuteRuleResponse {
    int32 matches_found = 1;
    int32 items_matched = 2;
    repeated string matched_item_ids = 3;
}

message AutoMatchPeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
    string executed_by = 3;
}

message AutoMatchPeriodResponse {
    int32 total_matches = 1;
    int32 auto_confirmed = 2;
    int32 suggested = 3;
    int32 remaining_unmatched = 4;
}

message ManualMatchRequest {
    string tenant_id = 1;
    string period_id = 2;
    string source_transaction_id = 3;
    string target_transaction_id = 4;
    string matched_by = 5;
}

message ManualMatchResponse {
    string transaction_pair_id = 1;
    string status = 2;
}

message TransactionPair {
    string id = 1;
    string source_transaction_id = 2;
    string target_transaction_id = 3;
    int64 source_amount_cents = 4;
    int64 target_amount_cents = 5;
    double match_confidence = 6;
    string status = 7;
}

message GetUnmatchedTransactionsRequest {
    string tenant_id = 1;
    string period_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetUnmatchedTransactionsResponse {
    repeated TransactionPair unmatched = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReconcilingItem {
    string id = 1;
    string tenant_id = 2;
    string period_id = 3;
    string item_id = 4;
    string item_type = 5;
    string description = 6;
    int64 amount_cents = 7;
    int32 aging_days = 8;
    bool auto_resolve = 9;
    string status = 10;
}

message CreateReconcilingItemRequest {
    string tenant_id = 1;
    string period_id = 2;
    string item_id = 3;
    string item_type = 4;
    string description = 5;
    int64 amount_cents = 6;
    string created_by = 7;
}

message CreateReconcilingItemResponse {
    ReconcilingItem reconciling_item = 1;
}

message GetReconcilingItemRequest {
    string tenant_id = 1;
    string reconciling_item_id = 2;
}

message GetReconcilingItemResponse {
    ReconcilingItem reconciling_item = 1;
}

message ListReconcilingItemsRequest {
    string tenant_id = 1;
    string period_id = 2;
    string status = 3;
    string item_type = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListReconcilingItemsResponse {
    repeated ReconcilingItem items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ResolveReconcilingItemRequest {
    string tenant_id = 1;
    string reconciling_item_id = 2;
    string resolution_notes = 3;
    string resolved_by = 4;
}

message ResolveReconcilingItemResponse {
    ReconcilingItem reconciling_item = 1;
}

message WriteOffReconcilingItemRequest {
    string tenant_id = 1;
    string reconciling_item_id = 2;
    string resolution_notes = 3;
    string written_off_by = 4;
}

message WriteOffReconcilingItemResponse {
    ReconcilingItem reconciling_item = 1;
}

message CertificationStatus {
    string period_id = 1;
    bool preparer_certified = 2;
    bool reviewer_certified = 3;
    string preparer_signed_at = 4;
    string reviewer_signed_at = 5;
}

message GetCertificationStatusRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetCertificationStatusResponse {
    CertificationStatus status = 1;
}

message CompletionDashboardEntry {
    string profile_name = 3;
    string period_name = 4;
    int32 total_items = 5;
    int32 matched_items = 6;
    int32 unmatched_items = 7;
    int32 open_exceptions = 8;
    string status = 9;
}

message GetCompletionDashboardRequest {
    string tenant_id = 1;
}

message GetCompletionDashboardResponse {
    repeated CompletionDashboardEntry entries = 1;
}

message AgingReportEntry {
    string item_type = 1;
    int32 zero_to_thirty_days = 2;
    int32 thirty_to_sixty_days = 3;
    int32 sixty_to_ninety_days = 4;
    int32 over_ninety_days = 5;
    int64 total_amount_cents = 6;
}

message GetAgingReportRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetAgingReportResponse {
    repeated AgingReportEntry entries = 1;
}

message ExceptionSummaryEntry {
    string exception_type = 1;
    int32 count = 2;
    int64 total_amount_cents = 3;
    int32 resolved_count = 4;
    int32 open_count = 5;
}

message GetExceptionSummaryRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message GetExceptionSummaryResponse {
    repeated ExceptionSummaryEntry entries = 1;
    int32 total_exceptions = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | Account balances, journal entries | Source/target balances for reconciliation |
| `ap-service` | AP transaction details, payment records | Supplier transaction matching |
| `ar-service` | AR transaction details, receipt records | Customer transaction matching |
| `cash-management` | Bank statement lines, cash transactions | Bank reconciliation matching |
| `intercompany-service` | Intercompany transaction pairs | Intercompany reconciliation |
| `workflow-service` | Approval workflows, certification routing | Period certification orchestration |
| `user-service` | User roles, preparer/reviewer assignments | Certification authority validation |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | Reconciliation adjustments, write-off journals | Post reconciliation adjustments |
| `reporting-service` | Reconciliation status, aging analytics | Financial close reporting |
| `close-management` | Period certification status | Financial close coordination |
| `audit-service` | Certification history, match decisions | Compliance audit trail |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ReconciliationPeriodOpened` | `recon.period.opened` | `{ tenant_id, period_id, profile_id, period_name, start_date, end_date }` | Published when a reconciliation period is opened |
| `AutoMatchCompleted` | `recon.auto-match.completed` | `{ tenant_id, period_id, total_matches, auto_confirmed, suggested, remaining_unmatched }` | Published when auto-match processing completes |
| `ItemMatched` | `recon.item.matched` | `{ tenant_id, item_id, period_id, match_rule_id, source_amount_cents, target_amount_cents }` | Published when a reconciliation item is matched |
| `ReconcilingItemCreated` | `recon.reconciling-item.created` | `{ tenant_id, reconciling_item_id, period_id, item_type, amount_cents, aging_days }` | Published when a reconciling item is created |
| `PeriodCertified` | `recon.period.certified` | `{ tenant_id, period_id, certifier_id, certification_type, signed_at }` | Published when a period is certified by preparer or reviewer |
