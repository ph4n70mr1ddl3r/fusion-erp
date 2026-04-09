# 75 - HR Help Desk Service Specification

## 1. Domain Overview
**Bounded Context:** HR Help Desk - Provides a centralized HR service desk for case management, knowledge base access, employee self-service, chatbot support, SLA tracking, and escalation management for HR-related inquiries and requests.

**Service Name:** hrhelpdesk-service
**Database:** hrhelpdesk_db
**HTTP Port:** 8107 | **gRPC Port:** 9107

## 2. Database Schema

```sql
CREATE TABLE service_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    category_name TEXT NOT NULL,
    category_code TEXT NOT NULL,
    parent_category_id TEXT,
    description TEXT,
    icon_url TEXT,
    display_order INTEGER DEFAULT 0,
    default_assignee_group TEXT,
    default_sla_id TEXT,
    required_fields TEXT,
    is_self_service_visible INTEGER NOT NULL DEFAULT 1,
    estimated_resolution_hours INTEGER
);

CREATE INDEX idx_svc_categories_tenant ON service_categories(tenant_id);
CREATE INDEX idx_svc_categories_parent ON service_categories(parent_category_id);
CREATE INDEX idx_svc_categories_code ON service_categories(category_code);

CREATE TABLE knowledge_articles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    title TEXT NOT NULL,
    article_number TEXT NOT NULL,
    content TEXT NOT NULL,
    summary TEXT,
    category_id TEXT,
    tags TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    visibility TEXT NOT NULL DEFAULT 'INTERNAL',
    view_count INTEGER DEFAULT 0,
    helpful_count INTEGER DEFAULT 0,
    not_helpful_count INTEGER DEFAULT 0,
    published_at TEXT,
    expires_at TEXT,
    reviewed_at TEXT,
    reviewed_by TEXT,
    related_articles TEXT,
    attachments TEXT,
    search_vector TEXT
);

CREATE INDEX idx_knowledge_tenant ON knowledge_articles(tenant_id);
CREATE INDEX idx_knowledge_status ON knowledge_articles(status);
CREATE INDEX idx_knowledge_category ON knowledge_articles(category_id);
CREATE INDEX idx_knowledge_number ON knowledge_articles(article_number);

CREATE TABLE hr_cases (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    case_number TEXT NOT NULL,
    subject TEXT NOT NULL,
    description TEXT NOT NULL,
    category_id TEXT NOT NULL,
    subcategory_id TEXT,
    priority TEXT NOT NULL DEFAULT 'MEDIUM',
    status TEXT NOT NULL DEFAULT 'OPEN',
    case_type TEXT NOT NULL DEFAULT 'INCIDENT',
    source TEXT NOT NULL DEFAULT 'PORTAL',
    requester_id TEXT NOT NULL,
    requester_name TEXT,
    requester_email TEXT,
    assigned_to TEXT,
    assigned_group TEXT,
    sla_id TEXT,
    due_date TEXT,
    resolved_at TEXT,
    closed_at TEXT,
    resolution_summary TEXT,
    parent_case_id TEXT,
    related_case_ids TEXT,
    custom_fields TEXT,
    tags TEXT,
    satisfaction_rating INTEGER,
    satisfaction_comment TEXT,
    first_response_at TEXT,
    resolution_deadline TEXT
);

CREATE INDEX idx_hr_cases_tenant ON hr_cases(tenant_id);
CREATE INDEX idx_hr_cases_number ON hr_cases(case_number);
CREATE INDEX idx_hr_cases_requester ON hr_cases(requester_id);
CREATE INDEX idx_hr_cases_assignee ON hr_cases(assigned_to);
CREATE INDEX idx_hr_cases_status ON hr_cases(status);
CREATE INDEX idx_hr_cases_priority ON hr_cases(priority);
CREATE INDEX idx_hr_cases_due ON hr_cases(due_date);

CREATE TABLE case_interactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    case_id TEXT NOT NULL,
    interaction_type TEXT NOT NULL,
    content TEXT NOT NULL,
    author_id TEXT NOT NULL,
    author_name TEXT,
    is_internal INTEGER NOT NULL DEFAULT 0,
    attachments TEXT,
    channel TEXT DEFAULT 'PORTAL',
    metadata TEXT,
    FOREIGN KEY (case_id) REFERENCES hr_cases(id)
);

CREATE INDEX idx_case_interactions_tenant ON case_interactions(tenant_id);
CREATE INDEX idx_case_interactions_case ON case_interactions(case_id);
CREATE INDEX idx_case_interactions_type ON case_interactions(interaction_type);

CREATE TABLE case_resolutions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    case_id TEXT NOT NULL,
    resolution_type TEXT NOT NULL,
    resolution_notes TEXT NOT NULL,
    knowledge_article_id TEXT,
    time_spent_minutes INTEGER DEFAULT 0,
    resolved_by TEXT NOT NULL,
    resolved_at TEXT NOT NULL,
    closure_category TEXT,
    root_cause TEXT,
    prevention_measures TEXT,
    FOREIGN KEY (case_id) REFERENCES hr_cases(id)
);

CREATE INDEX idx_case_resolutions_tenant ON case_resolutions(tenant_id);
CREATE INDEX idx_case_resolutions_case ON case_resolutions(case_id);

CREATE TABLE sla_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    sla_name TEXT NOT NULL,
    sla_code TEXT NOT NULL,
    category_id TEXT,
    priority TEXT,
    first_response_hours INTEGER NOT NULL,
    resolution_hours INTEGER NOT NULL,
    update_frequency_hours INTEGER,
    business_hours_only INTEGER NOT NULL DEFAULT 1,
    escalation_rule_id TEXT,
    notification_template_id TEXT,
    description TEXT
);

CREATE INDEX idx_sla_definitions_tenant ON sla_definitions(tenant_id);
CREATE INDEX idx_sla_definitions_code ON sla_definitions(sla_code);
CREATE INDEX idx_sla_definitions_category ON sla_definitions(category_id);

CREATE TABLE employee_self_service_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    request_type TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    request_data TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'SUBMITTED',
    auto_approved INTEGER NOT NULL DEFAULT 0,
    requires_approval INTEGER NOT NULL DEFAULT 1,
    approver_id TEXT,
    approved_at TEXT,
    fulfillment_date TEXT,
    case_id TEXT,
    notes TEXT
);

CREATE INDEX idx_ess_requests_tenant ON employee_self_service_requests(tenant_id);
CREATE INDEX idx_ess_requests_employee ON employee_self_service_requests(employee_id);
CREATE INDEX idx_ess_requests_status ON employee_self_service_requests(status);

CREATE TABLE chatbot_conversations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    session_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    channel TEXT NOT NULL DEFAULT 'WEB',
    messages TEXT NOT NULL,
    intent_detected TEXT,
    entities_extracted TEXT,
    case_created_id TEXT,
    articles_suggested TEXT,
    satisfaction_score INTEGER,
    ended_at TEXT,
    escalation_triggered INTEGER NOT NULL DEFAULT 0,
    language TEXT DEFAULT 'en',
    total_turns INTEGER DEFAULT 0
);

CREATE INDEX idx_chatbot_tenant ON chatbot_conversations(tenant_id);
CREATE INDEX idx_chatbot_session ON chatbot_conversations(session_id);
CREATE INDEX idx_chatbot_employee ON chatbot_conversations(employee_id);

CREATE TABLE case_satisfaction_surveys (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    case_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    overall_rating INTEGER NOT NULL,
    responsiveness_rating INTEGER,
    knowledge_rating INTEGER,
    resolution_rating INTEGER,
    comment TEXT,
    submitted_at TEXT NOT NULL,
    would_recommend INTEGER,
    nps_score INTEGER,
    FOREIGN KEY (case_id) REFERENCES hr_cases(id)
);

CREATE INDEX idx_surveys_tenant ON case_satisfaction_surveys(tenant_id);
CREATE INDEX idx_surveys_case ON case_satisfaction_surveys(case_id);
CREATE INDEX idx_surveys_employee ON case_satisfaction_surveys(employee_id);

CREATE TABLE escalation_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL,
    condition_expression TEXT NOT NULL,
    escalation_level INTEGER NOT NULL DEFAULT 1,
    escalation_target_type TEXT NOT NULL,
    escalation_target_id TEXT NOT NULL,
    notification_template TEXT,
    cooldown_minutes INTEGER DEFAULT 60,
    max_escalations INTEGER DEFAULT 3,
    category_id TEXT,
    priority_threshold TEXT,
    description TEXT
);

CREATE INDEX idx_escalation_rules_tenant ON escalation_rules(tenant_id);
CREATE INDEX idx_escalation_rules_type ON escalation_rules(rule_type);
```

