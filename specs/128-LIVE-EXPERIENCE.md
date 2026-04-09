# 128 - Live Experience Specification

## 1. Domain Overview

Live Experience provides real-time customer engagement capabilities including co-browsing, screen sharing, video calling, and live annotation. It enables agents and customers to collaborate visually during support interactions, reducing resolution time and improving customer satisfaction for complex issues.

**Bounded Context:** Real-Time Customer Engagement & Visual Support
**Service Name:** `liveexp-service`
**Database:** `data/liveexp.db`
**HTTP Port:** 8208 | **gRPC Port:** 9208

---

## 2. Database Schema

### 2.1 Engagement Sessions
```sql
CREATE TABLE le_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_code TEXT NOT NULL,          -- Shareable session code for customer join
    session_type TEXT NOT NULL CHECK(session_type IN ('CO_BROWSE','SCREEN_SHARE','VIDEO_CALL','VOICE_CALL','ANNOTATION')),
    status TEXT NOT NULL DEFAULT 'WAITING'
        CHECK(status IN ('WAITING','ACTIVE','PAUSED','ENDED')),
    agent_id TEXT NOT NULL,
    customer_id TEXT,
    customer_name TEXT,
    customer_email TEXT,
    channel TEXT,                        -- Originating channel
    request_id TEXT,                     -- Link to service request
    case_id TEXT,
    start_time TEXT,
    end_time TEXT,
    duration_seconds INTEGER NOT NULL DEFAULT 0,
    recording_enabled INTEGER NOT NULL DEFAULT 0,
    recording_url TEXT,
    resolution_notes TEXT,
    outcome TEXT CHECK(outcome IN ('RESOLVED','UNRESOLVED','ESCALATED','ABANDONED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, session_code)
);

CREATE INDEX idx_le_sessions_tenant_status ON le_sessions(tenant_id, status);
CREATE INDEX idx_le_sessions_tenant_agent ON le_sessions(tenant_id, agent_id, start_time DESC);
CREATE INDEX idx_le_sessions_tenant_customer ON le_sessions(tenant_id, customer_id, start_time DESC);
```

### 2.2 Session Participants
```sql
CREATE TABLE le_participants (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    participant_type TEXT NOT NULL CHECK(participant_type IN ('AGENT','CUSTOMER','SUPERVISOR','EXPERT')),
    user_id TEXT NOT NULL,
    display_name TEXT NOT NULL,
    join_time TEXT NOT NULL,
    leave_time TEXT,
    connection_status TEXT NOT NULL DEFAULT 'CONNECTED'
        CHECK(connection_status IN ('CONNECTED','RECONNECTING','DISCONNECTED')),
    device_type TEXT CHECK(device_type IN ('DESKTOP','MOBILE','TABLET')),
    browser TEXT,
    ip_address TEXT,
    geo_location TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (session_id) REFERENCES le_sessions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, session_id, user_id)
);
```

### 2.3 Co-Browse State
```sql
CREATE TABLE le_cobrowse_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    event_type TEXT NOT NULL CHECK(event_type IN ('NAVIGATE','CLICK','SCROLL','INPUT','FORM_FILL','HIGHLIGHT','BLACKOUT','UNBLACKOUT')),
    url TEXT,
    element_selector TEXT,
    event_data TEXT,                     -- JSON: event-specific data
    screenshot_url TEXT,                 -- Periodic screenshot captures
    participant_id TEXT NOT NULL,
    event_timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (session_id) REFERENCES le_sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_le_cobrowse_tenant_session ON le_cobrowse_events(tenant_id, session_id, event_timestamp);
```

### 2.4 Annotations
```sql
CREATE TABLE le_annotations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    participant_id TEXT NOT NULL,
    annotation_type TEXT NOT NULL CHECK(annotation_type IN ('ARROW','CIRCLE','RECTANGLE','TEXT','FREEHAND','HIGHLIGHT')),
    target_url TEXT,
    target_element TEXT,
    position_x REAL NOT NULL,
    position_y REAL NOT NULL,
    width REAL,
    height REAL,
    color TEXT NOT NULL DEFAULT '#FF0000',
    text_content TEXT,
    page_screenshot_url TEXT,
    annotation_timestamp TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (session_id) REFERENCES le_sessions(id) ON DELETE CASCADE
);
```

### 2.5 Recordings
```sql
CREATE TABLE le_recordings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    recording_type TEXT NOT NULL CHECK(recording_type IN ('VIDEO','SCREEN','AUDIO','TRANSCRIPT')),
    file_url TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    duration_seconds INTEGER NOT NULL,
    format TEXT NOT NULL DEFAULT 'WEBM',
    started_at TEXT NOT NULL,
    ended_at TEXT NOT NULL,
    transcript_text TEXT,                -- For transcript recordings
    consent_obtained INTEGER NOT NULL DEFAULT 0,
    retention_until TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (session_id) REFERENCES le_sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_le_recordings_tenant_session ON le_recordings(tenant_id, session_id);
```

### 2.6 Privacy Controls
```sql
CREATE TABLE le_privacy_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK(rule_type IN ('BLACKOUT_FIELD','BLACKOUT_URL','BLACKOUT_REGION','MASK_DATA','BLOCK_NAVIGATION')),
    selector TEXT NOT NULL,              -- CSS selector or URL pattern
    description TEXT,
    priority INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, rule_name)
);
```

---

