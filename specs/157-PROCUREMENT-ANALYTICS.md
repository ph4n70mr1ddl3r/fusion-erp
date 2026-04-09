# 157 - Procurement Analytics Service Specification

## 1. Domain Overview

Procurement Analytics provides spend analysis, supplier performance insights, contract compliance monitoring, and procurement process efficiency metrics. Supports spend categorization, maverick spend detection, savings tracking, PO cycle time analysis, supplier risk scoring, and contract utilization tracking. Enables procurement teams with prebuilt dashboards, custom analytics, and AI-powered insights for strategic sourcing decisions. Integrates with Procurement, Supplier Management, Sourcing, and Accounts Payable.

**Bounded Context:** Procurement Intelligence & Spend Analytics
**Service Name:** `proc-analytics-service`
**Database:** `data/proc_analytics.db`
**HTTP Port:** 8175 | **gRPC Port:** 9175

---

## 2. Database Schema

### 2.1 Spend Analysis
```sql
CREATE TABLE pa_spend_analysis (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,                   -- "2024-01"
    spend_category TEXT NOT NULL,
    sub_category TEXT,
    supplier_id TEXT,
    department_id TEXT,
    cost_center_id TEXT,
    legal_entity_id TEXT,
    total_spend_cents INTEGER NOT NULL DEFAULT 0,
    invoice_count INTEGER NOT NULL DEFAULT 0,
    po_count INTEGER NOT NULL DEFAULT 0,
    po_backed_spend_cents INTEGER NOT NULL DEFAULT 0,
    non_po_spend_cents INTEGER NOT NULL DEFAULT 0,
    contract_spend_cents INTEGER NOT NULL DEFAULT 0,
    maverick_spend_cents INTEGER NOT NULL DEFAULT 0,
    maverick_spend_pct REAL NOT NULL DEFAULT 0,
    duplicate_spend_cents INTEGER NOT NULL DEFAULT 0,
    savings_realized_cents INTEGER NOT NULL DEFAULT 0,
    avg_payment_terms_days REAL NOT NULL DEFAULT 0,
    early_payment_discount_cents INTEGER NOT NULL DEFAULT 0,
    late_payment_penalty_cents INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, period, spend_category, sub_category, supplier_id, department_id)
);

CREATE INDEX idx_pa_spend_period ON pa_spend_analysis(tenant_id, period DESC);
CREATE INDEX idx_pa_spend_category ON pa_spend_analysis(tenant_id, spend_category);
CREATE INDEX idx_pa_spend_supplier ON pa_spend_analysis(tenant_id, supplier_id);
```

### 2.2 Supplier Performance Analytics
```sql
CREATE TABLE pa_supplier_analytics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    evaluation_period TEXT NOT NULL,         -- "2024-Q1"
    total_spend_cents INTEGER NOT NULL DEFAULT 0,
    po_count INTEGER NOT NULL DEFAULT 0,
    on_time_delivery_pct REAL NOT NULL DEFAULT 0,
    quality_acceptance_pct REAL NOT NULL DEFAULT 0,
    invoice_accuracy_pct REAL NOT NULL DEFAULT 0,
    avg_lead_time_days REAL NOT NULL DEFAULT 0,
    lead_time_variability_days REAL NOT NULL DEFAULT 0,
    price_competitiveness_score REAL NOT NULL DEFAULT 0,
    responsiveness_score REAL NOT NULL DEFAULT 0,
    overall_score REAL NOT NULL DEFAULT 0,
    risk_score REAL NOT NULL DEFAULT 0,
    contract_compliance_pct REAL NOT NULL DEFAULT 0,
    sustainability_score REAL,
    incident_count INTEGER NOT NULL DEFAULT 0,
    corrective_actions_open INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, supplier_id, evaluation_period)
);

CREATE INDEX idx_pa_supplier_period ON pa_supplier_analytics(tenant_id, evaluation_period DESC);
CREATE INDEX idx_pa_supplier_score ON pa_supplier_analytics(tenant_id, overall_score);
```

