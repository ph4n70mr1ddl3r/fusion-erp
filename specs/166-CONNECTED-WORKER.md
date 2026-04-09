# 166 - Connected Worker Service Specification

## 1. Domain Overview

Connected Worker provides IoT-enabled workforce safety, productivity monitoring, and real-time worker engagement for industrial and deskless environments. Supports wearable device integration, location tracking, hazard monitoring, fatigue detection, safety compliance, and emergency response coordination. Enables real-time visibility into worker location, vital signs, environmental conditions, and task progress. Integrates with Workforce Safety, Manufacturing Execution, IoT Integration, and Workforce Scheduling.

**Bounded Context:** IoT-Connected Workforce Safety & Productivity
**Service Name:** `connected-worker-service`
**Database:** `data/connected_worker.db`
**HTTP Port:** 8184 | **gRPC Port:** 9184

---

## 2. Database Schema

### 2.1 Worker Devices
```sql
CREATE TABLE cw_worker_devices (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    device_type TEXT NOT NULL CHECK(device_type IN (
        'WEARABLE','SMART_HELMET','SMARTWATCH','TAG_BEACON','MOBILE','TABLET','CUSTOM'
    )),
    device_serial TEXT NOT NULL,
    assigned_worker_id TEXT,
    firmware_version TEXT,
    battery_level_pct REAL,
    last_heartbeat TEXT,
    capabilities TEXT NOT NULL,             -- JSON: ["location","heart_rate","temperature","gas_detection"]
    status TEXT NOT NULL DEFAULT 'INVENTORY'
        CHECK(status IN ('INVENTORY','ASSIGNED','ACTIVE','MAINTENANCE','DECOMMISSIONED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, device_id)
);

CREATE INDEX idx_cw_device_worker ON cw_worker_devices(assigned_worker_id, status);
CREATE INDEX idx_cw_device_type ON cw_worker_devices(tenant_id, device_type);
```

### 2.2 Worker Locations
```sql
CREATE TABLE cw_worker_locations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    worker_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    zone_id TEXT NOT NULL,
    zone_name TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    latitude REAL,
    longitude REAL,
    altitude REAL,
    floor_level INTEGER,
    accuracy_meters REAL,
    speed_mps REAL,
    heading_degrees REAL,

    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_cw_loc_worker ON cw_worker_locations(worker_id, timestamp DESC);
CREATE INDEX idx_cw_loc_zone ON cw_worker_locations(facility_id, zone_id, timestamp DESC);
CREATE INDEX idx_cw_loc_time ON cw_worker_locations(tenant_id, timestamp DESC);
```

### 2.3 Vital Signs & Sensors
```sql
CREATE TABLE cw_sensor_readings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    worker_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    reading_type TEXT NOT NULL CHECK(reading_type IN (
        'HEART_RATE','BODY_TEMPERATURE','BLOOD_OXYGEN','FATIGUE_SCORE',
        'ENVIRONMENTAL_TEMP','HUMIDITY','GAS_LEVEL','NOISE_LEVEL','UV_INDEX','CUSTOM'
    )),
    value REAL NOT NULL,
    unit TEXT NOT NULL,                     -- bpm, celsius, %, ppm, dB, etc.
    alert_level TEXT NOT NULL DEFAULT 'NORMAL'
        CHECK(alert_level IN ('NORMAL','WARNING','CRITICAL')),
    threshold_warning REAL,
    threshold_critical REAL,

    timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_cw_sensor_worker ON cw_sensor_readings(worker_id, reading_type, timestamp DESC);
CREATE INDEX idx_cw_sensor_alert ON cw_sensor_readings(tenant_id, alert_level, timestamp DESC);
```

