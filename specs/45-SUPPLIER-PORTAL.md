# 45 - Supplier Portal & Sourcing Service Specification

## 1. Domain Overview

Supplier Portal & Sourcing provides supplier self-registration and qualification management, collaborative sourcing events (RFI/RFQ/RFP/Auction), supplier bid submission and evaluation, performance scorecards with weighted metrics, supplier-managed catalog items (hosted and punchout), and supplier self-service invoice submission. Integrates with AP for supplier record creation and invoice processing, Procurement for sourcing and category management, Inventory for catalog item availability, GL for spend analysis accounting, and Workflow for approval routing.

**Bounded Context:** Supplier Collaboration, Sourcing & Procurement Portal
**Service Name:** `supplier-service`
**Database:** `data/supplier.db`
**HTTP Port:** 8072 | **gRPC Port:** 9072

---

## 2. Database Schema

### 2.1 Supplier Portal Users
```sql
CREATE TABLE supplier_portal_users (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,                   -- FK to supplier in AP-service
    username TEXT NOT NULL,
    email TEXT NOT NULL,
    display_name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'VIEWER'
        CHECK(role IN ('ADMIN','EDITOR','VIEWER')),
    is_active INTEGER NOT NULL DEFAULT 1,
    last_login TEXT,
    invitation_token TEXT,
    invitation_expiry TEXT,
    invited_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, username),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_portal_users_tenant_supplier ON supplier_portal_users(tenant_id, supplier_id);
CREATE INDEX idx_portal_users_tenant_active ON supplier_portal_users(tenant_id, is_active);
```

### 2.2 Supplier Registrations
```sql
CREATE TABLE supplier_registrations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    company_name TEXT NOT NULL,
    tax_id TEXT NOT NULL,
    registration_type TEXT NOT NULL DEFAULT 'NEW'
        CHECK(registration_type IN ('NEW','RENEWAL','UPDATE')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','UNDER_REVIEW','APPROVED','REJECTED')),
    business_type TEXT NOT NULL
        CHECK(business_type IN ('CORPORATION','LLC','PARTNERSHIP','SOLE_PROP')),
    year_established INTEGER,
    annual_revenue_cents INTEGER,
    employee_count INTEGER,
    certifications TEXT,                         -- JSON: array of certification objects
    insurance_info TEXT,                         -- JSON: insurance details
    submitted_at TEXT,
    reviewed_by TEXT,
    reviewed_at TEXT,
    review_notes TEXT,
    rejection_reason TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, tax_id)
);

CREATE INDEX idx_registrations_tenant_status ON supplier_registrations(tenant_id, status);
CREATE INDEX idx_registrations_tenant_type ON supplier_registrations(tenant_id, registration_type);
CREATE INDEX idx_registrations_tenant_submitted ON supplier_registrations(tenant_id, submitted_at);
```

### 2.3 Supplier Registration Contacts
```sql
CREATE TABLE supplier_registration_contacts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    registration_id TEXT NOT NULL,
    contact_type TEXT NOT NULL
        CHECK(contact_type IN ('PRIMARY','SALES','BILLING','TECHNICAL')),
    name TEXT NOT NULL,
    title TEXT,
    email TEXT NOT NULL,
    phone TEXT,
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (registration_id) REFERENCES supplier_registrations(id) ON DELETE CASCADE
);

CREATE INDEX idx_reg_contacts_tenant_registration ON supplier_registration_contacts(tenant_id, registration_id);
```

### 2.4 Supplier Registration Addresses
```sql
CREATE TABLE supplier_registration_addresses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    registration_id TEXT NOT NULL,
    address_type TEXT NOT NULL
        CHECK(address_type IN ('MAIN','WAREHOUSE','BILLING','SHIPPING')),
    address_line1 TEXT NOT NULL,
    address_line2 TEXT,
    city TEXT NOT NULL,
    state_province TEXT NOT NULL,
    postal_code TEXT NOT NULL,
    country_code TEXT NOT NULL,                  -- ISO 3166-1 alpha-2
    is_primary INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (registration_id) REFERENCES supplier_registrations(id) ON DELETE CASCADE
);

CREATE INDEX idx_reg_addresses_tenant_registration ON supplier_registration_addresses(tenant_id, registration_id);
CREATE INDEX idx_reg_addresses_tenant_country ON supplier_registration_addresses(tenant_id, country_code);
```

