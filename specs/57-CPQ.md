# 57 - Configure-Price-Quote Service Specification

## 1. Domain Overview

Configure-Price-Quote (CPQ) provides a rules-based product configuration engine, multi-level pricing calculation, quote lifecycle management with approval workflows, proposal document generation, and order conversion. The CPQ engine validates product configurations against defined constraints (compatibility, incompatibility, cardinality), calculates prices using configuration-dependent price rules, manages multi-level approval chains, and converts approved quotes into sales orders. Integrates with OM for order creation, PRICING for price rule resolution, MFG for configured BOM generation, and INV for real-time availability checks.

**Bounded Context:** Configure-Price-Quote & Proposal Management
**Service Name:** `cpq-service`
**Database:** `data/cpq.db`
**HTTP Port:** 8089 | **gRPC Port:** 9089

---

## 2. Database Schema

### 2.1 CPQ Product Models
```sql
CREATE TABLE cpq_product_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    base_product_id TEXT NOT NULL,                -- FK to Inventory item
    model_type TEXT NOT NULL
        CHECK(model_type IN ('STANDALONE','CONFIGURABLE','BUNDLE','KIT')),
    configuration_type TEXT NOT NULL DEFAULT 'RULES_BASED'
        CHECK(configuration_type IN ('RULES_BASED','CONSTRAINT_BASED','ATTRIBUTE_BASED')),
    base_price_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    min_quantity INTEGER NOT NULL DEFAULT 1,
    max_quantity INTEGER NOT NULL DEFAULT 999999,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','OBSOLETE')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_cpq_models_tenant_type ON cpq_product_models(tenant_id, model_type);
CREATE INDEX idx_cpq_models_tenant_status ON cpq_product_models(tenant_id, status);
CREATE INDEX idx_cpq_models_tenant_product ON cpq_product_models(tenant_id, base_product_id);
CREATE INDEX idx_cpq_models_tenant_dates ON cpq_product_models(tenant_id, effective_from, effective_to);
CREATE INDEX idx_cpq_models_tenant_active ON cpq_product_models(tenant_id, is_active);
```

### 2.2 Configuration Rules
```sql
CREATE TABLE configuration_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('COMPATIBILITY','INCOMPATIBILITY','REQUIREMENT','EXCLUSION',
                            'RECOMMENDATION','CARDINALITY','DEFAULT_VALUE')),
    priority INTEGER NOT NULL DEFAULT 100,
    condition_expression TEXT NOT NULL,            -- JSON: condition definition
    action_expression TEXT NOT NULL,               -- JSON: action when condition met
    message TEXT,                                  -- User-facing message
    is_blocking INTEGER NOT NULL DEFAULT 1,        -- 1 = error, 0 = warning
    effective_from TEXT,
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES cpq_product_models(id),
    UNIQUE(tenant_id, model_id, rule_code)
);

CREATE INDEX idx_config_rules_tenant_model ON configuration_rules(tenant_id, model_id);
CREATE INDEX idx_config_rules_tenant_type ON configuration_rules(tenant_id, rule_type);
CREATE INDEX idx_config_rules_tenant_priority ON configuration_rules(tenant_id, priority);
CREATE INDEX idx_config_rules_tenant_active ON configuration_rules(tenant_id, is_active);
```

