# 56 - Subscription Management Service Specification

## 1. Domain Overview

Subscription Management handles recurring revenue operations including subscription product and plan definition, tiered feature management, customer subscription lifecycle (create, amend, cancel, renew), metered usage recording and rating, automated billing run generation, payment processing and retry, and subscription analytics with churn, MRR, ARR, and cohort metrics. Integrates with AR for invoice generation, REV for revenue recognition of subscription fees, GL for accounting entries, and PRICING for plan price resolution.

**Bounded Context:** Subscription Billing & Recurring Revenue Management
**Service Name:** `subscription-service`
**Database:** `data/subscription.db`
**HTTP Port:** 8088 | **gRPC Port:** 9088

---

## 2. Database Schema

### 2.1 Subscription Products
```sql
CREATE TABLE subscription_products (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    product_type TEXT NOT NULL
        CHECK(product_type IN ('PHYSICAL','DIGITAL','SERVICE','BUNDLE')),
    billing_cycle TEXT NOT NULL
        CHECK(billing_cycle IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY','SEMI_ANNUAL','ANNUAL','CUSTOM')),
    custom_cycle_days INTEGER,                    -- Required when billing_cycle = CUSTOM
    trial_period_days INTEGER NOT NULL DEFAULT 0,
    setup_fee_cents INTEGER NOT NULL DEFAULT 0,
    is_metered INTEGER NOT NULL DEFAULT 0,
    metered_unit TEXT,                            -- e.g. "API_CALLS", "GB_STORAGE", "USERS"
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','DISCONTINUED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, product_code)
);

CREATE INDEX idx_sub_products_tenant_type ON subscription_products(tenant_id, product_type);
CREATE INDEX idx_sub_products_tenant_status ON subscription_products(tenant_id, status);
CREATE INDEX idx_sub_products_tenant_cycle ON subscription_products(tenant_id, billing_cycle);
CREATE INDEX idx_sub_products_tenant_active ON subscription_products(tenant_id, is_active);
```

### 2.2 Subscription Tiers
```sql
CREATE TABLE subscription_tiers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    tier_name TEXT NOT NULL,
    tier_code TEXT NOT NULL,
    description TEXT,
    price_cents INTEGER NOT NULL,                 -- Base recurring price in cents
    setup_fee_cents INTEGER NOT NULL DEFAULT 0,
    included_units INTEGER NOT NULL DEFAULT 0,    -- Included metered units
    overage_price_cents INTEGER NOT NULL DEFAULT 0,
    features TEXT,                                -- JSON array: feature identifiers
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES subscription_products(id),
    UNIQUE(tenant_id, product_id, tier_code)
);

CREATE INDEX idx_sub_tiers_tenant_product ON subscription_tiers(tenant_id, product_id);
CREATE INDEX idx_sub_tiers_tenant_active ON subscription_tiers(tenant_id, is_active);
CREATE INDEX idx_sub_tiers_tenant_default ON subscription_tiers(tenant_id, is_default);
```

### 2.3 Subscriptions
```sql
CREATE TABLE subscriptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    subscription_number TEXT NOT NULL,             -- SUB-2024-00001
    customer_id TEXT NOT NULL,                     -- FK to AR customer
    product_id TEXT NOT NULL,
    tier_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('TRIAL','ACTIVE','SUSPENDED','CANCELLED','EXPIRED','PENDING')),
    start_date TEXT NOT NULL,                      -- ISO 8601 date
    end_date TEXT,                                 -- NULL = evergreen
    trial_start_date TEXT,
    trial_end_date TEXT,
    current_period_start TEXT NOT NULL,
    current_period_end TEXT NOT NULL,
    billing_cycle TEXT NOT NULL,
    billing_day_of_month INTEGER NOT NULL DEFAULT 1,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    base_price_cents INTEGER NOT NULL,
    discount_percent REAL NOT NULL DEFAULT 0.0,
    total_price_cents INTEGER NOT NULL,
    auto_renew INTEGER NOT NULL DEFAULT 1,
    renewal_tier_id TEXT,                          -- For upgrade on renewal
    cancellation_reason TEXT,
    cancelled_at TEXT,
    cancelled_by TEXT,
    payment_method_id TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES subscription_products(id),
    FOREIGN KEY (tier_id) REFERENCES subscription_tiers(id),
    UNIQUE(tenant_id, subscription_number)
);

CREATE INDEX idx_subscriptions_tenant_customer ON subscriptions(tenant_id, customer_id);
CREATE INDEX idx_subscriptions_tenant_product ON subscriptions(tenant_id, product_id);
CREATE INDEX idx_subscriptions_tenant_status ON subscriptions(tenant_id, status);
CREATE INDEX idx_subscriptions_tenant_dates ON subscriptions(tenant_id, start_date, end_date);
CREATE INDEX idx_subscriptions_tenant_period ON subscriptions(tenant_id, current_period_start, current_period_end);
CREATE INDEX idx_subscriptions_tenant_active ON subscriptions(tenant_id, is_active);
```

