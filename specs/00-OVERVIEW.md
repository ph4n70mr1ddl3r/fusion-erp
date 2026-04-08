# Fusion ERP - Complete Specification Overview

## Project Vision

Build a full-featured, enterprise-grade ERP system cloning the capabilities of Oracle Fusion Cloud ERP. The system is designed as a microservices architecture, implemented entirely in Rust (backend + frontend via WebAssembly), backed by SQLite databases, and developed through spec-driven AI agent workflows.

## Target Feature Parity (Oracle Fusion Cloud ERP Modules)

| # | Module | Microservice | Spec File |
|---|--------|-------------|-----------|
| 1 | General Ledger | `gl-service` | `06-GENERAL-LEDGER.md` |
| 2 | Accounts Payable | `ap-service` | `07-ACCOUNTS-PAYABLE.md` |
| 3 | Accounts Receivable | `ar-service` | `08-ACCOUNTS-RECEIVABLE.md` |
| 4 | Fixed Assets | `fa-service` | `09-FIXED-ASSETS.md` |
| 5 | Cash Management | `cm-service` | `10-CASH-MANAGEMENT.md` |
| 6 | Procurement | `proc-service` | `11-PROCUREMENT.md` |
| 7 | Inventory | `inv-service` | `12-INVENTORY.md` |
| 8 | Order Management | `om-service` | `13-ORDER-MANAGEMENT.md` |
| 9 | Manufacturing | `mfg-service` | `14-MANUFACTURING.md` |
| 10 | Project Management | `pm-service` | `15-PROJECT-MANAGEMENT.md` |
| 11 | Workflow Engine | `workflow-service` | `16-WORKFLOW.md` |
| 12 | Reporting & Analytics | `report-service` | `17-REPORTING.md` |
| 13 | Tax Management | `tax-service` | `23-TAX-MANAGEMENT.md` |
| 14 | Intercompany Accounting | `ic-service` | `24-INTERCOMPANY.md` |
| 15 | Expense Management | `expense-service` | `25-EXPENSE-MANAGEMENT.md` |
| 16 | Revenue Management (ASC 606) | `rev-service` | `26-REVENUE-MANAGEMENT.md` |
| 17 | Advanced Pricing | `pricing-service` | `27-ADVANCED-PRICING.md` |
| 18 | Supply Chain Planning / MRP | `planning-service` | `28-SUPPLY-CHAIN-PLANNING.md` |
| 19 | Document Management | `dms-service` | `29-DOCUMENT-MANAGEMENT.md` |
| 20 | Data Import/Export | `etl-service` | `30-DATA-IMPORT-EXPORT.md` |
| 21 | Lease Accounting (IFRS 16/ASC 842) | `lease-service` | `31-LEASE-ACCOUNTING.md` |
| 22 | Collections & Credit Management | `collections-service` | `32-COLLECTIONS-CREDIT.md` |

## Supporting Infrastructure

| Component | Spec File |
|-----------|-----------|
| Microservices Architecture | `01-ARCHITECTURE.md` |
| Tech Stack (Rust + SQLite) | `02-TECH-STACK.md` |
| API Standards | `03-API-STANDARDS.md` |
| Database Design | `04-DATABASE.md` |
| Auth & Security | `05-AUTH-SECURITY.md` |
| Multi-Tenancy | `18-MULTI-TENANCY.md` |
| Inter-Service Integration | `19-INTEGRATION.md` |
| Frontend / UI | `20-FRONTEND.md` |
| Deployment | `21-DEPLOYMENT.md` |
| Testing | `22-TESTING.md` |

## Build Order (Dependency Sequence)

AI agents MUST follow this build order. Each stage depends on the prior stage being complete and tested.

### Stage 0 - Foundation
1. `02-TECH-STACK.md` - Set up workspace, shared crates, tooling
2. `01-ARCHITECTURE.md` - Scaffold all microservice crates
3. `04-DATABASE.md` - Shared database utilities, migration framework
4. `05-AUTH-SECURITY.md` - Auth service, RBAC, middleware

### Stage 1 - Core Financials
5. `06-GENERAL-LEDGER.md` - Chart of accounts, journal entries, posting
6. `07-ACCOUNTS-PAYABLE.md` - Suppliers, invoices, payments
7. `08-ACCOUNTS-RECEIVABLE.md` - Customers, invoices, receipts
8. `09-FIXED-ASSETS.md` - Asset registration, depreciation
9. `10-CASH-MANAGEMENT.md` - Bank accounts, cash positioning

