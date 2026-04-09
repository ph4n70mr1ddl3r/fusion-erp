# 97 - Loyalty Management Service Specification

## 1. Domain Overview

Loyalty Management provides comprehensive customer loyalty program management supporting points-based, tier-based, punch card, cashback, and hybrid loyalty programs. Manages program definition with configurable earning and redemption rules, tier hierarchies with upgrade/downgrade logic, member enrollment and lifecycle tracking, omnichannel points earning and redemption, rewards catalogs with inventory-controlled fulfillment, promotional multipliers and bonus campaigns, partner loyalty networks for cross-program earning, and loyalty analytics with member engagement scoring and tier distribution metrics. Integrates with OM for order-based points earning, AR for customer data, PROMO for promotional campaigns, and GL for loyalty liability accounting.

**Bounded Context:** Customer Loyalty Program & Rewards Management
**Service Name:** `loyalty-service`
**Database:** `data/loyalty.db`
**HTTP Port:** 8134 | **gRPC Port:** 9134

---

## 2. Database Schema

### 2.1 Loyalty Programs
```sql
CREATE TABLE loyalty_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_name TEXT NOT NULL,
    description TEXT,
    program_type TEXT NOT NULL
        CHECK(program_type IN ('POINTS','TIER','PUNCH_CARD','CASHBACK','HYBRID')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','SUSPENDED','CLOSED')),
    start_date TEXT NOT NULL,
    end_date TEXT,
    enrollment_type TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(enrollment_type IN ('OPEN','INVITATION','QUALIFICATION')),
    points_name TEXT NOT NULL DEFAULT 'Points',
    points_expiry_months INTEGER NOT NULL DEFAULT 0,
    terms_conditions TEXT,
    branding_config TEXT,                           -- JSON: colors, logos, theme
    max_tier_levels INTEGER NOT NULL DEFAULT 5,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_name)
);

CREATE INDEX idx_loyalty_programs_tenant_status ON loyalty_programs(tenant_id, status);
CREATE INDEX idx_loyalty_programs_tenant_type ON loyalty_programs(tenant_id, program_type);
CREATE INDEX idx_loyalty_programs_tenant_dates ON loyalty_programs(tenant_id, start_date, end_date);
CREATE INDEX idx_loyalty_programs_tenant_active ON loyalty_programs(tenant_id, is_active);
```

### 2.2 Loyalty Tiers
```sql
CREATE TABLE loyalty_tiers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    tier_name TEXT NOT NULL,
    tier_level INTEGER NOT NULL,
    min_points INTEGER NOT NULL DEFAULT 0,
    max_points INTEGER,
    benefits TEXT,                                  -- JSON: tier benefits and perks
    upgrade_bonus_points INTEGER NOT NULL DEFAULT 0,
    downgrade_grace_months INTEGER NOT NULL DEFAULT 0,
    color_code TEXT,
    icon_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES loyalty_programs(id),
    UNIQUE(tenant_id, program_id, tier_level)
);

CREATE INDEX idx_loyalty_tiers_tenant_program ON loyalty_tiers(tenant_id, program_id);
CREATE INDEX idx_loyalty_tiers_tenant_level ON loyalty_tiers(tenant_id, tier_level);
CREATE INDEX idx_loyalty_tiers_tenant_active ON loyalty_tiers(tenant_id, is_active);
```

### 2.3 Loyalty Members
```sql
CREATE TABLE loyalty_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    member_number TEXT NOT NULL,
    current_tier_id TEXT,
    lifetime_points INTEGER NOT NULL DEFAULT 0,
    current_points INTEGER NOT NULL DEFAULT 0,
    tier_progress_percentage REAL NOT NULL DEFAULT 0.0,
    enrollment_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUSPENDED','CANCELLED','EXPIRED')),
    last_activity_date TEXT,
    referred_by_member_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES loyalty_programs(id),
    FOREIGN KEY (current_tier_id) REFERENCES loyalty_tiers(id),
    UNIQUE(tenant_id, program_id, customer_id),
    UNIQUE(tenant_id, member_number)
);

CREATE INDEX idx_loyalty_members_tenant_program ON loyalty_members(tenant_id, program_id);
CREATE INDEX idx_loyalty_members_tenant_customer ON loyalty_members(tenant_id, customer_id);
CREATE INDEX idx_loyalty_members_tenant_status ON loyalty_members(tenant_id, status);
CREATE INDEX idx_loyalty_members_tenant_tier ON loyalty_members(tenant_id, current_tier_id);
CREATE INDEX idx_loyalty_members_tenant_activity ON loyalty_members(tenant_id, last_activity_date);
CREATE INDEX idx_loyalty_members_tenant_active ON loyalty_members(tenant_id, is_active);
```

