# 92 - Innovation Management Service Specification

## 1. Domain Overview

Innovation Management provides a structured product innovation pipeline for managing ideas from submission through evaluation, stage-gate progression, portfolio allocation, and collaborative refinement. Manages innovation campaigns with configurable themes, idea submission with categorization and tagging, multi-criteria evaluation with scoring rubrics, stage-gate processes with defined review criteria and approval gates, portfolio-level resource allocation across active innovations, innovation metrics tracking (time-to-market, conversion rates, ROI), collaboration spaces for team discussion and document sharing, and community voting for crowd-sourced idea prioritization. Integrates with PLM for product data, Project Management for execution, and GL for budget and cost tracking.

**Bounded Context:** Product Innovation Pipeline & Idea Management
**Service Name:** `innovation-service`
**Database:** `data/innovation.db`
**HTTP Port:** 8124 | **gRPC Port:** 9124

---

## 2. Database Schema

### 2.1 Innovation Campaigns
```sql
CREATE TABLE innovation_campaigns (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_name TEXT NOT NULL,
    campaign_code TEXT NOT NULL,
    description TEXT,
    campaign_type TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(campaign_type IN ('OPEN','TARGETED','HACKATHON','CONTINUOUS','CHALLENGE')),
    theme TEXT,
    objectives TEXT NOT NULL,                        -- JSON: campaign objectives and goals
    target_audience TEXT,                            -- JSON: eligible participant groups
    submission_start TEXT NOT NULL,
    submission_end TEXT NOT NULL,
    evaluation_start TEXT NOT NULL,
    evaluation_end TEXT NOT NULL,
    max_ideas_per_submitter INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','OPEN_FOR_SUBMISSION','UNDER_EVALUATION','COMPLETED','CANCELLED','ARCHIVED')),
    owner_id TEXT NOT NULL,
    review_committee TEXT,                           -- JSON array of evaluator user IDs
    reward_config TEXT,                              -- JSON: rewards and recognition config
    total_ideas INTEGER NOT NULL DEFAULT 0,
    total_evaluators INTEGER NOT NULL DEFAULT 0,
    total_votes INTEGER NOT NULL DEFAULT 0,
    budget_allocated_cents INTEGER NOT NULL DEFAULT 0,
    budget_consumed_cents INTEGER NOT NULL DEFAULT 0,
    tags TEXT,                                       -- JSON array of tags
    metadata TEXT,                                   -- JSON: extensible campaign metadata

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, campaign_code)
);

CREATE INDEX idx_campaigns_tenant_status ON innovation_campaigns(tenant_id, status);
CREATE INDEX idx_campaigns_tenant_type ON innovation_campaigns(tenant_id, campaign_type);
CREATE INDEX idx_campaigns_tenant_owner ON innovation_campaigns(tenant_id, owner_id);
CREATE INDEX idx_campaigns_tenant_dates ON innovation_campaigns(tenant_id, submission_start, submission_end);
CREATE INDEX idx_campaigns_tenant_active ON innovation_campaigns(tenant_id, is_active);
```

### 2.2 Ideas
```sql
CREATE TABLE ideas (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    idea_title TEXT NOT NULL,
    idea_code TEXT NOT NULL,
    idea_description TEXT NOT NULL,
    problem_statement TEXT NOT NULL,
    proposed_solution TEXT NOT NULL,
    expected_benefits TEXT,                          -- JSON: quantitative and qualitative benefits
    estimated_effort_cents INTEGER NOT NULL DEFAULT 0,
    estimated_revenue_cents INTEGER NOT NULL DEFAULT 0,
    estimated_cost_savings_cents INTEGER NOT NULL DEFAULT 0,
    priority TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(priority IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    idea_status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(idea_status IN ('SUBMITTED','UNDER_REVIEW','EVALUATED','APPROVED','IN_DEVELOPMENT','IMPLEMENTED','REJECTED','WITHDRAWN','DUPLICATE')),
    submission_source TEXT NOT NULL DEFAULT 'INTERNAL'
        CHECK(submission_source IN ('INTERNAL','CUSTOMER','PARTNER','EMPLOYEE_SUGGESTION','MINING')),
    submitter_id TEXT NOT NULL,
    submitter_department TEXT,
    category_id TEXT,
    assigned_reviewer_id TEXT,
    current_gate_id TEXT,
    current_gate_status TEXT,
    implementation_complexity TEXT DEFAULT 'MEDIUM'
        CHECK(implementation_complexity IN ('LOW','MEDIUM','HIGH','VERY_HIGH')),
    time_to_market_days INTEGER,
    target_market TEXT,
    attachments TEXT,                                -- JSON array of document IDs
    tags TEXT,                                       -- JSON array of tags

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (campaign_id) REFERENCES innovation_campaigns(id),
    FOREIGN KEY (category_id) REFERENCES idea_categories(id),
    UNIQUE(tenant_id, idea_code)
);

CREATE INDEX idx_ideas_tenant_campaign ON ideas(tenant_id, campaign_id);
CREATE INDEX idx_ideas_tenant_status ON ideas(tenant_id, idea_status);
CREATE INDEX idx_ideas_tenant_priority ON ideas(tenant_id, priority);
CREATE INDEX idx_ideas_tenant_submitter ON ideas(tenant_id, submitter_id);
CREATE INDEX idx_ideas_tenant_category ON ideas(tenant_id, category_id);
CREATE INDEX idx_ideas_tenant_gate ON ideas(tenant_id, current_gate_id);
CREATE INDEX idx_ideas_tenant_active ON ideas(tenant_id, is_active);
```

