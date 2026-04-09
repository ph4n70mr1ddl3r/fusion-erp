# 208 - Utilities Industry Solution Specification

## 1. Domain Overview

Utilities provides industry-specific capabilities for utility billing, meter management, tariff management, outage tracking, and energy trading operations. Supports customer billing for electric, gas, water, and telecommunications utilities with complex tariff structures, meter reading and consumption tracking, outage management with crew dispatch, regulatory rate case management, demand response program administration, and energy trading and risk management. Enables utility companies to manage the full customer-to-cash cycle, maintain grid reliability, and comply with regulatory requirements. Integrates with Financials, Customer Service, and Asset Management.

**Bounded Context:** Utility Operations, Billing & Grid Management
**Service Name:** `utilities-service`
**Database:** `data/utilities.db`
**HTTP Port:** 8226 | **gRPC Port:** 9226

---

## 2. Database Schema

### 2.1 Service Agreements
```sql
CREATE TABLE ut_service_agreements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_number TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    customer_name TEXT NOT NULL,
    service_type TEXT NOT NULL CHECK(service_type IN ('ELECTRIC','GAS','WATER','TELECOM','STEAM')),
    premise_address TEXT NOT NULL,
    meter_id TEXT,
    rate_schedule TEXT NOT NULL,
    contract_start TEXT NOT NULL,
    contract_end TEXT,
    billing_cycle TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(billing_cycle IN ('MONTHLY','BI_MONTHLY','QUARTERLY')),
    average_consumption REAL NOT NULL DEFAULT 0,
    budget_billing_flag INTEGER NOT NULL DEFAULT 0,
    autopay_flag INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUSPENDED','DISCONNECTED','CLOSED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, agreement_number)
);

CREATE INDEX idx_ut_sa_customer ON ut_service_agreements(customer_id);
CREATE INDEX idx_ut_sa_type ON ut_service_agreements(service_type, status);
CREATE INDEX idx_ut_sa_meter ON ut_service_agreements(meter_id);
```

### 2.2 Meter Management
```sql
CREATE TABLE ut_meters (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    meter_number TEXT NOT NULL,
    meter_type TEXT NOT NULL CHECK(meter_type IN ('ELECTRIC','GAS','WATER','SMART','INTERVAL')),
    manufacturer TEXT,
    model TEXT,
    install_date TEXT NOT NULL,
    last_calibration_date TEXT,
    next_calibration_date TEXT,
    location TEXT NOT NULL,
    reading_frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(reading_frequency IN ('HOURLY','DAILY','MONTHLY','QUARTERLY')),
    communication_type TEXT CHECK(communication_type IN ('AMI','AMR','MANUAL','CELLULAR')),
    current_reading REAL NOT NULL DEFAULT 0,
    previous_reading REAL NOT NULL DEFAULT 0,
    multiplier REAL NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','DISCONNECTED','DEFECTIVE','RETIRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, meter_number)
);

CREATE INDEX idx_ut_meter_type ON ut_meters(meter_type, status);
CREATE INDEX idx_ut_meter_calibration ON ut_meters(next_calibration_date);
```

### 2.3 Billing & Tariffs
```sql
CREATE TABLE ut_bills (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bill_number TEXT NOT NULL,
    agreement_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    billing_period_start TEXT NOT NULL,
    billing_period_end TEXT NOT NULL,
    consumption REAL NOT NULL DEFAULT 0,
    demand_kw REAL,
    rate_schedule TEXT NOT NULL,
    energy_charges_cents INTEGER NOT NULL DEFAULT 0,
    demand_charges_cents INTEGER NOT NULL DEFAULT 0,
    fixed_charges_cents INTEGER NOT NULL DEFAULT 0,
    taxes_cents INTEGER NOT NULL DEFAULT 0,
    fees_cents INTEGER NOT NULL DEFAULT 0,
    total_charges_cents INTEGER NOT NULL DEFAULT 0,
    payments_cents INTEGER NOT NULL DEFAULT 0,
    balance_due_cents INTEGER NOT NULL DEFAULT 0,
    due_date TEXT NOT NULL,
    paid_at TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','PAID','OVERDUE','DISPUTED','WRITTEN_OFF')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, bill_number)
);

CREATE INDEX idx_ut_bill_customer ON ut_bills(customer_id, status);
CREATE INDEX idx_ut_bill_period ON ut_bills(billing_period_start, billing_period_end);
CREATE INDEX idx_ut_bill_due ON ut_bills(due_date, status);
```