### 2.3 Configuration Attributes
```sql
CREATE TABLE configuration_attributes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    attribute_name TEXT NOT NULL,
    attribute_code TEXT NOT NULL,
    attribute_type TEXT NOT NULL
        CHECK(attribute_type IN ('TEXT','NUMBER','BOOLEAN','SELECT','MULTI_SELECT',
                                 'COLOR','DIMENSION','DATE')),
    data_type TEXT NOT NULL DEFAULT 'TEXT'
        CHECK(data_type IN ('TEXT','INTEGER','DECIMAL','BOOLEAN','DATE')),
    is_required INTEGER NOT NULL DEFAULT 0,
    default_value TEXT,
    validation_regex TEXT,
    min_value TEXT,
    max_value TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    group_name TEXT,                               -- Attribute grouping for UI
    is_configurable INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES cpq_product_models(id),
    UNIQUE(tenant_id, model_id, attribute_code)
);

CREATE INDEX idx_config_attrs_tenant_model ON configuration_attributes(tenant_id, model_id);
CREATE INDEX idx_config_attrs_tenant_type ON configuration_attributes(tenant_id, attribute_type);
CREATE INDEX idx_config_attrs_tenant_group ON configuration_attributes(tenant_id, group_name);
CREATE INDEX idx_config_attrs_tenant_active ON configuration_attributes(tenant_id, is_active);
```

### 2.4 Configuration Options
```sql
CREATE TABLE configuration_options (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    attribute_id TEXT NOT NULL,
    option_code TEXT NOT NULL,
    option_label TEXT NOT NULL,
    option_value TEXT NOT NULL,
    description TEXT,
    price_impact_cents INTEGER NOT NULL DEFAULT 0, -- Price delta for this option
    price_impact_type TEXT NOT NULL DEFAULT 'ABSOLUTE'
        CHECK(price_impact_type IN ('ABSOLUTE','PERCENTAGE','MULTIPLIER')),
    is_default INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    image_url TEXT,
    lead_time_delta_days INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (attribute_id) REFERENCES configuration_attributes(id),
    UNIQUE(tenant_id, attribute_id, option_code)
);

CREATE INDEX idx_config_options_tenant_attr ON configuration_options(tenant_id, attribute_id);
CREATE INDEX idx_config_options_tenant_default ON configuration_options(tenant_id, is_default);
CREATE INDEX idx_config_options_tenant_active ON configuration_options(tenant_id, is_active);
```

### 2.5 Price Rules
```sql
CREATE TABLE price_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('CONFIGURATION_BASED','QUANTITY_TIER','CUSTOMER_BASED',
                            'VOLUME_DISCOUNT','SURCHARGE','BUNDLE_DISCOUNT')),
    condition_expression TEXT NOT NULL,            -- JSON: when this rule applies
    price_adjustment_cents INTEGER NOT NULL DEFAULT 0,
    adjustment_type TEXT NOT NULL DEFAULT 'ABSOLUTE'
        CHECK(adjustment_type IN ('ABSOLUTE','PERCENTAGE','OVERRIDE')),
    priority INTEGER NOT NULL DEFAULT 100,
    stacking_behavior TEXT NOT NULL DEFAULT 'ADDITIVE'
        CHECK(stacking_behavior IN ('ADDITIVE','MAX_ONLY','EXCLUSIVE')),
    effective_from TEXT,
    effective_to TEXT,
    max_applications INTEGER NOT NULL DEFAULT 0,  -- 0 = unlimited

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES cpq_product_models(id),
    UNIQUE(tenant_id, model_id, rule_code)
);

CREATE INDEX idx_price_rules_tenant_model ON price_rules(tenant_id, model_id);
CREATE INDEX idx_price_rules_tenant_type ON price_rules(tenant_id, rule_type);
CREATE INDEX idx_price_rules_tenant_priority ON price_rules(tenant_id, priority);
CREATE INDEX idx_price_rules_tenant_dates ON price_rules(tenant_id, effective_from, effective_to);
CREATE INDEX idx_price_rules_tenant_active ON price_rules(tenant_id, is_active);
```

