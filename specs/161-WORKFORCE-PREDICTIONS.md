# 161 - Workforce Predictions Service Specification

## 1. Domain Overview

Workforce Predictions provides AI/ML-powered predictive analytics for workforce planning and management. Includes attrition prediction, performance trajectory modeling, promotion readiness scoring, skill obsolescence forecasting, and workforce demand prediction. Supports scenario modeling for organizational changes, merger integrations, and strategic workforce planning. Integrates with HCM Analytics for historical data, Core HR for employee attributes, and AI/ML Platform for model management.

**Bounded Context:** AI-Powered Workforce Predictions
**Service Name:** `workforce-predictions-service`
**Database:** `data/workforce_predictions.db`
**HTTP Port:** 8179 | **gRPC Port:** 9179

---

## 2. Database Schema

### 2.1 Prediction Models
```sql
CREATE TABLE wp_prediction_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_code TEXT NOT NULL,
    model_name TEXT NOT NULL,
    prediction_type TEXT NOT NULL CHECK(prediction_type IN (
        'ATTRITION','PERFORMANCE_TRAJECTORY','PROMOTION_READINESS',
        'SKILL_OBSOLESCENCE','DEMAND_FORECAST','ENGAGEMENT','FLIGHT_RISK','CUSTOM'
    )),
    model_algorithm TEXT NOT NULL CHECK(model_algorithm IN (
        'RANDOM_FOREST','GRADIENT_BOOST','NEURAL_NETWORK','LOGISTIC_REGRESSION','ENSEMBLE','CUSTOM'
    )),
    feature_config TEXT NOT NULL,           -- JSON: feature definitions and sources
    hyperparameters TEXT NOT NULL,          -- JSON: model hyperparameters
    training_config TEXT NOT NULL,          -- JSON: training window, validation split
    prediction_horizon TEXT NOT NULL,       -- "6_MONTHS","12_MONTHS"
    accuracy_metrics TEXT,                  -- JSON: AUC, precision, recall, F1
    last_trained_at TEXT,
    training_data_size INTEGER,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','TRAINING','TRAINED','ACTIVE','DEPRECATED')),
    refresh_frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(refresh_frequency IN ('WEEKLY','MONTHLY','QUARTERLY')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);

CREATE INDEX idx_wp_model_tenant ON wp_prediction_models(tenant_id, prediction_type, status);
```

### 2.2 Prediction Runs
```sql
CREATE TABLE wp_prediction_runs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    run_number TEXT NOT NULL,               -- WP-RUN-2024-00001
    run_type TEXT NOT NULL CHECK(run_type IN ('SCHEDULED','MANUAL','SCENARIO')),
    scenario_id TEXT,                       -- For scenario-based predictions
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','RUNNING','COMPLETED','FAILED')),
    prediction_date TEXT NOT NULL,
    prediction_horizon_months INTEGER NOT NULL DEFAULT 6,
    employees_scored INTEGER NOT NULL DEFAULT 0,
    started_at TEXT,
    completed_at TEXT,
    execution_time_ms INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    FOREIGN KEY (model_id) REFERENCES wp_prediction_models(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, run_number)
);

CREATE INDEX idx_wp_run_model ON wp_prediction_runs(model_id, prediction_date DESC);
```

### 2.3 Individual Predictions
```sql
CREATE TABLE wp_individual_predictions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    prediction_type TEXT NOT NULL,
    predicted_outcome TEXT NOT NULL,        -- HIGH/MEDIUM/LOW risk or score
    probability REAL NOT NULL DEFAULT 0,    -- 0-1 confidence score
    risk_score REAL NOT NULL DEFAULT 0,     -- Normalized 0-100
    key_factors TEXT NOT NULL,              -- JSON: top contributing factors
    recommended_actions TEXT,               -- JSON: AI-recommended interventions
    predicted_timeline TEXT,                -- When the predicted event is likely
    confidence_interval_lower REAL,
    confidence_interval_upper REAL,
    model_version TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (run_id) REFERENCES wp_prediction_runs(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, run_id, person_id)
);

CREATE INDEX idx_wp_pred_person ON wp_individual_predictions(person_id, prediction_type);
CREATE INDEX idx_wp_pred_risk ON wp_individual_predictions(tenant_id, risk_score DESC);
CREATE INDEX idx_wp_pred_type ON wp_individual_predictions(prediction_type, predicted_outcome);
```

