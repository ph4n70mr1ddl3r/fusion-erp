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
| 23 | Enterprise Performance Management | `epm-service` | `33-EPM.md` |
| 24 | Risk Management & Compliance | `risk-service` | `34-RISK-COMPLIANCE.md` |
| 25 | AI/ML Platform & AI Agents | `ai-service` | `35-AI-ML-PLATFORM.md` |
| 26 | Warehouse Management (WMS) | `wms-service` | `36-WAREHOUSE-MANAGEMENT.md` |
| 27 | Transportation Management (TMS) | `tms-service` | `37-TRANSPORTATION-MANAGEMENT.md` |
| 28 | Product Lifecycle Management | `plm-service` | `38-PRODUCT-LIFECYCLE.md` |
| 29 | Global Trade Management | `gtm-service` | `39-GLOBAL-TRADE.md` |
| 30 | Enterprise Asset Management | `eam-service` | `40-ENTERPRISE-ASSET.md` |
| 31 | Quality Management | `quality-service` | `41-QUALITY-MANAGEMENT.md` |
| 32 | Sustainability / ESG | `sus-service` | `42-SUSTAINABILITY.md` |
| 33 | Digital Assistant / Conversational AI | `assistant-service` | `43-DIGITAL-ASSISTANT.md` |
| 34 | IoT Integration | `iot-service` | `44-IOT-INTEGRATION.md` |
| 35 | Supplier Portal & Sourcing | `supplier-service` | `45-SUPPLIER-PORTAL.md` |
| 36 | Mobile Application Framework | `mobile-service` | `46-MOBILE.md` |
| 37 | Blockchain & Digital Thread | `chain-service` | `47-BLOCKCHAIN.md` |
| 38 | Cost Management | `costing-service` | `48-COST-MANAGEMENT.md` |
| 39 | Accounting Hub | `accounthub-service` | `49-ACCOUNTING-HUB.md` |
| 40 | Global Order Promising | `promising-service` | `50-GLOBAL-ORDER-PROMISING.md` |
| 41 | Product Hub / MDM | `producthub-service` | `51-PRODUCT-HUB.md` |
| 42 | Supply Chain Orchestration | `orchestration-service` | `52-SUPPLY-CHAIN-ORCHESTRATION.md` |
| 43 | Procurement Contracts | `contracts-service` | `53-PROCUREMENT-CONTRACTS.md` |
| 44 | Grant Management | `grant-service` | `54-GRANT-MANAGEMENT.md` |
| 45 | Joint Venture Management | `jv-service` | `55-JOINT-VENTURE.md` |
| 46 | Subscription Management | `subscription-service` | `56-SUBSCRIPTION-MANAGEMENT.md` |
| 47 | Configure Price Quote (CPQ) | `cpq-service` | `57-CPQ.md` |
| 48 | Product Configurator | `configurator-service` | `58-CONFIGURATOR.md` |
| 49 | Commerce (B2B/B2C) | `commerce-service` | `59-COMMERCE.md` |
| 50 | Customer Data Platform | `cdp-service` | `60-CUSTOMER-DATA-PLATFORM.md` |
| 51 | Marketing | `marketing-service` | `61-MARKETING.md` |
| 52 | Core HR | `hr-service` | `62-CORE-HR.md` |
| 53 | Payroll | `payroll-service` | `63-PAYROLL.md` |
| 54 | Time & Labor | `timelabor-service` | `64-TIME-LABOR.md` |
| 55 | Compensation | `compensation-service` | `65-COMPENSATION.md` |
| 56 | Benefits | `benefits-service` | `66-BENEFITS.md` |
| 57 | Recruiting | `recruiting-service` | `67-RECRUITING.md` |
| 58 | Performance Management | `performance-service` | `68-PERFORMANCE-MANAGEMENT.md` |
| 59 | Learning & Development | `learning-service` | `69-LEARNING-DEVELOPMENT.md` |
| 60 | Succession Planning | `succession-service` | `70-SUCCESSION-PLANNING.md` |
| 61 | Career Development | `career-service` | `71-CAREER-DEVELOPMENT.md` |
| 62 | Absence Management | `absence-service` | `72-ABSENCE-MANAGEMENT.md` |
| 63 | Workforce Scheduling | `scheduling-service` | `73-WORKFORCE-SCHEDULING.md` |
| 64 | Workforce Planning | `wfplanning-service` | `74-WORKFORCE-PLANNING.md` |
| 65 | HR Help Desk | `hrhelpdesk-service` | `75-HR-HELP-DESK.md` |
| 66 | Workforce Safety | `safety-service` | `76-WORKFORCE-SAFETY.md` |
| 67 | Sales Force Automation | `sales-service` | `77-SALES-AUTOMATION.md` |
| 68 | Sales Planning | `salesplanning-service` | `78-SALES-PLANNING.md` |
| 69 | Sales Performance | `salesperf-service` | `79-SALES-PERFORMANCE.md` |
| 70 | Field Service | `fieldservice-service` | `80-FIELD-SERVICE.md` |
| 71 | Customer Service | `customerservice-service` | `81-CUSTOMER-SERVICE.md` |
| 72 | Knowledge Management | `knowledge-service` | `82-KNOWLEDGE-MANAGEMENT.md` |
| 73 | Service Logistics | `servicelogistics-service` | `83-SERVICE-LOGISTICS.md` |
| 74 | Channel Management | `channel-service` | `84-CHANNEL-MANAGEMENT.md` |
| 75 | Tax Reporting (ASC 740) | `taxreport-service` | `85-TAX-REPORTING.md` |
| 76 | Narrative Reporting | `narrative-service` | `86-NARRATIVE-REPORTING.md` |
| 77 | Enterprise Data Management | `edm-service` | `87-ENTERPRISE-DATA-MANAGEMENT.md` |
| 78 | Profitability Management | `profitability-service` | `88-PROFITABILITY-MANAGEMENT.md` |
| 79 | Integration Cloud | `integration-service` | `89-INTEGRATION-CLOUD.md` |
| 80 | Visual Builder | `builder-service` | `90-VISUAL-BUILDER.md` |
| 81 | Process Automation (RPA) | `automation-service` | `91-PROCESS-AUTOMATION.md` |
| 82 | Innovation Management | `innovation-service` | `92-INNOVATION-MANAGEMENT.md` |

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

