# 193 - Automotive Industry Solution Service Specification

## 1. Domain Overview

Automotive Industry Solution provides comprehensive automotive manufacturing and dealer management supporting vehicle lifecycle tracking from production through dealer assignment and warranty, dealer network management with franchise types, capacity, certification levels, and performance metrics, warranty claims processing with defect codes, repair details, parts usage, and labor tracking, production order management with model configurations, bill of materials, plant and line scheduling, and supplier part management with JIT/KANBAN flags, lead times, and quality ratings. Enables end-to-end visibility from manufacturing floor to dealership lot with integrated warranty analytics and supplier quality management. Integrates with Order Management for vehicle orders, Quality for inspection results, and Receiving for supplier deliveries.

**Bounded Context:** Automotive Manufacturing & Dealer Management
**Service Name:** `automotive-service`
**Database:** `data/automotive.db`
**HTTP Port:** 8211 | **gRPC Port:** 9211

---

## 2. Database Schema

### 2.1 Vehicles
```sql
CREATE TABLE auto_vehicles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    vin TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    trim_level TEXT,
    exterior_color TEXT,
    interior_color TEXT,
    engine_type TEXT NOT NULL
        CHECK(engine_type IN ('GASOLINE','DIESEL','HYBRID','ELECTRIC','HYDROGEN')),
    transmission_type TEXT
        CHECK(transmission_type IN ('MANUAL','AUTOMATIC','CVT','DUAL_CLUTCH')),
    production_date TEXT NOT NULL,
    plant_id TEXT NOT NULL,
    dealer_id TEXT,
    warranty_start_date TEXT,
    warranty_end_date TEXT,
    warranty_miles_limit INTEGER,
    current_mileage INTEGER NOT NULL DEFAULT 0,
    warranty_status TEXT NOT NULL DEFAULT 'NOT_ACTIVE'
        CHECK(warranty_status IN ('NOT_ACTIVE','ACTIVE','EXPIRED','VOIDED')),
    status TEXT NOT NULL DEFAULT 'IN_PRODUCTION'
        CHECK(status IN ('IN_PRODUCTION','PRODUCED','IN_TRANSIT','AT_DEALER','SOLD','SCRAPPED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, vin)
);

CREATE INDEX idx_auto_vehicles_tenant_model ON auto_vehicles(tenant_id, model_code);
CREATE INDEX idx_auto_vehicles_tenant_dealer ON auto_vehicles(tenant_id, dealer_id);
CREATE INDEX idx_auto_vehicles_tenant_status ON auto_vehicles(tenant_id, status);
CREATE INDEX idx_auto_vehicles_tenant_warranty ON auto_vehicles(tenant_id, warranty_status);
CREATE INDEX idx_auto_vehicles_tenant_plant ON auto_vehicles(tenant_id, plant_id);
CREATE INDEX idx_auto_vehicles_tenant_date ON auto_vehicles(tenant_id, production_date);
CREATE INDEX idx_auto_vehicles_tenant_active ON auto_vehicles(tenant_id, is_active);
```

### 2.2 Dealer Network
```sql
CREATE TABLE auto_dealer_network (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dealer_code TEXT NOT NULL,
    dealer_name TEXT NOT NULL,
    address_line1 TEXT NOT NULL,
    address_line2 TEXT,
    city TEXT NOT NULL,
    state_province TEXT NOT NULL,
    postal_code TEXT NOT NULL,
    country TEXT NOT NULL,
    phone TEXT NOT NULL,
    email TEXT,
    franchise_types TEXT NOT NULL,                    -- JSON: array of franchise brand codes
    capacity_vehicles INTEGER NOT NULL DEFAULT 0,
    current_inventory INTEGER NOT NULL DEFAULT 0,
    certification_level TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(certification_level IN ('STANDARD','SILVER','GOLD','PLATINUM')),
    performance_score REAL NOT NULL DEFAULT 0.0,
    sales_volume_ytd INTEGER NOT NULL DEFAULT 0,
    customer_satisfaction_score REAL,
    region TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUSPENDED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dealer_code)
);

CREATE INDEX idx_auto_dealer_tenant_region ON auto_dealer_network(tenant_id, region);
CREATE INDEX idx_auto_dealer_tenant_cert ON auto_dealer_network(tenant_id, certification_level);
CREATE INDEX idx_auto_dealer_tenant_status ON auto_dealer_network(tenant_id, status);
CREATE INDEX idx_auto_dealer_tenant_active ON auto_dealer_network(tenant_id, is_active);
```

