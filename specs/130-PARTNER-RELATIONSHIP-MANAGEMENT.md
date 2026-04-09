# 130 - Partner Relationship Management Specification

## 1. Domain Overview

Partner Relationship Management (PRM) enables organizations to manage their indirect sales channels through partner portals, deal registration, lead distribution, pipeline visibility, partner tiering, and performance tracking. It extends CRM capabilities to resellers, distributors, system integrators, and referral partners.

**Bounded Context:** Channel Partners & Indirect Sales
**Service Name:** `prm-service`
**Database:** `data/prm.db`
**HTTP Port:** 8210 | **gRPC Port:** 9210

---

## 2. Database Schema

### 2.1 Partners
```sql
CREATE TABLE prm_partners (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    partner_code TEXT NOT NULL,
    company_name TEXT NOT NULL,
    partner_type TEXT NOT NULL CHECK(partner_type IN ('RESELLER','DISTRIBUTOR','SYSTEM_INTEGRATOR','REFERRAL','TECHNOLOGY','OEM','CONSULTANT')),
    tier TEXT NOT NULL DEFAULT 'STANDARD'
        CHECK(tier IN ('STANDARD','SILVER','GOLD','PLATINUM','DIAMOND')),
    status TEXT NOT NULL DEFAULT 'PROSPECT'
        CHECK(status IN ('PROSPECT','APPLIED','ACTIVE','SUSPENDED','TERMINATED')),
    primary_contact_name TEXT,
    primary_contact_email TEXT,
    primary_contact_phone TEXT,
    address_line1 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT NOT NULL,
    website TEXT,
    certifications TEXT,                 -- JSON array of certification IDs
    specialization TEXT,                 -- JSON array: ["cloud", "security", "erp"]
    territory TEXT,
    parent_partner_id TEXT,              -- For partner hierarchies
    contract_start_date TEXT,
    contract_end_date TEXT,
    discount_percentage REAL NOT NULL DEFAULT 0.0,
    credit_limit_cents INTEGER,
    performance_score REAL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (parent_partner_id) REFERENCES prm_partners(id) ON DELETE RESTRICT,
    UNIQUE(tenant_id, partner_code)
);

CREATE INDEX idx_prm_partners_tenant_status ON prm_partners(tenant_id, status);
CREATE INDEX idx_prm_partners_tenant_tier ON prm_partners(tenant_id, tier);
```

### 2.2 Partner Contacts
```sql
CREATE TABLE prm_partner_contacts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT NOT NULL,
    phone TEXT,
    title TEXT,
    role TEXT NOT NULL DEFAULT 'CONTACT'
        CHECK(role IN ('CONTACT','SALES_REP','TECHNICAL','EXECUTIVE','MARKETING','ADMIN')),
    is_primary INTEGER NOT NULL DEFAULT 0,
    portal_access_enabled INTEGER NOT NULL DEFAULT 0,
    portal_user_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (partner_id) REFERENCES prm_partners(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, partner_id, email)
);
```

### 2.3 Deal Registrations
```sql
CREATE TABLE prm_deal_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    deal_name TEXT NOT NULL,
    deal_code TEXT NOT NULL,
    customer_company_name TEXT NOT NULL,
    customer_contact_name TEXT,
    customer_contact_email TEXT,
    customer_country TEXT,
    deal_value_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    products TEXT,                       -- JSON array of product IDs and quantities
    expected_close_date TEXT NOT NULL,
    deal_stage TEXT NOT NULL DEFAULT 'IDENTIFIED'
        CHECK(deal_stage IN ('IDENTIFIED','QUALIFIED','PROPOSAL','NEGOTIATION','COMMITTED','WON','LOST')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','DUPLICATE','EXPIRED','CONVERTED')),
    conflict_status TEXT CHECK(conflict_status IN ('NONE','POTENTIAL','CONFLICT')),
    competing_partners TEXT,             -- JSON array of partner IDs also pursuing
    internal_owner_id TEXT,              -- Internal sales rep assigned
    registration_date TEXT NOT NULL DEFAULT (datetime('now')),
    approval_date TEXT,
    rejection_reason TEXT,
    protection_end_date TEXT,            -- Deal protection expiration
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, deal_code)
);

CREATE INDEX idx_prm_deals_tenant_partner ON prm_deal_registrations(tenant_id, partner_id);
CREATE INDEX idx_prm_deals_tenant_status ON prm_deal_registrations(tenant_id, status);
```

