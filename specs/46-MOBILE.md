# 46 - Mobile Application Framework Service Specification

## 1. Domain Overview

Mobile Application Framework provides device registration and management, push notification delivery with platform-specific routing (iOS APNs, Android FCM), offline-first action queueing with conflict resolution, mobile screen configuration management with role-based visibility, barcode/QR/NFC scanning with contextual action routing, and incremental data synchronization. Integrates with Auth for user authentication and token management, all domain services for data sync, Document for scan attachments, and Notification for push delivery orchestration.

**Bounded Context:** Mobile Application Support, Push Notifications & Offline Sync
**Service Name:** `mobile-service`
**Database:** `data/mobile.db`
**HTTP Port:** 8073 | **gRPC Port:** 9073

---

## 2. Database Schema

### 2.1 Mobile Device Registrations
```sql
CREATE TABLE mobile_device_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    device_token TEXT NOT NULL,                  -- Unique device identifier
    platform TEXT NOT NULL
        CHECK(platform IN ('IOS','ANDROID','PWA')),
    device_model TEXT,
    os_version TEXT,
    app_version TEXT,
    language TEXT NOT NULL DEFAULT 'en',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active INTEGER NOT NULL DEFAULT 1,
    registered_at TEXT NOT NULL DEFAULT (datetime('now')),
    last_active_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, user_id, device_token)
);

CREATE INDEX idx_device_reg_tenant_user ON mobile_device_registrations(tenant_id, user_id);
CREATE INDEX idx_device_reg_tenant_active ON mobile_device_registrations(tenant_id, is_active);
CREATE INDEX idx_device_reg_tenant_platform ON mobile_device_registrations(tenant_id, platform);
CREATE INDEX idx_device_reg_tenant_last_active ON mobile_device_registrations(tenant_id, last_active_at);
```

### 2.2 Mobile Push Tokens
```sql
CREATE TABLE mobile_push_tokens (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    device_id TEXT NOT NULL,                     -- FK to mobile_device_registrations
    push_token TEXT NOT NULL,
    platform TEXT NOT NULL
        CHECK(platform IN ('IOS','ANDROID','PWA')),
    bundle_id TEXT NOT NULL,                     -- Application bundle identifier
    is_active INTEGER NOT NULL DEFAULT 1,
    registered_at TEXT NOT NULL DEFAULT (datetime('now')),
    invalidated_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (device_id) REFERENCES mobile_device_registrations(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, user_id, push_token)
);

CREATE INDEX idx_push_tokens_tenant_user ON mobile_push_tokens(tenant_id, user_id);
CREATE INDEX idx_push_tokens_tenant_active ON mobile_push_tokens(tenant_id, is_active);
CREATE INDEX idx_push_tokens_tenant_platform ON mobile_push_tokens(tenant_id, platform);
```

### 2.3 Mobile Offline Queue
```sql
CREATE TABLE mobile_offline_queue (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    action_type TEXT NOT NULL
        CHECK(action_type IN ('CREATE','UPDATE','DELETE')),
    target_service TEXT NOT NULL,                -- e.g. "gl-service", "ap-service"
    target_endpoint TEXT NOT NULL,               -- e.g. "/api/v1/gl/journals"
    http_method TEXT NOT NULL
        CHECK(http_method IN ('POST','PUT','PATCH','DELETE')),
    payload TEXT NOT NULL,                       -- JSON: request body
    status TEXT NOT NULL DEFAULT 'QUEUED'
        CHECK(status IN ('QUEUED','SYNCING','SYNCED','CONFLICT','FAILED')),
    sync_attempts INTEGER NOT NULL DEFAULT 0,
    last_attempt_at TEXT,
    conflict_resolution TEXT,                    -- JSON: resolution strategy result
    server_version INTEGER,                      -- Server version for optimistic locking
    created_offline_at TEXT NOT NULL,
    synced_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_offline_queue_tenant_user ON mobile_offline_queue(tenant_id, user_id);
CREATE INDEX idx_offline_queue_tenant_status ON mobile_offline_queue(tenant_id, status);
CREATE INDEX idx_offline_queue_tenant_device ON mobile_offline_queue(tenant_id, device_id);
CREATE INDEX idx_offline_queue_tenant_service ON mobile_offline_queue(tenant_id, target_service);
```

