# 58 - Configurator Service Specification

## 1. Domain Overview

Configurator provides a rules-based product configuration engine supporting feature-option modeling, multi-type rule evaluation (compatibility, incompatibility, requirement, exclusion, recommendation), mathematical and logical constraint solving, configuration-to-BOM mapping, configuration-to-pricing impact mapping, valid configuration caching, and runtime session management. The engine evaluates user selections against a complete rule set in real time, prevents invalid configurations, and generates manufacturing-ready BOM outputs. Integrates with CPQ for quote-driven configuration, MFG for BOM generation, and PLM for product structure synchronization.

**Bounded Context:** Product Configuration Engine & Rule Management
**Service Name:** `configurator-service`
**Database:** `data/configurator.db`
**HTTP Port:** 8090 | **gRPC Port:** 9090

---

## 2. Database Schema

### 2.1 Configurator Models
```sql
CREATE TABLE configurator_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    product_family TEXT NOT NULL,                  -- Product family classification
    base_item_id TEXT NOT NULL,                    -- FK to Inventory base item
    model_version TEXT NOT NULL DEFAULT '1.0.0',
    configuration_strategy TEXT NOT NULL DEFAULT 'CONSTRAINT_PROPAGATION'
        CHECK(configuration_strategy IN ('CONSTRAINT_PROPAGATION','RULE_CHAIN','HYBRID')),
    max_configuration_depth INTEGER NOT NULL DEFAULT 5,
    validation_mode TEXT NOT NULL DEFAULT 'STRICT'
        CHECK(validation_mode IN ('STRICT','LENIENT','ADVISORY')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','DEPRECATED')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    last_rule_evaluation_ms INTEGER NOT NULL DEFAULT 0, -- Performance tracking

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_config_models_tenant_family ON configurator_models(tenant_id, product_family);
CREATE INDEX idx_config_models_tenant_status ON configurator_models(tenant_id, status);
CREATE INDEX idx_config_models_tenant_item ON configurator_models(tenant_id, base_item_id);
CREATE INDEX idx_config_models_tenant_dates ON configurator_models(tenant_id, effective_from, effective_to);
CREATE INDEX idx_config_models_tenant_active ON configurator_models(tenant_id, is_active);
```

### 2.2 Configurator Features
```sql
CREATE TABLE configurator_features (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    feature_name TEXT NOT NULL,
    feature_code TEXT NOT NULL,
    feature_type TEXT NOT NULL
        CHECK(feature_type IN ('SINGLE_SELECT','MULTI_SELECT','NUMERIC','TEXT','BOOLEAN','COLOR','DATE_RANGE')),
    description TEXT,
    is_required INTEGER NOT NULL DEFAULT 0,
    min_selections INTEGER NOT NULL DEFAULT 1,
    max_selections INTEGER NOT NULL DEFAULT 1,
    default_value TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    group_name TEXT,                               -- UI grouping
    parent_feature_id TEXT,                        -- For hierarchical features
    selection_triggers TEXT,                       -- JSON: cascading trigger definitions

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES configurator_models(id),
    FOREIGN KEY (parent_feature_id) REFERENCES configurator_features(id),
    UNIQUE(tenant_id, model_id, feature_code)
);

CREATE INDEX idx_config_features_tenant_model ON configurator_features(tenant_id, model_id);
CREATE INDEX idx_config_features_tenant_type ON configurator_features(tenant_id, feature_type);
CREATE INDEX idx_config_features_tenant_group ON configurator_features(tenant_id, group_name);
CREATE INDEX idx_config_features_tenant_parent ON configurator_features(tenant_id, parent_feature_id);
CREATE INDEX idx_config_features_tenant_active ON configurator_features(tenant_id, is_active);
```

