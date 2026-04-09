# 79 - Sales Performance Service Specification

## 1. Domain Overview

Sales Performance manages sales commission plans, incentive rules, credit assignments, commission calculations, payment processing, dispute management, and performance scorecards. Commission plans define how sales representatives earn incentives based on quota attainment, deal revenue, or custom performance metrics. Rules within plans specify calculation formulas, rate tables, tiers, and thresholds. Credit assignments allocate deal revenue to participating sales team members (direct sellers, overlays, managers). Commission calculations run periodically to compute earned commissions based on closed deals, applied credits, and plan rules. Disputes allow representatives to challenge commission amounts. Scorecards aggregate performance metrics across quotas, pipeline, activities, and commissions for individual and team review. Integrates with Sales Automation for deal and pipeline data, Payroll for commission disbursement, and General Ledger for accrual and expense posting.

**Bounded Context:** Commission & Incentive Management
**Service Name:** `salesperf-service`
**Database:** `data/salesperf.db`
**HTTP Port:** 8111 | **gRPC Port:** 9111

---

## 2. Database Schema

### 2.1 Commission Plans
```sql
CREATE TABLE salesperf_commission_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL
        CHECK(plan_type IN ('REVENUE','MARGIN','UNIT','QUOTA_ATTAINMENT','TIERED','DRAW_AGAINST','CUSTOM')),
    description TEXT,
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT,
    currency TEXT NOT NULL DEFAULT 'USD',
    calculation_frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(calculation_frequency IN ('WEEKLY','MONTHLY','QUARTERLY','ANNUALLY','DEAL_BASED')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','ARCHIVED')),
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','SUBMITTED','APPROVED','REJECTED')),
    approved_by TEXT,
    approved_at TEXT,
    max_commission_amount INTEGER,
    clawback_period_days INTEGER NOT NULL DEFAULT 0,
    minimum_threshold REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_commission_plans_tenant_status ON salesperf_commission_plans(tenant_id, status);
CREATE INDEX idx_commission_plans_tenant_dates ON salesperf_commission_plans(tenant_id, effective_start_date, effective_end_date);
CREATE INDEX idx_commission_plans_tenant_type ON salesperf_commission_plans(tenant_id, plan_type);
CREATE INDEX idx_commission_plans_tenant_active ON salesperf_commission_plans(tenant_id, is_active);
```

### 2.2 Commission Rules
```sql
CREATE TABLE salesperf_commission_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_order INTEGER NOT NULL DEFAULT 0,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('FLAT_RATE','TIERED_RATE','PERCENTAGE','FIXED_AMOUNT','BONUS','MULTIPLIER','SLIDING_SCALE')),
    base_type TEXT NOT NULL
        CHECK(base_type IN ('REVENUE','MARGIN','UNITS','ATTAINMENT_PERCENTAGE','CUSTOM')),
    rate_or_amount REAL NOT NULL,
    tier_config TEXT,
    condition_expression TEXT,
    minimum_base_value INTEGER,
    maximum_base_value INTEGER,
    is_retroactive INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_id, rule_name)
);

CREATE INDEX idx_commission_rules_tenant_plan ON salesperf_commission_rules(tenant_id, plan_id);
CREATE INDEX idx_commission_rules_tenant_type ON salesperf_commission_rules(tenant_id, rule_type);
CREATE INDEX idx_commission_rules_tenant_active ON salesperf_commission_rules(tenant_id, is_active);
```

### 2.3 Commission Participants
```sql
CREATE TABLE salesperf_commission_participants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'REP'
        CHECK(role IN ('REP','SENIOR_REP','MANAGER','DIRECTOR','OVERLAY')),
    credit_type TEXT NOT NULL DEFAULT 'PRIMARY'
        CHECK(credit_type IN ('PRIMARY','SHARED','OVERLAY','MANAGER_OVERRIDE')),
    credit_percentage REAL NOT NULL DEFAULT 100.0,
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT,
    draw_amount INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ON_LEAVE','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_id, user_id, effective_start_date)
);

CREATE INDEX idx_participants_tenant_plan ON salesperf_commission_participants(tenant_id, plan_id);
CREATE INDEX idx_participants_tenant_user ON salesperf_commission_participants(tenant_id, user_id);
CREATE INDEX idx_participants_tenant_status ON salesperf_commission_participants(tenant_id, status);
CREATE INDEX idx_participants_tenant_active ON salesperf_commission_participants(tenant_id, is_active);
```

### 2.4 Credit Assignments
```sql
CREATE TABLE salesperf_credit_assignments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    opportunity_id TEXT NOT NULL,
    deal_amount INTEGER NOT NULL DEFAULT 0,
    credit_type TEXT NOT NULL
        CHECK(credit_type IN ('PRIMARY','SHARED','OVERLAY','MANAGER_OVERRIDE','SPLIT')),
    participant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    credited_amount INTEGER NOT NULL DEFAULT 0,
    credit_percentage REAL NOT NULL DEFAULT 100.0,
    product_line TEXT,
    deal_close_date TEXT,
    period_id TEXT,
    is_clawed_back INTEGER NOT NULL DEFAULT 0,
    clawback_reason TEXT,
    clawback_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_credits_tenant_opportunity ON salesperf_credit_assignments(tenant_id, opportunity_id);
CREATE INDEX idx_credits_tenant_participant ON salesperf_credit_assignments(tenant_id, participant_id);
CREATE INDEX idx_credits_tenant_user ON salesperf_credit_assignments(tenant_id, user_id);
CREATE INDEX idx_credits_tenant_period ON salesperf_credit_assignments(tenant_id, period_id);
CREATE INDEX idx_credits_tenant_clawback ON salesperf_credit_assignments(tenant_id, is_clawed_back);
CREATE INDEX idx_credits_tenant_active ON salesperf_credit_assignments(tenant_id, is_active);
```

