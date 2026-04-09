# 187 - Calculation Manager Service Specification

## 1. Domain Overview

Calculation Manager provides centralized business rule calculation and formula execution management covering multi-type rule definitions (formula, allocation, aggregation, validation, and currency conversion), rule set orchestration with sequential, parallel, and conditional execution modes, dimension mapping for allocation and aggregation across organizational hierarchies, detailed execution logging with input/output capture and performance timing, and reusable rule templates for standardized calculation patterns. The system serves as the computational engine for financial consolidations, cost allocations, revenue recognitions, and data validation across enterprise modules. Integrates with GL for posting calculations, FA for asset depreciation, and Revenue Management for revenue allocation.

**Bounded Context:** Business Rule Calculation & Formula Engine
**Service Name:** `calculation-manager-service`
**Database:** `data/calculation_manager.db`
**HTTP Port:** 8205 | **gRPC Port:** 9205

---

## 2. Database Schema

### 2.1 Calculation Rules
```sql
CREATE TABLE calc_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    description TEXT,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('FORMULA','ALLOCATION','AGGREGATION','VALIDATION','CURRENCY_CONVERSION')),
    formula_expression TEXT NOT NULL,                -- Expression string or allocation definition
    input_parameters TEXT NOT NULL,                  -- JSON: array of parameter definitions (name, type, source, default)
    output_specification TEXT NOT NULL,              -- JSON: output field name, type, precision, rounding
    dimension_context TEXT,                          -- JSON: applicable dimension filters (entity, cost center, product)
    currency_code TEXT,
    precision_digits INTEGER NOT NULL DEFAULT 2
        CHECK(precision_digits >= 0 AND precision_digits <= 10),
    rounding_mode TEXT NOT NULL DEFAULT 'HALF_UP'
        CHECK(rounding_mode IN ('HALF_UP','HALF_DOWN','CEILING','FLOOR','UP','DOWN')),
    rule_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(rule_status IN ('DRAFT','ACTIVE','INACTIVE','DEPRECATED')),
    effective_start TEXT,
    effective_end TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_calc_rules_tenant_type ON calc_rules(tenant_id, rule_type);
CREATE INDEX idx_calc_rules_tenant_status ON calc_rules(tenant_id, rule_status);
CREATE INDEX idx_calc_rules_tenant_currency ON calc_rules(tenant_id, currency_code);
CREATE INDEX idx_calc_rules_tenant_dates ON calc_rules(tenant_id, effective_start, effective_end);
CREATE INDEX idx_calc_rules_tenant_active ON calc_rules(tenant_id, is_active);
```

### 2.2 Calculation Rule Sets
```sql
CREATE TABLE calc_rule_sets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_set_code TEXT NOT NULL,
    rule_set_name TEXT NOT NULL,
    description TEXT,
    execution_mode TEXT NOT NULL DEFAULT 'SEQUENTIAL'
        CHECK(execution_mode IN ('SEQUENTIAL','PARALLEL','CONDITIONAL')),
    rule_sequence TEXT NOT NULL,                     -- JSON: ordered array of rule_code with optional conditions
    input_mapping TEXT,                              -- JSON: mapping of external inputs to rule parameters
    output_aggregation TEXT,                         -- JSON: how outputs from multiple rules are combined
    stop_on_error INTEGER NOT NULL DEFAULT 1,
    max_execution_time_ms INTEGER NOT NULL DEFAULT 30000
        CHECK(max_execution_time_ms > 0),
    set_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(set_status IN ('DRAFT','ACTIVE','INACTIVE','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_set_code)
);

CREATE INDEX idx_calc_rule_sets_tenant_status ON calc_rule_sets(tenant_id, set_status);
CREATE INDEX idx_calc_rule_sets_tenant_mode ON calc_rule_sets(tenant_id, execution_mode);
CREATE INDEX idx_calc_rule_sets_tenant_active ON calc_rule_sets(tenant_id, is_active);
```

