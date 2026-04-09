# 140 - Demand Management Service Specification

## 1. Domain Overview

Demand Management provides statistical demand forecasting, demand sensing, causal analysis, and collaborative demand planning. Supports multiple forecasting methods including exponential smoothing, ARIMA, machine learning-based models, and composite forecasts. Enables demand analysts to incorporate external factors (promotions, events, weather, economic indicators), perform what-if simulations, manage forecast accuracy metrics, and publish consensus demand plans to Supply Chain Planning. Integrates with Marketing for promotional uplift, Sales for pipeline-informed forecasts, and EPM for financial planning alignment.

**Bounded Context:** Demand Forecasting & Sensing
**Service Name:** `demand-mgmt-service`
**Database:** `data/demand_mgmt.db`
**HTTP Port:** 8220 | **gRPC Port:** 9220

---

## 2. Database Schema

### 2.1 Forecast Models
```sql
CREATE TABLE dm_forecast_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL CHECK(model_type IN (
        'EXPONENTIAL_SMOOTHING','ARIMA','CROSTON','HOLT_WINTERS',
        'ML_REGRESSION','ML_NEURAL_NET','ML_GRADIENT_BOOST',
        'COMPOSITE','BASELINE','CUSTOM'
    )),
    description TEXT,
    parameters TEXT NOT NULL,               -- JSON: model-specific hyperparameters
    target_granularity TEXT NOT NULL CHECK(target_granularity IN ('DAILY','WEEKLY','MONTHLY','QUARTERLY')),
    horizon_periods INTEGER NOT NULL DEFAULT 12,
    training_window_periods INTEGER NOT NULL DEFAULT 36,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','TRAINING','TRAINED','ACTIVE','DEPRECATED')),
    accuracy_mape REAL,                     -- Mean Absolute Percentage Error
    accuracy_rmse REAL,                     -- Root Mean Square Error
    accuracy_bias REAL,                     -- Forecast bias
    last_trained_at TEXT,
    training_data_points INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_dm_model_tenant ON dm_forecast_models(tenant_id, status);
```

### 2.2 Forecast Runs
```sql
CREATE TABLE dm_forecast_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    run_number TEXT NOT NULL,               -- FR-2024-00001
    run_type TEXT NOT NULL CHECK(run_type IN ('SCHEDULED','MANUAL','TRIGGERED','WHATIF')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED','CANCELLED')),
    started_at TEXT,
    completed_at TEXT,
    periods_forecasted INTEGER NOT NULL DEFAULT 0,
    items_forecasted INTEGER NOT NULL DEFAULT 0,
    avg_accuracy_mape REAL,
    execution_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES dm_forecast_models(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, run_number)
);

CREATE INDEX idx_dm_run_tenant ON dm_forecast_runs(tenant_id, created_at DESC);
CREATE INDEX idx_dm_run_status ON dm_forecast_runs(tenant_id, status);
```

### 2.3 Forecast Values
```sql
CREATE TABLE dm_forecast_values (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    location_id TEXT,
    customer_group_id TEXT,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    forecast_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    forecast_revenue_cents INTEGER NOT NULL DEFAULT 0,
    lower_bound DECIMAL(18,4),
    upper_bound DECIMAL(18,4),
    confidence_level REAL NOT NULL DEFAULT 0.95,
    baseline_forecast DECIMAL(18,4),        -- Without causal factors
    seasonal_index REAL NOT NULL DEFAULT 1.0,
    trend_component REAL NOT NULL DEFAULT 0,
    causal_adjustment DECIMAL(18,4) NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (run_id) REFERENCES dm_forecast_runs(id) ON DELETE CASCADE
);

CREATE INDEX idx_dm_fv_run_item ON dm_forecast_values(run_id, item_id);
CREATE INDEX idx_dm_fv_item_period ON dm_forecast_values(tenant_id, item_id, period_start);
CREATE INDEX idx_dm_fv_location ON dm_forecast_values(tenant_id, location_id, period_start);
```

