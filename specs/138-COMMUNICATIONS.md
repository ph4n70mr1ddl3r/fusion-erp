# 138 - Communications Specification

## 1. Domain Overview

Communications provides industry-specific capabilities for telecommunications and media companies, including customer billing (subscription + usage), network inventory management, service provisioning, rating & charging, mediation, and revenue assurance. It extends the Fusion platform with telecom-specific billing models and network asset management.

**Bounded Context:** Telecommunications & Media Industry
**Service Name:** `comms-service`
**Database:** `data/comms.db`
**HTTP Port:** 8218 | **gRPC Port:** 9218

---

## 2. Database Schema

### 2.1 Service Plans
```sql
CREATE TABLE comms_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('MOBILE','BROADBAND','TV','LANDLINE','BUNDLE','IOT','ENTERPRISE')),
    description TEXT,
    base_price_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    billing_cycle TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(billing_cycle IN ('MONTHLY','QUARTERLY','ANNUAL','WEEKLY')),
    contract_duration_months INTEGER,
    included_allowances TEXT NOT NULL,    -- JSON: { "data_gb": 50, "minutes": 1000, "sms": 500 }
    overage_rates TEXT,                  -- JSON: { "data_per_gb_cents": 1000, "extra_minute_cents": 10 }
    features TEXT,                       -- JSON: ["5G", "WiFi_calling", "Roaming"]
    promotion_id TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','GRANDFATHERED','DISCONTINUED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);
```

### 2.2 Customer Subscriptions
```sql
CREATE TABLE comms_subscriptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    subscription_number TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PROVISIONING'
        CHECK(status IN ('PROVISIONING','ACTIVE','SUSPENDED','CANCELLED','PORTED_OUT')),
    start_date TEXT NOT NULL,
    end_date TEXT,
    contract_end_date TEXT,
    phone_number TEXT,
    sim_number TEXT,
    device_id TEXT,
    network_identifier TEXT,             -- IMSI, IMEI, or broadband circuit ID
    auto_renew INTEGER NOT NULL DEFAULT 1,
    bundle_id TEXT,                      -- For bundled subscriptions
    activation_date TEXT,
    cancellation_date TEXT,
    cancellation_reason TEXT,
    port_out_date TEXT,
    last_billed_date TEXT,
    next_bill_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES comms_plans(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, subscription_number)
);

CREATE INDEX idx_comms_subs_tenant_customer ON comms_subscriptions(tenant_id, customer_id);
CREATE INDEX idx_comms_subs_tenant_status ON comms_subscriptions(tenant_id, status);
CREATE INDEX idx_comms_subs_tenant_phone ON comms_subscriptions(tenant_id, phone_number);
```

### 2.3 Usage Records
```sql
CREATE TABLE comms_usage_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    subscription_id TEXT NOT NULL,
    usage_type TEXT NOT NULL CHECK(usage_type IN ('VOICE','DATA','SMS','MMS','ROAMING_VOICE','ROAMING_DATA','ROAMING_SMS','PREMIUM_SERVICE','CONTENT','IOT_DATA')),
    usage_date TEXT NOT NULL,
    start_time TEXT,
    end_time TEXT,
    duration_seconds INTEGER,
    volume_bytes INTEGER,
    quantity INTEGER NOT NULL DEFAULT 1,
    originating_number TEXT,
    destination_number TEXT,
    cell_tower_id TEXT,
    network_type TEXT CHECK(network_type IN ('2G','3G','4G','5G','WIFI')),
    roaming_country TEXT,
    roaming_network TEXT,
    raw_charge_cents INTEGER NOT NULL DEFAULT 0,
    rated_charge_cents INTEGER NOT NULL DEFAULT 0,
    is_included INTEGER NOT NULL DEFAULT 0,
    mediation_source TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (subscription_id) REFERENCES comms_subscriptions(id) ON DELETE RESTRICT
);

CREATE INDEX idx_comms_usage_tenant_sub_date ON comms_usage_records(tenant_id, subscription_id, usage_date);
CREATE INDEX idx_comms_usage_tenant_type ON comms_usage_records(tenant_id, usage_type, usage_date);
```