### 2.6 Quotes
```sql
CREATE TABLE quotes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    quote_number TEXT NOT NULL,                    -- QUOTE-2024-00001
    customer_id TEXT NOT NULL,
    opportunity_id TEXT,                           -- Optional CRM opportunity link
    sales_rep_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','CONFIGURING','PRICED','PENDING_APPROVAL','APPROVED',
                         'REJECTED','PRESENTED','ACCEPTED','CONVERTED','EXPIRED','CANCELLED')),
    valid_until TEXT NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    payment_terms TEXT,
    delivery_terms TEXT,
    notes TEXT,
    internal_notes TEXT,                           -- Internal-only notes
    probability_percent REAL NOT NULL DEFAULT 50.0,
    conversion_order_id TEXT,                      -- Set when quote converted to order

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, quote_number)
);

CREATE INDEX idx_quotes_tenant_customer ON quotes(tenant_id, customer_id);
CREATE INDEX idx_quotes_tenant_rep ON quotes(tenant_id, sales_rep_id);
CREATE INDEX idx_quotes_tenant_status ON quotes(tenant_id, status);
CREATE INDEX idx_quotes_tenant_dates ON quotes(tenant_id, created_at, valid_until);
CREATE INDEX idx_quotes_tenant_active ON quotes(tenant_id, is_active);
```

### 2.7 Quote Lines
```sql
CREATE TABLE quote_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    quote_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    model_id TEXT,
    product_id TEXT NOT NULL,                     -- Base product or configured product
    parent_line_id TEXT,                           -- For bundle hierarchy
    line_type TEXT NOT NULL
        CHECK(line_type IN ('PRODUCT','SERVICE','BUNDLE','CONFIGURATION','DISCOUNT','CHARGE')),
    description TEXT NOT NULL,
    configuration_json TEXT,                       -- JSON: stored configuration snapshot
    quantity INTEGER NOT NULL DEFAULT 1,
    unit_price_cents INTEGER NOT NULL,
    list_price_cents INTEGER NOT NULL,             -- Original list price
    discount_percent REAL NOT NULL DEFAULT 0.0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    subtotal_cents INTEGER NOT NULL,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    delivery_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (quote_id) REFERENCES quotes(id),
    FOREIGN KEY (parent_line_id) REFERENCES quote_lines(id)
);

CREATE INDEX idx_quote_lines_tenant_quote ON quote_lines(tenant_id, quote_id);
CREATE INDEX idx_quote_lines_tenant_product ON quote_lines(tenant_id, product_id);
CREATE INDEX idx_quote_lines_tenant_type ON quote_lines(tenant_id, line_type);
CREATE INDEX idx_quote_lines_tenant_parent ON quote_lines(tenant_id, parent_line_id);
```

### 2.8 Quote Pricing
```sql
CREATE TABLE quote_pricing (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    quote_id TEXT NOT NULL,
    quote_line_id TEXT NOT NULL,
    price_rule_id TEXT,                            -- Applied price rule
    price_source TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(price_source IN ('PRICE_LIST','PRICE_RULE','MANUAL','NEGOTIATED','DISCOUNT_SCHEDULE')),
    list_price_cents INTEGER NOT NULL,
    applied_price_cents INTEGER NOT NULL,
    adjustment_cents INTEGER NOT NULL DEFAULT 0,
    adjustment_reason TEXT,
    is_override INTEGER NOT NULL DEFAULT 0,        -- Manual price override
    override_approved_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (quote_id) REFERENCES quotes(id),
    FOREIGN KEY (quote_line_id) REFERENCES quote_lines(id)
);

CREATE INDEX idx_quote_pricing_tenant_quote ON quote_pricing(tenant_id, quote_id);
CREATE INDEX idx_quote_pricing_tenant_line ON quote_pricing(tenant_id, quote_line_id);
CREATE INDEX idx_quote_pricing_tenant_source ON quote_pricing(tenant_id, price_source);
CREATE INDEX idx_quote_pricing_tenant_override ON quote_pricing(tenant_id, is_override);
```

