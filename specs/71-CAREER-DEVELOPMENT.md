# 71 - Career Development Service Specification

## 1. Domain Overview

The Career Development service manages employee career growth through skills taxonomy management, employee skill assessment, career path mapping, development planning, skill gap analysis, and AI-driven career recommendations. It enables employees to explore career options and create actionable development plans aligned with organizational talent needs.

**Bounded Context:** Career Growth & Skill Management
**Service Name:** `career-service`
**Database:** `data/career.db`
**HTTP Port:** 8103 | **gRPC Port:** 9103

---

## 2. Database Schema

### 2.1 Skills Taxonomy
```sql
CREATE TABLE skills_taxonomy (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    skill_name TEXT NOT NULL,
    skill_code TEXT NOT NULL,
    skill_category TEXT NOT NULL CHECK(skill_category IN ('TECHNICAL','FUNCTIONAL','LEADERSHIP','BEHAVIORAL','DIGITAL','DOMAIN','CERTIFICATION','LANGUAGE','TOOL','PROCESS')),
    sub_category TEXT,
    description TEXT,
    parent_skill_id TEXT,
    difficulty_level TEXT DEFAULT 'INTERMEDIATE' CHECK(difficulty_level IN ('BEGINNER','INTERMEDIATE','ADVANCED','EXPERT')),
    assessable INTEGER DEFAULT 1,
    external_reference TEXT,
    market_demand TEXT DEFAULT 'MEDIUM' CHECK(market_demand IN ('LOW','MEDIUM','HIGH','VERY_HIGH')),
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','DRAFT','DEPRECATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, skill_code)
);

CREATE INDEX idx_skills_tax_tenant_category ON skills_taxonomy(tenant_id, skill_category);
CREATE INDEX idx_skills_tax_tenant_parent ON skills_taxonomy(tenant_id, parent_skill_id);
CREATE INDEX idx_skills_tax_tenant_demand ON skills_taxonomy(tenant_id, market_demand);
```

### 2.2 Skill Proficiency Levels
```sql
CREATE TABLE skill_proficiency_levels (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    proficiency_level INTEGER NOT NULL CHECK(proficiency_level BETWEEN 1 AND 5),
    level_name TEXT NOT NULL,
    description TEXT NOT NULL,
    assessment_criteria TEXT,
    behavioral_indicators TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES skills_taxonomy(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, skill_id, proficiency_level)
);

CREATE INDEX idx_proficiency_tenant_skill ON skill_proficiency_levels(tenant_id, skill_id);
```

### 2.3 Employee Skills
```sql
CREATE TABLE employee_skills (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    proficiency_level INTEGER NOT NULL CHECK(proficiency_level BETWEEN 1 AND 5),
    source TEXT NOT NULL DEFAULT 'SELF_ASSESSMENT' CHECK(source IN ('SELF_ASSESSMENT','MANAGER_ASSESSMENT','ASSESSMENT_TEST','CERTIFICATION','COURSE_COMPLETION','PEER_REVIEW','SYSTEM_INFERRED')),
    assessed_date TEXT NOT NULL,
    assessed_by TEXT,
    expiry_date TEXT,
    years_of_experience DECIMAL(4,1),
    last_used_date TEXT,
    interest_level TEXT CHECK(interest_level IN ('LOW','MEDIUM','HIGH')),
    endorsement_count INTEGER DEFAULT 0,
    verification_status TEXT DEFAULT 'UNVERIFIED' CHECK(verification_status IN ('UNVERIFIED','VERIFIED','DISPUTED')),
    evidence TEXT,  -- JSON: supporting evidence references

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES skills_taxonomy(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, employee_id, skill_id)
);

CREATE INDEX idx_emp_skills_tenant_employee ON employee_skills(tenant_id, employee_id);
CREATE INDEX idx_emp_skills_tenant_skill ON employee_skills(tenant_id, skill_id);
CREATE INDEX idx_emp_skills_tenant_proficiency ON employee_skills(tenant_id, proficiency_level);
```

