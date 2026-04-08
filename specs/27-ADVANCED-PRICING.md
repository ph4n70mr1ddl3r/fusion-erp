# 27 - Advanced Pricing Service Specification

## 1. Domain Overview

Advanced Pricing manages price lists, tiered/quantity-break pricing, discount schemes, promotional offers, surcharge rules, attribute-based pricing, cost-plus markup, customer/supplier price agreements, and multi-currency price conversion. The pricing engine resolves the final selling or purchase price through a configurable cascade. Integrates with OM for sales order pricing, Procurement for purchase pricing, and AR for invoice pricing.

**Bounded Context:** Advanced Pricing & Price Management
**Service Name:** `pricing-service`
**Database:** `data/pricing.db`
**HTTP Port:** 8026 | **gRPC Port:** 9026

---

## 2. Database Schema

### 2.1 Price Lists
```sql
CREATE TABLE price_lists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    price_list_number TEXT NOT NULL,        -- PL-2024-00001
    name TEXT NOT NULL,
    description TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    price_list_type TEXT NOT NULL DEFAULT 'SALE'
        CHECK(price_list_type IN ('SALE','PURCHASE')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','EXPIRED')),
    rounding_rule TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(rounding_rule IN ('STANDARD','ROUND_UP','ROUND_DOWN','ROUND_HALF_UP','ROUND_HALF_DOWN')),
    decimal_precision INTEGER NOT NULL DEFAULT 2,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, price_list_number)
);

CREATE INDEX idx_price_lists_tenant_type ON price_lists(tenant_id, price_list_type);
CREATE INDEX idx_price_lists_tenant_status ON price_lists(tenant_id, status);
CREATE INDEX idx_price_lists_tenant_dates ON price_lists(tenant_id, effective_from, effective_to);
```

### 2.2 Price List Lines
```sql
CREATE TABLE price_list_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    price_list_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    item_description TEXT,
    unit_of_measure TEXT NOT NULL,
    list_price_cents INTEGER NOT NULL,
    unit_cost_cents INTEGER,               -- Reference cost for margin calculations
    minimum_quantity DECIMAL(18,4) NOT NULL DEFAULT 1,
    maximum_quantity DECIMAL(18,4),
    pricing_unit TEXT DEFAULT 'EACH'
        CHECK(pricing_unit IN ('EACH','DOZEN','CASE','PALLET')),
    pricing_unit_factor REAL NOT NULL DEFAULT 1,
    effective_from TEXT,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (price_list_id) REFERENCES price_lists(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, price_list_id, item_id, unit_of_measure)
);

CREATE INDEX idx_price_list_lines_tenant_pl ON price_list_lines(tenant_id, price_list_id);
CREATE INDEX idx_price_list_lines_tenant_item ON price_list_lines(tenant_id, item_id);
```

### 2.3 Price Breaks (Quantity Tiers)
```sql
CREATE TABLE price_breaks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    price_list_line_id TEXT NOT NULL,
    break_type TEXT NOT NULL DEFAULT 'QUANTITY'
        CHECK(break_type IN ('QUANTITY','AMOUNT')),
    from_quantity DECIMAL(18,4) NOT NULL,
    to_quantity DECIMAL(18,4),
    break_price_cents INTEGER NOT NULL,
    discount_percent REAL,
    discount_amount_cents INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (price_list_line_id) REFERENCES price_list_lines(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, price_list_line_id, from_quantity)
);

CREATE INDEX idx_price_breaks_tenant_line ON price_breaks(tenant_id, price_list_line_id);
```

### 2.4 Pricing Strategies
```sql
CREATE TABLE pricing_strategies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    strategy_number TEXT NOT NULL,          -- PS-2024-00001
    name TEXT NOT NULL,
    description TEXT,
    strategy_type TEXT NOT NULL DEFAULT 'SALES'
        CHECK(strategy_type IN ('SALES','PURCHASE','BOTH')),
    applicable_context TEXT NOT NULL DEFAULT 'ALL'
        CHECK(applicable_context IN ('ALL','CUSTOMER_GROUP','ITEM_CATEGORY','REGION','CHANNEL')),
    priority INTEGER NOT NULL DEFAULT 10,  -- Lower number = higher priority
    effective_from TEXT,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, strategy_number)
);

CREATE INDEX idx_pricing_strategies_tenant_type ON pricing_strategies(tenant_id, strategy_type);
CREATE INDEX idx_pricing_strategies_tenant_status ON pricing_strategies(tenant_id, status);
CREATE INDEX idx_pricing_strategies_tenant_priority ON pricing_strategies(tenant_id, priority);
```

