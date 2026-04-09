# 32 - Collections & Credit Management Service Specification

## 1. Domain Overview

Collections & Credit Management provides customer credit scoring, credit limit management, credit holds, collection work queues, dunning management, payment promise tracking, dispute resolution, and bad debt write-off processing. Integrates with AR for invoice data, OM for order holds, GL for write-off entries, and Workflow for approvals.

**Bounded Context:** Credit & Collections Management
**Service Name:** `collections-service`
**Database:** `data/collections.db`
**HTTP Port:** 8032 | **gRPC Port:** 9032

---

## 2. Database Schema

### 2.1 Credit Policies
```sql
CREATE TABLE credit_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_name TEXT NOT NULL,
    description TEXT,

    -- Credit check rules
    auto_credit_check INTEGER NOT NULL DEFAULT 1,
    check_on_order INTEGER NOT NULL DEFAULT 1,
    check_on_invoice INTEGER NOT NULL DEFAULT 1,
    check_on_shipment INTEGER NOT NULL DEFAULT 0,

    -- Auto-approval
    auto_approve_below_cents INTEGER DEFAULT 0,
    auto_approve_good_credit INTEGER NOT NULL DEFAULT 0,

    -- Credit scoring weights
    payment_history_weight REAL NOT NULL DEFAULT 0.4,
    outstanding_balance_weight REAL NOT NULL DEFAULT 0.25,
    overdue_weight REAL NOT NULL DEFAULT 0.2,
    dispute_ratio_weight REAL NOT NULL DEFAULT 0.15,

    -- Scoring thresholds
    good_score_threshold REAL DEFAULT 80,
    acceptable_score_threshold REAL DEFAULT 60,
    poor_score_threshold REAL DEFAULT 40,

    -- Review frequency
    review_frequency_days INTEGER DEFAULT 90,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_name)
);
```

### 2.2 Credit Scores
```sql
CREATE TABLE credit_scores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    score_date TEXT NOT NULL,
    composite_score REAL NOT NULL,
    payment_history_score REAL NOT NULL,
    outstanding_balance_score REAL NOT NULL,
    overdue_score REAL NOT NULL,
    dispute_ratio_score REAL NOT NULL,
    risk_category TEXT NOT NULL
        CHECK(risk_category IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    score_factors TEXT,                     -- JSON: explanation of score drivers
    previous_score REAL,
    score_change REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, customer_id, score_date)
);

CREATE INDEX idx_credit_scores_tenant_customer ON credit_scores(tenant_id, customer_id);
CREATE INDEX idx_credit_scores_tenant_date ON credit_scores(tenant_id, score_date);
```

### 2.3 Credit Limit Reviews
```sql
CREATE TABLE credit_limit_reviews (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    requested_by TEXT NOT NULL,
    request_date TEXT NOT NULL,
    current_limit_cents INTEGER NOT NULL,
    requested_limit_cents INTEGER NOT NULL,
    approved_limit_cents INTEGER,
    review_reason TEXT NOT NULL,
    supporting_documents TEXT,              -- JSON: document IDs
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','CANCELLED')),
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, customer_id, request_date, requested_by)
);
```

### 2.4 Credit Hold Events
```sql
CREATE TABLE credit_hold_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    hold_type TEXT NOT NULL
        CHECK(hold_type IN ('CREDIT_LIMIT_EXCEEDED','OVERDUE_PAYMENTS','RISK_ASSESSMENT','MANUAL')),
    trigger_document_type TEXT,
    trigger_document_id TEXT,
    hold_reason TEXT NOT NULL,
    total_overdue_cents INTEGER NOT NULL DEFAULT 0,
    credit_exceeded_by_cents INTEGER NOT NULL DEFAULT 0,
    placed_by TEXT NOT NULL,
    placed_at TEXT NOT NULL DEFAULT (datetime('now')),

    release_type TEXT,
    released_by TEXT,
    released_at TEXT,
    release_reason TEXT,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','RELEASED','SUPERSEDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_credit_holds_tenant_customer ON credit_hold_events(tenant_id, customer_id);
CREATE INDEX idx_credit_holds_tenant_status ON credit_hold_events(tenant_id, status);
```

