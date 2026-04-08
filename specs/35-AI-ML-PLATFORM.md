# 35 - AI/ML Platform & AI Agents Service Specification

## 1. Domain Overview

The AI/ML Platform provides artificial intelligence, machine learning, and intelligent automation capabilities across the entire ERP suite. It exposes pre-built AI agents for document recognition, anomaly detection, financial forecasting, policy advisory, comparison analysis, classification, and natural language querying. The platform includes a model registry, training pipeline, inference engine, and audit logging. All AI features are multi-tenant isolated and integrate with every ERP module for predictive and prescriptive analytics.

**Bounded Context:** Artificial Intelligence, Machine Learning & Intelligent Automation
**Service Name:** `ai-service`
**Database:** `data/ai.db`
**HTTP Port:** 8062 | **gRPC Port:** 9062

---

## 2. Database Schema

### 2.1 AI Agent Definitions
```sql
CREATE TABLE ai_agent_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agent_name TEXT NOT NULL,
    agent_type TEXT NOT NULL
        CHECK(agent_type IN ('DOCUMENT_RECOGNITION','ANOMALY_DETECTION','FORECAST','POLICY_ADVISOR','COMPARISON','CLASSIFICATION','NLP_QUERY')),
    description TEXT,
    model_config TEXT NOT NULL,               -- JSON: model parameters, provider, endpoint
    input_schema TEXT,                        -- JSON: expected input format
    output_schema TEXT,                       -- JSON: expected output format
    is_active INTEGER NOT NULL DEFAULT 1,
    version INTEGER NOT NULL DEFAULT 1,
    created_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, agent_name, version)
);

CREATE INDEX idx_ai_agents_tenant ON ai_agent_definitions(tenant_id);
CREATE INDEX idx_ai_agents_type ON ai_agent_definitions(tenant_id, agent_type);
```

### 2.2 AI Model Registry
```sql
CREATE TABLE ai_model_registry (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL
        CHECK(model_type IN ('CLASSIFICATION','REGRESSION','CLUSTERING','NLP','VISION','FORECASTING')),
    framework TEXT NOT NULL DEFAULT 'ONNX'
        CHECK(framework IN ('ONNX','CUSTOM')),
    version TEXT NOT NULL,
    storage_path TEXT NOT NULL,
    accuracy_metrics TEXT,                    -- JSON: { "accuracy": 0.95, "f1": 0.93, "auc": 0.97 }
    training_date TEXT,
    status TEXT NOT NULL DEFAULT 'TRAINING'
        CHECK(status IN ('TRAINING','VALIDATING','DEPLOYED','DEPRECATED')),
    hyperparameters TEXT,                     -- JSON: training configuration

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, model_name, version)
);

CREATE INDEX idx_ai_models_tenant ON ai_model_registry(tenant_id);
CREATE INDEX idx_ai_models_status ON ai_model_registry(tenant_id, status);
```

### 2.3 AI Training Datasets
```sql
CREATE TABLE ai_training_datasets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dataset_name TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('QUERY','UPLOAD','SYNTHETIC','PIPELINE')),
    source_config TEXT NOT NULL,              -- JSON: connection/query/upload details
    record_count INTEGER NOT NULL DEFAULT 0,
    feature_columns TEXT,                     -- JSON array: ["col1","col2",...]
    target_column TEXT,
    created_by TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PREPARING'
        CHECK(status IN ('PREPARING','READY','TRAINING','COMPLETED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, dataset_name)
);

CREATE INDEX idx_ai_datasets_tenant ON ai_training_datasets(tenant_id);
CREATE INDEX idx_ai_datasets_status ON ai_training_datasets(tenant_id, status);
```

### 2.4 AI Training Jobs
```sql
CREATE TABLE ai_training_jobs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    dataset_id TEXT NOT NULL,
    job_type TEXT NOT NULL
        CHECK(job_type IN ('TRAIN','FINE_TUNE','EVALUATE')),
    status TEXT NOT NULL DEFAULT 'QUEUED'
        CHECK(status IN ('QUEUED','RUNNING','COMPLETED','FAILED')),
    started_at TEXT,
    completed_at TEXT,
    metrics TEXT,                             -- JSON: training metrics, loss curves, etc.
    error_message TEXT,
    triggered_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (model_id) REFERENCES ai_model_registry(id),
    FOREIGN KEY (dataset_id) REFERENCES ai_training_datasets(id)
);

CREATE INDEX idx_ai_jobs_tenant ON ai_training_jobs(tenant_id);
CREATE INDEX idx_ai_jobs_status ON ai_training_jobs(tenant_id, status);
CREATE INDEX idx_ai_jobs_model ON ai_training_jobs(tenant_id, model_id);
```

