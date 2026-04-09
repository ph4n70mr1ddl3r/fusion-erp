# 59 - Commerce Service Specification

## 1. Domain Overview

Commerce provides a B2B/B2C e-commerce platform with multi-storefront management, hierarchical product catalogs with category trees, shopping cart lifecycle management, checkout processing with payment integration, promotion and discount engine, shipping method management, product review and rating system, customer wishlists, and web order tracking. Supports both business-to-business (account-based, bulk ordering, negotiated pricing) and business-to-consumer (guest checkout, retail pricing) workflows. Integrates with OM for order fulfillment, INV for real-time availability, PRICING for price resolution, and CUSTOMER-DATA-PLATFORM for unified customer profiles.

**Bounded Context:** B2B/B2C E-Commerce & Digital Storefront
**Service Name:** `commerce-service`
**Database:** `data/commerce.db`
**HTTP Port:** 8091 | **gRPC Port:** 9091

---

## 2. Database Schema

### 2.1 Storefronts
```sql
CREATE TABLE storefronts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    store_type TEXT NOT NULL
        CHECK(store_type IN ('B2B','B2C','B2B2C','MARKETPLACE')),
    base_url TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    language_code TEXT NOT NULL DEFAULT 'en',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    theme_config TEXT,                             -- JSON: theme settings and branding
    seo_config TEXT,                               -- JSON: SEO metadata and URL patterns
    legal_config TEXT,                             -- JSON: terms, privacy policy, cookie consent
    tax_inclusive INTEGER NOT NULL DEFAULT 0,
    allow_guest_checkout INTEGER NOT NULL DEFAULT 1,
    allow_wishlist INTEGER NOT NULL DEFAULT 1,
    allow_reviews INTEGER NOT NULL DEFAULT 1,
    max_cart_items INTEGER NOT NULL DEFAULT 100,
    session_timeout_minutes INTEGER NOT NULL DEFAULT 30,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','MAINTENANCE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, store_code)
);

CREATE INDEX idx_storefronts_tenant_type ON storefronts(tenant_id, store_type);
CREATE INDEX idx_storefronts_tenant_status ON storefronts(tenant_id, status);
CREATE INDEX idx_storefronts_tenant_active ON storefronts(tenant_id, is_active);
```

### 2.2 Catalogs
```sql
CREATE TABLE catalogs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    catalog_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    catalog_type TEXT NOT NULL DEFAULT 'PRODUCT'
        CHECK(catalog_type IN ('PRODUCT','SERVICE','DIGITAL','MIXED')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    pricing_strategy TEXT NOT NULL DEFAULT 'RETAIL'
        CHECK(pricing_strategy IN ('RETAIL','NEGOTIATED','TIERED','CONTRACT')),
    is_default INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (store_id) REFERENCES storefronts(id),
    UNIQUE(tenant_id, catalog_code)
);

CREATE INDEX idx_catalogs_tenant_store ON catalogs(tenant_id, store_id);
CREATE INDEX idx_catalogs_tenant_type ON catalogs(tenant_id, catalog_type);
CREATE INDEX idx_catalogs_tenant_status ON catalogs(tenant_id, status);
CREATE INDEX idx_catalogs_tenant_active ON catalogs(tenant_id, is_active);
```

### 2.3 Catalog Categories
```sql
CREATE TABLE catalog_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    catalog_id TEXT NOT NULL,
    category_name TEXT NOT NULL,
    category_code TEXT NOT NULL,
    description TEXT,
    parent_category_id TEXT,                       -- NULL = root category
    level INTEGER NOT NULL DEFAULT 0,
    path TEXT NOT NULL,                            -- Materialized path (e.g. "/electronics/computers/laptops")
    image_url TEXT,
    banner_url TEXT,
    seo_title TEXT,
    seo_description TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_visible INTEGER NOT NULL DEFAULT 1,
    facet_config TEXT,                             -- JSON: filterable attributes for this category

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (catalog_id) REFERENCES catalogs(id),
    FOREIGN KEY (parent_category_id) REFERENCES catalog_categories(id),
    UNIQUE(tenant_id, catalog_id, category_code)
);

CREATE INDEX idx_categories_tenant_catalog ON catalog_categories(tenant_id, catalog_id);
CREATE INDEX idx_categories_tenant_parent ON catalog_categories(tenant_id, parent_category_id);
CREATE INDEX idx_categories_tenant_path ON catalog_categories(tenant_id, path);
CREATE INDEX idx_categories_tenant_level ON catalog_categories(tenant_id, level);
CREATE INDEX idx_categories_tenant_active ON catalog_categories(tenant_id, is_active);
```

