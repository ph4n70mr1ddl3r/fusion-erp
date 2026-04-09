# 117 - EPM Platform Service Specification

## 1. Domain Overview

The EPM Platform service provides the foundational technical platform for all EPM modules including the multidimensional calculation engine, Calculation Manager, data integration frameworks, security model, administration tools, and lifecycle management. It provides the shared infrastructure for Planning, Consolidation, Reconciliation, and Reporting modules.

**Bounded Context:** EPM Platform Infrastructure
**Service Name:** `epmplatform-service`
**Database:** `data/epmplatform.db`
**HTTP Port:** 8154 | **gRPC Port:** 9154

---

## 2. Database Schema

### 2.1 EPM Applications
```sql
CREATE TABLE epm_applications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_name TEXT NOT NULL,
    application_type TEXT NOT NULL CHECK(application_type IN ('PLANNING','CONSOLIDATION','RECONCILIATION','REPORTING','FREEFORM','PROFITABILITY')),
    cube_name TEXT NOT NULL,
    dimensionality TEXT NOT NULL,  -- JSON
    default_currency TEXT NOT NULL DEFAULT 'USD',
    fiscal_calendar_id TEXT,
    status TEXT NOT NULL DEFAULT 'CREATED' CHECK(status IN ('CREATED','CONFIGURING','ACTIVE','MAINTENANCE','DEPRECATED')),
    data_storage_type TEXT NOT NULL DEFAULT 'BLOCK_STORAGE' CHECK(data_storage_type IN ('BLOCK_STORAGE','AGGREGATE_STORAGE')),
    size_mb DECIMAL(10,2) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, application_name)
);

CREATE INDEX idx_epma_tenant_type ON epm_applications(tenant_id, application_type);
CREATE INDEX idx_epma_tenant_status ON epm_applications(tenant_id, status);
```

### 2.2 EPM Dimensions
```sql
CREATE TABLE epm_dimensions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    dimension_name TEXT NOT NULL,
    dimension_type TEXT NOT NULL CHECK(dimension_type IN ('ACCOUNT','ENTITY','SCENARIO','VERSION','PERIOD','YEAR','CUSTOM')),
    member_count INTEGER NOT NULL DEFAULT 0,
    hierarchy_depth INTEGER NOT NULL DEFAULT 1,
    storage_type TEXT NOT NULL DEFAULT 'SPARSE' CHECK(storage_type IN ('DENSE','SPARSE')),
    dynamic_calc_members INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES epm_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, application_id, dimension_name)
);

CREATE INDEX idx_epmd_tenant_app ON epm_dimensions(tenant_id, application_id);
CREATE INDEX idx_epmd_tenant_type ON epm_dimensions(tenant_id, dimension_type);
```

### 2.3 Dimension Members
```sql
CREATE TABLE dimension_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dimension_id TEXT NOT NULL,
    member_name TEXT NOT NULL,
    parent_member_id TEXT,
    member_type TEXT NOT NULL DEFAULT 'STORAGE' CHECK(member_type IN ('STORAGE','DYNAMIC_CALC','SHARED','LABEL_ONLY')),
    data_type TEXT NOT NULL DEFAULT 'NUMBER' CHECK(data_type IN ('CURRENCY','NUMBER','PERCENTAGE','TEXT','DATE')),
    aggregation TEXT NOT NULL DEFAULT 'SUM' CHECK(aggregation IN ('SUM','AVG','MIN','MAX','COUNT','NONE')),
    formula TEXT,
    generation INTEGER NOT NULL DEFAULT 1,
    level INTEGER NOT NULL DEFAULT 0,
    alias_names TEXT,  -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dimension_id) REFERENCES epm_dimensions(id) ON DELETE CASCADE
);

CREATE INDEX idx_dm_tenant_dim ON dimension_members(tenant_id, dimension_id);
CREATE INDEX idx_dm_tenant_parent ON dimension_members(tenant_id, parent_member_id);
CREATE INDEX idx_dm_tenant_name ON dimension_members(tenant_id, member_name);
CREATE INDEX idx_dm_tenant_type ON dimension_members(tenant_id, member_type);
```

