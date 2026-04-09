# 203 - Clinically Integrated Supply Chain Service Specification

## 1. Domain Overview

Clinically Integrated Supply Chain provides healthcare-specific supply chain management supporting medical product catalog with regulatory classifications (FDA Class I/II/III), UDI/GUDID tracking, and sterile/latex/MR-safe attributes, surgeon preference cards linking procedures to required products with alternate product support, consignment inventory management with vendor-owned and facility-owned quantity tracking and automated replenishment, recall management with FDA-mandated and voluntary recall workflows, quarantine actions, and patient impact assessment, and utilization analytics with cost-per-procedure benchmarking and waste tracking. Enables clinical supply optimization while maintaining strict regulatory compliance and patient safety standards. Integrates with Oracle Health for procedure scheduling and patient context, Supplier Management for vendor consignment agreements, Inventory for stock operations, Procurement for purchase orders, and Warehouse Management for distribution.

**Bounded Context:** Healthcare Supply Chain, Clinical Procurement & Inventory
**Service Name:** `clinical-supply-chain-service`
**Database:** `data/clinical_supply_chain.db`
**HTTP Port:** 8221 | **gRPC Port:** 9221

---

## 2. Database Schema

### 2.1 Medical Products
```sql
CREATE TABLE csc_medical_products (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_code TEXT NOT NULL,
    product_name TEXT NOT NULL,
    product_type TEXT NOT NULL
        CHECK(product_type IN ('DEVICE','PHARMACEUTICAL','SUPPLY','EQUIPMENT')),
    manufacturer TEXT NOT NULL,
    gudid TEXT,
    udi TEXT,
    regulatory_classification TEXT NOT NULL
        CHECK(regulatory_classification IN ('CLASS_I','CLASS_II','CLASS_III')),
    sterile_flag INTEGER NOT NULL DEFAULT 0,
    latex_flag INTEGER NOT NULL DEFAULT 0,
    mr_safe_flag INTEGER NOT NULL DEFAULT 0,
    shelf_life_days INTEGER,
    storage_requirements TEXT,                        -- JSON: {temperature_range, humidity, light_sensitive}
    unit_of_measure TEXT NOT NULL,
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','RECALLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, product_code)
);

CREATE INDEX idx_csc_products_tenant_type ON csc_medical_products(tenant_id, product_type);
CREATE INDEX idx_csc_products_tenant_class ON csc_medical_products(tenant_id, regulatory_classification);
CREATE INDEX idx_csc_products_tenant_status ON csc_medical_products(tenant_id, status);
CREATE INDEX idx_csc_products_tenant_manufacturer ON csc_medical_products(tenant_id, manufacturer);
CREATE INDEX idx_csc_products_tenant_gudid ON csc_medical_products(tenant_id, gudid);
CREATE INDEX idx_csc_products_tenant_active ON csc_medical_products(tenant_id, is_active);
```

### 2.2 Preference Cards
```sql
CREATE TABLE csc_preference_cards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    card_code TEXT NOT NULL,
    surgeon_id TEXT NOT NULL,
    procedure_code TEXT NOT NULL,
    procedure_name TEXT NOT NULL,
    required_products TEXT NOT NULL,                   -- JSON: [{product_id, quantity, alternate_product_id}]
    sterile_requirements TEXT,
    notes TEXT,
    last_used_at TEXT,
    usage_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, card_code)
);

CREATE INDEX idx_csc_prefcards_tenant_surgeon ON csc_preference_cards(tenant_id, surgeon_id);
CREATE INDEX idx_csc_prefcards_tenant_procedure ON csc_preference_cards(tenant_id, procedure_code);
CREATE INDEX idx_csc_prefcards_tenant_status ON csc_preference_cards(tenant_id, status);
CREATE INDEX idx_csc_prefcards_tenant_active ON csc_preference_cards(tenant_id, is_active);
```

