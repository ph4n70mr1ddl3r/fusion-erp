# 181 - Eloqua Service Specification

## 1. Domain Overview

Eloqua provides B2B marketing automation, lead management, and campaign orchestration for enterprise marketing teams. Supports multi-channel campaign execution across email, events, webinars, nurture tracks, and advertisements with sophisticated lead scoring models combining behavioral, demographic, and firmographic criteria. Enables marketers to build dynamic contact segments, execute A/B tested email campaigns, track engagement across the buyer journey, and automate lead-to-revenue workflows. Integrates with Sales Automation for lead handoff, Customer Data Platform for unified profiles, and CRM systems for closed-loop ROI analysis.

**Bounded Context:** B2B Marketing Automation & Lead Management
**Service Name:** `eloqua-service`
**Database:** `data/eloqua.db`
**HTTP Port:** 8199 | **gRPC Port:** 9199

---

## 2. Database Schema

### 2.1 Campaigns
```sql
CREATE TABLE elo_campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_code TEXT NOT NULL,
    campaign_name TEXT NOT NULL,
    campaign_type TEXT NOT NULL CHECK(campaign_type IN ('EMAIL','EVENT','WEBINAR','NURTURE','ADVERTISEMENT')),
    description TEXT,
    objective TEXT,
    target_audience TEXT,                        -- JSON: segmentation criteria
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    budget_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    expected_leads INTEGER NOT NULL DEFAULT 0,
    expected_pipeline_cents INTEGER NOT NULL DEFAULT 0,
    actual_leads INTEGER NOT NULL DEFAULT 0,
    actual_pipeline_cents INTEGER NOT NULL DEFAULT 0,
    roi_pct REAL NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SCHEDULED','ACTIVE','PAUSED','COMPLETED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, campaign_code)
);

CREATE INDEX idx_elo_camp_tenant ON elo_campaigns(tenant_id, status);
CREATE INDEX idx_elo_camp_dates ON elo_campaigns(start_date, end_date);
CREATE INDEX idx_elo_camp_owner ON elo_campaigns(owner_id);
```

### 2.2 Email Templates
```sql
CREATE TABLE elo_email_templates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    template_code TEXT NOT NULL,
    template_name TEXT NOT NULL,
    subject_line TEXT NOT NULL,
    preheader_text TEXT,
    html_body TEXT NOT NULL,
    plain_text_body TEXT,
    dynamic_content_regions TEXT,                -- JSON: regions with personalization rules
    ab_variant_group TEXT,                       -- Group ID for A/B testing
    ab_variant_label TEXT CHECK(ab_variant_label IS NULL OR ab_variant_label IN ('A','B','C')),
    from_name TEXT NOT NULL,
    from_email TEXT NOT NULL,
    reply_to_email TEXT,
    tracking_enabled INTEGER NOT NULL DEFAULT 1,
    unsubscribe_link INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','IN_REVIEW','APPROVED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, template_code)
);

CREATE INDEX idx_elo_tmpl_tenant ON elo_email_templates(tenant_id, status);
CREATE INDEX idx_elo_tmpl_variant ON elo_email_templates(ab_variant_group);
```

### 2.3 Lead Scoring Models
```sql
CREATE TABLE elo_lead_scoring_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    description TEXT,
    criteria TEXT NOT NULL,                      -- JSON: scoring criteria definitions
    behavioral_criteria TEXT,                    -- JSON: page visits, downloads, email engagement
    demographic_criteria TEXT,                   -- JSON: title, industry, company size
    firmographic_criteria TEXT,                  -- JSON: revenue, employee count, technology stack
    max_score INTEGER NOT NULL DEFAULT 100,
    hot_threshold INTEGER NOT NULL DEFAULT 80,
    warm_threshold INTEGER NOT NULL DEFAULT 50,
    cold_threshold INTEGER NOT NULL DEFAULT 20,
    grade_mapping TEXT NOT NULL,                 -- JSON: score-to-grade mapping (A/B/C/D/F)
    decay_rules TEXT,                            -- JSON: score decay over inactivity
    is_default INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_elo_score_tenant ON elo_lead_scoring_models(tenant_id, status);
CREATE INDEX idx_elo_score_default ON elo_lead_scoring_models(tenant_id, is_default);
```

### 2.4 Contact Lists
```sql
CREATE TABLE elo_contact_lists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    list_code TEXT NOT NULL,
    list_name TEXT NOT NULL,
    description TEXT,
    list_type TEXT NOT NULL CHECK(list_type IN ('DYNAMIC','STATIC')),
    filter_criteria TEXT,                        -- JSON: segment rules for dynamic lists
    contact_count INTEGER NOT NULL DEFAULT 0,
    growth_count INTEGER NOT NULL DEFAULT 0,     -- Net growth since creation
    last_refreshed_at TEXT,
    refresh_schedule TEXT,                       -- JSON: cron schedule for dynamic refresh
    last_growth_tracked_at TEXT,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, list_code)
);

CREATE INDEX idx_elo_list_tenant ON elo_contact_lists(tenant_id, status);
CREATE INDEX idx_elo_list_owner ON elo_contact_lists(owner_id);
CREATE INDEX idx_elo_list_type ON elo_contact_lists(tenant_id, list_type);
```

