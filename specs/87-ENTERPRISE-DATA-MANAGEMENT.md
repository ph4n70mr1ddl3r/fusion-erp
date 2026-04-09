# 87 - Enterprise Data Management Service Specification

## 1. Domain Overview

Enterprise Data Management (EDM) provides master data governance for financial dimensions including accounts, entities, products, and custom dimensions. Manages dimension member definitions, hierarchical tree structures, cross-system dimension mappings, data quality validation rules, change approval governance workflows, and synchronized distribution to connected systems. Supports versioned dimension hierarchies with point-in-time validity, bidirectional mappings between source and target systems, automated quality checks on member data, and full audit trail tracking of all dimension changes. Integrates with GL for chart of accounts, PRODUCT-HUB for product dimensions, and all services for dimension synchronization.

**Bounded Context:** Enterprise Data Governance & Dimension Management
**Service Name:** `edm-service`
**Database:** `data/edm.db`
**HTTP Port:** 8119 | **gRPC Port:** 9119

---

## 2. Database Schema

### 2.1 EDM Dimensions
```sql
CREATE TABLE edm_dimensions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_name TEXT NOT NULL,
    dimension_code TEXT NOT NULL,
    dimension_type TEXT NOT NULL
        CHECK(dimension_type IN ('ACCOUNT','ENTITY','PRODUCT','CUSTOMER','SUPPLIER','PROJECT','DEPARTMENT','COST_CENTER','INTERCOMPANY','CUSTOM')),
    description TEXT,
    source_system TEXT,                            -- System of record for this dimension
    source_system_id TEXT,
    is_hierarchical INTEGER NOT NULL DEFAULT 1,
    max_depth INTEGER NOT NULL DEFAULT 10,
    validity_start TEXT NOT NULL,
    validity_end TEXT,
    governance_level TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(governance_level IN ('OPEN','STANDARD','STRICT','LOCKED')),
    owner_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dimension_code)
);

CREATE INDEX idx_dimensions_tenant_type ON edm_dimensions(tenant_id, dimension_type);
CREATE INDEX idx_dimensions_tenant_active ON edm_dimensions(tenant_id, is_active);
CREATE INDEX idx_dimensions_tenant_owner ON edm_dimensions(tenant_id, owner_id);
CREATE INDEX idx_dimensions_tenant_governance ON edm_dimensions(tenant_id, governance_level);
```

### 2.2 Dimension Members
```sql
CREATE TABLE edm_dimension_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    member_code TEXT NOT NULL,
    member_name TEXT NOT NULL,
    member_description TEXT,
    parent_member_id TEXT,
    member_level INTEGER NOT NULL DEFAULT 0,
    member_path TEXT,                              -- Materialized path (e.g., /root/child/grandchild)
    sort_order INTEGER NOT NULL DEFAULT 0,
    member_type TEXT NOT NULL DEFAULT 'BASE'
        CHECK(member_type IN ('ROOT','GROUP','BASE','ALIAS')),
    alias_of_member_id TEXT,
    attributes TEXT,                               -- JSON: extensible member attributes
    validity_start TEXT NOT NULL,
    validity_end TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','PENDING_DELETION')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dimension_id) REFERENCES edm_dimensions(id),
    UNIQUE(tenant_id, dimension_id, member_code)
);

CREATE INDEX idx_members_tenant_dimension ON edm_dimension_members(tenant_id, dimension_id);
CREATE INDEX idx_members_tenant_parent ON edm_dimension_members(tenant_id, parent_member_id);
CREATE INDEX idx_members_tenant_path ON edm_dimension_members(tenant_id, member_path);
CREATE INDEX idx_members_tenant_status ON edm_dimension_members(tenant_id, status);
CREATE INDEX idx_members_tenant_type ON edm_dimension_members(tenant_id, member_type);
CREATE INDEX idx_members_tenant_validity ON edm_dimension_members(tenant_id, validity_start, validity_end);
```