### 2.4 Mobile Screen Configurations
```sql
CREATE TABLE mobile_screen_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    screen_name TEXT NOT NULL,
    screen_type TEXT NOT NULL
        CHECK(screen_type IN ('LIST','DETAIL','FORM','DASHBOARD','CAMERA','SCANNER')),
    layout_config TEXT NOT NULL,                 -- JSON: layout definition
    filters TEXT,                                -- JSON: available filter options
    sort_config TEXT,                            -- JSON: default sort configuration
    actions TEXT,                                -- JSON array: available screen actions
    is_offline_capable INTEGER NOT NULL DEFAULT 0,
    cache_ttl_seconds INTEGER NOT NULL DEFAULT 300,
    requires_auth INTEGER NOT NULL DEFAULT 1,
    role_permissions TEXT,                       -- JSON: role-based visibility rules

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, screen_name)
);

CREATE INDEX idx_screen_configs_tenant_type ON mobile_screen_configs(tenant_id, screen_type);
CREATE INDEX idx_screen_configs_tenant_active ON mobile_screen_configs(tenant_id, is_active);
```

### 2.5 Mobile App Configurations
```sql
CREATE TABLE mobile_app_configs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    config_key TEXT NOT NULL,
    config_value TEXT NOT NULL,                  -- JSON: configuration value
    config_type TEXT NOT NULL
        CHECK(config_type IN ('FEATURE_FLAG','THEME','BEHAVIOR','INTEGRATION')),
    platform TEXT NOT NULL DEFAULT 'ALL'
        CHECK(platform IN ('ALL','IOS','ANDROID','PWA')),
    min_app_version TEXT,                        -- Minimum app version for this config
    description TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, config_key, platform)
);

CREATE INDEX idx_app_configs_tenant_type ON mobile_app_configs(tenant_id, config_type);
CREATE INDEX idx_app_configs_tenant_platform ON mobile_app_configs(tenant_id, platform);
CREATE INDEX idx_app_configs_tenant_active ON mobile_app_configs(tenant_id, is_active);
```

### 2.6 Mobile Sync Audit
```sql
CREATE TABLE mobile_sync_audit (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    sync_type TEXT NOT NULL
        CHECK(sync_type IN ('FULL','INCREMENTAL','DELTA')),
    entities_synced TEXT,                        -- JSON: list of entity types synced
    records_pushed INTEGER NOT NULL DEFAULT 0,
    records_pulled INTEGER NOT NULL DEFAULT 0,
    conflicts_count INTEGER NOT NULL DEFAULT 0,
    duration_ms INTEGER NOT NULL DEFAULT 0,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    status TEXT NOT NULL DEFAULT 'SUCCESS'
        CHECK(status IN ('SUCCESS','PARTIAL','FAILED')),
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_sync_audit_tenant_user ON mobile_sync_audit(tenant_id, user_id);
CREATE INDEX idx_sync_audit_tenant_status ON mobile_sync_audit(tenant_id, status);
CREATE INDEX idx_sync_audit_tenant_device ON mobile_sync_audit(tenant_id, device_id);
CREATE INDEX idx_sync_audit_tenant_dates ON mobile_sync_audit(tenant_id, started_at, completed_at);
```

### 2.7 Mobile Barcode Scans
```sql
CREATE TABLE mobile_barcode_scans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    scan_type TEXT NOT NULL
        CHECK(scan_type IN ('BARCODE','QR','NFC')),
    scan_value TEXT NOT NULL,
    scan_context TEXT NOT NULL
        CHECK(scan_context IN ('RECEIVING','PICKING','COUNTING','ASSET_LOOKUP','GENERAL')),
    entity_type TEXT,                            -- Resolved entity type (e.g. "ITEM", "ASSET")
    entity_id TEXT,                              -- Resolved entity ID
    latitude REAL,
    longitude REAL,
    scanned_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_barcode_scans_tenant_user ON mobile_barcode_scans(tenant_id, user_id);
CREATE INDEX idx_barcode_scans_tenant_context ON mobile_barcode_scans(tenant_id, scan_context);
CREATE INDEX idx_barcode_scans_tenant_value ON mobile_barcode_scans(tenant_id, scan_value);
CREATE INDEX idx_barcode_scans_tenant_dates ON mobile_barcode_scans(tenant_id, scanned_at);
```

---

## 3. REST API Endpoints