### 2.4 Lead Distribution
```sql
CREATE TABLE prm_leads (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    lead_source TEXT NOT NULL,
    company_name TEXT NOT NULL,
    contact_name TEXT NOT NULL,
    contact_email TEXT NOT NULL,
    contact_phone TEXT,
    country TEXT,
    industry TEXT,
    estimated_value_cents INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    description TEXT,
    requirements TEXT,
    assigned_partner_id TEXT,
    assigned_date TEXT,
    status TEXT NOT NULL DEFAULT 'NEW'
        CHECK(status IN ('NEW','ASSIGNED','ACCEPTED','WORKING','QUALIFIED','CONVERTED','REJECTED','EXPIRED')),
    partner_feedback TEXT,
    conversion_date TEXT,
    converted_deal_id TEXT,
    expiration_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (assigned_partner_id) REFERENCES prm_partners(id) ON DELETE SET NULL,
    FOREIGN KEY (converted_deal_id) REFERENCES prm_deal_registrations(id) ON DELETE SET NULL
);

CREATE INDEX idx_prm_leads_tenant_status ON prm_leads(tenant_id, status);
CREATE INDEX idx_prm_leads_tenant_partner ON prm_leads(tenant_id, assigned_partner_id);
```

### 2.5 Partner Performance
```sql
CREATE TABLE prm_partner_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    period TEXT NOT NULL,                -- "2024-Q1", "2024-01"
    deals_registered INTEGER NOT NULL DEFAULT 0,
    deals_won INTEGER NOT NULL DEFAULT 0,
    win_rate_pct REAL NOT NULL DEFAULT 0.0,
    revenue_cents INTEGER NOT NULL DEFAULT 0,
    pipeline_value_cents INTEGER NOT NULL DEFAULT 0,
    leads_received INTEGER NOT NULL DEFAULT 0,
    leads_converted INTEGER NOT NULL DEFAULT 0,
    lead_conversion_pct REAL NOT NULL DEFAULT 0.0,
    certifications_earned INTEGER NOT NULL DEFAULT 0,
    training_completed INTEGER NOT NULL DEFAULT 0,
    customer_satisfaction_avg REAL,
    support_tickets INTEGER NOT NULL DEFAULT 0,
    composite_score REAL NOT NULL DEFAULT 0.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, partner_id, period)
);
```

### 2.6 Partner Programs
```sql
CREATE TABLE prm_programs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    program_name TEXT NOT NULL,
    description TEXT,
    tier_requirements TEXT NOT NULL,      -- JSON: { "SILVER": { "revenue_min": 100000, "certifications": 2 } }
    benefits TEXT NOT NULL,              -- JSON: { "discount_pct": 15, "marketing_fund_pct": 3, "training_credits": 50 }
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('DRAFT','ACTIVE','INACTIVE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, program_name)
);
```

---

## 3. REST API Endpoints

### 3.1 Partners
```
GET    /api/v1/prm/partners                             Permission: prm.partners.read
GET    /api/v1/prm/partners/{id}                        Permission: prm.partners.read
POST   /api/v1/prm/partners                             Permission: prm.partners.create
PUT    /api/v1/prm/partners/{id}                        Permission: prm.partners.update
PUT    /api/v1/prm/partners/{id}/tier                   Permission: prm.partners.tier
PUT    /api/v1/prm/partners/{id}/status                 Permission: prm.partners.status
GET    /api/v1/prm/partners/{id}/contacts               Permission: prm.partners.read
POST   /api/v1/prm/partners/{id}/contacts               Permission: prm.partners.update
GET    /api/v1/prm/partners/{id}/performance             Permission: prm.partners.read
```

### 3.2 Deal Registration
```
GET    /api/v1/prm/deals                                Permission: prm.deals.read
GET    /api/v1/prm/deals/{id}                           Permission: prm.deals.read
POST   /api/v1/prm/deals                                Permission: prm.deals.create
PUT    /api/v1/prm/deals/{id}                           Permission: prm.deals.update
POST   /api/v1/prm/deals/{id}/approve                   Permission: prm.deals.approve
POST   /api/v1/prm/deals/{id}/reject                    Permission: prm.deals.reject
POST   /api/v1/prm/deals/{id}/check-conflict            Permission: prm.deals.check
GET    /api/v1/prm/deals/pipeline                       Permission: prm.deals.read
```

### 3.3 Lead Distribution
```
GET    /api/v1/prm/leads                                Permission: prm.leads.read
POST   /api/v1/prm/leads                                Permission: prm.leads.create
PUT    /api/v1/prm/leads/{id}                           Permission: prm.leads.update
POST   /api/v1/prm/leads/{id}/assign                    Permission: prm.leads.assign
POST   /api/v1/prm/leads/{id}/accept                    Permission: prm.leads.partner_action
POST   /api/v1/prm/leads/{id}/reject                    Permission: prm.leads.partner_action
POST   /api/v1/prm/leads/auto-distribute                 Permission: prm.leads.distribute
```

