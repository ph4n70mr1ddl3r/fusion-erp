# 131 - CX Analytics Specification

## 1. Domain Overview

CX Analytics provides cross-channel customer experience analytics, combining journey analytics, sentiment analysis, customer lifetime value modeling, and predictive churn analysis. It delivers actionable insights across marketing, sales, and service touchpoints to optimize customer experience and drive retention.

**Bounded Context:** Customer Experience Analytics & Insights
**Service Name:** `cxanalytics-service`
**Database:** `data/cxanalytics.db`
**HTTP Port:** 8211 | **gRPC Port:** 9211

---

## 2. Database Schema

### 2.1 Customer Journey Maps
```sql
CREATE TABLE cx_journey_maps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    journey_name TEXT NOT NULL,
    description TEXT,
    journey_type TEXT NOT NULL CHECK(journey_type IN ('ONBOARDING','PURCHASE','SUPPORT','RENEWAL','CHURN','CUSTOM')),
    touchpoints TEXT NOT NULL,           -- JSON: [{ "step": 1, "channel": "WEB", "action": "visit", "page": "/products" }]
    milestones TEXT,                     -- JSON: key conversion milestones
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, journey_name)
);
```

### 2.2 Customer Journey Events
```sql
CREATE TABLE cx_journey_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    journey_map_id TEXT,
    session_id TEXT,
    touchpoint_channel TEXT NOT NULL CHECK(touchpoint_channel IN ('WEB','MOBILE','EMAIL','CHAT','PHONE','SOCIAL','IN_STORE','API')),
    touchpoint_action TEXT NOT NULL,     -- "page_view", "purchase", "support_call", "complaint"
    touchpoint_detail TEXT,              -- URL, subject, product ID, etc.
    timestamp TEXT NOT NULL,
    duration_seconds INTEGER,
    sentiment_score REAL CHECK(sentiment_score BETWEEN -1.0 AND 1.0),
    emotion TEXT CHECK(emotion IN ('HAPPY','FRUSTRATED','NEUTRAL','ANGRY','SATISFIED','CONFUSED')),
    conversion_event INTEGER NOT NULL DEFAULT 0,
    revenue_impact_cents INTEGER,
    source_system TEXT,                  -- "commerce", "customerservice", "marketing"
    source_event_id TEXT,
    metadata TEXT,                       -- JSON: additional context

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (journey_map_id) REFERENCES cx_journey_maps(id) ON DELETE SET NULL
);

CREATE INDEX idx_cx_journey_tenant_customer ON cx_journey_events(tenant_id, customer_id, timestamp DESC);
CREATE INDEX idx_cx_journey_tenant_channel ON cx_journey_events(tenant_id, touchpoint_channel, timestamp DESC);
CREATE INDEX idx_cx_journey_tenant_sentiment ON cx_journey_events(tenant_id, sentiment_score);
```

### 2.3 Sentiment Analysis Results
```sql
CREATE TABLE cx_sentiment_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('SURVEY','REVIEW','SOCIAL','SUPPORT_TICKET','CHAT','EMAIL','INTERVIEW')),
    source_id TEXT NOT NULL,
    customer_id TEXT,
    overall_sentiment TEXT NOT NULL CHECK(overall_sentiment IN ('POSITIVE','NEUTRAL','NEGATIVE','MIXED')),
    sentiment_score REAL NOT NULL CHECK(sentiment_score BETWEEN -1.0 AND 1.0),
    emotion TEXT,
    topics TEXT,                         -- JSON array: ["shipping", "quality", "price"]
    topic_sentiments TEXT,               -- JSON: { "shipping": -0.5, "quality": 0.8 }
    keywords TEXT,                       -- JSON array of extracted keywords
    language TEXT NOT NULL DEFAULT 'en',
    analyzed_at TEXT NOT NULL DEFAULT (datetime('now')),
    model_version TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, source_type, source_id)
);

CREATE INDEX idx_cx_sentiment_tenant_customer ON cx_sentiment_results(tenant_id, customer_id, analyzed_at DESC);
CREATE INDEX idx_cx_sentiment_tenant_source ON cx_sentiment_results(tenant_id, source_type, overall_sentiment);
```

### 2.4 Customer Lifetime Value
```sql
CREATE TABLE cx_clv_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    clv_historical_cents INTEGER NOT NULL DEFAULT 0,
    clv_predicted_cents INTEGER NOT NULL,
    clv_prediction_confidence REAL,
    average_order_value_cents INTEGER NOT NULL DEFAULT 0,
    purchase_frequency REAL NOT NULL DEFAULT 0.0,  -- Orders per month
    customer_age_days INTEGER NOT NULL DEFAULT 0,
    churn_probability REAL CHECK(churn_probability BETWEEN 0.0 AND 1.0),
    segment TEXT,                        -- "HIGH_VALUE", "GROWTH", "AT_RISK", "CHURNED"
    cohort TEXT,                         -- "2024-Q1", "2024-01"
    model_version TEXT,
    computed_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, customer_id)
);

CREATE INDEX idx_cx_clv_tenant_segment ON cx_clv_models(tenant_id, segment);
CREATE INDEX idx_cx_clv_tenant_churn ON cx_clv_models(tenant_id, churn_probability DESC);
```

