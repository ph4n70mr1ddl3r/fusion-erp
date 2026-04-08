# 38 - Product Lifecycle Management (PLM) Service Specification

## 1. Domain Overview

Product Lifecycle Management provides product development tracking, engineering BOM management, change request workflows, product configuration modeling, innovation pipeline management, document linking, and supplier design collaboration. Integrates with Inventory for item master data, Manufacturing for BOM release, Order Management for configurable products, and Document Management for engineering documents.

**Bounded Context:** Product Development, Engineering & Lifecycle Management
**Service Name:** `plm-service`
**Database:** `data/plm.db`
**HTTP Port:** 8065 | **gRPC Port:** 9065

---

## 2. Database Schema

### 2.1 Products
```sql
CREATE TABLE plm_products (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_code TEXT NOT NULL,
    product_name TEXT NOT NULL,
    product_type TEXT NOT NULL DEFAULT 'FINISHED_GOOD'
        CHECK(product_type IN ('FINISHED_GOOD','SEMI_FINISHED','COMPONENT','SERVICE')),
    lifecycle_stage TEXT NOT NULL DEFAULT 'CONCEPT'
        CHECK(lifecycle_stage IN ('CONCEPT','DEVELOPMENT','PROTOTYPE','PRODUCTION','PHASE_OUT','OBSOLETE')),
    description TEXT,
    category_id TEXT,
    brand TEXT,
    design_owner_id TEXT,
    target_launch_date TEXT,
    actual_launch_date TEXT,
    discontinuation_date TEXT,
    is_configurable INTEGER NOT NULL DEFAULT 0,
    specification TEXT,                      -- JSON: technical specifications
    regulatory_compliance TEXT,              -- JSON: compliance certifications and standards

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, product_code)
);

CREATE INDEX idx_plm_products_tenant ON plm_products(tenant_id);
CREATE INDEX idx_plm_products_tenant_stage ON plm_products(tenant_id, lifecycle_stage);
CREATE INDEX idx_plm_products_tenant_type ON plm_products(tenant_id, product_type);
CREATE INDEX idx_plm_products_tenant_category ON plm_products(tenant_id, category_id);
```

### 2.2 Product Structures (Engineering BOM)
```sql
CREATE TABLE plm_product_structures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    structure_type TEXT NOT NULL DEFAULT 'ENGINEERING'
        CHECK(structure_type IN ('ENGINEERING','MANUFACTURING','SERVICE')),
    revision TEXT NOT NULL DEFAULT 'A',
    description TEXT,
    quantity DECIMAL(18,4) NOT NULL DEFAULT 1,
    uom TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','RELEASED','SUPERSEDED')),
    effectivity_from TEXT,
    effectivity_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES plm_products(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, product_id, structure_type, revision)
);

CREATE INDEX idx_plm_structures_tenant_product ON plm_product_structures(tenant_id, product_id);
CREATE INDEX idx_plm_structures_tenant_status ON plm_product_structures(tenant_id, status);
```

### 2.3 Structure Components
```sql
CREATE TABLE plm_structure_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    structure_id TEXT NOT NULL,
    component_product_id TEXT NOT NULL,
    quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    uom TEXT NOT NULL,
    reference_designator TEXT,
    is_phantom INTEGER NOT NULL DEFAULT 0,
    substitute_components TEXT,              -- JSON: array of substitute product IDs with conditions
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (structure_id) REFERENCES plm_product_structures(id) ON DELETE CASCADE,
    FOREIGN KEY (component_product_id) REFERENCES plm_products(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, structure_id, component_product_id)
);

CREATE INDEX idx_plm_components_tenant_structure ON plm_structure_components(tenant_id, structure_id);
```

### 2.4 Change Requests (ECR/ECN/ECO)
```sql
CREATE TABLE plm_change_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    ecr_number TEXT NOT NULL,               -- ECR-2024-00001
    title TEXT NOT NULL,
    change_type TEXT NOT NULL DEFAULT 'ECR'
        CHECK(change_type IN ('ECN','ECR','ECO','DEVIATION')),
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    description TEXT,
    justification TEXT,
    affected_products TEXT,                 -- JSON: array of product IDs
    affected_documents TEXT,                -- JSON: array of document IDs
    requested_by TEXT NOT NULL,
    assigned_to TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED','IMPLEMENTED')),
    implementation_date TEXT,
    effective_date TEXT,
    disposition TEXT
        CHECK(disposition IN ('USE_AS_IS','REWORK','SCRAP')),
    approval_deadline TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, ecr_number)
);

CREATE INDEX idx_plm_cr_tenant_status ON plm_change_requests(tenant_id, status);
CREATE INDEX idx_plm_cr_tenant_type ON plm_change_requests(tenant_id, change_type);
CREATE INDEX idx_plm_cr_tenant_assigned ON plm_change_requests(tenant_id, assigned_to);
CREATE INDEX idx_plm_cr_tenant_priority ON plm_change_requests(tenant_id, priority);
```