### 3.4 Partner Portal (Partner-Facing)
```
GET    /api/v1/prm/portal/profile                       Permission: prm.portal.read
GET    /api/v1/prm/portal/deals                         Permission: prm.portal.read
POST   /api/v1/prm/portal/deals                         Permission: prm.portal.create
GET    /api/v1/prm/portal/leads                         Permission: prm.portal.read
PUT    /api/v1/prm/portal/leads/{id}/status             Permission: prm.portal.update
GET    /api/v1/prm/portal/performance                   Permission: prm.portal.read
GET    /api/v1/prm/portal/resources                     Permission: prm.portal.read
```

### 3.5 Programs & Tiering
```
GET    /api/v1/prm/programs                             Permission: prm.programs.read
POST   /api/v1/prm/programs                             Permission: prm.programs.create
PUT    /api/v1/prm/programs/{id}                        Permission: prm.programs.update
POST   /api/v1/prm/programs/{id}/evaluate               Permission: prm.programs.evaluate
```

---

## 4. Business Rules

### 4.1 Deal Registration
- Partners register deals to get price protection and avoid channel conflict
- Duplicate detection: same customer + similar products flags potential conflict
- Deal protection period (default 90 days) prevents other partners from pursuing
- Tier determines deal registration approval priority (higher tier = faster approval)
- Rejection requires documented reason

### 4.2 Lead Distribution
- Leads auto-distributed based on partner territory, specialization, and tier
- Distribution algorithm: weighted scoring across match criteria
- Partner MUST accept or reject within configurable timeframe (default 48 hours)
- Unaccepted leads auto-reclaimed and redistributed
- Lead quality score provided to partners for prioritization

### 4.3 Partner Tiering
- Tier advancement based on composite score (revenue, certifications, satisfaction)
- Annual tier evaluation cycle (configurable)
- Tier determines: discount rates, marketing funds, lead priority, training credits
- Tier downgrade has grace period before benefits are reduced
- Diamond/Platinum partners get dedicated internal account managers

### 4.4 Performance Tracking
- Monthly/quarterly metrics aggregated from deal and lead data
- Composite score = weighted average of win_rate (30%), revenue (30%), satisfaction (20%), certifications (20%)
- Benchmarking against peer partners in same tier
- Automated alerts for underperforming partners

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.prm.v1;

service PartnerRelationshipService {
    rpc RegisterDeal(RegisterDealRequest) returns (RegisterDealResponse);
    rpc ApproveDeal(ApproveDealRequest) returns (ApproveDealResponse);
    rpc DistributeLead(DistributeLeadRequest) returns (DistributeLeadResponse);
    rpc GetPartnerPerformance(GetPartnerPerformanceRequest) returns (GetPartnerPerformanceResponse);
    rpc CheckConflict(CheckConflictRequest) returns (CheckConflictResponse);
    rpc EvaluateTier(EvaluateTierRequest) returns (EvaluateTierResponse);
}

// Entity messages
message PrmPartner {
    string id = 1;
    string tenant_id = 2;
    string partner_code = 3;
    string company_name = 4;
    string partner_type = 5;
    string tier = 6;
    string status = 7;
    string primary_contact_name = 8;
    string primary_contact_email = 9;
    string primary_contact_phone = 10;
    string address_line1 = 11;
    string city = 12;
    string state_province = 13;
    string postal_code = 14;
    string country = 15;
    string website = 16;
    string certifications = 17;
    string specialization = 18;
    string territory = 19;
    string parent_partner_id = 20;
    string contract_start_date = 21;
    string contract_end_date = 22;
    double discount_percentage = 23;
    int64 credit_limit_cents = 24;
    double performance_score = 25;
    string created_at = 26;
    string updated_at = 27;
}

message PrmPartnerContact {
    string id = 1;
    string tenant_id = 2;
    string partner_id = 3;
    string first_name = 4;
    string last_name = 5;
    string email = 6;
    string phone = 7;
    string title = 8;
    string role = 9;
    bool is_primary = 10;
    bool portal_access_enabled = 11;
    string portal_user_id = 12;
    string created_at = 13;
    string updated_at = 14;
}

message PrmDealRegistration {
    string id = 1;
    string tenant_id = 2;
    string partner_id = 3;
    string deal_name = 4;
    string deal_code = 5;
    string customer_company_name = 6;
    string customer_contact_name = 7;
    string customer_contact_email = 8;
    string customer_country = 9;
    int64 deal_value_cents = 10;
    string currency_code = 11;
    string products = 12;
    string expected_close_date = 13;
    string deal_stage = 14;
    string status = 15;
    string conflict_status = 16;
    string competing_partners = 17;
    string internal_owner_id = 18;
    string registration_date = 19;
    string approval_date = 20;
    string rejection_reason = 21;
    string protection_end_date = 22;
    string notes = 23;
    string created_at = 24;
    string updated_at = 25;
}

