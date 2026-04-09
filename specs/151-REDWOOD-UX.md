# 151 - Redwood UX Design System Service Specification

## 1. Domain Overview

Redwood UX provides the unified design system, component library, and theming framework for the entire Fusion application suite. Defines the visual language, interaction patterns, accessibility standards, and responsive design specifications. Supports theme customization, brand theming, dark/light modes, RTL language support, and WCAG 2.1 AA compliance. Includes a token-based design system with color, typography, spacing, and motion primitives. Provides design-to-code tooling, component playground, and design token management. Integrates with Frontend (20) for component implementation and Application Composer (103) for custom UI extensions.

**Bounded Context:** Design System & UX Framework
**Service Name:** `redwood-ux-service`
**Database:** `data/redwood_ux.db`
**HTTP Port:** 8169 | **gRPC Port:** 9169

---

## 2. Database Schema

### 2.1 Design Tokens
```sql
CREATE TABLE ru_design_tokens (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    token_name TEXT NOT NULL,               -- e.g., "color-brand-primary"
    token_category TEXT NOT NULL CHECK(token_category IN (
        'COLOR','TYPOGRAPHY','SPACING','BORDER','SHADOW','MOTION','OPACITY','SIZE','Z_INDEX','CUSTOM'
    )),
    token_value TEXT NOT NULL,              -- CSS value or reference
    value_type TEXT NOT NULL CHECK(value_type IN ('COLOR','DIMENSION','FONT_FAMILY','FONT_SIZE','FONT_WEIGHT','DURATION','EASING','STRING','NUMBER')),
    description TEXT,
    deprecated INTEGER NOT NULL DEFAULT 0,
    deprecation_message TEXT,
    scope TEXT NOT NULL DEFAULT 'GLOBAL'
        CHECK(scope IN ('GLOBAL','COMPONENT','THEME')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, token_name)
);

CREATE INDEX idx_ru_token_category ON ru_design_tokens(tenant_id, token_category);
CREATE INDEX idx_ru_token_deprecated ON ru_design_tokens(tenant_id, deprecated);
```

### 2.2 Themes
```sql
CREATE TABLE ru_themes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    theme_code TEXT NOT NULL,
    theme_name TEXT NOT NULL,
    theme_mode TEXT NOT NULL DEFAULT 'LIGHT'
        CHECK(theme_mode IN ('LIGHT','DARK','HIGH_CONTRAST','AUTO')),
    base_theme_id TEXT,                     -- Inherit from base theme
    color_scheme TEXT NOT NULL,             -- JSON: { "primary": "#...", "secondary": "#...", ... }
    typography TEXT NOT NULL,               -- JSON: font families, sizes, weights
    spacing_scale TEXT NOT NULL,            -- JSON: { "xs": "4px", "sm": "8px", ... }
    border_radius TEXT NOT NULL,            -- JSON: { "sm": "2px", "md": "4px", "lg": "8px" }
    shadow_tokens TEXT,                     -- JSON: shadow definitions
    motion_tokens TEXT,                     -- JSON: duration and easing
    icon_set TEXT NOT NULL DEFAULT 'REDWOOD',
    logo_url TEXT,
    favicon_url TEXT,
    custom_css TEXT,
    rtl_support INTEGER NOT NULL DEFAULT 1,
    density TEXT NOT NULL DEFAULT 'DEFAULT'
        CHECK(density IN ('COMPACT','DEFAULT','COMFORTABLE')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, theme_code)
);

CREATE INDEX idx_ru_theme_tenant ON ru_themes(tenant_id, status);
```