### 2.5 Pricing Rules
```sql
CREATE TABLE pricing_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    strategy_id TEXT NOT NULL,
    rule_number INTEGER NOT NULL,           -- Sequence within strategy
    name TEXT NOT NULL,
    description TEXT,
    rule_type TEXT NOT NULL DEFAULT 'DISCOUNT'
        CHECK(rule_type IN ('DISCOUNT','SURCHARGE','PRICE_OVERRIDE','FREIGHT')),
    condition_operator TEXT NOT NULL DEFAULT 'AND'
        CHECK(condition_operator IN ('AND','OR')),

    -- Condition (JSON): { "field": "item.category_id", "operator": "IN", "values": ["cat1","cat2"] }
    conditions TEXT NOT NULL,               -- JSON array of condition objects

    -- Action (JSON): { "type": "PERCENT_DISCOUNT", "value": 10 } or { "type": "FIXED_PRICE", "value_cents": 5000 }
    action TEXT NOT NULL,                   -- JSON action object

    -- Applicability
    applies_to TEXT DEFAULT 'LINE'
        CHECK(applies_to IN ('LINE','ORDER','GROUP')),
    stacking_mode TEXT DEFAULT 'EXCLUSIVE'
        CHECK(stacking_mode IN ('EXCLUSIVE','STACKABLE','BEST_OF')),

    priority INTEGER NOT NULL DEFAULT 10,
    effective_from TEXT,
    effective_to TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (strategy_id) REFERENCES pricing_strategies(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, strategy_id, rule_number)
);

CREATE INDEX idx_pricing_rules_tenant_strategy ON pricing_rules(tenant_id, strategy_id);
CREATE INDEX idx_pricing_rules_tenant_type ON pricing_rules(tenant_id, rule_type);
```

### 2.6 Discount Schemes
```sql
CREATE TABLE discount_schemes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    discount_code TEXT NOT NULL,            -- DS-2024-00001
    name TEXT NOT NULL,
    description TEXT,
    discount_type TEXT NOT NULL DEFAULT 'PERCENTAGE'
        CHECK(discount_type IN ('PERCENTAGE','FIXED_AMOUNT','BUY_X_GET_Y','BUNDLE')),
    value REAL NOT NULL,                   -- Percentage or fixed amount
    value_cents INTEGER,                   -- For FIXED_AMOUNT type
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Buy X Get Y specifics
    buy_quantity DECIMAL(18,4),
    get_quantity DECIMAL(18,4),
    get_discount_percent REAL,

    -- Bundle specifics
    bundle_item_ids TEXT,                  -- JSON array of item IDs for bundle discount

    -- Applicability
    applies_to_item_id TEXT,
    applies_to_category_id TEXT,
    applies_to_customer_group TEXT,
    applies_to_region TEXT,
    minimum_order_amount_cents INTEGER,
    minimum_quantity DECIMAL(18,4),

    -- Validity
    effective_from TEXT,
    effective_to TEXT,
    max_uses INTEGER,                      -- NULL = unlimited
    current_uses INTEGER NOT NULL DEFAULT 0,
    max_uses_per_customer INTEGER,

    stacking_allowed INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','EXPIRED','EXHAUSTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, discount_code)
);

CREATE INDEX idx_discount_schemes_tenant_type ON discount_schemes(tenant_id, discount_type);
CREATE INDEX idx_discount_schemes_tenant_status ON discount_schemes(tenant_id, status);
CREATE INDEX idx_discount_schemes_tenant_dates ON discount_schemes(tenant_id, effective_from, effective_to);
```

### 2.7 Promotional Pricing
```sql
CREATE TABLE promotional_pricing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    promo_code TEXT NOT NULL,              -- PROMO-2024-00001
    name TEXT NOT NULL,
    description TEXT,
    promo_type TEXT NOT NULL DEFAULT 'COUPON'
        CHECK(promo_type IN ('COUPON','FLASH_SALE','SEASONAL','CLEARANCE','LOYALTY')),
    coupon_code TEXT,                      -- User-entered code (e.g., SUMMER2024)

    -- Offer
    offer_type TEXT NOT NULL DEFAULT 'PERCENT_OFF'
        CHECK(offer_type IN ('PERCENT_OFF','AMOUNT_OFF','FIXED_PRICE','FREE_SHIPPING','BUY_X_GET_Y','BUNDLE')),
    offer_value REAL,                      -- Percentage or multiplier
    offer_amount_cents INTEGER,            -- Fixed amount off or fixed price
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Buy X Get Y
    buy_quantity DECIMAL(18,4),
    get_quantity DECIMAL(18,4),
    get_discount_percent REAL,

    -- Conditions
    minimum_order_amount_cents INTEGER,
    minimum_quantity DECIMAL(18,4),
    maximum_discount_cents INTEGER,

    -- Targeting
    customer_group TEXT,
    region TEXT,
    channel TEXT DEFAULT 'ALL'
        CHECK(channel IN ('ALL','ONLINE','RETAIL','WHOLESALE','EDI')),
    item_ids TEXT,                         -- JSON array: specific items
    category_ids TEXT,                     -- JSON array: specific categories

    -- Validity
    effective_from TEXT NOT NULL,
    effective_to TEXT NOT NULL,
    usage_limit INTEGER,                   -- Total uses allowed
    current_usage INTEGER NOT NULL DEFAULT 0,
    per_customer_limit INTEGER,
    is_stackable INTEGER NOT NULL DEFAULT 0,

    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','EXPIRED','EXHAUSTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, promo_code)
);

CREATE INDEX idx_promo_tenant_type ON promotional_pricing(tenant_id, promo_type);
CREATE INDEX idx_promo_tenant_status ON promotional_pricing(tenant_id, status);
CREATE INDEX idx_promo_tenant_coupon ON promotional_pricing(tenant_id, coupon_code);
CREATE INDEX idx_promo_tenant_dates ON promotional_pricing(tenant_id, effective_from, effective_to);
```

