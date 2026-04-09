# 119 - Strategic Workforce Planning Service Specification

## 1. Domain Overview

The Strategic Workforce Planning service provides long-range workforce planning aligning talent supply with business demand over multi-year horizons. It includes gap analysis between current and future workforce capabilities, supply/demand modeling, retirement projections, skill gap forecasting, and strategic talent investment recommendations. Distinct from Workforce Planning (74) by focusing on multi-year strategic alignment rather than operational headcount management.

**Bounded Context:** Strategic Workforce Planning
**Service Name:** `strategicwf-service`
**Database:** `data/strategicwf.db`
**HTTP Port:** 8156 | **gRPC Port:** 9156

---

## 2. Database Schema

### 2.1 Strategic Plans
```sql
CREATE TABLE strategic_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    planning_horizon_years INTEGER NOT NULL,
    baseline_date TEXT NOT NULL,
    target_date TEXT NOT NULL,
    business_strategy_summary TEXT,
    growth_assumptions TEXT,  -- JSON
    revenue_projections TEXT,  -- JSON
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','UNDER_REVIEW','APPROVED','ARCHIVED')),
    owner_id TEXT NOT NULL,
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, plan_name)
);

CREATE INDEX idx_sp_tenant_status ON strategic_plans(tenant_id, status);
CREATE INDEX idx_sp_tenant_owner ON strategic_plans(tenant_id, owner_id);
CREATE INDEX idx_sp_tenant_horizon ON strategic_plans(tenant_id, planning_horizon_years);
```

### 2.2 Demand Forecasts
```sql
CREATE TABLE demand_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    job_family_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    year INTEGER NOT NULL,
    current_headcount INTEGER NOT NULL DEFAULT 0,
    projected_demand INTEGER NOT NULL DEFAULT 0,
    demand_driver TEXT NOT NULL CHECK(demand_driver IN ('REVENUE_GROWTH','EXPANSION','ATTRITION','RETIREMENT','NEW_BUSINESS','TECHNOLOGY')),
    confidence DECIMAL(5,2) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES strategic_plans(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, plan_id, job_family_id, department_id, location_id, year)
);

CREATE INDEX idx_df_tenant_plan ON demand_forecasts(tenant_id, plan_id);
CREATE INDEX idx_df_tenant_jf ON demand_forecasts(tenant_id, job_family_id);
CREATE INDEX idx_df_tenant_dept ON demand_forecasts(tenant_id, department_id);
CREATE INDEX idx_df_tenant_year ON demand_forecasts(tenant_id, year);
```

### 2.3 Supply Forecasts
```sql
CREATE TABLE supply_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    job_family_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    year INTEGER NOT NULL,
    current_supply INTEGER NOT NULL DEFAULT 0,
    projected_attrition INTEGER NOT NULL DEFAULT 0,
    projected_retirements INTEGER NOT NULL DEFAULT 0,
    projected_promotions_in INTEGER NOT NULL DEFAULT 0,
    projected_promotions_out INTEGER NOT NULL DEFAULT 0,
    projected_hires INTEGER NOT NULL DEFAULT 0,
    projected_transfers INTEGER NOT NULL DEFAULT 0,
    net_supply INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES strategic_plans(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, plan_id, job_family_id, department_id, location_id, year)
);

CREATE INDEX idx_sf_tenant_plan ON supply_forecasts(tenant_id, plan_id);
CREATE INDEX idx_sf_tenant_jf ON supply_forecasts(tenant_id, job_family_id);
CREATE INDEX idx_sf_tenant_dept ON supply_forecasts(tenant_id, department_id);
CREATE INDEX idx_sf_tenant_year ON supply_forecasts(tenant_id, year);
```

### 2.4 Workforce Gaps
```sql
CREATE TABLE workforce_gaps (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    job_family_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    location_id TEXT NOT NULL,
    year INTEGER NOT NULL,
    gap_type TEXT NOT NULL CHECK(gap_type IN ('SHORTAGE','SURPLUS')),
    gap_quantity INTEGER NOT NULL DEFAULT 0,
    gap_percentage DECIMAL(5,2) DEFAULT 0,
    severity TEXT NOT NULL DEFAULT 'MEDIUM' CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    root_cause TEXT,
    recommended_actions TEXT,  -- JSON
    estimated_cost_impact_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES strategic_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_wg_tenant_plan ON workforce_gaps(tenant_id, plan_id);
CREATE INDEX idx_wg_tenant_jf ON workforce_gaps(tenant_id, job_family_id);
CREATE INDEX idx_wg_tenant_dept ON workforce_gaps(tenant_id, department_id);
CREATE INDEX idx_wg_tenant_severity ON workforce_gaps(tenant_id, severity);
CREATE INDEX idx_wg_tenant_year ON workforce_gaps(tenant_id, year);
```