### 2.4 Safety Alerts
```sql
CREATE TABLE cw_safety_alerts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    alert_type TEXT NOT NULL CHECK(alert_type IN (
        'GAS_DETECTION','FATIGUE','HEAT_STRESS','FALL_DETECTION',
        'GEOFENCE_BREACH','SOS','MAN_DOWN','EVACUATION','EQUIPMENT_HAZARD','CUSTOM'
    )),
    severity TEXT NOT NULL CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL','EMERGENCY')),
    worker_id TEXT NOT NULL,
    worker_name TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    zone_id TEXT NOT NULL,
    location TEXT,                          -- JSON: lat, long, floor
    device_id TEXT NOT NULL,
    description TEXT NOT NULL,
    sensor_readings TEXT,                   -- JSON: associated readings
    response_actions TEXT,                  -- JSON: automated actions taken
    responders TEXT,                        -- JSON: notified responders
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ACKNOWLEDGED','RESPONDING','RESOLVED','FALSE_ALARM')),
    acknowledged_by TEXT,
    acknowledged_at TEXT,
    resolved_by TEXT,
    resolved_at TEXT,
    resolution_notes TEXT,
    duration_seconds INTEGER,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_cw_alert_tenant ON cw_safety_alerts(tenant_id, severity, status);
CREATE INDEX idx_cw_alert_worker ON cw_safety_alerts(worker_id, created_at DESC);
CREATE INDEX idx_cw_alert_facility ON cw_safety_alerts(facility_id, status);
```

### 2.5 Geofence Zones
```sql
CREATE TABLE cw_geofence_zones (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    zone_code TEXT NOT NULL,
    zone_name TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    zone_type TEXT NOT NULL CHECK(zone_type IN ('SAFE','RESTRICTED','HAZARDOUS','EMERGENCY_ASSEMBLY','EVACUATION_ROUTE')),
    boundary TEXT NOT NULL,                 -- JSON: polygon coordinates
    floor_level INTEGER,
    max_occupancy INTEGER,
    current_occupancy INTEGER NOT NULL DEFAULT 0,
    time_limit_minutes INTEGER,
    required_ppe TEXT,                      -- JSON: required PPE items
    hazard_types TEXT,                      -- JSON: hazard classifications
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, zone_code)
);

CREATE INDEX idx_cw_geo_facility ON cw_geofence_zones(facility_id, zone_type);
```

### 2.6 Shift Safety Summary
```sql
CREATE TABLE cw_shift_safety (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    worker_id TEXT NOT NULL,
    shift_date TEXT NOT NULL,
    shift_start TEXT NOT NULL,
    shift_end TEXT,
    facility_id TEXT NOT NULL,
    total_time_in_hazardous_minutes INTEGER NOT NULL DEFAULT 0,
    alerts_generated INTEGER NOT NULL DEFAULT 0,
    alerts_critical INTEGER NOT NULL DEFAULT 0,
    fatigue_events INTEGER NOT NULL DEFAULT 0,
    geofence_breaches INTEGER NOT NULL DEFAULT 0,
    avg_heart_rate REAL,
    max_heart_rate REAL,
    max_temperature REAL,
    ppe_compliance_pct REAL NOT NULL DEFAULT 100,
    safety_score REAL NOT NULL DEFAULT 100, -- 0-100

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, worker_id, shift_date, shift_start)
);

CREATE INDEX idx_cw_shift_worker ON cw_shift_safety(worker_id, shift_date DESC);
CREATE INDEX idx_cw_shift_score ON cw_shift_safety(tenant_id, safety_score);
```

---

## 3. API Endpoints

### 3.1 Worker Tracking
| Method | Path | Description |
|--------|------|-------------|
| POST | `/connected-worker/v1/locations` | Update worker location |
| GET | `/connected-worker/v1/workers/{id}/location` | Get current location |
| GET | `/connected-worker/v1/facilities/{id}/workers` | Workers in facility |
| GET | `/connected-worker/v1/zones/{id}/occupancy` | Zone occupancy |

