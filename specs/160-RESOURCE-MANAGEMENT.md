# 160 - Resource Management Service Specification

## 1. Domain Overview

Resource Management provides resource planning, assignment, capacity analysis, and utilization tracking for project-based organizations. Supports resource requests, staffing allocation, skill-based matching, availability forecasting, and resource conflict resolution. Enables resource managers to optimize resource deployment across projects, manage bench/available pools, and forecast future resource needs. Integrates with Project Management for project demands, Core HR for employee data, Time & Labor for actuals, and Skills for matching.

**Bounded Context:** Resource Planning & Staffing
**Service Name:** `resource-mgmt-service`
**Database:** `data/resource_mgmt.db`
**HTTP Port:** 8178 | **gRPC Port:** 9178

---

## 2. Database Schema

### 2.1 Resource Profiles
```sql
CREATE TABLE rm_resource_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    resource_type TEXT NOT NULL CHECK(resource_type IN ('EMPLOYEE','CONTRACTOR','CONSULTANT','TEAM')),
    primary_role TEXT NOT NULL,
    secondary_roles TEXT,                   -- JSON array
    skills TEXT NOT NULL,                   -- JSON: [{ "skill": "...", "proficiency": "EXPERT", "years": 5 }]
    certifications TEXT,                    -- JSON array
    availability_pct REAL NOT NULL DEFAULT 100,
    standard_hours_week REAL NOT NULL DEFAULT 40,
    cost_rate_hourly_cents INTEGER NOT NULL DEFAULT 0,
    bill_rate_hourly_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    home_department TEXT,
    home_location TEXT,
    calendar_id TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','ON_LEAVE','TERMINATED','ON_BENCH')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, person_id)
);

CREATE INDEX idx_rm_profile_role ON rm_resource_profiles(tenant_id, primary_role);
CREATE INDEX idx_rm_profile_status ON rm_resource_profiles(tenant_id, status);
CREATE INDEX idx_rm_profile_skill ON rm_resource_profiles(tenant_id);
```

### 2.2 Resource Requests
```sql
CREATE TABLE rm_resource_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    request_number TEXT NOT NULL,           -- RES-REQ-2024-00001
    project_id TEXT NOT NULL,
    task_id TEXT,
    requested_by TEXT NOT NULL,
    role_required TEXT NOT NULL,
    skills_required TEXT,                   -- JSON: required skills with minimum proficiency
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    hours_per_week REAL NOT NULL DEFAULT 40,
    allocation_pct REAL NOT NULL DEFAULT 100,
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    required_certifications TEXT,           -- JSON array
    location_requirement TEXT,
    status TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(status IN ('OPEN','MATCHED','PROPOSED','CONFIRMED','FILLED','CANCELLED')),
    assigned_resource_id TEXT,
    filled_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, request_number)
);

CREATE INDEX idx_rm_request_project ON rm_resource_requests(project_id, status);
CREATE INDEX idx_rm_request_date ON rm_resource_requests(tenant_id, start_date, status);
CREATE INDEX idx_rm_request_role ON rm_resource_requests(tenant_id, role_required, status);
```

### 2.3 Resource Allocations
```sql
CREATE TABLE rm_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    task_id TEXT,
    allocation_pct REAL NOT NULL,           -- 0-100%
    hours_per_week REAL NOT NULL,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    allocation_type TEXT NOT NULL CHECK(allocation_type IN ('HARD','SOFT','PROPOSED')),
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ACTIVE','COMPLETED','CANCELLED')),
    request_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (resource_id) REFERENCES rm_resource_profiles(id) ON DELETE CASCADE
);

CREATE INDEX idx_rm_alloc_resource ON rm_allocations(resource_id, start_date);
CREATE INDEX idx_rm_alloc_project ON rm_allocations(project_id, start_date);
CREATE INDEX idx_rm_alloc_date ON rm_allocations(tenant_id, start_date, end_date);
```

### 2.4 Capacity Forecasts
```sql
CREATE TABLE rm_capacity_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    period_start TEXT NOT NULL,             -- Week or month start
    period_end TEXT NOT NULL,
    period_type TEXT NOT NULL CHECK(period_type IN ('WEEKLY','MONTHLY')),
    available_hours REAL NOT NULL DEFAULT 0,
    allocated_hours REAL NOT NULL DEFAULT 0,
    actual_hours REAL NOT NULL DEFAULT 0,
    remaining_hours REAL NOT NULL DEFAULT 0,
    utilization_pct REAL NOT NULL DEFAULT 0,
    overallocated INTEGER NOT NULL DEFAULT 0,
    project_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, resource_id, period_start, period_type)
);

CREATE INDEX idx_rm_cap_resource ON rm_capacity_forecasts(resource_id, period_start);
CREATE INDEX idx_rm_cap_period ON rm_capacity_forecasts(tenant_id, period_start);
```