### 2.5 Commission Transactions
```sql
CREATE TABLE salesperf_commission_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('EARNED','ADJUSTMENT','CLAWBACK','DRAW','DRAW_RECOVERY','BONUS','HOLD','RELEASE')),
    participant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    rule_id TEXT,
    credit_assignment_id TEXT,
    period_id TEXT NOT NULL,
    opportunity_id TEXT,
    base_amount INTEGER NOT NULL DEFAULT 0,
    commission_rate REAL NOT NULL DEFAULT 0.0,
    commission_amount INTEGER NOT NULL DEFAULT 0,
    currency TEXT NOT NULL DEFAULT 'USD',
    transaction_date TEXT NOT NULL,
    description TEXT,
    reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, transaction_number)
);

CREATE INDEX idx_transactions_tenant_participant ON salesperf_commission_transactions(tenant_id, participant_id);
CREATE INDEX idx_transactions_tenant_user ON salesperf_commission_transactions(tenant_id, user_id);
CREATE INDEX idx_transactions_tenant_plan ON salesperf_commission_transactions(tenant_id, plan_id);
CREATE INDEX idx_transactions_tenant_period ON salesperf_commission_transactions(tenant_id, period_id);
CREATE INDEX idx_transactions_tenant_type ON salesperf_commission_transactions(tenant_id, transaction_type);
CREATE INDEX idx_transactions_tenant_date ON salesperf_commission_transactions(tenant_id, transaction_date);
CREATE INDEX idx_transactions_tenant_active ON salesperf_commission_transactions(tenant_id, is_active);
```

### 2.6 Commission Calculations
```sql
CREATE TABLE salesperf_commission_calculations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    calculation_name TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    calculation_date TEXT NOT NULL,
    total_credits INTEGER NOT NULL DEFAULT 0,
    total_commission INTEGER NOT NULL DEFAULT 0,
    participant_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','CANCELLED')),
    started_at TEXT,
    completed_at TEXT,
    error_message TEXT,
    run_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_calculations_tenant_plan ON salesperf_commission_calculations(tenant_id, plan_id);
CREATE INDEX idx_calculations_tenant_period ON salesperf_commission_calculations(tenant_id, period_id);
CREATE INDEX idx_calculations_tenant_status ON salesperf_commission_calculations(tenant_id, status);
CREATE INDEX idx_calculations_tenant_date ON salesperf_commission_calculations(tenant_id, calculation_date);
CREATE INDEX idx_calculations_tenant_active ON salesperf_commission_calculations(tenant_id, is_active);
```

### 2.7 Commission Periods
```sql
CREATE TABLE salesperf_commission_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('WEEKLY','MONTHLY','QUARTERLY','ANNUALLY')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    fiscal_year INTEGER NOT NULL,
    fiscal_quarter INTEGER,
    is_locked INTEGER NOT NULL DEFAULT 0,
    locked_by TEXT,
    locked_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_name)
);

CREATE INDEX idx_commission_periods_tenant_type ON salesperf_commission_periods(tenant_id, period_type);
CREATE INDEX idx_commission_periods_tenant_fiscal ON salesperf_commission_periods(tenant_id, fiscal_year, fiscal_quarter);
CREATE INDEX idx_commission_periods_tenant_dates ON salesperf_commission_periods(tenant_id, start_date, end_date);
CREATE INDEX idx_commission_periods_tenant_active ON salesperf_commission_periods(tenant_id, is_active);
```

### 2.8 Commission Disputes
```sql
CREATE TABLE salesperf_commission_disputes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dispute_number TEXT NOT NULL,
    transaction_id TEXT NOT NULL,
    participant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    dispute_type TEXT NOT NULL
        CHECK(dispute_type IN ('INCORRECT_AMOUNT','MISSING_CREDIT','DUPLICATE_CREDIT','WRONG_RULE','CLAWBACK_DISPUTE','OTHER')),
    disputed_amount INTEGER NOT NULL,
    original_amount INTEGER NOT NULL,
    requested_amount INTEGER,
    description TEXT NOT NULL,
    supporting_documents TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','UNDER_REVIEW','APPROVED','REJECTED','RESOLVED')),
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('HIGH','MEDIUM','LOW')),
    assigned_reviewer TEXT,
    resolution_notes TEXT,
    resolved_at TEXT,
    resolved_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dispute_number)
);

CREATE INDEX idx_disputes_tenant_status ON salesperf_commission_disputes(tenant_id, status);
CREATE INDEX idx_disputes_tenant_user ON salesperf_commission_disputes(tenant_id, user_id);
CREATE INDEX idx_disputes_tenant_transaction ON salesperf_commission_disputes(tenant_id, transaction_id);
CREATE INDEX idx_disputes_tenant_reviewer ON salesperf_commission_disputes(tenant_id, assigned_reviewer);
CREATE INDEX idx_disputes_tenant_active ON salesperf_commission_disputes(tenant_id, is_active);
```

