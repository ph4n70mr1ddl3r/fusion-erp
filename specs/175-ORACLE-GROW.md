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

## 5. Business Rules

1. **AI Analysis Frequency**: Skill gap analysis refreshed monthly or on demand
2. **Recommendation Limit**: Max 10 active AI recommendations per person at a time
3. **Mentor Matching**: AI scores based on skills, experience, availability, and network proximity
4. **Skill Scoring**: Skills scored 1-5 based on certifications, courses, projects, and peer feedback
5. **Activity Tracking**: Growth activities linked to specific skill development targets
6. **Manager Visibility**: Growth plans visible to manager when enabled; AI recommendations private by default

---

## 6. Integration Points

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
