# 113 - Project-Driven Manufacturing Service Specification

## 1. Domain Overview

The Project-Driven Manufacturing service bridges SCM Manufacturing with ERP Project Management, enabling production orders to be tied directly to projects for project-based cost tracking, billing, and revenue recognition. Designed for make-to-order and engineer-to-order environments, it supports AEC, defense, and project-based industries by linking work orders, bills of materials, routings, costs, milestones, and deliverables to project structures.

**Bounded Context:** Project-Driven Manufacturing
**Service Name:** `projectmfg-service`
**Database:** `data/projectmfg.db`
**HTTP Port:** 8150 | **gRPC Port:** 9150

---

## 2. Database Schema

### 2.1 Project Work Orders
```sql
CREATE TABLE project_work_orders (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    project_task_id TEXT,
    work_order_number TEXT NOT NULL,
    work_order_type TEXT NOT NULL CHECK(work_order_type IN ('MAKE_TO_ORDER','ENGINEER_TO_ORDER','MAKE_TO_PROJECT')),
    item_id TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    uom TEXT NOT NULL,
    planned_start TEXT,
    planned_end TEXT,
    actual_start TEXT,
    actual_end TEXT,
    status TEXT NOT NULL DEFAULT 'DRAFT' CHECK(status IN ('DRAFT','APPROVED','RELEASED','IN_PROGRESS','COMPLETED','CLOSED','CANCELLED')),
    priority TEXT DEFAULT 'NORMAL' CHECK(priority IN ('LOW','NORMAL','HIGH','URGENT')),
    customer_id TEXT,
    contract_id TEXT,
    milestone_id TEXT,
    cost_capitalizable INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, work_order_number)
);

CREATE INDEX idx_pwo_tenant_status ON project_work_orders(tenant_id, status);
CREATE INDEX idx_pwo_tenant_project ON project_work_orders(tenant_id, project_id);
CREATE INDEX idx_pwo_tenant_customer ON project_work_orders(tenant_id, customer_id);
CREATE INDEX idx_pwo_tenant_dates ON project_work_orders(tenant_id, planned_start, planned_end);
```

### 2.2 Project BOM Links
```sql
CREATE TABLE project_bom_links (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_work_order_id TEXT NOT NULL,
    bom_id TEXT,
    component_item_id TEXT NOT NULL,
    required_qty INTEGER NOT NULL,
    provided_qty INTEGER NOT NULL DEFAULT 0,
    cost_element TEXT NOT NULL CHECK(cost_element IN ('MATERIAL','LABOR','OVERHEAD')),
    cost_cents INTEGER NOT NULL DEFAULT 0,
    source TEXT NOT NULL DEFAULT 'STOCK' CHECK(source IN ('STOCK','PROCURED','SUBCONTRACTED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_work_order_id) REFERENCES project_work_orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_pbl_tenant_wo ON project_bom_links(tenant_id, project_work_order_id);
CREATE INDEX idx_pbl_tenant_item ON project_bom_links(tenant_id, component_item_id);
CREATE INDEX idx_pbl_tenant_source ON project_bom_links(tenant_id, source);
```

### 2.3 Project Routing Links
```sql
CREATE TABLE project_routing_links (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_work_order_id TEXT NOT NULL,
    operation_seq INTEGER NOT NULL,
    work_center_id TEXT NOT NULL,
    operation_description TEXT NOT NULL,
    setup_hours DECIMAL(10,2) DEFAULT 0,
    run_time_hours DECIMAL(10,2) DEFAULT 0,
    labor_cost_cents INTEGER NOT NULL DEFAULT 0,
    overhead_cost_cents INTEGER NOT NULL DEFAULT 0,
    project_task_mapping TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_work_order_id) REFERENCES project_work_orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_prl_tenant_wo ON project_routing_links(tenant_id, project_work_order_id);
CREATE INDEX idx_prl_tenant_wc ON project_routing_links(tenant_id, work_center_id);
```