### 2.4 Calculation Rules
```sql
CREATE TABLE calculation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('CALCULATION','ALLOCATION','AGGREGATION','CURRENCY','VALIDATION')),
    target_dimensions TEXT NOT NULL,  -- JSON
    calculation_script TEXT NOT NULL,
    execution_order INTEGER NOT NULL DEFAULT 1,
    run_mode TEXT NOT NULL DEFAULT 'ON_DEMAND' CHECK(run_mode IN ('ON_DEMAND','SCHEDULED','EVENT_TRIGGERED')),
    schedule_config TEXT,  -- JSON
    last_executed_at TEXT,
    avg_execution_seconds DECIMAL(10,2) DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES epm_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, application_id, rule_name)
);

CREATE INDEX idx_cr_tenant_app ON calculation_rules(tenant_id, application_id);
CREATE INDEX idx_cr_tenant_type ON calculation_rules(tenant_id, rule_type);
CREATE INDEX idx_cr_tenant_status ON calculation_rules(tenant_id, status);
```

### 2.5 Data Integration Jobs
```sql
CREATE TABLE data_integration_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    job_name TEXT NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('ERP','GL','FLAT_FILE','API','EXTERNAL')),
    target_cube TEXT NOT NULL,
    mapping_config TEXT NOT NULL,  -- JSON
    transformation_rules TEXT,  -- JSON
    load_type TEXT NOT NULL DEFAULT 'INCREMENTAL' CHECK(load_type IN ('REPLACE','INCREMENTAL','MERGE')),
    schedule_config TEXT,  -- JSON
    last_run_at TEXT,
    last_run_status TEXT CHECK(last_run_status IN ('SUCCESS','WARNING','FAILED')),
    rows_processed INTEGER DEFAULT 0,
    execution_seconds DECIMAL(10,2) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES epm_applications(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, application_id, job_name)
);

CREATE INDEX idx_dij_tenant_app ON data_integration_jobs(tenant_id, application_id);
CREATE INDEX idx_dij_tenant_source ON data_integration_jobs(tenant_id, source_type);
CREATE INDEX idx_dij_tenant_status ON data_integration_jobs(tenant_id, last_run_status);
```

### 2.6 EPM Security Policies
```sql
CREATE TABLE epm_security_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    role_id TEXT NOT NULL,
    dimension_access TEXT NOT NULL,  -- JSON
    member_access TEXT NOT NULL,  -- JSON
    data_filter_config TEXT,  -- JSON
    write_access INTEGER NOT NULL DEFAULT 0,
    approval_required INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES epm_applications(id) ON DELETE CASCADE
);

CREATE INDEX idx_epmsp_tenant_app ON epm_security_policies(tenant_id, application_id);
CREATE INDEX idx_epmsp_tenant_role ON epm_security_policies(tenant_id, role_id);
```

### 2.7 EPM Audit Log
```sql
CREATE TABLE epm_audit_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    application_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    action TEXT NOT NULL CHECK(action IN ('LOGIN','LOGOUT','CALCULATE','LOAD_DATA','EXPORT','ADMIN_CHANGE','SECURITY_CHANGE','MODEL_CHANGE')),
    target_object TEXT NOT NULL,
    before_state TEXT,
    after_state TEXT,
    session_id TEXT,
    timestamp TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (application_id) REFERENCES epm_applications(id)
);

CREATE INDEX idx_epmal_tenant_app ON epm_audit_log(tenant_id, application_id);
CREATE INDEX idx_epmal_tenant_user ON epm_audit_log(tenant_id, user_id);
CREATE INDEX idx_epmal_tenant_action ON epm_audit_log(tenant_id, action);
CREATE INDEX idx_epmal_tenant_ts ON epm_audit_log(tenant_id, timestamp);
```