### 2.3 PO Cycle Time Metrics
```sql
CREATE TABLE pa_po_metrics (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    period TEXT NOT NULL,
    department_id TEXT,
    buyer_id TEXT,
    total_pos INTEGER NOT NULL DEFAULT 0,
    avg_requisition_to_po_days REAL NOT NULL DEFAULT 0,
    avg_po_approval_days REAL NOT NULL DEFAULT 0,
    avg_po_to_receipt_days REAL NOT NULL DEFAULT 0,
    avg_receipt_to_invoice_days REAL NOT NULL DEFAULT 0,
    avg_invoice_to_payment_days REAL NOT NULL DEFAULT 0,
    avg_total_cycle_days REAL NOT NULL DEFAULT 0,
    auto_approved_pct REAL NOT NULL DEFAULT 0,
    exception_rate_pct REAL NOT NULL DEFAULT 0,
    touchless_processing_pct REAL NOT NULL DEFAULT 0,
    contract_coverage_pct REAL NOT NULL DEFAULT 0,
    catalog_usage_pct REAL NOT NULL DEFAULT 0,
    avg_lines_per_po REAL NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, period, department_id, buyer_id)
);

CREATE INDEX idx_pa_po_period ON pa_po_metrics(tenant_id, period DESC);
```

### 2.4 Contract Utilization
```sql
CREATE TABLE pa_contract_utilization (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    contract_id TEXT NOT NULL,
    period TEXT NOT NULL,
    contracted_value_cents INTEGER NOT NULL DEFAULT 0,
    utilized_value_cents INTEGER NOT NULL DEFAULT 0,
    utilization_pct REAL NOT NULL DEFAULT 0,
    off_contract_spend_cents INTEGER NOT NULL DEFAULT 0,
    savings_vs_market_cents INTEGER NOT NULL DEFAULT 0,
    remaining_value_cents INTEGER NOT NULL DEFAULT 0,
    days_remaining INTEGER NOT NULL DEFAULT 0,
    renewal_risk TEXT CHECK(renewal_risk IN ('LOW','MEDIUM','HIGH')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, contract_id, period)
);

CREATE INDEX idx_pa_contract_tenant ON pa_contract_utilization(tenant_id, period DESC);
CREATE INDEX idx_pa_contract_util ON pa_contract_utilization(tenant_id, utilization_pct);
```

### 2.5 Savings Tracking
```sql
CREATE TABLE pa_savings_tracking (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    initiative_name TEXT NOT NULL,
    initiative_type TEXT NOT NULL CHECK(initiative_type IN (
        'NEGOTIATION','CONSOLIDATION','SPECIFICATION_CHANGE','PROCESS_IMPROVEMENT',
        'SUPPLIER_SWITCH','AUCTION','CONTRACT_RENEWAL','DEMAND_REDUCTION','OTHER'
    )),
    baseline_spend_cents INTEGER NOT NULL DEFAULT 0,
    target_savings_cents INTEGER NOT NULL DEFAULT 0,
    realized_savings_cents INTEGER NOT NULL DEFAULT 0,
    savings_pct REAL NOT NULL DEFAULT 0,
    period TEXT NOT NULL,
    owner_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PLANNED'
        CHECK(status IN ('PLANNED','IN_PROGRESS','REALIZED','PARTIALLY_REALIZED','MISSED')),
    start_date TEXT NOT NULL,
    end_date TEXT NOT NULL,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    UNIQUE(tenant_id, initiative_name, period)
);

CREATE INDEX idx_pa_savings_tenant ON pa_savings_tracking(tenant_id, status);
CREATE INDEX idx_pa_savings_type ON pa_savings_tracking(tenant_id, initiative_type);
```

---

## 3. API Endpoints

### 3.1 Spend Analysis
| Method | Path | Description |
|--------|------|-------------|
| GET | `/proc-analytics/v1/spend` | Get spend analysis |
| GET | `/proc-analytics/v1/spend/by-category` | Spend by category tree |
| GET | `/proc-analytics/v1/spend/by-supplier` | Spend by supplier |
| GET | `/proc-analytics/v1/spend/by-department` | Spend by department |
| GET | `/proc-analytics/v1/spend/trend` | Spend trend over time |
| GET | `/proc-analytics/v1/spend/maverick` | Maverick spend detection |
| POST | `/proc-analytics/v1/spend/classify` | Re-classify spend categories |

### 3.2 Supplier Performance
| Method | Path | Description |
|--------|------|-------------|
| GET | `/proc-analytics/v1/supplier-performance` | Supplier scorecards |
| GET | `/proc-analytics/v1/supplier-performance/{supplierId}` | Single supplier details |
| GET | `/proc-analytics/v1/supplier-performance/ranking` | Supplier rankings |
| GET | `/proc-analytics/v1/supplier-performance/trend` | Performance trends |

### 3.3 Process Metrics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/proc-analytics/v1/po-metrics` | PO cycle time metrics |
| GET | `/proc-analytics/v1/po-metrics/trend` | Cycle time trends |
| GET | `/proc-analytics/v1/po-metrics/benchmark` | Industry benchmarks |

