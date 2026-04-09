# 177 - Access Governor Service Specification

## 1. Domain Overview

Access Governor provides access certification, segregation of duties (SoD) analysis, privileged access management, and access review workflows for Fusion applications. Supports periodic access reviews, role mining, SoD rule definition, conflict detection, remediation workflows, and compliance reporting. Enables security administrators and auditors to certify user access, detect SoD violations, manage privileged accounts, and maintain compliance with SOX, ICFR, and other regulatory requirements. Distinct from Auth & Security (05) which handles authentication; Access Governor focuses on authorization governance and compliance.

**Bounded Context:** Access Governance & Compliance
**Service Name:** `access-governor-service`
**Database:** `data/access_governor.db`
**HTTP Port:** 8195 | **gRPC Port:** 9195

---

## 2. Database Schema

### 2.1 SoD Rules
```sql
CREATE TABLE ag_sod_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    description TEXT NOT NULL,
    duty_a TEXT NOT NULL,                    -- First duty/function
    duty_b TEXT NOT NULL,                    -- Conflicting duty/function
    risk_level TEXT NOT NULL CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    regulation TEXT,                         -- SOX, ICFR, GDPR, etc.
    mitigating_control TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_ag_sod_risk ON ag_sod_rules(tenant_id, risk_level);
```

### 2.2 Access Certification Campaigns
```sql
CREATE TABLE ag_campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_code TEXT NOT NULL,
    campaign_name TEXT NOT NULL,
    campaign_type TEXT NOT NULL CHECK(campaign_type IN ('USER_ACCESS','ROLE_ACCESS','ENTITLEMENT','PRIVILEGED','ADHOC')),
    description TEXT,
    scope TEXT NOT NULL,                     -- JSON: what's being reviewed
    reviewers TEXT NOT NULL,                 -- JSON: assigned reviewers
    deadline TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ACTIVE','IN_REVIEW','COMPLETED','CANCELLED')),
    total_items INTEGER NOT NULL DEFAULT 0,
    certified_items INTEGER NOT NULL DEFAULT 0,
    revoked_items INTEGER NOT NULL DEFAULT 0,
    exception_items INTEGER NOT NULL DEFAULT 0,
    completion_pct REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, campaign_code)
);

CREATE INDEX idx_ag_campaign_tenant ON ag_campaigns(tenant_id, status);
```

### 2.3 Certification Items
```sql
CREATE TABLE ag_cert_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    user_name TEXT NOT NULL,
    resource_type TEXT NOT NULL CHECK(resource_type IN ('ROLE','PERMISSION','ENTITLEMENT','APPLICATION','DATA_ACCESS')),
    resource_id TEXT NOT NULL,
    resource_name TEXT NOT NULL,
    assigned_date TEXT NOT NULL,
    last_used_date TEXT,
    usage_frequency TEXT CHECK(usage_frequency IN ('FREQUENT','OCCASIONAL','RARE','NEVER','UNKNOWN')),
    reviewer_id TEXT NOT NULL,
    decision TEXT CHECK(decision IN ('CERTIFY','REVOKE','EXCEPTION','NEED_INFO')),
    decision_reason TEXT,
    decided_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','CERTIFIED','REVOKED','EXCEPTION','ESCALATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (campaign_id) REFERENCES ag_campaigns(id) ON DELETE CASCADE
);

CREATE INDEX idx_ag_cert_campaign ON ag_cert_items(campaign_id, status);
CREATE INDEX idx_ag_cert_reviewer ON ag_cert_items(reviewer_id, status);
```