---

## 3. REST API Endpoints

### Applications
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/epm-applications` | Create an EPM application |
| GET | `/api/v1/epm-applications` | List EPM applications |
| GET | `/api/v1/epm-applications/{id}` | Get application details |
| PUT | `/api/v1/epm-applications/{id}` | Update application |
| POST | `/api/v1/epm-applications/{id}/deploy` | Deploy application |
| POST | `/api/v1/epm-applications/{id}/refresh-schema` | Refresh application schema |
| GET | `/api/v1/epm-applications/{id}/status` | Get application status |

### Dimensions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/epm-applications/{id}/dimensions` | Create a dimension |
| GET | `/api/v1/epm-applications/{id}/dimensions` | List dimensions |
| GET | `/api/v1/dimensions/{id}` | Get dimension details |
| PUT | `/api/v1/dimensions/{id}` | Update dimension |
| POST | `/api/v1/dimensions/{id}/add-member` | Add a member to dimension |
| GET | `/api/v1/dimensions/{id}/hierarchy` | Get dimension hierarchy |

### Members
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/dimension-members` | Create a dimension member |
| GET | `/api/v1/dimension-members` | List dimension members |
| GET | `/api/v1/dimension-members/{id}` | Get member details |
| PUT | `/api/v1/dimension-members/{id}` | Update member |
| POST | `/api/v1/dimension-members/{id}/move` | Move member in hierarchy |
| POST | `/api/v1/dimensions/{id}/restructure` | Restructure dimension |

### Calculation Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/epm-applications/{id}/calculation-rules` | Create a calculation rule |
| GET | `/api/v1/epm-applications/{id}/calculation-rules` | List calculation rules |
| GET | `/api/v1/calculation-rules/{id}` | Get rule details |
| PUT | `/api/v1/calculation-rules/{id}` | Update rule |
| POST | `/api/v1/calculation-rules/{id}/execute` | Execute a calculation rule |
| POST | `/api/v1/calculation-rules/{id}/validate` | Validate a calculation rule |
| GET | `/api/v1/calculation-rules/{id}/execution-log` | Get execution log |

### Data Integration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/epm-applications/{id}/data-integration-jobs` | Create an integration job |
| GET | `/api/v1/epm-applications/{id}/data-integration-jobs` | List integration jobs |
| GET | `/api/v1/data-integration-jobs/{id}` | Get job details |
| PUT | `/api/v1/data-integration-jobs/{id}` | Update job |
| POST | `/api/v1/data-integration-jobs/{id}/execute` | Execute an integration job |
| GET | `/api/v1/data-integration-jobs/{id}/status` | Get job execution status |
| POST | `/api/v1/data-integration-jobs/{id}/schedule` | Schedule a job |

### Security
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/epm-applications/{id}/security-policies` | Create a security policy |
| GET | `/api/v1/epm-applications/{id}/security-policies` | List security policies |
| GET | `/api/v1/security-policies/{id}` | Get policy details |
| PUT | `/api/v1/security-policies/{id}` | Update policy |
| POST | `/api/v1/security-policies/{id}/validate-access` | Validate user access |

### Audit
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/epm-audit-log` | Get audit log entries |
| GET | `/api/v1/epm-audit-log/by-user/{userId}` | Get audit log by user |
| GET | `/api/v1/epm-audit-log/by-action/{action}` | Get audit log by action type |

---

## 4. Business Rules