### 3.4 Contract & Savings
| Method | Path | Description |
|--------|------|-------------|
| GET | `/proc-analytics/v1/contract-utilization` | Contract utilization |
| GET | `/proc-analytics/v1/savings` | Savings tracking |
| GET | `/proc-analytics/v1/savings/pipeline` | Savings pipeline |
| POST | `/proc-analytics/v1/savings` | Create savings initiative |
| PUT | `/proc-analytics/v1/savings/{id}` | Update savings tracking |

### 3.5 Dashboards
| Method | Path | Description |
|--------|------|-------------|
| GET | `/proc-analytics/v1/dashboard` | Procurement dashboard |
| GET | `/proc-analytics/v1/dashboard/executive` | Executive summary |
| POST | `/proc-analytics/v1/refresh` | Trigger analytics refresh |

---

## 4. Events

### 4.1 Published Events
| Event | Payload | Description |
|-------|---------|-------------|
| `procan.spend.anomaly` | `{ category, amount, deviation }` | Spend anomaly detected |
| `procan.maverick.detected` | `{ supplier_id, amount, category }` | Maverick spend detected |
| `procan.contract.underutilized` | `{ contract_id, utilization_pct }` | Contract under-utilization |
| `procan.savings.milestone` | `{ initiative_id, realized_pct }` | Savings milestone reached |

### 4.2 Consumed Events
| Event | Source | Action |
|-------|--------|--------|
| `invoice.posted` | AP | Update spend data |
| `po.approved` | Procurement | Update PO metrics |
| `goods_receipt.completed` | Procurement | Update delivery metrics |
| `supplier.assessed` | Supplier Mgmt | Update supplier performance |

---

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.proc_analytics.v1;

service ProcurementAnalyticsService {
    // Spend analysis
    rpc GetSpendAnalysis(GetSpendAnalysisRequest) returns (SpendAnalysisList);
    rpc GetSpendBySupplier(GetSpendBySupplierRequest) returns (SupplierSpendList);
    rpc GetMaverickSpend(GetMaverickSpendRequest) returns (MaverickSpendList);
    rpc ClassifySpend(ClassifySpendRequest) returns (ClassifySpendResponse);

    // Supplier performance
    rpc GetSupplierScorecard(GetSupplierScorecardRequest) returns (SupplierAnalytics);
    rpc GetSupplierRankings(GetSupplierRankingsRequest) returns (SupplierRankingList);

    // Contract & savings
    rpc GetContractUtilization(GetContractUtilizationRequest) returns (ContractUtilizationList);
    rpc CreateSavingsInitiative(CreateSavingsInitiativeRequest) returns (SavingsTracking);

    // Dashboard
    rpc GetDashboard(GetDashboardRequest) returns (DashboardData);
}

// Entity messages
message SpendAnalysisEntry {
    string id = 1;
    string tenant_id = 2;
    string period = 3;
    string spend_category = 4;
    string sub_category = 5;
    string supplier_id = 6;
    string department_id = 7;
    string cost_center_id = 8;
    string legal_entity_id = 9;
    int64 total_spend_cents = 10;
    int32 invoice_count = 11;
    int32 po_count = 12;
    int64 po_backed_spend_cents = 13;
    int64 non_po_spend_cents = 14;
    int64 contract_spend_cents = 15;
    int64 maverick_spend_cents = 16;
    double maverick_spend_pct = 17;
    int64 duplicate_spend_cents = 18;
    int64 savings_realized_cents = 19;
    double avg_payment_terms_days = 20;
    int64 early_payment_discount_cents = 21;
    int64 late_payment_penalty_cents = 22;
}

message SupplierAnalytics {
    string id = 1;
    string tenant_id = 2;
    string supplier_id = 3;
    string evaluation_period = 4;
    int64 total_spend_cents = 5;
    int32 po_count = 6;
    double on_time_delivery_pct = 7;
    double quality_acceptance_pct = 8;
    double invoice_accuracy_pct = 9;
    double avg_lead_time_days = 10;
    double lead_time_variability_days = 11;
    double price_competitiveness_score = 12;
    double responsiveness_score = 13;
    double overall_score = 14;
    double risk_score = 15;
    double contract_compliance_pct = 16;
    double sustainability_score = 17;
    int32 incident_count = 18;
    int32 corrective_actions_open = 19;
}

