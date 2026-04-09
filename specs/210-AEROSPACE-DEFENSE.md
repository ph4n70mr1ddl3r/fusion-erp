# 210 - Aerospace & Defense Industry Solution Specification

## 1. Domain Overview

Aerospace & Defense provides industry-specific capabilities for government contract management, DFARS/ITAR compliance, Earned Value Management (EVM), MIL-SPEC quality tracking, and security clearance management. Supports government contract types (Firm Fixed Price, Cost Plus, Time & Materials), contract deliverables and CDRL management, EVMS compliance with ANSI-748, ITAR export control tracking, DFARS cybersecurity requirements, security clearance lifecycle management, and government property accountability. Enables defense contractors to maintain compliance with federal acquisition regulations, track program performance, and manage classified programs. Integrates with Project Management, Procurement, Quality Management, and Financials.

**Bounded Context:** Government Contracting & Defense Compliance
**Service Name:** `aerospace-defense-service`
**Database:** `data/aerospace_defense.db`
**HTTP Port:** 8228 | **gRPC Port:** 9228

---

## 2. Database Schema

### 2.1 Government Contracts
```sql
CREATE TABLE ad_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_number TEXT NOT NULL,
    contract_type TEXT NOT NULL CHECK(contract_type IN ('FFP','CPFF','CPIF','CPAF','T&M','IDIQ','GSA_SCHEDULE','BPA')),
    program_name TEXT NOT NULL,
    contracting_agency TEXT NOT NULL,
    contracting_officer TEXT,
    contract_value_cents INTEGER NOT NULL DEFAULT 0,
    funded_value_cents INTEGER NOT NULL DEFAULT 0,
    obligated_value_cents INTEGER NOT NULL DEFAULT 0,
    invoiced_value_cents INTEGER NOT NULL DEFAULT 0,
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,
    option_years INTEGER NOT NULL DEFAULT 0,
    pop_start TEXT NOT NULL,
    pop_end TEXT NOT NULL,
    classification_level TEXT NOT NULL DEFAULT 'UNCLASSIFIED'
        CHECK(classification_level IN ('UNCLASSIFIED','CUI','CONFIDENTIAL','SECRET','TOP_SECRET')),
    dfars_applicable INTEGER NOT NULL DEFAULT 1,
    itar_applicable INTEGER NOT NULL DEFAULT 0,
    evms_required INTEGER NOT NULL DEFAULT 0,
    cdrl_items TEXT,                              -- JSON: contract deliverables
    status TEXT NOT NULL DEFAULT 'AWARDED'
        CHECK(status IN ('PROPOSAL','AWARDED','ACTIVE','MODIFIED','COMPLETED','TERMINATED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, contract_number)
);

CREATE INDEX idx_ad_contract_agency ON ad_contracts(contracting_agency, status);
CREATE INDEX idx_ad_contract_type ON ad_contracts(contract_type);
CREATE INDEX idx_ad_contract_class ON ad_contracts(classification_level);
CREATE INDEX idx_ad_contract_dates ON ad_contracts(pop_start, pop_end);
```

### 2.2 Earned Value Management
```sql
CREATE TABLE ad_evm_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    wbs_element TEXT NOT NULL,
    reporting_period TEXT NOT NULL,
    planned_value_cents INTEGER NOT NULL DEFAULT 0,
    earned_value_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    schedule_variance_cents INTEGER NOT NULL DEFAULT 0,
    cost_variance_cents INTEGER NOT NULL DEFAULT 0,
    spi REAL NOT NULL DEFAULT 1.0,
    cpi REAL NOT NULL DEFAULT 1.0,
    eac_cents INTEGER NOT NULL DEFAULT 0,
    etc_cents INTEGER NOT NULL DEFAULT 0,
    vac_cents INTEGER NOT NULL DEFAULT 0,
    bac_cents INTEGER NOT NULL DEFAULT 0,
    completion_pct REAL NOT NULL DEFAULT 0,
    variance_explanation TEXT,
    corrective_actions TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, contract_id, wbs_element, reporting_period)
);

CREATE INDEX idx_ad_evm_contract ON ad_evm_records(contract_id, reporting_period DESC);
CREATE INDEX idx_ad_evm_cpi ON ad_evm_records(cpi);
CREATE INDEX idx_ad_evm_spi ON ad_evm_records(spi);
```