### 2.5 Campaign Engagement
```sql
CREATE TABLE elo_campaign_engagement (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    contact_id TEXT NOT NULL,
    contact_email TEXT NOT NULL,
    engagement_type TEXT NOT NULL CHECK(engagement_type IN (
        'EMAIL_SENT','OPENED','CLICKED','BOUNCED','FORM_SUBMITTED',
        'PAGE_VISITED','WEBINAR_ATTENDED','DOWNLOAD','UNSUBSCRIBED'
    )),
    email_template_id TEXT,
    url_visited TEXT,
    ip_address TEXT,
    user_agent TEXT,
    engagement_data TEXT,                        -- JSON: event-specific details
    score_impact INTEGER NOT NULL DEFAULT 0,     -- Lead score change from this event
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (campaign_id) REFERENCES elo_campaigns(id) ON DELETE CASCADE
);

CREATE INDEX idx_elo_eng_campaign ON elo_campaign_engagement(campaign_id, engagement_type);
CREATE INDEX idx_elo_eng_contact ON elo_campaign_engagement(contact_id, timestamp DESC);
CREATE INDEX idx_elo_eng_type ON elo_campaign_engagement(tenant_id, engagement_type);
CREATE INDEX idx_elo_eng_time ON elo_campaign_engagement(timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Campaigns
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/eloqua/campaigns` | Create campaign |
| GET | `/api/v1/eloqua/campaigns` | List campaigns |
| GET | `/api/v1/eloqua/campaigns/{id}` | Get campaign details |
| PUT | `/api/v1/eloqua/campaigns/{id}` | Update campaign |
| POST | `/api/v1/eloqua/campaigns/{id}/launch` | Launch campaign |
| POST | `/api/v1/eloqua/campaigns/{id}/pause` | Pause campaign |
| POST | `/api/v1/eloqua/campaigns/{id}/complete` | Complete campaign |

### 3.2 Email Templates
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/eloqua/email-templates` | Create email template |
| GET | `/api/v1/eloqua/email-templates` | List templates |
| GET | `/api/v1/eloqua/email-templates/{id}` | Get template details |
| PUT | `/api/v1/eloqua/email-templates/{id}` | Update template |
| POST | `/api/v1/eloqua/email-templates/{id}/approve` | Approve template |
| POST | `/api/v1/eloqua/email-templates/{id}/preview` | Preview rendered template |

### 3.3 Lead Scoring
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/eloqua/lead-scoring/models` | Create scoring model |
| GET | `/api/v1/eloqua/lead-scoring/models` | List scoring models |
| GET | `/api/v1/eloqua/lead-scoring/models/{id}` | Get model details |
| PUT | `/api/v1/eloqua/lead-scoring/models/{id}` | Update model |
| GET | `/api/v1/eloqua/lead-scoring/contacts/{id}/score` | Get contact lead score |
| POST | `/api/v1/eloqua/lead-scoring/recalculate` | Recalculate all scores |

### 3.4 Contact Lists
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/eloqua/contact-lists` | Create contact list |
| GET | `/api/v1/eloqua/contact-lists` | List contact lists |
| GET | `/api/v1/eloqua/contact-lists/{id}` | Get list details |
| PUT | `/api/v1/eloqua/contact-lists/{id}` | Update list |
| POST | `/api/v1/eloqua/contact-lists/{id}/refresh` | Refresh dynamic list |
| POST | `/api/v1/eloqua/contact-lists/{id}/add-contacts` | Add contacts to static list |

### 3.5 Engagement Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/eloqua/campaigns/{id}/analytics` | Campaign performance |
| GET | `/api/v1/eloqua/contacts/{id}/engagement` | Contact engagement timeline |
| GET | `/api/v1/eloqua/analytics/dashboard` | Marketing dashboard |
| GET | `/api/v1/eloqua/analytics/email-performance` | Email performance report |
| GET | `/api/v1/eloqua/analytics/roi` | Campaign ROI report |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `eloqua.campaign.launched` | `{ campaign_id, campaign_type, contact_count }` | Campaign launched |
| `eloqua.lead.score_changed` | `{ contact_id, old_score, new_score, old_grade, new_grade }` | Lead score updated |
| `eloqua.engagement.high_intent` | `{ contact_id, campaign_id, engagement_type, score }` | High-intent engagement detected |
| `eloqua.contact.bounced` | `{ contact_id, email, bounce_type }` | Email bounced for contact |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `opportunity.created` | Sales Automation | Attribute pipeline to campaign |
| `contact.created` | Customer Data Platform | Add to Eloqua contact database |
| `lead.converted` | Sales Automation | Remove from active nurture campaigns |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.eloqua.v1;

