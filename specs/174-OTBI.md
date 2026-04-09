# 174 - OTBI (Transactional Business Intelligence) Service Specification

## 1. Domain Overview

OTBI provides real-time operational analytics directly on transactional data without ETL or data warehouse latency. Supports ad-hoc analysis, live dashboards, pivot tables, and embedded analytics within application pages. Enables business users to build real-time reports and analyses on current transactional data with drill-down, slicing, and dicing capabilities. Distinct from batch Reporting (17) which focuses on scheduled/static reports, OTBI focuses on interactive, real-time analytical queries. Integrates with all application modules for live data access.

**Bounded Context:** Real-Time Operational Analytics & Ad-Hoc Analysis
**Service Name:** `otbi-service`
**Database:** `data/otbi.db`
**HTTP Port:** 8192 | **gRPC Port:** 9192

---

## 2. Database Schema

### 2.1 Subject Areas
```sql
CREATE TABLE otbi_subject_areas (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    area_code TEXT NOT NULL,
    area_name TEXT NOT NULL,
    source_module TEXT NOT NULL,
    description TEXT,
    folder_hierarchy TEXT NOT NULL,          -- JSON: dimensional hierarchy
    available_metrics TEXT NOT NULL,         -- JSON: measure definitions
    available_dimensions TEXT NOT NULL,      -- JSON: dimension attributes
    available_filters TEXT,                  -- JSON: filterable fields
    default_time_dimension TEXT NOT NULL,
    row_count_estimate INTEGER,
    refresh_type TEXT NOT NULL DEFAULT 'REAL_TIME'
        CHECK(refresh_type IN ('REAL_TIME','NEAR_REAL_TIME','CACHED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, area_code)
);

CREATE INDEX idx_otbi_area_module ON otbi_subject_areas(source_module);
```

### 2.2 Saved Analyses
```sql
CREATE TABLE otbi_analyses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    analysis_name TEXT NOT NULL,
    subject_area_id TEXT NOT NULL,
    analysis_type TEXT NOT NULL CHECK(analysis_type IN ('TABLE','PIVOT','CHART','TRELLIS','DASHBOARD','COMPOUND')),
    description TEXT,
    columns TEXT NOT NULL,                   -- JSON: selected columns with formatting
    filters TEXT,                            -- JSON: filter conditions
    sort_config TEXT,                        -- JSON: sort order
    visualization_config TEXT,               -- JSON: chart type, colors, layout
    prompt_config TEXT,                      -- JSON: user prompts/parameters
    row_limit INTEGER NOT NULL DEFAULT 1000,
    is_shared INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    folder_path TEXT,
    last_run_at TEXT,
    last_row_count INTEGER NOT NULL DEFAULT 0,
    avg_execution_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, analysis_name, owner_id)
);

CREATE INDEX idx_otbi_analysis_owner ON otbi_analyses(owner_id);
CREATE INDEX idx_otbi_analysis_area ON otbi_analyses(subject_area_id);
```

### 2.3 Dashboard Definitions
```sql
CREATE TABLE otbi_dashboards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    dashboard_name TEXT NOT NULL,
    description TEXT,
    layout TEXT NOT NULL,                    -- JSON: grid layout with widget positions
    widgets TEXT NOT NULL,                   -- JSON: [{ "analysis_id": "...", "position": {...} }]
    prompts TEXT,                            -- JSON: dashboard-level prompts
    refresh_interval_seconds INTEGER NOT NULL DEFAULT 0,
    is_shared INTEGER NOT NULL DEFAULT 0,
    owner_id TEXT NOT NULL,
    is_default INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, dashboard_name, owner_id)
);
```

### 2.4 Query Log
```sql
CREATE TABLE otbi_query_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    analysis_id TEXT,
    subject_area_id TEXT,
    query_sql TEXT NOT NULL,
    execution_time_ms INTEGER NOT NULL,
    row_count INTEGER NOT NULL DEFAULT 0,
    cache_hit INTEGER NOT NULL DEFAULT 0,
    query_type TEXT NOT NULL CHECK(query_type IN ('TABLE','PIVOT','CHART','EXPORT','DRILL')),
    error_message TEXT,

    executed_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_otbi_query_user ON otbi_query_log(user_id, executed_at DESC);
CREATE INDEX idx_otbi_query_time ON otbi_query_log(tenant_id, executed_at DESC);
```

### 2.5 Cached Results
```sql
CREATE TABLE otbi_cache (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    cache_key TEXT NOT NULL,                 -- Hash of query + parameters
    subject_area_id TEXT NOT NULL,
    result_data TEXT NOT NULL,               -- JSON: cached result set
    row_count INTEGER NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    expires_at TEXT NOT NULL,
    hit_count INTEGER NOT NULL DEFAULT 0,

    UNIQUE(tenant_id, cache_key)
);

CREATE INDEX idx_otbi_cache_expiry ON otbi_cache(expires_at);
```

---

## 3. API Endpoints

### 3.1 Query Execution
| Method | Path | Description |
|--------|------|-------------|
| POST | `/otbi/v1/query` | Execute ad-hoc query |
| POST | `/otbi/v1/query/pivot` | Execute pivot query |
| POST | `/otbi/v1/query/drill` | Drill-down query |
| POST | `/otbi/v1/query/export` | Export query results |

### 3.2 Subject Areas
| Method | Path | Description |
|--------|------|-------------|
| GET | `/otbi/v1/subject-areas` | List subject areas |
| GET | `/otbi/v1/subject-areas/{id}` | Get area metadata |
| GET | `/otbi/v1/subject-areas/{id}/columns` | Available columns |