### 2.4 Billing Cycles
```sql
CREATE TABLE comms_billing_cycles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cycle_name TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    billing_date TEXT NOT NULL,
    due_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','RATING','BILLED','PAID','PAST_DUE','WRITTEN_OFF')),
    total_subscriptions INTEGER NOT NULL DEFAULT 0,
    total_base_charges_cents INTEGER NOT NULL DEFAULT 0,
    total_usage_charges_cents INTEGER NOT NULL DEFAULT 0,
    total_discounts_cents INTEGER NOT NULL DEFAULT 0,
    total_taxes_cents INTEGER NOT NULL DEFAULT 0,
    total_amount_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, cycle_name)
);
```

### 2.5 Network Inventory
```sql
CREATE TABLE comms_network_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_code TEXT NOT NULL,
    asset_name TEXT NOT NULL,
    asset_type TEXT NOT NULL CHECK(asset_type IN ('CELL_TOWER','FIBER_CABLE','ROUTER','SWITCH','ANTENNA','BASE_STATION','CABINET','SPLITTER','ONT','OLT','CPE','OTHER')),
    location TEXT,
    geo_coordinates TEXT,
    address TEXT,
    parent_asset_id TEXT,
    capacity TEXT,                       -- JSON: capacity specifications
    utilized_capacity_pct REAL NOT NULL DEFAULT 0.0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('PLANNED','ACTIVE','MAINTENANCE','DECOMMISSIONED')),
    installation_date TEXT,
    vendor TEXT,
    model TEXT,
    serial_number TEXT,
    coverage_area TEXT,                  -- GeoJSON or description

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_asset_id) REFERENCES comms_network_assets(id) ON DELETE SET NULL,
    UNIQUE(tenant_id, asset_code)
);

CREATE INDEX idx_comms_assets_tenant_type ON comms_network_assets(tenant_id, asset_type);
CREATE INDEX idx_comms_assets_tenant_status ON comms_network_assets(tenant_id, status);
```

### 2.6 Service Orders (Provisioning)
```sql
CREATE TABLE comms_service_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_number TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    subscription_id TEXT,
    order_type TEXT NOT NULL CHECK(order_type IN ('NEW_SERVICE','CHANGE_PLAN','ADD_FEATURE','REMOVE_FEATURE','SUSPEND','RESUME','DISCONNECT','PORT_IN','PORT_OUT','DEVICE_SWAP')),
    status TEXT NOT NULL DEFAULT 'RECEIVED'
        CHECK(status IN ('RECEIVED','VALIDATED','PROVISIONING','ACTIVATED','COMPLETED','FAILED','CANCELLED')),
    requested_date TEXT NOT NULL,
    scheduled_date TEXT,
    completed_date TEXT,
    assigned_team TEXT,
    network_changes TEXT,                -- JSON: provisioning steps and results
    failure_reason TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_comms_orders_tenant_status ON comms_service_orders(tenant_id, status);
CREATE INDEX idx_comms_orders_tenant_customer ON comms_service_orders(tenant_id, customer_id);
```

---

## 3. REST API Endpoints

### 3.1 Plans
```
GET    /api/v1/comms/plans                              Permission: comms.plans.read
GET    /api/v1/comms/plans/{id}                         Permission: comms.plans.read
POST   /api/v1/comms/plans                              Permission: comms.plans.create
PUT    /api/v1/comms/plans/{id}                         Permission: comms.plans.update
```

### 3.2 Subscriptions
```
GET    /api/v1/comms/subscriptions                      Permission: comms.subscriptions.read
GET    /api/v1/comms/subscriptions/{id}                 Permission: comms.subscriptions.read
POST   /api/v1/comms/subscriptions                      Permission: comms.subscriptions.create
PUT    /api/v1/comms/subscriptions/{id}                 Permission: comms.subscriptions.update
POST   /api/v1/comms/subscriptions/{id}/suspend         Permission: comms.subscriptions.suspend
POST   /api/v1/comms/subscriptions/{id}/resume          Permission: comms.subscriptions.resume
POST   /api/v1/comms/subscriptions/{id}/cancel          Permission: comms.subscriptions.cancel
GET    /api/v1/comms/subscriptions/{id}/usage
  ?from=...&to=...&usage_type=DATA                      Permission: comms.usage.read
```

