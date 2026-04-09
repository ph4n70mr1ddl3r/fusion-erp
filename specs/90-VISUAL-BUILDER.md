# 90 - Visual Builder Service Specification

## 1. Domain Overview

Visual Builder provides a low-code application builder for creating enterprise applications with drag-and-drop page design, configurable components, data bindings to backend services, visual workflow definitions, custom form builders, theme management, granular application permissions, versioned publishing with rollback, and a reusable component library. Enables business users and developers to rapidly build internal tools, dashboards, data entry forms, and workflow-driven applications without writing code. Integrates with the Frontend for rendering, all backend services for data access, and Workflow for process orchestration.

**Bounded Context:** Low-Code Application Builder
**Service Name:** `builder-service`
**Database:** `data/builder.db`
**HTTP Port:** 8122 | **gRPC Port:** 9122

---

## 2. Database Schema

### 2.1 Builder Applications
```sql
CREATE TABLE builder_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_name TEXT NOT NULL,
    app_code TEXT NOT NULL,
    description TEXT,
    app_type TEXT NOT NULL DEFAULT 'WEB'
        CHECK(app_type IN ('WEB','MOBILE','HYBRID','PORTAL','DASHBOARD')),
    category TEXT NOT NULL DEFAULT 'INTERNAL'
        CHECK(category IN ('INTERNAL','PARTNER','CUSTOMER','ADMIN','DASHBOARD')),
    icon TEXT,
    theme_id TEXT,
    default_layout TEXT NOT NULL DEFAULT 'RESPONSIVE'
        CHECK(default_layout IN ('RESPONSIVE','FIXED','FLUID','SIDEBAR','FULLSCREEN')),
    home_page_id TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','TESTING','PUBLISHED','DEPRECATED','ARCHIVED')),
    published_version INTEGER NOT NULL DEFAULT 0,
    current_version INTEGER NOT NULL DEFAULT 1,
    tags TEXT,                                       -- JSON array of tags
    metadata TEXT,                                   -- JSON: extensible app metadata

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, app_code)
);

CREATE INDEX idx_apps_tenant_status ON builder_applications(tenant_id, status);
CREATE INDEX idx_apps_tenant_type ON builder_applications(tenant_id, app_type);
CREATE INDEX idx_apps_tenant_category ON builder_applications(tenant_id, category);
CREATE INDEX idx_apps_tenant_active ON builder_applications(tenant_id, is_active);
```

### 2.2 Application Pages
```sql
CREATE TABLE application_pages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_id TEXT NOT NULL,
    page_name TEXT NOT NULL,
    page_code TEXT NOT NULL,
    page_type TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(page_type IN ('STANDARD','DASHBOARD','FORM','LIST','DETAIL','WIZARD','MODAL','TABS')),
    title TEXT,
    description TEXT,
    route_path TEXT NOT NULL,                        -- e.g., /orders/{id}
    parent_page_id TEXT,
    layout_id TEXT,
    is_home_page INTEGER NOT NULL DEFAULT 0,
    requires_auth INTEGER NOT NULL DEFAULT 1,
    required_permissions TEXT,                       -- JSON array of permission codes
    sort_order INTEGER NOT NULL DEFAULT 0,
    seo_title TEXT,
    seo_description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (app_id) REFERENCES builder_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, app_id, page_code)
);

CREATE INDEX idx_pages_tenant_app ON application_pages(tenant_id, app_id);
CREATE INDEX idx_pages_tenant_type ON application_pages(tenant_id, page_type);
CREATE INDEX idx_pages_tenant_parent ON application_pages(tenant_id, parent_page_id);
CREATE INDEX idx_pages_tenant_route ON application_pages(tenant_id, route_path);
CREATE INDEX idx_pages_tenant_active ON application_pages(tenant_id, is_active);
```