### 2.3 Component Registry
```sql
CREATE TABLE ru_components (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    component_code TEXT NOT NULL,
    component_name TEXT NOT NULL,
    component_type TEXT NOT NULL CHECK(component_type IN (
        'ATOM','MOLECULE','ORGANISM','TEMPLATE','PAGE','PATTERN'
    )),
    category TEXT NOT NULL,                 -- e.g., "Form Controls", "Navigation", "Data Display"
    description TEXT,
    props_schema TEXT NOT NULL,             -- JSON Schema: component props
    events_schema TEXT,                     -- JSON Schema: emitted events
    slots_schema TEXT,                      -- JSON: slot definitions
    default_props TEXT,                     -- JSON: default prop values
    accessibility_notes TEXT,
    design_spec_url TEXT,
    figma_url TEXT,
    storybook_url TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','STABLE','DEPRECATED','ARCHIVED')),
    version_number TEXT NOT NULL DEFAULT '1.0.0',
    dependencies TEXT,                      -- JSON array: component dependencies
    tags TEXT,                              -- JSON array: searchable tags

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, component_code)
);

CREATE INDEX idx_ru_comp_tenant ON ru_components(tenant_id, category);
CREATE INDEX idx_ru_comp_type ON ru_components(tenant_id, component_type);
```

### 2.4 Page Templates
```sql
CREATE TABLE ru_page_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_type TEXT NOT NULL CHECK(template_type IN (
        'LIST','DETAIL','FORM','DASHBOARD','WIZARD','CANVAS','SPLIT','CUSTOM'
    )),
    description TEXT,
    layout_config TEXT NOT NULL,            -- JSON: page layout structure
    responsive_breakpoints TEXT NOT NULL,   -- JSON: { "mobile": "...", "tablet": "...", "desktop": "..." }
    navigation_config TEXT,                 -- JSON: nav items, breadcrumbs
    header_config TEXT,                     -- JSON: header content
    sidebar_config TEXT,                    -- JSON: sidebar configuration
    footer_config TEXT,                     -- JSON: footer configuration
    accessibility_compliance TEXT NOT NULL DEFAULT 'WCAG_2_1_AA'
        CHECK(accessibility_compliance IN ('WCAG_2_0_A','WCAG_2_1_AA','WCAG_2_1_AAA','SECTION_508')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);
```

### 2.5 Accessibility Audit
```sql
CREATE TABLE ru_accessibility_audits (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    audit_scope TEXT NOT NULL,              -- Component or page being audited
    scope_type TEXT NOT NULL CHECK(scope_type IN ('COMPONENT','PAGE','APPLICATION')),
    scope_id TEXT NOT NULL,
    audit_date TEXT NOT NULL,
    auditor TEXT NOT NULL,
    compliance_level TEXT NOT NULL CHECK(compliance_level IN ('PASS','CONDITIONAL','FAIL')),
    total_checks INTEGER NOT NULL DEFAULT 0,
    passed_checks INTEGER NOT NULL DEFAULT 0,
    failed_checks INTEGER NOT NULL DEFAULT 0,
    warnings INTEGER NOT NULL DEFAULT 0,
    issues TEXT NOT NULL,                   -- JSON array: [{ "rule": "...", "severity": "...", "element": "..." }]
    remediation_notes TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','IN_REMEDIATION','RESOLVED','CERTIFIED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, scope_type, scope_id, audit_date)
);
```

---

## 3. API Endpoints

### 3.1 Design Tokens
| Method | Path | Description |
|--------|------|-------------|
| POST | `/redwood-ux/v1/tokens` | Create design token |
| GET | `/redwood-ux/v1/tokens` | List tokens (filterable by category) |
| GET | `/redwood-ux/v1/tokens/{id}` | Get token details |
| PUT | `/redwood-ux/v1/tokens/{id}` | Update token |
| DELETE | `/redwood-ux/v1/tokens/{id}` | Deprecate token |
| POST | `/redwood-ux/v1/tokens/export` | Export tokens as CSS/SCSS/JSON |

### 3.2 Themes
| Method | Path | Description |
|--------|------|-------------|
| POST | `/redwood-ux/v1/themes` | Create theme |
| GET | `/redwood-ux/v1/themes` | List themes |
| GET | `/redwood-ux/v1/themes/{id}` | Get theme details |
| PUT | `/redwood-ux/v1/themes/{id}` | Update theme |
| POST | `/redwood-ux/v1/themes/{id}/activate` | Activate theme |
| POST | `/redwood-ux/v1/themes/{id}/duplicate` | Duplicate theme |
| POST | `/redwood-ux/v1/themes/{id}/preview` | Generate theme preview |

