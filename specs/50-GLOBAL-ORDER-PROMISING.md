# 50 - Global Order Promising Service Specification

## 1. Domain Overview

Global Order Promising provides real-time Available-to-Promise (ATP) and Capable-to-Promise (CTP) checking across the entire supply chain. It considers current inventory, planned production, inbound shipments, safety stock, allocation rules, and substitution options to give accurate delivery date commitments at order entry time. Supports multiple promising methods including ATP based on inventory and planned receipts, CTP based on manufacturing capacity, and what-if scenario analysis for planning purposes.

**Bounded Context:** Order Promising, Availability & Delivery Date Commitment
**Service Name:** `promising-service`
**Database:** `data/promising.db`
**HTTP Port:** 8082 | **gRPC Port:** 9082

---

## 2. Database Schema

### 2.1 Promising Rules
```sql
CREATE TABLE promising_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('ATP','CTP','ATP_PLUS_CTP','BACK_TO_BACK')),
    item_category_id TEXT,
    channel TEXT,
    warehouse_id TEXT,
    promising_method TEXT NOT NULL DEFAULT 'FORWARD'
        CHECK(promising_method IN ('FORWARD','BACKWARD')),
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    safety_stock_offset INTEGER NOT NULL DEFAULT 0,
    include_planned_orders INTEGER NOT NULL DEFAULT 1,
    include_purchase_orders INTEGER NOT NULL DEFAULT 1,
    include_work_orders INTEGER NOT NULL DEFAULT 1,
    include_transfers INTEGER NOT NULL DEFAULT 1,
    partial_shipment_allowed INTEGER NOT NULL DEFAULT 0,
    priority INTEGER NOT NULL DEFAULT 100,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_promising_rules_tenant ON promising_rules(tenant_id, rule_type, priority);
```

### 2.2 Availability Buckets
```sql
CREATE TABLE availability_buckets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    bucket_date TEXT NOT NULL,
    bucket_type TEXT NOT NULL
        CHECK(bucket_type IN ('ON_HAND','PLANNED_RECEIPT','PURCHASE_ORDER','WORK_ORDER','TRANSFER_IN',
                              'SALES_ORDER','TRANSFER_OUT','SAFETY_STOCK','ALLOCATED')),
    quantity REAL NOT NULL DEFAULT 0,
    source_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, item_id, warehouse_id, bucket_date, bucket_type, source_reference)
);

CREATE INDEX idx_avail_buckets_item ON availability_buckets(tenant_id, item_id, warehouse_id, bucket_date);
CREATE INDEX idx_avail_buckets_date ON availability_buckets(tenant_id, bucket_date);
```

### 2.3 Supply Demands
```sql
CREATE TABLE supply_demands (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    record_type TEXT NOT NULL CHECK(record_type IN ('SUPPLY','DEMAND')),
    source_type TEXT NOT NULL
        CHECK(source_type IN ('SALES_ORDER','PURCHASE_ORDER','WORK_ORDER','TRANSFER','FORECAST','SAFETY_STOCK','ON_HAND')),
    source_id TEXT NOT NULL,
    quantity REAL NOT NULL,
    date_required TEXT NOT NULL,
    date_promised TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','PARTIALLY_FULFILLED','FULFILLED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, item_id, warehouse_id, source_type, source_id)
);

CREATE INDEX idx_supply_demands_item ON supply_demands(tenant_id, item_id, warehouse_id, date_required);
CREATE INDEX idx_supply_demands_type ON supply_demands(tenant_id, record_type, source_type);
```

