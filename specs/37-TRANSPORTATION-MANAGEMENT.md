# 37 - Transportation Management System (TMS) Service Specification

## 1. Domain Overview

Transportation Management provides carrier management, shipment planning and execution, load optimization, rate shopping, freight audit and payment, track and trace, and delivery scheduling. Integrates with Order Management for outbound shipments, Inventory for warehouse shipping/receiving, Procurement for inbound shipments, and General Ledger for freight cost accounting.

**Bounded Context:** Transportation, Logistics & Fleet Management
**Service Name:** `tms-service`
**Database:** `data/tms.db`
**HTTP Port:** 8064 | **gRPC Port:** 9064

---

## 2. Database Schema

### 2.1 Carriers
```sql
CREATE TABLE tms_carriers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    carrier_code TEXT NOT NULL,
    carrier_name TEXT NOT NULL,
    carrier_type TEXT NOT NULL DEFAULT 'FTL'
        CHECK(carrier_type IN ('PARCEL','LTL','FTL','AIR','OCEAN','RAIL')),
    mc_number TEXT,
    dot_number TEXT,
    scac_code TEXT,
    contact_name TEXT,
    contact_email TEXT,
    contact_phone TEXT,
    rating INTEGER DEFAULT 3 CHECK(rating BETWEEN 1 AND 5),
    is_active INTEGER NOT NULL DEFAULT 1,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, carrier_code)
);

CREATE INDEX idx_tms_carriers_tenant ON tms_carriers(tenant_id);
CREATE INDEX idx_tms_carriers_tenant_type ON tms_carriers(tenant_id, carrier_type);
```

### 2.2 Carrier Contracts
```sql
CREATE TABLE tms_carrier_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    carrier_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,
    service_level TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(service_level IN ('STANDARD','EXPRESS','NEXT_DAY','ECONOMY')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    rate_structure TEXT NOT NULL,            -- JSON: rate rules and conditions
    transit_days_min INTEGER,
    transit_days_max INTEGER,
    capacity_weight_kg REAL,
    capacity_volume_cubic REAL,
    terms TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (carrier_id) REFERENCES tms_carriers(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_tms_contracts_tenant_carrier ON tms_carrier_contracts(tenant_id, carrier_id);
CREATE INDEX idx_tms_contracts_tenant_status ON tms_carrier_contracts(tenant_id, status);
```

### 2.3 Carrier Rates
```sql
CREATE TABLE tms_carrier_rates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    carrier_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    origin_zone TEXT NOT NULL,
    destination_zone TEXT NOT NULL,
    rate_type TEXT NOT NULL DEFAULT 'FLAT'
        CHECK(rate_type IN ('FLAT','PER_KG','PER_CUBIC','PER_MILE','ZONE_BASED')),
    rate_amount_cents INTEGER NOT NULL DEFAULT 0,
    minimum_charge_cents INTEGER NOT NULL DEFAULT 0,
    fuel_surcharge_percent REAL NOT NULL DEFAULT 0,
    accessorial_charges TEXT,                -- JSON: additional charge types and amounts
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (carrier_id) REFERENCES tms_carriers(id) ON DELETE RESTRICT,
    FOREIGN KEY (contract_id) REFERENCES tms_carrier_contracts(id) ON DELETE RESTRICT
);

CREATE INDEX idx_tms_rates_tenant_carrier ON tms_carrier_rates(tenant_id, carrier_id);
CREATE INDEX idx_tms_rates_tenant_zones ON tms_carrier_rates(tenant_id, origin_zone, destination_zone);
CREATE INDEX idx_tms_rates_tenant_dates ON tms_carrier_rates(tenant_id, effective_from, effective_to);
```