### 2.4 SoD Violations
```sql
CREATE TABLE ag_violations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    user_name TEXT NOT NULL,
    duty_a TEXT NOT NULL,
    duty_b TEXT NOT NULL,
    detection_type TEXT NOT NULL CHECK(detection_type IN ('PROACTIVE','REACTIVE','PERIODIC_SCAN')),
    risk_level TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','MITIGATED','RESOLVED','ACCEPTED','EXEMPTED')),
    mitigation_control TEXT,
    mitigated_by TEXT,
    mitigated_at TEXT,
    resolved_by TEXT,
    resolved_at TEXT,
    resolution TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (rule_id) REFERENCES ag_sod_rules(id) ON DELETE RESTRICT
);

CREATE INDEX idx_ag_viol_user ON ag_violations(user_id, status);
CREATE INDEX idx_ag_viol_risk ON ag_violations(tenant_id, risk_level, status);
CREATE INDEX idx_ag_viol_rule ON ag_violations(rule_id);
```

### 2.5 Privileged Access
```sql
CREATE TABLE ag_privileged_access (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    privilege_type TEXT NOT NULL CHECK(privilege_type IN ('ADMIN','SUPERUSER','DBA','SERVICE_ACCOUNT','EMERGENCY','CUSTOM')),
    resource_scope TEXT NOT NULL,            -- JSON: what resources
    granted_by TEXT NOT NULL,
    granted_at TEXT NOT NULL,
    expiry_date TEXT,
    justification TEXT NOT NULL,
    approval_required INTEGER NOT NULL DEFAULT 1,
    approved_by TEXT,
    session_duration_minutes INTEGER,
    check_out INTEGER NOT NULL DEFAULT 0,   -- Checkout-based access
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','REVOKED','CHECKED_OUT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, user_id, privilege_type, resource_scope)
);

CREATE INDEX idx_ag_priv_user ON ag_privileged_access(user_id, status);
```

---

## 3. API Endpoints

### 3.1 SoD Rules
| Method | Path | Description |
|--------|------|-------------|
| POST | `/access-governor/v1/sod-rules` | Create SoD rule |
| GET | `/access-governor/v1/sod-rules` | List rules |
| PUT | `/access-governor/v1/sod-rules/{id}` | Update rule |
| POST | `/access-governor/v1/sod-rules/scan` | Run SoD violation scan |

### 3.2 Campaigns
| Method | Path | Description |
|--------|------|-------------|
| POST | `/access-governor/v1/campaigns` | Create campaign |
| GET | `/access-governor/v1/campaigns` | List campaigns |
| POST | `/access-governor/v1/campaigns/{id}/activate` | Activate campaign |
| GET | `/access-governor/v1/campaigns/{id}/progress` | Campaign progress |

### 3.3 Certification
| Method | Path | Description |
|--------|------|-------------|
| GET | `/access-governor/v1/campaigns/{id}/items` | List pending items |
| POST | `/access-governor/v1/items/{id}/certify` | Certify access |
| POST | `/access-governor/v1/items/{id}/revoke` | Revoke access |
| POST | `/access-governor/v1/items/{id}/exception` | Grant exception |

### 3.4 Violations & Privileged Access
| Method | Path | Description |
|--------|------|-------------|
| GET | `/access-governor/v1/violations` | List violations |
| POST | `/access-governor/v1/violations/{id}/mitigate` | Apply mitigation |
| GET | `/access-governor/v1/privileged-access` | List privileged access |
| POST | `/access-governor/v1/privileged-access` | Grant privileged access |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `agov.violation.detected` | `{ violation_id, user_id, rule_id, risk }` | SoD violation detected |
| `agov.access.revoked` | `{ user_id, resource, campaign_id }` | Access revoked |
| `agov.campaign.completed` | `{ campaign_id, certified, revoked }` | Campaign completed |
| `agov.privileged.expiring` | `{ user_id, privilege_type, expiry }` | Privileged access expiring |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `user.role.changed` | Auth & Security | Check for SoD violations |
| `user.created` | Auth & Security | Include in next campaign |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.access_governor.v1;

