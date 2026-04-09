# 61 - Marketing Automation Service Specification

## 1. Domain Overview

Marketing Automation provides campaign management and orchestration across multiple channels (email, social, web, events), audience targeting through segment-based and rule-based selection, creative asset management, multi-touch campaign execution with scheduled and triggered touches, lead-campaign association tracking, response and engagement analytics, configurable lead scoring models with per-lead score tracking, and marketing ROI analytics with attribution modeling and funnel metrics. Integrates with SALES-AUTOMATION for lead handoff, CDP for customer profiles and segments, and COMMERCE for web engagement tracking.

**Bounded Context:** Marketing Campaign Management, Lead Scoring & Marketing Analytics
**Service Name:** `marketing-service`
**Database:** `data/marketing.db`
**HTTP Port:** 8093 | **gRPC Port:** 9093

---

## 2. Database Schema

### 2.1 Campaigns
```sql
CREATE TABLE campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    campaign_type TEXT NOT NULL
        CHECK(campaign_type IN ('EMAIL','SOCIAL','WEB','EVENT','MULTI_CHANNEL','NURTURE','ABM','WEBINAR')),
    campaign_purpose TEXT NOT NULL DEFAULT 'AWARENESS'
        CHECK(campaign_purpose IN ('AWARENESS','LEAD_GENERATION','NURTURE','UPSELL','CROSS_SELL',
                                    'RETENTION','RE_ENGAGEMENT','EVENT_PROMOTION','PRODUCT_LAUNCH')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SCHEDULED','ACTIVE','PAUSED','COMPLETED','CANCELLED')),
    start_date TEXT NOT NULL,
    end_date TEXT,
    budget_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    expected_revenue_cents INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,                        -- Campaign manager
    team TEXT,                                     -- JSON array: team member IDs
    tags TEXT,                                     -- JSON array: campaign tags
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    parent_campaign_id TEXT,                       -- For sub-campaigns
    recurrence TEXT,                               -- JSON: recurrence rules for recurring campaigns

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, campaign_code)
);

CREATE INDEX idx_campaigns_tenant_type ON campaigns(tenant_id, campaign_type);
CREATE INDEX idx_campaigns_tenant_status ON campaigns(tenant_id, status);
CREATE INDEX idx_campaigns_tenant_owner ON campaigns(tenant_id, owner_id);
CREATE INDEX idx_campaigns_tenant_dates ON campaigns(tenant_id, start_date, end_date);
CREATE INDEX idx_campaigns_tenant_purpose ON campaigns(tenant_id, campaign_purpose);
CREATE INDEX idx_campaigns_tenant_parent ON campaigns(tenant_id, parent_campaign_id);
CREATE INDEX idx_campaigns_tenant_active ON campaigns(tenant_id, is_active);
```

### 2.2 Campaign Channels
```sql
CREATE TABLE campaign_channels (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    channel_type TEXT NOT NULL
        CHECK(channel_type IN ('EMAIL','SMS','PUSH','SOCIAL_FACEBOOK','SOCIAL_LINKEDIN',
                                'SOCIAL_TWITTER','WEB_BANNER','WEB_POPUP','DIRECT_MAIL',
                                'IN_APP','WHATSAPP','VOICE')),
    channel_config TEXT NOT NULL,                  -- JSON: channel-specific configuration
    sender_name TEXT,
    sender_address TEXT,
    reply_to TEXT,
    template_id TEXT,                              -- Link to content template
    daily_send_limit INTEGER NOT NULL DEFAULT 0,   -- 0 = unlimited
    total_send_limit INTEGER NOT NULL DEFAULT 0,
    sent_count INTEGER NOT NULL DEFAULT 0,
    delivered_count INTEGER NOT NULL DEFAULT 0,
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (campaign_id) REFERENCES campaigns(id)
);

CREATE INDEX idx_campaign_channels_tenant_campaign ON campaign_channels(tenant_id, campaign_id);
CREATE INDEX idx_campaign_channels_tenant_type ON campaign_channels(tenant_id, channel_type);
CREATE INDEX idx_campaign_channels_tenant_primary ON campaign_channels(tenant_id, is_primary);
```

