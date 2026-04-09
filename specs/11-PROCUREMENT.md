# 11 - Procurement Service Specification

## 1. Domain Overview

Procurement manages requisitions, purchase orders, supplier catalogs, goods receipt, and three-way matching. Integrates with AP for invoicing, Inventory for receiving, and GL for accounting entries.

**Bounded Context:** Procurement & Purchasing
**Service Name:** `proc-service`
**Database:** `data/proc.db`
**HTTP Port:** 8020 | **gRPC Port:** 9020

---

## 2. Database Schema

### 2.1 Requisitions
```sql
CREATE TABLE requisitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requisition_number TEXT NOT NULL,       -- REQ-2024-00001
    requester_id TEXT NOT NULL,
    description TEXT NOT NULL,
    required_date TEXT,
    priority TEXT DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED','CANCELLED','CONVERTED')),
    justification TEXT,
    department TEXT,
    project_id TEXT,
    total_estimated_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, requisition_number)
);

CREATE TABLE requisition_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requisition_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT,
    item_description TEXT NOT NULL,
    quantity REAL NOT NULL,
    unit_of_measure TEXT NOT NULL,
    unit_price_cents INTEGER,
    estimated_price_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    required_date TEXT,
    suggested_supplier_id TEXT,
    notes TEXT,

    -- Reference after conversion
    purchase_order_line_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (requisition_id) REFERENCES requisitions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, requisition_id, line_number)
);
```

### 2.2 Purchase Orders
```sql
CREATE TABLE purchase_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    po_number TEXT NOT NULL,               -- PO-2024-00001
    supplier_id TEXT NOT NULL,
    requisition_id TEXT,
    order_date TEXT NOT NULL,
    promised_date TEXT,
    ship_to_location_id TEXT,
    bill_to_location_id TEXT,
    payment_terms_id TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    exchange_rate REAL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','ISSUED','PARTIALLY_RECEIVED','RECEIVED','CLOSED','CANCELLED')),

    description TEXT,
    notes TEXT,
    shipping_method TEXT,
    freight_terms TEXT,

    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    freight_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    line_count INTEGER NOT NULL DEFAULT 0,

    -- Approval
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    -- supplier_id is a logical reference to ap-service suppliers(id), validated via gRPC
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_purchase_orders_tenant_supplier ON purchase_orders(tenant_id, supplier_id);
CREATE INDEX idx_purchase_orders_tenant_status ON purchase_orders(tenant_id, status);
CREATE INDEX idx_purchase_orders_tenant_date ON purchase_orders(tenant_id, order_date);

CREATE TABLE purchase_order_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    purchase_order_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT,
    item_description TEXT NOT NULL,
    supplier_part_number TEXT,
    quantity_ordered REAL NOT NULL,
    quantity_received REAL NOT NULL DEFAULT 0,
    quantity_invoiced REAL NOT NULL DEFAULT 0,
    quantity_cancelled REAL NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    unit_price_cents INTEGER NOT NULL,
    line_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_code TEXT,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    promised_date TEXT,
    expense_account_id TEXT,
    destination_type TEXT DEFAULT 'EXPENSE'
        CHECK(destination_type IN ('EXPENSE','INVENTORY','ASSET')),

    -- References
    requisition_line_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (purchase_order_id) REFERENCES purchase_orders(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, purchase_order_id, line_number)
);

CREATE INDEX idx_po_lines_tenant_po ON purchase_order_lines(tenant_id, purchase_order_id);
CREATE INDEX idx_po_lines_tenant_item ON purchase_order_lines(tenant_id, item_id);
```

### 2.3 Goods Receipts
```sql
CREATE TABLE goods_receipts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    receipt_number TEXT NOT NULL,          -- GR-2024-00001
    purchase_order_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    receipt_date TEXT NOT NULL,
    received_by TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','CONFIRMED','INSPECTED','CANCELLED')),
    notes TEXT,
    inspection_required INTEGER NOT NULL DEFAULT 0,
    inspection_status TEXT CHECK(inspection_status IN ('PENDING','PASSED','FAILED','PARTIAL')),
    inspection_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, receipt_number)
);

CREATE TABLE goods_receipt_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    goods_receipt_id TEXT NOT NULL,
    purchase_order_line_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT,
    item_description TEXT NOT NULL,
    quantity_received REAL NOT NULL,
    quantity_accepted REAL NOT NULL DEFAULT 0,
    quantity_rejected REAL NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    lot_number TEXT,
    serial_numbers TEXT,                   -- JSON array
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (goods_receipt_id) REFERENCES goods_receipts(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, goods_receipt_id, line_number)
);
```

### 2.4 Blanket Agreements
```sql
CREATE TABLE blanket_agreements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_number TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    description TEXT NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    total_amount_cents INTEGER NOT NULL,
    released_amount_cents INTEGER NOT NULL DEFAULT 0,
    remaining_amount_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, agreement_number)
);
```

---

## 3. REST API Endpoints

