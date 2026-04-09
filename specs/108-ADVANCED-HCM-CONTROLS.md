# 108 - Advanced HCM Controls Service Specification

## 1. Domain Overview

The Advanced HCM Controls service provides an automated compliance and audit framework for HCM configurations. It monitors setup changes, flags security risks, enforces segregation of duties (SoD), provides change-impact analysis, access certifications, and comprehensive audit trails for HCM data governance. The service supports SOX compliance requirements and ensures audit readiness across all HCM data and configuration changes.

**Bounded Context:** HCM Compliance, Audit & Governance
**Service Name:** `hcmcontrols-service`
**Database:** `data/hcmcontrols.db`
**HTTP Port:** 8145 | **gRPC Port:** 9145

---

## 2. Database Schema

### 2.1 Control Policies
```sql
CREATE TABLE control_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_name TEXT NOT NULL,
    policy_type TEXT NOT NULL CHECK(policy_type IN ('SOD','ACCESS_CERTIFICATION','CHANGE_MONITOR','CONFIGURATION_GOVERNANCE','DATA_PRIVACY')),
    description TEXT,
    severity TEXT NOT NULL DEFAULT 'WARNING' CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    is_active INTEGER NOT NULL DEFAULT 1,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    evaluation_frequency TEXT NOT NULL DEFAULT 'DAILY' CHECK(evaluation_frequency IN ('REAL_TIME','DAILY','WEEKLY','MONTHLY')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_name)
);

CREATE INDEX idx_policies_tenant_type ON control_policies(tenant_id, policy_type);
CREATE INDEX idx_policies_tenant_active ON control_policies(tenant_id, is_active);
CREATE INDEX idx_policies_tenant_frequency ON control_policies(tenant_id, evaluation_frequency);
```

### 2.2 Segregation of Duties Rules
```sql
CREATE TABLE sod_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    role_combination TEXT NOT NULL,     -- JSON: array of conflicting role pairs
    conflict_description TEXT NOT NULL,
    risk_level TEXT NOT NULL DEFAULT 'MEDIUM' CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    mitigation_action TEXT NOT NULL DEFAULT 'WARN' CHECK(mitigation_action IN ('PREVENT','WARN','REPORT')),
    exemptible INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (policy_id) REFERENCES control_policies(id)
);

CREATE INDEX idx_sod_tenant_policy ON sod_rules(tenant_id, policy_id);
CREATE INDEX idx_sod_tenant_risk ON sod_rules(tenant_id, risk_level);
```

### 2.3 Access Certification Campaigns
```sql
CREATE TABLE access_certification_campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_name TEXT NOT NULL,
    campaign_type TEXT NOT NULL CHECK(campaign_type IN ('USER_ACCESS','ROLE_ACCESS','ENTITLEMENT')),
    reviewer_id TEXT NOT NULL,
    scope TEXT,  -- JSON: scope definition (roles, applications, users)
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','CANCELLED')),
    total_items INTEGER NOT NULL DEFAULT 0,
    certified_items INTEGER NOT NULL DEFAULT 0,
    revoked_items INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_campaigns_tenant_status ON access_certification_campaigns(tenant_id, status);
CREATE INDEX idx_campaigns_tenant_reviewer ON access_certification_campaigns(tenant_id, reviewer_id);
CREATE INDEX idx_campaigns_tenant_dates ON access_certification_campaigns(tenant_id, start_date, end_date);
```

### 2.4 Certification Items
```sql
CREATE TABLE certification_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    certifier_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    role_id TEXT,
    entitlement_id TEXT,
    decision TEXT CHECK(decision IN ('CERTIFY','REVOKE','NOT_SURE')),
    decision_rationale TEXT,
    decision_date TEXT,
    risk_comment TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','CERTIFIED','REVOKED','ESCALATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (campaign_id) REFERENCES access_certification_campaigns(id) ON DELETE CASCADE
);

CREATE INDEX idx_certitems_tenant_campaign ON certification_items(tenant_id, campaign_id);
CREATE INDEX idx_certitems_tenant_certifier ON certification_items(tenant_id, certifier_id, status);
CREATE INDEX idx_certitems_tenant_user ON certification_items(tenant_id, user_id);
```