### 2.4 Loyalty Transactions
```sql
CREATE TABLE loyalty_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    member_id TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('EARN','REDEEM','ADJUST','EXPIRE','BONUS','REFUND')),
    points_amount INTEGER NOT NULL,
    reference_type TEXT
        CHECK(reference_type IN ('ORDER','PROMOTION','REFERRAL','MANUAL')),
    reference_id TEXT,
    description TEXT,
    balance_after INTEGER NOT NULL,
    transaction_date TEXT NOT NULL,
    expiry_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (member_id) REFERENCES loyalty_members(id)
);

CREATE INDEX idx_loyalty_trans_tenant_member ON loyalty_transactions(tenant_id, member_id);
CREATE INDEX idx_loyalty_trans_tenant_type ON loyalty_transactions(tenant_id, transaction_type);
CREATE INDEX idx_loyalty_trans_tenant_date ON loyalty_transactions(tenant_id, transaction_date);
CREATE INDEX idx_loyalty_trans_tenant_ref ON loyalty_transactions(tenant_id, reference_type, reference_id);
CREATE INDEX idx_loyalty_trans_tenant_expiry ON loyalty_transactions(tenant_id, expiry_date);
```

### 2.5 Loyalty Rewards
```sql
CREATE TABLE loyalty_rewards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    reward_name TEXT NOT NULL,
    reward_type TEXT NOT NULL
        CHECK(reward_type IN ('PRODUCT','DISCOUNT','SERVICE','CASH_BACK','EXPERIENCE','CHARITY')),
    points_cost INTEGER NOT NULL,
    description TEXT,
    image_reference TEXT,
    terms TEXT,
    available_quantity INTEGER,
    max_redemptions_per_member INTEGER NOT NULL DEFAULT 0,
    is_limited_time INTEGER NOT NULL DEFAULT 0,
    available_from TEXT,
    available_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','SOLD_OUT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES loyalty_programs(id)
);

CREATE INDEX idx_loyalty_rewards_tenant_program ON loyalty_rewards(tenant_id, program_id);
CREATE INDEX idx_loyalty_rewards_tenant_type ON loyalty_rewards(tenant_id, reward_type);
CREATE INDEX idx_loyalty_rewards_tenant_status ON loyalty_rewards(tenant_id, status);
CREATE INDEX idx_loyalty_rewards_tenant_cost ON loyalty_rewards(tenant_id, points_cost);
CREATE INDEX idx_loyalty_rewards_tenant_active ON loyalty_rewards(tenant_id, is_active);
```

### 2.6 Loyalty Redemptions
```sql
CREATE TABLE loyalty_redemptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    member_id TEXT NOT NULL,
    reward_id TEXT NOT NULL,
    points_spent INTEGER NOT NULL,
    redemption_code TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','FULFILLED','CANCELLED','EXPIRED')),
    redeemed_date TEXT NOT NULL,
    fulfilled_date TEXT,
    fulfillment_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (member_id) REFERENCES loyalty_members(id),
    FOREIGN KEY (reward_id) REFERENCES loyalty_rewards(id),
    UNIQUE(tenant_id, redemption_code)
);

CREATE INDEX idx_loyalty_redemptions_tenant_member ON loyalty_redemptions(tenant_id, member_id);
CREATE INDEX idx_loyalty_redemptions_tenant_reward ON loyalty_redemptions(tenant_id, reward_id);
CREATE INDEX idx_loyalty_redemptions_tenant_status ON loyalty_redemptions(tenant_id, status);
CREATE INDEX idx_loyalty_redemptions_tenant_date ON loyalty_redemptions(tenant_id, redeemed_date);
CREATE INDEX idx_loyalty_redemptions_tenant_code ON loyalty_redemptions(tenant_id, redemption_code);
```

### 2.7 Loyalty Rules
```sql
CREATE TABLE loyalty_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('EARN','REDEEM','TIER_UPGRADE','TIER_DOWNGRADE')),
    conditions TEXT NOT NULL,                       -- JSON: rule trigger conditions
    action TEXT NOT NULL,                           -- JSON: action to execute
    priority INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES loyalty_programs(id)
);

CREATE INDEX idx_loyalty_rules_tenant_program ON loyalty_rules(tenant_id, program_id);
CREATE INDEX idx_loyalty_rules_tenant_type ON loyalty_rules(tenant_id, rule_type);
CREATE INDEX idx_loyalty_rules_tenant_active ON loyalty_rules(tenant_id, is_active);
CREATE INDEX idx_loyalty_rules_tenant_effective ON loyalty_rules(tenant_id, effective_from, effective_to);
```