### 2.4 Skill Gap Analyses
```sql
CREATE TABLE skill_gap_analyses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    target_role_id TEXT,
    target_career_path_id TEXT,
    analysis_date TEXT NOT NULL,
    current_skill_profile TEXT NOT NULL,   -- JSON: { skill_id: proficiency }
    required_skill_profile TEXT NOT NULL,  -- JSON: { skill_id: required_proficiency }
    gap_details TEXT NOT NULL,             -- JSON: { skill_id: { current, required, gap } }
    total_skills_assessed INTEGER DEFAULT 0,
    skills_met INTEGER DEFAULT 0,
    skills_gap INTEGER DEFAULT 0,
    overall_readiness_percentage DECIMAL(5,2) DEFAULT 0,
    critical_gaps TEXT,  -- JSON array of critical skill gaps
    recommended_actions TEXT,  -- JSON array of development recommendations
    status TEXT DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','ARCHIVED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);

CREATE INDEX idx_skill_gaps_tenant_employee ON skill_gap_analyses(tenant_id, employee_id);
CREATE INDEX idx_skill_gaps_tenant_role ON skill_gap_analyses(tenant_id, target_role_id);
```

### 2.5 Career Paths
```sql
CREATE TABLE career_paths (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    path_name TEXT NOT NULL,
    path_code TEXT NOT NULL,
    path_type TEXT NOT NULL CHECK(path_type IN ('INDIVIDUAL_CONTRIBUTOR','MANAGEMENT','TECHNICAL','CROSS_FUNCTIONAL','LATERAL','DUAL_TRACK')),
    description TEXT,
    start_job_id TEXT,
    target_job_id TEXT,
    target_grade_id TEXT,
    estimated_duration_months INTEGER,
    milestones TEXT NOT NULL,  -- JSON array of milestone definitions
    required_skills TEXT NOT NULL,  -- JSON: { skill_id: required_proficiency }
    recommended_courses TEXT,  -- JSON array of course IDs
    prerequisite_paths TEXT,  -- JSON array of path IDs
    status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','INACTIVE','DRAFT')),
    popularity INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, path_code)
);

CREATE INDEX idx_career_paths_tenant_type ON career_paths(tenant_id, path_type);
CREATE INDEX idx_career_paths_tenant_target ON career_paths(tenant_id, target_job_id);
```

### 2.6 Career Milestones
```sql
CREATE TABLE career_milestones (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    career_path_id TEXT NOT NULL,
    milestone_name TEXT NOT NULL,
    milestone_type TEXT NOT NULL CHECK(milestone_type IN ('PROMOTION','CERTIFICATION','SKILL_ACQUISITION','PROJECT_COMPLETION','ROLE_CHANGE','EDUCATION','EXPERIENCE','ASSESSMENT')),
    target_date TEXT,
    achieved_date TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','IN_PROGRESS','ACHIEVED','MISSED','CANCELLED')),
    description TEXT,
    evidence TEXT,
    sequence_order INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (career_path_id) REFERENCES career_paths(id) ON DELETE CASCADE
);

CREATE INDEX idx_milestones_tenant_employee ON career_milestones(tenant_id, employee_id, career_path_id);
CREATE INDEX idx_milestones_tenant_status ON career_milestones(tenant_id, status);
```

### 2.7 Development Plans
```sql
CREATE TABLE development_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('CAREER','SKILL_GAP','SUCCESSION','PERFORMANCE','ONBOARDING','CUSTOM')),
    career_path_id TEXT,
    target_role_id TEXT,
    start_date TEXT NOT NULL,
    target_end_date TEXT NOT NULL,
    actual_end_date TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','ACTIVE','COMPLETED','CANCELLED','ON_HOLD')),
    progress_percentage DECIMAL(5,2) DEFAULT 0,
    manager_id TEXT,
    mentor_id TEXT,
    overall_objective TEXT,
    success_criteria TEXT,
    review_frequency TEXT DEFAULT 'MONTHLY' CHECK(review_frequency IN ('WEEKLY','BIWEEKLY','MONTHLY','QUARTERLY')),
    last_review_date TEXT,
    next_review_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (career_path_id) REFERENCES career_paths(id) ON DELETE SET NULL
);

CREATE INDEX idx_dev_plans_tenant_employee ON development_plans(tenant_id, employee_id, status);
CREATE INDEX idx_dev_plans_tenant_status ON development_plans(tenant_id, status);
CREATE INDEX idx_dev_plans_tenant_dates ON development_plans(tenant_id, target_end_date);
```

