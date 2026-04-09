# 211 - Media & Entertainment Industry Solution Specification

## 1. Domain Overview

Media & Entertainment provides industry-specific capabilities for content rights management, royalty processing, advertising sales, production budgeting, and audience analytics. Supports intellectual property and content rights tracking with territory and window management, royalty calculation and payment processing for creators and rights holders, advertising inventory management with yield optimization, production budget tracking with cost-to-complete forecasting, and audience measurement analytics. Enables media companies, publishers, broadcasters, and streaming platforms to manage content lifecycle, maximize revenue from rights and advertising, and track production economics. Integrates with Financials, Contract Management, and Analytics.

**Bounded Context:** Content Rights, Royalty & Advertising Management
**Service Name:** `media-entertainment-service`
**Database:** `data/media_entertainment.db`
**HTTP Port:** 8229 | **gRPC Port:** 9229

---

## 2. Database Schema

### 2.1 Content Rights
```sql
CREATE TABLE me_content_rights (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    content_id TEXT NOT NULL,
    title TEXT NOT NULL,
    content_type TEXT NOT NULL CHECK(content_type IN ('FILM','TV_SERIES','MUSIC','BOOK','PODCAST','GAME','LIVE_EVENT')),
    rights_holder TEXT NOT NULL,
    rights_type TEXT NOT NULL CHECK(rights_type IN ('EXCLUSIVE','NON_EXCLUSIVE','TERRITORIAL','PLATFORM','WINDOWED')),
    territory TEXT NOT NULL,                      -- JSON: countries/regions
    platform TEXT NOT NULL CHECK(platform IN ('THEATRICAL','LINEAR_TV','STREAMING','DIGITAL','PHYSICAL','RADIO','ALL')),
    window_start TEXT,
    window_end TEXT,
    acquisition_cost_cents INTEGER NOT NULL DEFAULT 0,
    minimum_guarantee_cents INTEGER NOT NULL DEFAULT 0,
    revenue_share_pct REAL NOT NULL DEFAULT 0,
    sub_licensing_allowed INTEGER NOT NULL DEFAULT 0,
    contract_reference TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','TERMINATED','DISPUTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, content_id, platform, territory)
);

CREATE INDEX idx_me_rights_content ON me_content_rights(content_id, status);
CREATE INDEX idx_me_rights_holder ON me_content_rights(rights_holder);
CREATE INDEX idx_me_rights_window ON me_content_rights(window_start, window_end);
```

### 2.2 Royalty Processing
```sql
CREATE TABLE me_royalties (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    content_id TEXT NOT NULL,
    rights_holder_id TEXT NOT NULL,
    rights_holder_name TEXT NOT NULL,
    royalty_period TEXT NOT NULL,
    revenue_source TEXT NOT NULL CHECK(revenue_source IN ('STREAMING','THEATRICAL','MERCHANDISE','SYNC_LICENSE','MECHANICAL','PERFORMANCE','OTHER')),
    gross_revenue_cents INTEGER NOT NULL DEFAULT 0,
    deductions_cents INTEGER NOT NULL DEFAULT 0,
    net_revenue_cents INTEGER NOT NULL DEFAULT 0,
    royalty_rate_pct REAL NOT NULL DEFAULT 0,
    royalty_amount_cents INTEGER NOT NULL DEFAULT 0,
    recoupable_advance_cents INTEGER NOT NULL DEFAULT 0,
    recouped_to_date_cents INTEGER NOT NULL DEFAULT 0,
    payable_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    payment_reference TEXT,
    paid_at TEXT,
    status TEXT NOT NULL DEFAULT 'CALCULATED'
        CHECK(status IN ('CALCULATED','APPROVED','PAID','DISPUTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, content_id, rights_holder_id, royalty_period, revenue_source)
);

CREATE INDEX idx_me_royalty_holder ON me_royalties(rights_holder_id, status);
CREATE INDEX idx_me_royalty_period ON me_royalties(royalty_period DESC);
CREATE INDEX idx_me_royalty_content ON me_royalties(content_id);
```