### 2.3 Page Components
```sql
CREATE TABLE page_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    page_id TEXT NOT NULL,
    component_type TEXT NOT NULL
        CHECK(component_type IN ('TEXT','HEADING','IMAGE','BUTTON','TABLE','FORM','INPUT','SELECT','CHECKBOX','RADIO','DATE_PICKER','FILE_UPLOAD','CHART','TAB_PANEL','ACCORDION','MODAL','CARD','CONTAINER','GRID','LIST','TREE','IFRAME','CUSTOM')),
    component_name TEXT NOT NULL,
    label TEXT,
    placeholder TEXT,
    config TEXT NOT NULL,                             -- JSON: component-specific configuration
    style_overrides TEXT,                            -- JSON: CSS style overrides
    visibility_condition TEXT,                       -- Expression for conditional visibility
    is_disabled INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    parent_component_id TEXT,
    slot_name TEXT,                                  -- Named slot within parent component

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (page_id) REFERENCES application_pages(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, page_id, component_name)
);

CREATE INDEX idx_components_tenant_page ON page_components(tenant_id, page_id);
CREATE INDEX idx_components_tenant_type ON page_components(tenant_id, component_type);
CREATE INDEX idx_components_tenant_parent ON page_components(tenant_id, parent_component_id);
CREATE INDEX idx_components_tenant_active ON page_components(tenant_id, is_active);
CREATE INDEX idx_components_tenant_order ON page_components(tenant_id, page_id, sort_order);
```

### 2.4 Page Layouts
```sql
CREATE TABLE page_layouts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_id TEXT NOT NULL,
    layout_name TEXT NOT NULL,
    layout_type TEXT NOT NULL DEFAULT 'GRID'
        CHECK(layout_type IN ('GRID','FLEX','ABSOLUTE','SIDEBAR','STACK','CUSTOM')),
    columns INTEGER NOT NULL DEFAULT 12,
    row_gap INTEGER NOT NULL DEFAULT 16,
    column_gap INTEGER NOT NULL DEFAULT 16,
    padding TEXT NOT NULL DEFAULT '16px',
    max_width TEXT,
    breakpoints TEXT NOT NULL,                       -- JSON: { sm, md, lg, xl } pixel values
    regions TEXT,                                    -- JSON: named layout regions with positions
    custom_css TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (app_id) REFERENCES builder_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, app_id, layout_name)
);

CREATE INDEX idx_layouts_tenant_app ON page_layouts(tenant_id, app_id);
CREATE INDEX idx_layouts_tenant_type ON page_layouts(tenant_id, layout_type);
CREATE INDEX idx_layouts_tenant_active ON page_layouts(tenant_id, is_active);
```

### 2.5 Component Data Bindings
```sql
CREATE TABLE component_data_bindings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    component_id TEXT NOT NULL,
    binding_name TEXT NOT NULL,
    binding_type TEXT NOT NULL
        CHECK(binding_type IN ('API_CALL','GRPC_CALL','EVENT_SUBSCRIPTION','STATE_VARIABLE','CONTEXT','STATIC')),
    source_service TEXT NOT NULL,                    -- Target service name
    source_endpoint TEXT NOT NULL,                   -- API path or gRPC method
    http_method TEXT DEFAULT 'GET'
        CHECK(http_method IN ('GET','POST','PUT','DELETE','PATCH')),
    request_mapping TEXT,                            -- JSON: request field mapping from component state
    response_mapping TEXT,                           -- JSON: response field mapping to component props
    error_mapping TEXT,                              -- JSON: error field mapping
    transform_script TEXT,                           -- Optional data transformation script
    cache_ttl_seconds INTEGER NOT NULL DEFAULT 0,
    debounce_ms INTEGER NOT NULL DEFAULT 0,
    trigger_event TEXT NOT NULL DEFAULT 'ON_LOAD'
        CHECK(trigger_event IN ('ON_LOAD','ON_CHANGE','ON_CLICK','ON_SUBMIT','ON_INTERVAL','MANUAL')),
    poll_interval_ms INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (component_id) REFERENCES page_components(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, component_id, binding_name)
);

CREATE INDEX idx_bindings_tenant_component ON component_data_bindings(tenant_id, component_id);
CREATE INDEX idx_bindings_tenant_service ON component_data_bindings(tenant_id, source_service);
CREATE INDEX idx_bindings_tenant_type ON component_data_bindings(tenant_id, binding_type);
```