message ContractUtilization {
    string id = 1;
    string tenant_id = 2;
    string contract_id = 3;
    string period = 4;
    int64 contracted_value_cents = 5;
    int64 utilized_value_cents = 6;
    double utilization_pct = 7;
    int64 off_contract_spend_cents = 8;
    int64 savings_vs_market_cents = 9;
    int64 remaining_value_cents = 10;
    int32 days_remaining = 11;
    string renewal_risk = 12;
}

message SavingsTracking {
    string id = 1;
    string tenant_id = 2;
    string initiative_name = 3;
    string initiative_type = 4;
    int64 baseline_spend_cents = 5;
    int64 target_savings_cents = 6;
    int64 realized_savings_cents = 7;
    double savings_pct = 8;
    string period = 9;
    string owner_id = 10;
    string status = 11;
    string start_date = 12;
    string end_date = 13;
}

// Request/Response messages
message GetSpendAnalysisRequest {
    string tenant_id = 1;
    string period = 2;
    string spend_category = 3;
    string department_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message SpendAnalysisList {
    repeated SpendAnalysisEntry entries = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetSpendBySupplierRequest {
    string tenant_id = 1;
    string period = 2;
    string supplier_id = 3;
}

message SupplierSpendList {
    repeated SpendAnalysisEntry entries = 1;
    int64 total_spend_cents = 2;
}

message GetMaverickSpendRequest {
    string tenant_id = 1;
    string period = 2;
    double threshold_pct = 3;
}

message MaverickSpendList {
    repeated SpendAnalysisEntry entries = 1;
    int64 total_maverick_cents = 2;
    double maverick_pct = 3;
}

message ClassifySpendRequest {
    string tenant_id = 1;
    string period = 2;
    bool reclassify_all = 3;
}

message ClassifySpendResponse {
    int32 classified_count = 1;
    int32 unchanged_count = 2;
}

message GetSupplierScorecardRequest {
    string tenant_id = 1;
    string supplier_id = 2;
    string evaluation_period = 3;
}

message GetSupplierRankingsRequest {
    string tenant_id = 1;
    string evaluation_period = 2;
    int32 page_size = 3;
}

message SupplierRankingList {
    repeated SupplierAnalytics suppliers = 1;
    int32 total_count = 2;
}

message GetContractUtilizationRequest {
    string tenant_id = 1;
    string period = 2;
    double min_utilization_pct = 3;
    double max_utilization_pct = 4;
}

message ContractUtilizationList {
    repeated ContractUtilization contracts = 1;
}

message CreateSavingsInitiativeRequest {
    string tenant_id = 1;
    string initiative_name = 2;
    string initiative_type = 3;
    int64 baseline_spend_cents = 4;
    int64 target_savings_cents = 5;
    string period = 6;
    string owner_id = 7;
    string start_date = 8;
    string end_date = 9;
}

message GetDashboardRequest {
    string tenant_id = 1;
    string period = 2;
}

message DashboardData {
    int64 total_spend_cents = 1;
    double maverick_spend_pct = 2;
    double contract_coverage_pct = 3;
    double avg_cycle_time_days = 4;
    int64 savings_realized_cents = 5;
    int32 supplier_count = 6;
}
```

## 6. Migration Order

| Migration | Table | Dependencies |
|-----------|-------|-------------|
| V001 | pa_spend_analysis | — |
| V002 | pa_supplier_analytics | — |
| V003 | pa_po_metrics | — |
| V004 | pa_contract_utilization | — |
| V005 | pa_savings_tracking | — |

---

## 7. Business Rules

1. **Spend Classification**: AI-powered spend auto-classification using invoice descriptions
2. **Maverick Detection**: Spend outside contracted suppliers flagged as maverick
3. **Cycle Time Benchmarking**: Internal benchmarks tracked; target improvement trend
4. **Savings Validation**: Realized savings require baseline comparison documentation
5. **Supplier Scoring**: Weighted composite of on-time (30%), quality (25%), cost (20%), service (15%), compliance (10%)
6. **Refresh Frequency**: Spend data refreshed daily; supplier scores monthly; savings quarterly
7. **Data Retention**: Monthly analytics retained for 5 years; weekly for 2 years

---

## 8. Inter-Service Integration

| Service | Integration |
|---------|-------------|
| Procurement (11) | Purchase order and requisition data |
| Accounts Payable (07) | Invoice and payment data |
| Supplier Management (139) | Supplier performance data |
| Sourcing (124) | Sourcing event results |
| Procurement Contracts (53) | Contract utilization data |
| ERP Analytics (149) | Financial analytics integration |
| Fusion Data Intelligence (102) | Cross-domain analytics |
| AI Agents (134) | Spend anomaly detection |