### 3.3 Usage & Rating
```
POST   /api/v1/comms/usage/ingest                       Permission: comms.usage.ingest
  Request: { "records": [{ "subscription_id": "...", "usage_type": "DATA", "volume_bytes": 1048576, ... }] }
POST   /api/v1/comms/usage/rate                         Permission: comms.usage.rate
GET    /api/v1/comms/usage/summary
  ?subscription_id={id}&period=...                      Permission: comms.usage.read
```

### 3.4 Billing
```
POST   /api/v1/comms/billing/run                        Permission: comms.billing.run
  Request: { "billing_date": "2024-01-15" }
GET    /api/v1/comms/billing/cycles                     Permission: comms.billing.read
GET    /api/v1/comms/billing/cycles/{id}                Permission: comms.billing.read
GET    /api/v1/comms/billing/cycles/{id}/invoices       Permission: comms.billing.read
GET    /api/v1/comms/subscriptions/{id}/invoice/latest  Permission: comms.billing.read
```

### 3.5 Network Inventory
```
GET    /api/v1/comms/network/assets                     Permission: comms.network.read
POST   /api/v1/comms/network/assets                     Permission: comms.network.create
PUT    /api/v1/comms/network/assets/{id}                Permission: comms.network.update
GET    /api/v1/comms/network/assets/{id}/coverage       Permission: comms.network.read
GET    /api/v1/comms/network/capacity
  ?asset_type=CELL_TOWER&region={id}                    Permission: comms.network.read
```

### 3.6 Service Orders
```
GET    /api/v1/comms/orders                             Permission: comms.orders.read
POST   /api/v1/comms/orders                             Permission: comms.orders.create
GET    /api/v1/comms/orders/{id}                        Permission: comms.orders.read
POST   /api/v1/comms/orders/{id}/provision              Permission: comms.orders.provision
POST   /api/v1/comms/orders/{id}/cancel                 Permission: comms.orders.cancel
```

---

## 4. Business Rules

### 4.1 Rating & Charging
- Usage rated against plan allowances in real-time or batch
- Included allowance consumed first; overage rates apply after depletion
- Pro-ration for mid-cycle plan changes
- Roaming usage rated per roaming partner agreements
- Premium services rated separately from base plan

### 4.2 Billing
- Billing runs execute per billing cycle schedule
- Invoice = base charges + usage charges + features + taxes - discounts
- Proration calculated for mid-cycle activations and changes
- Late payment fees applied per configured policy
- Revenue recognition follows ASC 606 for subscription revenue

### 4.3 Service Provisioning
- New service orders validated against network capacity
- Number porting follows regulatory requirements (port-in/port-out)
- Device swap updates SIM/IMEI association
- Service suspension limits network access (configurable restrictions)
- Activation confirmation required before subscription goes active

### 4.4 Network Management
- Capacity monitoring: utilization alerts at 70%, 85%, 95%
- Asset hierarchy: tower → antenna → sector → cell
- Coverage analysis based on geographic data
- Maintenance windows scheduled to minimize customer impact

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.comms.v1;

service CommunicationsService {
    rpc IngestUsage(IngestUsageRequest) returns (IngestUsageResponse);
    rpc RateUsage(RateUsageRequest) returns (RateUsageResponse);
    rpc RunBilling(RunBillingRequest) returns (RunBillingResponse);
    rpc ProvisionService(ProvisionServiceRequest) returns (ProvisionServiceResponse);
    rpc GetSubscriptionUsage(GetSubscriptionUsageRequest) returns (GetSubscriptionUsageResponse);
    rpc CheckNetworkCapacity(CheckNetworkCapacityRequest) returns (CheckNetworkCapacityResponse);
}