### 2.4 Catalog Products
```sql
CREATE TABLE catalog_products (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    catalog_id TEXT NOT NULL,
    category_id TEXT NOT NULL,
    inventory_item_id TEXT NOT NULL,               -- FK to Inventory item
    sku TEXT NOT NULL,
    name TEXT NOT NULL,
    short_description TEXT,
    long_description TEXT,
    product_url TEXT,                              -- SEO-friendly URL slug
    primary_image_url TEXT,
    additional_images TEXT,                        -- JSON array: image URLs
    price_cents INTEGER NOT NULL,                  -- Display price
    list_price_cents INTEGER NOT NULL,             -- Original list price (for showing discounts)
    currency_code TEXT NOT NULL DEFAULT 'USD',
    is_featured INTEGER NOT NULL DEFAULT 0,
    is_new INTEGER NOT NULL DEFAULT 0,
    is_bestseller INTEGER NOT NULL DEFAULT 0,
    min_order_quantity INTEGER NOT NULL DEFAULT 1,
    max_order_quantity INTEGER NOT NULL DEFAULT 99999,
    availability TEXT NOT NULL DEFAULT 'IN_STOCK'
        CHECK(availability IN ('IN_STOCK','LIMITED','PREORDER','BACKORDER','OUT_OF_STOCK','DISCONTINUED')),
    attributes TEXT,                               -- JSON: product attributes (size, color, weight, etc.)
    variants TEXT,                                 -- JSON: variant definitions
    related_products TEXT,                         -- JSON array: related product IDs
    search_keywords TEXT,                          -- Full-text search keywords
    sort_order INTEGER NOT NULL DEFAULT 0,
    published_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (catalog_id) REFERENCES catalogs(id),
    FOREIGN KEY (category_id) REFERENCES catalog_categories(id),
    UNIQUE(tenant_id, catalog_id, sku)
);

CREATE INDEX idx_catalog_products_tenant_catalog ON catalog_products(tenant_id, catalog_id);
CREATE INDEX idx_catalog_products_tenant_category ON catalog_products(tenant_id, category_id);
CREATE INDEX idx_catalog_products_tenant_sku ON catalog_products(tenant_id, sku);
CREATE INDEX idx_catalog_products_tenant_availability ON catalog_products(tenant_id, availability);
CREATE INDEX idx_catalog_products_tenant_featured ON catalog_products(tenant_id, is_featured);
CREATE INDEX idx_catalog_products_tenant_price ON catalog_products(tenant_id, price_cents);
CREATE INDEX idx_catalog_products_tenant_active ON catalog_products(tenant_id, is_active);
```

### 2.5 Shopping Carts
```sql
CREATE TABLE shopping_carts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    customer_id TEXT,                              -- NULL for guest carts
    session_id TEXT NOT NULL,                      -- Browser session identifier
    cart_type TEXT NOT NULL DEFAULT 'SHOPPING'
        CHECK(cart_type IN ('SHOPPING','WISHLIST','QUICK_ORDER','REORDER')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    shipping_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    applied_promotions TEXT,                       -- JSON array: applied promotion codes
    coupon_code TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ABANDONED','CONVERTED','EXPIRED')),
    converted_order_id TEXT,
    abandoned_at TEXT,
    expires_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, session_id, cart_type)
);

CREATE INDEX idx_carts_tenant_customer ON shopping_carts(tenant_id, customer_id);
CREATE INDEX idx_carts_tenant_session ON shopping_carts(tenant_id, session_id);
CREATE INDEX idx_carts_tenant_status ON shopping_carts(tenant_id, status);
CREATE INDEX idx_carts_tenant_dates ON shopping_carts(tenant_id, created_at);
CREATE INDEX idx_carts_tenant_expires ON shopping_carts(tenant_id, expires_at);
```

