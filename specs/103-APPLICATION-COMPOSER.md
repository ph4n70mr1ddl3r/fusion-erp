# 103 - Application Composer Service Specification

## 1. Domain Overview

Application Composer is a low-code extension platform for customizing and extending Oracle Fusion applications. It supports custom objects, custom fields, business logic scripting, UI page customization, workflow extensions, and sandbox-based lifecycle management. It enables citizen developers and administrators to tailor applications without code changes to the core platform.

**Bounded Context:** Application Extension & Customization
**Service Name:** `composer-service`
**Database:** `data/composer.db`
**HTTP Port:** 8140 | **gRPC Port:** 9140

---

## 2. Database Schema

### 2.1 Extensions
```sql
CREATE TABLE extensions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_name TEXT NOT NULL,
    extension_type TEXT NOT NULL
        CHECK(extension_type IN ('CUSTOM_OBJECT','CUSTOM_FIELD','PAGE_LAYOUT','BUSINESS_LOGIC','WORKFLOW','REPORT','DASHBOARD')),
    target_application TEXT NOT NULL
        CHECK(target_application IN ('ERP','SCM','HCM','CX','EPM')),
    target_object TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','ACTIVE','INACTIVE','DEPRECATED')),
    sandbox_id TEXT,
    deployed_date TEXT,
    deployed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, extension_name)
);

CREATE INDEX idx_extensions_tenant_type ON extensions(tenant_id, extension_type);
CREATE INDEX idx_extensions_tenant_app ON extensions(tenant_id, target_application);
CREATE INDEX idx_extensions_tenant_status ON extensions(tenant_id, status);
```

### 2.2 Custom Objects
```sql
CREATE TABLE custom_objects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_id TEXT NOT NULL,
    object_name TEXT NOT NULL,
    plural_name TEXT NOT NULL,
    description TEXT,
    is_activity_enabled INTEGER NOT NULL DEFAULT 0,
    is_searchable INTEGER NOT NULL DEFAULT 1,
    record_name_field TEXT,
    icon_reference TEXT,
    api_name TEXT NOT NULL,
    is_system INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, api_name)
);

CREATE INDEX idx_custom_objects_tenant_ext ON custom_objects(tenant_id, extension_id);
CREATE INDEX idx_custom_objects_tenant_api ON custom_objects(tenant_id, api_name);
```

### 2.3 Custom Fields
```sql
CREATE TABLE custom_fields (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_id TEXT NOT NULL,
    object_id TEXT NOT NULL,
    field_name TEXT NOT NULL,
    display_label TEXT NOT NULL,
    field_type TEXT NOT NULL
        CHECK(field_type IN ('TEXT','NUMBER','DATE','DATETIME','PICKLIST','CHECKBOX','LOOKUP','CURRENCY','FORMULA','FILE','RICH_TEXT')),
    default_value TEXT,
    is_required INTEGER NOT NULL DEFAULT 0,
    is_unique INTEGER NOT NULL DEFAULT 0,
    is_indexed INTEGER NOT NULL DEFAULT 0,
    display_order INTEGER NOT NULL DEFAULT 0,
    help_text TEXT,
    validation_rules TEXT,      -- JSON
    formula_definition TEXT,
    lookup_object_id TEXT,
    picklist_values TEXT,       -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE,
    FOREIGN KEY (object_id) REFERENCES custom_objects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, object_id, field_name)
);

CREATE INDEX idx_custom_fields_tenant_object ON custom_fields(tenant_id, object_id);
CREATE INDEX idx_custom_fields_tenant_type ON custom_fields(tenant_id, field_type);
```

### 2.4 Page Layouts
```sql
CREATE TABLE page_layouts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_id TEXT NOT NULL,
    object_id TEXT NOT NULL,
    layout_name TEXT NOT NULL,
    layout_type TEXT NOT NULL
        CHECK(layout_type IN ('CREATE','EDIT','DETAIL','LIST')),
    target_role_id TEXT,
    sections TEXT NOT NULL,       -- JSON
    fields_config TEXT NOT NULL,  -- JSON
    related_lists TEXT,           -- JSON
    actions_config TEXT,          -- JSON
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE,
    FOREIGN KEY (object_id) REFERENCES custom_objects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, object_id, layout_name)
);

CREATE INDEX idx_page_layouts_tenant_object ON page_layouts(tenant_id, object_id);
CREATE INDEX idx_page_layouts_tenant_type ON page_layouts(tenant_id, layout_type);
```

