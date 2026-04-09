# 94 - Dynamic Skills Service Specification

## 1. Domain Overview

The Dynamic Skills service provides AI-powered skills ontology, inference, gap analysis, and skills taxonomy management. It manages skill definitions, skill assessments, skill endorsements, AI-inferred skills from work history, and skill-based talent analytics to enable data-driven talent development and workforce planning.

**Bounded Context:** Skills Ontology & Talent Intelligence
**Service Name:** `skills-service`
**Database:** `data/skills.db`
**HTTP Port:** 8131 | **gRPC Port:** 9131

---

## 2. Database Schema

### 2.1 Skills
```sql
CREATE TABLE skills (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    skill_name TEXT NOT NULL,
    skill_category TEXT,
    description TEXT,
    skill_type TEXT NOT NULL CHECK(skill_type IN ('TECHNICAL','SOFT','LEADERSHIP','CERTIFICATION','LANGUAGE')),
    parent_skill_id TEXT,
    proficiency_levels TEXT,  -- JSON array defining proficiency level descriptors
    industry_standard_id TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, skill_name)
);

CREATE INDEX idx_skills_tenant_category ON skills(tenant_id, skill_category);
CREATE INDEX idx_skills_tenant_type ON skills(tenant_id, skill_type);
CREATE INDEX idx_skills_tenant_parent ON skills(tenant_id, parent_skill_id);
CREATE INDEX idx_skills_tenant_active ON skills(tenant_id, is_active);
```

### 2.2 Skill Categories
```sql
CREATE TABLE skill_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    parent_category_id TEXT,
    sort_order INTEGER DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, name)
);

CREATE INDEX idx_skill_cats_tenant_parent ON skill_categories(tenant_id, parent_category_id);
```

### 2.3 Employee Skills
```sql
CREATE TABLE employee_skills (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    proficiency_level INTEGER NOT NULL CHECK(proficiency_level BETWEEN 1 AND 5),
    source TEXT NOT NULL CHECK(source IN ('SELF_ASSESSED','MANAGER_RATED','AI_INFERRED','CERTIFICATION','ASSESSMENT','COURSE_COMPLETED')),
    assessed_date TEXT,
    expires_date TEXT,
    evidence_references TEXT,  -- JSON array of evidence references
    confidence_score DECIMAL(5,2),
    validated_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES skills(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, employee_id, skill_id)
);

CREATE INDEX idx_emp_skills_tenant_emp ON employee_skills(tenant_id, employee_id);
CREATE INDEX idx_emp_skills_tenant_skill ON employee_skills(tenant_id, skill_id);
CREATE INDEX idx_emp_skills_tenant_prof ON employee_skills(tenant_id, proficiency_level);
CREATE INDEX idx_emp_skills_tenant_source ON employee_skills(tenant_id, source);
```

### 2.4 Skill Assessments
```sql
CREATE TABLE skill_assessments (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    assessment_type TEXT NOT NULL CHECK(assessment_type IN ('SELF','MANAGER','PEER','TEST','CERTIFICATION')),
    assessor_id TEXT NOT NULL,
    rating INTEGER NOT NULL CHECK(rating BETWEEN 1 AND 5),
    evidence TEXT,
    comments TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING' CHECK(status IN ('PENDING','COMPLETED','DISPUTED')),
    assessment_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES skills(id) ON DELETE CASCADE
);

CREATE INDEX idx_skill_assessments_tenant_emp ON skill_assessments(tenant_id, employee_id);
CREATE INDEX idx_skill_assessments_tenant_skill ON skill_assessments(tenant_id, skill_id);
CREATE INDEX idx_skill_assessments_tenant_assessor ON skill_assessments(tenant_id, assessor_id);
CREATE INDEX idx_skill_assessments_tenant_status ON skill_assessments(tenant_id, status);
CREATE INDEX idx_skill_assessments_tenant_date ON skill_assessments(tenant_id, assessment_date);
```