### 2.3 Dimension Hierarchies
```sql
CREATE TABLE edm_dimension_hierarchies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    hierarchy_name TEXT NOT NULL,
    hierarchy_type TEXT NOT NULL DEFAULT 'MANAGEMENT'
        CHECK(hierarchy_type IN ('MANAGEMENT','LEGAL','REPORTING','CONSOLIDATION','REGULATORY','CUSTOM')),
    description TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    validity_start TEXT NOT NULL,
    validity_end TEXT,
    version_label TEXT NOT NULL DEFAULT '1.0',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dimension_id) REFERENCES edm_dimensions(id),
    UNIQUE(tenant_id, dimension_id, hierarchy_name)
);

CREATE INDEX idx_hierarchies_tenant_dimension ON edm_dimension_hierarchies(tenant_id, dimension_id);
CREATE INDEX idx_hierarchies_tenant_type ON edm_dimension_hierarchies(tenant_id, hierarchy_type);
CREATE INDEX idx_hierarchies_tenant_default ON edm_dimension_hierarchies(tenant_id, is_default);
CREATE INDEX idx_hierarchies_tenant_validity ON edm_dimension_hierarchies(tenant_id, validity_start, validity_end);
```

### 2.4 Hierarchy Nodes
```sql
CREATE TABLE edm_hierarchy_nodes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    hierarchy_id TEXT NOT NULL,
    member_id TEXT NOT NULL,
    parent_node_id TEXT,
    node_level INTEGER NOT NULL DEFAULT 0,
    node_path TEXT,                                -- Materialized path within hierarchy
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_leaf INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (hierarchy_id) REFERENCES edm_dimension_hierarchies(id) ON DELETE CASCADE,
    FOREIGN KEY (member_id) REFERENCES edm_dimension_members(id),
    UNIQUE(tenant_id, hierarchy_id, member_id)
);

CREATE INDEX idx_nodes_tenant_hierarchy ON edm_hierarchy_nodes(tenant_id, hierarchy_id);
CREATE INDEX idx_nodes_tenant_parent ON edm_hierarchy_nodes(tenant_id, parent_node_id);
CREATE INDEX idx_nodes_tenant_path ON edm_hierarchy_nodes(tenant_id, node_path);
CREATE INDEX idx_nodes_tenant_member ON edm_hierarchy_nodes(tenant_id, member_id);
```

### 2.5 Dimension Mappings
```sql
CREATE TABLE edm_dimension_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_system_id TEXT NOT NULL,
    target_system_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    source_member_code TEXT NOT NULL,
    source_member_name TEXT,
    target_member_code TEXT NOT NULL,
    target_member_name TEXT,
    mapping_type TEXT NOT NULL DEFAULT 'ONE_TO_ONE'
        CHECK(mapping_type IN ('ONE_TO_ONE','ONE_TO_MANY','MANY_TO_ONE','AGGREGATION')),
    mapping_direction TEXT NOT NULL DEFAULT 'BIDIRECTIONAL'
        CHECK(mapping_direction IN ('SOURCE_TO_TARGET','TARGET_TO_SOURCE','BIDIRECTIONAL')),
    confidence_score_real REAL NOT NULL DEFAULT 100.0,
    is_validated INTEGER NOT NULL DEFAULT 0,
    validated_by TEXT,
    validated_at TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dimension_id) REFERENCES edm_dimensions(id),
    UNIQUE(tenant_id, source_system_id, target_system_id, dimension_id, source_member_code)
);

CREATE INDEX idx_mappings_tenant_source ON edm_dimension_mappings(tenant_id, source_system_id);
CREATE INDEX idx_mappings_tenant_target ON edm_dimension_mappings(tenant_id, target_system_id);
CREATE INDEX idx_mappings_tenant_dimension ON edm_dimension_mappings(tenant_id, dimension_id);
CREATE INDEX idx_mappings_tenant_validated ON edm_dimension_mappings(tenant_id, is_validated);
CREATE INDEX idx_mappings_tenant_effective ON edm_dimension_mappings(tenant_id, effective_from, effective_to);
```

### 2.6 Dimension Validations
```sql
CREATE TABLE edm_dimension_validations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    validation_name TEXT NOT NULL,
    validation_type TEXT NOT NULL
        CHECK(validation_type IN ('REQUIRED_FIELD','UNIQUENESS','FORMAT','RANGE','CROSS_REFERENCE','HIERARCHY_INTEGRITY','CUSTOM')),
    validation_expression TEXT NOT NULL,            -- Expression or regex for validation
    error_message TEXT NOT NULL,
    error_severity TEXT NOT NULL DEFAULT 'ERROR'
        CHECK(error_severity IN ('WARNING','ERROR','CRITICAL')),
    is_active INTEGER NOT NULL DEFAULT 1,
    execution_order INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dimension_id) REFERENCES edm_dimensions(id),
    UNIQUE(tenant_id, dimension_id, validation_name)
);

CREATE INDEX idx_validations_tenant_dimension ON edm_dimension_validations(tenant_id, dimension_id);
CREATE INDEX idx_validations_tenant_type ON edm_dimension_validations(tenant_id, validation_type);
CREATE INDEX idx_validations_tenant_active ON edm_dimension_validations(tenant_id, is_active);
```

