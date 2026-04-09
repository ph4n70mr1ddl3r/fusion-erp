# 109 - Experience Design Studio Service Specification

## 1. Domain Overview

The Experience Design Studio service provides no-code/low-code tools for customizing and personalizing HCM application pages, creating custom dashboards, modifying layouts, and building role-specific views without development. It supports page variants, conditional display rules, theme customization, and A/B testing of UI configurations, enabling business users to tailor the application experience for their organization.

**Bounded Context:** UI Customization & Experience Design
**Service Name:** `exdesign-service`
**Database:** `data/exdesign.db`
**HTTP Port:** 8146 | **gRPC Port:** 9146

---

## 2. Database Schema

### 2.1 Design Projects
```sql
CREATE TABLE design_projects (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_name TEXT NOT NULL,
    target_application TEXT NOT NULL CHECK(target_application IN ('HCM','ERP','SCM','CX','EPM')),
    description TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','IN_REVIEW','ACTIVE','INACTIVE')),
    published_date TEXT,
    published_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, project_name)
);

CREATE INDEX idx_projects_tenant_status ON design_projects(tenant_id, status);
CREATE INDEX idx_projects_tenant_app ON design_projects(tenant_id, target_application);
```

### 2.2 Page Customizations
```sql
CREATE TABLE page_customizations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    page_key TEXT NOT NULL,
    page_name TEXT NOT NULL,
    base_template TEXT NOT NULL,
    custom_layout_config TEXT NOT NULL,  -- JSON: layout structure
    variant_name TEXT,
    target_roles TEXT,                   -- JSON: array of role IDs
    target_conditions TEXT,              -- JSON: audience conditions
    display_rules TEXT,                  -- JSON: conditional display logic
    priority INTEGER NOT NULL DEFAULT 0,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES design_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_pages_tenant_project ON page_customizations(tenant_id, project_id);
CREATE INDEX idx_pages_tenant_key ON page_customizations(tenant_id, page_key);
CREATE INDEX idx_pages_tenant_roles ON page_customizations(tenant_id, target_roles);
```

### 2.3 Custom Components
```sql
CREATE TABLE custom_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    component_name TEXT NOT NULL,
    component_type TEXT NOT NULL CHECK(component_type IN ('WIDGET','CARD','LIST','CHART','FORM','TAB','ACCORDION','BANNER')),
    data_source_config TEXT,   -- JSON: data binding configuration
    display_config TEXT,       -- JSON: visual presentation settings
    interaction_config TEXT,   -- JSON: click/hover/submit handlers
    refresh_interval_seconds INTEGER,
    cache_enabled INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES design_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_components_tenant_project ON custom_components(tenant_id, project_id);
CREATE INDEX idx_components_tenant_type ON custom_components(tenant_id, component_type);
```

### 2.4 Theme Configurations
```sql
CREATE TABLE theme_configurations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    theme_name TEXT NOT NULL,
    colors TEXT NOT NULL,           -- JSON: primary, secondary, accent, etc.
    fonts TEXT,                     -- JSON: font families and sizes
    spacing TEXT,                   -- JSON: margin/padding presets
    logo_reference TEXT,
    favicon_reference TEXT,
    custom_css TEXT,
    is_active INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES design_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_themes_tenant_project ON theme_configurations(tenant_id, project_id);
CREATE INDEX idx_themes_tenant_active ON theme_configurations(tenant_id, is_active);
```

### 2.5 Conditional Rules
```sql
CREATE TABLE conditional_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('VISIBILITY','ENABLEMENT','STYLING','REDIRECTION')),
    condition_expression TEXT NOT NULL,  -- JSON: boolean condition tree
    action_config TEXT NOT NULL,         -- JSON: action to take when true
    priority INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES design_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_rules_tenant_project ON conditional_rules(tenant_id, project_id);
CREATE INDEX idx_rules_tenant_type ON conditional_rules(tenant_id, rule_type, is_active);
```

