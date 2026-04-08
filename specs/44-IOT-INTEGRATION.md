# 44 - IoT Integration Service Specification

## 1. Domain Overview

IoT Integration provides comprehensive device management, sensor data ingestion, telemetry processing, edge computing configuration, and real-time alerting for connected operational assets. It registers and manages IoT devices (sensors, gateways, actuators, edge devices) across multiple protocols (MQTT, HTTP, CoAP, Modbus, OPC UA), ingests high-volume telemetry streams, computes aggregations, evaluates alert rules in real time, issues device commands, and supports edge-level data filtering and ML inference. Integrates with EAM for asset linkage, Inventory for sensor-tracked stock, AI Service for anomaly pattern detection, and Workflow for alert escalation.

**Bounded Context:** IoT Device Management, Sensor Data & Edge Integration
**Service Name:** `iot-service`
**Database:** `data/iot.db`
**HTTP Port:** 8071 | **gRPC Port:** 9071

---

## 2. Database Schema

### 2.1 IoT Device Registrations
```sql
CREATE TABLE iot_device_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    device_code TEXT NOT NULL,
    device_name TEXT NOT NULL,
    device_type TEXT NOT NULL
        CHECK(device_type IN ('SENSOR','GATEWAY','ACTUATOR','EDGE_DEVICE')),
    protocol TEXT NOT NULL
        CHECK(protocol IN ('MQTT','HTTP','COAP','MODBUS','OPC_UA')),
    manufacturer TEXT,
    model TEXT,
    firmware_version TEXT,
    serial_number TEXT,
    location_id TEXT NOT NULL,
    asset_id TEXT,                              -- FK to eam-service operating asset
    status TEXT NOT NULL DEFAULT 'REGISTERED'
        CHECK(status IN ('REGISTERED','ACTIVE','INACTIVE','DECOMMISSIONED')),
    last_heartbeat TEXT,
    registration_date TEXT NOT NULL DEFAULT (datetime('now')),
    certificates TEXT,                          -- JSON: TLS/security certificates
    config TEXT,                                -- JSON: device-specific configuration

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, device_code)
);

CREATE INDEX idx_devices_tenant ON iot_device_registrations(tenant_id);
CREATE INDEX idx_devices_tenant_type ON iot_device_registrations(tenant_id, device_type);
CREATE INDEX idx_devices_tenant_status ON iot_device_registrations(tenant_id, status);
CREATE INDEX idx_devices_tenant_location ON iot_device_registrations(tenant_id, location_id);
CREATE INDEX idx_devices_tenant_asset ON iot_device_registrations(tenant_id, asset_id);
CREATE INDEX idx_devices_tenant_protocol ON iot_device_registrations(tenant_id, protocol);
```

### 2.2 IoT Device Groups
```sql
CREATE TABLE iot_device_groups (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    group_name TEXT NOT NULL,
    group_type TEXT NOT NULL
        CHECK(group_type IN ('LOCATION','ASSET_TYPE','FUNCTION')),
    description TEXT,
    parent_group_id TEXT,                       -- For hierarchical grouping
    device_count INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_groups_tenant ON iot_device_groups(tenant_id);
CREATE INDEX idx_groups_tenant_type ON iot_device_groups(tenant_id, group_type);
CREATE INDEX idx_groups_tenant_parent ON iot_device_groups(tenant_id, parent_group_id);
```

### 2.3 IoT Device Group Members
```sql
CREATE TABLE iot_device_group_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    group_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    added_at TEXT NOT NULL DEFAULT (datetime('now')),
    added_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (group_id) REFERENCES iot_device_groups(id) ON DELETE CASCADE,
    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE,

    UNIQUE(tenant_id, group_id, device_id)
);

CREATE INDEX idx_group_members_tenant ON iot_device_group_members(tenant_id);
CREATE INDEX idx_group_members_group ON iot_device_group_members(tenant_id, group_id);
CREATE INDEX idx_group_members_device ON iot_device_group_members(tenant_id, device_id);
```