### 2.4 Project Production Costs
```sql
CREATE TABLE project_production_costs (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_work_order_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    cost_element TEXT NOT NULL CHECK(cost_element IN ('MATERIAL','DIRECT_LABOR','OVERHEAD','SUBCONTRACT','EXPENSE')),
    planned_cost_cents INTEGER NOT NULL DEFAULT 0,
    actual_cost_cents INTEGER NOT NULL DEFAULT 0,
    variance_cents INTEGER NOT NULL DEFAULT 0,
    committed_cost_cents INTEGER NOT NULL DEFAULT 0,
    remaining_cost_cents INTEGER NOT NULL DEFAULT 0,
    cost_date TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'ESTIMATED' CHECK(status IN ('ESTIMATED','COMMITTED','ACTUAL')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_work_order_id) REFERENCES project_work_orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_ppc_tenant_wo ON project_production_costs(tenant_id, project_work_order_id);
CREATE INDEX idx_ppc_tenant_project ON project_production_costs(tenant_id, project_id);
CREATE INDEX idx_ppc_tenant_element ON project_production_costs(tenant_id, cost_element);
CREATE INDEX idx_ppc_tenant_date ON project_production_costs(tenant_id, cost_date);
```

### 2.5 Project Milestones
```sql
CREATE TABLE project_milestones (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_id TEXT NOT NULL,
    milestone_name TEXT NOT NULL,
    planned_date TEXT,
    actual_date TEXT,
    billing_percentage DECIMAL(5,2) DEFAULT 0,
    production_percentage DECIMAL(5,2) DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'PLANNED' CHECK(status IN ('PLANNED','IN_PROGRESS','COMPLETED','OVERDUE')),
    billing_event_id TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_id) REFERENCES project_work_orders(project_id)
);

CREATE INDEX idx_pm_tenant_project ON project_milestones(tenant_id, project_id);
CREATE INDEX idx_pm_tenant_status ON project_milestones(tenant_id, status);
CREATE INDEX idx_pm_tenant_dates ON project_milestones(tenant_id, planned_date, actual_date);
```

### 2.6 Project Deliverables
```sql
CREATE TABLE project_deliverables (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    project_work_order_id TEXT NOT NULL,
    deliverable_name TEXT NOT NULL,
    deliverable_type TEXT NOT NULL CHECK(deliverable_type IN ('PRODUCT','SERVICE','DOCUMENT')),
    quantity INTEGER NOT NULL DEFAULT 1,
    quality_status TEXT NOT NULL DEFAULT 'PENDING' CHECK(quality_status IN ('PENDING','INSPECTED','APPROVED','REJECTED')),
    customer_acceptance_status TEXT NOT NULL DEFAULT 'PENDING' CHECK(customer_acceptance_status IN ('PENDING','ACCEPTED','REJECTED')),
    acceptance_date TEXT,
    shipment_reference TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (project_work_order_id) REFERENCES project_work_orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_pd_tenant_wo ON project_deliverables(tenant_id, project_work_order_id);
CREATE INDEX idx_pd_tenant_quality ON project_deliverables(tenant_id, quality_status);
CREATE INDEX idx_pd_tenant_acceptance ON project_deliverables(tenant_id, customer_acceptance_status);
```

---

## 3. REST API Endpoints

### Work Orders
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/project-work-orders` | Create a project work order |
| GET | `/api/v1/project-work-orders` | List project work orders |
| GET | `/api/v1/project-work-orders/{id}` | Get project work order details |
| PUT | `/api/v1/project-work-orders/{id}` | Update project work order |
| POST | `/api/v1/project-work-orders/{id}/release` | Release work order to production |
| POST | `/api/v1/project-work-orders/{id}/start` | Start production on work order |
| POST | `/api/v1/project-work-orders/{id}/complete` | Complete work order production |
| POST | `/api/v1/project-work-orders/{id}/close` | Close work order |
| GET | `/api/v1/project-work-orders/by-project/{projectId}` | List work orders for a project |

### BOM Links
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/project-work-orders/{woId}/bom-links` | Create a BOM link |
| GET | `/api/v1/project-work-orders/{woId}/bom-links` | List BOM links for work order |
| GET | `/api/v1/bom-links/{id}` | Get BOM link details |
| PUT | `/api/v1/bom-links/{id}` | Update BOM link |
| DELETE | `/api/v1/bom-links/{id}` | Delete BOM link |
| GET | `/api/v1/project-work-orders/{woId}/bom-links/cost-rollup` | Get rolled-up BOM costs |