### 2.5 Business Logic Scripts
```sql
CREATE TABLE business_logic_scripts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_id TEXT NOT NULL,
    object_id TEXT NOT NULL,
    script_name TEXT NOT NULL,
    trigger_type TEXT NOT NULL
        CHECK(trigger_type IN ('BEFORE_CREATE','AFTER_CREATE','BEFORE_UPDATE','AFTER_UPDATE','BEFORE_DELETE','AFTER_DELETE','ON_FIELD_CHANGE')),
    trigger_fields TEXT,  -- JSON
    script_body TEXT NOT NULL,
    language TEXT NOT NULL DEFAULT 'GROOVY'
        CHECK(language IN ('GROOVY','JAVASCRIPT')),
    error_handling_config TEXT,  -- JSON
    execution_order INTEGER NOT NULL DEFAULT 1,
    last_execution_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE,
    FOREIGN KEY (object_id) REFERENCES custom_objects(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, object_id, script_name)
);

CREATE INDEX idx_scripts_tenant_object ON business_logic_scripts(tenant_id, object_id);
CREATE INDEX idx_scripts_tenant_trigger ON business_logic_scripts(tenant_id, trigger_type);
```

### 2.6 Extension Sandboxes
```sql
CREATE TABLE extension_sandboxes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    sandbox_name TEXT NOT NULL,
    description TEXT,
    source_sandbox_id TEXT,
    status TEXT NOT NULL DEFAULT 'CREATED'
        CHECK(status IN ('CREATED','IN_DEVELOPMENT','IN_TESTING','APPROVED','PUBLISHED','DISCARDED')),
    published_by TEXT,
    published_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, sandbox_name)
);

CREATE INDEX idx_sandboxes_tenant_status ON extension_sandboxes(tenant_id, status);
```

### 2.7 Extension Dependencies
```sql
CREATE TABLE extension_dependencies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_id TEXT NOT NULL,
    depends_on_extension_id TEXT NOT NULL,
    dependency_type TEXT NOT NULL
        CHECK(dependency_type IN ('REFERENCES','EMBEDS','TRIGGERS','USES_DATA')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE,
    FOREIGN KEY (depends_on_extension_id) REFERENCES extensions(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, extension_id, depends_on_extension_id)
);

CREATE INDEX idx_deps_tenant_ext ON extension_dependencies(tenant_id, extension_id);
CREATE INDEX idx_deps_tenant_dep ON extension_dependencies(tenant_id, depends_on_extension_id);
```

### 2.8 Extension Audit Log
```sql
CREATE TABLE extension_audit_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    extension_id TEXT NOT NULL,
    action TEXT NOT NULL
        CHECK(action IN ('CREATED','MODIFIED','DEPLOYED','DEACTIVATED','DEPRECATED')),
    performed_by TEXT NOT NULL,
    details TEXT,  -- JSON
    sandbox_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE
);

CREATE INDEX idx_audit_log_tenant_ext ON extension_audit_log(tenant_id, extension_id);
CREATE INDEX idx_audit_log_tenant_action ON extension_audit_log(tenant_id, action);
CREATE INDEX idx_audit_log_tenant_date ON extension_audit_log(tenant_id, created_at);
```

---

## 3. REST API Endpoints

### Extensions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/composer/extensions` | Create an extension |
| GET | `/api/v1/composer/extensions` | List extensions |
| GET | `/api/v1/composer/extensions/{id}` | Get extension details |
| PUT | `/api/v1/composer/extensions/{id}` | Update extension |
| POST | `/api/v1/composer/extensions/{id}/deploy` | Deploy an extension |
| POST | `/api/v1/composer/extensions/{id}/deactivate` | Deactivate an extension |
| GET | `/api/v1/composer/extensions/{id}/dependencies` | Get extension dependencies |

