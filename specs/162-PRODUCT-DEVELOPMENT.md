# 162 - Product Development Service Specification

## 1. Domain Overview

Product Development manages engineering change orders (ECOs), bill of materials (BOM) lifecycle, item lifecycle management, new product introduction (NPI), and design collaboration. Supports configurable ECO workflows, BOM versioning, effectivity dates, engineering-to-manufacturing BOM transfer, and design document management. Enables engineering teams to manage product structures, track changes, collaborate with manufacturing, and control the product lifecycle from concept to obsolescence. Integrates with Product Hub, Manufacturing, PLM, Quality Management, and Cost Management.

**Bounded Context:** Engineering Change & Product Structure Management
**Service Name:** `product-dev-service`
**Database:** `data/product_dev.db`
**HTTP Port:** 8180 | **gRPC Port:** 9180

---

## 2. Database Schema

### 2.1 Engineering Change Orders
```sql
CREATE TABLE pd_ecos (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    eco_number TEXT NOT NULL,               -- ECO-2024-00001
    eco_type TEXT NOT NULL CHECK(eco_type IN (
        'ENGINEERING_CHANGE','DEVIATION','EXEMPTION','ECO','ECN','NPI_INTRODUCTION','OBSOLESCENCE'
    )),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    reason TEXT NOT NULL,
    change_category TEXT NOT NULL CHECK(change_category IN ('DESIGN','MATERIAL','PROCESS','DOCUMENTATION','SAFETY','COST','OTHER')),
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT','EMERGENCY')),
    requested_by TEXT NOT NULL,
    assigned_engineer TEXT NOT NULL,
    affected_items TEXT NOT NULL,           -- JSON array of item IDs
    affected_boms TEXT,                     -- JSON array of BOM IDs
    affected_routings TEXT,                 -- JSON array of routing IDs
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','APPROVED','IMPLEMENTED','REJECTED','CANCELLED')),
    effectivity_type TEXT NOT NULL CHECK(effectivity_type IN ('DATE','SERIAL_NUMBER','LOT','ORDER')),
    effectivity_start TEXT NOT NULL,
    effectivity_end TEXT,
    old_bom_version TEXT,
    new_bom_version TEXT,
    implementation_date TEXT,
    cost_impact_cents INTEGER,
    lead_time_impact_days INTEGER,
    approval_required INTEGER NOT NULL DEFAULT 1,
    implemented_at TEXT,
    implemented_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, eco_number)
);

CREATE INDEX idx_pd_eco_tenant ON pd_ecos(tenant_id, status);
CREATE INDEX idx_pd_eco_priority ON pd_ecos(tenant_id, priority, status);
CREATE INDEX idx_pd_eco_engineer ON pd_ecos(assigned_engineer, status);
```

### 2.2 ECO Lines (Affected Items)
```sql
CREATE TABLE pd_eco_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    eco_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    change_type TEXT NOT NULL CHECK(change_type IN ('ADD','MODIFY','DELETE','REPLACE','RECLASSIFY')),
    bom_id TEXT,
    old_component_item_id TEXT,
    new_component_item_id TEXT,
    old_quantity DECIMAL(18,4),
    new_quantity DECIMAL(18,4),
    old_position_number INTEGER,
    new_position_number INTEGER,
    old_effectivity TEXT,                   -- JSON
    new_effectivity TEXT,                   -- JSON
    description_of_change TEXT NOT NULL,
    reason TEXT,
    implementation_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (eco_id) REFERENCES pd_ecos(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, eco_id, item_id, bom_id)
);
```

### 2.3 Engineering BOMs
```sql
CREATE TABLE pd_engineering_boms (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bom_number TEXT NOT NULL,
    item_id TEXT NOT NULL,
    bom_type TEXT NOT NULL CHECK(bom_type IN ('ENGINEERING','MANUFACTURING','PHANTOM','TEMPLATE')),
    description TEXT,
    revision TEXT NOT NULL DEFAULT 'A',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','RELEASED','FROZEN','OBSOLETE')),
    alternate_bom_number TEXT,
    effectivity_start TEXT,
    effectivity_end TEXT,
    source_eco_id TEXT,
    released_at TEXT,
    released_by TEXT,
    transferred_to_mfg INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, bom_number, revision)
);

CREATE TABLE pd_bom_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bom_id TEXT NOT NULL,
    parent_item_id TEXT NOT NULL,
    component_item_id TEXT NOT NULL,
    position_number INTEGER NOT NULL,
    quantity DECIMAL(18,4) NOT NULL,
    uom TEXT NOT NULL,
    component_type TEXT NOT NULL CHECK(component_type IN ('NORMAL','PHANTOM','REFERENCE','PLANNING')),
    effectivity_type TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(effectivity_type IN ('OPEN','DATE','SERIAL','LOT')),
    effectivity_start TEXT,
    effectivity_end TEXT,
    reference_designator TEXT,
    notes TEXT,
    substitute_components TEXT,             -- JSON: substitute items with percentages
    find_number INTEGER,
    bom_level INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (bom_id) REFERENCES pd_engineering_boms(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, bom_id, component_item_id, position_number)
);

CREATE INDEX idx_pd_bom_item ON pd_engineering_boms(item_id, revision);
CREATE INDEX idx_pd_comp_bom ON pd_bom_components(bom_id);
```