### 2.6 Application Workflows
```sql
CREATE TABLE application_workflows (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_id TEXT NOT NULL,
    page_id TEXT,
    workflow_name TEXT NOT NULL,
    workflow_type TEXT NOT NULL
        CHECK(workflow_type IN ('NAVIGATION','DATA_SUBMIT','DATA_FETCH','VALIDATION','STATE_CHANGE','NOTIFICATION','CUSTOM')),
    description TEXT,
    trigger_type TEXT NOT NULL DEFAULT 'USER_ACTION'
        CHECK(trigger_type IN ('USER_ACTION','PAGE_LOAD','TIMER','EVENT','WEBHOOK')),
    trigger_config TEXT,                             -- JSON: trigger configuration
    steps TEXT NOT NULL,                             -- JSON: ordered workflow steps
    error_handling TEXT NOT NULL DEFAULT 'SHOW_MESSAGE'
        CHECK(error_handling IN ('SHOW_MESSAGE','SILENT','CUSTOM_HANDLER')),
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (app_id) REFERENCES builder_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, app_id, workflow_name)
);

CREATE INDEX idx_workflows_tenant_app ON application_workflows(tenant_id, app_id);
CREATE INDEX idx_workflows_tenant_type ON application_workflows(tenant_id, workflow_type);
CREATE INDEX idx_workflows_tenant_trigger ON application_workflows(tenant_id, trigger_type);
CREATE INDEX idx_workflows_tenant_active ON application_workflows(tenant_id, is_active);
```

### 2.7 Custom Form Definitions
```sql
CREATE TABLE custom_form_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_id TEXT NOT NULL,
    form_name TEXT NOT NULL,
    form_code TEXT NOT NULL,
    description TEXT,
    target_service TEXT NOT NULL,                    -- Backend service to submit to
    target_endpoint TEXT NOT NULL,                   -- API path for submission
    method TEXT NOT NULL DEFAULT 'POST'
        CHECK(method IN ('POST','PUT','PATCH')),
    fields TEXT NOT NULL,                            -- JSON: ordered field definitions
    validation_rules TEXT,                           -- JSON: field validation rules
    layout_config TEXT,                              -- JSON: form layout configuration
    submit_label TEXT NOT NULL DEFAULT 'Submit',
    cancel_label TEXT NOT NULL DEFAULT 'Cancel',
    success_message TEXT NOT NULL DEFAULT 'Submitted successfully',
    success_action TEXT NOT NULL DEFAULT 'STAY'
        CHECK(success_action IN ('STAY','NAVIGATE','CLOSE','RELOAD')),
    success_redirect_path TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (app_id) REFERENCES builder_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, app_id, form_code)
);

CREATE INDEX idx_forms_tenant_app ON custom_form_definitions(tenant_id, app_id);
CREATE INDEX idx_forms_tenant_active ON custom_form_definitions(tenant_id, is_active);
```

### 2.8 Application Themes
```sql
CREATE TABLE application_themes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    theme_name TEXT NOT NULL,
    theme_code TEXT NOT NULL,
    description TEXT,
    color_primary TEXT NOT NULL DEFAULT '#1976D2',
    color_secondary TEXT NOT NULL DEFAULT '#424242',
    color_accent TEXT NOT NULL DEFAULT '#82B1FF',
    color_background TEXT NOT NULL DEFAULT '#FFFFFF',
    color_surface TEXT NOT NULL DEFAULT '#FFFFFF',
    color_error TEXT NOT NULL DEFAULT '#FF5252',
    color_success TEXT NOT NULL DEFAULT '#4CAF50',
    color_warning TEXT NOT NULL DEFAULT '#FFC107',
    font_family TEXT NOT NULL DEFAULT 'Roboto, sans-serif',
    font_size_base TEXT NOT NULL DEFAULT '14px',
    border_radius TEXT NOT NULL DEFAULT '4px',
    spacing_unit TEXT NOT NULL DEFAULT '8px',
    custom_css TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, theme_code)
);

CREATE INDEX idx_themes_tenant_active ON application_themes(tenant_id, is_active);
CREATE INDEX idx_themes_tenant_default ON application_themes(tenant_id, is_default);
```

### 2.9 Application Permissions
```sql
CREATE TABLE application_permissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_id TEXT NOT NULL,
    permission_code TEXT NOT NULL,
    permission_name TEXT NOT NULL,
    description TEXT,
    permission_type TEXT NOT NULL DEFAULT 'FEATURE'
        CHECK(permission_type IN ('VIEW','EDIT','ADMIN','FEATURE','DATA_ACCESS','CUSTOM')),
    assigned_roles TEXT NOT NULL,                    -- JSON array of role IDs
    assigned_users TEXT,                             -- JSON array of user IDs
    page_ids TEXT,                                   -- JSON array of accessible page IDs
    component_ids TEXT,                              -- JSON array of accessible component IDs
    data_filters TEXT,                               -- JSON: row-level data access filters
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (app_id) REFERENCES builder_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, app_id, permission_code)
);

CREATE INDEX idx_permissions_tenant_app ON application_permissions(tenant_id, app_id);
CREATE INDEX idx_permissions_tenant_type ON application_permissions(tenant_id, permission_type);
CREATE INDEX idx_permissions_tenant_active ON application_permissions(tenant_id, is_active);
```