### 2.3 Campaign Audiences
```sql
CREATE TABLE campaign_audiences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    audience_name TEXT NOT NULL,
    audience_type TEXT NOT NULL
        CHECK(audience_type IN ('SEGMENT','RULE_BASED','LIST','CRM_QUERY','MANUAL')),
    segment_id TEXT,                               -- For SEGMENT audience type
    audience_rules TEXT,                           -- JSON: targeting criteria for RULE_BASED
    profile_list TEXT,                             -- JSON array: profile IDs for LIST/MANUAL
    estimated_size INTEGER NOT NULL DEFAULT 0,
    actual_size INTEGER NOT NULL DEFAULT 0,
    exclusion_list TEXT,                           -- JSON array: profile IDs to exclude
    suppression_segment_id TEXT,                   -- Segment of profiles to suppress
    is_control_group INTEGER NOT NULL DEFAULT 0,   -- Control group for A/B testing
    control_group_percent REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (campaign_id) REFERENCES campaigns(id)
);

CREATE INDEX idx_audiences_tenant_campaign ON campaign_audiences(tenant_id, campaign_id);
CREATE INDEX idx_audiences_tenant_type ON campaign_audiences(tenant_id, audience_type);
CREATE INDEX idx_audiences_tenant_segment ON campaign_audiences(tenant_id, segment_id);
```

### 2.4 Campaign Assets
```sql
CREATE TABLE campaign_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_code TEXT NOT NULL,
    asset_name TEXT NOT NULL,
    description TEXT,
    asset_type TEXT NOT NULL
        CHECK(asset_type IN ('EMAIL_TEMPLATE','LANDING_PAGE','IMAGE','VIDEO','PDF',
                              'SOCIAL_POST','WEB_BANNER','SMS_TEMPLATE','DOCUMENT','HTML')),
    content TEXT,                                  -- HTML/text content or URL reference
    content_json TEXT,                             -- JSON: structured content definition
    subject_line TEXT,                             -- For email templates
    preview_text TEXT,                             -- Email preview text
    from_name TEXT,
    from_email TEXT,
    url TEXT,                                      -- Landing page URL or external link
    thumbnail_url TEXT,
    file_size_bytes INTEGER,
    dimensions TEXT,                               -- JSON: width/height for images/banners
    personalization_fields TEXT,                   -- JSON array: merge fields available
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, asset_code)
);

CREATE INDEX idx_assets_tenant_type ON campaign_assets(tenant_id, asset_type);
CREATE INDEX idx_assets_tenant_status ON campaign_assets(tenant_id, status);
CREATE INDEX idx_assets_tenant_active ON campaign_assets(tenant_id, is_active);
```

### 2.5 Campaign Touches
```sql
CREATE TABLE campaign_touches (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NULL,
    campaign_id TEXT NOT NULL,
    channel_id TEXT NOT NULL,
    audience_id TEXT NOT NULL,
    asset_id TEXT,
    touch_name TEXT NOT NULL,
    touch_sequence INTEGER NOT NULL DEFAULT 1,     -- Order in nurture sequence
    touch_type TEXT NOT NULL DEFAULT 'SCHEDULED'
        CHECK(touch_type IN ('SCHEDULED','TRIGGERED','EVENT_BASED','CONDITIONAL')),
    trigger_event TEXT,                            -- For TRIGGERED touches: event that triggers
    trigger_delay_hours INTEGER NOT NULL DEFAULT 0,
    condition_expression TEXT,                     -- For CONDITIONAL touches
    scheduled_send_at TEXT,                        -- For SCHEDULED touches
    actual_sent_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','SENDING','SENT','COMPLETED','FAILED','CANCELLED')),
    total_recipients INTEGER NOT NULL DEFAULT 0,
    sent_count INTEGER NOT NULL DEFAULT 0,
    delivered_count INTEGER NOT NULL DEFAULT 0,
    opened_count INTEGER NOT NULL DEFAULT 0,
    clicked_count INTEGER NOT NULL DEFAULT 0,
    bounced_count INTEGER NOT NULL DEFAULT 0,
    unsubscribed_count INTEGER NOT NULL DEFAULT 0,
    converted_count INTEGER NOT NULL DEFAULT 0,
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (campaign_id) REFERENCES campaigns(id),
    FOREIGN KEY (channel_id) REFERENCES campaign_channels(id),
    FOREIGN KEY (audience_id) REFERENCES campaign_audiences(id),
    FOREIGN KEY (asset_id) REFERENCES campaign_assets(id)
);

CREATE INDEX idx_touches_tenant_campaign ON campaign_touches(tenant_id, campaign_id);
CREATE INDEX idx_touches_tenant_channel ON campaign_touches(tenant_id, channel_id);
CREATE INDEX idx_touches_tenant_status ON campaign_touches(tenant_id, status);
CREATE INDEX idx_touches_tenant_type ON campaign_touches(tenant_id, touch_type);
CREATE INDEX idx_touches_tenant_scheduled ON campaign_touches(tenant_id, scheduled_send_at);
CREATE INDEX idx_touches_tenant_sequence ON campaign_touches(tenant_id, touch_sequence);
```