### 2.4 Subscription Usage
```sql
CREATE TABLE subscription_usage (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    subscription_id TEXT NOT NULL,
    usage_date TEXT NOT NULL,                      -- ISO 8601 date
    quantity REAL NOT NULL DEFAULT 0.0,
    unit_of_measure TEXT NOT NULL,
    source_system TEXT,                            -- e.g. "API_GATEWAY", "STORAGE_SERVICE"
    source_record_id TEXT,                         -- Reference to external record
    rated_amount_cents INTEGER NOT NULL DEFAULT 0,
    is_invoiced INTEGER NOT NULL DEFAULT 0,
    invoice_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id)
);

CREATE INDEX idx_sub_usage_tenant_sub ON subscription_usage(tenant_id, subscription_id);
CREATE INDEX idx_sub_usage_tenant_date ON subscription_usage(tenant_id, usage_date);
CREATE INDEX idx_sub_usage_tenant_invoiced ON subscription_usage(tenant_id, is_invoiced);
CREATE INDEX idx_sub_usage_tenant_source ON subscription_usage(tenant_id, source_system);
```

### 2.5 Subscription Line Items
```sql
CREATE TABLE subscription_line_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    subscription_id TEXT NOT NULL,
    billing_run_id TEXT,
    charge_type TEXT NOT NULL
        CHECK(charge_type IN ('RECURRING','METERED','SETUP','ONE_TIME','PRORATION','DISCOUNT','TAX')),
    description TEXT NOT NULL,
    quantity REAL NOT NULL DEFAULT 1.0,
    unit_price_cents INTEGER NOT NULL,
    amount_cents INTEGER NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    invoice_id TEXT,
    invoice_line_id TEXT,                          -- FK to AR invoice line
    is_invoiced INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id)
);

CREATE INDEX idx_sub_lines_tenant_sub ON subscription_line_items(tenant_id, subscription_id);
CREATE INDEX idx_sub_lines_tenant_type ON subscription_line_items(tenant_id, charge_type);
CREATE INDEX idx_sub_lines_tenant_invoiced ON subscription_line_items(tenant_id, is_invoiced);
CREATE INDEX idx_sub_lines_tenant_billing_run ON subscription_line_items(tenant_id, billing_run_id);
CREATE INDEX idx_sub_lines_tenant_period ON subscription_line_items(tenant_id, period_start, period_end);
```

### 2.6 Subscription Amendments
```sql
CREATE TABLE subscription_amendments (
    id TEXT PRIMARY KEY NOT NOT NULL,
    tenant_id TEXT NOT NULL,
    amendment_number TEXT NOT NULL,                -- AMD-2024-00001
    subscription_id TEXT NOT NULL,
    amendment_type TEXT NOT NULL
        CHECK(amendment_type IN ('UPGRADE','DOWNGRADE','RENEWAL','CANCELLATION','PAUSE','RESUME','PRICE_CHANGE','TERMS_CHANGE')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PENDING_APPROVAL','APPROVED','REJECTED','COMPLETED')),
    effective_date TEXT NOT NULL,
    previous_tier_id TEXT,
    new_tier_id TEXT,
    previous_price_cents INTEGER,
    new_price_cents INTEGER,
    proration_amount_cents INTEGER NOT NULL DEFAULT 0,
    reason TEXT,
    approved_by TEXT,
    approved_at TEXT,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id),
    UNIQUE(tenant_id, amendment_number)
);

CREATE INDEX idx_sub_amendments_tenant_sub ON subscription_amendments(tenant_id, subscription_id);
CREATE INDEX idx_sub_amendments_tenant_type ON subscription_amendments(tenant_id, amendment_type);
CREATE INDEX idx_sub_amendments_tenant_status ON subscription_amendments(tenant_id, status);
CREATE INDEX idx_sub_amendments_tenant_dates ON subscription_amendments(tenant_id, effective_date);
CREATE INDEX idx_sub_amendments_tenant_active ON subscription_amendments(tenant_id, is_active);
```