### 2.5 Change Request Lines
```sql
CREATE TABLE plm_change_request_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    change_request_id TEXT NOT NULL,
    change_type TEXT NOT NULL
        CHECK(change_type IN ('ADD','REMOVE','MODIFY')),
    structure_id TEXT,
    component_id TEXT,
    old_value TEXT,                         -- JSON: previous state
    new_value TEXT,                         -- JSON: proposed state
    reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (change_request_id) REFERENCES plm_change_requests(id) ON DELETE CASCADE
);

CREATE INDEX idx_plm_crl_tenant_cr ON plm_change_request_lines(tenant_id, change_request_id);
```

### 2.6 Product Configurations (Configurator Models)
```sql
CREATE TABLE plm_product_configurations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_version TEXT NOT NULL DEFAULT '1.0',
    configuration_rules TEXT NOT NULL,      -- JSON: logical rules for option compatibility
    features TEXT NOT NULL,                 -- JSON: array of configurable features with options
    constraints TEXT NOT NULL,              -- JSON: validation constraints and cross-feature rules
    pricing_impact TEXT,                    -- JSON: pricing adjustments per option selection
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES plm_products(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, product_id, model_name)
);

CREATE INDEX idx_plm_configs_tenant_product ON plm_product_configurations(tenant_id, product_id);
CREATE INDEX idx_plm_configs_tenant_status ON plm_product_configurations(tenant_id, status);
```

### 2.7 Configuration Instances
```sql
CREATE TABLE plm_configuration_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    configuration_id TEXT NOT NULL,
    configuration_data TEXT NOT NULL,       -- JSON: full configuration state
    selected_options TEXT NOT NULL,         -- JSON: selected feature-option pairs
    calculated_price_cents INTEGER NOT NULL DEFAULT 0,
    bom_explosion TEXT,                     -- JSON: generated BOM from configuration
    created_by TEXT NOT NULL,
    quote_id TEXT,
    sales_order_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (configuration_id) REFERENCES plm_product_configurations(id) ON DELETE CASCADE
);

CREATE INDEX idx_plm_instances_tenant_config ON plm_configuration_instances(tenant_id, configuration_id);
CREATE INDEX idx_plm_instances_tenant_quote ON plm_configuration_instances(tenant_id, quote_id);
CREATE INDEX idx_plm_instances_tenant_order ON plm_configuration_instances(tenant_id, sales_order_id);
```

### 2.8 Innovation Ideas
```sql
CREATE TABLE plm_innovation_ideas (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    idea_name TEXT NOT NULL,
    description TEXT,
    submitter_id TEXT NOT NULL,
    category TEXT NOT NULL DEFAULT 'NEW_PRODUCT'
        CHECK(category IN ('NEW_PRODUCT','IMPROVEMENT','COST_REDUCTION','SUSTAINABILITY')),
    market_potential TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(market_potential IN ('LOW','MEDIUM','HIGH')),
    estimated_revenue_cents INTEGER NOT NULL DEFAULT 0,
    development_cost_cents INTEGER NOT NULL DEFAULT 0,
    roi_estimate REAL NOT NULL DEFAULT 0,
    score REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(status IN ('SUBMITTED','UNDER_REVIEW','ACCEPTED','REJECTED','IN_DEVELOPMENT')),
    assigned_team TEXT,                     -- JSON: array of team member IDs
    review_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, idea_name)
);

CREATE INDEX idx_plm_ideas_tenant_status ON plm_innovation_ideas(tenant_id, status);
CREATE INDEX idx_plm_ideas_tenant_category ON plm_innovation_ideas(tenant_id, category);
CREATE INDEX idx_plm_ideas_tenant_submitter ON plm_innovation_ideas(tenant_id, submitter_id);
```

### 2.9 Document Links
```sql
CREATE TABLE plm_document_links (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    document_id TEXT NOT NULL,              -- references dms-service
    document_type TEXT NOT NULL DEFAULT 'SPECIFICATION'
        CHECK(document_type IN ('SPECIFICATION','CAD_DRAWING','TEST_REPORT','CERTIFICATE','IMAGE','OTHER')),
    revision TEXT,
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (product_id) REFERENCES plm_products(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, product_id, document_id, document_type)
);

CREATE INDEX idx_plm_doclinks_tenant_product ON plm_document_links(tenant_id, product_id);
CREATE INDEX idx_plm_doclinks_tenant_type ON plm_document_links(tenant_id, document_type);
```

