# 175 - Oracle Grow Service Specification

## 1. Domain Overview

Oracle Grow provides AI-powered employee growth recommendations, personalized learning paths, skill gap identification, and career mobility suggestions. Uses AI to analyze employee skills, performance history, goals, and career aspirations to recommend development activities, learning courses, mentors, stretch assignments, and internal opportunities. Supports growth plans with milestones, skill development tracking, and progress analytics. Integrates with Career Development, Learning, Dynamic Skills, Goals, and Talent Review.

**Bounded Context:** AI-Powered Employee Growth & Development
**Service Name:** `grow-service`
**Database:** `data/grow.db`
**HTTP Port:** 8193 | **gRPC Port:** 9193

---

## 2. Database Schema

### 2.1 Growth Profiles
```sql
CREATE TABLE gr_growth_profiles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    career_aspiration TEXT,
    target_roles TEXT,                      -- JSON: roles the person aspires to
    target_skills TEXT,                     -- JSON: skills to develop
    current_skill_scores TEXT NOT NULL,     -- JSON: { "skill": score }
    skill_gaps TEXT NOT NULL,               -- JSON: [{ "skill": "...", "current": 3, "target": 5, "gap": 2 }]
    growth_readiness TEXT NOT NULL DEFAULT 'DEVELOPING'
        CHECK(growth_readiness IN ('EMERGING','DEVELOPING','READY','ADVANCED')),
    last_ai_analysis TEXT,
    ai_recommendations TEXT,                -- JSON: AI-generated growth suggestions

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, person_id)
);

CREATE INDEX idx_gr_profile_person ON gr_growth_profiles(person_id);
CREATE INDEX idx_gr_profile_readiness ON gr_growth_profiles(tenant_id, growth_readiness);
```

### 2.2 Growth Plans
```sql
CREATE TABLE gr_growth_plans (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    plan_name TEXT NOT NULL,
    plan_type TEXT NOT NULL CHECK(plan_type IN ('CAREER','SKILL_DEVELOPMENT','ROLE_PREPARATION','LEADERSHIP','CUSTOM')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','COMPLETED','PAUSED','CANCELLED')),
    target_completion_date TEXT,
    completion_pct REAL NOT NULL DEFAULT 0,
    total_milestones INTEGER NOT NULL DEFAULT 0,
    completed_milestones INTEGER NOT NULL DEFAULT 0,
    manager_visible INTEGER NOT NULL DEFAULT 1,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, person_id, plan_name)
);

CREATE INDEX idx_gr_plan_person ON gr_growth_plans(person_id, status);
```

### 2.3 Growth Activities
```sql
CREATE TABLE gr_activities (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    plan_id TEXT NOT NULL,
    activity_type TEXT NOT NULL CHECK(activity_type IN (
        'COURSE','CERTIFICATION','MENTORING','STRETCH_ASSIGNMENT','JOB_SHADOW',
        'PROJECT','BOOK','WEBINAR','CONFERENCE','COACHING','SELF_STUDY','COMMUNITY','CUSTOM'
    )),
    title TEXT NOT NULL,
    description TEXT,
    source TEXT NOT NULL CHECK(source IN ('AI_RECOMMENDED','MANAGER_SUGGESTED','SELF_SELECTED','SYSTEM_ASSIGNED')),
    ai_relevance_score REAL,                -- AI confidence in recommendation
    ai_reasoning TEXT,                      -- Why AI recommended this
    target_skill TEXT NOT NULL,
    target_skill_level TEXT CHECK(target_skill_level IN ('BEGINNER','INTERMEDIATE','ADVANCED','EXPERT')),
    external_reference_id TEXT,             -- Link to learning course, project, etc.
    due_date TEXT,
    completed_date TEXT,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ENROLLED','IN_PROGRESS','COMPLETED','DROPPED')),
    priority INTEGER NOT NULL DEFAULT 0,
    estimated_hours REAL NOT NULL DEFAULT 0,
    actual_hours REAL NOT NULL DEFAULT 0,
    outcome_notes TEXT,
    skill_score_before REAL,
    skill_score_after REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (plan_id) REFERENCES gr_growth_plans(id) ON DELETE CASCADE
);

CREATE INDEX idx_gr_activity_plan ON gr_activities(plan_id, status);
CREATE INDEX idx_gr_activity_type ON gr_activities(tenant_id, activity_type);
```