### 2.4 Item Lifecycle
```sql
CREATE TABLE pd_item_lifecycle (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    lifecycle_phase TEXT NOT NULL CHECK(lifecycle_phase IN (
        'CONCEPT','DESIGN','PROTOTYPE','PILOT','PRODUCTION','MATURE','PHASE_OUT','OBSOLETE'
    )),
    phase_entered_date TEXT NOT NULL,
    phase_exit_date TEXT,
    approved_by TEXT,
    gate_review_id TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, item_id, lifecycle_phase, phase_entered_date)
);

CREATE INDEX idx_pd_lifecycle_item ON pd_item_lifecycle(item_id, phase_entered_date DESC);
```

### 2.5 Design Documents
```sql
CREATE TABLE pd_design_documents (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_number TEXT NOT NULL,
    document_type TEXT NOT NULL CHECK(document_type IN (
        'SPECIFICATION','DRAWING','CAD_MODEL','TEST_REPORT','DATASHEET','SOP','OTHER'
    )),
    title TEXT NOT NULL,
    item_id TEXT,
    eco_id TEXT,
    storage_path TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type TEXT NOT NULL,
    revision TEXT NOT NULL DEFAULT 'A',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','RELEASED','SUPERSEDED','OBSOLETE')),
    released_at TEXT,
    released_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, document_number, revision)
);

CREATE INDEX idx_pd_doc_item ON pd_design_documents(item_id);
CREATE INDEX idx_pd_doc_eco ON pd_design_documents(eco_id);
```

---

## 3. API Endpoints

### 3.1 Engineering Change Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/product-dev/v1/ecos` | Create ECO |
| GET | `/product-dev/v1/ecos` | List ECOs |
| GET | `/product-dev/v1/ecos/{id}` | Get ECO details |
| PUT | `/product-dev/v1/ecos/{id}` | Update ECO |
| POST | `/product-dev/v1/ecos/{id}/submit` | Submit for review |
| POST | `/product-dev/v1/ecos/{id}/approve` | Approve ECO |
| POST | `/product-dev/v1/ecos/{id}/implement` | Implement ECO |
| POST | `/product-dev/v1/ecos/{id}/reject` | Reject ECO |

### 3.2 ECO Lines
| Method | Path | Description |
|--------|------|-------------|
| POST | `/product-dev/v1/ecos/{id}/lines` | Add affected item |
| GET | `/product-dev/v1/ecos/{id}/lines` | List affected items |
| PUT | `/product-dev/v1/eco-lines/{id}` | Update ECO line |
| DELETE | `/product-dev/v1/eco-lines/{id}` | Remove affected item |

### 3.3 Engineering BOMs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/product-dev/v1/boms` | Create engineering BOM |
| GET | `/product-dev/v1/boms` | List BOMs |
| GET | `/product-dev/v1/boms/{id}` | Get BOM with components |
| PUT | `/product-dev/v1/boms/{id}` | Update BOM header |
| POST | `/product-dev/v1/boms/{id}/components` | Add component |
| PUT | `/product-dev/v1/bom-components/{id}` | Update component |
| DELETE | `/product-dev/v1/bom-components/{id}` | Remove component |
| POST | `/product-dev/v1/boms/{id}/release` | Release BOM |
| POST | `/product-dev/v1/boms/{id}/transfer-to-mfg` | Transfer to manufacturing |

### 3.4 Item Lifecycle & Documents
| Method | Path | Description |
|--------|------|-------------|
| GET | `/product-dev/v1/items/{id}/lifecycle` | Get item lifecycle |
| POST | `/product-dev/v1/items/{id}/phase-transition` | Transition lifecycle phase |
| POST | `/product-dev/v1/documents` | Upload design document |
| GET | `/product-dev/v1/documents` | List documents |
| POST | `/product-dev/v1/documents/{id}/release` | Release document |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `pdev.eco.created` | `{ eco_id, eco_number, type, priority }` | ECO created |
| `pdev.eco.approved` | `{ eco_id, affected_items_count }` | ECO approved |
| `pdev.eco.implemented` | `{ eco_id, new_bom_version }` | ECO implemented |
| `pdev.bom.released` | `{ bom_id, item_id, revision }` | BOM released |
| `pdev.bom.transferred` | `{ bom_id, mfg_bom_ref }` | BOM transferred to manufacturing |
| `pdev.lifecycle.changed` | `{ item_id, old_phase, new_phase }` | Item lifecycle changed |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `quality.ncr.created` | Quality Management | Trigger ECO for corrective action |
| `item.created` | Product Hub | Initialize lifecycle |
| `cost.change.significant` | Cost Management | Trigger cost-related ECO |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.product_dev.v1;

