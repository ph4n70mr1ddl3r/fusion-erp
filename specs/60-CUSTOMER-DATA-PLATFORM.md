# 60 - Customer Data Platform Service Specification

## 1. Domain Overview

Customer Data Platform (CDP) provides unified 360-degree customer profile management, cross-system identity resolution and linkage, interaction history aggregation across all touchpoints, customer segmentation with rule-based evaluation, propensity and affinity scoring, connected data source management with field mapping, and profile deduplication through configurable merge rules. Builds a single source of truth for customer data by ingesting and reconciling records from AR, CRM, marketing, commerce, and support systems. Integrates with AR for financial customer data, SALES-AUTOMATION for CRM data, and MARKETING for campaign engagement data.

**Bounded Context:** Unified Customer Profile, Identity Resolution & Segmentation
**Service Name:** `cdp-service`
**Database:** `data/cdp.db`
**HTTP Port:** 8092 | **gRPC Port:** 9092

---

## 2. Database Schema

### 2.1 Customer Profiles
```sql
CREATE TABLE customer_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_uuid TEXT NOT NULL,                    -- Universal profile identifier
    primary_email TEXT,
    primary_phone TEXT,
    first_name TEXT,
    last_name TEXT,
    company_name TEXT,
    job_title TEXT,
    primary_address TEXT,                          -- JSON: primary address
    language_code TEXT NOT NULL DEFAULT 'en',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    customer_type TEXT NOT NULL DEFAULT 'INDIVIDUAL'
        CHECK(customer_type IN ('INDIVIDUAL','BUSINESS','PARTNER','EMPLOYEE')),
    lifecycle_stage TEXT NOT NULL DEFAULT 'PROSPECT'
        CHECK(lifecycle_stage IN ('PROSPECT','LEAD','ACTIVE','LOYAL','AT_RISK','CHURNED','INACTIVE')),
    total_lifetime_value_cents INTEGER NOT NULL DEFAULT 0,
    total_orders INTEGER NOT NULL DEFAULT 0,
    first_interaction_date TEXT,
    last_interaction_date TEXT,
    account_source TEXT,                           -- Originating system
    account_ref_id TEXT,                           -- ID in originating system
    ar_customer_id TEXT,                           -- FK to AR customer
    crm_contact_id TEXT,                           -- FK to CRM contact
    tags TEXT,                                     -- JSON array: free-form tags
    consent_status TEXT NOT NULL DEFAULT 'UNKNOWN'
        CHECK(consent_status IN ('UNKNOWN','GRANTED','DENIED','WITHDRAWN','EXPIRED')),
    consent_date TEXT,
    do_not_contact INTEGER NOT NULL DEFAULT 0,
    master_profile_id TEXT,                        -- Set when merged into another profile

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, profile_uuid)
);

CREATE INDEX idx_profiles_tenant_email ON customer_profiles(tenant_id, primary_email);
CREATE INDEX idx_profiles_tenant_phone ON customer_profiles(tenant_id, primary_phone);
CREATE INDEX idx_profiles_tenant_company ON customer_profiles(tenant_id, company_name);
CREATE INDEX idx_profiles_tenant_type ON customer_profiles(tenant_id, customer_type);
CREATE INDEX idx_profiles_tenant_stage ON customer_profiles(tenant_id, lifecycle_stage);
CREATE INDEX idx_profiles_tenant_source ON customer_profiles(tenant_id, account_source);
CREATE INDEX idx_profiles_tenant_ar ON customer_profiles(tenant_id, ar_customer_id);
CREATE INDEX idx_profiles_tenant_crm ON customer_profiles(tenant_id, crm_contact_id);
CREATE INDEX idx_profiles_tenant_active ON customer_profiles(tenant_id, is_active);
CREATE INDEX idx_profiles_tenant_ltv ON customer_profiles(tenant_id, total_lifetime_value_cents);
CREATE INDEX idx_profiles_tenant_master ON customer_profiles(tenant_id, master_profile_id);
```