### 2.3 Dimension Mappings
```sql
CREATE TABLE calc_dimension_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mapping_name TEXT NOT NULL,
    source_dimension TEXT NOT NULL,                  -- Source dimension (e.g., COST_CENTER)
    target_dimension TEXT NOT NULL,                  -- Target dimension (e.g., DEPARTMENT)
    allocation_method TEXT NOT NULL
        CHECK(allocation_method IN ('PROPORTIONAL','EQUAL','WEIGHTED','HIERARCHICAL','MANUAL')),
    mapping_rules TEXT NOT NULL,                     -- JSON: array of source-to-target mappings with weights/percentages
    basis_driver TEXT,                               -- Allocation basis (e.g., HEADCOUNT, REVENUE, SQUARE_FEET)
    effective_date TEXT NOT NULL,
    expiry_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, mapping_name, source_dimension, target_dimension)
);

CREATE INDEX idx_dim_mappings_tenant_source ON calc_dimension_mappings(tenant_id, source_dimension);
CREATE INDEX idx_dim_mappings_tenant_target ON calc_dimension_mappings(tenant_id, target_dimension);
CREATE INDEX idx_dim_mappings_tenant_method ON calc_dimension_mappings(tenant_id, allocation_method);
CREATE INDEX idx_dim_mappings_tenant_dates ON calc_dimension_mappings(tenant_id, effective_date, expiry_date);
CREATE INDEX idx_dim_mappings_tenant_active ON calc_dimension_mappings(tenant_id, is_active);
```

### 2.4 Execution Logs
```sql
CREATE TABLE calc_execution_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    execution_id TEXT NOT NULL,
    rule_id TEXT,
    rule_set_id TEXT,
    execution_type TEXT NOT NULL
        CHECK(execution_type IN ('SINGLE_RULE','RULE_SET','TEMPLATE')),
    inputs TEXT NOT NULL,                            -- JSON: snapshot of all input parameter values
    outputs TEXT,                                    -- JSON: calculated output values
    status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(status IN ('RUNNING','COMPLETED','FAILED','TIMEOUT','CANCELLED')),
    error_message TEXT,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    execution_time_ms INTEGER NOT NULL DEFAULT 0,
    triggered_by TEXT NOT NULL,
    period_context TEXT,                             -- JSON: fiscal period, scenario, ledger context

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, execution_id)
);

CREATE INDEX idx_exec_logs_tenant_rule ON calc_execution_logs(tenant_id, rule_id);
CREATE INDEX idx_exec_logs_tenant_set ON calc_execution_logs(tenant_id, rule_set_id);
CREATE INDEX idx_exec_logs_tenant_status ON calc_execution_logs(tenant_id, status);
CREATE INDEX idx_exec_logs_tenant_started ON calc_execution_logs(tenant_id, started_at);
CREATE INDEX idx_exec_logs_tenant_type ON calc_execution_logs(tenant_id, execution_type);
CREATE INDEX idx_exec_logs_tenant_triggered ON calc_execution_logs(tenant_id, triggered_by);
```

### 2.5 Rule Templates
```sql
CREATE TABLE calc_rule_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    category TEXT NOT NULL
        CHECK(category IN ('DEPRECIATION','AMORTIZATION','TAX','ALLOCATION','CONSOLIDATION','REVENUE','VALIDATION','CUSTOM')),
    template_expression TEXT NOT NULL,               -- Parameterized formula template
    parameter_definitions TEXT NOT NULL,             -- JSON: array of template parameters with types and defaults
    output_specification TEXT NOT NULL,              -- JSON: expected output structure
    usage_count INTEGER NOT NULL DEFAULT 0,
    template_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(template_status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_rule_templates_tenant_category ON calc_rule_templates(tenant_id, category);
CREATE INDEX idx_rule_templates_tenant_status ON calc_rule_templates(tenant_id, template_status);
CREATE INDEX idx_rule_templates_tenant_active ON calc_rule_templates(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Calculation Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/calc/rules` | List calculation rules |
| POST | `/api/v1/calc/rules` | Create calculation rule |
| GET | `/api/v1/calc/rules/{id}` | Get rule details |
| PUT | `/api/v1/calc/rules/{id}` | Update rule |
| POST | `/api/v1/calc/rules/{id}/execute` | Execute a single rule |
| POST | `/api/v1/calc/rules/{id}/validate` | Validate rule expression |
| PATCH | `/api/v1/calc/rules/{id}/status` | Update rule status |

### 3.2 Rule Sets
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/calc/rule-sets` | List rule sets |
| POST | `/api/v1/calc/rule-sets` | Create rule set |
| GET | `/api/v1/calc/rule-sets/{id}` | Get rule set details |
| PUT | `/api/v1/calc/rule-sets/{id}` | Update rule set |
| POST | `/api/v1/calc/rule-sets/{id}/execute` | Execute a rule set |
| GET | `/api/v1/calc/rule-sets/{id}/execution-history` | Get execution history |

### 3.3 Dimension Mappings
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/calc/dimension-mappings` | List dimension mappings |
| POST | `/api/v1/calc/dimension-mappings` | Create dimension mapping |
| GET | `/api/v1/calc/dimension-mappings/{id}` | Get mapping details |
| PUT | `/api/v1/calc/dimension-mappings/{id}` | Update mapping |