### 2.5 Skill Gap Forecasts
```sql
CREATE TABLE skill_gap_forecasts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    year INTEGER NOT NULL,
    current_proficient_count INTEGER NOT NULL DEFAULT 0,
    required_proficient_count INTEGER NOT NULL DEFAULT 0,
    gap_count INTEGER NOT NULL DEFAULT 0,
    gap_percentage DECIMAL(5,2) DEFAULT 0,
    training_capacity INTEGER NOT NULL DEFAULT 0,
    hiring_feasibility DECIMAL(5,2) DEFAULT 0,
    automation_potential DECIMAL(5,2) DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES strategic_plans(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, plan_id, skill_id, year)
);

CREATE INDEX idx_sgf_tenant_plan ON skill_gap_forecasts(tenant_id, plan_id);
CREATE INDEX idx_sgf_tenant_skill ON skill_gap_forecasts(tenant_id, skill_id);
CREATE INDEX idx_sgf_tenant_year ON skill_gap_forecasts(tenant_id, year);
```

### 2.6 Retirement Projections
```sql
CREATE TABLE retirement_projections (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    department_id TEXT NOT NULL,
    job_family_id TEXT NOT NULL,
    year INTEGER NOT NULL,
    eligible_retirements INTEGER NOT NULL DEFAULT 0,
    expected_retirements INTEGER NOT NULL DEFAULT 0,
    critical_role_retirements INTEGER NOT NULL DEFAULT 0,
    knowledge_transfer_status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(knowledge_transfer_status IN ('PLANNED','IN_PROGRESS','COMPLETE','AT_RISK')),
    succession_readiness TEXT NOT NULL DEFAULT 'NOT_READY' CHECK(succession_readiness IN ('READY','DEVELOPING','NOT_READY')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES strategic_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_rp_tenant_plan ON retirement_projections(tenant_id, plan_id);
CREATE INDEX idx_rp_tenant_dept ON retirement_projections(tenant_id, department_id);
CREATE INDEX idx_rp_tenant_jf ON retirement_projections(tenant_id, job_family_id);
CREATE INDEX idx_rp_tenant_year ON retirement_projections(tenant_id, year);
```

### 2.7 Talent Investments
```sql
CREATE TABLE talent_investments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    investment_type TEXT NOT NULL CHECK(investment_type IN ('HIRE','UPSKILL','RETAIN','AUTOMATE','OUTSOURCE','RESTRUCTURE')),
    target_job_family_id TEXT NOT NULL,
    target_department_id TEXT NOT NULL,
    year INTEGER NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    expected_roi_pct DECIMAL(5,2) DEFAULT 0,
    priority INTEGER NOT NULL DEFAULT 5,
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','APPROVED','IN_PROGRESS','COMPLETED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES strategic_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_ti_tenant_plan ON talent_investments(tenant_id, plan_id);
CREATE INDEX idx_ti_tenant_type ON talent_investments(tenant_id, investment_type);
CREATE INDEX idx_ti_tenant_jf ON talent_investments(tenant_id, target_job_family_id);
CREATE INDEX idx_ti_tenant_status ON talent_investments(tenant_id, status);
CREATE INDEX idx_ti_tenant_priority ON talent_investments(tenant_id, priority);
```

---

## 3. REST API Endpoints

### Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/strategic-plans` | Create a strategic plan |
| GET | `/api/v1/strategic-plans` | List strategic plans |
| GET | `/api/v1/strategic-plans/{id}` | Get plan details |
| PUT | `/api/v1/strategic-plans/{id}` | Update plan |
| POST | `/api/v1/strategic-plans/{id}/submit` | Submit plan for review |
| POST | `/api/v1/strategic-plans/{id}/approve` | Approve strategic plan |

### Demand Forecasts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/demand-forecasts` | Create a demand forecast |
| GET | `/api/v1/demand-forecasts` | List demand forecasts |
| GET | `/api/v1/demand-forecasts/{id}` | Get forecast details |
| PUT | `/api/v1/demand-forecasts/{id}` | Update forecast |
| POST | `/api/v1/strategic-plans/{id}/generate-demand` | Auto-generate demand forecasts |
| GET | `/api/v1/demand-forecasts/by-job-family/{jobFamilyId}` | Get forecasts by job family |