### 2.5 Configuration Changes
```sql
CREATE TABLE configuration_changes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    change_type TEXT NOT NULL CHECK(change_type IN ('CREATE','UPDATE','DELETE','IMPORT','EXPORT')),
    changed_object_type TEXT NOT NULL,
    changed_object_id TEXT NOT NULL,
    changed_by TEXT NOT NULL,
    change_timestamp TEXT NOT NULL,
    old_value TEXT,
    new_value TEXT,
    change_reason TEXT,
    impact_assessment TEXT,  -- JSON: impact analysis results
    approved_by TEXT,
    approval_required INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'AUTO_APPROVED' CHECK(status IN ('PENDING_APPROVAL','APPROVED','REJECTED','AUTO_APPROVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_changes_tenant_type ON configuration_changes(tenant_id, changed_object_type);
CREATE INDEX idx_changes_tenant_by ON configuration_changes(tenant_id, changed_by);
CREATE INDEX idx_changes_tenant_status ON configuration_changes(tenant_id, status);
CREATE INDEX idx_changes_tenant_timestamp ON configuration_changes(tenant_id, change_timestamp);
```

### 2.6 Control Violations
```sql
CREATE TABLE control_violations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_id TEXT NOT NULL,
    rule_id TEXT,
    violation_type TEXT NOT NULL,
    violated_by_user TEXT NOT NULL,
    violated_at TEXT NOT NULL,
    severity TEXT NOT NULL CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    description TEXT NOT NULL,
    affected_objects TEXT,  -- JSON: list of affected object references
    resolution_status TEXT NOT NULL DEFAULT 'OPEN' CHECK(resolution_status IN ('OPEN','IN_REVIEW','RESOLVED','WAIVED')),
    resolved_by TEXT,
    resolved_at TEXT,
    resolution_notes TEXT,
    waivable INTEGER NOT NULL DEFAULT 1,
    waived_by TEXT,
    waived_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (policy_id) REFERENCES control_policies(id)
);

CREATE INDEX idx_violations_tenant_policy ON control_violations(tenant_id, policy_id);
CREATE INDEX idx_violations_tenant_status ON control_violations(tenant_id, resolution_status);
CREATE INDEX idx_violations_tenant_severity ON control_violations(tenant_id, severity);
CREATE INDEX idx_violations_tenant_user ON control_violations(tenant_id, violated_by_user);
```

### 2.7 Audit Trails
```sql
CREATE TABLE audit_trails (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    audit_event_type TEXT NOT NULL,
    actor_id TEXT NOT NULL,
    actor_role TEXT,
    target_object_type TEXT NOT NULL,
    target_object_id TEXT NOT NULL,
    action TEXT NOT NULL,
    before_state TEXT,
    after_state TEXT,
    ip_address TEXT,
    user_agent TEXT,
    session_id TEXT,
    timestamp TEXT NOT NULL,
    classification TEXT NOT NULL DEFAULT 'NORMAL' CHECK(classification IN ('NORMAL','SUSPICIOUS','CRITICAL')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_audit_tenant_event ON audit_trails(tenant_id, audit_event_type);
CREATE INDEX idx_audit_tenant_actor ON audit_trails(tenant_id, actor_id);
CREATE INDEX idx_audit_tenant_target ON audit_trails(tenant_id, target_object_type, target_object_id);
CREATE INDEX idx_audit_tenant_timestamp ON audit_trails(tenant_id, timestamp);
CREATE INDEX idx_audit_tenant_class ON audit_trails(tenant_id, classification);
```

---

## 3. REST API Endpoints

### Policies
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/policies` | Create a control policy |
| GET | `/api/v1/policies` | List control policies |
| GET | `/api/v1/policies/{id}` | Get policy details |
| PUT | `/api/v1/policies/{id}` | Update policy |
| POST | `/api/v1/policies/{id}/evaluate` | Manually evaluate a policy |

### SoD Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sod-rules` | Create SoD rule |
| GET | `/api/v1/sod-rules` | List SoD rules |
| GET | `/api/v1/sod-rules/{id}` | Get SoD rule details |
| PUT | `/api/v1/sod-rules/{id}` | Update SoD rule |
| POST | `/api/v1/sod-rules/check-conflict` | Check for SoD conflicts |
| POST | `/api/v1/sod-rules/simulate-role-change` | Simulate role change impact |

