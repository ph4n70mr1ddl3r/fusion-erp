# 39 - Global Trade Management (GTM) Service Specification

## 1. Domain Overview

Global Trade Management provides international trade compliance, customs declaration management, restricted party screening, import/export license tracking, tariff classification, Free Trade Agreement (FTA) preference management, landed cost simulation, and trade compliance holds. Integrates with OM for export orders, Procurement for import purchases, INV for cross-border inventory movements, FA for customs duty capitalization, and GL for duty/tax accounting entries.

**Bounded Context:** International Trade, Customs & Compliance
**Service Name:** `gtm-service`
**Database:** `data/gtm.db`
**HTTP Port:** 8066 | **gRPC Port:** 9066

---

## 2. Database Schema

### 2.1 Trade Preferences
```sql
CREATE TABLE gtm_trade_preferences (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    agreement_name TEXT NOT NULL,
    agreement_code TEXT NOT NULL
        CHECK(agreement_code IN ('USMCA','EU_JAPAN','RCEP','CPTPP','EU_UK','ASEAN_AUSTRALIA_NEW_ZEALAND','EU_CANADA','KORUS','JAPAN_UK','OTHER')),
    participating_countries TEXT NOT NULL,      -- JSON array of ISO country codes
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    rules_of_origin TEXT NOT NULL,              -- JSON: origin qualification rules
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, agreement_code)
);
```

### 2.2 Tariff Schedules
```sql
CREATE TABLE gtm_tariff_schedules (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    hs_code TEXT NOT NULL,                     -- 10-digit Harmonized System code
    hs_chapter TEXT NOT NULL,                  -- First 2 digits of HS code
    description TEXT NOT NULL,
    general_rate_percent REAL NOT NULL DEFAULT 0,
    preferential_rates TEXT,                   -- JSON: {"USMCA": 0, "EU_JAPAN": 2.5, ...}
    unit_of_measure TEXT,
    statistical_unit TEXT,
    is_duty_free INTEGER NOT NULL DEFAULT 0,
    is_anti_dumping INTEGER NOT NULL DEFAULT 0,
    country_of_origin TEXT,                    -- ISO country code

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, hs_code, country_of_origin)
);

CREATE INDEX idx_tariffs_tenant_chapter ON gtm_tariff_schedules(tenant_id, hs_chapter);
CREATE INDEX idx_tariffs_tenant_duty_free ON gtm_tariff_schedules(tenant_id, is_duty_free);
```

### 2.3 Product Classifications
```sql
CREATE TABLE gtm_product_classifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_id TEXT NOT NULL,
    hs_code TEXT NOT NULL,
    eccn TEXT,                                 -- Export Control Classification Number
    country_of_origin TEXT NOT NULL,            -- ISO country code
    preferential_origin TEXT,                  -- ISO country code for FTA origin
    binding_ruling_number TEXT,
    ruling_expiry TEXT,
    classification_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(classification_status IN ('PENDING','CONFIRMED','REVIEW')),
    classified_by TEXT,
    classified_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, item_id, country_of_origin)
);

CREATE INDEX idx_classifications_tenant_item ON gtm_product_classifications(tenant_id, item_id);
CREATE INDEX idx_classifications_tenant_status ON gtm_product_classifications(tenant_id, classification_status);
```

### 2.4 Restricted Party Entries
```sql
CREATE TABLE gtm_restricted_party_entries (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    entity_name TEXT NOT NULL,
    entity_type TEXT NOT NULL
        CHECK(entity_type IN ('PERSON','ORGANIZATION','VESSEL')),
    country_code TEXT NOT NULL,                -- ISO country code
    address TEXT,                              -- Full address text
    list_source TEXT NOT NULL
        CHECK(list_source IN ('SDN','ENTITY','DENIED','UNVERIFIED','CONSOLIDATED')),
    list_entry_id TEXT NOT NULL,               -- External list reference ID
    aliases TEXT,                              -- JSON array of known aliases
    is_active INTEGER NOT NULL DEFAULT 1,
    last_verified TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_restricted_tenant_name ON gtm_restricted_party_entries(tenant_id, entity_name);
CREATE INDEX idx_restricted_tenant_country ON gtm_restricted_party_entries(tenant_id, country_code);
CREATE INDEX idx_restricted_tenant_source ON gtm_restricted_party_entries(tenant_id, list_source);
```