### 2.4 Promising Results
```sql
CREATE TABLE promising_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_number TEXT NOT NULL,
    item_id TEXT NOT NULL,
    warehouse_id TEXT NOT NULL,
    requested_quantity REAL NOT NULL,
    requested_date TEXT NOT NULL,
    promised_quantity REAL NOT NULL,
    promised_date TEXT,
    promising_method TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PROMISED'
        CHECK(status IN ('PROMISED','PARTIAL','NOT_AVAILABLE','EXPIRED','COMMITTED')),
    confidence_level TEXT NOT NULL DEFAULT 'HIGH'
        CHECK(confidence_level IN ('HIGH','MEDIUM','LOW')),
    substitution_item_id TEXT,
    allocation_references TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, request_number)
);

CREATE INDEX idx_promising_results_tenant ON promising_results(tenant_id, item_id, status);
CREATE INDEX idx_promising_results_date ON promising_results(tenant_id, promised_date);
```

### 2.5 Substitution Rules
```sql
CREATE TABLE substitution_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    primary_item_id TEXT NOT NULL,
    substitute_item_id TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 1,
    substitution_type TEXT NOT NULL
        CHECK(substitution_type IN ('ONE_WAY','TWO_WAY','UPGRADE','DOWNGRADE')),
    quantity_ratio REAL NOT NULL DEFAULT 1.0,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, primary_item_id, substitute_item_id)
);

CREATE INDEX idx_substitution_rules_primary ON substitution_rules(tenant_id, primary_item_id, priority);
```

### 2.6 Allocation Rules
```sql
CREATE TABLE allocation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    item_id TEXT,
    item_category_id TEXT,
    allocation_method TEXT NOT NULL
        CHECK(allocation_method IN ('PERCENTAGE','PRIORITY','FAIR_SHARE','FIRST_COME_FIRST_SERVED')),
    total_available_quantity REAL,
    priority INTEGER NOT NULL DEFAULT 100,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_allocation_rules_tenant ON allocation_rules(tenant_id, is_active, priority);
```

### 2.7 Allocation Details
```sql
CREATE TABLE allocation_details (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id NOT NULL,
    rule_id TEXT NOT NULL,
    channel TEXT,
    customer_id TEXT,
    customer_group TEXT,
    allocated_pct REAL NOT NULL,
    allocated_quantity REAL,
    priority INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (rule_id) REFERENCES allocation_rules(id) ON DELETE CASCADE
);

CREATE INDEX idx_allocation_details_rule ON allocation_details(rule_id, priority);
```

---

## 3. REST API Endpoints

### 3.1 Promising Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/promising/rules` | List promising rules |
| POST | `/api/v1/promising/rules` | Create promising rule |
| GET | `/api/v1/promising/rules/{id}` | Get rule |
| PUT | `/api/v1/promising/rules/{id}` | Update rule |
| DELETE | `/api/v1/promising/rules/{id}` | Deactivate rule |

### 3.2 Availability
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/promising/check-availability` | Check ATP/CTP for item(s) |
| GET | `/api/v1/promising/availability/{item_id}` | Get availability timeline |
| POST | `/api/v1/promising/availability/refresh` | Refresh availability buckets |

### 3.3 Promising
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/promising/promise` | Promise delivery date for order line |
| POST | `/api/v1/promising/promise/batch` | Batch promise for multiple lines |
| GET | `/api/v1/promising/results/{id}` | Get promising result |
| GET | `/api/v1/promising/results` | List promising results |
| POST | `/api/v1/promising/results/{id}/validate` | Validate existing promise |
| POST | `/api/v1/promising/results/{id}/cancel` | Cancel a promise |

### 3.4 Substitutions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/promising/substitutions` | List substitution rules |
| POST | `/api/v1/promising/substitutions` | Create substitution rule |
| PUT | `/api/v1/promising/substitutions/{id}` | Update rule |
| DELETE | `/api/v1/promising/substitutions/{id}` | Deactivate rule |

### 3.5 Allocations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/promising/allocations` | List allocation rules |
| POST | `/api/v1/promising/allocations` | Create allocation rule |
| PUT | `/api/v1/promising/allocations/{id}` | Update allocation |
| POST | `/api/v1/promising/allocations/{id}/run` | Run allocation |