### 2.5 AI Inference Requests
```sql
CREATE TABLE ai_inference_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agent_id TEXT,
    model_id TEXT,
    request_type TEXT NOT NULL,
    input_data TEXT NOT NULL,                 -- JSON: request payload
    output_data TEXT,                         -- JSON: response payload
    confidence_score REAL,
    execution_time_ms INTEGER,
    status TEXT NOT NULL DEFAULT 'SUCCESS'
        CHECK(status IN ('SUCCESS','ERROR','TIMEOUT')),
    source_service TEXT NOT NULL,             -- calling service identifier
    correlation_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (agent_id) REFERENCES ai_agent_definitions(id),
    FOREIGN KEY (model_id) REFERENCES ai_model_registry(id)
);

CREATE INDEX idx_ai_infer_tenant ON ai_inference_requests(tenant_id);
CREATE INDEX idx_ai_infer_agent ON ai_inference_requests(tenant_id, agent_id);
CREATE INDEX idx_ai_infer_correlation ON ai_inference_requests(correlation_id);
CREATE INDEX idx_ai_infer_status ON ai_inference_requests(tenant_id, status);
```

### 2.6 AI Document Recognition
```sql
CREATE TABLE ai_document_recognition (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    document_type TEXT NOT NULL
        CHECK(document_type IN ('INVOICE','PO','RECEIPT','CONTRACT','OTHER')),
    extracted_fields TEXT NOT NULL,           -- JSON: { "vendor": "...", "total": 123.45, ... }
    confidence_score REAL NOT NULL,
    requires_review INTEGER NOT NULL DEFAULT 0,
    reviewed_by TEXT,
    corrections TEXT,                         -- JSON: reviewer corrections
    ocr_engine TEXT NOT NULL DEFAULT 'tesseract',
    processing_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (document_id) REFERENCES documents(id)
);

CREATE INDEX idx_ai_docrec_tenant ON ai_document_recognition(tenant_id);
CREATE INDEX idx_ai_docrec_doc ON ai_document_recognition(document_id);
CREATE INDEX idx_ai_docrec_review ON ai_document_recognition(tenant_id, requires_review);
```

### 2.7 AI Anomaly Detections
```sql
CREATE TABLE ai_anomaly_detections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('JOURNAL','INVOICE','PAYMENT','EXPENSE','ORDER')),
    entity_id TEXT NOT NULL,
    anomaly_type TEXT NOT NULL
        CHECK(anomaly_type IN ('AMOUNT','PATTERN','FREQUENCY','BEHAVIORAL')),
    severity TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    anomaly_score REAL NOT NULL,
    description TEXT NOT NULL,
    suggested_action TEXT,
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','ACKNOWLEDGED','INVESTIGATING','RESOLVED','FALSE_POSITIVE')),
    investigated_by TEXT,
    resolution TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_by TEXT NOT NULL,

    FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

CREATE INDEX idx_ai_anomaly_tenant ON ai_anomaly_detections(tenant_id);
CREATE INDEX idx_ai_anomaly_entity ON ai_anomaly_detections(tenant_id, entity_type, entity_id);
CREATE INDEX idx_ai_anomaly_status ON ai_anomaly_detections(tenant_id, status);
CREATE INDEX idx_ai_anomaly_severity ON ai_anomaly_detections(tenant_id, severity);
```

### 2.8 AI Forecast Models
```sql
CREATE TABLE ai_forecast_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    forecast_type TEXT NOT NULL
        CHECK(forecast_type IN ('DEMAND','REVENUE','CASH_FLOW','EXPENSE','INVENTORY')),
    model_method TEXT NOT NULL
        CHECK(model_method IN ('ARIMA','EXPONENTIAL_SMOOTHING','PROPHET','MLP','ENSEMBLE')),
    time_grain TEXT NOT NULL DEFAULT 'MONTH'
        CHECK(time_grain IN ('DAY','WEEK','MONTH')),
    horizon_periods INTEGER NOT NULL DEFAULT 12,
    confidence_level REAL NOT NULL DEFAULT 0.95,
    seasonality_enabled INTEGER NOT NULL DEFAULT 1,
    last_trained_at TEXT,
    accuracy_mape REAL,
    accuracy_rmse REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, forecast_type, model_method, version)
);

CREATE INDEX idx_ai_fcast_tenant ON ai_forecast_models(tenant_id);
CREATE INDEX idx_ai_fcast_type ON ai_forecast_models(tenant_id, forecast_type);
```