### 2.8 Surcharge Rules
```sql
CREATE TABLE surcharge_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    surcharge_code TEXT NOT NULL,          -- SUR-2024-00001
    name TEXT NOT NULL,
    description TEXT,
    surcharge_type TEXT NOT NULL DEFAULT 'FREIGHT'
        CHECK(surcharge_type IN ('FREIGHT','HANDLING','SMALL_ORDER','FUEL','EXPEDITE','MISC')),
    calculation_method TEXT NOT NULL DEFAULT 'FIXED'
        CHECK(calculation_method IN ('FIXED','PERCENTAGE','PER_UNIT','TIERED','FORMULA')),
    fixed_amount_cents INTEGER,
    percentage_rate REAL,
    per_unit_amount_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Tiered surcharge (JSON): [{ "from": 0, "to": 10000, "amount_cents": 500 }, ...]
    tiers TEXT,

    -- Conditions
    minimum_order_amount_cents INTEGER,
    maximum_order_amount_cents INTEGER,
    weight_kg_min REAL,
    weight_kg_max REAL,
    region TEXT,
    ship_method TEXT,

    effective_from TEXT,
    effective_to TEXT,
    is_mandatory INTEGER NOT NULL DEFAULT 0,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, surcharge_code)
);

CREATE INDEX idx_surcharge_rules_tenant_type ON surcharge_rules(tenant_id, surcharge_type);
CREATE INDEX idx_surcharge_rules_tenant_status ON surcharge_rules(tenant_id, status);
```

### 2.9 Attribute Pricing
```sql
CREATE TABLE attribute_pricing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    attribute_context TEXT NOT NULL DEFAULT 'CUSTOMER'
        CHECK(attribute_context IN ('CUSTOMER','ITEM','ORDER','REGION')),

    -- Attribute match (JSON): { "field": "customer_group", "operator": "EQ", "value": "VIP" }
    attribute_condition TEXT NOT NULL,      -- JSON condition object

    adjustment_type TEXT NOT NULL DEFAULT 'PERCENTAGE'
        CHECK(adjustment_type IN ('PERCENTAGE','FIXED_AMOUNT','PRICE_OVERRIDE')),
    adjustment_value REAL,                 -- For percentage
    adjustment_amount_cents INTEGER,       -- For fixed amount or override
    currency_code TEXT NOT NULL DEFAULT 'USD',

    priority INTEGER NOT NULL DEFAULT 10,
    effective_from TEXT,
    effective_to TEXT,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, name)
);

CREATE INDEX idx_attribute_pricing_tenant_context ON attribute_pricing(tenant_id, attribute_context);
CREATE INDEX idx_attribute_pricing_tenant_status ON attribute_pricing(tenant_id, status);
```

### 2.10 Cost Plus Rules
```sql
CREATE TABLE cost_plus_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    markup_type TEXT NOT NULL DEFAULT 'PERCENT_MARGIN'
        CHECK(markup_type IN ('PERCENT_MARGIN','PERCENT_MARKUP','FIXED_MARKUP','TIERED_MARKUP')),
    markup_value REAL NOT NULL,            -- Percentage or multiplier
    fixed_markup_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Tiered markup (JSON): [{ "from_cost": 0, "to_cost": 10000, "markup_percent": 50 }, ...]
    tiers TEXT,

    -- Applicability
    applies_to_category_id TEXT,
    applies_to_item_id TEXT,
    minimum_cost_cents INTEGER,

    -- Rounding
    round_to_nearest_cents INTEGER NOT NULL DEFAULT 1,  -- Round result to nearest N cents

    effective_from TEXT,
    effective_to TEXT,
    priority INTEGER NOT NULL DEFAULT 10,

    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, name)
);

CREATE INDEX idx_cost_plus_rules_tenant_status ON cost_plus_rules(tenant_id, status);
```