### Routing Links
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/project-work-orders/{woId}/routing-links` | Create a routing link |
| GET | `/api/v1/project-work-orders/{woId}/routing-links` | List routing links for work order |
| GET | `/api/v1/routing-links/{id}` | Get routing link details |
| PUT | `/api/v1/routing-links/{id}` | Update routing link |
| DELETE | `/api/v1/routing-links/{id}` | Delete routing link |
| POST | `/api/v1/routing-links/{id}/sync-to-project` | Sync routing to project task |

### Production Costs
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/production-costs/record` | Record a production cost |
| GET | `/api/v1/production-costs/by-project/{projectId}` | List costs for a project |
| GET | `/api/v1/production-costs/variance-analysis` | Get cost variance analysis |
| GET | `/api/v1/production-costs/commitment-summary` | Get commitment cost summary |

### Milestones
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/milestones` | Create a milestone |
| GET | `/api/v1/milestones` | List milestones |
| GET | `/api/v1/milestones/{id}` | Get milestone details |
| PUT | `/api/v1/milestones/{id}` | Update milestone |
| POST | `/api/v1/milestones/{id}/complete` | Mark milestone as completed |
| GET | `/api/v1/milestones/billing-readiness` | Get billing readiness status |

### Deliverables
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/deliverables` | Create a deliverable |
| GET | `/api/v1/deliverables` | List deliverables |
| GET | `/api/v1/deliverables/{id}` | Get deliverable details |
| PUT | `/api/v1/deliverables/{id}` | Update deliverable |
| POST | `/api/v1/deliverables/{id}/inspect` | Record quality inspection |
| POST | `/api/v1/deliverables/{id}/customer-accept` | Record customer acceptance |

---

## 4. Business Rules

1. A project work order MUST reference a valid project_id and an item_id before it can be released.
2. Work order type MUST be one of MAKE_TO_ORDER, ENGINEER_TO_ORDER, or MAKE_TO_PROJECT, and MUST NOT be changed after the order is released.
3. The system MUST calculate variance_cents as actual_cost_cents minus planned_cost_cents for every production cost record.
4. Milestone billing_percentage values across all milestones for a project MUST NOT exceed 100 percent.
5. A deliverable MUST pass quality inspection (quality_status = APPROVED) before customer acceptance can be recorded.
6. The system MUST enforce that cost_capitalizable flag is set consistently across all BOM links of a single work order.
7. Production costs marked as ACTUAL MUST NOT be modified after the work order status transitions to CLOSED.
8. The system SHOULD auto-calculate remaining_cost_cents as planned_cost_cents minus actual_cost_cents.
9. A work order in ENGINEER_TO_ORDER type MUST have at least one routing link before it can be released.
10. Milestone status MUST transition to OVERDUE automatically when planned_date has passed without completion.
11. The system SHOULD roll up all BOM link costs to the work order level and propagate them to the project cost summary.
12. Customer acceptance rejection MUST trigger a rework workflow and set the deliverable quality_status back to PENDING.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package projectmfg.v1;

