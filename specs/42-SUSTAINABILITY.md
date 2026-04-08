# 42 - Sustainability / ESG Service Specification

## 1. Domain Overview

Sustainability/ESG Management provides comprehensive environmental, social, and governance (ESG) data collection, carbon emission tracking across all three scopes, energy consumption monitoring, waste management, water usage tracking, Scope 3 supply chain emission estimation, science-based target management, ESG report generation aligned with major frameworks (GRI, SASB, TCFD, CSRD, SEC), and supplier ESG rating assessment. The service enables organizations to measure, monitor, and reduce their environmental footprint while meeting regulatory disclosure requirements. Integrates with Procurement for supplier ESG data, INV for material waste tracking, FA for facility energy assets, and GL for environmental cost accounting.

**Bounded Context:** ESG Reporting, Carbon Tracking & Sustainability
**Service Name:** `sus-service`
**Database:** `data/sus.db`
**HTTP Port:** 8069 | **gRPC Port:** 9069

---

## 2. Database Schema

### 2.1 Emission Factors
```sql
CREATE TABLE sus_emission_factors (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    factor_name TEXT NOT NULL,
    category TEXT NOT NULL
        CHECK(category IN ('STATIONARY_COMBUSTION','MOBILE','ELECTRICITY','PROCESS','FUGITIVE')),
    fuel_type TEXT,
    factor_value REAL NOT NULL,
    factor_unit TEXT NOT NULL
        CHECK(factor_unit IN ('KG_CO2E_PER_KWH','KG_CO2E_PER_LITER','KG_CO2E_PER_KG')),
    scope INTEGER NOT NULL
        CHECK(scope IN (1, 2, 3)),
    source TEXT NOT NULL DEFAULT 'EPA'
        CHECK(source IN ('EPA','IPCC','DEFRA','CUSTOM')),
    effective_from TEXT NOT NULL,
    effective_to TEXT,
    country_code TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_emission_factors_tenant_category ON sus_emission_factors(tenant_id, category);
CREATE INDEX idx_emission_factors_tenant_scope ON sus_emission_factors(tenant_id, scope);
CREATE INDEX idx_emission_factors_tenant_fuel ON sus_emission_factors(tenant_id, fuel_type);
CREATE INDEX idx_emission_factors_tenant_country ON sus_emission_factors(tenant_id, country_code);
CREATE INDEX idx_emission_factors_tenant_active ON sus_emission_factors(tenant_id, is_active);
```

### 2.2 Emission Sources
```sql
CREATE TABLE sus_emission_sources (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_name TEXT NOT NULL,
    source_type TEXT NOT NULL
        CHECK(source_type IN ('FACILITY','VEHICLE','EQUIPMENT','PROCESS','SUPPLY_CHAIN')),
    facility_id TEXT,
    category TEXT NOT NULL,
    scope INTEGER NOT NULL
        CHECK(scope IN (1, 2, 3)),
    fuel_type TEXT,
    capacity REAL,
    capacity_unit TEXT,
    location TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, source_name)
);

CREATE INDEX idx_emission_sources_tenant_type ON sus_emission_sources(tenant_id, source_type);
CREATE INDEX idx_emission_sources_tenant_scope ON sus_emission_sources(tenant_id, scope);
CREATE INDEX idx_emission_sources_tenant_facility ON sus_emission_sources(tenant_id, facility_id);
CREATE INDEX idx_emission_sources_tenant_active ON sus_emission_sources(tenant_id, is_active);
```

