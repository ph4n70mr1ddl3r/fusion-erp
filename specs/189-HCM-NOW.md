# 189 - HCM Now Service Specification

## 1. Domain Overview

HCM Now provides rapid HCM deployment and configuration management covering solution templates with industry and region presets for accelerated implementation, quick configuration tools for organizational structures and pay scales, guided setup task management with dependency tracking and progress monitoring, adoption metrics tracking across module activation and utilization rates, and best practice compliance rules scoped by country and industry. The system enables organizations to quickly stand up HCM capabilities with pre-built configurations and continuously monitor adoption to ensure successful rollout. Integrates with Core HR for org structures, Compensation for pay scales, and Reporting for adoption dashboards.

**Bounded Context:** Rapid HCM Deployment & Adoption Management
**Service Name:** `hcm-now-service`
**Database:** `data/hcm_now.db`
**HTTP Port:** 8207 | **gRPC Port:** 9207

---

## 2. Database Schema

### 2.1 Solution Templates
```sql
CREATE TABLE hcmnow_solution_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    description TEXT,
    industry TEXT NOT NULL,                          -- JSON: array of applicable industry codes
    region TEXT NOT NULL,                            -- JSON: array of applicable region/country codes
    hcm_modules TEXT NOT NULL,                       -- JSON: array of HCM module IDs included in the template
    default_configurations TEXT NOT NULL,            -- JSON: pre-set configuration values for each module
    estimated_setup_days INTEGER NOT NULL DEFAULT 30
        CHECK(estimated_setup_days > 0),
    complexity_level TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(complexity_level IN ('QUICK_START','STANDARD','ADVANCED','ENTERPRISE')),
    template_version TEXT NOT NULL DEFAULT '1.0',
    template_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(template_status IN ('DRAFT','PUBLISHED','DEPRECATED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_sol_templates_tenant_industry ON hcmnow_solution_templates(tenant_id, industry);
CREATE INDEX idx_sol_templates_tenant_region ON hcmnow_solution_templates(tenant_id, region);
CREATE INDEX idx_sol_templates_tenant_complexity ON hcmnow_solution_templates(tenant_id, complexity_level);
CREATE INDEX idx_sol_templates_tenant_status ON hcmnow_solution_templates(tenant_id, template_status);
CREATE INDEX idx_sol_templates_tenant_active ON hcmnow_solution_templates(tenant_id, is_active);
```

### 2.2 Quick Configurations
```sql
CREATE TABLE hcmnow_quick_configurations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    configuration_id TEXT NOT NULL,
    template_id TEXT,
    configuration_type TEXT NOT NULL
        CHECK(configuration_type IN ('ORG_STRUCTURE','PAY_SCALE','JOB_FAMILY','GRADE_STEP','LEAVE_PLAN','WORKING_HOURS','BENEFIT_PLAN')),
    configuration_name TEXT NOT NULL,
    configuration_data TEXT NOT NULL,                -- JSON: the configuration payload specific to the type
    source_template_code TEXT,
    applied_at TEXT,
    applied_by TEXT,
    configuration_status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(configuration_status IN ('DRAFT','VALIDATED','APPLIED','ROLLED_BACK')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES hcmnow_solution_templates(id),
    UNIQUE(tenant_id, configuration_id)
);

CREATE INDEX idx_quick_configs_tenant_type ON hcmnow_quick_configurations(tenant_id, configuration_type);
CREATE INDEX idx_quick_configs_tenant_status ON hcmnow_quick_configurations(tenant_id, configuration_status);
CREATE INDEX idx_quick_configs_tenant_template ON hcmnow_quick_configurations(tenant_id, template_id);
CREATE INDEX idx_quick_configs_tenant_active ON hcmnow_quick_configurations(tenant_id, is_active);
```

