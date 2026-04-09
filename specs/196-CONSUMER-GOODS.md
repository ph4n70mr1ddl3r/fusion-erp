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

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.consumer_goods.v1;

service ConsumerGoodsService {
    rpc GetBrand(GetBrandRequest) returns (GetBrandResponse);
    rpc CreateBrand(CreateBrandRequest) returns (CreateBrandResponse);
    rpc GetTradePromotion(GetTradePromotionRequest) returns (GetTradePromotionResponse);
    rpc CreateTradePromotion(CreateTradePromotionRequest) returns (CreateTradePromotionResponse);
    rpc GetDistributor(GetDistributorRequest) returns (GetDistributorResponse);
    rpc GetDemandSensing(GetDemandSensingRequest) returns (GetDemandSensingResponse);
}

// Brand messages
message GetBrandRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetBrandResponse {
    CgBrand data = 1;
}

message CreateBrandRequest {
    string tenant_id = 1;
    string brand_code = 2;
    string brand_name = 3;
    string category = 4;
    string sub_brands = 5;
    string brand_manager_id = 6;
    double market_share_target_pct = 7;
    int64 revenue_target_cents = 8;
}

message CreateBrandResponse {
    CgBrand data = 1;
}

message CgBrand {
    string id = 1;
    string tenant_id = 2;
    string brand_code = 3;
    string brand_name = 4;
    string category = 5;
    string sub_brands = 6;
    string brand_manager_id = 7;
    double market_share_target_pct = 8;
    int64 revenue_target_cents = 9;
    string status = 10;
    string created_at = 11;
    string updated_at = 12;
}

// Trade promotion messages
message GetTradePromotionRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetTradePromotionResponse {
    CgTradePromotion data = 1;
}

message CreateTradePromotionRequest {
    string tenant_id = 1;
    string promotion_code = 2;
    string promotion_name = 3;
    string promotion_type = 4;
    string brand_id = 5;
    string start_date = 6;
    string end_date = 7;
    int64 funding_cents = 8;
    double expected_lift_pct = 9;
    string participating_retailers = 10;
}

message CreateTradePromotionResponse {
    CgTradePromotion data = 1;
}

message CgTradePromotion {
    string id = 1;
    string tenant_id = 2;
    string promotion_code = 3;
    string promotion_name = 4;
    string promotion_type = 5;
    string brand_id = 6;
    string start_date = 7;
    string end_date = 8;
    int64 funding_cents = 9;
    double expected_lift_pct = 10;
    double actual_lift_pct = 11;
    double roi_pct = 12;
    string participating_retailers = 13;
    string status = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Distributor messages
message GetDistributorRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetDistributorResponse {
    CgDistributor data = 1;
}

message CgDistributor {
    string id = 1;
    string tenant_id = 2;
    string distributor_code = 3;
    string distributor_name = 4;
    string territory = 5;
    string coverage_area = 6;
    double performance_score = 7;
    string settlement_terms = 8;
    bool secondary_sales_tracking = 9;
    string active_brands = 10;
    string contact_info = 11;
    string status = 12;
    string created_at = 13;
    string updated_at = 14;
}

// Demand sensing messages
message GetDemandSensingRequest {
    string tenant_id = 1;
    string sku = 2;
    string location = 3;
    string sensed_week = 4;
}

message GetDemandSensingResponse {
    CgDemandSensing data = 1;
}

message CgDemandSensing {
    string id = 1;
    string tenant_id = 2;
    string sku = 3;
    string location = 4;
    string sensed_week = 5;
    double sensed_demand_qty = 6;
    double confidence_score = 7;
    string signals = 8;
    double accuracy_pct = 9;
    string model_version = 10;
    string generated_at = 11;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | cg_brands | -- |
| V002 | cg_trade_promotions | V001 |
| V003 | cg_distributor_management | -- |
| V004 | cg_market_basket_analysis | -- |
| V005 | cg_demand_sensing | -- |

---

## 7. Business Rules

1. **Promotion Budget Control**: Trade promotion spend tracked against brand budget
2. **ROI Measurement**: Promotion ROI calculated from lift vs funding investment
3. **Distributor Scoring**: Performance score based on sell-through, coverage, and payment timeliness
4. **Demand Signal Fusion**: Multiple signals (POS, weather, events) weighted in sensing model
5. **Basket Privacy**: Market basket analysis uses aggregated anonymous transaction data
6. **Promotion Overlap**: System warns when promotions overlap for same brand/retailer

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| retail-service | `GetTransaction` | POS data and retail execution |
| inventory-service | `GetStockLevel` | Stock levels for replenishment |
| order-service | `CreateOrder` | Distributor order processing |
| marketing-service | `GetCampaign` | Campaign and content |
| sales-service | `GetAccount` | Account and opportunity data |
| fdi-service | `RunModel` | AI demand sensing models |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| bi-publisher-service | `GetReportData` | Trade promotion reporting |
