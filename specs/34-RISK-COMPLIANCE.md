# 34 - Risk Management & Compliance Service Specification

## 1. Domain Overview

Risk Management & Compliance provides separation of duties (SoD) enforcement, user access certification campaigns, audit policy management, audit event logging, transaction monitoring controls, and compliance reporting. Prevents unauthorized access combinations through SoD rule checks, manages periodic access recertification workflows, captures audit trails for all configuration and transaction changes, applies real-time transaction controls during document submission, and generates evidence packages for regulatory compliance (SOX, GDPR, HIPAA, PCI). Integrates with Auth for user/role data, AP/AR/GL for transaction monitoring, and Workflow for approval processes.

**Bounded Context:** Risk Management, Compliance & Governance
**Service Name:** `risk-service`
**Database:** `data/risk.db`
**HTTP Port:** 8061 | **gRPC Port:** 9061

---

## 2. Database Schema

### 2.1 Separation of Duties Rules
```sql
CREATE TABLE risk_sod_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    duty_one TEXT NOT NULL,                  -- Permission / duty identifier
    duty_two TEXT NOT NULL,                  -- Conflicting permission / duty identifier
    risk_level TEXT NOT NULL
        CHECK(risk_level IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    description TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    mitigating_control TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_sod_rules_tenant_active ON risk_sod_rules(tenant_id, is_active);
CREATE INDEX idx_sod_rules_tenant_risk ON risk_sod_rules(tenant_id, risk_level);
```

### 2.2 SoD Violations
```sql
CREATE TABLE risk_sod_violations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    duty_one TEXT NOT NULL,
    duty_two TEXT NOT NULL,
    detected_at TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','MITIGATED','EXEMPT','RESOLVED')),
    mitigated_by TEXT,
    mitigation_reason TEXT,
    exemption_expiry TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (rule_id) REFERENCES risk_sod_rules(id) ON DELETE RESTRICT
);

CREATE INDEX idx_sod_violations_tenant_user ON risk_sod_violations(tenant_id, user_id);
CREATE INDEX idx_sod_violations_tenant_status ON risk_sod_violations(tenant_id, status);
CREATE INDEX idx_sod_violations_tenant_rule ON risk_sod_violations(tenant_id, rule_id);
```

### 2.3 Access Certifications
```sql
CREATE TABLE risk_access_certifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_name TEXT NOT NULL,
    campaign_type TEXT NOT NULL
        CHECK(campaign_type IN ('USER_ACCESS','ROLE_ACCESS','SOD_REVIEW')),
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','OVERDUE')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    reviewer_id TEXT NOT NULL,
    certifier_id TEXT,
    total_items INTEGER NOT NULL DEFAULT 0,
    completed_items INTEGER NOT NULL DEFAULT 0,
    exceptions_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, campaign_name, start_date)
);

CREATE INDEX idx_certifications_tenant_status ON risk_access_certifications(tenant_id, status);
CREATE INDEX idx_certifications_tenant_dates ON risk_access_certifications(tenant_id, start_date, end_date);
```

### 2.4 Certification Items
```sql
CREATE TABLE risk_certification_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    certification_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    role_id TEXT,
    permission TEXT NOT NULL,
    decision TEXT
        CHECK(decision IN ('CERTIFY','REVOKE','MODIFY','EXCEPTION')),
    decision_reason TEXT,
    decided_by TEXT,
    decided_at TEXT,
    risk_flag INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (certification_id) REFERENCES risk_access_certifications(id) ON DELETE CASCADE
);

CREATE INDEX idx_cert_items_tenant_cert ON risk_certification_items(tenant_id, certification_id);
CREATE INDEX idx_cert_items_tenant_user ON risk_certification_items(tenant_id, user_id);
CREATE INDEX idx_cert_items_tenant_decision ON risk_certification_items(tenant_id, decision);
```

### 2.5 Audit Policies
```sql
CREATE TABLE risk_audit_policies (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_name TEXT NOT NULL,
    policy_type TEXT NOT NULL
        CHECK(policy_type IN ('TRANSACTION','CONFIGURATION','ACCESS','REPORT')),
    entity_type TEXT NOT NULL,
    condition_expression TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'INFO'
        CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    is_active INTEGER NOT NULL DEFAULT 1,
    notification_enabled INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, policy_name)
);

CREATE INDEX idx_audit_policies_tenant_active ON risk_audit_policies(tenant_id, is_active);
CREATE INDEX idx_audit_policies_tenant_type ON risk_audit_policies(tenant_id, policy_type);
```