### 2.6 Lead Campaigns
```sql
CREATE TABLE lead_campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lead_id TEXT NOT NULL,                         -- FK to SALES-AUTOMATION lead
    campaign_id TEXT NOT NULL,
    profile_id TEXT,                               -- FK to CDP profile
    first_touch_date TEXT NOT NULL,
    last_touch_date TEXT NOT NULL,
    touch_count INTEGER NOT NULL DEFAULT 0,
    lead_source TEXT NOT NULL DEFAULT 'CAMPAIGN'
        CHECK(lead_source IN ('CAMPAIGN','REFERRAL','ORGANIC','PAID_AD','EVENT','INBOUND','OUTBOUND')),
    conversion_status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(conversion_status IN ('ACTIVE','CONVERTED','LOST','DISQUALIFIED','NURTURE')),
    converted_at TEXT,
    converted_opportunity_id TEXT,                  -- FK to CRM opportunity
    attribution_weight REAL NOT NULL DEFAULT 1.0,  -- For multi-touch attribution

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (campaign_id) REFERENCES campaigns(id),
    UNIQUE(tenant_id, lead_id, campaign_id)
);

CREATE INDEX idx_lead_campaigns_tenant_lead ON lead_campaigns(tenant_id, lead_id);
CREATE INDEX idx_lead_campaigns_tenant_campaign ON lead_campaigns(tenant_id, campaign_id);
CREATE INDEX idx_lead_campaigns_tenant_status ON lead_campaigns(tenant_id, conversion_status);
CREATE INDEX idx_lead_campaigns_tenant_source ON lead_campaigns(tenant_id, lead_source);
CREATE INDEX idx_lead_campaigns_tenant_dates ON lead_campaigns(tenant_id, first_touch_date);
```

### 2.7 Campaign Responses
```sql
CREATE TABLE campaign_responses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    touch_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    lead_id TEXT,                                  -- If linked to a known lead
    response_type TEXT NOT NULL
        CHECK(response_type IN ('SENT','DELIVERED','OPENED','CLICKED','BOUNCED','UNSUBSCRIBED',
                                'REPLIED','FORM_SUBMITTED','PURCHASED','DOWNLOAD','PAGE_VIEWED',
                                'FORWARDED','SHARED','REGISTERED','ATTENDED')),
    channel_type TEXT NOT NULL,
    response_timestamp TEXT NOT NULL,               -- Actual response time
    url_clicked TEXT,
    landing_page TEXT,
    form_data TEXT,                                -- JSON: form submission data
    ip_address TEXT,
    user_agent TEXT,
    device_type TEXT,
    location TEXT,                                 -- JSON: geo data
    revenue_attributed_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (campaign_id) REFERENCES campaigns(id),
    FOREIGN KEY (touch_id) REFERENCES campaign_touches(id),
    FOREIGN KEY (profile_id) REFERENCES customer_profiles(id)
);

CREATE INDEX idx_responses_tenant_campaign ON campaign_responses(tenant_id, campaign_id);
CREATE INDEX idx_responses_tenant_touch ON campaign_responses(tenant_id, touch_id);
CREATE INDEX idx_responses_tenant_profile ON campaign_responses(tenant_id, profile_id);
CREATE INDEX idx_responses_tenant_type ON campaign_responses(tenant_id, response_type);
CREATE INDEX idx_responses_tenant_channel ON campaign_responses(tenant_id, channel_type);
CREATE INDEX idx_responses_tenant_timestamp ON campaign_responses(tenant_id, response_timestamp);
CREATE INDEX idx_responses_tenant_lead ON campaign_responses(tenant_id, lead_id);
```

