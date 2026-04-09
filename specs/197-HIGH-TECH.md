# 197 - High Tech Industry Solution Specification

## 1. Domain Overview

High Tech provides industry-specific capabilities for configure-to-order product management, component lifecycle tracking, engineering change order processing, contract manufacturing management, and return merchandise authorization. Supports complex product configuration models with rules and constraints, component lifecycle management from introduction through end-of-life with last-time-buy notifications, engineering change order workflows with approval chains, contract manufacturing order management with IP protection, and RMA processing with defect analysis and warranty linkage. Enables high technology manufacturers to manage complex product configurations, rapid component changes, and outsourced manufacturing. Integrates with Product Development, Quality, Order Management, and Inventory.

**Bounded Context:** High Technology Manufacturing, Config-to-Order & PLM
**Service Name:** `high-tech-service`
**Database:** `data/high_tech.db`
**HTTP Port:** 8215 | **gRPC Port:** 9215

---

## 2. Database Schema

### 2.1 Product Configurations
```sql
CREATE TABLE ht_product_configurations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    config_code TEXT NOT NULL,
    base_product_id TEXT NOT NULL,
    rules TEXT NOT NULL,                           -- JSON: configuration rules
    constraints TEXT NOT NULL,                     -- JSON: compatibility constraints
    compatible_options TEXT NOT NULL,              -- JSON: option compatibility matrix
    pricing_impact TEXT NOT NULL,                  -- JSON: price adjustments per option
    bom_mapping TEXT NOT NULL,                     -- JSON: config-to-BOM mapping
    validation_rules TEXT,                         -- JSON: validation logic
    effective_from TEXT,
    effective_to TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, config_code)
);

CREATE INDEX idx_ht_config_tenant ON ht_product_configurations(tenant_id, status);
CREATE INDEX idx_ht_config_product ON ht_product_configurations(base_product_id);
```

### 2.2 Component Lifecycle
```sql
CREATE TABLE ht_component_lifecycle (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    component_code TEXT NOT NULL,
    component_name TEXT NOT NULL,
    category TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    lifecycle_stage TEXT NOT NULL CHECK(lifecycle_stage IN ('INTRODUCTION','ACTIVE','EOL_SCHEDULED','END_OF_LIFE')),
    replacement_component_id TEXT,
    last_time_buy_date TEXT,
    inventory_on_hand INTEGER NOT NULL DEFAULT 0,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    risk_level TEXT NOT NULL DEFAULT 'LOW'
        CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    notification_sent INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, component_code)
);

CREATE INDEX idx_ht_comp_tenant ON ht_component_lifecycle(tenant_id, lifecycle_stage);
CREATE INDEX idx_ht_comp_supplier ON ht_component_lifecycle(supplier_id);
CREATE INDEX idx_ht_comp_risk ON ht_component_lifecycle(risk_level);
CREATE INDEX idx_ht_comp_eol ON ht_component_lifecycle(last_time_buy_date);
```

### 2.3 ECO Requests
```sql
CREATE TABLE ht_eco_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    eco_number TEXT NOT NULL,
    eco_type TEXT NOT NULL CHECK(eco_type IN ('ECN','ECO','ECR')),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    affected_assemblies TEXT NOT NULL,             -- JSON: affected BOM assemblies
    change_description TEXT NOT NULL,
    impact_assessment TEXT,                        -- JSON: impact analysis
    approval_chain TEXT,                           -- JSON: approver sequence
    submitted_by TEXT NOT NULL,
    submitted_at TEXT,
    approved_by TEXT,
    approved_at TEXT,
    implementation_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','IMPLEMENTED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, eco_number)
);

CREATE INDEX idx_ht_eco_tenant ON ht_eco_requests(tenant_id, status);
CREATE INDEX idx_ht_eco_submitter ON ht_eco_requests(submitted_by);
CREATE INDEX idx_ht_eco_dates ON ht_eco_requests(submitted_at DESC);
```

### 2.4 Contract Manufacturing
```sql
CREATE TABLE ht_contract_manufacturing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    order_number TEXT NOT NULL,
    cm_partner_id TEXT NOT NULL,
    bom TEXT NOT NULL,                             -- JSON: bill of materials
    yield_requirement_pct REAL NOT NULL DEFAULT 95.0,
    quality_specs TEXT NOT NULL,                   -- JSON: quality requirements
    ip_protection_rules TEXT NOT NULL,             -- JSON: IP safeguard rules
    production_start TEXT NOT NULL,
    production_end TEXT,
    quantity_ordered INTEGER NOT NULL DEFAULT 0,
    quantity_accepted INTEGER NOT NULL DEFAULT 0,
    quantity_rejected INTEGER NOT NULL DEFAULT 0,
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_ht_cm_tenant ON ht_contract_manufacturing(tenant_id, status);
CREATE INDEX idx_ht_cm_partner ON ht_contract_manufacturing(cm_partner_id);
CREATE INDEX idx_ht_cm_dates ON ht_contract_manufacturing(production_start);
```