### 2.6 A/B Tests
```sql
CREATE TABLE ab_tests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    test_name TEXT NOT NULL,
    test_type TEXT NOT NULL CHECK(test_type IN ('LAYOUT','CONTENT','THEME')),
    variant_a_config TEXT NOT NULL,   -- JSON: control configuration
    variant_b_config TEXT NOT NULL,   -- JSON: variant configuration
    target_audience_pct DECIMAL(5,2) NOT NULL DEFAULT 50.00,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    metric_type TEXT NOT NULL,
    variant_a_conversions INTEGER NOT NULL DEFAULT 0,
    variant_b_conversions INTEGER NOT NULL DEFAULT 0,
    total_participants INTEGER NOT NULL DEFAULT 0,
    winner TEXT CHECK(winner IN ('A','B','INCONCLUSIVE')),
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','RUNNING','COMPLETED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES design_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_abtests_tenant_project ON ab_tests(tenant_id, project_id);
CREATE INDEX idx_abtests_tenant_status ON ab_tests(tenant_id, status);
CREATE INDEX idx_abtests_tenant_dates ON ab_tests(tenant_id, start_date, end_date);
```

### 2.7 Design Sandboxes
```sql
CREATE TABLE design_sandboxes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    sandbox_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'CREATED' CHECK(status IN ('CREATED','TESTING','PUBLISHED','DISCARDED')),
    created_by TEXT NOT NULL,
    published_by TEXT,
    published_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES design_projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_sandboxes_tenant_project ON design_sandboxes(tenant_id, project_id);
CREATE INDEX idx_sandboxes_tenant_status ON design_sandboxes(tenant_id, status);
```

---

## 3. REST API Endpoints

### Projects
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/projects` | Create a design project |
| GET | `/api/v1/projects` | List design projects |
| GET | `/api/v1/projects/{id}` | Get project details |
| PUT | `/api/v1/projects/{id}` | Update project |
| POST | `/api/v1/projects/{id}/publish` | Publish a design project |

### Page Customizations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/page-customizations` | Create a page customization |
| GET | `/api/v1/page-customizations` | List page customizations |
| GET | `/api/v1/page-customizations/{id}` | Get customization details |
| PUT | `/api/v1/page-customizations/{id}` | Update page customization |
| POST | `/api/v1/page-customizations/{id}/preview` | Preview page customization |
| POST | `/api/v1/page-customizations/{id}/duplicate` | Duplicate a page customization |

### Components
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/components` | Create a custom component |
| GET | `/api/v1/components` | List custom components |
| GET | `/api/v1/components/{id}` | Get component details |
| PUT | `/api/v1/components/{id}` | Update component |
| POST | `/api/v1/components/{id}/render-preview` | Render component preview |

### Themes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/themes` | Create a theme configuration |
| GET | `/api/v1/themes` | List theme configurations |
| GET | `/api/v1/themes/{id}` | Get theme details |
| PUT | `/api/v1/themes/{id}` | Update theme |
| POST | `/api/v1/themes/{id}/activate` | Activate a theme |
| POST | `/api/v1/themes/reset-default` | Reset to default theme |

### Conditional Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/conditional-rules` | Create a conditional rule |
| GET | `/api/v1/conditional-rules` | List conditional rules |
| GET | `/api/v1/conditional-rules/{id}` | Get rule details |
| PUT | `/api/v1/conditional-rules/{id}` | Update rule |
| POST | `/api/v1/conditional-rules/{id}/evaluate` | Evaluate a conditional rule |

### A/B Tests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/ab-tests` | Create an A/B test |
| GET | `/api/v1/ab-tests` | List A/B tests |
| GET | `/api/v1/ab-tests/{id}` | Get test details |
| PUT | `/api/v1/ab-tests/{id}` | Update A/B test |
| POST | `/api/v1/ab-tests/{id}/start` | Start an A/B test |
| POST | `/api/v1/ab-tests/{id}/stop` | Stop an A/B test |
| GET | `/api/v1/ab-tests/{id}/results` | Get A/B test results |

### Sandboxes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sandboxes` | Create a sandbox |
| GET | `/api/v1/sandboxes` | List sandboxes |
| GET | `/api/v1/sandboxes/{id}` | Get sandbox details |
| PUT | `/api/v1/sandboxes/{id}` | Update sandbox |
| POST | `/api/v1/sandboxes/{id}/publish` | Publish sandbox to production |
| POST | `/api/v1/sandboxes/{id}/discard` | Discard sandbox changes |

---

## 4. Business Rules