### 2.7 Governance Workflows
```sql
CREATE TABLE edm_governance_workflows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT,
    workflow_name TEXT NOT NULL,
    workflow_type TEXT NOT NULL
        CHECK(workflow_type IN ('MEMBER_CREATE','MEMBER_UPDATE','MEMBER_DELETE','HIERARCHY_CHANGE','MAPPING_CHANGE','BULK_IMPORT')),
    description TEXT,
    requires_approval INTEGER NOT NULL DEFAULT 1,
    approval_levels INTEGER NOT NULL DEFAULT 1,
    auto_approve_threshold INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, workflow_name)
);

CREATE INDEX idx_workflows_tenant_dimension ON edm_governance_workflows(tenant_id, dimension_id);
CREATE INDEX idx_workflows_tenant_type ON edm_governance_workflows(tenant_id, workflow_type);
CREATE INDEX idx_workflows_tenant_active ON edm_governance_workflows(tenant_id, is_active);
```

### 2.8 Change Requests
```sql
CREATE TABLE edm_change_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    workflow_id TEXT NOT NULL,
    request_type TEXT NOT NULL
        CHECK(request_type IN ('CREATE_MEMBER','UPDATE_MEMBER','DELETE_MEMBER','MOVE_MEMBER','ADD_MAPPING','REMOVE_MAPPING','HIERARCHY_RESTRUCTURE')),
    requested_change TEXT NOT NULL,                -- JSON: description of the change
    before_snapshot TEXT,                          -- JSON: state before change
    after_snapshot TEXT,                           -- JSON: proposed state after change
    status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(status IN ('SUBMITTED','IN_REVIEW','APPROVED','REJECTED','IMPLEMENTED','ROLLED_BACK')),
    priority TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    justification TEXT,
    impact_assessment TEXT,                        -- JSON: affected systems and members
    implemented_at TEXT,
    rolled_back_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dimension_id) REFERENCES edm_dimensions(id),
    FOREIGN KEY (workflow_id) REFERENCES edm_governance_workflows(id)
);

CREATE INDEX idx_change_requests_tenant_dimension ON edm_change_requests(tenant_id, dimension_id);
CREATE INDEX idx_change_requests_tenant_status ON edm_change_requests(tenant_id, status);
CREATE INDEX idx_change_requests_tenant_type ON edm_change_requests(tenant_id, request_type);
CREATE INDEX idx_change_requests_tenant_workflow ON edm_change_requests(tenant_id, workflow_id);
CREATE INDEX idx_change_requests_tenant_priority ON edm_change_requests(tenant_id, priority);
```

### 2.9 System Registrations
```sql
CREATE TABLE edm_system_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    system_name TEXT NOT NULL,
    system_type TEXT NOT NULL
        CHECK(system_type IN ('GL','ERP','CRM','SCM','HCM','BI','DATA_WAREHOUSE','EPM','CUSTOM')),
    connection_config TEXT,                        -- JSON: endpoint, auth details
    dimensions_supported TEXT,                     -- JSON array of dimension IDs
    sync_direction TEXT NOT NULL DEFAULT 'BIDIRECTIONAL'
        CHECK(sync_direction IN ('PUSH','PULL','BIDIRECTIONAL')),
    last_sync_at TEXT,
    sync_status TEXT NOT NULL DEFAULT 'CONFIGURED'
        CHECK(sync_status IN ('CONFIGURED','CONNECTED','SYNCING','ERROR','DISCONNECTED')),
    health_check_url TEXT,
    error_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, system_name)
);

CREATE INDEX idx_systems_tenant_type ON edm_system_registrations(tenant_id, system_type);
CREATE INDEX idx_systems_tenant_status ON edm_system_registrations(tenant_id, sync_status);
CREATE INDEX idx_systems_tenant_active ON edm_system_registrations(tenant_id, is_active);
```

