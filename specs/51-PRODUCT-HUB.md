# 51 - Product Hub Service Specification

## 1. Domain Overview

Product Hub provides centralized product information management (Master Data Management) across the enterprise. It serves as the single source of truth for item definitions, specifications, classifications, attributes, relationships, and lifecycle states. Manages product data synchronization across all consuming systems including inventory, procurement, order management, and e-commerce. Supports flexible attribute models, multi-category classification, cross-references to external system identifiers, and controlled lifecycle transitions.

**Bounded Context:** Product Master Data Management & Information Governance
**Service Name:** `producthub-service`
**Database:** `data/producthub.db`
**HTTP Port:** 8083 | **gRPC Port:** 9083

---

## 2. Database Schema

### 2.1 Product Registrations
```sql
CREATE TABLE product_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_code TEXT NOT NULL,
    product_name TEXT NOT NULL,
    product_type TEXT NOT NULL
        CHECK(product_type IN ('FINISHED_GOOD','SEMI_FINISHED','RAW_MATERIAL','SERVICE','CONSUMABLE','DIGITAL','KIT')),
    description TEXT,
    brand TEXT,
    manufacturer TEXT,
    lifecycle_state TEXT NOT NULL DEFAULT 'CONCEPT'
        CHECK(lifecycle_state IN ('CONCEPT','PROTOTYPE','PILOT','ACTIVE','PHASE_OUT','OBSOLETE')),
    primary_uom TEXT NOT NULL,
    secondary_uom TEXT,
    weight REAL,
    weight_uom TEXT,
    volume REAL,
    volume_uom TEXT,
    hazardous_class TEXT,
    country_of_origin TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, product_code)
);

CREATE INDEX idx_product_reg_tenant ON product_registrations(tenant_id, product_type, lifecycle_state);
CREATE INDEX idx_product_reg_state ON product_registrations(tenant_id, lifecycle_state);
CREATE INDEX idx_product_reg_brand ON product_registrations(tenant_id, brand);
```

### 2.2 Product Attributes
```sql
CREATE TABLE product_attributes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    attribute_code TEXT NOT NULL,
    attribute_name TEXT NOT NULL,
    attribute_type TEXT NOT NULL
        CHECK(attribute_type IN ('TEXT','NUMBER','DATE','BOOLEAN','ENUM','MULTI_ENUM','URL','FILE')),
    uom TEXT,
    is_required INTEGER NOT NULL DEFAULT 0,
    is_searchable INTEGER NOT NULL DEFAULT 1,
    is_comparable INTEGER NOT NULL DEFAULT 0,
    enum_values TEXT,
    validation_rule TEXT,
    display_sequence INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, attribute_code)
);

CREATE INDEX idx_product_attrs_tenant ON product_attributes(tenant_id, attribute_type);
```

### 2.3 Product Attribute Values
```sql
CREATE TABLE product_attribute_values (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    attribute_id TEXT NOT NULL,
    text_value TEXT,
    number_value REAL,
    date_value TEXT,
    boolean_value INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES product_registrations(id) ON DELETE CASCADE,
    FOREIGN KEY (attribute_id) REFERENCES product_attributes(id),
    UNIQUE(tenant_id, product_id, attribute_id)
);

CREATE INDEX idx_attr_values_product ON product_attribute_values(tenant_id, product_id);
CREATE INDEX idx_attr_values_attr ON product_attribute_values(tenant_id, attribute_id);
```

### 2.4 Product Categories
```sql
CREATE TABLE product_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL,
    category_name TEXT NOT NULL,
    parent_category_id TEXT,
    category_level INTEGER NOT NULL DEFAULT 0,
    description TEXT,
    image_url TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_category_id) REFERENCES product_categories(id),
    UNIQUE(tenant_id, category_code)
);

CREATE INDEX idx_product_cats_tenant ON product_categories(tenant_id, parent_category_id);
```

### 2.5 Product Category Assignments
```sql
CREATE TABLE product_category_assignments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    category_id TEXT NOT NULL,
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES product_registrations(id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES product_categories(id),
    UNIQUE(tenant_id, product_id, category_id)
);

CREATE INDEX idx_cat_assign_product ON product_category_assignments(product_id);
CREATE INDEX idx_cat_assign_category ON product_category_assignments(category_id);
```

### 2.6 Product Relationships
```sql
CREATE TABLE product_relationships (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_product_id TEXT NOT NULL,
    target_product_id TEXT NOT NULL,
    relationship_type TEXT NOT NULL
        CHECK(relationship_type IN ('CROSS_SELL','UP_SELL','SUBSTITUTE','COMPLEMENT','ACCESSORY','REPLACEMENT','BUNDLE_COMPONENT','VARIANT')),
    quantity REAL NOT NULL DEFAULT 1.0,
    effective_from TEXT NOT NULL DEFAULT '2000-01-01',
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_product_id) REFERENCES product_registrations(id),
    FOREIGN KEY (target_product_id) REFERENCES product_registrations(id),
    UNIQUE(tenant_id, source_product_id, target_product_id, relationship_type)
);

CREATE INDEX idx_product_rels_source ON product_relationships(tenant_id, source_product_id, relationship_type);
CREATE INDEX idx_product_rels_target ON product_relationships(tenant_id, target_product_id);
```

