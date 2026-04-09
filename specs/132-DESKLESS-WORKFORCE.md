# 132 - Deskless Workforce Specification

## 1. Domain Overview

Deskless Workforce provides mobile-first HR capabilities for frontline and field workers who don't use traditional desktop computers. It delivers simplified HR workflows, quick actions, shift management, time capture, communication, and compliance training optimized for mobile devices with offline capability and simplified interfaces.

**Bounded Context:** Frontline Workforce & Mobile-First HR
**Service Name:** `deskless-service`
**Database:** `data/deskless.db`
**HTTP Port:** 8212 | **gRPC Port:** 9212

---

## 2. Database Schema

### 2.1 Worker Quick Actions
```sql
CREATE TABLE dw_quick_actions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    action_code TEXT NOT NULL,
    action_name TEXT NOT NULL,
    action_type TEXT NOT NULL CHECK(action_type IN ('CLOCK_IN','CLOCK_OUT','SHIFT_SWAP','TIME_OFF_REQUEST','AVAILABILITY_UPDATE','TASK_COMPLETE','INCIDENT_REPORT','TRAINING_ACKNOWLEDGE','ANNOUNCEMENT_READ','CHECKLIST_COMPLETE')),
    description TEXT,
    icon TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_available_offline INTEGER NOT NULL DEFAULT 1,
    required_fields TEXT,                -- JSON: fields needed for this action
    applicable_roles TEXT,               -- JSON array of role names
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, action_code)
);
```

### 2.2 Mobile Time Capture
```sql
CREATE TABLE dw_time_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    worker_id TEXT NOT NULL,
    entry_type TEXT NOT NULL CHECK(entry_type IN ('CLOCK_IN','CLOCK_OUT','BREAK_START','BREAK_END','TASK_START','TASK_END')),
    timestamp TEXT NOT NULL,
    geo_location TEXT,                   -- "lat,lng" at time of capture
    location_name TEXT,
    photo_url TEXT,                      -- Selfie verification
    device_id TEXT,
    offline_entry INTEGER NOT NULL DEFAULT 0,
    synced_at TEXT,
    notes TEXT,
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','APPROVED','REJECTED','DISPUTED')),
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,

    UNIQUE(tenant_id, worker_id, entry_type, timestamp)
);

CREATE INDEX idx_dw_time_tenant_worker ON dw_time_entries(tenant_id, worker_id, timestamp DESC);
CREATE INDEX idx_dw_time_tenant_approval ON dw_time_entries(tenant_id, approval_status);
```

### 2.3 Shift Swap Requests
```sql
CREATE TABLE dw_shift_swaps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    requester_id TEXT NOT NULL,
    shift_date TEXT NOT NULL,
    shift_start TEXT NOT NULL,
    shift_end TEXT NOT NULL,
    requester_shift_id TEXT,
    cover_worker_id TEXT,
    cover_shift_id TEXT,
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','CANCELLED','EXPIRED')),
    manager_approval_required INTEGER NOT NULL DEFAULT 1,
    approved_by TEXT,
    approved_at TEXT,
    rejection_reason TEXT,
    expires_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, requester_id, shift_date, shift_start)
);

CREATE INDEX idx_dw_swaps_tenant_status ON dw_shift_swaps(tenant_id, status);
```

### 2.4 Announcements
```sql
CREATE TABLE dw_announcements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    announcement_type TEXT NOT NULL CHECK(announcement_type IN ('GENERAL','URGENT','POLICY','SAFETY','SCHEDULE','TRAINING')),
    priority INTEGER NOT NULL DEFAULT 5,
    target_audience TEXT NOT NULL,       -- JSON: { "locations": [...], "roles": [...], "departments": [...] }
    requires_acknowledgement INTEGER NOT NULL DEFAULT 0,
    acknowledgement_deadline TEXT,
    attachment_ids TEXT,                 -- JSON array
    published_at TEXT,
    expires_at TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','EXPIRED','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);
```

### 2.5 Announcement Acknowledgements
```sql
CREATE TABLE dw_announcement_ack (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    announcement_id TEXT NOT NULL,
    worker_id TEXT NOT NULL,
    acknowledged_at TEXT NOT NULL DEFAULT (datetime('now')),
    geo_location TEXT,
    device_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (announcement_id) REFERENCES dw_announcements(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, announcement_id, worker_id)
);

CREATE INDEX idx_dw_ack_tenant_announcement ON dw_announcement_ack(tenant_id, announcement_id);
```

