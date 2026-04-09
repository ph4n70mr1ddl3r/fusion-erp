# 202 - Fusion Data Intelligence Platform Service Specification

## 1. Domain Overview

Fusion Data Intelligence Platform provides a unified data platform for enterprise-wide analytics, AI/ML model lifecycle management, and advanced data product governance. It supports configurable data pipelines with source-to-destination ETL/ELT workflows and cron-based scheduling with SLA enforcement, ML model registration, training, deployment, and monitoring with drift detection and automated retraining, centralized feature stores with computation logic, freshness SLAs, and statistical profiling, curated data products with schema definitions, quality scoring, and role-based access, and AI-generated insights spanning anomaly detection, trend analysis, recommendations, and predictions with confidence scoring and actionability tracking. Enables a data mesh architecture where domain teams produce and consume well-governed data products with automated quality enforcement. Integrates with AI/ML Platform for model execution, OTBI for operational dashboards, BI Publisher for formatted reporting, and Enterprise Scheduler for pipeline orchestration across all Fusion modules.

**Bounded Context:** Unified Data Platform, AI/ML & Advanced Analytics
**Service Name:** `fusion-data-intelligence-service`
**Database:** `data/fusion_data_intelligence.db`
**HTTP Port:** 8220 | **gRPC Port:** 9220

---

## 2. Database Schema

### 2.1 Data Pipelines
```sql
CREATE TABLE fdi_data_pipelines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    pipeline_code TEXT NOT NULL,
    pipeline_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('DATABASE','API','FILE','STREAM')),
    destination_system TEXT NOT NULL,
    transformations TEXT NOT NULL,                     -- JSON: [{step, type, config}]
    schedule_cron TEXT NOT NULL,
    sla_minutes INTEGER NOT NULL DEFAULT 60,
    avg_execution_time_ms INTEGER,
    last_run_at TEXT,
    last_run_status TEXT
        CHECK(last_run_status IN ('SUCCESS','FAILURE','TIMEOUT')),
    records_processed INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('ACTIVE','PAUSED','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, pipeline_code)
);

CREATE INDEX idx_fdi_pipelines_tenant_status ON fdi_data_pipelines(tenant_id, status);
CREATE INDEX idx_fdi_pipelines_tenant_source ON fdi_data_pipelines(tenant_id, source_system);
CREATE INDEX idx_fdi_pipelines_tenant_type ON fdi_data_pipelines(tenant_id, source_type);
CREATE INDEX idx_fdi_pipelines_tenant_run_status ON fdi_data_pipelines(tenant_id, last_run_status);
CREATE INDEX idx_fdi_pipelines_tenant_active ON fdi_data_pipelines(tenant_id, is_active);
```

### 2.2 ML Models
```sql
CREATE TABLE fdi_ml_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL
        CHECK(model_type IN ('CLASSIFICATION','REGRESSION','CLUSTERING','FORECASTING','NLP')),
    framework TEXT NOT NULL,
    model_artifact_url TEXT,
    training_dataset_id TEXT,
    accuracy_metrics TEXT,                            -- JSON: [{metric, value}]
    training_completed_at TEXT,
    deployed_at TEXT,
    endpoint_url TEXT,
    inference_count INTEGER NOT NULL DEFAULT 0,
    drift_score REAL,
    retraining_schedule TEXT,
    status TEXT NOT NULL DEFAULT 'REGISTERED'
        CHECK(status IN ('REGISTERED','TRAINING','DEPLOYED','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_fdi_models_tenant_type ON fdi_ml_models(tenant_id, model_type);
CREATE INDEX idx_fdi_models_tenant_status ON fdi_ml_models(tenant_id, status);
CREATE INDEX idx_fdi_models_tenant_framework ON fdi_ml_models(tenant_id, framework);
CREATE INDEX idx_fdi_models_tenant_deployed ON fdi_ml_models(tenant_id, deployed_at);
CREATE INDEX idx_fdi_models_tenant_active ON fdi_ml_models(tenant_id, is_active);
```