### Supply Forecasts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/supply-forecasts` | Create a supply forecast |
| GET | `/api/v1/supply-forecasts` | List supply forecasts |
| GET | `/api/v1/supply-forecasts/{id}` | Get forecast details |
| PUT | `/api/v1/supply-forecasts/{id}` | Update forecast |
| POST | `/api/v1/strategic-plans/{id}/generate-supply` | Auto-generate supply forecasts |
| GET | `/api/v1/supply-forecasts/net-supply` | Get net supply calculations |

### Gaps
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/workforce-gaps/analysis` | Get gap analysis for a plan |
| GET | `/api/v1/workforce-gaps/by-year` | Get gaps by year |
| GET | `/api/v1/workforce-gaps/by-department` | Get gaps by department |
| GET | `/api/v1/workforce-gaps/by-severity` | Get gaps by severity level |

### Skill Gaps
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/skill-gaps/forecast` | Get skill gap forecasts |
| GET | `/api/v1/skill-gaps/by-skill/{skillId}` | Get gaps by skill |
| POST | `/api/v1/skill-gaps/recommend-actions` | Get recommended actions for skill gaps |

### Retirement Projections
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/retirement-projections/by-year` | Get projections by year |
| GET | `/api/v1/retirement-projections/critical-roles` | Get critical role retirement risks |
| GET | `/api/v1/retirement-projections/knowledge-transfer-status` | Get knowledge transfer status |

### Investments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/talent-investments` | Create a talent investment |
| GET | `/api/v1/talent-investments` | List talent investments |
| GET | `/api/v1/talent-investments/{id}` | Get investment details |
| PUT | `/api/v1/talent-investments/{id}` | Update investment |
| POST | `/api/v1/talent-investments/prioritize` | Prioritize investment portfolio |
| GET | `/api/v1/talent-investments/roi-analysis` | Get ROI analysis for investments |

---

## 4. Business Rules

1. A strategic plan MUST have a planning_horizon_years value between 1 and 10.
2. Demand forecast confidence MUST be calculated as a weighted score based on data quality, historical accuracy, and forecast methodology.
3. The system MUST calculate net_supply as current_supply minus projected_attrition minus projected_retirements plus projected_promotions_in minus projected_promotions_out plus projected_hires plus projected_transfers.
4. Workforce gap severity MUST be classified as CRITICAL when gap_percentage exceeds 25, HIGH when exceeding 15, MEDIUM when exceeding 5, and LOW otherwise.
5. A plan in APPROVED status MUST NOT have its demand or supply forecasts modified without creating a new plan version.
6. Retirement projections MUST only include employees who will reach the organization's retirement eligibility age within the planning horizon.
7. Skill gap forecasts with automation_potential above 70 percent SHOULD generate an AUTOMATE investment recommendation.
8. Talent investment priority MUST be calculated based on gap severity, cost impact, and expected ROI.
9. The system MUST prevent deletion of demand or supply forecasts for a plan in ACTIVE or APPROVED status.
10. Knowledge transfer status for retirement projections MUST be flagged as AT_RISK when expected_retirements exceed 0 and succession_readiness is NOT_READY.
11. The system SHOULD generate recommended_actions for workforce gaps based on gap_type, severity, and available investment options.
12. All monetary values in talent_investments MUST be stored in cents as INTEGER values.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package strategicwf.v1;