### 2.4 Outage Management
```sql
CREATE TABLE ut_outages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    outage_number TEXT NOT NULL,
    outage_type TEXT NOT NULL CHECK(outage_type IN ('PLANNED','UNPLANNED','EMERGENCY','ROTATING')),
    service_type TEXT NOT NULL,
    affected_area TEXT NOT NULL,
    affected_customers INTEGER NOT NULL DEFAULT 0,
    reported_at TEXT NOT NULL,
    cause TEXT,
    crew_dispatched TEXT,                         -- JSON: crew IDs
    estimated_restoration TEXT,
    actual_restoration TEXT,
    duration_minutes INTEGER NOT NULL DEFAULT 0,
    restoration_steps TEXT,                       -- JSON: timeline of actions
    customer_notifications TEXT,                  -- JSON: notifications sent
    status TEXT NOT NULL DEFAULT 'REPORTED'
        CHECK(status IN ('REPORTED','DISPATCHED','IN_PROGRESS','RESTORED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, outage_number)
);

CREATE INDEX idx_ut_outage_status ON ut_outages(tenant_id, status);
CREATE INDEX idx_ut_outage_type ON ut_outages(outage_type);
CREATE INDEX idx_ut_outage_reported ON ut_outages(reported_at DESC);
```

### 2.5 Rate Schedules
```sql
CREATE TABLE ut_rate_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    schedule_code TEXT NOT NULL,
    schedule_name TEXT NOT NULL,
    service_type TEXT NOT NULL,
    customer_class TEXT NOT NULL CHECK(customer_class IN ('RESIDENTIAL','COMMERCIAL','INDUSTRIAL','GOVERNMENT')),
    rate_components TEXT NOT NULL,                 -- JSON: [{type, tier_from, tier_to, rate_per_unit}]
    demand_charge_per_kw INTEGER NOT NULL DEFAULT 0,
    fixed_charge_cents INTEGER NOT NULL DEFAULT 0,
    time_of_use TEXT,                             -- JSON: TOU periods and rates
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    regulatory_approval TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('DRAFT','ACTIVE','SUPERSEDED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, schedule_code)
);

CREATE INDEX idx_ut_rate_type ON ut_rate_schedules(service_type, customer_class);
CREATE INDEX idx_ut_rate_dates ON ut_rate_schedules(effective_from, effective_to);
```

---

## 3. API Endpoints

### 3.1 Service Agreements
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/utilities/agreements` | Create agreement |
| GET | `/api/v1/utilities/agreements` | List agreements |
| GET | `/api/v1/utilities/agreements/{id}` | Get agreement |
| PUT | `/api/v1/utilities/agreements/{id}` | Update agreement |
| POST | `/api/v1/utilities/agreements/{id}/transfer` | Transfer service |

### 3.2 Meters
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/utilities/meters` | Register meter |
| GET | `/api/v1/utilities/meters` | List meters |
| POST | `/api/v1/utilities/meters/{id}/reading` | Submit reading |
| GET | `/api/v1/utilities/meters/{id}/history` | Reading history |

### 3.3 Billing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/utilities/bills/generate` | Generate bills |
| GET | `/api/v1/utilities/bills` | List bills |
| GET | `/api/v1/utilities/bills/{id}` | Get bill |
| POST | `/api/v1/utilities/bills/{id}/dispute` | Dispute bill |

### 3.4 Outages
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/utilities/outages` | Report outage |
| GET | `/api/v1/utilities/outages` | List outages |
| GET | `/api/v1/utilities/outages/{id}` | Get outage |
| POST | `/api/v1/utilities/outages/{id}/dispatch` | Dispatch crew |
| POST | `/api/v1/utilities/outages/{id}/restore` | Mark restored |

### 3.5 Rates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/utilities/rates` | Create rate schedule |
| GET | `/api/v1/utilities/rates` | List rate schedules |
| GET | `/api/v1/utilities/rates/{id}` | Get rate schedule |
| PUT | `/api/v1/utilities/rates/{id}` | Update rate schedule |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `utility.bill.generated` | `{ bill_id, customer, amount }` | Bill generated |
| `utility.outage.reported` | `{ outage_id, area, customers }` | Outage reported |
| `utility.outage.restored` | `{ outage_id, duration_minutes }` | Service restored |
| `utility.meter.reading` | `{ meter_id, reading, consumption }` | Meter reading received |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `payment.received` | Payment (14) | Apply to bill |
| `asset.maintenance.completed` | Asset Mgmt (40) | Update meter status |
| `customer.complaint` | Customer Service (81) | Create outage ticket |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fusion.utilities.v1;