service ProductDevelopmentService {
    rpc GetECO(GetECORequest) returns (GetECOResponse);
    rpc CreateECO(CreateECORequest) returns (CreateECOResponse);
    rpc ApproveECO(ApproveECORequest) returns (ApproveECOResponse);
    rpc GetEngineeringBOM(GetEngineeringBOMRequest) returns (GetEngineeringBOMResponse);
    rpc ReleaseBOM(ReleaseBOMRequest) returns (ReleaseBOMResponse);
    rpc GetItemLifecycle(GetItemLifecycleRequest) returns (GetItemLifecycleResponse);
}

// ECO messages
message GetECORequest {
    string tenant_id = 1;
    string id = 2;
}

message GetECOResponse {
    EngineeringChangeOrder data = 1;
}

message EngineeringChangeOrder {
    string id = 1;
    string tenant_id = 2;
    string eco_number = 3;
    string eco_type = 4;
    string title = 5;
    string description = 6;
    string reason = 7;
    string change_category = 8;
    string priority = 9;
    string requested_by = 10;
    string assigned_engineer = 11;
    string affected_items = 12;
    string affected_boms = 13;
    string affected_routings = 14;
    string status = 15;
    string effectivity_type = 16;
    string effectivity_start = 17;
    string effectivity_end = 18;
    string old_bom_version = 19;
    string new_bom_version = 20;
    string implementation_date = 21;
    int64 cost_impact_cents = 22;
    int32 lead_time_impact_days = 23;
    string implemented_at = 24;
    string implemented_by = 25;
    string created_at = 26;
    string updated_at = 27;
}

message CreateECORequest {
    string tenant_id = 1;
    string eco_type = 2;
    string title = 3;
    string description = 4;
    string reason = 5;
    string change_category = 6;
    string priority = 7;
    string requested_by = 8;
    string assigned_engineer = 9;
    string affected_items = 10;
    string affected_boms = 11;
    string effectivity_type = 12;
    string effectivity_start = 13;
    string effectivity_end = 14;
}

message CreateECOResponse {
    EngineeringChangeOrder data = 1;
}

message ApproveECORequest {
    string tenant_id = 1;
    string id = 2;
}

message ApproveECOResponse {
    string id = 1;
    string status = 2;
}

// BOM messages
message GetEngineeringBOMRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetEngineeringBOMResponse {
    EngineeringBOM data = 1;
}

message EngineeringBOM {
    string id = 1;
    string tenant_id = 2;
    string bom_number = 3;
    string item_id = 4;
    string bom_type = 5;
    string description = 6;
    string revision = 7;
    string status = 8;
    string alternate_bom_number = 9;
    string effectivity_start = 10;
    string effectivity_end = 11;
    string source_eco_id = 12;
    string released_at = 13;
    string released_by = 14;
    int32 transferred_to_mfg = 15;
    string created_at = 16;
    string updated_at = 17;
}

message ReleaseBOMRequest {
    string tenant_id = 1;
    string id = 2;
}

message ReleaseBOMResponse {
    string id = 1;
    string status = 2;
    string revision = 3;
}

// Item Lifecycle messages
message GetItemLifecycleRequest {
    string tenant_id = 1;
    string item_id = 2;
}

message GetItemLifecycleResponse {
    repeated LifecyclePhase phases = 1;
}

message LifecyclePhase {
    string id = 1;
    string tenant_id = 2;
    string item_id = 3;
    string lifecycle_phase = 4;
    string phase_entered_date = 5;
    string phase_exit_date = 6;
    string approved_by = 7;
    string notes = 8;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | pd_ecos | -- |
| V002 | pd_eco_lines | V001 |
| V003 | pd_engineering_boms | -- |
| V004 | pd_bom_components | V003 |
| V005 | pd_item_lifecycle | -- |
| V006 | pd_design_documents | -- |

---

## 7. Business Rules

1. **ECO Approval**: All ECOs require engineering lead approval; safety-related require QA approval
2. **BOM Versioning**: Each BOM release creates new revision; historical versions preserved
3. **Effectivity Control**: BOM changes take effect based on date, serial, or lot configuration
4. **MFG Transfer**: BOMs must be released before transfer to manufacturing
5. **Document Linking**: All design documents linked to items and ECOs for traceability
6. **Impact Analysis**: ECOs must list all affected items, BOMs, and routings before submission
7. **Emergency ECOs**: EMERGENCY priority bypasses standard review timeline (24h approval)
8. **Obsolescence**: Items in OBSOLETE phase cannot be added to new BOMs

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| product-hub-service | `GetItem` | Read item master data for BOM/ECO references |
| manufacturing-service | `TransferBOM` | Transfer engineering BOM to manufacturing |
| quality-service | `GetNCR` | Read non-conformance reports for corrective ECOs |
| cost-service | `CalculateCostImpact` | Analyze cost impact of proposed changes |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| manufacturing-service | `GetEngineeringBOM` | Read released engineering BOMs |
| quality-service | `GetECO` | Read engineering change order details |
| inventory-service | `GetEffectivity` | Read BOM effectivity dates for allocation |
| configurator-service | `GetBOMStructure` | Read BOM structure for configuration models |