### 2.8 Development Activities
```sql
CREATE TABLE development_activities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    activity_type TEXT NOT NULL CHECK(activity_type IN ('COURSE','CERTIFICATION','MENTORING','COACHING','JOB_ROTATION','STRETCH_ASSIGNMENT','PROJECT','NETWORKING','READING','CONFERENCE','FEEDBACK_SESSION','SELF_STUDY')),
    title TEXT NOT NULL,
    description TEXT,
    target_skill_id TEXT,
    target_competency TEXT,
    start_date TEXT NOT NULL,
    target_completion_date TEXT NOT NULL,
    actual_completion_date TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','CANCELLED','OVERDUE')),
    progress_percentage DECIMAL(5,2) DEFAULT 0,
    priority TEXT DEFAULT 'MEDIUM' CHECK(priority IN ('HIGH','MEDIUM','LOW')),
    resource_link TEXT,
    responsible_person_id TEXT,
    outcome TEXT,
    notes TEXT,
    linked_enrollment_id TEXT,
    linked_certification_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (plan_id) REFERENCES development_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_dev_activities_tenant_plan ON development_activities(tenant_id, plan_id);
CREATE INDEX idx_dev_activities_tenant_status ON development_activities(tenant_id, status);
CREATE INDEX idx_dev_activities_tenant_type ON development_activities(tenant_id, activity_type);
```

### 2.9 Career Preferences
```sql
CREATE TABLE career_preferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    preferred_career_direction TEXT NOT NULL CHECK(preferred_career_direction IN ('MANAGEMENT','TECHNICAL','SPECIALIST','ENTREPRENEURIAL','UNDECIDED')),
    preferred_industries TEXT,     -- JSON array
    preferred_locations TEXT,      -- JSON array
    willing_to_relocate INTEGER DEFAULT 0,
    willing_to_travel INTEGER DEFAULT 0,
    interested_in_management INTEGER DEFAULT 0,
    work_style_preference TEXT CHECK(work_style_preference IN ('ONSITE','REMOTE','HYBRID','FLEXIBLE')),
    salary_expectations INTEGER,   -- cents
    desired_work_life_balance TEXT DEFAULT 'BALANCED' CHECK(desired_work_life_balance IN ('WORK_LIFE_BALANCED','CAREER_FOCUSED','LIFESTYLE_FOCUSED')),
    short_term_goals TEXT,
    long_term_goals TEXT,
    last_updated TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, employee_id)
);

CREATE INDEX idx_career_prefs_tenant_employee ON career_preferences(tenant_id, employee_id);
```

### 2.10 Skill Recommendations
```sql
CREATE TABLE skill_recommendations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    recommended_skill_id TEXT NOT NULL,
    recommendation_type TEXT NOT NULL CHECK(recommendation_type IN ('CAREER_PATH','SKILL_GAP','MARKET_DEMAND','PEER_INSPIRED','AI_SUGGESTED','ORGANIZATIONAL_NEED')),
    relevance_score DECIMAL(5,2),
    reason TEXT,
    target_proficiency INTEGER,
    estimated_effort_hours DECIMAL(7,2),
    recommended_resources TEXT,  -- JSON array of resource links
    source_career_path_id TEXT,
    generated_date TEXT NOT NULL,
    status TEXT DEFAULT 'ACTIVE' CHECK(status IN ('ACTIVE','DISMISSED','ACCEPTED','IN_PROGRESS')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (recommended_skill_id) REFERENCES skills_taxonomy(id) ON DELETE CASCADE
);

CREATE INDEX idx_skill_recs_tenant_employee ON skill_recommendations(tenant_id, employee_id, status);
CREATE INDEX idx_skill_recs_tenant_skill ON skill_recommendations(tenant_id, recommended_skill_id);
```

---

## 3. REST API Endpoints

### Skills Management
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/skills` | Create a skill in taxonomy |
| GET | `/api/v1/skills` | List skills with filters |
| GET | `/api/v1/skills/{id}` | Get skill details |
| PUT | `/api/v1/skills/{id}` | Update skill |
| GET | `/api/v1/skills/tree` | Get skills taxonomy tree |
| POST | `/api/v1/skills/{id}/proficiency-levels` | Define proficiency levels |

### Employee Skills
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees/{employeeId}/skills` | Assess employee skill |
| GET | `/api/v1/employees/{employeeId}/skills` | List employee skills |
| PUT | `/api/v1/employee-skills/{id}` | Update skill assessment |
| DELETE | `/api/v1/employee-skills/{id}` | Remove skill assessment |
| GET | `/api/v1/employees/{employeeId}/skill-profile` | Get full skill profile |