service ProjectDrivenManufacturingService {
    // Work Orders
    rpc CreateProjectWorkOrder(CreateProjectWorkOrderRequest) returns (CreateProjectWorkOrderResponse);
    rpc GetProjectWorkOrder(GetProjectWorkOrderRequest) returns (GetProjectWorkOrderResponse);
    rpc ListProjectWorkOrders(ListProjectWorkOrdersRequest) returns (ListProjectWorkOrdersResponse);
    rpc ReleaseWorkOrder(ReleaseWorkOrderRequest) returns (ReleaseWorkOrderResponse);
    rpc StartWorkOrder(StartWorkOrderRequest) returns (StartWorkOrderResponse);
    rpc CompleteWorkOrder(CompleteWorkOrderRequest) returns (CompleteWorkOrderResponse);
    rpc CloseWorkOrder(CloseWorkOrderRequest) returns (CloseWorkOrderResponse);

    // BOM Links
    rpc CreateBOMLink(CreateBOMLinkRequest) returns (CreateBOMLinkResponse);
    rpc ListBOMLinks(ListBOMLinksRequest) returns (ListBOMLinksResponse);
    rpc GetBOMCostRollup(GetBOMCostRollupRequest) returns (GetBOMCostRollupResponse);

    // Routing Links
    rpc CreateRoutingLink(CreateRoutingLinkRequest) returns (CreateRoutingLinkResponse);
    rpc ListRoutingLinks(ListRoutingLinksRequest) returns (ListRoutingLinksResponse);
    rpc SyncRoutingToProject(SyncRoutingToProjectRequest) returns (SyncRoutingToProjectResponse);

    // Production Costs
    rpc RecordProductionCost(RecordProductionCostRequest) returns (RecordProductionCostResponse);
    rpc GetCostVarianceAnalysis(GetCostVarianceAnalysisRequest) returns (GetCostVarianceAnalysisResponse);
    rpc GetCommitmentSummary(GetCommitmentSummaryRequest) returns (GetCommitmentSummaryResponse);

    // Milestones
    rpc CreateMilestone(CreateMilestoneRequest) returns (CreateMilestoneResponse);
    rpc CompleteMilestone(CompleteMilestoneRequest) returns (CompleteMilestoneResponse);
    rpc GetBillingReadiness(GetBillingReadinessRequest) returns (GetBillingReadinessResponse);

    // Deliverables
    rpc CreateDeliverable(CreateDeliverableRequest) returns (CreateDeliverableResponse);
    rpc InspectDeliverable(InspectDeliverableRequest) returns (InspectDeliverableResponse);
    rpc AcceptDeliverable(AcceptDeliverableRequest) returns (AcceptDeliverableResponse);
}

message ProjectWorkOrder {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string project_task_id = 4;
    string work_order_number = 5;
    string work_order_type = 6;
    string item_id = 7;
    int32 quantity = 8;
    string uom = 9;
    string planned_start = 10;
    string planned_end = 11;
    string actual_start = 12;
    string actual_end = 13;
    string status = 14;
    string priority = 15;
    string customer_id = 16;
    string contract_id = 17;
    string milestone_id = 18;
    bool cost_capitalizable = 19;
}

message CreateProjectWorkOrderRequest {
    string tenant_id = 1;
    string project_id = 2;
    string project_task_id = 3;
    string work_order_type = 4;
    string item_id = 5;
    int32 quantity = 6;
    string uom = 7;
    string planned_start = 8;
    string planned_end = 9;
    string priority = 10;
    string customer_id = 11;
    string contract_id = 12;
    string milestone_id = 13;
    bool cost_capitalizable = 14;
    string created_by = 15;
}

message CreateProjectWorkOrderResponse {
    ProjectWorkOrder work_order = 1;
}

message GetProjectWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
}

message GetProjectWorkOrderResponse {
    ProjectWorkOrder work_order = 1;
}

message ListProjectWorkOrdersRequest {
    string tenant_id = 1;
    string project_id = 2;
    string status = 3;
    string customer_id = 4;
    int32 page_size = 5;
    string page_token = 6;
}

message ListProjectWorkOrdersResponse {
    repeated ProjectWorkOrder work_orders = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message ReleaseWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string released_by = 3;
}

message ReleaseWorkOrderResponse {
    ProjectWorkOrder work_order = 1;
}

message StartWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string started_by = 3;
}

message StartWorkOrderResponse {
    ProjectWorkOrder work_order = 1;
}

message CompleteWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string actual_end = 3;
    string completed_by = 4;
}

message CompleteWorkOrderResponse {
    ProjectWorkOrder work_order = 1;
}

message CloseWorkOrderRequest {
    string tenant_id = 1;
    string work_order_id = 2;
    string closed_by = 3;
}

message CloseWorkOrderResponse {
    ProjectWorkOrder work_order = 1;
}

