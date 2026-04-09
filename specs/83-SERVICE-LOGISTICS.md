# 83 - Service Logistics Service Specification

## 1. Domain Overview

| Attribute | Value |
|---|---|
| **Bounded Context** | Service Logistics |
| **Service Name** | servicelogistics-service |
| **Database** | servicelogistics_db |
| **HTTP Port** | 8115 |
| **gRPC Port** | 9115 |

The Service Logistics module manages service parts inventory, return material authorizations (RMA), depot repair operations, replacement shipments, warranty claims processing, and service parts demand forecasting. It bridges post-sale service operations with supply chain logistics, ensuring that field technicians and repair depots have the parts needed to fulfill service commitments.

---

## 2. Database Schema

```sql
CREATE TABLE service_parts_inventory (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    part_number TEXT NOT NULL,
    part_name TEXT NOT NULL,
    part_description TEXT,
    part_category TEXT NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EACH',
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER NOT NULL DEFAULT 0,
    quantity_on_order INTEGER NOT NULL DEFAULT 0,
    reorder_point INTEGER NOT NULL DEFAULT 0,
    reorder_quantity INTEGER NOT NULL DEFAULT 0,
    safety_stock INTEGER NOT NULL DEFAULT 0,
    unit_cost INTEGER NOT NULL,
    standard_price INTEGER NOT NULL,
    warehouse_id TEXT NOT NULL,
    bin_location TEXT,
    lead_time_days INTEGER,
    preferred_vendor_id TEXT,
    alternate_part_number TEXT,
    service_bom_id TEXT,
    is_serial_tracked INTEGER NOT NULL DEFAULT 0,
    is_lot_tracked INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_service_parts_tenant ON service_parts_inventory(tenant_id);
CREATE INDEX idx_service_parts_part_number ON service_parts_inventory(tenant_id, part_number);
CREATE INDEX idx_service_parts_warehouse ON service_parts_inventory(tenant_id, warehouse_id);
CREATE INDEX idx_service_parts_category ON service_parts_inventory(tenant_id, part_category);

CREATE TABLE service_warehouses (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    warehouse_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    warehouse_type TEXT NOT NULL DEFAULT 'DEPOT',
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT,
    latitude TEXT,
    longitude TEXT,
    contact_name TEXT,
    contact_phone TEXT,
    contact_email TEXT,
    operating_hours TEXT,
    timezone TEXT,
    capacity_limit INTEGER,
    current_utilization INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_service_warehouses_tenant ON service_warehouses(tenant_id);
CREATE INDEX idx_service_warehouses_type ON service_warehouses(tenant_id, warehouse_type);

CREATE TABLE return_authorizations (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    rma_number TEXT NOT NULL UNIQUE,
    customer_id TEXT NOT NULL,
    contact_id TEXT,
    case_id TEXT,
    sales_order_id TEXT,
    rma_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    reason_code TEXT NOT NULL,
    reason_description TEXT,
    expected_return_date TEXT,
    received_date TEXT,
    receiving_warehouse_id TEXT,
    customer_po_number TEXT,
    total_credit_amount INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    resolution_type TEXT,
    notes TEXT,
    approved_by TEXT,
    approved_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_return_auths_tenant ON return_authorizations(tenant_id);
CREATE INDEX idx_return_auths_customer ON return_authorizations(tenant_id, customer_id);
CREATE INDEX idx_return_auths_status ON return_authorizations(tenant_id, status);
CREATE INDEX idx_return_auths_case ON return_authorizations(tenant_id, case_id);
CREATE INDEX idx_return_auths_rma ON return_authorizations(rma_number);

CREATE TABLE repair_orders (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    repair_order_number TEXT NOT NULL UNIQUE,
    rma_id TEXT,
    customer_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    serial_number TEXT,
    case_id TEXT,
    status TEXT NOT NULL DEFAULT 'RECEIVED',
    priority TEXT NOT NULL DEFAULT 'NORMAL',
    repair_type TEXT NOT NULL,
    depot_warehouse_id TEXT NOT NULL,
    assigned_technician_id TEXT,
    estimated_completion_date TEXT,
    actual_completion_date TEXT,
    estimated_labor_cost INTEGER,
    actual_labor_cost INTEGER,
    estimated_parts_cost INTEGER,
    actual_parts_cost INTEGER,
    total_repair_cost INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    warranty_coverage_type TEXT,
    warranty_id TEXT,
    customer_approval_required INTEGER NOT NULL DEFAULT 0,
    customer_approved INTEGER NOT NULL DEFAULT 0,
    customer_approved_at TEXT,
    diagnosis_summary TEXT,
    repair_summary TEXT,
    quality_check_passed INTEGER NOT NULL DEFAULT 0,
    quality_checked_by TEXT,
    quality_checked_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_repair_orders_tenant ON repair_orders(tenant_id);
CREATE INDEX idx_repair_orders_customer ON repair_orders(tenant_id, customer_id);
CREATE INDEX idx_repair_orders_status ON repair_orders(tenant_id, status);
CREATE INDEX idx_repair_orders_rma ON repair_orders(tenant_id, rma_id);
CREATE INDEX idx_repair_orders_asset ON repair_orders(tenant_id, asset_id);

CREATE TABLE repair_diagnostics (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    repair_order_id TEXT NOT NULL,
    diagnostic_step INTEGER NOT NULL,
    diagnostic_type TEXT NOT NULL,
    technician_id TEXT NOT NULL,
    findings TEXT NOT NULL,
    fault_code TEXT,
    root_cause TEXT,
    recommended_action TEXT,
    parts_required TEXT,
    estimated_hours INTEGER,
    performed_at TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_repair_diagnostics_order ON repair_diagnostics(tenant_id, repair_order_id);

CREATE TABLE replacement_shipments (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    shipment_number TEXT NOT NULL UNIQUE,
    repair_order_id TEXT,
    rma_id TEXT,
    customer_id TEXT NOT NULL,
    destination_address TEXT NOT NULL,
    destination_city TEXT,
    destination_state TEXT,
    destination_postal_code TEXT,
    destination_country TEXT,
    carrier_id TEXT,
    carrier_name TEXT,
    tracking_number TEXT,
    shipment_method TEXT NOT NULL DEFAULT 'STANDARD',
    status TEXT NOT NULL DEFAULT 'PENDING',
    shipped_at TEXT,
    estimated_delivery_at TEXT,
    delivered_at TEXT,
    ship_from_warehouse_id TEXT NOT NULL,
    total_value INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    insurance_value INTEGER,
    special_instructions TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_replacement_shipments_tenant ON replacement_shipments(tenant_id);
CREATE INDEX idx_replacement_shipments_customer ON replacement_shipments(tenant_id, customer_id);
CREATE INDEX idx_replacement_shipments_status ON replacement_shipments(tenant_id, status);
CREATE INDEX idx_replacement_shipments_tracking ON replacement_shipments(tracking_number);

CREATE TABLE service_bom_tables (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    bom_code TEXT NOT NULL UNIQUE,
    product_id TEXT NOT NULL,
    parent_part_number TEXT NOT NULL,
    component_part_number TEXT NOT NULL,
    component_quantity INTEGER NOT NULL DEFAULT 1,
    component_sequence INTEGER NOT NULL DEFAULT 0,
    is_replaceable INTEGER NOT NULL DEFAULT 1,
    is_field_replaceable INTEGER NOT NULL DEFAULT 1,
    estimated_replacement_time_minutes INTEGER,
    notes TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_service_bom_tenant ON service_bom_tables(tenant_id);
CREATE INDEX idx_service_bom_product ON service_bom_tables(tenant_id, product_id);
CREATE INDEX idx_service_bom_parent ON service_bom_tables(tenant_id, parent_part_number);

CREATE TABLE warranty_claims (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    claim_number TEXT NOT NULL UNIQUE,
    customer_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    serial_number TEXT,
    warranty_id TEXT NOT NULL,
    repair_order_id TEXT,
    case_id TEXT,
    claim_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'SUBMITTED',
    failure_date TEXT NOT NULL,
    failure_description TEXT NOT NULL,
    failure_code TEXT,
    labor_cost_claimed INTEGER,
    parts_cost_claimed INTEGER,
    total_cost_claimed INTEGER,
    labor_cost_approved INTEGER,
    parts_cost_approved INTEGER,
    total_cost_approved INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    vendor_id TEXT,
    vendor_claim_reference TEXT,
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_notes TEXT,
    rejection_reason TEXT,
    paid_at TEXT,
    payment_reference TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_warranty_claims_tenant ON warranty_claims(tenant_id);
CREATE INDEX idx_warranty_claims_customer ON warranty_claims(tenant_id, customer_id);
CREATE INDEX idx_warranty_claims_status ON warranty_claims(tenant_id, status);
CREATE INDEX idx_warranty_claims_warranty ON warranty_claims(tenant_id, warranty_id);
CREATE INDEX idx_warranty_claims_vendor ON warranty_claims(tenant_id, vendor_id);

CREATE TABLE service_parts_forecasts (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    part_number TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    forecast_period_start TEXT NOT NULL,
    forecast_period_end TEXT NOT NULL,
    forecast_method TEXT NOT NULL,
    predicted_demand INTEGER NOT NULL,
    predicted_demand_lower INTEGER,
    predicted_demand_upper INTEGER,
    confidence_score INTEGER,
    historical_basis_days INTEGER,
    seasonal_factor INTEGER,
    trend_factor INTEGER,
    notes TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_parts_forecast_tenant ON service_parts_forecasts(tenant_id);
CREATE INDEX idx_parts_forecast_part ON service_parts_forecasts(tenant_id, part_number);
CREATE INDEX idx_parts_forecast_period ON service_parts_forecasts(tenant_id, forecast_period_start, forecast_period_end);

CREATE TABLE core_exchange_tracking (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    exchange_number TEXT NOT NULL UNIQUE,
    customer_id TEXT NOT NULL,
    rma_id TEXT,
    repair_order_id TEXT,
    outgoing_part_number TEXT NOT NULL,
    outgoing_serial_number TEXT,
    incoming_part_number TEXT,
    incoming_serial_number TEXT,
    outgoing_shipment_id TEXT,
    incoming_shipment_id TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING',
    core_condition TEXT,
    core_value INTEGER,
    exchange_fee INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    core_received_at TEXT,
    core_inspected_at TEXT,
    core_inspected_by TEXT,
    inspection_notes TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_core_exchange_tenant ON core_exchange_tracking(tenant_id);
CREATE INDEX idx_core_exchange_customer ON core_exchange_tracking(tenant_id, customer_id);
CREATE INDEX idx_core_exchange_status ON core_exchange_tracking(tenant_id, status);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/parts` | Create a service part |
| GET | `/api/v1/parts/{partId}` | Get service part details |
| GET | `/api/v1/parts` | List/search service parts |
| PUT | `/api/v1/parts/{partId}` | Update service part |
| GET | `/api/v1/parts/availability` | Check parts availability across warehouses |
| POST | `/api/v1/parts/{partId}/reserve` | Reserve parts for a service order |
| POST | `/api/v1/parts/{partId}/release-reservation` | Release reserved parts |
| GET | `/api/v1/warehouses` | List service warehouses |
| POST | `/api/v1/warehouses` | Create service warehouse |
| GET | `/api/v1/warehouses/{warehouseId}` | Get warehouse details |
| PUT | `/api/v1/warehouses/{warehouseId}` | Update warehouse |
| GET | `/api/v1/warehouses/{warehouseId}/stock` | Get warehouse stock levels |
| POST | `/api/v1/rma` | Create return authorization (RMA) |
| GET | `/api/v1/rma/{rmaId}` | Get RMA details |
| GET | `/api/v1/rma` | List RMAs with filters |
| PUT | `/api/v1/rma/{rmaId}` | Update RMA |
| POST | `/api/v1/rma/{rmaId}/submit` | Submit RMA for approval |
| POST | `/api/v1/rma/{rmaId}/approve` | Approve RMA |
| POST | `/api/v1/rma/{rmaId}/receive` | Record RMA receipt |
| POST | `/api/v1/rma/{rmaId}/cancel` | Cancel RMA |
| POST | `/api/v1/repair-orders` | Create repair order |
| GET | `/api/v1/repair-orders/{repairOrderId}` | Get repair order details |
| GET | `/api/v1/repair-orders` | List repair orders |
| PUT | `/api/v1/repair-orders/{repairOrderId}` | Update repair order |
| POST | `/api/v1/repair-orders/{repairOrderId}/diagnose` | Add diagnostic entry |
| GET | `/api/v1/repair-orders/{repairOrderId}/diagnostics` | Get diagnostics history |
| POST | `/api/v1/repair-orders/{repairOrderId}/start-repair` | Start repair work |
| POST | `/api/v1/repair-orders/{repairOrderId}/complete-repair` | Complete repair |
| POST | `/api/v1/repair-orders/{repairOrderId}/quality-check` | Perform quality check |
| POST | `/api/v1/repair-orders/{repairOrderId}/approve` | Customer approve repair |
| POST | `/api/v1/replacement-shipments` | Create replacement shipment |
| GET | `/api/v1/replacement-shipments/{shipmentId}` | Get shipment details |
| GET | `/api/v1/replacement-shipments` | List replacement shipments |
| PUT | `/api/v1/replacement-shipments/{shipmentId}` | Update shipment |
| POST | `/api/v1/replacement-shipments/{shipmentId}/ship` | Mark shipment as shipped |
| POST | `/api/v1/replacement-shipments/{shipmentId}/deliver` | Mark shipment as delivered |
| POST | `/api/v1/warranty-claims` | Create warranty claim |
| GET | `/api/v1/warranty-claims/{claimId}` | Get warranty claim details |
| GET | `/api/v1/warranty-claims` | List warranty claims |
| PUT | `/api/v1/warranty-claims/{claimId}` | Update warranty claim |
| POST | `/api/v1/warranty-claims/{claimId}/submit` | Submit claim for review |
| POST | `/api/v1/warranty-claims/{claimId}/approve` | Approve warranty claim |
| POST | `/api/v1/warranty-claims/{claimId}/reject` | Reject warranty claim |
| POST | `/api/v1/warranty-claims/{claimId}/process-payment` | Process claim payment |
| GET | `/api/v1/service-bom/{productId}` | Get service BOM for product |
| POST | `/api/v1/service-bom` | Create service BOM entry |
| PUT | `/api/v1/service-bom/{bomId}` | Update service BOM entry |
| GET | `/api/v1/forecasts` | Get parts demand forecasts |
| POST | `/api/v1/forecasts/generate` | Generate demand forecasts |
| PUT | `/api/v1/forecasts/{forecastId}` | Update forecast |
| POST | `/api/v1/core-exchanges` | Create core exchange |
| GET | `/api/v1/core-exchanges/{exchangeId}` | Get core exchange details |
| GET | `/api/v1/core-exchanges` | List core exchanges |
| POST | `/api/v1/core-exchanges/{exchangeId}/receive-core` | Record core return |
| POST | `/api/v1/core-exchanges/{exchangeId}/inspect-core` | Record core inspection |