### 2.3 Idea Categories
```sql
CREATE TABLE idea_categories (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    category_name TEXT NOT NULL,
    category_code TEXT NOT NULL,
    description TEXT,
    parent_category_id TEXT,
    category_level INTEGER NOT NULL DEFAULT 0,
    icon TEXT,
    color TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, category_code)
);

CREATE INDEX idx_categories_tenant_parent ON idea_categories(tenant_id, parent_category_id);
CREATE INDEX idx_categories_tenant_active ON idea_categories(tenant_id, is_active);
```

### 2.4 Idea Evaluations
```sql
CREATE TABLE idea_evaluations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    idea_id TEXT NOT NULL,
    evaluator_id TEXT NOT NULL,
    evaluation_type TEXT NOT NULL DEFAULT 'INDIVIDUAL'
        CHECK(evaluation_type IN ('INDIVIDUAL','COMMITTEE','CROWD','AUTOMATED')),
    criteria_scores TEXT NOT NULL,                   -- JSON: { criteria_id: { score, weight, comments } }
    overall_score_real REAL NOT NULL DEFAULT 0.0,
    weighted_score_real REAL NOT NULL DEFAULT 0.0,
    recommendation TEXT NOT NULL DEFAULT 'HOLD'
        CHECK(recommendation IN ('STRONG_APPROVE','APPROVE','HOLD','REJECT','STRONG_REJECT')),
    strengths TEXT,
    weaknesses TEXT,
    comments TEXT,
    risk_assessment TEXT,                            -- JSON: risk factors and mitigations
    feasibility_score_real REAL NOT NULL DEFAULT 0.0,
    impact_score_real REAL NOT NULL DEFAULT 0.0,
    strategic_alignment_score_real REAL NOT NULL DEFAULT 0.0,
    is_completed INTEGER NOT NULL DEFAULT 0,
    completed_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (idea_id) REFERENCES ideas(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, idea_id, evaluator_id, evaluation_type)
);

CREATE INDEX idx_evaluations_tenant_idea ON idea_evaluations(tenant_id, idea_id);
CREATE INDEX idx_evaluations_tenant_evaluator ON idea_evaluations(tenant_id, evaluator_id);
CREATE INDEX idx_evaluations_tenant_type ON idea_evaluations(tenant_id, evaluation_type);
CREATE INDEX idx_evaluations_tenant_completed ON idea_evaluations(tenant_id, is_completed);
```

### 2.5 Stage Gate Processes
```sql
CREATE TABLE stage_gate_processes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    process_name TEXT NOT NULL,
    process_code TEXT NOT NULL,
    description TEXT,
    applicable_campaign_types TEXT,                  -- JSON array of campaign types
    gates TEXT NOT NULL,                             -- JSON: ordered gate definitions with criteria
    total_gates INTEGER NOT NULL DEFAULT 0,
    estimated_total_days INTEGER NOT NULL DEFAULT 0,
    approval_type TEXT NOT NULL DEFAULT 'MAJORITY'
        CHECK(approval_type IN ('CONSENSUS','MAJORITY','SINGLE_APPROVER','SCORING_THRESHOLD')),
    approval_threshold_real REAL NOT NULL DEFAULT 70.0,
    is_default INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, process_code)
);

CREATE INDEX idx_processes_tenant_active ON stage_gate_processes(tenant_id, is_active);
CREATE INDEX idx_processes_tenant_default ON stage_gate_processes(tenant_id, is_default);
```

### 2.6 Stage Gate Reviews
```sql
CREATE TABLE stage_gate_reviews (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    idea_id TEXT NOT NULL,
    process_id TEXT NOT NULL,
    gate_number INTEGER NOT NULL,
    gate_name TEXT NOT NULL,
    gate_criteria TEXT NOT NULL,                     -- JSON: gate-specific review criteria
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_REVIEW','CONDITIONAL_PASS','PASSED','FAILED','WAIVED')),
    review_date TEXT NOT NULL,
    reviewers TEXT NOT NULL,                         -- JSON array of reviewer user IDs
    review_scores TEXT,                              -- JSON: reviewer scores per criteria
    aggregate_score_real REAL NOT NULL DEFAULT 0.0,
    review_comments TEXT,
    conditions TEXT,                                 -- JSON: conditions for conditional pass
    decision_rationale TEXT,
    decider_id TEXT,
    decided_at TEXT,
    scheduled_date TEXT,
    duration_days INTEGER NOT NULL DEFAULT 14,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (idea_id) REFERENCES ideas(id) ON DELETE CASCADE,
    FOREIGN KEY (process_id) REFERENCES stage_gate_processes(id),
    UNIQUE(tenant_id, idea_id, gate_number)
);

CREATE INDEX idx_reviews_tenant_idea ON stage_gate_reviews(tenant_id, idea_id);
CREATE INDEX idx_reviews_tenant_process ON stage_gate_reviews(tenant_id, process_id);
CREATE INDEX idx_reviews_tenant_status ON stage_gate_reviews(tenant_id, status);
CREATE INDEX idx_reviews_tenant_gate ON stage_gate_reviews(tenant_id, gate_number);
CREATE INDEX idx_reviews_tenant_dates ON stage_gate_reviews(tenant_id, scheduled_date);
```