### 2.6 Audit Events
```sql
CREATE TABLE risk_audit_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    policy_id TEXT,
    event_type TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    action TEXT NOT NULL,
    user_id TEXT NOT NULL,
    old_value TEXT,
    new_value TEXT,
    ip_address TEXT,
    user_agent TEXT,
    severity TEXT NOT NULL DEFAULT 'INFO'
        CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    is_flagged INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (policy_id) REFERENCES risk_audit_policies(id) ON DELETE SET NULL
);

CREATE INDEX idx_audit_events_tenant_entity ON risk_audit_events(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_events_tenant_user ON risk_audit_events(tenant_id, user_id);
CREATE INDEX idx_audit_events_tenant_time ON risk_audit_events(tenant_id, created_at);
CREATE INDEX idx_audit_events_tenant_flagged ON risk_audit_events(tenant_id, is_flagged);
CREATE INDEX idx_audit_events_tenant_severity ON risk_audit_events(tenant_id, severity);
```

### 2.7 Transaction Controls
```sql
CREATE TABLE risk_transaction_controls (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    control_name TEXT NOT NULL,
    control_type TEXT NOT NULL
        CHECK(control_type IN ('AMOUNT_THRESHOLD','FREQUENCY','PATTERN_ANOMALY','BLACKLIST')),
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('AP_INVOICE','AP_PAYMENT','AR_INVOICE','GL_JOURNAL','EXPENSE')),
    condition_json TEXT NOT NULL,
    action TEXT NOT NULL
        CHECK(action IN ('BLOCK','WARN','REVIEW','NOTIFY')),
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, control_name)
);

CREATE INDEX idx_transaction_controls_tenant_active ON risk_transaction_controls(tenant_id, is_active);
CREATE INDEX idx_transaction_controls_tenant_entity ON risk_transaction_controls(tenant_id, entity_type);
```

### 2.8 Transaction Alerts
```sql
CREATE TABLE risk_transaction_alerts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    control_id TEXT NOT NULL,
    transaction_type TEXT NOT NULL,
    transaction_id TEXT NOT NULL,
    alert_level TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(alert_level IN ('INFO','WARNING','CRITICAL')),
    alert_message TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','INVESTIGATING','RESOLVED','FALSE_POSITIVE')),
    investigated_by TEXT,
    resolution_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (control_id) REFERENCES risk_transaction_controls(id) ON DELETE RESTRICT
);

CREATE INDEX idx_transaction_alerts_tenant_status ON risk_transaction_alerts(tenant_id, status);
CREATE INDEX idx_transaction_alerts_tenant_control ON risk_transaction_alerts(tenant_id, control_id);
CREATE INDEX idx_transaction_alerts_tenant_level ON risk_transaction_alerts(tenant_id, alert_level);
CREATE INDEX idx_transaction_alerts_tenant_tx ON risk_transaction_alerts(tenant_id, transaction_type, transaction_id);
```

### 2.9 Compliance Reports
```sql
CREATE TABLE risk_compliance_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_name TEXT NOT NULL,
    report_type TEXT NOT NULL
        CHECK(report_type IN ('SOX','GDPR','HIPAA','PCI','INTERNAL')),
    framework TEXT,
    frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(frequency IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY')),
    last_run_at TEXT,
    next_run_at TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, report_name)
);

CREATE INDEX idx_compliance_reports_tenant_type ON risk_compliance_reports(tenant_id, report_type);
CREATE INDEX idx_compliance_reports_tenant_status ON risk_compliance_reports(tenant_id, status);
```

---

## 3. REST API Endpoints

```
# Separation of Duties
GET/POST      /api/v1/risk/sod/rules                        Permission: risk.sod.read/create
GET/PUT       /api/v1/risk/sod/rules/{id}                    Permission: risk.sod.read/update
POST          /api/v1/risk/sod/analyze-user                  Permission: risk.sod.analyze
GET           /api/v1/risk/sod/violations                    Permission: risk.sod.read
POST          /api/v1/risk/sod/violations/{id}/mitigate      Permission: risk.sod.mitigate
POST          /api/v1/risk/sod/violations/{id}/exempt        Permission: risk.sod.exempt

# Access Certifications
GET/POST      /api/v1/risk/certifications                    Permission: risk.certifications.read/create
GET/PUT       /api/v1/risk/certifications/{id}               Permission: risk.certifications.read/update
POST          /api/v1/risk/certifications/{id}/start         Permission: risk.certifications.update
POST          /api/v1/risk/certifications/{id}/complete      Permission: risk.certifications.update
GET           /api/v1/risk/certifications/{id}/items         Permission: risk.certifications.read
POST          /api/v1/risk/certifications/{id}/items/{iid}/decide Permission: risk.certifications.update

# Audit Policies
GET/POST      /api/v1/risk/audit/policies                    Permission: risk.audit.read/create
GET/PUT       /api/v1/risk/audit/policies/{id}               Permission: risk.audit.read/update

# Audit Events
GET           /api/v1/risk/audit/events                      Permission: risk.audit.read
GET           /api/v1/risk/audit/events/{id}                 Permission: risk.audit.read
POST          /api/v1/risk/audit/query                       Permission: risk.audit.query

# Transaction Controls
GET/POST      /api/v1/risk/transaction-controls              Permission: risk.controls.read/create
GET/PUT       /api/v1/risk/transaction-controls/{id}         Permission: risk.controls.read/update

# Transaction Alerts
GET           /api/v1/risk/transaction-controls/alerts       Permission: risk.controls.read
POST          /api/v1/risk/transaction-controls/alerts/{id}/investigate Permission: risk.controls.update
POST          /api/v1/risk/transaction-controls/alerts/{id}/resolve     Permission: risk.controls.update

# Compliance Reports
GET/POST      /api/v1/risk/compliance/reports                Permission: risk.compliance.read/create
GET/PUT       /api/v1/risk/compliance/reports/{id}           Permission: risk.compliance.read/update
POST          /api/v1/risk/compliance/reports/{id}/run       Permission: risk.compliance.execute
GET           /api/v1/risk/compliance/reports/{id}/results   Permission: risk.compliance.read

# Dashboard
GET           /api/v1/risk/dashboard                         Permission: risk.dashboard.view
```