import "google/protobuf/timestamp.proto";

// ── Service ──────────────────────────────────────────────────────────
service UtilitiesService {
  // Service Agreements
  rpc CreateServiceAgreement(CreateServiceAgreementRequest) returns (ServiceAgreement);
  rpc GetServiceAgreement(GetServiceAgreementRequest) returns (ServiceAgreement);
  rpc ListServiceAgreements(ListServiceAgreementsRequest) returns (ListServiceAgreementsResponse);

  // Meters
  rpc RegisterMeter(RegisterMeterRequest) returns (Meter);
  rpc SubmitMeterReading(SubmitMeterReadingRequest) returns (Meter);

  // Billing
  rpc GenerateBills(GenerateBillsRequest) returns (GenerateBillsResponse);
  rpc GetBill(GetBillRequest) returns (Bill);

  // Outages
  rpc ReportOutage(ReportOutageRequest) returns (Outage);
  rpc RestoreOutage(RestoreOutageRequest) returns (Outage);

  // Rate Schedules
  rpc CreateRateSchedule(CreateRateScheduleRequest) returns (RateSchedule);
  rpc ListRateSchedules(ListRateSchedulesRequest) returns (ListRateSchedulesResponse);
}

// ── Enums ────────────────────────────────────────────────────────────
enum ServiceType {
  SERVICE_TYPE_UNSPECIFIED = 0;
  SERVICE_TYPE_ELECTRIC = 1;
  SERVICE_TYPE_GAS = 2;
  SERVICE_TYPE_WATER = 3;
  SERVICE_TYPE_TELECOM = 4;
  SERVICE_TYPE_STEAM = 5;
}

enum AgreementStatus {
  AGREEMENT_STATUS_UNSPECIFIED = 0;
  AGREEMENT_STATUS_ACTIVE = 1;
  AGREEMENT_STATUS_SUSPENDED = 2;
  AGREEMENT_STATUS_DISCONNECTED = 3;
  AGREEMENT_STATUS_CLOSED = 4;
}

enum BillingCycle {
  BILLING_CYCLE_UNSPECIFIED = 0;
  BILLING_CYCLE_MONTHLY = 1;
  BILLING_CYCLE_BI_MONTHLY = 2;
  BILLING_CYCLE_QUARTERLY = 3;
}

enum MeterType {
  METER_TYPE_UNSPECIFIED = 0;
  METER_TYPE_ELECTRIC = 1;
  METER_TYPE_GAS = 2;
  METER_TYPE_WATER = 3;
  METER_TYPE_SMART = 4;
  METER_TYPE_INTERVAL = 5;
}

enum MeterStatus {
  METER_STATUS_UNSPECIFIED = 0;
  METER_STATUS_ACTIVE = 1;
  METER_STATUS_DISCONNECTED = 2;
  METER_STATUS_DEFECTIVE = 3;
  METER_STATUS_RETIRED = 4;
}

enum CommunicationType {
  COMM_TYPE_UNSPECIFIED = 0;
  COMM_TYPE_AMI = 1;
  COMM_TYPE_AMR = 2;
  COMM_TYPE_MANUAL = 3;
  COMM_TYPE_CELLULAR = 4;
}

enum ReadingFrequency {
  READING_FREQ_UNSPECIFIED = 0;
  READING_FREQ_HOURLY = 1;
  READING_FREQ_DAILY = 2;
  READING_FREQ_MONTHLY = 3;
  READING_FREQ_QUARTERLY = 4;
}

enum BillStatus {
  BILL_STATUS_UNSPECIFIED = 0;
  BILL_STATUS_OPEN = 1;
  BILL_STATUS_PAID = 2;
  BILL_STATUS_OVERDUE = 3;
  BILL_STATUS_DISPUTED = 4;
  BILL_STATUS_WRITTEN_OFF = 5;
}

enum OutageType {
  OUTAGE_TYPE_UNSPECIFIED = 0;
  OUTAGE_TYPE_PLANNED = 1;
  OUTAGE_TYPE_UNPLANNED = 2;
  OUTAGE_TYPE_EMERGENCY = 3;
  OUTAGE_TYPE_ROTATING = 4;
}

