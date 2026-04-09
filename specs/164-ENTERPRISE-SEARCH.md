# 164 - Enterprise Search Service Specification

## 1. Domain Overview

Enterprise Search provides unified full-text search across all Fusion application data, documents, and knowledge bases. Supports faceted search, type-ahead suggestions, saved searches, search analytics, and federated search across multiple data sources. Uses Elasticsearch/OpenSearch for indexing and retrieval with relevance tuning, synonym management, and search result boosting. Integrates with all application modules for data indexing and search experiences.

**Bounded Context:** Unified Enterprise Search & Discovery
**Service Name:** `search-service`
**Database:** `data/enterprise_search.db`
**HTTP Port:** 8182 | **gRPC Port:** 9182

---

## 2. Database Schema

### 2.1 Search Index Configurations
```sql
CREATE TABLE es_index_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    index_name TEXT NOT NULL,
    source_module TEXT NOT NULL,
    source_table TEXT NOT NULL,
    display_name TEXT NOT NULL,
    fields TEXT NOT NULL,                   -- JSON: { field: { type, searchable, facetable, boost } }
    id_field TEXT NOT NULL DEFAULT 'id',
    display_fields TEXT NOT NULL,           -- JSON: fields shown in results
    search_weight REAL NOT NULL DEFAULT 1.0,
    freshness_boost_hours INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','REBUILDING','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, index_name)
);

CREATE INDEX idx_es_index_tenant ON es_index_configs(tenant_id, source_module);
```

### 2.2 Saved Searches
```sql
CREATE TABLE es_saved_searches (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    search_name TEXT NOT NULL,
    owner_id TEXT NOT NULL,
    query_text TEXT NOT NULL,
    filters TEXT,                           -- JSON: faceted filter selections
    sort_config TEXT,                       -- JSON: sort fields and directions
    result_fields TEXT,                     -- JSON: which fields to display
    scope_modules TEXT,                     -- JSON: limit to specific modules
    is_shared INTEGER NOT NULL DEFAULT 0,
    shared_with TEXT,                       -- JSON: user/group IDs
    alert_enabled INTEGER NOT NULL DEFAULT 0,
    alert_frequency TEXT CHECK(alert_frequency IN ('REAL_TIME','DAILY','WEEKLY')),
    last_alert_at TEXT,
    result_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, search_name, owner_id)
);

CREATE INDEX idx_es_saved_owner ON es_saved_searches(owner_id, is_active);
```

### 2.3 Search Suggestions Dictionary
```sql
CREATE TABLE es_suggestions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    term TEXT NOT NULL,
    suggestion_type TEXT NOT NULL CHECK(suggestion_type IN ('SYNONYM','AUTOCOMPLETE','SPELLING','POPULAR','MANUAL')),
    target_term TEXT,                       -- For synonyms/spelling corrections
    frequency INTEGER NOT NULL DEFAULT 1,
    last_used_at TEXT,
    context_modules TEXT,                   -- JSON: applicable modules

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, term, suggestion_type)
);

CREATE INDEX idx_es_suggest_term ON es_suggestions(tenant_id, term);
CREATE INDEX idx_es_suggest_freq ON es_suggestions(tenant_id, frequency DESC);
```

### 2.4 Search Analytics
```sql
CREATE TABLE es_search_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    search_date TEXT NOT NULL,
    total_searches INTEGER NOT NULL DEFAULT 0,
    unique_searchers INTEGER NOT NULL DEFAULT 0,
    zero_result_searches INTEGER NOT NULL DEFAULT 0,
    avg_results_per_search REAL NOT NULL DEFAULT 0,
    avg_response_time_ms REAL NOT NULL DEFAULT 0,
    top_searches TEXT NOT NULL,             -- JSON: top 20 search terms
    zero_result_terms TEXT,                 -- JSON: terms with no results
    click_through_rate REAL NOT NULL DEFAULT 0,
    module_distribution TEXT,               -- JSON: searches per module

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, search_date)
);

CREATE INDEX idx_es_analytics_date ON es_search_analytics(tenant_id, search_date DESC);
```