### 2.2 Profile Identities
```sql
CREATE TABLE profile_identities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    identity_type TEXT NOT NULL
        CHECK(identity_type IN ('EMAIL','PHONE','ACCOUNT_ID','SOCIAL_LOGIN','SSO',
                                'WEB_COOKIE','DEVICE_ID','CRM_ID','ERP_ID','COMMERCE_ID')),
    identity_value TEXT NOT NULL,                  -- The actual identifier value
    source_system TEXT NOT NULL,                   -- e.g. "ar-service", "crm-service", "commerce-service"
    source_record_id TEXT,                         -- ID in the source system
    is_verified INTEGER NOT NULL DEFAULT 0,
    verified_at TEXT,
    is_primary INTEGER NOT NULL DEFAULT 0,         -- Primary identity for this type
    confidence_score REAL NOT NULL DEFAULT 1.0,    -- Match confidence (0.0-1.0)
    last_seen_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (profile_id) REFERENCES customer_profiles(id),
    UNIQUE(tenant_id, identity_type, identity_value)
);

CREATE INDEX idx_identities_tenant_profile ON profile_identities(tenant_id, profile_id);
CREATE INDEX idx_identities_tenant_type ON profile_identities(tenant_id, identity_type);
CREATE INDEX idx_identities_tenant_source ON profile_identities(tenant_id, source_system);
CREATE INDEX idx_identities_tenant_verified ON profile_identities(tenant_id, is_verified);
CREATE INDEX idx_identities_tenant_primary ON profile_identities(tenant_id, is_primary);
CREATE INDEX idx_identities_tenant_seen ON profile_identities(tenant_id, last_seen_at);
```

### 2.3 Profile Attributes
```sql
CREATE TABLE profile_attributes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    attribute_name TEXT NOT NULL,
    attribute_value TEXT NOT NULL,
    attribute_type TEXT NOT NULL DEFAULT 'TEXT'
        CHECK(attribute_type IN ('TEXT','NUMBER','BOOLEAN','DATE','JSON')),
    source_system TEXT NOT NULL,
    source_confidence REAL NOT NULL DEFAULT 1.0,
    is_preferred INTEGER NOT NULL DEFAULT 0,       -- Preferred value when multiple sources conflict
    effective_from TEXT NOT NULL DEFAULT (datetime('now')),
    effective_to TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (profile_id) REFERENCES customer_profiles(id),
    UNIQUE(tenant_id, profile_id, attribute_name, source_system)
);

CREATE INDEX idx_profile_attrs_tenant_profile ON profile_attributes(tenant_id, profile_id);
CREATE INDEX idx_profile_attrs_tenant_name ON profile_attributes(tenant_id, attribute_name);
CREATE INDEX idx_profile_attrs_tenant_source ON profile_attributes(tenant_id, source_system);
CREATE INDEX idx_profile_attrs_tenant_preferred ON profile_attributes(tenant_id, is_preferred);
CREATE INDEX idx_profile_attrs_tenant_dates ON profile_attributes(tenant_id, effective_from, effective_to);
```

### 2.4 Profile Activities
```sql
CREATE TABLE profile_activities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    activity_type TEXT NOT NULL
        CHECK(activity_type IN ('PAGE_VIEW','PRODUCT_VIEW','ADD_TO_CART','PURCHASE','RETURN',
                                'EMAIL_OPEN','EMAIL_CLICK','FORM_SUBMIT','SUPPORT_TICKET',
                                'PHONE_CALL','MEETING','DOWNLOAD','SOCIAL_INTERACTION',
                                'REVIEW_POSTED','WISHLIST_ADD','SEARCH','LOGIN','LOGOUT')),
    source_system TEXT NOT NULL,
    source_record_id TEXT,
    activity_timestamp TEXT NOT NULL,               -- Actual event timestamp
    channel TEXT NOT NULL
        CHECK(channel IN ('WEB','MOBILE','EMAIL','PHONE','SOCIAL','IN_STORE','API','CHAT')),
    campaign_id TEXT,
    session_id TEXT,
    page_url TEXT,
    referrer_url TEXT,
    device_type TEXT
        CHECK(device_type IN ('DESKTOP','MOBILE','TABLET','UNKNOWN')),
    ip_address TEXT,
    location TEXT,                                  -- JSON: city, state, country
    metadata TEXT,                                  -- JSON: activity-specific data
    revenue_impact_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (profile_id) REFERENCES customer_profiles(id)
);

CREATE INDEX idx_activities_tenant_profile ON profile_activities(tenant_id, profile_id);
CREATE INDEX idx_activities_tenant_type ON profile_activities(tenant_id, activity_type);
CREATE INDEX idx_activities_tenant_channel ON profile_activities(tenant_id, channel);
CREATE INDEX idx_activities_tenant_timestamp ON profile_activities(tenant_id, activity_timestamp);
CREATE INDEX idx_activities_tenant_source ON profile_activities(tenant_id, source_system);
CREATE INDEX idx_activities_tenant_campaign ON profile_activities(tenant_id, campaign_id);
CREATE INDEX idx_activities_tenant_device ON profile_activities(tenant_id, device_type);
```