### 2.7 Product Cross References
```sql
CREATE TABLE product_cross_references (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    external_system TEXT NOT NULL,
    external_code TEXT NOT NULL,
    external_description TEXT,
    external_uom TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (product_id) REFERENCES product_registrations(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, product_id, external_system, external_code)
);

CREATE INDEX idx_cross_refs_product ON product_cross_references(tenant_id, product_id);
CREATE INDEX idx_cross_refs_external ON product_cross_references(tenant_id, external_system, external_code);
```

### 2.8 Catalog Publishing
```sql
CREATE TABLE catalog_publishing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    catalog_code TEXT NOT NULL,
    catalog_name TEXT NOT NULL,
    catalog_type TEXT NOT NULL
        CHECK(catalog_type IN ('INTERNAL','PROCUREMENT','SALES','ECOMMERCE','PARTNER')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','ARCHIVED')),
    published_at TEXT,
    published_by TEXT,
    product_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, catalog_code)
);

CREATE INDEX idx_catalog_pub_tenant ON catalog_publishing(tenant_id, catalog_type, status);
```

### 2.9 Catalog Items
```sql
CREATE TABLE catalog_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    catalog_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_featured INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (catalog_id) REFERENCES catalog_publishing(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES product_registrations(id),
    UNIQUE(tenant_id, catalog_id, product_id)
);

CREATE INDEX idx_catalog_items_catalog ON catalog_items(catalog_id, sort_order);
```

---

## 3. REST API Endpoints

### 3.1 Products
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/products` | List products (filter by type, state, category) |
| POST | `/api/v1/producthub/products` | Create product |
| GET | `/api/v1/producthub/products/{id}` | Get product with attributes |
| PUT | `/api/v1/producthub/products/{id}` | Update product |
| DELETE | `/api/v1/producthub/products/{id}` | Deactivate product |
| POST | `/api/v1/producthub/products/search` | Advanced search |
| POST | `/api/v1/producthub/products/bulk` | Bulk create/update products |

### 3.2 Attributes
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/attributes` | List attribute definitions |
| POST | `/api/v1/producthub/attributes` | Create attribute |
| PUT | `/api/v1/producthub/attributes/{id}` | Update attribute |
| DELETE | `/api/v1/producthub/attributes/{id}` | Deactivate attribute |

### 3.3 Product Attribute Values
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/products/{id}/attributes` | Get product attributes |
| PUT | `/api/v1/producthub/products/{id}/attributes` | Set attribute values |
| DELETE | `/api/v1/producthub/products/{id}/attributes/{attrId}` | Remove attribute value |

### 3.4 Categories
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/categories` | List categories (tree) |
| POST | `/api/v1/producthub/categories` | Create category |
| PUT | `/api/v1/producthub/categories/{id}` | Update category |
| DELETE | `/api/v1/producthub/categories/{id}` | Delete category |
| POST | `/api/v1/producthub/categories/{id}/products` | Assign products to category |

### 3.5 Relationships
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/products/{id}/relationships` | Get product relationships |
| POST | `/api/v1/producthub/relationships` | Create relationship |
| DELETE | `/api/v1/producthub/relationships/{id}` | Remove relationship |

### 3.6 Cross References
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/products/{id}/cross-refs` | Get cross references |
| POST | `/api/v1/producthub/cross-refs` | Create cross reference |
| PUT | `/api/v1/producthub/cross-refs/{id}` | Update cross reference |
| DELETE | `/api/v1/producthub/cross-refs/{id}` | Remove cross reference |

### 3.7 Lifecycle
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/producthub/products/{id}/lifecycle` | Change lifecycle state |
| GET | `/api/v1/producthub/products/{id}/lifecycle/history` | Get lifecycle history |

### 3.8 Catalogs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/producthub/catalogs` | List catalogs |
| POST | `/api/v1/producthub/catalogs` | Create catalog |
| GET | `/api/v1/producthub/catalogs/{id}` | Get catalog |
| POST | `/api/v1/producthub/catalogs/{id}/publish` | Publish catalog |
| POST | `/api/v1/producthub/catalogs/{id}/archive` | Archive catalog |

