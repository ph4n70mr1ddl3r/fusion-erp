# 142 - Supply Chain Collaboration Service Specification

## 1. Domain Overview

Supply Chain Collaboration enables real-time collaboration between buying organizations and their supply chain partners (suppliers, contract manufacturers, logistics providers). Supports collaborative demand/supply planning, order lifecycle visibility, document sharing, exception management, and performance tracking. Provides supplier workbench for partners to view forecasts, acknowledge orders, submit advance shipping notices, and manage capacity commitments. Integrates with Procurement, Inventory, Warehouse Management, and Demand Management for end-to-end supply network visibility.

**Bounded Context:** Supply Network Collaboration
**Service Name:** `sc-collaboration-service`
**Database:** `data/sc_collaboration.db`
**HTTP Port:** 8222 | **gRPC Port:** 9222

---

## 2. Database Schema

### 2.1 Collaboration Agreements
```sql
CREATE TABLE scc_agreements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    partner_type TEXT NOT NULL CHECK(partner_type IN ('SUPPLIER','CONTRACT_MANUFACTURER','LOGISTICS_PROVIDER','DISTRIBUTOR','CUSTOM')),
    agreement_code TEXT NOT NULL,
    agreement_name TEXT NOT NULL,
    collaboration_scope TEXT NOT NULL,       -- JSON: ["demand_visibility","order_collaboration","capacity_planning","shipment_tracking"]
    data_sharing_rules TEXT NOT NULL,       -- JSON: what data is shared with partner
    notification_preferences TEXT NOT NULL, -- JSON: alert preferences
    access_start_date TEXT NOT NULL,
    access_end_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','SUSPENDED','TERMINATED')),
    sla_response_time_hours INTEGER NOT NULL DEFAULT 24,
    auto_acknowledge_threshold_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, agreement_code)
);

CREATE INDEX idx_scc_agreement_tenant ON scc_agreements(tenant_id, partner_id, status);
```

### 2.2 Collaborative Forecasts
```sql
CREATE TABLE scc_collaborative_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_id TEXT NOT NULL,
    forecast_period TEXT NOT NULL,           -- "2024-Q1"
    item_ids TEXT NOT NULL,                 -- JSON array
    buyer_forecast_quantities TEXT NOT NULL, -- JSON: { item_id: { period: qty } }
    supplier_commit_quantities TEXT,        -- JSON: supplier's commitment
    supplier_comments TEXT,
    variance_pct REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PUBLISHED'
        CHECK(status IN ('PUBLISHED','UNDER_REVIEW','AGREED','DISPUTED','REVISED')),
    revision_number INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (agreement_id) REFERENCES scc_agreements(id) ON DELETE CASCADE
);

CREATE INDEX idx_scc_forecast_agreement ON scc_collaborative_forecasts(agreement_id, forecast_period DESC);
```

### 2.3 Order Collaboration
```sql
CREATE TABLE scc_order_collaboration (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_id TEXT NOT NULL,
    order_id TEXT NOT NULL,                 -- Reference to PO
    order_number TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    collaboration_type TEXT NOT NULL CHECK(collaboration_type IN ('ACKNOWLEDGMENT','STATUS_UPDATE','ASN','INVOICE','DELIVERY_CONFIRMATION')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','ACKNOWLEDGED','IN_PROGRESS','SHIPPED','DELIVERED','EXCEPTION','CANCELLED')),
    partner_acknowledged_at TEXT,
    committed_ship_date TEXT,
    actual_ship_date TEXT,
    tracking_number TEXT,
    carrier TEXT,
    exception_type TEXT CHECK(exception_type IN ('DELAY','SHORTAGE','QUALITY','MISSED_ACKNOWLEDGMENT','OTHER')),
    exception_notes TEXT,
    exception_resolution TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (agreement_id) REFERENCES scc_agreements(id) ON DELETE CASCADE
);

CREATE INDEX idx_scc_order_tenant ON scc_order_collaboration(tenant_id, order_id);
CREATE INDEX idx_scc_order_partner ON scc_order_collaboration(partner_id, status);
CREATE INDEX idx_scc_order_exception ON scc_order_collaboration(tenant_id, exception_type);
```