### 2.6 Worker Checklists
```sql
CREATE TABLE dw_checklists (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    checklist_name TEXT NOT NULL,
    checklist_type TEXT NOT NULL CHECK(checklist_type IN ('OPENING','CLOSING','SAFETY','CLEANING','EQUIPMENT','CUSTOM')),
    items TEXT NOT NULL,                 -- JSON: [{ "id": "c1", "text": "...", "is_required": true, "photo_required": false }]
    applicable_roles TEXT,
    applicable_locations TEXT,
    frequency TEXT NOT NULL CHECK(frequency IN ('DAILY','WEEKLY','PER_SHIFT','ONE_TIME')),
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, checklist_name)
);
```

### 2.7 Checklist Completions
```sql
CREATE TABLE dw_checklist_completions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    checklist_id TEXT NOT NULL,
    worker_id TEXT NOT NULL,
    location_id TEXT,
    shift_date TEXT NOT NULL,
    responses TEXT NOT NULL,             -- JSON: { "c1": { "completed": true, "photo_url": "...", "notes": "..." } }
    completion_pct REAL NOT NULL DEFAULT 0.0,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    geo_location TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (checklist_id) REFERENCES dw_checklists(id) ON DELETE RESTRICT
);

CREATE INDEX idx_dw_checklist_comp_tenant_date ON dw_checklist_completions(tenant_id, shift_date DESC);
```

---

## 3. REST API Endpoints

### 3.1 Quick Actions
```
GET    /api/v1/deskless/actions                          Permission: dw.actions.read
POST   /api/v1/deskless/actions                          Permission: dw.actions.create
PUT    /api/v1/deskless/actions/{id}                     Permission: dw.actions.update
POST   /api/v1/deskless/actions/{code}/execute           Permission: dw.actions.execute
  Request: { "worker_id": "...", "fields": { "location": "Store #42" }, "geo_location": "..." }
```

### 3.2 Time Capture
```
POST   /api/v1/deskless/time/clock-in                    Permission: dw.time.clock
POST   /api/v1/deskless/time/clock-out                   Permission: dw.time.clock
POST   /api/v1/deskless/time/break-start                 Permission: dw.time.clock
POST   /api/v1/deskless/time/break-end                   Permission: dw.time.clock
GET    /api/v1/deskless/time/entries
  ?worker_id={id}&from=...&to=...                        Permission: dw.time.read
GET    /api/v1/deskless/time/status?worker_id={id}       Permission: dw.time.read
  Response: Current clock status, today's hours, current shift
POST   /api/v1/deskless/time/sync                        Permission: dw.time.sync
  Request: { "entries": [...] }  -- Bulk offline sync
PUT    /api/v1/deskless/time/entries/{id}/approve        Permission: dw.time.approve
```

### 3.3 Shift Swaps
```
GET    /api/v1/deskless/shifts/swap                      Permission: dw.shifts.read
POST   /api/v1/deskless/shifts/swap                      Permission: dw.shifts.swap
POST   /api/v1/deskless/shifts/swap/{id}/accept          Permission: dw.shifts.swap
POST   /api/v1/deskless/shifts/swap/{id}/approve         Permission: dw.shifts.approve
POST   /api/v1/deskless/shifts/swap/{id}/reject          Permission: dw.shifts.approve
GET    /api/v1/deskless/shifts/mine                      Permission: dw.shifts.read
```

### 3.4 Announcements
```
GET    /api/v1/deskless/announcements                    Permission: dw.announcements.read
POST   /api/v1/deskless/announcements                    Permission: dw.announcements.create
PUT    /api/v1/deskless/announcements/{id}               Permission: dw.announcements.update
GET    /api/v1/deskless/announcements/{id}/ack-stats     Permission: dw.announcements.read
POST   /api/v1/deskless/announcements/{id}/acknowledge   Permission: dw.announcements.ack
```