### 2.3 Configurator Options
```sql
CREATE TABLE configurator_options (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    feature_id TEXT NOT NULL,
    option_code TEXT NOT NULL,
    option_label TEXT NOT NULL,
    option_value TEXT NOT NULL,
    description TEXT,
    numeric_value REAL,                           -- For numeric comparisons in rules
    color_value TEXT,                              -- Hex color for COLOR features
    image_url TEXT,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    weight_impact REAL NOT NULL DEFAULT 0.0,       -- Weight delta in kg
    cost_impact_cents INTEGER NOT NULL DEFAULT 0,  -- Cost delta
    is_default INTEGER NOT NULL DEFAULT 0,
    is_recommended INTEGER NOT NULL DEFAULT 0,
    display_order INTEGER NOT NULL DEFAULT 0,
    availability_status TEXT NOT NULL DEFAULT 'AVAILABLE'
        CHECK(availability_status IN ('AVAILABLE','LIMITED','DISCONTINUED','OUT_OF_STOCK')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (feature_id) REFERENCES configurator_features(id),
    UNIQUE(tenant_id, feature_id, option_code)
);

CREATE INDEX idx_config_options_tenant_feature ON configurator_options(tenant_id, feature_id);
CREATE INDEX idx_config_options_tenant_default ON configurator_options(tenant_id, is_default);
CREATE INDEX idx_config_options_tenant_availability ON configurator_options(tenant_id, availability_status);
CREATE INDEX idx_config_options_tenant_active ON configurator_options(tenant_id, is_active);
```

### 2.4 Configurator Rules
```sql
CREATE TABLE configurator_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('COMPATIBILITY','INCOMPATIBILITY','REQUIREMENT','EXCLUSION',
                            'RECOMMENDATION','ENUMERATION','RANGE','DEPENDENCY')),
    priority INTEGER NOT NULL DEFAULT 100,
    severity TEXT NOT NULL DEFAULT 'ERROR'
        CHECK(severity IN ('ERROR','WARNING','INFO')),
    source_feature_id TEXT NOT NULL,
    source_option_id TEXT,                         -- NULL = any option in source feature
    target_feature_id TEXT NOT NULL,
    target_option_id TEXT,                         -- NULL = any option in target feature
    condition_expression TEXT,                     -- JSON: additional complex conditions
    action_type TEXT NOT NULL DEFAULT 'HIDE'
        CHECK(action_type IN ('HIDE','SHOW','ENABLE','DISABLE','SET_DEFAULT','REQUIRE','SUGGEST')),
    message TEXT,                                  -- User-facing explanation
    effective_from TEXT,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES configurator_models(id),
    FOREIGN KEY (source_feature_id) REFERENCES configurator_features(id),
    FOREIGN KEY (target_feature_id) REFERENCES configurator_features(id),
    UNIQUE(tenant_id, model_id, rule_code)
);

CREATE INDEX idx_config_rules_tenant_model ON configurator_rules(tenant_id, model_id);
CREATE INDEX idx_config_rules_tenant_type ON configurator_rules(tenant_id, rule_type);
CREATE INDEX idx_config_rules_tenant_source ON configurator_rules(tenant_id, source_feature_id);
CREATE INDEX idx_config_rules_tenant_target ON configurator_rules(tenant_id, target_feature_id);
CREATE INDEX idx_config_rules_tenant_priority ON configurator_rules(tenant_id, priority);
CREATE INDEX idx_config_rules_tenant_active ON configurator_rules(tenant_id, is_active);
```

### 2.5 Configurator Constraints
```sql
CREATE TABLE configurator_constraints (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    constraint_name TEXT NOT NULL,
    constraint_code TEXT NOT NULL,
    constraint_type TEXT NOT NULL
        CHECK(constraint_type IN ('SUM_EQUAL','SUM_LESS_THAN','SUM_GREATER_THAN',
                                   'RANGE','UNIQUE','CUSTOM_EXPRESSION')),
    left_expression TEXT NOT NULL,                 -- JSON: feature/option/value reference
    operator TEXT NOT NULL
        CHECK(operator IN ('EQ','NE','LT','LTE','GT','GTE','IN','NOT_IN','BETWEEN')),
    right_expression TEXT NOT NULL,                -- JSON: literal value or feature reference
    tolerance_percent REAL NOT NULL DEFAULT 0.0,
    violation_message TEXT,
    is_hard_constraint INTEGER NOT NULL DEFAULT 1, -- 1 = must satisfy, 0 = advisory

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES configurator_models(id),
    UNIQUE(tenant_id, model_id, constraint_code)
);

CREATE INDEX idx_config_constraints_tenant_model ON configurator_constraints(tenant_id, model_id);
CREATE INDEX idx_config_constraints_tenant_type ON configurator_constraints(tenant_id, constraint_type);
CREATE INDEX idx_config_constraints_tenant_hard ON configurator_constraints(tenant_id, is_hard_constraint);
CREATE INDEX idx_config_constraints_tenant_active ON configurator_constraints(tenant_id, is_active);
```