### 2.6 Cart Items
```sql
CREATE TABLE cart_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cart_id TEXT NOT NULL,
    catalog_product_id TEXT NOT NULL,
    sku TEXT NOT NULL,
    product_name TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1,
    unit_price_cents INTEGER NOT NULL,
    list_price_cents INTEGER NOT NULL,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    subtotal_cents INTEGER NOT NULL,
    variant_selection TEXT,                        -- JSON: selected variant attributes
    configuration_json TEXT,                       -- JSON: product configuration (if configurable)
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (cart_id) REFERENCES shopping_carts(id) ON DELETE CASCADE
);

CREATE INDEX idx_cart_items_tenant_cart ON cart_items(tenant_id, cart_id);
CREATE INDEX idx_cart_items_tenant_product ON cart_items(tenant_id, catalog_product_id);
CREATE INDEX idx_cart_items_tenant_sku ON cart_items(tenant_id, sku);
```

### 2.7 Commerce Orders
```sql
CREATE TABLE commerce_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    order_number TEXT NOT NULL,                    -- ORD-2024-00001
    customer_id TEXT,
    guest_email TEXT,
    session_id TEXT NOT NULL,
    cart_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','CONFIRMED','PROCESSING','SHIPPED','DELIVERED',
                         'CANCELLED','REFUNDED','ON_HOLD')),
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    shipping_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    payment_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(payment_status IN ('PENDING','AUTHORIZED','CAPTURED','PARTIALLY_REFUNDED','REFUNDED','FAILED')),
    fulfillment_status TEXT NOT NULL DEFAULT 'UNFULFILLED'
        CHECK(fulfillment_status IN ('UNFULFILLED','PARTIALLY_FULFILLED','FULFILLED')),
    shipping_method TEXT,
    shipping_address TEXT,                         -- JSON: shipping address
    billing_address TEXT,                          -- JSON: billing address
    shipping_tracking TEXT,                        -- JSON array: tracking numbers
    notes TEXT,
    internal_notes TEXT,
    om_order_id TEXT,                              -- FK to Order Management order
    ip_address TEXT,
    user_agent TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_commerce_orders_tenant_customer ON commerce_orders(tenant_id, customer_id);
CREATE INDEX idx_commerce_orders_tenant_store ON commerce_orders(tenant_id, store_id);
CREATE INDEX idx_commerce_orders_tenant_status ON commerce_orders(tenant_id, status);
CREATE INDEX idx_commerce_orders_tenant_payment ON commerce_orders(tenant_id, payment_status);
CREATE INDEX idx_commerce_orders_tenant_dates ON commerce_orders(tenant_id, created_at);
CREATE INDEX idx_commerce_orders_tenant_active ON commerce_orders(tenant_id, is_active);
```

### 2.8 Commerce Payments
```sql
CREATE TABLE commerce_payments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_id TEXT NOT NULL,
    payment_number TEXT NOT NULL,                  -- CPAY-2024-00001
    payment_method TEXT NOT NULL
        CHECK(payment_method IN ('CREDIT_CARD','DEBIT_CARD','PAYPAL','BANK_TRANSFER',
                                 'INVOICE','WALLET','CRYPTO','COD')),
    amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','AUTHORIZED','CAPTURED','FAILED','VOIDED','REFUNDED')),
    gateway TEXT NOT NULL,                         -- Payment gateway identifier
    gateway_transaction_id TEXT,
    gateway_response TEXT,                         -- JSON: full gateway response
    card_last_four TEXT,
    card_brand TEXT,
    authorization_code TEXT,
    captured_at TEXT,
    refunded_at TEXT,
    failure_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (order_id) REFERENCES commerce_orders(id),
    UNIQUE(tenant_id, payment_number)
);

CREATE INDEX idx_commerce_payments_tenant_order ON commerce_payments(tenant_id, order_id);
CREATE INDEX idx_commerce_payments_tenant_status ON commerce_payments(tenant_id, status);
CREATE INDEX idx_commerce_payments_tenant_method ON commerce_payments(tenant_id, payment_method);
CREATE INDEX idx_commerce_payments_tenant_dates ON commerce_payments(tenant_id, created_at);
```