---

## 4. Business Rules

1. All monetary values (unit_cost, standard_price, total_repair_cost, claim amounts, etc.) MUST be stored as integers representing cents.
2. An RMA number MUST be auto-generated using the pattern `RMA-{YYYY}{MM}-{SEQ}` upon creation.
3. An RMA MUST be in APPROVED status before it can receive returned items; the receiving_warehouse_id MUST be specified before approval.
4. A repair order MUST be linked to either an RMA or a customer service case; standalone repair orders MUST still reference a valid customer_id and asset_id.
5. Repair diagnostics MUST be recorded before a repair order can transition to IN_PROGRESS status; at least one diagnostic entry is required.
6. The repair order total_cost MUST be calculated as the sum of actual_labor_cost and actual_parts_cost; this MUST be updated whenever related costs change.
7. Customer approval MUST be obtained for repair orders where the estimated total cost exceeds a configurable threshold (default: 50000 cents) and warranty_coverage_type is not FULL.
8. Parts availability MUST check quantity_available (quantity_on_hand - quantity_reserved) across all service warehouses; a part reservation MUST decrement quantity_available immediately.
9. Service BOM entries MUST be unique per (product_id, parent_part_number, component_part_number) combination within a tenant.
10. Warranty claims MUST validate that the failure_date falls within the warranty coverage period; claims outside the warranty period MUST be rejected automatically unless overridden by a supervisor.
11. Warranty claim total_cost_approved MUST NOT exceed total_cost_claimed; partial approvals are permitted but MUST include reviewer notes explaining the reduction.
12. Demand forecasts SHOULD be generated using a combination of historical consumption data, seasonal patterns, and installed base growth; forecast confidence_score MUST be an integer between 0 and 100.
13. Core exchange tracking MUST ensure that an outgoing part has been shipped before the incoming core can be marked as received; the system MUST enforce exchange completion within 30 days.
14. Replacement shipments MUST decrement the source warehouse inventory upon shipment confirmation; the system MUST validate sufficient stock before creating a shipment.
15. Quality checks MUST be performed by a different technician than the one who performed the repair; self-checks MUST be rejected.
16. All timestamps MUST be stored and transmitted in ISO 8601 format with UTC timezone.
17. Tenant isolation MUST be enforced on every query; no cross-tenant data access is permitted.
18. Soft delete MUST be used (is_active = 0) for all entities; hard deletes MUST NOT be permitted.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package servicelogistics.v1;

