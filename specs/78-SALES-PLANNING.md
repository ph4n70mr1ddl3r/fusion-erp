# 78 - Sales Planning Service Specification

## 1. Domain Overview

Sales Planning manages territory definition and alignment, quota planning and allocation, territory membership assignment, planning period management, and sales coverage analytics to optimize sales resource deployment. Territories are defined by geographic boundaries, industry verticals, product lines, or custom criteria using configurable rules. Quota plans define target revenue or unit goals for planning periods (quarterly, annually) and are allocated down to individual sales representatives or teams. Territory realignment tracks historical changes for audit and transition management. Attainment metrics compare actual pipeline and closed-won revenue against quotas. Integrates with Sales Automation for opportunity and pipeline data and with Compensation for commission-eligible quota targets.

**Bounded Context:** Territory & Quota Planning
**Service Name:** `salesplanning-service`
**Database:** `data/salesplanning.db`
**HTTP Port:** 8110 | **gRPC Port:** 9110

---

## 2. Database Schema

### 2.1 Territories
```sql
CREATE TABLE salesplanning_territories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    territory_code TEXT NOT NULL,
    territory_name TEXT NOT NULL,
    territory_type TEXT NOT NULL
        CHECK(territory_type IN ('GEOGRAPHIC','INDUSTRY','PRODUCT','ACCOUNT_BASED','CUSTOM')),
    parent_territory_id TEXT,
    description TEXT,
    geographic_region TEXT,
    country_code TEXT,
    state_province TEXT,
    city TEXT,
    postal_code_range_start TEXT,
    postal_code_range_end TEXT,
    industry_vertical TEXT,
    product_line TEXT,
    coverage_type TEXT NOT NULL DEFAULT 'EXCLUSIVE'
        CHECK(coverage_type IN ('EXCLUSIVE','SHARED','OVERLAY')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE','ARCHIVED')),
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, territory_code)
);

CREATE INDEX idx_territories_tenant_type ON salesplanning_territories(tenant_id, territory_type);
CREATE INDEX idx_territories_tenant_parent ON salesplanning_territories(tenant_id, parent_territory_id);
CREATE INDEX idx_territories_tenant_status ON salesplanning_territories(tenant_id, status);
CREATE INDEX idx_territories_tenant_active ON salesplanning_territories(tenant_id, is_active);
CREATE INDEX idx_territories_tenant_geo ON salesplanning_territories(tenant_id, country_code, state_province);
```

### 2.2 Territory Members
```sql
CREATE TABLE salesplanning_territory_members (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    territory_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'REP'
        CHECK(role IN ('REP','SENIOR_REP','MANAGER','DIRECTOR','OVERLAY')),
    allocation_percentage INTEGER NOT NULL DEFAULT 100,
    effective_start_date TEXT NOT NULL,
    effective_end_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','TRANSFERRED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, territory_id, user_id, effective_start_date)
);

CREATE INDEX idx_territory_members_tenant_territory ON salesplanning_territory_members(tenant_id, territory_id);
CREATE INDEX idx_territory_members_tenant_user ON salesplanning_territory_members(tenant_id, user_id);
CREATE INDEX idx_territory_members_tenant_status ON salesplanning_territory_members(tenant_id, status);
CREATE INDEX idx_territory_members_tenant_active ON salesplanning_territory_members(tenant_id, is_active);
```

### 2.3 Territory Rules
```sql
CREATE TABLE salesplanning_territory_rules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    territory_id TEXT NOT NULL,
    rule_name TEXT NOT NULL,
    rule_type TEXT NOT NULL
        CHECK(rule_type IN ('GEOGRAPHIC','INDUSTRY','REVENUE_RANGE','EMPLOYEE_RANGE','ACCOUNT_ATTRIBUTE','CUSTOM')),
    rule_condition TEXT NOT NULL,
    rule_priority INTEGER NOT NULL DEFAULT 0,
    match_logic TEXT NOT NULL DEFAULT 'INCLUDE'
        CHECK(match_logic IN ('INCLUDE','EXCLUDE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, territory_id, rule_name)
);

CREATE INDEX idx_territory_rules_tenant_territory ON salesplanning_territory_rules(tenant_id, territory_id);
CREATE INDEX idx_territory_rules_tenant_type ON salesplanning_territory_rules(tenant_id, rule_type);
CREATE INDEX idx_territory_rules_tenant_active ON salesplanning_territory_rules(tenant_id, is_active);
```