### 2.9 AI Forecast Results
```sql
CREATE TABLE ai_forecast_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    forecast_model_id TEXT NOT NULL,
    period_date TEXT NOT NULL,
    predicted_value_cents INTEGER NOT NULL,
    lower_bound_cents INTEGER NOT NULL,
    upper_bound_cents INTEGER NOT NULL,
    actual_value_cents INTEGER,
    variance_percent REAL,
    is_outlier INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (forecast_model_id) REFERENCES ai_forecast_models(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, forecast_model_id, period_date)
);

CREATE INDEX idx_ai_fresult_tenant ON ai_forecast_results(tenant_id);
CREATE INDEX idx_ai_fresult_model ON ai_forecast_results(forecast_model_id);
CREATE INDEX idx_ai_fresult_date ON ai_forecast_results(tenant_id, period_date);
```

### 2.10 AI NLP Queries
```sql
CREATE TABLE ai_nlp_queries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    query_text TEXT NOT NULL,
    interpreted_intent TEXT,                  -- JSON: detected intent and entities
    generated_sql TEXT,                       -- sanitized SELECT-only SQL
    result_summary TEXT,
    result_data TEXT,                         -- JSON: query results
    feedback_rating INTEGER
        CHECK(feedback_rating BETWEEN 1 AND 5),
    feedback_text TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_ai_nlp_tenant ON ai_nlp_queries(tenant_id);
CREATE INDEX idx_ai_nlp_user ON ai_nlp_queries(tenant_id, user_id);
```

---

## 3. REST API Endpoints

```
# Agent Management
GET/POST      /api/v1/ai/agents                        Permission: ai.agents.read/create
GET/PUT       /api/v1/ai/agents/{id}                   Permission: ai.agents.read/update
POST          /api/v1/ai/agents/{id}/execute           Permission: ai.agents.execute
GET           /api/v1/ai/agents/{id}/history           Permission: ai.agents.read

# Model Registry
GET/POST      /api/v1/ai/models                        Permission: ai.models.read/create
GET/PUT       /api/v1/ai/models/{id}                   Permission: ai.models.read/update
POST          /api/v1/ai/models/{id}/deploy            Permission: ai.models.deploy
POST          /api/v1/ai/models/{id}/retire            Permission: ai.models.deploy

# Training
GET/POST      /api/v1/ai/datasets                      Permission: ai.datasets.read/create
GET/POST      /api/v1/ai/training/jobs                 Permission: ai.training.read/create
GET           /api/v1/ai/training/jobs/{id}            Permission: ai.training.read
POST          /api/v1/ai/training/jobs/{id}/cancel     Permission: ai.training.update

# Inference
POST          /api/v1/ai/infer                         Permission: ai.infer
POST          /api/v1/ai/infer/batch                   Permission: ai.infer
GET           /api/v1/ai/infer/history                 Permission: ai.infer.read

# Document Recognition
POST          /api/v1/ai/document-recognize            Permission: ai.documents.recognize
GET           /api/v1/ai/document-recognize/{id}       Permission: ai.documents.read
POST          /api/v1/ai/document-recognize/{id}/review Permission: ai.documents.review

# Anomaly Detection
GET           /api/v1/ai/anomalies                     Permission: ai.anomalies.read
POST          /api/v1/ai/anomalies/scan                Permission: ai.anomalies.scan
POST          /api/v1/ai/anomalies/{id}/acknowledge    Permission: ai.anomalies.update
POST          /api/v1/ai/anomalies/{id}/resolve        Permission: ai.anomalies.update

# Forecasting
GET/POST      /api/v1/ai/forecasts/models             Permission: ai.forecasts.read/create
GET           /api/v1/ai/forecasts/models/{id}        Permission: ai.forecasts.read
POST          /api/v1/ai/forecasts/models/{id}/train  Permission: ai.forecasts.train
GET           /api/v1/ai/forecasts/models/{id}/results Permission: ai.forecasts.read
GET           /api/v1/ai/forecasts/accuracy           Permission: ai.forecasts.read

# Natural Language Query
POST          /api/v1/ai/query                         Permission: ai.query
GET           /api/v1/ai/query/history                 Permission: ai.query.read
POST          /api/v1/ai/query/{id}/feedback           Permission: ai.query.feedback
```

---

## 4. Business Rules

### 4.1 Agent Execution
```
1. AI agents operate asynchronously; long-running tasks return a job_id for polling
2. Each agent invocation is logged with input, output, confidence, and execution time
3. Agent definitions are versioned; invoking an agent uses the active version
4. Agent type determines the processing pipeline:
   - DOCUMENT_RECOGNITION -> OCR + field extraction + validation
   - ANOMALY_DETECTION -> Z-score + isolation forest scoring
   - FORECAST -> time-series model inference
   - POLICY_ADVISOR -> rule engine + compliance check
   - COMPARISON -> feature vector similarity
   - CLASSIFICATION -> trained model inference
   - NLP_QUERY -> intent detection + SQL generation
```