### 2.4 Capacity Commitments
```sql
CREATE TABLE scc_capacity_commitments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    facility_id TEXT,
    period TEXT NOT NULL,                   -- "2024-01"
    requested_quantity DECIMAL(18,4) NOT NULL,
    committed_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    remaining_capacity DECIMAL(18,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'REQUESTED'
        CHECK(status IN ('REQUESTED','COMMITTED','PARTIALLY_COMMITTED','DECLINED','CONSUMED')),
    partner_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (agreement_id) REFERENCES scc_agreements(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, agreement_id, item_id, facility_id, period)
);

CREATE INDEX idx_scc_capacity_partner ON scc_capacity_commitments(partner_id, period);
CREATE INDEX idx_scc_capacity_status ON scc_capacity_commitments(tenant_id, status);
```

### 2.5 Shared Documents
```sql
CREATE TABLE scc_shared_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_id TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN (
        'FORECAST','SPECIFICATION','DRAWING','CERTIFICATE','QUALITY_REPORT',
        'CONTRACT','SHIPPING_DOC','INVOICE','OTHER'
    )),
    document_name TEXT NOT NULL,
    storage_path TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type TEXT NOT NULL,
    uploaded_by TEXT NOT NULL,
    uploader_type TEXT NOT NULL CHECK(uploader_type IN ('BUYER','PARTNER','SYSTEM')),
    partner_visible INTEGER NOT NULL DEFAULT 1,
    version_number INTEGER NOT NULL DEFAULT 1,
    description TEXT,
    tags TEXT,                              -- JSON array

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (agreement_id) REFERENCES scc_agreements(id) ON DELETE CASCADE
);

CREATE INDEX idx_scc_doc_agreement ON scc_shared_documents(agreement_id, document_type);
```

### 2.6 Collaboration Exception Log
```sql
CREATE TABLE scc_exceptions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    exception_type TEXT NOT NULL CHECK(exception_type IN (
        'MISSED_ACKNOWLEDGMENT','CAPACITY_SHORTFALL','LATE_SHIPMENT','QUALITY_ISSUE',
        'FORECAST_MISMATCH','DOCUMENT_MISSING','SLA_BREACH','OTHER'
    )),
    severity TEXT NOT NULL CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    source_type TEXT NOT NULL CHECK(source_type IN ('AUTOMATED','MANUAL')),
    source_id TEXT,                         -- Reference to order, forecast, etc.
    description TEXT NOT NULL,
    root_cause TEXT,
    resolution TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','ACKNOWLEDGED','IN_PROGRESS','RESOLVED','ESCALATED','CLOSED')),
    assigned_to TEXT,
    due_date TEXT,
    resolved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (agreement_id) REFERENCES scc_agreements(id) ON DELETE CASCADE
);

CREATE INDEX idx_scc_except_partner ON scc_exceptions(partner_id, status, severity);
CREATE INDEX idx_scc_except_type ON scc_exceptions(tenant_id, exception_type);
```

---

## 3. API Endpoints

### 3.1 Collaboration Agreements
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sc-collaboration/v1/agreements` | Create collaboration agreement |
| GET | `/sc-collaboration/v1/agreements` | List agreements |
| GET | `/sc-collaboration/v1/agreements/{id}` | Get agreement details |
| PUT | `/sc-collaboration/v1/agreements/{id}` | Update agreement |
| POST | `/sc-collaboration/v1/agreements/{id}/activate` | Activate agreement |

### 3.2 Collaborative Forecasts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sc-collaboration/v1/forecasts` | Publish forecast to partner |
| GET | `/sc-collaboration/v1/forecasts` | List shared forecasts |
| GET | `/sc-collaboration/v1/forecasts/{id}` | Get forecast details |
| PUT | `/sc-collaboration/v1/forecasts/{id}/commit` | Partner submits commitment |
| POST | `/sc-collaboration/v1/forecasts/{id}/dispute` | Raise forecast dispute |

### 3.3 Order Collaboration
| Method | Path | Description |
|--------|------|-------------|
| GET | `/sc-collaboration/v1/orders` | List collaborative orders |
| POST | `/sc-collaboration/v1/orders/{id}/acknowledge` | Partner acknowledges order |
| POST | `/sc-collaboration/v1/orders/{id}/asn` | Submit advance shipping notice |
| POST | `/sc-collaboration/v1/orders/{id}/exception` | Report order exception |
| GET | `/sc-collaboration/v1/orders/{id}/timeline` | Get order event timeline |