### 2.5 Profile Segments
```sql
CREATE TABLE profile_segments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_code TEXT NOT NULL,
    segment_name TEXT NOT NULL,
    description TEXT,
    segment_type TEXT NOT NULL DEFAULT 'DYNAMIC'
        CHECK(segment_type IN ('STATIC','DYNAMIC','COMPOSITE')),
    membership_count INTEGER NOT NULL DEFAULT 0,
    evaluation_frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(evaluation_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY','MANUAL')),
    last_evaluated_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','ARCHIVED')),
    parent_segment_id TEXT,                        -- For composite segments
    created_from_campaign TEXT,                     -- Campaign that defined this segment

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, segment_code)
);

CREATE INDEX idx_segments_tenant_type ON profile_segments(tenant_id, segment_type);
CREATE INDEX idx_segments_tenant_status ON profile_segments(tenant_id, status);
CREATE INDEX idx_segments_tenant_active ON profile_segments(tenant_id, is_active);
CREATE INDEX idx_segments_tenant_parent ON profile_segments(tenant_id, parent_segment_id);
CREATE INDEX idx_segments_tenant_eval ON profile_segments(tenant_id, evaluation_frequency);
```

### 2.6 Segment Rules
```sql
CREATE TABLE segment_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    rule_group TEXT NOT NULL DEFAULT 'DEFAULT',    -- For OR groups
    operator TEXT NOT NULL DEFAULT 'AND'
        CHECK(operator IN ('AND','OR')),
    criteria_type TEXT NOT NULL
        CHECK(criteria_type IN ('PROFILE_ATTRIBUTE','ACTIVITY_COUNT','ACTIVITY_RECENCY',
                                'LIFETIME_VALUE','ORDER_COUNT','SEGMENT_MEMBERSHIP',
                                'SCORE_THRESHOLD','IDENTITY_EXISTS','CUSTOM_EXPRESSION')),
    field_name TEXT NOT NULL,                      -- Attribute or activity field
    comparison_operator TEXT NOT NULL
        CHECK(comparison_operator IN ('EQ','NE','GT','GTE','LT','LTE','IN','NOT_IN',
                                       'CONTAINS','STARTS_WITH','ENDS_WITH','BETWEEN',
                                       'IS_NULL','IS_NOT_NULL')),
    value TEXT NOT NULL,                           -- Comparison value (JSON for lists/ranges)
    lookback_days INTEGER,                         -- For activity-based criteria
    priority INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (segment_id) REFERENCES profile_segments(id)
);

CREATE INDEX idx_segment_rules_tenant_segment ON segment_rules(tenant_id, segment_id);
CREATE INDEX idx_segment_rules_tenant_type ON segment_rules(tenant_id, criteria_type);
CREATE INDEX idx_segment_rules_tenant_group ON segment_rules(tenant_id, rule_group);
```

### 2.7 Segment Membership
```sql
CREATE TABLE segment_membership (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_id TEXT NOT NULL,
    profile_id TEXT NOT NULL,
    membership_type TEXT NOT NULL DEFAULT 'AUTO'
        CHECK(membership_type IN ('AUTO','MANUAL','IMPORTED')),
    confidence_score REAL NOT NULL DEFAULT 1.0,
    entered_at TEXT NOT NULL DEFAULT (datetime('now')),
    expires_at TEXT,
    exit_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (segment_id) REFERENCES profile_segments(id),
    FOREIGN KEY (profile_id) REFERENCES customer_profiles(id),
    UNIQUE(tenant_id, segment_id, profile_id)
);

CREATE INDEX idx_membership_tenant_segment ON segment_membership(tenant_id, segment_id);
CREATE INDEX idx_membership_tenant_profile ON segment_membership(tenant_id, profile_id);
CREATE INDEX idx_membership_tenant_type ON segment_membership(tenant_id, membership_type);
CREATE INDEX idx_membership_tenant_entered ON segment_membership(tenant_id, entered_at);
CREATE INDEX idx_membership_tenant_expires ON segment_membership(tenant_id, expires_at);
```