### 2.4 Shipments
```sql
CREATE TABLE tms_shipments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_number TEXT NOT NULL,           -- SHP-2024-00001
    shipment_type TEXT NOT NULL DEFAULT 'OUTBOUND'
        CHECK(shipment_type IN ('OUTBOUND','INBOUND','TRANSFER','THIRD_PARTY')),
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','BOOKED','IN_TRANSIT','DELIVERED','EXCEPTION')),
    origin_address_id TEXT NOT NULL,
    destination_address_id TEXT NOT NULL,
    carrier_id TEXT,
    carrier_service TEXT,
    tracking_number TEXT,
    pro_number TEXT,
    total_weight_kg REAL NOT NULL DEFAULT 0,
    total_volume_cubic REAL NOT NULL DEFAULT 0,
    package_count INTEGER NOT NULL DEFAULT 0,
    freight_class TEXT,
    hazmat_classification TEXT,
    ship_date TEXT,
    estimated_delivery TEXT,
    actual_delivery TEXT,
    freight_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, shipment_number)
);

CREATE INDEX idx_tms_shipments_tenant_status ON tms_shipments(tenant_id, status);
CREATE INDEX idx_tms_shipments_tenant_type ON tms_shipments(tenant_id, shipment_type);
CREATE INDEX idx_tms_shipments_tenant_carrier ON tms_shipments(tenant_id, carrier_id);
CREATE INDEX idx_tms_shipments_tenant_dates ON tms_shipments(tenant_id, ship_date, estimated_delivery);
CREATE INDEX idx_tms_shipments_tracking ON tms_shipments(tracking_number);
```

### 2.5 Shipment Lines
```sql
CREATE TABLE tms_shipment_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('SALES_ORDER','TRANSFER_ORDER','PURCHASE_ORDER')),
    source_id TEXT NOT NULL,
    source_line_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    weight_kg REAL NOT NULL DEFAULT 0,
    volume_cubic REAL NOT NULL DEFAULT 0,
    lot_number TEXT,
    serial_number TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES tms_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_tms_shipment_lines_tenant_shipment ON tms_shipment_lines(tenant_id, shipment_id);
CREATE INDEX idx_tms_shipment_lines_tenant_source ON tms_shipment_lines(tenant_id, source_type, source_id);
```

### 2.6 Load Plans
```sql
CREATE TABLE tms_load_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','OPTIMIZING','CONFIRMED')),
    total_weight_kg REAL NOT NULL DEFAULT 0,
    total_volume_cubic REAL NOT NULL DEFAULT 0,
    shipment_ids TEXT NOT NULL,              -- JSON: array of shipment IDs
    equipment_type TEXT NOT NULL DEFAULT 'VAN'
        CHECK(equipment_type IN ('VAN','FLATBED','REEFER','CONTAINER')),
    stops TEXT NOT NULL,                     -- JSON: ordered sequence of stop objects
    estimated_cost_cents INTEGER NOT NULL DEFAULT 0,
    optimization_score REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_name)
);

CREATE INDEX idx_tms_load_plans_tenant_status ON tms_load_plans(tenant_id, status);
```

### 2.7 Routes
```sql
CREATE TABLE tms_routes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    route_name TEXT NOT NULL,
    origin_address_id TEXT NOT NULL,
    stops TEXT NOT NULL,                     -- JSON: array of stop addresses
    total_distance_km REAL NOT NULL DEFAULT 0,
    total_time_hours REAL NOT NULL DEFAULT 0,
    optimized_sequence TEXT,                 -- JSON: optimized stop order
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, route_name)
);

CREATE INDEX idx_tms_routes_tenant ON tms_routes(tenant_id);
```