### 2.4 IoT Data Streams
```sql
CREATE TABLE iot_data_streams (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    stream_name TEXT NOT NULL,
    device_id TEXT NOT NULL,
    data_type TEXT NOT NULL
        CHECK(data_type IN ('TEMPERATURE','HUMIDITY','PRESSURE','VIBRATION','FLOW','COUNTER','GPS','BOOLEAN','CUSTOM')),
    unit_of_measure TEXT,
    sampling_interval_seconds INTEGER NOT NULL DEFAULT 60,
    aggregation_type TEXT NOT NULL DEFAULT 'NONE'
        CHECK(aggregation_type IN ('NONE','AVG','MAX','MIN','SUM')),
    retention_days INTEGER NOT NULL DEFAULT 90,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE,

    UNIQUE(tenant_id, stream_name)
);

CREATE INDEX idx_streams_tenant ON iot_data_streams(tenant_id);
CREATE INDEX idx_streams_tenant_device ON iot_data_streams(tenant_id, device_id);
CREATE INDEX idx_streams_tenant_type ON iot_data_streams(tenant_id, data_type);
CREATE INDEX idx_streams_tenant_active ON iot_data_streams(tenant_id, is_active);
```

### 2.5 IoT Telemetry Raw
```sql
CREATE TABLE iot_telemetry_raw (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    stream_id TEXT NOT NULL,
    timestamp TEXT NOT NULL,
    value_real REAL,
    value_text TEXT,
    value_boolean INTEGER,
    quality TEXT NOT NULL DEFAULT 'GOOD'
        CHECK(quality IN ('GOOD','UNCERTAIN','BAD')),
    ingestion_time TEXT NOT NULL DEFAULT (datetime('now')),
    source_protocol TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE,
    FOREIGN KEY (stream_id) REFERENCES iot_data_streams(id) ON DELETE CASCADE
);

CREATE INDEX idx_telemetry_raw_tenant ON iot_telemetry_raw(tenant_id);
CREATE INDEX idx_telemetry_raw_tenant_device ON iot_telemetry_raw(tenant_id, device_id);
CREATE INDEX idx_telemetry_raw_tenant_stream ON iot_telemetry_raw(tenant_id, stream_id);
CREATE INDEX idx_telemetry_raw_tenant_timestamp ON iot_telemetry_raw(tenant_id, timestamp);
CREATE INDEX idx_telemetry_raw_device_timestamp ON iot_telemetry_raw(device_id, timestamp);
CREATE INDEX idx_telemetry_raw_stream_timestamp ON iot_telemetry_raw(stream_id, timestamp);
CREATE INDEX idx_telemetry_raw_quality ON iot_telemetry_raw(tenant_id, quality);
```

### 2.6 IoT Telemetry Aggregates
```sql
CREATE TABLE iot_telemetry_aggregates (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    stream_id TEXT NOT NULL,
    aggregation_period TEXT NOT NULL
        CHECK(aggregation_period IN ('MINUTE','HOUR','DAY')),
    period_start TEXT NOT NULL,
    avg_value REAL,
    min_value REAL,
    max_value REAL,
    sum_value REAL,
    sample_count INTEGER NOT NULL DEFAULT 0,
    std_dev REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE,
    FOREIGN KEY (stream_id) REFERENCES iot_data_streams(id) ON DELETE CASCADE,

    UNIQUE(tenant_id, stream_id, aggregation_period, period_start)
);

CREATE INDEX idx_telemetry_agg_tenant ON iot_telemetry_aggregates(tenant_id);
CREATE INDEX idx_telemetry_agg_tenant_device ON iot_telemetry_aggregates(tenant_id, device_id);
CREATE INDEX idx_telemetry_agg_tenant_stream ON iot_telemetry_aggregates(tenant_id, stream_id);
CREATE INDEX idx_telemetry_agg_tenant_period ON iot_telemetry_aggregates(tenant_id, aggregation_period, period_start);
CREATE INDEX idx_telemetry_agg_stream_period ON iot_telemetry_aggregates(stream_id, aggregation_period, period_start);
```