### 2.8 Profile Scores
```sql
CREATE TABLE profile_scores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NULL,
    profile_id TEXT NOT NULL,
    score_model TEXT NOT NULL
        CHECK(score_model IN ('PROPENSITY_PURCHASE','PROPENSITY_CHURN','AFFINITY_CATEGORY',
                               'ENGAGEMENT','LEAD_QUALITY','CREDIT_RISK','CUSTOM')),
    score_name TEXT NOT NULL,
    score_value REAL NOT NULL,                     -- Normalized 0.0 - 1.0
    score_label TEXT,                              -- e.g. "High", "Medium", "Low"
    model_version TEXT NOT NULL DEFAULT '1.0.0',
    contributing_factors TEXT,                      -- JSON: factors influencing the score
    calculated_at TEXT NOT NULL,
    expires_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (profile_id) REFERENCES customer_profiles(id),
    UNIQUE(tenant_id, profile_id, score_model, score_name)
);

CREATE INDEX idx_scores_tenant_profile ON profile_scores(tenant_id, profile_id);
CREATE INDEX idx_scores_tenant_model ON profile_scores(tenant_id, score_model);
CREATE INDEX idx_scores_tenant_value ON profile_scores(tenant_id, score_value);
CREATE INDEX idx_scores_tenant_label ON profile_scores(tenant_id, score_label);
CREATE INDEX idx_scores_tenant_calculated ON profile_scores(tenant_id, calculated_at);
```

### 2.9 Data Sources
```sql
CREATE TABLE data_sources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_code TEXT NOT NULL,
    source_name TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('ERP','CRM','COMMERCE','MARKETING','SUPPORT','ANALYTICS',
                               'SOCIAL','EXTERNAL_API','FILE_IMPORT','CUSTOM')),
    connection_config TEXT,                        -- JSON: connection parameters (encrypted sensitive fields)
    sync_frequency TEXT NOT NULL DEFAULT 'HOURLY'
        CHECK(sync_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY','MANUAL')),
    last_sync_at TEXT,
    last_sync_status TEXT NOT NULL DEFAULT 'NEVER'
        CHECK(last_sync_status IN ('NEVER','SUCCESS','PARTIAL','FAILED')),
    records_processed INTEGER NOT NULL DEFAULT 0,
    records_matched INTEGER NOT NULL DEFAULT 0,
    records_created INTEGER NOT NULL DEFAULT 0,
    records_failed INTEGER NOT NULL DEFAULT 0,
    is_readonly INTEGER NOT NULL DEFAULT 1,        -- Readonly sources cannot be written back

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, source_code)
);

CREATE INDEX idx_data_sources_tenant_type ON data_sources(tenant_id, source_type);
CREATE INDEX idx_data_sources_tenant_active ON data_sources(tenant_id, is_active);
CREATE INDEX idx_data_sources_tenant_sync ON data_sources(tenant_id, sync_frequency);
CREATE INDEX idx_data_sources_tenant_status ON data_sources(tenant_id, last_sync_status);
```

### 2.10 Data Source Mappings
```sql
CREATE TABLE data_source_mappings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NULL,
    source_id TEXT NOT NULL,
    source_field_name TEXT NOT NULL,               -- Field name in source system
    source_field_type TEXT NOT NULL DEFAULT 'TEXT'
        CHECK(source_field_type IN ('TEXT','NUMBER','DATE','BOOLEAN','JSON')),
    target_entity TEXT NOT NULL                    -- CDP target: PROFILE, IDENTITY, ATTRIBUTE, ACTIVITY
        CHECK(target_entity IN ('PROFILE','IDENTITY','ATTRIBUTE','ACTIVITY')),
    target_field_name TEXT NOT NULL,               -- CDP field name
    transformation_rule TEXT,                      -- JSON: transformation to apply
    is_required INTEGER NOT NULL DEFAULT 0,
    conflict_priority INTEGER NOT NULL DEFAULT 50, -- Higher = wins in conflicts
    default_value TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (source_id) REFERENCES data_sources(id),
    UNIQUE(tenant_id, source_id, source_field_name, target_field_name)
);

CREATE INDEX idx_source_mappings_tenant_source ON data_source_mappings(tenant_id, source_id);
CREATE INDEX idx_source_mappings_tenant_entity ON data_source_mappings(tenant_id, target_entity);
CREATE INDEX idx_source_mappings_tenant_target ON data_source_mappings(tenant_id, target_field_name);
```