### 2.3 ITAR Compliance
```sql
CREATE TABLE ad_itar_records (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    item_description TEXT NOT NULL,
    usml_category TEXT NOT NULL,                  -- US Munitions List category
    export_classification TEXT,
    license_number TEXT,
    license_type TEXT CHECK(license_type IN ('DSP_5','DSP_73','TAAs','MLA','WDA')),
    foreign_person_access INTEGER NOT NULL DEFAULT 0,
    countries_involved TEXT,                      -- JSON: country codes
    authorized_recipients TEXT,                   -- JSON: authorized parties
    approval_authority TEXT,
    approved_at TEXT,
    expiry_date TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','EXPIRED','REVOKED','PENDING')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, item_description, usml_category)
);

CREATE INDEX idx_ad_itar_usml ON ad_itar_records(usml_category);
CREATE INDEX idx_ad_itar_license ON ad_itar_records(license_number);
CREATE INDEX idx_ad_itar_expiry ON ad_itar_records(expiry_date);
```

### 2.4 Security Clearances
```sql
CREATE TABLE ad_clearances (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    person_id TEXT NOT NULL,
    clearance_level TEXT NOT NULL CHECK(clearance_level IN ('CONFIDENTIAL','SECRET','TOP_SECRET','TS_SCI')),
    investigation_type TEXT NOT NULL CHECK(investigation_type IN ('T1','T2','T3','T4','T5')),
    sponsoring_agency TEXT NOT NULL,
    granted_date TEXT NOT NULL,
    expiration_date TEXT NOT NULL,
    reinvestigation_date TEXT NOT NULL,
    eligibility TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(eligibility IN ('ACTIVE','SUSPENDED','REVOKED','EXPIRED','PENDING')),
    program_access TEXT,                          -- JSON: programs authorized
    foreign_travel_approval TEXT,
    continuous_evaluation INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, person_id, clearance_level)
);

CREATE INDEX idx_ad_clearance_person ON ad_clearances(person_id);
CREATE INDEX idx_ad_clearance_level ON ad_clearances(clearance_level, eligibility);
CREATE INDEX idx_ad_clearance_reinvest ON ad_clearances(reinvestigation_date);
```

### 2.5 CDRL Deliverables
```sql
CREATE TABLE ad_cdrl_deliverables (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    cdrl_number TEXT NOT NULL,
    deliverable_title TEXT NOT NULL,
    deliverable_type TEXT NOT NULL CHECK(deliverable_type IN ('REPORT','DATA','SOFTWARE','HARDWARE','DOCUMENTATION','TRAINING')),
    frequency TEXT NOT NULL CHECK(frequency IN ('ONE_TIME','MONTHLY','QUARTERLY','ANNUAL','AS_REQUIRED')),
    due_date TEXT NOT NULL,
    submitted_date TEXT,
    accepted_date TEXT,
    rejected_date TEXT,
    rejection_reason TEXT,
    government_approval_required INTEGER NOT NULL DEFAULT 1,
    classification_level TEXT NOT NULL DEFAULT 'UNCLASSIFIED',
    distribution TEXT,                            -- JSON: distribution list
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','IN_PROGRESS','SUBMITTED','ACCEPTED','REJECTED','OVERDUE')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (contract_id) REFERENCES ad_contracts(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, contract_id, cdrl_number)
);

CREATE INDEX idx_ad_cdrl_contract ON ad_cdrl_deliverables(contract_id, status);
CREATE INDEX idx_ad_cdrl_due ON ad_cdrl_deliverables(due_date);
CREATE INDEX idx_ad_cdrl_status ON ad_cdrl_deliverables(tenant_id, status);
```

---

## 3. API Endpoints

### 3.1 Contracts
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/aerospace-defense/contracts` | Register contract |
| GET | `/api/v1/aerospace-defense/contracts` | List contracts |
| GET | `/api/v1/aerospace-defense/contracts/{id}` | Get contract |
| PUT | `/api/v1/aerospace-defense/contracts/{id}` | Update contract |

### 3.2 EVM
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/aerospace-defense/evm` | Record EVM data |
| GET | `/api/v1/aerospace-defense/evm/{contractId}` | EVM dashboard |
| GET | `/api/v1/aerospace-defense/evm/{contractId}/trend` | EVM trends |
| POST | `/api/v1/aerospace-defense/evm/analyze` | Variance analysis |