### 2.5 RMA Processing
```sql
CREATE TABLE ht_rma_processing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rma_number TEXT NOT NULL,
    product_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    defect_code TEXT NOT NULL,
    defect_description TEXT,
    defect_analysis TEXT,                          -- JSON: failure analysis
    disposition TEXT NOT NULL CHECK(disposition IN ('REPAIR','REPLACE','REFUND','SCRAP')),
    warranty_id TEXT,
    repair_cost_cents INTEGER NOT NULL DEFAULT 0,
    replacement_shipped INTEGER NOT NULL DEFAULT 0,
    processed_by TEXT,
    processed_at TEXT,
    status TEXT NOT NULL DEFAULT 'REQUESTED'
        CHECK(status IN ('REQUESTED','APPROVED','IN_PROGRESS','COMPLETED','REJECTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rma_number)
);

CREATE INDEX idx_ht_rma_tenant ON ht_rma_processing(tenant_id, status);
CREATE INDEX idx_ht_rma_product ON ht_rma_processing(product_id);
CREATE INDEX idx_ht_rma_customer ON ht_rma_processing(customer_id);
CREATE INDEX idx_ht_rma_disposition ON ht_rma_processing(disposition);
```

---

## 3. API Endpoints

### 3.1 Product Configuration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/high-tech/v1/configurations` | Create configuration model |
| GET | `/high-tech/v1/configurations` | List configurations |
| GET | `/high-tech/v1/configurations/{id}` | Get configuration |
| PUT | `/high-tech/v1/configurations/{id}` | Update configuration |
| POST | `/high-tech/v1/configurations/{id}/validate` | Validate configuration |

### 3.2 Component Lifecycle
| Method | Path | Description |
|--------|------|-------------|
| GET | `/high-tech/v1/components` | List components |
| GET | `/high-tech/v1/components/{id}` | Get component |
| PUT | `/high-tech/v1/components/{id}` | Update lifecycle stage |
| GET | `/high-tech/v1/components/eol` | List EOL components |
| POST | `/high-tech/v1/components/{id}/notify-eol` | Send EOL notification |

### 3.3 ECO Processing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/high-tech/v1/ecos` | Create ECO |
| GET | `/high-tech/v1/ecos` | List ECOs |
| GET | `/high-tech/v1/ecos/{id}` | Get ECO details |
| POST | `/high-tech/v1/ecos/{id}/submit` | Submit for approval |
| POST | `/high-tech/v1/ecos/{id}/approve` | Approve ECO |
| POST | `/high-tech/v1/ecos/{id}/implement` | Implement ECO |

### 3.4 Contract Manufacturing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/high-tech/v1/cm-orders` | Create CM order |
| GET | `/high-tech/v1/cm-orders` | List CM orders |
| GET | `/high-tech/v1/cm-orders/{id}` | Get order details |
| PUT | `/high-tech/v1/cm-orders/{id}` | Update order |
| POST | `/high-tech/v1/cm-orders/{id}/receive` | Receive shipment |

### 3.5 RMA
| Method | Path | Description |
|--------|------|-------------|
| POST | `/high-tech/v1/rma` | Create RMA |
| GET | `/high-tech/v1/rma` | List RMAs |
| GET | `/high-tech/v1/rma/{id}` | Get RMA details |
| POST | `/high-tech/v1/rma/{id}/approve` | Approve RMA |
| POST | `/high-tech/v1/rma/{id}/process` | Process RMA |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `hightech.eco.submitted` | `{ eco_number, eco_type, assemblies }` | ECO submitted |
| `hightech.component.eol_notified` | `{ component_code, ltb_date }` | EOL notification sent |
| `hightech.rma.created` | `{ rma_number, product_id, defect }` | RMA created |
| `hightech.config.validated` | `{ config_code, valid, errors }` | Configuration validated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `order.return.requested` | Order Management (32) | Create RMA |
| `quality.defect.reported` | Quality Management (136) | Link to RMA |
| `supplier.shipment.received` | Receiving (10) | Update CM order |

---

## 5. Business Rules

1. **Configuration Validation**: All configuration rules and constraints validated before order acceptance
2. **EOL Notifications**: Automatic alerts 90/60/30 days before last-time-buy date
3. **ECO Impact**: ECOs require impact assessment covering affected BOMs, inventory, and open orders
4. **CM Yield**: Contract manufacturing acceptance rate tracked against yield requirement
5. **RMA Warranty**: RMAs auto-linked to warranty records for coverage verification
6. **IP Protection**: CM orders enforce IP protection rules and audit trail

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Product Development (162) | Engineering BOM and change management |
| Quality Management (136) | Quality specs and defect tracking |
| Order Management (32) | Returns and RMAs |
| Inventory (34) | Component availability and EOL |
| Supplier Management (139) | Supplier and CM partner data |
| Manufacturing (40) | Production order integration |