### 2.4 Causal Factors
```sql
CREATE TABLE dm_causal_factors (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    factor_code TEXT NOT NULL,
    factor_name TEXT NOT NULL,
    factor_type TEXT NOT NULL CHECK(factor_type IN (
        'PROMOTION','EVENT','PRICE_CHANGE','WEATHER','ECONOMIC',
        'HOLIDAY','COMPETITOR_ACTION','PRODUCT_LAUNCH','DISCONTINUATION','CUSTOM'
    )),
    description TEXT,
    impact_model TEXT NOT NULL CHECK(impact_model IN ('MULTIPLICATIVE','ADDITIVE','PERCENTAGE')),
    default_impact REAL NOT NULL DEFAULT 0,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    applicable_items TEXT,                  -- JSON array of item IDs (empty = all)
    applicable_locations TEXT,              -- JSON array of location IDs
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ACTIVE','COMPLETED','CANCELLED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, factor_code)
);

CREATE INDEX idx_dm_causal_tenant ON dm_causal_factors(tenant_id, factor_type, status);
CREATE INDEX idx_dm_causal_dates ON dm_causal_factors(tenant_id, start_date, end_date);
```

### 2.5 Demand Sensing Signals
```sql
CREATE TABLE dm_demand_signals (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    signal_type TEXT NOT NULL CHECK(signal_type IN (
        'POS_DATA','WEB_TRAFFIC','SOCIAL_MEDIA','SEARCH_TREND',
        'WEATHER_ALERT','ECONOMIC_INDICATOR','COMPETITOR_PRICE','INVENTORY_LEVEL','CUSTOM'
    )),
    source TEXT NOT NULL,
    signal_date TEXT NOT NULL,
    item_id TEXT,
    location_id TEXT,
    signal_value REAL NOT NULL,
    signal_unit TEXT NOT NULL,              -- e.g., "units", "dollars", "index"
    confidence REAL NOT NULL DEFAULT 1.0,
    processed INTEGER NOT NULL DEFAULT 0,

    received_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, signal_type, source, signal_date, item_id, location_id)
);

CREATE INDEX idx_dm_signal_type_date ON dm_demand_signals(tenant_id, signal_type, signal_date DESC);
CREATE INDEX idx_dm_signal_item ON dm_demand_signals(tenant_id, item_id, signal_date DESC);
```

### 2.6 Consensus Demand Plans
```sql
CREATE TABLE dm_consensus_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_period TEXT NOT NULL,              -- e.g., "2024-Q1", "2024"
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','UNDER_REVIEW','APPROVED','PUBLISHED','ARCHIVED')),
    statistical_forecast_run_id TEXT,
    marketing_adjustments TEXT,             -- JSON: period-level adjustments
    sales_adjustments TEXT,                 -- JSON: period-level adjustments
    executive_adjustments TEXT,             -- JSON: period-level adjustments
    final_quantities TEXT,                  -- JSON: { item_id: { period: qty } }
    total_revenue_cents INTEGER NOT NULL DEFAULT 0,
    approved_by TEXT,
    approved_at TEXT,
    published_to_planning INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_name, plan_period)
);

CREATE INDEX idx_dm_consensus_tenant ON dm_consensus_plans(tenant_id, status);
```

### 2.7 Forecast Accuracy Tracking
```sql
CREATE TABLE dm_forecast_accuracy (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    location_id TEXT,
    period TEXT NOT NULL,
    forecast_quantity DECIMAL(18,4) NOT NULL,
    actual_quantity DECIMAL(18,4) NOT NULL,
    absolute_error DECIMAL(18,4) NOT NULL,
    percentage_error REAL NOT NULL,
    weighted_error REAL NOT NULL DEFAULT 0,
    model_id TEXT NOT NULL,
    run_id TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, item_id, location_id, period, model_id)
);

CREATE INDEX idx_dm_accuracy_item ON dm_forecast_accuracy(tenant_id, item_id, period DESC);
CREATE INDEX idx_dm_accuracy_model ON dm_forecast_accuracy(tenant_id, model_id, period DESC);
```

---

## 3. API Endpoints