### 2.3 Warranty Claims
```sql
CREATE TABLE auto_warranty_claims (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    claim_number TEXT NOT NULL,
    vehicle_id TEXT NOT NULL,
    vin TEXT NOT NULL,
    dealer_id TEXT NOT NULL,
    defect_code TEXT NOT NULL,
    defect_description TEXT NOT NULL,
    component_category TEXT NOT NULL
        CHECK(component_category IN ('ENGINE','TRANSMISSION','ELECTRICAL','BODY','SUSPENSION','BRAKES','HVAC','INFOTAINMENT','OTHER')),
    repair_description TEXT NOT NULL,
    parts_used TEXT,                                  -- JSON: array of part number/quantity/cost
    labor_hours REAL NOT NULL DEFAULT 0.0,
    labor_rate_cents INTEGER NOT NULL DEFAULT 0,
    parts_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_claim_amount_cents INTEGER NOT NULL DEFAULT 0,
    repair_date TEXT NOT NULL,
    mileage_at_repair INTEGER NOT NULL DEFAULT 0,
    claim_type TEXT NOT NULL DEFAULT 'WARRANTY'
        CHECK(claim_type IN ('WARRANTY','GOODWILL','RECALL','EXTENDED_WARRANTY')),
    status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(status IN ('SUBMITTED','UNDER_REVIEW','APPROVED','PARTIALLY_APPROVED','REJECTED','PAID','DISPUTED')),
    reviewed_by TEXT,
    reviewed_at TEXT,
    payment_reference TEXT,
    paid_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (vehicle_id) REFERENCES auto_vehicles(id),
    FOREIGN KEY (dealer_id) REFERENCES auto_dealer_network(id),
    UNIQUE(tenant_id, claim_number)
);

CREATE INDEX idx_auto_warranty_tenant_vehicle ON auto_warranty_claims(tenant_id, vehicle_id);
CREATE INDEX idx_auto_warranty_tenant_dealer ON auto_warranty_claims(tenant_id, dealer_id);
CREATE INDEX idx_auto_warranty_tenant_status ON auto_warranty_claims(tenant_id, status);
CREATE INDEX idx_auto_warranty_tenant_type ON auto_warranty_claims(tenant_id, claim_type);
CREATE INDEX idx_auto_warranty_tenant_defect ON auto_warranty_claims(tenant_id, defect_code);
CREATE INDEX idx_auto_warranty_tenant_date ON auto_warranty_claims(tenant_id, repair_date);
CREATE INDEX idx_auto_warranty_tenant_active ON auto_warranty_claims(tenant_id, is_active);
```

### 2.4 Production Orders
```sql
CREATE TABLE auto_production_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_number TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    trim_configuration TEXT,                          -- JSON: trim, color, engine, options
    quantity_ordered INTEGER NOT NULL,
    quantity_completed INTEGER NOT NULL DEFAULT 0,
    quantity_scrapped INTEGER NOT NULL DEFAULT 0,
    plant_id TEXT NOT NULL,
    production_line TEXT NOT NULL,
    bill_of_materials TEXT NOT NULL,                  -- JSON: BOM with part numbers and quantities
    scheduled_start_date TEXT NOT NULL,
    scheduled_end_date TEXT NOT NULL,
    actual_start_date TEXT,
    actual_end_date TEXT,
    priority INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','RELEASED','IN_PROGRESS','COMPLETED','CLOSED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_auto_prod_tenant_model ON auto_production_orders(tenant_id, model_code);
CREATE INDEX idx_auto_prod_tenant_plant ON auto_production_orders(tenant_id, plant_id);
CREATE INDEX idx_auto_prod_tenant_status ON auto_production_orders(tenant_id, status);
CREATE INDEX idx_auto_prod_tenant_dates ON auto_production_orders(tenant_id, scheduled_start_date, scheduled_end_date);
CREATE INDEX idx_auto_prod_tenant_active ON auto_production_orders(tenant_id, is_active);
```