### 2.10 Published Applications
```sql
CREATE TABLE published_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    app_id TEXT NOT NULL,
    version_number INTEGER NOT NULL,
    publish_notes TEXT,
    published_config TEXT NOT NULL,                  -- JSON: complete app snapshot at publish time
    pages_snapshot TEXT NOT NULL,                    -- JSON: all pages and components
    workflows_snapshot TEXT NOT NULL,                -- JSON: all workflows
    forms_snapshot TEXT NOT NULL,                    -- JSON: all form definitions
    theme_snapshot TEXT NOT NULL,                    -- JSON: theme configuration
    permissions_snapshot TEXT NOT NULL,              -- JSON: permission configuration
    bundle_url TEXT,                                 -- CDN URL for built assets
    bundle_hash TEXT NOT NULL,                       -- Content hash for cache invalidation
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','SUPERSEDED','ROLLED_BACK','DEPRECATED')),
    published_by TEXT NOT NULL,
    published_at TEXT NOT NULL,
    superseded_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (app_id) REFERENCES builder_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, app_id, version_number)
);

CREATE INDEX idx_published_tenant_app ON published_applications(tenant_id, app_id);
CREATE INDEX idx_published_tenant_status ON published_applications(tenant_id, status);
CREATE INDEX idx_published_tenant_version ON published_applications(tenant_id, app_id, version_number);
```

### 2.11 Component Library
```sql
CREATE TABLE component_library (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    component_name TEXT NOT NULL,
    component_code TEXT NOT NULL,
    component_type TEXT NOT NULL
        CHECK(component_type IN ('LAYOUT','DATA_DISPLAY','DATA_INPUT','NAVIGATION','FEEDBACK','CHART','CUSTOM')),
    description TEXT,
    icon TEXT,
    category TEXT NOT NULL,
    tags TEXT,                                       -- JSON array of searchable tags
    config_schema TEXT NOT NULL,                     -- JSON: configurable properties schema
    default_config TEXT NOT NULL,                    -- JSON: default property values
    props_definition TEXT NOT NULL,                  -- JSON: component props/inputs
    events_definition TEXT,                          -- JSON: events emitted by component
    slots_definition TEXT,                           -- JSON: available child slots
    template TEXT NOT NULL,                          -- Component template (HTML/JSX)
    styles TEXT,                                     -- Component styles
    min_app_version INTEGER NOT NULL DEFAULT 1,
    is_builtin INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, component_code)
);

CREATE INDEX idx_library_tenant_type ON component_library(tenant_id, component_type);
CREATE INDEX idx_library_tenant_category ON component_library(tenant_id, category);
CREATE INDEX idx_library_tenant_active ON component_library(tenant_id, is_active);
CREATE INDEX idx_library_tenant_builtin ON component_library(tenant_id, is_builtin);
```

---

## 3. REST API Endpoints