### 2.3 Consignment Inventory
```sql
CREATE TABLE csc_consignment_inventory (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    vendor_id TEXT NOT NULL,
    vendor_owned_qty INTEGER NOT NULL DEFAULT 0,
    facility_owned_qty INTEGER NOT NULL DEFAULT 0,
    par_level INTEGER NOT NULL DEFAULT 0,
    auto_replenishment_flag INTEGER NOT NULL DEFAULT 0,
    replenishment_threshold INTEGER NOT NULL DEFAULT 0,
    last_replenished_at TEXT,
    usage_last_30_days INTEGER NOT NULL DEFAULT 0,
    total_value_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES csc_medical_products(id),
    UNIQUE(tenant_id, product_id, location_id)
);

CREATE INDEX idx_csc_consignment_tenant_product ON csc_consignment_inventory(tenant_id, product_id);
CREATE INDEX idx_csc_consignment_tenant_location ON csc_consignment_inventory(tenant_id, location_id);
CREATE INDEX idx_csc_consignment_tenant_vendor ON csc_consignment_inventory(tenant_id, vendor_id);
CREATE INDEX idx_csc_consignment_tenant_status ON csc_consignment_inventory(tenant_id, status);
CREATE INDEX idx_csc_consignment_tenant_auto_replen ON csc_consignment_inventory(tenant_id, auto_replenishment_flag);
CREATE INDEX idx_csc_consignment_tenant_active ON csc_consignment_inventory(tenant_id, is_active);
```

### 2.4 Recall Management
```sql
CREATE TABLE csc_recall_management (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    recall_number TEXT NOT NULL,
    product_id TEXT NOT NULL,
    manufacturer_recall_id TEXT NOT NULL,
    recall_type TEXT NOT NULL
        CHECK(recall_type IN ('VOLUNTARY','FDA_MANDATED','SAFETY')),
    affected_lot_numbers TEXT NOT NULL,                -- JSON: array of lot number strings
    affected_facilities TEXT NOT NULL,                 -- JSON: array of facility/location IDs
    description TEXT NOT NULL,
    risk_level TEXT NOT NULL
        CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    quarantine_actions TEXT NOT NULL,                  -- JSON: array of action items
    patient_impact_assessment TEXT,
    initiated_at TEXT NOT NULL,
    resolved_at TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INVESTIGATING','RESOLVED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES csc_medical_products(id),
    UNIQUE(tenant_id, recall_number)
);

CREATE INDEX idx_csc_recall_tenant_product ON csc_recall_management(tenant_id, product_id);
CREATE INDEX idx_csc_recall_tenant_risk ON csc_recall_management(tenant_id, risk_level);
CREATE INDEX idx_csc_recall_tenant_status ON csc_recall_management(tenant_id, status);
CREATE INDEX idx_csc_recall_tenant_type ON csc_recall_management(tenant_id, recall_type);
CREATE INDEX idx_csc_recall_tenant_initiated ON csc_recall_management(tenant_id, initiated_at);
CREATE INDEX idx_csc_recall_tenant_active ON csc_recall_management(tenant_id, is_active);
```

### 2.5 Utilization Analytics
```sql
CREATE TABLE csc_utilization_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,
    product_id TEXT NOT NULL,
    department TEXT NOT NULL,
    procedures_count INTEGER NOT NULL DEFAULT 0,
    units_used INTEGER NOT NULL DEFAULT 0,
    cost_per_procedure_cents INTEGER NOT NULL DEFAULT 0,
    waste_units INTEGER NOT NULL DEFAULT 0,
    waste_pct REAL NOT NULL DEFAULT 0.0,
    benchmark_cost_per_procedure_cents INTEGER,
    variance_from_benchmark_pct REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES csc_medical_products(id),
    UNIQUE(tenant_id, product_id, department, period)
);

CREATE INDEX idx_csc_util_tenant_period ON csc_utilization_analytics(tenant_id, period);
CREATE INDEX idx_csc_util_tenant_department ON csc_utilization_analytics(tenant_id, department);
CREATE INDEX idx_csc_util_tenant_product ON csc_utilization_analytics(tenant_id, product_id);
CREATE INDEX idx_csc_util_tenant_waste ON csc_utilization_analytics(tenant_id, waste_pct);
CREATE INDEX idx_csc_util_tenant_active ON csc_utilization_analytics(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Product Catalog
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/clinical-supply/products` | List medical products |
| POST | `/api/v1/clinical-supply/products` | Create medical product |
| GET | `/api/v1/clinical-supply/products/{id}` | Get product details |
| PUT | `/api/v1/clinical-supply/products/{id}` | Update product |
| PATCH | `/api/v1/clinical-supply/products/{id}/status` | Update product status |
| GET | `/api/v1/clinical-supply/products/search` | Search products by type/classification/manufacturer |
| GET | `/api/v1/clinical-supply/products/by-udi/{udi}` | Lookup product by UDI |

