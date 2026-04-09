# 145 - Shipping Execution Service Specification

## 1. Domain Overview

Shipping Execution manages outbound shipment planning, execution, confirmation, and tracking. Supports multi-order consolidations, carrier selection, rate shopping, shipping documentation (BOL, packing slips, commercial invoices), parcel and freight shipping, and real-time tracking integration. Enables pick-wave planning, packing workflows, carrier manifesting, and proof of delivery capture. Integrates with Order Management for shipment requests, Warehouse Management for pick/pack, Transportation Management for carrier routing, and Inventory for allocation and reservation.

**Bounded Context:** Outbound Shipping & Fulfillment Execution
**Service Name:** `shipping-exec-service`
**Database:** `data/shipping_exec.db`
**HTTP Port:** 8225 | **gRPC Port:** 9225

---

## 2. Database Schema

### 2.1 Shipments
```sql
CREATE TABLE se_shipments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_number TEXT NOT NULL,          -- SHP-2024-00001
    shipment_type TEXT NOT NULL CHECK(shipment_type IN ('PARCEL','LTL','FTL','AIR','OCEAN','RAIL','MULTIMODE')),
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','PICKING','PACKED','READY_TO_SHIP','SHIPPED','IN_TRANSIT',
                         'DELIVERED','PARTIALLY_DELIVERED','EXCEPTION','CANCELLED')),
    warehouse_id TEXT NOT NULL,
    dock_door_id TEXT,
    carrier_id TEXT NOT NULL,
    carrier_service_level TEXT NOT NULL,
    shipping_method TEXT NOT NULL CHECK(shipping_method IN ('STANDARD','EXPRESS','OVERNIGHT','GROUND','FREIGHT','CUSTOM')),
    ship_to_address TEXT NOT NULL,          -- JSON: full address
    ship_from_address TEXT NOT NULL,        -- JSON: full address
    total_weight_kg DECIMAL(10,2) NOT NULL DEFAULT 0,
    total_volume_cbm DECIMAL(10,4) NOT NULL DEFAULT 0,
    total_packages INTEGER NOT NULL DEFAULT 0,
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    planned_ship_date TEXT NOT NULL,
    actual_ship_date TEXT,
    estimated_delivery_date TEXT,
    actual_delivery_date TEXT,
    tracking_number TEXT,
    tracking_url TEXT,
    pro_number TEXT,                        -- Carrier PRO number
    bol_number TEXT,                        -- Bill of Lading number
    master_tracking TEXT,                   -- For multi-package shipments
    hazmat_flag INTEGER NOT NULL DEFAULT 0,
    hazmat_class TEXT,
    temperature_required REAL,              -- Cold chain: required temp in Celsius
    signature_required INTEGER NOT NULL DEFAULT 0,
    insurance_value_cents INTEGER NOT NULL DEFAULT 0,
    special_instructions TEXT,
    consolidation_group_id TEXT,            -- For consolidated shipments

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, shipment_number)
);

CREATE INDEX idx_se_shipment_tenant ON se_shipments(tenant_id, status);
CREATE INDEX idx_se_shipment_tracking ON se_shipments(tracking_number);
CREATE INDEX idx_se_shipment_date ON se_shipments(tenant_id, planned_ship_date);
CREATE INDEX idx_se_shipment_carrier ON se_shipments(tenant_id, carrier_id);
```

### 2.2 Shipment Lines
```sql
CREATE TABLE se_shipment_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    order_id TEXT NOT NULL,
    order_line_id TEXT NOT NULL,
    order_number TEXT NOT NULL,
    item_id TEXT NOT NULL,
    item_description TEXT,
    requested_quantity DECIMAL(18,4) NOT NULL,
    shipped_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    uom TEXT NOT NULL,
    lot_number TEXT,
    serial_numbers TEXT,                    -- JSON array for serialized items
    weight_kg DECIMAL(10,2),
    package_id TEXT,                        -- Reference to package
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PICKED','PACKED','SHIPPED','DELIVERED','SHORT_SHIPPED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES se_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_se_line_shipment ON se_shipment_lines(shipment_id);
CREATE INDEX idx_se_line_order ON se_shipment_lines(tenant_id, order_id);
```

### 2.3 Packages
```sql
CREATE TABLE se_packages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    package_number TEXT NOT NULL,           -- PKG-001
    package_type TEXT NOT NULL CHECK(package_type IN ('BOX','PALLET','CRATE','ENVELOPE','TOTE','DRUM','CUSTOM')),
    package_dimensions TEXT NOT NULL,       -- JSON: { "length": 30, "width": 20, "height": 15, "unit": "cm" }
    weight_kg DECIMAL(10,2) NOT NULL DEFAULT 0,
    tracking_number TEXT,
    carrier_label_url TEXT,
    customs_value_cents INTEGER NOT NULL DEFAULT 0,
    hazmat_contents INTEGER NOT NULL DEFAULT 0,
    contents_description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES se_shipments(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, shipment_id, package_number)
);
```

