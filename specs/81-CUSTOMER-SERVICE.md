# 81 - Customer Service Service Specification

## 1. Domain Overview

| Attribute | Value |
|---|---|
| **Bounded Context** | Customer Service |
| **Service Name** | customerservice-service |
| **Database** | customerservice_db |
| **HTTP Port** | 8113 |
| **gRPC Port** | 9113 |

The Customer Service module provides end-to-end case management capabilities including ticket lifecycle management, entitlement verification, SLA tracking, escalation workflows, chatbot session handling, customer self-service portals, and satisfaction survey management. It serves as the central hub for all post-sale customer interaction and support operations.

---

## 2. Database Schema

```sql
CREATE TABLE service_cases (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    case_number TEXT NOT NULL UNIQUE,
    customer_id TEXT NOT NULL,
    contact_id TEXT,
    subject TEXT NOT NULL,
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN',
    priority_code TEXT NOT NULL,
    category_code TEXT NOT NULL,
    sub_category_code TEXT,
    assigned_group_id TEXT,
    assigned_agent_id TEXT,
    channel TEXT NOT NULL DEFAULT 'WEB',
    source_system TEXT,
    related_case_id TEXT,
    product_id TEXT,
    asset_id TEXT,
    contract_id TEXT,
    entitlement_id TEXT,
    sla_id TEXT,
    sla_due_at TEXT,
    sla_breached INTEGER NOT NULL DEFAULT 0,
    first_response_at TEXT,
    resolved_at TEXT,
    closed_at TEXT,
    resolution_notes TEXT,
    root_cause_code TEXT,
    csat_score INTEGER,
    csat_submitted_at TEXT,
    tags TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_service_cases_tenant ON service_cases(tenant_id);
CREATE INDEX idx_service_cases_customer ON service_cases(tenant_id, customer_id);
CREATE INDEX idx_service_cases_status ON service_cases(tenant_id, status);
CREATE INDEX idx_service_cases_assigned_agent ON service_cases(tenant_id, assigned_agent_id);
CREATE INDEX idx_service_cases_sla_due ON service_cases(tenant_id, sla_due_at);
CREATE INDEX idx_service_cases_case_number ON service_cases(case_number);

CREATE TABLE case_priorities (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    priority_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    weight INTEGER NOT NULL DEFAULT 0,
    response_target_minutes INTEGER,
    resolution_target_minutes INTEGER,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_case_priorities_tenant ON case_priorities(tenant_id);

CREATE TABLE case_categories (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    parent_category_id TEXT,
    auto_assign_group_id TEXT,
    default_sla_id TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_case_categories_tenant ON case_categories(tenant_id);
CREATE INDEX idx_case_categories_parent ON case_categories(parent_category_id);

CREATE TABLE case_interactions (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    case_id TEXT NOT NULL,
    interaction_type TEXT NOT NULL,
    direction TEXT NOT NULL DEFAULT 'INBOUND',
    channel TEXT NOT NULL,
    agent_id TEXT,
    subject TEXT,
    body TEXT NOT NULL,
    attachments TEXT,
    is_internal INTEGER NOT NULL DEFAULT 0,
    time_spent_minutes INTEGER,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_case_interactions_case ON case_interactions(tenant_id, case_id);
CREATE INDEX idx_case_interactions_agent ON case_interactions(tenant_id, agent_id);
CREATE INDEX idx_case_interactions_type ON case_interactions(tenant_id, interaction_type);

CREATE TABLE case_assignments (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    case_id TEXT NOT NULL,
    assigned_to_type TEXT NOT NULL,
    assigned_to_id TEXT NOT NULL,
    assigned_by_type TEXT NOT NULL,
    assigned_by_id TEXT NOT NULL,
    assignment_reason TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_case_assignments_case ON case_assignments(tenant_id, case_id);

CREATE TABLE service_level_agreements (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    sla_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    priority_id TEXT NOT NULL,
    first_response_minutes INTEGER NOT NULL,
    next_response_minutes INTEGER,
    resolution_minutes INTEGER NOT NULL,
    business_hours_only INTEGER NOT NULL DEFAULT 1,
    escalation_rule TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_sla_tenant ON service_level_agreements(tenant_id);

CREATE TABLE entitlement_checks (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    case_id TEXT,
    customer_id TEXT NOT NULL,
    product_id TEXT,
    asset_id TEXT,
    contract_id TEXT,
    entitlement_status TEXT NOT NULL,
    coverage_type TEXT,
    coverage_start_date TEXT,
    coverage_end_date TEXT,
    remaining_incidents INTEGER,
    response_code TEXT,
    response_message TEXT,
    checked_at TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_entitlement_checks_case ON entitlement_checks(tenant_id, case_id);
CREATE INDEX idx_entitlement_checks_customer ON entitlement_checks(tenant_id, customer_id);

CREATE TABLE chatbot_sessions (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL UNIQUE,
    customer_id TEXT,
    contact_id TEXT,
    case_id TEXT,
    channel TEXT NOT NULL DEFAULT 'CHAT',
    status TEXT NOT NULL DEFAULT 'ACTIVE',
    language TEXT NOT NULL DEFAULT 'en',
    conversation_history TEXT NOT NULL,
    intent_detected TEXT,
    confidence_score INTEGER,
    escalated_to_agent INTEGER NOT NULL DEFAULT 0,
    agent_id TEXT,
    started_at TEXT NOT NULL,
    ended_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_chatbot_sessions_tenant ON chatbot_sessions(tenant_id);
CREATE INDEX idx_chatbot_sessions_session ON chatbot_sessions(session_id);
CREATE INDEX idx_chatbot_sessions_customer ON chatbot_sessions(tenant_id, customer_id);

CREATE TABLE customer_portals (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    portal_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    url TEXT NOT NULL,
    theme_config TEXT,
    allowed_channels TEXT,
    self_service_enabled INTEGER NOT NULL DEFAULT 1,
    knowledge_base_enabled INTEGER NOT NULL DEFAULT 1,
    chat_enabled INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_customer_portals_tenant ON customer_portals(tenant_id);

CREATE TABLE satisfaction_surveys (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    case_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    contact_id TEXT,
    survey_type TEXT NOT NULL DEFAULT 'CSAT',
    overall_score INTEGER NOT NULL,
    responsiveness_score INTEGER,
    professionalism_score INTEGER,
    resolution_score INTEGER,
    comments TEXT,
    submitted_at TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_satisfaction_surveys_case ON satisfaction_surveys(tenant_id, case_id);
CREATE INDEX idx_satisfaction_surveys_customer ON satisfaction_surveys(tenant_id, customer_id);

CREATE TABLE case_escalations (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    case_id TEXT NOT NULL,
    escalation_level INTEGER NOT NULL DEFAULT 1,
    from_agent_id TEXT,
    to_agent_id TEXT,
    to_group_id TEXT,
    reason TEXT NOT NULL,
    sla_status_at_escalation TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING',
    escalated_at TEXT NOT NULL,
    acknowledged_at TEXT,
    resolved_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_case_escalations_case ON case_escalations(tenant_id, case_id);
CREATE INDEX idx_case_escalations_status ON case_escalations(tenant_id, status);

CREATE TABLE resolution_templates (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    category_code TEXT,
    subject_template TEXT,
    body_template TEXT NOT NULL,
    resolution_code TEXT,
    root_cause_code TEXT,
    suggested_actions TEXT,
    usage_count INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_resolution_templates_tenant ON resolution_templates(tenant_id);
CREATE INDEX idx_resolution_templates_category ON resolution_templates(tenant_id, category_code);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/cases` | Create a new service case |
| GET | `/api/v1/cases/{caseId}` | Get case details by ID |
| GET | `/api/v1/cases` | List/search cases with filters |
| PUT | `/api/v1/cases/{caseId}` | Update case details |
| PATCH | `/api/v1/cases/{caseId}/status` | Update case status |
| POST | `/api/v1/cases/{caseId}/assign` | Assign case to agent or group |
| POST | `/api/v1/cases/{caseId}/escalate` | Escalate a case |
| POST | `/api/v1/cases/{caseId}/resolve` | Resolve a case |
| POST | `/api/v1/cases/{caseId}/close` | Close a case |
| POST | `/api/v1/cases/{caseId}/interactions` | Add interaction to case |
| GET | `/api/v1/cases/{caseId}/interactions` | List interactions for case |
| POST | `/api/v1/cases/{caseId}/entitlement-check` | Check customer entitlement |
| GET | `/api/v1/cases/{caseId}/escalations` | Get escalation history |
| GET | `/api/v1/cases/{caseId}/assignments` | Get assignment history |
| POST | `/api/v1/cases/{caseId}/surveys` | Submit satisfaction survey |
| GET | `/api/v1/cases/{caseId}/surveys` | Get surveys for case |
| GET | `/api/v1/sla` | List SLA policies |
| POST | `/api/v1/sla` | Create SLA policy |
| GET | `/api/v1/sla/{slaId}` | Get SLA policy details |
| PUT | `/api/v1/sla/{slaId}` | Update SLA policy |
| POST | `/api/v1/chatbot/sessions` | Start a chatbot session |
| PUT | `/api/v1/chatbot/sessions/{sessionId}` | Update chatbot session |
| POST | `/api/v1/chatbot/sessions/{sessionId}/message` | Send message in chatbot session |
| POST | `/api/v1/chatbot/sessions/{sessionId}/escalate` | Escalate chatbot to live agent |
| GET | `/api/v1/portals` | List customer portals |
| POST | `/api/v1/portals` | Create customer portal |
| PUT | `/api/v1/portals/{portalId}` | Update customer portal |
| GET | `/api/v1/knowledge/search` | Search knowledge base articles |
| GET | `/api/v1/resolution-templates` | List resolution templates |
| POST | `/api/v1/resolution-templates` | Create resolution template |
| GET | `/api/v1/resolution-templates/{templateId}` | Get resolution template |
| GET | `/api/v1/csat/dashboard` | Get CSAT dashboard metrics |
| GET | `/api/v1/csat/trends` | Get CSAT trend analysis |
| GET | `/api/v1/metrics/agent-performance` | Get agent performance metrics |
| GET | `/api/v1/metrics/case-volume` | Get case volume metrics |
| POST | `/api/v1/categories` | Create case category |
| GET | `/api/v1/categories` | List case categories |
| PUT | `/api/v1/categories/{categoryId}` | Update case category |
| POST | `/api/v1/priorities` | Create case priority |
| GET | `/api/v1/priorities` | List case priorities |
| PUT | `/api/v1/priorities/{priorityId}` | Update case priority |

