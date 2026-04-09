# 102 - Fusion Data Intelligence Service Specification

## 1. Domain Overview

Fusion Data Intelligence is a cross-application analytics platform providing pre-built data models, KPIs, augmented analytics with NLP querying, predictive analytics, interactive dashboards, and a centralized analytics warehouse optimized for Oracle Fusion data. It supports analytics across ERP, SCM, HCM, CX, and EPM domains, enabling data-driven decision making through unified insights.

**Bounded Context:** Cross-Domain Analytics & Data Intelligence
**Service Name:** `fdi-service`
**Database:** `data/fdi.db`
**HTTP Port:** 8139 | **gRPC Port:** 9139

---

## 2. Database Schema

### 2.1 Data Models
```sql
CREATE TABLE data_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL
        CHECK(model_type IN ('STAR','SNOWFLAKE','FLAT')),
    source_domain TEXT NOT NULL
        CHECK(source_domain IN ('ERP','SCM','HCM','CX','EPM','CROSS_DOMAIN')),
    description TEXT,
    schema_definition TEXT NOT NULL,  -- JSON
    refresh_frequency TEXT NOT NULL DEFAULT 'DAILY'
        CHECK(refresh_frequency IN ('REAL_TIME','HOURLY','DAILY','WEEKLY')),
    last_refreshed_at TEXT,
    row_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'BUILDING'
        CHECK(status IN ('ACTIVE','DEPRECATED','BUILDING')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_name)
);

CREATE INDEX idx_data_models_tenant_domain ON data_models(tenant_id, source_domain);
CREATE INDEX idx_data_models_tenant_status ON data_models(tenant_id, status);
```

### 2.2 KPI Definitions
```sql
CREATE TABLE kpi_definitions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_name TEXT NOT NULL,
    domain TEXT NOT NULL
        CHECK(domain IN ('ERP','SCM','HCM','CX','EPM')),
    description TEXT,
    calculation_formula TEXT NOT NULL,
    unit_type TEXT NOT NULL
        CHECK(unit_type IN ('CURRENCY','PERCENTAGE','COUNT','RATIO','DAYS')),
    target_value DECIMAL(15,4),
    warning_threshold DECIMAL(15,4),
    critical_threshold DECIMAL(15,4),
    trend_direction TEXT NOT NULL DEFAULT 'HIGHER_IS_BETTER'
        CHECK(trend_direction IN ('HIGHER_IS_BETTER','LOWER_IS_BETTER','ON_TARGET')),
    dimensions TEXT,  -- JSON
    filters TEXT,     -- JSON
    visualization_type TEXT NOT NULL DEFAULT 'LINE'
        CHECK(visualization_type IN ('LINE','BAR','GAUGE','TABLE','HEATMAP')),
    owner_id TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, kpi_name)
);

CREATE INDEX idx_kpi_defs_tenant_domain ON kpi_definitions(tenant_id, domain);
CREATE INDEX idx_kpi_defs_tenant_owner ON kpi_definitions(tenant_id, owner_id);
CREATE INDEX idx_kpi_defs_tenant_active ON kpi_definitions(tenant_id, is_active);
```

### 2.3 KPI Values
```sql
CREATE TABLE kpi_values (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    kpi_id TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY','ANNUAL')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    actual_value DECIMAL(15,4) NOT NULL DEFAULT 0,
    target_value DECIMAL(15,4) NOT NULL DEFAULT 0,
    variance DECIMAL(15,4),
    variance_pct DECIMAL(10,4),
    status TEXT NOT NULL DEFAULT 'ON_TRACK'
        CHECK(status IN ('ON_TRACK','WARNING','CRITICAL')),
    dimensions_applied TEXT,  -- JSON
    calculated_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (kpi_id) REFERENCES kpi_definitions(id) ON DELETE CASCADE
);

CREATE INDEX idx_kpi_values_tenant_kpi ON kpi_values(tenant_id, kpi_id);
CREATE INDEX idx_kpi_values_tenant_period ON kpi_values(tenant_id, period_start, period_end);
CREATE INDEX idx_kpi_values_tenant_status ON kpi_values(tenant_id, status);
```

