# 77 - Sales Automation Service Specification

## 1. Domain Overview

Sales Automation provides a comprehensive CRM core with lead management, account and contact lifecycle tracking, opportunity pipeline management, activity logging, sales team collaboration, competitor tracking, and sales forecasting. Leads are captured from multiple sources (web, referral, event, cold outreach) and scored using configurable criteria before qualification and conversion into opportunities. Opportunities follow a configurable stage-based pipeline with probability-weighted revenue tracking. Activities (calls, emails, meetings, tasks) are logged against leads, contacts, and opportunities for full audit trails. Sales teams manage territory assignment and collaboration. Competitor intelligence is tracked per opportunity. Forecasts aggregate pipeline data by period, team, and territory for revenue projection. Integrates with Accounts Receivable for customer invoicing, Order Management for quote-to-order conversion, and Customer Data Platform for unified customer profiles.

**Bounded Context:** Sales CRM & Pipeline
**Service Name:** `sales-service`
**Database:** `data/sales.db`
**HTTP Port:** 8109 | **gRPC Port:** 9109

---

## 2. Database Schema

### 2.1 Leads
```sql
CREATE TABLE sales_leads (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lead_number TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    company_name TEXT,
    title TEXT,
    source TEXT NOT NULL
        CHECK(source IN ('WEB','REFERRAL','EVENT','COLD_CALL','ADVERTISEMENT','SOCIAL_MEDIA','PARTNER','INBOUND_CALL','OTHER')),
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','CONTACTED','QUALIFIED','UNQUALIFIED','NURTURE','CONVERTED','DISQUALIFIED')),
    score INTEGER NOT NULL DEFAULT 0,
    score_criteria TEXT,
    industry TEXT,
    estimated_revenue INTEGER,
    estimated_close_date TEXT,
    description TEXT,
    assigned_to TEXT,
    account_id TEXT,
    contact_id TEXT,
    converted_opportunity_id TEXT,
    converted_at TEXT,
    last_activity_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, lead_number)
);

CREATE INDEX idx_leads_tenant_status ON sales_leads(tenant_id, status);
CREATE INDEX idx_leads_tenant_source ON sales_leads(tenant_id, source);
CREATE INDEX idx_leads_tenant_assigned ON sales_leads(tenant_id, assigned_to);
CREATE INDEX idx_leads_tenant_active ON sales_leads(tenant_id, is_active);
CREATE INDEX idx_leads_tenant_score ON sales_leads(tenant_id, score);
```

### 2.2 Accounts
```sql
CREATE TABLE sales_accounts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    account_number TEXT NOT NULL,
    account_name TEXT NOT NULL,
    account_type TEXT NOT NULL DEFAULT 'PROSPECT'
        CHECK(account_type IN ('PROSPECT','CUSTOMER','PARTNER','VENDOR','COMPETITOR','OTHER')),
    industry TEXT,
    annual_revenue INTEGER,
    number_of_employees INTEGER,
    website TEXT,
    phone TEXT,
    fax TEXT,
    billing_street TEXT,
    billing_city TEXT,
    billing_state TEXT,
    billing_postal_code TEXT,
    billing_country TEXT,
    shipping_street TEXT,
    shipping_city TEXT,
    shipping_state TEXT,
    shipping_postal_code TEXT,
    shipping_country TEXT,
    parent_account_id TEXT,
    sic_code TEXT,
    naics_code TEXT,
    ticker_symbol TEXT,
    ownership TEXT,
    description TEXT,
    rating TEXT CHECK(rating IN ('HOT','WARM','COLD')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, account_number)
);

CREATE INDEX idx_accounts_tenant_type ON sales_accounts(tenant_id, account_type);
CREATE INDEX idx_accounts_tenant_industry ON sales_accounts(tenant_id, industry);
CREATE INDEX idx_accounts_tenant_parent ON sales_accounts(tenant_id, parent_account_id);
CREATE INDEX idx_accounts_tenant_active ON sales_accounts(tenant_id, is_active);
CREATE INDEX idx_accounts_tenant_name ON sales_accounts(tenant_id, account_name);
```