### 2.5 Supplier Parts
```sql
CREATE TABLE auto_supplier_parts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    part_number TEXT NOT NULL,
    part_name TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    supplier_name TEXT NOT NULL,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    quality_rating REAL NOT NULL DEFAULT 0.0,
    is_jit_kanban INTEGER NOT NULL DEFAULT 0,
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    minimum_order_quantity INTEGER NOT NULL DEFAULT 1,
    applicable_models TEXT,                           -- JSON: list of model codes
    last_delivery_date TEXT,
    last_quality_score REAL,
    defect_rate_ppm REAL NOT NULL DEFAULT 0.0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','UNDER_REVIEW','SUSPENDED','DISCONTINUED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, part_number, supplier_id)
);

CREATE INDEX idx_auto_supplier_tenant_part ON auto_supplier_parts(tenant_id, part_number);
CREATE INDEX idx_auto_supplier_tenant_supplier ON auto_supplier_parts(tenant_id, supplier_id);
CREATE INDEX idx_auto_supplier_tenant_jit ON auto_supplier_parts(tenant_id, is_jit_kanban);
CREATE INDEX idx_auto_supplier_tenant_status ON auto_supplier_parts(tenant_id, status);
CREATE INDEX idx_auto_supplier_tenant_quality ON auto_supplier_parts(tenant_id, quality_rating);
CREATE INDEX idx_auto_supplier_tenant_active ON auto_supplier_parts(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Vehicles
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/automotive/vehicles` | List vehicles |
| POST | `/api/v1/automotive/vehicles` | Register vehicle |
| GET | `/api/v1/automotive/vehicles/{id}` | Get vehicle details |
| PUT | `/api/v1/automotive/vehicles/{id}` | Update vehicle |
| GET | `/api/v1/automotive/vehicles/vin/{vin}` | Lookup vehicle by VIN |
| PATCH | `/api/v1/automotive/vehicles/{id}/assign-dealer` | Assign vehicle to dealer |

### 3.2 Dealer Network
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/automotive/dealers` | List dealers |
| POST | `/api/v1/automotive/dealers` | Create dealer |
| GET | `/api/v1/automotive/dealers/{id}` | Get dealer details |
| PUT | `/api/v1/automotive/dealers/{id}` | Update dealer |
| PATCH | `/api/v1/automotive/dealers/{id}/certify` | Update dealer certification |
| GET | `/api/v1/automotive/dealers/{id}/performance` | Get dealer performance metrics |

### 3.3 Warranty
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/automotive/warranty/claims` | Submit warranty claim |
| GET | `/api/v1/automotive/warranty/claims` | List warranty claims |
| GET | `/api/v1/automotive/warranty/claims/{id}` | Get claim details |
| POST | `/api/v1/automotive/warranty/claims/{id}/approve` | Approve claim |
| POST | `/api/v1/automotive/warranty/claims/{id}/reject` | Reject claim |
| GET | `/api/v1/automotive/warranty/analysis` | Warranty analytics and trends |

### 3.4 Production
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/automotive/production-orders` | List production orders |
| POST | `/api/v1/automotive/production-orders` | Create production order |
| GET | `/api/v1/automotive/production-orders/{id}` | Get order details |
| PUT | `/api/v1/automotive/production-orders/{id}` | Update order |
| POST | `/api/v1/automotive/production-orders/{id}/release` | Release order to production |
| POST | `/api/v1/automotive/production-orders/{id}/complete` | Complete production order |

### 3.5 Supplier Parts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/automotive/supplier-parts` | List supplier parts |
| POST | `/api/v1/automotive/supplier-parts` | Create supplier part assignment |
| GET | `/api/v1/automotive/supplier-parts/{id}` | Get supplier part details |
| PUT | `/api/v1/automotive/supplier-parts/{id}` | Update supplier part |
| GET | `/api/v1/automotive/supplier-parts/quality-report` | Supplier quality report |