### Certifications
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/certifications` | Create certification campaign |
| GET | `/api/v1/certifications` | List certification campaigns |
| GET | `/api/v1/certifications/{id}` | Get campaign details |
| PUT | `/api/v1/certifications/{id}` | Update campaign |
| POST | `/api/v1/certifications/{id}/launch` | Launch certification campaign |
| POST | `/api/v1/certifications/{id}/certify-item` | Certify an access item |
| POST | `/api/v1/certifications/{id}/revoke-item` | Revoke an access item |
| GET | `/api/v1/certifications/{id}/progress` | Get campaign progress |

### Configuration Changes
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/config-changes` | Get configuration change log |
| POST | `/api/v1/config-changes/{id}/approve` | Approve a configuration change |
| POST | `/api/v1/config-changes/{id}/reject` | Reject a configuration change |
| GET | `/api/v1/config-changes/{id}/impact-analysis` | Get impact analysis for change |

### Violations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/violations` | List control violations |
| POST | `/api/v1/violations/{id}/resolve` | Resolve a violation |
| POST | `/api/v1/violations/{id}/waive` | Waive a violation |
| GET | `/api/v1/violations/summary` | Get violation summary statistics |

### Audit Trail
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/audit-trail` | Get audit trail log |
| GET | `/api/v1/audit-trail/by-user/{userId}` | Get audit entries by user |
| GET | `/api/v1/audit-trail/by-object` | Get audit entries by object |
| GET | `/api/v1/audit-trail/suspicious` | Get suspicious audit entries |
| POST | `/api/v1/audit-trail/export` | Export audit trail data |

---

## 4. Business Rules

1. SoD rules with mitigation_action PREVENT MUST block the role assignment transaction and MUST NOT allow the assignment to proceed without an approved exemption.
2. Access certification campaigns MUST complete all items before the campaign end_date; any uncertified items MUST be automatically escalated.
3. Configuration changes to security-sensitive objects MUST require approval (approval_required = 1) and MUST NOT be auto-approved.
4. Control violations with severity CRITICAL MUST trigger immediate notification to the tenant security administrator.
5. SoD rule exemptions MUST be reviewed and re-certified at least every 90 days; expired exemptions MUST be flagged as new violations.
6. Audit trail records MUST be immutable and MUST NOT be deleted or modified after creation.
7. The system MUST perform impact assessment on all configuration changes before they are applied, identifying downstream dependencies.
8. Certification items with decision NOT_SURE MUST be escalated to the campaign reviewer within 48 hours.
9. Violations that are waived MUST include a waived_reason and MUST be re-evaluated during the next policy evaluation cycle.
10. Audit entries classified as SUSPICIOUS or CRITICAL MUST be retained for a minimum of 7 years for SOX compliance.
11. The system SHOULD auto-detect SoD conflicts when role assignments are imported in bulk and SHOULD generate individual violations per conflict.
12. Campaign progress metrics (total_items, certified_items, revoked_items) MUST be recalculated in real-time as decisions are recorded.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package hcmcontrols.v1;

service AdvancedHCMControlsService {
    // Policies
    rpc CreatePolicy(CreatePolicyRequest) returns (CreatePolicyResponse);
    rpc GetPolicy(GetPolicyRequest) returns (GetPolicyResponse);
    rpc ListPolicies(ListPoliciesRequest) returns (ListPoliciesResponse);
    rpc EvaluatePolicy(EvaluatePolicyRequest) returns (EvaluatePolicyResponse);

    // SoD Rules
    rpc CreateSodRule(CreateSodRuleRequest) returns (CreateSodRuleResponse);
    rpc ListSodRules(ListSodRulesRequest) returns (ListSodRulesResponse);
    rpc CheckSodConflict(CheckSodConflictRequest) returns (CheckSodConflictResponse);
    rpc SimulateRoleChange(SimulateRoleChangeRequest) returns (SimulateRoleChangeResponse);

    // Certifications
    rpc CreateCampaign(CreateCampaignRequest) returns (CreateCampaignResponse);
    rpc GetCampaign(GetCampaignRequest) returns (GetCampaignResponse);
    rpc ListCampaigns(ListCampaignsRequest) returns (ListCampaignsResponse);
    rpc LaunchCampaign(LaunchCampaignRequest) returns (LaunchCampaignResponse);
    rpc CertifyItem(CertifyItemRequest) returns (CertifyItemResponse);
    rpc RevokeItem(RevokeItemRequest) returns (RevokeItemResponse);
    rpc GetCampaignProgress(GetCampaignProgressRequest) returns (GetCampaignProgressResponse);

    // Configuration Changes
    rpc ListConfigChanges(ListConfigChangesRequest) returns (ListConfigChangesResponse);
    rpc ApproveConfigChange(ApproveConfigChangeRequest) returns (ApproveConfigChangeResponse);
    rpc RejectConfigChange(RejectConfigChangeRequest) returns (RejectConfigChangeResponse);
    rpc GetImpactAnalysis(GetImpactAnalysisRequest) returns (GetImpactAnalysisResponse);

    // Violations
    rpc ListViolations(ListViolationsRequest) returns (ListViolationsResponse);
    rpc ResolveViolation(ResolveViolationRequest) returns (ResolveViolationResponse);
    rpc WaiveViolation(WaiveViolationRequest) returns (WaiveViolationResponse);
    rpc GetViolationSummary(GetViolationSummaryRequest) returns (GetViolationSummaryResponse);

    // Audit Trail
    rpc GetAuditTrail(GetAuditTrailRequest) returns (GetAuditTrailResponse);
    rpc GetAuditByUser(GetAuditByUserRequest) returns (GetAuditByUserResponse);
    rpc GetSuspiciousAuditEntries(GetSuspiciousAuditEntriesRequest) returns (GetSuspiciousAuditEntriesResponse);
    rpc ExportAuditTrail(ExportAuditTrailRequest) returns (ExportAuditTrailResponse);
}

message ControlPolicy {
    string id = 1;
    string tenant_id = 2;
    string policy_name = 3;
    string policy_type = 4;
    string description = 5;
    string severity = 6;
    bool is_active = 7;
    string effective_from = 8;
    string effective_to = 9;
    string evaluation_frequency = 10;
}

message CreatePolicyRequest {
    string tenant_id = 1;
    string policy_name = 2;
    string policy_type = 3;
    string description = 4;
    string severity = 5;
    string effective_from = 6;
    string evaluation_frequency = 7;
    string created_by = 8;
}

message CreatePolicyResponse {
    ControlPolicy policy = 1;
}

message GetPolicyRequest {
    string tenant_id = 1;
    string policy_id = 2;
}

message GetPolicyResponse {
    ControlPolicy policy = 1;
}

message ListPoliciesRequest {
    string tenant_id = 1;
    string policy_type = 2;
    bool active_only = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPoliciesResponse {
    repeated ControlPolicy policies = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message EvaluatePolicyRequest {
    string tenant_id = 1;
    string policy_id = 2;
    string evaluated_by = 3;
}

message EvaluatePolicyResponse {
    string policy_id = 1;
    int32 violations_found = 2;
    repeated string violation_ids = 3;
}

message SodRule {
    string id = 1;
    string tenant_id = 2;
    string policy_id = 3;
    string rule_name = 4;
    string role_combination = 5;
    string conflict_description = 6;
    string risk_level = 7;
    string mitigation_action = 8;
    bool exemptible = 9;
}

message CreateSodRuleRequest {
    string tenant_id = 1;
    string policy_id = 2;
    string rule_name = 3;
    string role_combination = 4;
    string conflict_description = 5;
    string risk_level = 6;
    string mitigation_action = 7;
    string created_by = 8;
}

message CreateSodRuleResponse {
    SodRule rule = 1;
}

message ListSodRulesRequest {
    string tenant_id = 1;
    string policy_id = 2;
    string risk_level = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListSodRulesResponse {
    repeated SodRule rules = 1;
    string next_page_token = 2;
}

message CheckSodConflictRequest {
    string tenant_id = 1;
    string user_id = 2;
    repeated string role_ids = 3;
    string new_role_id = 4;
}

message CheckSodConflictResponse {
    bool has_conflict = 1;
    repeated SodConflict conflicts = 2;
}

message SodConflict {
    string rule_id = 1;
    string rule_name = 2;
    string risk_level = 3;
    string conflict_description = 4;
    string mitigation_action = 5;
}

message SimulateRoleChangeRequest {
    string tenant_id = 1;
    string user_id = 2;
    repeated string current_roles = 3;
    repeated string proposed_roles = 4;
}

message SimulateRoleChangeResponse {
    repeated SodConflict new_conflicts = 1;
    repeated SodConflict resolved_conflicts = 2;
    int32 net_risk_change = 3;
}

message Campaign {
    string id = 1;
    string tenant_id = 2;
    string campaign_name = 3;
    string campaign_type = 4;
    string reviewer_id = 5;
    string start_date = 6;
    string end_date = 7;
    string status = 8;
    int32 total_items = 9;
    int32 certified_items = 10;
    int32 revoked_items = 11;
}

message CreateCampaignRequest {
    string tenant_id = 1;
    string campaign_name = 2;
    string campaign_type = 3;
    string reviewer_id = 4;
    string scope = 5;
    string start_date = 6;
    string end_date = 7;
    string created_by = 8;
}

message CreateCampaignResponse {
    Campaign campaign = 1;
}

message GetCampaignRequest {
    string tenant_id = 1;
    string campaign_id = 2;
}

message GetCampaignResponse {
    Campaign campaign = 1;
}

message ListCampaignsRequest {
    string tenant_id = 1;
    string status = 2;
    string campaign_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListCampaignsResponse {
    repeated Campaign campaigns = 1;
    string next_page_token = 2;
}

message LaunchCampaignRequest {
    string tenant_id = 1;
    string campaign_id = 2;
    string launched_by = 3;
}

message LaunchCampaignResponse {
    Campaign campaign = 1;
}

message CertifyItemRequest {
    string tenant_id = 1;
    string campaign_id = 2;
    string item_id = 3;
    string certifier_id = 4;
    string decision_rationale = 5;
}

message CertifyItemResponse {
    string item_id = 1;
    string status = 2;
}

message RevokeItemRequest {
    string tenant_id = 1;
    string campaign_id = 2;
    string item_id = 3;
    string certifier_id = 4;
    string risk_comment = 5;
}

message RevokeItemResponse {
    string item_id = 1;
    string status = 2;
}

message CampaignProgress {
    string campaign_id = 1;
    int32 total_items = 2;
    int32 certified_items = 3;
    int32 revoked_items = 4;
    int32 pending_items = 5;
    int32 escalated_items = 6;
    double completion_pct = 7;
}

message GetCampaignProgressRequest {
    string tenant_id = 1;
    string campaign_id = 2;
}

message GetCampaignProgressResponse {
    CampaignProgress progress = 1;
}

message ConfigChange {
    string id = 1;
    string tenant_id = 2;
    string change_type = 3;
    string changed_object_type = 4;
    string changed_object_id = 5;
    string changed_by = 6;
    string change_timestamp = 7;
    string old_value = 8;
    string new_value = 9;
    string status = 10;
}

message ListConfigChangesRequest {
    string tenant_id = 1;
    string changed_object_type = 2;
    string status = 3;
    string date_from = 4;
    string date_to = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListConfigChangesResponse {
    repeated ConfigChange changes = 1;
    string next_page_token = 2;
}

message ApproveConfigChangeRequest {
    string tenant_id = 1;
    string change_id = 2;
    string approved_by = 3;
}

message ApproveConfigChangeResponse {
    ConfigChange change = 1;
}

message RejectConfigChangeRequest {
    string tenant_id = 1;
    string change_id = 2;
    string rejected_by = 3;
    string reason = 4;
}

message RejectConfigChangeResponse {
    ConfigChange change = 1;
}

message ImpactAnalysis {
    string change_id = 1;
    repeated string affected_objects = 2;
    repeated string affected_users = 3;
    string risk_assessment = 4;
}

message GetImpactAnalysisRequest {
    string tenant_id = 1;
    string change_id = 2;
}

message GetImpactAnalysisResponse {
    ImpactAnalysis analysis = 1;
}

message Violation {
    string id = 1;
    string tenant_id = 2;
    string policy_id = 3;
    string violation_type = 4;
    string violated_by_user = 5;
    string severity = 6;
    string description = 7;
    string resolution_status = 8;
}

message ListViolationsRequest {
    string tenant_id = 1;
    string policy_id = 2;
    string resolution_status = 3;
    string severity = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListViolationsResponse {
    repeated Violation violations = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ResolveViolationRequest {
    string tenant_id = 1;
    string violation_id = 2;
    string resolved_by = 3;
    string resolution_notes = 4;
}

message ResolveViolationResponse {
    Violation violation = 1;
}

message WaiveViolationRequest {
    string tenant_id = 1;
    string violation_id = 2;
    string waived_by = 3;
    string waived_reason = 4;
}

message WaiveViolationResponse {
    Violation violation = 1;
}

message ViolationSummary {
    int32 total_open = 1;
    int32 total_resolved = 2;
    int32 total_waived = 3;
    int32 critical_open = 4;
}

message GetViolationSummaryRequest {
    string tenant_id = 1;
    string policy_id = 2;
}

message GetViolationSummaryResponse {
    ViolationSummary summary = 1;
}

message AuditEntry {
    string id = 1;
    string tenant_id = 2;
    string audit_event_type = 3;
    string actor_id = 4;
    string target_object_type = 5;
    string target_object_id = 6;
    string action = 7;
    string timestamp = 8;
    string classification = 9;
}

message GetAuditTrailRequest {
    string tenant_id = 1;
    string event_type = 2;
    string date_from = 3;
    string date_to = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message GetAuditTrailResponse {
    repeated AuditEntry entries = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetAuditByUserRequest {
    string tenant_id = 1;
    string user_id = 2;
    string date_from = 3;
    string date_to = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message GetAuditByUserResponse {
    repeated AuditEntry entries = 1;
    string next_page_token = 2;
}

message GetSuspiciousAuditEntriesRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetSuspiciousAuditEntriesResponse {
    repeated AuditEntry entries = 1;
    string next_page_token = 2;
}

message ExportAuditTrailRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
    string format = 4;
}

message ExportAuditTrailResponse {
    string export_reference = 1;
    int32 total_entries = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee records, role assignments, org structure | Validate SoD rules and scope certifications |
| `security-service` | User roles, entitlements, access grants | Evaluate access certifications and SoD conflicts |
| `workflow-service` | Approval workflow definitions | Route change approvals and violation remediations |
| `auth-service` | Session data, IP addresses, user agents | Enrich audit trail entries |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `security-service` | Access revocation decisions | Execute role and entitlement revocations |
| `notification-service` | Violation alerts, certification reminders, approval requests | Compliance communications |
| `reporting-service` | Violation metrics, audit summaries, certification progress | Compliance dashboards |
| `document-service` | Audit trail exports, compliance reports | Store generated compliance documents |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ViolationDetected` | `hcmcontrols.violation.detected` | `{ tenant_id, violation_id, policy_id, violation_type, severity, violated_by_user }` | Published when a control violation is detected |
| `ConfigurationChanged` | `hcmcontrols.config.changed` | `{ tenant_id, change_id, change_type, changed_object_type, changed_object_id, changed_by, status }` | Published when a configuration change is recorded |
| `CertificationCampaignLaunched` | `hcmcontrols.certification.launched` | `{ tenant_id, campaign_id, campaign_name, campaign_type, reviewer_id, total_items }` | Published when a certification campaign is launched |
| `AccessRevoked` | `hcmcontrols.access.revoked` | `{ tenant_id, campaign_id, user_id, role_id, entitlement_id, certifier_id }` | Published when access is revoked through certification |
| `SodConflictDetected` | `hcmcontrols.sod.conflict` | `{ tenant_id, rule_id, user_id, conflict_description, risk_level, mitigation_action }` | Published when an SoD conflict is identified |