### 2.11 Price Agreements
```sql
CREATE TABLE price_agreements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_number TEXT NOT NULL,         -- PA-2024-00001
    name TEXT NOT NULL,
    description TEXT,
    agreement_type TEXT NOT NULL DEFAULT 'CUSTOMER'
        CHECK(agreement_type IN ('CUSTOMER','SUPPLIER')),
    party_id TEXT NOT NULL,                 -- Customer ID or Supplier ID
    price_list_id TEXT,                     -- Base price list reference

    effective_from TEXT NOT NULL,
    effective_to TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','EXPIRED','CANCELLED')),

    -- Volume commitments
    minimum_volume_cents INTEGER,
    committed_volume_cents INTEGER NOT NULL DEFAULT 0,
    consumed_volume_cents INTEGER NOT NULL DEFAULT 0,

    -- Approval
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, agreement_number)
);

CREATE INDEX idx_price_agreements_tenant_type ON price_agreements(tenant_id, agreement_type);
CREATE INDEX idx_price_agreements_tenant_party ON price_agreements(tenant_id, party_id);
CREATE INDEX idx_price_agreements_tenant_status ON price_agreements(tenant_id, status);
CREATE INDEX idx_price_agreements_tenant_dates ON price_agreements(tenant_id, effective_from, effective_to);

CREATE TABLE price_agreement_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    price_agreement_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    item_description TEXT,
    unit_of_measure TEXT NOT NULL,
    negotiated_price_cents INTEGER NOT NULL,
    list_price_cents INTEGER,              -- Reference list price for comparison
    minimum_quantity DECIMAL(18,4) NOT NULL DEFAULT 1,
    maximum_quantity DECIMAL(18,4),
    discount_percent REAL,                 -- Or percentage off list price

    effective_from TEXT,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (price_agreement_id) REFERENCES price_agreements(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, price_agreement_id, item_id, unit_of_measure)
);

CREATE INDEX idx_pa_lines_tenant_agreement ON price_agreement_lines(tenant_id, price_agreement_id);
CREATE INDEX idx_pa_lines_tenant_item ON price_agreement_lines(tenant_id, item_id);
```

### 2.12 Currency Conversion Rules
```sql
CREATE TABLE currency_conversion_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    from_currency TEXT NOT NULL,
    to_currency TEXT NOT NULL,
    exchange_rate REAL NOT NULL,
    rate_type TEXT NOT NULL DEFAULT 'SPOT'
        CHECK(rate_type IN ('SPOT','FORWARD','AVERAGE','FIXED','USER_DEFINED')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    markup_percent REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, from_currency, to_currency, rate_type, effective_from)
);

CREATE INDEX idx_currency_conv_tenant_pair ON currency_conversion_rules(tenant_id, from_currency, to_currency);
CREATE INDEX idx_currency_conv_tenant_dates ON currency_conversion_rules(tenant_id, effective_from, effective_to);
```

### 2.13 Price Override Approvals
```sql
CREATE TABLE price_overrides (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    override_number TEXT NOT NULL,         -- POV-2024-00001
    source_type TEXT NOT NULL DEFAULT 'SALES_ORDER'
        CHECK(source_type IN ('SALES_ORDER','PURCHASE_ORDER','QUOTATION','MANUAL')),
    source_id TEXT,
    source_line_id TEXT,
    item_id TEXT NOT NULL,

    original_price_cents INTEGER NOT NULL,
    override_price_cents INTEGER NOT NULL,
    override_reason TEXT NOT NULL,
    discount_percent REAL,
    discount_amount_cents INTEGER,

    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','EXPIRED')),
    approved_by TEXT,
    approved_at TEXT,
    rejection_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, override_number)
);

CREATE INDEX idx_price_overrides_tenant_status ON price_overrides(tenant_id, status);
CREATE INDEX idx_price_overrides_tenant_source ON price_overrides(tenant_id, source_type, source_id);
```

### 2.14 Pricing Number Sequences
```sql
CREATE TABLE pricing_sequences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sequence_type TEXT NOT NULL,           -- 'PRICE_LIST', 'STRATEGY', 'DISCOUNT', 'PROMO', 'AGREEMENT', 'OVERRIDE'
    prefix TEXT NOT NULL,
    current_value INTEGER NOT NULL DEFAULT 0,
    fiscal_year INTEGER NOT NULL,
    UNIQUE(tenant_id, sequence_type, fiscal_year)
);
```

---

## 3. REST API Endpoints

### 3.1 Price Lists
```
GET/POST      /api/v1/pricing/price-lists                    Permission: pricing.price-lists.read/create
GET/PUT       /api/v1/pricing/price-lists/{id}                Permission: pricing.price-lists.read/update
DELETE        /api/v1/pricing/price-lists/{id}                Permission: pricing.price-lists.delete
POST          /api/v1/pricing/price-lists/{id}/activate       Permission: pricing.price-lists.update
POST          /api/v1/pricing/price-lists/{id}/deactivate     Permission: pricing.price-lists.update
GET/POST      /api/v1/pricing/price-lists/{id}/lines          Permission: pricing.price-lists.read/create
PUT/DELETE    /api/v1/pricing/price-lists/{id}/lines/{lid}    Permission: pricing.price-lists.update
POST          /api/v1/pricing/price-lists/{id}/copy           Permission: pricing.price-lists.create
```

### 3.2 Price Breaks
```
GET/POST      /api/v1/pricing/price-list-lines/{id}/breaks    Permission: pricing.price-lists.read/create
PUT/DELETE    /api/v1/pricing/price-breaks/{id}               Permission: pricing.price-lists.update
```

### 3.3 Pricing Strategies & Rules
```
GET/POST      /api/v1/pricing/strategies                      Permission: pricing.strategies.read/create
GET/PUT       /api/v1/pricing/strategies/{id}                  Permission: pricing.strategies.read/update
DELETE        /api/v1/pricing/strategies/{id}                  Permission: pricing.strategies.delete
POST          /api/v1/pricing/strategies/{id}/activate         Permission: pricing.strategies.update
GET/POST      /api/v1/pricing/strategies/{id}/rules            Permission: pricing.strategies.read/create
PUT/DELETE    /api/v1/pricing/strategies/{id}/rules/{rid}      Permission: pricing.strategies.update
```