### 2.4 Quota Plans
```sql
CREATE TABLE salesplanning_quota_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_code TEXT NOT NULL,
    plan_type TEXT NOT NULL
        CHECK(plan_type IN ('REVENUE','UNITS','MARGIN','ACTIVITY','CUSTOM')),
    period_id TEXT NOT NULL,
    total_quota_amount INTEGER NOT NULL DEFAULT 0,
    total_quota_units INTEGER NOT NULL DEFAULT 0,
    currency TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','ACTIVE','CLOSED','ARCHIVED')),
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','SUBMITTED','APPROVED','REJECTED')),
    approved_by TEXT,
    approved_at TEXT,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_code)
);

CREATE INDEX idx_quota_plans_tenant_status ON salesplanning_quota_plans(tenant_id, status);
CREATE INDEX idx_quota_plans_tenant_period ON salesplanning_quota_plans(tenant_id, period_id);
CREATE INDEX idx_quota_plans_tenant_type ON salesplanning_quota_plans(tenant_id, plan_type);
CREATE INDEX idx_quota_plans_tenant_active ON salesplanning_quota_plans(tenant_id, is_active);
```

### 2.5 Quota Allocations
```sql
CREATE TABLE salesplanning_quota_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    territory_id TEXT,
    user_id TEXT,
    allocated_amount INTEGER NOT NULL DEFAULT 0,
    allocated_units INTEGER NOT NULL DEFAULT 0,
    attainment_amount INTEGER NOT NULL DEFAULT 0,
    attainment_units INTEGER NOT NULL DEFAULT 0,
    attainment_percentage REAL NOT NULL DEFAULT 0.0,
    allocation_method TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(allocation_method IN ('MANUAL','EQUAL','WEIGHTED','TOP_DOWN','BOTTOM_UP')),
    weight REAL NOT NULL DEFAULT 1.0,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_id, territory_id, user_id)
);

CREATE INDEX idx_quota_allocations_tenant_plan ON salesplanning_quota_allocations(tenant_id, plan_id);
CREATE INDEX idx_quota_allocations_tenant_territory ON salesplanning_quota_allocations(tenant_id, territory_id);
CREATE INDEX idx_quota_allocations_tenant_user ON salesplanning_quota_allocations(tenant_id, user_id);
CREATE INDEX idx_quota_allocations_tenant_active ON salesplanning_quota_allocations(tenant_id, is_active);
```

### 2.6 Planning Periods
```sql
CREATE TABLE salesplanning_periods (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period_name TEXT NOT NULL,
    period_type TEXT NOT NULL
        CHECK(period_type IN ('MONTH','QUARTER','HALF_YEAR','YEAR','CUSTOM')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    fiscal_year INTEGER NOT NULL,
    fiscal_quarter INTEGER,
    is_current INTEGER NOT NULL DEFAULT 0,
    parent_period_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, period_name)
);

CREATE INDEX idx_periods_tenant_type ON salesplanning_periods(tenant_id, period_type);
CREATE INDEX idx_periods_tenant_fiscal ON salesplanning_periods(tenant_id, fiscal_year, fiscal_quarter);
CREATE INDEX idx_periods_tenant_dates ON salesplanning_periods(tenant_id, start_date, end_date);
CREATE INDEX idx_periods_tenant_active ON salesplanning_periods(tenant_id, is_active);
```

### 2.7 Territory Alignment History
```sql
CREATE TABLE salesplanning_territory_alignment_history (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    territory_id TEXT NOT NULL,
    realignment_type TEXT NOT NULL
        CHECK(realignment_type IN ('CREATED','MODIFIED','SPLIT','MERGED','DEACTIVATED','MEMBER_ADDED','MEMBER_REMOVED','MEMBER_TRANSFERRED')),
    previous_state TEXT,
    new_state TEXT,
    changed_by TEXT NOT NULL,
    change_reason TEXT,
    effective_date TEXT NOT NULL,
    impact_assessment TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_alignment_history_tenant_territory ON salesplanning_territory_alignment_history(tenant_id, territory_id);
CREATE INDEX idx_alignment_history_tenant_type ON salesplanning_territory_alignment_history(tenant_id, realignment_type);
CREATE INDEX idx_alignment_history_tenant_date ON salesplanning_territory_alignment_history(tenant_id, effective_date);
CREATE INDEX idx_alignment_history_tenant_active ON salesplanning_territory_alignment_history(tenant_id, is_active);
```