### 3.5 Checklists
```
GET    /api/v1/deskless/checklists                       Permission: dw.checklists.read
POST   /api/v1/deskless/checklists                       Permission: dw.checklists.create
GET    /api/v1/deskless/checklists/{id}                  Permission: dw.checklists.read
POST   /api/v1/deskless/checklists/{id}/start            Permission: dw.checklists.execute
PUT    /api/v1/deskless/checklists/{id}/complete         Permission: dw.checklists.execute
GET    /api/v1/deskless/checklists/completions           Permission: dw.checklists.read
```

---

## 4. Business Rules

### 4.1 Time Capture
- Clock-in/out MUST include geo-location when enabled
- Photo verification required for clock-in if policy mandates
- Offline entries timestamped locally, synced when connectivity restored
- Offline entries flagged for manager review
- Auto clock-out after configurable max shift duration (default 12 hours)
- Break tracking mandatory in jurisdictions requiring meal/rest breaks

### 4.2 Shift Swaps
- Workers can only request swaps for assigned future shifts
- Cover worker MUST have matching skills and availability
- Manager approval required unless auto-approval policy configured
- Swap requests expire if not accepted within configurable window
- Both workers' schedules updated upon approval

### 4.3 Announcements
- Urgent announcements push-notified immediately
- Non-urgent delivered during shift start
- Acknowledgement tracking with deadline enforcement
- Workers who don't acknowledge flagged to manager
- Multi-language support for diverse workforce

### 4.4 Offline Capability
- Quick actions available offline with local caching
- Entries queued locally and synced when online
- Conflict resolution: server timestamp wins over local
- Offline data encrypted on device
- Maximum offline storage period: 7 days

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.deskless.v1;

service DesklessWorkforceService {
    rpc ClockIn(ClockInRequest) returns (ClockInResponse);
    rpc ClockOut(ClockOutRequest) returns (ClockOutResponse);
    rpc SyncOffline(SyncOfflineRequest) returns (SyncOfflineResponse);
    rpc RequestShiftSwap(ShiftSwapRequest) returns (ShiftSwapResponse);
    rpc GetWorkerStatus(GetWorkerStatusRequest) returns (GetWorkerStatusResponse);
    rpc AcknowledgeAnnouncement(AcknowledgeAnnouncementRequest) returns (AcknowledgeAnnouncementResponse);
}

// Entity messages
message QuickAction {
    string id = 1;
    string tenant_id = 2;
    string action_code = 3;
    string action_name = 4;
    string action_type = 5;
    string description = 6;
    string icon = 7;
    int32 display_order = 8;
    int32 is_available_offline = 9;
    string required_fields = 10;
    string applicable_roles = 11;
    string status = 12;
    string created_at = 13;
    string updated_at = 14;
}

message TimeEntry {
    string id = 1;
    string tenant_id = 2;
    string worker_id = 3;
    string entry_type = 4;
    string timestamp = 5;
    string geo_location = 6;
    string location_name = 7;
    string photo_url = 8;
    string device_id = 9;
    int32 offline_entry = 10;
    string synced_at = 11;
    string notes = 12;
    string approval_status = 13;
    string approved_by = 14;
    string approved_at = 15;
    string created_at = 16;
    string updated_at = 17;
}

message ShiftSwap {
    string id = 1;
    string tenant_id = 2;
    string requester_id = 3;
    string shift_date = 4;
    string shift_start = 5;
    string shift_end = 6;
    string requester_shift_id = 7;
    string cover_worker_id = 8;
    string cover_shift_id = 9;
    string reason = 10;
    string status = 11;
    int32 manager_approval_required = 12;
    string approved_by = 13;
    string approved_at = 14;
    string rejection_reason = 15;
    string expires_at = 16;
    string created_at = 17;
    string updated_at = 18;
}

message Announcement {
    string id = 1;
    string tenant_id = 2;
    string title = 3;
    string content = 4;
    string announcement_type = 5;
    int32 priority = 6;
    string target_audience = 7;
    int32 requires_acknowledgement = 8;
    string acknowledgement_deadline = 9;
    string attachment_ids = 10;
    string published_at = 11;
    string expires_at = 12;
    string status = 13;
    string created_at = 14;
    string updated_at = 15;
}

message AnnouncementAck {
    string id = 1;
    string tenant_id = 2;
    string announcement_id = 3;
    string worker_id = 4;
    string acknowledged_at = 5;
    string geo_location = 6;
    string device_id = 7;
    string created_at = 8;
}