### 3.4 Capacity Planning
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sc-collaboration/v1/capacity-requests` | Request capacity commitment |
| GET | `/sc-collaboration/v1/capacity` | View capacity commitments |
| PUT | `/sc-collaboration/v1/capacity/{id}/commit` | Partner commits capacity |

### 3.5 Documents & Exceptions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/sc-collaboration/v1/documents` | Upload shared document |
| GET | `/sc-collaboration/v1/documents` | List shared documents |
| GET | `/sc-collaboration/v1/exceptions` | List exceptions |
| PUT | `/sc-collaboration/v1/exceptions/{id}` | Update exception |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `scc.forecast.published` | `{ agreement_id, partner_id, period }` | Forecast shared with partner |
| `scc.order.acknowledged` | `{ order_id, partner_id, committed_date }` | Partner acknowledged order |
| `scc.asn.submitted` | `{ order_id, tracking_number, ship_date }` | Advance shipping notice received |
| `scc.capacity.committed` | `{ agreement_id, item_id, period, qty }` | Capacity commitment received |
| `scc.exception.raised` | `{ exception_id, type, severity, partner_id }` | Collaboration exception detected |
| `scc.sla.breached` | `{ agreement_id, sla_type, partner_id }` | SLA breach detected |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `po.created` | Procurement | Share order with partner |
| `demand.forecast.completed` | Demand Management | Publish forecast to partners |
| `po.goods_receipt.completed` | Procurement | Confirm delivery to partner |
| `quality.inspection.failed` | Quality Management | Raise quality exception |

---


---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.sc_collaboration.v1;

service SCCollaborationService {
    rpc GetAgreement(GetAgreementRequest) returns (GetAgreementResponse);
    rpc GetCollaborativeForecast(GetCollaborativeForecastRequest) returns (GetCollaborativeForecastResponse);
    rpc GetOrderCollaboration(GetOrderCollaborationRequest) returns (GetOrderCollaborationResponse);
    rpc GetExceptions(GetExceptionsRequest) returns (GetExceptionsResponse);
}

message Agreement { string id = 1; string tenant_id = 2; string name = 3; string partner_id = 4; string status = 5; string created_at = 6; string updated_at = 7; }
message CollaborativeForecast { string id = 1; string tenant_id = 2; string agreement_id = 3; string period = 4; double forecast_value = 5; string created_at = 6; }
message OrderCollab { string id = 1; string tenant_id = 2; string agreement_id = 3; string order_id = 4; string status = 5; string created_at = 6; }
message Exception { string id = 1; string tenant_id = 2; string agreement_id = 3; string exception_type = 4; string description = 5; string severity = 6; string created_at = 7; }

message GetAgreementRequest { string tenant_id = 1; string id = 2; }
message GetAgreementResponse { Agreement data = 1; }
message GetCollaborativeForecastRequest { string tenant_id = 1; string agreement_id = 2; }
message GetCollaborativeForecastResponse { CollaborativeForecast data = 1; }
message GetOrderCollaborationRequest { string tenant_id = 1; string agreement_id = 2; }
message GetOrderCollaborationResponse { repeated OrderCollab items = 1; }
message GetExceptionsRequest { string tenant_id = 1; string agreement_id = 2; }
message GetExceptionsResponse { repeated Exception items = 1; }
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | scc_agreements | — |
| V002 | scc_collaborative_forecasts | V001 |
| V003 | scc_order_collaboration | V002 |
| V004 | scc_capacity_commitments | V003 |
| V005 | scc_shared_documents | V004 |
| V006 | scc_exceptions | V005 |

---

## 7. Business Rules

1. **SLA Monitoring**: Auto-detect SLA breaches (acknowledgment within configured hours)
2. **Auto-Escalation**: Unacknowledged orders escalate after 2x SLA response time
3. **Forecast Variance**: Alert when supplier commitment varies >20% from buyer forecast
4. **Capacity Shortfall**: Auto-exception when committed capacity < 80% of requested
5. **Data Visibility**: Partners can only see data within scope of their collaboration agreement
6. **Document Versioning**: Document updates create new versions; history always preserved
7. **Exception Timeout**: Unresolved CRITICAL exceptions escalate to management after 48 hours

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Procurement (11) | Purchase order sharing and acknowledgment |
| Demand Management (140) | Forecast data for sharing with partners |
| Warehouse Management (36) | Receipt confirmation against ASN |
| Inventory (12) | Inventory visibility for partners |
| Supplier Management (139) | Partner qualification status |
| Quality Management (41) | Quality data sharing |
| Supplier Portal (45) | Self-service partner access |
| AI Agents (134) | Automated exception detection and resolution suggestions |