### 2.8 Plan Metrics
```sql
CREATE TABLE salesplanning_plan_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    territory_id TEXT,
    user_id TEXT,
    period_id TEXT NOT NULL,
    metric_type TEXT NOT NULL
        CHECK(metric_type IN ('PIPELINE','WEIGHTED_PIPELINE','COMMITTED_FORECAST','CLOSED_WON','BOOKINGS','ATTAINMENT')),
    metric_value INTEGER NOT NULL DEFAULT 0,
    quota_value INTEGER NOT NULL DEFAULT 0,
    attainment_percentage REAL NOT NULL DEFAULT 0.0,
    calculated_at TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_id, territory_id, user_id, period_id, metric_type)
);

CREATE INDEX idx_plan_metrics_tenant_plan ON salesplanning_plan_metrics(tenant_id, plan_id);
CREATE INDEX idx_plan_metrics_tenant_territory ON salesplanning_plan_metrics(tenant_id, territory_id);
CREATE INDEX idx_plan_metrics_tenant_user ON salesplanning_plan_metrics(tenant_id, user_id);
CREATE INDEX idx_plan_metrics_tenant_period ON salesplanning_plan_metrics(tenant_id, period_id);
CREATE INDEX idx_plan_metrics_tenant_active ON salesplanning_plan_metrics(tenant_id, is_active);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/territories` | Create a new territory |
| GET | `/api/v1/territories` | List territories with filtering and pagination |
| GET | `/api/v1/territories/{id}` | Get territory by ID with members and rules |
| PUT | `/api/v1/territories/{id}` | Update territory definition |
| DELETE | `/api/v1/territories/{id}` | Deactivate a territory |
| GET | `/api/v1/territories/{id}/hierarchy` | Get territory hierarchy tree |
| POST | `/api/v1/territories/{id}/members` | Assign a member to a territory |
| GET | `/api/v1/territories/{id}/members` | List members of a territory |
| PUT | `/api/v1/territories/{id}/members/{memberId}` | Update member allocation or role |
| DELETE | `/api/v1/territories/{id}/members/{memberId}` | Remove a member from a territory |
| POST | `/api/v1/territories/{id}/rules` | Add a territory rule |
| GET | `/api/v1/territories/{id}/rules` | List rules for a territory |
| PUT | `/api/v1/territories/{id}/rules/{ruleId}` | Update a territory rule |
| DELETE | `/api/v1/territories/{id}/rules/{ruleId}` | Remove a territory rule |
| POST | `/api/v1/territories/{id}/realign` | Realign territory with change tracking |
| POST | `/api/v1/territories/{id}/split` | Split a territory into multiple territories |
| POST | `/api/v1/territories/merge` | Merge multiple territories into one |
| POST | `/api/v1/territories/evaluate` | Evaluate account matching against territory rules |
| POST | `/api/v1/plans` | Create a quota plan |
| GET | `/api/v1/plans` | List quota plans with filtering |
| GET | `/api/v1/plans/{id}` | Get plan with allocations and metrics |
| PUT | `/api/v1/plans/{id}` | Update plan details |
| DELETE | `/api/v1/plans/{id}` | Deactivate a quota plan |
| POST | `/api/v1/plans/{id}/allocations` | Create a quota allocation |
| GET | `/api/v1/plans/{id}/allocations` | List allocations for a plan |
| PUT | `/api/v1/plans/{id}/allocations/{allocationId}` | Update a quota allocation |
| POST | `/api/v1/plans/{id}/allocations/distribute` | Auto-distribute quotas across allocations |
| POST | `/api/v1/plans/{id}/submit` | Submit plan for approval |
| PATCH | `/api/v1/plans/{id}/approve` | Approve or reject a quota plan |
| POST | `/api/v1/periods` | Create a planning period |
| GET | `/api/v1/periods` | List planning periods |
| GET | `/api/v1/periods/{id}` | Get period details |
| PUT | `/api/v1/periods/{id}` | Update a planning period |
| GET | `/api/v1/alignment-history` | List territory alignment history |
| GET | `/api/v1/alignment-history/{territoryId}` | Get alignment history for a territory |
| GET | `/api/v1/metrics` | Get plan metrics with filtering |
| GET | `/api/v1/metrics/attainment` | Get attainment report by territory or user |
| GET | `/api/v1/metrics/coverage` | Get territory coverage analysis |
| POST | `/api/v1/metrics/recalculate` | Trigger recalculation of plan metrics |
| GET | `/api/v1/dashboard/summary` | Get planning dashboard summary |

