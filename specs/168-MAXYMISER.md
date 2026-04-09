# 168 - Maxymiser (CX Testing & Personalization) Service Specification

## 1. Domain Overview

Maxymiser provides A/B testing, multivariate testing (MVT), personalization, and experience optimization for digital customer touchpoints including websites, mobile apps, and email campaigns. Supports visual test designer, audience segmentation, behavioral targeting, real-time decisioning, and statistical significance calculation. Enables marketing and CX teams to test content variations, optimize conversion funnels, personalize experiences by customer segment, and measure impact on business KPIs. Integrates with Commerce for storefront testing, Marketing for campaign optimization, CDP for audience data, and CX Analytics for results.

**Bounded Context:** CX Testing, Personalization & Experience Optimization
**Service Name:** `maxymiser-service`
**Database:** `data/maxymiser.db`
**HTTP Port:** 8186 | **gRPC Port:** 9186

---

## 2. Database Schema

### 2.1 Tests
```sql
CREATE TABLE mx_tests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    test_code TEXT NOT NULL,
    test_name TEXT NOT NULL,
    test_type TEXT NOT NULL CHECK(test_type IN ('AB','MULTIVARIATE','SPLIT_URL','PERSONALIZATION','RECOMMENDATION')),
    description TEXT,
    target_url TEXT,                        -- Page/app being tested
    channel TEXT NOT NULL CHECK(channel IN ('WEB','MOBILE_APP','EMAIL','API','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','QA','RUNNING','PAUSED','COMPLETED','ARCHIVED')),
    hypothesis TEXT,
    primary_metric TEXT NOT NULL,           -- KPI being optimized
    secondary_metrics TEXT,                 -- JSON: additional metrics tracked
    targeting_rules TEXT NOT NULL,          -- JSON: audience/segment criteria
    traffic_allocation_pct REAL NOT NULL DEFAULT 50,
    test_start_date TEXT,
    test_end_date TEXT,
    estimated_duration_days INTEGER,
    confidence_level REAL NOT NULL DEFAULT 0.95,
    minimum_sample_size INTEGER,
    total_visitors INTEGER NOT NULL DEFAULT 0,
    total_conversions INTEGER NOT NULL DEFAULT 0,
    winner_variant_id TEXT,
    lift_pct REAL NOT NULL DEFAULT 0,
    statistical_significance REAL NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    campaign_id TEXT,                       -- Link to marketing campaign

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, test_code)
);

CREATE INDEX idx_mx_test_tenant ON mx_tests(tenant_id, status);
CREATE INDEX idx_mx_test_channel ON mx_tests(tenant_id, channel, status);
```

### 2.2 Test Variants
```sql
CREATE TABLE mx_variants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    test_id TEXT NOT NULL,
    variant_code TEXT NOT NULL,             -- "CONTROL", "VARIANT_A", "VARIANT_B"
    variant_name TEXT NOT NULL,
    description TEXT,
    content_changes TEXT NOT NULL,          -- JSON: DOM changes, CSS overrides, content swaps
    redirect_url TEXT,                      -- For split URL tests
    is_control INTEGER NOT NULL DEFAULT 0,
    traffic_weight REAL NOT NULL DEFAULT 50, -- Percentage allocation
    visitors INTEGER NOT NULL DEFAULT 0,
    conversions INTEGER NOT NULL DEFAULT 0,
    conversion_rate REAL NOT NULL DEFAULT 0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    avg_order_value_cents INTEGER NOT NULL DEFAULT 0,
    bounce_rate REAL NOT NULL DEFAULT 0,
    statistical_significance REAL NOT NULL DEFAULT 0,
    lift_vs_control REAL NOT NULL DEFAULT 0,
    confidence_interval_lower REAL,
    confidence_interval_upper REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (test_id) REFERENCES mx_tests(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, test_id, variant_code)
);

CREATE INDEX idx_mx_variant_test ON mx_variants(test_id);
```

### 2.3 Personalization Rules
```sql
CREATE TABLE mx_personalization (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    description TEXT,
    target_pages TEXT NOT NULL,             -- JSON: URL patterns
    channel TEXT NOT NULL CHECK(channel IN ('WEB','MOBILE_APP','EMAIL','ALL')),
    audience_segments TEXT NOT NULL,        -- JSON: segment criteria
    trigger_conditions TEXT NOT NULL,       -- JSON: when to apply (first visit, returning, etc.)
    content_variations TEXT NOT NULL,       -- JSON: segment → content mapping
    priority INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','ARCHIVED')),
    total_impressions INTEGER NOT NULL DEFAULT 0,
    total_conversions INTEGER NOT NULL DEFAULT 0,
    conversion_rate REAL NOT NULL DEFAULT 0,
    start_date TEXT,
    end_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);

CREATE INDEX idx_mx_pers_tenant ON mx_personalization(tenant_id, status);
CREATE INDEX mx_pers_channel ON mx_personalization(channel, status);
```