### 3.6 What-If Scenarios
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/promising/scenarios` | Create what-if scenario |
| GET | `/api/v1/promising/scenarios/{id}` | Get scenario results |

---

## 4. Business Rules

1. Promising rules MUST be evaluated in priority order when checking availability.
2. ATP MUST calculate available quantity as: on-hand + planned receipts - allocated - safety stock - open demand.
3. CTP MUST check manufacturing capacity and lead times to determine feasible delivery dates.
4. Forward promising MUST find the earliest date when quantity is available.
5. Backward promising MUST find the latest order date to meet a requested delivery date.
6. Substitution items MUST be offered only when primary item is unavailable and substitution rules exist.
7. Partial shipment promises MUST be allowed only when the promising rule permits it.
8. Promised dates MUST include configured lead time offsets for shipping and processing.
9. Allocation rules MUST limit total commitments per channel/customer to prevent over-commitment.
10. Promises MUST expire if not converted to orders within the configured time window.
11. Safety stock MUST be excluded from available-to-promise calculations unless overridden.
12. Each promising result MUST record the rule and method used for audit purposes.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.promising.v1;

service PromisingService {
    rpc CheckAvailability(CheckAvailabilityRequest) returns (AvailabilityResponse);
    rpc PromiseDeliveryDate(PromiseRequest) returns (PromisingResultResponse);
    rpc BatchPromise(BatchPromiseRequest) returns (BatchPromiseResponse);
    rpc ValidatePromise(ValidatePromiseRequest) returns (ValidatePromiseResponse);
    rpc CancelPromise(CancelPromiseRequest) returns (CancelPromiseResponse);
    rpc GetPromisingResult(GetPromisingResultRequest) returns (PromisingResultResponse);
    rpc RefreshAvailability(RefreshAvailabilityRequest) returns (RefreshAvailabilityResponse);
    rpc CreateScenario(CreateScenarioRequest) returns (ScenarioResponse);
    rpc GetScenario(GetScenarioRequest) returns (ScenarioResponse);
}

// Entity messages

message PromisingRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string item_category_id = 5;
    string channel = 6;
    string warehouse_id = 7;
    string promising_method = 8;
    int32 lead_time_days = 9;
    int32 safety_stock_offset = 10;
    bool include_planned_orders = 11;
    bool include_purchase_orders = 12;
    bool include_work_orders = 13;
    bool include_transfers = 14;
    bool partial_shipment_allowed = 15;
    int32 priority = 16;
    bool is_default = 17;
    string created_at = 18;
    string updated_at = 19;
}

message AvailabilityBucket {
    string id = 1;
    string tenant_id = 2;
    string item_id = 3;
    string warehouse_id = 4;
    string bucket_date = 5;
    string bucket_type = 6;
    double quantity = 7;
    string source_reference = 8;
    string created_at = 9;
    string updated_at = 10;
}

message SupplyDemand {
    string id = 1;
    string tenant_id = 2;
    string item_id = 3;
    string warehouse_id = 4;
    string record_type = 5;
    string source_type = 6;
    string source_id = 7;
    double quantity = 8;
    string date_required = 9;
    string date_promised = 10;
    string status = 11;
    string created_at = 12;
    string updated_at = 13;
}

message PromisingResult {
    string id = 1;
    string tenant_id = 2;
    string request_number = 3;
    string item_id = 4;
    string warehouse_id = 5;
    double requested_quantity = 6;
    string requested_date = 7;
    double promised_quantity = 8;
    string promised_date = 9;
    string promising_method = 10;
    string rule_id = 11;
    string status = 12;
    string confidence_level = 13;
    string substitution_item_id = 14;
    string allocation_references = 15;
    string created_at = 16;
    string updated_at = 17;
}

message SubstitutionRule {
    string id = 1;
    string tenant_id = 2;
    string primary_item_id = 3;
    string substitute_item_id = 4;
    int32 priority = 5;
    string substitution_type = 6;
    double quantity_ratio = 7;
    string effective_from = 8;
    string effective_to = 9;
    string created_at = 10;
    string updated_at = 11;
}

message AllocationRule {
    string id = 1;
    string tenant_id = 2;
    string rule_name = 3;
    string item_id = 4;
    string item_category_id = 5;
    string allocation_method = 6;
    double total_available_quantity = 7;
    int32 priority = 8;
    string created_at = 9;
    string updated_at = 10;
}

message AllocationDetail {
    string id = 1;
    string tenant_id = 2;
    string rule_id = 3;
    string channel = 4;
    string customer_id = 5;
    string customer_group = 6;
    double allocated_pct = 7;
    double allocated_quantity = 8;
    int32 priority = 9;
    string created_at = 10;
    string updated_at = 11;
}

// Request/Response messages

message CheckAvailabilityRequest {
    string tenant_id = 1;
    string item_id = 2;
    string warehouse_id = 3;
    double quantity = 4;
    string requested_date = 5;
    string rule_type = 6;
}

message AvailabilityResponse {
    string item_id = 1;
    string warehouse_id = 2;
    double available_quantity = 3;
    string available_date = 4;
    repeated AvailabilityBucket buckets = 5;
}

message PromiseRequest {
    string tenant_id = 1;
    string item_id = 2;
    string warehouse_id = 3;
    double requested_quantity = 4;
    string requested_date = 5;
    string rule_id = 6;
    bool allow_substitution = 7;
}

message PromisingResultResponse {
    PromisingResult data = 1;
}

message BatchPromiseRequest {
    string tenant_id = 1;
    repeated PromiseRequest lines = 2;
}

message BatchPromiseResponse {
    repeated PromisingResult results = 1;
}

message ValidatePromiseRequest {
    string tenant_id = 1;
    string result_id = 2;
}

message ValidatePromiseResponse {
    bool is_valid = 1;
    PromisingResult data = 2;
    string validation_message = 3;
}

message CancelPromiseRequest {
    string tenant_id = 1;
    string result_id = 2;
    string reason = 3;
}

message CancelPromiseResponse {
    string result_id = 1;
    string status = 2;
}

message GetPromisingResultRequest {
    string tenant_id = 1;
    string id = 2;
}

message RefreshAvailabilityRequest {
    string tenant_id = 1;
    string item_id = 2;
    string warehouse_id = 3;
}

message RefreshAvailabilityResponse {
    int32 buckets_updated = 1;
    string refreshed_at = 2;
}

message CreateScenarioRequest {
    string tenant_id = 1;
    string name = 2;
    repeated PromiseRequest lines = 3;
    string description = 4;
}

message ScenarioResponse {
    string scenario_id = 1;
    string name = 2;
    repeated PromisingResult results = 3;
    string status = 4;
}

message GetScenarioRequest {
    string tenant_id = 1;
    string scenario_id = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `inv-service` | On-hand inventory balances, warehouse details |
| `mfg-service` | Work orders, production schedules, capacity data |
| `proc-service` | Purchase orders, expected receipt dates |
| `om-service` | Sales orders (demand), order lines |
| `planning-service` | Planned orders, demand forecasts |
| `pricing-service` | Item pricing for substitution cost comparison |

### Published To
| Service | Data |
|---------|------|
| `om-service` | Promised delivery dates, availability confirmations |
| `planning-service` | Unfulfilled demand signals |
| `report-service` | Promise accuracy metrics, fill rate analytics |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `promising.availability.checked` | `{item_id, warehouse_id, available_qty, date}` | Availability checked |
| `promising.promise.created` | `{result_id, item_id, quantity, promised_date, method}` | Delivery date promised |
| `promising.promise.cancelled` | `{result_id, reason}` | Promise cancelled |
| `promising.promise.expired` | `{result_id, original_date}` | Promise expired |
| `promising.substitution.applied` | `{result_id, original_item, substitute_item}` | Substitution used |
| `promising.allocation.run` | `{rule_id, total_allocated, remaining}` | Allocation executed |
| `promising.scenario.completed` | `{scenario_id, summary}` | What-if scenario finished |
