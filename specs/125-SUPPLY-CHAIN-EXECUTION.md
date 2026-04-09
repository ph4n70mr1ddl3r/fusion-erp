# 125 - Supply Chain Execution Specification

## 1. Domain Overview

Supply Chain Execution provides end-to-end logistics orchestration, managing the physical flow of goods from warehouse to customer. It coordinates shipment planning, carrier selection, freight execution, proof of delivery, and logistics visibility across the entire supply chain network.

**Bounded Context:** Supply Chain Execution & Logistics Orchestration
**Service Name:** `scexec-service`
**Database:** `data/scexec.db`
**HTTP Port:** 8205 | **gRPC Port:** 9205

---

## 2. Database Schema

### 2.1 Shipments
```sql
CREATE TABLE sce_shipments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_number TEXT NOT NULL,
    shipment_type TEXT NOT NULL CHECK(shipment_type IN ('OUTBOUND','INBOUND','TRANSFER','THIRD_PARTY')),
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','CONFIRMED','PICKED','PACKED','SHIPPED','IN_TRANSIT','DELIVERED','CANCELLED','EXCEPTION')),
    origin_location_id TEXT NOT NULL,
    destination_location_id TEXT NOT NULL,
    carrier_id TEXT,
    carrier_service_level TEXT,
    mode_of_transport TEXT CHECK(mode_of_transport IN ('GROUND','AIR','OCEAN','RAIL','MULTIMODAL')),
    planned_ship_date TEXT NOT NULL,
    actual_ship_date TEXT,
    planned_delivery_date TEXT NOT NULL,
    actual_delivery_date TEXT,
    total_weight_kg REAL,
    total_volume_cbm REAL,
    total_package_count INTEGER NOT NULL DEFAULT 0,
    total_value_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    freight_cost_cents INTEGER,
    tracking_number TEXT,
    tracking_url TEXT,
    reference_type TEXT,                 -- 'SALES_ORDER', 'PURCHASE_ORDER', 'TRANSFER_ORDER'
    reference_id TEXT,
    driver_name TEXT,
    vehicle_id TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, shipment_number)
);

CREATE INDEX idx_sce_shipments_tenant_status ON sce_shipments(tenant_id, status);
CREATE INDEX idx_sce_shipments_tenant_dates ON sce_shipments(tenant_id, planned_ship_date, planned_delivery_date);
CREATE INDEX idx_sce_shipments_tenant_tracking ON sce_shipments(tenant_id, tracking_number);
```

### 2.2 Shipment Lines
```sql
CREATE TABLE sce_shipment_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT NOT NULL,
    item_description TEXT,
    ordered_quantity REAL NOT NULL,
    shipped_quantity REAL NOT NULL DEFAULT 0,
    received_quantity REAL NOT NULL DEFAULT 0,
    uom TEXT NOT NULL,
    lot_number TEXT,
    serial_numbers TEXT,                 -- JSON array for serialized items
    weight_kg REAL,
    volume_cbm REAL,
    package_count INTEGER NOT NULL DEFAULT 1,
    reference_line_id TEXT,              -- Link to order line
    source_location_id TEXT,
    destination_location_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (shipment_id) REFERENCES sce_shipments(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, shipment_id, line_number)
);
```

### 2.3 Shipment Stops
```sql
CREATE TABLE sce_shipment_stops (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    shipment_id TEXT NOT NULL,
    stop_number INTEGER NOT NULL,
    stop_type TEXT NOT NULL CHECK(stop_type IN ('PICKUP','DELIVERY','TRANSFER','HUB')),
    location_id TEXT NOT NULL,
    planned_arrival TEXT NOT NULL,
    planned_departure TEXT NOT NULL,
    actual_arrival TEXT,
    actual_departure TEXT,
    stop_status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(stop_status IN ('PLANNED','ARRIVED','DEPARTED','SKIPPED')),
    special_instructions TEXT,
    contact_name TEXT,
    contact_phone TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (shipment_id) REFERENCES sce_shipments(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, shipment_id, stop_number)
);
```