### 2.7 Innovation Portfolios
```sql
CREATE TABLE innovation_portfolios (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    portfolio_name TEXT NOT NULL,
    portfolio_code TEXT NOT NULL,
    description TEXT,
    portfolio_type TEXT NOT NULL DEFAULT 'STRATEGIC'
        CHECK(portfolio_type IN ('STRATEGIC','TACTICAL','OPERATIONAL','BALANCED','CUSTOM')),
    strategy_framework TEXT NOT NULL DEFAULT 'BCG_MATRIX'
        CHECK(strategy_framework IN ('BCG_MATRIX','MCKINSEY_3X3','RISK_REWARD','STAGE_GATE','CUSTOM')),
    target_allocation TEXT NOT NULL,                 -- JSON: allocation targets by category/risk/strategic area
    total_budget_cents INTEGER NOT NULL DEFAULT 0,
    allocated_budget_cents INTEGER NOT NULL DEFAULT 0,
    consumed_budget_cents INTEGER NOT NULL DEFAULT 0,
    total_ideas INTEGER NOT NULL DEFAULT 0,
    active_ideas INTEGER NOT NULL DEFAULT 0,
    completed_ideas INTEGER NOT NULL DEFAULT 0,
    failed_ideas INTEGER NOT NULL DEFAULT 0,
    expected_roi_real REAL NOT NULL DEFAULT 0.0,
    actual_roi_real REAL NOT NULL DEFAULT 0.0,
    owner_id TEXT NOT NULL,
    rebalancing_frequency TEXT NOT NULL DEFAULT 'QUARTERLY'
        CHECK(rebalancing_frequency IN ('MONTHLY','QUARTERLY','SEMI_ANNUALLY','ANNUALLY')),
    last_rebalanced_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, portfolio_code)
);

CREATE INDEX idx_portfolios_tenant_type ON innovation_portfolios(tenant_id, portfolio_type);
CREATE INDEX idx_portfolios_tenant_owner ON innovation_portfolios(tenant_id, owner_id);
CREATE INDEX idx_portfolios_tenant_active ON innovation_portfolios(tenant_id, is_active);
```

### 2.8 Resource Allocations
```sql
CREATE TABLE resource_allocations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    portfolio_id TEXT NOT NULL,
    idea_id TEXT NOT NULL,
    allocation_type TEXT NOT NULL DEFAULT 'BUDGET'
        CHECK(allocation_type IN ('BUDGET','HEADCOUNT','EQUIPMENT','FACILITY','ALL')),
    allocated_amount_cents INTEGER NOT NULL DEFAULT 0,
    consumed_amount_cents INTEGER NOT NULL DEFAULT 0,
    remaining_amount_cents INTEGER NOT NULL DEFAULT 0,
    allocated_headcount REAL NOT NULL DEFAULT 0.0,
    consumed_headcount REAL NOT NULL DEFAULT 0.0,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    cost_center TEXT,
    gl_account_id TEXT,
    project_id TEXT,                                 -- Link to PM for execution tracking
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','ACTIVE','ON_HOLD','COMPLETED','CANCELLED')),
    justification TEXT,
    approval_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(approval_status IN ('PENDING','APPROVED','REJECTED','REVISED')),
    approved_by TEXT,
    approved_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (portfolio_id) REFERENCES innovation_portfolios(id) ON DELETE CASCADE,
    FOREIGN KEY (idea_id) REFERENCES ideas(id)
);

CREATE INDEX idx_allocations_tenant_portfolio ON resource_allocations(tenant_id, portfolio_id);
CREATE INDEX idx_allocations_tenant_idea ON resource_allocations(tenant_id, idea_id);
CREATE INDEX idx_allocations_tenant_status ON resource_allocations(tenant_id, status);
CREATE INDEX idx_allocations_tenant_period ON resource_allocations(tenant_id, period_start, period_end);
CREATE INDEX idx_allocations_tenant_approval ON resource_allocations(tenant_id, approval_status);
```

### 2.9 Innovation Metrics
```sql
CREATE TABLE innovation_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    metric_date TEXT NOT NULL,
    portfolio_id TEXT,
    campaign_id TEXT,
    total_ideas_submitted INTEGER NOT NULL DEFAULT 0,
    total_ideas_approved INTEGER NOT NULL DEFAULT 0,
    total_ideas_implemented INTEGER NOT NULL DEFAULT 0,
    total_ideas_rejected INTEGER NOT NULL DEFAULT 0,
    conversion_rate_pct REAL NOT NULL DEFAULT 0.0,
    avg_time_to_evaluate_days REAL NOT NULL DEFAULT 0.0,
    avg_time_to_market_days REAL NOT NULL DEFAULT 0.0,
    avg_evaluation_score_real REAL NOT NULL DEFAULT 0.0,
    total_budget_allocated_cents INTEGER NOT NULL DEFAULT 0,
    total_budget_consumed_cents INTEGER NOT NULL DEFAULT 0,
    total_revenue_generated_cents INTEGER NOT NULL DEFAULT 0,
    total_cost_savings_cents INTEGER NOT NULL DEFAULT 0,
    innovation_roi_real REAL NOT NULL DEFAULT 0.0,
    ideas_per_employee REAL NOT NULL DEFAULT 0.0,
    participation_rate_pct REAL NOT NULL DEFAULT 0.0,
    evaluator_count INTEGER NOT NULL DEFAULT 0,
    active_gates_count INTEGER NOT NULL DEFAULT 0,
    gates_passed_count INTEGER NOT NULL DEFAULT 0,
    gates_failed_count INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, metric_date, portfolio_id, campaign_id)
);

CREATE INDEX idx_metrics_tenant_date ON innovation_metrics(tenant_id, metric_date);
CREATE INDEX idx_metrics_tenant_portfolio ON innovation_metrics(tenant_id, portfolio_id);
CREATE INDEX idx_metrics_tenant_campaign ON innovation_metrics(tenant_id, campaign_id);
```