### 2.11 Profile Merge Rules
```sql
CREATE TABLE profile_merge_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_code TEXT NOT NULL,
    match_criteria TEXT NOT NULL,                   -- JSON: field matching criteria
    match_type TEXT NOT NULL DEFAULT 'EXACT'
        CHECK(match_type IN ('EXACT','FUZZY','PHONETIC','REGEX')),
    fuzzy_threshold REAL NOT NULL DEFAULT 0.85,    -- For FUZZY matching (0.0-1.0)
    survivorship_rules TEXT NOT NULL,               -- JSON: which source wins per field
    auto_merge INTEGER NOT NULL DEFAULT 0,         -- Auto-merge or require review
    min_confidence_score REAL NOT NULL DEFAULT 0.9,
    is_default INTEGER NOT NULL DEFAULT 0,
    priority INTEGER NOT NULL DEFAULT 100,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_code)
);

CREATE INDEX idx_merge_rules_tenant_active ON profile_merge_rules(tenant_id, is_active);
CREATE INDEX idx_merge_rules_tenant_priority ON profile_merge_rules(tenant_id, priority);
CREATE INDEX idx_merge_rules_tenant_auto ON profile_merge_rules(tenant_id, auto_merge);
```

---

## 3. REST API Endpoints

```
# Profile Management
GET/POST       /api/v1/cdp/profiles                       Permission: cdp.profiles.manage
GET            /api/v1/cdp/profiles/{id}
PUT            /api/v1/cdp/profiles/{id}
PATCH          /api/v1/cdp/profiles/{id}/lifecycle        Permission: cdp.profiles.manage
GET            /api/v1/cdp/profiles/{id}/360              Permission: cdp.profiles.read
DELETE         /api/v1/cdp/profiles/{id}                  Permission: cdp.profiles.delete

# Profile Search
GET            /api/v1/cdp/profiles/search                Permission: cdp.profiles.read
POST           /api/v1/cdp/profiles/search/advanced       Permission: cdp.profiles.read
GET            /api/v1/cdp/profiles/search/suggest        Permission: cdp.profiles.read

# Identity Management
GET            /api/v1/cdp/profiles/{id}/identities       Permission: cdp.profiles.read
POST           /api/v1/cdp/profiles/{id}/identities       Permission: cdp.identities.manage
DELETE         /api/v1/cdp/identities/{id}                Permission: cdp.identities.manage
POST           /api/v1/cdp/identities/resolve              Permission: cdp.identities.resolve

# Profile Attributes
GET            /api/v1/cdp/profiles/{id}/attributes       Permission: cdp.profiles.read
POST           /api/v1/cdp/profiles/{id}/attributes       Permission: cdp.profiles.manage
PUT            /api/v1/cdp/profiles/{id}/attributes/{name}  Permission: cdp.profiles.manage
DELETE         /api/v1/cdp/attributes/{id}                Permission: cdp.profiles.manage

# Activity History
GET            /api/v1/cdp/profiles/{id}/activities       Permission: cdp.profiles.read
POST           /api/v1/cdp/activities/ingest              Permission: cdp.activities.ingest
GET            /api/v1/cdp/activities/timeline             Permission: cdp.profiles.read

# Profile Merging
POST           /api/v1/cdp/profiles/merge                 Permission: cdp.merge
POST           /api/v1/cdp/profiles/merge-preview         Permission: cdp.merge
POST           /api/v1/cdp/profiles/unmerge/{id}          Permission: cdp.merge

# Segment Management
GET/POST       /api/v1/cdp/segments                       Permission: cdp.segments.manage
GET/PUT        /api/v1/cdp/segments/{id}
DELETE         /api/v1/cdp/segments/{id}                  Permission: cdp.segments.manage

# Segment Rules
GET/POST       /api/v1/cdp/segments/{id}/rules            Permission: cdp.segments.manage
PUT            /api/v1/cdp/segment-rules/{id}
DELETE         /api/v1/cdp/segment-rules/{id}             Permission: cdp.segments.manage

# Segment Evaluation
POST           /api/v1/cdp/segments/{id}/evaluate         Permission: cdp.segments.evaluate
GET            /api/v1/cdp/segments/{id}/members          Permission: cdp.segments.read
POST           /api/v1/cdp/segments/evaluate-all          Permission: cdp.segments.evaluate
GET            /api/v1/cdp/profiles/{id}/segments         Permission: cdp.segments.read

# Scoring
GET            /api/v1/cdp/profiles/{id}/scores           Permission: cdp.scores.read
POST           /api/v1/cdp/scores/calculate/{model}       Permission: cdp.scores.calculate
GET            /api/v1/cdp/scores/models                   Permission: cdp.scores.read

# Data Sources
GET/POST       /api/v1/cdp/sources                        Permission: cdp.sources.manage
GET/PUT        /api/v1/cdp/sources/{id}
POST           /api/v1/cdp/sources/{id}/sync              Permission: cdp.sources.sync
GET            /api/v1/cdp/sources/{id}/status

# Data Source Mappings
GET/POST       /api/v1/cdp/sources/{id}/mappings          Permission: cdp.sources.manage
PUT            /api/v1/cdp/source-mappings/{id}
DELETE         /api/v1/cdp/source-mappings/{id}           Permission: cdp.sources.manage

# Merge Rules
GET/POST       /api/v1/cdp/merge-rules                    Permission: cdp.merge.manage
GET/PUT        /api/v1/cdp/merge-rules/{id}
DELETE         /api/v1/cdp/merge-rules/{id}               Permission: cdp.merge.manage
POST           /api/v1/cdp/merge-rules/detect-duplicates   Permission: cdp.merge

# Analytics
GET            /api/v1/cdp/analytics/segment-size
GET            /api/v1/cdp/analytics/profile-growth
GET            /api/v1/cdp/analytics/source-coverage
GET            /api/v1/cdp/analytics/identity-coverage
GET            /api/v1/cdp/analytics/engagement-distribution
```