### 2.3 Setup Tasks
```sql
CREATE TABLE hcmnow_setup_tasks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    template_id TEXT,
    task_name TEXT NOT NULL,
    description TEXT,
    task_category TEXT NOT NULL
        CHECK(task_category IN ('CORE_HR','PAYROLL','BENEFITS','COMPENSATION','TIME_LABOR','RECRUITING','LEARNING','PERFORMANCE')),
    task_sequence INTEGER NOT NULL DEFAULT 0
        CHECK(task_sequence >= 0),
    depends_on TEXT,                                 -- JSON: array of task_ids that must complete first
    assigned_to TEXT,
    estimated_duration_minutes INTEGER NOT NULL DEFAULT 15
        CHECK(estimated_duration_minutes > 0),
    task_type TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(task_type IN ('MANUAL','AUTOMATED','VALIDATION','APPROVAL')),
    completion_criteria TEXT,                        -- JSON: conditions that mark the task as complete
    task_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(task_status IN ('PENDING','IN_PROGRESS','COMPLETED','SKIPPED','BLOCKED','FAILED')),
    started_at TEXT,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (template_id) REFERENCES hcmnow_solution_templates(id),
    UNIQUE(tenant_id, task_id)
);

CREATE INDEX idx_setup_tasks_tenant_category ON hcmnow_setup_tasks(tenant_id, task_category);
CREATE INDEX idx_setup_tasks_tenant_status ON hcmnow_setup_tasks(tenant_id, task_status);
CREATE INDEX idx_setup_tasks_tenant_template ON hcmnow_setup_tasks(tenant_id, template_id);
CREATE INDEX idx_setup_tasks_tenant_assignee ON hcmnow_setup_tasks(tenant_id, assigned_to);
CREATE INDEX idx_setup_tasks_tenant_sequence ON hcmnow_setup_tasks(tenant_id, task_sequence);
CREATE INDEX idx_setup_tasks_tenant_active ON hcmnow_setup_tasks(tenant_id, is_active);
```

### 2.4 Adoption Metrics
```sql
CREATE TABLE hcmnow_adoption_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,
    hcm_module TEXT NOT NULL
        CHECK(hcm_module IN ('CORE_HR','PAYROLL','BENEFITS','COMPENSATION','TIME_LABOR','RECRUITING','LEARNING','PERFORMANCE','ABSENCE','SUCCESSION')),
    module_activated INTEGER NOT NULL DEFAULT 0,
    activation_date TEXT,
    active_users INTEGER NOT NULL DEFAULT 0,
    total_eligible_users INTEGER NOT NULL DEFAULT 0,
    utilization_rate REAL NOT NULL DEFAULT 0.0
        CHECK(utilization_rate >= 0.0 AND utilization_rate <= 100.0),
    feature_usage TEXT,                              -- JSON: feature-level usage counts and percentages
    completion_rate REAL NOT NULL DEFAULT 0.0
        CHECK(completion_rate >= 0.0 AND completion_rate <= 100.0),
    user_satisfaction_score REAL
        CHECK(user_satisfaction_score IS NULL OR (user_satisfaction_score >= 1.0 AND user_satisfaction_score <= 5.0)),
    milestone_reached TEXT,                          -- JSON: milestone name and date reached

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, metric_date, hcm_module)
);

CREATE INDEX idx_adoption_metrics_tenant_date ON hcmnow_adoption_metrics(tenant_id, metric_date);
CREATE INDEX idx_adoption_metrics_tenant_module ON hcmnow_adoption_metrics(tenant_id, hcm_module);
CREATE INDEX idx_adoption_metrics_tenant_utilization ON hcmnow_adoption_metrics(tenant_id, utilization_rate);
CREATE INDEX idx_adoption_metrics_tenant_activated ON hcmnow_adoption_metrics(tenant_id, module_activated);
CREATE INDEX idx_adoption_metrics_tenant_active ON hcmnow_adoption_metrics(tenant_id, is_active);
```