### 2.7 IoT Alert Rules
```sql
CREATE TABLE iot_alert_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    device_id TEXT,                             -- Nullable: null means apply to all devices in group
    stream_id TEXT NOT NULL,
    condition_type TEXT NOT NULL
        CHECK(condition_type IN ('THRESHOLD','RATE_OF_CHANGE','ABSENCE','PATTERN')),
    condition_config TEXT NOT NULL,             -- JSON: threshold values, rate limits, patterns
    severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    action_type TEXT NOT NULL
        CHECK(action_type IN ('NOTIFICATION','WEBHOOK','COMMAND','EAM_WORK_ORDER')),
    action_config TEXT,                         -- JSON: action-specific configuration
    is_active INTEGER NOT NULL DEFAULT 1,
    cooldown_seconds INTEGER NOT NULL DEFAULT 300,
    last_triggered_at TEXT,
    trigger_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE,
    FOREIGN KEY (stream_id) REFERENCES iot_data_streams(id) ON DELETE CASCADE
);

CREATE INDEX idx_alert_rules_tenant ON iot_alert_rules(tenant_id);
CREATE INDEX idx_alert_rules_tenant_device ON iot_alert_rules(tenant_id, device_id);
CREATE INDEX idx_alert_rules_tenant_stream ON iot_alert_rules(tenant_id, stream_id);
CREATE INDEX idx_alert_rules_tenant_active ON iot_alert_rules(tenant_id, is_active);
CREATE INDEX idx_alert_rules_tenant_severity ON iot_alert_rules(tenant_id, severity);
```

### 2.8 IoT Alerts
```sql
CREATE TABLE iot_alerts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    stream_id TEXT NOT NULL,
    alert_value REAL NOT NULL,
    threshold_value REAL,
    severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(severity IN ('INFO','WARNING','CRITICAL')),
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','ACKNOWLEDGED','RESOLVED')),
    acknowledged_by TEXT,
    acknowledged_at TEXT,
    resolved_at TEXT,
    resolution_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (rule_id) REFERENCES iot_alert_rules(id) ON DELETE CASCADE,
    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE,
    FOREIGN KEY (stream_id) REFERENCES iot_data_streams(id) ON DELETE CASCADE
);

CREATE INDEX idx_alerts_tenant ON iot_alerts(tenant_id);
CREATE INDEX idx_alerts_tenant_device ON iot_alerts(tenant_id, device_id);
CREATE INDEX idx_alerts_tenant_status ON iot_alerts(tenant_id, status);
CREATE INDEX idx_alerts_tenant_severity ON iot_alerts(tenant_id, severity);
CREATE INDEX idx_alerts_tenant_rule ON iot_alerts(tenant_id, rule_id);
CREATE INDEX idx_alerts_tenant_created ON iot_alerts(tenant_id, created_at);
```

### 2.9 IoT Device Commands
```sql
CREATE TABLE iot_device_commands (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    command_type TEXT NOT NULL,
    command_payload TEXT NOT NULL,              -- JSON: command-specific data
    status TEXT NOT NULL DEFAULT 'QUEUED'
        CHECK(status IN ('QUEUED','SENT','ACKNOWLEDGED','FAILED','COMPLETED')),
    sent_at TEXT,
    acknowledged_at TEXT,
    response_payload TEXT,                      -- JSON: device response data
    correlation_id TEXT,
    issued_by TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE
);

CREATE INDEX idx_commands_tenant ON iot_device_commands(tenant_id);
CREATE INDEX idx_commands_tenant_device ON iot_device_commands(tenant_id, device_id);
CREATE INDEX idx_commands_tenant_status ON iot_device_commands(tenant_id, status);
CREATE INDEX idx_commands_tenant_correlation ON iot_device_commands(correlation_id);
CREATE INDEX idx_commands_tenant_issued_by ON iot_device_commands(tenant_id, issued_by);
```