### 2.5 Supplier Qualifications
```sql
CREATE TABLE supplier_qualifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    qualification_type TEXT NOT NULL
        CHECK(qualification_type IN ('FINANCIAL','TECHNICAL','QUALITY','COMPLIANCE','SUSTAINABILITY')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','PASSED','FAILED','EXPIRED')),
    questionnaire_data TEXT,                     -- JSON: assessment questionnaire
    score REAL,
    assessor_id TEXT,
    assessment_date TEXT,
    expiry_date TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_qualifications_tenant_supplier ON supplier_qualifications(tenant_id, supplier_id);
CREATE INDEX idx_qualifications_tenant_status ON supplier_qualifications(tenant_id, status);
CREATE INDEX idx_qualifications_tenant_type ON supplier_qualifications(tenant_id, qualification_type);
CREATE INDEX idx_qualifications_tenant_expiry ON supplier_qualifications(tenant_id, expiry_date);
```

### 2.6 Supplier Sourcing Events
```sql
CREATE TABLE supplier_sourcing_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_number TEXT NOT NULL,
    event_type TEXT NOT NULL
        CHECK(event_type IN ('RFI','RFQ','RFP','AUCTION')),
    title TEXT NOT NULL,
    description TEXT,
    category_id TEXT,
    buyer_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','CLOSED','EVALUATION','AWARDED','CANCELLED')),
    publish_date TEXT,
    close_date TEXT,
    award_date TEXT,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    is_sealed INTEGER NOT NULL DEFAULT 0,
    allow_reverse_auction INTEGER NOT NULL DEFAULT 0,
    evaluation_criteria TEXT,                    -- JSON: weighted criteria
    terms_and_conditions TEXT,
    attachments TEXT,                            -- JSON: array of document references

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, event_number)
);

CREATE INDEX idx_sourcing_events_tenant_status ON supplier_sourcing_events(tenant_id, status);
CREATE INDEX idx_sourcing_events_tenant_type ON supplier_sourcing_events(tenant_id, event_type);
CREATE INDEX idx_sourcing_events_tenant_buyer ON supplier_sourcing_events(tenant_id, buyer_id);
CREATE INDEX idx_sourcing_events_tenant_dates ON supplier_sourcing_events(tenant_id, publish_date, close_date);
```

### 2.7 Supplier Sourcing Lines
```sql
CREATE TABLE supplier_sourcing_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT,                                -- nullable: may be generic description
    item_description TEXT NOT NULL,
    quantity DECIMAL(18,4) NOT NULL,
    uom TEXT NOT NULL,
    delivery_date_required TEXT,
    specifications TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (event_id) REFERENCES supplier_sourcing_events(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id, line_number)
);

CREATE INDEX idx_sourcing_lines_tenant_event ON supplier_sourcing_lines(tenant_id, event_id);
```

### 2.8 Supplier Bids
```sql
CREATE TABLE supplier_bids (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    bid_number TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','SUBMITTED','ACCEPTED','REJECTED','WITHDRAWN')),
    total_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    delivery_terms TEXT,
    valid_until TEXT,
    submitted_at TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, bid_number)
);

CREATE INDEX idx_bids_tenant_event ON supplier_bids(tenant_id, event_id);
CREATE INDEX idx_bids_tenant_supplier ON supplier_bids(tenant_id, supplier_id);
CREATE INDEX idx_bids_tenant_status ON supplier_bids(tenant_id, status);
```

### 2.9 Supplier Bid Lines
```sql
CREATE TABLE supplier_bid_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    bid_id TEXT NOT NULL,
    sourcing_line_id TEXT NOT NULL,
    unit_price_cents INTEGER NOT NULL DEFAULT 0,
    quantity DECIMAL(18,4) NOT NULL,
    total_price_cents INTEGER NOT NULL DEFAULT 0,
    lead_time_days INTEGER,
    delivery_date TEXT,
    specifications_compliance TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (bid_id) REFERENCES supplier_bids(id) ON DELETE CASCADE
);

CREATE INDEX idx_bid_lines_tenant_bid ON supplier_bid_lines(tenant_id, bid_id);
CREATE INDEX idx_bid_lines_tenant_sourcing_line ON supplier_bid_lines(tenant_id, sourcing_line_id);
```