### 2.5 Collection Strategies
```sql
CREATE TABLE collection_strategies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    strategy_name TEXT NOT NULL,
    description TEXT,
    strategy_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(strategy_type IN ('STANDARD','AGGRESSIVE','CONSERVATIVE','CUSTOM')),
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, strategy_name)
);

CREATE TABLE collection_strategy_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    strategy_id TEXT NOT NULL,
    from_days_overdue INTEGER NOT NULL,
    to_days_overdue INTEGER,
    action TEXT NOT NULL
        CHECK(action IN ('REMINDER_EMAIL','PHONE_CALL','DUNNING_LETTER','FINAL_NOTICE','LEGAL_REFERRAL','SUSPEND_SHIPMENT','SUSPEND_ORDER')),
    template_id TEXT,
    escalation_to TEXT,
    priority TEXT DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (strategy_id) REFERENCES collection_strategies(id) ON DELETE CASCADE
);
```

### 2.6 Collection Work Queues
```sql
CREATE TABLE collection_work_queues (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    queue_name TEXT NOT NULL,
    queue_type TEXT NOT NULL
        CHECK(queue_type IN ('OVERDUE','HIGH_RISK','PROMISE_TO_PAY','DISPUTE','WRITE_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, queue_name)
);

CREATE TABLE collection_work_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    queue_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    invoice_id TEXT,
    assigned_to TEXT,
    due_date TEXT,
    priority TEXT DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    overdue_amount_cents INTEGER NOT NULL DEFAULT 0,
    days_overdue INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_PROGRESS','COMPLETED','ESCALATED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (queue_id) REFERENCES collection_work_queues(id) ON DELETE CASCADE
);

CREATE INDEX idx_work_items_tenant_assigned ON collection_work_items(tenant_id, assigned_to);
CREATE INDEX idx_work_items_tenant_customer ON collection_work_items(tenant_id, customer_id);
CREATE INDEX idx_work_items_tenant_status ON collection_work_items(tenant_id, status);
```

### 2.7 Collection Activities
```sql
CREATE TABLE collection_activities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    invoice_id TEXT,
    work_item_id TEXT,
    activity_type TEXT NOT NULL
        CHECK(activity_type IN ('PHONE_CALL','EMAIL','LETTER','FAX','MEETING','NOTE','PAYMENT_PROMISE','DISPUTE_CREATED')),
    activity_date TEXT NOT NULL,
    contact_name TEXT,
    summary TEXT NOT NULL,
    outcome TEXT,
    follow_up_date TEXT,
    performed_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_activities_tenant_customer ON collection_activities(tenant_id, customer_id);
CREATE INDEX idx_activities_tenant_date ON collection_activities(tenant_id, activity_date);
```

### 2.8 Dunning Letters
```sql
CREATE TABLE dunning_letters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    letter_number TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    dunning_level INTEGER NOT NULL DEFAULT 1,
    letter_date TEXT NOT NULL,
    letter_type TEXT NOT NULL
        CHECK(letter_type IN ('REMINDER','FIRST_NOTICE','SECOND_NOTICE','FINAL_DEMAND','LEGAL_NOTICE')),
    total_amount_due_cents INTEGER NOT NULL DEFAULT 0,
    overdue_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    invoice_ids TEXT NOT NULL,              -- JSON array
    template_id TEXT,
    sent_method TEXT DEFAULT 'EMAIL'
        CHECK(sent_method IN ('EMAIL','MAIL','FAX')),
    sent_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SENT','FAILED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, letter_number)
);
```

### 2.9 Payment Promises
```sql
CREATE TABLE payment_promises (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    promised_amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    promised_date TEXT NOT NULL,
    invoice_ids TEXT,                       -- JSON array
    promised_by TEXT,                       -- Contact name
    notes TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PARTIALLY_FULFILLED','FULFILLED','BROKEN','CANCELLED')),
    actual_paid_cents INTEGER NOT NULL DEFAULT 0,
    broken_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL
);

CREATE INDEX idx_promises_tenant_customer ON payment_promises(tenant_id, customer_id);
CREATE INDEX idx_promises_tenant_date ON payment_promises(tenant_id, promised_date);
CREATE INDEX idx_promises_tenant_status ON payment_promises(tenant_id, status);
```