### 3.2 Preference Cards
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/clinical-supply/preference-cards` | List preference cards |
| POST | `/api/v1/clinical-supply/preference-cards` | Create preference card |
| GET | `/api/v1/clinical-supply/preference-cards/{id}` | Get preference card details |
| PUT | `/api/v1/clinical-supply/preference-cards/{id}` | Update preference card |
| GET | `/api/v1/clinical-supply/preference-cards/by-surgeon/{surgeon_id}` | List cards by surgeon |
| GET | `/api/v1/clinical-supply/preference-cards/by-procedure/{procedure_code}` | List cards by procedure |
| POST | `/api/v1/clinical-supply/preference-cards/{id}/record-usage` | Record preference card usage |

### 3.3 Consignment
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/clinical-supply/consignment` | List consignment inventory |
| POST | `/api/v1/clinical-supply/consignment` | Create consignment entry |
| GET | `/api/v1/clinical-supply/consignment/{id}` | Get consignment details |
| PUT | `/api/v1/clinical-supply/consignment/{id}` | Update consignment entry |
| POST | `/api/v1/clinical-supply/consignment/{id}/consume` | Record consignment consumption |
| POST | `/api/v1/clinical-supply/consignment/replenish-check` | Check auto-replenishment eligibility |
| GET | `/api/v1/clinical-supply/consignment/value-report` | Get consignment value report |

### 3.4 Recalls
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/clinical-supply/recalls` | List recalls |
| POST | `/api/v1/clinical-supply/recalls` | Initiate a recall |
| GET | `/api/v1/clinical-supply/recalls/{id}` | Get recall details |
| PUT | `/api/v1/clinical-supply/recalls/{id}` | Update recall |
| POST | `/api/v1/clinical-supply/recalls/{id}/quarantine` | Execute quarantine actions |
| POST | `/api/v1/clinical-supply/recalls/{id}/resolve` | Resolve a recall |
| GET | `/api/v1/clinical-supply/recalls/active-summary` | Get active recalls summary |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/clinical-supply/analytics` | List utilization analytics |
| POST | `/api/v1/clinical-supply/analytics` | Create utilization record |
| GET | `/api/v1/clinical-supply/analytics/{id}` | Get analytics record |
| GET | `/api/v1/clinical-supply/analytics/by-department` | Analytics by department |
| GET | `/api/v1/clinical-supply/analytics/cost-benchmark` | Cost benchmarking report |
| GET | `/api/v1/clinical-supply/analytics/waste-report` | Waste analysis report |

---

## 4. Business Rules

### 4.1 Medical Product Catalog
1. product_code MUST be unique within a tenant.
2. Products of type PHARMACEUTICAL MUST have shelf_life_days populated.
3. regulatory_classification MUST be one of CLASS_I, CLASS_II, or CLASS_III; CLASS_III products REQUIRE gudid and udi values.
4. Products with RECALLED status MUST NOT be available for new preference card entries or consignment replenishment.
5. storage_requirements JSON for sterile products (sterile_flag = 1) MUST include temperature range specifications.

### 4.2 Preference Card Management
6. card_code MUST be unique within a tenant.
7. required_products JSON MUST contain at least one product entry with product_id and quantity.
8. alternate_product_id in required_products MUST reference a valid ACTIVE product of the same product_type.
9. usage_count and last_used_at MUST be updated atomically when record-usage is called.
10. INACTIVE preference cards MUST NOT be auto-selected for new procedure scheduling.

### 4.3 Consignment Inventory
11. The combination of (tenant_id, product_id, location_id) MUST be unique.
12. vendor_owned_qty and facility_owned_qty MUST be non-negative integers.
13. Auto-replenishment (auto_replenishment_flag = 1) MUST have replenishment_threshold > 0 and par_level > 0.
14. total_value_cents MUST be recalculated as (vendor_owned_qty + facility_owned_qty) * unit_cost_cents from the product record.
15. Consumption MUST NOT reduce the sum of vendor_owned_qty + facility_owned_qty below zero.