---

## 4. Business Rules

1. **Territory Uniqueness**: Each territory MUST have a unique `territory_code` within a tenant. Territory names SHOULD be unique within the same `territory_type`.

2. **Territory Hierarchy**: A territory MAY have a parent territory. The system MUST prevent circular references in the territory hierarchy. The maximum hierarchy depth SHOULD NOT exceed 4 levels.

3. **Member Allocation**: A territory member's `allocation_percentage` MUST be between 1 and 100. For shared territories, the total of all active member allocation percentages SHOULD NOT exceed 100.

4. **Exclusive Coverage**: When a territory has `coverage_type` set to `EXCLUSIVE`, an account or lead MUST be assigned to only one territory. `SHARED` territories allow overlapping account assignments.

5. **Territory Rule Evaluation**: Territory rules MUST be evaluated in priority order (lower number = higher priority). The first matching `INCLUDE` rule assigns the account. `EXCLUDE` rules MUST take precedence over `INCLUDE` rules at the same priority level.

6. **Quota Plan Totals**: The sum of all quota allocations for a plan MUST NOT exceed the plan's `total_quota_amount`. The system MUST validate this constraint on allocation creation and update.

7. **Planning Period Overlap**: Planning periods of the same `period_type` MUST NOT have overlapping date ranges within a tenant. Periods of different types (e.g., quarterly and annual) MAY overlap.

8. **Territory Realignment**: When a territory is realigned, the system MUST create an alignment history record capturing the previous and new state. Active quota allocations on the territory SHOULD be recalculated after realignment.

9. **Territory Split**: Splitting a territory MUST create two or more new child territories under the original territory's parent. Members MUST be redistributed based on split criteria or manually reassigned. The original territory MUST be set to `INACTIVE` status.

10. **Territory Merge**: Merging territories MUST create a single new territory. All members from source territories MUST be reassigned to the merged territory. Source territories MUST be set to `INACTIVE` status with alignment history records.

11. **Quota Attainment**: Attainment percentage MUST be calculated as `(attainment_amount / allocated_amount) * 100`. When `allocated_amount` is zero, attainment MUST be reported as 0%.

12. **Plan Approval Workflow**: A quota plan in `DRAFT` status MUST be submitted before it can be approved. Only users with a manager or director role MAY approve plans. A plan MUST NOT be modified after `APPROVED` status without reverting to `DRAFT`.

13. **Allocation Method**: When using `EQUAL` distribution, all allocations within a plan receive an equal share of the total quota. When using `WEIGHTED`, distribution MUST be proportional to each allocation's `weight` value. The `TOP_DOWN` method distributes from parent territory to child territories first.

14. **Effective Dates**: Territory member assignments MUST have an `effective_start_date`. If an `effective_end_date` is specified, it MUST be after the `effective_start_date`. Members without an end date are considered active indefinitely.

15. **Metric Refresh**: Plan metrics MUST be recalculated whenever quota allocations change, territory membership changes, or pipeline data is updated from Sales Automation. Stale metrics SHOULD be flagged with the `calculated_at` timestamp.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package salesplanning;

option go_package = "github.com/fusion/salesplanningpb";

