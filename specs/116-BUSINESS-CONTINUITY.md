# 116 - Business Continuity Service Specification

## 1. Domain Overview

The Business Continuity Planning service provides ERP capability for disaster recovery and business continuity, ensuring financial operations can continue during disruptions. It includes backup processing plans, failover configuration, recovery procedures, continuity testing, impact analysis, and regulatory compliance for operational resilience.

**Bounded Context:** Business Continuity & Disaster Recovery
**Service Name:** `continuity-service`
**Database:** `data/continuity.db`
**HTTP Port:** 8153 | **gRPC Port:** 9153

---

## 2. Database Schema

### 2.1 Continuity Plans
```sql
CREATE TABLE continuity_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('DISASTER_RECOVERY','BUSINESS_CONTINUITY','INCIDENT_RESPONSE','PANDEMIC','CYBER_INCIDENT')),
    scope TEXT NOT NULL CHECK(scope IN ('FULL_SYSTEM','FINANCIAL_OPERATIONS','PAYROLL','REPORTING','CRITICAL_SERVICES')),
    description TEXT,
    owner_id TEXT NOT NULL,
    approver_id TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','APPROVED','ACTIVE','TESTING','OBSOLETE')),
    last_reviewed TEXT,
    next_review_date TEXT NOT NULL,
    plan_version INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_name)
);

CREATE INDEX idx_contp_tenant_status ON continuity_plans(tenant_id, status);
CREATE INDEX idx_contp_tenant_type ON continuity_plans(tenant_id, plan_type);
CREATE INDEX idx_contp_tenant_review ON continuity_plans(tenant_id, next_review_date);
```

### 2.2 Recovery Objectives
```sql
CREATE TABLE recovery_objectives (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    service_name TEXT NOT NULL,
    rto_hours INTEGER NOT NULL,
    rpo_hours INTEGER NOT NULL,
    criticality TEXT NOT NULL CHECK(criticality IN ('CRITICAL','HIGH','MEDIUM','LOW')),
    current_rto_hours INTEGER,
    current_rpo_hours INTEGER,
    gap_analysis TEXT,
    mitigation_plan TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES continuity_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_ro_tenant_plan ON recovery_objectives(tenant_id, plan_id);
CREATE INDEX idx_ro_tenant_criticality ON recovery_objectives(tenant_id, criticality);
```

### 2.3 Continuity Procedures
```sql
CREATE TABLE continuity_procedures (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    procedure_name TEXT NOT NULL,
    procedure_type TEXT NOT NULL CHECK(procedure_type IN ('FAILOVER','RESTORATION','MANUAL_PROCESS','COMMUNICATION','ESCALATION')),
    description TEXT NOT NULL,
    step_by_step TEXT NOT NULL,
    estimated_duration_minutes INTEGER NOT NULL,
    responsible_role_id TEXT NOT NULL,
    backup_role_id TEXT,
    prerequisites TEXT,  -- JSON
    dependencies TEXT,  -- JSON
    sequence_order INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES continuity_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_cp_tenant_plan ON continuity_procedures(tenant_id, plan_id);
CREATE INDEX idx_cp_tenant_type ON continuity_procedures(tenant_id, procedure_type);
CREATE INDEX idx_cp_tenant_seq ON continuity_procedures(tenant_id, plan_id, sequence_order);
```

### 2.4 Continuity Contacts
```sql
CREATE TABLE continuity_contacts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    contact_name TEXT NOT NULL,
    contact_role TEXT NOT NULL,
    primary_phone TEXT,
    secondary_phone TEXT,
    email TEXT NOT NULL,
    escalation_order INTEGER NOT NULL DEFAULT 1,
    availability_schedule TEXT,  -- JSON
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES continuity_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_cc_tenant_plan ON continuity_contacts(tenant_id, plan_id);
CREATE INDEX idx_cc_tenant_primary ON continuity_contacts(tenant_id, is_primary);
```

### 2.5 Continuity Tests
```sql
CREATE TABLE continuity_tests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    test_name TEXT NOT NULL,
    test_type TEXT NOT NULL CHECK(test_type IN ('TABLETOP','FULL_SIMULATION','PARTIAL','COMPONENT')),
    scheduled_date TEXT NOT NULL,
    executed_date TEXT,
    executed_by TEXT,
    results_summary TEXT,
    issues_found TEXT,  -- JSON
    rto_achieved_hours DECIMAL(10,2),
    rpo_achieved_hours DECIMAL(10,2),
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','EXECUTING','COMPLETED','FAILED')),
    pass_fail TEXT CHECK(pass_fail IN ('PASS','FAIL','PARTIAL')),
    follow_up_actions TEXT,  -- JSON

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES continuity_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_ct_tenant_plan ON continuity_tests(tenant_id, plan_id);
CREATE INDEX idx_ct_tenant_status ON continuity_tests(tenant_id, status);
CREATE INDEX idx_ct_tenant_date ON continuity_tests(tenant_id, scheduled_date);
```