## 3. REST API Endpoints

### 3.1 Session Management
```
POST   /api/v1/liveexp/sessions                         Permission: le.sessions.create
  Request: { "agent_id": "...", "session_type": "CO_BROWSE", "request_id": "..." }
  Response 201: { "data": { "session_id": "...", "session_code": "ABC123", "join_url": "https://..." } }

GET    /api/v1/liveexp/sessions/{id}                    Permission: le.sessions.read
GET    /api/v1/liveexp/sessions                         Permission: le.sessions.read
POST   /api/v1/liveexp/sessions/{id}/end                Permission: le.sessions.end
GET    /api/v1/liveexp/sessions/{id}/participants       Permission: le.sessions.read
```

### 3.2 Customer Join
```
POST   /api/v1/liveexp/join                              Permission: le.join (public)
  Request: { "session_code": "ABC123", "customer_name": "..." }
  Response: { "data": { "session_id": "...", "websocket_url": "wss://..." } }
```

### 3.3 Co-Browsing
```
POST   /api/v1/liveexp/sessions/{id}/cobrowse/events    Permission: le.cobrowse.write
GET    /api/v1/liveexp/sessions/{id}/cobrowse/state      Permission: le.cobrowse.read
POST   /api/v1/liveexp/sessions/{id}/cobrowse/navigate   Permission: le.cobrowse.control
POST   /api/v1/liveexp/sessions/{id}/cobrowse/highlight  Permission: le.cobrowse.annotate
```

### 3.4 Annotations
```
POST   /api/v1/liveexp/sessions/{id}/annotations        Permission: le.annotate.create
GET    /api/v1/liveexp/sessions/{id}/annotations        Permission: le.annotate.read
DELETE /api/v1/liveexp/sessions/{id}/annotations/{aid}   Permission: le.annotate.delete
```

### 3.5 Recordings
```
POST   /api/v1/liveexp/sessions/{id}/recordings/start   Permission: le.recordings.start
POST   /api/v1/liveexp/sessions/{id}/recordings/stop    Permission: le.recordings.stop
GET    /api/v1/liveexp/sessions/{id}/recordings         Permission: le.recordings.read
GET    /api/v1/liveexp/recordings/{id}                  Permission: le.recordings.read
GET    /api/v1/liveexp/recordings/{id}/download         Permission: le.recordings.download
```

### 3.6 Privacy
```
GET    /api/v1/liveexp/privacy/rules                    Permission: le.privacy.read
POST   /api/v1/liveexp/privacy/rules                    Permission: le.privacy.create
PUT    /api/v1/liveexp/privacy/rules/{id}               Permission: le.privacy.update
```

---

## 4. Business Rules

### 4.1 Session Management
- Session code is unique, 6-character alphanumeric, expires after 24 hours if unused
- Agent can invite customer via link, email, or SMS
- Only one active session per agent at a time (configurable)
- Supervisor can silently join any active session for coaching
- Session automatically ends if no activity for 30 minutes

### 4.2 Co-Browsing Rules
- Agent sees customer's browser in real-time (via WebSocket relay)
- Agent can only observe by default; control requires customer approval
- Privacy blackout rules hide sensitive fields (passwords, SSN, credit cards)
- Customer can revoke agent control at any time
- Navigation restricted to allowed domains only
- DOM mutations tracked and relayed to agent view

### 4.3 Recording & Consent
- Recording requires explicit customer consent before starting
- Consent captured with timestamp and stored
- Recordings encrypted at rest
- Automatic retention policy enforcement (default 90 days)
- Transcripts generated via AI for searchability

### 4.4 Privacy Controls
- Blackout rules applied on DOM elements matching selectors
- Agent-side rendering masks blacked-out content
- Data masking applied to form fields during replay
- Blocked navigation prevents customer from visiting unauthorized URLs during session

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.liveexp.v1;

service LiveExperienceService {
    rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);
    rpc JoinSession(JoinSessionRequest) returns (JoinSessionResponse);
    rpc EndSession(EndSessionRequest) returns (EndSessionResponse);
    rpc StreamCoBrowseEvents(stream CoBrowseEvent) returns (stream CoBrowseEvent);
    rpc GetRecording(GetRecordingRequest) returns (GetRecordingResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **customerservice-service**: Service request linkage, case management
- **b2cservice-service**: B2C service request integration
- **digitalcs-service**: Digital-first support integration
- **dms-service**: Recording storage, screenshots
- **knowledge-service**: Contextual help during co-browse
- **ai-service**: Real-time transcription, sentiment analysis

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `le.session.created` | Engagement session created | session_id, agent_id, type |
| `le.session.started` | Customer joined | session_id, customer_id |
| `le.session.ended` | Session concluded | session_id, duration, outcome |
| `le.recording.started` | Recording initiated | session_id, consent_obtained |
| `le.recording.completed` | Recording finished | session_id, recording_url |
| `le.annotation.created` | Visual annotation made | session_id, type, participant |
| `le.consent.captured` | Customer consent recorded | session_id, consent_type |

---

## 7. Migrations

### Migration Order for liveexp-service:
1. V001: `le_privacy_rules`
2. V002: `le_sessions`
3. V003: `le_participants`
4. V004: `le_cobrowse_events`
5. V005: `le_annotations`
6. V006: `le_recordings`
7. V007: Triggers for `updated_at`
8. V008: Seed data (default privacy rules for common sensitive fields)