### 2.9 Commerce Promotions
```sql
CREATE TABLE commerce_promotions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    promotion_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    promotion_type TEXT NOT NULL
        CHECK(promotion_type IN ('PERCENTAGE_OFF','FIXED_AMOUNT_OFF','FREE_SHIPPING',
                                 'BUY_X_GET_Y','TIERED_DISCOUNT','BOGO','GIFT_WITH_PURCHASE')),
    discount_percent REAL,
    discount_amount_cents INTEGER,
    min_purchase_cents INTEGER NOT NULL DEFAULT 0,
    max_discount_cents INTEGER,                   -- Cap on discount amount
    applies_to TEXT NOT NULL DEFAULT 'ENTIRE_ORDER'
        CHECK(applies_to IN ('ENTIRE_ORDER','SPECIFIC_CATEGORIES','SPECIFIC_PRODUCTS','SHIPPING')),
    target_ids TEXT,                               -- JSON array: category or product IDs
    usage_limit INTEGER NOT NULL DEFAULT 0,        -- 0 = unlimited
    usage_count INTEGER NOT NULL DEFAULT 0,
    per_customer_limit INTEGER NOT NULL DEFAULT 0,
    effective_from TEXT NOT NULL,
    effective_to TEXT NOT NULL,
    is_auto_applied INTEGER NOT NULL DEFAULT 0,    -- Applied without code entry
    stackable INTEGER NOT NULL DEFAULT 0,          -- Can combine with other promotions
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','EXPIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, promotion_code)
);

CREATE INDEX idx_promotions_tenant_store ON commerce_promotions(tenant_id, store_id);
CREATE INDEX idx_promotions_tenant_type ON commerce_promotions(tenant_id, promotion_type);
CREATE INDEX idx_promotions_tenant_status ON commerce_promotions(tenant_id, status);
CREATE INDEX idx_promotions_tenant_dates ON commerce_promotions(tenant_id, effective_from, effective_to);
CREATE INDEX idx_promotions_tenant_active ON commerce_promotions(tenant_id, is_active);
```

### 2.10 Commerce Shipping Methods
```sql
CREATE TABLE commerce_shipping_methods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    method_name TEXT NOT NULL,
    method_code TEXT NOT NULL,
    carrier TEXT NOT NULL,
    carrier_service_code TEXT,
    zone TEXT,                                     -- Shipping zone definition
    base_rate_cents INTEGER NOT NULL DEFAULT 0,
    per_unit_rate_cents INTEGER NOT NULL DEFAULT 0,
    free_shipping_threshold_cents INTEGER,
    estimated_min_days INTEGER NOT NULL DEFAULT 1,
    estimated_max_days INTEGER NOT NULL DEFAULT 7,
    weight_limit_kg REAL,
    is_default INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (store_id) REFERENCES storefronts(id),
    UNIQUE(tenant_id, store_id, method_code)
);

CREATE INDEX idx_shipping_methods_tenant_store ON commerce_shipping_methods(tenant_id, store_id);
CREATE INDEX idx_shipping_methods_tenant_carrier ON commerce_shipping_methods(tenant_id, carrier);
CREATE INDEX idx_shipping_methods_tenant_active ON commerce_shipping_methods(tenant_id, is_active);
```