```
# Applications
GET/POST      /api/v1/builder/applications                   Permission: builder.apps.read/create
GET/PUT       /api/v1/builder/applications/{id}              Permission: builder.apps.read/update
DELETE        /api/v1/builder/applications/{id}              Permission: builder.apps.delete
GET           /api/v1/builder/applications/{id}/preview      Permission: builder.apps.read
POST          /api/v1/builder/applications/{id}/duplicate    Permission: builder.apps.create

# Pages
GET/POST      /api/v1/builder/applications/{id}/pages        Permission: builder.pages.read/create
GET/PUT       /api/v1/builder/pages/{id}                     Permission: builder.pages.read/update
DELETE        /api/v1/builder/pages/{id}                     Permission: builder.pages.delete
GET           /api/v1/builder/applications/{id}/pages/tree   Permission: builder.pages.read
PATCH         /api/v1/builder/pages/{id}/reorder             Permission: builder.pages.update

# Components
GET/POST      /api/v1/builder/pages/{id}/components          Permission: builder.components.read/create
GET/PUT       /api/v1/builder/components/{id}                Permission: builder.components.read/update
DELETE        /api/v1/builder/components/{id}                Permission: builder.components.delete
PATCH         /api/v1/builder/components/{id}/move           Permission: builder.components.update
POST          /api/v1/builder/components/bulk-update         Permission: builder.components.update

# Layouts
GET/POST      /api/v1/builder/applications/{id}/layouts      Permission: builder.layouts.read/create
GET/PUT       /api/v1/builder/layouts/{id}                   Permission: builder.layouts.read/update
DELETE        /api/v1/builder/layouts/{id}                   Permission: builder.layouts.delete

# Data Bindings
GET/POST      /api/v1/builder/components/{id}/bindings       Permission: builder.bindings.read/create
GET/PUT       /api/v1/builder/bindings/{id}                  Permission: builder.bindings.read/update
DELETE        /api/v1/builder/bindings/{id}                  Permission: builder.bindings.delete
POST          /api/v1/builder/bindings/{id}/test             Permission: builder.bindings.test

# Workflows
GET/POST      /api/v1/builder/applications/{id}/workflows    Permission: builder.workflows.read/create
GET/PUT       /api/v1/builder/workflows/{id}                 Permission: builder.workflows.read/update
DELETE        /api/v1/builder/workflows/{id}                 Permission: builder.workflows.delete
POST          /api/v1/builder/workflows/{id}/test            Permission: builder.workflows.test

# Form Definitions
GET/POST      /api/v1/builder/applications/{id}/forms        Permission: builder.forms.read/create
GET/PUT       /api/v1/builder/forms/{id}                     Permission: builder.forms.read/update
DELETE        /api/v1/builder/forms/{id}                     Permission: builder.forms.delete
POST          /api/v1/builder/forms/{id}/validate            Permission: builder.forms.test

# Themes
GET/POST      /api/v1/builder/themes                         Permission: builder.themes.read/create
GET/PUT       /api/v1/builder/themes/{id}                    Permission: builder.themes.read/update
DELETE        /api/v1/builder/themes/{id}                    Permission: builder.themes.delete
PATCH         /api/v1/builder/themes/{id}/default            Permission: builder.themes.update

# Permissions
GET/POST      /api/v1/builder/applications/{id}/permissions  Permission: builder.permissions.read/create
GET/PUT       /api/v1/builder/permissions/{id}               Permission: builder.permissions.read/update
DELETE        /api/v1/builder/permissions/{id}               Permission: builder.permissions.delete

# Publishing
POST          /api/v1/builder/applications/{id}/publish      Permission: builder.publish.execute
GET           /api/v1/builder/applications/{id}/versions     Permission: builder.publish.read
POST          /api/v1/builder/applications/{id}/rollback     Permission: builder.publish.execute
GET           /api/v1/builder/applications/{id}/versions/{v} Permission: builder.publish.read

# Component Library
GET/POST      /api/v1/builder/library                        Permission: builder.library.read/create
GET/PUT       /api/v1/builder/library/{id}                   Permission: builder.library.read/update
DELETE        /api/v1/builder/library/{id}                   Permission: builder.library.delete
GET           /api/v1/builder/library/search                 Permission: builder.library.read
```

---

## 4. Business Rules

### 4.1 Application Management
1. Each application MUST have a unique code within a tenant
2. Applications in DRAFT status MAY be modified freely; PUBLISHED applications MUST be edited in a new version
3. Application deletion MUST soft-delete and archive; hard deletion requires explicit confirmation
4. Every application MUST have at least one designated home page
5. Application type determines the rendering framework and available component types
6. Tags on applications SHOULD be used for searchability and categorization

### 4.2 Page Design
1. Pages MUST have a unique route_path within an application
2. Pages marked as is_home_page MUST be set on exactly one page per application
3. Parent-child page relationships MUST NOT form cycles
4. Pages requiring authentication MUST enforce login before rendering
5. Page components MUST be rendered in sort_order within their parent container
6. Page type determines the default set of components and layout structure

### 4.3 Component Configuration
1. Components MUST reference a valid component_type from the component library
2. Nested components MUST use parent_component_id and slot_name for placement
3. Visibility conditions MUST be evaluated at render time using the page state
4. Component configuration MUST conform to the config_schema of its component type
5. Custom components from the library MUST be versioned; breaking changes create a new version
6. Style overrides MUST be scoped to the component instance, not applied globally

### 4.4 Data Binding
1. Each data binding MUST reference a valid backend service and endpoint
2. Request mapping MUST map component state fields to API request parameters
3. Response mapping MUST map API response fields to component properties
4. Bindings with cache_ttl_seconds greater than zero MUST cache responses per tenant
5. Debounced bindings MUST wait for the specified interval before triggering
6. Polling bindings (poll_interval_ms > 0) MUST be cancellable on component unmount
7. Error mappings SHOULD provide user-friendly messages for common error codes

