# 183 - CX Advertising Service Specification

## 1. Domain Overview

CX Advertising provides customer-driven advertising and audience activation, enabling marketers to leverage first-party customer data for targeted advertising across display, search, social, video, native, and connected TV channels. Supports campaign creation with budget management, audience segmentation and lookalike expansion, creative asset management with A/B variants, and cross-platform performance measurement. Enables activation of unified customer profiles into ad platforms (Google Ads, Meta, LinkedIn, etc.) with real-time sync and conversion attribution. Integrates with Customer Data Platform for audience data, Order Management for conversion tracking, and Sales for opportunity-based targeting.

**Bounded Context:** Customer-Driven Advertising & Audience Activation
**Service Name:** `cx-advertising-service`
**Database:** `data/cx_advertising.db`
**HTTP Port:** 8201 | **gRPC Port:** 9201

---

## 2. Database Schema

### 2.1 Ad Campaigns
```sql
CREATE TABLE cxa_ad_campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_code TEXT NOT NULL,
    campaign_name TEXT NOT NULL,
    campaign_type TEXT NOT NULL CHECK(campaign_type IN ('DISPLAY','SEARCH','SOCIAL','VIDEO','NATIVE','CONNECTED_TV')),
    description TEXT,
    objective TEXT NOT NULL CHECK(objective IN ('AWARENESS','TRAFFIC','ENGAGEMENT','LEADS','CONVERSIONS','RETARGETING')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    budget_cents INTEGER NOT NULL DEFAULT 0,
    daily_budget_cents INTEGER,
    spent_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    targeting_rules TEXT,                        -- JSON: geo, device, interest targeting
    audience_id TEXT,
    creative_ids TEXT,                           -- JSON: array of creative asset IDs
    bid_strategy TEXT NOT NULL DEFAULT 'AUTO'
        CHECK(bid_strategy IN ('AUTO','MANUAL','MAXIMIZE_CLICKS','MAXIMIZE_CONVERSIONS','TARGET_CPA')),
    bid_amount_cents INTEGER,
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

CREATE INDEX idx_cxa_camp_tenant ON cxa_ad_campaigns(tenant_id, status);
CREATE INDEX idx_cxa_camp_type ON cxa_ad_campaigns(campaign_type);
CREATE INDEX idx_cxa_camp_dates ON cxa_ad_campaigns(start_date, end_date);
CREATE INDEX idx_cxa_camp_owner ON cxa_ad_campaigns(owner_id);
```

### 2.2 Audiences
```sql
CREATE TABLE cxa_audiences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    audience_code TEXT NOT NULL,
    audience_name TEXT NOT NULL,
    description TEXT,
    criteria TEXT NOT NULL,                      -- JSON: segment definition rules
    audience_size INTEGER NOT NULL DEFAULT 0,
    data_source TEXT NOT NULL,                   -- Source of audience data (CDP, upload, etc.)
    lookalike_enabled INTEGER NOT NULL DEFAULT 0,
    lookalike_seed_audience_id TEXT,
    lookalike_expansion_pct REAL,
    lookalike_size INTEGER,
    activation_channels TEXT,                    -- JSON: array of activated ad platforms
    last_synced_at TEXT,
    sync_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(sync_status IN ('PENDING','SYNCING','SYNCED','FAILED')),
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','PAUSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, audience_code)
);

CREATE INDEX idx_cxa_aud_tenant ON cxa_audiences(tenant_id, status);
CREATE INDEX idx_cxa_aud_sync ON cxa_audiences(sync_status);
CREATE INDEX idx_cxa_aud_source ON cxa_audiences(data_source);
```

### 2.3 Creative Assets
```sql
CREATE TABLE cxa_creative_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    creative_code TEXT NOT NULL,
    creative_name TEXT NOT NULL,
    creative_type TEXT NOT NULL CHECK(creative_type IN ('IMAGE','VIDEO','HTML5','CAROUSEL','NATIVE_AD','TEXT_AD')),
    format TEXT,                                 -- e.g., "300x250", "1920x1080"
    dimensions TEXT,                             -- JSON: { width, height }
    file_url TEXT NOT NULL,
    file_size_kb INTEGER,
    duration_seconds INTEGER,                    -- For video creatives
    headline TEXT,
    body_text TEXT,
    call_to_action TEXT,
    landing_page_url TEXT,
    channels TEXT NOT NULL,                      -- JSON: array of supported ad platforms
    ab_variant_group TEXT,                       -- Group ID for A/B testing
    ab_variant_label TEXT CHECK(ab_variant_label IS NULL OR ab_variant_label IN ('A','B','C')),
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','IN_REVIEW','APPROVED','REJECTED')),
    rejection_reason TEXT,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, creative_code)
);

CREATE INDEX idx_cxa_cr_tenant ON cxa_creative_assets(tenant_id, approval_status);
CREATE INDEX idx_cxa_cr_type ON cxa_creative_assets(creative_type);
CREATE INDEX idx_cxa_cr_variant ON cxa_creative_assets(ab_variant_group);
```