### 2.6 Configurator BOM Mappings
```sql
CREATE TABLE configurator_bom_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    feature_id TEXT NOT NULL,
    option_id TEXT,                                -- NULL = applies to all options in feature
    bom_item_id TEXT NOT NULL,                     -- Inventory item to include in BOM
    bom_quantity REAL NOT NULL DEFAULT 1.0,
    uom_code TEXT NOT NULL DEFAULT 'EA',
    bom_position TEXT,                             -- BOM position/path (e.g. "1.2.3")
    is_optional INTEGER NOT NULL DEFAULT 0,
    substitute_item_id TEXT,                       -- Alternate BOM item

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES configurator_models(id),
    FOREIGN KEY (feature_id) REFERENCES configurator_features(id),
    FOREIGN KEY (option_id) REFERENCES configurator_options(id)
);

CREATE INDEX idx_bom_mappings_tenant_model ON configurator_bom_mappings(tenant_id, model_id);
CREATE INDEX idx_bom_mappings_tenant_feature ON configurator_bom_mappings(tenant_id, feature_id);
CREATE INDEX idx_bom_mappings_tenant_option ON configurator_bom_mappings(tenant_id, option_id);
CREATE INDEX idx_bom_mappings_tenant_item ON configurator_bom_mappings(tenant_id, bom_item_id);
CREATE INDEX idx_bom_mappings_tenant_active ON configurator_bom_mappings(tenant_id, is_active);
```

### 2.7 Configurator Pricing Mappings
```sql
CREATE TABLE configurator_pricing_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    feature_id TEXT NOT NULL,
    option_id TEXT,
    price_impact_cents INTEGER NOT NULL DEFAULT 0,
    impact_type TEXT NOT NULL DEFAULT 'ABSOLUTE'
        CHECK(impact_type IN ('ABSOLUTE','PERCENTAGE','MULTIPLIER','FORMULA')),
    price_formula TEXT,                            -- For FORMULA impact type
    applies_to TEXT NOT NULL DEFAULT 'UNIT_PRICE'
        CHECK(applies_to IN ('UNIT_PRICE','TOTAL_PRICE','SETUP_FEE','RECURRING')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    effective_from TEXT,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES configurator_models(id),
    FOREIGN KEY (feature_id) REFERENCES configurator_features(id),
    FOREIGN KEY (option_id) REFERENCES configurator_options(id)
);

CREATE INDEX idx_price_mappings_tenant_model ON configurator_pricing_mappings(tenant_id, model_id);
CREATE INDEX idx_price_mappings_tenant_feature ON configurator_pricing_mappings(tenant_id, feature_id);
CREATE INDEX idx_price_mappings_tenant_type ON configurator_pricing_mappings(tenant_id, impact_type);
CREATE INDEX idx_price_mappings_tenant_active ON configurator_pricing_mappings(tenant_id, is_active);
```

### 2.8 Configurator Valid Configurations
```sql
CREATE TABLE configurator_valid_configurations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    configuration_hash TEXT NOT NULL,              -- Hash of feature-option selections
    configuration_json TEXT NOT NULL,              -- JSON: complete feature-option map
    is_valid INTEGER NOT NULL DEFAULT 1,
    validation_errors TEXT,                        -- JSON array: error details if invalid
    rule_violations TEXT,                          -- JSON array: violated rule codes
    total_price_impact_cents INTEGER NOT NULL DEFAULT 0,
    bom_item_count INTEGER NOT NULL DEFAULT 0,
    total_lead_time_days INTEGER NOT NULL DEFAULT 0,
    total_weight_kg REAL NOT NULL DEFAULT 0.0,
    access_count INTEGER NOT NULL DEFAULT 0,
    last_accessed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, model_id, configuration_hash)
);

CREATE INDEX idx_valid_configs_tenant_model ON configurator_valid_configurations(tenant_id, model_id);
CREATE INDEX idx_valid_configs_tenant_valid ON configurator_valid_configurations(tenant_id, is_valid);
CREATE INDEX idx_valid_configs_tenant_hash ON configurator_valid_configurations(tenant_id, configuration_hash);
CREATE INDEX idx_valid_configs_tenant_access ON configurator_valid_configurations(tenant_id, last_accessed_at);
```