### 4.5 Workflow Configuration
1. Workflow steps MUST execute in the defined order within the steps JSON array
2. Navigation workflows MUST validate the target page exists before redirecting
3. Data submit workflows MUST validate form data before calling the target endpoint
4. Custom workflows MAY invoke multiple service endpoints in sequence
5. Workflow error handling MUST follow the configured strategy (show message, silent, custom handler)
6. Workflows triggered by page load MUST complete before the page is considered fully rendered

### 4.6 Form Definitions
1. Form fields MUST define a type, name, and label at minimum
2. Validation rules MUST be evaluated before form submission
3. Fields with required validation MUST prevent submission until satisfied
4. Form submission MUST include CSRF protection for security
5. Success actions MUST be configurable (stay on page, navigate, close, or reload)
6. Form definitions SHOULD support conditional field visibility based on other field values

### 4.7 Theme Management
1. Exactly one theme per tenant MUST be designated as the default theme
2. Color values MUST be valid CSS color strings (hex, rgb, or named colors)
3. Custom CSS in themes MUST be namespaced to avoid conflicts with component styles
4. Font families MUST specify fallbacks for cross-platform compatibility
5. Theme changes to a PUBLISHED application MUST create a new version
6. Builtin themes MUST NOT be deleted; they MAY be overridden by custom themes

### 4.8 Publishing and Versioning
1. Publishing MUST create a complete snapshot of the application configuration
2. Published versions are immutable and MUST NOT be modified after publication
3. Rollback MUST restore the application to a previous published version
4. Each publish increments the version_number monotonically
5. The bundle_hash MUST change with each publish to invalidate CDN caches
6. Superseded versions MUST be retained for rollback capability

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.builder.v1;

service BuilderService {
    rpc GetApplication(GetApplicationRequest) returns (GetApplicationResponse);
    rpc GetPage(GetPageRequest) returns (GetPageResponse);
    rpc RenderPage(RenderPageRequest) returns (RenderPageResponse);
    rpc ResolveBindings(ResolveBindingsRequest) returns (ResolveBindingsResponse);
    rpc ExecuteWorkflow(ExecuteWorkflowRequest) returns (ExecuteWorkflowResponse);
    rpc PublishApplication(PublishApplicationRequest) returns (PublishApplicationResponse);
    rpc GetComponentLibrary(GetComponentLibraryRequest) returns (GetComponentLibraryResponse);
}

// --- Builder Application ---
message BuilderApplication {
    string id = 1; string tenant_id = 2; string app_name = 3; string app_code = 4;
    string description = 5; string app_type = 6; string category = 7; string icon = 8;
    string theme_id = 9; string default_layout = 10; string home_page_id = 11;
    string status = 12; int32 published_version = 13; int32 current_version = 14;
    string tags = 15; string metadata = 16;
    string created_at = 17; string updated_at = 18; string created_by = 19; string updated_by = 20;
    int32 version = 21; bool is_active = 22;
}

// --- Application Page ---
message ApplicationPage {
    string id = 1; string tenant_id = 2; string app_id = 3; string page_name = 4;
    string page_code = 5; string page_type = 6; string title = 7; string description = 8;
    string route_path = 9; string parent_page_id = 10; string layout_id = 11;
    bool is_home_page = 12; bool requires_auth = 13; string required_permissions = 14;
    int32 sort_order = 15; string seo_title = 16; string seo_description = 17;
    string created_at = 18; string updated_at = 19; string created_by = 20; string updated_by = 21;
    int32 version = 22; bool is_active = 23;
}

// --- Page Component ---
message PageComponent {
    string id = 1; string tenant_id = 2; string page_id = 3; string component_type = 4;
    string component_name = 5; string label = 6; string placeholder = 7; string config = 8;
    string style_overrides = 9; string visibility_condition = 10; bool is_disabled = 11;
    int32 sort_order = 12; string parent_component_id = 13; string slot_name = 14;
    string created_at = 15; string updated_at = 16; string created_by = 17; string updated_by = 18;
    int32 version = 19; bool is_active = 20;
}

// --- Page Layout ---
message PageLayout {
    string id = 1; string tenant_id = 2; string app_id = 3; string layout_name = 4;
    string layout_type = 5; int32 columns = 6; int32 row_gap = 7; int32 column_gap = 8;
    string padding = 9; string max_width = 10; string breakpoints = 11; string regions = 12;
    string custom_css = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18; bool is_active = 19;
}