### 3.2 Sensor Data
| Method | Path | Description |
|--------|------|-------------|
| POST | `/connected-worker/v1/sensor-readings` | Submit sensor reading |
| POST | `/connected-worker/v1/sensor-readings/bulk` | Bulk sensor data |
| GET | `/connected-worker/v1/workers/{id}/vitals` | Worker vital signs |
| GET | `/connected-worker/v1/workers/{id}/vitals/history` | Historical vitals |

### 3.3 Safety Alerts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/connected-worker/v1/alerts` | Create safety alert |
| GET | `/connected-worker/v1/alerts` | List active alerts |
| GET | `/connected-worker/v1/alerts/{id}` | Get alert details |
| POST | `/connected-worker/v1/alerts/{id}/acknowledge` | Acknowledge alert |
| POST | `/connected-worker/v1/alerts/{id}/resolve` | Resolve alert |
| POST | `/connected-worker/v1/sos` | SOS emergency alert |

### 3.4 Geofence Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/connected-worker/v1/zones` | Create geofence zone |
| GET | `/connected-worker/v1/zones` | List zones |
| PUT | `/connected-worker/v1/zones/{id}` | Update zone |
| GET | `/connected-worker/v1/zones/{id}/history` | Zone entry/exit log |

### 3.5 Devices & Reports
| Method | Path | Description |
|--------|------|-------------|
| POST | `/connected-worker/v1/devices` | Register device |
| GET | `/connected-worker/v1/devices` | List devices |
| PUT | `/connected-worker/v1/devices/{id}/assign` | Assign to worker |
| GET | `/connected-worker/v1/dashboard` | Safety dashboard |
| GET | `/connected-worker/v1/reports/shift-summary` | Shift safety reports |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `cworker.alert.triggered` | `{ alert_id, type, severity, worker_id, zone }` | Safety alert triggered |
| `cworker.alert.acknowledged` | `{ alert_id, responder }` | Alert acknowledged |
| `cworker.geofence.breached` | `{ worker_id, zone_id, direction }` | Geofence breach |
| `cworker.sos.activated` | `{ worker_id, location }` | SOS emergency |
| `cworker.device.offline` | `{ device_id, worker_id, last_heartbeat }` | Device went offline |
| `cworker.fatigue.detected` | `{ worker_id, fatigue_score }` | Worker fatigue detected |
| `cworker.vitals.critical` | `{ worker_id, reading_type, value }` | Critical vital reading |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `shift.started` | Workforce Scheduling | Start shift safety tracking |
| `shift.ended` | Workforce Scheduling | Generate shift safety summary |
| `iot.device.data` | IoT Integration | Process sensor data |
| `maintenance.hazard.detected` | Enterprise Asset | Alert workers in area |

---

## 5. Business Rules

1. **Real-Time Monitoring**: Location updates processed within 10 seconds of receipt
2. **Alert Escalation**: CRITICAL alerts auto-escalate to safety manager within 2 minutes
3. **Geofence Compliance**: Workers in hazardous zones without required PPE auto-alerted
4. **Fatigue Detection**: Workers with fatigue score >70 flagged for mandatory break
5. **Occupancy Limits**: Zone over-occupancy triggers alert to facility manager
6. **Device Heartbeat**: Devices must report within 5 minutes; offline triggers alert
7. **Data Retention**: Location data retained 30 days; safety events retained 5 years
8. **Emergency Mode**: Emergency triggers evacuate-all notification to zone workers

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Workforce Safety (76) | Safety incident management |
| IoT Integration (44) | Device data ingestion |
| Workforce Scheduling (73) | Shift and assignment data |
| Manufacturing Execution (99) | Shop floor worker tracking |
| Enterprise Asset (40) | Maintenance zone alerts |
| Notification Center (165) | Alert delivery |
| Core HR (62) | Worker identity data |
| Workforce Labor Optimization (105) | Labor analytics |
| Mobile (46) | Mobile worker app |