### 2.10 IoT Edge Configurations
```sql
CREATE TABLE iot_edge_configurations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    edge_device_id TEXT NOT NULL,
    config_type TEXT NOT NULL
        CHECK(config_type IN ('FILTER','TRANSFORM','AGGREGATE','ML_INFERENCE')),
    config_body TEXT NOT NULL,                  -- JSON: configuration definition
    deployed_version INTEGER NOT NULL DEFAULT 0,
    deployment_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(deployment_status IN ('PENDING','DEPLOYED','FAILED')),
    last_deployed_at TEXT,
    deployed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (edge_device_id) REFERENCES iot_device_registrations(id) ON DELETE CASCADE
);

CREATE INDEX idx_edge_configs_tenant ON iot_edge_configurations(tenant_id);
CREATE INDEX idx_edge_configs_tenant_device ON iot_edge_configurations(tenant_id, edge_device_id);
CREATE INDEX idx_edge_configs_tenant_type ON iot_edge_configurations(tenant_id, config_type);
CREATE INDEX idx_edge_configs_tenant_status ON iot_edge_configurations(tenant_id, deployment_status);
```

---

## 3. REST API Endpoints

```
# Device Management
GET/POST      /api/v1/iot/devices                          Permission: iot.devices.read/create
GET/PUT       /api/v1/iot/devices/{id}                     Permission: iot.devices.read/update
POST          /api/v1/iot/devices/{id}/activate            Permission: iot.devices.manage
POST          /api/v1/iot/devices/{id}/deactivate          Permission: iot.devices.manage
GET           /api/v1/iot/devices/{id}/status              Permission: iot.devices.read
GET           /api/v1/iot/devices/{id}/heartbeat           Permission: iot.devices.read

# Device Groups
GET/POST      /api/v1/iot/groups                           Permission: iot.groups.read/create
GET/PUT       /api/v1/iot/groups/{id}                      Permission: iot.groups.read/update
POST          /api/v1/iot/groups/{id}/devices              Permission: iot.groups.manage
DELETE        /api/v1/iot/groups/{id}/devices/{did}        Permission: iot.groups.manage

# Data Streams
GET/POST      /api/v1/iot/streams                          Permission: iot.streams.read/create
GET/PUT       /api/v1/iot/streams/{id}                     Permission: iot.streams.read/update
GET           /api/v1/iot/streams/{id}/data                Permission: iot.streams.read
GET           /api/v1/iot/streams/{id}/aggregates          Permission: iot.streams.read

# Telemetry Ingestion
POST          /api/v1/iot/ingest                           Permission: iot.ingest
POST          /api/v1/iot/ingest/batch                     Permission: iot.ingest
GET           /api/v1/iot/telemetry/latest                 Permission: iot.telemetry.read
GET           /api/v1/iot/telemetry/history                Permission: iot.telemetry.read

# Alert Rules
GET/POST      /api/v1/iot/alert-rules                      Permission: iot.alerts.read/create
GET/PUT       /api/v1/iot/alert-rules/{id}                 Permission: iot.alerts.read/update
GET           /api/v1/iot/alerts                           Permission: iot.alerts.read
POST          /api/v1/iot/alerts/{id}/acknowledge          Permission: iot.alerts.update
POST          /api/v1/iot/alerts/{id}/resolve              Permission: iot.alerts.update

# Device Commands
POST          /api/v1/iot/devices/{id}/command             Permission: iot.commands.send
GET           /api/v1/iot/commands/{id}                    Permission: iot.commands.read

# Edge Configuration
GET/POST      /api/v1/iot/edge/configs                     Permission: iot.edge.read/create
POST          /api/v1/iot/edge/configs/{id}/deploy         Permission: iot.edge.deploy
GET           /api/v1/iot/edge/configs/{id}/status         Permission: iot.edge.read

# Dashboard
GET           /api/v1/iot/dashboard/device-map             Permission: iot.dashboard.read
GET           /api/v1/iot/dashboard/telemetry-overview     Permission: iot.dashboard.read
GET           /api/v1/iot/dashboard/alert-summary          Permission: iot.dashboard.read
```