service SalesPlanningService {
    // Territories
    rpc CreateTerritory(CreateTerritoryRequest) returns (TerritoryResponse);
    rpc GetTerritory(GetTerritoryRequest) returns (TerritoryResponse);
    rpc ListTerritories(ListTerritoriesRequest) returns (ListTerritoriesResponse);
    rpc UpdateTerritory(UpdateTerritoryRequest) returns (TerritoryResponse);
    rpc DeleteTerritory(DeleteTerritoryRequest) returns (DeleteResponse);
    rpc GetTerritoryHierarchy(GetTerritoryHierarchyRequest) returns (TerritoryHierarchyResponse);
    rpc RealignTerritory(RealignTerritoryRequest) returns (TerritoryResponse);
    rpc SplitTerritory(SplitTerritoryRequest) returns (SplitTerritoryResponse);
    rpc MergeTerritories(MergeTerritoriesRequest) returns (TerritoryResponse);
    rpc EvaluateTerritoryRules(EvaluateRulesRequest) returns (EvaluateRulesResponse);

    // Territory Members
    rpc AddTerritoryMember(AddTerritoryMemberRequest) returns (TerritoryMemberResponse);
    rpc ListTerritoryMembers(ListTerritoryMembersRequest) returns (ListTerritoryMembersResponse);
    rpc UpdateTerritoryMember(UpdateTerritoryMemberRequest) returns (TerritoryMemberResponse);
    rpc RemoveTerritoryMember(RemoveTerritoryMemberRequest) returns (DeleteResponse);

    // Territory Rules
    rpc AddTerritoryRule(AddTerritoryRuleRequest) returns (TerritoryRuleResponse);
    rpc ListTerritoryRules(ListTerritoryRulesRequest) returns (ListTerritoryRulesResponse);
    rpc UpdateTerritoryRule(UpdateTerritoryRuleRequest) returns (TerritoryRuleResponse);
    rpc RemoveTerritoryRule(RemoveTerritoryRuleRequest) returns (DeleteResponse);

    // Quota Plans
    rpc CreateQuotaPlan(CreateQuotaPlanRequest) returns (QuotaPlanResponse);
    rpc GetQuotaPlan(GetQuotaPlanRequest) returns (QuotaPlanResponse);
    rpc ListQuotaPlans(ListQuotaPlansRequest) returns (ListQuotaPlansResponse);
    rpc UpdateQuotaPlan(UpdateQuotaPlanRequest) returns (QuotaPlanResponse);
    rpc DeleteQuotaPlan(DeleteQuotaPlanRequest) returns (DeleteResponse);
    rpc SubmitQuotaPlan(SubmitQuotaPlanRequest) returns (QuotaPlanResponse);
    rpc ApproveQuotaPlan(ApproveQuotaPlanRequest) returns (QuotaPlanResponse);

    // Quota Allocations
    rpc CreateQuotaAllocation(CreateQuotaAllocationRequest) returns (QuotaAllocationResponse);
    rpc ListQuotaAllocations(ListQuotaAllocationsRequest) returns (ListQuotaAllocationsResponse);
    rpc UpdateQuotaAllocation(UpdateQuotaAllocationRequest) returns (QuotaAllocationResponse);
    rpc DistributeQuotas(DistributeQuotasRequest) returns (ListQuotaAllocationsResponse);

    // Planning Periods
    rpc CreatePeriod(CreatePeriodRequest) returns (PeriodResponse);
    rpc GetPeriod(GetPeriodRequest) returns (PeriodResponse);
    rpc ListPeriods(ListPeriodsRequest) returns (ListPeriodsResponse);
    rpc UpdatePeriod(UpdatePeriodRequest) returns (PeriodResponse);

    // Metrics
    rpc GetMetrics(GetMetricsRequest) returns (ListMetricsResponse);
    rpc GetAttainment(GetAttainmentRequest) returns (AttainmentResponse);
    rpc GetCoverage(GetCoverageRequest) returns (CoverageResponse);
    rpc RecalculateMetrics(RecalculateMetricsRequest) returns (ListMetricsResponse);

    // Alignment History
    rpc GetAlignmentHistory(GetAlignmentHistoryRequest) returns (ListAlignmentHistoryResponse);
}