### 2.8 Freight Invoices
```sql
CREATE TABLE tms_freight_invoices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    carrier_id TEXT NOT NULL,
    invoice_number TEXT NOT NULL,
    invoice_date TEXT NOT NULL,
    invoiced_amount_cents INTEGER NOT NULL DEFAULT 0,
    expected_amount_cents INTEGER NOT NULL DEFAULT 0,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    variance_reason TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING_REVIEW'
        CHECK(status IN ('PENDING_REVIEW','APPROVED','DISPUTED','PAID')),
    gl_account_id TEXT,
    payment_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (shipment_id) REFERENCES tms_shipments(id) ON DELETE RESTRICT,
    FOREIGN KEY (carrier_id) REFERENCES tms_carriers(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_tms_freight_invoices_tenant_status ON tms_freight_invoices(tenant_id, status);
CREATE INDEX idx_tms_freight_invoices_tenant_carrier ON tms_freight_invoices(tenant_id, carrier_id);
CREATE INDEX idx_tms_freight_invoices_tenant_shipment ON tms_freight_invoices(tenant_id, shipment_id);
```

### 2.9 Tracking Events
```sql
CREATE TABLE tms_tracking_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    event_type TEXT NOT NULL
        CHECK(event_type IN ('PICKUP','DEPARTURE','ARRIVAL','DELIVERY','EXCEPTION')),
    event_location TEXT,
    event_timestamp TEXT NOT NULL,
    description TEXT,
    latitude REAL,
    longitude REAL,
    carrier_event_code TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES tms_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_tms_tracking_tenant_shipment ON tms_tracking_events(tenant_id, shipment_id);
CREATE INDEX idx_tms_tracking_tenant_timestamp ON tms_tracking_events(tenant_id, event_timestamp);
CREATE INDEX idx_tms_tracking_tenant_type ON tms_tracking_events(tenant_id, event_type);
```

### 2.10 Delivery Windows
```sql
CREATE TABLE tms_delivery_windows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    shipment_id TEXT NOT NULL,
    window_start TEXT NOT NULL,
    window_end TEXT NOT NULL,
    appointment_type TEXT NOT NULL DEFAULT 'DELIVERY'
        CHECK(appointment_type IN ('DELIVERY','PICKUP')),
    is_confirmed INTEGER NOT NULL DEFAULT 0,
    confirmed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (shipment_id) REFERENCES tms_shipments(id) ON DELETE CASCADE
);

CREATE INDEX idx_tms_delivery_windows_tenant_shipment ON tms_delivery_windows(tenant_id, shipment_id);
CREATE INDEX idx_tms_delivery_windows_tenant_dates ON tms_delivery_windows(tenant_id, window_start, window_end);
```

---

## 3. REST API Endpoints

```
# Carrier Management
GET/POST      /api/v1/tms/carriers                          Permission: tms.carriers.read/create
GET/PUT       /api/v1/tms/carriers/{id}                     Permission: tms.carriers.read/update
GET/POST      /api/v1/tms/carriers/{id}/contracts           Permission: tms.contracts.read/create
GET/POST      /api/v1/tms/carriers/{id}/rates               Permission: tms.rates.read/create

# Shipment Management
GET/POST      /api/v1/tms/shipments                         Permission: tms.shipments.read/create
GET/PUT       /api/v1/tms/shipments/{id}                    Permission: tms.shipments.read/update
GET           /api/v1/tms/shipments/{id}/tracking           Permission: tms.tracking.read
POST          /api/v1/tms/shipments/{id}/book               Permission: tms.shipments.update
POST          /api/v1/tms/shipments/{id}/cancel             Permission: tms.shipments.update
GET/POST      /api/v1/tms/shipments/{id}/lines              Permission: tms.shipments.read/update

# Load Planning
GET/POST      /api/v1/tms/load-plans                        Permission: tms.loads.read/create
GET/PUT       /api/v1/tms/load-plans/{id}                   Permission: tms.loads.read/update
POST          /api/v1/tms/load-plans/{id}/optimize          Permission: tms.loads.update
POST          /api/v1/tms/load-plans/{id}/confirm           Permission: tms.loads.update

# Rate Shopping
POST          /api/v1/tms/rate-shop                         Permission: tms.rates.shop

# Freight Audit
GET/POST      /api/v1/tms/freight-invoices                  Permission: tms.freight.read/create
GET           /api/v1/tms/freight-invoices/{id}             Permission: tms.freight.read
POST          /api/v1/tms/freight-invoices/{id}/approve     Permission: tms.freight.update
POST          /api/v1/tms/freight-invoices/{id}/dispute     Permission: tms.freight.update

# Tracking
GET           /api/v1/tms/tracking/{tracking_number}        Permission: tms.tracking.read
POST          /api/v1/tms/tracking/events                   Permission: tms.tracking.update
GET           /api/v1/tms/tracking/shipments-in-transit     Permission: tms.tracking.read

# Delivery Scheduling
GET/POST      /api/v1/tms/delivery-windows                  Permission: tms.delivery.read/create
POST          /api/v1/tms/delivery-windows/{id}/confirm     Permission: tms.delivery.update

# Reports
GET           /api/v1/tms/reports/carrier-performance       Permission: tms.reports.view
GET           /api/v1/tms/reports/freight-cost-analysis     Permission: tms.reports.view
GET           /api/v1/tms/reports/on-time-delivery          Permission: tms.reports.view
GET           /api/v1/tms/reports/lane-analysis             Permission: tms.reports.view
```