### Stage 6 - Advanced Supply Chain
24. `36-WAREHOUSE-MANAGEMENT.md` - WMS, wave planning, picking, receiving, shipping
25. `37-TRANSPORTATION-MANAGEMENT.md` - TMS, carrier management, freight audit, routing
26. `38-PRODUCT-LIFECYCLE.md` - PLM, engineering BOM, change management, configurator
27. `39-GLOBAL-TRADE.md` - GTM, customs, restricted party screening, landed cost
28. `41-QUALITY-MANAGEMENT.md` - Quality inspections, NCR, CAPA, SPC, certificates

### Stage 7 - Enterprise Extensions
29. `33-EPM.md` - Planning, consolidation, reconciliation, allocations
30. `34-RISK-COMPLIANCE.md` - SoD, access certifications, audit, transaction controls
31. `40-ENTERPRISE-ASSET.md` - Asset operations, preventive/predictive maintenance
32. `42-SUSTAINABILITY.md` - ESG reporting, carbon tracking, sustainability targets

### Stage 8 - AI & Intelligence
33. `35-AI-ML-PLATFORM.md` - AI agents, ML models, forecasting, anomaly detection
34. `43-DIGITAL-ASSISTANT.md` - Conversational AI, notifications, quick actions
35. `44-IOT-INTEGRATION.md` - IoT devices, telemetry, edge computing, alerts

### Stage 9 - Collaboration & Delivery
36. `45-SUPPLIER-PORTAL.md` - Supplier self-service, sourcing, catalogs, scorecards
37. `46-MOBILE.md` - Mobile framework, offline sync, push notifications, scanning
38. `47-BLOCKCHAIN.md` - Blockchain traceability, provenance, certifications