### 2.9 Configuration Sessions
```sql
CREATE TABLE configuration_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    session_token TEXT NOT NULL,                   -- Unique session identifier
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','COMPLETED','ABANDONED','EXPIRED')),
    selections TEXT,                               -- JSON: current feature-option selections
    validation_result TEXT,                        -- JSON: latest validation result
    generated_bom TEXT,                            -- JSON: generated BOM output
    generated_price_cents INTEGER,                -- Calculated price from selections
    source_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(source_type IN ('MANUAL','CPQ','API','IMPORT')),
    source_ref_id TEXT,                            -- Reference to originating quote/record
    started_at TEXT NOT NULL DEFAULT (datetime('now')),
    completed_at TEXT,
    expires_at TEXT NOT NULL,                      -- Session expiration time

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, session_token)
);

CREATE INDEX idx_config_sessions_tenant_model ON configuration_sessions(tenant_id, model_id);
CREATE INDEX idx_config_sessions_tenant_user ON configuration_sessions(tenant_id, user_id);
CREATE INDEX idx_config_sessions_tenant_status ON configuration_sessions(tenant_id, status);
CREATE INDEX idx_config_sessions_tenant_source ON configuration_sessions(tenant_id, source_type);
CREATE INDEX idx_config_sessions_tenant_expires ON configuration_sessions(tenant_id, expires_at);
```

---

## 3. REST API Endpoints

```
# Model Management
GET/POST       /api/v1/configurator/models                Permission: configurator.models.manage
GET/PUT        /api/v1/configurator/models/{id}
PATCH          /api/v1/configurator/models/{id}/status    Permission: configurator.models.manage
GET            /api/v1/configurator/models/{id}/structure

# Feature Management
GET/POST       /api/v1/configurator/models/{id}/features  Permission: configurator.models.manage
GET/PUT        /api/v1/configurator/features/{id}
DELETE         /api/v1/configurator/features/{id}         Permission: configurator.models.manage

# Option Management
GET/POST       /api/v1/configurator/features/{id}/options Permission: configurator.models.manage
GET/PUT        /api/v1/configurator/options/{id}
DELETE         /api/v1/configurator/options/{id}          Permission: configurator.models.manage

# Rule Management
GET/POST       /api/v1/configurator/models/{id}/rules     Permission: configurator.rules.manage
GET/PUT        /api/v1/configurator/rules/{id}
DELETE         /api/v1/configurator/rules/{id}            Permission: configurator.rules.manage
POST           /api/v1/configurator/rules/validate-set    Permission: configurator.rules.manage

# Constraint Management
GET/POST       /api/v1/configurator/models/{id}/constraints  Permission: configurator.rules.manage
GET/PUT        /api/v1/configurator/constraints/{id}
DELETE         /api/v1/configurator/constraints/{id}         Permission: configurator.rules.manage

# BOM Mappings
GET/POST       /api/v1/configurator/models/{id}/bom-mappings     Permission: configurator.mappings.manage
GET/PUT        /api/v1/configurator/bom-mappings/{id}
DELETE         /api/v1/configurator/bom-mappings/{id}            Permission: configurator.mappings.manage

# Pricing Mappings
GET/POST       /api/v1/configurator/models/{id}/pricing-mappings  Permission: configurator.mappings.manage
GET/PUT        /api/v1/configurator/pricing-mappings/{id}
DELETE         /api/v1/configurator/pricing-mappings/{id}         Permission: configurator.mappings.manage

# Configuration Sessions
POST           /api/v1/configurator/sessions                Permission: configurator.configure
GET            /api/v1/configurator/sessions/{id}
DELETE         /api/v1/configurator/sessions/{id}

# Configuration Operations
POST           /api/v1/configurator/sessions/{id}/select    Permission: configurator.configure
DELETE         /api/v1/configurator/sessions/{id}/deselect/{feature_id}  Permission: configurator.configure
POST           /api/v1/configurator/sessions/{id}/validate  Permission: configurator.configure
GET            /api/v1/configurator/sessions/{id}/valid-options
GET            /api/v1/configurator/sessions/{id}/conflicts
GET            /api/v1/configurator/sessions/{id}/recommendations
POST           /api/v1/configurator/sessions/{id}/complete   Permission: configurator.configure

# Generated Outputs
GET            /api/v1/configurator/sessions/{id}/bom        Permission: configurator.configure
GET            /api/v1/configurator/sessions/{id}/price       Permission: configurator.configure
GET            /api/v1/configurator/sessions/{id}/summary     Permission: configurator.configure

# Valid Configuration Cache
GET            /api/v1/configurator/models/{id}/valid-configurations
POST           /api/v1/configurator/cache/rebuild             Permission: configurator.cache.manage

# Diagnostics
GET            /api/v1/configurator/models/{id}/rules/evaluate-dry-run  Permission: configurator.rules.manage
GET            /api/v1/configurator/diagnostics/rule-coverage
```