### 2.8 Lead Scoring Models
```sql
CREATE TABLE lead_scoring_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_code TEXT NOT NULL,
    description TEXT,
    model_type TEXT NOT NULL DEFAULT 'RULE_BASED'
        CHECK(model_type IN ('RULE_BASED','ML_BASED','HYBRID')),
    model_version TEXT NOT NULL DEFAULT '1.0.0',
    max_score INTEGER NOT NULL DEFAULT 100,
    hot_threshold INTEGER NOT NULL DEFAULT 80,
    warm_threshold INTEGER NOT NULL DEFAULT 50,
    is_default INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),
    last_recalculated_at TEXT,
    recalculation_frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(recalculation_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY','MANUAL')),
    decay_days INTEGER NOT NULL DEFAULT 90,        -- Score decay period

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_scoring_models_tenant_status ON lead_scoring_models(tenant_id, status);
CREATE INDEX idx_scoring_models_tenant_default ON lead_scoring_models(tenant_id, is_default);
CREATE INDEX idx_scoring_models_tenant_active ON lead_scoring_models(tenant_id, is_active);
```

### 2.9 Lead Score Rules
```sql
CREATE TABLE lead_score_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('DEMOGRAPHIC','BEHAVIORAL','FIRMOGRAPHIC','ENGAGEMENT','TIMING','CUSTOM')),
    criteria TEXT NOT NULL,                        -- JSON: condition definition
    score_points INTEGER NOT NULL,                 -- Points to add/subtract
    is_negative INTEGER NOT NULL DEFAULT 0,        -- Deduct points
    max_applications INTEGER NOT NULL DEFAULT 1,   -- How many times this rule can apply
    decay_factor REAL NOT NULL DEFAULT 1.0,        -- Score decay over time (0.0-1.0)
    priority INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES lead_scoring_models(id)
);

CREATE INDEX idx_score_rules_tenant_model ON lead_score_rules(tenant_id, model_id);
CREATE INDEX idx_score_rules_tenant_type ON lead_score_rules(tenant_id, rule_type);
CREATE INDEX idx_score_rules_tenant_priority ON lead_score_rules(tenant_id, priority);
```

### 2.10 Lead Scores
```sql
CREATE TABLE lead_scores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lead_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    total_score INTEGER NOT NULL DEFAULT 0,
    score_grade TEXT NOT NULL DEFAULT 'COLD'
        CHECK(score_grade IN ('HOT','WARM','COLD','DISQUALIFIED')),
    demographic_score INTEGER NOT NULL DEFAULT 0,
    behavioral_score INTEGER NOT NULL DEFAULT 0,
    firmographic_score INTEGER NOT NULL DEFAULT 0,
    engagement_score INTEGER NOT NULL DEFAULT 0,
    score_breakdown TEXT,                          -- JSON: detailed score by rule
    last_calculated_at TEXT NOT NULL,
    last_activity_at TEXT,
    next_recalculation_at TEXT,
    urgency_points INTEGER NOT NULL DEFAULT 0,     -- Time-based urgency scoring
    priority INTEGER NOT NULL DEFAULT 0,           -- Score * Urgency composite

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES lead_scoring_models(id),
    UNIQUE(tenant_id, lead_id, model_id)
);

CREATE INDEX idx_lead_scores_tenant_lead ON lead_scores(tenant_id, lead_id);
CREATE INDEX idx_lead_scores_tenant_model ON lead_scores(tenant_id, model_id);
CREATE INDEX idx_lead_scores_tenant_grade ON lead_scores(tenant_id, score_grade);
CREATE INDEX idx_lead_scores_tenant_score ON lead_scores(tenant_id, total_score);
CREATE INDEX idx_lead_scores_tenant_priority ON lead_scores(tenant_id, priority);
CREATE INDEX idx_lead_scores_tenant_recalc ON lead_scores(tenant_id, next_recalculation_at);
```