message Territory {
    string id = 1;
    string tenant_id = 2;
    string territory_code = 3;
    string territory_name = 4;
    string territory_type = 5;
    string parent_territory_id = 6;
    string geographic_region = 7;
    string country_code = 8;
    string state_province = 9;
    string industry_vertical = 10;
    string product_line = 11;
    string coverage_type = 12;
    string status = 13;
    string effective_start_date = 14;
    string effective_end_date = 15;
    string created_at = 16;
    string updated_at = 17;
    int32 version = 18;
}

message TerritoryMember {
    string id = 1;
    string tenant_id = 2;
    string territory_id = 3;
    string user_id = 4;
    string role = 5;
    int32 allocation_percentage = 6;
    string effective_start_date = 7;
    string effective_end_date = 8;
    string status = 9;
    string created_at = 10;
    string updated_at = 11;
    int32 version = 12;
}

message TerritoryRule {
    string id = 1;
    string tenant_id = 2;
    string territory_id = 3;
    string rule_name = 4;
    string rule_type = 5;
    string rule_condition = 6;
    int32 rule_priority = 7;
    string match_logic = 8;
    string created_at = 9;
    string updated_at = 10;
    int32 version = 11;
}

message QuotaPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    string plan_code = 4;
    string plan_type = 5;
    string period_id = 6;
    int64 total_quota_amount = 7;
    int64 total_quota_units = 8;
    string currency = 9;
    string status = 10;
    string approval_status = 11;
    string approved_by = 12;
    string approved_at = 13;
    string created_at = 14;
    string updated_at = 15;
    int32 version = 16;
}

message QuotaAllocation {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string territory_id = 4;
    string user_id = 5;
    int64 allocated_amount = 6;
    int64 allocated_units = 7;
    int64 attainment_amount = 8;
    int64 attainment_units = 9;
    double attainment_percentage = 10;
    string allocation_method = 11;
    double weight = 12;
    string created_at = 13;
    string updated_at = 14;
    int32 version = 15;
}

message PlanningPeriod {
    string id = 1;
    string tenant_id = 2;
    string period_name = 3;
    string period_type = 4;
    string start_date = 5;
    string end_date = 6;
    int32 fiscal_year = 7;
    int32 fiscal_quarter = 8;
    bool is_current = 9;
    string parent_period_id = 10;
    string created_at = 11;
    string updated_at = 12;
    int32 version = 13;
}

message PlanMetric {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string territory_id = 4;
    string user_id = 5;
    string period_id = 6;
    string metric_type = 7;
    int64 metric_value = 8;
    int64 quota_value = 9;
    double attainment_percentage = 10;
    string calculated_at = 11;
}

message AlignmentHistoryEntry {
    string id = 1;
    string tenant_id = 2;
    string territory_id = 3;
    string realignment_type = 4;
    string previous_state = 5;
    string new_state = 6;
    string changed_by = 7;
    string change_reason = 8;
    string effective_date = 9;
    string created_at = 10;
}

// Request/Response messages
message CreateTerritoryRequest { Territory territory = 1; }
message GetTerritoryRequest { string tenant_id = 1; string id = 2; }
message ListTerritoriesRequest {
    string tenant_id = 1;
    string territory_type = 2;
    string status = 3;
    int32 page = 4;
    int32 page_size = 5;
}
message UpdateTerritoryRequest { Territory territory = 1; }
message DeleteTerritoryRequest { string tenant_id = 1; string id = 2; }
message GetTerritoryHierarchyRequest { string tenant_id = 1; string id = 2; }
message RealignTerritoryRequest {
    string tenant_id = 1;
    string id = 2;
    string change_reason = 3;
    Territory new_state = 4;
    string effective_date = 5;
}
message SplitTerritoryRequest {
    string tenant_id = 1;
    string id = 2;
    repeated Territory new_territories = 3;
    string split_reason = 4;
}
message MergeTerritoriesRequest {
    string tenant_id = 1;
    repeated string territory_ids = 2;
    Territory merged_territory = 3;
    string merge_reason = 4;
}
message EvaluateRulesRequest {
    string tenant_id = 1;
    string account_id = 2;
    string industry = 3;
    string country_code = 4;
    string state_province = 5;
    int64 annual_revenue = 6;
}
message TerritoryResponse { Territory territory = 1; }
message ListTerritoriesResponse { repeated Territory territories = 1; int32 total = 2; }
message TerritoryHierarchyResponse {
    Territory territory = 1;
    repeated TerritoryHierarchyResponse children = 2;
}
message SplitTerritoryResponse { Territory original = 1; repeated Territory new_territories = 2; }
message EvaluateRulesResponse { repeated string matched_territory_ids = 1; }