### 2.7 Subscription Billing Runs
```sql
CREATE TABLE subscription_billing_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    billing_run_number TEXT NOT NULL,              -- BR-2024-00001
    run_type TEXT NOT NULL
        CHECK(run_type IN ('RECURRING','METERED','PRORATION','AD_HOC')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','PARTIALLY_COMPLETED')),
    billing_period_start TEXT NOT NULL,
    billing_period_end TEXT NOT NULL,
    total_subscriptions INTEGER NOT NULL DEFAULT 0,
    processed_subscriptions INTEGER NOT NULL DEFAULT 0,
    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    error_count INTEGER NOT NULL DEFAULT 0,
    started_at TEXT,
    completed_at TEXT,
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, billing_run_number)
);

CREATE INDEX idx_billing_runs_tenant_status ON subscription_billing_runs(tenant_id, status);
CREATE INDEX idx_billing_runs_tenant_type ON subscription_billing_runs(tenant_id, run_type);
CREATE INDEX idx_billing_runs_tenant_period ON subscription_billing_runs(tenant_id, billing_period_start, billing_period_end);
CREATE INDEX idx_billing_runs_tenant_dates ON subscription_billing_runs(tenant_id, created_at);
```

### 2.8 Subscription Payments
```sql
CREATE TABLE subscription_payments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    subscription_id TEXT NOT NULL,
    billing_run_id TEXT,
    payment_number TEXT NOT NULL,                  -- PAY-2024-00001
    amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    payment_method TEXT NOT NULL
        CHECK(payment_method IN ('CREDIT_CARD','ACH','WIRE','INVOICE','PAYPAL','OTHER')),
    payment_ref TEXT,                              -- External payment reference
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PROCESSING','COMPLETED','FAILED','REFUNDED','PARTIALLY_REFUNDED')),
    due_date TEXT NOT NULL,
    paid_date TEXT,
    failure_reason TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    next_retry_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id),
    UNIQUE(tenant_id, payment_number)
);

CREATE INDEX idx_sub_payments_tenant_sub ON subscription_payments(tenant_id, subscription_id);
CREATE INDEX idx_sub_payments_tenant_status ON subscription_payments(tenant_id, status);
CREATE INDEX idx_sub_payments_tenant_due ON subscription_payments(tenant_id, due_date);
CREATE INDEX idx_sub_payments_tenant_method ON subscription_payments(tenant_id, payment_method);
CREATE INDEX idx_sub_payments_tenant_dates ON subscription_payments(tenant_id, created_at);
```

### 2.9 Subscription Analytics
```sql
CREATE TABLE subscription_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,                     -- ISO 8601 date
    metric_type TEXT NOT NULL
        CHECK(metric_type IN ('MRR','ARR','CHURN_RATE','EXPANSION_REVENUE','CONTRACTION_REVENUE',
                              'NEW_MRR','REACTIVATION_MRR','CUSTOMER_COUNT','ARPU','LTV',
                              'TRIAL_CONVERSION_RATE')),
    product_id TEXT,
    tier_id TEXT,
    metric_value REAL NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    period_type TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(period_type IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY','ANNUAL')),
    metadata TEXT,                                 -- JSON: additional breakdown data

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, metric_date, metric_type, product_id, period_type)
);

CREATE INDEX idx_sub_analytics_tenant_date ON subscription_analytics(tenant_id, metric_date);
CREATE INDEX idx_sub_analytics_tenant_type ON subscription_analytics(tenant_id, metric_type);
CREATE INDEX idx_sub_analytics_tenant_product ON subscription_analytics(tenant_id, product_id);
CREATE INDEX idx_sub_analytics_tenant_period ON subscription_analytics(tenant_id, period_type);
```

---

## 3. REST API Endpoints

