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
| POST | `/utilities/v1/agreements` | Create agreement |
| GET | `/utilities/v1/agreements` | List agreements |
| GET | `/utilities/v1/agreements/{id}` | Get agreement |
| PUT | `/utilities/v1/agreements/{id}` | Update agreement |
| POST | `/utilities/v1/agreements/{id}/transfer` | Transfer service |

### 3.2 Meters
| Method | Path | Description |
|--------|------|-------------|
| POST | `/utilities/v1/meters` | Register meter |
| GET | `/utilities/v1/meters` | List meters |
| POST | `/utilities/v1/meters/{id}/reading` | Submit reading |
| GET | `/utilities/v1/meters/{id}/history` | Reading history |

### 3.3 Billing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/utilities/v1/bills/generate` | Generate bills |
| GET | `/utilities/v1/bills` | List bills |
| GET | `/utilities/v1/bills/{id}` | Get bill |
| POST | `/utilities/v1/bills/{id}/dispute` | Dispute bill |

### 3.4 Outages
| Method | Path | Description |
|--------|------|-------------|
| POST | `/utilities/v1/outages` | Report outage |
| GET | `/utilities/v1/outages` | List outages |
| GET | `/utilities/v1/outages/{id}` | Get outage |
| POST | `/utilities/v1/outages/{id}/dispatch` | Dispatch crew |
| POST | `/utilities/v1/outages/{id}/restore` | Mark restored |

### 3.5 Rates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/utilities/v1/rates` | Create rate schedule |
| GET | `/utilities/v1/rates` | List rate schedules |
| GET | `/utilities/v1/rates/{id}` | Get rate schedule |
| PUT | `/utilities/v1/rates/{id}` | Update rate schedule |

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

## 5. Business Rules

1. **Bill Calculation**: Bills calculated from meter readings using applicable rate schedule and tier structure
2. **Estimated Reads**: Missing meter readings estimated from historical consumption patterns
3. **Outage Response**: Unplanned outages require crew dispatch within SLA targets
4. **Rate Changes**: Rate schedule changes require regulatory approval reference
5. **Budget Billing**: Annual consumption averaged into equal monthly payments
6. **Demand Response**: Smart meter customers eligible for demand response incentives

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Accounts Receivable (08) | Invoice and payment processing |
| Customer Service (81) | Customer complaint handling |
| Asset Management (40) | Meter and grid asset tracking |
| Financials (06) | Revenue recognition |
| Notification Center (165) | Outage and billing alerts |
| Enterprise Scheduler (173) | Bill generation scheduling |
