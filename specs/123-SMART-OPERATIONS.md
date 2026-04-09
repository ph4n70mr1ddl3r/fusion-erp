# 123 - Smart Operations Specification

## 1. Domain Overview

Smart Operations provides IoT-enabled operational intelligence for manufacturing and asset-intensive industries. It combines real-time machine telemetry, predictive maintenance algorithms, and operational dashboards to optimize equipment effectiveness, reduce unplanned downtime, and improve Overall Equipment Effectiveness (OEE).

**Bounded Context:** Smart Operations & Predictive Maintenance
**Service Name:** `smartops-service`
**Database:** `data/smartops.db`
**HTTP Port:** 8203 | **gRPC Port:** 9203

---

## 2. Database Schema

### 2.1 Equipment Models
```sql
CREATE TABLE so_equipment_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_code TEXT NOT NULL,
    manufacturer TEXT,
    model_number TEXT,
    description TEXT,
    expected_lifetime_hours INTEGER,
    criticality TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(criticality IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    maintenance_strategy TEXT NOT NULL DEFAULT 'PREVENTIVE'
        CHECK(maintenance_strategy IN ('REACTIVE','PREVENTIVE','PREDICTIVE','CONDITION_BASED')),
    oee_target_pct REAL NOT NULL DEFAULT 85.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_code)
);
```

### 2.2 Equipment Instances
```sql
CREATE TABLE so_equipment_instances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    equipment_model_id TEXT NOT NULL,
    equipment_name TEXT NOT NULL,
    equipment_code TEXT NOT NULL,
    location TEXT,
    department TEXT,
    production_line TEXT,
    serial_number TEXT,
    installation_date TEXT,
    commissioning_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','IDLE','MAINTENANCE','DECOMMISSIONED')),
    operational_status TEXT NOT NULL DEFAULT 'RUNNING'
        CHECK(operational_status IN ('RUNNING','IDLE','DOWN','SETUP')),
    current_runtime_hours REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (equipment_model_id) REFERENCES so_equipment_models(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, equipment_code)
);

CREATE INDEX idx_so_equipment_tenant_status ON so_equipment_instances(tenant_id, status);
```

### 2.3 Telemetry Streams
```sql
CREATE TABLE so_telemetry_streams (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    equipment_id TEXT NOT NULL,
    stream_name TEXT NOT NULL,
    metric_type TEXT NOT NULL CHECK(metric_type IN ('TEMPERATURE','VIBRATION','PRESSURE','FLOW','SPEED','POWER','HUMIDITY','CUSTOM')),
    unit TEXT NOT NULL,
    sampling_interval_seconds INTEGER NOT NULL DEFAULT 60,
    normal_min REAL,
    normal_max REAL,
    warning_threshold REAL,
    critical_threshold REAL,
    is_monitored INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (equipment_id) REFERENCES so_equipment_instances(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, equipment_id, stream_name)
);
```

### 2.4 Telemetry Readings
```sql
CREATE TABLE so_telemetry_readings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    stream_id TEXT NOT NULL,
    equipment_id TEXT NOT NULL,
    reading_timestamp TEXT NOT NULL,
    value REAL NOT NULL,
    quality TEXT NOT NULL DEFAULT 'GOOD'
        CHECK(quality IN ('GOOD','UNCERTAIN','BAD')),
    is_anomaly INTEGER NOT NULL DEFAULT 0,
    anomaly_score REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (stream_id) REFERENCES so_telemetry_streams(id) ON DELETE CASCADE
);

CREATE INDEX idx_so_readings_tenant_stream ON so_telemetry_readings(tenant_id, stream_id, reading_timestamp DESC);
CREATE INDEX idx_so_readings_tenant_anomaly ON so_telemetry_readings(tenant_id, is_anomaly, reading_timestamp DESC);
```