// Entity messages
message ServicePlan {
    string id = 1;
    string tenant_id = 2;
    string plan_code = 3;
    string plan_name = 4;
    string plan_type = 5;
    string description = 6;
    int64 base_price_cents = 7;
    string currency_code = 8;
    string billing_cycle = 9;
    int32 contract_duration_months = 10;
    string included_allowances = 11;
    string overage_rates = 12;
    string features = 13;
    string promotion_id = 14;
    string effective_from = 15;
    string effective_to = 16;
    string status = 17;
    string created_at = 18;
    string updated_at = 19;
}

message Subscription {
    string id = 1;
    string tenant_id = 2;
    string customer_id = 3;
    string subscription_number = 4;
    string plan_id = 5;
    string status = 6;
    string start_date = 7;
    string end_date = 8;
    string contract_end_date = 9;
    string phone_number = 10;
    string sim_number = 11;
    string device_id = 12;
    string network_identifier = 13;
    int32 auto_renew = 14;
    string bundle_id = 15;
    string activation_date = 16;
    string cancellation_date = 17;
    string cancellation_reason = 18;
    string port_out_date = 19;
    string last_billed_date = 20;
    string next_bill_date = 21;
    string created_at = 22;
    string updated_at = 23;
}

message UsageRecord {
    string id = 1;
    string tenant_id = 2;
    string subscription_id = 3;
    string usage_type = 4;
    string usage_date = 5;
    string start_time = 6;
    string end_time = 7;
    int64 duration_seconds = 8;
    int64 volume_bytes = 9;
    int32 quantity = 10;
    string originating_number = 11;
    string destination_number = 12;
    string cell_tower_id = 13;
    string network_type = 14;
    string roaming_country = 15;
    string roaming_network = 16;
    int64 raw_charge_cents = 17;
    int64 rated_charge_cents = 18;
    int32 is_included = 19;
    string mediation_source = 20;
    string created_at = 21;
}

message BillingCycle {
    string id = 1;
    string tenant_id = 2;
    string cycle_name = 3;
    string period_start = 4;
    string period_end = 5;
    string billing_date = 6;
    string due_date = 7;
    string status = 8;
    int32 total_subscriptions = 9;
    int64 total_base_charges_cents = 10;
    int64 total_usage_charges_cents = 11;
    int64 total_discounts_cents = 12;
    int64 total_taxes_cents = 13;
    int64 total_amount_cents = 14;
    string created_at = 15;
    string updated_at = 16;
}

message NetworkAsset {
    string id = 1;
    string tenant_id = 2;
    string asset_code = 3;
    string asset_name = 4;
    string asset_type = 5;
    string location = 6;
    string geo_coordinates = 7;
    string address = 8;
    string parent_asset_id = 9;
    string capacity = 10;
    double utilized_capacity_pct = 11;
    string status = 12;
    string installation_date = 13;
    string vendor = 14;
    string model = 15;
    string serial_number = 16;
    string coverage_area = 17;
    string created_at = 18;
    string updated_at = 19;
}