### 2.10 Sync Jobs
```sql
CREATE TABLE edm_sync_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    system_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    sync_type TEXT NOT NULL
        CHECK(sync_type IN ('FULL','INCREMENTAL','DELTA','VALIDATION_ONLY')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','PARTIALLY_COMPLETED')),
    triggered_by TEXT NOT NULL
        CHECK(triggered_by IN ('MANUAL','SCHEDULED','EVENT','CHANGE_REQUEST')),
    change_request_id TEXT,
    records_processed INTEGER NOT NULL DEFAULT 0,
    records_created INTEGER NOT NULL DEFAULT 0,
    records_updated INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    error_details TEXT,                            -- JSON: error messages by record
    started_at TEXT,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (system_id) REFERENCES edm_system_registrations(id),
    FOREIGN KEY (dimension_id) REFERENCES edm_dimensions(id)
);

CREATE INDEX idx_sync_jobs_tenant_system ON edm_sync_jobs(tenant_id, system_id);
CREATE INDEX idx_sync_jobs_tenant_dimension ON edm_sync_jobs(tenant_id, dimension_id);
CREATE INDEX idx_sync_jobs_tenant_status ON edm_sync_jobs(tenant_id, status);
CREATE INDEX idx_sync_jobs_tenant_dates ON edm_sync_jobs(tenant_id, started_at, completed_at);
```

### 2.11 Audit Trails
```sql
CREATE TABLE edm_audit_trails (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT,
    member_id TEXT,
    action_type TEXT NOT NULL
        CHECK(action_type IN ('CREATE','UPDATE','DELETE','MOVE','MAP','UNMAP','VALIDATE','SYNC','APPROVE','REJECT')),
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('DIMENSION','MEMBER','HIERARCHY','NODE','MAPPING','VALIDATION','WORKFLOW','CHANGE_REQUEST')),
    entity_id TEXT NOT NULL,
    field_name TEXT,
    old_value TEXT,
    new_value TEXT,
    change_reason TEXT,
    source_ip TEXT,
    user_agent TEXT,
    change_request_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_audit_tenant_dimension ON edm_audit_trails(tenant_id, dimension_id);
CREATE INDEX idx_audit_tenant_member ON edm_audit_trails(tenant_id, member_id);
CREATE INDEX idx_audit_tenant_action ON edm_audit_trails(tenant_id, action_type);
CREATE INDEX idx_audit_tenant_entity ON edm_audit_trails(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_tenant_dates ON edm_audit_trails(tenant_id, created_at);
CREATE INDEX idx_audit_tenant_user ON edm_audit_trails(tenant_id, created_by);
```

---

## 3. REST API Endpoints