### 2.4 Dashboards
```sql
CREATE TABLE dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    domain TEXT NOT NULL
        CHECK(domain IN ('ERP','SCM','HCM','CX','EPM')),
    description TEXT,
    layout_config TEXT NOT NULL,  -- JSON
    widgets TEXT,                 -- JSON
    sharing_type TEXT NOT NULL DEFAULT 'PRIVATE'
        CHECK(sharing_type IN ('PRIVATE','SHARED','PUBLIC')),
    owner_id TEXT NOT NULL,
    folder_id TEXT,
    is_default INTEGER NOT NULL DEFAULT 0,
    refresh_interval_seconds INTEGER NOT NULL DEFAULT 300,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dashboard_name)
);

CREATE INDEX idx_dashboards_tenant_domain ON dashboards(tenant_id, domain);
CREATE INDEX idx_dashboards_tenant_owner ON dashboards(tenant_id, owner_id);
CREATE INDEX idx_dashboards_tenant_sharing ON dashboards(tenant_id, sharing_type);
```

### 2.5 Dashboard Widgets
```sql
CREATE TABLE dashboard_widgets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_id TEXT NOT NULL,
    widget_type TEXT NOT NULL
        CHECK(widget_type IN ('KPI_CHART','TABLE','FILTER','TEXT','IMAGE','IFRAME','PIVOT')),
    title TEXT NOT NULL,
    data_source_config TEXT NOT NULL,  -- JSON
    visualization_config TEXT NOT NULL, -- JSON
    position_config TEXT NOT NULL,      -- JSON
    filters TEXT,                       -- JSON
    refresh_frequency INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (dashboard_id) REFERENCES dashboards(id) ON DELETE CASCADE
);

CREATE INDEX idx_widgets_tenant_dashboard ON dashboard_widgets(tenant_id, dashboard_id);
CREATE INDEX idx_widgets_tenant_type ON dashboard_widgets(tenant_id, widget_type);
```

### 2.6 Analytics Queries
```sql
CREATE TABLE analytics_queries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    query_text TEXT NOT NULL,
    query_type TEXT NOT NULL
        CHECK(query_type IN ('NLP','SQL','DRAG_DROP')),
    domain TEXT NOT NULL
        CHECK(domain IN ('ERP','SCM','HCM','CX','EPM')),
    generated_sql TEXT,
    result_cache_key TEXT,
    executed_by TEXT NOT NULL,
    execution_time_ms INTEGER,
    row_count INTEGER,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED')),
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_queries_tenant_domain ON analytics_queries(tenant_id, domain);
CREATE INDEX idx_queries_tenant_status ON analytics_queries(tenant_id, status);
CREATE INDEX idx_queries_tenant_cache ON analytics_queries(tenant_id, result_cache_key);
```

### 2.7 Prediction Models
```sql
CREATE TABLE prediction_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL
        CHECK(model_type IN ('REGRESSION','CLASSIFICATION','TIME_SERIES','CLUSTERING','ANOMALY_DETECTION')),
    domain TEXT NOT NULL
        CHECK(domain IN ('ERP','SCM','HCM','CX','EPM')),
    input_features TEXT NOT NULL,  -- JSON
    target_variable TEXT NOT NULL,
    training_data_range TEXT,
    accuracy_score DECIMAL(5,4),
    last_trained_at TEXT,
    model_version TEXT,
    status TEXT NOT NULL DEFAULT 'EXPERIMENTAL'
        CHECK(status IN ('EXPERIMENTAL','PRODUCTION','DEPRECATED')),
    schedule TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_name)
);

CREATE INDEX idx_pred_models_tenant_domain ON prediction_models(tenant_id, domain);
CREATE INDEX idx_pred_models_tenant_status ON prediction_models(tenant_id, status);
```