### 2.5 Skill Gap Analysis
```sql
CREATE TABLE rm_skill_gaps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    role TEXT NOT NULL,
    skill TEXT NOT NULL,
    required_level TEXT NOT NULL CHECK(required_level IN ('BEGINNER','INTERMEDIATE','ADVANCED','EXPERT')),
    current_avg_level TEXT,
    gap_severity TEXT NOT NULL CHECK(gap_severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    affected_project_count INTEGER NOT NULL DEFAULT 0,
    resource_count_with_skill INTEGER NOT NULL DEFAULT 0,
    resource_count_needed INTEGER NOT NULL DEFAULT 0,
    analysis_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, role, skill, analysis_date)
);

CREATE INDEX idx_rm_gap_tenant ON rm_skill_gaps(tenant_id, gap_severity);
```

---

## 3. API Endpoints

### 3.1 Resource Profiles
| Method | Path | Description |
|--------|------|-------------|
| POST | `/resource-mgmt/v1/profiles` | Create resource profile |
| GET | `/resource-mgmt/v1/profiles` | List profiles (filterable) |
| GET | `/resource-mgmt/v1/profiles/{id}` | Get profile details |
| PUT | `/resource-mgmt/v1/profiles/{id}` | Update profile |
| GET | `/resource-mgmt/v1/profiles/search` | Search by skills/role |

### 3.2 Resource Requests
| Method | Path | Description |
|--------|------|-------------|
| POST | `/resource-mgmt/v1/requests` | Create resource request |
| GET | `/resource-mgmt/v1/requests` | List requests |
| GET | `/resource-mgmt/v1/requests/{id}` | Get request details |
| PUT | `/resource-mgmt/v1/requests/{id}` | Update request |
| POST | `/resource-mgmt/v1/requests/{id}/cancel` | Cancel request |
| POST | `/resource-mgmt/v1/requests/{id}/find-matches` | Find matching resources |

### 3.3 Allocations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/resource-mgmt/v1/allocations` | Create allocation |
| GET | `/resource-mgmt/v1/allocations` | List allocations |
| PUT | `/resource-mgmt/v1/allocations/{id}` | Update allocation |
| DELETE | `/resource-mgmt/v1/allocations/{id}` | Cancel allocation |
| GET | `/resource-mgmt/v1/resources/{id}/allocations` | Resource's allocations |
| GET | `/resource-mgmt/v1/projects/{id}/allocations` | Project's allocations |

### 3.4 Capacity & Utilization
| Method | Path | Description |
|--------|------|-------------|
| GET | `/resource-mgmt/v1/capacity` | Capacity overview |
| GET | `/resource-mgmt/v1/capacity/heatmap` | Capacity heatmap |
| GET | `/resource-mgmt/v1/utilization` | Utilization metrics |
| POST | `/resource-mgmt/v1/capacity/forecast` | Generate forecast |
| GET | `/resource-mgmt/v1/conflicts` | Detect over-allocations |

### 3.5 Skill Analysis
| Method | Path | Description |
|--------|------|-------------|
| GET | `/resource-mgmt/v1/skill-gaps` | Skill gap analysis |
| POST | `/resource-mgmt/v1/skill-gaps/analyze` | Run analysis |
| GET | `/resource-mgmt/v1/bench` | Available/bench resources |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `res.request.created` | `{ request_id, project_id, role }` | Resource request created |
| `res.request.filled` | `{ request_id, resource_id }` | Request filled |
| `res.allocation.created` | `{ allocation_id, resource_id, project_id }` | Resource allocated |
| `res.allocation.conflict` | `{ resource_id, date, projects }` | Over-allocation detected |
| `res.utilization.low` | `{ resource_id, utilization_pct }` | Low utilization alert |
| `res.skill.gap.detected` | `{ role, skill, severity }` | Skill gap identified |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `project.created` | Project Mgmt | Generate resource demands |
| `project.task.created` | Project Mgmt | Generate resource requests |
| `employee.hired` | Core HR | Create resource profile |
| `employee.terminated` | Core HR | Deactivate resource profile |
| `timesheet.submitted` | Time & Labor | Update actual utilization |
| `employee.leave.planned` | Absence Mgmt | Adjust availability |

---

## 5. Business Rules

1. **Over-Allocation Detection**: Alert when allocation exceeds 100% in any period
2. **Skill Matching**: Resource match scored by skill relevance, proficiency, and availability
3. **Soft vs Hard Allocation**: Soft allocations can be overridden; hard allocations require release
4. **Utilization Targets**: Configurable target utilization (default 80%); alerts below 60%
5. **Conflict Resolution**: Over-allocated resources flagged to resource manager for resolution
6. **Forecasting**: 12-week rolling capacity forecast auto-generated weekly
7. **Bench Management**: Resources with <20% allocation flagged as available for assignment

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Project Management (15) | Project demands, task assignments |
| Core HR (62) | Employee data, organization hierarchy |
| Time & Labor (64) | Actual hours worked |
| Absence Management (72) | Time-off availability |
| Dynamic Skills (94) | Skills data and proficiency |
| Project Costing (158) | Resource cost rates |
| Project Billing (159) | Resource bill rates |
| Opportunity Marketplace (93) | Internal gig assignments |