### 2.3 Contacts
```sql
CREATE TABLE sales_contacts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contact_number TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    mobile_phone TEXT,
    title TEXT,
    department TEXT,
    account_id TEXT,
    reports_to_contact_id TEXT,
    mailing_street TEXT,
    mailing_city TEXT,
    mailing_state TEXT,
    mailing_postal_code TEXT,
    mailing_country TEXT,
    lead_source TEXT,
    birthdate TEXT,
    preferred_contact_method TEXT CHECK(preferred_contact_method IN ('EMAIL','PHONE','SMS','IN_PERSON')),
    do_not_call INTEGER NOT NULL DEFAULT 0,
    do_not_email INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, contact_number)
);

CREATE INDEX idx_contacts_tenant_account ON sales_contacts(tenant_id, account_id);
CREATE INDEX idx_contacts_tenant_email ON sales_contacts(tenant_id, email);
CREATE INDEX idx_contacts_tenant_name ON sales_contacts(tenant_id, last_name, first_name);
CREATE INDEX idx_contacts_tenant_active ON sales_contacts(tenant_id, is_active);
```

### 2.4 Opportunities
```sql
CREATE TABLE sales_opportunities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    opportunity_number TEXT NOT NULL,
    opportunity_name TEXT NOT NULL,
    account_id TEXT NOT NULL,
    contact_id TEXT,
    stage_id TEXT NOT NULL,
    amount INTEGER NOT NULL DEFAULT 0,
    expected_revenue INTEGER NOT NULL DEFAULT 0,
    probability INTEGER NOT NULL DEFAULT 0,
    close_date TEXT NOT NULL,
    type TEXT CHECK(type IN ('NEW_BUSINESS','EXISTING_BUSINESS','RENEWAL','UPSELL','DOWNSSELL')),
    lead_source TEXT,
    campaign_id TEXT,
    assigned_to TEXT,
    team_id TEXT,
    next_step TEXT,
    description TEXT,
    lost_reason TEXT,
    competitor_id TEXT,
    closed_at TEXT,
    closed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, opportunity_number)
);

CREATE INDEX idx_opportunities_tenant_stage ON sales_opportunities(tenant_id, stage_id);
CREATE INDEX idx_opportunities_tenant_account ON sales_opportunities(tenant_id, account_id);
CREATE INDEX idx_opportunities_tenant_assigned ON sales_opportunities(tenant_id, assigned_to);
CREATE INDEX idx_opportunities_tenant_close_date ON sales_opportunities(tenant_id, close_date);
CREATE INDEX idx_opportunities_tenant_active ON sales_opportunities(tenant_id, is_active);
```

### 2.5 Opportunity Stages
```sql
CREATE TABLE sales_opportunity_stages (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    stage_name TEXT NOT NULL,
    stage_order INTEGER NOT NULL,
    probability INTEGER NOT NULL DEFAULT 0,
    is_won_stage INTEGER NOT NULL DEFAULT 0,
    is_lost_stage INTEGER NOT NULL DEFAULT 0,
    is_closed_stage INTEGER NOT NULL DEFAULT 0,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, stage_name)
);

CREATE INDEX idx_stages_tenant_order ON sales_opportunity_stages(tenant_id, stage_order);
CREATE INDEX idx_stages_tenant_active ON sales_opportunity_stages(tenant_id, is_active);
```

### 2.6 Activities
```sql
CREATE TABLE sales_activities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    activity_type TEXT NOT NULL
        CHECK(activity_type IN ('CALL','EMAIL','MEETING','TASK','NOTE','LOG_CALL','SEND_EMAIL','EVENT')),
    subject TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','CANCELLED')),
    priority TEXT CHECK(priority IN ('HIGH','MEDIUM','LOW')),
    lead_id TEXT,
    contact_id TEXT,
    account_id TEXT,
    opportunity_id TEXT,
    assigned_to TEXT NOT NULL,
    start_date TEXT,
    end_date TEXT,
    duration_minutes INTEGER,
    location TEXT,
    outcome TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_activities_tenant_type ON sales_activities(tenant_id, activity_type);
CREATE INDEX idx_activities_tenant_assigned ON sales_activities(tenant_id, assigned_to);
CREATE INDEX idx_activities_tenant_opportunity ON sales_activities(tenant_id, opportunity_id);
CREATE INDEX idx_activities_tenant_lead ON sales_activities(tenant_id, lead_id);
CREATE INDEX idx_activities_tenant_status ON sales_activities(tenant_id, status);
CREATE INDEX idx_activities_tenant_active ON sales_activities(tenant_id, is_active);
```