### 2.10 Collaboration Spaces
```sql
CREATE TABLE collaboration_spaces (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    idea_id TEXT NOT NULL,
    space_name TEXT NOT NULL,
    space_type TEXT NOT NULL DEFAULT 'DISCUSSION'
        CHECK(space_type IN ('DISCUSSION','WHITEBOARD','DOCUMENT','WIKI','CHAT')),
    description TEXT,
    content TEXT,                                    -- JSON: space content (posts, documents, etc.)
    participant_ids TEXT NOT NULL,                   -- JSON array of participant user IDs
    moderator_ids TEXT,                              -- JSON array of moderator user IDs
    is_private INTEGER NOT NULL DEFAULT 0,
    max_participants INTEGER NOT NULL DEFAULT 50,
    total_posts INTEGER NOT NULL DEFAULT 0,
    total_attachments INTEGER NOT NULL DEFAULT 0,
    last_activity_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (idea_id) REFERENCES ideas(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, idea_id, space_name)
);

CREATE INDEX idx_spaces_tenant_idea ON collaboration_spaces(tenant_id, idea_id);
CREATE INDEX idx_spaces_tenant_type ON collaboration_spaces(tenant_id, space_type);
CREATE INDEX idx_spaces_tenant_active ON collaboration_spaces(tenant_id, is_active);
CREATE INDEX idx_spaces_tenant_activity ON collaboration_spaces(tenant_id, last_activity_at);
```

### 2.11 Idea Votes
```sql
CREATE TABLE idea_votes (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    idea_id TEXT NOT NULL,
    voter_id TEXT NOT NULL,
    vote_type TEXT NOT NULL DEFAULT 'UPVOTE'
        CHECK(vote_type IN ('UPVOTE','DOWNVOTE','NEUTRAL')),
    vote_weight_real REAL NOT NULL DEFAULT 1.0,
    comment TEXT,
    voted_at TEXT NOT NULL DEFAULT (datetime('now')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (idea_id) REFERENCES ideas(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, idea_id, voter_id)
);

CREATE INDEX idx_votes_tenant_idea ON idea_votes(tenant_id, idea_id);
CREATE INDEX idx_votes_tenant_voter ON idea_votes(tenant_id, voter_id);
CREATE INDEX idx_votes_tenant_type ON idea_votes(tenant_id, vote_type);
CREATE INDEX idx_votes_tenant_dates ON idea_votes(tenant_id, voted_at);
```

---

## 3. REST API Endpoints