---

## 4. Business Rules

### 4.1 Vehicle Management
1. Each vehicle MUST have a unique VIN within a tenant; VINs MUST conform to the 17-character ISO 3779 standard.
2. Vehicle status transitions MUST follow: IN_PRODUCTION -> PRODUCED -> IN_TRANSIT -> AT_DEALER -> SOLD; SCRAPPED is a terminal state.
3. Warranty activation MUST occur when a vehicle transitions to SOLD status; warranty_end_date MUST be calculated from warranty_start_date.
4. A vehicle with warranty_status ACTIVE MUST validate mileage and date eligibility for warranty claims.

### 4.2 Dealer Network
5. Each dealer MUST have a unique dealer_code within a tenant.
6. Dealer certification_level transitions MUST follow: STANDARD -> SILVER -> GOLD -> PLATINUM; demotion requires documented justification.
7. A dealer in SUSPENDED or TERMINATED status MUST NOT receive new vehicle assignments or have warranty claims approved.
8. performance_score MUST be calculated as a weighted average of sales volume, customer satisfaction, and warranty claim ratios.
9. current_inventory MUST NOT exceed capacity_vehicles for any dealer.

### 4.3 Warranty Claims
10. claim_number MUST be auto-generated and unique within a tenant.
11. A warranty claim MUST reference a valid vehicle with warranty_status ACTIVE; the mileage_at_repair MUST NOT exceed warranty_miles_limit (if set).
12. total_claim_amount_cents MUST equal labor_hours * labor_rate_cents + sum of parts costs.
13. Parts used MUST reference valid supplier part numbers stored in auto_supplier_parts.
14. Claim status transitions MUST follow: SUBMITTED -> UNDER_REVIEW -> APPROVED/PARTIALLY_APPROVED/REJECTED -> PAID/DISPUTED.

### 4.4 Production Orders
15. Production order numbers MUST be auto-generated and unique within a tenant.
16. Order status transitions MUST follow: PLANNED -> RELEASED -> IN_PROGRESS -> COMPLETED -> CLOSED; CANCELLED is valid from PLANNED or RELEASED.
17. quantity_completed + quantity_scrapped MUST NOT exceed quantity_ordered.
18. Bill of materials JSON MUST reference valid part numbers from auto_supplier_parts.
19. Production orders with JIT/KANBAN parts SHOULD trigger automatic supplier notifications upon release.

### 4.5 Supplier Parts
20. The combination of (tenant_id, part_number, supplier_id) MUST be unique.
21. quality_rating MUST be between 0.0 and 100.0, calculated from defect rate and delivery performance.
22. JIT/KANBAN flagged parts (is_jit_kanban = 1) MUST have lead_time_days of 1 or less.
23. Supplier parts with quality_rating below 70.0 SHOULD trigger a status change to UNDER_REVIEW.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.automotive.v1;