### 2.4 AI Recommendations
```sql
CREATE TABLE gr_ai_recommendations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    recommendation_type TEXT NOT NULL CHECK(recommendation_type IN (
        'LEARNING_COURSE','CERTIFICATION','MENTOR','STRETCH_ASSIGNMENT',
        'INTERNAL_ROLE','NETWORKING','SKILL_FOCUS','CAREER_PATH','CUSTOM'
    )),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    relevance_score REAL NOT NULL DEFAULT 0,
    reasoning TEXT NOT NULL,
    supporting_data TEXT,                   -- JSON: data points behind recommendation
    target_skills TEXT NOT NULL,            -- JSON: skills this would develop
    estimated_impact TEXT,
    status TEXT NOT NULL DEFAULT 'SUGGESTED'
        CHECK(status IN ('SUGGESTED','ACCEPTED','DISMISSED','ADDED_TO_PLAN')),
    dismissed_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(id)
);

CREATE INDEX idx_gr_rec_person ON gr_ai_recommendations(person_id, status);
CREATE INDEX idx_gr_rec_type ON gr_ai_recommendations(recommendation_type);
```

### 2.5 Mentorship Connections
```sql
CREATE TABLE gr_mentorships (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    mentor_id TEXT NOT NULL,
    mentee_id TEXT NOT NULL,
    focus_area TEXT NOT NULL,
    goals TEXT NOT NULL,                    -- JSON: mentoring goals
    start_date TEXT NOT NULL,
    end_date TEXT,
    frequency TEXT NOT NULL DEFAULT 'MONTHLY'
        CHECK(frequency IN ('WEEKLY','BIWEEKLY','MONTHLY','ADHOC')),
    status TEXT NOT NULL DEFAULT 'REQUESTED'
        CHECK(status IN ('REQUESTED','ACTIVE','PAUSED','COMPLETED','DECLINED')),
    ai_match_score REAL NOT NULL DEFAULT 0,
    ai_match_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, mentor_id, mentee_id, focus_area)
);

CREATE INDEX idx_gr_mentor_mentee ON gr_mentorships(mentee_id, status);
CREATE INDEX idx_gr_mentor_mentor ON gr_mentorships(mentor_id, status);
```

---

## 3. API Endpoints

### 3.1 Growth Profiles
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grow/profiles/{personId}` | Get growth profile |
| PUT | `/api/v1/grow/profiles/{personId}` | Update aspirations |
| POST | `/api/v1/grow/profiles/{personId}/analyze` | Run AI skill gap analysis |

### 3.2 Growth Plans
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/grow/plans` | Create growth plan |
| GET | `/api/v1/grow/plans` | List plans |
| GET | `/api/v1/grow/plans/{id}` | Get plan with activities |
| PUT | `/api/v1/grow/plans/{id}` | Update plan |
| POST | `/api/v1/grow/plans/{id}/add-activity` | Add activity |
| PUT | `/api/v1/grow/activities/{id}` | Update activity |

### 3.3 AI Recommendations
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grow/recommendations/{personId}` | Get AI recommendations |
| POST | `/api/v1/grow/recommendations/{id}/accept` | Accept recommendation |
| POST | `/api/v1/grow/recommendations/{id}/dismiss` | Dismiss recommendation |
| POST | `/api/v1/grow/recommendations/refresh` | Refresh AI analysis |

### 3.4 Mentorship
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grow/mentorship/suggestions` | AI mentor suggestions |
| POST | `/api/v1/grow/mentorship` | Request mentor |
| PUT | `/api/v1/grow/mentorship/{id}` | Update mentorship |

### 3.5 Analytics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/grow/dashboard` | Growth dashboard |
| GET | `/api/v1/grow/skills-trend` | Skill development trends |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `grow.plan.created` | `{ plan_id, person_id, type }` | Growth plan created |
| `grow.activity.completed` | `{ activity_id, skill, score_delta }` | Activity completed |
| `grow.recommendation.accepted` | `{ rec_id, person_id, type }` | AI recommendation accepted |
| `grow.skill.improved` | `{ person_id, skill, old_score, new_score }` | Skill level improved |
| `grow.mentorship.requested` | `{ mentorship_id, mentor, mentee }` | Mentorship requested |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `performance.review.completed` | Performance Mgmt | Update skill scores |
| `learning.course.completed` | Learning | Complete learning activity |
| `goal.completed` | Goals | Refresh growth profile |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.grow.v1;

service GrowService {
    rpc GetGrowthProfile(GetGrowthProfileRequest) returns (GetGrowthProfileResponse);
    rpc AnalyzeSkillGaps(AnalyzeSkillGapsRequest) returns (AnalyzeSkillGapsResponse);
    rpc CreateGrowthPlan(CreateGrowthPlanRequest) returns (CreateGrowthPlanResponse);
    rpc GetGrowthPlan(GetGrowthPlanRequest) returns (GetGrowthPlanResponse);
    rpc GetRecommendations(GetRecommendationsRequest) returns (GetRecommendationsResponse);
    rpc RequestMentorship(RequestMentorshipRequest) returns (RequestMentorshipResponse);
}

// Growth Profile messages
message GetGrowthProfileRequest {
    string tenant_id = 1;
    string person_id = 2;
}