1. An EPM application MUST define at least 3 dimensions (Account, Entity, Period) before it can be deployed.
2. A dimension member of type DYNAMIC_CALC MUST have a valid formula defined.
3. The system MUST NOT allow deletion of a dimension that has member_count greater than 0.
4. Calculation rules MUST be validated for syntax correctness before they can be set to ACTIVE status.
5. Data integration jobs with load_type REPLACE MUST require explicit confirmation before execution on applications in ACTIVE status.
6. Security policies MUST be evaluated on every data access request and MUST deny access by default if no policy matches.
7. The system MUST log all ADMIN_CHANGE, SECURITY_CHANGE, and MODEL_CHANGE actions in the audit log with before_state and after_state.
8. An application in MAINTENANCE status MUST reject all calculation and data load requests.
9. Member hierarchy generation and level values MUST be recalculated automatically whenever a member is moved or added.
10. SCHEDULED calculation rules MUST NOT have overlapping execution windows within the same application.
11. The system SHOULD enforce a maximum of 20 dimensions per application to maintain performance.
12. Data integration job execution MUST be tracked with rows_processed and execution_seconds for performance monitoring.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package epmplatform.v1;

service EPMPlatformService {
    // Applications
    rpc CreateApplication(CreateApplicationRequest) returns (CreateApplicationResponse);
    rpc GetApplication(GetApplicationRequest) returns (GetApplicationResponse);
    rpc ListApplications(ListApplicationsRequest) returns (ListApplicationsResponse);
    rpc DeployApplication(DeployApplicationRequest) returns (DeployApplicationResponse);
    rpc RefreshSchema(RefreshSchemaRequest) returns (RefreshSchemaResponse);
    rpc GetApplicationStatus(GetApplicationStatusRequest) returns (GetApplicationStatusResponse);

    // Dimensions
    rpc CreateDimension(CreateDimensionRequest) returns (CreateDimensionResponse);
    rpc GetDimension(GetDimensionRequest) returns (GetDimensionResponse);
    rpc ListDimensions(ListDimensionsRequest) returns (ListDimensionsResponse);
    rpc AddDimensionMember(AddDimensionMemberRequest) returns (AddDimensionMemberResponse);
    rpc GetDimensionHierarchy(GetDimensionHierarchyRequest) returns (GetDimensionHierarchyResponse);

    // Members
    rpc CreateMember(CreateMemberRequest) returns (CreateMemberResponse);
    rpc GetMember(GetMemberRequest) returns (GetMemberResponse);
    rpc MoveMember(MoveMemberRequest) returns (MoveMemberResponse);
    rpc RestructureDimension(RestructureDimensionRequest) returns (RestructureDimensionResponse);

    // Calculation Rules
    rpc CreateCalculationRule(CreateCalculationRuleRequest) returns (CreateCalculationRuleResponse);
    rpc GetCalculationRule(GetCalculationRuleRequest) returns (GetCalculationRuleResponse);
    rpc ListCalculationRules(ListCalculationRulesRequest) returns (ListCalculationRulesResponse);
    rpc ExecuteCalculation(ExecuteCalculationRequest) returns (ExecuteCalculationResponse);
    rpc ValidateCalculation(ValidateCalculationRequest) returns (ValidateCalculationResponse);

    // Data Integration
    rpc CreateDataIntegrationJob(CreateDataIntegrationJobRequest) returns (CreateDataIntegrationJobResponse);
    rpc ExecuteDataIntegration(ExecuteDataIntegrationRequest) returns (ExecuteDataIntegrationResponse);
    rpc GetDataIntegrationStatus(GetDataIntegrationStatusRequest) returns (GetDataIntegrationStatusResponse);

    // Security
    rpc CreateSecurityPolicy(CreateSecurityPolicyRequest) returns (CreateSecurityPolicyResponse);
    rpc ListSecurityPolicies(ListSecurityPoliciesRequest) returns (ListSecurityPoliciesResponse);
    rpc ValidateAccess(ValidateAccessRequest) returns (ValidateAccessResponse);

    // Audit
    rpc GetAuditLog(GetAuditLogRequest) returns (GetAuditLogResponse);
    rpc GetAuditLogByUser(GetAuditLogByUserRequest) returns (GetAuditLogByUserResponse);
    rpc GetAuditLogByAction(GetAuditLogByActionRequest) returns (GetAuditLogByActionResponse);
}