### 3.3 ITAR
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/aerospace-defense/itar` | Register ITAR item |
| GET | `/api/v1/aerospace-defense/itar` | List ITAR items |
| POST | `/api/v1/aerospace-defense/itar/{id}/access-check` | Check foreign access |
| GET | `/api/v1/aerospace-defense/itar/compliance` | Compliance report |

### 3.4 Clearances
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/aerospace-defense/clearances` | Record clearance |
| GET | `/api/v1/aerospace-defense/clearances` | List clearances |
| GET | `/api/v1/aerospace-defense/clearances/{personId}` | Person clearances |
| GET | `/api/v1/aerospace-defense/clearances/reinvestigations` | Due reinvestigations |

### 3.5 CDRL
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/aerospace-defense/cdrl` | Create deliverable |
| GET | `/api/v1/aerospace-defense/cdrl` | List deliverables |
| POST | `/api/v1/aerospace-defense/cdrl/{id}/submit` | Submit deliverable |
| GET | `/api/v1/aerospace-defense/cdrl/overdue` | Overdue deliverables |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `aero.contract.awarded` | `{ contract_id, agency, value }` | Contract awarded |
| `aero.evm.threshold` | `{ contract_id, cpi, spi }` | EVM threshold breach |
| `aero.itar.violation` | `{ item_id, violation_type }` | ITAR violation detected |
| `aero.clearance.expiring` | `{ person_id, level, date }` | Clearance expiring |
| `aero.cdrl.overdue` | `{ cdrl_id, contract_id, days }` | CDRL overdue |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `project.milestone.completed` | Project Mgmt (15) | Update EVM earned value |
| `employee.terminated` | Core HR (62) | Revoke clearances |
| `quality.inspection.failed` | Quality (41) | Flag deliverable quality |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package fusion.aerospace_defense.v1;

import "google/protobuf/timestamp.proto";

// ── Service ──────────────────────────────────────────────────────────
service AerospaceDefenseService {
  // Government Contracts
  rpc CreateContract(CreateContractRequest) returns (GovernmentContract);
  rpc GetContract(GetContractRequest) returns (GovernmentContract);
  rpc ListContracts(ListContractsRequest) returns (ListContractsResponse);
  rpc UpdateContract(UpdateContractRequest) returns (GovernmentContract);

  // Earned Value Management
  rpc RecordEvm(RecordEvmRequest) returns (EvmRecord);
  rpc GetEvmDashboard(GetEvmDashboardRequest) returns (EvmDashboard);

  // ITAR Compliance
  rpc RegisterItarItem(RegisterItarItemRequest) returns (ItarRecord);
  rpc CheckForeignAccess(CheckForeignAccessRequest) returns (CheckForeignAccessResponse);
  rpc ListItarItems(ListItarItemsRequest) returns (ListItarItemsResponse);

  // Security Clearances
  rpc RecordClearance(RecordClearanceRequest) returns (SecurityClearance);
  rpc ListClearances(ListClearancesRequest) returns (ListClearancesResponse);
  rpc GetReinvestigations(GetReinvestigationsRequest) returns (ListClearancesResponse);

  // CDRL Deliverables
  rpc CreateCdrl(CreateCdrlRequest) returns (CdrlDeliverable);
  rpc SubmitCdrl(SubmitCdrlRequest) returns (CdrlDeliverable);
  rpc ListCdrls(ListCdrlsRequest) returns (ListCdrlsResponse);
}

// ── Enums ────────────────────────────────────────────────────────────
enum ContractType {
  CONTRACT_TYPE_UNSPECIFIED = 0;
  CONTRACT_TYPE_FFP = 1;
  CONTRACT_TYPE_CPFF = 2;
  CONTRACT_TYPE_CPIF = 3;
  CONTRACT_TYPE_CPAF = 4;
  CONTRACT_TYPE_TM = 5;
  CONTRACT_TYPE_IDIQ = 6;
  CONTRACT_TYPE_GSA_SCHEDULE = 7;
  CONTRACT_TYPE_BPA = 8;
}

enum ContractStatus {
  CONTRACT_STATUS_UNSPECIFIED = 0;
  CONTRACT_STATUS_PROPOSAL = 1;
  CONTRACT_STATUS_AWARDED = 2;
  CONTRACT_STATUS_ACTIVE = 3;
  CONTRACT_STATUS_MODIFIED = 4;
  CONTRACT_STATUS_COMPLETED = 5;
  CONTRACT_STATUS_TERMINATED = 6;
}

enum ClassificationLevel {
  CLASSIFICATION_UNSPECIFIED = 0;
  CLASSIFICATION_UNCLASSIFIED = 1;
  CLASSIFICATION_CUI = 2;
  CLASSIFICATION_CONFIDENTIAL = 3;
  CLASSIFICATION_SECRET = 4;
  CLASSIFICATION_TOP_SECRET = 5;
}

enum ItarStatus {
  ITAR_STATUS_UNSPECIFIED = 0;
  ITAR_STATUS_ACTIVE = 1;
  ITAR_STATUS_EXPIRED = 2;
  ITAR_STATUS_REVOKED = 3;
  ITAR_STATUS_PENDING = 4;
}

enum LicenseType {
  LICENSE_TYPE_UNSPECIFIED = 0;
  LICENSE_TYPE_DSP_5 = 1;
  LICENSE_TYPE_DSP_73 = 2;
  LICENSE_TYPE_TAAS = 3;
  LICENSE_TYPE_MLA = 4;
  LICENSE_TYPE_WDA = 5;
}

enum ClearanceLevel {
  CLEARANCE_LEVEL_UNSPECIFIED = 0;
  CLEARANCE_LEVEL_CONFIDENTIAL = 1;
  CLEARANCE_LEVEL_SECRET = 2;
  CLEARANCE_LEVEL_TOP_SECRET = 3;
  CLEARANCE_LEVEL_TS_SCI = 4;
}

enum InvestigationType {
  INVESTIGATION_TYPE_UNSPECIFIED = 0;
  INVESTIGATION_TYPE_T1 = 1;
  INVESTIGATION_TYPE_T2 = 2;
  INVESTIGATION_TYPE_T3 = 3;
  INVESTIGATION_TYPE_T4 = 4;
  INVESTIGATION_TYPE_T5 = 5;
}

enum ClearanceEligibility {
  CLEARANCE_ELIGIBILITY_UNSPECIFIED = 0;
  CLEARANCE_ELIGIBILITY_ACTIVE = 1;
  CLEARANCE_ELIGIBILITY_SUSPENDED = 2;
  CLEARANCE_ELIGIBILITY_REVOKED = 3;
  CLEARANCE_ELIGIBILITY_EXPIRED = 4;
  CLEARANCE_ELIGIBILITY_PENDING = 5;
}

enum DeliverableType {
  DELIVERABLE_TYPE_UNSPECIFIED = 0;
  DELIVERABLE_TYPE_REPORT = 1;
  DELIVERABLE_TYPE_DATA = 2;
  DELIVERABLE_TYPE_SOFTWARE = 3;
  DELIVERABLE_TYPE_HARDWARE = 4;
  DELIVERABLE_TYPE_DOCUMENTATION = 5;
  DELIVERABLE_TYPE_TRAINING = 6;
}

enum DeliverableFrequency {
  DELIVERABLE_FREQ_UNSPECIFIED = 0;
  DELIVERABLE_FREQ_ONE_TIME = 1;
  DELIVERABLE_FREQ_MONTHLY = 2;
  DELIVERABLE_FREQ_QUARTERLY = 3;
  DELIVERABLE_FREQ_ANNUAL = 4;
  DELIVERABLE_FREQ_AS_REQUIRED = 5;
}

enum CdrlStatus {
  CDRL_STATUS_UNSPECIFIED = 0;
  CDRL_STATUS_PENDING = 1;
  CDRL_STATUS_IN_PROGRESS = 2;
  CDRL_STATUS_SUBMITTED = 3;
  CDRL_STATUS_ACCEPTED = 4;
  CDRL_STATUS_REJECTED = 5;
  CDRL_STATUS_OVERDUE = 6;
}

// ── Common Messages ──────────────────────────────────────────────────
message AuditInfo {
  string created_at = 1;
  string updated_at = 2;
  string created_by = 3;
  string updated_by = 4;
  int32 version = 5;
}

// ── Government Contract Messages ─────────────────────────────────────
message GovernmentContract {
  string id = 1;
  string tenant_id = 2;
  string contract_number = 3;
  ContractType contract_type = 4;
  string program_name = 5;
  string contracting_agency = 6;
  string contracting_officer = 7;
  int64 contract_value_cents = 8;
  int64 funded_value_cents = 9;
  int64 obligated_value_cents = 10;
  int64 invoiced_value_cents = 11;
  string start_date = 12;
  string end_date = 13;
  int32 option_years = 14;
  string pop_start = 15;
  string pop_end = 16;
  ClassificationLevel classification_level = 17;
  bool dfars_applicable = 18;
  bool itar_applicable = 19;
  bool evms_required = 20;
  string cdrl_items = 21;              // JSON
  ContractStatus status = 22;
  AuditInfo audit = 23;
}

message CreateContractRequest {
  string tenant_id = 1;
  string contract_number = 2;
  ContractType contract_type = 3;
  string program_name = 4;
  string contracting_agency = 5;
  string contracting_officer = 6;
  int64 contract_value_cents = 7;
  int64 funded_value_cents = 8;
  string start_date = 9;
  string end_date = 10;
  int32 option_years = 11;
  string pop_start = 12;
  string pop_end = 13;
  ClassificationLevel classification_level = 14;
  bool dfars_applicable = 15;
  bool itar_applicable = 16;
  bool evms_required = 17;
  string user_id = 18;
}

message GetContractRequest {
  string id = 1;
  string tenant_id = 2;
}

message ListContractsRequest {
  string tenant_id = 1;
  ContractStatus status = 2;
  ContractType contract_type = 3;
  string contracting_agency = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListContractsResponse {
  repeated GovernmentContract contracts = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

message UpdateContractRequest {
  string id = 1;
  string tenant_id = 2;
  int64 funded_value_cents = 3;
  int64 obligated_value_cents = 4;
  int64 invoiced_value_cents = 5;
  ContractStatus status = 6;
  string user_id = 7;
}

// ── EVM Messages ─────────────────────────────────────────────────────
message EvmRecord {
  string id = 1;
  string tenant_id = 2;
  string contract_id = 3;
  string wbs_element = 4;
  string reporting_period = 5;
  int64 planned_value_cents = 6;
  int64 earned_value_cents = 7;
  int64 actual_cost_cents = 8;
  int64 schedule_variance_cents = 9;
  int64 cost_variance_cents = 10;
  double spi = 11;
  double cpi = 12;
  int64 eac_cents = 13;
  int64 etc_cents = 14;
  int64 vac_cents = 15;
  int64 bac_cents = 16;
  double completion_pct = 17;
  string variance_explanation = 18;
  string corrective_actions = 19;
  AuditInfo audit = 20;
}

message RecordEvmRequest {
  string tenant_id = 1;
  string contract_id = 2;
  string wbs_element = 3;
  string reporting_period = 4;
  int64 planned_value_cents = 5;
  int64 earned_value_cents = 6;
  int64 actual_cost_cents = 7;
  int64 eac_cents = 8;
  int64 etc_cents = 9;
  int64 bac_cents = 10;
  double completion_pct = 11;
  string variance_explanation = 12;
  string corrective_actions = 13;
  string user_id = 14;
}

message EvmDashboard {
  string contract_id = 1;
  repeated EvmRecord records = 2;
  double overall_spi = 3;
  double overall_cpi = 4;
  int64 total_planned_value_cents = 5;
  int64 total_earned_value_cents = 6;
  int64 total_actual_cost_cents = 7;
  double overall_completion_pct = 8;
}

message GetEvmDashboardRequest {
  string contract_id = 1;
  string tenant_id = 2;
  string reporting_period = 3;
}

// ── ITAR Messages ────────────────────────────────────────────────────
message ItarRecord {
  string id = 1;
  string tenant_id = 2;
  string item_description = 3;
  string usml_category = 4;
  string export_classification = 5;
  string license_number = 6;
  LicenseType license_type = 7;
  bool foreign_person_access = 8;
  string countries_involved = 9;       // JSON
  string authorized_recipients = 10;   // JSON
  string approval_authority = 11;
  string approved_at = 12;
  string expiry_date = 13;
  ItarStatus status = 14;
  AuditInfo audit = 15;
}

message RegisterItarItemRequest {
  string tenant_id = 1;
  string item_description = 2;
  string usml_category = 3;
  string export_classification = 4;
  string license_number = 5;
  LicenseType license_type = 6;
  string countries_involved = 7;       // JSON
  string authorized_recipients = 8;    // JSON
  string approval_authority = 9;
  string expiry_date = 10;
  string user_id = 11;
}

message CheckForeignAccessRequest {
  string itar_item_id = 1;
  string tenant_id = 2;
  string person_id = 3;
  string person_nationality = 4;
}

message CheckForeignAccessResponse {
  bool access_allowed = 1;
  string reason = 2;
  string applicable_license = 3;
}

message ListItarItemsRequest {
  string tenant_id = 1;
  ItarStatus status = 2;
  string usml_category = 3;
  int32 page_size = 4;
  string page_token = 5;
}

message ListItarItemsResponse {
  repeated ItarRecord items = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

// ── Security Clearance Messages ──────────────────────────────────────
message SecurityClearance {
  string id = 1;
  string tenant_id = 2;
  string person_id = 3;
  ClearanceLevel clearance_level = 4;
  InvestigationType investigation_type = 5;
  string sponsoring_agency = 6;
  string granted_date = 7;
  string expiration_date = 8;
  string reinvestigation_date = 9;
  ClearanceEligibility eligibility = 10;
  string program_access = 11;          // JSON
  string foreign_travel_approval = 12;
  bool continuous_evaluation = 13;
  AuditInfo audit = 14;
}

message RecordClearanceRequest {
  string tenant_id = 1;
  string person_id = 2;
  ClearanceLevel clearance_level = 3;
  InvestigationType investigation_type = 4;
  string sponsoring_agency = 5;
  string granted_date = 6;
  string expiration_date = 7;
  string reinvestigation_date = 8;
  string program_access = 9;           // JSON
  bool continuous_evaluation = 10;
  string user_id = 11;
}

message ListClearancesRequest {
  string tenant_id = 1;
  string person_id = 2;
  ClearanceLevel clearance_level = 3;
  ClearanceEligibility eligibility = 4;
  int32 page_size = 5;
  string page_token = 6;
}

message ListClearancesResponse {
  repeated SecurityClearance clearances = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

message GetReinvestigationsRequest {
  string tenant_id = 1;
  string before_date = 2;
  int32 page_size = 3;
  string page_token = 4;
}

// ── CDRL Deliverable Messages ────────────────────────────────────────
message CdrlDeliverable {
  string id = 1;
  string tenant_id = 2;
  string contract_id = 3;
  string cdrl_number = 4;
  string deliverable_title = 5;
  DeliverableType deliverable_type = 6;
  DeliverableFrequency frequency = 7;
  string due_date = 8;
  string submitted_date = 9;
  string accepted_date = 10;
  string rejected_date = 11;
  string rejection_reason = 12;
  bool government_approval_required = 13;
  ClassificationLevel classification_level = 14;
  string distribution = 15;            // JSON
  CdrlStatus status = 16;
  AuditInfo audit = 17;
}

message CreateCdrlRequest {
  string tenant_id = 1;
  string contract_id = 2;
  string cdrl_number = 3;
  string deliverable_title = 4;
  DeliverableType deliverable_type = 5;
  DeliverableFrequency frequency = 6;
  string due_date = 7;
  bool government_approval_required = 8;
  ClassificationLevel classification_level = 9;
  string distribution = 10;            // JSON
  string user_id = 11;
}

message SubmitCdrlRequest {
  string id = 1;
  string tenant_id = 2;
  string submitted_date = 3;
}

message ListCdrlsRequest {
  string tenant_id = 1;
  string contract_id = 2;
  CdrlStatus status = 3;
  int32 page_size = 4;
  string page_token = 5;
}

message ListCdrlsResponse {
  repeated CdrlDeliverable deliverables = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}
```