### 2.4 Carrier Rate Quotes
```sql
CREATE TABLE se_carrier_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    carrier_id TEXT NOT NULL,
    carrier_name TEXT NOT NULL,
    service_level TEXT NOT NULL,
    origin_zip TEXT NOT NULL,
    destination_zip TEXT NOT NULL,
    weight_kg DECIMAL(10,2) NOT NULL,
    package_type TEXT NOT NULL,
    rate_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    estimated_transit_days INTEGER NOT NULL,
    estimated_delivery_date TEXT,
    rate_valid_until TEXT,
    surcharges TEXT,                        -- JSON: fuel, residential, etc.
    total_cost_cents INTEGER NOT NULL,
    selected INTEGER NOT NULL DEFAULT 0,
    quote_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, carrier_id, service_level, origin_zip, destination_zip, weight_kg)
);

CREATE INDEX idx_se_rate_tenant ON se_carrier_rates(tenant_id, total_cost_cents);
```

### 2.5 Shipping Documents
```sql
CREATE TABLE se_shipping_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN (
        'BILL_OF_LADING','PACKING_SLIP','COMMERCIAL_INVOICE','CUSTOMS_DECLARATION',
        'CERTIFICATE_OF_ORIGIN','SHIPPER_LETTER','LABEL','PROOF_OF_DELIVERY','OTHER'
    )),
    document_number TEXT,
    storage_path TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type TEXT NOT NULL DEFAULT 'application/pdf',
    generated_at TEXT NOT NULL DEFAULT (datetime('now')),
    generated_by TEXT NOT NULL,
    is_void INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES se_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_se_doc_shipment ON se_shipping_documents(shipment_id, document_type);
```

### 2.6 Tracking Events
```sql
CREATE TABLE se_tracking_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    tracking_number TEXT NOT NULL,
    event_timestamp TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN (
        'PICKUP','IN_TRANSIT','ARRIVAL_AT_FACILITY','DEPARTURE_FROM_FACILITY',
        'OUT_FOR_DELIVERY','DELIVERED','EXCEPTION','CUSTOMS_HOLD','ATTEMPTED_DELIVERY','RETURNED'
    )),
    event_location TEXT,                    -- City, State, Country
    event_description TEXT,
    carrier_event_code TEXT,
    latitude REAL,
    longitude REAL,
    signed_by TEXT,                         -- For delivery events
    proof_of_delivery_url TEXT,

    received_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES se_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_se_track_shipment ON se_tracking_events(shipment_id, event_timestamp DESC);
CREATE INDEX idx_se_track_number ON se_tracking_events(tracking_number, event_timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Shipments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/shipping-exec/v1/shipments` | Create shipment |
| GET | `/shipping-exec/v1/shipments` | List shipments |
| GET | `/shipping-exec/v1/shipments/{id}` | Get shipment details |
| PUT | `/shipping-exec/v1/shipments/{id}` | Update shipment |
| POST | `/shipping-exec/v1/shipments/{id}/confirm` | Confirm shipment |
| POST | `/shipping-exec/v1/shipments/{id}/void` | Void shipment |
| GET | `/shipping-exec/v1/shipments/{id}/tracking` | Get tracking details |
| GET | `/shipping-exec/v1/shipments/{id}/documents` | Get shipping documents |
| POST | `/shipping-exec/v1/shipments/{id}/re-rate` | Re-rate shipment |

### 3.2 Rate Shopping
| Method | Path | Description |
|--------|------|-------------|
| POST | `/shipping-exec/v1/rate-quotes` | Get carrier rate quotes |
| GET | `/shipping-exec/v1/rate-quotes/{id}` | Get rate quote details |
| POST | `/shipping-exec/v1/rate-quotes/compare` | Compare rates across carriers |
| POST | `/shipping-exec/v1/rate-quotes/{id}/select` | Select rate quote |

### 3.3 Packages & Labels
| Method | Path | Description |
|--------|------|-------------|
| POST | `/shipping-exec/v1/shipments/{id}/packages` | Add package |
| GET | `/shipping-exec/v1/shipments/{id}/packages` | List packages |
| POST | `/shipping-exec/v1/shipments/{id}/labels` | Generate shipping labels |
| POST | `/shipping-exec/v1/shipments/{id}/manifest` | Generate carrier manifest |

### 3.4 Pick Waves
| Method | Path | Description |
|--------|------|-------------|
| POST | `/shipping-exec/v1/pick-waves` | Create pick wave |
| GET | `/shipping-exec/v1/pick-waves` | List pick waves |
| POST | `/shipping-exec/v1/pick-waves/{id}/release` | Release to warehouse |
| POST | `/shipping-exec/v1/pick-waves/{id}/confirm` | Confirm pick completion |