### 2.5 Best Practice Rules
```sql
CREATE TABLE hcmnow_best_practice_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    description TEXT,
    country_code TEXT NOT NULL,
    industry TEXT NOT NULL,
    hcm_module TEXT NOT NULL
        CHECK(hcm_module IN ('CORE_HR','PAYROLL','BENEFITS','COMPENSATION','TIME_LABOR','RECRUITING','LEARNING','PERFORMANCE','ABSENCE','SUCCESSION')),
    compliance_type TEXT NOT NULL
        CHECK(compliance_type IN ('MANDATORY','RECOMMENDED','OPTIONAL')),
    rule_definition TEXT NOT NULL,                   -- JSON: rule logic, parameters, and validation criteria
    violation_severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(violation_severity IN ('INFO','WARNING','CRITICAL')),
    remediation_steps TEXT,                          -- JSON: steps to achieve compliance
    effective_date TEXT NOT NULL,
    expiry_date TEXT,
    rule_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(rule_status IN ('DRAFT','ACTIVE','INACTIVE','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_bp_rules_tenant_country ON hcmnow_best_practice_rules(tenant_id, country_code);
CREATE INDEX idx_bp_rules_tenant_industry ON hcmnow_best_practice_rules(tenant_id, industry);
CREATE INDEX idx_bp_rules_tenant_module ON hcmnow_best_practice_rules(tenant_id, hcm_module);
CREATE INDEX idx_bp_rules_tenant_compliance ON hcmnow_best_practice_rules(tenant_id, compliance_type);
CREATE INDEX idx_bp_rules_tenant_status ON hcmnow_best_practice_rules(tenant_id, rule_status);
CREATE INDEX idx_bp_rules_tenant_active ON hcmnow_best_practice_rules(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Solution Templates
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/hcmnow/templates` | List solution templates |
| POST | `/api/v1/hcmnow/templates` | Create solution template |
| GET | `/api/v1/hcmnow/templates/{id}` | Get template details |
| PUT | `/api/v1/hcmnow/templates/{id}` | Update template |
| POST | `/api/v1/hcmnow/templates/{id}/publish` | Publish template |
| GET | `/api/v1/hcmnow/templates/recommend` | Get recommended templates for industry/region |

### 3.2 Quick Configurations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/hcmnow/configurations` | List quick configurations |
| POST | `/api/v1/hcmnow/configurations` | Create configuration |
| GET | `/api/v1/hcmnow/configurations/{id}` | Get configuration details |
| PUT | `/api/v1/hcmnow/configurations/{id}` | Update configuration |
| POST | `/api/v1/hcmnow/configurations/{id}/validate` | Validate configuration |
| POST | `/api/v1/hcmnow/configurations/{id}/apply` | Apply configuration |
| POST | `/api/v1/hcmnow/configurations/{id}/rollback` | Roll back configuration |

### 3.3 Setup Tasks
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/hcmnow/setup-tasks` | List setup tasks |
| POST | `/api/v1/hcmnow/setup-tasks` | Create setup task |
| GET | `/api/v1/hcmnow/setup-tasks/{id}` | Get task details |
| PUT | `/api/v1/hcmnow/setup-tasks/{id}` | Update task |
| POST | `/api/v1/hcmnow/setup-tasks/{id}/start` | Start task |
| POST | `/api/v1/hcmnow/setup-tasks/{id}/complete` | Complete task |
| POST | `/api/v1/hcmnow/setup-tasks/{id}/skip` | Skip task |
| GET | `/api/v1/hcmnow/setup-tasks/progress` | Get overall setup progress |

### 3.4 Adoption Metrics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/hcmnow/adoption/dashboard` | Get adoption dashboard |
| GET | `/api/v1/hcmnow/adoption/metrics` | List adoption metrics |
| POST | `/api/v1/hcmnow/adoption/metrics` | Record adoption metric |
| GET | `/api/v1/hcmnow/adoption/trends` | Get adoption trend data |
| GET | `/api/v1/hcmnow/adoption/modules` | Get module activation status |