### 2.11 Commerce Reviews
```sql
CREATE TABLE commerce_reviews (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    store_id TEXT NOT NULL,
    catalog_product_id TEXT NOT NULL,
    customer_id TEXT,
    author_name TEXT NOT NULL,
    rating INTEGER NOT NULL
        CHECK(rating BETWEEN 1 AND 5),
    title TEXT,
    body TEXT,
    is_verified_purchase INTEGER NOT NULL DEFAULT 0,
    is_approved INTEGER NOT NULL DEFAULT 0,
    approved_by TEXT,
    approved_at TEXT,
    helpful_count INTEGER NOT NULL DEFAULT 0,
    response_text TEXT,                            -- Merchant response
    responded_at TEXT,
    responded_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (catalog_product_id) REFERENCES catalog_products(id)
);

CREATE INDEX idx_reviews_tenant_product ON commerce_reviews(tenant_id, catalog_product_id);
CREATE INDEX idx_reviews_tenant_customer ON commerce_reviews(tenant_id, customer_id);
CREATE INDEX idx_reviews_tenant_rating ON commerce_reviews(tenant_id, rating);
CREATE INDEX idx_reviews_tenant_approved ON commerce_reviews(tenant_id, is_approved);
CREATE INDEX idx_reviews_tenant_dates ON commerce_reviews(tenant_id, created_at);
```

### 2.12 Commerce Wishlists
```sql
CREATE TABLE commerce_wishlists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    catalog_product_id TEXT NOT NULL,
    sku TEXT NOT NULL,
    product_name TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1,
    variant_selection TEXT,                        -- JSON: selected variant
    priority INTEGER NOT NULL DEFAULT 0,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, customer_id, catalog_product_id, variant_selection)
);

CREATE INDEX idx_wishlists_tenant_customer ON commerce_wishlists(tenant_id, customer_id);
CREATE INDEX idx_wishlists_tenant_product ON commerce_wishlists(tenant_id, catalog_product_id);
CREATE INDEX idx_wishlists_tenant_dates ON commerce_wishlists(tenant_id, created_at);
```

---

## 3. REST API Endpoints

```
# Storefront Management
GET/POST       /api/v1/commerce/storefronts               Permission: commerce.storefronts.manage
GET/PUT        /api/v1/commerce/storefronts/{id}
PATCH          /api/v1/commerce/storefronts/{id}/status   Permission: commerce.storefronts.manage

# Catalog Management
GET/POST       /api/v1/commerce/catalogs                  Permission: commerce.catalogs.manage
GET/PUT        /api/v1/commerce/catalogs/{id}

# Category Management
GET/POST       /api/v1/commerce/catalogs/{id}/categories   Permission: commerce.catalogs.manage
GET/PUT        /api/v1/commerce/categories/{id}
DELETE         /api/v1/commerce/categories/{id}            Permission: commerce.catalogs.manage
GET            /api/v1/commerce/catalogs/{id}/categories/tree

# Product Management
GET/POST       /api/v1/commerce/catalogs/{id}/products     Permission: commerce.products.manage
GET/PUT        /api/v1/commerce/products/{id}
PATCH          /api/v1/commerce/products/{id}/status       Permission: commerce.products.manage
POST           /api/v1/commerce/products/bulk-upload       Permission: commerce.products.manage

# Product Browsing (Public)
GET            /api/v1/commerce/storefronts/{code}/browse
GET            /api/v1/commerce/storefronts/{code}/search
GET            /api/v1/commerce/storefronts/{code}/categories/{id}/products
GET            /api/v1/commerce/products/{id}/detail
GET            /api/v1/commerce/products/{id}/reviews
GET            /api/v1/commerce/products/{id}/related

# Cart Management
POST           /api/v1/commerce/carts                      Permission: commerce.cart
GET            /api/v1/commerce/carts/{id}
POST           /api/v1/commerce/carts/{id}/items           Permission: commerce.cart
PUT            /api/v1/commerce/carts/{id}/items/{item_id}
DELETE         /api/v1/commerce/carts/{id}/items/{item_id}
POST           /api/v1/commerce/carts/{id}/apply-promo     Permission: commerce.cart
DELETE         /api/v1/commerce/carts/{id}/promotions/{code}
GET            /api/v1/commerce/carts/{id}/shipping-options
POST           /api/v1/commerce/carts/{id}/estimate-shipping

# Checkout
POST           /api/v1/commerce/checkout                   Permission: commerce.checkout
POST           /api/v1/commerce/checkout/{cart_id}/payment  Permission: commerce.checkout
POST           /api/v1/commerce/checkout/{cart_id}/complete  Permission: commerce.checkout

# Order Management
GET            /api/v1/commerce/orders
GET            /api/v1/commerce/orders/{id}
GET            /api/v1/commerce/orders/{id}/tracking
POST           /api/v1/commerce/orders/{id}/cancel         Permission: commerce.orders.manage
GET            /api/v1/commerce/customers/{id}/orders

# Promotion Management
GET/POST       /api/v1/commerce/promotions                Permission: commerce.promotions.manage
GET/PUT        /api/v1/commerce/promotions/{id}
PATCH          /api/v1/commerce/promotions/{id}/status     Permission: commerce.promotions.manage
POST           /api/v1/commerce/promotions/{id}/validate   Permission: commerce.cart

# Shipping Methods
GET/POST       /api/v1/commerce/shipping-methods           Permission: commerce.shipping.manage
GET/PUT        /api/v1/commerce/shipping-methods/{id}
DELETE         /api/v1/commerce/shipping-methods/{id}      Permission: commerce.shipping.manage

# Reviews
POST           /api/v1/commerce/products/{id}/reviews      Permission: commerce.reviews.create
GET            /api/v1/commerce/reviews                     Permission: commerce.reviews.moderate
PATCH          /api/v1/commerce/reviews/{id}/approve        Permission: commerce.reviews.moderate
PATCH          /api/v1/commerce/reviews/{id}/reject         Permission: commerce.reviews.moderate
POST           /api/v1/commerce/reviews/{id}/respond        Permission: commerce.reviews.moderate

# Wishlist
GET            /api/v1/commerce/wishlists                   Permission: commerce.wishlist
POST           /api/v1/commerce/wishlists                   Permission: commerce.wishlist
DELETE         /api/v1/commerce/wishlists/{id}              Permission: commerce.wishlist
POST           /api/v1/commerce/wishlists/{id}/move-to-cart Permission: commerce.wishlist
```