---

## 4. Business Rules

### 4.1 Device Registration and Lifecycle
```
1. Devices MUST be registered before accepting telemetry data
2. Registration assigns a unique device_code per tenant
3. Device lifecycle states: REGISTERED -> ACTIVE -> INACTIVE -> DECOMMISSIONED
4. Only ACTIVE devices accept telemetry ingestion
5. Device activation requires valid certificates (if protocol demands TLS)
6. Device decommission cascades: deactivates streams, cancels queued commands
7. Missing heartbeat for configured interval triggers device INACTIVE status
   - Default heartbeat timeout: 5 minutes (configurable per device)
   - Heartbeat check runs every 60 seconds as background job
```

### 4.2 Telemetry Ingestion
- Telemetry ingestion validates device identity and data format
- Ingestion validates device_id exists and status is ACTIVE
- Ingestion validates stream_id exists and matches device_id
- Raw telemetry is written to iot_telemetry_raw with ingestion_time
- Telemetry values are stored in typed columns: value_real, value_text, value_boolean
- Quality flag defaults to GOOD; device can report UNCERTAIN or BAD
- Batch ingestion supports up to 1000 data points per request
- Ingestion rate limits enforced per tenant: 10,000 points/minute default

### 4.3 Telemetry Aggregation
- Raw telemetry is aggregated on configurable intervals (MINUTE, HOUR, DAY)
- Aggregation background job runs at the start of each period
- MINUTE aggregation: computes at +1 minute past each minute boundary
- HOUR aggregation: computes at +1 minute past each hour boundary
- DAY aggregation: computes at 00:05 UTC each day for previous day
- Aggregation computes: avg_value, min_value, max_value, sum_value, sample_count, std_dev
- Aggregation type on stream defines which aggregate value is the primary summary
- Historical aggregates are never recomputed unless explicitly requested

### 4.4 Alert Rules and Evaluation
- Alert rules evaluate in real-time against incoming telemetry
- Condition types:
  - THRESHOLD: value exceeds configured threshold (e.g., temperature > 100)
  - RATE_OF_CHANGE: value change rate exceeds limit (e.g., pressure drops > 10/min)
  - ABSENCE: no data received within configured time window
  - PATTERN: value matches a configured pattern sequence
- Cooldown period prevents alert storms for sustained threshold violations
  - Default cooldown: 300 seconds (5 minutes)
  - Alert for same rule + device only fires once per cooldown window
- CRITICAL alerts bypass cooldown for the first occurrence in a 1-hour window
- Alert triggers increment rule trigger_count and update last_triggered_at
- Alert action_type determines what happens when rule fires:
  - NOTIFICATION: push to assistant-service notification system
  - WEBHOOK: POST to configured external URL
  - COMMAND: issue a command to the device or another device
  - EAM_WORK_ORDER: create a maintenance work order in eam-service

### 4.5 Device Commands
- Device commands are queued and tracked until acknowledged
- Command lifecycle: QUEUED -> SENT -> ACKNOWLEDGED/FAILED -> COMPLETED
- Commands are delivered via the device's registered protocol
- Failed delivery retries up to 3 times with exponential backoff
- Command timeout: QUEUED commands expire after 24 hours if not SENT
- correlation_id links command request to device response
- response_payload captured when device acknowledges with data

### 4.6 Edge Computing
- Edge configurations support data filtering and local ML inference
- Configuration types:
  - FILTER: discard telemetry matching filter criteria before forwarding
  - TRANSFORM: modify telemetry values (e.g., unit conversion)
  - AGGREGATE: pre-aggregate data at the edge before forwarding
  - ML_INFERENCE: run ML model at the edge, forward inference results