### 3.9 Sync
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/producthub/sync/{target_system}` | Sync products to target system |
| GET | `/api/v1/producthub/sync/status` | Get sync status |

---

## 4. Business Rules

1. Product codes MUST be unique within a tenant.
2. Lifecycle state transitions MUST follow the defined sequence: CONCEPT → PROTOTYPE → PILOT → ACTIVE → PHASE_OUT → OBSOLETE.
3. Products in OBSOLETE state MUST NOT be available for new orders.
4. Attribute values MUST conform to the attribute type validation rules.
5. Category hierarchies MUST NOT exceed 10 levels deep.
6. Cross references MUST reference valid product IDs and external system codes.
7. Catalog publishing MUST only include products in ACTIVE lifecycle state unless configured otherwise.
8. Bulk product operations MUST be processed asynchronously with progress tracking.
9. Product searches MUST support full-text search across name, description, and searchable attributes.
10. Relationship types CROSS_SELL and UP_SELL MUST be directional (source → target).
11. SUBSTITUTE relationships MAY be bidirectional depending on configuration.
12. Sync operations MUST be idempotent — repeated syncs MUST produce the same result.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.producthub.v1;

service ProductHubService {
    rpc CreateProduct(CreateProductRequest) returns (ProductResponse);
    rpc GetProduct(GetProductRequest) returns (ProductResponse);
    rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
    rpc UpdateProduct(UpdateProductRequest) returns (ProductResponse);
    rpc SearchProducts(SearchProductsRequest) returns (SearchProductsResponse);
    rpc BulkUpdateProducts(BulkUpdateProductsRequest) returns (BulkUpdateResponse);

    rpc SetAttributeValues(SetAttributeValuesRequest) returns (AttributeValuesResponse);
    rpc GetAttributeValues(GetAttributeValuesRequest) returns (AttributeValuesResponse);

    rpc GetCategories(GetCategoriesRequest) returns (CategoriesResponse);
    rpc AssignCategory(AssignCategoryRequest) returns (CategoryAssignmentResponse);

    rpc CreateRelationship(CreateRelationshipRequest) returns (RelationshipResponse);
    rpc GetRelationships(GetRelationshipsRequest) returns (RelationshipsResponse);

    rpc ChangeLifecycleState(ChangeLifecycleStateRequest) returns (ProductResponse);

    rpc PublishCatalog(PublishCatalogRequest) returns (CatalogResponse);
    rpc SyncToSystem(SyncToSystemRequest) returns (SyncResponse);
}

// Entity messages

message ProductRegistration {
    string id = 1;
    string tenant_id = 2;
    string product_code = 3;
    string product_name = 4;
    string product_type = 5;
    string description = 6;
    string brand = 7;
    string manufacturer = 8;
    string lifecycle_state = 9;
    string primary_uom = 10;
    string secondary_uom = 11;
    double weight = 12;
    string weight_uom = 13;
    double volume = 14;
    string volume_uom = 15;
    string hazardous_class = 16;
    string country_of_origin = 17;
    string created_at = 18;
    string updated_at = 19;
}

message ProductAttribute {
    string id = 1;
    string tenant_id = 2;
    string attribute_code = 3;
    string attribute_name = 4;
    string attribute_type = 5;
    string uom = 6;
    bool is_required = 7;
    bool is_searchable = 8;
    bool is_comparable = 9;
    string enum_values = 10;
    string validation_rule = 11;
    int32 display_sequence = 12;
    string created_at = 13;
    string updated_at = 14;
}

message ProductAttributeValue {
    string id = 1;
    string tenant_id = 2;
    string product_id = 3;
    string attribute_id = 4;
    string text_value = 5;
    double number_value = 6;
    string date_value = 7;
    bool boolean_value = 8;
    string created_at = 9;
    string updated_at = 10;
}

message ProductCategory {
    string id = 1;
    string tenant_id = 2;
    string category_code = 3;
    string category_name = 4;
    string parent_category_id = 5;
    int32 category_level = 6;
    string description = 7;
    string image_url = 8;
    string created_at = 9;
    string updated_at = 10;
}

message ProductCategoryAssignment {
    string id = 1;
    string tenant_id = 2;
    string product_id = 3;
    string category_id = 4;
    bool is_primary = 5;
    string created_at = 6;
    string updated_at = 7;
}

message ProductRelationship {
    string id = 1;
    string tenant_id = 2;
    string source_product_id = 3;
    string target_product_id = 4;
    string relationship_type = 5;
    double quantity = 6;
    string effective_from = 7;
    string effective_to = 8;
    string created_at = 9;
    string updated_at = 10;
}

message ProductCrossReference {
    string id = 1;
    string tenant_id = 2;
    string product_id = 3;
    string external_system = 4;
    string external_code = 5;
    string external_description = 6;
    string external_uom = 7;
    string created_at = 8;
    string updated_at = 9;
}

message CatalogPublishing {
    string id = 1;
    string tenant_id = 2;
    string catalog_code = 3;
    string catalog_name = 4;
    string catalog_type = 5;
    string status = 6;
    string published_at = 7;
    string published_by = 8;
    int32 product_count = 9;
    string created_at = 10;
    string updated_at = 11;
}

// Request/Response messages

message CreateProductRequest {
    string tenant_id = 1;
    string product_code = 2;
    string product_name = 3;
    string product_type = 4;
    string description = 5;
    string brand = 6;
    string manufacturer = 7;
    string primary_uom = 8;
    string secondary_uom = 9;
    double weight = 10;
    string weight_uom = 11;
    double volume = 12;
    string volume_uom = 13;
    string hazardous_class = 14;
    string country_of_origin = 15;
}

message ProductResponse {
    ProductRegistration data = 1;
}

message GetProductRequest {
    string tenant_id = 1;
    string id = 2;
}

message ListProductsRequest {
    string tenant_id = 1;
    string product_type = 2;
    string lifecycle_state = 3;
    string category_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListProductsResponse {
    repeated ProductRegistration items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateProductRequest {
    string tenant_id = 1;
    string id = 2;
    string product_name = 3;
    string description = 4;
    string brand = 5;
    string manufacturer = 6;
    double weight = 7;
    double volume = 8;
}

message SearchProductsRequest {
    string tenant_id = 1;
    string query = 2;
    string product_type = 3;
    string lifecycle_state = 4;
    string category_id = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message SearchProductsResponse {
    repeated ProductRegistration items = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message BulkUpdateProductsRequest {
    string tenant_id = 1;
    repeated CreateProductRequest products = 2;
}

message BulkUpdateResponse {
    int32 created = 1;
    int32 updated = 2;
    int32 failed = 3;
    repeated string errors = 4;
}

message SetAttributeValuesRequest {
    string tenant_id = 1;
    string product_id = 2;
    repeated ProductAttributeValue values = 3;
}

message AttributeValuesResponse {
    repeated ProductAttributeValue values = 1;
}

message GetAttributeValuesRequest {
    string tenant_id = 1;
    string product_id = 2;
}

message GetCategoriesRequest {
    string tenant_id = 1;
    string parent_category_id = 2;
}

message CategoriesResponse {
    repeated ProductCategory items = 1;
}

message AssignCategoryRequest {
    string tenant_id = 1;
    string product_id = 2;
    string category_id = 3;
    bool is_primary = 4;
}

message CategoryAssignmentResponse {
    ProductCategoryAssignment data = 1;
}

message CreateRelationshipRequest {
    string tenant_id = 1;
    string source_product_id = 2;
    string target_product_id = 3;
    string relationship_type = 4;
    double quantity = 5;
    string effective_from = 6;
    string effective_to = 7;
}

message RelationshipResponse {
    ProductRelationship data = 1;
}

message GetRelationshipsRequest {
    string tenant_id = 1;
    string product_id = 2;
    string relationship_type = 3;
}

message RelationshipsResponse {
    repeated ProductRelationship items = 1;
}

message ChangeLifecycleStateRequest {
    string tenant_id = 1;
    string product_id = 2;
    string new_state = 3;
    string reason = 4;
}

message PublishCatalogRequest {
    string tenant_id = 1;
    string catalog_id = 2;
}

message CatalogResponse {
    CatalogPublishing data = 1;
}

message SyncToSystemRequest {
    string tenant_id = 1;
    string target_system = 2;
    repeated string product_ids = 3;
}

message SyncResponse {
    string target_system = 1;
    int32 synced_count = 2;
    int32 error_count = 3;
    repeated string errors = 4;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `inv-service` | Item master sync-back (inventory dimensions, units) |
| `plm-service` | Engineering product definitions |
| `pricing-service` | Pricing attribute requirements |

### Published To
| Service | Data |
|---------|------|
| `inv-service` | Product master data for inventory items |
| `om-service` | Product catalog for sales |
| `proc-service` | Product catalog for purchasing |
| `pricing-service` | Product attributes for pricing rules |
| `commerce-service` | Product catalog for e-commerce |
| `plm-service` | Lifecycle state changes |

---

## 7. Events

| Event Type | Payload | Description |
|------------|---------|-------------|
| `producthub.product.created` | `{product_id, code, name, type}` | Product created |
| `producthub.product.updated` | `{product_id, changed_fields}` | Product updated |
| `producthub.lifecycle.changed` | `{product_id, old_state, new_state}` | Lifecycle state changed |
| `producthub.attributes.updated` | `{product_id, attribute_ids}` | Attribute values changed |
| `producthub.relationship.created` | `{source_id, target_id, type}` | Product relationship added |
| `producthub.catalog.published` | `{catalog_id, product_count}` | Catalog published |
| `producthub.sync.completed` | `{target_system, synced_count, errors}` | Sync to external system completed |
| `producthub.category.updated` | `{category_id, change_type}` | Category structure changed |
