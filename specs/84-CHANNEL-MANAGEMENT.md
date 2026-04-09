# 84 - Channel Management Service Specification

## 1. Domain Overview

| Attribute | Value |
|---|---|
| **Bounded Context** | Channel Management |
| **Service Name** | channel-service |
| **Database** | channel_db |
| **HTTP Port** | 8116 |
| **gRPC Port** | 9116 |

The Channel Management module provides comprehensive partner relationship management (PRM) capabilities including partner registration, agreement management, deal registration, lead distribution, co-marketing fund (MDF) administration, partner performance scoring, partner portal management, training tracking, and tier-based program management. It enables vendors to manage indirect sales channels and drive partner engagement and revenue.

---

## 2. Database Schema

```sql
CREATE TABLE partner_profiles (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    partner_code TEXT NOT NULL UNIQUE,
    company_name TEXT NOT NULL,
    legal_name TEXT,
    tax_id TEXT,
    partner_type_id TEXT NOT NULL,
    tier_id TEXT,
    status TEXT NOT NULL DEFAULT 'PROSPECT',
    primary_contact_id TEXT,
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state_province TEXT,
    postal_code TEXT,
    country TEXT,
    website_url TEXT,
    phone TEXT,
    email TEXT,
    industry TEXT,
    annual_revenue INTEGER,
    employee_count INTEGER,
    territories TEXT,
    certifications TEXT,
    competencies TEXT,
    parent_partner_id TEXT,
    bank_name TEXT,
    bank_account_number TEXT,
    payment_terms TEXT,
    credit_limit INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    notes TEXT,
    registered_at TEXT,
    activated_at TEXT,
    deactivated_at TEXT,
    deactivation_reason TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_partner_profiles_tenant ON partner_profiles(tenant_id);
CREATE INDEX idx_partner_profiles_type ON partner_profiles(tenant_id, partner_type_id);
CREATE INDEX idx_partner_profiles_status ON partner_profiles(tenant_id, status);
CREATE INDEX idx_partner_profiles_tier ON partner_profiles(tenant_id, tier_id);
CREATE INDEX idx_partner_profiles_country ON partner_profiles(tenant_id, country);
CREATE INDEX idx_partner_profiles_company ON partner_profiles(company_name);

CREATE TABLE partner_types (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    type_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    default_discount_percent INTEGER,
    deal_registration_enabled INTEGER NOT NULL DEFAULT 1,
    lead_distribution_enabled INTEGER NOT NULL DEFAULT 1,
    mdf_eligible INTEGER NOT NULL DEFAULT 0,
    training_required TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_partner_types_tenant ON partner_types(tenant_id);

CREATE TABLE partner_agreements (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    agreement_number TEXT NOT NULL UNIQUE,
    partner_id TEXT NOT NULL,
    agreement_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    auto_renew INTEGER NOT NULL DEFAULT 0,
    renewal_period_months INTEGER,
    discount_schedule TEXT,
    margin_percent INTEGER,
    payment_terms TEXT,
    minimum_commitment INTEGER,
    minimum_commitment_currency TEXT,
    territory TEXT,
    exclusivity_type TEXT,
    terms_and_conditions TEXT,
    signed_by_partner TEXT,
    signed_by_vendor TEXT,
    signed_at TEXT,
    terminated_at TEXT,
    termination_reason TEXT,
    notes TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_partner_agreements_tenant ON partner_agreements(tenant_id);
CREATE INDEX idx_partner_agreements_partner ON partner_agreements(tenant_id, partner_id);
CREATE INDEX idx_partner_agreements_status ON partner_agreements(tenant_id, status);
CREATE INDEX idx_partner_agreements_type ON partner_agreements(tenant_id, agreement_type);

CREATE TABLE deal_registrations (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    deal_number TEXT NOT NULL UNIQUE,
    partner_id TEXT NOT NULL,
    partner_contact_id TEXT,
    customer_name TEXT NOT NULL,
    customer_contact_name TEXT,
    customer_contact_email TEXT,
    customer_contact_phone TEXT,
    customer_industry TEXT,
    customer_city TEXT,
    customer_country TEXT,
    opportunity_name TEXT NOT NULL,
    opportunity_description TEXT,
    estimated_value INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    estimated_close_date TEXT,
    product_interest TEXT,
    competitive_situation TEXT,
    status TEXT NOT NULL DEFAULT 'SUBMITTED',
    review_notes TEXT,
    reviewed_by TEXT,
    reviewed_at TEXT,
    approved_value INTEGER,
    approved_at TEXT,
    expires_at TEXT,
    protection_period_days INTEGER NOT NULL DEFAULT 90,
    sales_order_id TEXT,
    commission_percent INTEGER,
    commission_amount INTEGER,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_deal_registrations_tenant ON deal_registrations(tenant_id);
CREATE INDEX idx_deal_registrations_partner ON deal_registrations(tenant_id, partner_id);
CREATE INDEX idx_deal_registrations_status ON deal_registrations(tenant_id, status);
CREATE INDEX idx_deal_registrations_customer ON deal_registrations(tenant_id, customer_name);

CREATE TABLE partner_lead_distributions (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    lead_number TEXT NOT NULL UNIQUE,
    partner_id TEXT NOT NULL,
    lead_source TEXT NOT NULL,
    lead_name TEXT NOT NULL,
    lead_company TEXT,
    lead_email TEXT,
    lead_phone TEXT,
    lead_title TEXT,
    lead_industry TEXT,
    lead_city TEXT,
    lead_country TEXT,
    lead_value_estimate INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    description TEXT,
    status TEXT NOT NULL DEFAULT 'ASSIGNED',
    assigned_at TEXT NOT NULL,
    accepted_at TEXT,
    declined_at TEXT,
    decline_reason TEXT,
    first_contact_at TEXT,
    qualified_at TEXT,
    converted_at TEXT,
    converted_opportunity_id TEXT,
    converted_sales_order_id TEXT,
    conversion_value INTEGER,
    lead_score INTEGER,
    follow_up_date TEXT,
    notes TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_lead_distributions_tenant ON partner_lead_distributions(tenant_id);
CREATE INDEX idx_lead_distributions_partner ON partner_lead_distributions(tenant_id, partner_id);
CREATE INDEX idx_lead_distributions_status ON partner_lead_distributions(tenant_id, status);

CREATE TABLE co_marketing_funds (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    fund_request_number TEXT NOT NULL UNIQUE,
    partner_id TEXT NOT NULL,
    program_name TEXT NOT NULL,
    fund_type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    activity_description TEXT NOT NULL,
    activity_start_date TEXT NOT NULL,
    activity_end_date TEXT NOT NULL,
    planned_activities TEXT,
    target_audience TEXT,
    expected_reach INTEGER,
    total_budget INTEGER,
    requested_amount INTEGER,
    approved_amount INTEGER,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    actual_spend INTEGER,
    proof_of_performance TEXT,
    activity_results TEXT,
    leads_generated INTEGER,
    pipeline_influenced INTEGER,
    revenue_attributed INTEGER,
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_notes TEXT,
    paid_amount INTEGER,
    paid_at TEXT,
    payment_reference TEXT,
    budget_period TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_co_marketing_funds_tenant ON co_marketing_funds(tenant_id);
CREATE INDEX idx_co_marketing_funds_partner ON co_marketing_funds(tenant_id, partner_id);
CREATE INDEX idx_co_marketing_funds_status ON co_marketing_funds(tenant_id, status);
CREATE INDEX idx_co_marketing_funds_type ON co_marketing_funds(tenant_id, fund_type);

CREATE TABLE partner_performance_scores (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    scoring_period TEXT NOT NULL,
    scoring_model TEXT NOT NULL,
    revenue_score INTEGER,
    revenue_amount INTEGER,
    deals_registered INTEGER,
    deals_won INTEGER,
    win_rate_percent INTEGER,
    lead_conversion_score INTEGER,
    leads_received INTEGER,
    leads_qualified INTEGER,
    leads_converted INTEGER,
    lead_conversion_rate_percent INTEGER,
    certification_score INTEGER,
    certified_individuals INTEGER,
    required_certifications INTEGER,
    certification_completion_percent INTEGER,
    customer_satisfaction_score INTEGER,
    average_csat INTEGER,
    support_tickets INTEGER,
    training_score INTEGER,
    training_courses_completed INTEGER,
    training_courses_required INTEGER,
    compliance_score INTEGER,
    compliance_violations INTEGER,
    overall_score INTEGER NOT NULL,
    tier_qualified TEXT,
    calculated_at TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_performance_scores_tenant ON partner_performance_scores(tenant_id);
CREATE INDEX idx_performance_scores_partner ON partner_performance_scores(tenant_id, partner_id);
CREATE INDEX idx_performance_scores_period ON partner_performance_scores(tenant_id, scoring_period);
CREATE UNIQUE INDEX idx_performance_scores_unique ON partner_performance_scores(tenant_id, partner_id, scoring_period, scoring_model);

CREATE TABLE partner_portals (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    portal_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    url TEXT NOT NULL,
    theme_config TEXT,
    enabled_modules TEXT,
    deal_registration_enabled INTEGER NOT NULL DEFAULT 1,
    lead_management_enabled INTEGER NOT NULL DEFAULT 1,
    mdf_management_enabled INTEGER NOT NULL DEFAULT 1,
    training_enabled INTEGER NOT NULL DEFAULT 1,
    content_library_enabled INTEGER NOT NULL DEFAULT 1,
    reporting_enabled INTEGER NOT NULL DEFAULT 1,
    default_language TEXT NOT NULL DEFAULT 'en',
    supported_languages TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_partner_portals_tenant ON partner_portals(tenant_id);

CREATE TABLE partner_training_records (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    partner_id TEXT NOT NULL,
    partner_user_id TEXT NOT NULL,
    course_id TEXT NOT NULL,
    course_name TEXT NOT NULL,
    course_type TEXT NOT NULL,
    enrollment_date TEXT,
    completion_date TEXT,
    expiration_date TEXT,
    status TEXT NOT NULL DEFAULT 'ENROLLED',
    score INTEGER,
    max_score INTEGER,
    passing_score INTEGER,
    passed INTEGER NOT NULL DEFAULT 0,
    attempts INTEGER NOT NULL DEFAULT 0,
    certificate_id TEXT,
    certificate_url TEXT,
    credit_hours INTEGER,
    mandatory INTEGER NOT NULL DEFAULT 0,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_training_records_tenant ON partner_training_records(tenant_id);
CREATE INDEX idx_training_records_partner ON partner_training_records(tenant_id, partner_id);
CREATE INDEX idx_training_records_user ON partner_training_records(tenant_id, partner_user_id);
CREATE INDEX idx_training_records_course ON partner_training_records(tenant_id, course_id);
CREATE INDEX idx_training_records_status ON partner_training_records(tenant_id, status);

CREATE TABLE partner_tier_programs (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    tier_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    level INTEGER NOT NULL,
    minimum_revenue INTEGER,
    minimum_deals INTEGER,
    minimum_certifications INTEGER,
    minimum_csat INTEGER,
    maximum_support_tickets INTEGER,
    discount_percent INTEGER,
    mdf_budget_percent INTEGER,
    lead_priority INTEGER,
    dedicated_support INTEGER NOT NULL DEFAULT 0,
    executive_sponsor INTEGER NOT NULL DEFAULT 0,
    enhanced_portal_access INTEGER NOT NULL DEFAULT 0,
    benefits_description TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_partner_tiers_tenant ON partner_tier_programs(tenant_id);
CREATE INDEX idx_partner_tiers_level ON partner_tier_programs(tenant_id, level);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/partners` | Register a new partner |
| GET | `/api/v1/partners/{partnerId}` | Get partner profile details |
| GET | `/api/v1/partners` | List/search partners with filters |
| PUT | `/api/v1/partners/{partnerId}` | Update partner profile |
| POST | `/api/v1/partners/{partnerId}/activate` | Activate a partner |
| POST | `/api/v1/partners/{partnerId}/deactivate` | Deactivate a partner |
| GET | `/api/v1/partner-types` | List partner types |
| POST | `/api/v1/partner-types` | Create partner type |
| PUT | `/api/v1/partner-types/{typeId}` | Update partner type |
| POST | `/api/v1/agreements` | Create partner agreement |
| GET | `/api/v1/agreements/{agreementId}` | Get agreement details |
| GET | `/api/v1/agreements` | List agreements with filters |
| PUT | `/api/v1/agreements/{agreementId}` | Update agreement |
| POST | `/api/v1/agreements/{agreementId}/activate` | Activate agreement |
| POST | `/api/v1/agreements/{agreementId}/terminate` | Terminate agreement |
| POST | `/api/v1/agreements/{agreementId}/renew` | Renew agreement |
| POST | `/api/v1/deal-registrations` | Submit a deal registration |
| GET | `/api/v1/deal-registrations/{dealId}` | Get deal registration details |
| GET | `/api/v1/deal-registrations` | List deal registrations |
| PUT | `/api/v1/deal-registrations/{dealId}` | Update deal registration |
| POST | `/api/v1/deal-registrations/{dealId}/approve` | Approve deal registration |
| POST | `/api/v1/deal-registrations/{dealId}/reject` | Reject deal registration |
| POST | `/api/v1/deal-registrations/{dealId}/close` | Close deal as won |
| POST | `/api/v1/leads/distribute` | Distribute lead to partner |
| GET | `/api/v1/leads/{leadId}` | Get lead distribution details |
| GET | `/api/v1/leads` | List distributed leads |
| PUT | `/api/v1/leads/{leadId}` | Update lead status |
| POST | `/api/v1/leads/{leadId}/accept` | Partner accepts lead |
| POST | `/api/v1/leads/{leadId}/decline` | Partner declines lead |
| POST | `/api/v1/leads/{leadId}/convert` | Convert lead to opportunity |
| POST | `/api/v1/mdf` | Create MDF request |
| GET | `/api/v1/mdf/{mdfId}` | Get MDF request details |
| GET | `/api/v1/mdf` | List MDF requests |
| PUT | `/api/v1/mdf/{mdfId}` | Update MDF request |
| POST | `/api/v1/mdf/{mdfId}/submit` | Submit MDF for approval |
| POST | `/api/v1/mdf/{mdfId}/approve` | Approve MDF request |
| POST | `/api/v1/mdf/{mdfId}/reject` | Reject MDF request |
| POST | `/api/v1/mdf/{mdfId}/settle` | Settle MDF payment |
| POST | `/api/v1/performance/calculate` | Calculate partner performance scores |
| GET | `/api/v1/performance/{partnerId}` | Get partner performance scores |
| GET | `/api/v1/performance` | Get performance scores across partners |
| GET | `/api/v1/performance/rankings` | Get partner performance rankings |
| GET | `/api/v1/portals` | List partner portals |
| POST | `/api/v1/portals` | Create partner portal |
| GET | `/api/v1/portals/{portalId}` | Get portal details |
| PUT | `/api/v1/portals/{portalId}` | Update portal configuration |
| GET | `/api/v1/training` | List training records |
| POST | `/api/v1/training/enroll` | Enroll partner user in training |
| GET | `/api/v1/training/{recordId}` | Get training record |
| PUT | `/api/v1/training/{recordId}` | Update training record |
| POST | `/api/v1/training/{recordId}/complete` | Mark training complete |
| GET | `/api/v1/tiers` | List tier programs |
| POST | `/api/v1/tiers` | Create tier program |
| GET | `/api/v1/tiers/{tierId}` | Get tier details |
| PUT | `/api/v1/tiers/{tierId}` | Update tier program |
| POST | `/api/v1/tiers/recalculate` | Recalculate partner tier assignments |