### 2.8 Data Lineage
```sql
CREATE TABLE data_lineage (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    target_model_id TEXT NOT NULL,
    source_service TEXT NOT NULL,
    source_table TEXT NOT NULL,
    source_field TEXT NOT NULL,
    transformation_rule TEXT,
    lineage_type TEXT NOT NULL DEFAULT 'DIRECT'
        CHECK(lineage_type IN ('DIRECT','DERIVED','AGGREGATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (target_model_id) REFERENCES data_models(id) ON DELETE CASCADE
);

CREATE INDEX idx_lineage_tenant_model ON data_lineage(tenant_id, target_model_id);
CREATE INDEX idx_lineage_tenant_source ON data_lineage(tenant_id, source_service);
```

---

## 3. REST API Endpoints

### Data Models
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/fdi/models` | Create a data model |
| GET | `/api/v1/fdi/models` | List data models |
| GET | `/api/v1/fdi/models/{id}` | Get model details |
| PUT | `/api/v1/fdi/models/{id}` | Update data model |
| POST | `/api/v1/fdi/models/{id}/refresh` | Refresh a data model |
| GET | `/api/v1/fdi/models/{id}/schema` | Get model schema |

### KPIs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/fdi/kpis` | Create a KPI definition |
| GET | `/api/v1/fdi/kpis` | List KPI definitions |
| GET | `/api/v1/fdi/kpis/{id}` | Get KPI details |
| PUT | `/api/v1/fdi/kpis/{id}` | Update KPI definition |
| GET | `/api/v1/fdi/kpis/{id}/values` | Get KPI values |
| POST | `/api/v1/fdi/kpis/{id}/calculate` | Calculate KPI value |
| GET | `/api/v1/fdi/kpis/{id}/trends` | Get KPI trend data |

### Dashboards
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/fdi/dashboards` | Create a dashboard |
| GET | `/api/v1/fdi/dashboards` | List dashboards |
| GET | `/api/v1/fdi/dashboards/{id}` | Get dashboard details |
| PUT | `/api/v1/fdi/dashboards/{id}` | Update dashboard |
| POST | `/api/v1/fdi/dashboards/{id}/share` | Share a dashboard |
| GET | `/api/v1/fdi/dashboards/shared` | Get shared dashboards |

### Widgets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/fdi/dashboards/{dashboardId}/widgets` | Create a widget |
| GET | `/api/v1/fdi/widgets/{id}` | Get widget details |
| PUT | `/api/v1/fdi/widgets/{id}` | Update widget |
| DELETE | `/api/v1/fdi/widgets/{id}` | Delete widget |
| POST | `/api/v1/fdi/dashboards/{dashboardId}/widgets/reorder` | Reorder widgets |

### Queries
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/fdi/queries/execute-nlp` | Execute an NLP query |
| POST | `/api/v1/fdi/queries/execute-sql` | Execute a SQL query |
| GET | `/api/v1/fdi/queries/results/{cacheKey}` | Get cached query results |

### Predictions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/fdi/predictions` | Create a prediction model |
| GET | `/api/v1/fdi/predictions` | List prediction models |
| GET | `/api/v1/fdi/predictions/{id}` | Get model details |
| PUT | `/api/v1/fdi/predictions/{id}` | Update prediction model |
| POST | `/api/v1/fdi/predictions/{id}/train` | Train a prediction model |
| POST | `/api/v1/fdi/predictions/{id}/predict` | Run prediction |
| GET | `/api/v1/fdi/predictions/{id}/accuracy` | Get model accuracy |

### Lineage
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/lineage/model/{modelId}` | Get lineage for a model |
| GET | `/api/v1/fdi/lineage/impact-analysis` | Get impact analysis |

### Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fdi/analytics/cross-domain-metrics` | Get cross-domain metrics |
| GET | `/api/v1/fdi/analytics/executive-summary` | Get executive summary |

