# 127 - Intelligent Advisor Specification

## 1. Domain Overview

Intelligent Advisor (formerly Oracle Policy Automation) provides rule-based policy automation and decision guides. It enables organizations to encode complex business rules, regulations, and policies into interactive decision trees, calculations, and interviews that produce consistent, auditable determinations. Used for eligibility checks, compliance assessments, benefits calculations, and guided interviews.

**Bounded Context:** Policy Automation & Rule-Based Decisioning
**Service Name:** `advisor-service`
**Database:** `data/advisor.db`
**HTTP Port:** 8207 | **gRPC Port:** 9207

---

## 2. Database Schema

### 2.1 Policy Models
```sql
CREATE TABLE ia_policy_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_code TEXT NOT NULL,
    description TEXT,
    version_number INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','DEPRECATED','ARCHIVED')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    goal TEXT NOT NULL,                  -- What the model determines (e.g., "eligibility", "tax_amount")
    input_schema TEXT NOT NULL,          -- JSON Schema for required inputs
    output_schema TEXT NOT NULL,         -- JSON Schema for determination outputs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code, version_number)
);
```

### 2.2 Rule Tables
```sql
CREATE TABLE ia_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('CONDITION','CALCULATION','VALIDATION','INFERENCE','DECISION_TABLE')),
    description TEXT,
    priority INTEGER NOT NULL DEFAULT 0,
    condition_expression TEXT NOT NULL,   -- Natural language or structured expression
    conclusion TEXT NOT NULL,            -- What this rule concludes
    rule_body TEXT NOT NULL,             -- JSON: full rule definition
    is_active INTEGER NOT NULL DEFAULT 1,
    dependencies TEXT,                   -- JSON array of rule_ids this depends on

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES ia_policy_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, rule_name)
);
```

### 2.3 Decision Tables
```sql
CREATE TABLE ia_decision_tables (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    table_name TEXT NOT NULL,
    description TEXT,
    input_columns TEXT NOT NULL,         -- JSON: [{ "name": "age", "type": "number" }, ...]
    output_columns TEXT NOT NULL,        -- JSON: [{ "name": "eligible", "type": "boolean" }, ...]
    rules_json TEXT NOT NULL,            -- JSON: array of { inputs: [...], outputs: [...] } rows
    default_outputs TEXT,                -- JSON: default output when no row matches

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES ia_policy_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, table_name)
);
```

### 2.4 Interview Definitions
```sql
CREATE TABLE ia_interviews (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    interview_name TEXT NOT NULL,
    description TEXT,
    screens TEXT NOT NULL,               -- JSON: ordered array of screen definitions
    branding TEXT,                       -- JSON: theme, logo, colors
    welcome_message TEXT,
    completion_message TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),
    locale TEXT NOT NULL DEFAULT 'en-US',

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (model_id) REFERENCES ia_policy_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, model_id, interview_name)
);
```

### 2.5 Screen Definitions
```sql
CREATE TABLE ia_screens (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    interview_id TEXT NOT NULL,
    screen_number INTEGER NOT NULL,
    screen_title TEXT NOT NULL,
    screen_type TEXT NOT NULL CHECK(screen_type IN ('QUESTION','INFORMATION','SUMMARY','RESULT','CALCULATION')),
    fields TEXT NOT NULL,                -- JSON: [{ "name": "income", "type": "currency", "label": "...", "required": true }]
    navigation_rule TEXT,                -- JSON: conditional next-screen logic
    help_text TEXT,
    is_optional INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (interview_id) REFERENCES ia_interviews(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, interview_id, screen_number)
);
```

### 2.6 Determinations (Execution Results)
```sql
CREATE TABLE ia_determinations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    interview_id TEXT,
    determination_type TEXT NOT NULL CHECK(determination_type IN ('INTERACTIVE','API','BATCH')),
    input_data TEXT NOT NULL,            -- JSON: provided inputs
    output_data TEXT NOT NULL,           -- JSON: determined outputs
    conclusions TEXT NOT NULL,           -- JSON: list of conclusions reached
    rules_fired TEXT NOT NULL,           -- JSON: list of rule_ids that fired
    decision_table_matches TEXT,         -- JSON: table rows that matched
    execution_time_ms INTEGER NOT NULL,
    is_conclusive INTEGER NOT NULL DEFAULT 1,
    session_id TEXT,                     -- Links to interview session
    user_id TEXT,
    reference_type TEXT,
    reference_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (model_id) REFERENCES ia_policy_models(id) ON DELETE RESTRICT
);

CREATE INDEX idx_ia_determinations_tenant_model ON ia_determinations(tenant_id, model_id, created_at DESC);
CREATE INDEX idx_ia_determinations_tenant_reference ON ia_determinations(tenant_id, reference_type, reference_id);
```

---

## 3. REST API Endpoints