message AddTerritoryMemberRequest { TerritoryMember member = 1; }
message ListTerritoryMembersRequest { string tenant_id = 1; string territory_id = 2; string status = 3; }
message UpdateTerritoryMemberRequest { TerritoryMember member = 1; }
message RemoveTerritoryMemberRequest { string tenant_id = 1; string id = 2; }
message TerritoryMemberResponse { TerritoryMember member = 1; }
message ListTerritoryMembersResponse { repeated TerritoryMember members = 1; int32 total = 2; }

message AddTerritoryRuleRequest { TerritoryRule rule = 1; }
message ListTerritoryRulesRequest { string tenant_id = 1; string territory_id = 2; }
message UpdateTerritoryRuleRequest { TerritoryRule rule = 1; }
message RemoveTerritoryRuleRequest { string tenant_id = 1; string id = 2; }
message TerritoryRuleResponse { TerritoryRule rule = 1; }
message ListTerritoryRulesResponse { repeated TerritoryRule rules = 1; int32 total = 2; }

message CreateQuotaPlanRequest { QuotaPlan plan = 1; }
message GetQuotaPlanRequest { string tenant_id = 1; string id = 2; }
message ListQuotaPlansRequest {
    string tenant_id = 1;
    string status = 2;
    string plan_type = 3;
    string period_id = 4;
    int32 page = 5;
    int32 page_size = 6;
}
message UpdateQuotaPlanRequest { QuotaPlan plan = 1; }
message DeleteQuotaPlanRequest { string tenant_id = 1; string id = 2; }
message SubmitQuotaPlanRequest { string tenant_id = 1; string id = 2; }
message ApproveQuotaPlanRequest { string tenant_id = 1; string id = 2; bool approve = 3; string reason = 4; }
message QuotaPlanResponse { QuotaPlan plan = 1; }
message ListQuotaPlansResponse { repeated QuotaPlan plans = 1; int32 total = 2; }

message CreateQuotaAllocationRequest { QuotaAllocation allocation = 1; }
message ListQuotaAllocationsRequest { string tenant_id = 1; string plan_id = 2; string territory_id = 3; string user_id = 4; }
message UpdateQuotaAllocationRequest { QuotaAllocation allocation = 1; }
message DistributeQuotasRequest { string tenant_id = 1; string plan_id = 2; string method = 3; }
message QuotaAllocationResponse { QuotaAllocation allocation = 1; }
message ListQuotaAllocationsResponse { repeated QuotaAllocation allocations = 1; int32 total = 2; }

message CreatePeriodRequest { PlanningPeriod period = 1; }
message GetPeriodRequest { string tenant_id = 1; string id = 2; }
message ListPeriodsRequest { string tenant_id = 1; string period_type = 2; int32 fiscal_year = 3; }
message UpdatePeriodRequest { PlanningPeriod period = 1; }
message PeriodResponse { PlanningPeriod period = 1; }
message ListPeriodsResponse { repeated PlanningPeriod periods = 1; int32 total = 2; }

message GetMetricsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string territory_id = 3;
    string user_id = 4;
    string period_id = 5;
}
message GetAttainmentRequest { string tenant_id = 1; string plan_id = 2; string period_id = 3; }
message GetCoverageRequest { string tenant_id = 1; string period_id = 2; }
message RecalculateMetricsRequest { string tenant_id = 1; string plan_id = 2; string period_id = 3; }
message ListMetricsResponse { repeated PlanMetric metrics = 1; int32 total = 2; }
message AttainmentItem { string user_id = 1; string territory_id = 2; int64 quota = 3; int64 attainment = 4; double percentage = 5; }
message AttainmentResponse { repeated AttainmentItem items = 1; }
message CoverageGap { string territory_id = 1; string territory_name = 2; int32 unassigned_accounts = 3; int32 missing_rules = 4; }
message CoverageResponse { repeated CoverageGap gaps = 1; double coverage_percentage = 2; }