service AutomotiveService {
    // Vehicle management
    rpc CreateVehicle(CreateVehicleRequest) returns (CreateVehicleResponse);
    rpc GetVehicle(GetVehicleRequest) returns (GetVehicleResponse);
    rpc GetVehicleByVIN(GetVehicleByVINRequest) returns (GetVehicleByVINResponse);
    rpc ListVehicles(ListVehiclesRequest) returns (ListVehiclesResponse);
    rpc AssignDealer(AssignDealerRequest) returns (AssignDealerResponse);

    // Dealer network
    rpc CreateDealer(CreateDealerRequest) returns (CreateDealerResponse);
    rpc GetDealer(GetDealerRequest) returns (GetDealerResponse);
    rpc ListDealers(ListDealersRequest) returns (ListDealersResponse);
    rpc CertifyDealer(CertifyDealerRequest) returns (CertifyDealerResponse);

    // Warranty
    rpc SubmitWarrantyClaim(SubmitWarrantyClaimRequest) returns (SubmitWarrantyClaimResponse);
    rpc GetWarrantyClaim(GetWarrantyClaimRequest) returns (GetWarrantyClaimResponse);
    rpc ListWarrantyClaims(ListWarrantyClaimsRequest) returns (ListWarrantyClaimsResponse);
    rpc ApproveClaim(ApproveClaimRequest) returns (ApproveClaimResponse);
    rpc GetWarrantyAnalysis(GetWarrantyAnalysisRequest) returns (GetWarrantyAnalysisResponse);

    // Production
    rpc CreateProductionOrder(CreateProductionOrderRequest) returns (CreateProductionOrderResponse);
    rpc GetProductionOrder(GetProductionOrderRequest) returns (GetProductionOrderResponse);
    rpc ListProductionOrders(ListProductionOrdersRequest) returns (ListProductionOrdersResponse);
    rpc ReleaseProductionOrder(ReleaseProductionOrderRequest) returns (ReleaseProductionOrderResponse);
    rpc CompleteProductionOrder(CompleteProductionOrderRequest) returns (CompleteProductionOrderResponse);

    // Supplier parts
    rpc CreateSupplierPart(CreateSupplierPartRequest) returns (CreateSupplierPartResponse);
    rpc ListSupplierParts(ListSupplierPartsRequest) returns (ListSupplierPartsResponse);
    rpc GetSupplierQualityReport(GetSupplierQualityReportRequest) returns (GetSupplierQualityReportResponse);
}

message CreateVehicleRequest {
    string tenant_id = 1;
    string vin = 2;
    string model_code = 3;
    string model_name = 4;
    string trim_level = 5;
    string exterior_color = 6;
    string interior_color = 7;
    string engine_type = 8;
    string transmission_type = 9;
    string production_date = 10;
    string plant_id = 11;
}

message CreateVehicleResponse {
    string vehicle_id = 1;
    string vin = 2;
    string status = 3;
}

message GetVehicleRequest {
    string tenant_id = 1;
    string vehicle_id = 2;
}

message GetVehicleResponse {
    string vehicle_id = 1;
    string vin = 2;
    string model_code = 3;
    string model_name = 4;
    string trim_level = 5;
    string engine_type = 6;
    string production_date = 7;
    string dealer_id = 8;
    string warranty_status = 9;
    string status = 10;
}

message GetVehicleByVINRequest {
    string tenant_id = 1;
    string vin = 2;
}

message GetVehicleByVINResponse {
    string vehicle_id = 1;
    string vin = 2;
    string model_name = 3;
    string status = 4;
    string warranty_status = 5;
}