### 2.8 Loyalty Promotions
```sql
CREATE TABLE loyalty_promotions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_id TEXT NOT NULL,
    promotion_name TEXT NOT NULL,
    promotion_type TEXT NOT NULL
        CHECK(promotion_type IN ('MULTIPLIER','BONUS_POINTS','DOUBLE_POINTS','EARLY_ACCESS')),
    multiplier_value REAL NOT NULL DEFAULT 1.0,
    bonus_points INTEGER NOT NULL DEFAULT 0,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    target_tier_id TEXT,
    target_products TEXT,                           -- JSON: product IDs or categories
    usage_limit INTEGER,
    usage_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','EXPIRED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (program_id) REFERENCES loyalty_programs(id)
);

CREATE INDEX idx_loyalty_promos_tenant_program ON loyalty_promotions(tenant_id, program_id);
CREATE INDEX idx_loyalty_promos_tenant_type ON loyalty_promotions(tenant_id, promotion_type);
CREATE INDEX idx_loyalty_promos_tenant_status ON loyalty_promotions(tenant_id, status);
CREATE INDEX idx_loyalty_promos_tenant_dates ON loyalty_promotions(tenant_id, start_date, end_date);
CREATE INDEX idx_loyalty_promos_tenant_active ON loyalty_promotions(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Programs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/loyalty/programs` | List loyalty programs |
| POST | `/api/v1/loyalty/programs` | Create loyalty program |
| GET | `/api/v1/loyalty/programs/{id}` | Get program details |
| PUT | `/api/v1/loyalty/programs/{id}` | Update program |
| PATCH | `/api/v1/loyalty/programs/{id}/status` | Update program status |

### 3.2 Tiers
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/loyalty/programs/{id}/tiers` | List tiers for program |
| POST | `/api/v1/loyalty/programs/{id}/tiers` | Create tier |
| GET | `/api/v1/loyalty/tiers/{id}` | Get tier details |
| PUT | `/api/v1/loyalty/tiers/{id}` | Update tier |
| DELETE | `/api/v1/loyalty/tiers/{id}` | Deactivate tier |

### 3.3 Members
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/loyalty/programs/{id}/enroll` | Enroll a customer |
| GET | `/api/v1/loyalty/members/{id}/profile` | Get member profile |
| GET | `/api/v1/loyalty/members/{id}/activity` | Get member activity history |
| GET | `/api/v1/loyalty/members` | List members (filter by program, status, tier) |
| PATCH | `/api/v1/loyalty/members/{id}/status` | Update member status |

### 3.4 Transactions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/loyalty/transactions/earn-points` | Record points earning |
| POST | `/api/v1/loyalty/transactions/redeem-points` | Record points redemption |
| POST | `/api/v1/loyalty/transactions/adjust` | Adjust member points |
| GET | `/api/v1/loyalty/transactions` | List transactions (filter by member, type, date) |
| GET | `/api/v1/loyalty/transactions/{id}` | Get transaction detail |

### 3.5 Rewards
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/loyalty/programs/{id}/rewards` | List rewards for program |
| POST | `/api/v1/loyalty/programs/{id}/rewards` | Create reward |
| GET | `/api/v1/loyalty/rewards/{id}` | Get reward details |
| PUT | `/api/v1/loyalty/rewards/{id}` | Update reward |
| DELETE | `/api/v1/loyalty/rewards/{id}` | Deactivate reward |

### 3.6 Redemptions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/loyalty/redemptions/redeem-reward` | Redeem a reward |
| GET | `/api/v1/loyalty/members/{id}/redemptions` | Get member redemption history |
| GET | `/api/v1/loyalty/redemptions/{id}` | Get redemption detail |
| POST | `/api/v1/loyalty/redemptions/{id}/fulfill` | Fulfill a redemption |
| POST | `/api/v1/loyalty/redemptions/{id}/cancel` | Cancel a redemption |

### 3.7 Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/loyalty/programs/{id}/rules` | List rules for program |
| POST | `/api/v1/loyalty/programs/{id}/rules` | Create rule |
| GET | `/api/v1/loyalty/rules/{id}` | Get rule details |
| PUT | `/api/v1/loyalty/rules/{id}` | Update rule |
| DELETE | `/api/v1/loyalty/rules/{id}` | Deactivate rule |