---

## 4. Business Rules

1. A service case MUST have a unique case number generated using the pattern `SR-{YYYY}{MM}-{SEQ}` at time of creation.
2. Every case MUST be associated with a valid customer_id and a priority_code from the case_priorities table.
3. Entitlement verification MUST be performed before case creation when a contract_id or product_id is provided; unentitled cases SHOULD be flagged and routed to a specialized queue.
4. SLA tracking MUST begin at the moment of case creation; the first_response_at timestamp MUST be recorded when the first agent interaction is logged.
5. An SLA breach MUST be detected when the current time exceeds sla_due_at and the case is not in RESOLVED or CLOSED status; the system MUST emit an SLABreached event immediately upon detection.
6. Case escalation MUST increment escalation_level; each escalation level MUST route the case to a higher-tier support group as defined in the escalation rule.
7. A case in RESOLVED status MUST wait a configurable cooling-off period (default 72 hours) before it can be transitioned to CLOSED status automatically.
8. A customer MAY reopen a CLOSED case within 30 days, which MUST create a new interaction and reset status to OPEN with the original priority.
9. Satisfaction surveys MUST be sent automatically upon case resolution; a survey is considered valid only if submitted within 14 days of case closure.
10. CSAT score MUST be an integer between 1 and 5; the overall CSAT for an agent or period MUST be computed as the arithmetic mean of all valid survey scores.
11. Chatbot sessions SHOULD attempt intent resolution using the knowledge base before offering escalation to a live agent; escalation MUST transfer full conversation history.
12. Case assignments MUST be recorded in the case_assignments table with both the assignor and assignee; a case SHOULD NOT be assigned to the same agent who escalated it.
13. Resolution templates SHOULD be suggested to agents based on case category and root cause code matching.
14. All monetary values related to service costs (if applicable) MUST be stored as integers in cents.
15. All timestamps MUST be stored and transmitted in ISO 8601 format with UTC timezone.
16. Soft delete MUST be used (is_active = 0) for all entities; hard deletes MUST NOT be permitted.
17. Tenant isolation MUST be enforced on every query; no cross-tenant data access is permitted.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package customerservice.v1;