### 4.4 Recall Management
16. recall_number MUST be unique within a tenant.
17. Recall status transitions MUST follow: ACTIVE -> INVESTIGATING -> RESOLVED -> CLOSED; reversal is not permitted.
18. FDA_MANDATED recalls MUST have risk_level of HIGH or CRITICAL.
19. quarantine_actions JSON MUST be populated before a recall can transition to INVESTIGATING status.
20. Affected products MUST be automatically status-changed to RECALLED upon recall creation with CRITICAL risk_level.

### 4.5 Utilization Analytics
21. The combination of (tenant_id, product_id, department, period) MUST be unique.
22. waste_pct MUST be calculated as (waste_units / units_used) * 100 when units_used > 0.
23. variance_from_benchmark_pct MUST be calculated relative to benchmark_cost_per_procedure_cents when available.
24. cost_per_procedure_cents MUST equal total product cost divided by procedures_count when procedures_count > 0.
25. Analytics records with waste_pct above 10% SHOULD trigger waste reduction recommendations.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.clinicalsupply.v1;

service ClinicalSupplyChainService {
    // Product catalog
    rpc CreateMedicalProduct(CreateMedicalProductRequest) returns (CreateMedicalProductResponse);
    rpc GetMedicalProduct(GetMedicalProductRequest) returns (GetMedicalProductResponse);
    rpc ListMedicalProducts(ListMedicalProductsRequest) returns (ListMedicalProductsResponse);
    rpc LookupByUDI(LookupByUDIRequest) returns (LookupByUDIResponse);

    // Preference cards
    rpc CreatePreferenceCard(CreatePreferenceCardRequest) returns (CreatePreferenceCardResponse);
    rpc GetPreferenceCard(GetPreferenceCardRequest) returns (GetPreferenceCardResponse);
    rpc ListPreferenceCards(ListPreferenceCardsRequest) returns (ListPreferenceCardsResponse);
    rpc RecordCardUsage(RecordCardUsageRequest) returns (RecordCardUsageResponse);

    // Consignment
    rpc CreateConsignment(CreateConsignmentRequest) returns (CreateConsignmentResponse);
    rpc GetConsignment(GetConsignmentRequest) returns (GetConsignmentResponse);
    rpc ListConsignment(ListConsignmentRequest) returns (ListConsignmentResponse);
    rpc ConsumeConsignment(ConsumeConsignmentRequest) returns (ConsumeConsignmentResponse);

    // Recalls
    rpc InitiateRecall(InitiateRecallRequest) returns (InitiateRecallResponse);
    rpc GetRecall(GetRecallRequest) returns (GetRecallResponse);
    rpc ListRecalls(ListRecallsRequest) returns (ListRecallsResponse);
    rpc QuarantineProducts(QuarantineProductsRequest) returns (QuarantineProductsResponse);
    rpc ResolveRecall(ResolveRecallRequest) returns (ResolveRecallResponse);

    // Analytics
    rpc GetUtilizationAnalytics(GetUtilizationAnalyticsRequest) returns (GetUtilizationAnalyticsResponse);
    rpc GetCostBenchmarkReport(GetCostBenchmarkReportRequest) returns (GetCostBenchmarkReportResponse);
    rpc GetWasteReport(GetWasteReportRequest) returns (GetWasteReportResponse);
}

message CreateMedicalProductRequest {
    string tenant_id = 1;
    string product_code = 2;
    string product_name = 3;
    string product_type = 4;
    string manufacturer = 5;
    string gudid = 6;
    string udi = 7;
    string regulatory_classification = 8;
    bool sterile_flag = 9;
    bool latex_flag = 10;
    bool mr_safe_flag = 11;
    int32 shelf_life_days = 12;
    string storage_requirements = 13;
    string unit_of_measure = 14;
    int64 unit_cost_cents = 15;
}

message CreateMedicalProductResponse {
    string product_id = 1;
    string product_code = 2;
    string product_name = 3;
    string status = 4;
}

message GetMedicalProductRequest {
    string tenant_id = 1;
    string product_id = 2;
}

message GetMedicalProductResponse {
    string product_id = 1;
    string product_code = 2;
    string product_name = 3;
    string product_type = 4;
    string manufacturer = 5;
    string regulatory_classification = 6;
    int64 unit_cost_cents = 7;
    string status = 8;
}