### 2.3 Activity Data
```sql
CREATE TABLE sus_activity_data (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    activity_type TEXT NOT NULL
        CHECK(activity_type IN ('ENERGY','FUEL','TRANSPORT','WASTE','WATER','MATERIAL')),
    activity_value REAL NOT NULL,
    activity_unit TEXT NOT NULL
        CHECK(activity_unit IN ('KWH','LITERS','KG','TONS','M3')),
    emission_factor_id TEXT,
    calculated_emission_tonnes REAL NOT NULL DEFAULT 0,
    data_source TEXT NOT NULL DEFAULT 'INVOICE'
        CHECK(data_source IN ('METER_READING','INVOICE','ESTIMATE','SENSOR')),
    evidence_document_id TEXT,
    quality_score TEXT NOT NULL DEFAULT 'MEDIUM'
        CHECK(quality_score IN ('HIGH','MEDIUM','LOW')),
    verified_by TEXT,
    verified_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_id) REFERENCES sus_emission_sources(id),
    FOREIGN KEY (emission_factor_id) REFERENCES sus_emission_factors(id) ON DELETE SET NULL
);

CREATE INDEX idx_activity_data_tenant_source ON sus_activity_data(tenant_id, source_id);
CREATE INDEX idx_activity_data_tenant_period ON sus_activity_data(tenant_id, period_start, period_end);
CREATE INDEX idx_activity_data_tenant_type ON sus_activity_data(tenant_id, activity_type);
CREATE INDEX idx_activity_data_tenant_quality ON sus_activity_data(tenant_id, quality_score);
CREATE INDEX idx_activity_data_tenant_verified ON sus_activity_data(tenant_id, verified_at);
```

### 2.4 Emission Records
```sql
CREATE TABLE sus_emission_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    source_id TEXT NOT NULL,
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    scope INTEGER NOT NULL
        CHECK(scope IN (1, 2, 3)),
    co2_tonnes REAL NOT NULL DEFAULT 0,
    ch4_tonnes REAL NOT NULL DEFAULT 0,
    n2o_tonnes REAL NOT NULL DEFAULT 0,
    co2e_tonnes REAL NOT NULL DEFAULT 0,
    methodology TEXT NOT NULL DEFAULT 'EMISSION_FACTOR'
        CHECK(methodology IN ('CALCULATION','MEASUREMENT','EMISSION_FACTOR')),
    is_verified INTEGER NOT NULL DEFAULT 0,
    verification_body TEXT,
    verification_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (source_id) REFERENCES sus_emission_sources(id)
);

CREATE INDEX idx_emission_records_tenant_scope ON sus_emission_records(tenant_id, scope);
CREATE INDEX idx_emission_records_tenant_period ON sus_emission_records(tenant_id, period_start, period_end);
CREATE INDEX idx_emission_records_tenant_source ON sus_emission_records(tenant_id, source_id);
CREATE INDEX idx_emission_records_tenant_verified ON sus_emission_records(tenant_id, is_verified);
```

### 2.5 Energy Consumption
```sql
CREATE TABLE sus_energy_consumption (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    energy_type TEXT NOT NULL
        CHECK(energy_type IN ('ELECTRICITY','NATURAL_GAS','DIESEL','SOLAR','WIND','OTHER')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    consumption_value REAL NOT NULL,
    consumption_unit TEXT NOT NULL,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    renewable_percent REAL NOT NULL DEFAULT 0,
    supplier_id TEXT,
    meter_number TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_energy_tenant_facility ON sus_energy_consumption(tenant_id, facility_id);
CREATE INDEX idx_energy_tenant_type ON sus_energy_consumption(tenant_id, energy_type);
CREATE INDEX idx_energy_tenant_period ON sus_energy_consumption(tenant_id, period_start, period_end);
CREATE INDEX idx_energy_tenant_supplier ON sus_energy_consumption(tenant_id, supplier_id);
```

### 2.6 Waste Records
```sql
CREATE TABLE sus_waste_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    waste_type TEXT NOT NULL
        CHECK(waste_type IN ('HAZARDOUS','NON_HAZARDOUS','EWASTE','RECYCLABLE','ORGANIC')),
    waste_category TEXT,
    quantity_kg REAL NOT NULL DEFAULT 0,
    disposal_method TEXT NOT NULL
        CHECK(disposal_method IN ('LANDFILL','RECYCLE','COMPOST','INCINERATE','REUSE')),
    disposal_vendor_id TEXT,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    period_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_waste_tenant_facility ON sus_waste_records(tenant_id, facility_id);
CREATE INDEX idx_waste_tenant_type ON sus_waste_records(tenant_id, waste_type);
CREATE INDEX idx_waste_tenant_method ON sus_waste_records(tenant_id, disposal_method);
CREATE INDEX idx_waste_tenant_date ON sus_waste_records(tenant_id, period_date);
CREATE INDEX idx_waste_tenant_vendor ON sus_waste_records(tenant_id, disposal_vendor_id);
```