```
# Product & Plan Management
GET/POST       /api/v1/subscriptions/products                Permission: subscription.products.manage
GET/PUT        /api/v1/subscriptions/products/{id}
PATCH          /api/v1/subscriptions/products/{id}/status    Permission: subscription.products.manage

# Tier Management
GET/POST       /api/v1/subscriptions/products/{id}/tiers     Permission: subscription.products.manage
GET/PUT        /api/v1/subscriptions/tiers/{id}
DELETE         /api/v1/subscriptions/tiers/{id}              Permission: subscription.products.manage

# Subscription Lifecycle
GET/POST       /api/v1/subscriptions/subscriptions           Permission: subscription.subscriptions.manage
GET/PUT        /api/v1/subscriptions/subscriptions/{id}
POST           /api/v1/subscriptions/subscriptions/{id}/cancel   Permission: subscription.subscriptions.cancel
POST           /api/v1/subscriptions/subscriptions/{id}/renew   Permission: subscription.subscriptions.renew
POST           /api/v1/subscriptions/subscriptions/{id}/suspend Permission: subscription.subscriptions.manage
POST           /api/v1/subscriptions/subscriptions/{id}/resume  Permission: subscription.subscriptions.manage

# Amendments
GET/POST       /api/v1/subscriptions/amendments              Permission: subscription.amendments.manage
GET            /api/v1/subscriptions/amendments/{id}
POST           /api/v1/subscriptions/amendments/{id}/approve  Permission: subscription.amendments.approve
POST           /api/v1/subscriptions/amendments/{id}/reject   Permission: subscription.amendments.approve

# Usage Recording
POST           /api/v1/subscriptions/usage                   Permission: subscription.usage.record
GET            /api/v1/subscriptions/usage/{subscription_id}
GET            /api/v1/subscriptions/usage/summary            Permission: subscription.usage.read

# Billing
POST           /api/v1/subscriptions/billing-runs            Permission: subscription.billing.run
GET            /api/v1/subscriptions/billing-runs
GET            /api/v1/subscriptions/billing-runs/{id}
POST           /api/v1/subscriptions/billing-runs/{id}/retry Permission: subscription.billing.run

# Payments
GET            /api/v1/subscriptions/payments
GET            /api/v1/subscriptions/payments/{id}
POST           /api/v1/subscriptions/payments/{id}/retry     Permission: subscription.payments.manage
POST           /api/v1/subscriptions/payments/{id}/refund    Permission: subscription.payments.refund

# Analytics
GET            /api/v1/subscriptions/analytics/overview       Permission: subscription.analytics.read
GET            /api/v1/subscriptions/analytics/mrr
GET            /api/v1/subscriptions/analytics/arr
GET            /api/v1/subscriptions/analytics/churn
GET            /api/v1/subscriptions/analytics/cohorts
GET            /api/v1/subscriptions/analytics/lTV
POST           /api/v1/subscriptions/analytics/refresh        Permission: subscription.analytics.manage

# Customer Subscriptions
GET            /api/v1/subscriptions/customers/{id}/subscriptions
GET            /api/v1/subscriptions/customers/{id}/usage
GET            /api/v1/subscriptions/customers/{id}/billing-history
```

---

## 4. Business Rules

### 4.1 Subscription Products & Tiers
```
Product and tier management:
  1. Product code MUST be unique within a tenant
  2. Each product MUST have at least one active tier before subscriptions can be created
  3. Tier code MUST be unique within a product
  4. Exactly one tier per product SHOULD be marked as default (is_default = 1)
  5. Discontinued products MUST NOT allow new subscriptions; existing subscriptions continue until end_date
  6. Custom billing cycle MUST specify custom_cycle_days > 0
  7. Trial period (trial_period_days) applies to new subscriptions only
```

### 4.2 Subscription Lifecycle
- Subscriptions are created with status PENDING and transition to ACTIVE or TRIAL on start_date
- Evergreen subscriptions (no end_date) auto-renew at the end of each billing period when auto_renew = 1
- Term subscriptions expire on end_date unless renewed before expiration
- Status transitions: PENDING -> TRIAL -> ACTIVE -> SUSPENDED -> CANCELLED/EXPIRED
- A SUSPENDED subscription can be RESUMED; a CANCELLED subscription cannot be reactivated (requires new subscription)
- Cancellation requires a cancellation_reason
- Proration is calculated when amendment effective_date is mid-period

### 4.3 Amendments
- Amendments MUST be in DRAFT status before submission for approval
- Upgrade amendments (UPGRADE) SHOULD take effect immediately or at next billing period based on configuration
- Downgrade amendments (DOWNGRADE) SHOULD take effect at the next billing period
- Price changes MUST generate proration line items when effective mid-period
- Amendments creating price changes exceeding 20% SHOULD require approval
- Cancellation amendments are processed as amendments for audit trail