service StrategicWorkforcePlanningService {
    // Plans
    rpc CreateStrategicPlan(CreateStrategicPlanRequest) returns (CreateStrategicPlanResponse);
    rpc GetStrategicPlan(GetStrategicPlanRequest) returns (GetStrategicPlanResponse);
    rpc ListStrategicPlans(ListStrategicPlansRequest) returns (ListStrategicPlansResponse);
    rpc SubmitPlan(SubmitPlanRequest) returns (SubmitPlanResponse);
    rpc ApprovePlan(ApprovePlanRequest) returns (ApprovePlanResponse);

    // Demand Forecasts
    rpc CreateDemandForecast(CreateDemandForecastRequest) returns (CreateDemandForecastResponse);
    rpc ListDemandForecasts(ListDemandForecastsRequest) returns (ListDemandForecastsResponse);
    rpc GenerateDemandForecasts(GenerateDemandForecastsRequest) returns (GenerateDemandForecastsResponse);

    // Supply Forecasts
    rpc CreateSupplyForecast(CreateSupplyForecastRequest) returns (CreateSupplyForecastResponse);
    rpc ListSupplyForecasts(ListSupplyForecastsRequest) returns (ListSupplyForecastsResponse);
    rpc GenerateSupplyForecasts(GenerateSupplyForecastsRequest) returns (GenerateSupplyForecastsResponse);
    rpc GetNetSupply(GetNetSupplyRequest) returns (GetNetSupplyResponse);

    // Gaps
    rpc GetGapAnalysis(GetGapAnalysisRequest) returns (GetGapAnalysisResponse);
    rpc GetGapsByYear(GetGapsByYearRequest) returns (GetGapsByYearResponse);
    rpc GetGapsByDepartment(GetGapsByDepartmentRequest) returns (GetGapsByDepartmentResponse);
    rpc GetGapsBySeverity(GetGapsBySeverityRequest) returns (GetGapsBySeverityResponse);

    // Skill Gaps
    rpc GetSkillGapForecast(GetSkillGapForecastRequest) returns (GetSkillGapForecastResponse);
    rpc GetSkillGapsBySkill(GetSkillGapsBySkillRequest) returns (GetSkillGapsBySkillResponse);
    rpc RecommendSkillActions(RecommendSkillActionsRequest) returns (RecommendSkillActionsResponse);

    // Retirement Projections
    rpc GetRetirementsByYear(GetRetirementsByYearRequest) returns (GetRetirementsByYearResponse);
    rpc GetCriticalRoleRetirements(GetCriticalRoleRetirementsRequest) returns (GetCriticalRoleRetirementsResponse);
    rpc GetKnowledgeTransferStatus(GetKnowledgeTransferStatusRequest) returns (GetKnowledgeTransferStatusResponse);

    // Investments
    rpc CreateTalentInvestment(CreateTalentInvestmentRequest) returns (CreateTalentInvestmentResponse);
    rpc ListTalentInvestments(ListTalentInvestmentsRequest) returns (ListTalentInvestmentsResponse);
    rpc PrioritizeInvestments(PrioritizeInvestmentsRequest) returns (PrioritizeInvestmentsResponse);
    rpc GetROIAnalysis(GetROIAnalysisRequest) returns (GetROIAnalysisResponse);
}

message StrategicPlan {
    string id = 1;
    string tenant_id = 2;
    string plan_name = 3;
    int32 planning_horizon_years = 4;
    string baseline_date = 5;
    string target_date = 6;
    string business_strategy_summary = 7;
    string growth_assumptions = 8;
    string revenue_projections = 9;
    string status = 10;
    string owner_id = 11;
    string approved_by = 12;
    string approved_at = 13;
}

message CreateStrategicPlanRequest {
    string tenant_id = 1;
    string plan_name = 2;
    int32 planning_horizon_years = 3;
    string baseline_date = 4;
    string target_date = 5;
    string business_strategy_summary = 6;
    string growth_assumptions = 7;
    string revenue_projections = 8;
    string owner_id = 9;
    string created_by = 10;
}

message CreateStrategicPlanResponse {
    StrategicPlan plan = 1;
}

message GetStrategicPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetStrategicPlanResponse {
    StrategicPlan plan = 1;
}

message ListStrategicPlansRequest {
    string tenant_id = 1;
    string status = 2;
    string owner_id = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListStrategicPlansResponse {
    repeated StrategicPlan plans = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SubmitPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string submitted_by = 3;
}

message SubmitPlanResponse {
    StrategicPlan plan = 1;
}

message ApprovePlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string approved_by = 3;
}

message ApprovePlanResponse {
    StrategicPlan plan = 1;
}

message DemandForecast {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string job_family_id = 4;
    string department_id = 5;
    string location_id = 6;
    int32 year = 7;
    int32 current_headcount = 8;
    int32 projected_demand = 9;
    string demand_driver = 10;
    double confidence = 11;
}

message CreateDemandForecastRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string job_family_id = 3;
    string department_id = 4;
    string location_id = 5;
    int32 year = 6;
    int32 current_headcount = 7;
    int32 projected_demand = 8;
    string demand_driver = 9;
    string created_by = 10;
}

message CreateDemandForecastResponse {
    DemandForecast forecast = 1;
}

message ListDemandForecastsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string job_family_id = 3;
    string department_id = 4;
    int32 year = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListDemandForecastsResponse {
    repeated DemandForecast forecasts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GenerateDemandForecastsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string generated_by = 3;
}

message GenerateDemandForecastsResponse {
    repeated DemandForecast forecasts = 1;
    int32 total_generated = 2;
}

message SupplyForecast {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string job_family_id = 4;
    string department_id = 5;
    string location_id = 6;
    int32 year = 7;
    int32 current_supply = 8;
    int32 projected_attrition = 9;
    int32 projected_retirements = 10;
    int32 projected_promotions_in = 11;
    int32 projected_promotions_out = 12;
    int32 projected_hires = 13;
    int32 projected_transfers = 14;
    int32 net_supply = 15;
}

message CreateSupplyForecastRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string job_family_id = 3;
    string department_id = 4;
    string location_id = 5;
    int32 year = 6;
    int32 current_supply = 7;
    int32 projected_attrition = 8;
    int32 projected_retirements = 9;
    int32 projected_promotions_in = 10;
    int32 projected_promotions_out = 11;
    int32 projected_hires = 12;
    int32 projected_transfers = 13;
    string created_by = 14;
}

message CreateSupplyForecastResponse {
    SupplyForecast forecast = 1;
}

message ListSupplyForecastsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string job_family_id = 3;
    string department_id = 4;
    int32 year = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListSupplyForecastsResponse {
    repeated SupplyForecast forecasts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GenerateSupplyForecastsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string generated_by = 3;
}

message GenerateSupplyForecastsResponse {
    repeated SupplyForecast forecasts = 1;
    int32 total_generated = 2;
}

message GetNetSupplyRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string job_family_id = 3;
    string department_id = 4;
    int32 year = 5;
}

message GetNetSupplyResponse {
    int32 total_demand = 1;
    int32 total_net_supply = 2;
    int32 net_gap = 3;
    string gap_type = 4;
}

message WorkforceGap {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string job_family_id = 4;
    string department_id = 5;
    string location_id = 6;
    int32 year = 7;
    string gap_type = 8;
    int32 gap_quantity = 9;
    double gap_percentage = 10;
    string severity = 11;
    string root_cause = 12;
    int64 estimated_cost_impact_cents = 13;
}

message GetGapAnalysisRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetGapAnalysisResponse {
    repeated WorkforceGap gaps = 1;
    int32 total_shortage_positions = 2;
    int32 total_surplus_positions = 3;
    int64 total_cost_impact_cents = 4;
    repeated GapSummaryByYear yearly_summary = 5;
}

message GapSummaryByYear {
    int32 year = 1;
    int32 shortage_count = 2;
    int32 surplus_count = 3;
    int64 cost_impact_cents = 4;
}

message GetGapsByYearRequest {
    string tenant_id = 1;
    string plan_id = 2;
    int32 year = 3;
}

message GetGapsByYearResponse {
    repeated WorkforceGap gaps = 1;
}

message GetGapsByDepartmentRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string department_id = 3;
}

message GetGapsByDepartmentResponse {
    repeated WorkforceGap gaps = 1;
}

message GetGapsBySeverityRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string severity = 3;
}

message GetGapsBySeverityResponse {
    repeated WorkforceGap gaps = 1;
}

message SkillGapForecast {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string skill_id = 4;
    int32 year = 5;
    int32 current_proficient_count = 6;
    int32 required_proficient_count = 7;
    int32 gap_count = 8;
    double gap_percentage = 9;
    int32 training_capacity = 10;
    double hiring_feasibility = 11;
    double automation_potential = 12;
}

message GetSkillGapForecastRequest {
    string tenant_id = 1;
    string plan_id = 2;
    int32 year = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetSkillGapForecastResponse {
    repeated SkillGapForecast forecasts = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetSkillGapsBySkillRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string skill_id = 3;
}

message GetSkillGapsBySkillResponse {
    repeated SkillGapForecast forecasts = 1;
}

message RecommendSkillActionsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string skill_id = 3;
    int32 year = 4;
}

message RecommendSkillActionsResponse {
    repeated SkillActionRecommendation recommendations = 1;
}

message SkillActionRecommendation {
    string action_type = 1;
    string description = 2;
    int32 target_count = 3;
    int64 estimated_cost_cents = 4;
    double feasibility_score = 5;
    int32 timeframe_months = 6;
}