### 3.5 Best Practice Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/hcmnow/best-practices` | List best practice rules |
| POST | `/api/v1/hcmnow/best-practices` | Create best practice rule |
| GET | `/api/v1/hcmnow/best-practices/{id}` | Get rule details |
| PUT | `/api/v1/hcmnow/best-practices/{id}` | Update rule |
| POST | `/api/v1/hcmnow/best-practices/validate` | Validate configuration against rules |

---

## 4. Business Rules

### 4.1 Solution Templates
1. Each template MUST have a unique template_code within a tenant.
2. Templates MUST specify at least one industry and one region in their respective JSON fields.
3. hcm_modules MUST list at least one valid HCM module identifier.
4. A template in PUBLISHED status MUST NOT have its hcm_modules or default_configurations modified; a new version MUST be created.
5. estimated_setup_days MUST be a positive integer reflecting realistic implementation timelines.

### 4.2 Quick Configurations
6. configuration_data MUST conform to the schema expected by the configuration_type.
7. A configuration in APPLIED status MUST NOT be modified directly; it MUST be rolled back first.
8. When a configuration is applied, the applied_at and applied_by fields MUST be populated.
9. ORG_STRUCTURE configurations MUST include at least one top-level organization unit.
10. PAY_SCALE configurations MUST define at least one grade with a valid compensation range.

### 4.3 Setup Tasks
11. A task with depends_on references MUST NOT start until all dependent tasks have status COMPLETED.
12. task_sequence MUST be a non-negative integer determining the display and execution order.
13. Task status transitions MUST follow: PENDING -> IN_PROGRESS -> COMPLETED or PENDING -> BLOCKED or IN_PROGRESS -> FAILED.
14. A task of type AUTOMATED MUST include executable completion_criteria that can be programmatically verified.
15. The overall setup progress percentage MUST be calculated as completed tasks divided by total tasks.
16. A SKIPPED task MUST record a skip reason in the completion_criteria field.

### 4.4 Adoption Metrics
17. utilization_rate MUST be calculated as (active_users / total_eligible_users) * 100 when total_eligible_users > 0.
18. metric_date MUST be unique per tenant and hcm_module combination.
19. user_satisfaction_score, when provided, MUST be between 1.0 and 5.0 inclusive.
20. feature_usage MUST contain usage data for individual features within the module as a JSON object.
21. module_activated MUST be 0 or 1; activation_date MUST be set when module_activated is 1.

### 4.5 Best Practice Rules
22. Each rule MUST have a unique rule_code within a tenant.
23. MANDATORY compliance_type rules MUST generate CRITICAL violations when not followed.
24. rule_definition MUST contain machine-readable validation logic that can be executed against configurations.
25. Rules with an expiry_date MUST be automatically set to INACTIVE when the current date exceeds expiry_date.
26. A country_code and industry combination MAY have multiple rules per hcm_module.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.hcmnow.v1;