```
# Campaigns
GET/POST      /api/v1/innovation/campaigns                     Permission: innovation.campaigns.read/create
GET/PUT       /api/v1/innovation/campaigns/{id}                Permission: innovation.campaigns.read/update
DELETE        /api/v1/innovation/campaigns/{id}                Permission: innovation.campaigns.delete
PATCH         /api/v1/innovation/campaigns/{id}/status         Permission: innovation.campaigns.update
GET           /api/v1/innovation/campaigns/{id}/dashboard      Permission: innovation.campaigns.read

# Ideas
GET/POST      /api/v1/innovation/campaigns/{id}/ideas          Permission: innovation.ideas.read/create
GET/PUT       /api/v1/innovation/ideas/{id}                    Permission: innovation.ideas.read/update
DELETE        /api/v1/innovation/ideas/{id}                    Permission: innovation.ideas.delete
PATCH         /api/v1/innovation/ideas/{id}/status             Permission: innovation.ideas.update
GET           /api/v1/innovation/ideas/search                  Permission: innovation.ideas.read
GET           /api/v1/innovation/ideas/{id}/timeline           Permission: innovation.ideas.read

# Categories
GET/POST      /api/v1/innovation/categories                    Permission: innovation.categories.read/create
GET/PUT       /api/v1/innovation/categories/{id}               Permission: innovation.categories.read/update
DELETE        /api/v1/innovation/categories/{id}               Permission: innovation.categories.delete
GET           /api/v1/innovation/categories/tree               Permission: innovation.categories.read

# Evaluations
GET/POST      /api/v1/innovation/ideas/{id}/evaluations        Permission: innovation.evaluations.read/create
GET/PUT       /api/v1/innovation/evaluations/{id}              Permission: innovation.evaluations.read/update
GET           /api/v1/innovation/ideas/{id}/evaluations/summary Permission: innovation.evaluations.read
POST          /api/v1/innovation/evaluations/bulk-assign       Permission: innovation.evaluations.create

# Stage Gate Processes
GET/POST      /api/v1/innovation/stage-gates                   Permission: innovation.gates.read/create
GET/PUT       /api/v1/innovation/stage-gates/{id}              Permission: innovation.gates.read/update
DELETE        /api/v1/innovation/stage-gates/{id}              Permission: innovation.gates.delete

# Stage Gate Reviews
GET           /api/v1/innovation/ideas/{id}/gate-reviews       Permission: innovation.reviews.read
POST          /api/v1/innovation/ideas/{id}/advance-gate       Permission: innovation.reviews.execute
GET/PUT       /api/v1/innovation/gate-reviews/{id}             Permission: innovation.reviews.read/update
POST          /api/v1/innovation/gate-reviews/{id}/decide      Permission: innovation.reviews.execute

# Portfolios
GET/POST      /api/v1/innovation/portfolios                    Permission: innovation.portfolios.read/create
GET/PUT       /api/v1/innovation/portfolios/{id}               Permission: innovation.portfolios.read/update
DELETE        /api/v1/innovation/portfolios/{id}               Permission: innovation.portfolios.delete
GET           /api/v1/innovation/portfolios/{id}/analysis      Permission: innovation.portfolios.read
POST          /api/v1/innovation/portfolios/{id}/rebalance     Permission: innovation.portfolios.execute

# Resource Allocations
GET/POST      /api/v1/innovation/portfolios/{id}/allocations   Permission: innovation.allocations.read/create
GET/PUT       /api/v1/innovation/allocations/{id}             Permission: innovation.allocations.read/update
DELETE        /api/v1/innovation/allocations/{id}             Permission: innovation.allocations.delete
POST          /api/v1/innovation/allocations/{id}/approve     Permission: innovation.allocations.execute

# Metrics
GET           /api/v1/innovation/metrics/summary               Permission: innovation.metrics.read
GET           /api/v1/innovation/metrics/trends                 Permission: innovation.metrics.read
GET           /api/v1/innovation/metrics/conversion             Permission: innovation.metrics.read
GET           /api/v1/innovation/metrics/roi                    Permission: innovation.metrics.read
POST          /api/v1/innovation/metrics/refresh                Permission: innovation.metrics.execute

# Collaboration
GET/POST      /api/v1/innovation/ideas/{id}/spaces             Permission: innovation.collaboration.read/create
GET/PUT       /api/v1/innovation/spaces/{id}                   Permission: innovation.collaboration.read/update
DELETE        /api/v1/innovation/spaces/{id}                   Permission: innovation.collaboration.delete
POST          /api/v1/innovation/spaces/{id}/posts             Permission: innovation.collaboration.create
GET           /api/v1/innovation/spaces/{id}/posts             Permission: innovation.collaboration.read

# Voting
POST          /api/v1/innovation/ideas/{id}/vote               Permission: innovation.voting.create
DELETE        /api/v1/innovation/ideas/{id}/vote               Permission: innovation.voting.delete
GET           /api/v1/innovation/ideas/{id}/votes              Permission: innovation.voting.read
GET           /api/v1/innovation/campaigns/{id}/leaderboard    Permission: innovation.voting.read
```

---

## 4. Business Rules

### 4.1 Campaign Management
1. Each campaign MUST have a unique code within a tenant
2. Campaign dates MUST follow the sequence: submission_start, submission_end, evaluation_start, evaluation_end
3. Ideas MUST NOT be accepted after the submission_end date unless the campaign status is CONTINUOUS
4. Campaign owners MUST be valid users within the tenant
5. max_ideas_per_submitter of 0 means unlimited submissions; positive values enforce the limit
6. Campaign budget allocated MUST NOT exceed the total approved budget
7. Campaign status transitions MUST follow the defined state machine (DRAFT -> OPEN_FOR_SUBMISSION -> UNDER_EVALUATION -> COMPLETED)

### 4.2 Idea Submission
1. Each idea MUST have a unique code within a tenant
2. Ideas MUST be associated with a valid campaign that is in OPEN_FOR_SUBMISSION status
3. Idea status transitions MUST follow: SUBMITTED -> UNDER_REVIEW -> EVALUATED -> APPROVED/REJECTED -> IN_DEVELOPMENT -> IMPLEMENTED
4. Withdrawn ideas MUST be re-submittable as a new idea version
5. Duplicate ideas SHOULD be flagged and linked to the original
6. Estimated monetary values MUST be stored in cents to avoid floating-point precision issues
7. Ideas MAY be tagged with multiple categories and custom tags

### 4.3 Evaluation Process
1. Each evaluator MAY submit only one evaluation per idea per evaluation_type
2. Evaluation criteria scores MUST be within the defined scale (typically 1-10)
3. Overall score MUST be computed as the weighted average of individual criteria scores
4. Weighted score MUST account for criteria weights defined at the campaign or process level
5. Automated evaluations SHOULD use AI-assisted scoring where configured
6. Committee evaluations MUST aggregate individual scores into a collective recommendation
7. Incomplete evaluations MUST NOT be factored into aggregate scoring

### 4.4 Stage Gate Process
1. Gate definitions MUST specify clear pass/fail criteria
2. Gate reviews MUST execute in sequential order (gate 1, then gate 2, etc.)
3. A gate MUST NOT be reviewed until the previous gate has PASSED or been WAIVED
4. CONDITIONAL_PASS MUST list specific conditions that must be met within a defined timeframe
5. Approval type determines the decision method: consensus requires all reviewers to agree; majority requires >50%
6. Ideas failing a gate MAY be allowed to resubmit for the same gate based on process configuration
7. Waived gates MUST document the justification for bypassing the standard review

### 4.5 Portfolio Management
1. Each portfolio MUST have a unique code within a tenant
2. Target allocation MUST define percentage distribution across categories or strategic areas
3. Actual allocation MUST be compared against targets during rebalancing
4. Portfolio expected ROI MUST be calculated as the weighted average of constituent idea ROI projections
5. Rebalancing frequency determines when portfolio reviews are triggered automatically
6. Budget consumed MUST NOT exceed the total portfolio budget without approval
7. Portfolio analysis MUST support multiple strategy frameworks (BCG Matrix, McKinsey 3x3, etc.)