message GetGrowthProfileResponse {
    GrowthProfile data = 1;
}

message GrowthProfile {
    string id = 1;
    string tenant_id = 2;
    string person_id = 3;
    string career_aspiration = 4;
    string target_roles = 5;
    string target_skills = 6;
    string current_skill_scores = 7;
    string skill_gaps = 8;
    string growth_readiness = 9;
    string last_ai_analysis = 10;
    string ai_recommendations = 11;
    string created_at = 12;
    string updated_at = 13;
}

// Skill Gap Analysis messages
message AnalyzeSkillGapsRequest {
    string tenant_id = 1;
    string person_id = 2;
    string target_role_id = 3;
}

message AnalyzeSkillGapsResponse {
    GrowthProfile profile = 1;
    repeated SkillGap gaps = 2;
}

message SkillGap {
    string skill = 1;
    double current_score = 2;
    double target_score = 3;
    double gap = 4;
}

// Growth Plan messages
message CreateGrowthPlanRequest {
    string tenant_id = 1;
    string person_id = 2;
    string plan_name = 3;
    string plan_type = 4;
    string target_completion_date = 5;
    string notes = 6;
}

message CreateGrowthPlanResponse {
    GrowthPlan data = 1;
}

message GetGrowthPlanRequest {
    string tenant_id = 1;
    string id = 2;
}

message GetGrowthPlanResponse {
    GrowthPlan data = 1;
    repeated GrowthActivity activities = 2;
}

message GrowthPlan {
    string id = 1;
    string tenant_id = 2;
    string person_id = 3;
    string plan_name = 4;
    string plan_type = 5;
    string status = 6;
    string target_completion_date = 7;
    double completion_pct = 8;
    int32 total_milestones = 9;
    int32 completed_milestones = 10;
    int32 manager_visible = 11;
    string created_at = 12;
    string updated_at = 13;
}

message GrowthActivity {
    string id = 1;
    string tenant_id = 2;
    string plan_id = 3;
    string activity_type = 4;
    string title = 5;
    string description = 6;
    string source = 7;
    double ai_relevance_score = 8;
    string target_skill = 9;
    string status = 10;
    string due_date = 11;
    string completed_date = 12;
    double estimated_hours = 13;
    double actual_hours = 14;
    double skill_score_before = 15;
    double skill_score_after = 16;
    string created_at = 17;
}

// Recommendation messages
message GetRecommendationsRequest {
    string tenant_id = 1;
    string person_id = 2;
    string recommendation_type = 3;
}

message GetRecommendationsResponse {
    repeated AIRecommendation recommendations = 1;
}

message AIRecommendation {
    string id = 1;
    string tenant_id = 2;
    string person_id = 3;
    string recommendation_type = 4;
    string title = 5;
    string description = 6;
    double relevance_score = 7;
    string reasoning = 8;
    string target_skills = 9;
    string status = 10;
    string created_at = 11;
}

// Mentorship messages
message RequestMentorshipRequest {
    string tenant_id = 1;
    string mentor_id = 2;
    string mentee_id = 3;
    string focus_area = 4;
    string goals = 5;
    string frequency = 6;
}

message RequestMentorshipResponse {
    Mentorship data = 1;
}

message Mentorship {
    string id = 1;
    string tenant_id = 2;
    string mentor_id = 3;
    string mentee_id = 4;
    string focus_area = 5;
    string goals = 6;
    string start_date = 7;
    string end_date = 8;
    string frequency = 9;
    string status = 10;
    double ai_match_score = 11;
    string ai_match_reason = 12;
    string created_at = 13;
    string updated_at = 14;
}
```

---

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | gr_growth_profiles | -- |
| V002 | gr_growth_plans | -- |
| V003 | gr_activities | V002 |
| V004 | gr_ai_recommendations | -- |
| V005 | gr_mentorships | -- |

---

## 7. Business Rules

1. **AI Analysis Frequency**: Skill gap analysis refreshed monthly or on demand
2. **Recommendation Limit**: Max 10 active AI recommendations per person at a time
3. **Mentor Matching**: AI scores based on skills, experience, availability, and network proximity
4. **Skill Scoring**: Skills scored 1-5 based on certifications, courses, projects, and peer feedback
5. **Activity Tracking**: Growth activities linked to specific skill development targets
6. **Manager Visibility**: Growth plans visible to manager when enabled; AI recommendations private by default

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Career Development (71) | Career paths and aspirations |
| Learning & Development (69) | Course catalog, completions |
| Dynamic Skills (94) | Skills ontology and inference |
| Goals Management (169) | Development goals |
| Talent Review (146) | Talent assessment data |
| Performance Management (68) | Performance history |
| Opportunity Marketplace (93) | Stretch assignments and projects |
| AI/ML Platform (35) | Recommendation model execution |