### 2.10 Supplier Scorecards
```sql
CREATE TABLE supplier_scorecards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    period TEXT NOT NULL,                        -- e.g. "2025-Q1", "2025-01"
    quality_score REAL NOT NULL DEFAULT 0,
    delivery_score REAL NOT NULL DEFAULT 0,
    cost_score REAL NOT NULL DEFAULT 0,
    responsiveness_score REAL NOT NULL DEFAULT 0,
    overall_score REAL NOT NULL DEFAULT 0,
    rating TEXT NOT NULL DEFAULT 'CONDITIONAL'
        CHECK(rating IN ('PREFERRED','APPROVED','CONDITIONAL','DISAPPROVED')),
    on_time_delivery_percent REAL,
    defect_rate_ppm REAL,
    invoice_accuracy_percent REAL,
    total_spend_cents INTEGER NOT NULL DEFAULT 0,
    review_notes TEXT,
    reviewed_by TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, period)
);

CREATE INDEX idx_scorecards_tenant_supplier ON supplier_scorecards(tenant_id, supplier_id);
CREATE INDEX idx_scorecards_tenant_rating ON supplier_scorecards(tenant_id, rating);
CREATE INDEX idx_scorecards_tenant_period ON supplier_scorecards(tenant_id, period);
```

### 2.11 Supplier Catalog Items
```sql
CREATE TABLE supplier_catalog_items (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    item_id TEXT,                                -- nullable for external/punchout items
    supplier_sku TEXT NOT NULL,
    item_name TEXT NOT NULL,
    description TEXT,
    category_id TEXT,
    unit_price_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    uom TEXT NOT NULL,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    minimum_order_quantity INTEGER NOT NULL DEFAULT 1,
    is_contract_pricing INTEGER NOT NULL DEFAULT 0,
    contract_id TEXT,
    punchout_url TEXT,
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, supplier_sku)
);

CREATE INDEX idx_catalog_tenant_supplier ON supplier_catalog_items(tenant_id, supplier_id);
CREATE INDEX idx_catalog_tenant_category ON supplier_catalog_items(tenant_id, category_id);
CREATE INDEX idx_catalog_tenant_active ON supplier_catalog_items(tenant_id, is_active);
CREATE INDEX idx_catalog_tenant_item ON supplier_catalog_items(tenant_id, item_id);
```

### 2.12 Supplier Invoice Submissions
```sql
CREATE TABLE supplier_invoice_submissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    po_number TEXT NOT NULL,
    invoice_number TEXT NOT NULL,
    invoice_date TEXT NOT NULL,
    invoice_amount_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'SUBMITTED'
        CHECK(status IN ('SUBMITTED','UNDER_REVIEW','APPROVED','REJECTED')),
    po_matched INTEGER NOT NULL DEFAULT 0,
    rejection_reason TEXT,
    ap_invoice_id TEXT,                          -- link to AP-service invoice record
    document_id TEXT,                            -- FK to document-service
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, supplier_id, invoice_number)
);

CREATE INDEX idx_invoice_sub_tenant_supplier ON supplier_invoice_submissions(tenant_id, supplier_id);
CREATE INDEX idx_invoice_sub_tenant_status ON supplier_invoice_submissions(tenant_id, status);
CREATE INDEX idx_invoice_sub_tenant_po ON supplier_invoice_submissions(tenant_id, po_number);
CREATE INDEX idx_invoice_sub_tenant_dates ON supplier_invoice_submissions(tenant_id, invoice_date);
```

---

## 3. REST API Endpoints