### Skill Gap Analysis
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/skill-gap-analysis` | Run skill gap analysis |
| GET | `/api/v1/skill-gap-analysis/{id}` | Get analysis results |
| GET | `/api/v1/employees/{employeeId}/skill-gaps` | List employee gap analyses |

### Career Paths
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/career-paths` | Create career path |
| GET | `/api/v1/career-paths` | List career paths |
| GET | `/api/v1/career-paths/{id}` | Get path details |
| PUT | `/api/v1/career-paths/{id}` | Update career path |
| GET | `/api/v1/career-paths/{id}/progression` | Get path progression details |
| GET | `/api/v1/employees/{employeeId}/eligible-paths` | Get eligible paths for employee |

### Milestones
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees/{employeeId}/milestones` | Create career milestone |
| GET | `/api/v1/employees/{employeeId}/milestones` | List milestones |
| PUT | `/api/v1/milestones/{id}` | Update milestone |
| POST | `/api/v1/milestones/{id}/achieve` | Mark milestone achieved |

### Development Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/development-plans` | Create development plan |
| GET | `/api/v1/development-plans` | List development plans |
| GET | `/api/v1/development-plans/{id}` | Get plan with activities |
| PUT | `/api/v1/development-plans/{id}` | Update plan |
| POST | `/api/v1/development-plans/{id}/activate` | Activate plan |
| GET | `/api/v1/employees/{employeeId}/development-plans` | Get employee plans |

### Development Activities
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/development-plans/{planId}/activities` | Add activity to plan |
| GET | `/api/v1/development-plans/{planId}/activities` | List plan activities |
| PUT | `/api/v1/activities/{id}` | Update activity |
| POST | `/api/v1/activities/{id}/complete` | Mark activity complete |

### Career Preferences
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/employees/{employeeId}/career-preferences` | Set career preferences |
| GET | `/api/v1/employees/{employeeId}/career-preferences` | Get career preferences |
| PUT | `/api/v1/career-preferences/{id}` | Update preferences |