message ServiceOrder {
    string id = 1;
    string tenant_id = 2;
    string order_number = 3;
    string customer_id = 4;
    string subscription_id = 5;
    string order_type = 6;
    string status = 7;
    string requested_date = 8;
    string scheduled_date = 9;
    string completed_date = 10;
    string assigned_team = 11;
    string network_changes = 12;
    string failure_reason = 13;
    int32 retry_count = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Request/Response messages
message UsageRecordInput {
    string subscription_id = 1;
    string usage_type = 2;
    string usage_date = 3;
    string start_time = 4;
    string end_time = 5;
    int64 duration_seconds = 6;
    int64 volume_bytes = 7;
    int32 quantity = 8;
    string originating_number = 9;
    string destination_number = 10;
    string cell_tower_id = 11;
    string network_type = 12;
    string roaming_country = 13;
    string roaming_network = 14;
}

message IngestUsageRequest {
    string tenant_id = 1;
    repeated UsageRecordInput records = 2;
}

message IngestUsageResponse {
    int32 ingested_count = 1;
    int32 failed_count = 2;
    repeated string record_ids = 3;
}

message RateUsageRequest {
    string tenant_id = 1;
    string subscription_id = 2;
    string period_start = 3;
    string period_end = 4;
}

message UsageRating {
    string usage_type = 1;
    int64 included_quantity = 2;
    int64 used_quantity = 3;
    int64 overage_quantity = 4;
    int64 included_charge_cents = 5;
    int64 overage_charge_cents = 6;
    int64 total_charge_cents = 7;
}

message RateUsageResponse {
    string subscription_id = 1;
    int64 total_base_charges_cents = 2;
    int64 total_usage_charges_cents = 3;
    int64 total_charges_cents = 4;
    repeated UsageRating ratings = 5;
}

message RunBillingRequest {
    string tenant_id = 1;
    string billing_date = 2;
}

message RunBillingResponse {
    string cycle_id = 1;
    string cycle_name = 2;
    int32 total_subscriptions = 3;
    int64 total_amount_cents = 4;
    string status = 5;
}

message ProvisionServiceRequest {
    string tenant_id = 1;
    string order_id = 2;
    string subscription_id = 3;
    string order_type = 4;
    string scheduled_date = 5;
}

message ProvisionServiceResponse {
    string order_id = 1;
    string subscription_id = 2;
    string status = 3;
    string network_changes = 4;
}

message GetSubscriptionUsageRequest {
    string tenant_id = 1;
    string subscription_id = 2;
    string from_date = 3;
    string to_date = 4;
    string usage_type = 5;
}

message GetSubscriptionUsageResponse {
    string subscription_id = 1;
    repeated UsageRecord records = 2;
    int64 total_raw_charge_cents = 3;
    int64 total_rated_charge_cents = 4;
}

message CheckNetworkCapacityRequest {
    string tenant_id = 1;
    string asset_type = 2;
    string region = 3;
    double utilization_threshold = 4;
}

message CapacityEntry {
    string asset_id = 1;
    string asset_code = 2;
    string asset_name = 3;
    string asset_type = 4;
    double utilized_capacity_pct = 5;
    string status = 6;
    string location = 7;
}

message CheckNetworkCapacityResponse {
    repeated CapacityEntry assets = 1;
    int32 total_assets = 2;
    int32 assets_above_threshold = 3;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **subscription-service**: Subscription lifecycle and billing integration
- **ar-service**: Invoice generation and payment collection
- **rev-service**: Revenue recognition for subscription contracts
- **tax-service**: Telecom-specific tax calculation (FUSF, 911, state/local)
- **cdp-service**: Customer profiles and segmentation
- **customerservice-service**: Support for service issues
- **eam-service**: Network asset maintenance

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `comms.subscription.created` | New subscription activated | subscription_id, plan_id, customer_id |
| `comms.subscription.suspended` | Subscription suspended | subscription_id, reason |
| `comms.subscription.cancelled` | Subscription terminated | subscription_id, cancellation_reason |
| `comms.usage.ingested` | Usage record processed | subscription_id, usage_type, volume |
| `comms.usage.overage` | Allowance exceeded | subscription_id, usage_type, overage_amount |
| `comms.billing.completed` | Billing cycle finished | cycle_id, total_amount |
| `comms.order.provisioned` | Service order completed | order_id, subscription_id |
| `comms.network.capacity.warning` | Network threshold exceeded | asset_id, utilization_pct |

---

## 7. Migrations

### Migration Order for comms-service:
1. V001: `comms_plans`
2. V002: `comms_subscriptions`
3. V003: `comms_usage_records`
4. V004: `comms_billing_cycles`
5. V005: `comms_network_assets`
6. V006: `comms_service_orders`
7. V007: Triggers for `updated_at`
8. V008: Seed data (sample plans, network asset types, usage rating rules)