option go_package = "github.com/fusion/protos/customerservice/v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service CustomerService {
    // Case Lifecycle
    rpc CreateCase(CreateCaseRequest) returns (CaseResponse);
    rpc GetCase(GetCaseRequest) returns (CaseResponse);
    rpc ListCases(ListCasesRequest) returns (ListCasesResponse);
    rpc UpdateCase(UpdateCaseRequest) returns (CaseResponse);
    rpc UpdateCaseStatus(UpdateCaseStatusRequest) returns (CaseResponse);
    rpc AssignCase(AssignCaseRequest) returns (CaseResponse);
    rpc EscalateCase(EscalateCaseRequest) returns (CaseResponse);
    rpc ResolveCase(ResolveCaseRequest) returns (CaseResponse);
    rpc CloseCase(CloseCaseRequest) returns (CaseResponse);

    // Interactions
    rpc AddInteraction(AddInteractionRequest) returns (InteractionResponse);
    rpc ListInteractions(ListInteractionsRequest) returns (ListInteractionsResponse);

    // Entitlement
    rpc CheckEntitlement(CheckEntitlementRequest) returns (EntitlementResponse);

    // SLA
    rpc CreateSLA(CreateSLARequest) returns (SLAResponse);
    rpc GetSLA(GetSLARequest) returns (SLAResponse);
    rpc ListSLAs(ListSLAsRequest) returns (ListSLAsResponse);
    rpc UpdateSLA(UpdateSLARequest) returns (SLAResponse);

    // Chatbot
    rpc StartChatbotSession(StartChatbotSessionRequest) returns (ChatbotSessionResponse);
    rpc SendChatbotMessage(SendChatbotMessageRequest) returns (ChatbotMessageResponse);
    rpc EscalateChatbot(EscalateChatbotRequest) returns (ChatbotSessionResponse);

    // Surveys
    rpc SubmitSurvey(SubmitSurveyRequest) returns (SurveyResponse);
    rpc GetCSATDashboard(GetCSATDashboardRequest) returns (CSATDashboardResponse);

    // Knowledge Search
    rpc SearchKnowledge(SearchKnowledgeRequest) returns (SearchKnowledgeResponse);

    // Resolution Templates
    rpc ListResolutionTemplates(ListResolutionTemplatesRequest) returns (ListResolutionTemplatesResponse);
    rpc GetResolutionTemplate(GetResolutionTemplateRequest) returns (ResolutionTemplateResponse);
}