option go_package = "github.com/fusion/protos/servicelogistics/v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service ServiceLogistics {
    // Service Parts Inventory
    rpc CreateServicePart(CreateServicePartRequest) returns (ServicePartResponse);
    rpc GetServicePart(GetServicePartRequest) returns (ServicePartResponse);
    rpc ListServiceParts(ListServicePartsRequest) returns (ListServicePartsResponse);
    rpc UpdateServicePart(UpdateServicePartRequest) returns (ServicePartResponse);
    rpc CheckPartsAvailability(CheckPartsAvailabilityRequest) returns (PartsAvailabilityResponse);
    rpc ReserveParts(ReservePartsRequest) returns (ReservationResponse);
    rpc ReleaseReservation(ReleaseReservationRequest) returns (google.protobuf.Empty);

    // Warehouses
    rpc CreateWarehouse(CreateWarehouseRequest) returns (WarehouseResponse);
    rpc GetWarehouse(GetWarehouseRequest) returns (WarehouseResponse);
    rpc ListWarehouses(ListWarehousesRequest) returns (ListWarehousesResponse);
    rpc UpdateWarehouse(UpdateWarehouseRequest) returns (WarehouseResponse);
    rpc GetWarehouseStock(GetWarehouseStockRequest) returns (WarehouseStockResponse);

    // Return Authorizations (RMA)
    rpc CreateRMA(CreateRMARequest) returns (RMAResponse);
    rpc GetRMA(GetRMARequest) returns (RMAResponse);
    rpc ListRMAs(ListRMAsRequest) returns (ListRMAsResponse);
    rpc UpdateRMA(UpdateRMARequest) returns (RMAResponse);
    rpc ApproveRMA(ApproveRMARequest) returns (RMAResponse);
    rpc ReceiveRMA(ReceiveRMARequest) returns (RMAResponse);
    rpc CancelRMA(CancelRMARequest) returns (RMAResponse);

    // Repair Orders
    rpc CreateRepairOrder(CreateRepairOrderRequest) returns (RepairOrderResponse);
    rpc GetRepairOrder(GetRepairOrderRequest) returns (RepairOrderResponse);
    rpc ListRepairOrders(ListRepairOrdersRequest) returns (ListRepairOrdersResponse);
    rpc UpdateRepairOrder(UpdateRepairOrderRequest) returns (RepairOrderResponse);
    rpc AddDiagnosis(AddDiagnosisRequest) returns (DiagnosisResponse);
    rpc ListDiagnoses(ListDiagnosesRequest) returns (ListDiagnosesResponse);
    rpc StartRepair(StartRepairRequest) returns (RepairOrderResponse);
    rpc CompleteRepair(CompleteRepairRequest) returns (RepairOrderResponse);
    rpc PerformQualityCheck(PerformQualityCheckRequest) returns (RepairOrderResponse);
    rpc ApproveRepairOrder(ApproveRepairOrderRequest) returns (RepairOrderResponse);

    // Replacement Shipments
    rpc CreateReplacementShipment(CreateReplacementShipmentRequest) returns (ShipmentResponse);
    rpc GetReplacementShipment(GetReplacementShipmentRequest) returns (ShipmentResponse);
    rpc ListReplacementShipments(ListReplacementShipmentsRequest) returns (ListShipmentsResponse);
    rpc UpdateReplacementShipment(UpdateReplacementShipmentRequest) returns (ShipmentResponse);
    rpc ShipReplacement(ShipReplacementRequest) returns (ShipmentResponse);
    rpc DeliverReplacement(DeliverReplacementRequest) returns (ShipmentResponse);

    // Warranty Claims
    rpc CreateWarrantyClaim(CreateWarrantyClaimRequest) returns (WarrantyClaimResponse);
    rpc GetWarrantyClaim(GetWarrantyClaimRequest) returns (WarrantyClaimResponse);
    rpc ListWarrantyClaims(ListWarrantyClaimsRequest) returns (ListWarrantyClaimsResponse);
    rpc UpdateWarrantyClaim(UpdateWarrantyClaimRequest) returns (WarrantyClaimResponse);
    rpc ApproveWarrantyClaim(ApproveWarrantyClaimRequest) returns (WarrantyClaimResponse);
    rpc RejectWarrantyClaim(RejectWarrantyClaimRequest) returns (WarrantyClaimResponse);
    rpc ProcessClaimPayment(ProcessClaimPaymentRequest) returns (WarrantyClaimResponse);

    // Service BOM
    rpc GetServiceBOM(GetServiceBOMRequest) returns (ServiceBOMResponse);
    rpc CreateServiceBOMEntry(CreateServiceBOMEntryRequest) returns (ServiceBOMEntryResponse);
    rpc UpdateServiceBOMEntry(UpdateServiceBOMEntryRequest) returns (ServiceBOMEntryResponse);

    // Forecasts
    rpc GetForecasts(GetForecastsRequest) returns (ForecastsResponse);
    rpc GenerateForecasts(GenerateForecastsRequest) returns (ForecastsResponse);
    rpc UpdateForecast(UpdateForecastRequest) returns (ForecastResponse);

    // Core Exchanges
    rpc CreateCoreExchange(CreateCoreExchangeRequest) returns (CoreExchangeResponse);
    rpc GetCoreExchange(GetCoreExchangeRequest) returns (CoreExchangeResponse);
    rpc ListCoreExchanges(ListCoreExchangesRequest) returns (ListCoreExchangesResponse);
    rpc ReceiveCore(ReceiveCoreRequest) returns (CoreExchangeResponse);
    rpc InspectCore(InspectCoreRequest) returns (CoreExchangeResponse);
}