### Recommendations
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/recommendations/generate` | Generate skill recommendations |
| GET | `/api/v1/employees/{employeeId}/recommendations` | Get employee recommendations |
| POST | `/api/v1/recommendations/{id}/accept` | Accept recommendation |
| POST | `/api/v1/recommendations/{id}/dismiss` | Dismiss recommendation |

---

## 4. Business Rules

1. A skill in the taxonomy MUST have a unique code within a tenant.
2. Employee skill proficiency MUST be between 1 and the maximum defined proficiency level.
3. A skill gap analysis MUST compare current proficiency against the target role's required proficiency.
4. Career paths MUST define required skills for each stage of progression.
5. Development plan activities SHOULD map to specific skill gaps identified in analysis.
6. An employee MUST NOT have duplicate skill entries for the same skill.
7. Skill recommendations MUST include a relevance score and a reason for the recommendation.
8. Career milestones MUST have a target date and MUST be ordered sequentially within a path.
9. Development plans MUST have at least one activity before activation.
10. The system SHOULD generate recommendations based on career preferences, market demand, and organizational needs.
11. Proficiency levels MUST be defined for all assessable skills.
12. Career preference updates MUST trigger re-evaluation of eligible career paths.
13. Development plan progress MUST be auto-calculated based on activity completion.
14. Skill endorsements from peers MUST NOT change proficiency level without assessment validation.
15. Career paths marked as `INACTIVE` MUST NOT appear in employee recommendations.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package career.v1;

service CareerService {
    // Skills
    rpc CreateSkill(CreateSkillRequest) returns (CreateSkillResponse);
    rpc GetSkill(GetSkillRequest) returns (GetSkillResponse);
    rpc ListSkills(ListSkillsRequest) returns (ListSkillsResponse);

    // Employee skills
    rpc AssessEmployeeSkill(AssessEmployeeSkillRequest) returns (AssessEmployeeSkillResponse);
    rpc GetEmployeeSkillProfile(GetEmployeeSkillProfileRequest) returns (GetEmployeeSkillProfileResponse);

    // Skill gaps
    rpc AnalyzeSkillGap(AnalyzeSkillGapRequest) returns (AnalyzeSkillGapResponse);
    rpc GetSkillGapAnalysis(GetSkillGapAnalysisRequest) returns (GetSkillGapAnalysisResponse);

    // Career paths
    rpc CreateCareerPath(CreateCareerPathRequest) returns (CreateCareerPathResponse);
    rpc GetCareerPath(GetCareerPathRequest) returns (GetCareerPathResponse);
    rpc ListCareerPaths(ListCareerPathsRequest) returns (ListCareerPathsResponse);
    rpc GetEligiblePaths(GetEligiblePathsRequest) returns (GetEligiblePathsResponse);

    // Milestones
    rpc CreateMilestone(CreateMilestoneRequest) returns (CreateMilestoneResponse);
    rpc AchieveMilestone(AchieveMilestoneRequest) returns (AchieveMilestoneResponse);

    // Development plans
    rpc CreateDevelopmentPlan(CreateDevelopmentPlanRequest) returns (CreateDevelopmentPlanResponse);
    rpc GetDevelopmentPlan(GetDevelopmentPlanRequest) returns (GetDevelopmentPlanResponse);
    rpc ListDevelopmentPlans(ListDevelopmentPlansRequest) returns (ListDevelopmentPlansResponse);

    // Recommendations
    rpc GenerateRecommendations(GenerateRecommendationsRequest) returns (GenerateRecommendationsResponse);
    rpc GetRecommendations(GetRecommendationsRequest) returns (GetRecommendationsResponse);
}

message Skill {
    string id = 1;
    string tenant_id = 2;
    string skill_name = 3;
    string skill_code = 4;
    string skill_category = 5;
    string difficulty_level = 6;
    string market_demand = 7;
}

message CreateSkillRequest {
    string tenant_id = 1;
    string skill_name = 2;
    string skill_code = 3;
    string skill_category = 4;
    string description = 5;
    string created_by = 6;
}

message CreateSkillResponse {
    Skill skill = 1;
}

message GetSkillRequest {
    string tenant_id = 1;
    string skill_id = 2;
}

message GetSkillResponse {
    Skill skill = 1;
}

message ListSkillsRequest {
    string tenant_id = 1;
    string skill_category = 2;
    string market_demand = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListSkillsResponse {
    repeated Skill skills = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message EmployeeSkill {
    string id = 1;
    string skill_id = 2;
    string skill_name = 3;
    int32 proficiency_level = 4;
    string source = 5;
    string assessed_date = 6;
    string interest_level = 7;
}

message AssessEmployeeSkillRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string skill_id = 3;
    int32 proficiency_level = 4;
    string source = 5;
    string assessed_by = 6;
}

message AssessEmployeeSkillResponse {
    EmployeeSkill skill = 1;
}

message GetEmployeeSkillProfileRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message GetEmployeeSkillProfileResponse {
    repeated EmployeeSkill skills = 1;
    int32 total_skills = 2;
    double average_proficiency = 3;
}

message SkillGapDetail {
    string skill_id = 1;
    string skill_name = 2;
    int32 current_level = 3;
    int32 required_level = 4;
    int32 gap = 5;
}

message SkillGapAnalysis {
    string id = 1;
    string employee_id = 2;
    string target_role_id = 3;
    double overall_readiness_percentage = 4;
    repeated SkillGapDetail gaps = 5;
    int32 skills_met = 6;
    int32 skills_gap = 7;
}

message AnalyzeSkillGapRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string target_role_id = 3;
    string target_career_path_id = 4;
}

message AnalyzeSkillGapResponse {
    SkillGapAnalysis analysis = 1;
}

message GetSkillGapAnalysisRequest {
    string tenant_id = 1;
    string analysis_id = 2;
}

message GetSkillGapAnalysisResponse {
    SkillGapAnalysis analysis = 1;
}

message CareerPath {
    string id = 1;
    string tenant_id = 2;
    string path_name = 3;
    string path_code = 4;
    string path_type = 5;
    string target_job_id = 6;
    int32 estimated_duration_months = 7;
    string status = 8;
}

message CreateCareerPathRequest {
    string tenant_id = 1;
    string path_name = 2;
    string path_code = 3;
    string path_type = 4;
    string start_job_id = 5;
    string target_job_id = 6;
    string required_skills = 7;
    string created_by = 8;
}

message CreateCareerPathResponse {
    CareerPath path = 1;
}

message GetCareerPathRequest {
    string tenant_id = 1;
    string path_id = 2;
}

message GetCareerPathResponse {
    CareerPath path = 1;
}

message ListCareerPathsRequest {
    string tenant_id = 1;
    string path_type = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCareerPathsResponse {
    repeated CareerPath paths = 1;
    string next_page_token = 2;
}

message GetEligiblePathsRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message GetEligiblePathsResponse {
    repeated CareerPath eligible_paths = 1;
    repeated double readiness_percentages = 2;
}

message Milestone {
    string id = 1;
    string career_path_id = 2;
    string milestone_name = 3;
    string milestone_type = 4;
    string status = 5;
    string target_date = 6;
}

message CreateMilestoneRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string career_path_id = 3;
    string milestone_name = 4;
    string milestone_type = 5;
    string target_date = 6;
    string created_by = 7;
}

message CreateMilestoneResponse {
    Milestone milestone = 1;
}

message AchieveMilestoneRequest {
    string tenant_id = 1;
    string milestone_id = 2;
    string achieved_date = 3;
    string evidence = 4;
}

message AchieveMilestoneResponse {
    Milestone milestone = 1;
}

message DevelopmentPlan {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string plan_name = 4;
    string plan_type = 5;
    string status = 6;
    double progress_percentage = 7;
    string target_end_date = 8;
}

message CreateDevelopmentPlanRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string plan_name = 3;
    string plan_type = 4;
    string career_path_id = 5;
    string start_date = 6;
    string target_end_date = 7;
    string created_by = 8;
}

message CreateDevelopmentPlanResponse {
    DevelopmentPlan plan = 1;
}

message GetDevelopmentPlanRequest {
    string tenant_id = 1;
    string plan_id = 2;
}

message GetDevelopmentPlanResponse {
    DevelopmentPlan plan = 1;
}

message ListDevelopmentPlansRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string status = 3;
}

message ListDevelopmentPlansResponse {
    repeated DevelopmentPlan plans = 1;
}

message SkillRecommendation {
    string id = 1;
    string skill_id = 2;
    string skill_name = 3;
    string recommendation_type = 4;
    double relevance_score = 5;
    string reason = 6;
    string status = 7;
}

message GenerateRecommendationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
}

message GenerateRecommendationsResponse {
    repeated SkillRecommendation recommendations = 1;
}

message GetRecommendationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string status = 3;
}

message GetRecommendationsResponse {
    repeated SkillRecommendation recommendations = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee assignments, jobs, grades, org structure | Map career paths to roles |
| `performance-service` | Performance ratings, goals | Inform skill assessments |
| `learning-service` | Course completions, certifications | Update skill proficiency |
| `succession-service` | Succession candidacy, readiness data | Align career with succession |
| `compensation-service` | Salary benchmarks | Inform career expectations |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `learning-service` | Skill gaps, development needs | Recommend training |
| `succession-service` | Career path progress, skill updates | Update readiness |
| `reporting-service` | Career development analytics | Skills and career reports |
| `hr-service` | Career milestone achievements | Update employee records |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `SkillAssessed` | `career.skill.assessed` | `{ tenant_id, employee_id, skill_id, skill_name, proficiency_level, source }` | Published when an employee skill is assessed |
| `DevelopmentPlanCreated` | `career.plan.created` | `{ tenant_id, plan_id, employee_id, plan_name, plan_type, target_end_date }` | Published when a development plan is created |
| `CareerPathUpdated` | `career.path.updated` | `{ tenant_id, path_id, path_name, path_type, required_skills_count }` | Published when a career path definition changes |
| `MilestoneAchieved` | `career.milestone.achieved` | `{ tenant_id, milestone_id, employee_id, milestone_name, career_path_id }` | Published when a career milestone is completed |
| `SkillGapIdentified` | `career.gap.identified` | `{ tenant_id, employee_id, skill_id, skill_name, current_level, required_level }` | Published when a skill gap is found |
| `RecommendationGenerated` | `career.recommendation.generated` | `{ tenant_id, employee_id, skill_id, recommendation_type, relevance_score }` | Published when a new recommendation is created |
| `DevelopmentPlanCompleted` | `career.plan.completed` | `{ tenant_id, plan_id, employee_id, plan_name, completion_date }` | Published when all plan activities are done |