---

## 4. Business Rules

### 4.1 Shipment Lifecycle
```
PLANNED → BOOKED → IN_TRANSIT → DELIVERED
                     ↘ EXCEPTION → (resolved) → DELIVERED
                     ↘ EXCEPTION → (cancelled)
```
1. **PLANNED:** Shipment created with origin, destination, and contents. Carrier not yet assigned.
2. **BOOKED:** Carrier selected via rate shopping, tracking number assigned.
3. **IN_TRANSIT:** First tracking event (PICKUP or DEPARTURE) received.
4. **DELIVERED:** Delivery tracking event confirmed.
5. **EXCEPTION:** Delay, damage, or other issue; requires manual resolution.

### 4.2 Carrier Selection and Rate Shopping
- Carrier selection MUST use rate shopping to compare carriers on cost, transit time, and rating
- Rate shopping returns top 3 carrier options sorted by best value score
- Best value score = weighted combination: 40% cost, 30% transit time, 20% carrier rating, 10% on-time history
- Only carriers with active contracts and matching service levels are considered

### 4.3 Load Optimization
- Load optimization MUST consider weight limits, volume constraints, and delivery windows
- Shipments are consolidated by geographic proximity and compatible delivery windows
- Equipment type selection based on total weight, volume, and hazmat requirements
- Optimization score measures how efficiently the load utilizes equipment capacity

### 4.4 Freight Audit and Payment
- Freight invoices MUST be matched against expected charges; variances > 5% flagged for review
- Variance calculation: `variance_cents = invoiced_amount_cents - expected_amount_cents`
- Invoices with variance <= 5% can be auto-approved if tenant has auto-approval enabled
- Disputed invoices require carrier resolution before payment
- Approved invoices generate GL journal entries for freight cost allocation

### 4.5 Tracking and Tracing
- Tracking events are updated by carrier integration and in-app scanning
- DELIVERY event triggers shipment status change to DELIVERED
- EXCEPTION events trigger alerts to the shipment owner and customer service
- Location data (latitude/longitude) stored for route visualization

### 4.6 Hazardous Materials
- Hazardous materials MUST be declared at shipment creation
- Hazmat shipments MUST use carriers with certified hazmat handling
- Hazmat classification must conform to UN/USDOT hazard classes
- Additional documentation and surcharges apply automatically