### 2.6 Impact Assessments
```sql
CREATE TABLE impact_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    scenario_type TEXT NOT NULL CHECK(scenario_type IN ('NATURAL_DISASTER','CYBER_ATTACK','SYSTEM_FAILURE','PANDEMIC','SUPPLY_DISRUPTION','KEY_PERSON')),
    affected_services TEXT NOT NULL,  -- JSON
    financial_impact_per_hour_cents INTEGER NOT NULL DEFAULT 0,
    operational_impact TEXT NOT NULL CHECK(operational_impact IN ('NONE','LOW','MEDIUM','HIGH','CRITICAL')),
    recovery_complexity TEXT NOT NULL CHECK(recovery_complexity IN ('LOW','MEDIUM','HIGH')),
    probability TEXT NOT NULL CHECK(probability IN ('LOW','MEDIUM','HIGH')),
    risk_score DECIMAL(5,2) DEFAULT 0,
    mitigation_status TEXT NOT NULL DEFAULT 'UNMITIGATED' CHECK(mitigation_status IN ('UNMITIGATED','PARTIALLY','MITIGATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES continuity_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_ia_tenant_plan ON impact_assessments(tenant_id, plan_id);
CREATE INDEX idx_ia_tenant_scenario ON impact_assessments(tenant_id, scenario_type);
CREATE INDEX idx_ia_tenant_risk ON impact_assessments(tenant_id, risk_score);
```

### 2.7 Incident Logs
```sql
CREATE TABLE incident_logs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT,
    incident_type TEXT NOT NULL,
    severity TEXT NOT NULL CHECK(severity IN ('P1','P2','P3','P4')),
    description TEXT NOT NULL,
    detected_at TEXT NOT NULL,
    detected_by TEXT,
    response_started_at TEXT,
    resolved_at TEXT,
    duration_minutes INTEGER,
    services_affected TEXT,  -- JSON
    root_cause TEXT,
    resolution_summary TEXT,
    post_mortem_completed INTEGER NOT NULL DEFAULT 0,
    lessons_learned TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES continuity_plans(id)
);

CREATE INDEX idx_il_tenant_severity ON incident_logs(tenant_id, severity);
CREATE INDEX idx_il_tenant_status ON incident_logs(tenant_id, resolved_at);
CREATE INDEX idx_il_tenant_type ON incident_logs(tenant_id, incident_type);
CREATE INDEX idx_il_tenant_date ON incident_logs(tenant_id, detected_at);
```

---

## 3. REST API Endpoints

### Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/continuity-plans` | Create a continuity plan |
| GET | `/api/v1/continuity-plans` | List continuity plans |
| GET | `/api/v1/continuity-plans/{id}` | Get plan details |
| PUT | `/api/v1/continuity-plans/{id}` | Update plan |
| POST | `/api/v1/continuity-plans/{id}/approve` | Approve a plan |
| POST | `/api/v1/continuity-plans/{id}/activate` | Activate a plan |
| POST | `/api/v1/continuity-plans/{id}/review` | Conduct plan review |

### Objectives
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recovery-objectives` | Create a recovery objective |
| GET | `/api/v1/recovery-objectives` | List recovery objectives |
| GET | `/api/v1/recovery-objectives/{id}` | Get objective details |
| PUT | `/api/v1/recovery-objectives/{id}` | Update objective |
| GET | `/api/v1/continuity-plans/{id}/gap-analysis` | Get RTO/RPO gap analysis |
| POST | `/api/v1/recovery-objectives/recalculate` | Recalculate gaps |

### Procedures
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/continuity-procedures` | Create a procedure |
| GET | `/api/v1/continuity-procedures` | List procedures |
| GET | `/api/v1/continuity-procedures/{id}` | Get procedure details |
| PUT | `/api/v1/continuity-procedures/{id}` | Update procedure |
| POST | `/api/v1/continuity-procedures/reorder` | Reorder procedure sequence |

### Contacts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/continuity-contacts` | Create a contact |
| GET | `/api/v1/continuity-contacts` | List contacts |
| GET | `/api/v1/continuity-contacts/{id}` | Get contact details |
| PUT | `/api/v1/continuity-contacts/{id}` | Update contact |
| GET | `/api/v1/continuity-plans/{id}/contacts` | Get contacts by plan |