// --- Component Data Binding ---
message ComponentDataBinding {
    string id = 1; string tenant_id = 2; string component_id = 3; string binding_name = 4;
    string binding_type = 5; string source_service = 6; string source_endpoint = 7;
    string http_method = 8; string request_mapping = 9; string response_mapping = 10;
    string error_mapping = 11; string transform_script = 12; int32 cache_ttl_seconds = 13;
    int32 debounce_ms = 14; string trigger_event = 15; int32 poll_interval_ms = 16;
    string created_at = 17; string updated_at = 18; string created_by = 19; string updated_by = 20;
    int32 version = 21;
}

// --- Application Workflow ---
message ApplicationWorkflow {
    string id = 1; string tenant_id = 2; string app_id = 3; string page_id = 4;
    string workflow_name = 5; string workflow_type = 6; string description = 7;
    string trigger_type = 8; string trigger_config = 9; string steps = 10;
    string error_handling = 11; bool is_active = 12;
    string created_at = 13; string updated_at = 14; string created_by = 15; string updated_by = 16;
    int32 version = 17;
}

// --- Custom Form Definition ---
message CustomFormDefinition {
    string id = 1; string tenant_id = 2; string app_id = 3; string form_name = 4;
    string form_code = 5; string description = 6; string target_service = 7;
    string target_endpoint = 8; string method = 9; string fields = 10;
    string validation_rules = 11; string layout_config = 12; string submit_label = 13;
    string cancel_label = 14; string success_message = 15; string success_action = 16;
    string success_redirect_path = 17; bool is_active = 18;
    string created_at = 19; string updated_at = 20; string created_by = 21; string updated_by = 22;
    int32 version = 23;
}

// --- Application Theme ---
message ApplicationTheme {
    string id = 1; string tenant_id = 2; string theme_name = 3; string theme_code = 4;
    string description = 5; string color_primary = 6; string color_secondary = 7;
    string color_accent = 8; string color_background = 9; string color_surface = 10;
    string color_error = 11; string color_success = 12; string color_warning = 13;
    string font_family = 14; string font_size_base = 15; string border_radius = 16;
    string spacing_unit = 17; string custom_css = 18; bool is_default = 19; bool is_active = 20;
    string created_at = 21; string updated_at = 22; string created_by = 23; string updated_by = 24;
    int32 version = 25;
}