```
# Supplier Portal Auth
POST          /api/v1/supplier/auth/login
POST          /api/v1/supplier/auth/register
POST          /api/v1/supplier/auth/forgot-password
GET           /api/v1/supplier/auth/profile
PUT           /api/v1/supplier/auth/profile

# Registration
GET/POST      /api/v1/supplier/registrations               Permission: supplier.registrations.manage
GET/PUT       /api/v1/supplier/registrations/{id}
POST          /api/v1/supplier/registrations/{id}/approve
POST          /api/v1/supplier/registrations/{id}/reject
GET/POST      /api/v1/supplier/registrations/{id}/contacts
GET/POST      /api/v1/supplier/registrations/{id}/addresses

# Qualifications
GET/POST      /api/v1/supplier/qualifications
GET/PUT       /api/v1/supplier/qualifications/{id}
POST          /api/v1/supplier/qualifications/{id}/assess   Permission: supplier.qualifications.assess

# Sourcing Events
GET/POST      /api/v1/supplier/sourcing/events              Permission: supplier.sourcing.manage
GET/PUT       /api/v1/supplier/sourcing/events/{id}
POST          /api/v1/supplier/sourcing/events/{id}/publish
POST          /api/v1/supplier/sourcing/events/{id}/close
POST          /api/v1/supplier/sourcing/events/{id}/award
GET/POST      /api/v1/supplier/sourcing/events/{id}/lines

# Bids
GET           /api/v1/supplier/sourcing/events/{eid}/bids
POST          /api/v1/supplier/bids                         Permission: supplier.bids.submit
GET/PUT       /api/v1/supplier/bids/{id}
POST          /api/v1/supplier/bids/{id}/submit
POST          /api/v1/supplier/bids/{id}/withdraw

# Scorecards
GET           /api/v1/supplier/scorecards
GET           /api/v1/supplier/scorecards/{supplier_id}
POST          /api/v1/supplier/scorecards/calculate         Permission: supplier.scorecards.manage

# Catalog
GET/POST      /api/v1/supplier/catalog                      Permission: supplier.catalog.manage
GET/PUT       /api/v1/supplier/catalog/{id}
POST          /api/v1/supplier/catalog/{id}/activate
POST          /api/v1/supplier/catalog/{id}/deactivate
POST          /api/v1/supplier/catalog/punchout-request
POST          /api/v1/supplier/catalog/punchout-return

# Invoice Submission (Supplier Self-Service)
GET/POST      /api/v1/supplier/invoices
GET           /api/v1/supplier/invoices/{id}
GET           /api/v1/supplier/invoices/{id}/status

# Reports
GET           /api/v1/supplier/reports/supplier-performance
GET           /api/v1/supplier/reports/spend-analysis
GET           /api/v1/supplier/reports/sourcing-summary
GET           /api/v1/supplier/reports/diversity-compliance
```

---

## 4. Business Rules

### 4.1 Supplier Portal Authentication
```
Supplier portal users are separate from internal users with limited scope:
  1. Portal users authenticate via dedicated supplier auth endpoints
  2. Roles: ADMIN (full supplier management), EDITOR (bid/catalog), VIEWER (read-only)
  3. Invitation tokens expire after 7 days
  4. Each supplier can have multiple portal users
  5. Last login timestamp updated on every authentication
```

### 4.2 Supplier Registration
- Registration approval creates supplier record in AP-service via gRPC
- New registrations default to PENDING status
- UNDER_REVIEW status assigned when a reviewer is assigned
- Rejection requires a reason stored in rejection_reason
- Renewal/Update registrations reference existing supplier record
- All registration contacts must include at least one PRIMARY contact
- All registration addresses must include at least one MAIN address

### 4.3 Supplier Qualifications
- Qualifications must pass all required assessment types before supplier is fully approved
- Each qualification type (FINANCIAL, TECHNICAL, QUALITY, COMPLIANCE, SUSTAINABILITY) is assessed independently
- Score threshold: PASSED requires score >= 70.0
- Expired qualifications must be re-assessed before supplier can participate in sourcing
- Assessment questionnaires are stored as JSON for flexible evaluation criteria

### 4.4 Sourcing Events
- Event number auto-generated: SRE-{YYYY}-{NNNNN}
- Sealed bids are not visible to buyers until close date
- Reverse auctions allow real-time price reduction by suppliers
- Published events cannot be modified (only cancelled)
- Close date must be after publish date
- Award date must be on or after close date
- Evaluation criteria stored as JSON with weighted scoring