### 3.3 Components
| Method | Path | Description |
|--------|------|-------------|
| POST | `/redwood-ux/v1/components` | Register component |
| GET | `/redwood-ux/v1/components` | List components |
| GET | `/redwood-ux/v1/components/{id}` | Get component details |
| PUT | `/redwood-ux/v1/components/{id}` | Update component |
| GET | `/redwood-ux/v1/components/{id}/playground` | Component playground |

### 3.4 Page Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/redwood-ux/v1/page-templates` | Create page template |
| GET | `/redwood-ux/v1/page-templates` | List templates |
| GET | `/redwood-ux/v1/page-templates/{id}` | Get template details |
| PUT | `/redwood-ux/v1/page-templates/{id}` | Update template |
| POST | `/redwood-ux/v1/page-templates/{id}/preview` | Preview template |

### 3.5 Accessibility
| Method | Path | Description |
|--------|------|-------------|
| POST | `/redwood-ux/v1/accessibility/audit` | Run accessibility audit |
| GET | `/redwood-ux/v1/accessibility/audits` | List audits |
| GET | `/redwood-ux/v1/accessibility/audits/{id}` | Get audit results |
| GET | `/redwood-ux/v1/accessibility/compliance-report` | Compliance summary |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `ux.theme.activated` | `{ theme_id, theme_code }` | Theme activated |
| `ux.theme.changed` | `{ theme_id, changed_tokens }` | Theme tokens modified |
| `ux.component.registered` | `{ component_id, code, type }` | New component registered |
| `ux.component.deprecated` | `{ component_id, replacement }` | Component deprecated |
| `ux.audit.completed` | `{ audit_id, scope, compliance_level }` | Accessibility audit completed |
| `ux.audit.issue_found` | `{ audit_id, rule, severity }` | Accessibility issue detected |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `deployment.started` | Deployment | Validate theme before deployment |
| `frontend.build.started` | Frontend | Export design tokens for build |

---


---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.redwood_ux.v1;

service RedwoodUxService {
    // Design tokens
    rpc CreateToken(CreateTokenRequest) returns (DesignToken);
    rpc GetToken(GetTokenRequest) returns (DesignToken);
    rpc ListTokens(ListTokensRequest) returns (TokenList);
    rpc ExportTokens(ExportTokensRequest) returns (ExportTokensResponse);

    // Themes
    rpc CreateTheme(CreateThemeRequest) returns (Theme);
    rpc ActivateTheme(ActivateThemeRequest) returns (Theme);

    // Components
    rpc RegisterComponent(RegisterComponentRequest) returns (Component);
    rpc GetComponent(GetComponentRequest) returns (Component);

    // Accessibility
    rpc RunAudit(RunAuditRequest) returns (AccessibilityAudit);
}

// Entity messages
message DesignToken {
    string id = 1;
    string tenant_id = 2;
    string token_name = 3;
    string token_category = 4;
    string token_value = 5;
    string value_type = 6;
    string description = 7;
    bool deprecated = 8;
    string deprecation_message = 9;
    string scope = 10;
    int32 version = 11;
}

message Theme {
    string id = 1;
    string tenant_id = 2;
    string theme_code = 3;
    string theme_name = 4;
    string theme_mode = 5;
    string base_theme_id = 6;
    string color_scheme = 7;
    string typography = 8;
    string spacing_scale = 9;
    string border_radius = 10;
    string shadow_tokens = 11;
    string motion_tokens = 12;
    string icon_set = 13;
    string logo_url = 14;
    string favicon_url = 15;
    string custom_css = 16;
    bool rtl_support = 17;
    string density = 18;
    string status = 19;
    int32 version = 20;
}

