# 192 - Retail Industry Solution Service Specification

## 1. Domain Overview

Retail Industry Solution provides comprehensive retail operations management supporting store location management with multiple format types, merchandise hierarchy with department/class/subclass/style/color/size levels and buying attributes, pricing strategies with everyday/promotional/clearance/markdown/dynamic pricing rules and zone-based price management, point-of-sale transaction capture with line items, discounts, tender types, and loyalty card integration, and automated replenishment with min/max stock, reorder point, economic order quantity, and vendor allocation rules. Enables unified commerce across physical and digital channels with real-time inventory visibility and demand-driven replenishment. Integrates with Inventory for stock management, Order Management for fulfillment, and Loyalty for customer rewards.

**Bounded Context:** Retail Operations, Merchandising & Store Management
**Service Name:** `retail-service`
**Database:** `data/retail.db`
**HTTP Port:** 8210 | **gRPC Port:** 9210

---

## 2. Database Schema

### 2.1 Stores
```sql
CREATE TABLE ret_stores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_code TEXT NOT NULL,
    store_name TEXT NOT NULL,
    address_line1 TEXT NOT NULL,
    address_line2 TEXT,
    city TEXT NOT NULL,
    state_province TEXT NOT NULL,
    postal_code TEXT NOT NULL,
    country TEXT NOT NULL,
    latitude REAL,
    longitude REAL,
    phone TEXT,
    store_format TEXT NOT NULL
        CHECK(store_format IN ('FLAGSHIP','STANDARD','OUTLET','POP_UP')),
    operating_hours TEXT NOT NULL,                    -- JSON: day-of-week hours
    region TEXT NOT NULL,
    district TEXT,
    attributes TEXT,                                  -- JSON: custom store attributes
    total_square_feet INTEGER,
    opening_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','RENOVATING','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, store_code)
);

CREATE INDEX idx_ret_stores_tenant_region ON ret_stores(tenant_id, region);
CREATE INDEX idx_ret_stores_tenant_format ON ret_stores(tenant_id, store_format);
CREATE INDEX idx_ret_stores_tenant_status ON ret_stores(tenant_id, status);
CREATE INDEX idx_ret_stores_tenant_city ON ret_stores(tenant_id, city);
CREATE INDEX idx_ret_stores_tenant_active ON ret_stores(tenant_id, is_active);
```

### 2.2 Merchandise Hierarchy
```sql
CREATE TABLE ret_merchandise_hierarchy (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    parent_id TEXT,
    level_name TEXT NOT NULL
        CHECK(level_name IN ('DEPARTMENT','CLASS','SUBCLASS','STYLE','COLOR','SIZE')),
    level_code TEXT NOT NULL,
    level_description TEXT,
    buying_attributes TEXT,                           -- JSON: buying group, season, vendor prefs
    sort_order INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','DISCONTINUED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_id) REFERENCES ret_merchandise_hierarchy(id),
    UNIQUE(tenant_id, level_code)
);

CREATE INDEX idx_ret_merch_tenant_parent ON ret_merchandise_hierarchy(tenant_id, parent_id);
CREATE INDEX idx_ret_merch_tenant_level ON ret_merchandise_hierarchy(tenant_id, level_name);
CREATE INDEX idx_ret_merch_tenant_status ON ret_merchandise_hierarchy(tenant_id, status);
CREATE INDEX idx_ret_merch_tenant_active ON ret_merchandise_hierarchy(tenant_id, is_active);
```

### 2.3 Price Strategies
```sql
CREATE TABLE ret_price_strategies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    strategy_name TEXT NOT NULL,
    strategy_type TEXT NOT NULL
        CHECK(strategy_type IN ('EVERYDAY','PROMOTIONAL','CLEARANCE','MARKDOWN','DYNAMIC')),
    merch_hierarchy_id TEXT,                          -- Optional: specific to hierarchy node
    price_zones TEXT,                                 -- JSON: list of zone IDs
    rules TEXT NOT NULL,                              -- JSON: pricing rules and conditions
    base_price_cents INTEGER NOT NULL DEFAULT 0,
    min_price_cents INTEGER,
    max_price_cents INTEGER,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    priority INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','EXPIRED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, strategy_name)
);

CREATE INDEX idx_ret_price_tenant_type ON ret_price_strategies(tenant_id, strategy_type);
CREATE INDEX idx_ret_price_tenant_status ON ret_price_strategies(tenant_id, status);
CREATE INDEX idx_ret_price_tenant_dates ON ret_price_strategies(tenant_id, effective_from, effective_to);
CREATE INDEX idx_ret_price_tenant_merch ON ret_price_strategies(tenant_id, merch_hierarchy_id);
CREATE INDEX idx_ret_price_tenant_active ON ret_price_strategies(tenant_id, is_active);
```

