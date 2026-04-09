# 120 - Self-Service Procurement Specification

## 1. Domain Overview

Self-Service Procurement enables employees across the organization to browse catalogs, create requisitions, and request goods and services without specialized procurement training. It provides an Amazon-like shopping experience with guided buying, smart suggestions, and policy-compliant requisition generation.

**Bounded Context:** Self-Service Procurement & Employee Buying
**Service Name:** `selfproc-service`
**Database:** `data/selfproc.db`
**HTTP Port:** 8200 | **gRPC Port:** 9200

---

## 2. Database Schema

### 2.1 Shopping Catalogs
```sql
CREATE TABLE ssp_catalogs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    catalog_name TEXT NOT NULL,
    catalog_code TEXT NOT NULL,
    description TEXT,
    catalog_type TEXT NOT NULL DEFAULT 'INTERNAL'
        CHECK(catalog_type IN ('INTERNAL','EXTERNAL','PUNCHOUT')),
    supplier_id TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','ARCHIVED')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    locale TEXT NOT NULL DEFAULT 'en-US',
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, catalog_code)
);

CREATE INDEX idx_ssp_catalogs_tenant_status ON ssp_catalogs(tenant_id, status);
```

### 2.2 Catalog Items
```sql
CREATE TABLE ssp_catalog_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    catalog_id TEXT NOT NULL,
    item_number TEXT NOT NULL,
    item_name TEXT NOT NULL,
    description TEXT,
    category_id TEXT,
    supplier_id TEXT,
    supplier_item_code TEXT,
    unit_of_measure TEXT NOT NULL,
    unit_price_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    image_url TEXT,
    punchout_url TEXT,
    keywords TEXT,                       -- Comma-separated search keywords
    specifications TEXT,                 -- JSON blob of technical specs
    minimum_order_qty INTEGER NOT NULL DEFAULT 1,
    maximum_order_qty INTEGER,
    is_featured INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (catalog_id) REFERENCES ssp_catalogs(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, catalog_id, item_number)
);

CREATE INDEX idx_ssp_items_tenant_catalog ON ssp_catalog_items(tenant_id, catalog_id);
CREATE INDEX idx_ssp_items_tenant_category ON ssp_catalog_items(tenant_id, category_id);
CREATE INDEX idx_ssp_items_tenant_search ON ssp_catalog_items(tenant_id, item_name, keywords);
```

### 2.3 Shopping Categories
```sql
CREATE TABLE ssp_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_name TEXT NOT NULL,
    category_code TEXT NOT NULL,
    parent_category_id TEXT,
    description TEXT,
    icon TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    display_name TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_category_id) REFERENCES ssp_categories(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, category_code)
);
```

### 2.4 Shopping Carts
```sql
CREATE TABLE ssp_carts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requester_id TEXT NOT NULL,
    cart_name TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUBMITTED','CONVERTED','ABANDONED')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    item_count INTEGER NOT NULL DEFAULT 0,
    delivery_address TEXT,
    delivery_date TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, requester_id, status)
);
```

### 2.5 Cart Lines
```sql
CREATE TABLE ssp_cart_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cart_id TEXT NOT NULL,
    catalog_item_id TEXT,
    item_number TEXT NOT NULL,
    item_name TEXT NOT NULL,
    description TEXT,
    supplier_id TEXT,
    quantity INTEGER NOT NULL DEFAULT 1,
    unit_of_measure TEXT NOT NULL,
    unit_price_cents INTEGER NOT NULL,
    line_total_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    delivery_date TEXT,
    charge_account_id TEXT,             -- GL account for charge
    project_id TEXT,                     -- Optional project charge
    notes TEXT,
    punchout_url TEXT,
    attachments TEXT,                    -- JSON array of attachment IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (cart_id) REFERENCES ssp_carts(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, cart_id, id)
);
```

### 2.6 Buying Policies
```sql
CREATE TABLE ssp_buying_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_name TEXT NOT NULL,
    description TEXT,
    policy_type TEXT NOT NULL CHECK(policy_type IN ('SPEND_LIMIT','PREFERRED_SUPPLIER','CATEGORY_RESTRICTION','APPROVAL_RULE')),
    condition_json TEXT NOT NULL,        -- JSON: { "field": "total_amount", "operator": ">", "value": 50000 }
    action_json TEXT NOT NULL,           -- JSON: { "type": "route_to_supplier", "supplier_id": "..." }
    priority INTEGER NOT NULL DEFAULT 0,
    is_enforced INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_name)
);
```

### 2.7 Smart Suggestions
```sql
CREATE TABLE ssp_purchase_history (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requester_id TEXT NOT NULL,
    catalog_item_id TEXT NOT NULL,
    item_number TEXT NOT NULL,
    item_name TEXT NOT NULL,
    purchase_count INTEGER NOT NULL DEFAULT 1,
    last_purchased_at TEXT NOT NULL,
    average_quantity INTEGER,
    average_price_cents INTEGER,
    frequency_days INTEGER,             -- Average days between purchases
    supplier_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, requester_id, catalog_item_id)
);

CREATE INDEX idx_ssp_history_tenant_requester ON ssp_purchase_history(tenant_id, requester_id, last_purchased_at DESC);
```

---

## 3. REST API Endpoints

