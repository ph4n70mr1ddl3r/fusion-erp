# 199 - Travel & Transportation Industry Solution Specification

## 1. Domain Overview

Travel & Transportation provides industry-specific capabilities for fleet asset management, route operations, revenue management with dynamic pricing, crew scheduling with compliance, and booking management. Supports multi-modal fleet tracking (aircraft, vessel, vehicle, rail) with maintenance scheduling, route definitions with service frequency, fare class revenue management with demand forecasting, crew duty time tracking with rest period compliance, and passenger/shipper booking lifecycle management. Enables travel and transportation operators to maximize fleet utilization, optimize revenue per route, ensure crew compliance, and deliver seamless booking experiences. Integrates with Maintenance Cloud, Financials, Order Management, and Asset Management.

**Bounded Context:** Travel & Transportation Operations, Fleet & Revenue Management
**Service Name:** `travel-transportation-service`
**Database:** `data/travel_transportation.db`
**HTTP Port:** 8217 | **gRPC Port:** 9217

---

## 2. Database Schema

### 2.1 Fleet Assets
```sql
CREATE TABLE tt_fleet_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_code TEXT NOT NULL,
    asset_type TEXT NOT NULL CHECK(asset_type IN ('AIRCRAFT','VESSEL','VEHICLE','RAIL')),
    registration_number TEXT NOT NULL,
    make TEXT NOT NULL,
    model TEXT NOT NULL,
    year INTEGER,
    capacity_passengers INTEGER NOT NULL DEFAULT 0,
    capacity_cargo_kg REAL NOT NULL DEFAULT 0,
    fuel_type TEXT NOT NULL,
    maintenance_schedule_id TEXT,
    current_location TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','MAINTENANCE','RETIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, asset_code)
);

CREATE INDEX idx_tt_fleet_tenant ON tt_fleet_assets(tenant_id, status);
CREATE INDEX idx_tt_fleet_type ON tt_fleet_assets(asset_type);
```

### 2.2 Routes
```sql
CREATE TABLE tt_routes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    route_code TEXT NOT NULL,
    origin_code TEXT NOT NULL,
    destination_code TEXT NOT NULL,
    distance_km REAL NOT NULL DEFAULT 0,
    duration_minutes INTEGER NOT NULL DEFAULT 0,
    service_type TEXT NOT NULL CHECK(service_type IN ('PASSENGER','CARGO','MIXED')),
    frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(frequency IN ('DAILY','WEEKLY','ON_DEMAND')),
    operating_days TEXT,                           -- JSON: day numbers
    peak_season TEXT,                              -- JSON: {start, end}
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SEASONAL','DISCONTINUED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, route_code)
);

CREATE INDEX idx_tt_route_origin ON tt_routes(origin_code);
CREATE INDEX idx_tt_route_dest ON tt_routes(destination_code);
CREATE INDEX idx_tt_route_status ON tt_routes(tenant_id, status);
```

### 2.3 Revenue Management
```sql
CREATE TABLE tt_revenue_management (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    route_id TEXT NOT NULL,
    fare_class TEXT NOT NULL CHECK(fare_class IN ('ECONOMY','BUSINESS','FIRST','CARGO_STANDARD','CARGO_EXPRESS')),
    period TEXT NOT NULL,
    demand_forecast_qty INTEGER NOT NULL DEFAULT 0,
    yield_target_cents INTEGER NOT NULL DEFAULT 0,
    dynamic_pricing_rules TEXT,                    -- JSON: pricing adjustment rules
    current_avg_fare_cents INTEGER NOT NULL DEFAULT 0,
    competitor_avg_fare_cents INTEGER,
    load_factor_pct REAL NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (route_id) REFERENCES tt_routes(id),
    UNIQUE(tenant_id, route_id, fare_class, period)
);

CREATE INDEX idx_tt_rev_route ON tt_revenue_management(route_id, period DESC);
CREATE INDEX idx_tt_rev_class ON tt_revenue_management(fare_class);
```

### 2.4 Crew Scheduling
```sql
CREATE TABLE tt_crew_scheduling (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    crew_id TEXT NOT NULL,
    crew_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK(role IN ('PILOT','CO_PILOT','FLIGHT_ATTENDANT','DRIVER','CAPTAIN')),
    qualifications TEXT NOT NULL,                   -- JSON: qualification list
    certifications TEXT NOT NULL,                   -- JSON: cert IDs with expiry
    duty_hours_period REAL NOT NULL DEFAULT 0,
    max_duty_hours REAL NOT NULL DEFAULT 60,
    rest_period_hours REAL NOT NULL DEFAULT 10,
    assigned_route_id TEXT,
    assignment_date TEXT,
    assignment_status TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(assignment_status IN ('SCHEDULED','CHECKED_IN','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (assigned_route_id) REFERENCES tt_routes(id)
);

CREATE INDEX idx_tt_crew_route ON tt_crew_scheduling(assigned_route_id, assignment_date);
CREATE INDEX idx_tt_crew_role ON tt_crew_scheduling(role);
CREATE INDEX idx_tt_crew_status ON tt_crew_scheduling(tenant_id, assignment_status);
```