message GetAlignmentHistoryRequest { string tenant_id = 1; string territory_id = 2; string effective_date = 3; int32 page = 4; int32 page_size = 5; }
message ListAlignmentHistoryResponse { repeated AlignmentHistoryEntry entries = 1; int32 total = 2; }

message DeleteResponse { bool success = 1; }
```

---

## 6. Inter-Service Integration

| Integration | Service | Direction | Purpose |
|-------------|---------|-----------|---------|
| Sales Automation | `sales-service` | Inbound | Receive opportunity pipeline data, closed-won amounts, and account assignments for attainment calculation |
| Sales Automation | `sales-service` | Outbound | Push territory assignments to leads and accounts; sync territory member list |
| Compensation | `salesperf-service` | Outbound | Publish quota allocation targets for commission plan eligibility and threshold calculation |
| Compensation | `salesperf-service` | Inbound | Receive commission-eligible quota attainment data for incentive correlation |
| Customer Data Platform | `cdp-service` | Inbound | Receive account demographic data (industry, revenue, geography) for territory rule evaluation |
| General Ledger | `gl-service` | Inbound | Receive recognized revenue data for actual vs. quota comparison |
| Identity Service | `identity-service` | Inbound | Validate user roles for territory assignment and plan approval authority |
| Order Management | `om-service` | Inbound | Receive booking amounts for quota attainment tracking |
| Enterprise Planning | `epm-service` | Outbound | Publish quota plans and territory structures for enterprise-wide planning consolidation |
| Enterprise Planning | `epm-service` | Inbound | Receive top-down revenue targets for quota plan initialization |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `TerritoryCreated` | `{ tenant_id, territory_id, territory_code, territory_name, territory_type, coverage_type, created_at }` | Published when a new territory is defined |
| `TerritoryUpdated` | `{ tenant_id, territory_id, territory_code, changed_fields, updated_at }` | Published when territory attributes are modified |
| `TerritoryRealigned` | `{ tenant_id, territory_id, realignment_type, previous_state, new_state, changed_by, effective_date }` | Published when a territory undergoes structural realignment |
| `TerritorySplit` | `{ tenant_id, original_territory_id, new_territory_ids, split_reason, effective_date }` | Published when a territory is split into multiple territories |
| `TerritoriesMerged` | `{ tenant_id, source_territory_ids, merged_territory_id, merge_reason, effective_date }` | Published when multiple territories are merged into one |
| `TerritoryMemberAdded` | `{ tenant_id, territory_id, user_id, role, allocation_percentage, effective_start_date }` | Published when a new member is assigned to a territory |
| `TerritoryMemberRemoved` | `{ tenant_id, territory_id, user_id, removal_reason, effective_date }` | Published when a member is removed from a territory |
| `QuotaPlanCreated` | `{ tenant_id, plan_id, plan_code, plan_name, plan_type, period_id, total_quota_amount, created_at }` | Published when a new quota plan is created |
| `QuotaAllocated` | `{ tenant_id, plan_id, allocation_id, territory_id, user_id, allocated_amount, allocation_method, created_at }` | Published when a quota allocation is created or updated |
| `QuotaPlanSubmitted` | `{ tenant_id, plan_id, plan_code, submitted_by, submitted_at }` | Published when a quota plan is submitted for approval |
| `QuotaPlanApproved` | `{ tenant_id, plan_id, plan_code, approved_by, approved_at }` | Published when a quota plan is approved |
| `QuotaPlanRejected` | `{ tenant_id, plan_id, plan_code, rejected_by, rejection_reason, rejected_at }` | Published when a quota plan approval is rejected |
| `MetricsRecalculated` | `{ tenant_id, plan_id, period_id, territory_count, user_count, recalculated_at }` | Published when plan metrics are recalculated |
| `AttainmentUpdated` | `{ tenant_id, plan_id, allocation_id, user_id, attainment_amount, attainment_percentage, updated_at }` | Published when attainment values are refreshed |