### 4.6 Resource Allocation
1. Allocations MUST reference a valid portfolio and idea
2. Remaining amount MUST equal allocated amount minus consumed amount
3. Period start and end dates MUST define a valid date range
4. Cost center and GL account MUST be validated against the GL service before allocation approval
5. Allocations in PLANNED status MUST be approved before becoming ACTIVE
6. Cancelled allocations MUST return remaining budget to the portfolio
7. Budget amounts MUST be tracked in cents

### 4.7 Collaboration and Voting
1. Each idea MAY have multiple collaboration spaces of different types
2. Private spaces MUST restrict access to listed participant_ids only
3. Each user MAY cast exactly one vote per idea (upvote, downvote, or neutral)
4. Vote weights MAY differ based on voter role or expertise level
5. Leaderboards MUST rank ideas by total vote score (upvotes minus downvotes, weighted)
6. Collaboration spaces MUST support file attachments with size limits
7. Moderators MAY remove inappropriate content from collaboration spaces

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.innovation.v1;

service InnovationService {
    rpc GetCampaign(GetCampaignRequest) returns (GetCampaignResponse);
    rpc SubmitIdea(SubmitIdeaRequest) returns (SubmitIdeaResponse);
    rpc EvaluateIdea(EvaluateIdeaRequest) returns (EvaluateIdeaResponse);
    rpc AdvanceGate(AdvanceGateRequest) returns (AdvanceGateResponse);
    rpc GetPortfolioAnalysis(GetPortfolioAnalysisRequest) returns (GetPortfolioAnalysisResponse);
    rpc AllocateResources(AllocateResourcesRequest) returns (AllocateResourcesResponse);
    rpc GetMetrics(GetMetricsRequest) returns (GetMetricsResponse);
    rpc VoteIdea(VoteIdeaRequest) returns (VoteIdeaResponse);
}

// --- Innovation Campaign ---
message InnovationCampaign {
    string id = 1; string tenant_id = 2; string campaign_name = 3; string campaign_code = 4;
    string description = 5; string campaign_type = 6; string theme = 7; string objectives = 8;
    string target_audience = 9; string submission_start = 10; string submission_end = 11;
    string evaluation_start = 12; string evaluation_end = 13; int32 max_ideas_per_submitter = 14;
    string status = 15; string owner_id = 16; string review_committee = 17;
    string reward_config = 18; int32 total_ideas = 19; int32 total_evaluators = 20;
    int32 total_votes = 21; int64 budget_allocated_cents = 22; int64 budget_consumed_cents = 23;
    string tags = 24; string metadata = 25;
    string created_at = 26; string updated_at = 27; string created_by = 28; string updated_by = 29;
    int32 version = 30; bool is_active = 31;
}

// --- Idea ---
message Idea {
    string id = 1; string tenant_id = 2; string campaign_id = 3; string idea_title = 4;
    string idea_code = 5; string idea_description = 6; string problem_statement = 7;
    string proposed_solution = 8; string expected_benefits = 9;
    int64 estimated_effort_cents = 10; int64 estimated_revenue_cents = 11;
    int64 estimated_cost_savings_cents = 12; string priority = 13; string idea_status = 14;
    string submission_source = 15; string submitter_id = 16; string submitter_department = 17;
    string category_id = 18; string assigned_reviewer_id = 19; string current_gate_id = 20;
    string current_gate_status = 21; string implementation_complexity = 22;
    int32 time_to_market_days = 23; string target_market = 24; string attachments = 25;
    string tags = 26;
    string created_at = 27; string updated_at = 28; string created_by = 29; string updated_by = 30;
    int32 version = 31; bool is_active = 32;
}

// --- Idea Category ---
message IdeaCategory {
    string id = 1; string tenant_id = 2; string category_name = 3; string category_code = 4;
    string description = 5; string parent_category_id = 6; int32 category_level = 7;
    string icon = 8; string color = 9; int32 sort_order = 10; bool is_active = 11;
    string created_at = 12; string updated_at = 13; string created_by = 14; string updated_by = 15;
    int32 version = 16;
}

// --- Idea Evaluation ---
message IdeaEvaluation {
    string id = 1; string tenant_id = 2; string idea_id = 3; string evaluator_id = 4;
    string evaluation_type = 5; string criteria_scores = 6; double overall_score_real = 7;
    double weighted_score_real = 8; string recommendation = 9; string strengths = 10;
    string weaknesses = 11; string comments = 12; string risk_assessment = 13;
    double feasibility_score_real = 14; double impact_score_real = 15;
    double strategic_alignment_score_real = 16; bool is_completed = 17; string completed_at = 18;
    string created_at = 19; string updated_at = 20; string created_by = 21; string updated_by = 22;
    int32 version = 23;
}

// --- Stage Gate Process ---
message StageGateProcess {
    string id = 1; string tenant_id = 2; string process_name = 3; string process_code = 4;
    string description = 5; string applicable_campaign_types = 6; string gates = 7;
    int32 total_gates = 8; int32 estimated_total_days = 9; string approval_type = 10;
    double approval_threshold_real = 11; bool is_default = 12; bool is_active = 13;
    string created_at = 14; string updated_at = 15; string created_by = 16; string updated_by = 17;
    int32 version = 18;
}