### 2.7 Water Usage
```sql
CREATE TABLE sus_water_usage (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    facility_id TEXT NOT NULL,
    water_source TEXT NOT NULL
        CHECK(water_source IN ('MUNICIPAL','GROUND','SURFACE','RECLAIMED')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    consumption_cubic_meters REAL NOT NULL DEFAULT 0,
    discharge_cubic_meters REAL NOT NULL DEFAULT 0,
    cost_cents INTEGER NOT NULL DEFAULT 0,
    recycling_percent REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL
);

CREATE INDEX idx_water_tenant_facility ON sus_water_usage(tenant_id, facility_id);
CREATE INDEX idx_water_tenant_source ON sus_water_usage(tenant_id, water_source);
CREATE INDEX idx_water_tenant_period ON sus_water_usage(tenant_id, period_start, period_end);
```

### 2.8 Supply Chain Emissions
```sql
CREATE TABLE sus_supply_chain_emissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    category TEXT NOT NULL
        CHECK(category IN ('PURCHASED_GOODS','TRANSPORT','WASTE_SUPPLY','BUSINESS_TRAVEL','EMPLOYEE_COMMUTE')),
    period_start TEXT NOT NULL,
    period_end TEXT NOT NULL,
    estimated_emission_tonnes REAL NOT NULL DEFAULT 0,
    data_quality TEXT NOT NULL DEFAULT 'ESTIMATED'
        CHECK(data_quality IN ('PRIMARY','SECONDARY','ESTIMATED')),
    spend_amount_cents INTEGER NOT NULL DEFAULT 0,
    emission_intensity REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_supply_chain_tenant_supplier ON sus_supply_chain_emissions(tenant_id, supplier_id);
CREATE INDEX idx_supply_chain_tenant_category ON sus_supply_chain_emissions(tenant_id, category);
CREATE INDEX idx_supply_chain_tenant_period ON sus_supply_chain_emissions(tenant_id, period_start, period_end);
CREATE INDEX idx_supply_chain_tenant_quality ON sus_supply_chain_emissions(tenant_id, data_quality);
```

### 2.9 Sustainability Targets
```sql
CREATE TABLE sus_sustainability_targets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    target_name TEXT NOT NULL,
    target_type TEXT NOT NULL
        CHECK(target_type IN ('EMISSION_REDUCTION','ENERGY_EFFICIENCY','WASTE_DIVERSION','WATER_REDUCTION','RENEWABLE_ENERGY')),
    baseline_year INTEGER NOT NULL,
    target_year INTEGER NOT NULL,
    baseline_value REAL NOT NULL,
    target_value REAL NOT NULL,
    unit TEXT NOT NULL,
    current_value REAL NOT NULL DEFAULT 0,
    progress_percent REAL NOT NULL DEFAULT 0,
    is_science_based INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'ON_TRACK'
        CHECK(status IN ('ON_TRACK','AT_RISK','BEHIND')),
    owner_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_targets_tenant_type ON sus_sustainability_targets(tenant_id, target_type);
CREATE INDEX idx_targets_tenant_status ON sus_sustainability_targets(tenant_id, status);
CREATE INDEX idx_targets_tenant_year ON sus_sustainability_targets(tenant_id, target_year);
CREATE INDEX idx_targets_tenant_owner ON sus_sustainability_targets(tenant_id, owner_id);
```