### 2.3 Advertising Sales
```sql
CREATE TABLE me_ad_inventory (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    inventory_code TEXT NOT NULL,
    content_id TEXT,
    placement_type TEXT NOT NULL CHECK(placement_type IN ('PRE_ROLL','MID_ROLL','POST_ROLL','DISPLAY','SPONSORSHIP','PRODUCT_PLACEMENT','BILLBOARD')),
    platform TEXT NOT NULL,
    available_impressions INTEGER NOT NULL DEFAULT 0,
    sold_impressions INTEGER NOT NULL DEFAULT 0,
    remaining_impressions INTEGER NOT NULL DEFAULT 0,
    cpm_rate_cents INTEGER NOT NULL DEFAULT 0,
    minimum_commitment_cents INTEGER NOT NULL DEFAULT 0,
    demographic_targeting TEXT,                   -- JSON: demo criteria
    geo_targeting TEXT,                           -- JSON: geo criteria
    flight_start TEXT NOT NULL,
    flight_end TEXT NOT NULL,
    advertiser_id TEXT,
    agency_id TEXT,
    status TEXT NOT NULL DEFAULT 'AVAILABLE'
        CHECK(status IN ('AVAILABLE','RESERVED','SOLD','EXPIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, inventory_code)
);

CREATE INDEX idx_me_ad_status ON me_ad_inventory(tenant_id, status);
CREATE INDEX idx_me_ad_flight ON me_ad_inventory(flight_start, flight_end);
CREATE INDEX idx_me_ad_advertiser ON me_ad_inventory(advertiser_id);
```

### 2.4 Production Budgets
```sql
CREATE TABLE me_production_budgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    production_id TEXT NOT NULL,
    production_name TEXT NOT NULL,
    production_type TEXT NOT NULL CHECK(production_type IN ('FILM','TV_EPISODE','SERIES','COMMERCIAL','MUSIC_VIDEO','LIVE_EVENT')),
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    above_the_line_cents INTEGER NOT NULL DEFAULT 0,
    below_the_line_cents INTEGER NOT NULL DEFAULT 0,
    post_production_cents INTEGER NOT NULL DEFAULT 0,
    marketing_cents INTEGER NOT NULL DEFAULT 0,
    committed_cents INTEGER NOT NULL DEFAULT 0,
    spent_cents INTEGER NOT NULL DEFAULT 0,
    remaining_cents INTEGER NOT NULL DEFAULT 0,
    estimated_to_complete_cents INTEGER NOT NULL DEFAULT 0,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    shooting_days_total INTEGER NOT NULL DEFAULT 0,
    shooting_days_completed INTEGER NOT NULL DEFAULT 0,
    production_start TEXT,
    production_end TEXT,
    producer_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'GREEN'
        CHECK(status IN ('GREEN','YELLOW','RED','COMPLETED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, production_id)
);

CREATE INDEX idx_me_budget_status ON me_production_budgets(producer_id, status);
CREATE INDEX idx_me_budget_type ON me_production_budgets(production_type);
```

### 2.5 Audience Analytics
```sql
CREATE TABLE me_audience_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    content_id TEXT NOT NULL,
    period TEXT NOT NULL,
    platform TEXT NOT NULL,
    total_views INTEGER NOT NULL DEFAULT 0,
    unique_viewers INTEGER NOT NULL DEFAULT 0,
    avg_watch_time_minutes REAL NOT NULL DEFAULT 0,
    completion_rate_pct REAL NOT NULL DEFAULT 0,
    demographic_breakdown TEXT,                   -- JSON: age/gender distribution
    geo_breakdown TEXT,                           -- JSON: country distribution
    social_mentions INTEGER NOT NULL DEFAULT 0,
    sentiment_score REAL NOT NULL DEFAULT 0,
    revenue_per_view_cents REAL NOT NULL DEFAULT 0,
    subscriber_conversions INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, content_id, period, platform)
);

CREATE INDEX idx_me_audience_content ON me_audience_analytics(content_id, period DESC);
CREATE INDEX idx_me_audience_platform ON me_audience_analytics(platform, period DESC);
```

---

## 3. API Endpoints