```
# Dimensions
GET/POST      /api/v1/edm/dimensions                         Permission: edm.dimensions.read/create
GET/PUT       /api/v1/edm/dimensions/{id}                    Permission: edm.dimensions.read/update
PATCH         /api/v1/edm/dimensions/{id}/deactivate         Permission: edm.dimensions.update
DELETE        /api/v1/edm/dimensions/{id}                    Permission: edm.dimensions.delete

# Dimension Members
GET/POST      /api/v1/edm/dimensions/{id}/members            Permission: edm.members.read/create
GET/PUT       /api/v1/edm/members/{id}                       Permission: edm.members.read/update
PATCH         /api/v1/edm/members/{id}/deactivate            Permission: edm.members.update
GET           /api/v1/edm/dimensions/{id}/members/tree       Permission: edm.members.read
POST          /api/v1/edm/members/bulk-create                Permission: edm.members.create
GET           /api/v1/edm/members/{id}/children              Permission: edm.members.read
GET           /api/v1/edm/members/{id}/ancestors             Permission: edm.members.read

# Dimension Hierarchies
GET/POST      /api/v1/edm/dimensions/{id}/hierarchies        Permission: edm.hierarchies.read/create
GET/PUT       /api/v1/edm/hierarchies/{id}                   Permission: edm.hierarchies.read/update
GET           /api/v1/edm/hierarchies/{id}/tree              Permission: edm.hierarchies.read
POST          /api/v1/edm/hierarchies/{id}/restructure       Permission: edm.hierarchies.update
GET           /api/v1/edm/hierarchies/{id}/compare/{other_id} Permission: edm.hierarchies.read

# Dimension Mappings
GET/POST      /api/v1/edm/mappings                           Permission: edm.mappings.read/create
GET/PUT       /api/v1/edm/mappings/{id}                      Permission: edm.mappings.read/update
GET           /api/v1/edm/mappings/translate                  Permission: edm.mappings.read
POST          /api/v1/edm/mappings/bulk-import                Permission: edm.mappings.create
POST          /api/v1/edm/mappings/{id}/validate             Permission: edm.mappings.update
GET           /api/v1/edm/mappings/unmapped                   Permission: edm.mappings.read

# Validations
GET/POST      /api/v1/edm/dimensions/{id}/validations        Permission: edm.validations.read/create
GET/PUT       /api/v1/edm/validations/{id}                   Permission: edm.validations.read/update
POST          /api/v1/edm/dimensions/{id}/validate           Permission: edm.validations.execute
GET           /api/v1/edm/dimensions/{id}/validation-results Permission: edm.validations.read
DELETE        /api/v1/edm/validations/{id}                   Permission: edm.validations.delete

# Governance Workflows
GET/POST      /api/v1/edm/governance/workflows               Permission: edm.governance.read/create
GET/PUT       /api/v1/edm/governance/workflows/{id}          Permission: edm.governance.read/update
DELETE        /api/v1/edm/governance/workflows/{id}          Permission: edm.governance.delete

# Change Requests
GET/POST      /api/v1/edm/change-requests                    Permission: edm.changerequests.read/create
GET           /api/v1/edm/change-requests/{id}               Permission: edm.changerequests.read
POST          /api/v1/edm/change-requests/{id}/approve       Permission: edm.changerequests.approve
POST          /api/v1/edm/change-requests/{id}/reject        Permission: edm.changerequests.reject
POST          /api/v1/edm/change-requests/{id}/implement     Permission: edm.changerequests.execute
POST          /api/v1/edm/change-requests/{id}/rollback      Permission: edm.changerequests.execute
GET           /api/v1/edm/change-requests/{id}/impact        Permission: edm.changerequests.read

# System Registrations
GET/POST      /api/v1/edm/systems                            Permission: edm.systems.read/create
GET/PUT       /api/v1/edm/systems/{id}                       Permission: edm.systems.read/update
GET           /api/v1/edm/systems/{id}/health                Permission: edm.systems.read
DELETE        /api/v1/edm/systems/{id}                       Permission: edm.systems.delete

# Sync Jobs
POST          /api/v1/edm/sync/trigger                       Permission: edm.sync.execute
GET           /api/v1/edm/sync/jobs                          Permission: edm.sync.read
GET           /api/v1/edm/sync/jobs/{id}                     Permission: edm.sync.read
POST          /api/v1/edm/sync/jobs/{id}/retry               Permission: edm.sync.execute
GET           /api/v1/edm/sync/status                        Permission: edm.sync.read

# Audit Trail
GET           /api/v1/edm/audit-trail                        Permission: edm.audit.read
GET           /api/v1/edm/audit-trail/dimension/{id}         Permission: edm.audit.read
GET           /api/v1/edm/audit-trail/member/{id}            Permission: edm.audit.read
GET           /api/v1/edm/audit-trail/export                 Permission: edm.audit.export
```

---

## 4. Business Rules

### 4.1 Dimension Management
1. Each dimension MUST have a unique code within a tenant
2. Dimensions with governance_level LOCKED MUST require approval for all member changes
3. Dimension validity periods MUST overlap correctly for continuous coverage
4. Dimensions marked as hierarchical MUST support tree structures; non-hierarchical dimensions MUST be flat lists
5. A dimension cannot be deactivated if it has active members or active mappings
6. Custom dimensions MUST support user-defined attribute schemas stored as JSON

### 4.2 Member Management
1. Member codes MUST be unique within a dimension for each tenant
2. Member paths MUST use materialized path pattern for efficient tree queries
3. Moving a member to a new parent MUST update the paths of all descendants
4. Members with ACTIVE status can be used in transactions; DRAFT members cannot
5. Alias members MUST reference a valid base member within the same dimension
6. Member validity periods MUST not create gaps in time coverage for active members
7. Pending deletion members MUST be reviewed and approved before actual deletion

### 4.3 Hierarchy Rules
1. Each dimension MUST have at least one default hierarchy
2. Hierarchy nodes MUST not exceed the max_depth defined on the dimension
3. Circular references in hierarchy nodes MUST be prevented
4. Hierarchy restructuring MUST preserve existing mappings and audit trails
5. Hierarchy versions MUST support point-in-time queries for historical reporting
6. Multiple hierarchy types (management, legal, reporting) MAY coexist for the same dimension