### 2.7 Sales Teams
```sql
CREATE TABLE sales_teams (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    team_name TEXT NOT NULL,
    team_code TEXT NOT NULL,
    manager_id TEXT NOT NULL,
    description TEXT,
    territory_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, team_code)
);

CREATE TABLE sales_team_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    team_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'MEMBER'
        CHECK(role IN ('MEMBER','LEAD','MANAGER','ADMIN')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, team_id, user_id)
);

CREATE INDEX idx_teams_tenant_active ON sales_teams(tenant_id, is_active);
CREATE INDEX idx_team_members_tenant_team ON sales_team_members(tenant_id, team_id);
CREATE INDEX idx_team_members_tenant_user ON sales_team_members(tenant_id, user_id);
```

### 2.8 Competitors
```sql
CREATE TABLE sales_competitors (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    competitor_name TEXT NOT NULL,
    competitor_company TEXT,
    strengths TEXT,
    weaknesses TEXT,
    threat_level TEXT CHECK(threat_level IN ('HIGH','MEDIUM','LOW')),
    website TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, competitor_name)
);

CREATE INDEX idx_competitors_tenant_active ON sales_competitors(tenant_id, is_active);
```

### 2.9 Sales Forecasts
```sql
CREATE TABLE sales_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    forecast_name TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    forecast_type TEXT NOT NULL
        CHECK(forecast_type IN ('REVENUE','UNITS','PIPELINE')),
    owner_id TEXT NOT NULL,
    team_id TEXT,
    territory_id TEXT,
    pipeline_amount INTEGER NOT NULL DEFAULT 0,
    weighted_amount INTEGER NOT NULL DEFAULT 0,
    committed_amount INTEGER NOT NULL DEFAULT 0,
    best_case_amount INTEGER NOT NULL DEFAULT 0,
    closed_won_amount INTEGER NOT NULL DEFAULT 0,
    quota_amount INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','APPROVED','FINAL')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_forecasts_tenant_period ON sales_forecasts(tenant_id, period_start, period_end);
CREATE INDEX idx_forecasts_tenant_owner ON sales_forecasts(tenant_id, owner_id);
CREATE INDEX idx_forecasts_tenant_team ON sales_forecasts(tenant_id, team_id);
CREATE INDEX idx_forecasts_tenant_status ON sales_forecasts(tenant_id, status);
CREATE INDEX idx_forecasts_tenant_active ON sales_forecasts(tenant_id, is_active);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/leads` | Create a new lead |
| GET | `/api/v1/leads` | List leads with filtering and pagination |
| GET | `/api/v1/leads/{id}` | Get lead by ID |
| PUT | `/api/v1/leads/{id}` | Update a lead |
| DELETE | `/api/v1/leads/{id}` | Delete (deactivate) a lead |
| POST | `/api/v1/leads/{id}/convert` | Convert a qualified lead to account, contact, and opportunity |
| POST | `/api/v1/leads/{id}/score` | Calculate and update lead score |
| GET | `/api/v1/leads/by-source` | Get leads grouped by source |
| POST | `/api/v1/accounts` | Create a new account |
| GET | `/api/v1/accounts` | List accounts with filtering and pagination |
| GET | `/api/v1/accounts/{id}` | Get account by ID with related contacts and opportunities |
| PUT | `/api/v1/accounts/{id}` | Update an account |
| DELETE | `/api/v1/accounts/{id}` | Delete (deactivate) an account |
| GET | `/api/v1/accounts/{id}/hierarchy` | Get account parent-child hierarchy |
| POST | `/api/v1/contacts` | Create a new contact |
| GET | `/api/v1/contacts` | List contacts with filtering and pagination |
| GET | `/api/v1/contacts/{id}` | Get contact by ID |
| PUT | `/api/v1/contacts/{id}` | Update a contact |
| DELETE | `/api/v1/contacts/{id}` | Delete (deactivate) a contact |
| POST | `/api/v1/opportunities` | Create a new opportunity |
| GET | `/api/v1/opportunities` | List opportunities with filtering and pagination |
| GET | `/api/v1/opportunities/{id}` | Get opportunity by ID with stage history |
| PUT | `/api/v1/opportunities/{id}` | Update an opportunity (including stage changes) |
| DELETE | `/api/v1/opportunities/{id}` | Delete (deactivate) an opportunity |
| PATCH | `/api/v1/opportunities/{id}/stage` | Advance or change opportunity stage |
| GET | `/api/v1/opportunities/pipeline` | Get pipeline summary by stage with totals |
| POST | `/api/v1/stages` | Create a new pipeline stage |
| GET | `/api/v1/stages` | List all pipeline stages in order |
| PUT | `/api/v1/stages/{id}` | Update a pipeline stage |
| DELETE | `/api/v1/stages/{id}` | Delete (deactivate) a pipeline stage |
| POST | `/api/v1/activities` | Log a new activity |
| GET | `/api/v1/activities` | List activities with filtering and pagination |
| GET | `/api/v1/activities/{id}` | Get activity by ID |
| PUT | `/api/v1/activities/{id}` | Update an activity |
| DELETE | `/api/v1/activities/{id}` | Delete an activity |
| GET | `/api/v1/activities/by-opportunity/{id}` | List activities for an opportunity |
| GET | `/api/v1/activities/by-contact/{id}` | List activities for a contact |
| POST | `/api/v1/teams` | Create a sales team |
| GET | `/api/v1/teams` | List sales teams |
| GET | `/api/v1/teams/{id}` | Get team with members |
| PUT | `/api/v1/teams/{id}` | Update team details |
| POST | `/api/v1/teams/{id}/members` | Add member to team |
| DELETE | `/api/v1/teams/{id}/members/{memberId}` | Remove member from team |
| POST | `/api/v1/competitors` | Create a competitor record |
| GET | `/api/v1/competitors` | List competitors |
| PUT | `/api/v1/competitors/{id}` | Update competitor |
| POST | `/api/v1/forecasts` | Create a sales forecast |
| GET | `/api/v1/forecasts` | List forecasts by period |
| GET | `/api/v1/forecasts/{id}` | Get forecast with pipeline breakdown |
| PUT | `/api/v1/forecasts/{id}` | Update forecast amounts |
| POST | `/api/v1/forecasts/{id}/submit` | Submit forecast for approval |
| PATCH | `/api/v1/forecasts/{id}/approve` | Approve a submitted forecast |
| GET | `/api/v1/forecasts/rollup` | Get rolled-up forecast by team or territory |
| GET | `/api/v1/dashboard/pipeline` | Get pipeline dashboard data |
| GET | `/api/v1/dashboard/performance` | Get sales performance metrics |