## 3. REST API Endpoints

### Service Categories
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/service-categories` | Create a service category |
| GET | `/api/v1/service-categories` | List categories (tree structure) |
| GET | `/api/v1/service-categories/{id}` | Get category details |
| PUT | `/api/v1/service-categories/{id}` | Update category |
| DELETE | `/api/v1/service-categories/{id}` | Deactivate category |

### Knowledge Articles
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/knowledge-articles` | Create a knowledge article |
| GET | `/api/v1/knowledge-articles` | List articles with filters |
| GET | `/api/v1/knowledge-articles/{id}` | Get article by ID |
| PUT | `/api/v1/knowledge-articles/{id}` | Update article |
| POST | `/api/v1/knowledge-articles/{id}/submit-review` | Submit article for review |
| POST | `/api/v1/knowledge-articles/{id}/publish` | Publish approved article |
| POST | `/api/v1/knowledge-articles/{id}/rate` | Rate article helpfulness |
| GET | `/api/v1/knowledge-articles/search` | Full-text search articles |

### HR Cases
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/hr-cases` | Create a new HR case |
| GET | `/api/v1/hr-cases` | List cases with filters and sorting |
| GET | `/api/v1/hr-cases/{id}` | Get case full details |
| PUT | `/api/v1/hr-cases/{id}` | Update case |
| PATCH | `/api/v1/hr-cases/{id}/status` | Update case status |
| POST | `/api/v1/hr-cases/{id}/assign` | Assign case to agent/group |
| POST | `/api/v1/hr-cases/{id}/resolve` | Resolve case |
| POST | `/api/v1/hr-cases/{id}/close` | Close resolved case |
| POST | `/api/v1/hr-cases/{id}/reopen` | Reopen closed case |
| GET | `/api/v1/employees/{employeeId}/cases` | Get cases for employee |

### Case Interactions
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/hr-cases/{caseId}/interactions` | Add interaction to case |
| GET | `/api/v1/hr-cases/{caseId}/interactions` | List case interactions |
| PUT | `/api/v1/case-interactions/{id}` | Update interaction |