### 3.8 Promotions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/loyalty/programs/{id}/promotions` | List promotions for program |
| POST | `/api/v1/loyalty/programs/{id}/promotions` | Create promotion |
| GET | `/api/v1/loyalty/promotions/{id}` | Get promotion details |
| PUT | `/api/v1/loyalty/promotions/{id}` | Update promotion |
| PATCH | `/api/v1/loyalty/promotions/{id}/status` | Update promotion status |

### 3.9 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/loyalty/analytics/program-metrics` | Get program-level metrics |
| GET | `/api/v1/loyalty/analytics/member-analytics` | Get member analytics |
| GET | `/api/v1/loyalty/analytics/tier-distribution` | Get tier distribution breakdown |
| GET | `/api/v1/loyalty/analytics/points-liability` | Get outstanding points liability |

---

## 4. Business Rules

### 4.1 Program Management
1. Each loyalty program MUST have a unique name within a tenant.
2. Program status transitions MUST follow the defined state machine: DRAFT -> ACTIVE -> SUSPENDED -> CLOSED.
3. A program in CLOSED status MUST NOT allow new enrollments, points earning, or redemptions.
4. The enrollment_type field determines how customers join: OPEN allows self-enrollment, INVITATION requires an invite code, QUALIFICATION requires meeting defined criteria.
5. points_expiry_months of 0 means points do not expire; a positive value causes unredeemed points to expire after the specified months from earning date.

### 4.2 Tier Management
6. Tiers within a program MUST have unique tier_level values and non-overlapping min_points/max_points ranges.
7. Tier progression MUST be evaluated after every EARN transaction; if lifetime_points crosses the next tier's min_points threshold, the member MUST be upgraded.
8. Tier downgrade SHOULD apply the downgrade_grace_months period before demoting a member whose activity falls below the tier's min_points.
9. The lowest tier in a program MUST have min_points set to 0.

### 4.3 Member Management
10. A customer MUST NOT be enrolled in the same program more than once; the combination of (tenant_id, program_id, customer_id) MUST be unique.
11. Member numbers MUST be auto-generated and unique within a tenant.
12. Member status transitions MUST follow: ACTIVE -> SUSPENDED -> CANCELLED -> EXPIRED; a CANCELLED or EXPIRED member MUST NOT earn or redeem points.

### 4.4 Points Transactions
13. Points transactions are immutable once created; corrections MUST use ADJUST transactions referencing the original transaction.
14. EARN transactions MUST result in a positive points_amount; REDEEM, EXPIRE, and REFUND transactions MUST result in a negative or zero points_amount.
15. Redemption MUST fail if the member's current_points balance is insufficient to cover the points_cost of the reward.
16. Points earned MUST carry an expiry_date calculated as transaction_date + points_expiry_months (when applicable); the system MUST run a scheduled process to expire points past their expiry_date.

### 4.5 Rewards and Redemptions
17. A reward with available_quantity set MUST NOT be redeemed when the quantity reaches zero.
18. max_redemptions_per_member greater than 0 MUST be enforced per member per reward.
19. Limited-time rewards (is_limited_time = 1) MUST only be redeemable between available_from and available_to dates.
20. Redemption codes MUST be unique and generated using a secure random algorithm.

### 4.6 Fraud Prevention
21. Points earning from the same reference_type and reference_id MUST NOT be duplicated; the system MUST check for idempotency before creating an EARN transaction.
22. Unusual earning patterns (e.g., more than 10 transactions in a 24-hour period for a single member) SHOULD trigger a fraud alert and MAY suspend the member pending review.
23. Points adjustments (type ADJUST) MUST require a description and SHOULD require supervisor authorization.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.loyalty.v1;