### 3.1 Forecast Models
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/demand/models` | Create forecast model |
| GET | `/api/v1/demand/models` | List models |
| GET | `/api/v1/demand/models/{id}` | Get model details |
| PUT | `/api/v1/demand/models/{id}` | Update model |
| POST | `/api/v1/demand/models/{id}/train` | Train model on historical data |
| DELETE | `/api/v1/demand/models/{id}` | Deactivate model |

### 3.2 Forecast Runs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/demand/runs` | Execute forecast run |
| GET | `/api/v1/demand/runs` | List runs |
| GET | `/api/v1/demand/runs/{id}` | Get run details with results |
| GET | `/api/v1/demand/runs/{id}/values` | Get forecast values |
| POST | `/api/v1/demand/runs/{id}/cancel` | Cancel running forecast |
| POST | `/api/v1/demand/runs/compare` | Compare multiple runs |

### 3.3 Causal Factors
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/demand/causal-factors` | Create causal factor |
| GET | `/api/v1/demand/causal-factors` | List causal factors |
| PUT | `/api/v1/demand/causal-factors/{id}` | Update factor |
| POST | `/api/v1/demand/causal-factors/apply` | Apply factors to forecast |

### 3.4 Demand Sensing
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/demand/signals` | Ingest demand signal |
| POST | `/api/v1/demand/signals/bulk` | Bulk ingest signals |
| GET | `/api/v1/demand/signals` | Query signals |
| POST | `/api/v1/demand/sensing/analyze` | Run demand sensing analysis |

### 3.5 Consensus Planning
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/demand/consensus-plans` | Create consensus plan |
| GET | `/api/v1/demand/consensus-plans` | List plans |
| PUT | `/api/v1/demand/consensus-plans/{id}/adjust` | Submit adjustments |
| POST | `/api/v1/demand/consensus-plans/{id}/approve` | Approve plan |
| POST | `/api/v1/demand/consensus-plans/{id}/publish` | Publish to SCP |

### 3.6 Accuracy & Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/demand/accuracy` | Get forecast accuracy metrics |
| GET | `/api/v1/demand/accuracy/by-model` | Compare model accuracy |
| GET | `/api/v1/demand/accuracy/by-item` | Item-level accuracy |
| GET | `/api/v1/demand/accuracy/trend` | Accuracy trend over time |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `demand.forecast.completed` | `{ run_id, model_id, items_count, avg_mape }` | Forecast run completed |
| `demand.forecast.failed` | `{ run_id, model_id, error }` | Forecast run failed |
| `demand.signal.anomaly` | `{ signal_type, item_id, deviation_pct }` | Demand signal anomaly detected |
| `demand.consensus.approved` | `{ plan_id, total_revenue, period }` | Consensus plan approved |
| `demand.consensus.published` | `{ plan_id, published_to }` | Plan published to planning system |
| `demand.accuracy.alert` | `{ item_id, mape, threshold }` | Accuracy fell below threshold |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `order.completed` | Order Management | Feed actual demand data |
| `invoice.posted` | AR | Revenue actuals for accuracy tracking |
| `promotion.scheduled` | Marketing | Create causal factor |
| `item.launched` | Product Hub | New item forecast initialization |
| `price.changed` | Advanced Pricing | Price impact causal factor |

---

## 5. Business Rules

1. **Model Selection**: Auto-select best-fit model using tournament approach across multiple methods
2. **Accuracy Threshold**: Alert when MAPE exceeds 25% for any item over 3 consecutive periods
3. **Causal Override**: Causal factors can override statistical forecast by max 200% (configurable)
4. **Consensus Process**: Statistical + Marketing + Sales + Executive adjustments; final = weighted blend
5. **Signal Processing**: Demand signals processed within 1 hour of ingestion for near-real-time sensing
6. **Model Retraining**: Auto-retrain monthly or when accuracy degrades below threshold
7. **Cold Start**: New items use category-level forecast for first 3 periods until item-level data available
8. **Horizon Defaults**: Monthly granularity uses 12-month horizon; weekly uses 52-week horizon

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Supply Chain Planning (28) | Published consensus demand plans feed supply planning |
| Marketing (61) | Promotional calendars drive causal factors |
| Order Management (13) | Actual order data for accuracy measurement |
| Product Hub (51) | New item introductions, lifecycle changes |
| Advanced Pricing (27) | Price change impacts on demand |
| EPM (33) | Demand plan feeds financial budget |
| AI/ML Platform (35) | ML model training, hyperparameter optimization |
| Inventory (12) | Inventory signals for demand sensing |