---

## 4. Business Rules

1. A partner profile MUST have a unique partner_code generated using the pattern `P-{YYYY}{MM}-{SEQ}` upon registration.
2. A partner MUST be assigned a valid partner_type_id at creation; the partner type determines eligibility for deal registration, lead distribution, and MDF programs.
3. Partner agreements MUST have a start_date and end_date; an agreement in ACTIVE status MUST have been signed by both the partner and the vendor.
4. Deal registrations MUST validate that the customer does not already have an active deal registered by another partner for the same opportunity; duplicate deals MUST be flagged for manual resolution.
5. Approved deal registrations MUST have a protection period (default 90 days) during which no other partner may register the same deal; expired protections MAY be renewed once.
6. Lead distribution MUST assign leads to partners based on territory matching, partner capacity, and lead score; the system SHOULD use a round-robin algorithm for partners with equal eligibility.
7. A partner MUST accept or decline a distributed lead within 5 business days; unresponded leads MUST be automatically reassigned.
8. MDF requested amounts MUST NOT exceed the partner's allocated MDF budget for the current budget period; the budget MUST be calculated as a percentage of the partner's tier-level MDF allocation.
9. MDF settlements MUST require proof of performance documentation; actual_spend MUST be verified against submitted receipts before payment.
10. Partner performance scores MUST be calculated quarterly using a weighted model across revenue (40%), lead conversion (25%), certifications (15%), customer satisfaction (10%), and training (10%).
11. Partner tier assignments MUST be recalculated annually based on performance scores; a partner qualifying for a higher tier SHOULD be upgraded automatically, while downgrades MUST require manager approval.
12. Training courses marked as mandatory MUST be completed by the partner's certified individuals within 90 days of enrollment; incomplete mandatory training MUST negatively impact the training score.
13. Commission amounts on deal registrations MUST be calculated as approved_value multiplied by commission_percent; commission_percent MUST NOT exceed the maximum defined in the partner agreement.
14. All monetary values (revenue, deal values, MDF amounts, commissions, credit limits) MUST be stored as integers representing cents.
15. All timestamps MUST be stored and transmitted in ISO 8601 format with UTC timezone.
16. Soft delete MUST be used (is_active = 0) for all entities; hard deletes MUST NOT be permitted.
17. Tenant isolation MUST be enforced on every query; no cross-tenant data access is permitted.
18. Partner portal access MUST be restricted to partners with an ACTIVE status and at least one ACTIVE agreement.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package channel.v1;