---

## 4. Business Rules

1. A data model MUST define its schema before it can be activated for querying.
2. KPI calculations MUST use the formula defined in the KPI definition; the system MUST NOT allow ad-hoc overrides at calculation time.
3. NLP queries MUST be sanitized to prevent injection attacks; the system MUST reject queries containing destructive SQL keywords (DROP, DELETE, UPDATE, INSERT).
4. A prediction model MUST achieve a minimum accuracy score threshold before it MAY be promoted to `PRODUCTION` status.
5. Dashboard sharing MUST respect the tenant boundary; a dashboard MUST NOT be shared across tenants.
6. Data model refresh operations SHOULD be rate-limited to prevent excessive load on source systems; real-time refresh SHOULD only be available for models with fewer than one million rows.
7. KPI threshold evaluations MUST be performed after every calculation; the system MUST publish a `KPIThresholdBreached` event when a value crosses the warning or critical threshold.
8. A prediction model in `EXPERIMENTAL` status MUST NOT be used for production analytics or executive dashboards.
9. Query results MUST be cached using the result cache key; cache duration MUST be determined by the data model refresh frequency.
10. Data lineage MUST be tracked for every data model; the system MUST NOT allow a model to activate without at least one lineage record.
11. Widgets on a dashboard MUST be positioned according to the layout config; overlapping positions MUST be rejected by the system.
12. SQL queries executed through the query interface MUST be read-only; the system MUST enforce a query timeout of 30 seconds.
13. A deprecated data model MUST NOT be used as a source for new KPI definitions or widgets.
14. Prediction model training MUST log the training data range, accuracy score, and model version for reproducibility.
15. Executive summary analytics MUST aggregate data from all domains and MUST be refreshed at least daily.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fdi.v1;