1. Only one theme per project MAY be active at a time; activating a theme MUST deactivate all other themes for the same project.
2. Page customizations MUST be evaluated by priority order, and the first matching rule set MUST determine the displayed variant.
3. A/B test target_audience_pct MUST be between 1.00 and 100.00 and MUST NOT exceed available traffic allocation.
4. Sandboxes MUST be discarded or published before a new sandbox can be created for the same project by the same user.
5. Projects in ACTIVE status MUST NOT be deletable; they MUST be moved to INACTIVE status first.
6. Conditional rules MUST support nested boolean expressions up to 5 levels deep.
7. Custom components with cache_enabled MUST respect the refresh_interval_seconds setting and MUST serve stale data for no longer than the configured interval.
8. A/B tests MUST run for a minimum of 7 days before a winner can be declared to ensure statistical significance.
9. Published design projects MUST create an immutable snapshot of all associated configurations at the time of publication.
10. Theme inheritance MUST follow a cascade from project-level to tenant-level to system-default.
11. Page customizations targeting specific roles MUST validate that referenced role IDs exist before saving.
12. Only one A/B test per page_key per project MAY be in RUNNING status at any time.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package exdesign.v1;

service ExperienceDesignStudioService {
    // Projects
    rpc CreateProject(CreateProjectRequest) returns (CreateProjectResponse);
    rpc GetProject(GetProjectRequest) returns (GetProjectResponse);
    rpc ListProjects(ListProjectsRequest) returns (ListProjectsResponse);
    rpc UpdateProject(UpdateProjectRequest) returns (UpdateProjectResponse);
    rpc PublishProject(PublishProjectRequest) returns (PublishProjectResponse);

    // Page Customizations
    rpc CreatePageCustomization(CreatePageCustomizationRequest) returns (CreatePageCustomizationResponse);
    rpc GetPageCustomization(GetPageCustomizationRequest) returns (GetPageCustomizationResponse);
    rpc ListPageCustomizations(ListPageCustomizationsRequest) returns (ListPageCustomizationsResponse);
    rpc PreviewPageCustomization(PreviewPageCustomizationRequest) returns (PreviewPageCustomizationResponse);
    rpc DuplicatePageCustomization(DuplicatePageCustomizationRequest) returns (DuplicatePageCustomizationResponse);

    // Components
    rpc CreateComponent(CreateComponentRequest) returns (CreateComponentResponse);
    rpc GetComponentGetComponentRequest) returns (GetComponentResponse);
    rpc ListComponents(ListComponentsRequest) returns (ListComponentsResponse);
    rpc RenderComponentPreview(RenderComponentPreviewRequest) returns (RenderComponentPreviewResponse);

    // Themes
    rpc CreateTheme(CreateThemeRequest) returns (CreateThemeResponse);
    rpc ListThemes(ListThemesRequest) returns (ListThemesResponse);
    rpc ActivateTheme(ActivateThemeRequest) returns (ActivateThemeResponse);
    rpc ResetDefaultTheme(ResetDefaultThemeRequest) returns (ResetDefaultThemeResponse);

    // Conditional Rules
    rpc CreateConditionalRule(CreateConditionalRuleRequest) returns (CreateConditionalRuleResponse);
    rpc ListConditionalRules(ListConditionalRulesRequest) returns (ListConditionalRulesResponse);
    rpc EvaluateConditionalRule(EvaluateConditionalRuleRequest) returns (EvaluateConditionalRuleResponse);

    // A/B Tests
    rpc CreateABTest(CreateABTestRequest) returns (CreateABTestResponse);
    rpc GetABTest(GetABTestRequest) returns (GetABTestResponse);
    rpc ListABTests(ListABTestsRequest) returns (ListABTestsResponse);
    rpc StartABTest(StartABTestRequest) returns (StartABTestResponse);
    rpc StopABTest(StopABTestRequest) returns (StopABTestResponse);
    rpc GetABTestResults(GetABTestResultsRequest) returns (GetABTestResultsResponse);

    // Sandboxes
    rpc CreateSandbox(CreateSandboxRequest) returns (CreateSandboxResponse);
    rpc PublishSandbox(PublishSandboxRequest) returns (PublishSandboxResponse);
    rpc DiscardSandbox(DiscardSandboxRequest) returns (DiscardSandboxResponse);
}

message DesignProject {
    string id = 1;
    string tenant_id = 2;
    string project_name = 3;
    string target_application = 4;
    string description = 5;
    string status = 6;
    string published_date = 7;
    string published_by = 8;
}

message CreateProjectRequest {
    string tenant_id = 1;
    string project_name = 2;
    string target_application = 3;
    string description = 4;
    string created_by = 5;
}

message CreateProjectResponse {
    DesignProject project = 1;
}

message GetProjectRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message GetProjectResponse {
    DesignProject project = 1;
}