### Custom Objects
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/composer/custom-objects` | Create a custom object |
| GET | `/api/v1/composer/custom-objects` | List custom objects |
| GET | `/api/v1/composer/custom-objects/{id}` | Get custom object details |
| PUT | `/api/v1/composer/custom-objects/{id}` | Update custom object |
| POST | `/api/v1/composer/custom-objects/{id}/add-field` | Add a field to custom object |

### Custom Fields
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/composer/custom-fields` | Create a custom field |
| GET | `/api/v1/composer/custom-fields` | List custom fields |
| GET | `/api/v1/composer/custom-fields/{id}` | Get custom field details |
| PUT | `/api/v1/composer/custom-fields/{id}` | Update custom field |
| POST | `/api/v1/composer/custom-fields/{id}/validate` | Validate custom field |

### Page Layouts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/composer/page-layouts` | Create a page layout |
| GET | `/api/v1/composer/page-layouts` | List page layouts |
| GET | `/api/v1/composer/page-layouts/{id}` | Get page layout details |
| PUT | `/api/v1/composer/page-layouts/{id}` | Update page layout |
| POST | `/api/v1/composer/page-layouts/{id}/preview` | Preview page layout |

### Business Logic Scripts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/composer/scripts` | Create a business logic script |
| GET | `/api/v1/composer/scripts` | List scripts |
| GET | `/api/v1/composer/scripts/{id}` | Get script details |
| PUT | `/api/v1/composer/scripts/{id}` | Update script |
| POST | `/api/v1/composer/scripts/{id}/test` | Test a script |
| POST | `/api/v1/composer/scripts/{id}/execute` | Execute a script |

### Sandboxes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/composer/sandboxes` | Create a sandbox |
| GET | `/api/v1/composer/sandboxes` | List sandboxes |
| GET | `/api/v1/composer/sandboxes/{id}` | Get sandbox details |
| PUT | `/api/v1/composer/sandboxes/{id}` | Update sandbox |
| POST | `/api/v1/composer/sandboxes/{id}/publish` | Publish a sandbox |
| POST | `/api/v1/composer/sandboxes/{id}/discard` | Discard a sandbox |
| POST | `/api/v1/composer/sandboxes/{id}/clone` | Clone a sandbox |

### Dependencies
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/composer/extensions/{id}/dependency-tree` | Get dependency tree |
| GET | `/api/v1/composer/extensions/{id}/impact-analysis` | Get impact analysis |

### Audit
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/composer/audit-log` | Get extension audit log |

---

## 4. Business Rules

1. A custom field name MUST follow the naming convention `cx_<object_api_name>_<field_name>` and MUST contain only lowercase alphanumeric characters and underscores.
2. A custom field of type `FORMULA` MUST have a valid `formula_definition`; the system MUST validate the formula syntax before saving.
3. Business logic scripts MUST complete execution within 10 seconds; scripts exceeding this limit MUST be terminated and an error MUST be logged.
4. An extension in `DRAFT` status MUST NOT be accessible by end users of the target application.
5. A sandbox in `PUBLISHED` status MUST NOT be modified; changes MUST be made in a new sandbox cloned from the published one.
6. Extension dependencies MUST be validated before deployment; the system MUST NOT deploy an extension that references a non-existent or inactive dependent extension.
7. A page layout MUST have at least one section and one field configured; empty layouts MUST be rejected.
8. Business logic scripts with the same trigger type on the same object MUST be executed in ascending `execution_order`.
9. The system SHOULD limit each tenant to a maximum of 500 custom fields per custom object and 100 custom objects per application.
10. An extension MAY be deprecated only after all dependent extensions have been updated or removed.
11. The system MUST log all extension lifecycle actions (create, modify, deploy, deactivate, deprecate) in the extension audit log.
12. A sandbox MUST NOT be published if it contains extensions with compilation or validation errors.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package composer.v1;