---

## 4. Business Rules

### 4.1 Model & Feature Management
```
Model lifecycle rules:
  1. Model code MUST be unique within a tenant
  2. A model MUST have at least one feature before activation
  3. Feature codes MUST be unique within a model
  4. Features marked as is_required MUST have a selection before configuration completion
  5. Hierarchical features (parent_feature_id set) create dependent sub-feature groups
  6. max_configuration_depth limits the nesting of hierarchical features
  7. Model version follows semantic versioning (major.minor.patch)
  8. DEPRECATED models MUST NOT allow new sessions; existing sessions continue until expiry
```

### 4.2 Rule Evaluation
- Rules are evaluated in priority order (lower number = higher priority)
- COMPATIBILITY rules: if source is selected, target MUST be available for selection
- INCOMPATIBILITY rules: if source is selected, target MUST be hidden or disabled
- REQUIREMENT rules: if source is selected, target MUST also be selected
- EXCLUSION rules: if source is selected, target MUST NOT be selectable
- RECOMMENDATION rules: if source is selected, target SHOULD be suggested (non-blocking)
- DEPENDENCY rules: target option availability depends on source selection state
- Rules with severity ERROR block completion; WARNING and INFO are advisory
- Rules are scoped by effective_from/effective_to date ranges
- All rules for a model are loaded into memory at session start for fast evaluation

### 4.3 Constraint Solving
- Constraints apply mathematical validation across feature selections
- SUM_EQUAL: sum of referenced numeric values must equal target
- SUM_LESS_THAN / SUM_GREATER_THAN: sum comparison against threshold
- RANGE: selected numeric value must fall within defined range
- UNIQUE: selected values across features must be distinct
- CUSTOM_EXPRESSION: arbitrary boolean expression evaluation
- Hard constraints (is_hard_constraint = 1) prevent configuration completion
- Soft constraints (is_hard_constraint = 0) generate advisory messages
- Constraint evaluation uses constraint propagation to prune invalid option spaces early

### 4.4 BOM Generation
- BOM mappings translate feature-option selections into manufacturing bill of materials
- Each mapping links a feature-option pair to an inventory item with quantity
- BOM position defines hierarchical structure (e.g. "1.2.3" = assembly 1, sub-assembly 2, component 3)
- Optional BOM items (is_optional = 1) are included only when the mapping feature-option is selected
- Substitute items provide fallback when primary item is unavailable
- Generated BOM MUST include all required components and selected optional components
- BOM generation fails if any mapped inventory item is discontinued

### 4.5 Session Management
- Sessions expire after a configurable timeout (default: 24 hours)
- Expired sessions are marked EXPIRED and cannot be modified
- Each session tracks the full selection state as JSON
- Validation is performed on every selection change (incremental validation)
- Full validation is performed on session completion
- Sessions track performance metrics (rule evaluation time)
- Abandoned sessions are cleaned up after 7 days