---

## 4. Business Rules

1. **Lead Scoring**: The system MUST calculate lead scores based on configurable criteria including source quality, engagement level, company fit (industry, size), and demographic match. Score MUST be an integer between 0 and 100.

2. **Lead Conversion**: A lead MUST have a status of `QUALIFIED` before conversion. The convert operation MUST create at minimum an account and contact, and SHOULD optionally create an opportunity. Upon conversion, the lead status MUST be set to `CONVERTED` and the `converted_at` timestamp MUST be recorded.

3. **Opportunity Stage Pipeline**: Opportunities MUST progress through stages defined by the tenant in ascending `stage_order`. A stage change MUST record the prior stage and timestamp. Moving an opportunity to a `won` or `lost` stage MUST set the `closed_at` field and the `closed_by` field.

4. **Probability Weighting**: When an opportunity stage changes, the system MUST update the `probability` field from the stage's default probability. The `expected_revenue` MUST be recalculated as `amount * probability / 100`.

5. **Account Hierarchy**: An account MAY be linked to a parent account. The system MUST prevent circular references in the account hierarchy. The hierarchy depth SHOULD NOT exceed 5 levels.

6. **Activity Association**: An activity MUST be associated with at least one entity (lead, contact, account, or opportunity). Activities with a `COMPLETED` status MUST have the `outcome` field populated.

7. **Duplicate Detection**: The system SHOULD detect duplicate leads based on email, phone, or company name and flag them for review. The system MUST NOT automatically merge duplicates without user confirmation.

