# 13 - Order Management Service Specification

## 1. Domain Overview

Order Management handles sales orders from quote to fulfillment: order entry, pricing, allocation, shipping, and returns. Integrates with AR for invoicing, Inventory for stock allocation/fulfillment, and GL for revenue recognition.

**Bounded Context:** Sales Order Management
**Service Name:** `om-service`
**Database:** `data/om.db`
**HTTP Port:** 8022 | **gRPC Port:** 9022

---

## 2. Database Schema

### 2.1 Sales Orders (Header)
```sql
CREATE TABLE sales_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_number TEXT NOT NULL,            -- SO-2024-00001
    customer_id TEXT NOT NULL,
    customer_po_number TEXT,
    order_date TEXT NOT NULL,
    requested_ship_date TEXT,
    promised_ship_date TEXT,
    actual_ship_date TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    payment_terms_id TEXT,
    price_list_id TEXT,
    sales_rep_id TEXT,

    -- Addresses
    bill_to_address_id TEXT,
    ship_to_address_id TEXT,
    ship_method TEXT,
    shipping_terms TEXT,                   -- FOB, CIF, etc.
    freight_carrier TEXT,
    tracking_number TEXT,

    description TEXT,
    notes TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','CONFIRMED','ALLOCATED','PICKING','PACKED','SHIPPED','INVOICED','COMPLETED','CANCELLED','ON_HOLD')),

    -- Totals
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    freight_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    cost_of_goods_cents INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,

    -- Source
    source_type TEXT DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','QUOTATION','EDI','WEB','PHONE')),
    source_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_sales_orders_tenant_customer ON sales_orders(tenant_id, customer_id);
CREATE INDEX idx_sales_orders_tenant_status ON sales_orders(tenant_id, status);
CREATE INDEX idx_sales_orders_tenant_date ON sales_orders(tenant_id, order_date);
CREATE INDEX idx_sales_orders_tenant_ship_date ON sales_orders(tenant_id, requested_ship_date);
```

### 2.2 Sales Order Lines
```sql
CREATE TABLE sales_order_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sales_order_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    line_type TEXT NOT NULL DEFAULT 'ITEM'
        CHECK(line_type IN ('ITEM','SERVICE','CHARGE','FREIGHT','DISCOUNT')),

    item_id TEXT,
    item_description TEXT NOT NULL,
    quantity_ordered REAL NOT NULL,
    quantity_allocated REAL NOT NULL DEFAULT 0,
    quantity_picked REAL NOT NULL DEFAULT 0,
    quantity_packed REAL NOT NULL DEFAULT 0,
    quantity_shipped REAL NOT NULL DEFAULT 0,
    quantity_invoiced REAL NOT NULL DEFAULT 0,
    quantity_returned REAL NOT NULL DEFAULT 0,
    quantity_cancelled REAL NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    unit_price_cents INTEGER NOT NULL,
    unit_cost_cents INTEGER,
    discount_percent REAL DEFAULT 0,
    discount_amount_cents INTEGER NOT NULL DEFAULT 0,
    line_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_code TEXT,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    -- Fulfillment
    warehouse_id TEXT,
    ship_from_location_id TEXT,
    promised_ship_date TEXT,
    delivery_date TEXT,

    -- Reference to invoice after invoicing
    invoice_line_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (sales_order_id) REFERENCES sales_orders(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, sales_order_id, line_number)
);

CREATE INDEX idx_so_lines_tenant_order ON sales_order_lines(tenant_id, sales_order_id);
CREATE INDEX idx_so_lines_tenant_item ON sales_order_lines(tenant_id, item_id);
```

### 2.3 Shipments
```sql
CREATE TABLE shipments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_number TEXT NOT NULL,         -- SHP-2024-00001
    sales_order_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    shipment_date TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    carrier TEXT,
    tracking_number TEXT,
    shipping_method TEXT,
    packages_count INTEGER DEFAULT 1,
    weight_kg REAL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PICKED','PACKED','SHIPPED','DELIVERED','CANCELLED')),
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, shipment_number)
);

CREATE TABLE shipment_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    order_line_id TEXT NOT NULL,
    item_id TEXT,
    quantity_shipped REAL NOT NULL,
    lot_number TEXT,
    serial_number TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES shipments(id) ON DELETE RESTRICT
);
```

### 2.4 Returns (RMA)
```sql
CREATE TABLE returns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    return_number TEXT NOT NULL,           -- RMA-2024-00001
    sales_order_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    return_date TEXT NOT NULL,
    return_type TEXT NOT NULL
        CHECK(return_type IN ('REFUND','REPLACEMENT','REPAIR','CREDIT')),
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'REQUESTED'
        CHECK(status IN ('REQUESTED','APPROVED','RECEIVED','INSPECTED','PROCESSED','CANCELLED')),
    refund_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    credit_memo_id TEXT,                   -- Link to AR credit memo

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, return_number)
);

CREATE TABLE return_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    return_id TEXT NOT NULL,
    order_line_id TEXT NOT NULL,
    item_id TEXT,
    quantity_returned REAL NOT NULL,
    condition TEXT DEFAULT 'GOOD'
        CHECK(condition IN ('GOOD','DAMAGED','DEFECTIVE','WRONG_ITEM')),
    disposition TEXT DEFAULT 'RETURN_TO_STOCK'
        CHECK(disposition IN ('RETURN_TO_STOCK','SCRAP','REPAIR','VENDOR_RETURN')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (return_id) REFERENCES returns(id) ON DELETE RESTRICT
);
```