## 6. Migration Order

| Order | Table | Depends On |
|-------|-------|------------|
| 1 | `ad_contracts` | — |
| 2 | `ad_evm_records` | `ad_contracts` |
| 3 | `ad_itar_records` | — |
| 4 | `ad_clearances` | — |
| 5 | `ad_cdrl_deliverables` | `ad_contracts` |

---

## 7. Business Rules

1. **EVM Thresholds**: CPI or SPI below 0.90 triggers management review; below 0.85 triggers corrective action
2. **ITAR Compliance**: Foreign person access to ITAR items blocked without valid license
3. **Clearance Reinvestigation**: Clearances reinvestigated per schedule (T5 every 5 years, T3 every 10 years)
4. **CDRL Timeliness**: Late CDRL submissions tracked and reported to contracting officer
5. **DFARS Cybersecurity**: NIST SP 800-171 compliance required for all CUI contracts
6. **Cost Accounting**: CAS (Cost Accounting Standards) compliance for covered contracts

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Project Management (15) | WBS and milestones for EVM |
| Project Costing (158) | Cost collection for EVM |
| Quality Management (41) | MIL-SPEC inspection |
| Core HR (62) | Personnel and clearance tracking |
| Procurement (11) | Government procurement compliance |
| Financial Reporting (154) | Government billing (SF 1034) |
| Compliance (34) | Regulatory compliance monitoring |