service LoyaltyManagementService {
    // Program management
    rpc CreateProgram(CreateProgramRequest) returns (CreateProgramResponse);
    rpc GetProgram(GetProgramRequest) returns (GetProgramResponse);
    rpc ListPrograms(ListProgramsRequest) returns (ListProgramsResponse);

    // Tier management
    rpc CreateTier(CreateTierRequest) returns (CreateTierResponse);
    rpc ListTiers(ListTiersRequest) returns (ListTiersResponse);

    // Member management
    rpc EnrollMember(EnrollMemberRequest) returns (EnrollMemberResponse);
    rpc GetMemberProfile(GetMemberProfileRequest) returns (GetMemberProfileResponse);
    rpc GetMemberActivity(GetMemberActivityRequest) returns (GetMemberActivityResponse);

    // Points transactions
    rpc EarnPoints(EarnPointsRequest) returns (EarnPointsResponse);
    rpc RedeemPoints(RedeemPointsRequest) returns (RedeemPointsResponse);
    rpc AdjustPoints(AdjustPointsRequest) returns (AdjustPointsResponse);

    // Rewards and redemptions
    rpc ListRewards(ListRewardsRequest) returns (ListRewardsResponse);
    rpc RedeemReward(RedeemRewardRequest) returns (RedeemRewardResponse);

    // Analytics
    rpc GetProgramMetrics(GetProgramMetricsRequest) returns (GetProgramMetricsResponse);
    rpc GetTierDistribution(GetTierDistributionRequest) returns (GetTierDistributionResponse);
}

message CreateProgramRequest {
    string tenant_id = 1;
    string program_name = 2;
    string description = 3;
    string program_type = 4;
    string start_date = 5;
    string end_date = 6;
    string enrollment_type = 7;
    string points_name = 8;
    int32 points_expiry_months = 9;
    string terms_conditions = 10;
    string branding_config = 11;
}

message CreateProgramResponse {
    string program_id = 1;
    string program_name = 2;
    string status = 3;
}

message GetProgramRequest {
    string tenant_id = 1;
    string program_id = 2;
}

message GetProgramResponse {
    string program_id = 1;
    string program_name = 2;
    string program_type = 3;
    string status = 4;
    int32 total_members = 5;
    int64 total_points_issued = 6;
    int64 total_points_redeemed = 7;
}