message EPMApplication {
    string id = 1;
    string tenant_id = 2;
    string application_name = 3;
    string application_type = 4;
    string cube_name = 5;
    string dimensionality = 6;
    string default_currency = 7;
    string fiscal_calendar_id = 8;
    string status = 9;
    string data_storage_type = 10;
    double size_mb = 11;
}

message CreateApplicationRequest {
    string tenant_id = 1;
    string application_name = 2;
    string application_type = 3;
    string cube_name = 4;
    string dimensionality = 5;
    string default_currency = 6;
    string fiscal_calendar_id = 7;
    string data_storage_type = 8;
    string created_by = 9;
}

message CreateApplicationResponse {
    EPMApplication application = 1;
}

message GetApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
}

message GetApplicationResponse {
    EPMApplication application = 1;
}

message ListApplicationsRequest {
    string tenant_id = 1;
    string application_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListApplicationsResponse {
    repeated EPMApplication applications = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message DeployApplicationRequest {
    string tenant_id = 1;
    string application_id = 2;
    string deployed_by = 3;
}

message DeployApplicationResponse {
    EPMApplication application = 1;
}

message RefreshSchemaRequest {
    string tenant_id = 1;
    string application_id = 2;
    string refreshed_by = 3;
}

message RefreshSchemaResponse {
    bool success = 1;
    int32 dimension_count = 2;
    int32 total_member_count = 3;
}

message GetApplicationStatusRequest {
    string tenant_id = 1;
    string application_id = 2;
}

message GetApplicationStatusResponse {
    string status = 1;
    double size_mb = 2;
    int32 dimension_count = 3;
    int32 member_count = 4;
    int32 rule_count = 5;
    string last_calculation_at = 6;
}

message EPMDimension {
    string id = 1;
    string tenant_id = 2;
    string application_id = 3;
    string dimension_name = 4;
    string dimension_type = 5;
    int32 member_count = 6;
    int32 hierarchy_depth = 7;
    string storage_type = 8;
    int32 dynamic_calc_members = 9;
    string description = 10;
}

message CreateDimensionRequest {
    string tenant_id = 1;
    string application_id = 2;
    string dimension_name = 3;
    string dimension_type = 4;
    string storage_type = 5;
    string description = 6;
    string created_by = 7;
}

message CreateDimensionResponse {
    EPMDimension dimension = 1;
}

message GetDimensionRequest {
    string tenant_id = 1;
    string dimension_id = 2;
}

message GetDimensionResponse {
    EPMDimension dimension = 1;
}

message ListDimensionsRequest {
    string tenant_id = 1;
    string application_id = 2;
    string dimension_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListDimensionsResponse {
    repeated EPMDimension dimensions = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message AddDimensionMemberRequest {
    string tenant_id = 1;
    string dimension_id = 2;
    string member_name = 3;
    string parent_member_id = 4;
    string member_type = 5;
    string data_type = 6;
    string aggregation = 7;
    string formula = 8;
    string added_by = 9;
}

message AddDimensionMemberResponse {
    string member_id = 1;
    int32 generation = 2;
    int32 level = 3;
}

message GetDimensionHierarchyRequest {
    string tenant_id = 1;
    string dimension_id = 2;
}

message GetDimensionHierarchyResponse {
    repeated HierarchyNode nodes = 1;
    int32 total_members = 2;
    int32 max_depth = 3;
}

message HierarchyNode {
    string member_id = 1;
    string member_name = 2;
    string parent_member_id = 3;
    int32 generation = 4;
    int32 level = 5;
    string member_type = 6;
    repeated HierarchyNode children = 7;
}

message DimensionMember {
    string id = 1;
    string tenant_id = 2;
    string dimension_id = 3;
    string member_name = 4;
    string parent_member_id = 5;
    string member_type = 6;
    string data_type = 7;
    string aggregation = 8;
    string formula = 9;
    int32 generation = 10;
    int32 level = 11;
}

message CreateMemberRequest {
    string tenant_id = 1;
    string dimension_id = 2;
    string member_name = 3;
    string parent_member_id = 4;
    string member_type = 5;
    string data_type = 6;
    string aggregation = 7;
    string formula = 8;
    string created_by = 9;
}

message CreateMemberResponse {
    DimensionMember member = 1;
}

message GetMemberRequest {
    string tenant_id = 1;
    string member_id = 2;
}

message GetMemberResponse {
    DimensionMember member = 1;
}

message MoveMemberRequest {
    string tenant_id = 1;
    string member_id = 2;
    string new_parent_id = 3;
    string moved_by = 4;
}

message MoveMemberResponse {
    DimensionMember member = 1;
}

message RestructureDimensionRequest {
    string tenant_id = 1;
    string dimension_id = 2;
    string restructured_by = 3;
}

message RestructureDimensionResponse {
    bool success = 1;
    int32 members_updated = 2;
}

message CalculationRule {
    string id = 1;
    string tenant_id = 2;
    string application_id = 3;
    string rule_name = 4;
    string rule_type = 5;
    string target_dimensions = 6;
    string calculation_script = 7;
    int32 execution_order = 8;
    string run_mode = 9;
    string status = 10;
    string last_executed_at = 11;
    double avg_execution_seconds = 12;
}

message CreateCalculationRuleRequest {
    string tenant_id = 1;
    string application_id = 2;
    string rule_name = 3;
    string rule_type = 4;
    string target_dimensions = 5;
    string calculation_script = 6;
    int32 execution_order = 7;
    string run_mode = 8;
    string created_by = 9;
}

message CreateCalculationRuleResponse {
    CalculationRule rule = 1;
}

message GetCalculationRuleRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message GetCalculationRuleResponse {
    CalculationRule rule = 1;
}

message ListCalculationRulesRequest {
    string tenant_id = 1;
    string application_id = 2;
    string rule_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListCalculationRulesResponse {
    repeated CalculationRule rules = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ExecuteCalculationRequest {
    string tenant_id = 1;
    string rule_id = 2;
    string executed_by = 3;
}

message ExecuteCalculationResponse {
    bool success = 1;
    double execution_seconds = 2;
    int32 cells_calculated = 3;
}

message ValidateCalculationRequest {
    string tenant_id = 1;
    string rule_id = 2;
}

message ValidateCalculationResponse {
    bool is_valid = 1;
    repeated string errors = 2;
    repeated string warnings = 3;
}

message DataIntegrationJob {
    string id = 1;
    string tenant_id = 2;
    string application_id = 3;
    string job_name = 4;
    string source_type = 5;
    string target_cube = 6;
    string mapping_config = 7;
    string load_type = 8;
    string last_run_at = 9;
    string last_run_status = 10;
    int32 rows_processed = 11;
    double execution_seconds = 12;
}

message CreateDataIntegrationJobRequest {
    string tenant_id = 1;
    string application_id = 2;
    string job_name = 3;
    string source_type = 4;
    string target_cube = 5;
    string mapping_config = 6;
    string transformation_rules = 7;
    string load_type = 8;
    string created_by = 9;
}

message CreateDataIntegrationJobResponse {
    DataIntegrationJob job = 1;
}

message ExecuteDataIntegrationRequest {
    string tenant_id = 1;
    string job_id = 2;
    string executed_by = 3;
}

message ExecuteDataIntegrationResponse {
    bool success = 1;
    int32 rows_processed = 2;
    double execution_seconds = 3;
    string status = 4;
}

message GetDataIntegrationStatusRequest {
    string tenant_id = 1;
    string job_id = 2;
}

message GetDataIntegrationStatusResponse {
    string last_run_at = 1;
    string last_run_status = 2;
    int32 rows_processed = 3;
    double execution_seconds = 4;
}

message EPMSecurityPolicy {
    string id = 1;
    string tenant_id = 2;
    string application_id = 3;
    string role_id = 4;
    string dimension_access = 5;
    string member_access = 6;
    string data_filter_config = 7;
    bool write_access = 8;
    bool approval_required = 9;
}

message CreateSecurityPolicyRequest {
    string tenant_id = 1;
    string application_id = 2;
    string role_id = 3;
    string dimension_access = 4;
    string member_access = 5;
    string data_filter_config = 6;
    bool write_access = 7;
    bool approval_required = 8;
    string created_by = 9;
}

message CreateSecurityPolicyResponse {
    EPMSecurityPolicy policy = 1;
}

message ListSecurityPoliciesRequest {
    string tenant_id = 1;
    string application_id = 2;
    string role_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListSecurityPoliciesResponse {
    repeated EPMSecurityPolicy policies = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ValidateAccessRequest {
    string tenant_id = 1;
    string application_id = 2;
    string user_id = 3;
    string dimension_id = 4;
    string member_id = 5;
    string access_type = 6;
}

message ValidateAccessResponse {
    bool has_access = 1;
    string policy_id = 2;
}

message AuditLogEntry {
    string id = 1;
    string tenant_id = 2;
    string application_id = 3;
    string user_id = 4;
    string action = 5;
    string target_object = 6;
    string before_state = 7;
    string after_state = 8;
    string session_id = 9;
    string timestamp = 10;
}

message GetAuditLogRequest {
    string tenant_id = 1;
    string application_id = 2;
    string date_from = 3;
    string date_to = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message GetAuditLogResponse {
    repeated AuditLogEntry entries = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetAuditLogByUserRequest {
    string tenant_id = 1;
    string user_id = 2;
    string date_from = 3;
    string date_to = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message GetAuditLogByUserResponse {
    repeated AuditLogEntry entries = 1;
    string next_page_token = 2;
}

message GetAuditLogByActionRequest {
    string tenant_id = 1;
    string action = 2;
    string date_from = 3;
    string date_to = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message GetAuditLogByActionResponse {
    repeated AuditLogEntry entries = 1;
    string next_page_token = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | GL balances, account structures | Load financial data into EPM cubes |
| `hr-service` | Organization structure, employee data | Populate Entity and workforce dimensions |
| `auth-service` | User roles, permissions | Enforce EPM security policies |
| `file-service` | Uploaded data files | Import flat file data into cubes |
| `scheduler-service` | Job scheduling triggers | Execute scheduled calculations and loads |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `planning-service` | Application metadata, dimensions, rules | Planning module foundation |
| `consolidation-service` | Application metadata, currency settings | Consolidation module foundation |
| `reporting-service` | Calculated cube data | Financial reporting and dashboards |
| `audit-service` | EPM audit events | Enterprise audit trail |
| `notification-service` | Calculation completion, load status | User notifications |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ApplicationDeployed` | `epmplatform.application.deployed` | `{ tenant_id, application_id, application_name, application_type, cube_name, dimension_count }` | Published when an EPM application is deployed |
| `CalculationCompleted` | `epmplatform.calculation.completed` | `{ tenant_id, application_id, rule_id, rule_name, execution_seconds, cells_calculated }` | Published when a calculation rule finishes execution |
| `DataIntegrationCompleted` | `epmplatform.data-integration.completed` | `{ tenant_id, application_id, job_id, job_name, rows_processed, execution_seconds, status }` | Published when a data integration job completes |
| `SecurityPolicyChanged` | `epmplatform.security.changed` | `{ tenant_id, application_id, policy_id, role_id, change_type, changed_by }` | Published when a security policy is created or modified |
| `ModelStructureChanged` | `epmplatform.model.changed` | `{ tenant_id, application_id, dimension_id, change_type, member_count }` | Published when dimension structure or members are modified |