service ApplicationComposerService {
    // Extensions
    rpc CreateExtension(CreateExtensionRequest) returns (CreateExtensionResponse);
    rpc GetExtension(GetExtensionRequest) returns (GetExtensionResponse);
    rpc ListExtensions(ListExtensionsRequest) returns (ListExtensionsResponse);
    rpc UpdateExtension(UpdateExtensionRequest) returns (UpdateExtensionResponse);
    rpc DeployExtension(DeployExtensionRequest) returns (DeployExtensionResponse);
    rpc DeactivateExtension(DeactivateExtensionRequest) returns (DeactivateExtensionResponse);
    rpc GetExtensionDependencies(GetExtensionDependenciesRequest) returns (GetExtensionDependenciesResponse);

    // Custom Objects
    rpc CreateCustomObject(CreateCustomObjectRequest) returns (CreateCustomObjectResponse);
    rpc GetCustomObject(GetCustomObjectRequest) returns (GetCustomObjectResponse);
    rpc ListCustomObjects(ListCustomObjectsRequest) returns (ListCustomObjectsResponse);
    rpc UpdateCustomObject(UpdateCustomObjectRequest) returns (UpdateCustomObjectResponse);
    rpc AddFieldToCustomObject(AddFieldToCustomObjectRequest) returns (AddFieldToCustomObjectResponse);

    // Custom Fields
    rpc CreateCustomField(CreateCustomFieldRequest) returns (CreateCustomFieldResponse);
    rpc GetCustomField(GetCustomFieldRequest) returns (GetCustomFieldResponse);
    rpc ListCustomFields(ListCustomFieldsRequest) returns (ListCustomFieldsResponse);
    rpc UpdateCustomField(UpdateCustomFieldRequest) returns (UpdateCustomFieldResponse);
    rpc ValidateCustomField(ValidateCustomFieldRequest) returns (ValidateCustomFieldResponse);

    // Page Layouts
    rpc CreatePageLayout(CreatePageLayoutRequest) returns (CreatePageLayoutResponse);
    rpc GetPageLayout(GetPageLayoutRequest) returns (GetPageLayoutResponse);
    rpc ListPageLayouts(ListPageLayoutsRequest) returns (ListPageLayoutsResponse);
    rpc UpdatePageLayout(UpdatePageLayoutRequest) returns (UpdatePageLayoutResponse);
    rpc PreviewPageLayout(PreviewPageLayoutRequest) returns (PreviewPageLayoutResponse);

    // Business Logic Scripts
    rpc CreateScript(CreateScriptRequest) returns (CreateScriptResponse);
    rpc GetScript(GetScriptRequest) returns (GetScriptResponse);
    rpc ListScripts(ListScriptsRequest) returns (ListScriptsResponse);
    rpc UpdateScript(UpdateScriptRequest) returns (UpdateScriptResponse);
    rpc TestScript(TestScriptRequest) returns (TestScriptResponse);
    rpc ExecuteScript(ExecuteScriptRequest) returns (ExecuteScriptResponse);

    // Sandboxes
    rpc CreateSandbox(CreateSandboxRequest) returns (CreateSandboxResponse);
    rpc GetSandbox(GetSandboxRequest) returns (GetSandboxResponse);
    rpc ListSandboxes(ListSandboxesRequest) returns (ListSandboxesResponse);
    rpc PublishSandbox(PublishSandboxRequest) returns (PublishSandboxResponse);
    rpc DiscardSandbox(DiscardSandboxRequest) returns (DiscardSandboxResponse);
    rpc CloneSandbox(CloneSandboxRequest) returns (CloneSandboxResponse);

    // Dependencies
    rpc GetDependencyTree(GetDependencyTreeRequest) returns (GetDependencyTreeResponse);
    rpc GetImpactAnalysis(GetImpactAnalysisRequest) returns (GetImpactAnalysisResponse);

    // Audit
    rpc GetAuditLog(GetAuditLogRequest) returns (GetAuditLogResponse);
}

message Extension {
    string id = 1;
    string tenant_id = 2;
    string extension_name = 3;
    string extension_type = 4;
    string target_application = 5;
    string target_object = 6;
    string description = 7;
    string version = 8;
    string status = 9;
    string sandbox_id = 10;
    string deployed_date = 11;
}