### 4.6 Valid Configuration Caching
- Valid configurations are cached by configuration_hash for performance
- Cache hits avoid full rule re-evaluation, returning pre-computed results
- Cache is invalidated when model rules, features, or options change
- Cache rebuild can be triggered manually or automatically on model publish
- Cache stores total price impact, BOM item count, lead time, and weight
- Access count and last_accessed_at track cache utilization
- Stale cache entries (not accessed in 30 days) are purged

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `configurator.model.created` | New model created | PLM |
| `configurator.model.published` | Model activated | CPQ, MFG |
| `configurator.rule.violated` | Rule violation during session | CPQ, Notification |
| `configurator.configuration.completed` | Session completed with valid config | CPQ, MFG |
| `configurator.configuration.failed` | Session completed with violations | Notification |
| `configurator.bom.generated` | BOM output generated | MFG, INV |
| `configurator.cache.invalidated` | Configuration cache cleared | Reporting |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.configurator.v1;

service ConfiguratorService {
    rpc CreateModel(CreateModelRequest) returns (CreateModelResponse);
    rpc StartSession(StartSessionRequest) returns (StartSessionResponse);
    rpc SelectOption(SelectOptionRequest) returns (SelectOptionResponse);
    rpc DeselectOption(DeselectOptionRequest) returns (DeselectOptionResponse);
    rpc ValidateConfiguration(ValidateConfigurationRequest) returns (ValidateConfigurationResponse);
    rpc GetValidOptions(GetValidOptionsRequest) returns (GetValidOptionsResponse);
    rpc CompleteSession(CompleteSessionRequest) returns (CompleteSessionResponse);
    rpc GenerateBOM(GenerateBOMRequest) returns (GenerateBOMResponse);
}

// Entity messages
message ConfiguratorModel {
    string id = 1;
    string tenant_id = 2;
    string model_code = 3;
    string name = 4;
    string description = 5;
    string product_family = 6;
    string base_item_id = 7;
    string model_version = 8;
    string configuration_strategy = 9;
    int32 max_configuration_depth = 10;
    string validation_mode = 11;
    string status = 12;
    string effective_from = 13;
    string effective_to = 14;
    int32 last_rule_evaluation_ms = 15;
    string created_at = 16;
    string updated_at = 17;
}

message ConfiguratorFeature {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string feature_name = 4;
    string feature_code = 5;
    string feature_type = 6;
    string description = 7;
    int32 is_required = 8;
    int32 min_selections = 9;
    int32 max_selections = 10;
    string default_value = 11;
    int32 display_order = 12;
    string group_name = 13;
    string parent_feature_id = 14;
    string selection_triggers = 15;
    string created_at = 16;
    string updated_at = 17;
}

message ConfiguratorOption {
    string id = 1;
    string tenant_id = 2;
    string feature_id = 3;
    string option_code = 4;
    string option_label = 5;
    string option_value = 6;
    string description = 7;
    double numeric_value = 8;
    string color_value = 9;
    string image_url = 10;
    int32 lead_time_days = 11;
    double weight_impact = 12;
    int64 cost_impact_cents = 13;
    int32 is_default = 14;
    int32 is_recommended = 15;
    int32 display_order = 16;
    string availability_status = 17;
    string created_at = 18;
    string updated_at = 19;
}

message ConfiguratorRule {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string rule_name = 4;
    string rule_code = 5;
    string rule_type = 6;
    int32 priority = 7;
    string severity = 8;
    string source_feature_id = 9;
    string source_option_id = 10;
    string target_feature_id = 11;
    string target_option_id = 12;
    string condition_expression = 13;
    string action_type = 14;
    string message = 15;
    string effective_from = 16;
    string effective_to = 17;
    string created_at = 18;
    string updated_at = 19;
}

message ConfigurationSession {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string user_id = 4;
    string session_token = 5;
    string status = 6;
    string selections = 7;
    string validation_result = 8;
    string generated_bom = 9;
    int64 generated_price_cents = 10;
    string source_type = 11;
    string source_ref_id = 12;
    string started_at = 13;
    string completed_at = 14;
    string expires_at = 15;
    string created_at = 16;
    string updated_at = 17;
}

