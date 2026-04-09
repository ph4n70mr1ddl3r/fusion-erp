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
| POST | `/media/v1/rights` | Register content rights |
| GET | `/media/v1/rights` | List rights |
| GET | `/media/v1/rights/{id}` | Get rights |
| GET | `/media/v1/rights/availability` | Check rights availability |

### 3.2 Royalties
| Method | Path | Description |
|--------|------|-------------|
| POST | `/media/v1/royalties/calculate` | Calculate royalties |
| GET | `/media/v1/royalties` | List royalties |
| POST | `/media/v1/royalties/{id}/approve` | Approve royalty |
| GET | `/media/v1/royalties/statements` | Royalty statements |

### 3.3 Advertising
| Method | Path | Description |
|--------|------|-------------|
| GET | `/media/v1/ad-inventory` | List ad inventory |
| POST | `/media/v1/ad-inventory/{id}/reserve` | Reserve inventory |
| POST | `/media/v1/ad-inventory/{id}/sell` | Sell inventory |
| GET | `/media/v1/ad-inventory/yield` | Yield report |

### 3.4 Production Budgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/media/v1/productions` | Create production |
| GET | `/media/v1/productions` | List productions |
| GET | `/media/v1/productions/{id}` | Get production |
| POST | `/media/v1/productions/{id}/cost-report` | Submit cost report |

### 3.5 Audience Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/media/v1/audience` | Audience dashboard |
| GET | `/media/v1/audience/{contentId}` | Content performance |
| GET | `/media/v1/audience/trends` | Audience trends |

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

## 5. Business Rules

1. **Rights Windowing**: Content availability enforced per territorial and platform window rules
2. **Royalty Recoupment**: Advances recouped from royalties before payments issued
3. **Ad Yield Optimization**: Unsold inventory dynamically priced based on demand signals
4. **Budget Traffic Lights**: Green (<5% variance), Yellow (5-10%), Red (>10%) production status
5. **Content Performance**: Underperforming content flagged for promotion or removal decisions
6. **Territorial Compliance**: Content blocked in territories without valid rights

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Contract Lifecycle (83) | Rights and licensing contracts |
| Accounts Payable (07) | Royalty payments |
| Marketing (61) | Content promotion |
| CX Advertising (183) | Ad campaign coordination |
| Financials (06) | Revenue recognition |
| Fusion Data Intelligence (202) | Audience analytics ML |