message ListProjectsRequest {
    string tenant_id = 1;
    string target_application = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListProjectsResponse {
    repeated DesignProject projects = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateProjectRequest {
    string tenant_id = 1;
    string project_id = 2;
    string project_name = 3;
    string description = 4;
    string updated_by = 5;
}

message UpdateProjectResponse {
    DesignProject project = 1;
}

message PublishProjectRequest {
    string tenant_id = 1;
    string project_id = 2;
    string published_by = 3;
}

message PublishProjectResponse {
    DesignProject project = 1;
}

message PageCustomization {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string page_key = 4;
    string page_name = 5;
    string base_template = 6;
    string custom_layout_config = 7;
    string variant_name = 8;
    string target_roles = 9;
    string target_conditions = 10;
    string display_rules = 11;
    int32 priority = 12;
    bool is_default = 13;
}

message CreatePageCustomizationRequest {
    string tenant_id = 1;
    string project_id = 2;
    string page_key = 3;
    string page_name = 4;
    string base_template = 5;
    string custom_layout_config = 6;
    string variant_name = 7;
    string target_roles = 8;
    string target_conditions = 9;
    string created_by = 10;
}

message CreatePageCustomizationResponse {
    PageCustomization customization = 1;
}

message GetPageCustomizationRequest {
    string tenant_id = 1;
    string customization_id = 2;
}

message GetPageCustomizationResponse {
    PageCustomization customization = 1;
}

message ListPageCustomizationsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string page_key = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPageCustomizationsResponse {
    repeated PageCustomization customizations = 1;
    string next_page_token = 2;
}

message PreviewPageCustomizationRequest {
    string tenant_id = 1;
    string customization_id = 2;
    string preview_context = 3;
}

message PreviewPageCustomizationResponse {
    string rendered_html = 1;
    string applied_rules = 2;
}

message DuplicatePageCustomizationRequest {
    string tenant_id = 1;
    string customization_id = 2;
    string new_variant_name = 3;
    string duplicated_by = 4;
}

message DuplicatePageCustomizationResponse {
    PageCustomization customization = 1;
}

message CustomComponent {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string component_name = 4;
    string component_type = 5;
    string data_source_config = 6;
    string display_config = 7;
    string interaction_config = 8;
    int32 refresh_interval_seconds = 9;
    bool cache_enabled = 10;
}

message CreateComponentRequest {
    string tenant_id = 1;
    string project_id = 2;
    string component_name = 3;
    string component_type = 4;
    string data_source_config = 5;
    string display_config = 6;
    string interaction_config = 7;
    string created_by = 8;
}

message CreateComponentResponse {
    CustomComponent component = 1;
}

message GetComponentRequest {
    string tenant_id = 1;
    string component_id = 2;
}

message GetComponentResponse {
    CustomComponent component = 1;
}

message ListComponentsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string component_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListComponentsResponse {
    repeated CustomComponent components = 1;
    string next_page_token = 2;
}

message RenderComponentPreviewRequest {
    string tenant_id = 1;
    string component_id = 2;
    string preview_data = 3;
}

message RenderComponentPreviewResponse {
    string rendered_output = 1;
}

message ThemeConfiguration {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string theme_name = 4;
    string colors = 5;
    string fonts = 6;
    string spacing = 7;
    string logo_reference = 8;
    string favicon_reference = 9;
    string custom_css = 10;
    bool is_active = 11;
}

message CreateThemeRequest {
    string tenant_id = 1;
    string project_id = 2;
    string theme_name = 3;
    string colors = 4;
    string fonts = 5;
    string spacing = 6;
    string logo_reference = 7;
    string favicon_reference = 8;
    string custom_css = 9;
    string created_by = 10;
}

message CreateThemeResponse {
    ThemeConfiguration theme = 1;
}

message ListThemesRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message ListThemesResponse {
    repeated ThemeConfiguration themes = 1;
}

message ActivateThemeRequest {
    string tenant_id = 1;
    string theme_id = 2;
    string activated_by = 3;
}

message ActivateThemeResponse {
    ThemeConfiguration theme = 1;
}

message ResetDefaultThemeRequest {
    string tenant_id = 1;
    string project_id = 2;
    string reset_by = 3;
}

message ResetDefaultThemeResponse {
    ThemeConfiguration theme = 1;
}

message ConditionalRule {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string rule_name = 4;
    string rule_type = 5;
    string condition_expression = 6;
    string action_config = 7;
    int32 priority = 8;
    bool is_active = 9;
}

message CreateConditionalRuleRequest {
    string tenant_id = 1;
    string project_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string condition_expression = 5;
    string action_config = 6;
    int32 priority = 7;
    string created_by = 8;
}

message CreateConditionalRuleResponse {
    ConditionalRule rule = 1;
}

message ListConditionalRulesRequest {
    string tenant_id = 1;
    string project_id = 2;
    string rule_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListConditionalRulesResponse {
    repeated ConditionalRule rules = 1;
    string next_page_token = 2;
}

message EvaluateConditionalRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string evaluation_context = 3;
}

message EvaluateConditionalRuleResponse {
    bool matches = 1;
    string action_to_take = 2;
}

message ABTest {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string test_name = 4;
    string test_type = 5;
    string variant_a_config = 6;
    string variant_b_config = 7;
    double target_audience_pct = 8;
    string start_date = 9;
    string end_date = 10;
    string metric_type = 11;
    int32 variant_a_conversions = 12;
    int32 variant_b_conversions = 13;
    int32 total_participants = 14;
    string winner = 15;
    string status = 16;
}

message CreateABTestRequest {
    string tenant_id = 1;
    string project_id = 2;
    string test_name = 3;
    string test_type = 4;
    string variant_a_config = 5;
    string variant_b_config = 6;
    double target_audience_pct = 7;
    string start_date = 8;
    string end_date = 9;
    string metric_type = 10;
    string created_by = 11;
}

message CreateABTestResponse {
    ABTest test = 1;
}

message GetABTestRequest {
    string tenant_id = 1;
    string test_id = 2;
}

message GetABTestResponse {
    ABTest test = 1;
}

message ListABTestsRequest {
    string tenant_id = 1;
    string project_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListABTestsResponse {
    repeated ABTest tests = 1;
    string next_page_token = 2;
}

message StartABTestRequest {
    string tenant_id = 1;
    string test_id = 2;
    string started_by = 3;
}

message StartABTestResponse {
    ABTest test = 1;
}

message StopABTestRequest {
    string tenant_id = 1;
    string test_id = 2;
    string stopped_by = 3;
}

message StopABTestResponse {
    ABTest test = 1;
}

message ABTestResult {
    string test_id = 1;
    string winner = 2;
    double variant_a_conversion_rate = 3;
    double variant_b_conversion_rate = 4;
    double confidence_level = 5;
    int32 total_participants = 6;
    bool statistically_significant = 7;
}

message GetABTestResultsRequest {
    string tenant_id = 1;
    string test_id = 2;
}

message GetABTestResultsResponse {
    ABTestResult result = 1;
}

message Sandbox {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string sandbox_name = 4;
    string status = 5;
    string created_by = 6;
    string published_by = 7;
    string published_date = 8;
}

message CreateSandboxRequest {
    string tenant_id = 1;
    string project_id = 2;
    string sandbox_name = 3;
    string created_by = 4;
}

message CreateSandboxResponse {
    Sandbox sandbox = 1;
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
}

message DiscardSandboxResponse {
    Sandbox sandbox = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee roles, org assignments | Evaluate conditional rules for role-based visibility |
| `security-service` | User permissions, role definitions | Validate access to design tools and published views |
| `auth-service` | User sessions, tenant context | Resolve per-user experience variants |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `gateway-service` | Active themes, page configurations, routing rules | Serve customized UI to end users |
| `analytics-service` | A/B test conversion data, page view metrics | Measure experience effectiveness |
| `notification-service` | Design review requests, publication notifications | Collaborative design workflow |
| `cdn-service` | Theme assets, logos, favicons, custom CSS | Distribute static experience assets |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ProjectPublished` | `exdesign.project.published` | `{ tenant_id, project_id, project_name, target_application, published_by }` | Published when a design project is published to production |
| `PageCustomized` | `exdesign.page.customized` | `{ tenant_id, project_id, customization_id, page_key, variant_name }` | Published when a page customization is created or updated |
| `ThemeActivated` | `exdesign.theme.activated` | `{ tenant_id, project_id, theme_id, theme_name, activated_by }` | Published when a theme is activated for a project |
| `ABTestStarted` | `exdesign.abtest.started` | `{ tenant_id, project_id, test_id, test_name, test_type, target_audience_pct }` | Published when an A/B test begins running |
| `ABTestCompleted` | `exdesign.abtest.completed` | `{ tenant_id, project_id, test_id, winner, variant_a_conversions, variant_b_conversions, total_participants }` | Published when an A/B test concludes and a winner is determined |