### Stage 2 - Supply Chain
10. `11-PROCUREMENT.md` - Requisitions, POs, receiving
11. `12-INVENTORY.md` - Items, warehouses, stock movements
12. `13-ORDER-MANAGEMENT.md` - Sales orders, fulfillment, shipping
13. `14-MANUFACTURING.md` - BOM, work orders, production

### Stage 3 - Operations
14. `15-PROJECT-MANAGEMENT.md` - Projects, tasks, costing
15. `16-WORKFLOW.md` - Approval flows, business rules engine

### Stage 4 - Financial Extensions
16. `23-TAX-MANAGEMENT.md` - Tax engine, determination, reporting
17. `24-INTERCOMPANY.md` - Intercompany transactions, elimination
18. `25-EXPENSE-MANAGEMENT.md` - Employee expenses, policies, per diems
19. `26-REVENUE-MANAGEMENT.md` - ASC 606 revenue recognition
20. `31-LEASE-ACCOUNTING.md` - IFRS 16/ASC 842 lease accounting

### Stage 5 - Supply Chain Extensions
21. `27-ADVANCED-PRICING.md` - Price lists, promotions, discount engine
22. `28-SUPPLY-CHAIN-PLANNING.md` - MRP, demand forecasting, planned orders
23. `32-COLLECTIONS-CREDIT.md` - Collections, credit scoring, dunning

### Stage 6 - Cross-Cutting Services & Delivery
24. `29-DOCUMENT-MANAGEMENT.md` - Universal file attachments
25. `30-DATA-IMPORT-EXPORT.md` - Bulk data loading, templates
26. `20-FRONTEND.md` - Web UI (Rust/WASM + HTML/CSS)
27. `21-DEPLOYMENT.md` - Containerization, orchestration
28. `22-TESTING.md` - Integration testing, load testing

### Cross-Cutting (Build in parallel with Stage 1+)
- `18-MULTI-TENANCY.md` - Tenant isolation layer
- `19-INTEGRATION.md` - Event bus, service mesh

## Repository Structure

```
fusion/
├── specs/                          # This specification directory
│   ├── 00-OVERVIEW.md
│   ├── AGENT-GUIDE.md
│   ├── 01-ARCHITECTURE.md
│   ├── ...
│   └── 22-TESTING.md
├── Cargo.toml                      # Workspace root
├── crates/
│   ├── fusion-core/                # Shared types, traits, error handling
│   ├── fusion-db/                  # SQLite connection pool, migrations
│   ├── fusion-auth/                # Auth primitives (JWT, RBAC types)
│   ├── fusion-client/              # Generated API client
│   ├── fusion-proto/               # Shared protobuf/gRPC definitions
│   └── fusion-ui/                  # Shared UI components (WASM)
├── services/
│   ├── auth-service/               # Authentication & authorization
│   ├── gl-service/                 # General Ledger
│   ├── ap-service/                 # Accounts Payable
│   ├── ar-service/                 # Accounts Receivable
│   ├── fa-service/                 # Fixed Assets
│   ├── cm-service/                 # Cash Management
│   ├── proc-service/               # Procurement
│   ├── inv-service/                # Inventory
│   ├── om-service/                 # Order Management
│   ├── mfg-service/                # Manufacturing
│   ├── pm-service/                 # Project Management
│   ├── workflow-service/           # Workflow & Approvals
│   ├── report-service/             # Reporting & Analytics
│   ├── tax-service/                # Tax Management
│   ├── ic-service/                 # Intercompany Accounting
│   ├── expense-service/            # Expense Management
│   ├── rev-service/                # Revenue Management (ASC 606)
│   ├── pricing-service/            # Advanced Pricing
│   ├── planning-service/           # MRP & Supply Chain Planning
│   ├── dms-service/                # Document Management
│   ├── etl-service/                # Data Import/Export
│   ├── lease-service/              # Lease Accounting
│   ├── collections-service/        # Collections & Credit Management
│   └── gateway-service/            # API Gateway / BFF
├── migrations/
│   ├── auth/
│   ├── gl/
│   ├── ap/
│   └── ...
├── frontend/
│   ├── index.html
│   ├── styles/
│   └── app/                        # WASM application
├── deploy/
│   ├── Dockerfile.gateway
│   ├── Dockerfile.service          # Multi-service Dockerfile
│   └── docker-compose.yml
└── tests/
    └── integration/
```

## Conventions

- All specs use RFC 2119 keywords: MUST, SHOULD, MAY, MUST NOT
- Every API endpoint MUST have request/response schemas defined
- Every database table MUST have a migration file with up/down
- Every service MUST expose health check, readiness, and metrics endpoints
- All monetary values MUST use fixed-point decimal (i64 of minor units)
- All timestamps MUST be UTC, stored as ISO 8601 strings
- All services MUST be independently deployable and testable