message ServicePartMessage {
    string id = 1;
    string tenant_id = 2;
    string part_number = 3;
    string part_name = 4;
    string part_description = 5;
    string part_category = 6;
    string unit_of_measure = 7;
    int32 quantity_on_hand = 8;
    int32 quantity_reserved = 9;
    int32 quantity_available = 10;
    int32 quantity_on_order = 11;
    int32 reorder_point = 12;
    int32 reorder_quantity = 13;
    int32 safety_stock = 14;
    int64 unit_cost = 15;
    int64 standard_price = 16;
    string warehouse_id = 17;
    string bin_location = 18;
    int32 lead_time_days = 19;
    string preferred_vendor_id = 20;
    string alternate_part_number = 21;
    bool is_serial_tracked = 22;
    bool is_lot_tracked = 23;
    bool is_active = 24;
    google.protobuf.Timestamp created_at = 25;
    google.protobuf.Timestamp updated_at = 26;
    string created_by = 27;
    string updated_by = 28;
    int32 version = 29;
}

message CreateServicePartRequest {
    string tenant_id = 1;
    string part_number = 2;
    string part_name = 3;
    string part_description = 4;
    string part_category = 5;
    string unit_of_measure = 6;
    int32 reorder_point = 7;
    int32 reorder_quantity = 8;
    int32 safety_stock = 9;
    int64 unit_cost = 10;
    int64 standard_price = 11;
    string warehouse_id = 12;
    string bin_location = 13;
    int32 lead_time_days = 14;
    string preferred_vendor_id = 15;
    string alternate_part_number = 16;
    bool is_serial_tracked = 17;
    bool is_lot_tracked = 18;
}