8. **Forecast Aggregation**: A forecast's `pipeline_amount` MUST equal the sum of all open opportunity amounts for the forecast owner in the forecast period. The `weighted_amount` MUST equal the sum of all `expected_revenue` values. The `closed_won_amount` MUST equal the sum of all won opportunities in the period.

9. **Team Membership**: A user MAY belong to multiple sales teams but MUST have exactly one role per team. Removing a team manager MUST require assigning a new manager first.

10. **Opportunity Amount**: The `amount` field MUST be a positive integer representing cents. The system MUST NOT allow negative opportunity amounts. Zero is permitted for non-revenue opportunities.

11. **Close Date Validation**: An opportunity `close_date` MUST NOT be earlier than the opportunity creation date when the stage is not a closed stage. For closed-won or closed-lost opportunities, the `close_date` SHOULD match the `closed_at` date.

12. **Competitor Association**: A competitor MAY be associated with multiple opportunities. When an opportunity is marked lost, the `lost_reason` and `competitor_id` fields SHOULD be populated to support competitive analysis.

13. **Data Ownership**: Each lead, opportunity, and account MUST have an assigned owner (`assigned_to`). The system MUST enforce that only the assigned owner, their manager, or an administrator can reassign ownership.

14. **Forecast Submission**: A forecast MUST be in `DRAFT` status to be edited. Once `SUBMITTED`, the forecast owner MUST NOT modify amounts until the forecast is rejected or returned to `DRAFT` status. Only a manager MAY approve or reject a submitted forecast.

15. **Contact Do-Not-Contact**: When a contact has `do_not_call` or `do_not_email` set to 1, the system MUST prevent creation of call or email activities for that contact and SHOULD display a warning to the user.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package sales;

option go_package = "github.com/fusion/salespb";

service SalesService {
    // Leads
    rpc CreateLead(CreateLeadRequest) returns (LeadResponse);
    rpc GetLead(GetLeadRequest) returns (LeadResponse);
    rpc ListLeads(ListLeadsRequest) returns (ListLeadsResponse);
    rpc UpdateLead(UpdateLeadRequest) returns (LeadResponse);
    rpc DeleteLead(DeleteLeadRequest) returns (DeleteResponse);
    rpc ConvertLead(ConvertLeadRequest) returns (ConvertLeadResponse);
    rpc ScoreLead(ScoreLeadRequest) returns (LeadResponse);

    // Accounts
    rpc CreateAccount(CreateAccountRequest) returns (AccountResponse);
    rpc GetAccount(GetAccountRequest) returns (AccountResponse);
    rpc ListAccounts(ListAccountsRequest) returns (ListAccountsResponse);
    rpc UpdateAccount(UpdateAccountRequest) returns (AccountResponse);
    rpc DeleteAccount(DeleteAccountRequest) returns (DeleteResponse);
    rpc GetAccountHierarchy(GetAccountHierarchyRequest) returns (AccountHierarchyResponse);

    // Contacts
    rpc CreateContact(CreateContactRequest) returns (ContactResponse);
    rpc GetContact(GetContactRequest) returns (ContactResponse);
    rpc ListContacts(ListContactsRequest) returns (ListContactsResponse);
    rpc UpdateContact(UpdateContactRequest) returns (ContactResponse);
    rpc DeleteContact(DeleteContactRequest) returns (DeleteResponse);

    // Opportunities
    rpc CreateOpportunity(CreateOpportunityRequest) returns (OpportunityResponse);
    rpc GetOpportunity(GetOpportunityRequest) returns (OpportunityResponse);
    rpc ListOpportunities(ListOpportunitiesRequest) returns (ListOpportunitiesResponse);
    rpc UpdateOpportunity(UpdateOpportunityRequest) returns (OpportunityResponse);
    rpc DeleteOpportunity(DeleteOpportunityRequest) returns (DeleteResponse);
    rpc ChangeStage(ChangeStageRequest) returns (OpportunityResponse);
    rpc GetPipeline(PipelineRequest) returns (PipelineResponse);

    // Activities
    rpc CreateActivity(CreateActivityRequest) returns (ActivityResponse);
    rpc ListActivities(ListActivitiesRequest) returns (ListActivitiesResponse);
    rpc UpdateActivity(UpdateActivityRequest) returns (ActivityResponse);
    rpc DeleteActivity(DeleteActivityRequest) returns (DeleteResponse);

    // Forecasts
    rpc CreateForecast(CreateForecastRequest) returns (ForecastResponse);
    rpc GetForecast(GetForecastRequest) returns (ForecastResponse);
    rpc ListForecasts(ListForecastsRequest) returns (ListForecastsResponse);
    rpc UpdateForecast(UpdateForecastRequest) returns (ForecastResponse);
    rpc SubmitForecast(SubmitForecastRequest) returns (ForecastResponse);
    rpc ApproveForecast(ApproveForecastRequest) returns (ForecastResponse);
    rpc GetForecastRollup(ForecastRollupRequest) returns (ForecastRollupResponse);
}