### 2.4 Ad Performance
```sql
CREATE TABLE cxa_ad_performance (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    creative_id TEXT,
    channel TEXT NOT NULL,
    period_date TEXT NOT NULL,
    impressions INTEGER NOT NULL DEFAULT 0,
    clicks INTEGER NOT NULL DEFAULT 0,
    conversions INTEGER NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    spend_cents INTEGER NOT NULL DEFAULT 0,
    ctr REAL NOT NULL DEFAULT 0,                -- Click-through rate
    cpc_cents INTEGER NOT NULL DEFAULT 0,       -- Cost per click
    cpa_cents INTEGER NOT NULL DEFAULT 0,       -- Cost per acquisition
    cpm_cents INTEGER NOT NULL DEFAULT 0,       -- Cost per mille
    roas REAL NOT NULL DEFAULT 0,               -- Return on ad spend
    view_rate REAL NOT NULL DEFAULT 0,          -- For video: completion rate
    engagement_rate REAL NOT NULL DEFAULT 0,
    reach INTEGER NOT NULL DEFAULT 0,
    frequency REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (campaign_id) REFERENCES cxa_ad_campaigns(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, campaign_id, creative_id, channel, period_date)
);

CREATE INDEX idx_cxa_perf_camp ON cxa_ad_performance(campaign_id, period_date DESC);
CREATE INDEX idx_cxa_perf_channel ON cxa_ad_performance(tenant_id, channel);
CREATE INDEX idx_cxa_perf_date ON cxa_ad_performance(period_date DESC);
```

### 2.5 Channel Configurations
```sql
CREATE TABLE cxa_channel_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    channel_name TEXT NOT NULL CHECK(channel_name IN ('GOOGLE_ADS','META','LINKEDIN','TWITTER','TIKTOK','SNAPCHAT','PINTEREST','PROGRAMMATIC','CONNECTED_TV')),
    display_name TEXT NOT NULL,
    account_id TEXT NOT NULL,                    -- External platform account ID
    credentials TEXT NOT NULL,                   -- JSON: encrypted OAuth tokens/keys
    api_version TEXT,
    sync_enabled INTEGER NOT NULL DEFAULT 1,
    last_sync_at TEXT,
    sync_status TEXT NOT NULL DEFAULT 'CONNECTED'
        CHECK(sync_status IN ('CONNECTED','SYNCING','DISCONNECTED','ERROR')),
    supported_campaign_types TEXT NOT NULL,      -- JSON: array of supported types
    supported_objectives TEXT,                   -- JSON: array of supported objectives
    conversion_tracking_enabled INTEGER NOT NULL DEFAULT 0,
    pixel_id TEXT,                               -- Tracking pixel/tag ID
    daily_sync_limit INTEGER NOT NULL DEFAULT 100000,
    configuration TEXT,                          -- JSON: platform-specific settings

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, channel_name, account_id)
);

CREATE INDEX idx_cxa_chan_tenant ON cxa_channel_configs(tenant_id, is_active);
CREATE INDEX idx_cxa_chan_sync ON cxa_channel_configs(sync_status);
```

---

## 3. API Endpoints

### 3.1 Campaigns
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/cx-advertising/campaigns` | Create ad campaign |
| GET | `/api/v1/cx-advertising/campaigns` | List campaigns |
| GET | `/api/v1/cx-advertising/campaigns/{id}` | Get campaign details |
| PUT | `/api/v1/cx-advertising/campaigns/{id}` | Update campaign |
| POST | `/api/v1/cx-advertising/campaigns/{id}/launch` | Launch campaign |
| POST | `/api/v1/cx-advertising/campaigns/{id}/pause` | Pause campaign |

### 3.2 Audiences
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/cx-advertising/audiences` | Create audience |
| GET | `/api/v1/cx-advertising/audiences` | List audiences |
| GET | `/api/v1/cx-advertising/audiences/{id}` | Get audience details |
| PUT | `/api/v1/cx-advertising/audiences/{id}` | Update audience |
| POST | `/api/v1/cx-advertising/audiences/{id}/sync` | Sync audience to platforms |
| POST | `/api/v1/cx-advertising/audiences/{id}/lookalike` | Generate lookalike |