### 2.9 Performance Scorecards
```sql
CREATE TABLE salesperf_performance_scorecards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    quota_amount INTEGER NOT NULL DEFAULT 0,
    attainment_amount INTEGER NOT NULL DEFAULT 0,
    attainment_percentage REAL NOT NULL DEFAULT 0.0,
    total_pipeline INTEGER NOT NULL DEFAULT 0,
    weighted_pipeline INTEGER NOT NULL DEFAULT 0,
    deals_closed INTEGER NOT NULL DEFAULT 0,
    deals_won INTEGER NOT NULL DEFAULT 0,
    deals_lost INTEGER NOT NULL DEFAULT 0,
    win_rate REAL NOT NULL DEFAULT 0.0,
    average_deal_size INTEGER NOT NULL DEFAULT 0,
    average_sales_cycle_days INTEGER NOT NULL DEFAULT 0,
    total_commission_earned INTEGER NOT NULL DEFAULT 0,
    activity_count INTEGER NOT NULL DEFAULT 0,
    meetings_completed INTEGER NOT NULL DEFAULT 0,
    rank_in_team INTEGER,
    overall_score REAL NOT NULL DEFAULT 0.0,
    calculated_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, user_id, period_id)
);

CREATE INDEX idx_scorecards_tenant_user ON salesperf_performance_scorecards(tenant_id, user_id);
CREATE INDEX idx_scorecards_tenant_period ON salesperf_performance_scorecards(tenant_id, period_id);
CREATE INDEX idx_scorecards_tenant_rank ON salesperf_performance_scorecards(tenant_id, period_id, rank_in_team);
CREATE INDEX idx_scorecards_tenant_active ON salesperf_performance_scorecards(tenant_id, is_active);
```

### 2.10 Commission Payments
```sql
CREATE TABLE salesperf_commission_payments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    payment_number TEXT NOT NULL,
    participant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    period_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    gross_commission INTEGER NOT NULL DEFAULT 0,
    draw_deduction INTEGER NOT NULL DEFAULT 0,
    adjustment_amount INTEGER NOT NULL DEFAULT 0,
    clawback_amount INTEGER NOT NULL DEFAULT 0,
    net_commission INTEGER NOT NULL DEFAULT 0,
    currency TEXT NOT NULL DEFAULT 'USD',
    payment_date TEXT,
    payment_method TEXT CHECK(payment_method IN ('PAYROLL','WIRE','CHECK','OTHER')),
    payment_reference TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','PAID','CANCELLED','ON_HOLD')),
    approved_by TEXT,
    approved_at TEXT,
    paid_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, payment_number)
);

CREATE INDEX idx_payments_tenant_participant ON salesperf_commission_payments(tenant_id, participant_id);
CREATE INDEX idx_payments_tenant_user ON salesperf_commission_payments(tenant_id, user_id);
CREATE INDEX idx_payments_tenant_period ON salesperf_commission_payments(tenant_id, period_id);
CREATE INDEX idx_payments_tenant_status ON salesperf_commission_payments(tenant_id, status);
CREATE INDEX idx_payments_tenant_date ON salesperf_commission_payments(tenant_id, payment_date);
CREATE INDEX idx_payments_tenant_active ON salesperf_commission_payments(tenant_id, is_active);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/plans` | Create a commission plan |
| GET | `/api/v1/plans` | List commission plans with filtering |
| GET | `/api/v1/plans/{id}` | Get plan with rules and participants |
| PUT | `/api/v1/plans/{id}` | Update plan details |
| DELETE | `/api/v1/plans/{id}` | Deactivate a commission plan |
| POST | `/api/v1/plans/{id}/submit` | Submit plan for approval |
| PATCH | `/api/v1/plans/{id}/approve` | Approve or reject a commission plan |
| POST | `/api/v1/plans/{planId}/rules` | Create a commission rule |
| GET | `/api/v1/plans/{planId}/rules` | List rules for a plan |
| GET | `/api/v1/plans/{planId}/rules/{id}` | Get rule by ID |
| PUT | `/api/v1/plans/{planId}/rules/{id}` | Update a commission rule |
| DELETE | `/api/v1/plans/{planId}/rules/{id}` | Remove a commission rule |
| POST | `/api/v1/plans/{planId}/participants` | Add participant to a plan |
| GET | `/api/v1/plans/{planId}/participants` | List participants for a plan |
| PUT | `/api/v1/plans/{planId}/participants/{id}` | Update participant details |
| DELETE | `/api/v1/plans/{planId}/participants/{id}` | Remove participant from plan |
| POST | `/api/v1/credits` | Create a credit assignment |
| GET | `/api/v1/credits` | List credit assignments with filtering |
| GET | `/api/v1/credits/{id}` | Get credit assignment by ID |
| PUT | `/api/v1/credits/{id}` | Update a credit assignment |
| DELETE | `/api/v1/credits/{id}` | Remove a credit assignment |
| POST | `/api/v1/credits/{id}/clawback` | Claw back a credit assignment |
| GET | `/api/v1/credits/by-opportunity/{id}` | Get all credits for an opportunity |
| POST | `/api/v1/calculations` | Trigger a commission calculation run |
| GET | `/api/v1/calculations` | List calculation runs |
| GET | `/api/v1/calculations/{id}` | Get calculation run details |
| POST | `/api/v1/calculations/{id}/cancel` | Cancel a running calculation |
| GET | `/api/v1/transactions` | List commission transactions with filtering |
| GET | `/api/v1/transactions/{id}` | Get transaction by ID |
| POST | `/api/v1/transactions/{id}/adjust` | Create a commission adjustment |
| GET | `/api/v1/transactions/by-user/{userId}` | List transactions for a user |
| GET | `/api/v1/transactions/by-period/{periodId}` | List transactions for a period |
| POST | `/api/v1/periods` | Create a commission period |
| GET | `/api/v1/periods` | List commission periods |
| GET | `/api/v1/periods/{id}` | Get period details |
| PUT | `/api/v1/periods/{id}` | Update a commission period |
| POST | `/api/v1/periods/{id}/lock` | Lock a period to prevent modifications |
| POST | `/api/v1/periods/{id}/unlock` | Unlock a period |
| POST | `/api/v1/disputes` | Create a commission dispute |
| GET | `/api/v1/disputes` | List disputes with filtering |
| GET | `/api/v1/disputes/{id}` | Get dispute by ID |
| PUT | `/api/v1/disputes/{id}` | Update a dispute |
| PATCH | `/api/v1/disputes/{id}/resolve` | Resolve a dispute |
| PATCH | `/api/v1/disputes/{id}/assign` | Assign dispute to reviewer |
| GET | `/api/v1/payments` | List commission payments with filtering |
| GET | `/api/v1/payments/{id}` | Get payment by ID |
| POST | `/api/v1/payments/{id}/approve` | Approve a commission payment |
| POST | `/api/v1/payments/batch-approve` | Batch approve multiple payments |
| POST | `/api/v1/payments/generate` | Generate payments for a period |
| GET | `/api/v1/scorecards` | List performance scorecards |
| GET | `/api/v1/scorecards/{id}` | Get scorecard by ID |
| GET | `/api/v1/scorecards/by-user/{userId}` | Get scorecards for a user |
| POST | `/api/v1/scorecards/recalculate` | Recalculate scorecards for a period |
| GET | `/api/v1/dashboard/summary` | Get commission dashboard summary |
| GET | `/api/v1/dashboard/leaderboard` | Get sales leaderboard by commission or attainment |