### 2.4 Tracking Events
```sql
CREATE TABLE sce_tracking_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN ('PICKED_UP','IN_TRANSIT','AT_HUB','OUT_FOR_DELIVERY','DELIVERED','EXCEPTION','DELAYED')),
    event_timestamp TEXT NOT NULL,
    location_description TEXT,
    latitude REAL,
    longitude REAL,
    carrier_event_code TEXT,
    notes TEXT,
    proof_document_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES sce_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_sce_tracking_tenant_shipment ON sce_tracking_events(tenant_id, shipment_id, event_timestamp DESC);
CREATE INDEX idx_sce_tracking_tenant_type ON sce_tracking_events(tenant_id, event_type);
```

### 2.5 Proof of Delivery
```sql
CREATE TABLE sce_proof_of_delivery (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    delivery_date TEXT NOT NULL,
    received_by_name TEXT NOT NULL,
    received_by_signature TEXT,          -- Base64 signature image
    received_by_title TEXT,
    condition_at_delivery TEXT NOT NULL DEFAULT 'GOOD'
        CHECK(condition_at_delivery IN ('GOOD','DAMAGED','PARTIAL','REFUSED')),
    damage_notes TEXT,
    damage_photos TEXT,                  -- JSON array of photo document IDs
    pod_document_id TEXT,                -- Signed POD document
    geo_location TEXT,                   -- "lat,lng" at delivery
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    FOREIGN KEY (shipment_id) REFERENCES sce_shipments(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, shipment_id)
);
```

### 2.6 Logistics Network
```sql
CREATE TABLE sce_network_nodes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    node_code TEXT NOT NULL,
    node_name TEXT NOT NULL,
    node_type TEXT NOT NULL CHECK(node_type IN ('WAREHOUSE','DISTRIBUTION_CENTER','STORE','SUPPLIER','CUSTOMER','PORT','HUB')),
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT NOT NULL,
    latitude REAL,
    longitude REAL,
    timezone TEXT,
    operating_hours TEXT,                -- JSON: { "mon": {"open": "08:00", "close": "17:00"}, ... }
    contact_name TEXT,
    contact_phone TEXT,
    contact_email TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, node_code)
);
```

---

## 3. REST API Endpoints

### 3.1 Shipments
```
GET    /api/v1/scexec/shipments                          Permission: sce.shipments.read
GET    /api/v1/scexec/shipments/{id}                     Permission: sce.shipments.read
POST   /api/v1/scexec/shipments                          Permission: sce.shipments.create
PUT    /api/v1/scexec/shipments/{id}                     Permission: sce.shipments.update
POST   /api/v1/scexec/shipments/{id}/confirm             Permission: sce.shipments.execute
POST   /api/v1/scexec/shipments/{id}/ship                Permission: sce.shipments.execute
POST   /api/v1/scexec/shipments/{id}/cancel              Permission: sce.shipments.cancel
GET    /api/v1/scexec/shipments/{id}/lines                Permission: sce.shipments.read
POST   /api/v1/scexec/shipments/{id}/lines               Permission: sce.shipments.update
```

### 3.2 Tracking
```
GET    /api/v1/scexec/shipments/{id}/tracking            Permission: sce.tracking.read
POST   /api/v1/scexec/shipments/{id}/tracking/events     Permission: sce.tracking.update
GET    /api/v1/scexec/tracking/search?tracking_number=xx  Permission: sce.tracking.read
GET    /api/v1/scexec/shipments/{id}/stops               Permission: sce.shipments.read
PUT    /api/v1/scexec/shipments/{id}/stops/{stop_id}     Permission: sce.shipments.execute
```

### 3.3 Proof of Delivery
```
POST   /api/v1/scexec/shipments/{id}/pod                 Permission: sce.pod.create
GET    /api/v1/scexec/shipments/{id}/pod                 Permission: sce.pod.read
PUT    /api/v1/scexec/shipments/{id}/pod                 Permission: sce.pod.update
```

### 3.4 Logistics Network
```
GET    /api/v1/scexec/network/nodes                      Permission: sce.network.read
GET    /api/v1/scexec/network/nodes/{id}                 Permission: sce.network.read
POST   /api/v1/scexec/network/nodes                      Permission: sce.network.create
PUT    /api/v1/scexec/network/nodes/{id}                 Permission: sce.network.update
GET    /api/v1/scexec/network/routes
  ?origin={id}&destination={id}                          Permission: sce.network.read
```