### 3.3 Creatives
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/cx-advertising/creatives` | Upload creative asset |
| GET | `/api/v1/cx-advertising/creatives` | List creative assets |
| GET | `/api/v1/cx-advertising/creatives/{id}` | Get creative details |
| PUT | `/api/v1/cx-advertising/creatives/{id}` | Update creative |
| POST | `/api/v1/cx-advertising/creatives/{id}/approve` | Approve creative |
| POST | `/api/v1/cx-advertising/creatives/{id}/reject` | Reject creative |

### 3.4 Performance
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/cx-advertising/campaigns/{id}/performance` | Campaign performance |
| GET | `/api/v1/cx-advertising/performance/dashboard` | Advertising dashboard |
| GET | `/api/v1/cx-advertising/performance/channel-comparison` | Compare channels |
| GET | `/api/v1/cx-advertising/performance/roi` | ROI analysis |

### 3.5 Channel Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/cx-advertising/channels` | Configure channel |
| GET | `/api/v1/cx-advertising/channels` | List channel configurations |
| PUT | `/api/v1/cx-advertising/channels/{id}` | Update configuration |
| POST | `/api/v1/cx-advertising/channels/{id}/test` | Test channel connection |
| POST | `/api/v1/cx-advertising/channels/{id}/sync` | Trigger full sync |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `cxadv.campaign.launched` | `{ campaign_id, campaign_type, budget, channels }` | Campaign launched across channels |
| `cxadv.audience.synced` | `{ audience_id, channel, size, sync_status }` | Audience synced to ad platform |
| `cxadv.conversion.attributed` | `{ campaign_id, conversion_id, revenue, attribution_model }` | Conversion attributed to ad |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `customer.segment.changed` | Customer Data Platform | Refresh audience segments |
| `order.completed` | Order Management | Attribute conversion to campaign |
| `opportunity.closed_won` | Sales Automation | Attribute pipeline to B2B campaigns |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.cx_advertising.v1;

service CxAdvertisingService {
    rpc GetAdCampaign(GetAdCampaignRequest) returns (GetAdCampaignResponse);
    rpc CreateAdCampaign(CreateAdCampaignRequest) returns (CreateAdCampaignResponse);
    rpc GetAudience(GetAudienceRequest) returns (GetAudienceResponse);
    rpc CreateAudience(CreateAudienceRequest) returns (CreateAudienceResponse);
    rpc GetCreative(GetCreativeRequest) returns (GetCreativeResponse);
    rpc GetCampaignPerformance(GetCampaignPerformanceRequest) returns (GetCampaignPerformanceResponse);
}

// Campaign messages
message GetAdCampaignRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetAdCampaignResponse {
    CxaAdCampaign data = 1;
}

message CreateAdCampaignRequest {
    string tenant_id = 1;
    string campaign_code = 2;
    string campaign_name = 3;
    string campaign_type = 4;
    string description = 5;
    string objective = 6;
    string start_date = 7;
    string end_date = 8;
    int64 budget_cents = 9;
    int64 daily_budget_cents = 10;
    string currency_code = 11;
    string targeting_rules = 12;
    string audience_id = 13;
    string creative_ids = 14;
    string bid_strategy = 15;
    int64 bid_amount_cents = 16;
    string owner_id = 17;
}

message CreateAdCampaignResponse {
    CxaAdCampaign data = 1;
}

message CxaAdCampaign {
    string id = 1;
    string tenant_id = 2;
    string campaign_code = 3;
    string campaign_name = 4;
    string campaign_type = 5;
    string description = 6;
    string objective = 7;
    string start_date = 8;
    string end_date = 9;
    int64 budget_cents = 10;
    int64 daily_budget_cents = 11;
    int64 spent_cents = 12;
    string currency_code = 13;
    string targeting_rules = 14;
    string audience_id = 15;
    string creative_ids = 16;
    string bid_strategy = 17;
    int64 bid_amount_cents = 18;
    string owner_id = 19;
    string status = 20;
    string created_at = 21;
    string updated_at = 22;
}