```
# Requisitions
GET/POST      /api/v1/proc/requisitions                Permission: proc.requisitions.read/create
GET/PUT       /api/v1/proc/requisitions/{id}            Permission: proc.requisitions.read/update
POST          /api/v1/proc/requisitions/{id}/submit     Permission: proc.requisitions.create
POST          /api/v1/proc/requisitions/{id}/approve    Permission: proc.requisitions.approve
POST          /api/v1/proc/requisitions/{id}/reject     Permission: proc.requisitions.approve
POST          /api/v1/proc/requisitions/{id}/cancel     Permission: proc.requisitions.update
GET/POST      /api/v1/proc/requisitions/{id}/lines      Permission: proc.requisitions.read/create

# Purchase Orders
GET/POST      /api/v1/proc/purchase-orders              Permission: proc.purchase-orders.read/create
GET/PUT       /api/v1/proc/purchase-orders/{id}          Permission: proc.purchase-orders.read/update
POST          /api/v1/proc/purchase-orders/{id}/submit   Permission: proc.purchase-orders.create
POST          /api/v1/proc/purchase-orders/{id}/approve  Permission: proc.purchase-orders.approve
POST          /api/v1/proc/purchase-orders/{id}/issue    Permission: proc.purchase-orders.update
POST          /api/v1/proc/purchase-orders/{id}/cancel   Permission: proc.purchase-orders.update
POST          /api/v1/proc/purchase-orders/{id}/close    Permission: proc.purchase-orders.update
GET/POST      /api/v1/proc/purchase-orders/{id}/lines    Permission: proc.purchase-orders.read/create
POST          /api/v1/proc/requisitions/{id}/convert-to-po  Permission: proc.purchase-orders.create

# Goods Receipts
GET/POST      /api/v1/proc/goods-receipts               Permission: proc.goods-receipts.read/create
GET           /api/v1/proc/goods-receipts/{id}           Permission: proc.goods-receipts.read
POST          /api/v1/proc/goods-receipts/{id}/confirm   Permission: proc.goods-receipts.update
POST          /api/v1/proc/goods-receipts/{id}/inspect   Permission: proc.goods-receipts.update

# Blanket Agreements
GET/POST      /api/v1/proc/blanket-agreements           Permission: proc.agreements.read/create
GET/PUT       /api/v1/proc/blanket-agreements/{id}      Permission: proc.agreements.read/update
```

---

## 4. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.proc.v1;

service ProcurementService {
    rpc GetRequisition(GetRequisitionRequest) returns (GetRequisitionResponse);
    rpc CreateRequisition(CreateRequisitionRequest) returns (CreateRequisitionResponse);
    rpc GetPurchaseOrder(GetPurchaseOrderRequest) returns (GetPurchaseOrderResponse);
    rpc CreateGoodsReceipt(CreateGoodsReceiptRequest) returns (CreateGoodsReceiptResponse);
    rpc GetSupplierBalance(GetSupplierBalanceRequest) returns (GetSupplierBalanceResponse);
}

message Requisition { string id = 1; string tenant_id = 2; string requisition_number = 3; string description = 4; string status = 5; string requester_id = 6; int64 total_cents = 7; string created_at = 8; string updated_at = 9; }
message PurchaseOrder { string id = 1; string tenant_id = 2; string po_number = 3; string supplier_id = 4; string status = 5; int64 total_cents = 6; string currency_code = 7; string created_at = 8; string updated_at = 9; }
message GoodsReceipt { string id = 1; string tenant_id = 2; string receipt_number = 3; string po_id = 4; string receipt_date = 5; string status = 6; string created_at = 7; string updated_at = 8; }

message GetRequisitionRequest { string tenant_id = 1; string id = 2; }
message GetRequisitionResponse { Requisition data = 1; }
message CreateRequisitionRequest { string tenant_id = 1; string description = 2; string requester_id = 3; }
message CreateRequisitionResponse { Requisition data = 1; }
message GetPurchaseOrderRequest { string tenant_id = 1; string id = 2; }
message GetPurchaseOrderResponse { PurchaseOrder data = 1; }
message CreateGoodsReceiptRequest { string tenant_id = 1; string po_id = 2; string receipt_date = 3; }
message CreateGoodsReceiptResponse { GoodsReceipt data = 1; }
message GetSupplierBalanceRequest { string tenant_id = 1; string supplier_id = 2; }
message GetSupplierBalanceResponse { int64 outstanding_cents = 1; int64 overdue_cents = 2; }
```

## 5. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | requisitions | — |
| V002 | requisition_lines | V001 |
| V003 | purchase_orders | V002 |
| V004 | purchase_order_lines | V003 |
| V005 | goods_receipts | V004 |
| V006 | goods_receipt_lines | V005 |
| V007 | blanket_agreements | V006 |

---

## 6. Business Rules

### 7.1 Requisition to PO Flow
1. Employee creates requisition → DRAFT
2. Submit for approval → SUBMITTED
3. Approver reviews → APPROVED or REJECTED
4. Buyer converts approved requisition to PO → requisition status = CONVERTED
5. PO lines reference original requisition lines

### 7.2 PO to Goods Receipt
1. PO issued to supplier → ISSUED
2. On goods receipt: update PO line quantity_received
3. PO status transitions: PARTIALLY_RECEIVED (some lines) or RECEIVED (all lines)
4. Cannot receive more than quantity_ordered (configurable over-receipt tolerance)

### 7.3 Three-Way Match (PO → GR → Invoice)
1. AP invoice references PO and GR
2. Match check: PO quantity ordered ≥ GR quantity received ≥ Invoice quantity invoiced
3. Match check: PO unit price = Invoice unit price (within tolerance)
4. Status: UNMATCHED → 2_WAY (PO+Invoice) → 3_WAY (PO+GR+Invoice)

### 7.4 GL Integration
- PO itself does NOT create GL entries (encumbrance optional)
- Goods Receipt → GL: Debit Inventory/Expense, Credit Goods Received Not Invoiced (GRNI)
- Invoice matching → GL: Debit GRNI, Credit AP

### 7.5 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `proc.requisition.approved` | Requisition approved | — |
| `proc.po.issued` | PO issued to supplier | — |
| `proc.po.received` | Goods receipt confirmed | AP, INV |
| `proc.goods.received` | Individual line received | INV (update stock) |