### 2.5 OEE Records
```sql
CREATE TABLE so_oee_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    equipment_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    availability_pct REAL NOT NULL,
    performance_pct REAL NOT NULL,
    quality_pct REAL NOT NULL,
    oee_pct REAL NOT NULL,
    planned_production_minutes INTEGER NOT NULL,
    actual_production_minutes INTEGER NOT NULL,
    downtime_minutes INTEGER NOT NULL DEFAULT 0,
    ideal_cycle_time_seconds REAL NOT NULL,
    total_pieces INTEGER NOT NULL,
    good_pieces INTEGER NOT NULL,
    defect_pieces INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, equipment_id, period_start, period_end)
);

CREATE INDEX idx_so_oee_tenant_equipment ON so_oee_records(tenant_id, equipment_id, period_start DESC);
```

### 2.6 Predictive Models
```sql
CREATE TABLE so_predictive_models (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_name TEXT NOT NULL,
    model_type TEXT NOT NULL CHECK(model_type IN ('FAILURE_PREDICTION','REMAINING_USEFUL_LIFE','ANOMALY_DETECTION','QUALITY_PREDICTION')),
    equipment_model_id TEXT,
    description TEXT,
    algorithm TEXT NOT NULL,             -- 'ISOLATION_FOREST', 'LSTM', 'PROPHET', etc.
    feature_streams TEXT NOT NULL,       -- JSON array of stream_ids used as features
    parameters TEXT,                     -- JSON: model hyperparameters
    training_date TEXT,
    accuracy_score REAL,
    status TEXT NOT NULL DEFAULT 'TRAINING'
        CHECK(status IN ('TRAINING','VALIDATING','ACTIVE','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, model_name)
);
```

### 2.7 Predictions & Alerts
```sql
CREATE TABLE so_predictions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    model_id TEXT NOT NULL,
    equipment_id TEXT NOT NULL,
    prediction_type TEXT NOT NULL,
    predicted_value REAL,
    predicted_date TEXT,
    confidence_pct REAL NOT NULL,
    severity TEXT NOT NULL DEFAULT 'LOW'
        CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    description TEXT NOT NULL,
    recommended_action TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','ACKNOWLEDGED','ACTION_TAKEN','DISMISSED','EXPIRED')),
    acknowledged_by TEXT,
    acknowledged_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (model_id) REFERENCES so_predictive_models(id) ON DELETE RESTRICT,
    FOREIGN KEY (equipment_id) REFERENCES so_equipment_instances(id) ON DELETE RESTRICT
);

CREATE INDEX idx_so_predictions_tenant_status ON so_predictions(tenant_id, status, severity);
CREATE INDEX idx_so_predictions_tenant_equipment ON so_predictions(tenant_id, equipment_id);
```

---

## 3. REST API Endpoints

### 3.1 Equipment Management
```
GET    /api/v1/smartops/equipment                       Permission: so.equipment.read
GET    /api/v1/smartops/equipment/{id}                  Permission: so.equipment.read
POST   /api/v1/smartops/equipment                       Permission: so.equipment.create
PUT    /api/v1/smartops/equipment/{id}                  Permission: so.equipment.update
GET    /api/v1/smartops/equipment/{id}/status            Permission: so.equipment.read
PUT    /api/v1/smartops/equipment/{id}/status            Permission: so.equipment.operate
```

### 3.2 Telemetry
```
GET    /api/v1/smartops/equipment/{id}/streams           Permission: so.telemetry.read
POST   /api/v1/smartops/equipment/{id}/streams           Permission: so.telemetry.configure
GET    /api/v1/smartops/streams/{id}/readings
  ?from=2024-01-01T00:00:00Z&to=2024-01-31T23:59:59Z
  ?resolution=raw|1m|5m|1h|1d                            Permission: so.telemetry.read
POST   /api/v1/smartops/streams/{id}/readings            Permission: so.telemetry.ingest
  Request: { "readings": [{ "timestamp": "...", "value": 72.5 }] }
POST   /api/v1/smartops/streams/{id}/readings/batch      Permission: so.telemetry.ingest
GET    /api/v1/smartops/streams/{id}/latest              Permission: so.telemetry.read
```

### 3.3 OEE
```
GET    /api/v1/smartops/oee
  ?equipment_id={id}
  ?period_start=2024-01-01&period_end=2024-01-31
  ?group_by=day|week|month                              Permission: so.oee.read
GET    /api/v1/smartops/oee/dashboard
  ?department={dept}&production_line={line}              Permission: so.oee.read
  Response: { "data": { "overall_oee": 78.5, "equipment_oee": [...], "trends": [...] } }
POST   /api/v1/smartops/oee/calculate                    Permission: so.oee.calculate
```