### 2.5 NPS & CSAT Surveys
```sql
CREATE TABLE cx_surveys (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    survey_name TEXT NOT NULL,
    survey_type TEXT NOT NULL CHECK(survey_type IN ('NPS','CSAT','CES','CUSTOM')),
    trigger_event TEXT NOT NULL,         -- "purchase", "support_close", "onboarding_complete"
    trigger_delay_hours INTEGER NOT NULL DEFAULT 0,
    questions TEXT NOT NULL,             -- JSON: [{ "id": "q1", "type": "rating", "scale": [0, 10], "text": "..." }]
    channels TEXT NOT NULL,              -- JSON: ["EMAIL","SMS","IN_APP"]
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','PAUSED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, survey_name)
);
```

### 2.6 Survey Responses
```sql
CREATE TABLE cx_survey_responses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    survey_id TEXT NOT NULL,
    customer_id TEXT NOT NULL,
    trigger_event TEXT,
    trigger_reference_id TEXT,           -- Order ID, case ID, etc.
    responses TEXT NOT NULL,             -- JSON: { "q1": 8, "q2": "Great service", "q3": ["shipping", "price"] }
    nps_score INTEGER CHECK(nps_score BETWEEN 0 AND 10),
    csat_score REAL CHECK(csat_score BETWEEN 1.0 AND 5.0),
    overall_sentiment TEXT,
    completed_at TEXT NOT NULL DEFAULT (datetime('now')),
    response_channel TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (survey_id) REFERENCES cx_surveys(id) ON DELETE CASCADE
);

CREATE INDEX idx_cx_responses_tenant_survey ON cx_survey_responses(tenant_id, survey_id, completed_at DESC);
CREATE INDEX idx_cx_responses_tenant_customer ON cx_survey_responses(tenant_id, customer_id);
CREATE INDEX idx_cx_responses_tenant_nps ON cx_survey_responses(tenant_id, nps_score);
```

### 2.7 Analytics Dashboards
```sql
CREATE TABLE cx_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    dashboard_type TEXT NOT NULL CHECK(dashboard_type IN ('EXECUTIVE','JOURNEY','SENTIMENT','CLV','NPS','CUSTOM')),
    description TEXT,
    widgets TEXT NOT NULL,               -- JSON: widget definitions with data sources and visualizations
    filters TEXT,                        -- JSON: available filter configurations
    refresh_interval_seconds INTEGER NOT NULL DEFAULT 300,
    is_shared INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dashboard_name)
);
```

---

## 3. REST API Endpoints

### 3.1 Journey Analytics
```
GET    /api/v1/cxanalytics/journeys                      Permission: cx.journeys.read
POST   /api/v1/cxanalytics/journeys                      Permission: cx.journeys.create
GET    /api/v1/cxanalytics/journeys/{id}                 Permission: cx.journeys.read
GET    /api/v1/cxanalytics/journeys/{id}/analysis
  ?customer_id={id}&from=...&to=...                      Permission: cx.journeys.read
  Response: Journey flow analysis with drop-off rates, conversion rates, time between steps
GET    /api/v1/cxanalytics/journeys/funnel
  ?journey_type=ONBOARDING&from=...&to=...               Permission: cx.journeys.read
POST   /api/v1/cxanalytics/journeys/events               Permission: cx.journeys.ingest
```

### 3.2 Sentiment Analysis
```
GET    /api/v1/cxanalytics/sentiment
  ?source_type=SUPPORT_TICKET&from=...&to=...            Permission: cx.sentiment.read
GET    /api/v1/cxanalytics/sentiment/trends
  ?granularity=day|week|month&from=...&to=...            Permission: cx.sentiment.read
GET    /api/v1/cxanalytics/sentiment/topics              Permission: cx.sentiment.read
POST   /api/v1/cxanalytics/sentiment/analyze             Permission: cx.sentiment.analyze
  Request: { "text": "...", "source_type": "REVIEW" }
  Response: Sentiment score, topics, emotions
GET    /api/v1/cxanalytics/sentiment/word-cloud
  ?topic={topic}&sentiment=NEGATIVE                      Permission: cx.sentiment.read
```

### 3.3 Customer Lifetime Value
```
GET    /api/v1/cxanalytics/clv
  ?segment=HIGH_VALUE&sort=-clv_predicted_cents           Permission: cx.clv.read
GET    /api/v1/cxanalytics/clv/{customer_id}              Permission: cx.clv.read
POST   /api/v1/cxanalytics/clv/compute                    Permission: cx.clv.compute
GET    /api/v1/cxanalytics/clv/segments                   Permission: cx.clv.read
GET    /api/v1/cxanalytics/clv/churn-risk
  ?min_probability=0.7                                    Permission: cx.clv.read
```