---

## 4. Business Rules

### 4.1 Storefront & Catalog
```
Storefront management rules:
  1. Store code MUST be unique within a tenant
  2. A storefront MUST have at least one active catalog before going live
  3. Only one default catalog per storefront is allowed
  4. Category codes MUST be unique within a catalog
  5. Category paths use materialized path pattern for efficient tree queries
  6. Product SKU MUST be unique within a catalog
  7. Products MUST be assigned to exactly one primary category
  8. Catalog products linked to discontinued inventory items MUST be marked OUT_OF_STOCK
  9. Storefront in MAINTENANCE mode displays maintenance page to all visitors
  10. B2B storefronts require authenticated customer accounts; B2C allows guest checkout
```

### 4.2 Shopping Cart
- Carts expire after session_timeout_minutes (default: 30 minutes of inactivity)
- Maximum items per cart is enforced by storefront max_cart_items setting
- Cart totals are recalculated on every item change
- Inventory availability is checked when items are added and at checkout
- Out-of-stock items trigger a notification but remain in cart with updated availability
- Variant selection must be complete before item can be added to cart
- Quantity must respect product min_order_quantity and max_order_quantity
- Guest carts are merged into customer carts on login

### 4.3 Checkout & Payment
- Checkout requires valid shipping address, billing address, and shipping method selection
- Payment authorization MUST succeed before order confirmation
- Order numbers are auto-generated in format ORD-{YEAR}-{SEQ}
- Failed payment authorization does not create an order
- Authorized payments are captured on order confirmation
- Guest checkout requires email address for order confirmation communication
- B2B checkout may use invoice payment terms (PAYMENT_TERMS from customer account)

### 4.4 Promotions
- Promotion codes are case-insensitive
- Expired promotions (past effective_to) cannot be applied
- Usage limits are decremented atomically to prevent overselling
- Per-customer limits are tracked by customer_id (authenticated) or session_id (guest)
- Non-stackable promotions replace the current best discount
- Stackable promotions are additive up to max_discount_cents cap
- Auto-applied promotions are evaluated and applied without code entry
- Promotions targeting SPECIFIC_CATEGORIES or SPECIFIC_PRODUCTS check target_ids