### 2.10 Supplier Collaboration
```sql
CREATE TABLE plm_supplier_collaboration (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    collaboration_type TEXT NOT NULL DEFAULT 'DESIGN'
        CHECK(collaboration_type IN ('DESIGN','SOURCING','QUALITY')),
    status TEXT NOT NULL DEFAULT 'INVITED'
        CHECK(status IN ('INVITED','ACTIVE','COMPLETED')),
    shared_documents TEXT,                  -- JSON: array of shared document IDs
    access_level TEXT NOT NULL DEFAULT 'VIEW'
        CHECK(access_level IN ('VIEW','COMMENT','EDIT')),
    nda_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES plm_products(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, product_id, supplier_id, collaboration_type)
);

CREATE INDEX idx_plm_supplier_collab_tenant_product ON plm_supplier_collaboration(tenant_id, product_id);
CREATE INDEX idx_plm_supplier_collab_tenant_supplier ON plm_supplier_collaboration(tenant_id, supplier_id);
```

---

## 3. REST API Endpoints

```
# Product Management
GET/POST      /api/v1/plm/products                                     Permission: plm.products.read/create
GET/PUT       /api/v1/plm/products/{id}                                Permission: plm.products.read/update
POST          /api/v1/plm/products/{id}/lifecycle-transition           Permission: plm.products.lifecycle
GET           /api/v1/plm/products/{id}/where-used                     Permission: plm.products.read
GET           /api/v1/plm/products/{id}/traceability                   Permission: plm.products.read

# Product Structures
GET/POST      /api/v1/plm/products/{id}/structures                     Permission: plm.structures.read/create
GET/PUT       /api/v1/plm/structures/{id}                              Permission: plm.structures.read/update
GET/POST      /api/v1/plm/structures/{id}/components                   Permission: plm.structures.read/create
POST          /api/v1/plm/structures/{id}/release                      Permission: plm.structures.update
POST          /api/v1/plm/structures/{id}/revise                       Permission: plm.structures.update

# Change Management
GET/POST      /api/v1/plm/change-requests                              Permission: plm.changes.read/create
GET/PUT       /api/v1/plm/change-requests/{id}                         Permission: plm.changes.read/update
GET/POST      /api/v1/plm/change-requests/{id}/lines                   Permission: plm.changes.read/create
POST          /api/v1/plm/change-requests/{id}/submit                  Permission: plm.changes.update
POST          /api/v1/plm/change-requests/{id}/approve                 Permission: plm.changes.approve
POST          /api/v1/plm/change-requests/{id}/reject                  Permission: plm.changes.approve
POST          /api/v1/plm/change-requests/{id}/implement               Permission: plm.changes.implement

# Configurator
GET/POST      /api/v1/plm/products/{id}/configurations                 Permission: plm.configurations.read/create
GET/PUT       /api/v1/plm/configurations/{id}                          Permission: plm.configurations.read/update
POST          /api/v1/plm/configurations/{id}/validate                 Permission: plm.configurations.update
POST          /api/v1/plm/configurations/{id}/explode-bom              Permission: plm.configurations.update
POST          /api/v1/plm/configurations/{id}/price                    Permission: plm.configurations.update

# Innovation
GET/POST      /api/v1/plm/innovations                                  Permission: plm.innovations.read/create
GET/PUT       /api/v1/plm/innovations/{id}                             Permission: plm.innovations.read/update
POST          /api/v1/plm/innovations/{id}/score                       Permission: plm.innovations.update
POST          /api/v1/plm/innovations/{id}/accept                      Permission: plm.innovations.manage
POST          /api/v1/plm/innovations/{id}/reject                      Permission: plm.innovations.manage

# Document Links
GET/POST      /api/v1/plm/products/{id}/documents                      Permission: plm.documents.read/create
DELETE        /api/v1/plm/products/{id}/documents/{did}                Permission: plm.documents.delete

# Supplier Collaboration
GET/POST      /api/v1/plm/products/{id}/suppliers                      Permission: plm.suppliers.read/create
GET/PUT       /api/v1/plm/supplier-collaboration/{id}                   Permission: plm.suppliers.read/update

# Reports
GET           /api/v1/plm/reports/product-lifecycle-summary            Permission: plm.reports.view
GET           /api/v1/plm/reports/change-request-aging                 Permission: plm.reports.view
GET           /api/v1/plm/reports/time-to-market                       Permission: plm.reports.view
GET           /api/v1/plm/reports/innovation-pipeline                  Permission: plm.reports.view
```

---

## 4. Business Rules

### 4.1 Product Lifecycle Transitions
```
CONCEPT → DEVELOPMENT → PROTOTYPE → PRODUCTION → PHASE_OUT → OBSOLETE
```
1. **CONCEPT:** Initial idea, no manufacturing capability. Editable.
2. **DEVELOPMENT:** Active design work, BOM creation allowed.
3. **PROTOTYPE:** First articles being built and tested.
4. **PRODUCTION:** Released for mass manufacturing and sales.
5. **PHASE_OUT:** No new orders accepted, existing orders fulfilled.
6. **OBSOLETE:** No longer supported, all references archived.