### 4.4 Usage Recording
- Usage records are immutable once created; corrections require offsetting records
- Usage records MUST reference a valid metered subscription (product with is_metered = 1)
- Usage quantity MUST be non-negative
- Rated amount is calculated using tier overage pricing or contract-specific rates
- Usage records not yet invoiced (is_invoiced = 0) are picked up in the next billing run
- Usage MUST be recorded within the current billing period; late usage recording is flagged

### 4.5 Billing Runs
- Recurring billing runs are scheduled based on billing_cycle and billing_day_of_month
- A billing run generates line items for all ACTIVE subscriptions in the billing period
- Line item types: RECURRING (base charge), METERED (usage-based), PRORATION (mid-period changes)
- Failed billing runs MUST be retried before the period closes
- Partially completed billing runs record processed_subscriptions count for recovery
- Each subscription is billed at most once per billing period per run type

### 4.6 Payment Processing
- Payments are created after billing run completion
- Failed payments are retried with exponential backoff: 1 day, 3 days, 7 days
- Maximum retry count is 3; after which the subscription is flagged for suspension
- Refunds MUST NOT exceed the original payment amount
- Partial refunds are supported with reason documentation
- Payment method defaults from customer profile if not specified on subscription

### 4.7 Analytics
- MRR (Monthly Recurring Revenue) is calculated as sum of active subscription base prices normalized to monthly
- ARR (Annual Recurring Revenue) = MRR * 12
- Churn rate is calculated as cancelled subscriptions / total subscriptions at period start
- LTV (Lifetime Value) is estimated as ARPU / churn rate
- Analytics are computed asynchronously and cached in subscription_analytics table
- Analytics refresh SHOULD be triggered after billing run completion

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `subscription.created` | New subscription created | AR, REV, GL |
| `subscription.amended` | Subscription amendment completed | AR, REV |
| `subscription.cancelled` | Subscription cancelled | AR, REV, GL |
| `subscription.renewed` | Subscription renewed | AR, REV |
| `subscription.suspended` | Subscription suspended | AR, Notification |
| `subscription.resumed` | Subscription resumed | AR, Notification |
| `subscription.usage.recorded` | Metered usage recorded | Billing, Analytics |
| `subscription.billing.completed` | Billing run completed | AR, GL, REV |
| `subscription.payment.completed` | Payment processed successfully | AR, GL |
| `subscription.payment.failed` | Payment processing failed | Notification |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.subscription.v1;

service SubscriptionService {
    rpc CreateSubscription(CreateSubscriptionRequest) returns (CreateSubscriptionResponse);
    rpc GetSubscription(GetSubscriptionRequest) returns (GetSubscriptionResponse);
    rpc CancelSubscription(CancelSubscriptionRequest) returns (CancelSubscriptionResponse);
    rpc AmendSubscription(AmendSubscriptionRequest) returns (AmendSubscriptionResponse);
    rpc RecordUsage(RecordUsageRequest) returns (RecordUsageResponse);
    rpc RunBilling(RunBillingRequest) returns (RunBillingResponse);
    rpc GetSubscriptionAnalytics(GetSubscriptionAnalyticsRequest) returns (GetSubscriptionAnalyticsResponse);
    rpc GetCustomerSubscriptions(GetCustomerSubscriptionsRequest) returns (GetCustomerSubscriptionsResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **AR (Accounts Receivable):** Customer data for subscription assignment, invoice creation status
- **PRICING (Advanced Pricing):** Price list resolution for plan pricing, promotional discounts
- **REV (Revenue Management):** Revenue recognition templates for deferred revenue allocation
- **GL (General Ledger):** Account codes for billing accounting entries
- **Auth:** User identity for subscription owner assignment, approval workflows

### 6.2 Data Published To
- **AR (Accounts Receivable):** Invoice generation requests from billing runs, customer balance updates
- **REV (Revenue Management):** Revenue arrangement creation for subscription revenue recognition
- **GL (General Ledger):** Journal entries for billing revenue, deferred revenue, payment entries
- **Notification:** Subscription lifecycle alerts (trial ending, payment failed, renewal upcoming)
- **Workflow:** Approval routing for amendments exceeding thresholds, payment retry workflows
- **Reporting:** Subscription metrics, billing summaries, churn analytics

---

## 7. Migrations

1. V001: `subscription_products`
2. V002: `subscription_tiers`
3. V003: `subscriptions`
4. V004: `subscription_usage`
5. V005: `subscription_line_items`
6. V006: `subscription_amendments`
7. V007: `subscription_billing_runs`
8. V008: `subscription_payments`
9. V009: `subscription_analytics`
10. V010: Triggers for `updated_at`