### SLA Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/sla-definitions` | Create SLA definition |
| GET | `/api/v1/sla-definitions` | List SLA definitions |
| GET | `/api/v1/sla-definitions/{id}` | Get SLA details |
| PUT | `/api/v1/sla-definitions/{id}` | Update SLA definition |
| GET | `/api/v1/hr-cases/{caseId}/sla-status` | Get SLA status for case |
| GET | `/api/v1/sla-dashboard` | Get SLA compliance dashboard |

### Employee Self-Service
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/self-service-requests` | Submit self-service request |
| GET | `/api/v1/self-service-requests` | List self-service requests |
| GET | `/api/v1/self-service-requests/{id}` | Get request details |
| POST | `/api/v1/self-service-requests/{id}/approve` | Approve self-service request |
| POST | `/api/v1/self-service-requests/{id}/reject` | Reject self-service request |

### Chatbot
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/chatbot/start` | Start a chatbot conversation |
| POST | `/api/v1/chatbot/message` | Send message to chatbot |
| GET | `/api/v1/chatbot/conversations/{sessionId}` | Get conversation history |
| POST | `/api/v1/chatbot/escalate` | Escalate chatbot to live agent |

### Surveys & Escalations
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/case-surveys` | Submit satisfaction survey |
| GET | `/api/v1/case-surveys` | List survey responses |
| GET | `/api/v1/case-surveys/analytics` | Get survey analytics |
| POST | `/api/v1/escalation-rules` | Create escalation rule |
| GET | `/api/v1/escalation-rules` | List escalation rules |
| POST | `/api/v1/hr-cases/{caseId}/escalate` | Manually escalate case |
| GET | `/api/v1/helpdesk-analytics/dashboard` | Get helpdesk analytics |

## 4. Business Rules

1. A case number MUST be auto-generated with format HR-YYYYMMDD-NNNNN on creation.
2. Every case MUST be assigned to at least one agent or group within the first_response_hours of the associated SLA.
3. The system MUST track first_response_at timestamp from case creation.
4. An escalation rule MUST NOT trigger more than max_escalations times for a single case.
5. Knowledge articles MUST go through a review workflow before publication.
6. The system MUST calculate SLA breach risk based on remaining time and current priority.
7. A case MUST NOT be closed without a resolution record.
8. Self-service requests flagged as auto_approved SHOULD be fulfilled immediately without agent intervention.
9. The chatbot MUST attempt to resolve inquiries using knowledge articles before creating a case.
10. Satisfaction surveys SHOULD be sent automatically within 24 hours of case closure.
11. The system MUST enforce the escalation cooldown period to prevent notification flooding.
12. Case interactions marked as internal MUST NOT be visible to the requesting employee.
13. The system SHOULD suggest relevant knowledge articles based on case category and description.
14. SLA compliance percentage MUST be calculated and available in real-time on the dashboard.
15. A reopened case MUST retain all prior interactions and reset the SLA clock.

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package hrhelpdesk;

service HRHelpDeskService {
    // Service Categories
    rpc CreateServiceCategory(CreateServiceCategoryRequest) returns (ServiceCategoryResponse);
    rpc ListServiceCategories(ListCategoriesRequest) returns (ServiceCategoryListResponse);
    rpc GetServiceCategory(GetByIdRequest) returns (ServiceCategoryResponse);

    // Knowledge Articles
    rpc CreateKnowledgeArticle(CreateKnowledgeArticleRequest) returns (KnowledgeArticleResponse);
    rpc GetKnowledgeArticle(GetByIdRequest) returns (KnowledgeArticleResponse);
    rpc SearchKnowledgeArticles(SearchArticlesRequest) returns (ArticleSearchResponse);
    rpc PublishArticle(PublishArticleRequest) returns (KnowledgeArticleResponse);
    rpc RateArticle(RateArticleRequest) returns (RateArticleResponse);

    // HR Cases
    rpc CreateCase(CreateCaseRequest) returns (CaseResponse);
    rpc GetCase(GetByIdRequest) returns (CaseResponse);
    rpc ListCases(ListCasesRequest) returns (CaseListResponse);
    rpc UpdateCase(UpdateCaseRequest) returns (CaseResponse);
    rpc AssignCase(AssignCaseRequest) returns (CaseResponse);
    rpc ResolveCase(ResolveCaseRequest) returns (CaseResponse);
    rpc CloseCase(CloseCaseRequest) returns (CaseResponse);
    rpc ReopenCase(ReopenCaseRequest) returns (CaseResponse);

    // Case Interactions
    rpc AddInteraction(AddInteractionRequest) returns (InteractionResponse);
    rpc ListInteractions(ListInteractionsRequest) returns (InteractionListResponse);

    // SLA Management
    rpc CreateSLADefinition(CreateSLARequest) returns (SLADefinitionResponse);
    rpc GetSLAStatus(GetSLAStatusRequest) returns (SLAStatusResponse);
    rpc GetSLADashboard(GetSLADashboardRequest) returns (SLADashboardResponse);

    // Self-Service
    rpc SubmitSelfServiceRequest(SubmitSelfServiceRequestReq) returns (SelfServiceResponse);
    rpc ApproveSelfServiceRequest(ApproveSelfServiceRequest) returns (SelfServiceResponse);

    // Chatbot
    rpc StartChatbotConversation(StartChatbotRequest) returns (ChatbotSessionResponse);
    rpc SendChatbotMessage(SendChatbotMessageRequest) returns (ChatbotMessageResponse);

    // Escalation
    rpc EscalateCase(EscalateCaseRequest) returns (CaseResponse);
    rpc CreateEscalationRule(CreateEscalationRuleRequest) returns (EscalationRuleResponse);

    // Surveys & Analytics
    rpc SubmitSurvey(SubmitSurveyRequest) returns (SurveyResponse);
    rpc GetHelpdeskAnalytics(GetAnalyticsRequest) returns (AnalyticsResponse);
}

message GetByIdRequest {
    string id = 1;
    string tenant_id = 2;
}

message CreateServiceCategoryRequest {
    string tenant_id = 1;
    string category_name = 2;
    string category_code = 3;
    string parent_category_id = 4;
    string description = 5;
    string default_assignee_group = 6;
    string default_sla_id = 7;
    string created_by = 8;
}

message ServiceCategoryResponse {
    string id = 1;
    string category_name = 2;
    string category_code = 3;
    string parent_category_id = 4;
    string default_sla_id = 5;
    int32 version = 6;
}

message ListCategoriesRequest {
    string tenant_id = 1;
    string parent_category_id = 2;
    int32 page = 3;
    int32 page_size = 4;
}

message ServiceCategoryListResponse {
    repeated ServiceCategoryResponse items = 1;
    int32 total_count = 2;
}

message CreateKnowledgeArticleRequest {
    string tenant_id = 1;
    string title = 2;
    string content = 3;
    string summary = 4;
    string category_id = 5;
    string tags = 6;
    string visibility = 7;
    string created_by = 8;
}

message KnowledgeArticleResponse {
    string id = 1;
    string title = 2;
    string article_number = 3;
    string status = 4;
    string visibility = 5;
    int32 view_count = 6;
    string published_at = 7;
    int32 version = 8;
}

message SearchArticlesRequest {
    string tenant_id = 1;
    string query = 2;
    string category_id = 3;
    int32 page = 4;
    int32 page_size = 5;
}

message ArticleSearchResponse {
    repeated KnowledgeArticleResponse items = 1;
    int32 total_count = 2;
    double search_score = 3;
}

message PublishArticleRequest {
    string id = 1;
    string tenant_id = 2;
    string published_by = 3;
}

message RateArticleRequest {
    string article_id = 1;
    string tenant_id = 2;
    bool helpful = 3;
    string rated_by = 4;
}

message RateArticleResponse {
    string article_id = 1;
    int32 helpful_count = 2;
    int32 not_helpful_count = 3;
}

message CreateCaseRequest {
    string tenant_id = 1;
    string subject = 2;
    string description = 3;
    string category_id = 4;
    string priority = 5;
    string case_type = 6;
    string source = 7;
    string requester_id = 8;
    string created_by = 9;
}

message CaseResponse {
    string id = 1;
    string case_number = 2;
    string subject = 3;
    string status = 4;
    string priority = 5;
    string assigned_to = 6;
    string due_date = 7;
    string first_response_at = 8;
    int32 version = 9;
}

message ListCasesRequest {
    string tenant_id = 1;
    string status = 2;
    string priority = 3;
    string assigned_to = 4;
    string requester_id = 5;
    int32 page = 6;
    int32 page_size = 7;
}

message CaseListResponse {
    repeated CaseResponse items = 1;
    int32 total_count = 2;
}

message UpdateCaseRequest {
    string id = 1;
    string tenant_id = 2;
    string subject = 3;
    string description = 4;
    string priority = 5;
    string updated_by = 6;
    int32 version = 7;
}

message AssignCaseRequest {
    string id = 1;
    string tenant_id = 2;
    string assigned_to = 3;
    string assigned_group = 4;
    string assigned_by = 5;
}

message ResolveCaseRequest {
    string id = 1;
    string tenant_id = 2;
    string resolution_type = 3;
    string resolution_notes = 4;
    string knowledge_article_id = 5;
    int32 time_spent_minutes = 6;
    string resolved_by = 7;
}

message CloseCaseRequest {
    string id = 1;
    string tenant_id = 2;
    string closed_by = 3;
}

message ReopenCaseRequest {
    string id = 1;
    string tenant_id = 2;
    string reason = 3;
    string reopened_by = 4;
}

message AddInteractionRequest {
    string tenant_id = 1;
    string case_id = 2;
    string interaction_type = 3;
    string content = 4;
    string author_id = 5;
    bool is_internal = 6;
    string channel = 7;
}

message InteractionResponse {
    string id = 1;
    string case_id = 2;
    string interaction_type = 3;
    string author_name = 4;
    string created_at = 5;
    bool is_internal = 6;
}

message ListInteractionsRequest {
    string case_id = 1;
    string tenant_id = 2;
    bool include_internal = 3;
}

message InteractionListResponse {
    repeated InteractionResponse items = 1;
}

message CreateSLARequest {
    string tenant_id = 1;
    string sla_name = 2;
    string sla_code = 3;
    int32 first_response_hours = 4;
    int32 resolution_hours = 5;
    string category_id = 6;
    string created_by = 7;
}

message SLADefinitionResponse {
    string id = 1;
    string sla_name = 2;
    int32 first_response_hours = 3;
    int32 resolution_hours = 4;
}

message GetSLAStatusRequest {
    string case_id = 1;
    string tenant_id = 2;
}

message SLAStatusResponse {
    string case_id = 1;
    bool first_response_breached = 2;
    bool resolution_breached = 3;
    string first_response_deadline = 4;
    string resolution_deadline = 5;
    int32 remaining_minutes = 6;
}

message GetSLADashboardRequest {
    string tenant_id = 1;
    string period = 2;
}

message SLADashboardResponse {
    double compliance_percentage = 1;
    int32 total_cases = 2;
    int32 breached_cases = 3;
    double avg_first_response_hours = 4;
    double avg_resolution_hours = 5;
}

message SubmitSelfServiceRequestReq {
    string tenant_id = 1;
    string request_type = 2;
    string employee_id = 3;
    string request_data = 4;
    string created_by = 5;
}

message SelfServiceResponse {
    string id = 1;
    string request_type = 2;
    string status = 3;
    string case_id = 4;
    string created_at = 5;
}

message ApproveSelfServiceRequest {
    string id = 1;
    string tenant_id = 2;
    string approver_id = 3;
    string notes = 4;
}

message StartChatbotRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string channel = 3;
    string language = 4;
}

message ChatbotSessionResponse {
    string session_id = 1;
    string greeting_message = 2;
}

message SendChatbotMessageRequest {
    string tenant_id = 1;
    string session_id = 2;
    string message = 3;
}

message ChatbotMessageResponse {
    string reply = 1;
    string intent_detected = 2;
    bool case_created = 3;
    string case_id = 4;
    repeated string articles_suggested = 5;
    bool escalation_triggered = 6;
}

message EscalateCaseRequest {
    string case_id = 1;
    string tenant_id = 2;
    string reason = 3;
    string escalated_by = 4;
}

message CreateEscalationRuleRequest {
    string tenant_id = 1;
    string rule_name = 2;
    string rule_type = 3;
    string condition_expression = 4;
    string escalation_target_id = 5;
    string created_by = 6;
}

message EscalationRuleResponse {
    string id = 1;
    string rule_name = 2;
    string rule_type = 3;
    int32 escalation_level = 4;
}

message SubmitSurveyRequest {
    string tenant_id = 1;
    string case_id = 2;
    string employee_id = 3;
    int32 overall_rating = 4;
    int32 responsiveness_rating = 5;
    int32 resolution_rating = 6;
    string comment = 7;
}

message SurveyResponse {
    string id = 1;
    string case_id = 2;
    int32 overall_rating = 3;
    string submitted_at = 4;
}

message GetAnalyticsRequest {
    string tenant_id = 1;
    string period_start = 2;
    string period_end = 3;
    string group_by = 4;
}

message AnalyticsResponse {
    int32 total_cases = 1;
    int32 resolved_cases = 2;
    int32 open_cases = 3;
    double avg_resolution_hours = 4;
    double avg_satisfaction = 5;
    double first_contact_resolution_rate = 6;
}
```

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data/Events | Purpose |
|---------------|-------------|---------|
| CORE-HR | Employee profiles, org structure, manager hierarchy | Requester validation, auto-assignment, escalation routing |
| BENEFITS | Benefit enrollments, claims, plan details | Benefits-related case resolution and self-service |
| ABSENCE | Leave balances, absence records, policies | Absence-related inquiries and self-service requests |