### 3.1 Content Rights
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/media/rights` | Register content rights |
| GET | `/api/v1/media/rights` | List rights |
| GET | `/api/v1/media/rights/{id}` | Get rights |
| GET | `/api/v1/media/rights/availability` | Check rights availability |

### 3.2 Royalties
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/media/royalties/calculate` | Calculate royalties |
| GET | `/api/v1/media/royalties` | List royalties |
| POST | `/api/v1/media/royalties/{id}/approve` | Approve royalty |
| GET | `/api/v1/media/royalties/statements` | Royalty statements |

### 3.3 Advertising
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/media/ad-inventory` | List ad inventory |
| POST | `/api/v1/media/ad-inventory/{id}/reserve` | Reserve inventory |
| POST | `/api/v1/media/ad-inventory/{id}/sell` | Sell inventory |
| GET | `/api/v1/media/ad-inventory/yield` | Yield report |

### 3.4 Production Budgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/media/productions` | Create production |
| GET | `/api/v1/media/productions` | List productions |
| GET | `/api/v1/media/productions/{id}` | Get production |
| POST | `/api/v1/media/productions/{id}/cost-report` | Submit cost report |

### 3.5 Audience Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/media/audience` | Audience dashboard |
| GET | `/api/v1/media/audience/{contentId}` | Content performance |
| GET | `/api/v1/media/audience/trends` | Audience trends |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `media.rights.expiring` | `{ rights_id, content, window_end }` | Rights window expiring |
| `media.royalty.calculated` | `{ royalty_id, holder, amount }` | Royalty calculated |
| `media.ad.sold` | `{ inventory_id, advertiser, amount }` | Ad inventory sold |
| `media.budget.overrun` | `{ production_id, variance }` | Budget overrun detected |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `contract.signed` | Contract (83) | Activate content rights |
| `payment.processed` | Payment (14) | Record royalty payment |
| `campaign.launched` | Marketing (61) | Track marketing impact |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fusion.media_entertainment.v1;

import "google/protobuf/timestamp.proto";

// ── Service ──────────────────────────────────────────────────────────
service MediaEntertainmentService {
  // Content Rights
  rpc RegisterContentRights(RegisterContentRightsRequest) returns (ContentRights);
  rpc GetContentRights(GetContentRightsRequest) returns (ContentRights);
  rpc ListContentRights(ListContentRightsRequest) returns (ListContentRightsResponse);
  rpc CheckRightsAvailability(CheckRightsAvailabilityRequest) returns (CheckRightsAvailabilityResponse);

  // Royalties
  rpc CalculateRoyalties(CalculateRoyaltiesRequest) returns (CalculateRoyaltiesResponse);
  rpc ApproveRoyalty(ApproveRoyaltyRequest) returns (Royalty);

  // Advertising
  rpc ListAdInventory(ListAdInventoryRequest) returns (ListAdInventoryResponse);
  rpc ReserveAdInventory(ReserveAdInventoryRequest) returns (AdInventory);
  rpc SellAdInventory(SellAdInventoryRequest) returns (AdInventory);

  // Production Budgets
  rpc CreateProduction(CreateProductionRequest) returns (ProductionBudget);
  rpc GetProduction(GetProductionRequest) returns (ProductionBudget);

  // Audience Analytics
  rpc GetContentPerformance(GetContentPerformanceRequest) returns (AudienceAnalytics);
  rpc GetAudienceTrends(GetAudienceTrendsRequest) returns (GetAudienceTrendsResponse);
}

// ── Enums ────────────────────────────────────────────────────────────
enum ContentType {
  CONTENT_TYPE_UNSPECIFIED = 0;
  CONTENT_TYPE_FILM = 1;
  CONTENT_TYPE_TV_SERIES = 2;
  CONTENT_TYPE_MUSIC = 3;
  CONTENT_TYPE_BOOK = 4;
  CONTENT_TYPE_PODCAST = 5;
  CONTENT_TYPE_GAME = 6;
  CONTENT_TYPE_LIVE_EVENT = 7;
}