### 2.4 POS Transactions
```sql
CREATE TABLE ret_pos_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    transaction_number TEXT NOT NULL,
    register_number TEXT NOT NULL,
    transaction_date TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('SALE','RETURN','EXCHANGE','VOID')),
    line_items TEXT NOT NULL,                         -- JSON: array of line item objects
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    discount_total_cents INTEGER NOT NULL DEFAULT 0,
    tax_total_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    tender_details TEXT,                              -- JSON: payment methods and amounts
    loyalty_card_number TEXT,
    customer_id TEXT,
    cashier_id TEXT,
    status TEXT NOT NULL DEFAULT 'COMPLETED'
        CHECK(status IN ('COMPLETED','VOIDED','REFUNDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (store_id) REFERENCES ret_stores(id),
    UNIQUE(tenant_id, store_id, transaction_number)
);

CREATE INDEX idx_ret_pos_tenant_store ON ret_pos_transactions(tenant_id, store_id);
CREATE INDEX idx_ret_pos_tenant_date ON ret_pos_transactions(tenant_id, transaction_date);
CREATE INDEX idx_ret_pos_tenant_type ON ret_pos_transactions(tenant_id, transaction_type);
CREATE INDEX idx_ret_pos_tenant_loyalty ON ret_pos_transactions(tenant_id, loyalty_card_number);
CREATE INDEX idx_ret_pos_tenant_customer ON ret_pos_transactions(tenant_id, customer_id);
CREATE INDEX idx_ret_pos_tenant_register ON ret_pos_transactions(tenant_id, store_id, register_number);
```

### 2.5 Replenishment Rules
```sql
CREATE TABLE ret_replenishment_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    merch_hierarchy_id TEXT NOT NULL,                 -- SKU level
    min_stock INTEGER NOT NULL DEFAULT 0,
    max_stock INTEGER NOT NULL DEFAULT 0,
    reorder_point INTEGER NOT NULL DEFAULT 0,
    economic_order_quantity INTEGER NOT NULL DEFAULT 0,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    vendor_id TEXT,
    vendor_allocation_pct REAL NOT NULL DEFAULT 100.0,
    replenishment_method TEXT NOT NULL DEFAULT 'MIN_MAX'
        CHECK(replenishment_method IN ('MIN_MAX','REORDER_POINT','DEMAND_BASED','FORECAST_DRIVEN')),
    review_cycle_days INTEGER NOT NULL DEFAULT 1,
    safety_stock INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','SUSPENDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (store_id) REFERENCES ret_stores(id),
    FOREIGN KEY (merch_hierarchy_id) REFERENCES ret_merchandise_hierarchy(id),
    UNIQUE(tenant_id, store_id, merch_hierarchy_id)
);

CREATE INDEX idx_ret_replen_tenant_store ON ret_replenishment_rules(tenant_id, store_id);
CREATE INDEX idx_ret_replen_tenant_merch ON ret_replenishment_rules(tenant_id, merch_hierarchy_id);
CREATE INDEX idx_ret_replen_tenant_vendor ON ret_replenishment_rules(tenant_id, vendor_id);
CREATE INDEX idx_ret_replen_tenant_status ON ret_replenishment_rules(tenant_id, status);
CREATE INDEX idx_ret_replen_tenant_active ON ret_replenishment_rules(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Stores
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/retail/stores` | List stores |
| POST | `/api/v1/retail/stores` | Create store |
| GET | `/api/v1/retail/stores/{id}` | Get store details |
| PUT | `/api/v1/retail/stores/{id}` | Update store |
| PATCH | `/api/v1/retail/stores/{id}/status` | Update store status |
| GET | `/api/v1/retail/stores/search` | Search stores by location |

### 3.2 Merchandise Hierarchy
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/retail/merchandise` | List merchandise hierarchy |
| POST | `/api/v1/retail/merchandise` | Create hierarchy node |
| GET | `/api/v1/retail/merchandise/{id}` | Get hierarchy node |
| PUT | `/api/v1/retail/merchandise/{id}` | Update hierarchy node |
| GET | `/api/v1/retail/merchandise/{id}/children` | Get child nodes |
| GET | `/api/v1/retail/merchandise/{id}/tree` | Get full subtree |

### 3.3 Pricing
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/retail/pricing` | List price strategies |
| POST | `/api/v1/retail/pricing` | Create price strategy |
| GET | `/api/v1/retail/pricing/{id}` | Get price strategy |
| PUT | `/api/v1/retail/pricing/{id}` | Update price strategy |
| POST | `/api/v1/retail/pricing/{id}/activate` | Activate price strategy |
| GET | `/api/v1/retail/pricing/lookup` | Lookup effective price by SKU/zone |