### Tests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/continuity-tests` | Schedule a test |
| GET | `/api/v1/continuity-tests` | List tests |
| GET | `/api/v1/continuity-tests/{id}` | Get test details |
| PUT | `/api/v1/continuity-tests/{id}` | Update test |
| POST | `/api/v1/continuity-tests/{id}/execute` | Execute a test |
| POST | `/api/v1/continuity-tests/{id}/record-results` | Record test results |

### Impact Assessments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/impact-assessments` | Create an impact assessment |
| GET | `/api/v1/impact-assessments` | List impact assessments |
| GET | `/api/v1/impact-assessments/{id}` | Get assessment details |
| PUT | `/api/v1/impact-assessments/{id}` | Update assessment |
| POST | `/api/v1/impact-assessments/{id}/calculate-risk-score` | Calculate risk score |
| GET | `/api/v1/impact-assessments/risk-matrix` | Get risk matrix view |

### Incidents
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/incidents/declare` | Declare a new incident |
| GET | `/api/v1/incidents` | List incidents |
| GET | `/api/v1/incidents/{id}` | Get incident details |
| POST | `/api/v1/incidents/{id}/update` | Update incident status |
| POST | `/api/v1/incidents/{id}/resolve` | Resolve an incident |
| POST | `/api/v1/incidents/{id}/post-mortem` | Complete post-mortem |

---

## 4. Business Rules

1. A continuity plan MUST be reviewed at least annually as indicated by the next_review_date field.
2. Recovery objectives for services with CRITICAL criticality MUST have an RTO of 4 hours or less.
3. The system MUST calculate risk_score as the product of operational_impact numeric value and probability numeric value.
4. A continuity plan in DRAFT status MUST NOT have associated incidents declared against it.
5. P1 severity incidents MUST trigger immediate notification to all primary contacts on the associated plan.
6. A continuity test MUST NOT be marked as PASS if the rto_achieved_hours exceeds the defined RTO for any CRITICAL service.
7. The system MUST prevent deletion of continuity contacts that are marked as is_primary.
8. Impact assessments with a risk_score above the tenant-defined threshold MUST require mitigation_plan to be populated.
9. Incident resolution MUST include a root_cause and resolution_summary before the post_mortem_completed flag can be set.
10. Continuity procedures MUST be executed in sequence_order during failover scenarios.
11. The system SHOULD generate follow_up_actions automatically when a test result is FAIL or PARTIAL.
12. Plans that have not been reviewed within 90 days past their next_review_date MUST be flagged as requiring review.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package continuity.v1;

service BusinessContinuityService {
    // Plans
    rpc CreatePlan(CreatePlanRequest) returns (CreatePlanResponse);
    rpc GetPlan(GetPlanRequest) returns (GetPlanResponse);
    rpc ListPlans(ListPlansRequest) returns (ListPlansResponse);
    rpc ApprovePlan(ApprovePlanRequest) returns (ApprovePlanResponse);
    rpc ActivatePlan(ActivatePlanRequest) returns (ActivatePlanResponse);
    rpc ReviewPlan(ReviewPlanRequest) returns (ReviewPlanResponse);

    // Recovery Objectives
    rpc CreateRecoveryObjective(CreateRecoveryObjectiveRequest) returns (CreateRecoveryObjectiveResponse);
    rpc ListRecoveryObjectives(ListRecoveryObjectivesRequest) returns (ListRecoveryObjectivesResponse);
    rpc GetGapAnalysis(GetGapAnalysisRequest) returns (GetGapAnalysisResponse);

    // Procedures
    rpc CreateProcedure(CreateProcedureRequest) returns (CreateProcedureResponse);
    rpc ListProcedures(ListProceduresRequest) returns (ListProceduresResponse);
    rpc ReorderProcedures(ReorderProceduresRequest) returns (ReorderProceduresResponse);

    // Tests
    rpc CreateTest(CreateTestRequest) returns (CreateTestResponse);
    rpc ExecuteTest(ExecuteTestRequest) returns (ExecuteTestResponse);
    rpc RecordTestResults(RecordTestResultsRequest) returns (RecordTestResultsResponse);

    // Impact Assessments
    rpc CreateImpactAssessment(CreateImpactAssessmentRequest) returns (CreateImpactAssessmentResponse);
    rpc CalculateRiskScore(CalculateRiskScoreRequest) returns (CalculateRiskScoreResponse);
    rpc GetRiskMatrix(GetRiskMatrixRequest) returns (GetRiskMatrixResponse);

    // Incidents
    rpc DeclareIncident(DeclareIncidentRequest) returns (DeclareIncidentResponse);
    rpc UpdateIncident(UpdateIncidentRequest) returns (UpdateIncidentResponse);
    rpc ResolveIncident(ResolveIncidentRequest) returns (ResolveIncidentResponse);
    rpc CompletePostMortem(CompletePostMortemRequest) returns (CompletePostMortemResponse);
}

message ContinuityPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_type = 4;
    string scope = 5;
    string description = 6;
    string owner_id = 7;
    string approver_id = 8;
    string status = 9;
    string last_reviewed = 10;
    string next_review_date = 11;
    int32 version = 12;
}

message CreatePlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    string plan_type = 3;
    string scope = 4;
    string description = 5;
    string owner_id = 6;
    string next_review_date = 7;
    string created_by = 8;
}

message CreatePlanResponse {
    ContinuityPlan plan = 1;
}

message GetPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetPlanResponse {
    ContinuityPlan plan = 1;
}

message ListPlansRequest {
    string tenant_id = 1;
    string plan_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPlansResponse {
    repeated ContinuityPlan plans = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ApprovePlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string approved_by = 3;
}

message ApprovePlanResponse {
    ContinuityPlan plan = 1;
}

message ActivatePlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string activated_by = 3;
}

message ActivatePlanResponse {
    ContinuityPlan plan = 1;
}

message ReviewPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string reviewed_by = 3;
    string review_notes = 4;
}

message ReviewPlanResponse {
    ContinuityPlan plan = 1;
}

message RecoveryObjective {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string service_name = 4;
    int32 rto_hours = 5;
    int32 rpo_hours = 6;
    string criticality = 7;
    int32 current_rto_hours = 8;
    int32 current_rpo_hours = 9;
    string gap_analysis = 10;
    string mitigation_plan = 11;
}

message CreateRecoveryObjectiveRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string service_name = 3;
    int32 rto_hours = 4;
    int32 rpo_hours = 5;
    string criticality = 6;
    int32 current_rto_hours = 7;
    int32 current_rpo_hours = 8;
    string created_by = 9;
}

message CreateRecoveryObjectiveResponse {
    RecoveryObjective objective = 1;
}

message ListRecoveryObjectivesRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string criticality = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListRecoveryObjectivesResponse {
    repeated RecoveryObjective objectives = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetGapAnalysisRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetGapAnalysisResponse {
    repeated GapItem gaps = 1;
    int32 total_services = 2;
    int32 services_with_gaps = 3;
    int32 services_compliant = 4;
}

message GapItem {
    string service_name = 1;
    int32 target_rto_hours = 2;
    int32 current_rto_hours = 3;
    int32 rto_gap_hours = 4;
    int32 target_rpo_hours = 5;
    int32 current_rpo_hours = 6;
    int32 rpo_gap_hours = 7;
    string criticality = 8;
}

message ContinuityProcedure {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string procedure_name = 4;
    string procedure_type = 5;
    string description = 6;
    string step_by_step = 7;
    int32 estimated_duration_minutes = 8;
    string responsible_role_id = 9;
    string backup_role_id = 10;
    int32 sequence_order = 11;
}

message CreateProcedureRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string procedure_name = 3;
    string procedure_type = 4;
    string description = 5;
    string step_by_step = 6;
    int32 estimated_duration_minutes = 7;
    string responsible_role_id = 8;
    string backup_role_id = 9;
    int32 sequence_order = 10;
    string created_by = 11;
}

message CreateProcedureResponse {
    ContinuityProcedure procedure = 1;
}

message ListProceduresRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string procedure_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListProceduresResponse {
    repeated ContinuityProcedure procedures = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReorderProceduresRequest {
    string tenant_id = 1;
    string plan_id = 2;
    repeated ProcedureOrder orders = 3;
    string reordered_by = 4;
}

message ProcedureOrder {
    string procedure_id = 1;
    int32 new_sequence_order = 2;
}

message ReorderProceduresResponse {
    bool success = 1;
}

message ContinuityTest {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string test_name = 4;
    string test_type = 5;
    string scheduled_date = 6;
    string executed_date = 7;
    string executed_by = 8;
    string results_summary = 9;
    double rto_achieved_hours = 10;
    double rpo_achieved_hours = 11;
    string status = 12;
    string pass_fail = 13;
}

message CreateTestRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string test_name = 3;
    string test_type = 4;
    string scheduled_date = 5;
    string created_by = 6;
}

message CreateTestResponse {
    ContinuityTest test = 1;
}

message ExecuteTestRequest {
    string tenant_id = 1;
    string test_id = 2;
    string executed_by = 3;
}

message ExecuteTestResponse {
    ContinuityTest test = 1;
}

message RecordTestResultsRequest {
    string tenant_id = 1;
    string test_id = 2;
    string results_summary = 3;
    double rto_achieved_hours = 4;
    double rpo_achieved_hours = 5;
    string pass_fail = 6;
    string recorded_by = 7;
}

message RecordTestResultsResponse {
    ContinuityTest test = 1;
}

message ImpactAssessment {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string scenario_type = 4;
    string affected_services = 5;
    int64 financial_impact_per_hour_cents = 6;
    string operational_impact = 7;
    string recovery_complexity = 8;
    string probability = 9;
    double risk_score = 10;
    string mitigation_status = 11;
}

message CreateImpactAssessmentRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string scenario_type = 3;
    string affected_services = 4;
    int64 financial_impact_per_hour_cents = 5;
    string operational_impact = 6;
    string recovery_complexity = 7;
    string probability = 8;
    string created_by = 9;
}

message CreateImpactAssessmentResponse {
    ImpactAssessment assessment = 1;
}

message CalculateRiskScoreRequest {
    string tenant_id = 1;
    string assessment_id = 2;
}

message CalculateRiskScoreResponse {
    double risk_score = 1;
    string risk_level = 2;
}

message GetRiskMatrixRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetRiskMatrixResponse {
    repeated RiskMatrixEntry entries = 1;
    double avg_risk_score = 2;
    int32 high_risk_count = 3;
    int32 mitigated_count = 4;
}

message RiskMatrixEntry {
    string scenario_type = 1;
    string operational_impact = 2;
    string probability = 3;
    double risk_score = 4;
    string mitigation_status = 5;
}

message IncidentLog {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string incident_type = 4;
    string severity = 5;
    string description = 6;
    string detected_at = 7;
    string detected_by = 8;
    string response_started_at = 9;
    string resolved_at = 10;
    int32 duration_minutes = 11;
    string services_affected = 12;
    string root_cause = 13;
    string resolution_summary = 14;
    bool post_mortem_completed = 15;
}

message DeclareIncidentRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string incident_type = 3;
    string severity = 4;
    string description = 5;
    string detected_at = 6;
    string detected_by = 7;
    string declared_by = 8;
}

message DeclareIncidentResponse {
    IncidentLog incident = 1;
}

message UpdateIncidentRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string services_affected = 3;
    string update_description = 4;
    string updated_by = 5;
}

message UpdateIncidentResponse {
    IncidentLog incident = 1;
}

message ResolveIncidentRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string resolution_summary = 3;
    string root_cause = 4;
    string resolved_by = 5;
}

message ResolveIncidentResponse {
    IncidentLog incident = 1;
}

message CompletePostMortemRequest {
    string tenant_id = 1;
    string incident_id = 2;
    string lessons_learned = 3;
    string completed_by = 4;
}

message CompletePostMortemResponse {
    IncidentLog incident = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee profiles, roles, contact info | Populate continuity contacts |
| `infrastructure-service` | System health, service registry | Monitor service availability |
| `notification-service` | Alert channels, escalation paths | Send incident notifications |
| `audit-service` | Audit trails, access logs | Investigate incident root causes |
| `compliance-service` | Regulatory requirements | Ensure plan compliance |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `notification-service` | Plan activations, incident alerts | Send alerts and escalations |
| `infrastructure-service` | Failover commands, recovery triggers | Execute recovery procedures |
| `audit-service` | Plan changes, test results, incident logs | Maintain compliance audit trail |
| `reporting-service` | Compliance reports, risk summaries | Executive dashboards |
| `compliance-service` | Test compliance status, plan coverage | Regulatory reporting |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `PlanActivated` | `continuity.plan.activated` | `{ tenant_id, plan_id, plan_name, plan_type, activated_by }` | Published when a continuity plan is activated |
| `TestExecuted` | `continuity.test.executed` | `{ tenant_id, test_id, plan_id, test_name, test_type, pass_fail, rto_achieved_hours, rpo_achieved_hours }` | Published when a continuity test is executed |
| `IncidentDeclared` | `continuity.incident.declared` | `{ tenant_id, incident_id, plan_id, incident_type, severity, description, detected_at }` | Published when a new incident is declared |
| `IncidentResolved` | `continuity.incident.resolved` | `{ tenant_id, incident_id, severity, duration_minutes, root_cause, resolved_at }` | Published when an incident is resolved |
| `ImpactAssessmentCompleted` | `continuity.impact-assessment.completed` | `{ tenant_id, assessment_id, plan_id, scenario_type, risk_score, mitigation_status }` | Published when an impact assessment is completed |