### 3.4 Discount Schemes
```
GET/POST      /api/v1/pricing/discounts                        Permission: pricing.discounts.read/create
GET/PUT       /api/v1/pricing/discounts/{id}                    Permission: pricing.discounts.read/update
DELETE        /api/v1/pricing/discounts/{id}                    Permission: pricing.discounts.delete
POST          /api/v1/pricing/discounts/{id}/activate           Permission: pricing.discounts.update
POST          /api/v1/pricing/discounts/{id}/deactivate         Permission: pricing.discounts.update
```

### 3.5 Promotional Pricing
```
GET/POST      /api/v1/pricing/promotions                       Permission: pricing.promotions.read/create
GET/PUT       /api/v1/pricing/promotions/{id}                   Permission: pricing.promotions.read/update
DELETE        /api/v1/pricing/promotions/{id}                   Permission: pricing.promotions.delete
POST          /api/v1/pricing/promotions/{id}/activate          Permission: pricing.promotions.update
POST          /api/v1/pricing/promotions/{id}/deactivate        Permission: pricing.promotions.update
GET           /api/v1/pricing/promotions/validate-coupon        Permission: pricing.promotions.read
POST          /api/v1/pricing/promotions/{id}/redeem            Permission: pricing.promotions.update
```

### 3.6 Surcharge Rules
```
GET/POST      /api/v1/pricing/surcharges                       Permission: pricing.surcharges.read/create
GET/PUT       /api/v1/pricing/surcharges/{id}                   Permission: pricing.surcharges.read/update
DELETE        /api/v1/pricing/surcharges/{id}                   Permission: pricing.surcharges.delete
```

### 3.7 Attribute Pricing
```
GET/POST      /api/v1/pricing/attributes                        Permission: pricing.attributes.read/create
GET/PUT       /api/v1/pricing/attributes/{id}                    Permission: pricing.attributes.read/update
DELETE        /api/v1/pricing/attributes/{id}                    Permission: pricing.attributes.delete
```

### 3.8 Cost Plus Rules
```
GET/POST      /api/v1/pricing/cost-plus                         Permission: pricing.cost-plus.read/create
GET/PUT       /api/v1/pricing/cost-plus/{id}                     Permission: pricing.cost-plus.read/update
DELETE        /api/v1/pricing/cost-plus/{id}                     Permission: pricing.cost-plus.delete
```

### 3.9 Price Agreements
```
GET/POST      /api/v1/pricing/agreements                        Permission: pricing.agreements.read/create
GET/PUT       /api/v1/pricing/agreements/{id}                    Permission: pricing.agreements.read/update
DELETE        /api/v1/pricing/agreements/{id}                    Permission: pricing.agreements.delete
POST          /api/v1/pricing/agreements/{id}/activate           Permission: pricing.agreements.update
POST          /api/v1/pricing/agreements/{id}/cancel             Permission: pricing.agreements.update
GET/POST      /api/v1/pricing/agreements/{id}/lines              Permission: pricing.agreements.read/create
PUT/DELETE    /api/v1/pricing/agreements/{id}/lines/{lid}        Permission: pricing.agreements.update
```

### 3.10 Currency Conversion
```
GET/POST      /api/v1/pricing/currency-rates                    Permission: pricing.currency.read/create
GET/PUT       /api/v1/pricing/currency-rates/{id}                Permission: pricing.currency.read/update
DELETE        /api/v1/pricing/currency-rates/{id}                Permission: pricing.currency.delete
POST          /api/v1/pricing/currency-rates/convert             Permission: pricing.currency.read
```

### 3.11 Price Override Approvals
```
GET/POST      /api/v1/pricing/overrides                          Permission: pricing.overrides.read/create
GET           /api/v1/pricing/overrides/{id}                     Permission: pricing.overrides.read
POST          /api/v1/pricing/overrides/{id}/approve             Permission: pricing.overrides.approve
POST          /api/v1/pricing/overrides/{id}/reject              Permission: pricing.overrides.approve
```

### 3.12 Price Engine (Actions)
```
POST          /api/v1/pricing/engine/calculate                   Permission: pricing.engine.calculate
POST          /api/v1/pricing/engine/calculate-batch             Permission: pricing.engine.calculate
GET           /api/v1/pricing/engine/explain                     Permission: pricing.engine.read
```

### 3.13 Reports
```
GET           /api/v1/pricing/reports/price-variance             Permission: pricing.reports.view
GET           /api/v1/pricing/reports/discount-analysis          Permission: pricing.reports.view
GET           /api/v1/pricing/reports/margin-analysis            Permission: pricing.reports.view
GET           /api/v1/pricing/reports/promotion-effectiveness    Permission: pricing.reports.view
GET           /api/v1/pricing/reports/agreement-utilization      Permission: pricing.reports.view
```

---

## 4. Business Rules

### 4.1 Price Determination Engine (Cascade)