### 2.5 Booking Records
```sql
CREATE TABLE tt_booking_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    booking_reference TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    customer_type TEXT NOT NULL CHECK(customer_type IN ('PASSENGER','SHIPPER')),
    route_id TEXT NOT NULL,
    fare_class TEXT NOT NULL,
    booking_date TEXT NOT NULL DEFAULT (datetime('now')),
    travel_date TEXT NOT NULL,
    fare_cents INTEGER NOT NULL DEFAULT 0,
    taxes_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'CONFIRMED'
        CHECK(status IN ('CONFIRMED','CANCELLED','COMPLETED','NO_SHOW')),
    ancillary_services TEXT,                       -- JSON: extra services
    payment_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (route_id) REFERENCES tt_routes(id),
    UNIQUE(tenant_id, booking_reference)
);

CREATE INDEX idx_tt_booking_customer ON tt_booking_records(customer_id, travel_date DESC);
CREATE INDEX idx_tt_booking_route ON tt_booking_records(route_id, travel_date);
CREATE INDEX idx_tt_booking_status ON tt_booking_records(tenant_id, status);
CREATE INDEX idx_tt_booking_date ON tt_booking_records(travel_date);
```

---

## 3. API Endpoints

### 3.1 Fleet
| Method | Path | Description |
|--------|------|-------------|
| POST | `/travel/v1/fleet` | Register asset |
| GET | `/travel/v1/fleet` | List fleet |
| GET | `/travel/v1/fleet/{id}` | Get asset |
| PUT | `/travel/v1/fleet/{id}` | Update asset |

### 3.2 Routes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/travel/v1/routes` | Create route |
| GET | `/travel/v1/routes` | List routes |
| GET | `/travel/v1/routes/{id}` | Get route |
| PUT | `/travel/v1/routes/{id}` | Update route |

### 3.3 Revenue Management
| Method | Path | Description |
|--------|------|-------------|
| GET | `/travel/v1/revenue` | Revenue dashboard |
| GET | `/travel/v1/revenue/{route_id}` | Route revenue |
| POST | `/travel/v1/revenue/pricing` | Update dynamic pricing |
| GET | `/travel/v1/revenue/forecast` | Demand forecast |

### 3.4 Crew
| Method | Path | Description |
|--------|------|-------------|
| GET | `/travel/v1/crew` | List crew |
| POST | `/travel/v1/crew/assign` | Assign crew |
| GET | `/travel/v1/crew/{id}/schedule` | Crew schedule |
| POST | `/travel/v1/crew/check-duty` | Check duty compliance |

### 3.5 Bookings
| Method | Path | Description |
|--------|------|-------------|
| POST | `/travel/v1/bookings` | Create booking |
| GET | `/travel/v1/bookings` | List bookings |
| GET | `/travel/v1/bookings/{id}` | Get booking |
| POST | `/travel/v1/bookings/{id}/cancel` | Cancel booking |
| POST | `/travel/v1/bookings/{id}/check-in` | Check in |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `travel.booking.confirmed` | `{ booking_ref, customer_id, route, fare }` | Booking confirmed |
| `travel.departure.completed` | `{ route_id, asset_id, load_factor }` | Departure completed |
| `travel.crew.assigned` | `{ crew_id, route_id, date }` | Crew assigned |
| `travel.fleet.maintenance_due` | `{ asset_id, due_date }` | Maintenance due |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `maintenance.scheduled` | Maintenance Cloud (167) | Update fleet status |
| `payment.completed` | Payment (14) | Confirm booking |
| `weather.alert` | External | Issue travel advisories |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.travel_transportation.v1;

service TravelTransportationService {
    rpc GetFleetAsset(GetFleetAssetRequest) returns (GetFleetAssetResponse);
    rpc GetRoute(GetRouteRequest) returns (GetRouteResponse);
    rpc CreateBooking(CreateBookingRequest) returns (CreateBookingResponse);
    rpc GetBooking(GetBookingRequest) returns (GetBookingResponse);
    rpc AssignCrew(AssignCrewRequest) returns (AssignCrewResponse);
    rpc GetRevenueManagement(GetRevenueManagementRequest) returns (GetRevenueManagementResponse);
}