option go_package = "github.com/fusion/protos/channel/v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service ChannelManagement {
    // Partner Profiles
    rpc RegisterPartner(RegisterPartnerRequest) returns (PartnerResponse);
    rpc GetPartner(GetPartnerRequest) returns (PartnerResponse);
    rpc ListPartners(ListPartnersRequest) returns (ListPartnersResponse);
    rpc UpdatePartner(UpdatePartnerRequest) returns (PartnerResponse);
    rpc ActivatePartner(ActivatePartnerRequest) returns (PartnerResponse);
    rpc DeactivatePartner(DeactivatePartnerRequest) returns (PartnerResponse);

    // Partner Types
    rpc CreatePartnerType(CreatePartnerTypeRequest) returns (PartnerTypeResponse);
    rpc ListPartnerTypes(ListPartnerTypesRequest) returns (ListPartnerTypesResponse);
    rpc UpdatePartnerType(UpdatePartnerTypeRequest) returns (PartnerTypeResponse);

    // Agreements
    rpc CreateAgreement(CreateAgreementRequest) returns (AgreementResponse);
    rpc GetAgreement(GetAgreementRequest) returns (AgreementResponse);
    rpc ListAgreements(ListAgreementsRequest) returns (ListAgreementsResponse);
    rpc UpdateAgreement(UpdateAgreementRequest) returns (AgreementResponse);
    rpc ActivateAgreement(ActivateAgreementRequest) returns (AgreementResponse);
    rpc TerminateAgreement(TerminateAgreementRequest) returns (AgreementResponse);
    rpc RenewAgreement(RenewAgreementRequest) returns (AgreementResponse);

    // Deal Registrations
    rpc SubmitDealRegistration(SubmitDealRegistrationRequest) returns (DealRegistrationResponse);
    rpc GetDealRegistration(GetDealRegistrationRequest) returns (DealRegistrationResponse);
    rpc ListDealRegistrations(ListDealRegistrationsRequest) returns (ListDealRegistrationsResponse);
    rpc UpdateDealRegistration(UpdateDealRegistrationRequest) returns (DealRegistrationResponse);
    rpc ApproveDealRegistration(ApproveDealRegistrationRequest) returns (DealRegistrationResponse);
    rpc RejectDealRegistration(RejectDealRegistrationRequest) returns (DealRegistrationResponse);
    rpc CloseDealRegistration(CloseDealRegistrationRequest) returns (DealRegistrationResponse);

    // Lead Distribution
    rpc DistributeLead(DistributeLeadRequest) returns (LeadDistributionResponse);
    rpc GetLeadDistribution(GetLeadDistributionRequest) returns (LeadDistributionResponse);
    rpc ListLeadDistributions(ListLeadDistributionsRequest) returns (ListLeadDistributionsResponse);
    rpc UpdateLeadDistribution(UpdateLeadDistributionRequest) returns (LeadDistributionResponse);
    rpc AcceptLead(AcceptLeadRequest) returns (LeadDistributionResponse);
    rpc DeclineLead(DeclineLeadRequest) returns (LeadDistributionResponse);
    rpc ConvertLead(ConvertLeadRequest) returns (LeadDistributionResponse);

    // Co-Marketing Funds (MDF)
    rpc CreateMDFRequest(CreateMDFRequestRequest) returns (MDFResponse);
    rpc GetMDFRequest(GetMDFRequestRequest) returns (MDFResponse);
    rpc ListMDFRequests(ListMDFRequestsRequest) returns (ListMDFResponse);
    rpc UpdateMDFRequest(UpdateMDFRequestRequest) returns (MDFResponse);
    rpc ApproveMDFRequest(ApproveMDFRequestRequest) returns (MDFResponse);
    rpc RejectMDFRequest(RejectMDFRequestRequest) returns (MDFResponse);
    rpc SettleMDFRequest(SettleMDFRequestRequest) returns (MDFResponse);

    // Performance Scoring
    rpc CalculatePerformance(CalculatePerformanceRequest) returns (PerformanceScoreResponse);
    rpc GetPartnerPerformance(GetPartnerPerformanceRequest) returns (PerformanceScoreResponse);
    rpc ListPerformances(ListPerformancesRequest) returns (ListPerformancesResponse);
    rpc GetPerformanceRankings(GetPerformanceRankingsRequest) returns (PerformanceRankingsResponse);

    // Partner Portals
    rpc CreatePortal(CreatePortalRequest) returns (PortalResponse);
    rpc GetPortal(GetPortalRequest) returns (PortalResponse);
    rpc ListPortals(ListPortalsRequest) returns (ListPortalsResponse);
    rpc UpdatePortal(UpdatePortalRequest) returns (PortalResponse);

    // Training
    rpc EnrollTraining(EnrollTrainingRequest) returns (TrainingRecordResponse);
    rpc GetTrainingRecord(GetTrainingRecordRequest) returns (TrainingRecordResponse);
    rpc ListTrainingRecords(ListTrainingRecordsRequest) returns (ListTrainingRecordsResponse);
    rpc UpdateTrainingRecord(UpdateTrainingRecordRequest) returns (TrainingRecordResponse);
    rpc CompleteTraining(CompleteTrainingRequest) returns (TrainingRecordResponse);

    // Tier Programs
    rpc CreateTier(CreateTierRequest) returns (TierResponse);
    rpc GetTier(GetTierRequest) returns (TierResponse);
    rpc ListTiers(ListTiersRequest) returns (ListTiersResponse);
    rpc UpdateTier(UpdateTierRequest) returns (TierResponse);
    rpc RecalculateTiers(RecalculateTiersRequest) returns (RecalculateTiersResponse);
}