### 2.3 Feature Stores
```sql
CREATE TABLE fdi_feature_stores (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    feature_group TEXT NOT NULL,
    feature_name TEXT NOT NULL,
    feature_type TEXT NOT NULL
        CHECK(feature_type IN ('NUMERIC','CATEGORICAL','TEXT','BOOLEAN')),
    computation_logic TEXT NOT NULL,
    source_pipeline_id TEXT,
    freshness_sla_minutes INTEGER NOT NULL DEFAULT 60,
    last_computed_at TEXT,
    statistics TEXT,                                   -- JSON: {mean, std_dev, min, max, null_pct}
    data_type TEXT NOT NULL,
    entity_key TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_pipeline_id) REFERENCES fdi_data_pipelines(id),
    UNIQUE(tenant_id, feature_group, feature_name)
);

CREATE INDEX idx_fdi_features_tenant_group ON fdi_feature_stores(tenant_id, feature_group);
CREATE INDEX idx_fdi_features_tenant_type ON fdi_feature_stores(tenant_id, feature_type);
CREATE INDEX idx_fdi_features_tenant_entity ON fdi_feature_stores(tenant_id, entity_key);
CREATE INDEX idx_fdi_features_tenant_status ON fdi_feature_stores(tenant_id, status);
CREATE INDEX idx_fdi_features_tenant_active ON fdi_feature_stores(tenant_id, is_active);
```

### 2.4 Data Products
```sql
CREATE TABLE fdi_data_products (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    product_code TEXT NOT NULL,
    product_name TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_definition TEXT NOT NULL,                   -- JSON: column definitions and types
    source_pipelines TEXT NOT NULL,                    -- JSON: array of pipeline IDs
    sla TEXT NOT NULL,
    access_roles TEXT NOT NULL,                        -- JSON: array of role identifiers
    usage_count INTEGER NOT NULL DEFAULT 0,
    quality_score REAL
        CHECK(quality_score IS NULL OR (quality_score >= 0 AND quality_score <= 100)),
    owner_id TEXT NOT NULL,
    published_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('ACTIVE','DEPRECATED','DRAFT')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, product_code)
);

CREATE INDEX idx_fdi_products_tenant_status ON fdi_data_products(tenant_id, status);
CREATE INDEX idx_fdi_products_tenant_owner ON fdi_data_products(tenant_id, owner_id);
CREATE INDEX idx_fdi_products_tenant_quality ON fdi_data_products(tenant_id, quality_score);
CREATE INDEX idx_fdi_products_tenant_active ON fdi_data_products(tenant_id, is_active);
```

### 2.5 AI Insights
```sql
CREATE TABLE fdi_ai_insights (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    insight_type TEXT NOT NULL
        CHECK(insight_type IN ('ANOMALY','TREND','RECOMMENDATION','PREDICTION')),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    confidence REAL NOT NULL
        CHECK(confidence >= 0 AND confidence <= 1.0),
    source_data TEXT NOT NULL,                         -- JSON: references to source datasets/models
    recommended_actions TEXT,                          -- JSON: array of action suggestions
    affected_module TEXT NOT NULL,
    affected_entity_id TEXT,
    consumer_feedback TEXT
        CHECK(consumer_feedback IS NULL OR consumer_feedback IN ('POSITIVE','NEGATIVE','NEUTRAL')),
    dismissed_flag INTEGER NOT NULL DEFAULT 0,
    generated_at TEXT NOT NULL,
    expires_at TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','ACTED_UPON')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_fdi_insights_tenant_type ON fdi_ai_insights(tenant_id, insight_type);
CREATE INDEX idx_fdi_insights_tenant_module ON fdi_ai_insights(tenant_id, affected_module);
CREATE INDEX idx_fdi_insights_tenant_status ON fdi_ai_insights(tenant_id, status);
CREATE INDEX idx_fdi_insights_tenant_confidence ON fdi_ai_insights(tenant_id, confidence);
CREATE INDEX idx_fdi_insights_tenant_expires ON fdi_ai_insights(tenant_id, expires_at);
CREATE INDEX idx_fdi_insights_tenant_active ON fdi_ai_insights(tenant_id, is_active);
```