---

## 4. Business Rules

### 4.1 Profile Management
```
Unified profile rules:
  1. Each customer has exactly one active unified profile (master_profile_id IS NULL)
  2. profile_uuid is immutable and globally unique within a tenant
  3. A profile MUST have at least one linked identity
  4. primary_email and primary_phone are denormalized from identities for fast lookup
  5. lifecycle_stage transitions follow a defined path: PROSPECT -> LEAD -> ACTIVE -> LOYAL -> AT_RISK -> CHURNED
  6. total_lifetime_value_cents is aggregated from linked order and payment data
  7. Profiles with do_not_contact = 1 MUST NOT be included in marketing campaigns
  8. consent_status MUST be GRANTED before adding profile to marketing segments
  9. Profiles can be soft-deleted (is_active = 0) but not physically deleted if they have activities
```

### 4.2 Identity Resolution
- Each identity (type + value combination) maps to exactly one active profile
- Identity types define different identifier namespaces (email, phone, account ID, etc.)
- When a new identity matches an existing identity, profiles are linked or merged
- confidence_score reflects the certainty of the identity-to-profile mapping
- is_primary flag indicates the preferred identity for each type
- Identity resolution runs on every new identity ingestion
- Duplicate identities (same type + value across profiles) trigger merge review

### 4.3 Activity Tracking
- Activities are append-only and immutable once recorded
- Each activity references exactly one profile
- activity_timestamp reflects the actual event time, not ingestion time
- Revenue-impacting activities (PURCHASE, RETURN) update profile lifetime value
- Activities from different source_systems are reconciled by source_record_id
- Activities older than 2 years are archived to cold storage
- Channel and device_type enable cross-channel journey analysis

### 4.4 Segmentation
- Dynamic segments are re-evaluated based on evaluation_frequency
- Static segments have members added/removed manually
- Composite segments combine multiple segments using AND/OR logic
- Segment rules within a rule_group are ANDed; rule_groups are ORed
- Criteria types support: attribute matching, activity counts, recency, value thresholds
- lookback_days limits activity-based criteria to a time window
- Segment membership history is tracked for exit analysis