message RetirementProjection {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string department_id = 4;
    string job_family_id = 5;
    int32 year = 6;
    int32 eligible_retirements = 7;
    int32 expected_retirements = 8;
    int32 critical_role_retirements = 9;
    string knowledge_transfer_status = 10;
    string succession_readiness = 11;
}

message GetRetirementsByYearRequest {
    string tenant_id = 1;
    string plan_id = 2;
    int32 year = 3;
}

message GetRetirementsByYearResponse {
    repeated RetirementProjection projections = 1;
}

message GetCriticalRoleRetirementsRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetCriticalRoleRetirementsResponse {
    repeated RetirementProjection projections = 1;
    int32 total_critical_at_risk = 2;
}

message GetKnowledgeTransferStatusRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetKnowledgeTransferStatusResponse {
    int32 planned_count = 1;
    int32 in_progress_count = 2;
    int32 complete_count = 3;
    int32 at_risk_count = 4;
    repeated RetirementProjection at_risk_projections = 5;
}

message TalentInvestment {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string investment_type = 4;
    string target_job_family_id = 5;
    string target_department_id = 6;
    int32 year = 7;
    int32 quantity = 8;
    int64 cost_cents = 9;
    double expected_roi_pct = 10;
    int32 priority = 11;
    string status = 12;
}

message CreateTalentInvestmentRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string investment_type = 3;
    string target_job_family_id = 4;
    string target_department_id = 5;
    int32 year = 6;
    int32 quantity = 7;
    int64 cost_cents = 8;
    double expected_roi_pct = 9;
    int32 priority = 10;
    string created_by = 11;
}

message CreateTalentInvestmentResponse {
    TalentInvestment investment = 1;
}

message ListTalentInvestmentsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string investment_type = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListTalentInvestmentsResponse {
    repeated TalentInvestment investments = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message PrioritizeInvestmentsRequest {
    string tenant_id = 1;
    string plan_id = 2;
    string prioritized_by = 3;
}

message PrioritizeInvestmentsResponse {
    repeated TalentInvestment prioritized_investments = 1;
    int64 total_cost_cents = 2;
    double weighted_avg_roi_pct = 3;
}

message GetROIAnalysisRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetROIAnalysisResponse {
    int64 total_investment_cents = 1;
    int64 total_expected_return_cents = 2;
    double overall_roi_pct = 3;
    repeated InvestmentROIItem items = 4;
}

message InvestmentROIItem {
    string investment_id = 1;
    string investment_type = 2;
    int64 cost_cents = 3;
    double expected_roi_pct = 4;
    int64 expected_return_cents = 5;
    int32 priority = 6;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee headcount, demographics, job families | Baseline workforce data |
| `workforce-planning-service` | Operational headcount plans | Validate against operational plans |
| `skills-service` | Skill inventories, proficiency levels | Populate skill gap analysis |
| `finance-service` | Revenue projections, budget constraints | Drive demand forecasting |
| `org-service` | Department structures, location hierarchies | Organizational dimension data |
| `compensation-service` | Salary benchmarks, cost models | Calculate investment costs |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Strategic headcount targets, hiring plans | Inform recruiting strategy |
| `recruiting-service` | Long-range hiring demand | Strategic recruiting pipeline |
| `learning-service` | Skill development targets, upskilling plans | Training program planning |
| `compensation-service` | Workforce cost projections | Budget planning |
| `reporting-service` | Gap analysis reports, investment ROI | Executive dashboards |
| `succession-service` | Retirement risks, succession gaps | Succession planning triggers |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `StrategicPlanCreated` | `strategicwf.plan.created` | `{ tenant_id, plan_id, plan_name, planning_horizon_years, baseline_date, target_date, owner_id }` | Published when a new strategic plan is created |
| `GapAnalysisCompleted` | `strategicwf.gap-analysis.completed` | `{ tenant_id, plan_id, total_shortage_positions, total_surplus_positions, total_cost_impact_cents, critical_gaps }` | Published when a gap analysis run completes |
| `RetirementProjectionUpdated` | `strategicwf.retirement.updated` | `{ tenant_id, plan_id, department_id, job_family_id, year, eligible_retirements, critical_role_retirements, knowledge_transfer_status }` | Published when retirement projections are recalculated |
| `InvestmentApproved` | `strategicwf.investment.approved` | `{ tenant_id, plan_id, investment_id, investment_type, target_job_family_id, quantity, cost_cents, expected_roi_pct }` | Published when a talent investment is approved |