message GetServicePartRequest {
    string tenant_id = 1;
    string part_id = 2;
}

message ListServicePartsRequest {
    string tenant_id = 1;
    string part_category = 2;
    string warehouse_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListServicePartsResponse {
    repeated ServicePartMessage parts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateServicePartRequest {
    string tenant_id = 1;
    string part_id = 2;
    string part_name = 3;
    string part_description = 4;
    int32 reorder_point = 5;
    int32 reorder_quantity = 6;
    int32 safety_stock = 7;
    int64 unit_cost = 8;
    int64 standard_price = 9;
    string bin_location = 10;
    int32 version = 11;
}

message ServicePartResponse {
    ServicePartMessage part = 1;
}

message CheckPartsAvailabilityRequest {
    string tenant_id = 1;
    repeated string part_numbers = 2;
    repeated int32 quantities = 3;
    string preferred_warehouse_id = 4;
}

message PartsAvailabilityResponse {
    repeated PartAvailabilityResult results = 1;
}

message PartAvailabilityResult {
    string part_number = 2;
    int32 quantity_requested = 3;
    int32 quantity_available = 4;
    string available_warehouse_id = 5;
    bool is_available = 6;
}

message ReservePartsRequest {
    string tenant_id = 1;
    string part_id = 2;
    int32 quantity = 3;
    string reference_type = 4;
    string reference_id = 5;
}

message ReservationResponse {
    string reservation_id = 1;
    string part_id = 2;
    int32 quantity_reserved = 3;
    int32 quantity_remaining = 4;
}

message ReleaseReservationRequest {
    string tenant_id = 1;
    string reservation_id = 2;
}

message WarehouseMessage {
    string id = 1;
    string tenant_id = 2;
    string warehouse_code = 3;
    string name = 4;
    string description = 5;
    string warehouse_type = 6;
    string address_line1 = 7;
    string city = 8;
    string state_province = 9;
    string postal_code = 10;
    string country = 11;
    string contact_name = 12;
    string contact_phone = 13;
    string contact_email = 14;
    string operating_hours = 15;
    string timezone = 16;
    int32 capacity_limit = 17;
    int32 current_utilization = 18;
    bool is_active = 19;
    google.protobuf.Timestamp created_at = 20;
    google.protobuf.Timestamp updated_at = 21;
    int32 version = 22;
}

message CreateWarehouseRequest {
    string tenant_id = 1;
    string warehouse_code = 2;
    string name = 3;
    string description = 4;
    string warehouse_type = 5;
    string address_line1 = 6;
    string city = 7;
    string state_province = 8;
    string postal_code = 9;
    string country = 10;
    string contact_name = 11;
    string contact_phone = 12;
    string contact_email = 13;
    string operating_hours = 14;
    string timezone = 15;
    int32 capacity_limit = 16;
}

message GetWarehouseRequest {
    string tenant_id = 1;
    string warehouse_id = 2;
}

message ListWarehousesRequest {
    string tenant_id = 1;
    string warehouse_type = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListWarehousesResponse {
    repeated WarehouseMessage warehouses = 1;
    string next_page_token = 2;
}

message UpdateWarehouseRequest {
    string tenant_id = 1;
    string warehouse_id = 2;
    string name = 3;
    string description = 4;
    string address_line1 = 5;
    string city = 6;
    string contact_name = 7;
    string contact_phone = 8;
    string contact_email = 9;
    string operating_hours = 10;
    int32 capacity_limit = 11;
    int32 version = 12;
}

message WarehouseResponse {
    WarehouseMessage warehouse = 1;
}

message GetWarehouseStockRequest {
    string tenant_id = 1;
    string warehouse_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message WarehouseStockResponse {
    repeated StockItemMessage items = 1;
    string next_page_token = 2;
}

message StockItemMessage {
    string part_number = 1;
    string part_name = 2;
    int32 quantity_on_hand = 3;
    int32 quantity_reserved = 4;
    int32 quantity_available = 5;
    string bin_location = 6;
}

message RMAMessage {
    string id = 1;
    string tenant_id = 2;
    string rma_number = 3;
    string customer_id = 4;
    string contact_id = 5;
    string case_id = 6;
    string rma_type = 7;
    string status = 8;
    string reason_code = 9;
    string reason_description = 10;
    google.protobuf.Timestamp expected_return_date = 11;
    google.protobuf.Timestamp received_date = 12;
    string receiving_warehouse_id = 13;
    int64 total_credit_amount = 14;
    string currency_code = 15;
    string resolution_type = 16;
    string notes = 17;
    string approved_by = 18;
    google.protobuf.Timestamp approved_at = 19;
    bool is_active = 20;
    google.protobuf.Timestamp created_at = 21;
    google.protobuf.Timestamp updated_at = 22;
    string created_by = 23;
    string updated_by = 24;
    int32 version = 25;
}

message CreateRMARequest {
    string tenant_id = 1;
    string customer_id = 2;
    string contact_id = 3;
    string case_id = 4;
    string rma_type = 5;
    string reason_code = 6;
    string reason_description = 7;
    string expected_return_date = 8;
    string receiving_warehouse_id = 9;
    string resolution_type = 10;
    string notes = 11;
}

message GetRMARequest {
    string tenant_id = 1;
    string rma_id = 2;
}

message ListRMAsRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string status = 3;
    string rma_type = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListRMAsResponse {
    repeated RMAMessage rmas = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateRMARequest {
    string tenant_id = 1;
    string rma_id = 2;
    string reason_description = 3;
    string receiving_warehouse_id = 4;
    string resolution_type = 5;
    string notes = 6;
    int32 version = 7;
}

message ApproveRMARequest {
    string tenant_id = 1;
    string rma_id = 2;
    string notes = 3;
}

message ReceiveRMARequest {
    string tenant_id = 1;
    string rma_id = 2;
    string received_condition = 3;
    string notes = 4;
}

message CancelRMARequest {
    string tenant_id = 1;
    string rma_id = 2;
    string cancellation_reason = 3;
}

message RMAResponse {
    RMAMessage rma = 1;
}

message RepairOrderMessage {
    string id = 1;
    string tenant_id = 2;
    string repair_order_number = 3;
    string rma_id = 4;
    string customer_id = 5;
    string asset_id = 6;
    string product_id = 7;
    string serial_number = 8;
    string case_id = 9;
    string status = 10;
    string priority = 11;
    string repair_type = 12;
    string depot_warehouse_id = 13;
    string assigned_technician_id = 14;
    google.protobuf.Timestamp estimated_completion_date = 15;
    google.protobuf.Timestamp actual_completion_date = 16;
    int64 estimated_labor_cost = 17;
    int64 actual_labor_cost = 18;
    int64 estimated_parts_cost = 19;
    int64 actual_parts_cost = 20;
    int64 total_repair_cost = 21;
    string currency_code = 22;
    string warranty_coverage_type = 23;
    string warranty_id = 24;
    bool customer_approval_required = 25;
    bool customer_approved = 26;
    google.protobuf.Timestamp customer_approved_at = 27;
    string diagnosis_summary = 28;
    string repair_summary = 29;
    bool quality_check_passed = 30;
    string quality_checked_by = 31;
    google.protobuf.Timestamp quality_checked_at = 32;
    bool is_active = 33;
    google.protobuf.Timestamp created_at = 34;
    google.protobuf.Timestamp updated_at = 35;
    string created_by = 36;
    string updated_by = 37;
    int32 version = 38;
}

message CreateRepairOrderRequest {
    string tenant_id = 1;
    string rma_id = 2;
    string customer_id = 3;
    string asset_id = 4;
    string product_id = 5;
    string serial_number = 6;
    string case_id = 7;
    string priority = 8;
    string repair_type = 9;
    string depot_warehouse_id = 10;
    string warranty_id = 11;
    google.protobuf.Timestamp estimated_completion_date = 12;
    int64 estimated_labor_cost = 13;
    int64 estimated_parts_cost = 14;
}

message GetRepairOrderRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
}

message ListRepairOrdersRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string status = 3;
    string depot_warehouse_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListRepairOrdersResponse {
    repeated RepairOrderMessage repair_orders = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateRepairOrderRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    string assigned_technician_id = 3;
    google.protobuf.Timestamp estimated_completion_date = 4;
    int64 estimated_labor_cost = 5;
    int64 estimated_parts_cost = 6;
    int32 version = 7;
}

message DiagnosisMessage {
    string id = 1;
    string tenant_id = 2;
    string repair_order_id = 3;
    int32 diagnostic_step = 4;
    string diagnostic_type = 5;
    string technician_id = 6;
    string findings = 7;
    string fault_code = 8;
    string root_cause = 9;
    string recommended_action = 10;
    string parts_required = 11;
    int32 estimated_hours = 12;
    google.protobuf.Timestamp performed_at = 13;
}

message AddDiagnosisRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    string diagnostic_type = 3;
    string findings = 4;
    string fault_code = 5;
    string root_cause = 6;
    string recommended_action = 7;
    string parts_required = 8;
    int32 estimated_hours = 9;
}

message DiagnosisResponse {
    DiagnosisMessage diagnosis = 1;
}

message ListDiagnosesRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
}

message ListDiagnosesResponse {
    repeated DiagnosisMessage diagnoses = 1;
}

message StartRepairRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    string assigned_technician_id = 3;
}

message CompleteRepairRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    string repair_summary = 3;
    int64 actual_labor_cost = 4;
    int64 actual_parts_cost = 5;
}

message PerformQualityCheckRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    bool passed = 3;
    string notes = 4;
}

message ApproveRepairOrderRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    bool approved = 3;
    string comments = 4;
}

message RepairOrderResponse {
    RepairOrderMessage repair_order = 1;
}

message ShipmentMessage {
    string id = 1;
    string tenant_id = 2;
    string shipment_number = 3;
    string repair_order_id = 4;
    string rma_id = 5;
    string customer_id = 6;
    string destination_address = 7;
    string destination_city = 8;
    string destination_state = 9;
    string destination_postal_code = 10;
    string destination_country = 11;
    string carrier_id = 12;
    string carrier_name = 13;
    string tracking_number = 14;
    string shipment_method = 15;
    string status = 16;
    google.protobuf.Timestamp shipped_at = 17;
    google.protobuf.Timestamp estimated_delivery_at = 18;
    google.protobuf.Timestamp delivered_at = 19;
    string ship_from_warehouse_id = 20;
    int64 total_value = 21;
    string currency_code = 22;
    int64 insurance_value = 23;
    string special_instructions = 24;
    bool is_active = 25;
    google.protobuf.Timestamp created_at = 26;
    google.protobuf.Timestamp updated_at = 27;
    int32 version = 28;
}

message CreateReplacementShipmentRequest {
    string tenant_id = 1;
    string repair_order_id = 2;
    string rma_id = 3;
    string customer_id = 4;
    string destination_address = 5;
    string destination_city = 6;
    string destination_state = 7;
    string destination_postal_code = 8;
    string destination_country = 9;
    string shipment_method = 10;
    string ship_from_warehouse_id = 11;
    int64 total_value = 12;
    string currency_code = 13;
    int64 insurance_value = 14;
    string special_instructions = 15;
}

message GetReplacementShipmentRequest {
    string tenant_id = 1;
    string shipment_id = 2;
}

message ListReplacementShipmentsRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListShipmentsResponse {
    repeated ShipmentMessage shipments = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateReplacementShipmentRequest {
    string tenant_id = 1;
    string shipment_id = 2;
    string carrier_id = 3;
    string carrier_name = 4;
    string tracking_number = 5;
    string shipment_method = 6;
    string special_instructions = 7;
    int32 version = 8;
}

message ShipReplacementRequest {
    string tenant_id = 1;
    string shipment_id = 2;
    string carrier_id = 3;
    string carrier_name = 4;
    string tracking_number = 5;
}

message DeliverReplacementRequest {
    string tenant_id = 1;
    string shipment_id = 2;
    string delivery_confirmation = 3;
}

message ShipmentResponse {
    ShipmentMessage shipment = 1;
}

message WarrantyClaimMessage {
    string id = 1;
    string tenant_id = 2;
    string claim_number = 3;
    string customer_id = 4;
    string product_id = 5;
    string asset_id = 6;
    string serial_number = 7;
    string warranty_id = 8;
    string repair_order_id = 9;
    string case_id = 10;
    string claim_type = 11;
    string status = 12;
    string failure_date = 13;
    string failure_description = 14;
    string failure_code = 15;
    int64 labor_cost_claimed = 16;
    int64 parts_cost_claimed = 17;
    int64 total_cost_claimed = 18;
    int64 labor_cost_approved = 19;
    int64 parts_cost_approved = 20;
    int64 total_cost_approved = 21;
    string currency_code = 22;
    string vendor_id = 23;
    string vendor_claim_reference = 24;
    string reviewed_by = 25;
    google.protobuf.Timestamp reviewed_at = 26;
    string review_notes = 27;
    string rejection_reason = 28;
    google.protobuf.Timestamp paid_at = 29;
    string payment_reference = 30;
    bool is_active = 31;
    google.protobuf.Timestamp created_at = 32;
    google.protobuf.Timestamp updated_at = 33;
    string created_by = 34;
    string updated_by = 35;
    int32 version = 36;
}

message CreateWarrantyClaimRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string product_id = 3;
    string asset_id = 4;
    string serial_number = 5;
    string warranty_id = 6;
    string repair_order_id = 7;
    string case_id = 8;
    string claim_type = 9;
    string failure_date = 10;
    string failure_description = 11;
    string failure_code = 12;
    int64 labor_cost_claimed = 13;
    int64 parts_cost_claimed = 14;
    string vendor_id = 15;
}

message GetWarrantyClaimRequest {
    string tenant_id = 1;
    string claim_id = 2;
}

message ListWarrantyClaimsRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string status = 3;
    string vendor_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListWarrantyClaimsResponse {
    repeated WarrantyClaimMessage claims = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateWarrantyClaimRequest {
    string tenant_id = 1;
    string claim_id = 2;
    string failure_description = 3;
    string failure_code = 4;
    int64 labor_cost_claimed = 5;
    int64 parts_cost_claimed = 6;
    string vendor_claim_reference = 7;
    int32 version = 8;
}

message ApproveWarrantyClaimRequest {
    string tenant_id = 1;
    string claim_id = 2;
    int64 labor_cost_approved = 3;
    int64 parts_cost_approved = 4;
    string review_notes = 5;
}

message RejectWarrantyClaimRequest {
    string tenant_id = 1;
    string claim_id = 2;
    string rejection_reason = 3;
    string review_notes = 4;
}

message ProcessClaimPaymentRequest {
    string tenant_id = 1;
    string claim_id = 2;
    string payment_reference = 3;
}

message WarrantyClaimResponse {
    WarrantyClaimMessage claim = 1;
}

message GetServiceBOMRequest {
    string tenant_id = 1;
    string product_id = 2;
}

message ServiceBOMResponse {
    string product_id = 1;
    repeated ServiceBOMEntryMessage entries = 2;
}

message ServiceBOMEntryMessage {
    string id = 1;
    string bom_code = 2;
    string parent_part_number = 3;
    string component_part_number = 4;
    int32 component_quantity = 5;
    int32 component_sequence = 6;
    bool is_replaceable = 7;
    bool is_field_replaceable = 8;
    int32 estimated_replacement_time_minutes = 9;
    string notes = 10;
}

message CreateServiceBOMEntryRequest {
    string tenant_id = 1;
    string bom_code = 2;
    string product_id = 3;
    string parent_part_number = 4;
    string component_part_number = 5;
    int32 component_quantity = 6;
    int32 component_sequence = 7;
    bool is_replaceable = 8;
    bool is_field_replaceable = 9;
    int32 estimated_replacement_time_minutes = 10;
}

message UpdateServiceBOMEntryRequest {
    string tenant_id = 1;
    string bom_id = 2;
    int32 component_quantity = 3;
    bool is_replaceable = 4;
    bool is_field_replaceable = 5;
    int32 estimated_replacement_time_minutes = 6;
    int32 version = 7;
}

message ServiceBOMEntryResponse {
    ServiceBOMEntryMessage entry = 1;
}

message ForecastMessage {
    string id = 1;
    string tenant_id = 2;
    string part_number = 3;
    string warehouse_id = 4;
    string forecast_period_start = 5;
    string forecast_period_end = 6;
    string forecast_method = 7;
    int32 predicted_demand = 8;
    int32 predicted_demand_lower = 9;
    int32 predicted_demand_upper = 10;
    int32 confidence_score = 11;
    int32 historical_basis_days = 12;
    int32 seasonal_factor = 13;
    int32 trend_factor = 14;
}

message GetForecastsRequest {
    string tenant_id = 1;
    string part_number = 2;
    string warehouse_id = 3;
    string period_start = 4;
    string period_end = 5;
}

message ForecastsResponse {
    repeated ForecastMessage forecasts = 1;
}

message GenerateForecastsRequest {
    string tenant_id = 1;
    repeated string part_numbers = 2;
    string warehouse_id = 3;
    int32 forecast_horizon_days = 4;
    string forecast_method = 5;
}

message UpdateForecastRequest {
    string tenant_id = 1;
    string forecast_id = 2;
    int32 predicted_demand = 3;
    int32 confidence_score = 4;
    string notes = 5;
    int32 version = 6;
}

message ForecastResponse {
    ForecastMessage forecast = 1;
}

message CoreExchangeMessage {
    string id = 1;
    string tenant_id = 2;
    string exchange_number = 3;
    string customer_id = 4;
    string rma_id = 5;
    string repair_order_id = 6;
    string outgoing_part_number = 7;
    string outgoing_serial_number = 8;
    string incoming_part_number = 9;
    string incoming_serial_number = 10;
    string status = 11;
    string core_condition = 12;
    int64 core_value = 13;
    int64 exchange_fee = 14;
    string currency_code = 15;
    google.protobuf.Timestamp core_received_at = 16;
    google.protobuf.Timestamp core_inspected_at = 17;
    string core_inspected_by = 18;
    string inspection_notes = 19;
    bool is_active = 20;
    google.protobuf.Timestamp created_at = 21;
    google.protobuf.Timestamp updated_at = 22;
    int32 version = 23;
}

message CreateCoreExchangeRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string rma_id = 3;
    string repair_order_id = 4;
    string outgoing_part_number = 5;
    string outgoing_serial_number = 6;
    string incoming_part_number = 7;
    int64 core_value = 8;
    int64 exchange_fee = 9;
    string currency_code = 10;
}

