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

## 5. Business Rules

1. **Spend Classification**: AI-powered spend auto-classification using invoice descriptions
2. **Maverick Detection**: Spend outside contracted suppliers flagged as maverick
3. **Cycle Time Benchmarking**: Internal benchmarks tracked; target improvement trend
4. **Savings Validation**: Realized savings require baseline comparison documentation
5. **Supplier Scoring**: Weighted composite of on-time (30%), quality (25%), cost (20%), service (15%), compliance (10%)
6. **Refresh Frequency**: Spend data refreshed daily; supplier scores monthly; savings quarterly
7. **Data Retention**: Monthly analytics retained for 5 years; weekly for 2 years

---

## 6. Integration Points

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