service HCMNowService {
    // Solution templates
    rpc CreateTemplate(CreateTemplateRequest) returns (CreateTemplateResponse);
    rpc GetTemplate(GetTemplateRequest) returns (GetTemplateResponse);
    rpc ListTemplates(ListTemplatesRequest) returns (ListTemplatesResponse);
    rpc PublishTemplate(PublishTemplateRequest) returns (PublishTemplateResponse);
    rpc RecommendTemplates(RecommendTemplatesRequest) returns (RecommendTemplatesResponse);

    // Quick configurations
    rpc CreateConfiguration(CreateConfigurationRequest) returns (CreateConfigurationResponse);
    rpc GetConfiguration(GetConfigurationRequest) returns (GetConfigurationResponse);
    rpc ListConfigurations(ListConfigurationsRequest) returns (ListConfigurationsResponse);
    rpc ValidateConfiguration(ValidateConfigurationRequest) returns (ValidateConfigurationResponse);
    rpc ApplyConfiguration(ApplyConfigurationRequest) returns (ApplyConfigurationResponse);
    rpc RollbackConfiguration(RollbackConfigurationRequest) returns (RollbackConfigurationResponse);

    // Setup tasks
    rpc CreateSetupTask(CreateSetupTaskRequest) returns (CreateSetupTaskResponse);
    rpc GetSetupTask(GetSetupTaskRequest) returns (GetSetupTaskResponse);
    rpc ListSetupTasks(ListSetupTasksRequest) returns (ListSetupTasksResponse);
    rpc StartSetupTask(StartSetupTaskRequest) returns (StartSetupTaskResponse);
    rpc CompleteSetupTask(CompleteSetupTaskRequest) returns (CompleteSetupTaskResponse);
    rpc GetSetupProgress(GetSetupProgressRequest) returns (GetSetupProgressResponse);

    // Adoption metrics
    rpc GetAdoptionDashboard(GetAdoptionDashboardRequest) returns (GetAdoptionDashboardResponse);
    rpc RecordAdoptionMetric(RecordAdoptionMetricRequest) returns (RecordAdoptionMetricResponse);
    rpc GetAdoptionTrends(GetAdoptionTrendsRequest) returns (GetAdoptionTrendsResponse);

    // Best practice rules
    rpc CreateBestPracticeRule(CreateBestPracticeRuleRequest) returns (CreateBestPracticeRuleResponse);
    rpc GetBestPracticeRule(GetBestPracticeRuleRequest) returns (GetBestPracticeRuleResponse);
    rpc ListBestPracticeRules(ListBestPracticeRulesRequest) returns (ListBestPracticeRulesResponse);
    rpc ValidateAgainstBestPractices(ValidateAgainstBestPracticesRequest) returns (ValidateAgainstBestPracticesResponse);
}

message CreateTemplateRequest {
    string tenant_id = 1;
    string template_code = 2;
    string template_name = 3;
    string description = 4;
    string industry = 5;
    string region = 6;
    string hcm_modules = 7;
    string default_configurations = 8;
    int32 estimated_setup_days = 9;
    string complexity_level = 10;
    string template_version = 11;
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
    string description = 4;
    string industry = 5;
    string region = 6;
    string hcm_modules = 7;
    string default_configurations = 8;
    int32 estimated_setup_days = 9;
    string complexity_level = 10;
    string template_version = 11;
    string template_status = 12;
}