message GetCoreExchangeRequest {
    string tenant_id = 1;
    string exchange_id = 2;
}

message ListCoreExchangesRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCoreExchangesResponse {
    repeated CoreExchangeMessage exchanges = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReceiveCoreRequest {
    string tenant_id = 1;
    string exchange_id = 2;
    string incoming_serial_number = 3;
    string core_condition = 4;
}

message InspectCoreRequest {
    string tenant_id = 1;
    string exchange_id = 2;
    string core_condition = 3;
    string inspection_notes = 4;
}

message CoreExchangeResponse {
    CoreExchangeMessage exchange = 1;
}
```

---

## 6. Inter-Service Integration

### Inbound Dependencies

| Source Service | Integration Type | Purpose |
|---|---|---|
| Inventory (INV) | REST/gRPC | Sync on-hand quantities, receive goods receipts, trigger replenishment orders |
| Warehouse Management (WMS) | REST/gRPC | Warehouse operations, pick-pack-ship for replacement orders, bin location management |
| Field Service | REST/gRPC | Parts requirements from field service work orders, technician inventory allocation |
| Enterprise Asset Management (EAM) | REST/gRPC | Asset details, warranty information, maintenance history for repair context |
| Customer Service | REST/gRPC | Case linkage for RMAs, repair orders, and warranty claims |
| Product Registry | REST | Product and part master data, BOM definitions |
| Finance / General Ledger | REST | Cost posting for repairs, warranty claim payments, credit memos for returns |
| Shipping Carriers | REST | Tracking number creation, rate quotes, label generation |

### Outbound Events Consumed By

| Target Service | Event | Purpose |
|---|---|---|
| Inventory (INV) | PartsReserved, PartsConsumed | Update aggregate inventory levels |
| Warehouse Management (WMS) | ReplacementShipmentCreated | Initiate pick-pack-ship process |
| Field Service | RepairCompleted, ReplacementShipped | Update field service work order status |
| Enterprise Asset Management (EAM) | RepairCompleted, WarrantyClaimApproved | Update asset condition and warranty records |
| Customer Service | RMACreated, RepairStarted, RepairCompleted | Update linked service case status |
| Finance / General Ledger | WarrantyClaimPaid, CreditIssued | Post financial transactions |
| Supply Chain Planning | PartsForecastGenerated | Feed demand signals into procurement planning |

---

## 7. Events

| Event | Payload | Description |
|---|---|---|
| `RMACreated` | `{ rma_id, rma_number, tenant_id, customer_id, rma_type, reason_code, receiving_warehouse_id }` | Emitted when a new return authorization is created |
| `RMAApproved` | `{ rma_id, tenant_id, rma_number, approved_by, total_credit_amount }` | Emitted when an RMA is approved |
| `RMAReceived` | `{ rma_id, tenant_id, rma_number, received_date, receiving_warehouse_id }` | Emitted when returned items are physically received |
| `RepairStarted` | `{ repair_order_id, tenant_id, repair_order_number, assigned_technician_id, asset_id, depot_warehouse_id }` | Emitted when repair work begins |
| `RepairCompleted` | `{ repair_order_id, tenant_id, repair_order_number, total_repair_cost, quality_check_passed }` | Emitted when repair work is completed |
| `ReplacementShipped` | `{ shipment_id, tenant_id, shipment_number, customer_id, carrier_name, tracking_number, estimated_delivery_at }` | Emitted when a replacement shipment is dispatched |
| `ReplacementDelivered` | `{ shipment_id, tenant_id, shipment_number, customer_id, delivered_at }` | Emitted when a replacement shipment is confirmed delivered |
| `WarrantyClaimFiled` | `{ claim_id, claim_number, tenant_id, customer_id, warranty_id, claim_type, total_cost_claimed }` | Emitted when a new warranty claim is submitted |
| `WarrantyClaimApproved` | `{ claim_id, tenant_id, claim_number, total_cost_approved, reviewed_by }` | Emitted when a warranty claim is approved |
| `WarrantyClaimRejected` | `{ claim_id, tenant_id, claim_number, rejection_reason, reviewed_by }` | Emitted when a warranty claim is rejected |
| `WarrantyClaimPaid` | `{ claim_id, tenant_id, claim_number, total_cost_approved, payment_reference }` | Emitted when a warranty claim payment is processed |
| `PartsReserved` | `{ part_id, tenant_id, part_number, quantity_reserved, quantity_remaining, reference_type, reference_id }` | Emitted when service parts are reserved |
| `PartsLowStock` | `{ part_id, tenant_id, part_number, quantity_available, reorder_point, warehouse_id }` | Emitted when a part falls below reorder point |
| `ForecastGenerated` | `{ tenant_id, part_number, warehouse_id, predicted_demand, confidence_score, forecast_period }` | Emitted when a demand forecast is generated |
| `CoreExchangeCompleted` | `{ exchange_id, tenant_id, exchange_number, core_condition, core_value }` | Emitted when a core exchange cycle is fully completed |