---

## 3. REST API Endpoints

```
# Sales Orders
GET/POST      /api/v1/om/orders                      Permission: om.orders.read/create
GET/PUT       /api/v1/om/orders/{id}                  Permission: om.orders.read/update
POST          /api/v1/om/orders/{id}/submit           Permission: om.orders.create
POST          /api/v1/om/orders/{id}/approve          Permission: om.orders.approve
POST          /api/v1/om/orders/{id}/confirm          Permission: om.orders.update
POST          /api/v1/om/orders/{id}/allocate         Permission: om.orders.update
POST          /api/v1/om/orders/{id}/cancel           Permission: om.orders.update
POST          /api/v1/om/orders/{id}/hold             Permission: om.orders.update
GET/POST      /api/v1/om/orders/{id}/lines            Permission: om.orders.read/create

# Shipments
GET/POST      /api/v1/om/shipments                    Permission: om.shipments.read/create
GET           /api/v1/om/shipments/{id}                Permission: om.shipments.read
POST          /api/v1/om/shipments/{id}/pick           Permission: om.shipments.update
POST          /api/v1/om/shipments/{id}/pack           Permission: om.shipments.update
POST          /api/v1/om/shipments/{id}/ship           Permission: om.shipments.update
POST          /api/v1/om/shipments/{id}/deliver        Permission: om.shipments.update

# Invoicing
POST          /api/v1/om/orders/{id}/invoice          Permission: om.orders.update

# Returns
GET/POST      /api/v1/om/returns                      Permission: om.returns.read/create
GET           /api/v1/om/returns/{id}                  Permission: om.returns.read
POST          /api/v1/om/returns/{id}/approve          Permission: om.returns.approve
POST          /api/v1/om/returns/{id}/receive          Permission: om.returns.update
POST          /api/v1/om/returns/{id}/process          Permission: om.returns.update

# Reports
GET           /api/v1/om/reports/sales-by-customer    Permission: om.reports.view
GET           /api/v1/om/reports/sales-by-item         Permission: om.reports.view
GET           /api/v1/om/reports/backlog                Permission: om.reports.view
```

---

## 4. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.om.v1;

service OrderManagementService {
    rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
    rpc FulfillOrder(FulfillOrderRequest) returns (FulfillOrderResponse);
    rpc GetOrderStatus(GetOrderStatusRequest) returns (GetOrderStatusResponse);
}

message SalesOrder { string id = 1; string tenant_id = 2; string order_number = 3; string customer_id = 4; string order_date = 5; string status = 6; int64 subtotal_cents = 7; int64 tax_cents = 8; int64 total_cents = 9; string currency_code = 10; string created_at = 11; string updated_at = 12; }
message Shipment { string id = 1; string tenant_id = 2; string shipment_number = 3; string order_id = 4; string carrier = 5; string tracking_number = 6; string status = 7; string shipped_date = 8; string created_at = 9; string updated_at = 10; }

message GetOrderRequest { string tenant_id = 1; string id = 2; }
message GetOrderResponse { SalesOrder data = 1; }
message CreateOrderRequest { string tenant_id = 1; string customer_id = 2; string order_date = 3; }
message CreateOrderResponse { SalesOrder data = 1; }
message FulfillOrderRequest { string tenant_id = 1; string order_id = 2; string warehouse_id = 3; }
message FulfillOrderResponse { Shipment data = 1; }
message GetOrderStatusRequest { string tenant_id = 1; string id = 2; }
message GetOrderStatusResponse { string order_id = 1; string status = 2; string tracking_number = 3; }
```

## 5. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | sales_orders | — |
| V002 | sales_order_lines | V001 |
| V003 | shipments | V002 |
| V004 | shipment_lines | V003 |
| V005 | returns | V004 |
| V006 | return_lines | V005 |

---

## 6. Business Rules

### 7.1 Order Processing Flow
```
DRAFT → SUBMITTED → APPROVED → CONFIRMED → ALLOCATED → PICKING → PACKED → SHIPPED → INVOICED → COMPLETED
                                                                                               ↕
                                                                                          CANCELLED / ON_HOLD
```

### 7.2 Stock Allocation
- On confirm: check inventory availability via INV gRPC
- Reserve stock for the order (update on_hand_quantities.quantity_reserved)
- Backorder if insufficient stock (partial allocation allowed)

### 7.3 Invoicing
- On ship: eligible for invoicing
- Create invoice via AR gRPC: Debit AR, Credit Revenue
- Can invoice partial shipment

### 7.4 GL Integration
- **Shipment:** Debit COGS, Credit Inventory (via INV service)
- **Invoice:** Debit AR, Credit Revenue (via AR service)
- **Return:** Reverse COGS and revenue entries

### 7.5 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `om.order.created` | Order created | — |
| `om.order.confirmed` | Order confirmed | INV (reserve stock) |
| `om.order.shipped` | Order shipped | AR (invoice), INV (issue stock) |
| `om.order.invoiced` | Order invoiced | GL |
| `om.return.approved` | Return approved | — |
| `om.return.received` | Return received | INV (return to stock), AR (credit memo) |