### 3.1 Catalog Browsing
```
GET    /api/v1/selfproc/catalogs                    Permission: ssp.catalogs.read
GET    /api/v1/selfproc/catalogs/{id}               Permission: ssp.catalogs.read
GET    /api/v1/selfproc/catalogs/{id}/items         Permission: ssp.catalogs.read
  ?filter[category]={id}
  ?filter[search]={keyword}
  ?sort=+item_name,-unit_price_cents
  ?cursor=xxx&limit=50
GET    /api/v1/selfproc/categories                  Permission: ssp.catalogs.read
GET    /api/v1/selfproc/categories/{id}/items       Permission: ssp.catalogs.read
GET    /api/v1/selfproc/items/{id}                  Permission: ssp.catalogs.read
POST   /api/v1/selfproc/catalogs                    Permission: ssp.catalogs.create
PUT    /api/v1/selfproc/catalogs/{id}               Permission: ssp.catalogs.update
POST   /api/v1/selfproc/catalogs/{id}/items         Permission: ssp.catalogs.create
PUT    /api/v1/selfproc/catalogs/{id}/items/bulk    Permission: ssp.catalogs.update
```

### 3.2 Shopping Cart
```
GET    /api/v1/selfproc/cart                         Permission: ssp.cart.read
POST   /api/v1/selfproc/cart/items                   Permission: ssp.cart.update
PUT    /api/v1/selfproc/cart/items/{id}              Permission: ssp.cart.update
DELETE /api/v1/selfproc/cart/items/{id}              Permission: ssp.cart.update
POST   /api/v1/selfproc/cart/checkout                Permission: ssp.cart.checkout
  Request: { "delivery_address": "...", "delivery_date": "2024-03-01", "notes": "..." }
  Response 201: { "data": { "requisition_id": "...", "requisition_number": "REQ-2024-00123" } }
```

### 3.3 Smart Suggestions
```
GET    /api/v1/selfproc/suggestions                  Permission: ssp.suggestions.read
  ?limit=10
  Response: Items frequently purchased by the user, sorted by recency and frequency
GET    /api/v1/selfproc/suggestions/trending         Permission: ssp.suggestions.read
  Response: Popular items across the tenant
GET    /api/v1/selfproc/suggestions/reorder          Permission: ssp.suggestions.read
  Response: Items the user regularly reorders based on frequency_days
```

### 3.4 Buying Policies
```
GET    /api/v1/selfproc/policies                     Permission: ssp.policies.read
POST   /api/v1/selfproc/policies                     Permission: ssp.policies.create
PUT    /api/v1/selfproc/policies/{id}                Permission: ssp.policies.update
DELETE /api/v1/selfproc/policies/{id}                Permission: ssp.policies.delete
```

### 3.5 Punchout
```
POST   /api/v1/selfproc/punchout/setup               Permission: ssp.punchout.initiate
  Request: { "catalog_id": "...", "supplier_id": "..." }
  Response: { "data": { "punchout_url": "https://...", "cart_id": "..." } }

POST   /api/v1/selfproc/punchout/return              Permission: ssp.punchout.receive
  Request: { "cart_id": "...", "punchout_data": { ... } }
  Response: Cart updated with punchout items
```

---

## 4. Business Rules

### 4.1 Cart Checkout
- Checkout converts cart to a procurement requisition (calls proc-service via gRPC)
- Each cart line becomes a requisition line
- Buying policies are evaluated before checkout
- If any policy violation is found, checkout is blocked with clear messaging
- Cart status changes to CONVERTED after successful requisition creation

### 4.2 Smart Suggestions
- Suggestions are based on purchase history for the requesting user
- Items are scored by: recency (40%), frequency (40%), spend amount (20%)
- "Reorder" suggestions appear when frequency_days has elapsed since last purchase
- New employees see trending items tenant-wide until personal history builds up

### 4.3 Buying Policies
- Policies are evaluated in priority order (higher priority first)
- Spend limit policies check cart total against thresholds
- Preferred supplier policies suggest or enforce supplier selection
- Category restriction policies allow/block specific categories per role/department
- Non-enforced policies show warnings; enforced policies block checkout

### 4.4 Punchout Integration
- Punchout catalogs redirect to external supplier sites via cXML/OCI protocol
- Return data (selected items, prices) is mapped back to cart lines
- Punchout items retain supplier pricing at checkout time
- Punchout session timeout enforced (configurable, default 30 minutes)

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.selfproc.v1;

service SelfServiceProcurementService {
    rpc GetCatalogItems(GetCatalogItemsRequest) returns (GetCatalogItemsResponse);
    rpc SearchItems(SearchItemsRequest) returns (SearchItemsResponse);
    rpc GetSuggestions(GetSuggestionsRequest) returns (GetSuggestionsResponse);
    rpc CheckoutCart(CheckoutCartRequest) returns (CheckoutCartResponse);
    rpc ValidateBuyingPolicies(ValidatePoliciesRequest) returns (ValidatePoliciesResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **proc-service**: Checkout creates requisitions via gRPC
- **inv-service**: Catalog items reference inventory items
- **workflow-service**: Approval routing for requisitions
- **dms-service**: Item images and attachments
- **ai-service**: Smart suggestion engine, natural language search

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `ssp.cart.created` | Cart created | cart_id, requester_id |
| `ssp.cart.item_added` | Item added to cart | cart_id, catalog_item_id, quantity |
| `ssp.cart.checked_out` | Cart converted to requisition | cart_id, requisition_id |
| `ssp.catalog.updated` | Catalog items refreshed | catalog_id |
| `ssp.suggestion.generated` | Suggestions generated for user | requester_id, item_ids |
| `ssp.policy.violated` | Buying policy violation detected | cart_id, policy_id, violation_details |

---

## 7. Migrations

### Migration Order for selfproc-service:
1. V001: `ssp_categories`
2. V002: `ssp_catalogs`
3. V003: `ssp_catalog_items`
4. V004: `ssp_carts`
5. V005: `ssp_cart_lines`
6. V006: `ssp_buying_policies`
7. V007: `ssp_purchase_history`
8. V008: Triggers for `updated_at`
9. V009: Seed data (default categories, sample catalog)