Product lifecycle transitions MUST follow the defined stage sequence. Backward transitions are not allowed. Skipping stages requires PLM admin override.

### 4.2 Engineering Change Management
```
DRAFT → SUBMITTED → UNDER_REVIEW → APPROVED → IMPLEMENTED
                                       ↘ REJECTED
```
- ECRs for CRITICAL priority changes require multi-level approval (engineering lead + product manager + quality)
- Change requests MUST list all affected products and structures via `affected_products` and `affected_documents` fields
- Implementing an approved ECR creates new structure revisions automatically
- The `disposition` field controls how existing inventory of the old revision is handled:
  - USE_AS_IS: Continue using old stock until depleted
  - REWORK: Existing stock must be reworked to new specification
  - SCRAP: Existing stock is written off

### 4.3 Product Structure Management
- Released product structures are immutable; changes require a new revision
- Revising a structure creates a copy with incremented revision letter (A → B → C)
- Structure components can reference phantom assemblies that are exploded during BOM generation
- Substitute components are tracked with conditions for automatic substitution during planning

### 4.4 Product Configurator
- Product configurations MUST validate all constraints before saving
- Constraint validation checks cross-feature compatibility rules defined in the configuration model
- BOM explosion from configuration generates manufacturing orders with the exact component list
- Pricing impact is calculated from selected options and the pricing rules in the configuration model
- Saved configuration instances can be linked to quotes and sales orders

### 4.5 Innovation Scoring
- Innovation scoring uses weighted criteria:
  - Market potential: 30% weight
  - ROI estimate: 25% weight
  - Strategic fit: 20% weight
  - Technical feasibility: 15% weight
  - Time to market: 10% weight
- Score is normalized to a 0-100 scale
- Ideas with score >= 75 are recommended for acceptance
- Accepted ideas can be linked to new product records in plm_products

### 4.6 Supplier Collaboration Access Control
- Supplier collaboration access is scoped to specific products and document types
- Access levels control what external suppliers can do:
  - VIEW: Read-only access to shared documents
  - COMMENT: View + add comments and annotations
  - EDIT: View + comment + modify shared design documents
- NDA reference is required before granting any access level
- Collaboration status transitions: INVITED → ACTIVE → COMPLETED

### 4.7 Where-Used and Traceability
- Where-used query returns all parent structures referencing a given component
- Traceability provides full lifecycle history: creation → all changes → current state
- Both queries traverse multi-level BOM hierarchies recursively

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `plm.product.created` | New product registered | INV, OM |
| `plm.product.stage_changed` | Lifecycle transition applied | OM, MFG |
| `plm.structure.released` | BOM structure approved | MFG, Planning |
| `plm.change_request.submitted` | ECR submitted for review | Workflow |
| `plm.change_request.approved` | ECR approved by reviewers | MFG |
| `plm.change_request.implemented` | ECR changes applied to structures | MFG, INV |
| `plm.configuration.validated` | Configuration passes validation | OM, Pricing |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.plm.v1;

service PlmService {
    rpc GetProductStructure(GetProductStructureRequest) returns (GetProductStructureResponse);
    rpc ValidateConfiguration(ValidateConfigurationRequest) returns (ValidateConfigurationResponse);
    rpc ExplodeBom(ExplodeBomRequest) returns (ExplodeBomResponse);
    rpc GetWhereUsed(GetWhereUsedRequest) returns (GetWhereUsedResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **INV:** Item master data, item categories, unit of measure definitions
- **MFG:** Manufacturing BOM requirements, work center capabilities
- **OM:** Configurable product orders, product catalog requirements
- **DMS:** Engineering documents, CAD files, specifications, test reports
- **Proc:** Supplier master data, sourcing information
- **Workflow:** Approval workflows for change requests

### 6.2 Data Published To
- **INV:** New product registrations, product attribute updates, lifecycle stage changes
- **MFG:** Released engineering BOMs, structure revisions, change order implementations
- **OM:** Configurable product models, pricing impact rules, product availability by lifecycle stage
- **Planning:** Updated BOM structures for MRP re-planning
- **DMS:** Document link associations, access control for supplier collaboration
- **Proc:** Component sourcing requirements from new product designs

---

## 7. Migrations

1. V001: `plm_products`
2. V002: `plm_product_structures`
3. V003: `plm_structure_components`
4. V004: `plm_change_requests`
5. V005: `plm_change_request_lines`
6. V006: `plm_product_configurations`
7. V007: `plm_configuration_instances`
8. V008: `plm_innovation_ideas`
9. V009: `plm_document_links`
10. V010: `plm_supplier_collaboration`
11. V011: Triggers for `updated_at`