### 2.11 Marketing Analytics
```sql
CREATE TABLE marketing_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,                      -- ISO 8601 date
    campaign_id TEXT,
    channel_type TEXT,
    metric_type TEXT NOT NULL
        CHECK(metric_type IN ('IMPRESSIONS','DELIVERED','OPENS','CLICKS','BOUNCES','UNSUBSCRIBES',
                               'CONVERSIONS','REVENUE','COST','ROI','CPA','CPL','CTR',
                               'OPEN_RATE','CLICK_TO_OPEN_RATE','FUNNEL_STAGE',
                               'ATTRIBUTION_FIRST_TOUCH','ATTRIBUTION_LAST_TOUCH',
                               'ATTRIBUTION_LINEAR','ATTRIBUTION_TIME_DECAY')),
    metric_value REAL NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    period_type TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(period_type IN ('HOURLY','DAILY','WEEKLY','MONTHLY','QUARTERLY','ANNUAL')),
    dimension_1 TEXT,                               -- Optional breakdown dimension
    dimension_2 TEXT,                               -- Optional second dimension
    metadata TEXT,                                  -- JSON: additional metric context

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, metric_date, campaign_id, channel_type, metric_type, period_type)
);

CREATE INDEX idx_analytics_tenant_date ON marketing_analytics(tenant_id, metric_date);
CREATE INDEX idx_analytics_tenant_campaign ON marketing_analytics(tenant_id, campaign_id);
CREATE INDEX idx_analytics_tenant_channel ON marketing_analytics(tenant_id, channel_type);
CREATE INDEX idx_analytics_tenant_type ON marketing_analytics(tenant_id, metric_type);
CREATE INDEX idx_analytics_tenant_period ON marketing_analytics(tenant_id, period_type);
```

---

## 3. REST API Endpoints

```
# Campaign Management
GET/POST       /api/v1/marketing/campaigns                Permission: marketing.campaigns.manage
GET/PUT        /api/v1/marketing/campaigns/{id}
PATCH          /api/v1/marketing/campaigns/{id}/status    Permission: marketing.campaigns.manage
POST           /api/v1/marketing/campaigns/{id}/launch    Permission: marketing.campaigns.launch
POST           /api/v1/marketing/campaigns/{id}/pause     Permission: marketing.campaigns.manage
POST           /api/v1/marketing/campaigns/{id}/cancel    Permission: marketing.campaigns.manage
POST           /api/v1/marketing/campaigns/{id}/clone     Permission: marketing.campaigns.manage

# Campaign Channels
GET/POST       /api/v1/marketing/campaigns/{id}/channels  Permission: marketing.campaigns.manage
GET/PUT        /api/v1/marketing/channels/{id}
DELETE         /api/v1/marketing/channels/{id}            Permission: marketing.campaigns.manage

# Campaign Audiences
GET/POST       /api/v1/marketing/campaigns/{id}/audiences Permission: marketing.campaigns.manage
GET/PUT        /api/v1/marketing/audiences/{id}
DELETE         /api/v1/marketing/audiences/{id}           Permission: marketing.campaigns.manage
POST           /api/v1/marketing/audiences/{id}/preview   Permission: marketing.campaigns.manage
POST           /api/v1/marketing/audiences/{id}/refresh   Permission: marketing.campaigns.manage

# Campaign Assets
GET/POST       /api/v1/marketing/assets                   Permission: marketing.assets.manage
GET/PUT        /api/v1/marketing/assets/{id}
DELETE         /api/v1/marketing/assets/{id}              Permission: marketing.assets.manage
POST           /api/v1/marketing/assets/{id}/preview      Permission: marketing.assets.manage

# Campaign Touches
GET/POST       /api/v1/marketing/campaigns/{id}/touches   Permission: marketing.touches.manage
GET/PUT        /api/v1/marketing/touches/{id}
DELETE         /api/v1/marketing/touches/{id}             Permission: marketing.touches.manage
POST           /api/v1/marketing/touches/{id}/send        Permission: marketing.touches.send
POST           /api/v1/marketing/touches/{id}/cancel      Permission: marketing.touches.manage

# Lead-Campaign Association
GET            /api/v1/marketing/leads/{id}/campaigns
POST           /api/v1/marketing/leads/{id}/associate     Permission: marketing.leads.manage
DELETE         /api/v1/marketing/lead-campaigns/{id}      Permission: marketing.leads.manage

# Campaign Responses
GET            /api/v1/marketing/campaigns/{id}/responses
POST           /api/v1/marketing/responses/track          Permission: marketing.responses.track
GET            /api/v1/marketing/responses
GET            /api/v1/marketing/profiles/{id}/responses

# Lead Scoring Models
GET/POST       /api/v1/marketing/scoring-models           Permission: marketing.scoring.manage
GET/PUT        /api/v1/marketing/scoring-models/{id}
DELETE         /api/v1/marketing/scoring-models/{id}      Permission: marketing.scoring.manage

# Lead Score Rules
GET/POST       /api/v1/marketing/scoring-models/{id}/rules  Permission: marketing.scoring.manage
GET/PUT        /api/v1/marketing/scoring-rules/{id}
DELETE         /api/v1/marketing/scoring-rules/{id}         Permission: marketing.scoring.manage

# Lead Scores
GET            /api/v1/marketing/leads/{id}/scores        Permission: marketing.scoring.read
POST           /api/v1/marketing/scoring-models/{id}/recalculate  Permission: marketing.scoring.recalculate
GET            /api/v1/marketing/scores/leaderboard       Permission: marketing.scoring.read
GET            /api/v1/marketing/scores/hot-leads         Permission: marketing.scoring.read

# Analytics
GET            /api/v1/marketing/analytics/overview       Permission: marketing.analytics.read
GET            /api/v1/marketing/analytics/campaign/{id}
GET            /api/v1/marketing/analytics/channel-performance
GET            /api/v1/marketing/analytics/funnel
GET            /api/v1/marketing/analytics/attribution
GET            /api/v1/marketing/analytics/roi
GET            /api/v1/marketing/analytics/engagement-trends
POST           /api/v1/marketing/analytics/refresh        Permission: marketing.analytics.manage

# Lead Handoff
POST           /api/v1/marketing/leads/{id}/handoff       Permission: marketing.leads.handoff
GET            /api/v1/marketing/leads/handoff-history     Permission: marketing.leads.read
```