// --- Application Permission ---
message ApplicationPermission {
    string id = 1; string tenant_id = 2; string app_id = 3; string permission_code = 4;
    string permission_name = 5; string description = 6; string permission_type = 7;
    string assigned_roles = 8; string assigned_users = 9; string page_ids = 10;
    string component_ids = 11; string data_filters = 12; bool is_active = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- Published Application ---
message PublishedApplication {
    string id = 1; string tenant_id = 2; string app_id = 3; int32 version_number = 4;
    string publish_notes = 5; string published_config = 6; string pages_snapshot = 7;
    string workflows_snapshot = 8; string forms_snapshot = 9; string theme_snapshot = 10;
    string permissions_snapshot = 11; string bundle_url = 12; string bundle_hash = 13;
    string status = 14; string published_by = 15; string published_at = 16; string superseded_at = 17;
    string created_at = 18; string updated_at = 19; string created_by = 20; string updated_by = 21;
}

// --- Component Library Item ---
message ComponentLibraryItem {
    string id = 1; string tenant_id = 2; string component_name = 3; string component_code = 4;
    string component_type = 5; string description = 6; string icon = 7; string category = 8;
    string tags = 9; string config_schema = 10; string default_config = 11;
    string props_definition = 12; string events_definition = 13; string slots_definition = 14;
    string template = 15; string styles = 16; int32 min_app_version = 17;
    bool is_builtin = 18; bool is_active = 19;
    string created_at = 20; string updated_at = 21; string created_by = 22; string updated_by = 23;
    int32 version = 24;
}

// --- RPC Request/Response Messages ---
message GetApplicationRequest { string tenant_id = 1; string id = 2; }
message GetApplicationResponse { BuilderApplication data = 1; }

message GetPageRequest { string tenant_id = 1; string id = 2; }
message GetPageResponse { ApplicationPage data = 1; repeated PageComponent components = 2; }

message RenderPageRequest { string tenant_id = 1; string page_id = 2; }
message RenderPageResponse { ApplicationPage page = 1; repeated PageComponent components = 2; PageLayout layout = 3; }

message ResolveBindingsRequest { string tenant_id = 1; string page_id = 2; }
message ResolveBindingsResponse { repeated ComponentDataBinding bindings = 1; }

message ExecuteWorkflowRequest { string tenant_id = 1; string workflow_id = 2; string input_data = 3; }
message ExecuteWorkflowResponse { string output_data = 1; string status = 2; }

message PublishApplicationRequest { string tenant_id = 1; string app_id = 2; string publish_notes = 3; }
message PublishApplicationResponse { PublishedApplication published = 1; }

message GetComponentLibraryRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; string component_type = 4; }
message GetComponentLibraryResponse { repeated ComponentLibraryItem items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListApplicationsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListApplicationsResponse { repeated BuilderApplication items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListPagesRequest { string tenant_id = 1; string app_id = 2; int32 page_size = 3; string page_token = 4; }
message ListPagesResponse { repeated ApplicationPage items = 1; int32 total_count = 2; string next_page_token = 3; }
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Auth:** User identity, role assignments, permission checks for application access
- **Workflow:** Workflow definitions, approval processes, task management integration
- **Frontend:** Rendering engine, component registry, client-side routing
- **All Services:** API contracts, endpoint definitions, data schemas for data binding configuration

### 6.2 Data Published To
- **Frontend:** Application bundles, page configurations, component definitions for rendering
- **Workflow:** Workflow execution requests, process initiation from application interactions
- **Auth:** Permission definitions, role-to-permission mappings for access control
- **Notification:** Publish notifications, form submission alerts, workflow status updates
- **All Services:** API calls via data bindings to fetch and submit data

### 6.3 Integration Matrix

| Source Service | Integration Type | Data Exchanged |
|----------------|------------------|----------------|
| Auth | API call | User authentication, permission validation, role resolution |
| Frontend | Direct | Application bundles, page definitions, component configurations |
| Workflow | Event-driven | Workflow trigger requests, step execution results |
| GL | Data binding | Financial data display, journal entry forms |
| AP | Data binding | Invoice entry forms, supplier data display |
| AR | Data binding | Customer data display, receipt entry forms |
| Procurement | Data binding | Purchase order forms, supplier catalog display |
| Inventory | Data binding | Stock level dashboards, movement forms |
| Order-Management | Data binding | Order entry forms, fulfillment tracking |
| Reporting | Data binding | Embedded reports, dashboard charts |
| HCM (Core-HR) | Data binding | Employee data forms, org chart display |
| Notification | Event-driven | Publish alerts, form submission confirmations |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `builder.application.created` | `{ app_id, app_name, app_type, created_by, timestamp }` | New application created |
| `builder.page.added` | `{ app_id, page_id, page_name, page_type, created_by, timestamp }` | New page added to application |
| `builder.component.configured` | `{ page_id, component_id, component_type, created_by, timestamp }` | Component added or updated on a page |
| `builder.binding.created` | `{ component_id, binding_id, source_service, source_endpoint, timestamp }` | Data binding configured for a component |
| `builder.workflow.configured` | `{ app_id, workflow_id, workflow_name, workflow_type, timestamp }` | Application workflow defined or updated |
| `builder.application.published` | `{ app_id, version_number, published_by, bundle_hash, timestamp }` | Application version published |
| `builder.application.rollback` | `{ app_id, from_version, to_version, rolled_back_by, timestamp }` | Application rolled back to previous version |
| `builder.form.submitted` | `{ app_id, form_id, form_code, submitted_by, timestamp }` | Custom form submitted through application |
| `builder.theme.updated` | `{ theme_id, theme_name, updated_by, timestamp }` | Application theme modified |
| `builder.permission.updated` | `{ app_id, permission_id, permission_code, updated_by, timestamp }` | Application permission modified |
| `builder.library.component.added` | `{ component_id, component_code, component_type, created_by, timestamp }` | New component added to library |

---

## 8. Migrations

1. V001: `builder_applications`
2. V002: `application_pages`
3. V003: `page_components`
4. V004: `page_layouts`
5. V005: `component_data_bindings`
6. V006: `application_workflows`
7. V007: `custom_form_definitions`
8. V008: `application_themes`
9. V009: `application_permissions`
10. V010: `published_applications`
11. V011: `component_library`
12. V012: Triggers for `updated_at`