### 4.5 Order Processing
- Commerce orders are forwarded to Order Management for fulfillment
- The om_order_id links the commerce order to the fulfillment order
- Order status is synced from Order Management fulfillment status
- Cancellation is allowed only for orders in PENDING or CONFIRMED status
- Refunds are processed through the payment gateway using stored transaction IDs
- Order confirmation emails are triggered on status CONFIRMED
- Shipping tracking updates are pulled from carrier integrations

### 4.6 Reviews & Ratings
- Reviews require a rating between 1 and 5
- Reviews with body text longer than 1000 characters are truncated for display
- Verified purchase status is set when reviewer has a completed order for the product
- Reviews are moderated: new reviews require approval before display (configurable per storefront)
- A customer can submit at most one review per product
- Merchant responses are visible alongside the review
- Average rating is recalculated on review approval/deletion

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `commerce.cart.created` | New cart created | CDP |
| `commerce.cart.updated` | Cart item added/removed | CDP |
| `commerce.checkout.initiated` | Checkout process started | CDP, PRICING |
| `commerce.order.placed` | Order confirmed and paid | OM, INV, CDP |
| `commerce.payment.processed` | Payment authorized/captured | AR, GL |
| `commerce.payment.refunded` | Payment refund processed | AR, GL |
| `commerce.promotion.applied` | Promotion code applied to cart | Reporting |
| `commerce.review.submitted` | Product review submitted | Notification |
| `commerce.wishlist.updated` | Wishlist item added/removed | CDP |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.commerce.v1;

service CommerceService {
    rpc BrowseCatalog(BrowseCatalogRequest) returns (BrowseCatalogResponse);
    rpc SearchProducts(SearchProductsRequest) returns (SearchProductsResponse);
    rpc GetProductDetail(GetProductDetailRequest) returns (GetProductDetailResponse);
    rpc ManageCart(ManageCartRequest) returns (ManageCartResponse);
    rpc Checkout(CheckoutRequest) returns (CheckoutResponse);
    rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
    rpc CheckAvailability(CheckAvailabilityRequest) returns (CheckAvailabilityResponse);
}

// Entity messages
message Storefront {
    string id = 1;
    string tenant_id = 2;
    string store_code = 3;
    string name = 4;
    string description = 5;
    string store_type = 6;
    string base_url = 7;
    string currency_code = 8;
    string language_code = 9;
    string timezone = 10;
    string theme_config = 11;
    string seo_config = 12;
    string legal_config = 13;
    int32 tax_inclusive = 14;
    int32 allow_guest_checkout = 15;
    int32 allow_wishlist = 16;
    int32 allow_reviews = 17;
    int32 max_cart_items = 18;
    int32 session_timeout_minutes = 19;
    string status = 20;
    string created_at = 21;
    string updated_at = 22;
}

message CatalogProduct {
    string id = 1;
    string tenant_id = 2;
    string catalog_id = 3;
    string category_id = 4;
    string inventory_item_id = 5;
    string sku = 6;
    string name = 7;
    string short_description = 8;
    string long_description = 9;
    string product_url = 10;
    string primary_image_url = 11;
    string additional_images = 12;
    int64 price_cents = 13;
    int64 list_price_cents = 14;
    string currency_code = 15;
    int32 is_featured = 16;
    int32 is_new = 17;
    int32 is_bestseller = 18;
    int32 min_order_quantity = 19;
    int32 max_order_quantity = 20;
    string availability_status = 21;
    string created_at = 22;
    string updated_at = 23;
}

message CartItem {
    string id = 1;
    string tenant_id = 2;
    string cart_id = 3;
    string product_id = 4;
    string sku = 5;
    string name = 6;
    int32 quantity = 7;
    int64 unit_price_cents = 8;
    int64 line_total_cents = 9;
    string image_url = 10;
    string created_at = 11;
    string updated_at = 12;
}