### 2.9 Quote Approvals
```sql
CREATE TABLE quote_approvals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    quote_id TEXT NOT NULL,
    approval_level INTEGER NOT NULL DEFAULT 1,
    approver_id TEXT NOT NULL,
    approval_type TEXT NOT NULL
        CHECK(approval_type IN ('DISCOUNT','PRICING','TERMS','CREDIT','MANUAL')),
    threshold_value_cents INTEGER,                 -- Approval threshold that triggered
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','EXPIRED')),
    comments TEXT,
    submitted_at TEXT NOT NULL DEFAULT (datetime('now')),
    actioned_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (quote_id) REFERENCES quotes(id)
);

CREATE INDEX idx_quote_approvals_tenant_quote ON quote_approvals(tenant_id, quote_id);
CREATE INDEX idx_quote_approvals_tenant_approver ON quote_approvals(tenant_id, approver_id);
CREATE INDEX idx_quote_approvals_tenant_status ON quote_approvals(tenant_id, status);
CREATE INDEX idx_quote_approvals_tenant_type ON quote_approvals(tenant_id, approval_type);
```

### 2.10 Quote Templates
```sql
CREATE TABLE quote_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_name TEXT NOT NULL,
    template_code TEXT NOT NULL,
    description TEXT,
    document_format TEXT NOT NULL DEFAULT 'PDF'
        CHECK(document_format IN ('PDF','DOCX','HTML')),
    template_content TEXT NOT NULL,                -- Template markup (HTML/Markdown)
    header_image_url TEXT,
    footer_text TEXT,
    include_pricing INTEGER NOT NULL DEFAULT 1,
    include_terms INTEGER NOT NULL DEFAULT 1,
    include_configuration INTEGER NOT NULL DEFAULT 1,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_quote_templates_tenant_active ON quote_templates(tenant_id, is_active);
CREATE INDEX idx_quote_templates_tenant_default ON quote_templates(tenant_id, is_default);
```

---

## 3. REST API Endpoints

```
# Product Models
GET/POST       /api/v1/cpq/models                          Permission: cpq.models.manage
GET/PUT        /api/v1/cpq/models/{id}
PATCH          /api/v1/cpq/models/{id}/status              Permission: cpq.models.manage

# Configuration Attributes
GET/POST       /api/v1/cpq/models/{id}/attributes          Permission: cpq.models.manage
GET/PUT        /api/v1/cpq/attributes/{id}
DELETE         /api/v1/cpq/attributes/{id}                 Permission: cpq.models.manage

# Configuration Options
GET/POST       /api/v1/cpq/attributes/{id}/options         Permission: cpq.models.manage
GET/PUT        /api/v1/cpq/options/{id}
DELETE         /api/v1/cpq/options/{id}                    Permission: cpq.models.manage

# Configuration Rules
GET/POST       /api/v1/cpq/models/{id}/rules               Permission: cpq.rules.manage
GET/PUT        /api/v1/cpq/rules/{id}
DELETE         /api/v1/cpq/rules/{id}                      Permission: cpq.rules.manage

# Price Rules
GET/POST       /api/v1/cpq/models/{id}/price-rules         Permission: cpq.pricing.manage
GET/PUT        /api/v1/cpq/price-rules/{id}
DELETE         /api/v1/cpq/price-rules/{id}                Permission: cpq.pricing.manage

# Configuration Session
POST           /api/v1/cpq/configure/start                  Permission: cpq.configure
POST           /api/v1/cpq/configure/{session_id}/select    Permission: cpq.configure
POST           /api/v1/cpq/configure/{session_id}/validate  Permission: cpq.configure
GET            /api/v1/cpq/configure/{session_id}/valid-options
GET            /api/v1/cpq/configure/{session_id}/summary
DELETE         /api/v1/cpq/configure/{session_id}

# Quotes
GET/POST       /api/v1/cpq/quotes                          Permission: cpq.quotes.manage
GET/PUT        /api/v1/cpq/quotes/{id}
PATCH          /api/v1/cpq/quotes/{id}/status              Permission: cpq.quotes.manage

# Quote Lines
GET/POST       /api/v1/cpq/quotes/{id}/lines               Permission: cpq.quotes.manage
GET/PUT        /api/v1/cpq/quote-lines/{id}
DELETE         /api/v1/cpq/quote-lines/{id}                Permission: cpq.quotes.manage

# Quote Pricing
POST           /api/v1/cpq/quotes/{id}/calculate-pricing   Permission: cpq.quotes.manage
GET            /api/v1/cpq/quotes/{id}/pricing
PUT            /api/v1/cpq/quotes/{id}/lines/{line_id}/override-price  Permission: cpq.pricing.override

# Quote Approval
POST           /api/v1/cpq/quotes/{id}/submit              Permission: cpq.quotes.manage
POST           /api/v1/cpq/approvals/{id}/approve           Permission: cpq.approvals.manage
POST           /api/v1/cpq/approvals/{id}/reject            Permission: cpq.approvals.manage
GET            /api/v1/cpq/approvals/pending                 Permission: cpq.approvals.manage

# Proposal Generation
POST           /api/v1/cpq/quotes/{id}/generate-proposal    Permission: cpq.quotes.manage
GET            /api/v1/cpq/quotes/{id}/proposals

# Order Conversion
POST           /api/v1/cpq/quotes/{id}/convert-to-order     Permission: cpq.quotes.convert

# Templates
GET/POST       /api/v1/cpq/templates                       Permission: cpq.templates.manage
GET/PUT        /api/v1/cpq/templates/{id}
DELETE         /api/v1/cpq/templates/{id}                  Permission: cpq.templates.manage

# Search & Reports
GET            /api/v1/cpq/quotes/search
GET            /api/v1/cpq/reports/pipeline
GET            /api/v1/cpq/reports/win-rate
GET            /api/v1/cpq/reports/average-discount
```

