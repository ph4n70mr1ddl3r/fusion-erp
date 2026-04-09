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

## 5. Business Rules

1. **EVM Thresholds**: CPI or SPI below 0.90 triggers management review; below 0.85 triggers corrective action
2. **ITAR Compliance**: Foreign person access to ITAR items blocked without valid license
3. **Clearance Reinvestigation**: Clearances reinvestigated per schedule (T5 every 5 years, T3 every 10 years)
4. **CDRL Timeliness**: Late CDRL submissions tracked and reported to contracting officer
5. **DFARS Cybersecurity**: NIST SP 800-171 compliance required for all CUI contracts
6. **Cost Accounting**: CAS (Cost Accounting Standards) compliance for covered contracts

---

## 6. Integration Points

| Service | Integration |
|---------|-------------|
| Project Management (15) | WBS and milestones for EVM |
| Project Costing (158) | Cost collection for EVM |
| Quality Management (41) | MIL-SPEC inspection |
| Core HR (62) | Personnel and clearance tracking |
| Procurement (11) | Government procurement compliance |
| Financial Reporting (154) | Government billing (SF 1034) |
| Compliance (34) | Regulatory compliance monitoring |
