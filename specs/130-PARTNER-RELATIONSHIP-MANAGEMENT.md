# 130 - Partner Relationship Management Specification

## 1. Domain Overview

Partner Relationship Management (PRM) enables organizations to manage their indirect sales channels through partner portals, deal registration, lead distribution, pipeline visibility, partner tiering, and performance tracking. It extends CRM capabilities to resellers, distributors, system integrators, and referral partners.

**Bounded Context:** Channel Partners & Indirect Sales
**Service Name:** `prm-service`
**Database:** `data/prm.db`
**HTTP Port:** 8150 | **gRPC Port:** 9150

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