enum OutageStatus {
  OUTAGE_STATUS_UNSPECIFIED = 0;
  OUTAGE_STATUS_REPORTED = 1;
  OUTAGE_STATUS_DISPATCHED = 2;
  OUTAGE_STATUS_IN_PROGRESS = 3;
  OUTAGE_STATUS_RESTORED = 4;
  OUTAGE_STATUS_CANCELLED = 5;
}

enum CustomerClass {
  CUSTOMER_CLASS_UNSPECIFIED = 0;
  CUSTOMER_CLASS_RESIDENTIAL = 1;
  CUSTOMER_CLASS_COMMERCIAL = 2;
  CUSTOMER_CLASS_INDUSTRIAL = 3;
  CUSTOMER_CLASS_GOVERNMENT = 4;
}

enum RateScheduleStatus {
  RATE_STATUS_UNSPECIFIED = 0;
  RATE_STATUS_DRAFT = 1;
  RATE_STATUS_ACTIVE = 2;
  RATE_STATUS_SUPERSEDED = 3;
}

// ── Common Messages ──────────────────────────────────────────────────
message AuditInfo {
  string created_at = 1;
  string updated_at = 2;
  string created_by = 3;
  string updated_by = 4;
  int32 version = 5;
}

// ── Service Agreement Messages ───────────────────────────────────────
message ServiceAgreement {
  string id = 1;
  string tenant_id = 2;
  string agreement_number = 3;
  string customer_id = 4;
  string customer_name = 5;
  ServiceType service_type = 6;
  string premise_address = 7;
  string meter_id = 8;
  string rate_schedule = 9;
  string contract_start = 10;
  string contract_end = 11;
  BillingCycle billing_cycle = 12;
  double average_consumption = 13;
  bool budget_billing_flag = 14;
  bool autopay_flag = 15;
  AgreementStatus status = 16;
  AuditInfo audit = 17;
}

message CreateServiceAgreementRequest {
  string tenant_id = 1;
  string agreement_number = 2;
  string customer_id = 3;
  string customer_name = 4;
  ServiceType service_type = 5;
  string premise_address = 6;
  string meter_id = 7;
  string rate_schedule = 8;
  string contract_start = 9;
  string contract_end = 10;
  BillingCycle billing_cycle = 11;
  string user_id = 12;
}

message GetServiceAgreementRequest {
  string id = 1;
  string tenant_id = 2;
}