// --- Stage Gate Review ---
message StageGateReview {
    string id = 1; string tenant_id = 2; string idea_id = 3; string process_id = 4;
    int32 gate_number = 5; string gate_name = 6; string gate_criteria = 7; string status = 8;
    string review_date = 9; string reviewers = 10; string review_scores = 11;
    double aggregate_score_real = 12; string review_comments = 13; string conditions = 14;
    string decision_rationale = 15; string decider_id = 16; string decided_at = 17;
    string scheduled_date = 18; int32 duration_days = 19;
    string created_at = 20; string updated_at = 21; string created_by = 22; string updated_by = 23;
    int32 version = 24;
}

// --- Innovation Portfolio ---
message InnovationPortfolio {
    string id = 1; string tenant_id = 2; string portfolio_name = 3; string portfolio_code = 4;
    string description = 5; string portfolio_type = 6; string strategy_framework = 7;
    string target_allocation = 8; int64 total_budget_cents = 9; int64 allocated_budget_cents = 10;
    int64 consumed_budget_cents = 11; int32 total_ideas = 12; int32 active_ideas = 13;
    int32 completed_ideas = 14; int32 failed_ideas = 15; double expected_roi_real = 16;
    double actual_roi_real = 17; string owner_id = 18; string rebalancing_frequency = 19;
    string last_rebalanced_at = 20; bool is_active = 21;
    string created_at = 22; string updated_at = 23; string created_by = 24; string updated_by = 25;
    int32 version = 26;
}

// --- Resource Allocation ---
message ResourceAllocation {
    string id = 1; string tenant_id = 2; string portfolio_id = 3; string idea_id = 4;
    string allocation_type = 5; int64 allocated_amount_cents = 6; int64 consumed_amount_cents = 7;
    int64 remaining_amount_cents = 8; double allocated_headcount = 9; double consumed_headcount = 10;
    string period_start = 11; string period_end = 12; string cost_center = 13;
    string gl_account_id = 14; string project_id = 15; string status = 16;
    string justification = 17; string approval_status = 18; string approved_by = 19;
    string approved_at = 20;
    string created_at = 21; string updated_at = 22; string created_by = 23; string updated_by = 24;
    int32 version = 25;
}

// --- Innovation Metrics ---
message InnovationMetrics {
    string id = 1; string tenant_id = 2; string metric_date = 3; string portfolio_id = 4;
    string campaign_id = 5; int32 total_ideas_submitted = 6; int32 total_ideas_approved = 7;
    int32 total_ideas_implemented = 8; int32 total_ideas_rejected = 9;
    double conversion_rate_pct = 10; double avg_time_to_evaluate_days = 11;
    double avg_time_to_market_days = 12; double avg_evaluation_score_real = 13;
    int64 total_budget_allocated_cents = 14; int64 total_budget_consumed_cents = 15;
    int64 total_revenue_generated_cents = 16; int64 total_cost_savings_cents = 17;
    double innovation_roi_real = 18; double ideas_per_employee = 19;
    double participation_rate_pct = 20; int32 evaluator_count = 21;
    int32 active_gates_count = 22; int32 gates_passed_count = 23; int32 gates_failed_count = 24;
    string created_at = 25; string updated_at = 26; string created_by = 27; string updated_by = 28;
}

// --- Collaboration Space ---
message CollaborationSpace {
    string id = 1; string tenant_id = 2; string idea_id = 3; string space_name = 4;
    string space_type = 5; string description = 6; string content = 7;
    string participant_ids = 8; string moderator_ids = 9; bool is_private = 10;
    int32 max_participants = 11; int32 total_posts = 12; int32 total_attachments = 13;
    string last_activity_at = 14; bool is_active = 15;
    string created_at = 16; string updated_at = 17; string created_by = 18; string updated_by = 19;
    int32 version = 20;
}

// --- Idea Vote ---
message IdeaVote {
    string id = 1; string tenant_id = 2; string idea_id = 3; string voter_id = 4;
    string vote_type = 5; double vote_weight_real = 6; string comment = 7; string voted_at = 8;
    string created_at = 9; string updated_at = 10; string created_by = 11; string updated_by = 12;
}

// --- RPC Request/Response Messages ---
message GetCampaignRequest { string tenant_id = 1; string id = 2; }
message GetCampaignResponse { InnovationCampaign data = 1; }

message SubmitIdeaRequest { string tenant_id = 1; string campaign_id = 2; string idea_title = 3; string idea_description = 4; string problem_statement = 5; string proposed_solution = 6; string submitter_id = 7; }
message SubmitIdeaResponse { Idea data = 1; }

message EvaluateIdeaRequest { string tenant_id = 1; string idea_id = 2; string evaluator_id = 3; string evaluation_type = 4; string criteria_scores = 5; string recommendation = 6; }
message EvaluateIdeaResponse { IdeaEvaluation data = 1; }

message AdvanceGateRequest { string tenant_id = 1; string idea_id = 2; int32 gate_number = 3; string decision = 4; string decision_rationale = 5; string decider_id = 6; }
message AdvanceGateResponse { StageGateReview review = 1; Idea idea = 2; }

message GetPortfolioAnalysisRequest { string tenant_id = 1; string portfolio_id = 2; }
message GetPortfolioAnalysisResponse { InnovationPortfolio portfolio = 1; repeated ResourceAllocation allocations = 2; repeated InnovationMetrics metrics = 3; }