### 2.5 Search Audit Log
```sql
CREATE TABLE es_search_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    query_text TEXT NOT NULL,
    modules TEXT,                           -- JSON: searched modules
    filters TEXT,
    result_count INTEGER NOT NULL DEFAULT 0,
    response_time_ms INTEGER NOT NULL DEFAULT 0,
    clicked_result_id TEXT,
    clicked_result_module TEXT,
    clicked_position INTEGER,

    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_es_log_tenant ON es_search_log(tenant_id, timestamp DESC);
CREATE INDEX idx_es_log_user ON es_search_log(user_id, timestamp DESC);
```

---

## 3. API Endpoints

### 3.1 Search
| Method | Path | Description |
|--------|------|-------------|
| POST | `/search/v1/query` | Execute search query |
| GET | `/search/v1/suggest` | Type-ahead suggestions |
| GET | `/search/v1/facets` | Get facet counts for filters |
| POST | `/search/v1/federated` | Federated search across all modules |

### 3.2 Saved Searches
| Method | Path | Description |
|--------|------|-------------|
| POST | `/search/v1/saved` | Save search |
| GET | `/search/v1/saved` | List saved searches |
| PUT | `/search/v1/saved/{id}` | Update saved search |
| DELETE | `/search/v1/saved/{id}` | Delete saved search |
| POST | `/search/v1/saved/{id}/execute` | Execute saved search |
| POST | `/search/v1/saved/{id}/alert` | Configure search alert |

### 3.3 Index Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/search/v1/indexes` | Create index config |
| GET | `/search/v1/indexes` | List indexes |
| PUT | `/search/v1/indexes/{id}` | Update index config |
| POST | `/search/v1/indexes/{id}/rebuild` | Rebuild index |
| POST | `/search/v1/indexes/{id}/reindex` | Delta reindex |

### 3.4 Suggestions & Synonyms
| Method | Path | Description |
|--------|------|-------------|
| POST | `/search/v1/synonyms` | Add synonym mapping |
| GET | `/search/v1/synonyms` | List synonyms |
| DELETE | `/search/v1/synonyms/{id}` | Remove synonym |
| GET | `/search/v1/popular-terms` | Popular search terms |
| GET | `/search/v1/zero-results` | Zero-result search terms |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/search/v1/analytics/dashboard` | Search analytics dashboard |
| GET | `/search/v1/analytics/trends` | Search trends |
| GET | `/search/v1/analytics/no-results` | Gap analysis |
| POST | `/search/v1/click-tracking` | Track result click |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `search.index.updated` | `{ index_name, documents_count }` | Search index updated |
| `search.alert.triggered` | `{ saved_search_id, new_results_count }` | Search alert fired |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `*.created` | All modules | Index new records |
| `*.updated` | All modules | Re-index updated records |
| `*.deleted` | All modules | Remove from index |

---

## 5. Business Rules

1. **Relevance Tuning**: Search relevance tuned by field weight, freshness, and click-through data
2. **Security Trimming**: Search results filtered based on user's data access permissions
3. **Multi-Language**: Search supports multi-language with language-specific analyzers
4. **Index Latency**: Delta indexing within 5 seconds of data change; full rebuild on schedule
5. **Result Limit**: Default 20 results per page; max 100 per request
6. **Synonym Expansion**: Synonyms applied at query time for broader result matching
7. **Zero-Result Alerts**: Administrators notified of high-frequency zero-result searches
8. **Facet Counts**: Facet counts approximate and refreshed periodically

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| All Modules | Data indexing and search |
| Document Management (29) | Full-text document search |
| Auth & Security (05) | Search result permission filtering |
| Knowledge Management (82) | Knowledge base search |
| Product Hub (51) | Product search |
| Core HR (62) | People search |
| Digital Assistant (43) | Conversational search |
| Frontend (20) | Search UI components |