---

## 4. Business Rules

### 4.1 Product Configuration
```
Configuration validation rules:
  1. A configuration session MUST reference exactly one active product model
  2. All required attributes MUST have values before a configuration is considered complete
  3. Incompatibility rules (INCOMPATIBILITY) MUST prevent conflicting options from being selected together
  4. Compatibility rules (COMPATIBILITY) MUST require that if option A is selected, option B is also available
  5. Requirement rules (REQUIREMENT) MUST enforce that selecting option A requires selecting option B
  6. Exclusion rules (EXCLUSION) MUST prevent selecting option A when option B is already selected
  7. Blocking rules (is_blocking = 1) MUST prevent quote submission until resolved
  8. Non-blocking rules (is_blocking = 0) SHOULD display warnings but allow submission
  9. Default values are applied when the configuration session starts
  10. Cardinality rules enforce min/max selections for multi-select attributes
```

### 4.2 Pricing Calculation
- Base price is derived from the product model's base_price_cents
- Configuration options apply price_impact_cents (absolute, percentage, or multiplier)
- Price rules are evaluated in priority order (lower number = higher priority)
- EXCLUSIVE price rules replace all lower-priority rules
- MAX_ONLY stacking applies only the highest-value rule
- ADDITIVE stacking sums all applicable rule adjustments
- Manual price overrides require approval based on threshold configuration
- Pricing is recalculated when configuration changes or quantity changes

### 4.3 Quote Management
- Quote numbers are auto-generated in format QUOTE-{YEAR}-{SEQ}
- Each quote MUST have at least one line item before submission
- Quote valid_until MUST be a future date at the time of creation
- Expired quotes (past valid_until) MUST NOT be editable
- Quotes in DRAFT status can be freely modified
- Quote status progression: DRAFT -> CONFIGURING -> PRICED -> PENDING_APPROVAL -> APPROVED -> PRESENTED -> ACCEPTED/REJECTED -> CONVERTED
- Cancelled quotes cannot be reactivated