### 3.4 Predictive Analytics
```
GET    /api/v1/smartops/models                           Permission: so.models.read
POST   /api/v1/smartops/models                           Permission: so.models.create
POST   /api/v1/smartops/models/{id}/train                Permission: so.models.train
POST   /api/v1/smartops/models/{id}/activate             Permission: so.models.activate
GET    /api/v1/smartops/predictions                      Permission: so.predictions.read
GET    /api/v1/smartops/predictions/{id}                 Permission: so.predictions.read
PUT    /api/v1/smartops/predictions/{id}/acknowledge     Permission: so.predictions.acknowledge
PUT    /api/v1/smartops/predictions/{id}/dismiss         Permission: so.predictions.dismiss
GET    /api/v1/smartops/equipment/{id}/health            Permission: so.equipment.read
  Response: { "data": { "health_score": 87.5, "anomalies": [...], "predictions": [...] } }
```

---

## 4. Business Rules

### 4.1 Telemetry Ingestion
- Readings are validated against stream normal ranges
- Values outside normal_min/normal_max trigger threshold warnings
- Values beyond critical_threshold trigger immediate alerts
- Anomaly detection runs on ingested data using active predictive models
- Readings with quality='BAD' are flagged but stored for audit

### 4.2 OEE Calculation
- OEE = Availability × Performance × Quality
- Availability = Actual Production Time / Planned Production Time
- Performance = (Total Pieces × Ideal Cycle Time) / Actual Production Time
- Quality = Good Pieces / Total Pieces
- OEE is recalculated when equipment status changes or at shift boundaries
- Target OEE from equipment model used for comparison dashboards

### 4.3 Predictive Maintenance
- Models trained on historical telemetry and maintenance records
- Remaining Useful Life (RUL) predicts days until failure
- Anomaly detection uses isolation forest or similar algorithms
- Predictions with confidence > 80% and severity HIGH/CRITICAL create maintenance work orders
- False positive rate tracked per model; models retrained quarterly

### 4.4 Alerting
- Alerts generated for: threshold violations, anomaly detection, prediction triggers
- Alert severity determines notification channel (email, SMS, push, pager)
- Alerts MUST be acknowledged within SLA timeframes
- Unacknowledged CRITICAL alerts escalate after 15 minutes

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.smartops.v1;

service SmartOperationsService {
    rpc IngestTelemetry(IngestTelemetryRequest) returns (IngestTelemetryResponse);
    rpc GetEquipmentHealth(GetEquipmentHealthRequest) returns (GetEquipmentHealthResponse);
    rpc GetOEE(GetOEERequest) returns (GetOEEResponse);
    rpc GetPredictions(GetPredictionsRequest) returns (GetPredictionsResponse);
    rpc RunAnomalyDetection(RunAnomalyDetectionRequest) returns (RunAnomalyDetectionResponse);
}

// Entity messages
message SoEquipmentInstance {
    string id = 1;
    string tenant_id = 2;
    string equipment_model_id = 3;
    string equipment_name = 4;
    string equipment_code = 5;
    string location = 6;
    string department = 7;
    string production_line = 8;
    string serial_number = 9;
    string installation_date = 10;
    string commissioning_date = 11;
    string status = 12;
    string operational_status = 13;
    double current_runtime_hours = 14;
    string created_at = 15;
    string updated_at = 16;
}

message SoTelemetryReading {
    string id = 1;
    string tenant_id = 2;
    string stream_id = 3;
    string equipment_id = 4;
    string reading_timestamp = 5;
    double value = 6;
    string quality = 7;
    bool is_anomaly = 8;
    double anomaly_score = 9;
    string created_at = 10;
}