enum RightsType {
  RIGHTS_TYPE_UNSPECIFIED = 0;
  RIGHTS_TYPE_EXCLUSIVE = 1;
  RIGHTS_TYPE_NON_EXCLUSIVE = 2;
  RIGHTS_TYPE_TERRITORIAL = 3;
  RIGHTS_TYPE_PLATFORM = 4;
  RIGHTS_TYPE_WINDOWED = 5;
}

enum Platform {
  PLATFORM_UNSPECIFIED = 0;
  PLATFORM_THEATRICAL = 1;
  PLATFORM_LINEAR_TV = 2;
  PLATFORM_STREAMING = 3;
  PLATFORM_DIGITAL = 4;
  PLATFORM_PHYSICAL = 5;
  PLATFORM_RADIO = 6;
  PLATFORM_ALL = 7;
}

enum ContentRightsStatus {
  CONTENT_RIGHTS_STATUS_UNSPECIFIED = 0;
  CONTENT_RIGHTS_STATUS_ACTIVE = 1;
  CONTENT_RIGHTS_STATUS_EXPIRED = 2;
  CONTENT_RIGHTS_STATUS_TERMINATED = 3;
  CONTENT_RIGHTS_STATUS_DISPUTED = 4;
}

enum RevenueSource {
  REVENUE_SOURCE_UNSPECIFIED = 0;
  REVENUE_SOURCE_STREAMING = 1;
  REVENUE_SOURCE_THEATRICAL = 2;
  REVENUE_SOURCE_MERCHANDISE = 3;
  REVENUE_SOURCE_SYNC_LICENSE = 4;
  REVENUE_SOURCE_MECHANICAL = 5;
  REVENUE_SOURCE_PERFORMANCE = 6;
  REVENUE_SOURCE_OTHER = 7;
}

enum RoyaltyStatus {
  ROYALTY_STATUS_UNSPECIFIED = 0;
  ROYALTY_STATUS_CALCULATED = 1;
  ROYALTY_STATUS_APPROVED = 2;
  ROYALTY_STATUS_PAID = 3;
  ROYALTY_STATUS_DISPUTED = 4;
}

enum PlacementType {
  PLACEMENT_TYPE_UNSPECIFIED = 0;
  PLACEMENT_TYPE_PRE_ROLL = 1;
  PLACEMENT_TYPE_MID_ROLL = 2;
  PLACEMENT_TYPE_POST_ROLL = 3;
  PLACEMENT_TYPE_DISPLAY = 4;
  PLACEMENT_TYPE_SPONSORSHIP = 5;
  PLACEMENT_TYPE_PRODUCT_PLACEMENT = 6;
  PLACEMENT_TYPE_BILLBOARD = 7;
}

enum AdInventoryStatus {
  AD_STATUS_UNSPECIFIED = 0;
  AD_STATUS_AVAILABLE = 1;
  AD_STATUS_RESERVED = 2;
  AD_STATUS_SOLD = 3;
  AD_STATUS_EXPIRED = 4;
}

enum ProductionType {
  PRODUCTION_TYPE_UNSPECIFIED = 0;
  PRODUCTION_TYPE_FILM = 1;
  PRODUCTION_TYPE_TV_EPISODE = 2;
  PRODUCTION_TYPE_SERIES = 3;
  PRODUCTION_TYPE_COMMERCIAL = 4;
  PRODUCTION_TYPE_MUSIC_VIDEO = 5;
  PRODUCTION_TYPE_LIVE_EVENT = 6;
}

enum ProductionStatus {
  PRODUCTION_STATUS_UNSPECIFIED = 0;
  PRODUCTION_STATUS_GREEN = 1;
  PRODUCTION_STATUS_YELLOW = 2;
  PRODUCTION_STATUS_RED = 3;
  PRODUCTION_STATUS_COMPLETED = 4;
}

// ── Common Messages ──────────────────────────────────────────────────
message AuditInfo {
  string created_at = 1;
  string updated_at = 2;
  string created_by = 3;
  string updated_by = 4;
  int32 version = 5;
}