message ListTemplatesRequest {
    string tenant_id = 1;
    string industry = 2;
    string region = 3;
    string complexity_level = 4;
    string template_status = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListTemplatesResponse {
    repeated GetTemplateResponse templates = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
}

message PublishTemplateResponse {
    string template_id = 1;
    string template_status = 2;
}

message RecommendTemplatesRequest {
    string tenant_id = 1;
    string industry = 2;
    string region = 3;
}

message RecommendTemplatesResponse {
    repeated GetTemplateResponse templates = 1;
}

message CreateConfigurationRequest {
    string tenant_id = 1;
    string configuration_id = 2;
    string template_id = 3;
    string configuration_type = 4;
    string configuration_name = 5;
    string configuration_data = 6;
    string source_template_code = 7;
}

message CreateConfigurationResponse {
    string id = 1;
    string configuration_id = 2;
    string configuration_status = 3;
}

message GetConfigurationRequest {
    string tenant_id = 1;
    string configuration_id = 2;
}

message GetConfigurationResponse {
    string id = 1;
    string configuration_id = 2;
    string template_id = 3;
    string configuration_type = 4;
    string configuration_name = 5;
    string configuration_data = 6;
    string source_template_code = 7;
    string applied_at = 8;
    string applied_by = 9;
    string configuration_status = 10;
}

message ListConfigurationsRequest {
    string tenant_id = 1;
    string configuration_type = 2;
    string configuration_status = 3;
    string template_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListConfigurationsResponse {
    repeated GetConfigurationResponse configurations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ValidateConfigurationRequest {
    string tenant_id = 1;
    string configuration_id = 2;
}

message ValidateConfigurationResponse {
    bool is_valid = 1;
    repeated string errors = 2;
    repeated string warnings = 3;
}

message ApplyConfigurationRequest {
    string tenant_id = 1;
    string configuration_id = 2;
}

message ApplyConfigurationResponse {
    string configuration_id = 1;
    string configuration_status = 2;
    string applied_at = 3;
}

message RollbackConfigurationRequest {
    string tenant_id = 1;
    string configuration_id = 2;
    string rollback_reason = 3;
}

message RollbackConfigurationResponse {
    string configuration_id = 1;
    string configuration_status = 2;
}

message CreateSetupTaskRequest {
    string tenant_id = 1;
    string task_id = 2;
    string template_id = 3;
    string task_name = 4;
    string description = 5;
    string task_category = 6;
    int32 task_sequence = 7;
    string depends_on = 8;
    string assigned_to = 9;
    int32 estimated_duration_minutes = 10;
    string task_type = 11;
    string completion_criteria = 12;
}

message CreateSetupTaskResponse {
    string task_id = 1;
    string task_status = 2;
}

message GetSetupTaskRequest {
    string tenant_id = 1;
    string task_id = 2;
}

message GetSetupTaskResponse {
    string id = 1;
    string task_id = 2;
    string template_id = 3;
    string task_name = 4;
    string description = 5;
    string task_category = 6;
    int32 task_sequence = 7;
    string depends_on = 8;
    string assigned_to = 9;
    int32 estimated_duration_minutes = 10;
    string task_type = 11;
    string completion_criteria = 12;
    string task_status = 13;
    string started_at = 14;
    string completed_at = 15;
}

message ListSetupTasksRequest {
    string tenant_id = 1;
    string task_category = 2;
    string task_status = 3;
    string template_id = 4;
    string assigned_to = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListSetupTasksResponse {
    repeated GetSetupTaskResponse tasks = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message StartSetupTaskRequest {
    string tenant_id = 1;
    string task_id = 2;
}

message StartSetupTaskResponse {
    string task_id = 1;
    string task_status = 2;
    string started_at = 3;
}

message CompleteSetupTaskRequest {
    string tenant_id = 1;
    string task_id = 2;
    string completion_notes = 3;
}

message CompleteSetupTaskResponse {
    string task_id = 1;
    string task_status = 2;
    string completed_at = 3;
}

message GetSetupProgressRequest {
    string tenant_id = 1;
    string template_id = 2;
}

message GetSetupProgressResponse {
    int32 total_tasks = 1;
    int32 completed_tasks = 2;
    int32 pending_tasks = 3;
    int32 blocked_tasks = 4;
    int32 failed_tasks = 5;
    double progress_percentage = 6;
    string estimated_completion_date = 7;
}

message GetAdoptionDashboardRequest {
    string tenant_id = 1;
    string from_date = 2;
    string to_date = 3;
}

message GetAdoptionDashboardResponse {
    double overall_utilization_rate = 1;
    int32 modules_activated = 2;
    int32 total_modules = 3;
    repeated ModuleAdoptionSummary modules = 4;
}

message ModuleAdoptionSummary {
    string hcm_module = 1;
    bool module_activated = 2;
    double utilization_rate = 3;
    int32 active_users = 4;
    int32 total_eligible_users = 5;
    double user_satisfaction_score = 6;
}

message RecordAdoptionMetricRequest {
    string tenant_id = 1;
    string metric_date = 2;
    string hcm_module = 3;
    bool module_activated = 4;
    string activation_date = 5;
    int32 active_users = 6;
    int32 total_eligible_users = 7;
    double utilization_rate = 8;
    string feature_usage = 9;
    double completion_rate = 10;
    double user_satisfaction_score = 11;
    string milestone_reached = 12;
}

message RecordAdoptionMetricResponse {
    string metric_id = 1;
    string hcm_module = 2;
    double utilization_rate = 3;
}

message GetAdoptionTrendsRequest {
    string tenant_id = 1;
    string hcm_module = 2;
    string from_date = 3;
    string to_date = 4;
}

message GetAdoptionTrendsResponse {
    repeated AdoptionDataPoint data_points = 1;
}

message AdoptionDataPoint {
    string metric_date = 1;
    double utilization_rate = 2;
    int32 active_users = 3;
    double completion_rate = 4;
}

message CreateBestPracticeRuleRequest {
    string tenant_id = 1;
    string rule_code = 2;
    string rule_name = 3;
    string description = 4;
    string country_code = 5;
    string industry = 6;
    string hcm_module = 7;
    string compliance_type = 8;
    string rule_definition = 9;
    string violation_severity = 10;
    string remediation_steps = 11;
    string effective_date = 12;
    string expiry_date = 13;
}

message CreateBestPracticeRuleResponse {
    string rule_id = 1;
    string rule_code = 2;
    string rule_status = 3;
}

message GetBestPracticeRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message GetBestPracticeRuleResponse {
    string rule_id = 1;
    string rule_code = 2;
    string rule_name = 3;
    string description = 4;
    string country_code = 5;
    string industry = 6;
    string hcm_module = 7;
    string compliance_type = 8;
    string rule_definition = 9;
    string violation_severity = 10;
    string remediation_steps = 11;
    string effective_date = 12;
    string expiry_date = 13;
    string rule_status = 14;
}

message ListBestPracticeRulesRequest {
    string tenant_id = 1;
    string country_code = 2;
    string industry = 3;
    string hcm_module = 4;
    string compliance_type = 5;
    string rule_status = 6;
    int32 page_size = 7;
    string page_token = 8;
}

message ListBestPracticeRulesResponse {
    repeated GetBestPracticeRuleResponse rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ValidateAgainstBestPracticesRequest {
    string tenant_id = 1;
    string configuration_id = 2;
    string country_code = 3;
    string industry = 4;
}

message ValidateAgainstBestPracticesResponse {
    bool is_compliant = 1;
    repeated BestPracticeViolation violations = 2;
    int32 total_rules_checked = 3;
}

message BestPracticeViolation {
    string rule_code = 1;
    string rule_name = 2;
    string compliance_type = 3;
    string violation_severity = 4;
    string violation_detail = 5;
    string remediation_steps = 6;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `core-hr-service` | Organization structures, job definitions, location hierarchies |
| `compensation-service` | Pay scale structures, grade-step definitions, compensation matrices |
| `auth-service` | User identity, role assignments, implementation team membership |
| `reference-data-service` | Country codes, industry classifications, regulatory calendars |
| `tenant-service` | Tenant onboarding state, tenant configuration metadata |

### Published To
| Service | Data |
|---------|------|
| `core-hr-service` | Organization configuration payloads, job family structures |
| `compensation-service` | Pay scale configurations, grade-step setups |
| `reporting-service` | Adoption dashboards, setup progress reports, utilization analytics |
| `notification-service` | Setup task assignments, completion reminders, milestone alerts |
| `audit-service` | Configuration change logs, template usage tracking, compliance audit trail |
| `workflow-service` | Setup task dependency routing, approval chains for configuration apply |

---

## 7. Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `hcmnow.setup.completed` | `hcmnow.events` | `{ template_id, template_code, total_tasks, completed_tasks, completed_at }` | All setup tasks for a template completed |
| `hcmnow.module.activated` | `hcmnow.events` | `{ hcm_module, activation_date, active_users, total_eligible_users }` | HCM module activated for tenant |
| `hcmnow.adoption.milestone` | `hcmnow.events` | `{ milestone_name, hcm_module, metric_date, utilization_rate, milestone_reached }` | Adoption milestone reached |
| `hcmnow.configuration.applied` | `hcmnow.events` | `{ configuration_id, configuration_type, configuration_name, applied_by, applied_at }` | Quick configuration applied |

---

## 8. Migrations

1. V001: `hcmnow_solution_templates`
2. V002: `hcmnow_quick_configurations`
3. V003: `hcmnow_setup_tasks`
4. V004: `hcmnow_adoption_metrics`
5. V005: `hcmnow_best_practice_rules`
6. V006: Triggers for `updated_at`