### 2.5 Screening Results
```sql
CREATE TABLE gtm_screening_results (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    screening_type TEXT NOT NULL
        CHECK(screening_type IN ('EXPORT','IMPORT','SANCTIONS')),
    screened_entity_name TEXT NOT NULL,
    screened_country TEXT NOT NULL,            -- ISO country code
    match_type TEXT NOT NULL
        CHECK(match_type IN ('EXACT','PARTIAL','FUZZY')),
    match_score REAL NOT NULL DEFAULT 0,       -- 0.0 - 1.0
    matched_entry_id TEXT,                     -- FK to gtm_restricted_party_entries
    status TEXT NOT NULL DEFAULT 'NO_MATCH'
        CHECK(status IN ('NO_MATCH','POTENTIAL_MATCH','CONFIRMED_MATCH','FALSE_POSITIVE')),
    reviewed_by TEXT,
    review_notes TEXT,
    transaction_type TEXT,                     -- e.g. SALES_ORDER, PURCHASE_ORDER
    transaction_id TEXT,                       -- FK to the originating transaction

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (matched_entry_id) REFERENCES gtm_restricted_party_entries(id)
);

CREATE INDEX idx_screening_tenant_status ON gtm_screening_results(tenant_id, status);
CREATE INDEX idx_screening_tenant_transaction ON gtm_screening_results(tenant_id, transaction_type, transaction_id);
CREATE INDEX idx_screening_tenant_date ON gtm_screening_results(tenant_id, created_at);
```

### 2.6 Licenses
```sql
CREATE TABLE gtm_licenses (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    license_number TEXT NOT NULL,
    license_type TEXT NOT NULL
        CHECK(license_type IN ('IMPORT','EXPORT','RE_EXPORT','TEMPORARY')),
    issuing_country TEXT NOT NULL,             -- ISO country code
    license_category TEXT NOT NULL
        CHECK(license_category IN ('DUAL_USE','MILITARY','AGRICULTURAL','TECHNOLOGY')),
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','SUSPENDED','REVOKED')),
    valid_from TEXT NOT NULL,
    valid_to TEXT NOT NULL,
    authorized_countries TEXT NOT NULL,         -- JSON array of ISO country codes
    authorized_items TEXT NOT NULL,             -- JSON array of item IDs or ECCNs
    quantity_limit DECIMAL(18,4),
    quantity_used DECIMAL(18,4) NOT NULL DEFAULT 0,
    value_limit_cents INTEGER NOT NULL DEFAULT 0,
    value_used_cents INTEGER NOT NULL DEFAULT 0,
    conditions TEXT,                           -- License conditions/restrictions

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, license_number)
);

CREATE INDEX idx_licenses_tenant_status ON gtm_licenses(tenant_id, status);
CREATE INDEX idx_licenses_tenant_expiry ON gtm_licenses(tenant_id, valid_to);
```

### 2.7 Customs Declarations
```sql
CREATE TABLE gtm_customs_declarations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    declaration_number TEXT NOT NULL,
    declaration_type TEXT NOT NULL
        CHECK(declaration_type IN ('IMPORT','EXPORT','IN_TRANSIT','WAREHOUSE_ENTRY')),
    shipment_id TEXT,
    port_of_entry TEXT,
    port_of_exit TEXT,
    carrier_id TEXT,
    country_of_export TEXT,                    -- ISO country code
    country_of_import TEXT,                    -- ISO country code
    total_value_cents INTEGER NOT NULL DEFAULT 0,
    total_duty_cents INTEGER NOT NULL DEFAULT 0,
    total_tax_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','FILED','ACCEPTED','REJECTED','CLEARED')),
    filed_date TEXT,
    accepted_date TEXT,
    cleared_date TEXT,
    broker_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, declaration_number)
);

CREATE INDEX idx_declarations_tenant_status ON gtm_customs_declarations(tenant_id, status);
CREATE INDEX idx_declarations_tenant_type ON gtm_customs_declarations(tenant_id, declaration_type);
CREATE INDEX idx_declarations_tenant_shipment ON gtm_customs_declarations(tenant_id, shipment_id);
CREATE INDEX idx_declarations_tenant_dates ON gtm_customs_declarations(tenant_id, filed_date, cleared_date);
```