---

## 4. Business Rules

### 4.1 Separation of Duties
1. SoD rules MUST be checked on every role/permission assignment
2. Users cannot be assigned duties that violate CRITICAL SoD rules (hard block)
3. HIGH SoD violations can be mitigated with documented compensating controls
4. SoD analysis returns a comprehensive conflict report for a given user
5. Exemptions have a mandatory expiry date and require justification

### 4.2 Access Certification
1. Access certification campaigns MUST be completed within the defined period
2. Uncertified access is automatically flagged for revocation review after campaign end date
3. Campaign status transitions: PLANNED -> IN_PROGRESS -> COMPLETED (or OVERDUE)
4. Each certification item requires a decision: CERTIFY, REVOKE, MODIFY, or EXCEPTION
5. REVOKE decisions trigger immediate role/permission removal via Auth service

### 4.3 Audit Logging
1. All configuration changes MUST be audit-logged
2. Audit events capture old_value and new_value for change tracking
3. Events matching audit policy conditions are automatically flagged
4. Audit event queries support filtering by entity, user, time range, and severity
5. Audit log entries are immutable and cannot be modified or deleted

### 4.4 Transaction Controls
1. Transaction controls evaluate in real-time during document submission
2. BLOCK action prevents document submission entirely
3. WARN action allows submission but flags for review
4. REVIEW action routes document to a designated reviewer queue
5. NOTIFY action sends alerts to configured recipients without blocking
6. Controls can target specific entity types: AP invoices, payments, AR invoices, GL journals, expenses

### 4.5 Compliance Reporting
1. Compliance reports generate evidence packages for auditors
2. Reports run on defined schedules (daily, weekly, monthly, quarterly)
3. Results include findings summary, flagged transactions, and remediation status
4. Supported frameworks: SOX, GDPR, HIPAA, PCI, and custom internal policies

### 4.6 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `risk.sod.violation.detected` | SoD conflict found | Auth, Workflow |
| `risk.audit.event.created` | Audit event captured | Reporting |
| `risk.transaction.alert.created` | Transaction control triggered | AP, AR, GL, Expense |
| `risk.certification.overdue` | Campaign past end date | Auth, Workflow |
| `risk.compliance.report.completed` | Report run finishes | Reporting |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.risk.v1;

service RiskService {
    rpc CheckSodCompliance(CheckSodComplianceRequest) returns (CheckSodComplianceResponse);
    rpc LogAuditEvent(LogAuditEventRequest) returns (LogAuditEventResponse);
    rpc EvaluateTransaction(EvaluateTransactionRequest) returns (EvaluateTransactionResponse);
    rpc GetUserRiskProfile(GetUserRiskProfileRequest) returns (GetUserRiskProfileResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Consumes From
- **Auth:** User and role data for SoD checks, access certifications, and risk profiling
- **AP:** Invoice and payment data for transaction monitoring
- **AR:** Invoice data for transaction monitoring
- **GL:** Journal entry data for transaction monitoring
- **Expense:** Expense report data for transaction monitoring
- **Workflow:** Approval status for mitigation and certification workflows

### 6.2 Publishes To
- **Auth:** SoD violation blocks (prevent role assignment), revocation requests from certifications
- **AP/AR/GL/Expense:** Transaction control BLOCK/WARN responses during submission
- **Workflow:** Approval requests for mitigations, certification reviews, alert investigations
- **Reporting:** Audit event streams, compliance report results, risk dashboard data

---

## 7. Migrations

1. V001: `risk_sod_rules`
2. V002: `risk_sod_violations`
3. V003: `risk_access_certifications`
4. V004: `risk_certification_items`
5. V005: `risk_audit_policies`
6. V006: `risk_audit_events`
7. V007: `risk_transaction_controls`
8. V008: `risk_transaction_alerts`
9. V009: `risk_compliance_reports`
10. V010: Triggers for `updated_at`