### 3.1 Policy Models
```
GET    /api/v1/advisor/models                           Permission: ia.models.read
GET    /api/v1/advisor/models/{id}                      Permission: ia.models.read
POST   /api/v1/advisor/models                           Permission: ia.models.create
PUT    /api/v1/advisor/models/{id}                      Permission: ia.models.update
POST   /api/v1/advisor/models/{id}/activate             Permission: ia.models.activate
GET    /api/v1/advisor/models/{id}/rules                 Permission: ia.models.read
POST   /api/v1/advisor/models/{id}/rules                Permission: ia.rules.create
PUT    /api/v1/advisor/rules/{id}                       Permission: ia.rules.update
GET    /api/v1/advisor/models/{id}/tables                Permission: ia.models.read
POST   /api/v1/advisor/models/{id}/tables               Permission: ia.tables.create
```

### 3.2 Determinations (API Execution)
```
POST   /api/v1/advisor/determine                        Permission: ia.determine.execute
  Request: {
    "model_code": "BENEFIT_ELIGIBILITY",
    "input_data": { "age": 35, "income_cents": 7500000, "dependents": 2, "state": "CA" }
  }
  Response 200: {
    "data": {
      "determination_id": "...",
      "output_data": { "eligible": true, "benefit_amount_cents": 250000, "program": "MEDICAID" },
      "conclusions": ["income_below_threshold", "has_dependents", "state_qualified"],
      "rules_fired": ["rule_001", "rule_005", "rule_012"],
      "execution_time_ms": 12
    }
  }

POST   /api/v1/advisor/determine/batch                  Permission: ia.determine.batch
GET    /api/v1/advisor/determinations                    Permission: ia.determine.read
GET    /api/v1/advisor/determinations/{id}               Permission: ia.determine.read
```

### 3.3 Interviews
```
GET    /api/v1/advisor/interviews                        Permission: ia.interviews.read
POST   /api/v1/advisor/interviews                        Permission: ia.interviews.create
PUT    /api/v1/advisor/interviews/{id}                   Permission: ia.interviews.update
GET    /api/v1/advisor/interviews/{id}/screens           Permission: ia.interviews.read
POST   /api/v1/advisor/interviews/{id}/start             Permission: ia.interviews.start
  Response: First screen with fields
POST   /api/v1/advisor/interviews/{id}/next              Permission: ia.interviews.continue
  Request: { "session_id": "...", "screen_data": { "income": 75000 } }
  Response: Next screen or determination result
POST   /api/v1/advisor/interviews/{id}/back              Permission: ia.interviews.continue
GET    /api/v1/advisor/interviews/{id}/summary            Permission: ia.interviews.read
```

---

## 4. Business Rules

### 4.1 Rule Engine
- Rules evaluated in priority order (higher priority first)
- Rule conditions use structured expressions (not arbitrary code)
- Circular dependencies detected and rejected at model activation
- Rules can reference conclusions from other rules (forward chaining)
- All rule evaluations are deterministic for same inputs

### 4.2 Decision Tables
- First-match semantics: first row where all inputs match wins
- Default outputs applied when no row matches
- Decision tables can be referenced by rules
- Input ranges supported: "18-65", ">100", "<=50"

### 4.3 Interviews
- Screen flow determined by navigation rules (can be conditional)
- Screen visibility based on prior answers
- Progress tracking (% complete)
- Sessions timeout after configurable idle period
- Partial saves allow resuming later
- Interview results produce a determination

### 4.4 Audit & Compliance
- Every determination stored with full input/output/rules-fired audit trail
- Determinations are immutable once created
- Model version locked at determination time for reproducibility
- Change history maintained for all rule modifications

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.advisor.v1;

service IntelligentAdvisorService {
    rpc Determine(DetermineRequest) returns (DetermineResponse);
    rpc StartInterview(StartInterviewRequest) returns (StartInterviewResponse);
    rpc AdvanceInterview(AdvanceInterviewRequest) returns (AdvanceInterviewResponse);
    rpc GetModelSchema(GetModelSchemaRequest) returns (GetModelSchemaResponse);
    rpc ValidateModel(ValidateModelRequest) returns (ValidateModelResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **workflow-service**: Rule-driven workflow routing
- **hr-service**: Employee eligibility determinations
- **benefits-service**: Benefits eligibility and calculations
- **tax-service**: Tax calculation rules
- **customerservice-service**: Customer-facing guided interviews
- **ai-service**: Natural language rule authoring assistance

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `ia.model.activated` | Policy model activated | model_id, version |
| `ia.determination.created` | Determination executed | determination_id, model_id, outputs |
| `ia.determination.inconclusive` | No conclusive result | determination_id, model_id |
| `ia.interview.started` | Interview session started | interview_id, session_id |
| `ia.interview.completed` | Interview finished with result | session_id, determination_id |
| `ia.rule.violation` | Validation rule failed | model_id, rule_id, input_data |

---

## 7. Migrations

### Migration Order for advisor-service:
1. V001: `ia_policy_models`
2. V002: `ia_rules`
3. V003: `ia_decision_tables`
4. V004: `ia_interviews`
5. V005: `ia_screens`
6. V006: `ia_determinations`
7. V007: Triggers for `updated_at`
8. V008: Seed data (sample eligibility model, decision table template)