### 2.8 Declaration Lines
```sql
CREATE TABLE gtm_declaration_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    declaration_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT NOT NULL,
    hs_code TEXT NOT NULL,
    country_of_origin TEXT NOT NULL,            -- ISO country code
    quantity DECIMAL(18,4) NOT NULL,
    uom TEXT NOT NULL,
    value_cents INTEGER NOT NULL DEFAULT 0,
    duty_rate_percent REAL NOT NULL DEFAULT 0,
    duty_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_rate_percent REAL NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    preferential_program TEXT,                 -- FTA code if applicable
    fta_applied INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (declaration_id) REFERENCES gtm_customs_declarations(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, declaration_id, line_number)
);

CREATE INDEX idx_declaration_lines_tenant_declaration ON gtm_declaration_lines(tenant_id, declaration_id);
CREATE INDEX idx_declaration_lines_tenant_item ON gtm_declaration_lines(tenant_id, item_id);
```

### 2.9 Landed Cost Simulations
```sql
CREATE TABLE gtm_landed_cost_simulations (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    simulation_name TEXT NOT NULL,
    item_id TEXT NOT NULL,
    origin_country TEXT NOT NULL,               -- ISO country code
    destination_country TEXT NOT NULL,           -- ISO country code
    incoterm TEXT NOT NULL
        CHECK(incoterm IN ('EXW','FCA','CPT','CIP','DAP','DPU','DDP','FAS','FOB','CFR','CIF')),
    unit_cost_cents INTEGER NOT NULL DEFAULT 0,
    freight_cost_cents INTEGER NOT NULL DEFAULT 0,
    insurance_cost_cents INTEGER NOT NULL DEFAULT 0,
    duty_amount_cents INTEGER NOT NULL DEFAULT 0,
    tax_amount_cents INTEGER NOT NULL DEFAULT 0,
    other_charges_cents INTEGER NOT NULL DEFAULT 0,
    total_landed_cost_cents INTEGER NOT NULL DEFAULT 0,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    effective_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_landed_cost_tenant_item ON gtm_landed_cost_simulations(tenant_id, item_id);
CREATE INDEX idx_landed_cost_tenant_date ON gtm_landed_cost_simulations(tenant_id, effective_date);
```

### 2.10 Trade Compliance Holds
```sql
CREATE TABLE gtm_trade_compliance_hold (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    transaction_type TEXT NOT NULL,             -- SALES_ORDER, PURCHASE_ORDER, SHIPMENT, etc.
    transaction_id TEXT NOT NULL,               -- FK to the originating transaction
    hold_reason TEXT NOT NULL
        CHECK(hold_reason IN ('RESTRICTED_PARTY','LICENSE_REQUIRED','CLASSIFICATION_MISSING','COUNTRY_EMBARGO','CUSTOMS_HOLD')),
    severity TEXT NOT NULL DEFAULT 'WARNING'
        CHECK(severity IN ('WARNING','BLOCK')),
    resolution TEXT
        CHECK(resolution IN ('RELEASED','RESUBMITTED','CANCELLED','ESCALATED')),
    resolved_by TEXT,
    resolution_notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, transaction_type, transaction_id, hold_reason)
);

CREATE INDEX idx_compliance_hold_tenant_severity ON gtm_trade_compliance_hold(tenant_id, severity);
CREATE INDEX idx_compliance_hold_tenant_resolution ON gtm_trade_compliance_hold(tenant_id, resolution);
```

---

## 3. REST API Endpoints