### 2.10 Dispute Cases
```sql
CREATE TABLE dispute_cases (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dispute_number TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    invoice_id TEXT,
    dispute_type TEXT NOT NULL
        CHECK(dispute_type IN ('PRICING','QUANTITY','QUALITY','DUPLICATE','UNRECEIVED','INCORRECT','OTHER')),
    disputed_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    description TEXT NOT NULL,
    supporting_documents TEXT,              -- JSON: document IDs
    assigned_to TEXT,
    resolution TEXT,
    resolution_amount_cents INTEGER,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','UNDER_INVESTIGATION','RESOLVED','ESCALATED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dispute_number)
);

CREATE INDEX idx_disputes_tenant_customer ON dispute_cases(tenant_id, customer_id);
CREATE INDEX idx_disputes_tenant_status ON dispute_cases(tenant_id, status);
```

### 2.11 Write-Off Requests
```sql
CREATE TABLE write_off_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    invoice_id TEXT,
    write_off_amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    reason TEXT NOT NULL,
    aging_days INTEGER NOT NULL,
    collection_activities_count INTEGER NOT NULL DEFAULT 0,
    assigned_to TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','POSTED')),
    approved_by TEXT,
    approved_at TEXT,
    rejection_reason TEXT,
    gl_journal_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
```

---

## 3. REST API Endpoints

```
# Credit Policies
GET/POST      /api/v1/collections/credit-policies         Permission: collections.policies.read/create
GET/PUT       /api/v1/collections/credit-policies/{id}     Permission: collections.policies.read/update

# Credit Scores
GET           /api/v1/collections/credit-scores            Permission: collections.scores.read
POST          /api/v1/collections/credit-scores/calculate  Permission: collections.scores.create
GET           /api/v1/collections/credit-scores/{customer_id}/history Permission: collections.scores.read

# Credit Limit Reviews
GET/POST      /api/v1/collections/credit-reviews           Permission: collections.reviews.read/create
POST          /api/v1/collections/credit-reviews/{id}/approve Permission: collections.reviews.approve
POST          /api/v1/collections/credit-reviews/{id}/reject  Permission: collections.reviews.approve

# Credit Holds
GET           /api/v1/collections/credit-holds             Permission: collections.holds.read
POST          /api/v1/collections/credit-holds/place       Permission: collections.holds.create
POST          /api/v1/collections/credit-holds/{id}/release Permission: collections.holds.update

# Collection Work Queues
GET           /api/v1/collections/work-queues              Permission: collections.queues.read
GET           /api/v1/collections/work-queues/{id}/items    Permission: collections.queues.read
POST          /api/v1/collections/work-items/{id}/assign   Permission: collections.queues.update
POST          /api/v1/collections/work-items/{id}/complete  Permission: collections.queues.update

# Collection Activities
GET/POST      /api/v1/collections/activities               Permission: collections.activities.read/create

# Dunning
GET/POST      /api/v1/collections/dunning-letters          Permission: collections.dunning.read/create
POST          /api/v1/collections/dunning-letters/{id}/send Permission: collections.dunning.update
POST          /api/v1/collections/dunning/run               Permission: collections.dunning.run

# Payment Promises
GET/POST      /api/v1/collections/payment-promises         Permission: collections.promises.read/create
POST          /api/v1/collections/payment-promises/{id}/fulfill Permission: collections.promises.update
POST          /api/v1/collections/payment-promises/{id}/break  Permission: collections.promises.update

# Disputes
GET/POST      /api/v1/collections/disputes                 Permission: collections.disputes.read/create
GET/PUT       /api/v1/collections/disputes/{id}             Permission: collections.disputes.read/update
POST          /api/v1/collections/disputes/{id}/resolve    Permission: collections.disputes.update

# Write-Offs
GET/POST      /api/v1/collections/write-offs               Permission: collections.writeoffs.read/create
POST          /api/v1/collections/write-offs/{id}/approve  Permission: collections.writeoffs.approve
POST          /api/v1/collections/write-offs/{id}/reject   Permission: collections.writeoffs.approve

# Reports
GET           /api/v1/collections/reports/aging            Permission: collections.reports.view
GET           /api/v1/collections/reports/dso              Permission: collections.reports.view
GET           /api/v1/collections/reports/collector-performance Permission: collections.reports.view
GET           /api/v1/collections/reports/risk-summary     Permission: collections.reports.view
GET           /api/v1/collections/reports/promise-fulfillment Permission: collections.reports.view
```