### 3.4 Execution Logs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/calc/executions` | List execution logs |
| GET | `/api/v1/calc/executions/{id}` | Get execution details |
| POST | `/api/v1/calc/executions/{id}/cancel` | Cancel running execution |
| GET | `/api/v1/calc/executions/{id}/inputs` | Get execution inputs |
| GET | `/api/v1/calc/executions/{id}/outputs` | Get execution outputs |

### 3.5 Rule Templates
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/calc/templates` | List rule templates |
| POST | `/api/v1/calc/templates` | Create rule template |
| GET | `/api/v1/calc/templates/{id}` | Get template details |
| PUT | `/api/v1/calc/templates/{id}` | Update template |
| POST | `/api/v1/calc/templates/{id}/instantiate` | Create rule from template |

---

## 4. Business Rules

### 4.1 Calculation Rules
1. Each rule MUST have a unique rule_code within a tenant.
2. Formula expressions MUST be syntactically validated before a rule can be set to ACTIVE status.
3. Input parameters MUST define a type (TEXT, NUMERIC, BOOLEAN, DATE) and MAY specify a default value.
4. Currency conversion rules MUST specify a source currency_code and include exchange rate lookup logic.
5. Rules with effective_start and effective_end MUST NOT execute outside their effective date range.
6. Precision digits MUST be between 0 and 10; rounding_mode MUST be a valid IEEE 754 rounding mode.

### 4.2 Rule Sets
7. Rule set rule_sequence MUST reference only ACTIVE rules that exist in the tenant scope.
8. In SEQUENTIAL mode, rules MUST execute in array order; a failed rule MUST halt execution if stop_on_error is 1.
9. In PARALLEL mode, all rules execute simultaneously; outputs MUST be aggregated per output_aggregation specification.
10. In CONDITIONAL mode, each rule entry MUST include a condition expression evaluated before execution.
11. Rule set execution MUST time out and status MUST be set to TIMEOUT if max_execution_time_ms is exceeded.

### 4.3 Dimension Mappings
12. Mapping weights in PROPORTIONAL allocation MUST sum to 100 percent across all target entries.
13. HIERARCHICAL allocation MUST resolve parent-child relationships from the source dimension tree.
14. Mapping rules MUST contain at least one source-to-target entry with a non-zero weight.
15. An active mapping MUST NOT overlap in effective date range with another mapping of the same source-target pair.

### 4.4 Execution Logging
16. Every rule or rule set execution MUST create an execution log with a unique execution_id.
17. Inputs MUST be captured as a JSON snapshot before execution begins.
18. Outputs MUST be recorded only when the execution status is COMPLETED.
19. Execution time MUST be recorded in milliseconds from started_at to completed_at.
20. Failed executions MUST capture the error_message with sufficient detail for debugging.

### 4.5 Rule Templates
21. Template expressions MUST use parameterized placeholders (e.g., `{param_name}`) that match parameter_definitions.
22. Instantiating a rule from a template MUST validate all required parameters are provided.
23. The usage_count MUST be incremented each time a template is used to instantiate a new rule.
24. Template parameters MUST NOT be removed if existing derived rules reference them.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.calc.v1;

service CalculationManagerService {
    // Rules
    rpc CreateRule(CreateRuleRequest) returns (CreateRuleResponse);
    rpc GetRule(GetRuleRequest) returns (GetRuleResponse);
    rpc ListRules(ListRulesRequest) returns (ListRulesResponse);
    rpc UpdateRule(UpdateRuleRequest) returns (UpdateRuleResponse);
    rpc ExecuteRule(ExecuteRuleRequest) returns (ExecuteRuleResponse);
    rpc ValidateRuleExpression(ValidateRuleExpressionRequest) returns (ValidateRuleExpressionResponse);

    // Rule sets
    rpc CreateRuleSet(CreateRuleSetRequest) returns (CreateRuleSetResponse);
    rpc GetRuleSet(GetRuleSetRequest) returns (GetRuleSetResponse);
    rpc ListRuleSets(ListRuleSetsRequest) returns (ListRuleSetsResponse);
    rpc ExecuteRuleSet(ExecuteRuleSetRequest) returns (ExecuteRuleSetResponse);

    // Dimension mappings
    rpc CreateDimensionMapping(CreateDimensionMappingRequest) returns (CreateDimensionMappingResponse);
    rpc GetDimensionMapping(GetDimensionMappingRequest) returns (GetDimensionMappingResponse);
    rpc ListDimensionMappings(ListDimensionMappingsRequest) returns (ListDimensionMappingsResponse);

    // Execution logs
    rpc GetExecutionLog(GetExecutionLogRequest) returns (GetExecutionLogResponse);
    rpc ListExecutionLogs(ListExecutionLogsRequest) returns (ListExecutionLogsResponse);
    rpc CancelExecution(CancelExecutionRequest) returns (CancelExecutionResponse);

    // Templates
    rpc CreateTemplate(CreateTemplateRequest) returns (CreateTemplateResponse);
    rpc GetTemplate(GetTemplateRequest) returns (GetTemplateResponse);
    rpc ListTemplates(ListTemplatesRequest) returns (ListTemplatesResponse);
    rpc InstantiateTemplate(InstantiateTemplateRequest) returns (InstantiateTemplateResponse);
}

message CreateRuleRequest {
    string tenant_id = 1;
    string rule_code = 2;
    string rule_name = 3;
    string description = 4;
    string rule_type = 5;
    string formula_expression = 6;
    string input_parameters = 7;
    string output_specification = 8;
    string dimension_context = 9;
    string currency_code = 10;
    int32 precision_digits = 11;
    string rounding_mode = 12;
    string effective_start = 13;
    string effective_end = 14;
}

message CreateRuleResponse {
    string rule_id = 1;
    string rule_code = 2;
    string rule_status = 3;
}

message GetRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message GetRuleResponse {
    string rule_id = 1;
    string rule_code = 2;
    string rule_name = 3;
    string rule_type = 4;
    string formula_expression = 5;
    string input_parameters = 6;
    string output_specification = 7;
    string dimension_context = 8;
    string currency_code = 9;
    int32 precision_digits = 10;
    string rounding_mode = 11;
    string rule_status = 12;
    int32 version = 13;
}

message ListRulesRequest {
    string tenant_id = 1;
    string rule_type = 2;
    string rule_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListRulesResponse {
    repeated GetRuleResponse rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string rule_name = 3;
    string formula_expression = 4;
    string input_parameters = 5;
    string output_specification = 6;
    string rule_status = 7;
}

message UpdateRuleResponse {
    string rule_id = 1;
    string rule_status = 2;
    int32 version = 3;
}

message ExecuteRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string inputs = 3;
    string period_context = 4;
}

message ExecuteRuleResponse {
    string execution_id = 1;
    string outputs = 2;
    string status = 3;
    int32 execution_time_ms = 4;
}

message ValidateRuleExpressionRequest {
    string tenant_id = 1;
    string formula_expression = 2;
    string input_parameters = 3;
}

message ValidateRuleExpressionResponse {
    bool is_valid = 1;
    repeated string errors = 2;
    repeated string warnings = 3;
}

message CreateRuleSetRequest {
    string tenant_id = 1;
    string rule_set_code = 2;
    string rule_set_name = 3;
    string description = 4;
    string execution_mode = 5;
    string rule_sequence = 6;
    string input_mapping = 7;
    string output_aggregation = 8;
    bool stop_on_error = 9;
    int32 max_execution_time_ms = 10;
}

message CreateRuleSetResponse {
    string rule_set_id = 1;
    string rule_set_code = 2;
    string set_status = 3;
}

message GetRuleSetRequest {
    string tenant_id = 1;
    string rule_set_id = 2;
}

message GetRuleSetResponse {
    string rule_set_id = 1;
    string rule_set_code = 2;
    string rule_set_name = 3;
    string execution_mode = 4;
    string rule_sequence = 5;
    string input_mapping = 6;
    string output_aggregation = 7;
    bool stop_on_error = 8;
    int32 max_execution_time_ms = 9;
    string set_status = 10;
    int32 version = 11;
}

message ListRuleSetsRequest {
    string tenant_id = 1;
    string set_status = 2;
    string execution_mode = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListRuleSetsResponse {
    repeated GetRuleSetResponse rule_sets = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ExecuteRuleSetRequest {
    string tenant_id = 1;
    string rule_set_id = 2;
    string inputs = 3;
    string period_context = 4;
}

message ExecuteRuleSetResponse {
    string execution_id = 1;
    string outputs = 2;
    string status = 3;
    int32 execution_time_ms = 4;
    int32 rules_executed = 5;
    int32 rules_failed = 6;
}

message CreateDimensionMappingRequest {
    string tenant_id = 1;
    string mapping_name = 2;
    string source_dimension = 3;
    string target_dimension = 4;
    string allocation_method = 5;
    string mapping_rules = 6;
    string basis_driver = 7;
    string effective_date = 8;
    string expiry_date = 9;
}

message CreateDimensionMappingResponse {
    string mapping_id = 1;
    string mapping_name = 2;
}

message GetDimensionMappingRequest {
    string tenant_id = 1;
    string mapping_id = 2;
}

message GetDimensionMappingResponse {
    string mapping_id = 1;
    string mapping_name = 2;
    string source_dimension = 3;
    string target_dimension = 4;
    string allocation_method = 5;
    string mapping_rules = 6;
    string basis_driver = 7;
    string effective_date = 8;
    string expiry_date = 9;
    int32 version = 10;
}

message ListDimensionMappingsRequest {
    string tenant_id = 1;
    string source_dimension = 2;
    string target_dimension = 3;
    string allocation_method = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListDimensionMappingsResponse {
    repeated GetDimensionMappingResponse mappings = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetExecutionLogRequest {
    string tenant_id = 1;
    string execution_id = 2;
}

message GetExecutionLogResponse {
    string id = 1;
    string execution_id = 2;
    string rule_id = 3;
    string rule_set_id = 4;
    string execution_type = 5;
    string inputs = 6;
    string outputs = 7;
    string status = 8;
    string error_message = 9;
    string started_at = 10;
    string completed_at = 11;
    int32 execution_time_ms = 12;
    string triggered_by = 13;
    string period_context = 14;
}

message ListExecutionLogsRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string rule_set_id = 3;
    string status = 4;
    string from_date = 5;
    string to_date = 6;
    int32 page_size = 7;
    string page_token = 8;
}

message ListExecutionLogsResponse {
    repeated GetExecutionLogResponse logs = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message CancelExecutionRequest {
    string tenant_id = 1;
    string execution_id = 2;
}

message CancelExecutionResponse {
    string execution_id = 1;
    string status = 2;
}

message CreateTemplateRequest {
    string tenant_id = 1;
    string template_code = 2;
    string template_name = 3;
    string description = 4;
    string category = 5;
    string template_expression = 6;
    string parameter_definitions = 7;
    string output_specification = 8;
}

message CreateTemplateResponse {
    string template_id = 1;
    string template_code = 2;
    string template_status = 3;
}

message GetTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
}

message GetTemplateResponse {
    string template_id = 1;
    string template_code = 2;
    string template_name = 3;
    string category = 4;
    string template_expression = 5;
    string parameter_definitions = 6;
    string output_specification = 7;
    int32 usage_count = 8;
    string template_status = 9;
}

message ListTemplatesRequest {
    string tenant_id = 1;
    string category = 2;
    string template_status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListTemplatesResponse {
    repeated GetTemplateResponse templates = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message InstantiateTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
    string rule_code = 3;
    string rule_name = 4;
    string parameter_values = 5;
}

message InstantiateTemplateResponse {
    string rule_id = 1;
    string rule_code = 2;
    string rule_status = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `gl-service` | Account structures, balance data, journal entry context for calculation inputs |
| `fa-service` | Asset cost, salvage values, useful life for depreciation calculations |
| `currency-service` | Exchange rates for currency conversion rules |
| `dimension-service` | Hierarchies and member lists for allocation and aggregation |
| `period-service` | Fiscal calendar, open periods, scenario definitions |
| `auth-service` | User identity for execution triggering and audit trail |

### Published To
| Service | Data |
|---------|------|
| `gl-service` | Calculated journal lines, allocation posting results |
| `reporting-service` | Execution statistics, rule performance metrics, calculation audit trail |
| `notification-service` | Execution failure alerts, timeout warnings, rule status change notifications |
| `audit-service` | Calculation execution logs, input/output snapshots for compliance |
| `workflow-service` | Approval routing for rule activation, escalation for execution failures |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `calc.rule.executed` | `calc.events` | `{ execution_id, rule_id, rule_code, rule_type, status, execution_time_ms, triggered_by }` | Calculation rule execution completed |
| `calc.rule.error` | `calc.events` | `{ execution_id, rule_id, rule_code, error_message, inputs }` | Calculation rule execution failed |
| `calc.set.completed` | `calc.events` | `{ execution_id, rule_set_id, rule_set_code, status, rules_executed, rules_failed, execution_time_ms }` | Rule set execution finished |

---

## 8. Migrations

1. V001: `calc_rules`
2. V002: `calc_rule_sets`
3. V003: `calc_dimension_mappings`
4. V004: `calc_execution_logs`
5. V005: `calc_rule_templates`
6. V006: Triggers for `updated_at`