### Stage 10 - Cross-Cutting Services & Delivery
39. `29-DOCUMENT-MANAGEMENT.md` - Universal file attachments
40. `30-DATA-IMPORT-EXPORT.md` - Bulk data loading, templates
41. `20-FRONTEND.md` - Web UI (Rust/WASM + HTML/CSS)
42. `21-DEPLOYMENT.md` - Containerization, orchestration
43. `22-TESTING.md` - Integration testing, load testing

### Stage 11 - Financial Extensions II
44. `48-COST-MANAGEMENT.md` - Product costing, cost rollup, variance analysis
45. `49-ACCOUNTING-HUB.md` - Centralized accounting for third-party systems
46. `54-GRANT-MANAGEMENT.md` - Federal grant lifecycle, compliance
47. `55-JOINT-VENTURE.md` - JV cost sharing, partner billing

### Stage 12 - Supply Chain Extensions II
48. `50-GLOBAL-ORDER-PROMISING.md` - ATP/CTP, delivery date promising
49. `51-PRODUCT-HUB.md` - Product MDM, catalog publishing
50. `52-SUPPLY-CHAIN-ORCHESTRATION.md` - Saga-based process coordination
51. `53-PROCUREMENT-CONTRACTS.md` - Contract lifecycle, compliance

### Stage 13 - Revenue & Commerce
52. `56-SUBSCRIPTION-MANAGEMENT.md` - Recurring billing, subscription analytics
53. `57-CPQ.md` - Configure-Price-Quote engine
54. `58-CONFIGURATOR.md` - Product configuration rules engine
55. `59-COMMERCE.md` - B2B/B2C e-commerce platform
56. `60-CUSTOMER-DATA-PLATFORM.md` - Unified customer profiles
57. `61-MARKETING.md` - Campaign management, marketing automation

### Stage 14 - Human Capital Management
58. `62-CORE-HR.md` - Employee records, org structures
59. `63-PAYROLL.md` - Payroll processing, payslips, accounting
60. `64-TIME-LABOR.md` - Time tracking, overtime, labor distribution
61. `65-COMPENSATION.md` - Salary plans, merit cycles, equity
62. `66-BENEFITS.md` - Benefits enrollment, plan administration
63. `67-RECRUITING.md` - Talent acquisition, hiring pipeline
64. `68-PERFORMANCE-MANAGEMENT.md` - Reviews, goals, PIPs
65. `69-LEARNING-DEVELOPMENT.md` - LMS, certifications, compliance training
66. `70-SUCCESSION-PLANNING.md` - Talent pipeline, readiness
67. `71-CAREER-DEVELOPMENT.md` - Skills, career paths, development plans
68. `72-ABSENCE-MANAGEMENT.md` - Leave tracking, accruals, calendars
69. `73-WORKFORCE-SCHEDULING.md` - Shift scheduling, labor optimization
70. `74-WORKFORCE-PLANNING.md` - Strategic headcount planning
71. `75-HR-HELP-DESK.md` - HR case management, knowledge base
72. `76-WORKFORCE-SAFETY.md` - Incident reporting, OSHA compliance

### Stage 15 - Customer Experience (CX/CRM)
73. `77-SALES-AUTOMATION.md` - CRM, leads, opportunities, pipeline
74. `78-SALES-PLANNING.md` - Territory and quota management
75. `79-SALES-PERFORMANCE.md` - Commissions, incentive management
76. `80-FIELD-SERVICE.md` - Technician dispatch, service operations
77. `81-CUSTOMER-SERVICE.md` - Support tickets, SLA management
78. `82-KNOWLEDGE-MANAGEMENT.md` - Knowledge base, article management
79. `83-SERVICE-LOGISTICS.md` - Service parts, depot repair, RMA
80. `84-CHANNEL-MANAGEMENT.md` - Partner portal, deal registration