### 2.10 ESG Reports
```sql
CREATE TABLE sus_esg_reports (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    report_name TEXT NOT NULL,
    reporting_framework TEXT NOT NULL
        CHECK(reporting_framework IN ('GRI','SASB','TCFD','CSRD','SEC')),
    reporting_period_start TEXT NOT NULL,
    reporting_period_end TEXT NOT NULL,
    total_scope1_tonnes REAL NOT NULL DEFAULT 0,
    total_scope2_tonnes REAL NOT NULL DEFAULT 0,
    total_scope3_tonnes REAL NOT NULL DEFAULT 0,
    energy_consumption_total REAL,
    waste_generated_total REAL,
    water_consumed_total REAL,
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','REVIEW','PUBLISHED')),
    published_at TEXT,
    published_by TEXT,
    report_data TEXT,                          -- JSON: full report content

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_esg_reports_tenant_framework ON sus_esg_reports(tenant_id, reporting_framework);
CREATE INDEX idx_esg_reports_tenant_status ON sus_esg_reports(tenant_id, status);
CREATE INDEX idx_esg_reports_tenant_period ON sus_esg_reports(tenant_id, reporting_period_start, reporting_period_end);
CREATE INDEX idx_esg_reports_tenant_published ON sus_esg_reports(tenant_id, published_at);
```

### 2.11 Supplier ESG Ratings
```sql
CREATE TABLE sus_supplier_esg_ratings (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    assessment_date TEXT NOT NULL,
    environmental_score REAL NOT NULL DEFAULT 0,
    social_score REAL NOT NULL DEFAULT 0,
    governance_score REAL NOT NULL DEFAULT 0,
    overall_score REAL NOT NULL DEFAULT 0,
    rating TEXT NOT NULL DEFAULT 'C'
        CHECK(rating IN ('A','B','C','D','F')),
    certifications TEXT,                       -- JSON: array of certifications
    risk_flags TEXT,                           -- JSON: array of risk indicators
    assessed_by TEXT NOT NULL,
    next_assessment_date TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, supplier_id, assessment_date)
);

CREATE INDEX idx_supplier_esg_tenant_supplier ON sus_supplier_esg_ratings(tenant_id, supplier_id);
CREATE INDEX idx_supplier_esg_tenant_rating ON sus_supplier_esg_ratings(tenant_id, rating);
CREATE INDEX idx_supplier_esg_tenant_assessment ON sus_supplier_esg_ratings(tenant_id, assessment_date);
CREATE INDEX idx_supplier_esg_tenant_next ON sus_supplier_esg_ratings(tenant_id, next_assessment_date);
```

---

## 3. REST API Endpoints

```
# Emission Factors
GET/POST      /api/v1/sus/emission-factors                       Permission: sus.factors.read/create
GET/PUT       /api/v1/sus/emission-factors/{id}                  Permission: sus.factors.read/update
POST          /api/v1/sus/emission-factors/import                 Permission: sus.factors.import

# Emission Sources
GET/POST      /api/v1/sus/emission-sources                       Permission: sus.sources.read/create
GET/PUT       /api/v1/sus/emission-sources/{id}                  Permission: sus.sources.read/update

# Activity Data
GET/POST      /api/v1/sus/activity-data                          Permission: sus.activity.read/create
GET/PUT       /api/v1/sus/activity-data/{id}                     Permission: sus.activity.read/update
POST          /api/v1/sus/activity-data/{id}/verify              Permission: sus.activity.verify
POST          /api/v1/sus/activity-data/bulk-import               Permission: sus.activity.import

# Emissions
GET           /api/v1/sus/emissions                              Permission: sus.emissions.read
GET           /api/v1/sus/emissions/summary                      Permission: sus.emissions.read
GET           /api/v1/sus/emissions/by-scope                     Permission: sus.emissions.read
GET           /api/v1/sus/emissions/by-source                    Permission: sus.emissions.read

# Energy
GET/POST      /api/v1/sus/energy                                 Permission: sus.energy.read/create
GET           /api/v1/sus/energy/summary                         Permission: sus.energy.read
GET           /api/v1/sus/energy/trend                           Permission: sus.energy.read

# Waste
GET/POST      /api/v1/sus/waste                                  Permission: sus.waste.read/create
GET           /api/v1/sus/waste/summary                          Permission: sus.waste.read
GET           /api/v1/sus/waste/diversion-rate                   Permission: sus.waste.read

# Water
GET/POST      /api/v1/sus/water                                  Permission: sus.water.read/create
GET           /api/v1/sus/water/summary                          Permission: sus.water.read

# Supply Chain Emissions
GET/POST      /api/v1/sus/supply-chain                           Permission: sus.supply-chain.read/create
GET           /api/v1/sus/supply-chain/hotspot-analysis          Permission: sus.supply-chain.read

# Targets
GET/POST      /api/v1/sus/targets                                Permission: sus.targets.read/create
GET/PUT       /api/v1/sus/targets/{id}                           Permission: sus.targets.read/update
GET           /api/v1/sus/targets/{id}/progress                  Permission: sus.targets.read

# ESG Reports
GET/POST      /api/v1/sus/reports                                Permission: sus.reports.read/create
GET           /api/v1/sus/reports/{id}                           Permission: sus.reports.read
POST          /api/v1/sus/reports/{id}/generate                  Permission: sus.reports.generate
POST          /api/v1/sus/reports/{id}/publish                   Permission: sus.reports.publish

# Supplier ESG
GET/POST      /api/v1/sus/supplier-esg                           Permission: sus.supplier-esg.read/create
GET           /api/v1/sus/supplier-esg/{supplier_id}             Permission: sus.supplier-esg.read

# Dashboard
GET           /api/v1/sus/dashboard                              Permission: sus.dashboard.read
GET           /api/v1/sus/dashboard/carbon-footprint              Permission: sus.dashboard.read
GET           /api/v1/sus/dashboard/net-zero-trajectory           Permission: sus.dashboard.read
```