message AllocateResourcesRequest { string tenant_id = 1; string portfolio_id = 2; string idea_id = 3; string allocation_type = 4; int64 allocated_amount_cents = 5; string period_start = 6; string period_end = 7; }
message AllocateResourcesResponse { ResourceAllocation data = 1; }

message GetMetricsRequest { string tenant_id = 1; string metric_date = 2; string portfolio_id = 3; string campaign_id = 4; }
message GetMetricsResponse { repeated InnovationMetrics metrics = 1; }

message VoteIdeaRequest { string tenant_id = 1; string idea_id = 2; string voter_id = 3; string vote_type = 4; double vote_weight_real = 5; string comment = 6; }
message VoteIdeaResponse { IdeaVote data = 1; }

message ListCampaignsRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListCampaignsResponse { repeated InnovationCampaign items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListIdeasRequest { string tenant_id = 1; string campaign_id = 2; int32 page_size = 3; string page_token = 4; }
message ListIdeasResponse { repeated Idea items = 1; int32 total_count = 2; string next_page_token = 3; }

message ListPortfoliosRequest { string tenant_id = 1; int32 page_size = 2; string page_token = 3; }
message ListPortfoliosResponse { repeated InnovationPortfolio items = 1; int32 total_count = 2; string next_page_token = 3; }
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **PLM:** Product data, component structures, engineering change orders for idea feasibility assessment
- **Project-Management:** Project execution status, resource utilization, timeline tracking for implemented ideas
- **GL:** Budget availability checks, cost center validation, actual expenditure data for allocation tracking
- **Auth:** User identity, organizational hierarchy, role-based permissions for campaign and idea management
- **Workflow:** Approval process definitions, gate review routing, escalation rules
- **Document-Management:** File attachments for ideas, evaluation evidence, gate review documents

### 6.2 Data Published To
- **PLM:** Approved ideas transitioning to product development, new product initiation requests
- **Project-Management:** Approved ideas with allocations as new project proposals, resource assignments
- **GL:** Budget commitment entries, allocation postings, actual cost tracking for innovation spend
- **Reporting:** Innovation metrics, portfolio analysis, campaign dashboards, trend reports
- **Workflow:** Gate review approval tasks, evaluation assignment tasks, allocation approval routing
- **Notification:** Campaign launch alerts, evaluation assignment notifications, gate review reminders, status change alerts

### 6.3 Integration Matrix

| Source Service | Integration Type | Data Exchanged |
|----------------|------------------|----------------|
| PLM | API call | Product feasibility data, component costs, engineering estimates |
| PM | API call + Events | Project creation for approved ideas, status synchronization |
| GL | API call + Events | Budget checks, allocation postings, actual cost data |
| Auth | API call | User validation, role resolution, permission checks |
| Workflow | Event-driven | Approval routing, gate review tasks, escalation |
| Document-Management | API call | File attachments, document storage, content retrieval |
| Reporting | Data push | Metrics aggregation, portfolio data, campaign statistics |
| Notification | Event-driven | Campaign alerts, evaluation reminders, status changes |

---

## 7. Events

| Event | Payload | Description |
|-------|---------|-------------|
| `innovation.campaign.launched` | `{ campaign_id, campaign_name, campaign_type, submission_start, submission_end, timestamp }` | Campaign opened for idea submission |
| `innovation.idea.submitted` | `{ idea_id, idea_title, campaign_id, submitter_id, timestamp }` | New idea submitted to a campaign |
| `innovation.idea.evaluated` | `{ idea_id, evaluation_id, evaluator_id, overall_score, recommendation, timestamp }` | Idea evaluation completed |
| `innovation.gate.review.completed` | `{ idea_id, gate_number, gate_name, status, aggregate_score, decider_id, timestamp }` | Stage gate review decision made |
| `innovation.idea.approved` | `{ idea_id, idea_title, campaign_id, final_score, approved_by, timestamp }` | Idea fully approved through all gates |
| `innovation.idea.rejected` | `{ idea_id, idea_title, campaign_id, rejection_reason, timestamp }` | Idea rejected at a gate |
| `innovation.portfolio.rebalanced` | `{ portfolio_id, portfolio_name, total_budget_cents, active_ideas_count, timestamp }` | Portfolio allocation adjusted |
| `innovation.allocation.approved` | `{ allocation_id, portfolio_id, idea_id, allocated_amount_cents, approved_by, timestamp }` | Resource allocation approved |
| `innovation.idea.implemented` | `{ idea_id, campaign_id, time_to_market_days, actual_revenue_cents, timestamp }` | Idea fully implemented in production |
| `innovation.vote.cast` | `{ idea_id, voter_id, vote_type, vote_weight, total_votes, timestamp }` | Vote recorded on an idea |
| `innovation.metrics.updated` | `{ metric_date, total_ideas, conversion_rate_pct, innovation_roi_real, timestamp }` | Innovation metrics recalculated |

---

## 8. Migrations

1. V001: `idea_categories`
2. V002: `innovation_campaigns`
3. V003: `ideas`
4. V004: `idea_evaluations`
5. V005: `stage_gate_processes`
6. V006: `stage_gate_reviews`
7. V007: `innovation_portfolios`
8. V008: `resource_allocations`
9. V009: `innovation_metrics`
10. V010: `collaboration_spaces`
11. V011: `idea_votes`
12. V012: Triggers for `updated_at`