```
# Device Registration
POST          /api/v1/mobile/devices/register
PUT           /api/v1/mobile/devices/{id}
DELETE        /api/v1/mobile/devices/{id}
GET           /api/v1/mobile/devices/my-devices

# Push Notifications
POST          /api/v1/mobile/push/register-token
DELETE        /api/v1/mobile/push/unregister-token
POST          /api/v1/mobile/push/send                    Permission: mobile.push.send
POST          /api/v1/mobile/push/broadcast               Permission: mobile.push.broadcast

# Offline Sync
GET           /api/v1/mobile/sync/status
POST          /api/v1/mobile/sync/push                    Permission: mobile.sync
GET           /api/v1/mobile/sync/pull                    Permission: mobile.sync
POST          /api/v1/mobile/sync/resolve-conflict/{id}   Permission: mobile.sync
GET           /api/v1/mobile/sync/queue

# Screen Configuration
GET           /api/v1/mobile/screens                      Permission: mobile.config.read
GET/PUT       /api/v1/mobile/screens/{name}               Permission: mobile.config.manage

# App Configuration
GET           /api/v1/mobile/config
GET/PUT       /api/v1/mobile/config/{key}                 Permission: mobile.config.manage

# Barcode/Scanning
POST          /api/v1/mobile/scan                         Permission: mobile.scan
GET           /api/v1/mobile/scan/history
POST          /api/v1/mobile/scan/batch-lookup

# Dashboard
GET           /api/v1/mobile/dashboard
```

---

## 4. Business Rules

### 4.1 Device Management
```
Device registration lifecycle:
  1. Device tokens MUST be refreshed on each app launch
  2. A user may register multiple devices (phone, tablet)
  3. Inactive devices (no activity for 90 days) are auto-deactivated
  4. Device model, OS version, and app version tracked for compatibility
  5. Device deactivation invalidates all associated push tokens
  6. Re-registration of a known device_token updates existing record
```

### 4.2 Push Notifications
- Push tokens are platform-specific: APNs for iOS, FCM for Android
- Push notifications respect user preferences and quiet hours
- Quiet hours configured per user (default: 22:00 - 07:00 local time)
- Failed delivery attempts invalidate the push token after 3 consecutive failures
- Broadcast notifications respect tenant scope and role targeting
- Notification payload size limits: APNs 4KB, FCM 4KB (standard)

### 4.3 Offline Sync
- Offline queue actions are processed in FIFO order during sync
- Conflict resolution uses server-wins or user-chooses strategy based on entity type:
  - Financial data (GL, AP, AR): server-wins
  - Operational data (Inventory, Orders): user-chooses
  - Reference data (Items, Suppliers): server-wins
- Maximum offline queue size: 1000 actions per device
- Actions older than 30 days in FAILED status are auto-purged
- Full sync is required on first login; subsequent syncs are incremental/delta
- Delta sync uses server_version for optimistic concurrency control

### 4.4 Screen Configuration
- Screen configurations support role-based visibility
- Screen layouts are cached on device for offline access
- Cache TTL determines refresh frequency (default: 5 minutes)
- Camera and Scanner screen types require hardware capability detection
- Form screens support field validation rules in layout_config
- Dashboard screens aggregate data from multiple services

### 4.5 App Configuration
- App version below min_app_version MUST show update required message
- Feature flags evaluated at config fetch time (not cached long-term)
- Theme configurations applied immediately without app restart
- Platform-specific configs override ALL configs when present
- Configuration changes are versioned for rollback support

### 4.6 Barcode Scanning
- Barcode scans trigger contextual actions based on scan_context:
  - RECEIVING: match against expected receipts/purchase orders
  - PICKING: validate against pick list lines
  - COUNTING: populate physical inventory count
  - ASSET_LOOKUP: retrieve asset details and maintenance history
  - GENERAL: universal entity lookup
- Batch lookup supports multiple scan values in single request
- GPS coordinates captured with each scan for location validation
- Scan history retained for 90 days per device

### 4.7 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `mobile.device.registered` | New device registered | Notification |
| `mobile.sync.completed` | Sync operation finished | Audit |
| `mobile.sync.conflict` | Sync conflict detected | Workflow |
| `mobile.scan.processed` | Barcode scan resolved | Context-specific service |
| `mobile.push.delivered` | Push notification delivered | Notification |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.mobile.v1;

service MobileService {
    rpc RegisterDevice(RegisterDeviceRequest) returns (RegisterDeviceResponse);
    rpc SyncData(stream SyncRequest) returns (stream SyncResponse);
    rpc SendPushNotification(SendPushNotificationRequest) returns (SendPushNotificationResponse);
    rpc ProcessBarcodeScan(ProcessBarcodeScanRequest) returns (ProcessBarcodeScanResponse);
}

// Entity messages

message MobileDeviceRegistration {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string device_token = 4;
    string platform = 5;
    string device_model = 6;
    string os_version = 7;
    string app_version = 8;
    string language = 9;
    string timezone = 10;
    string registered_at = 11;
    string last_active_at = 12;
    string created_at = 13;
    string updated_at = 14;
}