---

## 4. Business Rules

### 4.1 Emission Calculation
```
Emission calculation formula:
  emissions (tonnes CO2e) = activity_data_value × emission_factor_value ÷ 1000

Steps:
  1. Retrieve activity data for the source and period
  2. Match emission factor by category, fuel_type, scope, and country_code
  3. Validate emission factor is within effective_from / effective_to date range
  4. Calculate: activity_value × factor_value = raw emissions (kg CO2e)
  5. Convert to tonnes: raw emissions ÷ 1000
  6. Store in sus_emission_records with breakdown (CO2, CH4, N2O)
  7. Aggregate CO2e using GWP values: CH4 × 28 + N2O × 265
```

### 4.2 Scope Classification
- **Scope 1:** Direct emissions from owned/controlled sources (stationary combustion, mobile sources, process emissions, fugitive emissions)
- **Scope 2:** Indirect emissions from purchased energy (electricity, heating, cooling, steam)
- **Scope 3:** All other indirect emissions across value chain (purchased goods, transportation, waste, business travel, employee commute)
- Scope 1 and 2 MUST use primary data; Scope 3 allows secondary/estimated data
- Dual reporting for Scope 2: location-based and market-based methods

### 4.3 Activity Data Quality
- Activity data with LOW quality_score MUST be verified before inclusion in ESG reports
- Verification requires an authorized user to approve the data entry
- Quality score thresholds:
  - HIGH: Meter reading or sensor data with direct measurement
  - MEDIUM: Invoice-based data or validated estimates
  - LOW: Unverified estimates or extrapolated data
- Only VERIFIED activity data feeds into published emission records
- Bulk-imported data defaults to LOW quality_score unless evidence documents attached

### 4.4 ESG Report Generation
- ESG reports aggregate all emission records for the reporting period (reporting_period_start to reporting_period_end)
- Reports must include total Scope 1, Scope 2, and Scope 3 emissions in tonnes CO2e
- Energy consumption, waste generated, and water consumed aggregated from respective tables
- Supported frameworks: GRI (Global Reporting Initiative), SASB (Sustainability Accounting Standards Board), TCFD (Task Force on Climate-related Financial Disclosures), CSRD (Corporate Sustainability Reporting Directive), SEC (Securities and Exchange Commission)
- Reports progress through DRAFT -> REVIEW -> PUBLISHED states
- PUBLISHED reports are immutable; corrections require a new report version