The pricing engine resolves the final price for an item using the following precedence cascade. The first applicable source wins; lower-priority sources are evaluated only if higher-priority sources do not apply.

```
1. Price Agreement (customer/supplier specific, highest priority)
2. Promotional Pricing (active promotions with matching conditions)
3. Discount Schemes (applicable discounts evaluated, best of stackable applied)
4. Attribute Pricing (customer group / item category / region adjustments)
5. Price List (base price from applicable price list with quantity breaks)
6. List Price (item.list_price_cents from Inventory service)
7. Cost-Plus Rule (markup over cost as fallback)
```

For each pricing request the engine:
1. Identifies the context (customer, item, quantity, date, region, channel)
2. Evaluates each level in cascade order
3. Applies quantity/price breaks if applicable
4. Stacks eligible discounts per stacking rules
5. Adds applicable surcharges
6. Converts currency if needed
7. Applies rounding rules
8. Returns the net price with a full breakdown explanation

### 4.2 Tiered/Quantity Break Pricing
- Price breaks defined on price_list_lines via the price_breaks table
- Break ranges are half-open intervals: `from_quantity <= quantity < to_quantity`
- The `to_quantity` of the final tier may be NULL (unlimited)
- When quantity falls within a break range, the break_price_cents replaces the list_price_cents
- If both break_price_cents and discount_percent are provided, discount_percent is applied to the base list price

### 4.3 Volume Discount Aggregation
- Multiple discounts MAY be combined when `stacking_mode = 'STACKABLE'`
- When `stacking_mode = 'BEST_OF'`, only the highest-value discount is applied
- When `stacking_mode = 'EXCLUSIVE'`, only the first matching rule in priority order is applied
- Maximum discount cap enforced: `maximum_discount_cents` on discount schemes and promotions
- Volume is tracked per agreement: `consumed_volume_cents += net_price * quantity`

### 4.4 Promotional Pricing with Validity Periods
- Promotion MUST have `effective_from` and `effective_to`; pricing outside this range is rejected
- Coupon codes are case-insensitive and MUST match exactly
- Usage tracking: `current_usage` incremented on each redemption
- If `usage_limit` is set and `current_usage >= usage_limit`, status transitions to EXHAUSTED
- Per-customer limits tracked via event sourcing against `per_customer_limit`
- Expired promotions (effective_to < today) are excluded from the cascade automatically

### 4.5 Attribute-Based Pricing
- Conditions evaluated against customer group, item category, region, and order attributes
- Attribute condition JSON uses operators: EQ, NEQ, IN, NOT_IN, GT, LT, GTE, LTE, BETWEEN
- Matching attribute pricing rules are applied as adjustments to the base price:
  - `PERCENTAGE`: `base_price * (1 + adjustment_value / 100)`
  - `FIXED_AMOUNT`: `base_price + adjustment_amount_cents`
  - `PRICE_OVERRIDE`: directly set to `adjustment_amount_cents`
- Multiple attribute rules evaluated by priority; EXCLUSIVE mode applies only the highest-priority match

### 4.6 Cost-Plus Pricing
Used as fallback when no other pricing source applies:
- **PERCENT_MARGIN:** `price = cost / (1 - markup_value / 100)`
- **PERCENT_MARKUP:** `price = cost * (1 + markup_value / 100)`
- **FIXED_MARKUP:** `price = cost + fixed_markup_cents`
- **TIERED_MARKUP:** look up the applicable tier by cost range and apply the tier's markup percent
- Cost retrieved from Inventory service via gRPC (average_cost_cents or last_purchase_cost_cents)
- Result rounded to `round_to_nearest_cents`

### 4.7 Competitive Price Matching
- Enabled via a pricing rule of type `PRICE_OVERRIDE` with condition matching a competitor price field
- If `competitive_price_cents` is provided in the pricing request, the engine MAY apply a match
- Match policy configured per strategy: `MATCH_EXACTLY`, `MATCH_BEAT_BY_PERCENT`, `MATCH_BEAT_BY_AMOUNT`
- Requires approval workflow if the matched price is below cost

### 4.8 Price Override with Approval Workflow
- When the calculated price is manually overridden, a price_overrides record is created
- Override threshold: if `discount_percent > tenant_config.override_threshold_percent`, approval is required
- Approval flow via Workflow service (gRPC call)
- Override is item-level and source-specific (tied to an order or quotation)
- Approved overrides are cached for the duration of the source transaction

### 4.9 Multi-Currency Price Conversion
- Each price list has a base `currency_code`
- When the pricing request specifies a different currency:
  1. Look up the applicable conversion rule by date and currency pair
  2. Apply the exchange rate: `converted_amount = original_amount * exchange_rate`
  3. Apply markup_percent if set: `converted_amount *= (1 + markup_percent / 100)`
  4. Apply the target currency's rounding rules
- If no conversion rule exists for the pair, the pricing engine returns an error

### 4.10 Rounding Rules per Currency
- Rounding rules are defined at the price list level
- `STANDARD`: standard banker's rounding (round half to even)
- `ROUND_UP`: always round up to nearest `decimal_precision` decimal
- `ROUND_DOWN`: always round down (truncate)
- `ROUND_HALF_UP`: round half away from zero
- `ROUND_HALF_DOWN`: round half toward zero
- Rounding is applied as the final step after all calculations