```
# Trade Preferences
GET/POST      /api/v1/gtm/preferences                       Permission: gtm.preferences.read/create
GET/PUT       /api/v1/gtm/preferences/{id}                   Permission: gtm.preferences.read/update
GET           /api/v1/gtm/preferences/{id}/eligibility       Permission: gtm.preferences.read

# Tariff & Classification
GET/POST      /api/v1/gtm/tariffs                            Permission: gtm.tariffs.read/create
GET           /api/v1/gtm/tariffs/{hs_code}                  Permission: gtm.tariffs.read
GET/POST      /api/v1/gtm/classifications                    Permission: gtm.classifications.read/create
GET/PUT       /api/v1/gtm/classifications/{id}               Permission: gtm.classifications.read/update
POST          /api/v1/gtm/classifications/{id}/confirm       Permission: gtm.classifications.update

# Restricted Party Screening
POST          /api/v1/gtm/screening/screen                   Permission: gtm.screening.execute
GET           /api/v1/gtm/screening/results                   Permission: gtm.screening.read
POST          /api/v1/gtm/screening/results/{id}/review      Permission: gtm.screening.review

# License Management
GET/POST      /api/v1/gtm/licenses                           Permission: gtm.licenses.read/create
GET/PUT       /api/v1/gtm/licenses/{id}                      Permission: gtm.licenses.read/update
POST          /api/v1/gtm/licenses/{id}/check-availability   Permission: gtm.licenses.read
GET           /api/v1/gtm/licenses/{id}/usage                 Permission: gtm.licenses.read

# Customs Declarations
GET/POST      /api/v1/gtm/declarations                       Permission: gtm.declarations.read/create
GET/PUT       /api/v1/gtm/declarations/{id}                  Permission: gtm.declarations.read/update
GET/POST      /api/v1/gtm/declarations/{id}/lines            Permission: gtm.declarations.read/create
POST          /api/v1/gtm/declarations/{id}/file             Permission: gtm.declarations.file
POST          /api/v1/gtm/declarations/{id}/cancel           Permission: gtm.declarations.update

# Landed Cost
POST          /api/v1/gtm/landed-cost/simulate               Permission: gtm.landed-cost.simulate
GET           /api/v1/gtm/landed-cost/simulations             Permission: gtm.landed-cost.read
GET           /api/v1/gtm/landed-cost/simulations/{id}       Permission: gtm.landed-cost.read

# Compliance
GET           /api/v1/gtm/compliance/holds                    Permission: gtm.compliance.read
POST          /api/v1/gtm/compliance/holds/{id}/resolve      Permission: gtm.compliance.resolve
GET           /api/v1/gtm/dashboard                           Permission: gtm.dashboard.read

# Reports
GET           /api/v1/gtm/reports/duty-savings               Permission: gtm.reports.view
GET           /api/v1/gtm/reports/compliance-summary          Permission: gtm.reports.view
GET           /api/v1/gtm/reports/fta-utilization             Permission: gtm.reports.view
```

---

## 4. Business Rules

### 4.1 Restricted Party Screening
```
Before any export or cross-border transaction:
  1. Extract all party names (buyer, consignee, end-user, carrier)
  2. Run screening against gtm_restricted_party_entries
  3. Apply fuzzy matching (Levenshtein distance, phonetic algorithms)
  4. Score matches: EXACT (1.0), PARTIAL (>0.85), FUZZY (>0.70)
  5. If CONFIRMED_MATCH: create BLOCK severity hold
  6. If POTENTIAL_MATCH: create WARNING severity hold for review
  7. Log all screening results regardless of match status
```

### 4.2 Product Classification
- Items without HS classification MUST be blocked from cross-border shipments
- Classification status PENDING items require review before use in declarations
- Binding ruling numbers override standard classification when present
- ECCN is required for all items subject to export controls
- Classification changes trigger re-screening of open shipments

### 4.3 License Management
- License availability MUST be checked and decremented for controlled items
- Quantity and value limits tracked in real-time
- Warning at 80% utilization threshold
- Auto-expiration check: licenses past valid_to set to EXPIRED
- Suspended/Revoked licenses immediately block associated transactions
- License conditions MUST be validated against transaction parameters

### 4.4 FTA Preference Qualification
- FTA qualification requires valid proof of origin and compliant bill of materials
- Rules of origin evaluated against product composition
- Regional value content calculated per FTA formula
- Tariff shift rules verified against HS code changes
- Preference programs tracked per declaration line

### 4.5 Landed Cost Simulation
- Landed cost simulation MUST include all applicable duties, taxes, and surcharges
- Components: unit cost + freight + insurance + duty + tax + other charges
- Preferential rates applied when FTA-eligible
- Anti-dumping duties included when applicable
- Incoterm determines which costs are buyer responsibility
- Simulation results are point-in-time and not guaranteed

### 4.6 Customs Declarations
- Customs declarations MUST validate all required fields before filing
- At minimum: declaration type, port, carrier, country of export/import, line items
- Each line requires: HS code, origin, quantity, UOM, value
- Declaration number auto-generated: GTM-{YYYY}-{NNNNN}
- Filed declarations cannot be modified (only cancelled and re-filed)
- Accepted declarations tracked to clearance for duty finalization