### 4.5 Bid Submission
- Bids can only be submitted against PUBLISHED sourcing events
- Submitted bids cannot be modified (only withdrawn)
- Withdrawn bids can be resubmitted as new bid
- Bid total amount is calculated from bid line items
- Sealed bid amounts hidden from all parties until event close
- Reverse auction bids must be lower than current lowest bid

### 4.6 Supplier Scorecards
- Scorecard calculations use weighted criteria:
  - Quality: 30%
  - Delivery: 30%
  - Cost: 25%
  - Responsiveness: 15%
- Rating thresholds:
  - PREFERRED: overall_score >= 90
  - APPROVED: overall_score >= 75
  - CONDITIONAL: overall_score >= 60
  - DISAPPROVED: overall_score < 60
- Scorecards are calculated per period (monthly or quarterly)
- Historical scorecards retained for trend analysis

### 4.7 Supplier Catalog
- Supplier catalog pricing can reference contract pricing or market pricing
- Punchout catalog uses OCI/cXML protocol for external catalog integration
- Catalog items require effective_from date
- Expired items (past effective_to) are automatically deactivated
- Minimum order quantity enforced during requisition
- Contract pricing items reference valid contract_id

### 4.8 Supplier Invoice Submission
- Supplier invoice submissions are matched against POs before AP processing
- PO matching validates: PO number exists, line items match, amounts within tolerance
- Matched invoices forwarded to AP-service via gRPC for payment processing
- Rejected invoices require supplier to correct and resubmit
- Invoice number must be unique per supplier

### 4.9 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `supplier.registration.submitted` | New registration submitted | Workflow |
| `supplier.registration.approved` | Registration approved | AP-service, Workflow |
| `supplier.sourcing.published` | Sourcing event published | Notification |
| `supplier.bid.submitted` | Bid submitted by supplier | Notification, Sourcing |
| `supplier.sourcing.awarded` | Sourcing event awarded | Procurement, GL |
| `supplier.scorecard.updated` | Scorecard recalculated | Procurement |
| `supplier.invoice.submitted` | Invoice submitted by supplier | AP-service |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.supplier.v1;

service SupplierPortalService {
    rpc GetSupplierProfile(GetSupplierProfileRequest) returns (GetSupplierProfileResponse);
    rpc SubmitBid(SubmitBidRequest) returns (SubmitBidResponse);
    rpc GetScorecard(GetScorecardRequest) returns (GetScorecardResponse);
    rpc ValidateCatalogItem(ValidateCatalogItemRequest) returns (ValidateCatalogItemResponse);
}

// Entity messages

message SupplierPortalUser {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string username = 4;
    string email = 5;
    string display_name = 6;
    string role = 7;
    string last_login = 8;
    string invitation_token = 9;
    string invitation_expiry = 10;
    string invited_by = 11;
    string created_at = 12;
    string updated_at = 13;
}

message SupplierRegistration {
    string id = 1;
    string tenant_id = 2;
    string company_name = 3;
    string tax_id = 4;
    string registration_type = 5;
    string status = 6;
    string business_type = 7;
    int32 year_established = 8;
    int64 annual_revenue_cents = 9;
    int32 employee_count = 10;
    string certifications = 11;
    string insurance_info = 12;
    string submitted_at = 13;
    string reviewed_by = 14;
    string reviewed_at = 15;
    string review_notes = 16;
    string rejection_reason = 17;
    string created_at = 18;
    string updated_at = 19;
}

message SupplierQualification {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string qualification_type = 4;
    string status = 5;
    string questionnaire_data = 6;
    double score = 7;
    string assessor_id = 8;
    string assessment_date = 9;
    string expiry_date = 10;
    string notes = 11;
    string created_at = 12;
    string updated_at = 13;
}

message SupplierSourcingEvent {
    string id = 1;
    string tenant_id = 2;
    string event_number = 3;
    string event_type = 4;
    string title = 5;
    string description = 6;
    string category_id = 7;
    string buyer_id = 8;
    string status = 9;
    string publish_date = 10;
    string close_date = 11;
    string award_date = 12;
    string currency_code = 13;
    bool is_sealed = 14;
    bool allow_reverse_auction = 15;
    string evaluation_criteria = 16;
    string terms_and_conditions = 17;
    string attachments = 18;
    string created_at = 19;
    string updated_at = 20;
}