// Audience messages
message GetAudienceRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetAudienceResponse {
    CxaAudience data = 1;
}

message CreateAudienceRequest {
    string tenant_id = 1;
    string audience_code = 2;
    string audience_name = 3;
    string description = 4;
    string criteria = 5;
    string data_source = 6;
    string owner_id = 7;
}

message CreateAudienceResponse {
    CxaAudience data = 1;
}

message CxaAudience {
    string id = 1;
    string tenant_id = 2;
    string audience_code = 3;
    string audience_name = 4;
    string description = 5;
    string criteria = 6;
    int32 audience_size = 7;
    string data_source = 8;
    bool lookalike_enabled = 9;
    string lookalike_seed_audience_id = 10;
    double lookalike_expansion_pct = 11;
    int32 lookalike_size = 12;
    string activation_channels = 13;
    string last_synced_at = 14;
    string sync_status = 15;
    string owner_id = 16;
    string status = 17;
    string created_at = 18;
    string updated_at = 19;
}

// Creative messages
message GetCreativeRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetCreativeResponse {
    CxaCreativeAsset data = 1;
}

message CxaCreativeAsset {
    string id = 1;
    string tenant_id = 2;
    string creative_code = 3;
    string creative_name = 4;
    string creative_type = 5;
    string format = 6;
    string dimensions = 7;
    string file_url = 8;
    int32 file_size_kb = 9;
    int32 duration_seconds = 10;
    string headline = 11;
    string body_text = 12;
    string call_to_action = 13;
    string landing_page_url = 14;
    string channels = 15;
    string ab_variant_group = 16;
    string ab_variant_label = 17;
    string approval_status = 18;
    string owner_id = 19;
    string status = 20;
    string created_at = 21;
    string updated_at = 22;
}

// Performance messages
message GetCampaignPerformanceRequest {
    string tenant_id = 1;
    string campaign_id = 2;
    string period_date = 3;
}

message GetCampaignPerformanceResponse {
    CxaAdPerformance data = 1;
}

message CxaAdPerformance {
    string id = 1;
    string tenant_id = 2;
    string campaign_id = 3;
    string creative_id = 4;
    string channel = 5;
    string period_date = 6;
    int32 impressions = 7;
    int32 clicks = 8;
    int32 conversions = 9;
    int64 revenue_cents = 10;
    int64 spend_cents = 11;
    double ctr = 12;
    int64 cpc_cents = 13;
    int64 cpa_cents = 14;
    int64 cpm_cents = 15;
    double roas = 16;
    double view_rate = 17;
    double engagement_rate = 18;
    int32 reach = 19;
    double frequency = 20;
    string created_at = 21;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | cxa_channel_configs | -- |
| V002 | cxa_audiences | -- |
| V003 | cxa_creative_assets | -- |
| V004 | cxa_ad_campaigns | V002, V003 |
| V005 | cxa_ad_performance | V004 |

---

## 7. Business Rules

1. **Budget Pacing**: Daily spend automatically paced to distribute budget evenly across campaign duration
2. **Creative Approval**: Creatives must pass platform policy review before activation
3. **Audience Sync**: Audience changes propagated to ad platforms within configurable sync window
4. **Attribution Window**: Conversions attributed within configurable lookback window (default 30 days)
5. **Frequency Capping**: Cross-channel frequency limits prevent ad fatigue
6. **Lookalike Limits**: Lookalike expansion capped at 10x seed audience size
7. **Credential Rotation**: Channel credentials rotated per platform security requirements

---

## 8. Inter-Service Integration

### 8.1 Services Consumed
| Service | Method | Purpose |
|---------|--------|---------|
| cdp-service | `GetSegment` / `GetProfile` | Audience segments and customer profiles |
| order-service | `GetOrder` | Conversion tracking and revenue attribution |
| sales-service | `GetOpportunity` | Opportunity-based B2B campaign targeting |
| content-management-service | `GetContentItem` | Creative asset storage and management |
| notification-service | `SendAlert` | Campaign alerts and budget warnings |

### 8.2 Services Provided
| Consumer | Method | Purpose |
|----------|--------|---------|
| cx-analytics-service | `GetPerformance` | Unified advertising and marketing analytics |
| eloqua-service | `GetAudience` | B2B campaign coordination and lead attribution |
| responsys-service | `GetAudience` | Cross-channel audience sharing |