message PrmLead {
    string id = 1;
    string tenant_id = 2;
    string lead_source = 3;
    string company_name = 4;
    string contact_name = 5;
    string contact_email = 6;
    string contact_phone = 7;
    string country = 8;
    string industry = 9;
    int64 estimated_value_cents = 10;
    string currency_code = 11;
    string description = 12;
    string requirements = 13;
    string assigned_partner_id = 14;
    string assigned_date = 15;
    string status = 16;
    string partner_feedback = 17;
    string conversion_date = 18;
    string converted_deal_id = 19;
    string expiration_date = 20;
    string created_at = 21;
    string updated_at = 22;
}

message PrmPartnerMetrics {
    string id = 1;
    string tenant_id = 2;
    string partner_id = 3;
    string period = 4;
    int32 deals_registered = 5;
    int32 deals_won = 6;
    double win_rate_pct = 7;
    int64 revenue_cents = 8;
    int64 pipeline_value_cents = 9;
    int32 leads_received = 10;
    int32 leads_converted = 11;
    double lead_conversion_pct = 12;
    int32 certifications_earned = 13;
    int32 training_completed = 14;
    double customer_satisfaction_avg = 15;
    int32 support_tickets = 16;
    double composite_score = 17;
    string created_at = 18;
    string updated_at = 19;
}

message PrmProgram {
    string id = 1;
    string tenant_id = 2;
    string program_name = 3;
    string description = 4;
    string tier_requirements = 5;
    string benefits = 6;
    string status = 7;
    string created_at = 8;
    string updated_at = 9;
}

// Request/Response messages
message RegisterDealRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string deal_name = 3;
    string customer_company_name = 4;
    string customer_contact_name = 5;
    string customer_contact_email = 6;
    string customer_country = 7;
    int64 deal_value_cents = 8;
    string currency_code = 9;
    string products = 10;
    string expected_close_date = 11;
    string notes = 12;
}

message RegisterDealResponse {
    PrmDealRegistration data = 1;
    bool conflict_detected = 2;
}

message ApproveDealRequest {
    string tenant_id = 1;
    string id = 2;
    string internal_owner_id = 3;
    string protection_end_date = 4;
}

message ApproveDealResponse {
    PrmDealRegistration data = 1;
}

message DistributeLeadRequest {
    string tenant_id = 1;
    string lead_id = 2;
    string partner_id = 3;
    repeated string criteria = 4;
}

message DistributeLeadResponse {
    PrmLead data = 1;
    string assigned_partner_id = 2;
}

message GetPartnerPerformanceRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string period = 3;
}

message GetPartnerPerformanceResponse {
    PrmPartnerMetrics metrics = 1;
    PrmPartner partner = 2;
    repeated PrmPartnerMetrics history = 3;
}

message ConflictMatch {
    string deal_id = 1;
    string deal_name = 2;
    string partner_id = 3;
    string partner_name = 4;
    string conflict_type = 5;
    double match_score = 6;
}

message CheckConflictRequest {
    string tenant_id = 1;
    string customer_company_name = 2;
    string products = 3;
    string customer_country = 4;
}

message CheckConflictResponse {
    bool has_conflict = 1;
    repeated ConflictMatch conflicts = 2;
}

message EvaluateTierRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string program_id = 3;
}

message EvaluateTierResponse {
    string partner_id = 1;
    string current_tier = 2;
    string recommended_tier = 3;
    double composite_score = 4;
    bool tier_changed = 5;
    PrmPartnerMetrics latest_metrics = 6;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **sales-service**: Lead and opportunity synchronization
- **commerce-service**: Partner storefront, pricing
- **cpq-service**: Partner-specific pricing and quoting
- **channel-service**: Existing channel management integration
- **customerservice-service**: Partner support cases
- **marketing-service**: Co-marketing campaigns, MDF management
- **workflow-service**: Deal approval workflows

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `prm.partner.created` | New partner registered | partner_id, tier, type |
| `prm.partner.tier.changed` | Partner tier upgraded/downgraded | partner_id, old_tier, new_tier |
| `prm.deal.registered` | Deal registration submitted | deal_id, partner_id, value |
| `prm.deal.approved` | Deal registration approved | deal_id, protection_end_date |
| `prm.deal.conflict` | Potential conflict detected | deal_id, conflicting_partner_ids |
| `prm.lead.assigned` | Lead distributed to partner | lead_id, partner_id |
| `prm.lead.converted` | Lead converted to deal | lead_id, deal_id |
| `prm.performance.calculated` | Period metrics computed | partner_id, period, score |

---

## 7. Migrations

### Migration Order for prm-service:
1. V001: `prm_programs`
2. V002: `prm_partners`
3. V003: `prm_partner_contacts`
4. V004: `prm_deal_registrations`
5. V005: `prm_leads`
6. V006: `prm_partner_metrics`
7. V007: Triggers for `updated_at`
8. V008: Seed data (default partner program, tier requirements)