---

## 4. Business Rules

1. **Plan Uniqueness**: Each commission plan MUST have a unique `plan_code` within a tenant. A plan with `ACTIVE` status MUST have at least one active rule and one active participant.

2. **Rule Evaluation Order**: Commission rules within a plan MUST be evaluated in ascending `rule_order`. The system MUST stop evaluating rules for a transaction once a matching rule produces a non-zero commission, unless the plan specifies cumulative rule evaluation.

3. **Credit Allocation**: The total `credit_percentage` across all participants for a single opportunity MUST NOT exceed 100%. Each opportunity MUST have exactly one `PRIMARY` credit assignment.

4. **Commission Calculation**: Commission amounts MUST be calculated as integer cents. For percentage-based rules, the commission MUST be `base_amount * rate / 100` rounded to the nearest cent. For tiered rules, each tier MUST apply its rate only to the portion of the base amount falling within that tier.

5. **Clawback Processing**: When a credit is clawed back, the system MUST reverse the corresponding commission transactions by creating `CLAWBACK` type transactions with negative amounts. The clawback MUST reference the original transaction.

6. **Draw Against Commission**: For `DRAW_AGAINST` plans, the system MUST track cumulative draw payments against earned commissions. Earned commissions MUST first offset outstanding draw balances before any net payment is generated.

7. **Period Locking**: Once a commission period is locked, no new credit assignments, transaction adjustments, or recalculation requests MUST be accepted for that period. A period MUST be unlocked only by an administrator.

8. **Dispute Workflow**: A dispute MUST be created in `OPEN` status and progress through `UNDER_REVIEW`, then to `APPROVED`, `REJECTED`, or `RESOLVED`. Only the assigned reviewer or an administrator MAY change the dispute status. An approved dispute MUST generate an adjustment transaction.

9. **Payment Approval**: A commission payment MUST be in `PENDING` status to be approved. The approving user MUST NOT be the same as the payment recipient. Approved payments MUST generate a corresponding payroll integration event.

10. **Maximum Commission Cap**: If a plan has a `max_commission_amount` set, the total commission earned by any participant under that plan for any period MUST NOT exceed this cap. Excess commission MUST be flagged but NOT paid.

11. **Minimum Threshold**: If a plan has a `minimum_threshold`, the participant's attainment percentage MUST meet or exceed the threshold before any commission is earned. Below-threshold performance MUST result in zero commission.

12. **Transaction Integrity**: Each commission transaction MUST reference a valid credit assignment and plan rule. Transactions in `HOLD` status MUST be excluded from payment calculations until released.