message ListVehiclesRequest {
    string tenant_id = 1;
    string model_code = 2;
    string status = 3;
    string dealer_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListVehiclesResponse {
    repeated GetVehicleResponse vehicles = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message AssignDealerRequest {
    string tenant_id = 1;
    string vehicle_id = 2;
    string dealer_id = 3;
}

message AssignDealerResponse {
    string vehicle_id = 1;
    string dealer_id = 2;
    string status = 3;
}

message CreateDealerRequest {
    string tenant_id = 1;
    string dealer_code = 2;
    string dealer_name = 3;
    string address_line1 = 4;
    string city = 5;
    string state_province = 6;
    string postal_code = 7;
    string country = 8;
    string phone = 9;
    string email = 10;
    string franchise_types = 11;
    int32 capacity_vehicles = 12;
    string region = 13;
}

message CreateDealerResponse {
    string dealer_id = 1;
    string dealer_code = 2;
    string dealer_name = 3;
    string certification_level = 4;
}

message GetDealerRequest {
    string tenant_id = 1;
    string dealer_id = 2;
}

message GetDealerResponse {
    string dealer_id = 1;
    string dealer_code = 2;
    string dealer_name = 3;
    string region = 4;
    string certification_level = 5;
    double performance_score = 6;
    int32 current_inventory = 7;
    int32 capacity_vehicles = 8;
    string status = 9;
}

message ListDealersRequest {
    string tenant_id = 1;
    string region = 2;
    string certification_level = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListDealersResponse {
    repeated GetDealerResponse dealers = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CertifyDealerRequest {
    string tenant_id = 1;
    string dealer_id = 2;
    string certification_level = 3;
    string justification = 4;
}

message CertifyDealerResponse {
    string dealer_id = 1;
    string certification_level = 2;
}

message SubmitWarrantyClaimRequest {
    string tenant_id = 1;
    string vehicle_id = 2;
    string dealer_id = 3;
    string defect_code = 4;
    string defect_description = 5;
    string component_category = 6;
    string repair_description = 7;
    string parts_used = 8;
    double labor_hours = 9;
    int64 labor_rate_cents = 10;
    int64 parts_cost_cents = 11;
    string repair_date = 12;
    int32 mileage_at_repair = 13;
    string claim_type = 14;
}

message SubmitWarrantyClaimResponse {
    string claim_id = 1;
    string claim_number = 2;
    string status = 3;
}

message GetWarrantyClaimRequest {
    string tenant_id = 1;
    string claim_id = 2;
}

message GetWarrantyClaimResponse {
    string claim_id = 1;
    string claim_number = 2;
    string vin = 3;
    string dealer_id = 4;
    string defect_code = 5;
    string component_category = 6;
    int64 total_claim_amount_cents = 7;
    string status = 8;
    string claim_type = 9;
}

message ListWarrantyClaimsRequest {
    string tenant_id = 1;
    string vehicle_id = 2;
    string dealer_id = 3;
    string status = 4;
    string claim_type = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListWarrantyClaimsResponse {
    repeated GetWarrantyClaimResponse claims = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ApproveClaimRequest {
    string tenant_id = 1;
    string claim_id = 2;
    string approved_by = 3;
    int64 approved_amount_cents = 4;
    string comments = 5;
}

message ApproveClaimResponse {
    string claim_id = 1;
    string status = 2;
    int64 approved_amount_cents = 3;
    string approved_at = 4;
}

message GetWarrantyAnalysisRequest {
    string tenant_id = 1;
    string model_code = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetWarrantyAnalysisResponse {
    int32 total_claims = 1;
    int64 total_cost_cents = 2;
    repeated DefectAnalysis defects = 3;
    repeated DealerClaimAnalysis dealer_claims = 4;
}

message DefectAnalysis {
    string defect_code = 1;
    int32 claim_count = 2;
    int64 total_cost_cents = 3;
    double percentage = 4;
}

message DealerClaimAnalysis {
    string dealer_id = 1;
    string dealer_name = 2;
    int32 claim_count = 3;
    double avg_resolution_days = 4;
}

message CreateProductionOrderRequest {
    string tenant_id = 1;
    string model_code = 2;
    string model_name = 3;
    string trim_configuration = 4;
    int32 quantity_ordered = 5;
    string plant_id = 6;
    string production_line = 7;
    string bill_of_materials = 8;
    string scheduled_start_date = 9;
    string scheduled_end_date = 10;
    int32 priority = 11;
}

message CreateProductionOrderResponse {
    string order_id = 1;
    string order_number = 2;
    string status = 3;
}

message GetProductionOrderRequest {
    string tenant_id = 1;
    string order_id = 2;
}

message GetProductionOrderResponse {
    string order_id = 1;
    string order_number = 2;
    string model_code = 3;
    string model_name = 4;
    int32 quantity_ordered = 5;
    int32 quantity_completed = 6;
    string plant_id = 7;
    string production_line = 8;
    string status = 9;
    string scheduled_start_date = 10;
    string scheduled_end_date = 11;
}

message ListProductionOrdersRequest {
    string tenant_id = 1;
    string model_code = 2;
    string plant_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListProductionOrdersResponse {
    repeated GetProductionOrderResponse orders = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReleaseProductionOrderRequest {
    string tenant_id = 1;
    string order_id = 2;
}

message ReleaseProductionOrderResponse {
    string order_id = 1;
    string status = 2;
}

message CompleteProductionOrderRequest {
    string tenant_id = 1;
    string order_id = 2;
    int32 quantity_completed = 3;
    int32 quantity_scrapped = 4;
}

message CompleteProductionOrderResponse {
    string order_id = 1;
    string status = 2;
    string actual_end_date = 3;
}

message CreateSupplierPartRequest {
    string tenant_id = 1;
    string part_number = 2;
    string part_name = 3;
    string supplier_id = 4;
    string supplier_name = 5;
    int32 lead_time_days = 6;
    double quality_rating = 7;
    bool is_jit_kanban = 8;
    int64 unit_cost_cents = 9;
    int32 minimum_order_quantity = 10;
    string applicable_models = 11;
}

message CreateSupplierPartResponse {
    string supplier_part_id = 1;
    string part_number = 2;
    string supplier_name = 3;
    string status = 4;
}

message ListSupplierPartsRequest {
    string tenant_id = 1;
    string part_number = 2;
    string supplier_id = 3;
    bool jit_only = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListSupplierPartsResponse {
    repeated SupplierPartInfo parts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SupplierPartInfo {
    string id = 1;
    string part_number = 2;
    string part_name = 3;
    string supplier_name = 4;
    int32 lead_time_days = 5;
    double quality_rating = 6;
    bool is_jit_kanban = 7;
    string status = 8;
}

message GetSupplierQualityReportRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetSupplierQualityReportResponse {
    string supplier_id = 1;
    string supplier_name = 2;
    int32 total_parts = 3;
    double avg_quality_rating = 4;
    double avg_defect_rate_ppm = 5;
    repeated PartQualityDetail parts = 6;
}

message PartQualityDetail {
    string part_number = 1;
    string part_name = 2;
    double quality_rating = 3;
    double defect_rate_ppm = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `om-service` | Vehicle orders from dealers, order creation and modification events |
| `qual-service` | Quality inspection results, defect classifications, non-conformance reports |
| `receiving-service` | Supplier delivery confirmations, receipt of raw materials and parts |
| `inv-service` | Parts inventory levels, warehouse stock for BOM availability checks |
| `auth-service` | User identity for dealer portal access, claim reviewer authorization |

### Published To
| Service | Data |
|---------|------|
| `om-service` | Vehicle production completion events for order fulfillment tracking |
| `inv-service` | Parts consumption from production, finished goods vehicle receipts |
| `gl-service` | Warranty expense accruals, production cost journal entries |
| `ap-service` | Supplier payments for parts, warranty claim dealer reimbursements |
| `reporting-service` | Production efficiency metrics, warranty analytics, dealer performance dashboards |
| `notification-service` | Production milestone alerts, warranty claim status updates, dealer certification changes |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `auto.vehicle.produced` | `automotive.events` | `{ vehicle_id, vin, model_code, plant_id, production_date }` | Vehicle completed production |
| `auto.warranty.claimed` | `automotive.events` | `{ claim_id, claim_number, vin, dealer_id, defect_code, total_amount_cents }` | Warranty claim submitted |
| `auto.dealer.certified` | `automotive.events` | `{ dealer_id, dealer_code, certification_level, previous_level }` | Dealer certification level changed |
| `auto.production.completed` | `automotive.events` | `{ order_id, order_number, model_code, quantity_completed, actual_end_date }` | Production order completed |

---

## 8. Migrations

1. V001: `auto_vehicles`
2. V002: `auto_dealer_network`
3. V003: `auto_warranty_claims`
4. V004: `auto_production_orders`
5. V005: `auto_supplier_parts`
6. V006: Triggers for `updated_at`