### 4.4 Mapping Management
1. Dimension mappings MUST reference valid members in both source and target systems
2. ONE_TO_MANY and MANY_TO_ONE mappings MUST be flagged for review
3. Mapping confidence scores below 80% SHOULD trigger manual validation
4. Bidirectional mappings MUST be consistent in both directions
5. Mapping effective dates MUST not overlap for the same source-target-member combination
6. Unmapped members MUST be reported during sync operations

### 4.5 Data Quality Validation
1. All validations MUST run when a member is created or updated
2. Validation errors of severity CRITICAL MUST block the change
3. Validation warnings SHOULD be logged but allow the change to proceed
4. Custom validations MUST use a supported expression language (regex, JSON path, or SQL-like)
5. Cross-reference validations MUST check referenced member existence in target dimensions
6. Hierarchy integrity validations MUST verify no orphaned nodes exist

### 4.6 Change Governance
1. Change requests for STRICT or LOCKED governance dimensions MUST require approval
2. Before and after snapshots MUST be captured for all change requests
3. Impact assessment MUST list all affected systems and downstream dependencies
4. Approved changes MUST be implemented atomically (all or nothing)
5. Failed implementations MUST be rolled back automatically
6. Change requests MAY NOT be modified after approval; a new request is required

### 4.7 Sync Operations
1. Full sync replaces all dimension data in the target system
2. Incremental sync only sends changes since the last successful sync
3. Delta sync sends only the specific changes from a change request
4. Sync failures after 3 retries MUST alert the dimension owner
5. Sync jobs MUST log all records created, updated, and failed
6. Validation-only sync checks data without making changes

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `edm.dimension.created` | New dimension defined | All services |
| `edm.member.added` | Member created in a dimension | GL, all dimension consumers |
| `edm.member.updated` | Member data changed | GL, all dimension consumers |
| `edm.hierarchy.changed` | Hierarchy structure modified | GL, EPM, Reporting |
| `edm.change.approved` | Change request approved | Workflow, Notification |
| `edm.change.implemented` | Change deployed to dimension | All dimension consumers |
| `edm.sync.completed` | Sync job finished | Notification, target systems |
| `edm.sync.failed` | Sync job failed | Notification, dimension owner |
| `edm.validation.failed` | Quality validation failed | Notification, governance team |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.edm.v1;

service EdmService {
    rpc GetDimension(GetDimensionRequest) returns (GetDimensionResponse);
    rpc GetMembers(GetMembersRequest) returns (GetMembersResponse);
    rpc GetHierarchyTree(GetHierarchyTreeRequest) returns (GetHierarchyTreeResponse);
    rpc TranslateMapping(TranslateMappingRequest) returns (TranslateMappingResponse);
    rpc ValidateDimension(ValidateDimensionRequest) returns (ValidateDimensionResponse);
    rpc TriggerSync(TriggerSyncRequest) returns (TriggerSyncResponse);
    rpc GetAuditTrail(GetAuditTrailRequest) returns (GetAuditTrailResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **GL:** Chart of accounts structure, account segment definitions, account types and categories
- **PRODUCT-HUB:** Product dimension definitions, product categories and hierarchies
- **Auth:** User identity for governance workflows, approval routing, and audit attribution
- **Workflow:** Approval process definitions, task assignment, escalation rules
- **All Services:** Dimension requirements and supported dimension types for system registration

### 6.2 Data Published To
- **GL:** Account dimension members, hierarchy updates, member deactivations
- **PRODUCT-HUB:** Product dimension synchronizations, category hierarchy updates
- **EPM:** Consolidation entity hierarchies, reporting dimension structures
- **Reporting:** All dimension and hierarchy data for reporting metadata
- **All Registered Systems:** Dimension sync data via configured sync jobs
- **Workflow:** Change request approval tasks, governance workflow routing
- **Notification:** Change request notifications, sync completion/failure alerts, validation failure alerts

---

## 7. Migrations

1. V001: `edm_dimensions`
2. V002: `edm_dimension_members`
3. V003: `edm_dimension_hierarchies`
4. V004: `edm_hierarchy_nodes`
5. V005: `edm_dimension_mappings`
6. V006: `edm_dimension_validations`
7. V007: `edm_governance_workflows`
8. V008: `edm_change_requests`
9. V009: `edm_system_registrations`
10. V010: `edm_sync_jobs`
11. V011: `edm_audit_trails`
12. V012: Triggers for `updated_at`