### 2.5 Skill Endorsements
```sql
CREATE TABLE skill_endorsements (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    endorser_id TEXT NOT NULL,
    endorse_employee_id TEXT NOT NULL,
    endorsement_strength TEXT NOT NULL CHECK(endorsement_strength IN ('BEGINNER','INTERMEDIATE','ADVANCED','EXPERT')),
    comment TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES skills(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, skill_id, endorser_id, endorse_employee_id)
);

CREATE INDEX idx_skill_endorsements_tenant_skill ON skill_endorsements(tenant_id, skill_id);
CREATE INDEX idx_skill_endorsements_tenant_endorser ON skill_endorsements(tenant_id, endorser_id);
CREATE INDEX idx_skill_endorsements_tenant_endorsed ON skill_endorsements(tenant_id, endorse_employee_id);
```

### 2.6 Skill Gap Analyses
```sql
CREATE TABLE skill_gap_analyses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    target_role_id TEXT,
    current_skills TEXT NOT NULL,  -- JSON: array of { skill_id, proficiency }
    required_skills TEXT NOT NULL,  -- JSON: array of { skill_id, proficiency }
    gap_skills TEXT NOT NULL,  -- JSON: array of { skill_id, current, required, gap }
    match_percentage DECIMAL(5,2) NOT NULL,
    recommended_actions TEXT,  -- JSON: array of recommended learning/actions
    analysis_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_skill_gaps_tenant_emp ON skill_gap_analyses(tenant_id, employee_id);
CREATE INDEX idx_skill_gaps_tenant_role ON skill_gap_analyses(tenant_id, target_role_id);
CREATE INDEX idx_skill_gaps_tenant_date ON skill_gap_analyses(tenant_id, analysis_date);
```

### 2.7 AI Skill Inferences
```sql
CREATE TABLE ai_skill_inferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    employee_id TEXT NOT NULL,
    skill_id TEXT NOT NULL,
    inferred_proficiency DECIMAL(5,2) NOT NULL,
    confidence DECIMAL(5,2) NOT NULL,
    source_type TEXT NOT NULL CHECK(source_type IN ('WORK_HISTORY','PROJECT_HISTORY','LEARNING','PEER_ENDORSEMENT')),
    source_references TEXT,  -- JSON array of source reference IDs
    model_version TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (skill_id) REFERENCES skills(id) ON DELETE CASCADE
);

CREATE INDEX idx_ai_inferences_tenant_emp ON ai_skill_inferences(tenant_id, employee_id);
CREATE INDEX idx_ai_inferences_tenant_skill ON ai_skill_inferences(tenant_id, skill_id);
CREATE INDEX idx_ai_inferences_tenant_source ON ai_skill_inferences(tenant_id, source_type);
CREATE INDEX idx_ai_inferences_tenant_confidence ON ai_skill_inferences(tenant_id, confidence DESC);
```

---

## 3. REST API Endpoints

### Skills
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/skills` | Create a new skill definition |
| GET | `/api/v1/skills` | List skills with filters |
| GET | `/api/v1/skills/{id}` | Get skill details |
| PUT | `/api/v1/skills/{id}` | Update skill definition |
| DELETE | `/api/v1/skills/{id}` | Deactivate skill |

### Categories
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/skill-categories` | Create skill category |
| GET | `/api/v1/skill-categories` | List skill categories |
| GET | `/api/v1/skill-categories/{id}` | Get category details |
| PUT | `/api/v1/skill-categories/{id}` | Update category |
| GET | `/api/v1/skill-categories/tree` | Get category hierarchy tree |