### Stage 16 - EPM & Governance Extensions
81. `85-TAX-REPORTING.md` - Tax provision (ASC 740), deferred tax
82. `86-NARRATIVE-REPORTING.md` - SEC filings, report authoring, XBRL
83. `87-ENTERPRISE-DATA-MANAGEMENT.md` - Dimension governance, MDM for finance
84. `88-PROFITABILITY-MANAGEMENT.md` - Activity-based costing, margin analysis

### Stage 17 - Platform & Integration
85. `89-INTEGRATION-CLOUD.md` - iPaaS, adapters, flow designer
86. `90-VISUAL-BUILDER.md` - Low-code app builder, page designer
87. `91-PROCESS-AUTOMATION.md` - RPA, process mining, bot execution
88. `92-INNOVATION-MANAGEMENT.md` - Innovation pipeline, stage-gate

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
│   ├── epm-service/                # Enterprise Performance Management
│   ├── risk-service/               # Risk Management & Compliance
│   ├── ai-service/                 # AI/ML Platform & AI Agents
│   ├── wms-service/                # Warehouse Management
│   ├── tms-service/                # Transportation Management
│   ├── plm-service/                # Product Lifecycle Management
│   ├── gtm-service/                # Global Trade Management
│   ├── eam-service/                # Enterprise Asset Management
│   ├── quality-service/            # Quality Management
│   ├── sus-service/                # Sustainability / ESG
│   ├── assistant-service/          # Digital Assistant
│   ├── iot-service/                # IoT Integration
│   ├── supplier-service/           # Supplier Portal & Sourcing
│   ├── mobile-service/             # Mobile Application Framework
│   ├── chain-service/              # Blockchain & Digital Thread
│   ├── costing-service/            # Cost Management
│   ├── accounthub-service/         # Accounting Hub
│   ├── promising-service/          # Global Order Promising
│   ├── producthub-service/         # Product Hub / MDM
│   ├── orchestration-service/      # Supply Chain Orchestration
│   ├── contracts-service/          # Procurement Contracts
│   ├── grant-service/              # Grant Management
│   ├── jv-service/                 # Joint Venture Management
│   ├── subscription-service/       # Subscription Management
│   ├── cpq-service/                # Configure Price Quote
│   ├── configurator-service/       # Product Configurator
│   ├── commerce-service/           # Commerce (B2B/B2C)
│   ├── cdp-service/                # Customer Data Platform
│   ├── marketing-service/          # Marketing
│   ├── hr-service/                 # Core HR
│   ├── payroll-service/            # Payroll
│   ├── timelabor-service/          # Time & Labor
│   ├── compensation-service/       # Compensation
│   ├── benefits-service/           # Benefits
│   ├── recruiting-service/         # Recruiting
│   ├── performance-service/        # Performance Management
│   ├── learning-service/           # Learning & Development
│   ├── succession-service/         # Succession Planning
│   ├── career-service/             # Career Development
│   ├── absence-service/            # Absence Management
│   ├── scheduling-service/         # Workforce Scheduling
│   ├── wfplanning-service/         # Workforce Planning
│   ├── hrhelpdesk-service/         # HR Help Desk
│   ├── safety-service/             # Workforce Safety
│   ├── sales-service/              # Sales Force Automation
│   ├── salesplanning-service/      # Sales Planning
│   ├── salesperf-service/          # Sales Performance
│   ├── fieldservice-service/       # Field Service
│   ├── customerservice-service/    # Customer Service
│   ├── knowledge-service/          # Knowledge Management
│   ├── servicelogistics-service/   # Service Logistics
│   ├── channel-service/            # Channel Management
│   ├── taxreport-service/          # Tax Reporting
│   ├── narrative-service/          # Narrative Reporting
│   ├── edm-service/                # Enterprise Data Management
│   ├── profitability-service/      # Profitability Management
│   ├── integration-service/        # Integration Cloud
│   ├── builder-service/            # Visual Builder
│   ├── automation-service/         # Process Automation
│   ├── innovation-service/         # Innovation Management
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