message ListMedicalProductsRequest {
    string tenant_id = 1;
    string product_type = 2;
    string regulatory_classification = 3;
    string status = 4;
    string manufacturer = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListMedicalProductsResponse {
    repeated GetMedicalProductResponse products = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message LookupByUDIRequest {
    string tenant_id = 1;
    string udi = 2;
}

message LookupByUDIResponse {
    string product_id = 1;
    string product_code = 2;
    string product_name = 3;
    string manufacturer = 4;
    string status = 5;
}

message CreatePreferenceCardRequest {
    string tenant_id = 1;
    string card_code = 2;
    string surgeon_id = 3;
    string procedure_code = 4;
    string procedure_name = 5;
    string required_products = 6;
    string sterile_requirements = 7;
    string notes = 8;
}

message CreatePreferenceCardResponse {
    string card_id = 1;
    string card_code = 2;
    string surgeon_id = 3;
    string procedure_name = 4;
    string status = 5;
}

message GetPreferenceCardRequest {
    string tenant_id = 1;
    string card_id = 2;
}

message GetPreferenceCardResponse {
    string card_id = 1;
    string card_code = 2;
    string surgeon_id = 3;
    string procedure_code = 4;
    string procedure_name = 5;
    string required_products = 6;
    int32 usage_count = 7;
    string last_used_at = 8;
    string status = 9;
}

message ListPreferenceCardsRequest {
    string tenant_id = 1;
    string surgeon_id = 2;
    string procedure_code = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListPreferenceCardsResponse {
    repeated GetPreferenceCardResponse cards = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RecordCardUsageRequest {
    string tenant_id = 1;
    string card_id = 2;
    string used_at = 3;
}

message RecordCardUsageResponse {
    string card_id = 1;
    int32 usage_count = 2;
    string last_used_at = 3;
}

message CreateConsignmentRequest {
    string tenant_id = 1;
    string product_id = 2;
    string location_id = 3;
    string vendor_id = 4;
    int32 vendor_owned_qty = 5;
    int32 facility_owned_qty = 6;
    int32 par_level = 7;
    bool auto_replenishment_flag = 8;
    int32 replenishment_threshold = 9;
}

message CreateConsignmentResponse {
    string consignment_id = 1;
    string product_id = 2;
    string location_id = 3;
    string status = 4;
}

message GetConsignmentRequest {
    string tenant_id = 1;
    string consignment_id = 2;
}

message GetConsignmentResponse {
    string consignment_id = 1;
    string product_id = 2;
    string location_id = 3;
    int32 vendor_owned_qty = 4;
    int32 facility_owned_qty = 5;
    int32 total_qty = 6;
    int64 total_value_cents = 7;
    string status = 8;
}

message ListConsignmentRequest {
    string tenant_id = 1;
    string location_id = 2;
    string vendor_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListConsignmentResponse {
    repeated GetConsignmentResponse consignments = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ConsumeConsignmentRequest {
    string tenant_id = 1;
    string consignment_id = 2;
    int32 quantity = 3;
    string source = 4;  // VENDOR_OWNED or FACILITY_OWNED
}

message ConsumeConsignmentResponse {
    string consignment_id = 1;
    int32 remaining_qty = 2;
    bool replenishment_triggered = 3;
}

message InitiateRecallRequest {
    string tenant_id = 1;
    string recall_number = 2;
    string product_id = 3;
    string manufacturer_recall_id = 4;
    string recall_type = 5;
    string affected_lot_numbers = 6;
    string affected_facilities = 7;
    string description = 8;
    string risk_level = 9;
    string quarantine_actions = 10;
    string patient_impact_assessment = 11;
}

message InitiateRecallResponse {
    string recall_id = 1;
    string recall_number = 2;
    string risk_level = 3;
    string status = 4;
}

message GetRecallRequest {
    string tenant_id = 1;
    string recall_id = 2;
}

message GetRecallResponse {
    string recall_id = 1;
    string recall_number = 2;
    string product_id = 3;
    string recall_type = 4;
    string risk_level = 5;
    string quarantine_actions = 6;
    string initiated_at = 7;
    string status = 8;
}

message ListRecallsRequest {
    string tenant_id = 1;
    string product_id = 2;
    string risk_level = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListRecallsResponse {
    repeated GetRecallResponse recalls = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message QuarantineProductsRequest {
    string tenant_id = 1;
    string recall_id = 2;
    string quarantine_details = 3;
}

message QuarantineProductsResponse {
    string recall_id = 1;
    int32 items_quarantined = 2;
    repeated string affected_locations = 3;
}

message ResolveRecallRequest {
    string tenant_id = 1;
    string recall_id = 2;
    string resolution_notes = 3;
}

message ResolveRecallResponse {
    string recall_id = 1;
    string resolved_at = 2;
    string status = 3;
}

message GetUtilizationAnalyticsRequest {
    string tenant_id = 1;
    string product_id = 2;
    string department = 3;
    string period = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message GetUtilizationAnalyticsResponse {
    repeated UtilizationRecord records = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UtilizationRecord {
    string id = 1;
    string product_id = 2;
    string department = 3;
    string period = 4;
    int32 procedures_count = 5;
    int32 units_used = 6;
    int64 cost_per_procedure_cents = 7;
    double waste_pct = 8;
    double variance_from_benchmark_pct = 9;
}

message GetCostBenchmarkReportRequest {
    string tenant_id = 1;
    string department = 2;
    string from_period = 3;
    string to_period = 4;
}

message GetCostBenchmarkReportResponse {
    repeated CostBenchmarkEntry entries = 1;
    double avg_variance_pct = 2;
    int32 total_products_analyzed = 3;
}

message CostBenchmarkEntry {
    string product_id = 1;
    string product_name = 2;
    int64 cost_per_procedure_cents = 3;
    int64 benchmark_cost_per_procedure_cents = 4;
    double variance_pct = 5;
}

message GetWasteReportRequest {
    string tenant_id = 1;
    string department = 2;
    string from_period = 3;
    string to_period = 4;
}

message GetWasteReportResponse {
    repeated WasteEntry entries = 1;
    double overall_waste_pct = 2;
    int64 total_waste_value_cents = 3;
}

message WasteEntry {
    string product_id = 1;
    string product_name = 2;
    int32 waste_units = 3;
    double waste_pct = 4;
    int64 waste_value_cents = 5;
}
```

---

## 6. Events

### Published Events
| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `clinSupply.recall.issued` | `clinical-supply.events` | `{ recall_id, recall_number, product_id, risk_level, affected_facilities }` | New recall initiated for a medical product |
| `clinSupply.consignment.depleted` | `clinical-supply.events` | `{ consignment_id, product_id, location_id, remaining_qty, replenishment_triggered }` | Consignment stock below threshold |
| `clinSupply.preference.updated` | `clinical-supply.events` | `{ card_id, surgeon_id, procedure_code, product_count }` | Preference card products updated |
| `clinSupply.recall.quarantined` | `clinical-supply.events` | `{ recall_id, items_quarantined, affected_locations }` | Products quarantined due to recall |

### Consumed Events
| Event | Source | Description |
|-------|--------|-------------|
| `patient.procedure.scheduled` | Oracle Health (186) | Procedure scheduling triggers preference card product reservation |
| `supplier.recall.notice` | Supplier Management (139) | Vendor-initiated recall notifications for product safety |
| `inventory.consumed` | Inventory (34) | Inventory consumption updates for utilization analytics |

---

## 7. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `oraclehealth-service` (186) | Procedure schedules, surgeon assignments, and patient context for preference card activation |
| `supplier-service` (139) | Vendor consignment agreements, recall notifications, and product catalog updates |
| `inv-service` (34) | Real-time inventory levels, stock consumption events, and warehouse locations |
| `procurement-service` (11) | Purchase order status and pricing for consignment replenishment |
| `wm-service` (37) | Warehouse locations, storage conditions, and distribution routing |

### Published To
| Service | Data |
|---------|------|
| `inv-service` (34) | Consignment consumption transactions, quarantine holds, and replenishment requests |
| `procurement-service` (11) | Consignment replenishment purchase requisitions and vendor-owned stock orders |
| `oraclehealth-service` (186) | Product availability status, recall alerts, and preference card validation results |
| `supplier-service` (139) | Consignment usage reports, recall acknowledgments, and product quality feedback |
| `wm-service` (37) | Quarantine isolation orders, storage requirement changes, and distribution instructions |

---

## 8. Migrations

1. V001: `csc_medical_products`
2. V002: `csc_preference_cards`
3. V003: `csc_consignment_inventory`
4. V004: `csc_recall_management`
5. V005: `csc_utilization_analytics`
6. V006: Triggers for `updated_at`