13. **Retroactive Adjustments**: Commission rules marked as `is_retroactive = 0` MUST NOT apply to credit assignments with deal close dates before the rule's effective date. Retroactive rules SHOULD be used with caution and require manager approval.

14. **Scorecard Ranking**: Scorecard rankings within a team MUST be based on `overall_score` in descending order. Ties in `overall_score` MUST be broken by `attainment_amount` in descending order.

15. **Calculation Idempotency**: Re-running a commission calculation for the same plan and period MUST produce identical results unless underlying credits or rules have changed. Previous calculation results MUST be archived before overwriting.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package salesperf;

option go_package = "github.com/fusion/salesperfpb";

service SalesPerformanceService {
    // Commission Plans
    rpc CreateCommissionPlan(CreateCommissionPlanRequest) returns (CommissionPlanResponse);
    rpc GetCommissionPlan(GetCommissionPlanRequest) returns (CommissionPlanResponse);
    rpc ListCommissionPlans(ListCommissionPlansRequest) returns (ListCommissionPlansResponse);
    rpc UpdateCommissionPlan(UpdateCommissionPlanRequest) returns (CommissionPlanResponse);
    rpc DeleteCommissionPlan(DeleteCommissionPlanRequest) returns (DeleteResponse);
    rpc SubmitCommissionPlan(SubmitCommissionPlanRequest) returns (CommissionPlanResponse);
    rpc ApproveCommissionPlan(ApproveCommissionPlanRequest) returns (CommissionPlanResponse);

    // Commission Rules
    rpc CreateCommissionRule(CreateCommissionRuleRequest) returns (CommissionRuleResponse);
    rpc ListCommissionRules(ListCommissionRulesRequest) returns (ListCommissionRulesResponse);
    rpc GetCommissionRule(GetCommissionRuleRequest) returns (CommissionRuleResponse);
    rpc UpdateCommissionRule(UpdateCommissionRuleRequest) returns (CommissionRuleResponse);
    rpc DeleteCommissionRule(DeleteCommissionRuleRequest) returns (DeleteResponse);

    // Participants
    rpc AddParticipant(AddParticipantRequest) returns (ParticipantResponse);
    rpc ListParticipants(ListParticipantsRequest) returns (ListParticipantsResponse);
    rpc UpdateParticipant(UpdateParticipantRequest) returns (ParticipantResponse);
    rpc RemoveParticipant(RemoveParticipantRequest) returns (DeleteResponse);

    // Credit Assignments
    rpc CreateCreditAssignment(CreateCreditAssignmentRequest) returns (CreditAssignmentResponse);
    rpc ListCreditAssignments(ListCreditAssignmentsRequest) returns (ListCreditAssignmentsResponse);
    rpc GetCreditAssignment(GetCreditAssignmentRequest) returns (CreditAssignmentResponse);
    rpc UpdateCreditAssignment(UpdateCreditAssignmentRequest) returns (CreditAssignmentResponse);
    rpc RemoveCreditAssignment(RemoveCreditAssignmentRequest) returns (DeleteResponse);
    rpc ClawbackCredit(ClawbackCreditRequest) returns (CreditAssignmentResponse);

    // Commission Calculations
    rpc RunCalculation(RunCalculationRequest) returns (CalculationResponse);
    rpc GetCalculation(GetCalculationRequest) returns (CalculationResponse);
    rpc ListCalculations(ListCalculationsRequest) returns (ListCalculationsResponse);
    rpc CancelCalculation(CancelCalculationRequest) returns (CalculationResponse);

    // Commission Transactions
    rpc ListTransactions(ListTransactionsRequest) returns (ListTransactionsResponse);
    rpc GetTransaction(GetTransactionRequest) returns (TransactionResponse);
    rpc AdjustTransaction(AdjustTransactionRequest) returns (TransactionResponse);

    // Commission Periods
    rpc CreatePeriod(CreatePeriodRequest) returns (PeriodResponse);
    rpc ListPeriods(ListPeriodsRequest) returns (ListPeriodsResponse);
    rpc GetPeriod(GetPeriodRequest) returns (PeriodResponse);
    rpc UpdatePeriod(UpdatePeriodRequest) returns (PeriodResponse);
    rpc LockPeriod(LockPeriodRequest) returns (PeriodResponse);
    rpc UnlockPeriod(UnlockPeriodRequest) returns (PeriodResponse);

    // Disputes
    rpc CreateDispute(CreateDisputeRequest) returns (DisputeResponse);
    rpc ListDisputes(ListDisputesRequest) returns (ListDisputesResponse);
    rpc GetDispute(GetDisputeRequest) returns (DisputeResponse);
    rpc UpdateDispute(UpdateDisputeRequest) returns (DisputeResponse);
    rpc ResolveDispute(ResolveDisputeRequest) returns (DisputeResponse);
    rpc AssignDispute(AssignDisputeRequest) returns (DisputeResponse);

    // Payments
    rpc ListPayments(ListPaymentsRequest) returns (ListPaymentsResponse);
    rpc GetPayment(GetPaymentRequest) returns (PaymentResponse);
    rpc ApprovePayment(ApprovePaymentRequest) returns (PaymentResponse);
    rpc BatchApprovePayments(BatchApprovePaymentsRequest) returns (BatchApprovePaymentsResponse);
    rpc GeneratePayments(GeneratePaymentsRequest) returns (ListPaymentsResponse);

    // Scorecards
    rpc ListScorecards(ListScorecardsRequest) returns (ListScorecardsResponse);
    rpc GetScorecard(GetScorecardRequest) returns (ScorecardResponse);
    rpc GetScorecardsByUser(GetScorecardsByUserRequest) returns (ListScorecardsResponse);
    rpc RecalculateScorecards(RecalculateScorecardsRequest) returns (ListScorecardsResponse);
}

message CommissionPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_code = 4;
    string plan_type = 5;
    string description = 6;
    string effective_start_date = 7;
    string effective_end_date = 8;
    string currency = 9;
    string calculation_frequency = 10;
    string status = 11;
    string approval_status = 12;
    int64 max_commission_amount = 13;
    int32 clawback_period_days = 14;
    double minimum_threshold = 15;
    string created_at = 16;
    string updated_at = 17;
    int32 version = 18;
}

message CommissionRule {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string rule_name = 4;
    int32 rule_order = 5;
    string rule_type = 6;
    string base_type = 7;
    double rate_or_amount = 8;
    string tier_config = 9;
    string condition_expression = 10;
    int64 minimum_base_value = 11;
    int64 maximum_base_value = 12;
    bool is_retroactive = 13;
    string created_at = 14;
    string updated_at = 15;
    int32 version = 16;
}

message Participant {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string user_id = 4;
    string role = 5;
    string credit_type = 6;
    double credit_percentage = 7;
    string effective_start_date = 8;
    string effective_end_date = 9;
    int64 draw_amount = 10;
    string status = 11;
    string created_at = 12;
    string updated_at = 13;
    int32 version = 14;
}

message CreditAssignment {
    string id = 1;
    string tenant_id = 2;
    string opportunity_id = 3;
    int64 deal_amount = 4;
    string credit_type = 5;
    string participant_id = 6;
    string user_id = 7;
    int64 credited_amount = 8;
    double credit_percentage = 9;
    string product_line = 10;
    string deal_close_date = 11;
    string period_id = 12;
    bool is_clawed_back = 13;
    string created_at = 14;
    string updated_at = 15;
    int32 version = 16;
}

message CommissionTransaction {
    string id = 1;
    string tenant_id = 2;
    string transaction_number = 3;
    string transaction_type = 4;
    string participant_id = 5;
    string user_id = 6;
    string plan_id = 7;
    string rule_id = 8;
    string credit_assignment_id = 9;
    string period_id = 10;
    string opportunity_id = 11;
    int64 base_amount = 12;
    double commission_rate = 13;
    int64 commission_amount = 14;
    string currency = 15;
    string transaction_date = 16;
    string description = 17;
    string created_at = 18;
    string updated_at = 19;
    int32 version = 20;
}

message Calculation {
    string id = 1;
    string tenant_id = 2;
    string calculation_name = 3;
    string plan_id = 4;
    string period_id = 5;
    string calculation_date = 6;
    int64 total_credits = 7;
    int64 total_commission = 8;
    int32 participant_count = 9;
    string status = 10;
    string started_at = 11;
    string completed_at = 12;
    string created_at = 13;
    string updated_at = 14;
    int32 version = 15;
}

message CommissionPeriod {
    string id = 1;
    string tenant_id = 2;
    string period_name = 3;
    string period_type = 4;
    string start_date = 5;
    string end_date = 6;
    int32 fiscal_year = 7;
    int32 fiscal_quarter = 8;
    bool is_locked = 9;
    string created_at = 10;
    string updated_at = 11;
    int32 version = 12;
}

message Dispute {
    string id = 1;
    string tenant_id = 2;
    string dispute_number = 3;
    string transaction_id = 4;
    string participant_id = 5;
    string user_id = 6;
    string dispute_type = 7;
    int64 disputed_amount = 8;
    int64 original_amount = 9;
    int64 requested_amount = 10;
    string description = 11;
    string status = 12;
    string priority = 13;
    string assigned_reviewer = 14;
    string resolution_notes = 15;
    string resolved_at = 16;
    string resolved_by = 17;
    string created_at = 18;
    string updated_at = 19;
    int32 version = 20;
}

message Scorecard {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string period_id = 4;
    int64 quota_amount = 5;
    int64 attainment_amount = 6;
    double attainment_percentage = 7;
    int64 total_pipeline = 8;
    int64 weighted_pipeline = 9;
    int32 deals_closed = 10;
    int32 deals_won = 11;
    int32 deals_lost = 12;
    double win_rate = 13;
    int64 average_deal_size = 14;
    int32 average_sales_cycle_days = 15;
    int64 total_commission_earned = 16;
    int32 rank_in_team = 17;
    double overall_score = 18;
    string calculated_at = 19;
    string created_at = 20;
    string updated_at = 21;
    int32 version = 22;
}

message Payment {
    string id = 1;
    string tenant_id = 2;
    string payment_number = 3;
    string participant_id = 4;
    string user_id = 5;
    string period_id = 6;
    string plan_id = 7;
    int64 gross_commission = 8;
    int64 draw_deduction = 9;
    int64 adjustment_amount = 10;
    int64 clawback_amount = 11;
    int64 net_commission = 12;
    string currency = 13;
    string payment_date = 14;
    string payment_method = 15;
    string status = 16;
    string approved_by = 17;
    string approved_at = 18;
    string created_at = 19;
    string updated_at = 20;
    int32 version = 21;
}