message CaseMessage {
    string id = 1;
    string tenant_id = 2;
    string case_number = 3;
    string customer_id = 4;
    string contact_id = 5;
    string subject = 6;
    string description = 7;
    string status = 8;
    string priority_code = 9;
    string category_code = 10;
    string sub_category_code = 11;
    string assigned_group_id = 12;
    string assigned_agent_id = 13;
    string channel = 14;
    string source_system = 15;
    string related_case_id = 16;
    string product_id = 17;
    string asset_id = 18;
    string contract_id = 19;
    string entitlement_id = 20;
    string sla_id = 21;
    google.protobuf.Timestamp sla_due_at = 22;
    bool sla_breached = 23;
    google.protobuf.Timestamp first_response_at = 24;
    google.protobuf.Timestamp resolved_at = 25;
    google.protobuf.Timestamp closed_at = 26;
    string resolution_notes = 27;
    string root_cause_code = 28;
    int32 csat_score = 29;
    string tags = 30;
    bool is_active = 31;
    google.protobuf.Timestamp created_at = 32;
    google.protobuf.Timestamp updated_at = 33;
    string created_by = 34;
    string updated_by = 35;
    int32 version = 36;
}

message CreateCaseRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string contact_id = 3;
    string subject = 4;
    string description = 5;
    string priority_code = 6;
    string category_code = 7;
    string sub_category_code = 8;
    string channel = 9;
    string product_id = 10;
    string asset_id = 11;
    string contract_id = 12;
    string tags = 13;
}