### 4.4 Approval Workflow
- Approval is required when: discount exceeds threshold, total exceeds threshold, payment terms deviation, credit risk
- Multi-level approvals are processed sequentially by approval_level
- All approvals at a given level MUST be obtained before advancing to the next level
- Approval requests expire after 7 days if no action is taken
- Rejection at any level stops the approval chain
- Comments are mandatory when rejecting a quote

### 4.5 Proposal Generation
- Proposal documents are generated from templates with quote data merged
- Templates support conditional sections (pricing, terms, configuration details)
- Generated proposals are stored as documents in the Document service
- Proposal format determined by template document_format setting
- A quote can have multiple proposal versions (e.g., revised after negotiation)

### 4.6 Order Conversion
- Only APPROVED or ACCEPTED quotes can be converted to orders
- Conversion creates a sales order in Order Management with all quote line details
- Configuration JSON is passed to OM for manufacturing BOM generation
- Quote status changes to CONVERTED after successful order creation
- Converted quotes cannot be modified but can be cloned for revision
- The conversion_order_id links the quote to the resulting order

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `cpq.quote.created` | New quote created | Notification, Reporting |
| `cpq.configuration.validated` | Configuration validation completed | MFG |
| `cpq.quote.priced` | Pricing calculation completed | Reporting |
| `cpq.quote.submitted` | Quote submitted for approval | Workflow |
| `cpq.quote.approved` | All approvals obtained | Notification |
| `cpq.quote.rejected` | Quote rejected by approver | Notification |
| `cpq.quote.converted` | Quote converted to order | OM, MFG |
| `cpq.quote.expired` | Quote past valid_until date | Notification |
| `cpq.price.override` | Manual price override applied | Workflow, Reporting |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.cpq.v1;

service CPQService {
    rpc CreateQuote(CreateQuoteRequest) returns (CreateQuoteResponse);
    rpc GetQuote(GetQuoteRequest) returns (GetQuoteResponse);
    rpc StartConfiguration(StartConfigurationRequest) returns (StartConfigurationResponse);
    rpc SelectOption(SelectOptionRequest) returns (SelectOptionResponse);
    rpc ValidateConfiguration(ValidateConfigurationRequest) returns (ValidateConfigurationResponse);
    rpc GetValidOptions(GetValidOptionsRequest) returns (GetValidOptionsResponse);
    rpc CalculatePricing(CalculatePricingRequest) returns (CalculatePricingResponse);
    rpc GenerateProposal(GenerateProposalRequest) returns (GenerateProposalResponse);
    rpc ConvertToOrder(ConvertToOrderRequest) returns (ConvertToOrderResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **PRICING (Advanced Pricing):** Base price lists, discount schedules, customer-specific pricing agreements
- **INV (Inventory):** Real-time product availability, ATP (Available-to-Promise) dates
- **MFG (Manufacturing):** Configured BOM templates, manufacturing lead times, routing information
- **Auth:** Sales representative assignments, approval authority mappings, role-based access
- **Document:** Proposal document storage, template rendering

### 6.2 Data Published To
- **OM (Order Management):** Sales order creation from quote conversion, configured product specifications
- **MFG (Manufacturing):** Configured BOM generation requests for made-to-order products
- **PRICING (Advanced Pricing):** Pricing feedback for negotiated prices and override patterns
- **INV (Inventory):** Reservation requests for quote line items (soft allocation)
- **Workflow:** Approval routing for quote discounts, price overrides, credit checks
- **Notification:** Quote status change alerts, approval request notifications, expiration warnings
- **Reporting:** Quote pipeline metrics, win/loss analytics, discount analysis

---

## 7. Migrations

1. V001: `cpq_product_models`
2. V002: `configuration_rules`
3. V003: `configuration_attributes`
4. V004: `configuration_options`
5. V005: `price_rules`
6. V006: `quotes`
7. V007: `quote_lines`
8. V008: `quote_pricing`
9. V009: `quote_approvals`
10. V010: `quote_templates`
11. V011: Triggers for `updated_at`