message PartnerMessage {
    string id = 1;
    string tenant_id = 2;
    string partner_code = 3;
    string company_name = 4;
    string legal_name = 5;
    string tax_id = 6;
    string partner_type_id = 7;
    string tier_id = 8;
    string status = 9;
    string primary_contact_id = 10;
    string address_line1 = 11;
    string city = 12;
    string state_province = 13;
    string postal_code = 14;
    string country = 15;
    string website_url = 16;
    string phone = 17;
    string email = 18;
    string industry = 19;
    int64 annual_revenue = 20;
    int32 employee_count = 21;
    string territories = 22;
    string certifications = 23;
    string competencies = 24;
    string parent_partner_id = 25;
    string bank_name = 26;
    string payment_terms = 27;
    int64 credit_limit = 28;
    string currency_code = 29;
    string notes = 30;
    google.protobuf.Timestamp registered_at = 31;
    google.protobuf.Timestamp activated_at = 32;
    bool is_active = 33;
    google.protobuf.Timestamp created_at = 34;
    google.protobuf.Timestamp updated_at = 35;
    string created_by = 36;
    string updated_by = 37;
    int32 version = 38;
}

message RegisterPartnerRequest {
    string tenant_id = 1;
    string company_name = 2;
    string legal_name = 3;
    string tax_id = 4;
    string partner_type_id = 5;
    string primary_contact_id = 6;
    string address_line1 = 7;
    string city = 8;
    string state_province = 9;
    string postal_code = 10;
    string country = 11;
    string website_url = 12;
    string phone = 13;
    string email = 14;
    string industry = 15;
    string territories = 16;
    string notes = 17;
}