// Fleet asset messages
message GetFleetAssetRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetFleetAssetResponse {
    TtFleetAsset data = 1;
}

message TtFleetAsset {
    string id = 1;
    string tenant_id = 2;
    string asset_code = 3;
    string asset_type = 4;
    string registration_number = 5;
    string make = 6;
    string model = 7;
    int32 year = 8;
    int32 capacity_passengers = 9;
    double capacity_cargo_kg = 10;
    string fuel_type = 11;
    string maintenance_schedule_id = 12;
    string current_location = 13;
    string status = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Route messages
message GetRouteRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetRouteResponse {
    TtRoute data = 1;
}

message TtRoute {
    string id = 1;
    string tenant_id = 2;
    string route_code = 3;
    string origin_code = 4;
    string destination_code = 5;
    double distance_km = 6;
    int32 duration_minutes = 7;
    string service_type = 8;
    string frequency = 9;
    string operating_days = 10;
    string peak_season = 11;
    string status = 12;
    string created_at = 13;
    string updated_at = 14;
}

// Booking messages
message CreateBookingRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string customer_type = 3;
    string route_id = 4;
    string fare_class = 5;
    string travel_date = 6;
    int64 fare_cents = 7;
    int64 taxes_cents = 8;
    int64 total_cents = 9;
    string ancillary_services = 10;
    string payment_reference = 11;
}

message CreateBookingResponse {
    TtBookingRecord data = 1;
}

message GetBookingRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetBookingResponse {
    TtBookingRecord data = 1;
}

message TtBookingRecord {
    string id = 1;
    string tenant_id = 2;
    string booking_reference = 3;
    string customer_id = 4;
    string customer_type = 5;
    string route_id = 6;
    string fare_class = 7;
    string booking_date = 8;
    string travel_date = 9;
    int64 fare_cents = 10;
    int64 taxes_cents = 11;
    int64 total_cents = 12;
    string status = 13;
    string ancillary_services = 14;
    string payment_reference = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Crew messages
message AssignCrewRequest {
    string tenant_id = 1;
    string crew_id = 2;
    string crew_name = 3;
    string role = 4;
    string qualifications = 5;
    string certifications = 6;
    string assigned_route_id = 7;
    string assignment_date = 8;
}

message AssignCrewResponse {
    string id = 1;
    string crew_id = 2;
    string assignment_status = 3;
    double duty_hours_period = 4;
}

// Revenue management messages
message GetRevenueManagementRequest {
    string tenant_id = 1;
    string route_id = 2;
    string fare_class = 3;
    string period = 4;
}

message GetRevenueManagementResponse {
    TtRevenueManagement data = 1;
}

message TtRevenueManagement {
    string id = 1;
    string tenant_id = 2;
    string route_id = 3;
    string fare_class = 4;
    string period = 5;
    int32 demand_forecast_qty = 6;
    int64 yield_target_cents = 7;
    string dynamic_pricing_rules = 8;
    int64 current_avg_fare_cents = 9;
    int64 competitor_avg_fare_cents = 10;
    double load_factor_pct = 11;
    int64 revenue_cents = 12;
    string created_at = 13;
    string updated_at = 14;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | tt_fleet_assets | -- |
| V002 | tt_routes | -- |
| V003 | tt_revenue_management | V002 |
| V004 | tt_crew_scheduling | V002 |
| V005 | tt_booking_records | V002 |

---

## 7. Business Rules

1. **Duty Time Compliance**: Crew cannot be assigned if rest period requirements not met
2. **Dynamic Pricing**: Fares adjusted based on demand forecast, competitor pricing, and load factor
3. **Overbooking**: Configurable overbooking ratio per fare class and route
4. **Maintenance Windows**: Fleet assets blocked from scheduling during maintenance
5. **No-Show Tracking**: Repeated no-shows flagged for review
6. **Ancillary Revenue**: Additional services tracked separately for revenue attribution

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| maintenance-service | `GetSchedule` | Fleet maintenance scheduling |
| gl-service | `PostJournal` | Revenue recognition and AP |
| order-service | `GetOrder` | Cargo booking integration |
| asset-service | `GetAsset` | Fleet asset registry |
| payment-service | `ProcessPayment` | Booking payment processing |
| notification-service | `SendConfirmation` | Booking confirmations and alerts |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| financials-service | `GetRevenue` | Revenue reporting |
| maintenance-service | `GetFleetStatus` | Fleet availability for maintenance |