### 3.5 Shipment Planning
```
POST   /api/v1/scexec/plan/consolidate                    Permission: sce.planning.create
  Request: { "order_ids": [...], "strategy": "MAXIMIZE_UTILIZATION" }
  Response: Planned shipments with optimal consolidation
POST   /api/v1/scexec/plan/carrier-select                 Permission: sce.planning.create
  Request: { "shipment_id": "...", "criteria": ["COST","DELIVERY_TIME"] }
GET    /api/v1/scexec/dashboard                           Permission: sce.dashboard.read
  Response: Active shipments, exceptions, on-time delivery %, avg transit time
```

---

## 4. Business Rules

### 4.1 Shipment Lifecycle
- PLANNED → CONFIRMED → PICKED → PACKED → SHIPPED → IN_TRANSIT → DELIVERED
- Shipments can enter EXCEPTION status at any point after CONFIRMED
- Cancellation only allowed before SHIPPED status
- Actual dates MUST be recorded when status transitions occur
- Shipment becomes immutable after DELIVERED

### 4.2 Consolidation Rules
- Multiple orders to same destination can be consolidated into one shipment
- Consolidation strategies: maximize container utilization, minimize shipments, meet delivery windows
- Weight/volume limits enforced per carrier service level
- Hazardous materials cannot be consolidated with general cargo

### 4.3 Tracking & Visibility
- Carrier tracking events ingested via API or EDI
- Real-time location updates for in-transit shipments
- Exception alerts for delays, damages, refusals
- Customer-facing tracking portal with filtered events
- Estimated delivery date recalculated on each tracking event

### 4.4 Proof of Delivery
- POD required for all outbound shipments
- Digital signature capture on mobile devices
- Photo evidence required for damaged deliveries
- Delivery geolocation captured and stored
- Partial delivery: shipped_quantity > received_quantity triggers exception

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.scexec.v1;

service SupplyChainExecutionService {
    rpc CreateShipment(CreateShipmentRequest) returns (CreateShipmentResponse);
    rpc UpdateShipmentStatus(UpdateShipmentStatusRequest) returns (UpdateShipmentStatusResponse);
    rpc GetTrackingInfo(GetTrackingInfoRequest) returns (GetTrackingInfoResponse);
    rpc RecordTrackingEvent(RecordTrackingEventRequest) returns (RecordTrackingEventResponse);
    rpc RecordDelivery(RecordDeliveryRequest) returns (RecordDeliveryResponse);
    rpc ConsolidateShipments(ConsolidateShipmentsRequest) returns (ConsolidateShipmentsResponse);
    rpc GetShipmentByReference(GetShipmentByReferenceRequest) returns (GetShipmentByReferenceResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **om-service**: Sales order fulfillment triggers outbound shipments
- **wms-service**: Pick/pack confirmation, warehouse operations
- **tms-service**: Carrier management, freight rates, routing
- **inv-service**: Inventory reservations, stock movements
- **inv-service**: Location/warehouse master data
- **dms-service**: POD documents, photos, signatures

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `sce.shipment.created` | Shipment planned | shipment_id, type, origin, destination |
| `sce.shipment.shipped` | Shipment dispatched | shipment_id, carrier_id, tracking_number |
| `sce.shipment.in_transit` | In-transit update | shipment_id, location, eta |
| `sce.shipment.delivered` | Delivery confirmed | shipment_id, pod_id, delivered_date |
| `sce.shipment.exception` | Exception reported | shipment_id, exception_type, details |
| `sce.tracking.updated` | New tracking event | shipment_id, event_type, location |
| `sce.pod.recorded` | POD captured | shipment_id, condition, received_by |

---

## 7. Migrations

### Migration Order for scexec-service:
1. V001: `sce_network_nodes`
2. V002: `sce_shipments`
3. V003: `sce_shipment_lines`
4. V004: `sce_shipment_stops`
5. V005: `sce_tracking_events`
6. V006: `sce_proof_of_delivery`
7. V007: Triggers for `updated_at`
8. V008: Seed data (sample network nodes, carrier service levels)