---

## 3. REST API Endpoints

### 3.1 Pipelines
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/pipelines` | List data pipelines |
| POST | `/api/v1/fdi/pipelines` | Create data pipeline |
| GET | `/api/v1/fdi/pipelines/{id}` | Get pipeline details |
| PUT | `/api/v1/fdi/pipelines/{id}` | Update pipeline |
| POST | `/api/v1/fdi/pipelines/{id}/activate` | Activate pipeline |
| POST | `/api/v1/fdi/pipelines/{id}/pause` | Pause pipeline |
| POST | `/api/v1/fdi/pipelines/{id}/run` | Trigger manual pipeline run |
| GET | `/api/v1/fdi/pipelines/{id}/history` | Get pipeline execution history |

### 3.2 ML Models
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/models` | List ML models |
| POST | `/api/v1/fdi/models` | Register new ML model |
| GET | `/api/v1/fdi/models/{id}` | Get model details |
| PUT | `/api/v1/fdi/models/{id}` | Update model metadata |
| POST | `/api/v1/fdi/models/{id}/train` | Trigger model training |
| POST | `/api/v1/fdi/models/{id}/deploy` | Deploy model to endpoint |
| GET | `/api/v1/fdi/models/{id}/drift` | Get model drift analysis |
| POST | `/api/v1/fdi/models/{id}/deprecate` | Deprecate model |

### 3.3 Feature Stores
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/features` | List feature store entries |
| POST | `/api/v1/fdi/features` | Register new feature |
| GET | `/api/v1/fdi/features/{id}` | Get feature details |
| PUT | `/api/v1/fdi/features/{id}` | Update feature definition |
| GET | `/api/v1/fdi/features/by-group` | List features by feature group |
| POST | `/api/v1/fdi/features/{id}/compute` | Trigger feature computation |
| GET | `/api/v1/fdi/features/{id}/statistics` | Get feature statistics |

### 3.4 Data Products
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/data-products` | List data products |
| POST | `/api/v1/fdi/data-products` | Create data product |
| GET | `/api/v1/fdi/data-products/{id}` | Get data product details |
| PUT | `/api/v1/fdi/data-products/{id}` | Update data product |
| POST | `/api/v1/fdi/data-products/{id}/publish` | Publish data product |
| POST | `/api/v1/fdi/data-products/{id}/deprecate` | Deprecate data product |
| GET | `/api/v1/fdi/data-products/{id}/quality` | Get data product quality score |