service AccessGovernorService {
    rpc GetSodRule(GetSodRuleRequest) returns (GetSodRuleResponse);
    rpc CreateCampaign(CreateCampaignRequest) returns (CreateCampaignResponse);
    rpc GetCampaign(GetCampaignRequest) returns (GetCampaignResponse);
    rpc CertifyItem(CertifyItemRequest) returns (CertifyItemResponse);
    rpc ScanViolations(ScanViolationsRequest) returns (ScanViolationsResponse);
    rpc GetViolation(GetViolationRequest) returns (GetViolationResponse);
}

// SoD Rule messages
message GetSodRuleRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetSodRuleResponse {
    AgSodRule data = 1;
}

message AgSodRule {
    string id = 1;
    string tenant_id = 2;
    string rule_code = 3;
    string rule_name = 4;
    string description = 5;
    string duty_a = 6;
    string duty_b = 7;
    string risk_level = 8;
    string regulation = 9;
    string mitigating_control = 10;
    string status = 11;
    string created_at = 12;
    string updated_at = 13;
}

// Campaign messages
message CreateCampaignRequest {
    string tenant_id = 1;
    string campaign_code = 2;
    string campaign_name = 3;
    string campaign_type = 4;
    string description = 5;
    string scope = 6;
    string reviewers = 7;
    string deadline = 8;
}

message CreateCampaignResponse {
    AgCampaign data = 1;
}

message GetCampaignRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetCampaignResponse {
    AgCampaign data = 1;
}

message AgCampaign {
    string id = 1;
    string tenant_id = 2;
    string campaign_code = 3;
    string campaign_name = 4;
    string campaign_type = 5;
    string description = 6;
    string scope = 7;
    string reviewers = 8;
    string deadline = 9;
    string status = 10;
    int32 total_items = 11;
    int32 certified_items = 12;
    int32 revoked_items = 13;
    int32 exception_items = 14;
    double completion_pct = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Certification messages
message CertifyItemRequest {
    string tenant_id = 1;
    string id = 2;
    string decision = 3;
    string decision_reason = 4;
}

message CertifyItemResponse {
    string id = 1;
    string status = 2;
    string decision = 3;
}

// Violation messages
message ScanViolationsRequest {
    string tenant_id = 1;
    bool full_scan = 2;
}

message ScanViolationsResponse {
    int32 violations_found = 1;
    int32 new_violations = 2;
}

message GetViolationRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetViolationResponse {
    AgViolation data = 1;
}

message AgViolation {
    string id = 1;
    string tenant_id = 2;
    string rule_id = 3;
    string user_id = 4;
    string user_name = 5;
    string duty_a = 6;
    string duty_b = 7;
    string detection_type = 8;
    string risk_level = 9;
    string status = 10;
    string mitigation_control = 11;
    string mitigated_by = 12;
    string mitigated_at = 13;
    string created_at = 14;
    string updated_at = 15;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | ag_sod_rules | -- |
| V002 | ag_campaigns | -- |
| V003 | ag_cert_items | V002 |
| V004 | ag_violations | V001 |
| V005 | ag_privileged_access | -- |

---

## 7. Business Rules

1. **SoD Scanning**: Automatic SoD scan triggered on role changes; periodic full scan weekly
2. **Campaign Cadence**: Access certification campaigns run quarterly at minimum
3. **Reviewer Assignment**: Managers review direct reports; security admin reviews privileged access
4. **Revocation Execution**: Revoked access auto-executed through Auth & Security integration
5. **Mitigation Tracking**: All mitigations tracked with expiry; re-review required
6. **Emergency Access**: Emergency privileged access auto-revoked after configured duration

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| auth-service | `GetUserRoles` | User roles and permissions |
| workflow-service | `SubmitApproval` | Certification approval workflows |
| notification-service | `SendNotification` | Review reminders |
| hr-service | `GetManager` | Manager-employee relationships |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| risk-service | `GetViolations` | Compliance reporting |
| reporting-service | `GetCampaignProgress` | Compliance audit reports |
| auth-service | `RevokeAccess` | Execute access revocations |