service FusionDataIntelligenceService {
    // Data Models
    rpc CreateDataModel(CreateDataModelRequest) returns (CreateDataModelResponse);
    rpc GetDataModel(GetDataModelRequest) returns (GetDataModelResponse);
    rpc ListDataModels(ListDataModelsRequest) returns (ListDataModelsResponse);
    rpc UpdateDataModel(UpdateDataModelRequest) returns (UpdateDataModelResponse);
    rpc RefreshDataModel(RefreshDataModelRequest) returns (RefreshDataModelResponse);
    rpc GetDataModelSchema(GetDataModelSchemaRequest) returns (GetDataModelSchemaResponse);

    // KPIs
    rpc CreateKPI(CreateKPIRequest) returns (CreateKPIResponse);
    rpc GetKPI(GetKPIRequest) returns (GetKPIResponse);
    rpc ListKPIs(ListKPIsRequest) returns (ListKPIsResponse);
    rpc UpdateKPI(UpdateKPIRequest) returns (UpdateKPIResponse);
    rpc GetKPIValues(GetKPIValuesRequest) returns (GetKPIValuesResponse);
    rpc CalculateKPI(CalculateKPIRequest) returns (CalculateKPIResponse);
    rpc GetKPITrends(GetKPITrendsRequest) returns (GetKPITrendsResponse);

    // Dashboards
    rpc CreateDashboard(CreateDashboardRequest) returns (CreateDashboardResponse);
    rpc GetDashboard(GetDashboardRequest) returns (GetDashboardResponse);
    rpc ListDashboards(ListDashboardsRequest) returns (ListDashboardsResponse);
    rpc UpdateDashboard(UpdateDashboardRequest) returns (UpdateDashboardResponse);
    rpc ShareDashboard(ShareDashboardRequest) returns (ShareDashboardResponse);
    rpc GetSharedDashboards(GetSharedDashboardsRequest) returns (GetSharedDashboardsResponse);

    // Widgets
    rpc CreateWidget(CreateWidgetRequest) returns (CreateWidgetResponse);
    rpc GetWidget(GetWidgetRequest) returns (GetWidgetResponse);
    rpc UpdateWidget(UpdateWidgetRequest) returns (UpdateWidgetResponse);
    rpc DeleteWidget(DeleteWidgetRequest) returns (DeleteWidgetResponse);
    rpc ReorderWidgets(ReorderWidgetsRequest) returns (ReorderWidgetsResponse);

    // Queries
    rpc ExecuteNLPQuery(ExecuteNLPQueryRequest) returns (ExecuteNLPQueryResponse);
    rpc ExecuteSQLQuery(ExecuteSQLQueryRequest) returns (ExecuteSQLQueryResponse);
    rpc GetQueryResults(GetQueryResultsRequest) returns (GetQueryResultsResponse);

    // Predictions
    rpc CreatePredictionModel(CreatePredictionModelRequest) returns (CreatePredictionModelResponse);
    rpc GetPredictionModel(GetPredictionModelRequest) returns (GetPredictionModelResponse);
    rpc ListPredictionModels(ListPredictionModelsRequest) returns (ListPredictionModelsResponse);
    rpc UpdatePredictionModel(UpdatePredictionModelRequest) returns (UpdatePredictionModelResponse);
    rpc TrainPredictionModel(TrainPredictionModelRequest) returns (TrainPredictionModelResponse);
    rpc RunPrediction(RunPredictionRequest) returns (RunPredictionResponse);
    rpc GetModelAccuracy(GetModelAccuracyRequest) returns (GetModelAccuracyResponse);

    // Lineage
    rpc GetLineageForModel(GetLineageForModelRequest) returns (GetLineageForModelResponse);
    rpc GetImpactAnalysis(GetImpactAnalysisRequest) returns (GetImpactAnalysisResponse);

    // Analytics
    rpc GetCrossDomainMetrics(GetCrossDomainMetricsRequest) returns (GetCrossDomainMetricsResponse);
    rpc GetExecutiveSummary(GetExecutiveSummaryRequest) returns (GetExecutiveSummaryResponse);
}

message DataModel {
    string id = 1;
    string tenant_id = 2;
    string model_name = 3;
    string model_type = 4;
    string source_domain = 5;
    string description = 6;
    string schema_definition = 7;
    string refresh_frequency = 8;
    string last_refreshed_at = 9;
    int64 row_count = 10;
    string status = 11;
}

message CreateDataModelRequest {
    string tenant_id = 1;
    string model_name = 2;
    string model_type = 3;
    string source_domain = 4;
    string description = 5;
    string schema_definition = 6;
    string refresh_frequency = 7;
    string created_by = 8;
}

message CreateDataModelResponse {
    DataModel model = 1;
}

message GetDataModelRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetDataModelResponse {
    DataModel model = 1;
}