message CommerceOrder {
    string id = 1;
    string tenant_id = 2;
    string order_number = 3;
    string store_id = 4;
    string customer_id = 5;
    string status = 6;
    string currency_code = 7;
    int64 subtotal_cents = 8;
    int64 discount_cents = 9;
    int64 tax_cents = 10;
    int64 shipping_cents = 11;
    int64 total_cents = 12;
    string payment_method = 13;
    string payment_status = 14;
    string shipping_method = 15;
    string billing_address = 16;
    string shipping_address = 17;
    string promotion_code = 18;
    string created_at = 19;
    string updated_at = 20;
}

// Request/Response messages
message BrowseCatalogRequest {
    string tenant_id = 1;
    string store_id = 2;
    string category_id = 3;
    int32 page = 4;
    int32 page_size = 5;
    string sort_by = 6;
    string sort_order = 7;
}

message BrowseCatalogResponse {
    repeated CatalogProduct items = 1;
    int32 total_count = 2;
    string next_page_token = 3;
}

message SearchProductsRequest {
    string tenant_id = 1;
    string store_id = 2;
    string query = 3;
    string category_id = 4;
    int64 min_price_cents = 5;
    int64 max_price_cents = 6;
    int32 page = 7;
    int32 page_size = 8;
}

message SearchProductsResponse {
    repeated CatalogProduct items = 1;
    int32 total_count = 2;
    string next_page_token = 3;
}

message GetProductDetailRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetProductDetailResponse {
    CatalogProduct data = 1;
    int64 available_quantity = 2;
    string availability_status = 3;
}

message ManageCartRequest {
    string tenant_id = 1;
    string cart_id = 2;
    string action = 3;
    string product_id = 4;
    int32 quantity = 5;
}

message ManageCartResponse {
    string cart_id = 1;
    repeated CartItem items = 2;
    int64 subtotal_cents = 3;
    int64 total_cents = 4;
}

message CheckoutRequest {
    string tenant_id = 1;
    string cart_id = 2;
    string customer_id = 3;
    string payment_method = 4;
    string payment_token = 5;
    string shipping_method_id = 6;
    string billing_address = 7;
    string shipping_address = 8;
    string promotion_code = 9;
}

message CheckoutResponse {
    CommerceOrder order = 1;
    string payment_status = 2;
}

message GetOrderRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetOrderResponse {
    CommerceOrder data = 1;
}

message CheckAvailabilityRequest {
    string tenant_id = 1;
    string inventory_item_id = 2;
    int32 quantity = 3;
}

message CheckAvailabilityResponse {
    string inventory_item_id = 1;
    int64 available_quantity = 2;
    string availability_status = 3;
    string available_date = 4;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **INV (Inventory):** Real-time product availability, stock levels, ATP dates, inventory item master data
- **PRICING (Advanced Pricing):** Price resolution for catalog products, customer-specific pricing, volume discounts
- **CDP (Customer Data Platform):** Unified customer profiles, purchase history, segment membership
- **Auth:** Customer authentication, session management, role-based access for B2B accounts
- **Document:** Product image storage, document attachments

### 6.2 Data Published To
- **OM (Order Management):** Sales order creation from commerce orders, fulfillment requests, order status sync
- **INV (Inventory):** Inventory reservation on checkout, availability queries, stock decrement
- **PRICING (Advanced Pricing):** Promotional price calculations, coupon validation
- **AR (Accounts Receivable):** Invoice generation for B2B orders, payment recording
- **GL (General Ledger):** Revenue journal entries, tax entries, discount entries
- **CDP (Customer Data Platform):** Browse behavior, cart activity, purchase events, wishlist activity
- **Notification:** Order confirmation, shipping updates, review moderation alerts
- **Reporting:** Sales analytics, conversion funnels, promotion effectiveness

---

## 7. Migrations

1. V001: `storefronts`
2. V002: `catalogs`
3. V003: `catalog_categories`
4. V004: `catalog_products`
5. V005: `shopping_carts`
6. V006: `cart_items`
7. V007: `commerce_orders`
8. V008: `commerce_payments`
9. V009: `commerce_promotions`
10. V010: `commerce_shipping_methods`
11. V011: `commerce_reviews`
12. V012: `commerce_wishlists`
13. V013: Triggers for `updated_at`