message CreateExtensionRequest {
    string tenant_id = 1;
    string extension_name = 2;
    string extension_type = 3;
    string target_application = 4;
    string target_object = 5;
    string description = 6;
    string sandbox_id = 7;
    string created_by = 8;
}

message CreateExtensionResponse {
    Extension extension = 1;
}

message GetExtensionRequest {
    string tenant_id = 1;
    string extension_id = 2;
}

message GetExtensionResponse {
    Extension extension = 1;
}

message ListExtensionsRequest {
    string tenant_id = 1;
    string extension_type = 2;
    string target_application = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListExtensionsResponse {
    repeated Extension extensions = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateExtensionRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string extension_name = 3;
    string description = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateExtensionResponse {
    Extension extension = 1;
}

message DeployExtensionRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string sandbox_id = 3;
    string deployed_by = 4;
}

message DeployExtensionResponse {
    Extension extension = 1;
}

message DeactivateExtensionRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string deactivated_by = 3;
}

message DeactivateExtensionResponse {
    Extension extension = 1;
}

message GetExtensionDependenciesRequest {
    string tenant_id = 1;
    string extension_id = 2;
}

message GetExtensionDependenciesResponse {
    repeated ExtensionDependency dependencies = 1;
}

message ExtensionDependency {
    string id = 1;
    string extension_id = 2;
    string depends_on_extension_id = 3;
    string depends_on_extension_name = 4;
    string dependency_type = 5;
}

message CustomObject {
    string id = 1;
    string tenant_id = 2;
    string extension_id = 3;
    string object_name = 4;
    string plural_name = 5;
    string description = 6;
    bool is_activity_enabled = 7;
    bool is_searchable = 8;
    string record_name_field = 9;
    string api_name = 10;
    bool is_system = 11;
}

message CreateCustomObjectRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string object_name = 3;
    string plural_name = 4;
    string description = 5;
    bool is_activity_enabled = 6;
    bool is_searchable = 7;
    string record_name_field = 8;
    string icon_reference = 9;
    string created_by = 10;
}

message CreateCustomObjectResponse {
    CustomObject custom_object = 1;
}

message GetCustomObjectRequest {
    string tenant_id = 1;
    string object_id = 2;
}

message GetCustomObjectResponse {
    CustomObject custom_object = 1;
}

