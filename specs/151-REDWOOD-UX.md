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

## 5. Business Rules

1. **Token Immutability**: Published tokens cannot be deleted, only deprecated with replacement
2. **Theme Inheritance**: Custom themes inherit from base theme; overrides explicitly defined
3. **Accessibility**: All components must pass WCAG 2.1 AA before marked STABLE
4. **Version Compatibility**: Component versioning follows semver; major changes require migration guide
5. **Responsive Design**: All page templates must define mobile, tablet, and desktop breakpoints
6. **RTL Support**: Themes with RTL support must mirror all directional tokens
7. **Dark Mode**: All themes must support dark mode variant with sufficient contrast ratios
8. **Component Isolation**: Components must be self-contained with no global style dependencies

---

## 6. Integration Points

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