### 2.4 Visitor Events
```sql
CREATE TABLE mx_visitor_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    visitor_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    test_id TEXT,
    variant_id TEXT,
    personalization_id TEXT,
    event_type TEXT NOT NULL CHECK(event_type IN (
        'IMPRESSION','CLICK','CONVERSION','PURCHASE','BOUNCE','CUSTOM_GOAL','PAGE_VIEW','FORM_SUBMIT'
    )),
    page_url TEXT NOT NULL,
    referrer_url TEXT,
    device_type TEXT CHECK(device_type IN ('DESKTOP','MOBILE','TABLET')),
    browser TEXT,
    os TEXT,
    country TEXT,
    city TEXT,
    utm_source TEXT,
    utm_campaign TEXT,
    utm_medium TEXT,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    custom_data TEXT,                       -- JSON: event-specific data
    server_time TEXT NOT NULL DEFAULT (datetime('now')),
    client_time TEXT,

    UNIQUE(id)
);

CREATE INDEX idx_mx_event_test ON mx_visitor_events(test_id, event_type, server_time DESC);
CREATE INDEX idx_mx_event_visitor ON mx_visitor_events(visitor_id, server_time DESC);
CREATE INDEX idx_mx_event_time ON mx_visitor_events(tenant_id, server_time DESC);
```

### 2.5 Audience Segments
```sql
CREATE TABLE mx_segments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    segment_code TEXT NOT NULL,
    segment_name TEXT NOT NULL,
    description TEXT,
    criteria TEXT NOT NULL,                 -- JSON: AND/OR conditions on visitor attributes/behavior
    estimated_size INTEGER NOT NULL DEFAULT 0,
    is_dynamic INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, segment_code)
);
```

---

## 3. API Endpoints

### 3.1 Tests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maxymiser/v1/tests` | Create test |
| GET | `/maxymiser/v1/tests` | List tests |
| GET | `/maxymiser/v1/tests/{id}` | Get test with results |
| PUT | `/maxymiser/v1/tests/{id}` | Update test |
| POST | `/maxymiser/v1/tests/{id}/start` | Start test |
| POST | `/maxymiser/v1/tests/{id}/pause` | Pause test |
| POST | `/maxymiser/v1/tests/{id}/complete` | Complete test |
| GET | `/maxymiser/v1/tests/{id}/results` | Get detailed results |

### 3.2 Variants
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maxymiser/v1/tests/{id}/variants` | Add variant |
| GET | `/maxymiser/v1/tests/{id}/variants` | List variants |
| PUT | `/maxymiser/v1/variants/{id}` | Update variant |

### 3.3 Personalization
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maxymiser/v1/personalization` | Create personalization rule |
| GET | `/maxymiser/v1/personalization` | List rules |
| PUT | `/maxymiser/v1/personalization/{id}` | Update rule |
| POST | `/maxymiser/v1/personalization/{id}/activate` | Activate rule |

### 3.4 Segments & Events
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maxymiser/v1/segments` | Create segment |
| GET | `/maxymiser/v1/segments` | List segments |
| POST | `/maxymiser/v1/events` | Track visitor event |
| POST | `/maxymiser/v1/events/bulk` | Bulk event tracking |
| GET | `/maxymiser/v1/events/aggregate` | Aggregated event data |

### 3.5 Decisioning API (Real-Time)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/maxymiser/v1/decide` | Get real-time variant/personalization decision |
| GET | `/maxymiser/v1/decide/visitor/{visitorId}` | Get all active decisions for visitor |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `maxy.test.started` | `{ test_id, type, variants_count }` | Test started |
| `maxy.test.completed` | `{ test_id, winner_id, lift_pct }` | Test completed with winner |
| `maxy.test.significance_reached` | `{ test_id, confidence }` | Statistical significance reached |
| `maxy.conversion.recorded` | `{ test_id, variant_id, visitor_id }` | Conversion event recorded |
| `maxy.personalization.applied` | `{ rule_id, visitor_id, segment }` | Personalization served |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `customer.profile.updated` | CDP | Update segment membership |
| `order.completed` | Order Management | Track purchase conversions |
| `cart.abandoned` | Commerce | Trigger personalization rules |
| `campaign.launched` | Marketing | Coordinate test with campaign |

---

## 5. Business Rules

1. **Sample Size**: Tests must reach minimum sample size before declaring a winner
2. **Traffic Split**: Control always receives at least equal traffic allocation
3. **Statistical Method**: Sequential testing with configurable confidence level (default 95%)
4. **Mutual Exclusion**: Visitors can only be in one test per page to avoid interference
5. **Bot Filtering**: Known bots and crawlers excluded from test data
6. **Personalization Priority**: Higher priority rules override lower when multiple match
7. **Data Retention**: Raw event data retained 90 days; aggregated results retained 2 years

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Commerce (59) | Storefront A/B testing |
| Marketing (61) | Campaign and email testing |
| Customer Data Platform (60) | Segment data for targeting |
| CX Analytics (131) | Unified analytics with test results |
| Loyalty Management (97) | Loyalty member personalization |
| Frontend (20) | JavaScript tag for event tracking |
| Channel Management (84) | Partner experience testing |