### 2.4 Aggregated Predictions
```sql
CREATE TABLE wp_aggregated_predictions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    run_id TEXT NOT NULL,
    dimension_type TEXT NOT NULL CHECK(dimension_type IN ('DEPARTMENT','LOCATION','JOB_FAMILY','GRADE','TENURE_BAND')),
    dimension_value TEXT NOT NULL,
    prediction_type TEXT NOT NULL,
    population_size INTEGER NOT NULL DEFAULT 0,
    high_risk_count INTEGER NOT NULL DEFAULT 0,
    medium_risk_count INTEGER NOT NULL DEFAULT 0,
    low_risk_count INTEGER NOT NULL DEFAULT 0,
    avg_risk_score REAL NOT NULL DEFAULT 0,
    predicted_attrition_count INTEGER NOT NULL DEFAULT 0,
    predicted_attrition_pct REAL NOT NULL DEFAULT 0,
    estimated_replacement_cost_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (run_id) REFERENCES wp_prediction_runs(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, run_id, dimension_type, dimension_value)
);

CREATE INDEX idx_wp_agg_run ON wp_aggregated_predictions(run_id);
CREATE INDEX idx_wp_agg_type ON wp_aggregated_predictions(tenant_id, dimension_type, avg_risk_score DESC);
```

### 2.5 Scenario Models
```sql
CREATE TABLE wp_scenarios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    scenario_name TEXT NOT NULL,
    scenario_type TEXT NOT NULL CHECK(scenario_type IN (
        'REORG','MERGER','LAYOFF','EXPANSION','SKILL_SHIFT','AUTOMATION','CUSTOM'
    )),
    description TEXT,
    assumptions TEXT NOT NULL,              -- JSON: scenario parameters
    model_ids TEXT NOT NULL,               -- JSON: prediction models to run
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','RUNNING','COMPLETED','ARCHIVED')),
    results_summary TEXT,                   -- JSON: key findings

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, scenario_name)
);
```

---

## 3. API Endpoints

### 3.1 Prediction Models
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/workforce-predictions/models` | Create prediction model |
| GET | `/api/v1/workforce-predictions/models` | List models |
| GET | `/api/v1/workforce-predictions/models/{id}` | Get model details |
| POST | `/api/v1/workforce-predictions/models/{id}/train` | Train model |
| PUT | `/api/v1/workforce-predictions/models/{id}` | Update model |

### 3.2 Predictions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/workforce-predictions/runs` | Execute prediction run |
| GET | `/api/v1/workforce-predictions/runs` | List runs |
| GET | `/api/v1/workforce-predictions/runs/{id}` | Get run results |
| GET | `/api/v1/workforce-predictions/runs/{id}/individuals` | Individual predictions |
| GET | `/api/v1/workforce-predictions/runs/{id}/aggregated` | Aggregated predictions |
| GET | `/api/v1/workforce-predictions/person/{personId}/predictions` | Person's predictions |

### 3.3 Scenarios
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/workforce-predictions/scenarios` | Create scenario |
| GET | `/api/v1/workforce-predictions/scenarios` | List scenarios |
| POST | `/api/v1/workforce-predictions/scenarios/{id}/run` | Run scenario predictions |
| GET | `/api/v1/workforce-predictions/scenarios/{id}/results` | Get scenario results |

### 3.4 Insights
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/workforce-predictions/insights/attrition-risk` | Attrition risk dashboard |
| GET | `/api/v1/workforce-predictions/insights/flight-risk` | Flight risk report |
| GET | `/api/v1/workforce-predictions/insights/skill-gaps` | Predicted skill gaps |
| GET | `/api/v1/workforce-predictions/insights/promotion-pipeline` | Promotion readiness |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `wpredict.run.completed` | `{ run_id, model_id, scored_count }` | Prediction run completed |
| `wpredict.high_risk.identified` | `{ person_id, risk_type, score }` | High-risk individual flagged |
| `wpredict.skill.obsolescence` | `{ skill, affected_count, timeline }` | Skill obsolescence predicted |
| `wpredict.scenario.completed` | `{ scenario_id, findings }` | Scenario analysis completed |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `employee.data.updated` | Core HR | Trigger model retraining |
| `performance.review.completed` | Performance Mgmt | Update training data |
| `employee.terminated` | Core HR | Validate attrition predictions |

---

## 5. Business Rules

1. **Data Privacy**: Individual predictions visible only to authorized HR roles; employee self-view limited
2. **Bias Monitoring**: Models monitored for demographic bias; alerts on fairness threshold breaches
3. **Model Accuracy**: Models auto-retrained when accuracy drops below 70% AUC
4. **Prediction Freshness**: Predictions refreshed at configured frequency; timestamp always shown
5. **Explainability**: All predictions include key contributing factors for transparency
6. **Scenario Isolation**: Scenario predictions don't affect individual risk scores
7. **Action Tracking**: Recommended actions tracked to completion; outcomes feed back to model

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Core HR (62) | Employee attributes, lifecycle events |
| Performance Management (68) | Performance history |
| HCM Analytics (148) | Historical workforce metrics |
| AI/ML Platform (35) | Model training infrastructure |
| Talent Review (146) | Calibration and talent assessment |
| Succession Planning (70) | Succession readiness |
| Career Development (71) | Development plan data |
| Strategic Workforce Planning (119) | Long-term workforce models |