message ListServiceAgreementsRequest {
  string tenant_id = 1;
  ServiceType service_type = 2;
  AgreementStatus status = 3;
  string customer_id = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListServiceAgreementsResponse {
  repeated ServiceAgreement agreements = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

// ── Meter Messages ───────────────────────────────────────────────────
message Meter {
  string id = 1;
  string tenant_id = 2;
  string meter_number = 3;
  MeterType meter_type = 4;
  string manufacturer = 5;
  string model = 6;
  string install_date = 7;
  string last_calibration_date = 8;
  string next_calibration_date = 9;
  string location = 10;
  ReadingFrequency reading_frequency = 11;
  CommunicationType communication_type = 12;
  double current_reading = 13;
  double previous_reading = 14;
  double multiplier = 15;
  MeterStatus status = 16;
  AuditInfo audit = 17;
}

message RegisterMeterRequest {
  string tenant_id = 1;
  string meter_number = 2;
  MeterType meter_type = 3;
  string manufacturer = 4;
  string model = 5;
  string install_date = 6;
  string location = 7;
  ReadingFrequency reading_frequency = 8;
  CommunicationType communication_type = 9;
  double multiplier = 10;
  string user_id = 11;
}

message SubmitMeterReadingRequest {
  string meter_id = 1;
  string tenant_id = 2;
  double reading = 3;
  string user_id = 4;
}

// ── Bill Messages ────────────────────────────────────────────────────
message Bill {
  string id = 1;
  string tenant_id = 2;
  string bill_number = 3;
  string agreement_id = 4;
  string customer_id = 5;
  string billing_period_start = 6;
  string billing_period_end = 7;
  double consumption = 8;
  double demand_kw = 9;
  string rate_schedule = 10;
  int64 energy_charges_cents = 11;
  int64 demand_charges_cents = 12;
  int64 fixed_charges_cents = 13;
  int64 taxes_cents = 14;
  int64 fees_cents = 15;
  int64 total_charges_cents = 16;
  int64 payments_cents = 17;
  int64 balance_due_cents = 18;
  string due_date = 19;
  string paid_at = 20;
  BillStatus status = 21;
  string created_at = 22;
  string updated_at = 23;
}

message GenerateBillsRequest {
  string tenant_id = 1;
  string billing_period_start = 2;
  string billing_period_end = 3;
  repeated string agreement_ids = 4;
  string user_id = 5;
}

message GenerateBillsResponse {
  repeated Bill bills = 1;
  int32 total_generated = 2;
}

message GetBillRequest {
  string id = 1;
  string tenant_id = 2;
}

// ── Outage Messages ──────────────────────────────────────────────────
message Outage {
  string id = 1;
  string tenant_id = 2;
  string outage_number = 3;
  OutageType outage_type = 4;
  ServiceType service_type = 5;
  string affected_area = 6;
  int32 affected_customers = 7;
  string reported_at = 8;
  string cause = 9;
  string crew_dispatched = 10;         // JSON
  string estimated_restoration = 11;
  string actual_restoration = 12;
  int32 duration_minutes = 13;
  string restoration_steps = 14;       // JSON
  string customer_notifications = 15;  // JSON
  OutageStatus status = 16;
  string created_at = 17;
  string updated_at = 18;
}

message ReportOutageRequest {
  string tenant_id = 1;
  OutageType outage_type = 2;
  ServiceType service_type = 3;
  string affected_area = 4;
  int32 affected_customers = 5;
  string estimated_restoration = 6;
  string user_id = 7;
}

message RestoreOutageRequest {
  string id = 1;
  string tenant_id = 2;
  string actual_restoration = 3;
  string restoration_steps = 4;        // JSON
  string user_id = 5;
}

// ── Rate Schedule Messages ───────────────────────────────────────────
message RateSchedule {
  string id = 1;
  string tenant_id = 2;
  string schedule_code = 3;
  string schedule_name = 4;
  ServiceType service_type = 5;
  CustomerClass customer_class = 6;
  string rate_components = 7;          // JSON
  int64 demand_charge_per_kw_cents = 8;
  int64 fixed_charge_cents = 9;
  string time_of_use = 10;             // JSON
  string effective_from = 11;
  string effective_to = 12;
  string regulatory_approval = 13;
  RateScheduleStatus status = 14;
  AuditInfo audit = 15;
}

message CreateRateScheduleRequest {
  string tenant_id = 1;
  string schedule_code = 2;
  string schedule_name = 3;
  ServiceType service_type = 4;
  CustomerClass customer_class = 5;
  string rate_components = 6;          // JSON
  int64 demand_charge_per_kw_cents = 7;
  int64 fixed_charge_cents = 8;
  string time_of_use = 9;              // JSON
  string effective_from = 10;
  string effective_to = 11;
  string regulatory_approval = 12;
  string user_id = 13;
}

message ListRateSchedulesRequest {
  string tenant_id = 1;
  ServiceType service_type = 2;
  CustomerClass customer_class = 3;
  RateScheduleStatus status = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListRateSchedulesResponse {
  repeated RateSchedule rate_schedules = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}
```

## 6. Migration Order

| Order | Table | Depends On |
|-------|-------|------------|
| 1 | `ut_service_agreements` | — |
| 2 | `ut_meters` | — |
| 3 | `ut_rate_schedules` | — |
| 4 | `ut_bills` | `ut_service_agreements` |
| 5 | `ut_outages` | — |

---

## 7. Business Rules

1. **Bill Calculation**: Bills calculated from meter readings using applicable rate schedule and tier structure
2. **Estimated Reads**: Missing meter readings estimated from historical consumption patterns
3. **Outage Response**: Unplanned outages require crew dispatch within SLA targets
4. **Rate Changes**: Rate schedule changes require regulatory approval reference
5. **Budget Billing**: Annual consumption averaged into equal monthly payments
6. **Demand Response**: Smart meter customers eligible for demand response incentives

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Accounts Receivable (08) | Invoice and payment processing |
| Customer Service (81) | Customer complaint handling |
| Asset Management (40) | Meter and grid asset tracking |
| Financials (06) | Revenue recognition |
| Notification Center (165) | Outage and billing alerts |
| Enterprise Scheduler (173) | Bill generation scheduling |