### 4.2 Document Recognition
- Confidence below 70% MUST flag `requires_review = 1`
- Extracted fields are stored as JSON keyed by document type field map
- Reviewer corrections are tracked to improve future recognition
- OCR processing time is logged for performance monitoring

### 4.3 Anomaly Detection
- Uses Z-score algorithm for statistical outliers: `z = (x - mean) / stddev`
- Uses isolation forest algorithm for multidimensional anomaly detection
- Anomaly scores above threshold trigger severity classification:
  - LOW: score 0.5-0.6
  - MEDIUM: score 0.6-0.75
  - HIGH: score 0.75-0.9
  - CRITICAL: score > 0.9
- All anomalies include a suggested action based on anomaly type
- Entities scanned: journals, invoices, payments, expenses, orders

### 4.4 Forecasting
- Forecasting models MUST track MAPE and RMSE accuracy metrics
- MAPE = `AVG(|actual - predicted| / actual) * 100`
- RMSE = `sqrt(AVG((actual - predicted)^2))`
- Models with accuracy below threshold MUST be flagged for retraining
- Confidence intervals stored as lower_bound and upper_bound
- Actual values are backfilled as they become available for accuracy tracking
- Outlier detection flags anomalous actual vs predicted variances

### 4.5 Natural Language Query
- NLP queries are sanitized to prevent SQL injection
- Only SELECT queries are generated; no INSERT, UPDATE, DELETE, or DDL
- Intent detection maps user question to queryable ERP entities
- Query results are summarized in natural language
- User feedback (rating + text) is captured for model improvement

### 4.6 Model Governance
- All inference results are logged for audit trail and model improvement
- Bias detection MUST be run on classification models before deployment
- Tenant data MUST NOT leak between tenants in shared models
- Model deployment workflow: TRAINING -> VALIDATING -> DEPLOYED
- Deprecated models remain accessible for audit but cannot serve inference
- Training datasets are validated for completeness before training begins

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.ai.v1;

service AiService {
    rpc ExecuteAgent(ExecuteAgentRequest) returns (ExecuteAgentResponse);
    rpc RecognizeDocument(RecognizeDocumentRequest) returns (RecognizeDocumentResponse);
    rpc DetectAnomalies(DetectAnomaliesRequest) returns (DetectAnomaliesResponse);
    rpc GetForecast(GetForecastRequest) returns (GetForecastResponse);
    rpc NaturalLanguageQuery(NaturalLanguageQueryRequest) returns (NaturalLanguageQueryResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **DMS:** Document files for OCR/recognition processing
- **GL:** Journal entries for anomaly detection
- **AP:** Invoices for anomaly detection and document recognition
- **AR:** Receipts and payments for anomaly detection
- **INV:** Inventory levels for demand forecasting
- **OM:** Sales orders for demand forecasting and anomaly detection
- **Proc:** Purchase orders for spend anomaly detection
- **Expense:** Expense reports for fraud/anomaly detection
- **CM:** Cash flow data for cash flow forecasting
- **Auth:** User context for NLP query permissions

### 6.2 Data Published To
- **GL:** Anomaly flags on journal entries
- **AP:** Extracted invoice data from document recognition
- **INV:** Demand forecast results for planning
- **OM:** Demand forecast results for order planning
- **Proc:** Spend anomaly alerts
- **Expense:** Expense anomaly alerts
- **Reporting:** All AI/ML metrics and results for dashboards

### 6.3 Events Consumed
| Event | Action |
|-------|--------|
| `dms.document.uploaded` | Trigger document recognition |
| `gl.journal.posted` | Trigger anomaly scan |
| `ap.invoice.created` | Trigger anomaly scan |
| `ar.payment.received` | Trigger anomaly scan |
| `expense.report.submitted` | Trigger anomaly scan |

### 6.4 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `ai.inference.completed` | Inference request processed | Reporting, Audit |
| `ai.anomaly.detected` | Anomaly found | GL, AP, Expense, WF |
| `ai.document.recognized` | Document OCR complete | AP, DMS |
| `ai.forecast.updated` | Forecast model retrained | INV, OM, Planning |
| `ai.model.deployed` | Model promoted to production | Reporting |
| `ai.training.completed` | Training job finished | Reporting |

---

## 7. Migrations

1. V001: `ai_agent_definitions`
2. V002: `ai_model_registry`
3. V003: `ai_training_datasets`
4. V004: `ai_training_jobs`
5. V005: `ai_inference_requests`
6. V006: `ai_document_recognition`
7. V007: `ai_anomaly_detections`
8. V008: `ai_forecast_models`
9. V009: `ai_forecast_results`
10. V010: `ai_nlp_queries`
11. V011: Indexes for all tables
12. V012: Triggers for `updated_at`