message BOMLink {
    string id = 1;
    string tenant_id = 2;
    string project_work_order_id = 3;
    string component_item_id = 4;
    int32 required_qty = 5;
    int32 provided_qty = 6;
    string cost_element = 7;
    int64 cost_cents = 8;
    string source = 9;
}

message CreateBOMLinkRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
    string component_item_id = 3;
    int32 required_qty = 4;
    string cost_element = 5;
    int64 cost_cents = 6;
    string source = 7;
    string created_by = 8;
}

message CreateBOMLinkResponse {
    BOMLink bom_link = 1;
}

message ListBOMLinksRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListBOMLinksResponse {
    repeated BOMLink bom_links = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetBOMCostRollupRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
}

message GetBOMCostRollupResponse {
    int64 total_material_cents = 1;
    int64 total_labor_cents = 2;
    int64 total_overhead_cents = 3;
    int64 grand_total_cents = 4;
}

message RoutingLink {
    string id = 1;
    string tenant_id = 2;
    string project_work_order_id = 3;
    int32 operation_seq = 4;
    string work_center_id = 5;
    string operation_description = 6;
    double setup_hours = 7;
    double run_time_hours = 8;
    int64 labor_cost_cents = 9;
    int64 overhead_cost_cents = 10;
    string project_task_mapping = 11;
}

message CreateRoutingLinkRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
    int32 operation_seq = 3;
    string work_center_id = 4;
    string operation_description = 5;
    double setup_hours = 6;
    double run_time_hours = 7;
    int64 labor_cost_cents = 8;
    int64 overhead_cost_cents = 9;
    string project_task_mapping = 10;
    string created_by = 11;
}

message CreateRoutingLinkResponse {
    RoutingLink routing_link = 1;
}

message ListRoutingLinksRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListRoutingLinksResponse {
    repeated RoutingLink routing_links = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message SyncRoutingToProjectRequest {
    string tenant_id = 1;
    string routing_link_id = 2;
    string synced_by = 3;
}

message SyncRoutingToProjectResponse {
    bool success = 1;
    string project_task_id = 2;
}

message ProductionCost {
    string id = 1;
    string tenant_id = 2;
    string project_work_order_id = 3;
    string project_id = 4;
    string cost_element = 5;
    int64 planned_cost_cents = 6;
    int64 actual_cost_cents = 7;
    int64 variance_cents = 8;
    int64 committed_cost_cents = 9;
    int64 remaining_cost_cents = 10;
    string cost_date = 11;
    string status = 12;
}

message RecordProductionCostRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
    string project_id = 3;
    string cost_element = 4;
    int64 actual_cost_cents = 5;
    int64 committed_cost_cents = 6;
    string cost_date = 7;
    string status = 8;
    string recorded_by = 9;
}

message RecordProductionCostResponse {
    ProductionCost production_cost = 1;
}

message GetCostVarianceAnalysisRequest {
    string tenant_id = 1;
    string project_id = 2;
    string cost_element = 3;
    string date_from = 4;
    string date_to = 5;
}

message GetCostVarianceAnalysisResponse {
    repeated CostVarianceItem items = 1;
    int64 total_planned_cents = 2;
    int64 total_actual_cents = 3;
    int64 total_variance_cents = 4;
}

message CostVarianceItem {
    string cost_element = 1;
    int64 planned_cents = 2;
    int64 actual_cents = 3;
    int64 variance_cents = 4;
    double variance_percentage = 5;
}

message GetCommitmentSummaryRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message GetCommitmentSummaryResponse {
    int64 total_committed_cents = 1;
    int64 total_actual_cents = 2;
    int64 total_remaining_cents = 3;
    repeated CommitmentLine lines = 4;
}

message CommitmentLine {
    string cost_element = 1;
    int64 committed_cents = 2;
    int64 actual_cents = 3;
    int64 remaining_cents = 4;
}

message Milestone {
    string id = 1;
    string tenant_id = 2;
    string project_id = 3;
    string milestone_name = 4;
    string planned_date = 5;
    string actual_date = 6;
    double billing_percentage = 7;
    double production_percentage = 8;
    string status = 9;
    string billing_event_id = 10;
}