message Lead {
    string id = 1;
    string tenant_id = 2;
    string lead_number = 3;
    string first_name = 4;
    string last_name = 5;
    string email = 6;
    string phone = 7;
    string company_name = 8;
    string title = 9;
    string source = 10;
    string status = 11;
    int32 score = 12;
    string score_criteria = 13;
    string industry = 14;
    int64 estimated_revenue = 15;
    string estimated_close_date = 16;
    string description = 17;
    string assigned_to = 18;
    string account_id = 19;
    string contact_id = 20;
    string converted_opportunity_id = 21;
    string converted_at = 22;
    string last_activity_date = 23;
    string created_at = 24;
    string updated_at = 25;
    int32 version = 26;
}

message Account {
    string id = 1;
    string tenant_id = 2;
    string account_number = 3;
    string account_name = 4;
    string account_type = 5;
    string industry = 6;
    int64 annual_revenue = 7;
    int32 number_of_employees = 8;
    string website = 9;
    string phone = 10;
    string billing_city = 11;
    string billing_state = 12;
    string billing_country = 13;
    string parent_account_id = 14;
    string rating = 15;
    string created_at = 16;
    string updated_at = 17;
    int32 version = 18;
}

message Contact {
    string id = 1;
    string tenant_id = 2;
    string contact_number = 3;
    string first_name = 4;
    string last_name = 5;
    string email = 6;
    string phone = 7;
    string title = 8;
    string account_id = 9;
    string created_at = 10;
    string updated_at = 11;
    int32 version = 12;
}

message Opportunity {
    string id = 1;
    string tenant_id = 2;
    string opportunity_number = 3;
    string opportunity_name = 4;
    string account_id = 5;
    string contact_id = 6;
    string stage_id = 7;
    int64 amount = 8;
    int64 expected_revenue = 9;
    int32 probability = 10;
    string close_date = 11;
    string type = 12;
    string assigned_to = 13;
    string closed_at = 14;
    string created_at = 15;
    string updated_at = 16;
    int32 version = 17;
}

message Activity {
    string id = 1;
    string tenant_id = 2;
    string activity_type = 3;
    string subject = 4;
    string description = 5;
    string status = 6;
    string priority = 7;
    string lead_id = 8;
    string contact_id = 9;
    string account_id = 10;
    string opportunity_id = 11;
    string assigned_to = 12;
    string start_date = 13;
    string end_date = 14;
    int32 duration_minutes = 15;
    string outcome = 16;
    string created_at = 17;
    string updated_at = 18;
    int32 version = 19;
}

message Forecast {
    string id = 1;
    string tenant_id = 2;
    string forecast_name = 3;
    string period_start = 4;
    string period_end = 5;
    string forecast_type = 6;
    string owner_id = 7;
    int64 pipeline_amount = 8;
    int64 weighted_amount = 9;
    int64 committed_amount = 10;
    int64 best_case_amount = 11;
    int64 closed_won_amount = 12;
    int64 quota_amount = 13;
    string status = 14;
    string created_at = 15;
    string updated_at = 16;
    int32 version = 17;
}