### 4.11 Net Price Calculation
```
net_price = base_price
          - line_discount_amount
          - order_discount_amount (allocated per line)
          + surcharge_amount
```
Where:
- `base_price` is determined from the cascade (agreement > promo > price list > list price > cost-plus)
- `line_discount_amount = base_price * (SUM(applicable_line_discounts_percent) / 100)`
- `order_discount_amount` is pro-rated across lines by line amount
- `surcharge_amount` is the sum of all applicable surcharge rules

### 4.12 Effective Date Management
- All pricing entities support `effective_from` and `effective_to` dates
- Scheduled price changes: create new price list lines with future `effective_from`
- At query time, the engine selects the price record where `effective_from <= today AND (effective_to IS NULL OR effective_to >= today)`
- Expired records are NOT deleted; they remain for historical reference and audit
- Price list status auto-transitions: ACTIVE to EXPIRED when `effective_to < today`

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.pricing.v1;

service PricingService {
    // Price calculation
    rpc CalculatePrice(CalculatePriceRequest) returns (CalculatePriceResponse);
    rpc CalculatePriceBatch(CalculatePriceBatchRequest) returns (CalculatePriceBatchResponse);
    rpc ExplainPrice(ExplainPriceRequest) returns (ExplainPriceResponse);

    // Price list lookups
    rpc GetPriceList(GetPriceListRequest) returns (GetPriceListResponse);
    rpc GetItemPrice(GetItemPriceRequest) returns (GetItemPriceResponse);

    // Price agreements
    rpc GetPriceAgreement(GetPriceAgreementRequest) returns (GetPriceAgreementResponse);

    // Currency conversion
    rpc ConvertCurrency(ConvertCurrencyRequest) returns (ConvertCurrencyResponse);

    // Discount / promo validation
    rpc ValidateCoupon(ValidateCouponRequest) returns (ValidateCouponResponse);
}

message PriceLineItem {
    string item_id = 1;
    string unit_of_measure = 2;
    int32 quantity = 3;
    int32 unit_cost_cents = 4;          // Optional: from inventory
    string category_id = 5;
    map<string, string> attributes = 6; // Additional item attributes
}

message PricingContext {
    string tenant_id = 1;
    string customer_id = 2;
    string customer_group = 3;
    string supplier_id = 4;
    string price_list_id = 5;
    string region = 6;
    string channel = 7;
    string currency_code = 8;
    string pricing_date = 9;            // ISO 8601
    string coupon_code = 10;
    int32 total_order_amount_cents = 11;
    string order_id = 12;
}

message CalculatePriceRequest {
    PricingContext context = 1;
    repeated PriceLineItem line_items = 2;
}

message CalculatedLinePrice {
    string item_id = 1;
    int32 list_price_cents = 2;
    int32 base_price_cents = 3;
    int32 discount_amount_cents = 4;
    string discount_description = 5;
    int32 surcharge_amount_cents = 6;
    string surcharge_description = 7;
    int32 net_price_cents = 8;
    int32 net_line_total_cents = 9;
    string currency_code = 10;
    string pricing_source = 11;         // AGREEMENT, PROMOTION, PRICE_LIST, LIST_PRICE, COST_PLUS
    string price_agreement_id = 12;
    string promotion_id = 13;
    string discount_scheme_id = 14;
}

message CalculatePriceResponse {
    repeated CalculatedLinePrice lines = 1;
    int32 total_discount_cents = 2;
    int32 total_surcharge_cents = 3;
    int32 total_net_cents = 4;
    string currency_code = 5;
}

message CalculatePriceBatchRequest {
    repeated CalculatePriceRequest requests = 1;
}

message CalculatePriceBatchResponse {
    repeated CalculatePriceResponse responses = 1;
}

message ExplainPriceRequest {
    PricingContext context = 1;
    PriceLineItem line_item = 2;
}

message PricingExplanationStep {
    int32 step = 1;
    string source = 2;
    string description = 3;
    int32 price_before_cents = 4;
    int32 adjustment_cents = 5;
    int32 price_after_cents = 6;
}

message ExplainPriceResponse {
    string item_id = 1;
    repeated PricingExplanationStep steps = 2;
    int32 final_net_price_cents = 3;
}

message GetPriceListRequest {
    string tenant_id = 1;
    string price_list_id = 2;
}

message GetPriceListResponse {
    string id = 1;
    string name = 2;
    string currency_code = 3;
    string status = 4;
}

message GetItemPriceRequest {
    string tenant_id = 1;
    string item_id = 2;
    string price_list_id = 3;
    int32 quantity = 4;
}

message GetItemPriceResponse {
    int32 unit_price_cents = 1;
    int32 break_price_cents = 2;
    bool has_break = 3;
    string currency_code = 4;
}

message GetPriceAgreementRequest {
    string tenant_id = 1;
    string party_id = 2;
    string item_id = 3;
    string pricing_date = 4;
}