### 3.5 AI Insights
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/insights` | List AI insights |
| GET | `/api/v1/fdi/insights/{id}` | Get insight details |
| POST | `/api/v1/fdi/insights/{id}/dismiss` | Dismiss an insight |
| POST | `/api/v1/fdi/insights/{id}/feedback` | Submit consumer feedback |
| POST | `/api/v1/fdi/insights/{id}/act` | Mark insight as acted upon |
| GET | `/api/v1/fdi/insights/by-module` | List insights by affected module |
| GET | `/api/v1/fdi/insights/by-type` | List insights by type |

---

## 4. Business Rules

### 4.1 Data Pipeline Management
1. pipeline_code MUST be unique within a tenant.
2. Pipeline status transitions MUST follow: DRAFT -> ACTIVE -> PAUSED -> ACTIVE; ACTIVE pipelines MUST have a valid schedule_cron.
3. sla_minutes MUST be a positive integer; pipeline runs exceeding SLA MUST trigger a TIMEOUT status.
4. transformations JSON MUST contain at least one step with type and config fields.
5. source_system and destination_system MUST reference valid registered data sources.

### 4.2 ML Model Lifecycle
6. model_code MUST be unique within a tenant.
7. Model status transitions MUST follow: REGISTERED -> TRAINING -> DEPLOYED -> DEPRECATED.
8. A model in DEPLOYED status MUST have a valid endpoint_url and deployed_at timestamp.
9. drift_score above 0.15 SHOULD trigger an automated retraining recommendation.
10. accuracy_metrics JSON MUST include at least one metric entry (e.g., accuracy, F1, RMSE, MAE).

### 4.3 Feature Store Governance
11. The combination of (tenant_id, feature_group, feature_name) MUST be unique.
12. feature_type MUST determine valid computation_logic patterns; NUMERIC features MUST produce numeric results.
13. freshness_sla_minutes MUST be a positive integer; features exceeding freshness SLA MUST be flagged.
14. statistics JSON MUST be recalculated on each feature computation cycle.
15. entity_key MUST be consistent within a feature_group for join compatibility.

### 4.4 Data Product Management
16. product_code MUST be unique within a tenant.
17. Data product status transitions MUST follow: DRAFT -> ACTIVE -> DEPRECATED.
18. quality_score MUST be between 0 and 100; products below 80 SHOULD trigger a quality review.
19. access_roles JSON MUST contain at least one role; empty access lists prevent all consumption.
20. source_pipelines JSON MUST reference only ACTIVE pipelines.

### 4.5 AI Insights
21. confidence MUST be between 0.0 and 1.0; insights below 0.5 SHOULD be flagged as low confidence.
22. Insight status transitions MUST follow: ACTIVE -> ACTED_UPON or ACTIVE -> EXPIRED.
23. Insights past expires_at timestamp MUST be automatically transitioned to EXPIRED status.
24. recommended_actions JSON SHOULD contain at least one actionable suggestion.
25. consumer_feedback MUST be one of POSITIVE, NEGATIVE, or NEUTRAL; feedback drives insight relevance scoring.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.fdi.v1;

service FusionDataIntelligenceService {
    // Pipelines
    rpc CreatePipeline(CreatePipelineRequest) returns (CreatePipelineResponse);
    rpc GetPipeline(GetPipelineRequest) returns (GetPipelineResponse);
    rpc ListPipelines(ListPipelinesRequest) returns (ListPipelinesResponse);
    rpc RunPipeline(RunPipelineRequest) returns (RunPipelineResponse);

    // ML Models
    rpc RegisterModel(RegisterModelRequest) returns (RegisterModelResponse);
    rpc GetModel(GetModelRequest) returns (GetModelResponse);
    rpc ListModels(ListModelsRequest) returns (ListModelsResponse);
    rpc DeployModel(DeployModelRequest) returns (DeployModelResponse);
    rpc GetModelDrift(GetModelDriftRequest) returns (GetModelDriftResponse);

    // Feature Stores
    rpc RegisterFeature(RegisterFeatureRequest) returns (RegisterFeatureResponse);
    rpc GetFeature(GetFeatureRequest) returns (GetFeatureResponse);
    rpc ListFeatures(ListFeaturesRequest) returns (ListFeaturesResponse);
    rpc ComputeFeature(ComputeFeatureRequest) returns (ComputeFeatureResponse);

    // Data Products
    rpc CreateDataProduct(CreateDataProductRequest) returns (CreateDataProductResponse);
    rpc GetDataProduct(GetDataProductRequest) returns (GetDataProductResponse);
    rpc ListDataProducts(ListDataProductsRequest) returns (ListDataProductsResponse);
    rpc PublishDataProduct(PublishDataProductRequest) returns (PublishDataProductResponse);

    // AI Insights
    rpc GetInsight(GetInsightRequest) returns (GetInsightResponse);
    rpc ListInsights(ListInsightsRequest) returns (ListInsightsResponse);
    rpc DismissInsight(DismissInsightRequest) returns (DismissInsightResponse);
    rpc SubmitInsightFeedback(SubmitInsightFeedbackRequest) returns (SubmitInsightFeedbackResponse);
}

message CreatePipelineRequest {
    string tenant_id = 1;
    string pipeline_code = 2;
    string pipeline_name = 3;
    string source_system = 4;
    string source_type = 5;
    string destination_system = 6;
    string transformations = 7;
    string schedule_cron = 8;
    int32 sla_minutes = 9;
}

message CreatePipelineResponse {
    string pipeline_id = 1;
    string pipeline_code = 2;
    string pipeline_name = 3;
    string status = 4;
}

message GetPipelineRequest {
    string tenant_id = 1;
    string pipeline_id = 2;
}

message GetPipelineResponse {
    string pipeline_id = 1;
    string pipeline_code = 2;
    string pipeline_name = 3;
    string source_system = 4;
    string destination_system = 5;
    string schedule_cron = 6;
    string last_run_status = 7;
    int32 records_processed = 8;
    string status = 9;
}

message ListPipelinesRequest {
    string tenant_id = 1;
    string status = 2;
    string source_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPipelinesResponse {
    repeated GetPipelineResponse pipelines = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message RunPipelineRequest {
    string tenant_id = 1;
    string pipeline_id = 2;
}

message RunPipelineResponse {
    string pipeline_id = 1;
    string run_status = 2;
    int32 records_processed = 3;
    int64 execution_time_ms = 4;
}

message RegisterModelRequest {
    string tenant_id = 1;
    string model_code = 2;
    string model_name = 3;
    string model_type = 4;
    string framework = 5;
    string model_artifact_url = 6;
    string training_dataset_id = 7;
    string retraining_schedule = 8;
}

message RegisterModelResponse {
    string model_id = 1;
    string model_code = 2;
    string model_name = 3;
    string status = 4;
}

message GetModelRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetModelResponse {
    string model_id = 1;
    string model_code = 2;
    string model_name = 3;
    string model_type = 4;
    string framework = 5;
    string accuracy_metrics = 6;
    string endpoint_url = 7;
    int32 inference_count = 8;
    double drift_score = 9;
    string status = 10;
}

message ListModelsRequest {
    string tenant_id = 1;
    string model_type = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListModelsResponse {
    repeated GetModelResponse models = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message DeployModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string endpoint_url = 3;
}

message DeployModelResponse {
    string model_id = 1;
    string endpoint_url = 2;
    string deployed_at = 3;
    string status = 4;
}

message GetModelDriftRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetModelDriftResponse {
    string model_id = 1;
    double drift_score = 2;
    bool retraining_recommended = 3;
    string drift_details = 4;
}

message RegisterFeatureRequest {
    string tenant_id = 1;
    string feature_group = 2;
    string feature_name = 3;
    string feature_type = 4;
    string computation_logic = 5;
    string source_pipeline_id = 6;
    int32 freshness_sla_minutes = 7;
    string data_type = 8;
    string entity_key = 9;
}

message RegisterFeatureResponse {
    string feature_id = 1;
    string feature_group = 2;
    string feature_name = 3;
    string status = 4;
}

message GetFeatureRequest {
    string tenant_id = 1;
    string feature_id = 2;
}

message GetFeatureResponse {
    string feature_id = 1;
    string feature_group = 2;
    string feature_name = 3;
    string feature_type = 4;
    string statistics = 5;
    string last_computed_at = 6;
    string status = 7;
}

message ListFeaturesRequest {
    string tenant_id = 1;
    string feature_group = 2;
    string feature_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListFeaturesResponse {
    repeated GetFeatureResponse features = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ComputeFeatureRequest {
    string tenant_id = 1;
    string feature_id = 2;
}

message ComputeFeatureResponse {
    string feature_id = 1;
    string computed_at = 2;
    string statistics = 3;
}

message CreateDataProductRequest {
    string tenant_id = 1;
    string product_code = 2;
    string product_name = 3;
    string description = 4;
    string schema_definition = 5;
    string source_pipelines = 6;
    string sla = 7;
    string access_roles = 8;
    string owner_id = 9;
}

message CreateDataProductResponse {
    string product_id = 1;
    string product_code = 2;
    string product_name = 3;
    string status = 4;
}

message GetDataProductRequest {
    string tenant_id = 1;
    string product_id = 2;
}

message GetDataProductResponse {
    string product_id = 1;
    string product_code = 2;
    string product_name = 3;
    string description = 4;
    string schema_definition = 5;
    int32 usage_count = 6;
    double quality_score = 7;
    string status = 8;
}

message ListDataProductsRequest {
    string tenant_id = 1;
    string status = 2;
    string owner_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListDataProductsResponse {
    repeated GetDataProductResponse products = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PublishDataProductRequest {
    string tenant_id = 1;
    string product_id = 2;
}

message PublishDataProductResponse {
    string product_id = 1;
    string published_at = 2;
    string status = 3;
}

message GetInsightRequest {
    string tenant_id = 1;
    string insight_id = 2;
}

message GetInsightResponse {
    string insight_id = 1;
    string insight_type = 2;
    string title = 3;
    string description = 4;
    double confidence = 5;
    string recommended_actions = 6;
    string affected_module = 7;
    string status = 8;
}

message ListInsightsRequest {
    string tenant_id = 1;
    string insight_type = 2;
    string affected_module = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListInsightsResponse {
    repeated GetInsightResponse insights = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message DismissInsightRequest {
    string tenant_id = 1;
    string insight_id = 2;
}

message DismissInsightResponse {
    string insight_id = 1;
    string status = 2;
}

message SubmitInsightFeedbackRequest {
    string tenant_id = 1;
    string insight_id = 2;
    string feedback = 3;
}

message SubmitInsightFeedbackResponse {
    string insight_id = 1;
    string feedback = 2;
}
```