message GetPartnerRequest {
    string tenant_id = 1;
    string partner_id = 2;
}

message ListPartnersRequest {
    string tenant_id = 1;
    string status = 2;
    string partner_type_id = 3;
    string tier_id = 4;
    string country = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListPartnersResponse {
    repeated PartnerMessage partners = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdatePartnerRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string company_name = 3;
    string legal_name = 4;
    string address_line1 = 5;
    string city = 6;
    string state_province = 7;
    string postal_code = 8;
    string country = 9;
    string website_url = 10;
    string phone = 11;
    string email = 12;
    string industry = 13;
    string territories = 14;
    string notes = 15;
    int32 version = 16;
}

message ActivatePartnerRequest {
    string tenant_id = 1;
    string partner_id = 2;
}

message DeactivatePartnerRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string reason = 3;
}

message PartnerResponse {
    PartnerMessage partner = 1;
}

message PartnerTypeMessage {
    string id = 1;
    string tenant_id = 2;
    string type_code = 3;
    string name = 4;
    string description = 5;
    int32 default_discount_percent = 6;
    bool deal_registration_enabled = 7;
    bool lead_distribution_enabled = 8;
    bool mdf_eligible = 9;
}

message CreatePartnerTypeRequest {
    string tenant_id = 1;
    string type_code = 2;
    string name = 3;
    string description = 4;
    int32 default_discount_percent = 5;
    bool deal_registration_enabled = 6;
    bool lead_distribution_enabled = 7;
    bool mdf_eligible = 8;
}

message PartnerTypeResponse {
    PartnerTypeMessage partner_type = 1;
}

message ListPartnerTypesRequest {
    string tenant_id = 1;
}

message ListPartnerTypesResponse {
    repeated PartnerTypeMessage partner_types = 1;
}

message UpdatePartnerTypeRequest {
    string tenant_id = 1;
    string type_id = 2;
    string name = 3;
    string description = 4;
    int32 default_discount_percent = 5;
    int32 version = 6;
}

message AgreementMessage {
    string id = 1;
    string tenant_id = 2;
    string agreement_number = 3;
    string partner_id = 4;
    string agreement_type = 5;
    string status = 6;
    string start_date = 7;
    string end_date = 8;
    bool auto_renew = 9;
    int32 renewal_period_months = 10;
    int32 discount_percent = 11;
    int32 margin_percent = 12;
    string payment_terms = 13;
    int64 minimum_commitment = 14;
    string territory = 15;
    string exclusivity_type = 16;
    string signed_by_partner = 17;
    string signed_by_vendor = 18;
    google.protobuf.Timestamp signed_at = 19;
    bool is_active = 20;
    google.protobuf.Timestamp created_at = 21;
    google.protobuf.Timestamp updated_at = 22;
    int32 version = 23;
}

message CreateAgreementRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string agreement_type = 3;
    string start_date = 4;
    string end_date = 5;
    bool auto_renew = 6;
    int32 renewal_period_months = 7;
    int32 discount_percent = 8;
    int32 margin_percent = 9;
    string payment_terms = 10;
    int64 minimum_commitment = 11;
    string territory = 12;
    string exclusivity_type = 13;
    string terms_and_conditions = 14;
}

message GetAgreementRequest {
    string tenant_id = 1;
    string agreement_id = 2;
}

message ListAgreementsRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string status = 3;
    string agreement_type = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListAgreementsResponse {
    repeated AgreementMessage agreements = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateAgreementRequest {
    string tenant_id = 1;
    string agreement_id = 2;
    string end_date = 3;
    int32 discount_percent = 4;
    int32 margin_percent = 5;
    string payment_terms = 6;
    int64 minimum_commitment = 7;
    int32 version = 8;
}

message ActivateAgreementRequest {
    string tenant_id = 1;
    string agreement_id = 2;
    string signed_by_partner = 3;
    string signed_by_vendor = 4;
}

message TerminateAgreementRequest {
    string tenant_id = 1;
    string agreement_id = 2;
    string reason = 3;
}

message RenewAgreementRequest {
    string tenant_id = 1;
    string agreement_id = 2;
    string new_end_date = 3;
}

message AgreementResponse {
    AgreementMessage agreement = 1;
}

message DealRegistrationMessage {
    string id = 1;
    string tenant_id = 2;
    string deal_number = 3;
    string partner_id = 4;
    string partner_contact_id = 5;
    string customer_name = 6;
    string customer_contact_name = 7;
    string customer_contact_email = 8;
    string opportunity_name = 9;
    string opportunity_description = 10;
    int64 estimated_value = 11;
    string currency_code = 12;
    string estimated_close_date = 13;
    string product_interest = 14;
    string competitive_situation = 15;
    string status = 16;
    string review_notes = 17;
    string reviewed_by = 18;
    google.protobuf.Timestamp reviewed_at = 19;
    int64 approved_value = 20;
    google.protobuf.Timestamp approved_at = 21;
    google.protobuf.Timestamp expires_at = 22;
    int32 protection_period_days = 23;
    int32 commission_percent = 24;
    int64 commission_amount = 25;
    bool is_active = 26;
    google.protobuf.Timestamp created_at = 27;
    google.protobuf.Timestamp updated_at = 28;
    string created_by = 29;
    string updated_by = 30;
    int32 version = 31;
}

message SubmitDealRegistrationRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string partner_contact_id = 3;
    string customer_name = 4;
    string customer_contact_name = 5;
    string customer_contact_email = 6;
    string opportunity_name = 7;
    string opportunity_description = 8;
    int64 estimated_value = 9;
    string currency_code = 10;
    string estimated_close_date = 11;
    string product_interest = 12;
    string competitive_situation = 13;
}