// Request/Response messages
message CreateCommissionPlanRequest { CommissionPlan plan = 1; }
message GetCommissionPlanRequest { string tenant_id = 1; string id = 2; }
message ListCommissionPlansRequest {
    string tenant_id = 1;
    string status = 2;
    string plan_type = 3;
    int32 page = 4;
    int32 page_size = 5;
}
message UpdateCommissionPlanRequest { CommissionPlan plan = 1; }
message DeleteCommissionPlanRequest { string tenant_id = 1; string id = 2; }
message SubmitCommissionPlanRequest { string tenant_id = 1; string id = 2; }
message ApproveCommissionPlanRequest { string tenant_id = 1; string id = 2; bool approve = 3; string reason = 4; }
message CommissionPlanResponse { CommissionPlan plan = 1; }
message ListCommissionPlansResponse { repeated CommissionPlan plans = 1; int32 total = 2; }

message CreateCommissionRuleRequest { CommissionRule rule = 1; }
message ListCommissionRulesRequest { string tenant_id = 1; string plan_id = 2; }
message GetCommissionRuleRequest { string tenant_id = 1; string id = 2; }
message UpdateCommissionRuleRequest { CommissionRule rule = 1; }
message DeleteCommissionRuleRequest { string tenant_id = 1; string id = 2; }
message CommissionRuleResponse { CommissionRule rule = 1; }
message ListCommissionRulesResponse { repeated CommissionRule rules = 1; int32 total = 2; }

message AddParticipantRequest { Participant participant = 1; }
message ListParticipantsRequest { string tenant_id = 1; string plan_id = 2; string status = 3; }
message UpdateParticipantRequest { Participant participant = 1; }
message RemoveParticipantRequest { string tenant_id = 1; string id = 2; }
message ParticipantResponse { Participant participant = 1; }
message ListParticipantsResponse { repeated Participant participants = 1; int32 total = 2; }

message CreateCreditAssignmentRequest { CreditAssignment credit = 1; }
message ListCreditAssignmentsRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string user_id = 3;
    string period_id = 4;
    string credit_type = 5;
}
message GetCreditAssignmentRequest { string tenant_id = 1; string id = 2; }
message UpdateCreditAssignmentRequest { CreditAssignment credit = 1; }
message RemoveCreditAssignmentRequest { string tenant_id = 1; string id = 2; }
message ClawbackCreditRequest { string tenant_id = 1; string id = 2; string reason = 3; }
message CreditAssignmentResponse { CreditAssignment credit = 1; }
message ListCreditAssignmentsResponse { repeated CreditAssignment credits = 1; int32 total = 2; }

message RunCalculationRequest { string tenant_id = 1; string plan_id = 2; string period_id = 3; string calculation_name = 4; }
message GetCalculationRequest { string tenant_id = 1; string id = 2; }
message ListCalculationsRequest { string tenant_id = 1; string plan_id = 2; string period_id = 3; string status = 4; }
message CancelCalculationRequest { string tenant_id = 1; string id = 2; }
message CalculationResponse { Calculation calculation = 1; }
message ListCalculationsResponse { repeated Calculation calculations = 1; int32 total = 2; }

message ListTransactionsRequest {
    string tenant_id = 1;
    string user_id = 2;
    string plan_id = 3;
    string period_id = 4;
    string transaction_type = 5;
    int32 page = 6;
    int32 page_size = 7;
}
message GetTransactionRequest { string tenant_id = 1; string id = 2; }
message AdjustTransactionRequest { string tenant_id = 1; string id = 2; int64 adjustment_amount = 3; string reason = 4; }
message TransactionResponse { CommissionTransaction transaction = 1; }
message ListTransactionsResponse { repeated CommissionTransaction transactions = 1; int32 total = 2; }

message CreatePeriodRequest { CommissionPeriod period = 1; }
message ListPeriodsRequest { string tenant_id = 1; string period_type = 2; int32 fiscal_year = 3; }
message GetPeriodRequest { string tenant_id = 1; string id = 2; }
message UpdatePeriodRequest { CommissionPeriod period = 1; }
message LockPeriodRequest { string tenant_id = 1; string id = 2; }
message UnlockPeriodRequest { string tenant_id = 1; string id = 2; }
message PeriodResponse { CommissionPeriod period = 1; }
message ListPeriodsResponse { repeated CommissionPeriod periods = 1; int32 total = 2; }

