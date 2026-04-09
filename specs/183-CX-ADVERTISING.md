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

## 5. Business Rules

1. **Budget Pacing**: Daily spend automatically paced to distribute budget evenly across campaign duration
2. **Creative Approval**: Creatives must pass platform policy review before activation
3. **Audience Sync**: Audience changes propagated to ad platforms within configurable sync window
4. **Attribution Window**: Conversions attributed within configurable lookback window (default 30 days)
5. **Frequency Capping**: Cross-channel frequency limits prevent ad fatigue
6. **Lookalike Limits**: Lookalike expansion capped at 10x seed audience size
7. **Credential Rotation**: Channel credentials rotated per platform security requirements

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Customer Data Platform (60) | Audience segments and customer profiles |
| Order Management (39) | Conversion tracking and revenue attribution |
| Sales Automation (77) | Opportunity-based B2B campaign targeting |
| CX Analytics (131) | Unified advertising and marketing analytics |
| Eloqua (181) | B2B campaign coordination and lead attribution |
| Responsys (182) | Cross-channel audience sharing |
| Content Management (184) | Creative asset storage and management |
| Notification Center (165) | Campaign alerts and budget warnings |