message GetDealRegistrationRequest {
    string tenant_id = 1;
    string deal_id = 2;
}

message ListDealRegistrationsRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListDealRegistrationsResponse {
    repeated DealRegistrationMessage deals = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateDealRegistrationRequest {
    string tenant_id = 1;
    string deal_id = 2;
    string opportunity_description = 3;
    int64 estimated_value = 4;
    string estimated_close_date = 5;
    string product_interest = 6;
    int32 version = 7;
}

message ApproveDealRegistrationRequest {
    string tenant_id = 1;
    string deal_id = 2;
    int64 approved_value = 3;
    int32 commission_percent = 4;
    string review_notes = 5;
}

message RejectDealRegistrationRequest {
    string tenant_id = 1;
    string deal_id = 2;
    string reason = 3;
}

message CloseDealRegistrationRequest {
    string tenant_id = 1;
    string deal_id = 2;
    string sales_order_id = 3;
    int64 final_value = 4;
}

message DealRegistrationResponse {
    DealRegistrationMessage deal = 1;
}

message LeadDistributionMessage {
    string id = 1;
    string tenant_id = 2;
    string lead_number = 3;
    string partner_id = 4;
    string lead_source = 5;
    string lead_name = 6;
    string lead_company = 7;
    string lead_email = 8;
    string lead_phone = 9;
    string lead_title = 10;
    string description = 11;
    string status = 12;
    google.protobuf.Timestamp assigned_at = 13;
    google.protobuf.Timestamp accepted_at = 14;
    google.protobuf.Timestamp declined_at = 15;
    google.protobuf.Timestamp qualified_at = 16;
    google.protobuf.Timestamp converted_at = 17;
    string converted_opportunity_id = 18;
    int64 conversion_value = 19;
    int32 lead_score = 20;
    string follow_up_date = 21;
    bool is_active = 22;
    google.protobuf.Timestamp created_at = 23;
    google.protobuf.Timestamp updated_at = 24;
    int32 version = 25;
}

message DistributeLeadRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string lead_source = 3;
    string lead_name = 4;
    string lead_company = 5;
    string lead_email = 6;
    string lead_phone = 7;
    string lead_title = 8;
    string lead_industry = 9;
    int64 lead_value_estimate = 10;
    string description = 11;
    int32 lead_score = 12;
}

message GetLeadDistributionRequest {
    string tenant_id = 1;
    string lead_id = 2;
}

message ListLeadDistributionsRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListLeadDistributionsResponse {
    repeated LeadDistributionMessage leads = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateLeadDistributionRequest {
    string tenant_id = 1;
    string lead_id = 2;
    string description = 3;
    string follow_up_date = 4;
    int32 version = 5;
}

message AcceptLeadRequest {
    string tenant_id = 1;
    string lead_id = 2;
}

message DeclineLeadRequest {
    string tenant_id = 1;
    string lead_id = 2;
    string reason = 3;
}

message ConvertLeadRequest {
    string tenant_id = 1;
    string lead_id = 2;
    string opportunity_id = 3;
    int64 conversion_value = 4;
}

message LeadDistributionResponse {
    LeadDistributionMessage lead = 1;
}

message MDFMessage {
    string id = 1;
    string tenant_id = 2;
    string fund_request_number = 3;
    string partner_id = 4;
    string program_name = 5;
    string fund_type = 6;
    string status = 7;
    string activity_description = 8;
    string activity_start_date = 9;
    string activity_end_date = 10;
    int64 total_budget = 11;
    int64 requested_amount = 12;
    int64 approved_amount = 13;
    string currency_code = 14;
    int64 actual_spend = 15;
    int32 leads_generated = 16;
    int64 pipeline_influenced = 17;
    int64 revenue_attributed = 18;
    string reviewed_by = 19;
    google.protobuf.Timestamp reviewed_at = 20;
    int64 paid_amount = 21;
    google.protobuf.Timestamp paid_at = 22;
    string budget_period = 23;
    bool is_active = 24;
    google.protobuf.Timestamp created_at = 25;
    google.protobuf.Timestamp updated_at = 26;
    int32 version = 27;
}

message CreateMDFRequestRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string program_name = 3;
    string fund_type = 4;
    string activity_description = 5;
    string activity_start_date = 6;
    string activity_end_date = 7;
    int64 total_budget = 8;
    int64 requested_amount = 9;
    string currency_code = 10;
    string budget_period = 11;
}

message GetMDFRequestRequest {
    string tenant_id = 1;
    string mdf_id = 2;
}

message ListMDFRequestsRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string status = 3;
    string fund_type = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListMDFResponse {
    repeated MDFMessage requests = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateMDFRequestRequest {
    string tenant_id = 1;
    string mdf_id = 2;
    string activity_description = 3;
    int64 requested_amount = 4;
    int32 version = 5;
}

message ApproveMDFRequestRequest {
    string tenant_id = 1;
    string mdf_id = 2;
    int64 approved_amount = 3;
    string review_notes = 4;
}

message RejectMDFRequestRequest {
    string tenant_id = 1;
    string mdf_id = 2;
    string reason = 3;
}

message SettleMDFRequestRequest {
    string tenant_id = 1;
    string mdf_id = 2;
    int64 actual_spend = 3;
    int64 paid_amount = 4;
    string payment_reference = 5;
}

message MDFResponse {
    MDFMessage mdf = 1;
}

message PerformanceScoreMessage {
    string id = 1;
    string tenant_id = 2;
    string partner_id = 3;
    string scoring_period = 4;
    string scoring_model = 5;
    int32 revenue_score = 6;
    int64 revenue_amount = 7;
    int32 deals_registered = 8;
    int32 deals_won = 9;
    int32 win_rate_percent = 10;
    int32 lead_conversion_score = 11;
    int32 leads_received = 12;
    int32 leads_converted = 13;
    int32 certification_score = 14;
    int32 certified_individuals = 15;
    int32 customer_satisfaction_score = 16;
    int32 average_csat = 17;
    int32 training_score = 18;
    int32 overall_score = 19;
    string tier_qualified = 20;
    google.protobuf.Timestamp calculated_at = 21;
}

message CalculatePerformanceRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string scoring_period = 3;
}

message GetPartnerPerformanceRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string scoring_period = 3;
}

message PerformanceScoreResponse {
    PerformanceScoreMessage score = 1;
}