message CreateMilestoneRequest {
    string tenant_id = 1;
    string project_id = 2;
    string milestone_name = 3;
    string planned_date = 4;
    double billing_percentage = 5;
    double production_percentage = 6;
    string created_by = 7;
}

message CreateMilestoneResponse {
    Milestone milestone = 1;
}

message CompleteMilestoneRequest {
    string tenant_id = 1;
    string milestone_id = 2;
    string actual_date = 3;
    string completed_by = 4;
}

message CompleteMilestoneResponse {
    Milestone milestone = 1;
}

message GetBillingReadinessRequest {
    string tenant_id = 1;
    string project_id = 2;
}

message GetBillingReadinessResponse {
    repeated BillingReadinessItem items = 1;
    double total_billable_percentage = 2;
    double total_billed_percentage = 3;
}

message BillingReadinessItem {
    string milestone_id = 1;
    string milestone_name = 2;
    double billing_percentage = 3;
    bool production_complete = 4;
    bool billing_ready = 5;
}

message Deliverable {
    string id = 1;
    string tenant_id = 2;
    string project_work_order_id = 3;
    string deliverable_name = 4;
    string deliverable_type = 5;
    int32 quantity = 6;
    string quality_status = 7;
    string customer_acceptance_status = 8;
    string acceptance_date = 9;
    string shipment_reference = 10;
}

message CreateDeliverableRequest {
    string tenant_id = 1;
    string project_work_order_id = 2;
    string deliverable_name = 3;
    string deliverable_type = 4;
    int32 quantity = 5;
    string created_by = 6;
}

message CreateDeliverableResponse {
    Deliverable deliverable = 1;
}

message InspectDeliverableRequest {
    string tenant_id = 1;
    string deliverable_id = 2;
    string quality_status = 3;
    string inspected_by = 4;
}

message InspectDeliverableResponse {
    Deliverable deliverable = 1;
}

message AcceptDeliverableRequest {
    string tenant_id = 1;
    string deliverable_id = 2;
    string customer_acceptance_status = 3;
    string accepted_by = 4;
}

message AcceptDeliverableResponse {
    Deliverable deliverable = 1;
}
```

---

## 6. Inter-Service Integration

### Consumed From
| Source Service | Data | Purpose |
|----------------|------|---------|
| `project-service` | Project structure, tasks, billing events | Link work orders to project tasks |
| `manufacturing-service` | Item master, BOMs, routings, work centers | Define manufacturing parameters |
| `inventory-service` | Item availability, stock levels | Determine material sourcing |
| `costing-service` | Cost elements, standard costs | Calculate planned and actual costs |
| `customer-service` | Customer profiles, contracts | Associate work orders with customers |

### Published To
| Target Service | Data | Purpose |
|----------------|------|---------|
| `project-service` | Production costs, milestone completion, deliverable status | Update project progress and costs |
| `billing-service` | Milestone billing events, billing readiness | Trigger project billing |
| `costing-service` | Actual production costs, variances | Update cost accounting |
| `inventory-service` | Material requirements, component consumption | Drive material reservations |
| `revenue-service` | Deliverable acceptance, revenue triggers | Recognize project revenue |

---

## 7. Events

### Produced Events

| Event | Topic | Payload | Description |
|-------|-------|---------|-------------|
| `ProjectWorkOrderCreated` | `projectmfg.work-order.created` | `{ tenant_id, work_order_id, work_order_number, project_id, work_order_type, item_id, quantity }` | Published when a new project work order is created |
| `ProductionCostRecorded` | `projectmfg.production-cost.recorded` | `{ tenant_id, cost_id, work_order_id, project_id, cost_element, actual_cost_cents, variance_cents }` | Published when a production cost is recorded |
| `MilestoneCompleted` | `projectmfg.milestone.completed` | `{ tenant_id, milestone_id, project_id, milestone_name, billing_percentage, actual_date }` | Published when a milestone is marked completed |
| `DeliverableAccepted` | `projectmfg.deliverable.accepted` | `{ tenant_id, deliverable_id, work_order_id, deliverable_name, acceptance_date, customer_acceptance_status }` | Published when a deliverable receives customer acceptance |