### 4.5 Science-Based Targets
- Science-based targets MUST align with 1.5 deg C or 2 deg C pathways
- Progress is auto-calculated as: `progress_percent = ((baseline_value - current_value) / (baseline_value - target_value)) * 100`
- Status is auto-determined:
  - ON_TRACK: progress_percent >= expected_progress_for_current_year
  - AT_RISK: progress_percent < expected_progress but > 50% of expected
  - BEHIND: progress_percent < 50% of expected progress
- Targets with target_year within 2 years and status BEHIND trigger escalation alerts
- EMISSION_REDUCTION targets must specify baseline_year with validated baseline emissions

### 4.6 Supplier ESG Ratings
- overall_score = (environmental_score * 0.4) + (social_score * 0.3) + (governance_score * 0.3)
- Rating thresholds: A >= 90, B >= 80, C >= 70, D >= 60, F < 60
- Supplier ESG ratings below C trigger procurement review notification
- Risk flags are auto-generated for: recent environmental violations, missing certifications, declining score trend
- Assessments should be conducted annually; next_assessment_date auto-set to 12 months after assessment_date
- Certifications tracked as JSON array (e.g., ISO 14001, ISO 50001, Science Based Target initiative)

### 4.7 Carbon Footprint and Net-Zero
- Carbon footprint calculated at organizational level (all scopes) and optionally at product/facility level
- Net-zero trajectory modeled against reduction targets:
  - Baseline year emissions define the starting point
  - Linear or exponential reduction curve based on target ambition
  - Actual emissions plotted against trajectory to show gap/overshoot
- Dashboard shows year-over-year emission trends, scope breakdown, and target progress
- Waste diversion rate = ((recycled + composted + reused) / total_waste) * 100

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `sus.emission.calculated` | Emission record created or updated | Reporting, Dashboard |
| `sus.activity_data.verified` | Activity data verified by authorized user | Emissions |
| `sus.target.progress_updated` | Target progress recalculated | Workflow, Reporting |
| `sus.esg_report.published` | ESG report status set to PUBLISHED | DMS, Compliance |
| `sus.supplier_esg.rating_changed` | Supplier ESG rating changes | Procurement, Risk |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.sus.v1;

service SustainabilityService {
    rpc CalculateEmissions(CalculateEmissionsRequest) returns (CalculateEmissionsResponse);
    rpc GetCarbonFootprint(GetCarbonFootprintRequest) returns (GetCarbonFootprintResponse);
    rpc GetEsgScore(GetEsgScoreRequest) returns (GetEsgScoreResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Procurement:** Supplier master data for ESG assessments, purchase spend data for Scope 3 calculations, vendor details for waste disposal
- **INV:** Material consumption data for waste tracking, item master for material-based emission factors
- **FA:** Facility asset records, equipment details for emission source mapping
- **MFG:** Production output data for process emission calculations, energy consumption from manufacturing
- **CM:** Transportation data for Scope 3 mobile emission calculations
- **DMS:** Evidence document storage, report document generation
- **Workflow:** Approval workflows for report publication, target setting approvals

### 6.2 Data Published To
- **Procurement:** Supplier ESG rating updates, low-rating supplier alerts for procurement review
- **GL:** Environmental cost entries (energy costs, waste disposal costs, water costs, carbon credit purchases)
- **Risk:** ESG risk flags for supplier risk profiles, target behind-schedule alerts
- **Reporting:** Emission summaries, ESG KPI data for consolidated reporting
- **DMS:** Published ESG reports, evidence documents
- **Workflow:** Report publication approval, target escalation workflows
- **Dashboard:** Real-time carbon footprint data, net-zero trajectory metrics

---

## 7. Migrations

1. V001: `sus_emission_factors`
2. V002: `sus_emission_sources`
3. V003: `sus_activity_data`
4. V004: `sus_emission_records`
5. V005: `sus_energy_consumption`
6. V006: `sus_waste_records`
7. V007: `sus_water_usage`
8. V008: `sus_supply_chain_emissions`
9. V009: `sus_sustainability_targets`
10. V010: `sus_esg_reports`
11. V011: `sus_supplier_esg_ratings`
12. V012: Triggers for `updated_at`