message ListPerformancesRequest {
    string tenant_id = 1;
    string scoring_period = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListPerformancesResponse {
    repeated PerformanceScoreMessage scores = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetPerformanceRankingsRequest {
    string tenant_id = 1;
    string scoring_period = 2;
    int32 max_results = 3;
}

message PerformanceRankingsResponse {
    repeated PerformanceRankingEntry rankings = 1;
}

message PerformanceRankingEntry {
    string partner_id = 1;
    string partner_name = 2;
    int32 overall_score = 3;
    string tier_qualified = 4;
    int32 rank = 5;
}

message PortalMessage {
    string id = 1;
    string tenant_id = 2;
    string portal_code = 3;
    string name = 4;
    string description = 5;
    string url = 6;
    string theme_config = 7;
    string enabled_modules = 8;
    bool deal_registration_enabled = 9;
    bool lead_management_enabled = 10;
    bool mdf_management_enabled = 11;
    bool training_enabled = 12;
    bool reporting_enabled = 13;
    string default_language = 14;
    bool is_active = 15;
    google.protobuf.Timestamp created_at = 16;
    google.protobuf.Timestamp updated_at = 17;
    int32 version = 18;
}

message CreatePortalRequest {
    string tenant_id = 1;
    string portal_code = 2;
    string name = 3;
    string description = 4;
    string url = 5;
    string theme_config = 6;
    string enabled_modules = 7;
    string default_language = 8;
}

message GetPortalRequest {
    string tenant_id = 1;
    string portal_id = 2;
}

message ListPortalsRequest {
    string tenant_id = 1;
}

message ListPortalsResponse {
    repeated PortalMessage portals = 1;
}

message UpdatePortalRequest {
    string tenant_id = 1;
    string portal_id = 2;
    string name = 3;
    string description = 4;
    string theme_config = 5;
    string enabled_modules = 6;
    int32 version = 7;
}

message PortalResponse {
    PortalMessage portal = 1;
}

message TrainingRecordMessage {
    string id = 1;
    string tenant_id = 2;
    string partner_id = 3;
    string partner_user_id = 4;
    string course_id = 5;
    string course_name = 6;
    string course_type = 7;
    string status = 8;
    int32 score = 9;
    int32 max_score = 10;
    int32 passing_score = 11;
    bool passed = 12;
    int32 attempts = 13;
    string certificate_id = 14;
    google.protobuf.Timestamp completion_date = 15;
    google.protobuf.Timestamp expiration_date = 16;
    bool mandatory = 17;
    bool is_active = 18;
    google.protobuf.Timestamp created_at = 19;
    google.protobuf.Timestamp updated_at = 20;
    int32 version = 21;
}

message EnrollTrainingRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string partner_user_id = 3;
    string course_id = 4;
    string course_name = 5;
    string course_type = 6;
    int32 passing_score = 7;
    bool mandatory = 8;
}

message GetTrainingRecordRequest {
    string tenant_id = 1;
    string record_id = 2;
}

message ListTrainingRecordsRequest {
    string tenant_id = 1;
    string partner_id = 2;
    string partner_user_id = 3;
    string status = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListTrainingRecordsResponse {
    repeated TrainingRecordMessage records = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateTrainingRecordRequest {
    string tenant_id = 1;
    string record_id = 2;
    int32 score = 3;
    int32 version = 4;
}

message CompleteTrainingRequest {
    string tenant_id = 1;
    string record_id = 2;
    int32 score = 3;
    bool passed = 4;
    string certificate_id = 5;
}

message TrainingRecordResponse {
    TrainingRecordMessage record = 1;
}

message TierMessage {
    string id = 1;
    string tenant_id = 2;
    string tier_code = 3;
    string name = 4;
    string description = 5;
    int32 level = 6;
    int64 minimum_revenue = 7;
    int32 minimum_deals = 8;
    int32 minimum_certifications = 9;
    int32 minimum_csat = 10;
    int32 discount_percent = 11;
    int32 mdf_budget_percent = 12;
    int32 lead_priority = 13;
    bool dedicated_support = 14;
    bool executive_sponsor = 15;
    string benefits_description = 16;
    bool is_active = 17;
    google.protobuf.Timestamp created_at = 18;
    google.protobuf.Timestamp updated_at = 19;
    int32 version = 20;
}

message CreateTierRequest {
    string tenant_id = 1;
    string tier_code = 2;
    string name = 3;
    string description = 4;
    int32 level = 5;
    int64 minimum_revenue = 6;
    int32 minimum_deals = 7;
    int32 minimum_certifications = 8;
    int32 minimum_csat = 9;
    int32 discount_percent = 10;
    int32 mdf_budget_percent = 11;
    int32 lead_priority = 12;
    bool dedicated_support = 13;
    bool executive_sponsor = 14;
    string benefits_description = 15;
}

message GetTierRequest {
    string tenant_id = 1;
    string tier_id = 2;
}

message ListTiersRequest {
    string tenant_id = 1;
}

message ListTiersResponse {
    repeated TierMessage tiers = 1;
}

message UpdateTierRequest {
    string tenant_id = 1;
    string tier_id = 2;
    string name = 3;
    int64 minimum_revenue = 4;
    int32 minimum_deals = 5;
    int32 minimum_certifications = 6;
    int32 discount_percent = 7;
    int32 mdf_budget_percent = 8;
    string benefits_description = 9;
    int32 version = 10;
}

message TierResponse {
    TierMessage tier = 1;
}

message RecalculateTiersRequest {
    string tenant_id = 1;
    string scoring_period = 2;
}

message RecalculateTiersResponse {
    repeated TierAssignmentResult results = 1;
}

message TierAssignmentResult {
    string partner_id = 2;
    string previous_tier = 3;
    string new_tier = 4;
    bool changed = 5;
}
```

---

## 6. Inter-Service Integration

### Inbound Dependencies

| Source Service | Integration Type | Purpose |
|---|---|---|
| Sales Automation | REST/gRPC | Sync deal and opportunity data, validate deal registrations against vendor pipeline |
| Commerce | REST | Access product catalog and pricing for deal registration product interest |
| Learning | REST/gRPC | Retrieve course catalog, sync training completion records, manage certifications |
| Identity Service | REST | Partner user authentication, authorization, and role-based portal access |
| Notification Service | Async | Send partner notifications for deal status, lead assignments, MDF updates |
| Finance / General Ledger | REST | MDF payment processing, commission payout, credit limit management |
| Customer Service | REST | Link partner-associated service cases to partner performance scoring |
| Reporting & Analytics | REST | Aggregate partner performance data for executive dashboards |

### Outbound Events Consumed By

| Target Service | Event | Purpose |
|---|---|---|
| Sales Automation | DealSubmitted, DealApproved, DealClosed | Create/update vendor-side opportunity records for partner deals |
| Sales Automation | LeadDistributed, LeadConverted | Track lead pipeline and conversion metrics |
| Commerce | DealApproved | Provision partner pricing and discounts in the commerce catalog |
| Learning | PartnerTrainingCompleted | Update certification records and trigger recertification schedules |
| Finance / General Ledger | MDFApproved, MDFSettled, CommissionEarned | Post financial transactions for MDF payments and partner commissions |
| Notification Service | PartnerRegistered, DealSubmitted, LeadDistributed, MDFRequested | Deliver partner-facing and internal notifications |
| Reporting & Analytics | PartnerScored, TierRecalculated | Feed performance data into executive analytics dashboards |

---

## 7. Events

| Event | Payload | Description |
|---|---|---|
| `PartnerRegistered` | `{ partner_id, partner_code, tenant_id, company_name, partner_type_id, country, registered_at }` | Emitted when a new partner is registered in the system |
| `PartnerActivated` | `{ partner_id, tenant_id, partner_code, company_name, tier_id, activated_at }` | Emitted when a partner profile is activated |
| `PartnerDeactivated` | `{ partner_id, tenant_id, partner_code, deactivation_reason, deactivated_at }` | Emitted when a partner is deactivated |
| `AgreementActivated` | `{ agreement_id, tenant_id, partner_id, agreement_number, agreement_type, start_date, end_date }` | Emitted when a partner agreement goes into effect |
| `AgreementTerminated` | `{ agreement_id, tenant_id, partner_id, termination_reason, terminated_at }` | Emitted when a partner agreement is terminated |
| `DealSubmitted` | `{ deal_id, deal_number, tenant_id, partner_id, customer_name, opportunity_name, estimated_value, currency_code }` | Emitted when a partner submits a deal registration |
| `DealApproved` | `{ deal_id, tenant_id, partner_id, approved_value, commission_percent, commission_amount, protection_period_days, expires_at }` | Emitted when a deal registration is approved |
| `DealRejected` | `{ deal_id, tenant_id, partner_id, reason, reviewed_by }` | Emitted when a deal registration is rejected |
| `DealClosed` | `{ deal_id, tenant_id, partner_id, sales_order_id, final_value, commission_amount }` | Emitted when a registered deal is closed as won |
| `LeadDistributed` | `{ lead_id, lead_number, tenant_id, partner_id, lead_source, lead_name, lead_score, assigned_at }` | Emitted when a lead is distributed to a partner |
| `LeadAccepted` | `{ lead_id, tenant_id, partner_id, accepted_at }` | Emitted when a partner accepts a distributed lead |
| `LeadDeclined` | `{ lead_id, tenant_id, partner_id, decline_reason }` | Emitted when a partner declines a lead |
| `LeadConverted` | `{ lead_id, tenant_id, partner_id, converted_opportunity_id, conversion_value }` | Emitted when a distributed lead is converted to an opportunity |
| `MDFRequested` | `{ mdf_id, fund_request_number, tenant_id, partner_id, program_name, requested_amount, currency_code }` | Emitted when a partner submits an MDF request |
| `MDFApproved` | `{ mdf_id, tenant_id, partner_id, approved_amount, reviewed_by }` | Emitted when an MDF request is approved |
| `MDFSettled` | `{ mdf_id, tenant_id, partner_id, paid_amount, payment_reference }` | Emitted when an MDF payment is settled |
| `PartnerScored` | `{ partner_id, tenant_id, scoring_period, overall_score, tier_qualified, revenue_amount, deals_won, win_rate_percent }` | Emitted when a partner performance score is calculated |
| `PartnerTierChanged` | `{ partner_id, tenant_id, previous_tier, new_tier, scoring_period }` | Emitted when a partner's tier assignment changes |
| `TrainingCompleted` | `{ record_id, tenant_id, partner_id, partner_user_id, course_id, course_name, passed, score }` | Emitted when a partner user completes a training course |