### 4.5 Profile Scoring
- Score values are normalized to 0.0 - 1.0 range
- Score labels (High/Medium/Low) are derived from configurable thresholds
- Contributing factors explain what drove the score value
- Scores expire and must be recalculated periodically
- Multiple score models can be active simultaneously per profile
- Score calculations are asynchronous and batched for performance

### 4.6 Data Source Integration
- Data sources define external systems that feed data into the CDP
- Field mappings transform source data to CDP data model
- Conflict priority determines which source wins when values conflict
- Sync status tracks success/failure of each data ingestion
- Readonly sources cannot receive data push-back from the CDP
- Real-time sync uses event streaming; batch sync uses scheduled jobs

### 4.7 Profile Merging
- Merge rules define criteria for detecting duplicate profiles
- Matching can be exact, fuzzy, phonetic, or regex-based
- Fuzzy matching requires confidence_score above fuzzy_threshold
- Auto-merge rules merge duplicates without human review when confidence is above min_confidence_score
- Manual merge requires user confirmation before executing
- Survivorship rules determine which source profile's data is retained
- Merged profiles have master_profile_id set to the surviving profile
- Unmerge reverses a merge, restoring the subsumed profile

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `cdp.profile.created` | New unified profile created | Marketing, Notification |
| `cdp.profile.updated` | Profile attributes changed | All consumers |
| `cdp.profile.merged` | Profiles merged into one | All consumers |
| `cdp.profile.unmerged` | Merged profile split back | All consumers |
| `cdp.identity.linked` | New identity linked to profile | Reporting |
| `cdp.activity.ingested` | New activity recorded | Scoring, Segmentation |
| `cdp.segment.membership.added` | Profile added to segment | Marketing |
| `cdp.segment.membership.removed` | Profile removed from segment | Marketing |
| `cdp.score.calculated` | Profile score computed | Marketing, Sales |
| `cdp.source.sync.completed` | Data source sync finished | Notification |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.cdp.v1;

service CDPService {
    rpc GetProfile(GetProfileRequest) returns (GetProfileResponse);
    rpc GetProfile360(GetProfile360Request) returns (GetProfile360Response);
    rpc SearchProfiles(SearchProfilesRequest) returns (SearchProfilesResponse);
    rpc MergeProfiles(MergeProfilesRequest) returns (MergeProfilesResponse);
    rpc IngestActivity(IngestActivityRequest) returns (IngestActivityResponse);
    rpc EvaluateSegment(EvaluateSegmentRequest) returns (EvaluateSegmentResponse);
    rpc GetSegmentMembers(GetSegmentMembersRequest) returns (GetSegmentMembersResponse);
    rpc ResolveIdentity(ResolveIdentityRequest) returns (ResolveIdentityResponse);
}

// Entity messages
message CustomerProfile {
    string id = 1;
    string tenant_id = 2;
    string profile_uuid = 3;
    string primary_email = 4;
    string primary_phone = 5;
    string first_name = 6;
    string last_name = 7;
    string company_name = 8;
    string job_title = 9;
    string primary_address = 10;
    string language_code = 11;
    string timezone = 12;
    string customer_type = 13;
    string lifecycle_stage = 14;
    int64 total_lifetime_value_cents = 15;
    int32 total_orders = 16;
    string first_interaction_date = 17;
    string last_interaction_date = 18;
    string account_source = 19;
    string account_ref_id = 20;
    string ar_customer_id = 21;
    string crm_contact_id = 22;
    string tags = 23;
    string consent_status = 24;
    string consent_date = 25;
    int32 do_not_contact = 26;
    string master_profile_id = 27;
    string created_at = 28;
    string updated_at = 29;
}

message ProfileIdentity {
    string id = 1;
    string tenant_id = 2;
    string profile_id = 3;
    string identity_type = 4;
    string identity_value = 5;
    string source_system = 6;
    string source_record_id = 7;
    int32 is_verified = 8;
    string verified_at = 9;
    int32 is_primary = 10;
    double confidence_score = 11;
    string last_seen_at = 12;
    string created_at = 13;
    string updated_at = 14;
}

message ProfileActivity {
    string id = 1;
    string tenant_id = 2;
    string profile_id = 3;
    string activity_type = 4;
    string source_system = 5;
    string source_record_id = 6;
    string activity_timestamp = 7;
    string details = 8;
    double value = 9;
    string currency_code = 10;
    string created_at = 11;
}