// ── Content Rights Messages ──────────────────────────────────────────
message ContentRights {
  string id = 1;
  string tenant_id = 2;
  string content_id = 3;
  string title = 4;
  ContentType content_type = 5;
  string rights_holder = 6;
  RightsType rights_type = 7;
  string territory = 8;                // JSON
  Platform platform = 9;
  string window_start = 10;
  string window_end = 11;
  int64 acquisition_cost_cents = 12;
  int64 minimum_guarantee_cents = 13;
  double revenue_share_pct = 14;
  bool sub_licensing_allowed = 15;
  string contract_reference = 16;
  ContentRightsStatus status = 17;
  AuditInfo audit = 18;
}

message RegisterContentRightsRequest {
  string tenant_id = 1;
  string content_id = 2;
  string title = 3;
  ContentType content_type = 4;
  string rights_holder = 5;
  RightsType rights_type = 6;
  string territory = 7;                // JSON
  Platform platform = 8;
  string window_start = 9;
  string window_end = 10;
  int64 acquisition_cost_cents = 11;
  int64 minimum_guarantee_cents = 12;
  double revenue_share_pct = 13;
  bool sub_licensing_allowed = 14;
  string contract_reference = 15;
  string user_id = 16;
}

message GetContentRightsRequest {
  string id = 1;
  string tenant_id = 2;
}

message ListContentRightsRequest {
  string tenant_id = 1;
  ContentType content_type = 2;
  ContentRightsStatus status = 3;
  Platform platform = 4;
  string rights_holder = 5;
  int32 page_size = 6;
  string page_token = 7;
}