message SupplierBid {
    string id = 1;
    string tenant_id = 2;
    string event_id = 3;
    string supplier_id = 4;
    string bid_number = 5;
    string status = 6;
    int64 total_amount_cents = 7;
    string currency_code = 8;
    string delivery_terms = 9;
    string valid_until = 10;
    string submitted_at = 11;
    string notes = 12;
    string created_at = 13;
    string updated_at = 14;
}

message SupplierScorecard {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string period = 4;
    double quality_score = 5;
    double delivery_score = 6;
    double cost_score = 7;
    double responsiveness_score = 8;
    double overall_score = 9;
    string rating = 10;
    double on_time_delivery_percent = 11;
    double defect_rate_ppm = 12;
    double invoice_accuracy_percent = 13;
    int64 total_spend_cents = 14;
    string review_notes = 15;
    string reviewed_by = 16;
    string created_at = 17;
    string updated_at = 18;
}

message SupplierCatalogItem {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string item_id = 4;
    string supplier_sku = 5;
    string item_name = 6;
    string description = 7;
    string category_id = 8;
    int64 unit_price_cents = 9;
    string currency_code = 10;
    string uom = 11;
    int32 lead_time_days = 12;
    int32 minimum_order_quantity = 13;
    bool is_contract_pricing = 14;
    string contract_id = 15;
    string punchout_url = 16;
    string effective_from = 17;
    string effective_to = 18;
    string created_at = 19;
    string updated_at = 20;
}

message SupplierInvoiceSubmission {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string po_number = 4;
    string invoice_number = 5;
    string invoice_date = 6;
    int64 invoice_amount_cents = 7;
    string currency_code = 8;
    int64 tax_amount_cents = 9;
    string status = 10;
    bool po_matched = 11;
    string rejection_reason = 12;
    string ap_invoice_id = 13;
    string document_id = 14;
    string notes = 15;
    string created_at = 16;
    string updated_at = 17;
}

// Request/Response messages

message GetSupplierProfileRequest {
    string tenant_id = 1;
    string supplier_id = 2;
}

message GetSupplierProfileResponse {
    SupplierPortalUser user = 1;
    SupplierScorecard scorecard = 2;
}

message SubmitBidRequest {
    string tenant_id = 1;
    string event_id = 2;
    string supplier_id = 3;
    int64 total_amount_cents = 4;
    string currency_code = 5;
    string delivery_terms = 6;
    string valid_until = 7;
    string notes = 8;
}

message SubmitBidResponse {
    SupplierBid data = 1;
}

message GetScorecardRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string period = 3;
}

message GetScorecardResponse {
    SupplierScorecard data = 1;
}

message ValidateCatalogItemRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string supplier_sku = 3;
    string effective_date = 4;
}

message ValidateCatalogItemResponse {
    bool valid = 1;
    SupplierCatalogItem catalog_item = 2;
    string validation_message = 3;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **AP:** Supplier master records, payment terms, supplier status
- **Procurement:** Categories, items, requisitions, purchase orders
- **Inventory:** Item availability, warehouse locations for catalog fulfillment
- **Document:** Attachment storage for registrations, bids, and invoices
- **Auth:** User authentication context for portal access

### 6.2 Data Published To
- **AP:** New supplier records from approved registrations, supplier invoices for payment processing
- **Procurement:** Sourcing event awards create purchase contracts, catalog items for requisition
- **GL:** Spend analysis data, sourcing award journal entries
- **Inventory:** Catalog item updates for stock availability
- **Workflow:** Registration approval routing, bid evaluation workflows
- **Notification:** Supplier-facing notifications for events, bid status changes

---

## 7. Migrations

1. V001: `supplier_portal_users`
2. V002: `supplier_registrations`
3. V003: `supplier_registration_contacts`
4. V004: `supplier_registration_addresses`
5. V005: `supplier_qualifications`
6. V006: `supplier_sourcing_events`
7. V007: `supplier_sourcing_lines`
8. V008: `supplier_bids`
9. V009: `supplier_bid_lines`
10. V010: `supplier_scorecards`
11. V011: `supplier_catalog_items`
12. V012: `supplier_invoice_submissions`
13. V013: Triggers for `updated_at`