### Employee Skills
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/employees/{id}/skills` | Get employee skill profile |
| POST | `/api/v1/employees/{id}/skills` | Add skill to employee profile |
| PUT | `/api/v1/employees/{id}/skills/{skillId}` | Update employee skill proficiency |
| POST | `/api/v1/employees/{id}/skills/assess` | Submit skill assessment |
| POST | `/api/v1/employees/{id}/skills/endorse` | Endorse an employee skill |
| GET | `/api/v1/employees/{id}/skills/endorsements` | Get endorsements for employee |

### AI Inference
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/ai/infer-skills` | Run AI skill inference for an employee |
| GET | `/api/v1/employees/{id}/ai-skills` | Get AI-inferred skills for employee |
| GET | `/api/v1/ai/recommendations` | Get skill development recommendations |

### Gap Analysis
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/gap-analysis` | Run skill gap analysis for an employee |
| GET | `/api/v1/employees/{id}/gaps` | Get skill gap analyses for employee |
| GET | `/api/v1/gap-analysis/{id}` | Get specific gap analysis details |

### Search
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/search-by-skills` | Search employees by skill criteria |
| GET | `/api/v1/skills/{id}/employees` | Find employees with a specific skill |

---

## 4. Business Rules

1. Each skill MUST have a unique name within a tenant.
2. Employee skill proficiency levels MUST be integers between 1 and 5 inclusive.
3. AI-inferred skills with a confidence score below the tenant-configured threshold (default 60%) MUST be flagged as low-confidence and SHOULD NOT be used for automated decisions.
4. A skill endorsement MUST be provided by a different employee than the one being endorsed.
5. The effective proficiency level for an employee skill MUST be calculated as a weighted average of all assessment sources, with AI-inferred scores weighted lower than human-rated assessments.
6. Skill gap analyses MUST compare current proficiency against target role requirements and identify all skills where the current level falls below the required level.
7. The `match_percentage` in a gap analysis MUST be calculated as the ratio of met skill requirements to total skill requirements.
8. Self-assessed skill ratings SHOULD be capped at level 4 unless validated by a manager or certification.
9. Skill assessments with status DISPUTED MUST NOT be included in proficiency calculations until the dispute is resolved.
10. The AI inference engine MUST NOT infer skills from source data older than the tenant-configured retention period.
11. Skill categories MUST support at least 3 levels of nesting via the `parent_category_id` hierarchy.
12. Endorsement strength values MUST be mapped to proficiency levels: BEGINNER maps to 1-2, INTERMEDIATE to 3, ADVANCED to 4, and EXPERT to 5.
13. The system MUST recalculate AI skill inferences when an employee's work history or project assignments change.
14. Expired skills (past `expires_date`) MUST be flagged and SHOULD prompt reassessment.
15. The system MUST support bulk import of skills from industry-standard taxonomies.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package skills.v1;

service DynamicSkillsService {
    // Skills CRUD
    rpc CreateSkill(CreateSkillRequest) returns (CreateSkillResponse);
    rpc GetSkill(GetSkillRequest) returns (GetSkillResponse);
    rpc ListSkills(ListSkillsRequest) returns (ListSkillsResponse);
    rpc UpdateSkill(UpdateSkillRequest) returns (UpdateSkillResponse);

    // Categories
    rpc CreateCategory(CreateCategoryRequest) returns (CreateCategoryResponse);
    rpc ListCategories(ListCategoriesRequest) returns (ListCategoriesResponse);
    rpc GetCategoryTree(GetCategoryTreeRequest) returns (GetCategoryTreeResponse);

    // Employee Skills
    rpc GetEmployeeSkills(GetEmployeeSkillsRequest) returns (GetEmployeeSkillsResponse);
    rpc AddEmployeeSkill(AddEmployeeSkillRequest) returns (AddEmployeeSkillResponse);
    rpc UpdateEmployeeSkill(UpdateEmployeeSkillRequest) returns (UpdateEmployeeSkillResponse);
    rpc AssessSkill(AssessSkillRequest) returns (AssessSkillResponse);
    rpc EndorseSkill(EndorseSkillRequest) returns (EndorseSkillResponse);

    // AI Inference
    rpc InferSkills(InferSkillsRequest) returns (InferSkillsResponse);
    rpc GetAIInferredSkills(GetAIInferredSkillsRequest) returns (GetAIInferredSkillsResponse);
    rpc GetRecommendations(GetRecommendationsRequest) returns (GetRecommendationsResponse);

    // Gap Analysis
    rpc AnalyzeGap(AnalyzeGapRequest) returns (AnalyzeGapResponse);
    rpc GetGapAnalyses(GetGapAnalysesRequest) returns (GetGapAnalysesResponse);

    // Search
    rpc SearchBySkills(SearchBySkillsRequest) returns (SearchBySkillsResponse);
}