message GetCaseRequest {
    string tenant_id = 1;
    string case_id = 2;
}

message ListCasesRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string status = 3;
    string priority_code = 4;
    string assigned_agent_id = 5;
    string category_code = 6;
    int32 page_size = 7;
    string page_token = 8;
}

message ListCasesResponse {
    repeated CaseMessage cases = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateCaseRequest {
    string tenant_id = 1;
    string case_id = 2;
    string subject = 3;
    string description = 4;
    string priority_code = 5;
    string category_code = 6;
    string sub_category_code = 7;
    string tags = 8;
    int32 version = 9;
}

message UpdateCaseStatusRequest {
    string tenant_id = 1;
    string case_id = 2;
    string status = 3;
    string reason = 4;
    int32 version = 5;
}

message AssignCaseRequest {
    string tenant_id = 1;
    string case_id = 2;
    string assigned_to_type = 3;
    string assigned_to_id = 4;
    string assignment_reason = 5;
}

message EscalateCaseRequest {
    string tenant_id = 1;
    string case_id = 2;
    string to_agent_id = 3;
    string to_group_id = 4;
    string reason = 5;
}

message ResolveCaseRequest {
    string tenant_id = 1;
    string case_id = 2;
    string resolution_notes = 3;
    string root_cause_code = 4;
    string resolution_template_id = 5;
}

message CloseCaseRequest {
    string tenant_id = 1;
    string case_id = 2;
    string closing_comments = 3;
}

message CaseResponse {
    CaseMessage case = 1;
}

message InteractionMessage {
    string id = 1;
    string tenant_id = 2;
    string case_id = 3;
    string interaction_type = 4;
    string direction = 5;
    string channel = 6;
    string agent_id = 7;
    string subject = 8;
    string body = 9;
    string attachments = 10;
    bool is_internal = 11;
    int32 time_spent_minutes = 12;
    google.protobuf.Timestamp created_at = 13;
}

message AddInteractionRequest {
    string tenant_id = 1;
    string case_id = 2;
    string interaction_type = 3;
    string direction = 4;
    string channel = 5;
    string subject = 6;
    string body = 7;
    string attachments = 8;
    bool is_internal = 9;
    int32 time_spent_minutes = 10;
}

message InteractionResponse {
    InteractionMessage interaction = 1;
}

message ListInteractionsRequest {
    string tenant_id = 1;
    string case_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListInteractionsResponse {
    repeated InteractionMessage interactions = 1;
    string next_page_token = 2;
}

message CheckEntitlementRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string product_id = 3;
    string asset_id = 4;
    string contract_id = 5;
}

message EntitlementResponse {
    bool is_entitled = 1;
    string entitlement_status = 2;
    string coverage_type = 3;
    string coverage_start_date = 4;
    string coverage_end_date = 5;
    int32 remaining_incidents = 6;
    string response_code = 7;
    string response_message = 8;
}

message SLAMessage {
    string id = 1;
    string tenant_id = 2;
    string sla_code = 3;
    string name = 4;
    string description = 5;
    string priority_id = 6;
    int32 first_response_minutes = 7;
    int32 next_response_minutes = 8;
    int32 resolution_minutes = 9;
    bool business_hours_only = 10;
    string escalation_rule = 11;
    google.protobuf.Timestamp created_at = 12;
    google.protobuf.Timestamp updated_at = 13;
    int32 version = 14;
}

message CreateSLARequest {
    string tenant_id = 1;
    string sla_code = 2;
    string name = 3;
    string description = 4;
    string priority_id = 5;
    int32 first_response_minutes = 6;
    int32 next_response_minutes = 7;
    int32 resolution_minutes = 8;
    bool business_hours_only = 9;
    string escalation_rule = 10;
}

message GetSLARequest {
    string tenant_id = 1;
    string sla_id = 2;
}

message ListSLAsRequest {
    string tenant_id = 1;
    int32 page_size = 2;
    string page_token = 3;
}

message ListSLAsResponse {
    repeated SLAMessage slas = 1;
    string next_page_token = 2;
}

message UpdateSLARequest {
    string tenant_id = 1;
    string sla_id = 2;
    string name = 3;
    string description = 4;
    int32 first_response_minutes = 5;
    int32 next_response_minutes = 6;
    int32 resolution_minutes = 7;
    string escalation_rule = 8;
    int32 version = 9;
}

message SLAResponse {
    SLAMessage sla = 1;
}

message StartChatbotSessionRequest {
    string tenant_id = 1;
    string customer_id = 2;
    string contact_id = 3;
    string channel = 4;
    string language = 5;
}

message ChatbotSessionResponse {
    string session_id = 1;
    string status = 2;
    string agent_id = 3;
}

message SendChatbotMessageRequest {
    string tenant_id = 1;
    string session_id = 2;
    string message = 3;
}

message ChatbotMessageResponse {
    string reply = 1;
    string intent_detected = 2;
    int32 confidence_score = 3;
    repeated string suggested_articles = 4;
    bool escalation_suggested = 5;
}

message EscalateChatbotRequest {
    string tenant_id = 1;
    string session_id = 2;
    string reason = 3;
}

message SubmitSurveyRequest {
    string tenant_id = 1;
    string case_id = 2;
    string customer_id = 3;
    string survey_type = 4;
    int32 overall_score = 5;
    int32 responsiveness_score = 6;
    int32 professionalism_score = 7;
    int32 resolution_score = 8;
    string comments = 9;
}

message SurveyResponse {
    string id = 1;
    string case_id = 2;
    int32 overall_score = 3;
    google.protobuf.Timestamp submitted_at = 4;
}

message GetCSATDashboardRequest {
    string tenant_id = 1;
    string date_from = 2;
    string date_to = 3;
}

message CSATDashboardResponse {
    double average_csat = 1;
    int32 total_surveys = 2;
    double response_rate = 3;
    repeated AgentCSAT agent_scores = 4;
    repeated MonthlyCSAT monthly_trends = 5;
}

message AgentCSAT {
    string agent_id = 1;
    string agent_name = 2;
    double average_csat = 3;
    int32 case_count = 4;
}

message MonthlyCSAT {
    string month = 1;
    double average_csat = 2;
    int32 survey_count = 3;
}

message SearchKnowledgeRequest {
    string tenant_id = 1;
    string query = 2;
    string category_code = 3;
    int32 max_results = 4;
}

message SearchKnowledgeResponse {
    repeated KnowledgeArticleResult results = 1;
}

message KnowledgeArticleResult {
    string article_id = 1;
    string title = 2;
    string summary = 3;
    int32 relevance_score = 4;
}

message ListResolutionTemplatesRequest {
    string tenant_id = 1;
    string category_code = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListResolutionTemplatesResponse {
    repeated ResolutionTemplateMessage templates = 1;
    string next_page_token = 2;
}

message GetResolutionTemplateRequest {
    string tenant_id = 1;
    string template_id = 2;
}

message ResolutionTemplateMessage {
    string id = 1;
    string tenant_id = 2;
    string template_code = 3;
    string name = 4;
    string body_template = 5;
    string resolution_code = 6;
    string root_cause_code = 7;
    string suggested_actions = 8;
    int32 usage_count = 9;
}

message ResolutionTemplateResponse {
    ResolutionTemplateMessage template = 1;
}
```

---

## 6. Inter-Service Integration

### Inbound Dependencies

| Source Service | Integration Type | Purpose |
|---|---|---|
| Knowledge Management | REST/gRPC | Search and retrieve knowledge base articles for agent assistance and chatbot resolution |
| Sales Automation | REST | Retrieve customer and contact details, opportunity context for case linkage |
| Field Service | REST/gRPC | Create and track field service work orders from service cases |
| Identity Service | REST | Agent authentication, authorization, and user profile retrieval |
| Contract Management | REST | Retrieve contract and entitlement details for coverage verification |
| Product Registry | REST | Retrieve product and asset information for case context |
| Notification Service | Async | Send case notifications to customers and agents |

### Outbound Events Consumed By

| Target Service | Event | Purpose |
|---|---|---|
| Field Service | CaseEscalated | Trigger field service dispatch for on-site resolution |
| Sales Automation | CaseResolved | Update customer health score and account intelligence |
| Notification Service | CaseCreated, CaseAssigned, SLABreached | Deliver real-time notifications |
| Knowledge Management | CaseResolved | Suggest new knowledge article creation for novel resolutions |
| Digital Assistant | CaseCreated | Enable proactive customer communication |

---

## 7. Events

| Event | Payload | Description |
|---|---|---|
| `CaseCreated` | `{ case_id, case_number, tenant_id, customer_id, priority_code, category_code, channel, sla_due_at }` | Emitted when a new service case is created |
| `CaseAssigned` | `{ case_id, tenant_id, assigned_to_type, assigned_to_id, assigned_by_id, assignment_reason }` | Emitted when a case is assigned to an agent or group |
| `CaseEscalated` | `{ case_id, tenant_id, escalation_level, from_agent_id, to_agent_id, to_group_id, reason }` | Emitted when a case is escalated to higher-tier support |
| `CaseResolved` | `{ case_id, tenant_id, resolved_by, root_cause_code, resolution_notes, time_to_resolve_minutes }` | Emitted when a case is marked as resolved |
| `CaseClosed` | `{ case_id, tenant_id, closed_by, csat_score, total_interactions, total_time_spent_minutes }` | Emitted when a case is fully closed after cooling-off |
| `SLABreached` | `{ case_id, tenant_id, sla_id, sla_due_at, breach_type, current_status, assigned_agent_id }` | Emitted when an SLA deadline is exceeded without resolution |
| `InteractionAdded` | `{ case_id, tenant_id, interaction_id, interaction_type, agent_id, is_internal }` | Emitted when a new interaction is added to a case |
| `EntitlementChecked` | `{ tenant_id, customer_id, case_id, entitlement_status, coverage_type, remaining_incidents }` | Emitted when an entitlement check is performed |
| `SurveySubmitted` | `{ case_id, tenant_id, customer_id, overall_score, survey_type }` | Emitted when a customer submits a satisfaction survey |
| `ChatbotEscalated` | `{ session_id, tenant_id, case_id, customer_id, agent_id, conversation_summary }` | Emitted when a chatbot session is escalated to a live agent |
| `CaseReopened` | `{ case_id, tenant_id, reopened_by, original_priority_code, new_status }` | Emitted when a closed case is reopened by a customer |