---

## 6. Events

### Published Events
| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `fdi.pipeline.completed` | `fdi.events` | `{ pipeline_id, pipeline_code, status, records_processed, execution_time_ms }` | Data pipeline execution completed |
| `fdi.model.deployed` | `fdi.events` | `{ model_id, model_code, model_type, endpoint_url, deployed_at }` | ML model deployed to inference endpoint |
| `fdi.insight.generated` | `fdi.events` | `{ insight_id, insight_type, title, confidence, affected_module, affected_entity_id }` | New AI insight generated |
| `fdi.data_product.updated` | `fdi.events` | `{ product_id, product_code, quality_score, status }` | Data product updated or quality score changed |

### Consumed Events
| Event | Source | Description |
|-------|--------|-------------|
| `any.data.changed` | All Fusion Modules | Data change notifications from any module to trigger pipeline execution |
| `model.training.requested` | AI/ML Platform (35) | External model training requests to register and track |

---

## 7. Inter-Service Integration

### Consumed From
| Service | Data |
|---------|------|
| `aiml-service` (35) | Model training results, drift detection alerts, inference endpoint status |
| `otbi-service` (174) | Operational dashboard queries for embedded analytics and KPI extraction |
| `bipublisher-service` (150) | Report definitions and formatted output for data product enrichment |
| `scheduler-service` (173) | Cron-based pipeline scheduling, SLA monitoring, and retry orchestration |
| All Fusion Modules | Data change events (CDC) for pipeline source ingestion across all domains |

### Published To
| Service | Data |
|---------|------|
| `aiml-service` (35) | Feature vectors for model training, data quality metrics, drift monitoring data |
| `otbi-service` (174) | Curated data products for dashboard rendering and operational analytics |
| `bipublisher-service` (150) | Pre-aggregated datasets for formatted report generation |
| `scheduler-service` (173) | Pipeline execution status, SLA compliance metrics, and scheduling metadata |
| `notification-service` | Pipeline failure alerts, model drift warnings, insight notifications |

---

## 8. Migrations

1. V001: `fdi_data_pipelines`
2. V002: `fdi_ml_models`
3. V003: `fdi_feature_stores`
4. V004: `fdi_data_products`
5. V005: `fdi_ai_insights`
6. V006: Triggers for `updated_at`