### 4.7 Compliance Holds
- BLOCK severity compliance holds prevent transaction processing entirely
- WARNING holds allow processing with audit trail
- Country embargoes result in automatic BLOCK on all transactions
- Holds resolved by authorized compliance officers only
- All hold activities logged with user, timestamp, and notes

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `gtm.screening.match_found` | Screening produces a match | Workflow, OM |
| `gtm.compliance.hold.created` | Compliance hold placed | Workflow, OM |
| `gtm.declaration.filed` | Customs declaration filed | GL |
| `gtm.declaration.cleared` | Customs clearance received | INV, GL |
| `gtm.license.expiry_warning` | License approaching expiry | Workflow |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.gtm.v1;

service GtmService {
    rpc ScreenEntity(ScreenEntityRequest) returns (ScreenEntityResponse);
    rpc CheckLicenseAvailability(CheckLicenseRequest) returns (CheckLicenseResponse);
    rpc CalculateLandedCost(CalculateLandedCostRequest) returns (CalculateLandedCostResponse);
    rpc ValidateShipment(ValidateShipmentRequest) returns (ValidateShipmentResponse);
}

message ScreenEntityRequest {
    string tenant_id = 1;
    string entity_name = 2;
    string entity_type = 3;
    string country_code = 4;
    string transaction_id = 5;
}

message ScreeningResult {
    string match_status = 1;
    double match_score = 2;
    string restricted_party_name = 3;
    string list_source = 4;
}

message ScreenEntityResponse {
    string screening_id = 1;
    repeated ScreeningResult results = 2;
    bool has_match = 3;
    string severity = 4;
}

message CheckLicenseRequest {
    string tenant_id = 1;
    string license_id = 2;
    string item_id = 3;
    int32 quantity = 4;
    string destination_country = 5;
}

message CheckLicenseResponse {
    bool available = 1;
    int32 remaining_quantity = 2;
    string license_status = 3;
    string valid_to = 4;
}

message LandedCostLine {
    string item_id = 1;
    int64 unit_cost_cents = 2;
    int64 freight_cents = 3;
    int64 insurance_cents = 4;
    int64 duty_cents = 5;
    int64 tax_cents = 6;
    int64 other_charges_cents = 7;
    int64 total_landed_cents = 8;
    string duty_rate = 9;
}

message CalculateLandedCostRequest {
    string tenant_id = 1;
    string origin_country = 2;
    string destination_country = 3;
    string incoterm = 4;
    repeated LandedCostLine lines = 5;
    string currency_code = 6;
}

message CalculateLandedCostResponse {
    repeated LandedCostLine lines = 1;
    int64 total_landed_cents = 2;
    string currency_code = 3;
    string simulation_id = 4;
}

message ValidateShipmentRequest {
    string tenant_id = 1;
    string shipment_id = 2;
    string origin_country = 3;
    string destination_country = 4;
    repeated string item_ids = 5;
    string consignee_id = 6;
}

message ComplianceIssue {
    string issue_type = 1;
    string severity = 2;
    string description = 3;
    string resolution = 4;
}

message ValidateShipmentResponse {
    bool is_compliant = 1;
    repeated ComplianceIssue issues = 2;
    bool blocked = 3;
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **OM:** Sales orders for export screening, shipping details for declarations
- **Proc:** Purchase orders for import classification, vendor details for screening
- **INV:** Item master for HS codes, inventory movements for customs
- **FA:** Asset records for customs duty capitalization
- **CM:** Carrier information for customs declarations

### 6.2 Data Published To
- **OM:** Compliance hold status, screening results block shipment release
- **Proc:** Import compliance status, restricted party screening results
- **INV:** Customs clearance events trigger inventory receipt
- **GL:** Duty and tax accounting entries from declarations
- **FA:** Customs duties capitalized into asset cost basis
- **Workflow:** Compliance holds trigger approval workflows

---

## 7. Migrations

1. V001: `gtm_trade_preferences`
2. V002: `gtm_tariff_schedules`
3. V003: `gtm_product_classifications`
4. V004: `gtm_restricted_party_entries`
5. V005: `gtm_screening_results`
6. V006: `gtm_licenses`
7. V007: `gtm_customs_declarations`
8. V008: `gtm_declaration_lines`
9. V009: `gtm_landed_cost_simulations`
10. V010: `gtm_trade_compliance_hold`
11. V011: Triggers for `updated_at`