### Published To
| Target Service | Data/Events | Purpose |
|---------------|-------------|---------|
| CORE-HR | Case metadata, resolution data | Employee record updates from case outcomes |
| NOTIFICATION | Case alerts, SLA warnings, survey invitations | Agent and employee notifications |
| KNOWLEDGE-MGMT | Article suggestions, usage analytics | Knowledge base improvement |

## 7. Events

### Produced Events
| Event | Payload | Description |
|-------|---------|-------------|
| `CaseCreated` | `{ case_id, case_number, tenant_id, category_id, priority, requester_id, created_at }` | Emitted when a new HR case is created |
| `CaseUpdated` | `{ case_id, case_number, tenant_id, field_changed, old_value, new_value, updated_by }` | Emitted when a case is updated |
| `CaseResolved` | `{ case_id, case_number, tenant_id, resolution_type, resolved_by, time_spent_minutes }` | Emitted when a case is resolved |
| `SLABreached` | `{ case_id, case_number, tenant_id, sla_type, breach_minutes, assigned_to }` | Emitted when an SLA deadline is breached |
| `EscalationTriggered` | `{ case_id, tenant_id, escalation_level, escalation_target_id, rule_name }` | Emitted when an escalation rule fires |
| `CaseReopened` | `{ case_id, case_number, tenant_id, reason, reopened_by }` | Emitted when a closed case is reopened |
| `SurveySubmitted` | `{ case_id, tenant_id, overall_rating, employee_id }` | Emitted when a satisfaction survey is submitted |
| `ChatbotEscalated` | `{ session_id, tenant_id, employee_id, intent_detected, case_id }` | Emitted when a chatbot conversation is escalated to a live agent |