message Skill {
    string id = 1;
    string tenant_id = 2;
    string skill_name = 3;
    string skill_category = 4;
    string description = 5;
    string skill_type = 6;
    string parent_skill_id = 7;
    string proficiency_levels = 8;
    bool is_active = 9;
}

message CreateSkillRequest {
    string tenant_id = 1;
    string skill_name = 2;
    string skill_category = 3;
    string description = 4;
    string skill_type = 5;
    string parent_skill_id = 6;
    string proficiency_levels = 7;
    string created_by = 8;
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
    string skill_type = 2;
    string skill_category = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListSkillsResponse {
    repeated Skill skills = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateSkillRequest {
    string tenant_id = 1;
    string skill_id = 2;
    string skill_name = 3;
    string description = 4;
    string skill_type = 5;
    string updated_by = 6;
}

message UpdateSkillResponse {
    Skill skill = 1;
}

message SkillCategory {
    string id = 1;
    string tenant_id = 2;
    string name = 3;
    string description = 4;
    string parent_category_id = 5;
    int32 sort_order = 6;
}

message CreateCategoryRequest {
    string tenant_id = 1;
    string name = 2;
    string description = 3;
    string parent_category_id = 4;
    int32 sort_order = 5;
    string created_by = 6;
}

message CreateCategoryResponse {
    SkillCategory category = 1;
}

message ListCategoriesRequest {
    string tenant_id = 1;
    string parent_category_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCategoriesResponse {
    repeated SkillCategory categories = 1;
    string next_page_token = 2;
}

message CategoryTreeNode {
    SkillCategory category = 1;
    repeated CategoryTreeNode children = 2;
}

message GetCategoryTreeRequest {
    string tenant_id = 1;
}

message GetCategoryTreeResponse {
    repeated CategoryTreeNode root_nodes = 1;
}

message EmployeeSkill {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string skill_id = 4;
    int32 proficiency_level = 5;
    string source = 6;
    string assessed_date = 7;
    double confidence_score = 8;
    string validated_by = 9;
}

message GetEmployeeSkillsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string skill_type = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message GetEmployeeSkillsResponse {
    repeated EmployeeSkill skills = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message AddEmployeeSkillRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string skill_id = 3;
    int32 proficiency_level = 4;
    string source = 5;
    string created_by = 6;
}

message AddEmployeeSkillResponse {
    EmployeeSkill employee_skill = 1;
}

message UpdateEmployeeSkillRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string skill_id = 3;
    int32 proficiency_level = 4;
    string source = 5;
    string updated_by = 6;
}

message UpdateEmployeeSkillResponse {
    EmployeeSkill employee_skill = 1;
}

message AssessSkillRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string skill_id = 3;
    string assessment_type = 4;
    string assessor_id = 5;
    int32 rating = 6;
    string evidence = 7;
    string comments = 8;
    string created_by = 9;
}

message AssessSkillResponse {
    string assessment_id = 1;
    string status = 2;
}

message EndorseSkillRequest {
    string tenant_id = 1;
    string skill_id = 2;
    string endorser_id = 3;
    string endorse_employee_id = 4;
    string endorsement_strength = 5;
    string comment = 6;
}

message EndorseSkillResponse {
    string endorsement_id = 1;
}

message AISkillInference {
    string id = 1;
    string skill_id = 2;
    double inferred_proficiency = 3;
    double confidence = 4;
    string source_type = 5;
    string model_version = 6;
}

message InferSkillsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string source_type = 3;
    string requested_by = 4;
}