message MobilePushToken {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string device_id = 4;
    string push_token = 5;
    string platform = 6;
    string bundle_id = 7;
    string registered_at = 8;
    string invalidated_at = 9;
    string created_at = 10;
    string updated_at = 11;
}

message MobileOfflineQueueItem {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string device_id = 4;
    string action_type = 5;
    string target_service = 6;
    string target_endpoint = 7;
    string http_method = 8;
    string payload = 9;
    string status = 10;
    int32 sync_attempts = 11;
    string last_attempt_at = 12;
    string conflict_resolution = 13;
    int32 server_version = 14;
    string created_offline_at = 15;
    string synced_at = 16;
    string created_at = 17;
    string updated_at = 18;
}

message MobileScreenConfig {
    string id = 1;
    string tenant_id = 2;
    string screen_name = 3;
    string screen_type = 4;
    string layout_config = 5;
    string filters = 6;
    string sort_config = 7;
    string actions = 8;
    bool is_offline_capable = 9;
    int32 cache_ttl_seconds = 10;
    bool requires_auth = 11;
    string role_permissions = 12;
    string created_at = 13;
    string updated_at = 14;
}

message MobileSyncAudit {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string device_id = 4;
    string sync_type = 5;
    string entities_synced = 6;
    int32 records_pushed = 7;
    int32 records_pulled = 8;
    int32 conflicts_count = 9;
    int32 duration_ms = 10;
    string started_at = 11;
    string completed_at = 12;
    string status = 13;
    string error_message = 14;
    string created_at = 15;
    string updated_at = 16;
}

message MobileBarcodeScan {
    string id = 1;
    string tenant_id = 2;
    string user_id = 3;
    string device_id = 4;
    string scan_type = 5;
    string scan_value = 6;
    string scan_context = 7;
    string entity_type = 8;
    string entity_id = 9;
    double latitude = 10;
    double longitude = 11;
    string scanned_at = 12;
    string created_at = 13;
    string updated_at = 14;
}

// Request/Response messages

message RegisterDeviceRequest {
    string tenant_id = 1;
    string user_id = 2;
    string device_token = 3;
    string platform = 4;
    string device_model = 5;
    string os_version = 6;
    string app_version = 7;
    string language = 8;
    string timezone = 9;
}

message RegisterDeviceResponse {
    MobileDeviceRegistration data = 1;
}

message SyncRequest {
    string tenant_id = 1;
    string user_id = 2;
    string device_id = 3;
    string sync_type = 4;
    string payload = 5;
}

message SyncResponse {
    string tenant_id = 1;
    string status = 2;
    string sync_data = 3;
    int32 records_processed = 4;
    int32 conflicts = 5;
    string server_version = 6;
}

message SendPushNotificationRequest {
    string tenant_id = 1;
    string user_id = 2;
    string title = 3;
    string body = 4;
    string data_payload = 5;
    string priority = 6;
}

message SendPushNotificationResponse {
    bool success = 1;
    string message_id = 2;
    string platform = 3;
}

message ProcessBarcodeScanRequest {
    string tenant_id = 1;
    string user_id = 2;
    string device_id = 3;
    string scan_type = 4;
    string scan_value = 5;
    string scan_context = 6;
    double latitude = 7;
    double longitude = 8;
}

message ProcessBarcodeScanResponse {
    MobileBarcodeScan scan = 1;
    string entity_type = 2;
    string entity_id = 3;
    string resolved_data = 4;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Auth:** User authentication tokens, session validation, role assignments
- **All Domain Services:** Entity data for sync pull operations, entity resolution for barcode scans
- **Notification:** Push notification templates, delivery preferences
- **Document:** Attachment storage for scan images and documents

### 6.2 Data Published To
- **All Domain Services:** Offline queue actions forwarded to target services during sync push
- **Inventory:** Receiving scan events, counting scan events, picking confirmation
- **EAM:** Asset lookup scan events, asset barcode resolution
- **Workflow:** Sync conflict resolution workflows, device approval workflows
- **Notification:** Push notification delivery status, token invalidation
- **Audit:** Sync audit records, scan audit records

---

## 7. Migrations

1. V001: `mobile_device_registrations`
2. V002: `mobile_push_tokens`
3. V003: `mobile_offline_queue`
4. V004: `mobile_screen_configs`
5. V005: `mobile_app_configs`
6. V006: `mobile_sync_audit`
7. V007: `mobile_barcode_scans`
8. V008: Triggers for `updated_at`