service EloquaService {
    rpc GetCampaign(GetCampaignRequest) returns (GetCampaignResponse);
    rpc CreateCampaign(CreateCampaignRequest) returns (CreateCampaignResponse);
    rpc GetEmailTemplate(GetEmailTemplateRequest) returns (GetEmailTemplateResponse);
    rpc GetLeadScore(GetLeadScoreRequest) returns (GetLeadScoreResponse);
    rpc RecordEngagement(RecordEngagementRequest) returns (RecordEngagementResponse);
    rpc GetCampaignAnalytics(GetCampaignAnalyticsRequest) returns (GetCampaignAnalyticsResponse);
}

// Campaign messages
message GetCampaignRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetCampaignResponse {
    EloCampaign data = 1;
}

message CreateCampaignRequest {
    string tenant_id = 1;
    string campaign_code = 2;
    string campaign_name = 3;
    string campaign_type = 4;
    string description = 5;
    string objective = 6;
    string target_audience = 7;
    string start_date = 8;
    string end_date = 9;
    int64 budget_cents = 10;
    string currency_code = 11;
    int32 expected_leads = 12;
    int64 expected_pipeline_cents = 13;
    string owner_id = 14;
}

message CreateCampaignResponse {
    EloCampaign data = 1;
}

message EloCampaign {
    string id = 1;
    string tenant_id = 2;
    string campaign_code = 3;
    string campaign_name = 4;
    string campaign_type = 5;
    string description = 6;
    string objective = 7;
    string target_audience = 8;
    string start_date = 9;
    string end_date = 10;
    int64 budget_cents = 11;
    int64 actual_cost_cents = 12;
    string currency_code = 13;
    int32 expected_leads = 14;
    int64 expected_pipeline_cents = 15;
    int32 actual_leads = 16;
    int64 actual_pipeline_cents = 17;
    double roi_pct = 18;
    string owner_id = 19;
    string status = 20;
    string created_at = 21;
    string updated_at = 22;
}

// Email template messages
message GetEmailTemplateRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetEmailTemplateResponse {
    EloEmailTemplate data = 1;
}

message EloEmailTemplate {
    string id = 1;
    string tenant_id = 2;
    string template_code = 3;
    string template_name = 4;
    string subject_line = 5;
    string preheader_text = 6;
    string html_body = 7;
    string plain_text_body = 8;
    string from_name = 9;
    string from_email = 10;
    string reply_to_email = 11;
    string ab_variant_group = 12;
    string ab_variant_label = 13;
    string status = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Lead scoring messages
message GetLeadScoreRequest {
    string tenant_id = 1;
    string contact_id = 2;
}

message GetLeadScoreResponse {
    int32 score = 1;
    string grade = 2;
    string score_trend = 3;
}

// Engagement messages
message RecordEngagementRequest {
    string tenant_id = 1;
    string campaign_id = 2;
    string contact_id = 3;
    string contact_email = 4;
    string engagement_type = 5;
    string email_template_id = 6;
    string url_visited = 7;
    string engagement_data = 8;
}

message RecordEngagementResponse {
    string id = 1;
    string engagement_type = 2;
    int32 score_impact = 3;
    string timestamp = 4;
}

// Analytics messages
message GetCampaignAnalyticsRequest {
    string tenant_id = 1;
    string campaign_id = 2;
}

message GetCampaignAnalyticsResponse {
    string campaign_id = 1;
    int32 emails_sent = 2;
    int32 opens = 3;
    int32 clicks = 4;
    int32 bounces = 5;
    int32 form_submissions = 6;
    double open_rate = 7;
    double click_rate = 8;
    int32 leads_generated = 9;
    int64 pipeline_cents = 10;
    double roi_pct = 11;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | elo_campaigns | -- |
| V002 | elo_email_templates | -- |
| V003 | elo_lead_scoring_models | -- |
| V004 | elo_contact_lists | -- |
| V005 | elo_campaign_engagement | V001 |

---

## 7. Business Rules

1. **Lead Scoring Thresholds**: Contacts exceeding hot threshold trigger automatic sales handoff notification
2. **Score Decay**: Inactive contacts lose points over time based on configurable decay rules
3. **Bounce Management**: Hard bounces suppress future email sends; soft bounces allow limited retries
4. **Dynamic Lists**: Segments refresh on schedule; contacts auto-added/removed based on criteria
5. **Unsubscribe Compliance**: Unsubscribe events immediately suppress all future email communications
6. **A/B Testing**: Campaign emails with variant groups split audience 50/50; auto-promote winner after test period
7. **Budget Tracking**: Campaign actual costs update in real-time as sends execute; alert on budget threshold breach

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| sales-service | `GetOpportunity` / `GetLead` | Lead handoff and opportunity attribution |
| cdp-service | `GetProfile` / `GetSegment` | Unified contact profiles and segmentation |
| marketing-service | `GetContent` | Shared content and campaign coordination |
| workflow-service | `SubmitApproval` | Campaign approval workflows |
| notification-service | `SendAlert` | Alert on high-intent leads |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| cx-analytics-service | `GetCampaignAnalytics` | Unified marketing performance analytics |
| sales-engagement-service | `EnrollContact` | Sequence enrollment for hot leads |
