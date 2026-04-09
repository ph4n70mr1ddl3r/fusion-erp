# 196 - Consumer Goods Industry Solution Specification

## 1. Domain Overview

Consumer Goods provides industry-specific capabilities for brand portfolio management, trade promotion management, distributor lifecycle management, market basket analysis, and demand sensing. Supports multi-brand portfolio tracking with market share targets, trade promotion planning with ROI measurement, distributor network management with secondary sales tracking, market basket analysis for cross-sell opportunities, and AI-driven demand sensing using POS, weather, events, and social signals. Enables consumer goods manufacturers to optimize trade spend, improve retail execution, and sense demand shifts in real-time. Integrates with Retail, Inventory, Marketing, and Sales.

**Bounded Context:** Consumer Goods Manufacturing, Trade Promotion & Retail Execution
**Service Name:** `consumer-goods-service`
**Database:** `data/consumer_goods.db`
**HTTP Port:** 8214 | **gRPC Port:** 9214

---

## 2. Database Schema

### 2.1 Brands
```sql
CREATE TABLE cg_brands (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    brand_code TEXT NOT NULL,
    brand_name TEXT NOT NULL,
    category TEXT NOT NULL,
    sub_brands TEXT,                              -- JSON: sub-brand list
    brand_manager_id TEXT NOT NULL,
    market_share_target_pct REAL NOT NULL DEFAULT 0,
    revenue_target_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, brand_code)
);

CREATE INDEX idx_cg_brand_tenant ON cg_brands(tenant_id, status);
CREATE INDEX idx_cg_brand_category ON cg_brands(category);
```

### 2.2 Trade Promotions
```sql
CREATE TABLE cg_trade_promotions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    promotion_code TEXT NOT NULL,
    promotion_name TEXT NOT NULL,
    promotion_type TEXT NOT NULL CHECK(promotion_type IN ('SCAN_BACK','OFF_INVOICE','DISPLAY','COUPON','SAMPLING')),
    brand_id TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    funding_cents INTEGER NOT NULL DEFAULT 0,
    expected_lift_pct REAL NOT NULL DEFAULT 0,
    actual_lift_pct REAL,
    roi_pct REAL,
    participating_retailers TEXT,                 -- JSON: retailer IDs
    terms_and_conditions TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (brand_id) REFERENCES cg_brands(id),
    UNIQUE(tenant_id, promotion_code)
);

CREATE INDEX idx_cg_promo_brand ON cg_trade_promotions(brand_id, status);
CREATE INDEX idx_cg_promo_dates ON cg_trade_promotions(start_date, end_date);
CREATE INDEX idx_cg_promo_status ON cg_trade_promotions(tenant_id, status);
```

### 2.3 Distributor Management
```sql
CREATE TABLE cg_distributor_management (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    distributor_code TEXT NOT NULL,
    distributor_name TEXT NOT NULL,
    territory TEXT,                               -- JSON: territory coverage
    coverage_area TEXT,
    performance_score REAL NOT NULL DEFAULT 0,
    settlement_terms TEXT,
    secondary_sales_tracking INTEGER NOT NULL DEFAULT 0,
    active_brands TEXT,                           -- JSON: brand IDs
    contact_info TEXT,                            -- JSON: contact details
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','SUSPENDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, distributor_code)
);

CREATE INDEX idx_cg_dist_tenant ON cg_distributor_management(tenant_id, status);
CREATE INDEX idx_cg_dist_perf ON cg_distributor_management(performance_score);
```

### 2.4 Market Basket Analysis
```sql
CREATE TABLE cg_market_basket_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    analysis_period TEXT NOT NULL,
    product_combinations TEXT NOT NULL,            -- JSON: [{product_a, product_b, co_occurrence_rate, cross_sell_score}]
    segment_insights TEXT,                        -- JSON: segment-level findings
    total_transactions INTEGER NOT NULL DEFAULT 0,
    avg_basket_size REAL NOT NULL DEFAULT 0,
    generated_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, analysis_period)
);

CREATE INDEX idx_cg_basket_period ON cg_market_basket_analysis(analysis_period DESC);
```