message BomMapping {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string feature_id = 4;
    string option_id = 5;
    string bom_item_id = 6;
    double bom_quantity = 7;
    string uom_code = 8;
    string bom_position = 9;
    int32 is_optional = 10;
    string substitute_item_id = 11;
    string created_at = 12;
    string updated_at = 13;
}

message BomEntry {
    string item_id = 1;
    string item_name = 2;
    double quantity = 3;
    string uom_code = 4;
    string bom_position = 5;
    int32 is_optional = 6;
    string substitute_item_id = 7;
}

// Request/Response messages
message CreateModelRequest {
    string tenant_id = 1;
    string model_code = 2;
    string name = 3;
    string description = 4;
    string product_family = 5;
    string base_item_id = 6;
    string configuration_strategy = 7;
    int32 max_configuration_depth = 8;
    string validation_mode = 9;
    string effective_from = 10;
    string effective_to = 11;
}

message CreateModelResponse {
    ConfiguratorModel data = 1;
}

message StartSessionRequest {
    string tenant_id = 1;
    string model_id = 2;
    string source_type = 3;
    string source_ref_id = 4;
}

message StartSessionResponse {
    ConfigurationSession data = 1;
    repeated ConfiguratorFeature features = 2;
}

message SelectOptionRequest {
    string tenant_id = 1;
    string session_id = 2;
    string feature_id = 3;
    string option_id = 4;
}

message SelectOptionResponse {
    ConfigurationSession session = 1;
    repeated ConfiguratorOption valid_options = 2;
    repeated string violations = 3;
}

message DeselectOptionRequest {
    string tenant_id = 1;
    string session_id = 2;
    string feature_id = 3;
    string option_id = 4;
}

message DeselectOptionResponse {
    ConfigurationSession session = 1;
    repeated ConfiguratorOption valid_options = 2;
}

message ValidateConfigurationRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message ValidateConfigurationResponse {
    int32 is_valid = 1;
    repeated string errors = 2;
    repeated string warnings = 3;
    repeated string rule_violations = 4;
}

message GetValidOptionsRequest {
    string tenant_id = 1;
    string session_id = 2;
    string feature_id = 3;
}

message GetValidOptionsResponse {
    repeated ConfiguratorOption options = 1;
}

message CompleteSessionRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message CompleteSessionResponse {
    ConfigurationSession session = 1;
    int32 is_valid = 2;
    int64 total_price_cents = 3;
}

message GenerateBOMRequest {
    string tenant_id = 1;
    string session_id = 2;
}

message GenerateBOMResponse {
    string model_id = 1;
    repeated BomEntry items = 2;
    int32 item_count = 3;
    int64 total_cost_cents = 4;
    int32 total_lead_time_days = 5;
    double total_weight_kg = 6;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **PLM (Product Lifecycle):** Product structure definitions, item master data, engineering change orders for model updates
- **INV (Inventory):** Item availability for option availability_status, substitute item validation
- **CPQ (Configure-Price-Quote):** Quote-initiated configuration sessions, pricing context
- **Auth:** User identity for session ownership, model management authorization

### 6.2 Data Published To
- **CPQ (Configure-Price-Quote):** Completed configuration results, validation status, pricing impact data
- **MFG (Manufacturing):** Generated BOM structures for configured products, routing suggestions
- **INV (Inventory):** Configured product item creation requests, BOM component requirements
- **PLM (Product Lifecycle):** Model usage analytics, configuration pattern insights
- **Notification:** Rule violation alerts, session expiration warnings
- **Reporting:** Configuration analytics, popular configurations, rule violation frequency

---

## 7. Migrations

1. V001: `configurator_models`
2. V002: `configurator_features`
3. V003: `configurator_options`
4. V004: `configurator_rules`
5. V005: `configurator_constraints`
6. V006: `configurator_bom_mappings`
7. V007: `configurator_pricing_mappings`
8. V008: `configurator_valid_configurations`
9. V009: `configuration_sessions`
10. V010: Triggers for `updated_at`