message ListCustomObjectsRequest {
    string tenant_id = 1;
    string extension_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCustomObjectsResponse {
    repeated CustomObject objects = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateCustomObjectRequest {
    string tenant_id = 1;
    string object_id = 2;
    string object_name = 3;
    string description = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateCustomObjectResponse {
    CustomObject custom_object = 1;
}

message AddFieldToCustomObjectRequest {
    string tenant_id = 1;
    string object_id = 2;
    string field_name = 3;
    string display_label = 4;
    string field_type = 5;
    string default_value = 6;
    bool is_required = 7;
    string created_by = 8;
}

message AddFieldToCustomObjectResponse {
    string field_id = 1;
    string field_name = 2;
}

message CustomField {
    string id = 1;
    string tenant_id = 2;
    string extension_id = 3;
    string object_id = 4;
    string field_name = 5;
    string display_label = 6;
    string field_type = 7;
    string default_value = 8;
    bool is_required = 9;
    bool is_unique = 10;
    bool is_indexed = 11;
    int32 display_order = 12;
    string formula_definition = 13;
}

message CreateCustomFieldRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string object_id = 3;
    string field_name = 4;
    string display_label = 5;
    string field_type = 6;
    string default_value = 7;
    bool is_required = 8;
    bool is_unique = 9;
    bool is_indexed = 10;
    int32 display_order = 11;
    string validation_rules = 12;
    string formula_definition = 13;
    string lookup_object_id = 14;
    string picklist_values = 15;
    string created_by = 16;
}

message CreateCustomFieldResponse {
    CustomField field = 1;
}

message GetCustomFieldRequest {
    string tenant_id = 1;
    string field_id = 2;
}

message GetCustomFieldResponse {
    CustomField field = 1;
}

message ListCustomFieldsRequest {
    string tenant_id = 1;
    string object_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCustomFieldsResponse {
    repeated CustomField fields = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateCustomFieldRequest {
    string tenant_id = 1;
    string field_id = 2;
    string display_label = 3;
    string default_value = 4;
    bool is_required = 5;
    string validation_rules = 6;
    string updated_by = 7;
    int32 version = 8;
}

message UpdateCustomFieldResponse {
    CustomField field = 1;
}

message ValidateCustomFieldRequest {
    string tenant_id = 1;
    string field_id = 2;
    string test_value = 3;
}

message ValidateCustomFieldResponse {
    bool is_valid = 1;
    repeated string validation_errors = 2;
}

message PageLayout {
    string id = 1;
    string tenant_id = 2;
    string extension_id = 3;
    string object_id = 4;
    string layout_name = 5;
    string layout_type = 6;
    string target_role_id = 7;
    string sections = 8;
    string fields_config = 9;
    bool is_default = 10;
}

message CreatePageLayoutRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string object_id = 3;
    string layout_name = 4;
    string layout_type = 5;
    string target_role_id = 6;
    string sections = 7;
    string fields_config = 8;
    string related_lists = 9;
    string actions_config = 10;
    bool is_default = 11;
    string created_by = 12;
}

message CreatePageLayoutResponse {
    PageLayout layout = 1;
}

message GetPageLayoutRequest {
    string tenant_id = 1;
    string layout_id = 2;
}

message GetPageLayoutResponse {
    PageLayout layout = 1;
}

message ListPageLayoutsRequest {
    string tenant_id = 1;
    string object_id = 2;
    string layout_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPageLayoutsResponse {
    repeated PageLayout layouts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdatePageLayoutRequest {
    string tenant_id = 1;
    string layout_id = 2;
    string layout_name = 3;
    string sections = 4;
    string fields_config = 5;
    string updated_by = 6;
    int32 version = 7;
}

message UpdatePageLayoutResponse {
    PageLayout layout = 1;
}

message PreviewPageLayoutRequest {
    string tenant_id = 1;
    string layout_id = 2;
}

message PreviewPageLayoutResponse {
    string rendered_layout = 1;  -- JSON representation
}

message BusinessLogicScript {
    string id = 1;
    string tenant_id = 2;
    string extension_id = 3;
    string object_id = 4;
    string script_name = 5;
    string trigger_type = 6;
    string trigger_fields = 7;
    string script_body = 8;
    string language = 9;
    int32 execution_order = 10;
    bool is_active = 11;
}

message CreateScriptRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string object_id = 3;
    string script_name = 4;
    string trigger_type = 5;
    string trigger_fields = 6;
    string script_body = 7;
    string language = 8;
    string error_handling_config = 9;
    int32 execution_order = 10;
    string created_by = 11;
}

message CreateScriptResponse {
    BusinessLogicScript script = 1;
}

message GetScriptRequest {
    string tenant_id = 1;
    string script_id = 2;
}

message GetScriptResponse {
    BusinessLogicScript script = 1;
}

message ListScriptsRequest {
    string tenant_id = 1;
    string object_id = 2;
    string trigger_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListScriptsResponse {
    repeated BusinessLogicScript scripts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateScriptRequest {
    string tenant_id = 1;
    string script_id = 2;
    string script_body = 3;
    string updated_by = 4;
    int32 version = 5;
}

message UpdateScriptResponse {
    BusinessLogicScript script = 1;
}

message TestScriptRequest {
    string tenant_id = 1;
    string script_id = 2;
    string test_input = 3;  -- JSON
}

message TestScriptResponse {
    bool success = 1;
    string output = 2;
    repeated string errors = 3;
    int64 execution_time_ms = 4;
}

message ExecuteScriptRequest {
    string tenant_id = 1;
    string script_id = 2;
    string context_data = 3;  -- JSON
}

message ExecuteScriptResponse {
    bool success = 1;
    string result = 2;
    repeated string errors = 3;
}

message Sandbox {
    string id = 1;
    string tenant_id = 2;
    string sandbox_name = 3;
    string description = 4;
    string source_sandbox_id = 5;
    string status = 6;
    string created_by = 7;
    string published_by = 8;
    string published_date = 9;
}

message CreateSandboxRequest {
    string tenant_id = 1;
    string sandbox_name = 2;
    string description = 3;
    string source_sandbox_id = 4;
    string created_by = 5;
}

message CreateSandboxResponse {
    Sandbox sandbox = 1;
}

message GetSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
}

message GetSandboxResponse {
    Sandbox sandbox = 1;
}

message ListSandboxesRequest {
    string tenant_id = 1;
    string status = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListSandboxesResponse {
    repeated Sandbox sandboxes = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string published_by = 3;
}

message PublishSandboxResponse {
    Sandbox sandbox = 1;
}

message DiscardSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string discarded_by = 3;
}

message DiscardSandboxResponse {
    bool success = 1;
}

message CloneSandboxRequest {
    string tenant_id = 1;
    string sandbox_id = 2;
    string new_sandbox_name = 3;
    string cloned_by = 4;
}

message CloneSandboxResponse {
    Sandbox sandbox = 1;
}

message DependencyNode {
    string extension_id = 1;
    string extension_name = 2;
    string dependency_type = 3;
    repeated DependencyNode children = 4;
}

message GetDependencyTreeRequest {
    string tenant_id = 1;
    string extension_id = 2;
}

message GetDependencyTreeResponse {
    DependencyTree tree = 1;
}

message DependencyTree {
    string root_extension_id = 1;
    string root_extension_name = 2;
    repeated DependencyNode dependencies = 3;
}

message ImpactEntry {
    string extension_id = 1;
    string extension_name = 2;
    string extension_type = 3;
    string impact_type = 4;
}

message GetImpactAnalysisRequest {
    string tenant_id = 1;
    string extension_id = 2;
}

message GetImpactAnalysisResponse {
    repeated ImpactEntry impacts = 1;
    int32 total_affected = 2;
}

message AuditLogEntry {
    string id = 1;
    string tenant_id = 2;
    string extension_id = 3;
    string action = 4;
    string performed_by = 5;
    string details = 6;
    string created_at = 7;
}

message GetAuditLogRequest {
    string tenant_id = 1;
    string extension_id = 2;
    string action = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetAuditLogResponse {
    repeated AuditLogEntry entries = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `user-service` | User roles, permissions, role hierarchy | Page layout role targeting and access control |
| `workflow-service` | Workflow definitions, trigger conditions | Extension workflow integration |
| `auth-service` | Tenant configuration, feature flags | Extension capability validation |
| `document-service` | File attachments for custom fields | File and rich text field storage |
| `reporting-service` | Report template definitions | Extension report customization |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `frontend-service` | Page layouts, custom field definitions | Dynamic UI rendering |
| `workflow-service` | Business logic triggers, event handlers | Custom workflow execution |
| `search-service` | Custom object search indexes | Searchable custom object data |
| `integration-service` | Custom object API definitions | External integration endpoints |
| `audit-service` | Extension lifecycle events | Governance and compliance |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ExtensionCreated` | `composer.extension.created` | `{ tenant_id, extension_id, extension_name, extension_type, target_application, created_by }` | Published when a new extension is created |
| `ExtensionDeployed` | `composer.extension.deployed` | `{ tenant_id, extension_id, extension_name, deployed_by, sandbox_id, deployed_date }` | Published when an extension is deployed to production |
| `SandboxPublished` | `composer.sandbox.published` | `{ tenant_id, sandbox_id, sandbox_name, published_by, extension_count }` | Published when a sandbox is published |
| `CustomObjectCreated` | `composer.custom-object.created` | `{ tenant_id, object_id, object_name, api_name, extension_id }` | Published when a new custom object is created |
| `BusinessLogicExecuted` | `composer.script.executed` | `{ tenant_id, script_id, script_name, object_id, trigger_type, success, execution_time_ms }` | Published when a business logic script is executed |