- Edge configs are versioned; deployment increments deployed_version
- Deployment status tracks whether config was successfully pushed to edge device
- Failed deployments are retried automatically once, then require manual intervention

### 4.7 Telemetry Retention
- Telemetry retention follows stream-level retention policy
- Default retention_days: 90
- Background job purges raw telemetry older than retention_days per stream
- Aggregated data is NOT purged (retained indefinitely for historical analysis)
- Retention policy changes apply prospectively only
- Purge job runs daily at 02:00 UTC

### 4.8 Device Heartbeat Monitoring
- Devices expected to send heartbeat at regular intervals
- Missing heartbeat for configured interval triggers device INACTIVE status
- Heartbeat interval is configurable per device via config JSON (default: 300 seconds)
- Status transition: ACTIVE -> INACTIVE on missed heartbeat
- Status transition: INACTIVE -> ACTIVE on next heartbeat received
- INACTIVE device triggers `iot.device.offline` event
- INACTIVE device stops accepting telemetry; telemetry rejected with error

### 4.9 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `iot.telemetry.received` | Telemetry data point ingested | AI Service, Reporting |
| `iot.alert.triggered` | Alert rule evaluates true | Assistant, EAM, Workflow |
| `iot.alert.resolved` | Alert manually resolved | Assistant, Reporting |
| `iot.device.offline` | Device heartbeat missed | Assistant, EAM |
| `iot.device.registered` | New device registered | Reporting |
| `iot.command.completed` | Device command response received | Reporting, Audit |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.iot.v1;

service IotService {
    rpc IngestTelemetry(IngestTelemetryRequest) returns (IngestTelemetryResponse);
    rpc GetLatestTelemetry(GetLatestTelemetryRequest) returns (GetLatestTelemetryResponse);
    rpc SendDeviceCommand(SendDeviceCommandRequest) returns (SendDeviceCommandResponse);
    rpc SubscribeAlerts(SubscribeAlertsRequest) returns (stream AlertEvent);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **EAM:** Operating asset records for device-to-asset linkage, location data
- **Auth:** User identity and permissions for device management actions
- **AI Service:** ML models for edge inference deployment, anomaly pattern detection
- **Inventory:** Warehouse and location data for device placement context
- **Workflow:** Approval workflow definitions for alert escalation

### 6.2 Data Published To
- **EAM:** Meter readings from telemetry streams, alert-triggered work orders, device health status
- **Assistant:** Alert notifications for proactive delivery, device status change alerts
- **AI Service:** Telemetry data for anomaly detection training, pattern analysis
- **Reporting:** Telemetry aggregations, alert summaries, device uptime statistics
- **Workflow:** Alert escalation workflows for CRITICAL severity events

### 6.3 Events Consumed
| Event | Action |
|-------|--------|
| `eam.asset.created` | Suggest device registration for new assets |
| `eam.asset.decommissioned` | Trigger device decommission for linked devices |
| `ai.anomaly.detected` | Create IoT alert from AI-detected anomaly in telemetry |

### 6.4 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `iot.telemetry.received` | Telemetry data point ingested | AI Service, Reporting |
| `iot.alert.triggered` | Alert rule evaluates true | Assistant, EAM, Workflow |
| `iot.alert.resolved` | Alert manually resolved | Assistant, Reporting |
| `iot.device.offline` | Device heartbeat missed | Assistant, EAM |
| `iot.device.registered` | New device registered | Reporting |
| `iot.command.completed` | Device command response received | Reporting, Audit |

---

## 7. Migrations

1. V001: `iot_device_registrations`
2. V002: `iot_device_groups`
3. V003: `iot_device_group_members`
4. V004: `iot_data_streams`
5. V005: `iot_telemetry_raw`
6. V006: `iot_telemetry_aggregates`
7. V007: `iot_alert_rules`
8. V008: `iot_alerts`
9. V009: `iot_device_commands`
10. V010: `iot_edge_configurations`
11. V011: Indexes for all tables
12. V012: Triggers for `updated_at`