---

## 4. Business Rules

### 4.1 Campaign Management
```
Campaign lifecycle rules:
  1. Campaign code MUST be unique within a tenant
  2. A campaign MUST have at least one channel and one audience before launch
  3. Status transitions: DRAFT -> SCHEDULED -> ACTIVE -> PAUSED/COMPLETED/CANCELLED
  4. Scheduled campaigns with start_date in the past MUST NOT be launched
  5. Active campaigns can be PAUSED and resumed; CANCELLED campaigns cannot be resumed
  6. Budget tracking: actual_cost_cents MUST NOT exceed budget_cents without approval
  7. Campaign cloning creates a new campaign in DRAFT status with copied configuration
  8. Multi-channel campaigns execute touches across channels in configured sequence
  9. Nurture campaigns support multi-touch sequences with conditional branching
  10. ABM (Account-Based Marketing) campaigns target specific business accounts
```

### 4.2 Audience Targeting
- SEGMENT audiences reference a CDP segment and refresh membership on evaluation
- RULE_BASED audiences evaluate profile attributes at send time
- LIST audiences use a static list of profile IDs
- Exclusion lists prevent specific profiles from receiving touches
- Suppression segments exclude profiles meeting defined criteria (e.g., unsubscribed)
- Control groups are excluded from campaign touches for A/B comparison
- Estimated size is calculated before launch; actual_size is updated post-send

### 4.3 Touch Execution
- SCHEDULED touches execute at the configured scheduled_send_at time
- TRIGGERED touches execute when the specified trigger_event occurs
- EVENT_BASED touches respond to external events (e.g., cart abandonment, form submit)
- CONDITIONAL touches evaluate condition_expression before sending
- Each touch tracks delivery metrics: sent, delivered, opened, clicked, bounced
- Bounced recipients are suppressed from future touches in the campaign
- Unsubscribed recipients are globally suppressed across all campaigns
- Touch sequence ordering is enforced for nurture campaign flows

### 4.4 Campaign Responses
- Responses are recorded for every interaction with campaign content
- Response types track the full engagement funnel: SENT -> DELIVERED -> OPENED -> CLICKED -> PURCHASED
- Revenue attribution is calculated for conversion responses
- Attribution models supported: first-touch, last-touch, linear, time-decay
- Multi-touch attribution distributes credit across all campaign touches
- Response data feeds back into lead scoring and CDP activity tracking

### 4.5 Lead Scoring
- Lead scores are calculated by evaluating all active rules in the scoring model
- Score points are additive unless is_negative = 1 (deduction)
- Score grades are assigned based on thresholds: HOT (>= hot_threshold), WARM (>= warm_threshold), COLD (< warm_threshold)
- Score decay reduces behavioral points over time based on decay_factor
- Urgency scoring increases priority for leads with recent high-value activity
- Composite priority = total_score * urgency multiplier
- Default scoring model is applied when no model is explicitly specified
- Scores are recalculated based on recalculation_frequency setting