message ListDataModelsRequest {
    string tenant_id = 1;
    string source_domain = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListDataModelsResponse {
    repeated DataModel models = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateDataModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string model_name = 3;
    string schema_definition = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateDataModelResponse {
    DataModel model = 1;
}

message RefreshDataModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string refreshed_by = 3;
}

message RefreshDataModelResponse {
    DataModel model = 1;
    int64 row_count = 2;
}

message GetDataModelSchemaRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetDataModelSchemaResponse {
    string schema_definition = 1;
}

message KPIDefinition {
    string id = 1;
    string tenant_id = 2;
    string kpi_name = 3;
    string domain = 4;
    string description = 5;
    string calculation_formula = 6;
    string unit_type = 7;
    double target_value = 8;
    string trend_direction = 9;
    string visualization_type = 10;
    string owner_id = 11;
}

message CreateKPIRequest {
    string tenant_id = 1;
    string kpi_name = 2;
    string domain = 3;
    string description = 4;
    string calculation_formula = 5;
    string unit_type = 6;
    double target_value = 7;
    double warning_threshold = 8;
    double critical_threshold = 9;
    string trend_direction = 10;
    string dimensions = 11;
    string filters = 12;
    string visualization_type = 13;
    string owner_id = 14;
    string created_by = 15;
}

message CreateKPIResponse {
    KPIDefinition kpi = 1;
}

message GetKPIRequest {
    string tenant_id = 1;
    string kpi_id = 2;
}

message GetKPIResponse {
    KPIDefinition kpi = 1;
}

message ListKPIsRequest {
    string tenant_id = 1;
    string domain = 2;
    string owner_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListKPIsResponse {
    repeated KPIDefinition kpis = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateKPIRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    string kpi_name = 3;
    string calculation_formula = 4;
    double target_value = 5;
    string updated_by = 6;
    int32 version = 7;
}

message UpdateKPIResponse {
    KPIDefinition kpi = 1;
}

message KPIValue {
    string id = 1;
    string kpi_id = 2;
    string period_type = 3;
    string period_start = 4;
    string period_end = 5;
    double actual_value = 6;
    double target_value = 7;
    double variance = 8;
    double variance_pct = 9;
    string status = 10;
    string calculated_at = 11;
}

message GetKPIValuesRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    string period_type = 3;
    string period_start = 4;
    string period_end = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message GetKPIValuesResponse {
    repeated KPIValue values = 1;
    string next_page_token = 2;
}

message CalculateKPIRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    string period_type = 3;
    string period_start = 4;
    string period_end = 5;
    string calculated_by = 6;
}

message CalculateKPIResponse {
    KPIValue value = 1;
}

message KPITrendPoint {
    string period_start = 1;
    double actual_value = 2;
    double target_value = 3;
    string status = 4;
}

message GetKPITrendsRequest {
    string tenant_id = 1;
    string kpi_id = 2;
    string period_type = 3;
    int32 period_count = 4;
}

message GetKPITrendsResponse {
    repeated KPITrendPoint trends = 1;
}

message Dashboard {
    string id = 1;
    string tenant_id = 2;
    string dashboard_name = 3;
    string domain = 4;
    string description = 5;
    string layout_config = 6;
    string sharing_type = 7;
    string owner_id = 8;
    bool is_default = 9;
    int32 refresh_interval_seconds = 10;
    string status = 11;
}

message CreateDashboardRequest {
    string tenant_id = 1;
    string dashboard_name = 2;
    string domain = 3;
    string description = 4;
    string layout_config = 5;
    string owner_id = 6;
    int32 refresh_interval_seconds = 7;
    string created_by = 8;
}

message CreateDashboardResponse {
    Dashboard dashboard = 1;
}

message GetDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
}

message GetDashboardResponse {
    Dashboard dashboard = 1;
}