### 3.5 Tracking & Documents
| Method | Path | Description |
|--------|------|-------------|
| GET | `/shipping-exec/v1/tracking/{trackingNumber}` | Track by tracking number |
| POST | `/shipping-exec/v1/tracking/webhook` | Receive carrier tracking updates |
| POST | `/shipping-exec/v1/shipments/{id}/documents/generate` | Generate documents |
| POST | `/shipping-exec/v1/shipments/{id}/pod` | Upload proof of delivery |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `shipping.shipment.created` | `{ shipment_id, order_ids, carrier }` | Shipment created |
| `shipping.shipment.picked` | `{ shipment_id, warehouse_id }` | Pick completed |
| `shipping.shipment.shipped` | `{ shipment_id, tracking_number, carrier }` | Shipment dispatched |
| `shipping.tracking.update` | `{ shipment_id, event_type, location }` | Tracking event received |
| `shipping.shipment.delivered` | `{ shipment_id, delivered_at, signed_by }` | Delivery confirmed |
| `shipping.shipment.exception` | `{ shipment_id, exception_type }` | Shipping exception |
| `shipping.rate.quoted` | `{ carrier_id, service_level, cost_cents }` | Rate quote returned |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `order.released_for_fulfillment` | Order Management | Create shipment |
| `inventory.picked` | Warehouse Management | Update shipment status |
| `tms.route.assigned` | Transportation Mgmt | Set carrier and route |
| `return.authorized` | Order Management | Create return shipment |

---


---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.shipping.v1;

service ShippingExecutionService {
    rpc GetShipment(GetShipmentRequest) returns (GetShipmentResponse);
    rpc RateShipment(RateShipmentRequest) returns (RateShipmentResponse);
    rpc ConfirmShipment(ConfirmShipmentRequest) returns (ConfirmShipmentResponse);
    rpc GetTrackingStatus(GetTrackingStatusRequest) returns (GetTrackingStatusResponse);
}

message Shipment { string id = 1; string tenant_id = 2; string shipment_number = 3; string carrier = 4; string tracking_number = 5; string status = 6; string shipped_date = 7; string estimated_delivery = 8; string created_at = 9; string updated_at = 10; }
message CarrierRate { string id = 1; string tenant_id = 2; string carrier = 3; string service_level = 4; int64 rate_cents = 5; string currency_code = 6; int32 transit_days = 7; }
message TrackingEvent { string id = 1; string tenant_id = 2; string shipment_id = 3; string status = 4; string location = 5; string event_date = 6; string description = 7; string created_at = 8; }

message GetShipmentRequest { string tenant_id = 1; string id = 2; }
message GetShipmentResponse { Shipment data = 1; }
message RateShipmentRequest { string tenant_id = 1; string shipment_id = 2; string carrier = 3; }
message RateShipmentResponse { repeated CarrierRate rates = 1; }
message ConfirmShipmentRequest { string tenant_id = 1; string id = 2; string carrier_id = 3; }
message ConfirmShipmentResponse { string tracking_number = 1; string label_url = 2; }
message GetTrackingStatusRequest { string tenant_id = 1; string shipment_id = 2; }
message GetTrackingStatusResponse { repeated TrackingEvent events = 1; string status = 2; }
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | se_shipments | — |
| V002 | se_shipment_lines | V001 |
| V003 | se_packages | V002 |
| V004 | se_carrier_rates | V003 |
| V005 | se_shipping_documents | V004 |
| V006 | se_tracking_events | V005 |

---

## 7. Business Rules

1. **Carrier Selection**: Auto-select lowest cost carrier meeting delivery date; manual override allowed
2. **Consolidation**: Orders to same destination within 4-hour window eligible for consolidation
3. **Label Generation**: Labels auto-generated upon carrier selection; support ZPL and PDF
4. **Tracking Refresh**: Carrier tracking polled every 2 hours for active shipments
5. **Proof of Delivery**: Required for shipments >$500 value or hazmat items
6. **Rate Validity**: Rate quotes expire after 24 hours; auto re-quote if expired
7. **Hazmat Handling**: Hazmat shipments require additional documentation and carrier certification
8. **Cold Chain**: Temperature-monitored shipments generate alerts on deviation >2 degrees

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Order Management (13) | Shipment requests, order fulfillment |
| Warehouse Management (36) | Pick/pack execution |
| Transportation Management (37) | Carrier routing, freight management |
| Inventory (12) | Allocation, reservation, on-hand checks |
| Global Trade (39) | Customs documentation, compliance |
| Advanced Pricing (27) | Shipping cost calculation |
| Billing/AR (08) | Shipping charge invoicing |
| Document Management (29) | Document storage |