### 4.6 Lead Handoff
- Leads meeting HOT threshold are flagged for sales handoff
- Handoff creates a lead/opportunity in SALES-AUTOMATION with full context
- Handoff includes: campaign history, engagement data, score breakdown, profile data
- Leads handed off are tracked in lead_campaigns with conversion_status = CONVERTED
- Lost or disqualified leads can be returned to nurture (status = NURTURE)
- Lead handoff requires minimum data: name, email, source campaign

### 4.7 Marketing Analytics
- Analytics are aggregated daily from response data
- ROI = (revenue_attributed - actual_cost) / actual_cost * 100
- CPA (Cost Per Acquisition) = actual_cost / conversion_count
- CPL (Cost Per Lead) = actual_cost / lead_count
- CTR (Click-Through Rate) = click_count / delivered_count * 100
- Open Rate = open_count / delivered_count * 100
- Attribution reports distribute revenue across campaign touchpoints
- Funnel metrics track progression through campaign engagement stages

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `marketing.campaign.launched` | Campaign status changed to ACTIVE | Notification, CDP |
| `marketing.campaign.completed` | Campaign end_date reached or manually completed | Reporting |
| `marketing.touch.delivered` | Touch successfully delivered to recipient | CDP |
| `marketing.touch.opened` | Touch email/page opened by recipient | CDP, Scoring |
| `marketing.touch.clicked` | Link in touch clicked by recipient | CDP, Scoring |
| `marketing.response.captured` | Any campaign response recorded | CDP, Reporting |
| `marketing.lead.score.updated` | Lead score recalculated | SALES-AUTOMATION |
| `marketing.lead.handed.off` | Lead transferred to sales | SALES-AUTOMATION, Notification |
| `marketing.lead.created` | New lead captured from campaign | CDP, SALES-AUTOMATION |
| `marketing.campaign.budget.exceeded` | Actual cost exceeds budget | Notification, Workflow |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.marketing.v1;

service MarketingService {
    rpc CreateCampaign(CreateCampaignRequest) returns (CreateCampaignResponse);
    rpc GetCampaign(GetCampaignRequest) returns (GetCampaignResponse);
    rpc ScheduleTouch(ScheduleTouchRequest) returns (ScheduleTouchResponse);
    rpc TrackResponse(TrackResponseRequest) returns (TrackResponseResponse);
    rpc CalculateLeadScore(CalculateLeadScoreRequest) returns (CalculateLeadScoreResponse);
    rpc GetCampaignAnalytics(GetCampaignAnalyticsRequest) returns (GetCampaignAnalyticsResponse);
    rpc HandoffLead(HandoffLeadRequest) returns (HandoffLeadResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **CDP (Customer Data Platform):** Unified customer profiles, segment membership, activity history, identity resolution
- **SALES-AUTOMATION (CRM):** Lead and opportunity status, sales stage updates, lead acceptance/rejection feedback
- **COMMERCE:** Web engagement events, purchase events, cart abandonment events, product interest signals
- **Auth:** User identity for campaign ownership, team assignments, approval authority
- **Document:** Asset storage for email templates, images, documents

### 6.2 Data Published To
- **SALES-AUTOMATION (CRM):** Lead creation and handoff, campaign engagement data for sales context, lead scores
- **CDP (Customer Data Platform):** Campaign activity events, response interactions, engagement scores
- **COMMERCE:** Personalization data from campaign responses, product recommendation signals
- **Notification:** Campaign launch alerts, lead handoff notifications, budget threshold warnings
- **Workflow:** Approval routing for high-budget campaigns, lead handoff approval flows
- **Reporting:** Campaign performance metrics, lead generation funnel, attribution analysis
- **GL (General Ledger):** Campaign cost journal entries, revenue attribution entries

---

## 7. Migrations

1. V001: `campaigns`
2. V002: `campaign_channels`
3. V003: `campaign_audiences`
4. V004: `campaign_assets`
5. V005: `campaign_touches`
6. V006: `lead_campaigns`
7. V007: `campaign_responses`
8. V008: `lead_scoring_models`
9. V009: `lead_score_rules`
10. V010: `lead_scores`
11. V011: `marketing_analytics`
12. V012: Triggers for `updated_at`