message CreateDisputeRequest { Dispute dispute = 1; }
message ListDisputesRequest {
    string tenant_id = 1;
    string status = 2;
    string user_id = 3;
    string assigned_reviewer = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message GetDisputeRequest { string tenant_id = 1; string id = 2; }
message UpdateDisputeRequest { Dispute dispute = 1; }
message ResolveDisputeRequest { string tenant_id = 1; string id = 2; string resolution = 3; int64 adjusted_amount = 4; bool approved = 5; }
message AssignDisputeRequest { string tenant_id = 1; string id = 2; string reviewer_id = 3; }
message DisputeResponse { Dispute dispute = 1; }
message ListDisputesResponse { repeated Dispute disputes = 1; int32 total = 2; }

message ListPaymentsRequest {
    string tenant_id = 1;
    string user_id = 2;
    string period_id = 3;
    string status = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message GetPaymentRequest { string tenant_id = 1; string id = 2; }
message ApprovePaymentRequest { string tenant_id = 1; string id = 2; }
message BatchApprovePaymentsRequest { string tenant_id = 1; repeated string payment_ids = 2; }
message GeneratePaymentsRequest { string tenant_id = 1; string period_id = 2; string plan_id = 3; }
message PaymentResponse { Payment payment = 1; }
message ListPaymentsResponse { repeated Payment payments = 1; int32 total = 2; }
message BatchApprovePaymentsResponse { repeated Payment payments = 1; int32 approved_count = 2; int32 failed_count = 3; }

message ListScorecardsRequest {
    string tenant_id = 1;
    string period_id = 2;
    string user_id = 3;
    int32 page = 4;
    int32 page_size = 5;
}
message GetScorecardRequest { string tenant_id = 1; string id = 2; }
message GetScorecardsByUserRequest { string tenant_id = 1; string user_id = 2; int32 fiscal_year = 3; }
message RecalculateScorecardsRequest { string tenant_id = 1; string period_id = 2; }
message ScorecardResponse { Scorecard scorecard = 1; }
message ListScorecardsResponse { repeated Scorecard scorecards = 1; int32 total = 2; }

message DeleteResponse { bool success = 1; }
```

---

## 6. Inter-Service Integration

| Integration | Service | Direction | Purpose |
|-------------|---------|-----------|---------|
| Sales Automation | `sales-service` | Inbound | Receive opportunity won/lost events, deal amounts, close dates, and assigned users for credit creation |
| Sales Automation | `sales-service` | Inbound | Receive pipeline data, activity counts, and deal metrics for scorecard calculation |
| Sales Planning | `salesplanning-service` | Inbound | Receive quota allocations and attainment data for quota-based commission rules |
| Compensation | `salesplanning-service` | Outbound | Publish commission-eligible attainment data for territory planning adjustments |
| Payroll | `payroll-service` | Outbound | Send approved commission payments as payroll earnings entries for disbursement |
| Payroll | `payroll-service` | Inbound | Receive payment confirmation and payroll run status updates |
| General Ledger | `gl-service` | Outbound | Post commission expense journal entries (debit commission expense, credit accrued commissions payable) |
| General Ledger | `gl-service` | Inbound | Receive GL period close status to prevent commission adjustments in closed periods |
| Identity Service | `identity-service` | Inbound | Validate participant user IDs, roles, and organizational hierarchy for credit allocation |
| Enterprise Planning | `epm-service` | Outbound | Publish commission forecast data for expense budgeting and planning |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `PlanCreated` | `{ tenant_id, plan_id, plan_code, plan_name, plan_type, effective_start_date, created_at }` | Published when a new commission plan is created |
| `PlanApproved` | `{ tenant_id, plan_id, plan_code, approved_by, approved_at }` | Published when a commission plan is approved |
| `CreditsAssigned` | `{ tenant_id, credit_id, opportunity_id, participant_id, user_id, credited_amount, credit_percentage, credit_type, created_at }` | Published when a credit assignment is created or updated |
| `CreditsClawedBack` | `{ tenant_id, credit_id, opportunity_id, user_id, clawback_amount, clawback_reason, clawback_date }` | Published when a credit assignment is clawed back |
| `CalculationStarted` | `{ tenant_id, calculation_id, plan_id, period_id, started_at }` | Published when a commission calculation run begins |
| `CalculationCompleted` | `{ tenant_id, calculation_id, plan_id, period_id, total_credits, total_commission, participant_count, completed_at }` | Published when a commission calculation run finishes |
| `CommissionCalculated` | `{ tenant_id, transaction_id, participant_id, user_id, plan_id, base_amount, commission_rate, commission_amount, period_id, calculated_at }` | Published when an individual commission transaction is generated |
| `CommissionAdjusted` | `{ tenant_id, transaction_id, original_amount, adjustment_amount, new_amount, reason, adjusted_at }` | Published when a commission transaction is manually adjusted |
| `DisputeCreated` | `{ tenant_id, dispute_id, dispute_number, transaction_id, user_id, disputed_amount, dispute_type, created_at }` | Published when a commission dispute is filed |
| `DisputeResolved` | `{ tenant_id, dispute_id, dispute_number, resolution, adjusted_amount, resolved_by, resolved_at }` | Published when a commission dispute is resolved |
| `PaymentGenerated` | `{ tenant_id, payment_id, payment_number, user_id, period_id, net_commission, generated_at }` | Published when a commission payment is generated |
| `PaymentApproved` | `{ tenant_id, payment_id, payment_number, user_id, net_commission, approved_by, approved_at }` | Published when a commission payment is approved for disbursement |
| `PeriodLocked` | `{ tenant_id, period_id, period_name, locked_by, locked_at }` | Published when a commission period is locked |
| `ScorecardUpdated` | `{ tenant_id, scorecard_id, user_id, period_id, attainment_percentage, total_commission_earned, rank_in_team, updated_at }` | Published when a performance scorecard is recalculated |