### 4.7 Delivery Confirmation Integration
- Delivery confirmation triggers inventory and order management updates
- OUTBOUND delivery: updates sales order fulfillment, triggers customer notification
- INBOUND delivery: updates purchase order receipt, triggers inventory put-away
- TRANSFER delivery: updates both origin (ship) and destination (receive) warehouses

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `tms.shipment.booked` | Carrier assigned to shipment | OM |
| `tms.shipment.in_transit` | First tracking event received | OM |
| `tms.shipment.delivered` | Delivery confirmed | OM, INV, AP |
| `tms.shipment.exception` | Exception tracking event | Notifications |
| `tms.freight.audit.variance` | Invoice variance exceeds threshold | GL |
| `tms.load.optimized` | Load plan optimization complete | — |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.tms.v1;

service TmsService {
    rpc RateShop(RateShopRequest) returns (RateShopResponse);
    rpc BookShipment(BookShipmentRequest) returns (BookShipmentResponse);
    rpc UpdateTracking(UpdateTrackingRequest) returns (UpdateTrackingResponse);
    rpc GetShipmentCost(GetShipmentCostRequest) returns (GetShipmentCostResponse);
}

message RateShopRequest {
    string tenant_id = 1;
    string origin_address_id = 2;
    string destination_address_id = 3;
    repeated ShipmentLineItem lines = 4;
    string ship_date = 5;
    string service_level = 6;
}

message ShipmentLineItem {
    string item_id = 1;
    int32 quantity = 2;
    double weight_kg = 3;
    double volume_cubic = 4;
    bool is_hazmat = 5;
}

message CarrierOption {
    string carrier_id = 1;
    string carrier_name = 2;
    string service_level = 3;
    int64 total_cost_cents = 4;
    int32 transit_days = 5;
    double best_value_score = 6;
}

message RateShopResponse {
    repeated CarrierOption options = 1;
    string currency_code = 2;
}

message BookShipmentRequest {
    string tenant_id = 1;
    string shipment_id = 2;
    string carrier_id = 3;
    string carrier_service = 4;
    string booked_by = 5;
}

message BookShipmentResponse {
    string shipment_id = 1;
    string tracking_number = 2;
    string status = 3;
}

message TrackingEvent {
    string event_type = 1;
    string timestamp = 2;
    string location = 3;
    string description = 4;
    double latitude = 5;
    double longitude = 6;
}

message UpdateTrackingRequest {
    string tenant_id = 1;
    string tracking_number = 2;
    TrackingEvent event = 3;
}

message UpdateTrackingResponse {
    string shipment_id = 1;
    string status = 2;
}

message GetShipmentCostRequest {
    string tenant_id = 1;
    string shipment_id = 2;
}

message ShipmentCostBreakdown {
    int64 freight_cost_cents = 1;
    int64 fuel_surcharge_cents = 2;
    int64 accessorials_cents = 3;
    int64 total_cost_cents = 4;
    string currency_code = 5;
}

message GetShipmentCostResponse {
    ShipmentCostBreakdown cost = 1;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **OM:** Sales orders (outbound shipment demand), order fulfillment status
- **INV:** Warehouse addresses, inventory availability, pick-pack confirmation
- **Proc:** Purchase orders (inbound shipment demand), supplier addresses
- **GL:** Freight cost accounts, cost center allocation rules
- **DMS:** Shipping documents, bills of lading, customs documentation

### 6.2 Data Published To
- **OM:** Shipment status updates, tracking numbers, delivery confirmation
- **INV:** Inbound receiving notifications, transfer order ship/receive events
- **Proc:** Inbound shipment tracking, carrier and freight cost data
- **AP:** Freight invoices for payment processing
- **GL:** Freight cost journal entries, accrual postings
- **Notifications:** Shipment status alerts, exception notifications

---

## 7. Migrations

1. V001: `tms_carriers`
2. V002: `tms_carrier_contracts`
3. V003: `tms_carrier_rates`
4. V004: `tms_shipments`
5. V005: `tms_shipment_lines`
6. V006: `tms_load_plans`
7. V007: `tms_routes`
8. V008: `tms_freight_invoices`
9. V009: `tms_tracking_events`
10. V010: `tms_delivery_windows`
11. V011: Triggers for `updated_at`