message ListProgramsRequest {
    string tenant_id = 1;
    string status = 2;
    string program_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListProgramsResponse {
    repeated GetProgramResponse programs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CreateTierRequest {
    string tenant_id = 1;
    string program_id = 2;
    string tier_name = 3;
    int32 tier_level = 4;
    int64 min_points = 5;
    int64 max_points = 6;
    string benefits = 7;
    int64 upgrade_bonus_points = 8;
    int32 downgrade_grace_months = 9;
    string color_code = 10;
    string icon_reference = 11;
}

message CreateTierResponse {
    string tier_id = 1;
    string tier_name = 2;
    int32 tier_level = 3;
}

message ListTiersRequest {
    string tenant_id = 1;
    string program_id = 2;
}

message ListTiersResponse {
    repeated TierInfo tiers = 1;
}

message TierInfo {
    string tier_id = 1;
    string tier_name = 2;
    int32 tier_level = 3;
    int64 min_points = 4;
    int64 max_points = 5;
    string benefits = 6;
}

message EnrollMemberRequest {
    string tenant_id = 1;
    string program_id = 2;
    string customer_id = 3;
    string referred_by_member_id = 4;
}

message EnrollMemberResponse {
    string member_id = 1;
    string member_number = 2;
    string enrollment_date = 3;
    string current_tier_id = 4;
}

message GetMemberProfileRequest {
    string tenant_id = 1;
    string member_id = 2;
}

message GetMemberProfileResponse {
    string member_id = 1;
    string member_number = 2;
    string customer_id = 3;
    string program_id = 4;
    string current_tier_id = 5;
    int64 lifetime_points = 6;
    int64 current_points = 7;
    double tier_progress_percentage = 8;
    string status = 9;
    string enrollment_date = 10;
    string last_activity_date = 11;
}

message GetMemberActivityRequest {
    string tenant_id = 1;
    string member_id = 2;
    string transaction_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetMemberActivityResponse {
    repeated TransactionInfo transactions = 1;
    string next_page_token = 2;
}

message TransactionInfo {
    string transaction_id = 1;
    string transaction_type = 2;
    int64 points_amount = 3;
    string reference_type = 4;
    string reference_id = 5;
    string description = 6;
    int64 balance_after = 7;
    string transaction_date = 8;
    string expiry_date = 9;
}

message EarnPointsRequest {
    string tenant_id = 1;
    string member_id = 2;
    int64 points_amount = 3;
    string reference_type = 4;
    string reference_id = 5;
    string description = 6;
    string promotion_id = 7;
}

message EarnPointsResponse {
    string transaction_id = 1;
    int64 points_earned = 2;
    int64 balance_after = 3;
    bool tier_upgraded = 4;
    string new_tier_id = 5;
}

message RedeemPointsRequest {
    string tenant_id = 1;
    string member_id = 2;
    int64 points_amount = 3;
    string reference_type = 4;
    string reference_id = 5;
    string description = 6;
}

message RedeemPointsResponse {
    string transaction_id = 1;
    int64 points_redeemed = 2;
    int64 balance_after = 3;
}

message AdjustPointsRequest {
    string tenant_id = 1;
    string member_id = 2;
    int64 points_amount = 3;
    string description = 4;
}

message AdjustPointsResponse {
    string transaction_id = 1;
    int64 points_adjusted = 2;
    int64 balance_after = 3;
}

message ListRewardsRequest {
    string tenant_id = 1;
    string program_id = 2;
    string reward_type = 3;
    string status = 4;
}

message ListRewardsResponse {
    repeated RewardInfo rewards = 1;
}

message RewardInfo {
    string reward_id = 1;
    string reward_name = 2;
    string reward_type = 3;
    int64 points_cost = 4;
    string description = 5;
    string status = 6;
}

message RedeemRewardRequest {
    string tenant_id = 1;
    string member_id = 2;
    string reward_id = 3;
}

message RedeemRewardResponse {
    string redemption_id = 1;
    string redemption_code = 2;
    int64 points_spent = 3;
    string status = 4;
    string redeemed_date = 5;
}

message GetProgramMetricsRequest {
    string tenant_id = 1;
    string program_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message GetProgramMetricsResponse {
    string program_id = 1;
    int32 total_members = 2;
    int32 active_members = 3;
    int64 total_points_issued = 4;
    int64 total_points_redeemed = 5;
    int64 total_points_expired = 6;
    int64 outstanding_points_liability = 7;
    double average_points_per_member = 8;
    int32 total_redemptions = 9;
}

message GetTierDistributionRequest {
    string tenant_id = 1;
    string program_id = 2;
}

message GetTierDistributionResponse {
    repeated TierDistributionInfo tiers = 1;
}

message TierDistributionInfo {
    string tier_id = 1;
    string tier_name = 2;
    int32 member_count = 3;
    double percentage = 4;
    int64 avg_lifetime_points = 5;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `ar-service` | Customer data for enrollment, customer profile and contact information |
| `om-service` | Order completion events for points earning, order line items for earning rules |
| `pricing-service` | Promotional pricing and discount rules for loyalty promotions |
| `promo-service` | Promotion campaigns and coupon configurations |
| `gl-service` | GL account codes for loyalty liability accounting |
| `auth-service` | User identity for member assignment and audit trails |

### Published To
| Service | Data |
|---------|------|
| `ar-service` | Customer loyalty tier updates for customer profile enrichment |
| `om-service` | Discount application requests from loyalty reward redemptions |
| `gl-service` | Loyalty liability journal entries, points issuance and redemption accounting |
| `notification-service` | Tier upgrade notifications, points expiry warnings, reward availability alerts |
| `reporting-service` | Loyalty program metrics, member engagement data, tier distribution analytics |
| `workflow-service` | Fraud alert routing, manual adjustment approval workflows |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `loyalty.member.enrolled` | `loyalty.events` | `{ member_id, program_id, customer_id, member_number, enrollment_date }` | New member enrolled in loyalty program |
| `loyalty.points.earned` | `loyalty.events` | `{ member_id, transaction_id, points_amount, balance_after, reference_type, reference_id }` | Points earned by a member |
| `loyalty.points.redeemed` | `loyalty.events` | `{ member_id, transaction_id, points_amount, balance_after, reference_type, reference_id }` | Points redeemed by a member |
| `loyalty.tier.upgraded` | `loyalty.events` | `{ member_id, program_id, old_tier_id, new_tier_id, lifetime_points }` | Member promoted to a higher tier |
| `loyalty.reward.redeemed` | `loyalty.events` | `{ redemption_id, member_id, reward_id, points_spent, redemption_code }` | Reward redeemed by a member |
| `loyalty.promotion.applied` | `loyalty.events` | `{ member_id, promotion_id, promotion_type, bonus_points, multiplier_value }` | Promotion bonus applied to member |

---

## 8. Migrations

1. V001: `loyalty_programs`
2. V002: `loyalty_tiers`
3. V003: `loyalty_members`
4. V004: `loyalty_transactions`
5. V005: `loyalty_rewards`
6. V006: `loyalty_redemptions`
7. V007: `loyalty_rules`
8. V008: `loyalty_promotions`
9. V009: Triggers for `updated_at`