### 2.5 Demand Sensing
```sql
CREATE TABLE cg_demand_sensing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sku TEXT NOT NULL,
    location TEXT NOT NULL,
    sensed_week TEXT NOT NULL,
    sensed_demand_qty REAL NOT NULL DEFAULT 0,
    confidence_score REAL NOT NULL DEFAULT 0,
    signals TEXT,                                 -- JSON: [{signal_type, weight, value}]
    accuracy_pct REAL,
    model_version TEXT,
    generated_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, sku, location, sensed_week)
);

CREATE INDEX idx_cg_demand_sku ON cg_demand_sensing(sku, sensed_week);
CREATE INDEX idx_cg_demand_location ON cg_demand_sensing(location, sensed_week);
```

---

## 3. API Endpoints

### 3.1 Brands
| Method | Path | Description |
|--------|------|-------------|
| POST | `/consumer-goods/v1/brands` | Create brand |
| GET | `/consumer-goods/v1/brands` | List brands |
| GET | `/consumer-goods/v1/brands/{id}` | Get brand |
| PUT | `/consumer-goods/v1/brands/{id}` | Update brand |

### 3.2 Trade Promotions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/consumer-goods/v1/promotions` | Create promotion |
| GET | `/consumer-goods/v1/promotions` | List promotions |
| GET | `/consumer-goods/v1/promotions/{id}` | Get promotion |
| PUT | `/consumer-goods/v1/promotions/{id}` | Update promotion |
| POST | `/consumer-goods/v1/promotions/{id}/activate` | Activate promotion |
| GET | `/consumer-goods/v1/promotions/{id}/roi` | Get promotion ROI |

### 3.3 Distributors
| Method | Path | Description |
|--------|------|-------------|
| POST | `/consumer-goods/v1/distributors` | Add distributor |
| GET | `/consumer-goods/v1/distributors` | List distributors |
| GET | `/consumer-goods/v1/distributors/{id}` | Get distributor |
| PUT | `/consumer-goods/v1/distributors/{id}` | Update distributor |

### 3.4 Market Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/consumer-goods/v1/market-basket` | Market basket analysis |
| GET | `/consumer-goods/v1/market-basket/{period}` | Period analysis |
| POST | `/consumer-goods/v1/market-basket/refresh` | Refresh analysis |

### 3.5 Demand Sensing
| Method | Path | Description |
|--------|------|-------------|
| GET | `/consumer-goods/v1/demand-sensing` | Demand signals |
| GET | `/consumer-goods/v1/demand-sensing/{sku}` | SKU demand |
| POST | `/consumer-goods/v1/demand-sensing/refresh` | Refresh sensing model |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `cgoods.promotion.launched` | `{ promotion_id, brand_id, funding }` | Promotion activated |
| `cgoods.distributor.order.placed` | `{ distributor_id, order_id, amount }` | Distributor order |
| `cgoods.demand.signal.detected` | `{ sku, location, signal_type, delta }` | Demand shift detected |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `retail.pos.transaction` | Retail (192) | Feed demand sensing |
| `shipment.delivered` | Shipping (145) | Update distributor delivery |
| `campaign.launched` | Marketing (61) | Link to trade promotion |

---

## 5. Business Rules

1. **Promotion Budget Control**: Trade promotion spend tracked against brand budget
2. **ROI Measurement**: Promotion ROI calculated from lift vs funding investment
3. **Distributor Scoring**: Performance score based on sell-through, coverage, and payment timeliness
4. **Demand Signal Fusion**: Multiple signals (POS, weather, events) weighted in sensing model
5. **Basket Privacy**: Market basket analysis uses aggregated anonymous transaction data
6. **Promotion Overlap**: System warns when promotions overlap for same brand/retailer

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Retail (192) | POS data and retail execution |
| Inventory (34) | Stock levels for replenishment |
| Order Management (32) | Distributor order processing |
| Marketing (61) | Campaign and content |
| Sales Automation (77) | Account and opportunity data |
| BI Publisher (150) | Trade promotion reporting |
| Fusion Data Intelligence (202) | AI demand sensing models |