### 3.4 POS Transactions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/retail/pos/transactions` | Record POS transaction |
| GET | `/api/v1/retail/pos/transactions` | List POS transactions |
| GET | `/api/v1/retail/pos/transactions/{id}` | Get transaction details |
| POST | `/api/v1/retail/pos/transactions/{id}/void` | Void a transaction |
| GET | `/api/v1/retail/pos/sales-summary` | Get sales summary by store/date |

### 3.5 Replenishment
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/retail/replenishment` | List replenishment rules |
| POST | `/api/v1/retail/replenishment` | Create replenishment rule |
| GET | `/api/v1/retail/replenishment/{id}` | Get rule details |
| PUT | `/api/v1/retail/replenishment/{id}` | Update rule |
| POST | `/api/v1/retail/replenishment/generate-orders` | Generate replenishment orders |
| GET | `/api/v1/retail/replenishment/recommendations` | Get replenishment recommendations |

---

## 4. Business Rules

### 4.1 Store Management
1. Each store MUST have a unique store_code within a tenant.
2. Store status transitions MUST follow: ACTIVE -> INACTIVE -> CLOSED; RENOVATING is a sub-status of ACTIVE.
3. A store in CLOSED status MUST NOT accept new POS transactions.
4. Operating hours stored as JSON MUST cover all seven days of the week.
5. Stores MUST be assigned to exactly one region; district assignment is optional.

### 4.2 Merchandise Hierarchy
6. Merchandise hierarchy MUST follow the fixed level sequence: DEPARTMENT -> CLASS -> SUBCLASS -> STYLE -> COLOR -> SIZE.
7. A hierarchy node at level CLASS MUST have a parent at level DEPARTMENT; nodes at level DEPARTMENT MUST have no parent.
8. level_code values MUST be unique within a tenant across all hierarchy levels.
9. Nodes with INACTIVE or DISCONTINUED status MUST NOT be referenced in new POS transactions.

### 4.3 Pricing Management
10. Price strategy names MUST be unique within a tenant.
11. DYNAMIC pricing strategies MUST include rules JSON with algorithm configuration.
12. Price strategies with overlapping effective dates and same merch_hierarchy_id MUST use priority to resolve conflicts.
13. base_price_cents MUST be a positive value; if min_price_cents is set, it MUST be less than or equal to base_price_cents.
14. Price zones referenced in the price_zones JSON MUST be validated against the store region configuration.

### 4.4 POS Transactions
15. transaction_number MUST be unique within the scope of (tenant_id, store_id).
16. POS transactions of type VOID MUST reference the original transaction.
17. total_cents MUST equal subtotal_cents minus discount_total_cents plus tax_total_cents.
18. line_items JSON MUST contain at least one line item with SKU, quantity, and unit_price.
19. Completed transactions MUST NOT be modified; corrections MUST use RETURN or VOID transaction types.

### 4.5 Replenishment
20. The combination of (tenant_id, store_id, merch_hierarchy_id) MUST be unique for replenishment rules.
21. max_stock MUST be greater than min_stock; reorder_point MUST be between min_stock and max_stock.
22. economic_order_quantity MUST be a positive integer.
23. vendor_allocation_pct values for the same store/SKU combination across vendors MUST sum to 100.
24. Replenishment orders generated MUST NOT exceed max_stock minus current stock on hand.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.retail.v1;

service RetailService {
    // Store management
    rpc CreateStore(CreateStoreRequest) returns (CreateStoreResponse);
    rpc GetStore(GetStoreRequest) returns (GetStoreResponse);
    rpc ListStores(ListStoresRequest) returns (ListStoresResponse);

    // Merchandise hierarchy
    rpc CreateMerchandiseNode(CreateMerchandiseNodeRequest) returns (CreateMerchandiseNodeResponse);
    rpc GetMerchandiseTree(GetMerchandiseTreeRequest) returns (GetMerchandiseTreeResponse);
    rpc ListMerchandiseNodes(ListMerchandiseNodesRequest) returns (ListMerchandiseNodesResponse);

    // Pricing
    rpc CreatePriceStrategy(CreatePriceStrategyRequest) returns (CreatePriceStrategyResponse);
    rpc LookupPrice(LookupPriceRequest) returns (LookupPriceResponse);

    // POS transactions
    rpc RecordPOSTransaction(RecordPOSTransactionRequest) returns (RecordPOSTransactionResponse);
    rpc GetPOSTransaction(GetPOSTransactionRequest) returns (GetPOSTransactionResponse);
    rpc GetSalesSummary(GetSalesSummaryRequest) returns (GetSalesSummaryResponse);

    // Replenishment
    rpc CreateReplenishmentRule(CreateReplenishmentRuleRequest) returns (CreateReplenishmentRuleResponse);
    rpc GenerateReplenishmentOrders(GenerateReplenishmentOrdersRequest) returns (GenerateReplenishmentOrdersResponse);
    rpc GetReplenishmentRecommendations(GetReplenishmentRecommendationsRequest) returns (GetReplenishmentRecommendationsResponse);
}

message CreateStoreRequest {
    string tenant_id = 1;
    string store_code = 2;
    string store_name = 3;
    string address_line1 = 4;
    string address_line2 = 5;
    string city = 6;
    string state_province = 7;
    string postal_code = 8;
    string country = 9;
    double latitude = 10;
    double longitude = 11;
    string phone = 12;
    string store_format = 13;
    string operating_hours = 14;
    string region = 15;
    string district = 16;
    string attributes = 17;
    int32 total_square_feet = 18;
    string opening_date = 19;
}

message CreateStoreResponse {
    string store_id = 1;
    string store_code = 2;
    string store_name = 3;
    string status = 4;
}

message GetStoreRequest {
    string tenant_id = 1;
    string store_id = 2;
}

message GetStoreResponse {
    string store_id = 1;
    string store_code = 2;
    string store_name = 3;
    string store_format = 4;
    string city = 5;
    string region = 6;
    string status = 7;
    string operating_hours = 8;
}

message ListStoresRequest {
    string tenant_id = 1;
    string region = 2;
    string store_format = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListStoresResponse {
    repeated GetStoreResponse stores = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CreateMerchandiseNodeRequest {
    string tenant_id = 1;
    string parent_id = 2;
    string level_name = 3;
    string level_code = 4;
    string level_description = 5;
    string buying_attributes = 6;
    int32 sort_order = 7;
}

message CreateMerchandiseNodeResponse {
    string node_id = 1;
    string level_code = 2;
    string level_name = 3;
}

message GetMerchandiseTreeRequest {
    string tenant_id = 1;
    string root_node_id = 2;
    int32 max_depth = 3;
}

message GetMerchandiseTreeResponse {
    repeated MerchandiseNodeInfo nodes = 1;
}

message MerchandiseNodeInfo {
    string node_id = 1;
    string parent_id = 2;
    string level_name = 3;
    string level_code = 4;
    string level_description = 5;
    repeated MerchandiseNodeInfo children = 6;
}

message ListMerchandiseNodesRequest {
    string tenant_id = 1;
    string level_name = 2;
    string parent_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListMerchandiseNodesResponse {
    repeated MerchandiseNodeInfo nodes = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CreatePriceStrategyRequest {
    string tenant_id = 1;
    string strategy_name = 2;
    string strategy_type = 3;
    string merch_hierarchy_id = 4;
    string price_zones = 5;
    string rules = 6;
    int64 base_price_cents = 7;
    int64 min_price_cents = 8;
    int64 max_price_cents = 9;
    string effective_from = 10;
    string effective_to = 11;
    int32 priority = 12;
}

message CreatePriceStrategyResponse {
    string strategy_id = 1;
    string strategy_name = 2;
    string status = 3;
}

message LookupPriceRequest {
    string tenant_id = 1;
    string merch_hierarchy_id = 2;
    string store_id = 3;
    string lookup_date = 4;
}

message LookupPriceResponse {
    string strategy_id = 1;
    int64 base_price_cents = 2;
    int64 effective_price_cents = 3;
    string strategy_type = 4;
}

message RecordPOSTransactionRequest {
    string tenant_id = 1;
    string store_id = 2;
    string transaction_number = 3;
    string register_number = 4;
    string transaction_date = 5;
    string transaction_type = 6;
    string line_items = 7;
    int64 subtotal_cents = 8;
    int64 discount_total_cents = 9;
    int64 tax_total_cents = 10;
    int64 total_cents = 11;
    string tender_details = 12;
    string loyalty_card_number = 13;
    string customer_id = 14;
    string cashier_id = 15;
}

message RecordPOSTransactionResponse {
    string transaction_id = 1;
    string transaction_number = 2;
    string status = 3;
}

message GetPOSTransactionRequest {
    string tenant_id = 1;
    string transaction_id = 2;
}

message GetPOSTransactionResponse {
    string transaction_id = 1;
    string transaction_number = 2;
    string store_id = 3;
    string transaction_date = 4;
    string transaction_type = 5;
    int64 total_cents = 6;
    string status = 7;
}

message GetSalesSummaryRequest {
    string tenant_id = 1;
    string store_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetSalesSummaryResponse {
    int32 total_transactions = 1;
    int64 total_sales_cents = 2;
    int64 total_discounts_cents = 3;
    int64 total_tax_cents = 4;
    double average_transaction_value = 5;
}

message CreateReplenishmentRuleRequest {
    string tenant_id = 1;
    string store_id = 2;
    string merch_hierarchy_id = 3;
    int32 min_stock = 4;
    int32 max_stock = 5;
    int32 reorder_point = 6;
    int32 economic_order_quantity = 7;
    int32 lead_time_days = 8;
    string vendor_id = 9;
    double vendor_allocation_pct = 10;
    string replenishment_method = 11;
    int32 review_cycle_days = 12;
    int32 safety_stock = 13;
}

message CreateReplenishmentRuleResponse {
    string rule_id = 1;
    string store_id = 2;
    string merch_hierarchy_id = 3;
    string status = 4;
}

message GenerateReplenishmentOrdersRequest {
    string tenant_id = 1;
    repeated string store_ids = 2;
    repeated string merch_hierarchy_ids = 3;
}

message GenerateReplenishmentOrdersResponse {
    int32 orders_generated = 1;
    repeated ReplenishmentOrderSummary orders = 2;
}

message ReplenishmentOrderSummary {
    string store_id = 1;
    string merch_hierarchy_id = 2;
    int32 quantity = 3;
    string vendor_id = 4;
}

message GetReplenishmentRecommendationsRequest {
    string tenant_id = 1;
    string store_id = 2;
    int32 page_size = 3;
}

message GetReplenishmentRecommendationsResponse {
    repeated ReplenishmentRecommendation recommendations = 1;
    int32 total_count = 2;
}

message ReplenishmentRecommendation {
    string rule_id = 1;
    string store_id = 2;
    string merch_hierarchy_id = 3;
    int32 current_stock = 4;
    int32 recommended_quantity = 5;
    string reason = 6;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `inv-service` | Real-time inventory levels and stock-on-hand for replenishment calculations |
| `om-service` | Order fulfillment status and shipment confirmations for inventory reconciliation |
| `loyalty-service` | Loyalty member data, card lookups, and points earning triggers from POS |
| `pricing-service` | Base pricing and promotional pricing rules for price strategy resolution |
| `product-hub-service` | Product master data, SKU details, and merchandise attributes |
| `auth-service` | User identity for cashier assignment and store manager role resolution |

### Published To
| Service | Data |
|---------|------|
| `inv-service` | POS sales deduction transactions, replenishment order requests |
| `om-service` | Replenishment purchase requisitions for vendor order creation |
| `loyalty-service` | POS transaction events for points earning, tier evaluation |
| `gl-service` | Sales revenue journal entries, tax liability postings |
| `reporting-service` | Store performance metrics, sales analytics, inventory turnover dashboards |
| `notification-service` | Low stock alerts, replenishment order confirmations, price change notices |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `retail.sale.completed` | `retail.events` | `{ transaction_id, store_id, total_cents, line_item_count, loyalty_card }` | POS sale transaction completed |
| `retail.inventory.threshold` | `retail.events` | `{ store_id, merch_hierarchy_id, current_stock, reorder_point, min_stock }` | Store inventory crossed replenishment threshold |
| `retail.price.changed` | `retail.events` | `{ strategy_id, merch_hierarchy_id, old_price_cents, new_price_cents, effective_from }` | Price strategy activated or updated |
| `retail.store.opened` | `retail.events` | `{ store_id, store_code, store_name, store_format, region, opening_date }` | New store location opened |

---

## 8. Migrations

1. V001: `ret_stores`
2. V002: `ret_merchandise_hierarchy`
3. V003: `ret_price_strategies`
4. V004: `ret_pos_transactions`
5. V005: `ret_replenishment_rules`
6. V006: Triggers for `updated_at`