### 3.3 Analyses
| Method | Path | Description |
|--------|------|-------------|
| POST | `/otbi/v1/analyses` | Save analysis |
| GET | `/otbi/v1/analyses` | List analyses |
| GET | `/otbi/v1/analyses/{id}` | Get analysis |
| PUT | `/otbi/v1/analyses/{id}` | Update analysis |
| DELETE | `/otbi/v1/analyses/{id}` | Delete analysis |
| POST | `/otbi/v1/analyses/{id}/run` | Run saved analysis |

### 3.4 Dashboards
| Method | Path | Description |
|--------|------|-------------|
| POST | `/otbi/v1/dashboards` | Create dashboard |
| GET | `/otbi/v1/dashboards` | List dashboards |
| GET | `/otbi/v1/dashboards/{id}` | Get dashboard with data |
| PUT | `/otbi/v1/dashboards/{id}` | Update dashboard |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `otbi.query.slow` | `{ query_id, duration_ms, subject_area }` | Slow query detected |
| `otbi.cache.invalidated` | `{ subject_area, cache_keys }` | Cache invalidated |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `*.created` | All modules | Invalidate affected cache entries |
| `*.updated` | All modules | Invalidate affected cache entries |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.otbi.v1;

service OtbiService {
    rpc ExecuteQuery(ExecuteQueryRequest) returns (ExecuteQueryResponse);
    rpc GetSubjectArea(GetSubjectAreaRequest) returns (GetSubjectAreaResponse);
    rpc SaveAnalysis(SaveAnalysisRequest) returns (SaveAnalysisResponse);
    rpc GetAnalysis(GetAnalysisRequest) returns (GetAnalysisResponse);
    rpc CreateDashboard(CreateDashboardRequest) returns (CreateDashboardResponse);
    rpc GetDashboard(GetDashboardRequest) returns (GetDashboardResponse);
}

// Query messages
message ExecuteQueryRequest {
    string tenant_id = 1;
    string subject_area_id = 2;
    string columns = 3;
    string filters = 4;
    string sort_config = 5;
    int32 row_limit = 6;
    string query_type = 7;
    string prompt_values = 8;
}

message ExecuteQueryResponse {
    string result_data = 1;
    int32 row_count = 2;
    int32 execution_time_ms = 3;
    int32 cache_hit = 4;
}

// Subject Area messages
message GetSubjectAreaRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetSubjectAreaResponse {
    SubjectArea data = 1;
}

message SubjectArea {
    string id = 1;
    string tenant_id = 2;
    string area_code = 3;
    string area_name = 4;
    string source_module = 5;
    string description = 6;
    string folder_hierarchy = 7;
    string available_metrics = 8;
    string available_dimensions = 9;
    string available_filters = 10;
    string default_time_dimension = 11;
    int32 row_count_estimate = 12;
    string refresh_type = 13;
    string created_at = 14;
    string updated_at = 15;
}

// Analysis messages
message SaveAnalysisRequest {
    string tenant_id = 1;
    string analysis_name = 2;
    string subject_area_id = 3;
    string analysis_type = 4;
    string description = 5;
    string columns = 6;
    string filters = 7;
    string sort_config = 8;
    string visualization_config = 9;
    string prompt_config = 10;
    int32 row_limit = 11;
    string owner_id = 12;
    string folder_path = 13;
}

message SaveAnalysisResponse {
    Analysis data = 1;
}

message GetAnalysisRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetAnalysisResponse {
    Analysis data = 1;
}

message Analysis {
    string id = 1;
    string tenant_id = 2;
    string analysis_name = 3;
    string subject_area_id = 4;
    string analysis_type = 5;
    string description = 6;
    string columns = 7;
    string filters = 8;
    string sort_config = 9;
    string visualization_config = 10;
    string prompt_config = 11;
    int32 row_limit = 12;
    int32 is_shared = 13;
    string owner_id = 14;
    string folder_path = 15;
    string last_run_at = 16;
    int32 last_row_count = 17;
    int32 avg_execution_time_ms = 18;
    string created_at = 19;
    string updated_at = 20;
}

// Dashboard messages
message CreateDashboardRequest {
    string tenant_id = 1;
    string dashboard_name = 2;
    string description = 3;
    string layout = 4;
    string widgets = 5;
    string prompts = 6;
    int32 refresh_interval_seconds = 7;
    string owner_id = 8;
}

message CreateDashboardResponse {
    Dashboard data = 1;
}

message GetDashboardRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetDashboardResponse {
    Dashboard data = 1;
}

message Dashboard {
    string id = 1;
    string tenant_id = 2;
    string dashboard_name = 3;
    string description = 4;
    string layout = 5;
    string widgets = 6;
    string prompts = 7;
    int32 refresh_interval_seconds = 8;
    int32 is_shared = 9;
    string owner_id = 10;
    int32 is_default = 11;
    string created_at = 12;
    string updated_at = 13;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | otbi_subject_areas | -- |
| V002 | otbi_analyses | V001 |
| V003 | otbi_dashboards | -- |
| V004 | otbi_query_log | -- |
| V005 | otbi_cache | V001 |

---

## 7. Business Rules

1. **Row Limits**: Queries limited to configured max rows (default 1000); export allows more
2. **Query Timeout**: Queries exceeding 60 seconds killed and error returned
3. **Cache Strategy**: Real-time subject areas not cached; cached areas refreshed per schedule
4. **Access Control**: Data filtered by user's role-based data security
5. **Slow Query Alert**: Queries >10 seconds logged for optimization review
6. **Concurrent Limits**: Max 5 concurrent queries per user; 50 per tenant

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| All Modules | Transactional data source |
| Reporting (17) | Report catalog integration |
| BI Publisher (150) | Report rendering |
| Fusion Data Intelligence (102) | Cross-domain analytics |
| Frontend (20) | Embedded analytics widgets |
| Auth & Security (05) | Data access security |