message ListContentRightsResponse {
  repeated ContentRights rights = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

message CheckRightsAvailabilityRequest {
  string tenant_id = 1;
  string content_id = 2;
  Platform platform = 3;
  string territory = 4;
  string check_date = 5;
}

message CheckRightsAvailabilityResponse {
  bool available = 1;
  repeated ContentRights matching_rights = 2;
  string reason = 3;
}

// ── Royalty Messages ─────────────────────────────────────────────────
message Royalty {
  string id = 1;
  string tenant_id = 2;
  string content_id = 3;
  string rights_holder_id = 4;
  string rights_holder_name = 5;
  string royalty_period = 6;
  RevenueSource revenue_source = 7;
  int64 gross_revenue_cents = 8;
  int64 deductions_cents = 9;
  int64 net_revenue_cents = 10;
  double royalty_rate_pct = 11;
  int64 royalty_amount_cents = 12;
  int64 recoupable_advance_cents = 13;
  int64 recouped_to_date_cents = 14;
  int64 payable_amount_cents = 15;
  string currency_code = 16;
  string payment_reference = 17;
  string paid_at = 18;
  RoyaltyStatus status = 19;
  string created_at = 20;
  string updated_at = 21;
}

message CalculateRoyaltiesRequest {
  string tenant_id = 1;
  string royalty_period = 2;
  repeated string content_ids = 3;
  string user_id = 4;
}

message CalculateRoyaltiesResponse {
  repeated Royalty royalties = 1;
  int32 total_calculated = 2;
  int64 total_payable_cents = 3;
}

message ApproveRoyaltyRequest {
  string id = 1;
  string tenant_id = 2;
  string approved_by = 3;
}

// ── Ad Inventory Messages ────────────────────────────────────────────
message AdInventory {
  string id = 1;
  string tenant_id = 2;
  string inventory_code = 3;
  string content_id = 4;
  PlacementType placement_type = 5;
  Platform platform = 6;
  int32 available_impressions = 7;
  int32 sold_impressions = 8;
  int32 remaining_impressions = 9;
  int64 cpm_rate_cents = 10;
  int64 minimum_commitment_cents = 11;
  string demographic_targeting = 12;   // JSON
  string geo_targeting = 13;           // JSON
  string flight_start = 14;
  string flight_end = 15;
  string advertiser_id = 16;
  string agency_id = 17;
  AdInventoryStatus status = 18;
  AuditInfo audit = 19;
}

message ListAdInventoryRequest {
  string tenant_id = 1;
  AdInventoryStatus status = 2;
  Platform platform = 3;
  PlacementType placement_type = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListAdInventoryResponse {
  repeated AdInventory items = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

message ReserveAdInventoryRequest {
  string id = 1;
  string tenant_id = 2;
  string advertiser_id = 3;
  string agency_id = 4;
  int32 impressions = 5;
  string user_id = 6;
}

message SellAdInventoryRequest {
  string id = 1;
  string tenant_id = 2;
  string advertiser_id = 3;
  string agency_id = 4;
  int32 impressions = 5;
  int64 cpm_rate_cents = 6;
  string user_id = 7;
}

// ── Production Budget Messages ───────────────────────────────────────
message ProductionBudget {
  string id = 1;
  string tenant_id = 2;
  string production_id = 3;
  string production_name = 4;
  ProductionType production_type = 5;
  int64 total_budget_cents = 6;
  int64 above_the_line_cents = 7;
  int64 below_the_line_cents = 8;
  int64 post_production_cents = 9;
  int64 marketing_cents = 10;
  int64 committed_cents = 11;
  int64 spent_cents = 12;
  int64 remaining_cents = 13;
  int64 estimated_to_complete_cents = 14;
  int64 variance_cents = 15;
  int32 shooting_days_total = 16;
  int32 shooting_days_completed = 17;
  string production_start = 18;
  string production_end = 19;
  string producer_id = 20;
  ProductionStatus status = 21;
  AuditInfo audit = 22;
}

message CreateProductionRequest {
  string tenant_id = 1;
  string production_id = 2;
  string production_name = 3;
  ProductionType production_type = 4;
  int64 total_budget_cents = 5;
  int64 above_the_line_cents = 6;
  int64 below_the_line_cents = 7;
  int64 post_production_cents = 8;
  int64 marketing_cents = 9;
  int32 shooting_days_total = 10;
  string production_start = 11;
  string production_end = 12;
  string producer_id = 13;
  string user_id = 14;
}

message GetProductionRequest {
  string id = 1;
  string tenant_id = 2;
}

// ── Audience Analytics Messages ──────────────────────────────────────
message AudienceAnalytics {
  string id = 1;
  string tenant_id = 2;
  string content_id = 3;
  string period = 4;
  string platform = 5;
  int32 total_views = 6;
  int32 unique_viewers = 7;
  double avg_watch_time_minutes = 8;
  double completion_rate_pct = 9;
  string demographic_breakdown = 10;   // JSON
  string geo_breakdown = 11;           // JSON
  int32 social_mentions = 12;
  double sentiment_score = 13;
  double revenue_per_view_cents = 14;
  int32 subscriber_conversions = 15;
  string created_at = 16;
}

message GetContentPerformanceRequest {
  string content_id = 1;
  string tenant_id = 2;
  string period = 3;
  string platform = 4;
}

message GetAudienceTrendsRequest {
  string tenant_id = 1;
  string period_from = 2;
  string period_to = 3;
  string platform = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message GetAudienceTrendsResponse {
  repeated AudienceAnalytics analytics = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}
```

## 6. Migration Order

| Order | Table | Depends On |
|-------|-------|------------|
| 1 | `me_content_rights` | — |
| 2 | `me_royalties` | `me_content_rights` |
| 3 | `me_ad_inventory` | — |
| 4 | `me_production_budgets` | — |
| 5 | `me_audience_analytics` | `me_content_rights` |

---

## 7. Business Rules

1. **Rights Windowing**: Content availability enforced per territorial and platform window rules
2. **Royalty Recoupment**: Advances recouped from royalties before payments issued
3. **Ad Yield Optimization**: Unsold inventory dynamically priced based on demand signals
4. **Budget Traffic Lights**: Green (<5% variance), Yellow (5-10%), Red (>10%) production status
5. **Content Performance**: Underperforming content flagged for promotion or removal decisions
6. **Territorial Compliance**: Content blocked in territories without valid rights

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Contract Lifecycle (83) | Rights and licensing contracts |
| Accounts Payable (07) | Royalty payments |
| Marketing (61) | Content promotion |
| CX Advertising (183) | Ad campaign coordination |
| Financials (06) | Revenue recognition |
| Fusion Data Intelligence (202) | Audience analytics ML |