message ProfileSegment {
    string id = 1;
    string tenant_id = 2;
    string segment_name = 3;
    string segment_code = 4;
    string description = 5;
    string segment_type = 6;
    string rules = 7;
    int32 estimated_size = 8;
    int32 actual_size = 9;
    string status = 10;
    string created_at = 11;
    string updated_at = 12;
}

message ProfileScore {
    string id = 1;
    string tenant_id = 2;
    string profile_id = 3;
    string score_model_id = 4;
    double score_value = 5;
    string score_level = 6;
    string scored_at = 7;
    string created_at = 8;
    string updated_at = 9;
}

// Request/Response messages
message GetProfileRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetProfileResponse {
    CustomerProfile data = 1;
    repeated ProfileIdentity identities = 2;
}

message GetProfile360Request {
    string tenant_id = 1;
    string id = 2;
}

message GetProfile360Response {
    CustomerProfile profile = 1;
    repeated ProfileIdentity identities = 2;
    repeated ProfileActivity recent_activities = 3;
    repeated ProfileScore scores = 4;
    repeated string segment_ids = 5;
}

message SearchProfilesRequest {
    string tenant_id = 1;
    string query = 2;
    string customer_type = 3;
    string lifecycle_stage = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message SearchProfilesResponse {
    repeated CustomerProfile items = 1;
    int32 total_count = 2;
    string next_page_token = 3;
}

message MergeProfilesRequest {
    string tenant_id = 1;
    string survivor_profile_id = 2;
    string subsumed_profile_id = 3;
    string merge_reason = 4;
}

message MergeProfilesResponse {
    CustomerProfile merged_profile = 1;
    repeated ProfileIdentity merged_identities = 2;
}

message IngestActivityRequest {
    string tenant_id = 1;
    string profile_id = 2;
    string activity_type = 3;
    string source_system = 4;
    string source_record_id = 5;
    string activity_timestamp = 6;
    string details = 7;
}

message IngestActivityResponse {
    ProfileActivity data = 1;
}

message EvaluateSegmentRequest {
    string tenant_id = 1;
    string segment_id = 2;
}

message EvaluateSegmentResponse {
    string segment_id = 1;
    int32 total_matching = 2;
    int32 added = 3;
    int32 removed = 4;
}

message GetSegmentMembersRequest {
    string tenant_id = 1;
    string segment_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetSegmentMembersResponse {
    repeated CustomerProfile items = 1;
    int32 total_count = 2;
    string next_page_token = 3;
}

message ResolveIdentityRequest {
    string tenant_id = 1;
    string identity_type = 2;
    string identity_value = 3;
}

message ResolveIdentityResponse {
    string profile_id = 1;
    double confidence_score = 2;
    repeated ProfileIdentity matched_identities = 3;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **AR (Accounts Receivable):** Customer master data, invoice and payment history, credit status
- **SALES-AUTOMATION (CRM):** Contact and account data, opportunity history, lead source
- **MARKETING:** Campaign engagement data, email interaction events, lead scores
- **COMMERCE:** Browse behavior, cart activity, purchase history, product preferences
- **Auth:** User account data, login sessions, device information
- **INV (Inventory):** Product catalog data for activity enrichment

### 6.2 Data Published To
- **MARKETING:** Unified profiles for campaign targeting, segment membership lists, engagement scores
- **SALES-AUTOMATION (CRM):** Enriched contact data, customer insights, propensity scores
- **COMMERCE:** Customer preferences, personalization data, recommendation inputs
- **Notification:** Profile milestone alerts, segment membership changes, merge notifications
- **Workflow:** Merge review workflows, approval routing for manual merges
- **Reporting:** Profile growth analytics, segment distribution, source coverage reports
- **AI/ML Platform:** Training data for scoring models, segmentation model inputs

---

## 7. Migrations

1. V001: `customer_profiles`
2. V002: `profile_identities`
3. V003: `profile_attributes`
4. V004: `profile_activities`
5. V005: `profile_segments`
6. V006: `segment_rules`
7. V007: `segment_membership`
8. V008: `profile_scores`
9. V009: `data_sources`
10. V010: `data_source_mappings`
11. V011: `profile_merge_rules`
12. V012: Triggers for `updated_at`