message Component {
    string id = 1;
    string tenant_id = 2;
    string component_code = 3;
    string component_name = 4;
    string component_type = 5;
    string category = 6;
    string description = 7;
    string props_schema = 8;
    string events_schema = 9;
    string slots_schema = 10;
    string default_props = 11;
    string accessibility_notes = 12;
    string design_spec_url = 13;
    string figma_url = 14;
    string storybook_url = 15;
    string status = 16;
    string version_number = 17;
    string dependencies = 18;
    string tags = 19;
}

message AccessibilityAudit {
    string id = 1;
    string tenant_id = 2;
    string audit_scope = 3;
    string scope_type = 4;
    string scope_id = 5;
    string audit_date = 6;
    string auditor = 7;
    string compliance_level = 8;
    int32 total_checks = 9;
    int32 passed_checks = 10;
    int32 failed_checks = 11;
    int32 warnings = 12;
    string issues = 13;
    string remediation_notes = 14;
    string status = 15;
}

// Request/Response messages
message CreateTokenRequest {
    string tenant_id = 1;
    string token_name = 2;
    string token_category = 3;
    string token_value = 4;
    string value_type = 5;
    string description = 6;
    string scope = 7;
}

message GetTokenRequest {
    string id = 1;
    string tenant_id = 2;
}

message ListTokensRequest {
    string tenant_id = 1;
    string token_category = 2;
    bool deprecated = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message TokenList {
    repeated DesignToken tokens = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ExportTokensRequest {
    string tenant_id = 1;
    string export_format = 2;
    repeated string token_categories = 3;
}

message ExportTokensResponse {
    string content = 1;
    string format = 2;
    int32 token_count = 3;
}

message CreateThemeRequest {
    string tenant_id = 1;
    string theme_code = 2;
    string theme_name = 3;
    string theme_mode = 4;
    string base_theme_id = 5;
    string color_scheme = 6;
    string typography = 7;
    string spacing_scale = 8;
    string border_radius = 9;
    string density = 10;
}

message ActivateThemeRequest {
    string id = 1;
    string tenant_id = 2;
}

message RegisterComponentRequest {
    string tenant_id = 1;
    string component_code = 2;
    string component_name = 3;
    string component_type = 4;
    string category = 5;
    string description = 6;
    string props_schema = 7;
    string events_schema = 8;
    string slots_schema = 9;
}

message GetComponentRequest {
    string id = 1;
    string tenant_id = 2;
}

message RunAuditRequest {
    string tenant_id = 1;
    string scope_type = 2;
    string scope_id = 3;
    string auditor = 4;
}
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | ru_design_tokens | -- |
| V002 | ru_themes | V001 |
| V003 | ru_components | -- |
| V004 | ru_page_templates | V002, V003 |
| V005 | ru_accessibility_audits | V003, V004 |

---

## 7. Business Rules

1. **Token Immutability**: Published tokens cannot be deleted, only deprecated with replacement
2. **Theme Inheritance**: Custom themes inherit from base theme; overrides explicitly defined
3. **Accessibility**: All components must pass WCAG 2.1 AA before marked STABLE
4. **Version Compatibility**: Component versioning follows semver; major changes require migration guide
5. **Responsive Design**: All page templates must define mobile, tablet, and desktop breakpoints
6. **RTL Support**: Themes with RTL support must mirror all directional tokens
7. **Dark Mode**: All themes must support dark mode variant with sufficient contrast ratios
8. **Component Isolation**: Components must be self-contained with no global style dependencies

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Frontend (20) | Component library consumption, design token build |
| Application Composer (103) | Custom UI component extensions |
| Visual Builder (90) | Design-time component palette |
| Experience Design Studio (109) | Theme and layout customization |
| Mobile (46) | Mobile-specific design tokens |
| Multi-Tenancy (18) | Per-tenant theme management |
| Digital Assistant (43) | Chat widget theming |
| Deployment (21) | Theme validation in CI/CD |