message Checklist {
    string id = 1;
    string tenant_id = 2;
    string checklist_name = 3;
    string checklist_type = 4;
    string items = 5;
    string applicable_roles = 6;
    string applicable_locations = 7;
    string frequency = 8;
    int32 is_active = 9;
    string created_at = 10;
    string updated_at = 11;
}

message ChecklistCompletion {
    string id = 1;
    string tenant_id = 2;
    string checklist_id = 3;
    string worker_id = 4;
    string location_id = 5;
    string shift_date = 6;
    string responses = 7;
    double completion_pct = 8;
    string started_at = 9;
    string completed_at = 10;
    string geo_location = 11;
    string created_at = 12;
    string updated_at = 13;
}

// Request/Response messages
message ClockInRequest {
    string tenant_id = 1;
    string worker_id = 2;
    string timestamp = 3;
    string geo_location = 4;
    string location_name = 5;
    string photo_url = 6;
    string device_id = 7;
    string notes = 8;
}

message ClockInResponse {
    TimeEntry data = 1;
}

message ClockOutRequest {
    string tenant_id = 1;
    string worker_id = 2;
    string timestamp = 3;
    string geo_location = 4;
    string photo_url = 5;
    string device_id = 6;
    string notes = 7;
}

message ClockOutResponse {
    TimeEntry data = 1;
    int64 shift_duration_seconds = 2;
}

message OfflineEntry {
    string entry_type = 1;
    string timestamp = 2;
    string geo_location = 3;
    string location_name = 4;
    string photo_url = 5;
    string device_id = 6;
    string notes = 7;
}

message SyncOfflineRequest {
    string tenant_id = 1;
    string worker_id = 2;
    repeated OfflineEntry entries = 3;
}

message SyncOfflineResponse {
    int32 synced_count = 1;
    int32 failed_count = 2;
    repeated string synced_entry_ids = 3;
}

message ShiftSwapRequest {
    string tenant_id = 1;
    string requester_id = 2;
    string shift_date = 3;
    string shift_start = 4;
    string shift_end = 5;
    string requester_shift_id = 6;
    string cover_worker_id = 7;
    string cover_shift_id = 8;
    string reason = 9;
    string expires_at = 10;
}

message ShiftSwapResponse {
    ShiftSwap data = 1;
}

message GetWorkerStatusRequest {
    string tenant_id = 1;
    string worker_id = 2;
}

message GetWorkerStatusResponse {
    string worker_id = 1;
    string clock_status = 2;
    string current_shift_start = 3;
    double today_hours = 4;
    string current_location = 5;
    string approval_status = 6;
}

message AcknowledgeAnnouncementRequest {
    string tenant_id = 1;
    string announcement_id = 2;
    string worker_id = 3;
    string geo_location = 4;
    string device_id = 5;
}

message AcknowledgeAnnouncementResponse {
    AnnouncementAck data = 1;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **hr-service**: Worker profiles, employment data
- **timelabor-service**: Time entry integration, labor rules
- **scheduling-service**: Shift data, availability
- **absence-service**: Time-off balances and requests
- **learning-service**: Training acknowledgements
- **mobile-service**: Push notifications, offline sync framework
- **safety-service**: Safety incident reporting

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `dw.time.clock_in` | Worker clocks in | worker_id, location, timestamp |
| `dw.time.clock_out` | Worker clocks out | worker_id, shift_duration |
| `dw.shift_swap.requested` | Swap request created | swap_id, requester_id, shift_date |
| `dw.shift_swap.approved` | Swap approved | swap_id, requester_id, cover_worker_id |
| `dw.announcement.published` | Announcement sent | announcement_id, type, target_count |
| `dw.announcement.acknowledged` | Worker acknowledges | announcement_id, worker_id |
| `dw.checklist.completed` | Checklist finished | checklist_id, worker_id, completion_pct |
| `dw.offline.synced` | Offline entries synced | worker_id, entry_count |

---

## 7. Migrations

### Migration Order for deskless-service:
1. V001: `dw_quick_actions`
2. V002: `dw_time_entries`
3. V003: `dw_shift_swaps`
4. V004: `dw_announcements`
5. V005: `dw_announcement_ack`
6. V006: `dw_checklists`
7. V007: `dw_checklist_completions`
8. V008: Triggers for `updated_at`
9. V009: Seed data (default quick actions, sample checklists)