message CreateLeadRequest { Lead lead = 1; }
message GetLeadRequest { string tenant_id = 1; string id = 2; }
message ListLeadsRequest {
    string tenant_id = 1;
    string status = 2;
    string source = 3;
    string assigned_to = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message UpdateLeadRequest { Lead lead = 1; }
message DeleteLeadRequest { string tenant_id = 1; string id = 2; }
message ConvertLeadRequest {
    string tenant_id = 1;
    string id = 2;
    bool create_opportunity = 3;
    string opportunity_name = 4;
    int64 amount = 5;
    string close_date = 6;
}
message ScoreLeadRequest { string tenant_id = 1; string id = 2; }
message LeadResponse { Lead lead = 1; }
message ListLeadsResponse { repeated Lead leads = 1; int32 total = 2; }
message ConvertLeadResponse {
    Lead lead = 1;
    Account account = 2;
    Contact contact = 3;
    Opportunity opportunity = 4;
}

message CreateAccountRequest { Account account = 1; }
message GetAccountRequest { string tenant_id = 1; string id = 2; }
message ListAccountsRequest {
    string tenant_id = 1;
    string account_type = 2;
    string industry = 3;
    int32 page = 4;
    int32 page_size = 5;
}
message UpdateAccountRequest { Account account = 1; }
message DeleteAccountRequest { string tenant_id = 1; string id = 2; }
message GetAccountHierarchyRequest { string tenant_id = 1; string id = 2; }
message AccountResponse { Account account = 1; }
message ListAccountsResponse { repeated Account accounts = 1; int32 total = 2; }
message AccountHierarchyResponse {
    Account account = 1;
    repeated AccountHierarchyResponse children = 2;
}

message CreateContactRequest { Contact contact = 1; }
message GetContactRequest { string tenant_id = 1; string id = 2; }
message ListContactsRequest {
    string tenant_id = 1;
    string account_id = 2;
    int32 page = 3;
    int32 page_size = 4;
}
message UpdateContactRequest { Contact contact = 1; }
message DeleteContactRequest { string tenant_id = 1; string id = 2; }
message ContactResponse { Contact contact = 1; }
message ListContactsResponse { repeated Contact contacts = 1; int32 total = 2; }

message CreateOpportunityRequest { Opportunity opportunity = 1; }
message GetOpportunityRequest { string tenant_id = 1; string id = 2; }
message ListOpportunitiesRequest {
    string tenant_id = 1;
    string stage_id = 2;
    string account_id = 3;
    string assigned_to = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message UpdateOpportunityRequest { Opportunity opportunity = 1; }
message DeleteOpportunityRequest { string tenant_id = 1; string id = 2; }
message ChangeStageRequest { string tenant_id = 1; string id = 2; string stage_id = 3; }
message PipelineRequest { string tenant_id = 1; string team_id = 2; string owner_id = 3; }
message OpportunityResponse { Opportunity opportunity = 1; }
message ListOpportunitiesResponse { repeated Opportunity opportunities = 1; int32 total = 2; }
message PipelineStage {
    string stage_id = 1;
    string stage_name = 2;
    int32 count = 3;
    int64 total_amount = 4;
    int64 weighted_amount = 5;
}
message PipelineResponse {
    repeated PipelineStage stages = 1;
    int64 total_pipeline = 2;
    int64 total_weighted = 3;
}

message CreateActivityRequest { Activity activity = 1; }
message ListActivitiesRequest {
    string tenant_id = 1;
    string opportunity_id = 2;
    string contact_id = 3;
    string lead_id = 4;
    string activity_type = 5;
    int32 page = 6;
    int32 page_size = 7;
}
message UpdateActivityRequest { Activity activity = 1; }
message DeleteActivityRequest { string tenant_id = 1; string id = 2; }
message ActivityResponse { Activity activity = 1; }
message ListActivitiesResponse { repeated Activity activities = 1; int32 total = 2; }

message CreateForecastRequest { Forecast forecast = 1; }
message GetForecastRequest { string tenant_id = 1; string id = 2; }
message ListForecastsRequest {
    string tenant_id = 1;
    string period_start = 2;
    string period_end = 3;
    string owner_id = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message UpdateForecastRequest { Forecast forecast = 1; }
message SubmitForecastRequest { string tenant_id = 1; string id = 2; }
message ApproveForecastRequest { string tenant_id = 1; string id = 2; bool approve = 3; }
message ForecastRollupRequest {
    string tenant_id = 1;
    string period_start = 2;
    string period_end = 3;
    string team_id = 4;
}
message ForecastResponse { Forecast forecast = 1; }
message ListForecastsResponse { repeated Forecast forecasts = 1; int32 total = 2; }
message ForecastRollupResponse {
    repeated Forecast forecasts = 1;
    int64 total_pipeline = 2;
    int64 total_weighted = 3;
    int64 total_committed = 4;
    int64 total_closed_won = 5;
    int64 total_quota = 6;
}

message DeleteResponse { bool success = 1; }
```

---

## 6. Inter-Service Integration

| Integration | Service | Direction | Purpose |
|-------------|---------|-----------|---------|
| Accounts Receivable | `ar-service` | Outbound | Create customer in AR when opportunity is won; sync account billing details |
| Accounts Receivable | `ar-service` | Inbound | Receive payment status updates linked to customer accounts |
| Order Management | `om-service` | Outbound | Create sales order from won opportunity; pass quote and pricing details |
| Order Management | `om-service` | Inbound | Receive order fulfillment status updates for opportunity tracking |
| Customer Data Platform | `cdp-service` | Outbound | Publish lead, contact, and account data for unified customer profiles |
| Customer Data Platform | `cdp-service` | Inbound | Receive enriched customer data, segmentation, and engagement scores |
| Sales Planning | `salesplanning-service` | Outbound | Provide territory assignment and quota data for pipeline reports |
| Sales Planning | `salesplanning-service` | Inbound | Receive territory and quota allocations for forecasting |
| Sales Performance | `salesperf-service` | Outbound | Publish opportunity win/loss events and revenue data for commission calculation |
| Sales Performance | `salesperf-service` | Inbound | Receive commission credit assignments linked to opportunities |
| General Ledger | `gl-service` | Inbound | Receive revenue recognition data for booked deals |
| Identity Service | `identity-service` | Inbound | Validate user assignments, team membership, and role permissions |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `LeadCreated` | `{ tenant_id, lead_id, lead_number, source, assigned_to, score, created_at }` | Published when a new lead is created in the system |
| `LeadScored` | `{ tenant_id, lead_id, old_score, new_score, score_criteria, scored_at }` | Published when a lead's score is calculated or updated |
| `LeadQualified` | `{ tenant_id, lead_id, lead_number, qualified_by, qualified_at }` | Published when a lead status changes to QUALIFIED |
| `LeadConverted` | `{ tenant_id, lead_id, account_id, contact_id, opportunity_id, converted_by, converted_at }` | Published when a qualified lead is converted to account/contact/opportunity |
| `AccountCreated` | `{ tenant_id, account_id, account_number, account_name, account_type, industry, created_at }` | Published when a new account is created |
| `AccountUpdated` | `{ tenant_id, account_id, account_number, changed_fields, updated_at }` | Published when account details are modified |
| `ContactCreated` | `{ tenant_id, contact_id, contact_number, account_id, email, created_at }` | Published when a new contact is created |
| `OpportunityCreated` | `{ tenant_id, opportunity_id, opportunity_number, account_id, amount, stage_id, assigned_to, created_at }` | Published when a new opportunity is created |
| `OpportunityStageChanged` | `{ tenant_id, opportunity_id, from_stage_id, to_stage_id, amount, probability, changed_at }` | Published when an opportunity moves to a different pipeline stage |
| `OpportunityWon` | `{ tenant_id, opportunity_id, opportunity_number, account_id, amount, closed_at, closed_by }` | Published when an opportunity is marked as won |
| `OpportunityLost` | `{ tenant_id, opportunity_id, opportunity_number, account_id, lost_reason, competitor_id, closed_at, closed_by }` | Published when an opportunity is marked as lost |
| `ActivityCreated` | `{ tenant_id, activity_id, activity_type, subject, lead_id, contact_id, opportunity_id, assigned_to, created_at }` | Published when a new activity is logged |
| `ActivityCompleted` | `{ tenant_id, activity_id, activity_type, outcome, completed_at }` | Published when an activity is marked as completed |
| `ForecastSubmitted` | `{ tenant_id, forecast_id, owner_id, period_start, period_end, committed_amount, submitted_at }` | Published when a sales forecast is submitted for approval |
| `ForecastApproved` | `{ tenant_id, forecast_id, approved_by, approved_at }` | Published when a forecast is approved by management |