message GetPriceAgreementResponse {
    string agreement_id = 1;
    int32 negotiated_price_cents = 2;
    int32 minimum_quantity = 3;
    string effective_from = 4;
    string effective_to = 5;
    bool found = 6;
}

message ConvertCurrencyRequest {
    string tenant_id = 1;
    string from_currency = 2;
    string to_currency = 3;
    int32 amount_cents = 4;
    string conversion_date = 5;
}

message ConvertCurrencyResponse {
    int32 converted_amount_cents = 1;
    double exchange_rate = 2;
    string rate_type = 3;
}

message ValidateCouponRequest {
    string tenant_id = 1;
    string coupon_code = 2;
    string customer_id = 3;
    string customer_group = 4;
    int32 order_amount_cents = 5;
}

message ValidateCouponResponse {
    bool valid = 1;
    string promotion_id = 2;
    string promotion_name = 3;
    string offer_type = 4;
    double offer_value = 5;
    int32 offer_amount_cents = 6;
    string rejection_reason = 7;
}
```

---

## 6. Inter-Service Integration

### 6.1 Order Management (om-service)
- OM calls `CalculatePrice` gRPC when creating or updating sales order lines
- Price overrides from OM create price_overrides records requiring approval
- OM receives net price breakdown for display on order confirmation

### 6.2 Procurement (proc-service)
- Procurement calls `GetItemPrice` for purchase price lists to populate PO line prices
- Supplier price agreements consulted for negotiated purchase pricing
- Cost-plus rules used when no purchase price list is available

### 6.3 Accounts Receivable (ar-service)
- AR calls `CalculatePrice` for credit memo pricing (return pricing)
- Invoice pricing locked at order time; re-evaluation only on explicit request

### 6.4 Inventory (inv-service)
- Pricing calls INV gRPC to retrieve `average_cost_cents` and `last_purchase_cost_cents` for cost-plus calculations
- INV provides item attributes (category, UOM, type) for attribute pricing evaluation

### 6.5 Workflow (workflow-service)
- Price overrides exceeding threshold trigger approval workflow via Workflow gRPC
- Promotional pricing activation MAY require workflow approval

### 6.6 Integration Summary
| Consumer | gRPC Call | Purpose |
|----------|-----------|---------|
| OM | `CalculatePrice` | Sales order line pricing |
| OM | `CalculatePriceBatch` | Bulk order pricing |
| Procurement | `GetItemPrice` | Purchase order line pricing |
| Procurement | `GetPriceAgreement` | Supplier agreement lookup |
| AR | `CalculatePrice` | Credit memo / return pricing |
| Pricing (outbound) | INV `GetItemCost` | Retrieve item cost for cost-plus |
| Pricing (outbound) | Workflow `StartProcess` | Override approval flow |

---

## 7. Events Published

| Event | Trigger | Consumers |
|-------|---------|-----------|
| `pricing.price_list.created` | Price list created | OM, Proc |
| `pricing.price_list.activated` | Price list activated | OM, Proc |
| `pricing.price_list.expired` | Price list expired | OM, Proc, Report |
| `pricing.promotion.activated` | Promotion activated | OM, Report |
| `pricing.promotion.redeemed` | Coupon redeemed | OM, Report |
| `pricing.promotion.expired` | Promotion expired | OM, Report |
| `pricing.discount.created` | Discount scheme created | OM |
| `pricing.agreement.created` | Price agreement created | OM, Proc, Report |
| `pricing.agreement.expired` | Price agreement expired | OM, Proc, Report |
| `pricing.override.requested` | Price override requested | Workflow |
| `pricing.override.approved` | Price override approved | OM |
| `pricing.override.rejected` | Price override rejected | OM |
| `pricing.price_calculated` | Price calculation completed | OM, AR, Report |
| `pricing.currency_rate.updated` | Exchange rate updated | GL, CM |

---

## 8. Migrations

```
migrations/pricing/
  001_create_price_lists.up.sql
  001_create_price_lists.down.sql
  002_create_price_list_lines.up.sql
  002_create_price_list_lines.down.sql
  003_create_price_breaks.up.sql
  003_create_price_breaks.down.sql
  004_create_pricing_strategies.up.sql
  004_create_pricing_strategies.down.sql
  005_create_pricing_rules.up.sql
  005_create_pricing_rules.down.sql
  006_create_discount_schemes.up.sql
  006_create_discount_schemes.down.sql
  007_create_promotional_pricing.up.sql
  007_create_promotional_pricing.down.sql
  008_create_surcharge_rules.up.sql
  008_create_surcharge_rules.down.sql
  009_create_attribute_pricing.up.sql
  009_create_attribute_pricing.down.sql
  010_create_cost_plus_rules.up.sql
  010_create_cost_plus_rules.down.sql
  011_create_price_agreements.up.sql
  011_create_price_agreements.down.sql
  012_create_price_agreement_lines.up.sql
  012_create_price_agreement_lines.down.sql
  013_create_currency_conversion_rules.up.sql
  013_create_currency_conversion_rules.down.sql
  014_create_price_overrides.up.sql
  014_create_price_overrides.down.sql
  015_create_pricing_sequences.up.sql
  015_create_pricing_sequences.down.sql
```