message ListDashboardsRequest {
    string tenant_id = 1;
    string domain = 2;
    string owner_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListDashboardsResponse {
    repeated Dashboard dashboards = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    string dashboard_name = 3;
    string layout_config = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdateDashboardResponse {
    Dashboard dashboard = 1;
}

message ShareDashboardRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    string sharing_type = 3;
    repeated string share_with_user_ids = 4;
    string shared_by = 5;
}

message ShareDashboardResponse {
    Dashboard dashboard = 1;
}

message GetSharedDashboardsRequest {
    string tenant_id = 1;
    string user_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetSharedDashboardsResponse {
    repeated Dashboard dashboards = 1;
    string next_page_token = 2;
}

message Widget {
    string id = 1;
    string tenant_id = 2;
    string dashboard_id = 3;
    string widget_type = 4;
    string title = 5;
    string data_source_config = 6;
    string visualization_config = 7;
    string position_config = 8;
}

message CreateWidgetRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    string widget_type = 3;
    string title = 4;
    string data_source_config = 5;
    string visualization_config = 6;
    string position_config = 7;
    string created_by = 8;
}

message CreateWidgetResponse {
    Widget widget = 1;
}

message GetWidgetRequest {
    string tenant_id = 1;
    string widget_id = 2;
}

message GetWidgetResponse {
    Widget widget = 1;
}

message UpdateWidgetRequest {
    string tenant_id = 1;
    string widget_id = 2;
    string title = 3;
    string data_source_config = 4;
    string visualization_config = 5;
    string position_config = 6;
    string updated_by = 7;
    int32 version = 8;
}

message UpdateWidgetResponse {
    Widget widget = 1;
}

message DeleteWidgetRequest {
    string tenant_id = 1;
    string widget_id = 2;
}

message DeleteWidgetResponse {
    bool success = 1;
}

message ReorderWidgetsRequest {
    string tenant_id = 1;
    string dashboard_id = 2;
    repeated string widget_ids_in_order = 3;
    string reordered_by = 4;
}

message ReorderWidgetsResponse {
    bool success = 1;
}

message AnalyticsQuery {
    string id = 1;
    string tenant_id = 2;
    string query_text = 3;
    string query_type = 4;
    string domain = 5;
    string generated_sql = 6;
    string result_cache_key = 7;
    string status = 8;
}

message ExecuteNLPQueryRequest {
    string tenant_id = 1;
    string query_text = 2;
    string domain = 3;
    string executed_by = 4;
}

message ExecuteNLPQueryResponse {
    AnalyticsQuery query = 1;
    string result_data = 2;  -- JSON
    int32 row_count = 3;
}

message ExecuteSQLQueryRequest {
    string tenant_id = 1;
    string sql = 2;
    string domain = 3;
    string executed_by = 4;
}

message ExecuteSQLQueryResponse {
    AnalyticsQuery query = 1;
    string result_data = 2;  -- JSON
    int32 row_count = 3;
}

message GetQueryResultsRequest {
    string tenant_id = 1;
    string cache_key = 2;
}

message GetQueryResultsResponse {
    string result_data = 1;  -- JSON
    int32 row_count = 2;
    string cached_at = 3;
}

message PredictionModel {
    string id = 1;
    string tenant_id = 2;
    string model_name = 3;
    string model_type = 4;
    string domain = 5;
    string input_features = 6;
    string target_variable = 7;
    double accuracy_score = 8;
    string model_version = 9;
    string status = 10;
}

message CreatePredictionModelRequest {
    string tenant_id = 1;
    string model_name = 2;
    string model_type = 3;
    string domain = 4;
    string input_features = 5;
    string target_variable = 6;
    string training_data_range = 7;
    string created_by = 8;
}

message CreatePredictionModelResponse {
    PredictionModel model = 1;
}

message GetPredictionModelRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetPredictionModelResponse {
    PredictionModel model = 1;
}

message ListPredictionModelsRequest {
    string tenant_id = 1;
    string domain = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListPredictionModelsResponse {
    repeated PredictionModel models = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdatePredictionModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string model_name = 3;
    string input_features = 4;
    string updated_by = 5;
    int32 version = 6;
}

message UpdatePredictionModelResponse {
    PredictionModel model = 1;
}

message TrainPredictionModelRequest {
    string tenant_id = 1;
    string model_id = 2;
    string training_data_range = 3;
    string trained_by = 4;
}

message TrainPredictionModelResponse {
    string model_version = 1;
    double accuracy_score = 2;
    string trained_at = 3;
}

message RunPredictionRequest {
    string tenant_id = 1;
    string model_id = 2;
    string input_data = 3;  -- JSON
    string predicted_by = 4;
}

message RunPredictionResponse {
    string prediction_result = 1;  -- JSON
    double confidence = 2;
    string predicted_at = 3;
}

message GetModelAccuracyRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetModelAccuracyResponse {
    double accuracy_score = 1;
    string model_version = 2;
    string last_trained_at = 3;
    string training_data_range = 4;
}

message LineageEntry {
    string id = 1;
    string target_model_id = 2;
    string source_service = 3;
    string source_table = 4;
    string source_field = 5;
    string transformation_rule = 6;
    string lineage_type = 7;
}

message GetLineageForModelRequest {
    string tenant_id = 1;
    string model_id = 2;
}

message GetLineageForModelResponse {
    repeated LineageEntry entries = 1;
}

message ImpactAnalysisEntry {
    string source_service = 1;
    string source_table = 2;
    repeated string affected_models = 3;
    repeated string affected_kpis = 4;
    repeated string affected_dashboards = 5;
}

message GetImpactAnalysisRequest {
    string tenant_id = 1;
    string source_service = 2;
    string source_table = 3;
}

message GetImpactAnalysisResponse {
    repeated ImpactAnalysisEntry impacts = 1;
}

message CrossDomainMetric {
    string domain = 1;
    string metric_name = 2;
    double value = 3;
    string unit_type = 4;
    string period = 5;
}

message GetCrossDomainMetricsRequest {
    string tenant_id = 1;
    string period_start = 2;
    string period_end = 3;
}

message GetCrossDomainMetricsResponse {
    repeated CrossDomainMetric metrics = 1;
}

message ExecutiveSummaryEntry {
    string domain = 1;
    string summary_title = 2;
    string summary_text = 3;
    double key_metric_value = 4;
    string key_metric_unit = 5;
    string trend = 6;
}

message GetExecutiveSummaryRequest {
    string tenant_id = 1;
    string period_start = 2;
    string period_end = 3;
}

message GetExecutiveSummaryResponse {
    repeated ExecutiveSummaryEntry entries = 1;
    string generated_at = 2;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `gl-service` | GL balances, journal summaries | Financial KPIs and analytics |
| `ap-service` | AP aging, payment summaries | Payables analytics |
| `ar-service` | AR aging, collection summaries | Receivables analytics |
| `procurement-service` | Purchase order volumes, spend data | Procurement analytics |
| `inventory-service` | Inventory levels, turnover data | Supply chain analytics |
| `order-management` | Order volumes, fulfillment metrics | Sales analytics |
| `payroll-service` | Payroll totals, headcount data | HCM analytics |
| `compensation-service` | Compensation costs, budget data | Workforce cost analytics |
| `sales-service` | Revenue pipeline, win rates | CX analytics |
| `customer-service` | Case volumes, resolution times | Customer experience analytics |
| `epm-service` | Budget vs actual, forecast data | Planning analytics |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `reporting-service` | KPI values, analytics results | Standardized reporting |
| `ai-ml-platform` | Prediction model training requests | ML model management |
| `notification-service` | Threshold breach alerts | Proactive alerting |
| `document-service` | Dashboard exports, reports | Document archival |
| `workflow-service` | Analytics-driven triggers | Automated workflows |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `DataModelRefreshed` | `fdi.model.refreshed` | `{ tenant_id, model_id, model_name, row_count, refreshed_at }` | Published when a data model refresh completes |
| `KPIValueCalculated` | `fdi.kpi.calculated` | `{ tenant_id, kpi_id, kpi_name, actual_value, target_value, variance_pct, status }` | Published when a KPI value is calculated |
| `KPIThresholdBreached` | `fdi.kpi.threshold-breached` | `{ tenant_id, kpi_id, kpi_name, actual_value, threshold_type, threshold_value }` | Published when a KPI crosses a warning or critical threshold |
| `PredictionCompleted` | `fdi.prediction.completed` | `{ tenant_id, model_id, model_name, prediction_result, confidence, predicted_at }` | Published when a prediction execution completes |
| `DashboardShared` | `fdi.dashboard.shared` | `{ tenant_id, dashboard_id, dashboard_name, shared_by, share_with_user_ids }` | Published when a dashboard is shared with users |