---

## 4. Business Rules

### 4.1 Credit Scoring
Composite score calculated from weighted factors:
```
composite = (payment_history_score × 0.40) +
            (outstanding_balance_score × 0.25) +
            (overdue_score × 0.20) +
            (dispute_ratio_score × 0.15)

Risk categories:
  80-100 = LOW
  60-79  = MEDIUM
  40-59  = HIGH
  0-39   = CRITICAL
```

### 4.2 Credit Hold Rules
- **Automatic hold:** Credit used > credit limit
- **Automatic hold:** Customer has invoices overdue > X days
- **Risk hold:** Credit score drops below threshold
- **Manual hold:** Collector places manual hold
- **Release conditions:** Payment received, credit limit increased, manual release

### 4.3 Collection Strategy Assignment
- Auto-assign strategy based on customer risk category and aging
- Escalate work items based on strategy rules (days overdue triggers)
- Work items assignable to collectors (round-robin or skill-based)

### 4.4 Dunning Escalation
- Level 1 (1-30 days overdue): Friendly reminder email
- Level 2 (31-60 days): First notice (email + letter)
- Level 3 (61-90 days): Second notice (stronger language)
- Level 4 (91-120 days): Final demand (legal notice)
- Level 5 (120+ days): Legal referral or write-off

### 4.5 Payment Promise Tracking
- Record customer promise to pay (amount, date)
- Monitor for fulfillment on promised date
- Auto-mark as BROKEN if payment not received by promised date + grace period
- Broken promises increase risk score

### 4.6 Write-Off Process
1. Collector submits write-off request
2. Approval based on amount threshold (manager → director → CFO)
3. On approval: create GL journal (Debit Bad Debt Expense, Credit AR)
4. Update invoice status to WRITE_OFF

### 4.7 Key Metrics
- **DSO (Days Sales Outstanding):** `((AR Balance / Credit Sales) × Days in Period)`
- **CEI (Collection Effectiveness Index):** `(Amount Collected / Amount Collectible) × 100`
- **Aging:** Buckets: Current, 1-30, 31-60, 61-90, 91-120, 120+

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `collections.credit_score.updated` | Score recalculated | OM |
| `collections.credit_hold.placed` | Hold placed | OM |
| `collections.credit_hold.released` | Hold released | OM |
| `collections.dunning.sent` | Dunning letter sent | — |
| `collections.promise.fulfilled` | Promise kept | — |
| `collections.promise.broken` | Promise broken | — |
| `collections.write_off.approved` | Write-off approved | GL, AR |
| `collections.dispute.created` | Dispute opened | AR |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.collections.v1;

service CollectionsCreditService {
    rpc CheckCredit(CheckCreditRequest) returns (CheckCreditResponse);
    rpc GetCreditScore(GetCreditScoreRequest) returns (GetCreditScoreResponse);
    rpc PlaceCreditHold(PlaceCreditHoldRequest) returns (PlaceCreditHoldResponse);
    rpc ReleaseCreditHold(ReleaseCreditHoldRequest) returns (ReleaseCreditHoldResponse);
    rpc GetAging(GetAgingRequest) returns (GetAgingResponse);
    rpc GetCustomerRisk(GetCustomerRiskRequest) returns (GetCustomerRiskResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Consumes From
- **AR:** Invoice data, receipt data, customer balances
- **OM:** Order data for credit check before order confirmation
- **Auth:** User data for collector assignment

### 6.2 Publishes To
- **GL:** Write-off journal entries
- **OM:** Credit hold/release notifications
- **AR:** Invoice status updates (disputes, write-offs)
- **Workflow:** Approval requests (credit reviews, write-offs)

---

## 7. Migrations

1. V001: `credit_policies`
2. V002: `credit_scores`
3. V003: `credit_limit_reviews`
4. V004: `credit_hold_events`
5. V005: `collection_strategies`
6. V006: `collection_strategy_rules`
7. V007: `collection_work_queues`
8. V008: `collection_work_items`
9. V009: `collection_activities`
10. V010: `dunning_letters`
11. V011: `payment_promises`
12. V012: `dispute_cases`
13. V013: `write_off_requests`
14. V014: Triggers for `updated_at`