### 3.4 Surveys
```
GET    /api/v1/cxanalytics/surveys                        Permission: cx.surveys.read
POST   /api/v1/cxanalytics/surveys                        Permission: cx.surveys.create
PUT    /api/v1/cxanalytics/surveys/{id}                   Permission: cx.surveys.update
GET    /api/v1/cxanalytics/surveys/{id}/results
  ?from=...&to=...                                        Permission: cx.surveys.read
GET    /api/v1/cxanalytics/surveys/{id}/nps-summary       Permission: cx.surveys.read
  Response: { "data": { "promoters": 45, "passives": 30, "detractors": 25, "nps_score": 20 } }
POST   /api/v1/cxanalytics/surveys/{id}/responses         Permission: cx.surveys.respond
```

### 3.5 Dashboards
```
GET    /api/v1/cxanalytics/dashboards                     Permission: cx.dashboards.read
POST   /api/v1/cxanalytics/dashboards                     Permission: cx.dashboards.create
GET    /api/v1/cxanalytics/dashboards/{id}                Permission: cx.dashboards.read
GET    /api/v1/cxanalytics/dashboards/{id}/data            Permission: cx.dashboards.read
  Response: Pre-computed dashboard data based on widget definitions
```

---

## 4. Business Rules

### 4.1 Journey Analytics
- Journey events ingested from all customer touchpoints in real-time
- Journey mapping supports both predefined and auto-discovered paths
- Drop-off rate calculated per step: (started - completed) / started
- Conversion attribution: last-touch, first-touch, and multi-touch models
- Journey anomaly detection alerts on unexpected path deviations

### 4.2 Sentiment Analysis
- Real-time sentiment scoring on support tickets, chats, reviews, social posts
- Topic extraction identifies key themes in customer feedback
- Sentiment trends tracked over time for early warning detection
- Multi-language support for global operations
- Sentiment score: -1.0 (very negative) to +1.0 (very positive)

### 4.3 CLV Computation
- Historical CLV: sum of all past revenue from customer
- Predicted CLV: ML model based on purchase patterns, frequency, recency
- Churn probability: percentage likelihood customer will not repurchase
- Segments: HIGH_VALUE (top 20%), GROWTH (increasing spend), AT_RISK (declining), CHURNED (no activity)
- CLV recomputed monthly or on significant events

### 4.4 NPS/CSAT Rules
- NPS: Promoters (9-10), Passives (7-8), Detractors (0-6)
- NPS Score = % Promoters - % Detractors
- CSAT: average satisfaction rating on 1-5 scale
- CES (Customer Effort Score): measures ease of interaction
- Survey throttling: max 1 survey per customer per 7 days
- Response rate tracked; low response triggers channel adjustment

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.cxanalytics.v1;

service CXAnalyticsService {
    rpc IngestJourneyEvent(IngestJourneyEventRequest) returns (IngestJourneyEventResponse);
    rpc AnalyzeSentiment(AnalyzeSentimentRequest) returns (AnalyzeSentimentResponse);
    rpc GetCustomerCLV(GetCustomerCLVRequest) returns (GetCustomerCLVResponse);
    rpc GetJourneyAnalysis(GetJourneyAnalysisRequest) returns (GetJourneyAnalysisResponse);
    rpc ComputeCLVBatch(ComputeCLVBatchRequest) returns (ComputeCLVBatchResponse);
    rpc GetChurnRisk(GetChurnRiskRequest) returns (GetChurnRiskResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **cdp-service**: Unified customer profiles, interaction history
- **commerce-service**: Purchase events for journey tracking and CLV
- **customerservice-service**: Support interaction events, CSAT data
- **b2cservice-service**: B2C interaction data
- **marketing-service**: Campaign engagement events, survey distribution
- **sales-service**: Sales pipeline data for CLV
- **ai-service**: Sentiment analysis models, churn prediction, CLV models
- **fdi-service**: Cross-application analytics data warehouse

### 6.2 Events Consumed

| Source Event | Action |
|-------------|--------|
| `commerce.order.placed` | Record journey purchase event |
| `commerce.order.cancelled` | Record journey cancellation |
| `b2c.request.created` | Record support journey event |
| `b2c.request.resolved` | Trigger CSAT survey |
| `marketing.campaign.opened` | Record marketing touchpoint |
| `marketing.email.clicked` | Record engagement event |

### 6.3 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `cx.journey.anomaly` | Unexpected journey deviation | customer_id, journey_type, deviation_detail |
| `cx.sentiment.alert` | Negative sentiment spike | source_type, sentiment_score, topics |
| `cx.churn.risk.high` | Customer churn probability exceeds threshold | customer_id, probability, clv |
| `cx.nps.updated` | NPS score recalculated | period, nps_score, response_count |
| `cx.segment.changed` | Customer CLV segment changed | customer_id, old_segment, new_segment |
| `cx.survey.response` | Survey response received | survey_id, customer_id, nps_score |

---

## 7. Migrations

### Migration Order for cxanalytics-service:
1. V001: `cx_journey_maps`
2. V002: `cx_journey_events`
3. V003: `cx_sentiment_results`
4. V004: `cx_clv_models`
5. V005: `cx_surveys`
6. V006: `cx_survey_responses`
7. V007: `cx_dashboards`
8. V008: Triggers for `updated_at`
9. V009: Seed data (default journey templates, NPS survey template, sample dashboards)