message InferSkillsResponse {
    repeated AISkillInference inferred_skills = 1;
    int32 total_inferred = 2;
    string model_version = 3;
}

message GetAIInferredSkillsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    double min_confidence = 3;
}

message GetAIInferredSkillsResponse {
    repeated AISkillInference skills = 1;
}

message SkillRecommendation {
    string skill_id = 1;
    string skill_name = 2;
    string recommendation_type = 3;
    string description = 4;
    int32 priority = 5;
}

message GetRecommendationsRequest {
    string tenant_id = 1;
    string employee_id = 2;
    int32 max_results = 3;
}

message GetRecommendationsResponse {
    repeated SkillRecommendation recommendations = 1;
}

message GapAnalysis {
    string id = 1;
    string tenant_id = 2;
    string employee_id = 3;
    string target_role_id = 4;
    double match_percentage = 5;
    string gap_skills = 6;
    string recommended_actions = 7;
    string analysis_date = 8;
}

message AnalyzeGapRequest {
    string tenant_id = 1;
    string employee_id = 2;
    string target_role_id = 3;
    string requested_by = 4;
}

message AnalyzeGapResponse {
    GapAnalysis analysis = 1;
}

message GetGapAnalysesRequest {
    string tenant_id = 1;
    string employee_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message GetGapAnalysesResponse {
    repeated GapAnalysis analyses = 1;
    string next_page_token = 2;
}

message EmployeeSkillMatch {
    string employee_id = 1;
    string employee_name = 2;
    string skill_id = 3;
    int32 proficiency_level = 4;
    double match_score = 5;
}

message SearchBySkillsRequest {
    string tenant_id = 1;
    repeated string skill_ids = 2;
    int32 min_proficiency = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message SearchBySkillsResponse {
    repeated EmployeeSkillMatch matches = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `hr-service` | Employee profiles, job/role definitions, org structure | Map skills to roles and employees |
| `marketplace-service` | Opportunity skills, completed assignment data | Infer skills from project participation |
| `learning-service` | Course completions, certifications | Add skills from completed learning |
| `recruiting-service` | Candidate skill profiles | Pre-populate skills for new hires |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `marketplace-service` | Employee skill profiles, match scores | Power skill-based opportunity matching |
| `career-service` | Skill gap analyses, skill profiles | Support career path planning |
| `learning-service` | Skill gap recommendations | Drive personalized learning suggestions |
| `reporting-service` | Skills analytics, gap reports | Talent intelligence dashboards |
| `hr-service` | Employee skill summaries | Enrich employee profiles |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `SkillAssessed` | `skills.assessment.completed` | `{ tenant_id, assessment_id, employee_id, skill_id, rating, assessment_type, assessor_id }` | Published when a skill assessment is completed |
| `SkillsInferred` | `skills.ai.inferred` | `{ tenant_id, employee_id, inferred_count, model_version, confidence_avg }` | Published when AI completes skill inference for an employee |
| `GapAnalysisCompleted` | `skills.gap.completed` | `{ tenant_id, analysis_id, employee_id, target_role_id, match_percentage, gap_count }` | Published when a skill gap analysis is finalized |
| `SkillEndorsed` | `skills.endorsement.created` | `{ tenant_id, endorsement_id, skill_id, endorser_id, endorse_employee_id, endorsement_strength }` | Published when an employee endorses another employee's skill |
| `SkillProfileUpdated` | `skills.profile.updated` | `{ tenant_id, employee_id, skill_id, proficiency_level, source }` | Published when an employee's skill profile is modified |