message SoOeeRecord {
    string id = 1;
    string tenant_id = 2;
    string equipment_id = 3;
    string period_start = 4;
    string period_end = 5;
    double availability_pct = 6;
    double performance_pct = 7;
    double quality_pct = 8;
    double oee_pct = 9;
    int32 planned_production_minutes = 10;
    int32 actual_production_minutes = 11;
    int32 downtime_minutes = 12;
    double ideal_cycle_time_seconds = 13;
    int32 total_pieces = 14;
    int32 good_pieces = 15;
    int32 defect_pieces = 16;
    string created_at = 17;
    string updated_at = 18;
}

message SoPrediction {
    string id = 1;
    string tenant_id = 2;
    string model_id = 3;
    string equipment_id = 4;
    string prediction_type = 5;
    double predicted_value = 6;
    string predicted_date = 7;
    double confidence_pct = 8;
    string severity = 9;
    string description = 10;
    string recommended_action = 11;
    string status = 12;
    string acknowledged_by = 13;
    string acknowledged_at = 14;
    string created_at = 15;
    string updated_at = 16;
}

// Request/Response messages
message TelemetryReadingInput {
    string timestamp = 1;
    double value = 2;
}

message IngestTelemetryRequest {
    string tenant_id = 1;
    string stream_id = 2;
    string equipment_id = 3;
    repeated TelemetryReadingInput readings = 4;
}

message IngestTelemetryResponse {
    int32 accepted_count = 1;
    int32 anomaly_count = 2;
    repeated SoTelemetryReading anomalies = 3;
}

message GetEquipmentHealthRequest {
    string tenant_id = 1;
    string equipment_id = 2;
}

message HealthAnomaly {
    string stream_id = 1;
    string stream_name = 2;
    double value = 3;
    double anomaly_score = 4;
    string reading_timestamp = 5;
}

message GetEquipmentHealthResponse {
    SoEquipmentInstance equipment = 1;
    double health_score = 2;
    repeated HealthAnomaly anomalies = 3;
    repeated SoPrediction predictions = 4;
}

message GetOEERequest {
    string tenant_id = 1;
    string equipment_id = 2;
    string period_start = 3;
    string period_end = 4;
    string group_by = 5;
}

message GetOEEResponse {
    repeated SoOeeRecord records = 1;
    double overall_oee = 2;
}

message GetPredictionsRequest {
    string tenant_id = 1;
    string equipment_id = 2;
    string severity = 3;
    string status = 4;
    int32 limit = 5;
}

message GetPredictionsResponse {
    repeated SoPrediction predictions = 1;
}

message RunAnomalyDetectionRequest {
    string tenant_id = 1;
    string equipment_id = 2;
    repeated string stream_ids = 3;
    string analysis_window_start = 4;
    string analysis_window_end = 5;
}

message AnomalyResult {
    string stream_id = 1;
    string reading_id = 2;
    double value = 3;
    double anomaly_score = 4;
    string reading_timestamp = 5;
}

message RunAnomalyDetectionResponse {
    repeated AnomalyResult anomalies = 1;
    int32 total_readings_analyzed = 2;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **iot-service**: Telemetry data ingestion pipeline
- **eam-service**: Equipment registry, maintenance work orders
- **mfg-service**: Production count data for OEE
- **quality-service**: Defect counts for OEE quality calculation
- **ai-service**: ML model training, inference execution
- **prodsched-service**: Equipment downtime scheduling

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `so.telemetry.threshold.exceeded` | Reading exceeds threshold | stream_id, value, threshold |
| `so.telemetry.anomaly.detected` | Anomaly detected | equipment_id, stream_id, score |
| `so.oee.calculated` | OEE record computed | equipment_id, oee_pct, period |
| `so.prediction.created` | New prediction generated | equipment_id, type, severity |
| `so.equipment.status.changed` | Equipment operational status change | equipment_id, old_status, new_status |
| `so.maintenance.recommended` | Predictive model recommends maintenance | equipment_id, work_order_id |

---

## 7. Migrations

### Migration Order for smartops-service:
1. V001: `so_equipment_models`
2. V002: `so_equipment_instances`
3. V003: `so_telemetry_streams`
4. V004: `so_telemetry_readings`
5. V005: `so_oee_records`
6. V006: `so_predictive_models`
7. V007: `so_predictions`
8. V008: Triggers for `updated_at`
9. V009: Seed data (sample equipment models, OEE targets)
