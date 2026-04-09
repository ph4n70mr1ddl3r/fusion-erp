# Spec Index by Domain

Quick-reference index grouping all 211 module specs by domain category.
For full module details, see `00-OVERVIEW.md`. For build order, see `AGENT-GUIDE.md`.

---

## Foundation & Infrastructure (11 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `01-ARCHITECTURE.md` | Microservices Architecture | — |
| `02-TECH-STACK.md` | Technology Stack (Rust + SQLite) | — |
| `03-API-STANDARDS.md` | API Design Standards | — |
| `04-DATABASE.md` | Database Design | — |
| `05-AUTH-SECURITY.md` | Authentication & Security | `auth-service` |
| `18-MULTI-TENANCY.md` | Multi-Tenancy | — |
| `19-INTEGRATION.md` | Inter-Service Integration | — |
| `20-FRONTEND.md` | Frontend / UI (Rust/WASM) | — |
| `21-DEPLOYMENT.md` | Deployment | — |
| `22-TESTING.md` | Testing Strategy | — |
| `151-REDWOOD-UX.md` | Redwood UX Design System | — |

---

## Core Financials (11 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `06-GENERAL-LEDGER.md` | General Ledger | `gl-service` |
| `07-ACCOUNTS-PAYABLE.md` | Accounts Payable | `ap-service` |
| `08-ACCOUNTS-RECEIVABLE.md` | Accounts Receivable | `ar-service` |
| `09-FIXED-ASSETS.md` | Fixed Assets | `fa-service` |
| `10-CASH-MANAGEMENT.md` | Cash Management | `cm-service` |
| `23-TAX-MANAGEMENT.md` | Tax Management | `tax-service` |
| `24-INTERCOMPANY.md` | Intercompany Accounting | `ic-service` |
| `25-EXPENSE-MANAGEMENT.md` | Expense Management | `expense-service` |
| `26-REVENUE-MANAGEMENT.md` | Revenue Management (ASC 606) | `rev-service` |
| `31-LEASE-ACCOUNTING.md` | Lease Accounting (IFRS 16/ASC 842) | `lease-service` |
| `32-COLLECTIONS-CREDIT.md` | Collections & Credit Management | `collections-service` |

## Financial Extensions (13 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `48-COST-MANAGEMENT.md` | Cost Management | `costing-service` |
| `49-ACCOUNTING-HUB.md` | Accounting Hub | `accounthub-service` |
| `54-GRANT-MANAGEMENT.md` | Grant Management | `grant-service` |
| `55-JOINT-VENTURE.md` | Joint Venture Management | `jv-service` |
| `100-FINANCIAL-CONSOLIDATION.md` | Financial Consolidation & Close | `consolidation-service` |
| `101-ACCOUNT-RECONCILIATION.md` | Account Reconciliation | `recon-service` |
| `104-FEDERAL-FINANCIALS.md` | U.S. Federal Financials | `federal-service` |
| `154-FINANCIAL-REPORTING.md` | Financial Reporting | `financialreport-service` |
| `158-PROJECT-COSTING.md` | Project Costing | `projectcosting-service` |
| `159-PROJECT-BILLING.md` | Project Billing | `projectbilling-service` |
| `185-PLANNING-BUDGETING.md` | Planning and Budgeting | `planningbudget-service` |
| `187-CALCULATION-MANAGER.md` | Calculation Manager | `calcmanager-service` |
| `202-FUSION-DATA-INTELLIGENCE.md` | Fusion Data Intelligence Platform | `fdiplatform-service` |

---

## Supply Chain - Core (6 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `11-PROCUREMENT.md` | Procurement | `proc-service` |
| `12-INVENTORY.md` | Inventory | `inv-service` |
| `13-ORDER-MANAGEMENT.md` | Order Management | `om-service` |
| `14-MANUFACTURING.md` | Manufacturing | `mfg-service` |
| `27-ADVANCED-PRICING.md` | Advanced Pricing | `pricing-service` |
| `28-SUPPLY-CHAIN-PLANNING.md` | Supply Chain Planning / MRP | `planning-service` |

## Supply Chain - Extended (11 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `36-WAREHOUSE-MANAGEMENT.md` | Warehouse Management (WMS) | `wms-service` |
| `37-TRANSPORTATION-MANAGEMENT.md` | Transportation Management (TMS) | `tms-service` |
| `38-PRODUCT-LIFECYCLE.md` | Product Lifecycle Management | `plm-service` |
| `39-GLOBAL-TRADE.md` | Global Trade Management | `gtm-service` |
| `41-QUALITY-MANAGEMENT.md` | Quality Management | `quality-service` |
| `50-GLOBAL-ORDER-PROMISING.md` | Global Order Promising | `promising-service` |
| `51-PRODUCT-HUB.md` | Product Hub / MDM | `producthub-service` |
| `52-SUPPLY-CHAIN-ORCHESTRATION.md` | Supply Chain Orchestration | `orchestration-service` |
| `53-PROCUREMENT-CONTRACTS.md` | Procurement Contracts | `contracts-service` |
| `98-CONTRACT-MANUFACTURING.md` | Contract Manufacturing | `contractmfg-service` |
| `99-MANUFACTURING-EXECUTION.md` | Manufacturing Execution System (MES) | `mes-service` |

## Supply Chain - Advanced (9 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `120-SELF-SERVICE-PROCUREMENT.md` | Self-Service Procurement | `selfproc-service` |
| `121-PROCESS-MANUFACTURING.md` | Process Manufacturing | `processmfg-service` |
| `122-PRODUCTION-SCHEDULING.md` | Production Scheduling | `prodsched-service` |
| `123-SMART-OPERATIONS.md` | Smart Operations | `smartops-service` |
| `124-SOURCING.md` | Sourcing | `sourcing-service` |
| `125-SUPPLY-CHAIN-EXECUTION.md` | Supply Chain Execution | `scexec-service` |
| `139-SUPPLIER-MANAGEMENT.md` | Supplier Management | `suppliermgmt-service` |
| `140-DEMAND-MANAGEMENT.md` | Demand Management | `demand-service` |
| `141-SALES-OPERATIONS-PLANNING.md` | Sales & Operations Planning (S&OP) | `sop-service` |

## Supply Chain - Logistics & Fulfillment (4 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `142-SUPPLY-CHAIN-COLLABORATION.md` | Supply Chain Collaboration | `sccollab-service` |
| `143-BACKLOG-MANAGEMENT.md` | Backlog Management | `backlog-service` |
| `144-MIXED-MODE-MANUFACTURING.md` | Mixed-Mode Manufacturing | `mixedmfg-service` |
| `145-SHIPPING-EXECUTION.md` | Shipping Execution | `shipping-service` |

---

## Operations & Projects (7 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `15-PROJECT-MANAGEMENT.md` | Project Management | `pm-service` |
| `16-WORKFLOW.md` | Workflow & Approval Engine | `workflow-service` |
| `17-REPORTING.md` | Reporting & Analytics | `report-service` |
| `29-DOCUMENT-MANAGEMENT.md` | Document Management | `dms-service` |
| `30-DATA-IMPORT-EXPORT.md` | Data Import/Export Framework | `etl-service` |
| `113-PROJECT-DRIVEN-MANUFACTURING.md` | Project-Driven Manufacturing | `projectmfg-service` |
| `115-PROJECT-ASSET-MANAGEMENT.md` | Project Asset Management | `projectasset-service` |
| `160-RESOURCE-MANAGEMENT.md` | Resource Management | `resource-service` |

---

## Human Capital Management - Core (11 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `62-CORE-HR.md` | Core HR | `hr-service` |
| `63-PAYROLL.md` | Payroll | `payroll-service` |
| `64-TIME-LABOR.md` | Time & Labor | `timelabor-service` |
| `65-COMPENSATION.md` | Compensation | `compensation-service` |
| `66-BENEFITS.md` | Benefits | `benefits-service` |
| `67-RECRUITING.md` | Recruiting | `recruiting-service` |
| `68-PERFORMANCE-MANAGEMENT.md` | Performance Management | `performance-service` |
| `69-LEARNING-DEVELOPMENT.md` | Learning & Development | `learning-service` |
| `70-SUCCESSION-PLANNING.md` | Succession Planning | `succession-service` |
| `71-CAREER-DEVELOPMENT.md` | Career Development | `career-service` |
| `72-ABSENCE-MANAGEMENT.md` | Absence Management | `absence-service` |

## HCM - Workforce (9 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `73-WORKFORCE-SCHEDULING.md` | Workforce Scheduling | `scheduling-service` |
| `74-WORKFORCE-PLANNING.md` | Workforce Planning | `wfplanning-service` |
| `75-HR-HELP-DESK.md` | HR Help Desk | `hrhelpdesk-service` |
| `76-WORKFORCE-SAFETY.md` | Workforce Safety | `safety-service` |
| `105-WORKFORCE-LABOR-OPTIMIZATION.md` | Workforce Labor Optimization | `laboropt-service` |
| `111-WORKFORCE-MODELING.md` | Workforce Modeling | `wfmodel-service` |
| `119-STRATEGIC-WORKFORCE-PLANNING.md` | Strategic Workforce Planning | `strategicwf-service` |
| `133-PAYROLL-INTERFACE-CONNECT.md` | Payroll Interface & Connect | `payrollint-service` |
| `166-CONNECTED-WORKER.md` | Connected Worker | `connectedworker-service` |

## HCM - Talent & Experience (18 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `93-OPPORTUNITY-MARKETPLACE.md` | Opportunity Marketplace | `marketplace-service` |
| `94-DYNAMIC-SKILLS.md` | Dynamic Skills | `skills-service` |
| `95-EMPLOYEE-EXPERIENCE.md` | Employee Experience | `experience-service` |
| `96-ONBOARDING.md` | Onboarding & Transitions | `onboarding-service` |
| `106-MANAGER-EDGE.md` | Manager Edge | `manageredge-service` |
| `107-WORK-LIFE.md` | Work Life | `worklife-service` |
| `108-ADVANCED-HCM-CONTROLS.md` | Advanced HCM Controls | `hcmcontrols-service` |
| `109-EXPERIENCE-DESIGN-STUDIO.md` | Experience Design Studio | `exdesign-service` |
| `110-ACTIVITY-CENTERS.md` | Activity Centers | `activity-service` |
| `132-DESKLESS-WORKFORCE.md` | Deskless Workforce | `deskless-service` |
| `146-TALENT-REVIEW.md` | Talent Review | `talentreview-service` |
| `147-ORACLE-JOURNEYS.md` | Oracle Journeys | `journeys-service` |
| `161-WORKFORCE-PREDICTIONS.md` | Workforce Predictions | `wfpredictions-service` |
| `169-GOALS-MANAGEMENT.md` | Goals Management | `goals-service` |
| `170-TOUCHPOINTS.md` | Touchpoints (Continuous Check-ins) | `touchpoints-service` |
| `171-GUIDED-LEARNING.md` | Guided Learning | `guidedlearning-service` |
| `175-ORACLE-GROW.md` | Oracle Grow | `grow-service` |
| `189-HCM-NOW.md` | HCM Now | `hcmnow-service` |

## HCM Analytics (1 spec)

| Spec | Module | Microservice |
|------|--------|-------------|
| `148-HCM-ANALYTICS.md` | HCM Analytics | `hcmanalytics-service` |

---

## Customer Experience - Sales & Service (14 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `77-SALES-AUTOMATION.md` | Sales Force Automation | `sales-service` |
| `78-SALES-PLANNING.md` | Sales Planning | `salesplanning-service` |
| `79-SALES-PERFORMANCE.md` | Sales Performance | `salesperf-service` |
| `80-FIELD-SERVICE.md` | Field Service | `fieldservice-service` |
| `81-CUSTOMER-SERVICE.md` | Customer Service | `customerservice-service` |
| `82-KNOWLEDGE-MANAGEMENT.md` | Knowledge Management | `knowledge-service` |
| `83-SERVICE-LOGISTICS.md` | Service Logistics | `servicelogistics-service` |
| `84-CHANNEL-MANAGEMENT.md` | Channel Management | `channel-service` |
| `126-B2C-SERVICE.md` | B2C Service | `b2cservice-service` |
| `127-INTELLIGENT-ADVISOR.md` | Intelligent Advisor | `advisor-service` |
| `128-LIVE-EXPERIENCE.md` | Live Experience | `liveexp-service` |
| `129-SERVICE-CENTER.md` | Service Center | `svccenter-service` |
| `130-PARTNER-RELATIONSHIP-MANAGEMENT.md` | Partner Relationship Management | `prm-service` |
| `179-SALES-ENGAGEMENT.md` | CX Sales Engagement | `salesengagement-service` |

## CX - Marketing & Commerce (12 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `56-SUBSCRIPTION-MANAGEMENT.md` | Subscription Management | `subscription-service` |
| `57-CPQ.md` | Configure Price Quote | `cpq-service` |
| `58-CONFIGURATOR.md` | Product Configurator | `configurator-service` |
| `59-COMMERCE.md` | Commerce (B2B/B2C) | `commerce-service` |
| `60-CUSTOMER-DATA-PLATFORM.md` | Customer Data Platform | `cdp-service` |
| `61-MARKETING.md` | Marketing Automation | `marketing-service` |
| `97-LOYALTY-MANAGEMENT.md` | Loyalty Management | `loyalty-service` |
| `168-MAXYMISER.md` | Maxymiser (CX Testing) | `maxymiser-service` |
| `180-MDF-MANAGEMENT.md` | MDF Management | `mdf-service` |
| `181-ELOQUA.md` | Eloqua (B2B Marketing) | `eloqua-service` |
| `182-RESPONSYS.md` | Responsys (B2C Marketing) | `responsys-service` |
| `183-CX-ADVERTISING.md` | CX Advertising | `cxadvertising-service` |

## CX Analytics (1 spec)

| Spec | Module | Microservice |
|------|--------|-------------|
| `131-CX-ANALYTICS.md` | CX Analytics | `cxanalytics-service` |

---

## Enterprise Performance Management (11 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `33-EPM.md` | Enterprise Performance Management | `epm-service` |
| `85-TAX-REPORTING.md` | Tax Reporting (ASC 740) | `taxreport-service` |
| `86-NARRATIVE-REPORTING.md` | Narrative Reporting | `narrative-service` |
| `87-ENTERPRISE-DATA-MANAGEMENT.md` | Enterprise Data Management | `edm-service` |
| `88-PROFITABILITY-MANAGEMENT.md` | Profitability Management | `profitability-service` |
| `117-EPM-PLATFORM.md` | EPM Platform | `epmplatform-service` |
| `118-FREEFORM-PLANNING.md` | Freeform Planning (Essbase) | `freeform-service` |
| `152-DISCLOSURE-MANAGEMENT.md` | Disclosure Management | `disclosure-service` |
| `153-SUPPLEMENTAL-DATA.md` | Supplemental Data Management | `supplementaldata-service` |
| `155-SMART-VIEW-OFFICE.md` | Smart View for Office | `smartview-service` |
| `156-EPM-AUTOMATE.md` | EPM Automate | `epmautomate-service` |

---

## Platform & Cross-Cutting (22 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `34-RISK-COMPLIANCE.md` | Risk Management & Compliance | `risk-service` |
| `35-AI-ML-PLATFORM.md` | AI/ML Platform & AI Agents | `ai-service` |
| `40-ENTERPRISE-ASSET.md` | Enterprise Asset Management | `eam-service` |
| `42-SUSTAINABILITY.md` | Sustainability / ESG | `sus-service` |
| `43-DIGITAL-ASSISTANT.md` | Digital Assistant / Conversational AI | `assistant-service` |
| `44-IOT-INTEGRATION.md` | IoT Integration | `iot-service` |
| `45-SUPPLIER-PORTAL.md` | Supplier Portal & Sourcing | `supplier-service` |
| `46-MOBILE.md` | Mobile Application Framework | `mobile-service` |
| `47-BLOCKCHAIN.md` | Blockchain & Digital Thread | `chain-service` |
| `89-INTEGRATION-CLOUD.md` | Integration Cloud (iPaaS) | `integration-service` |
| `90-VISUAL-BUILDER.md` | Visual Builder (Low-Code) | `builder-service` |
| `91-PROCESS-AUTOMATION.md` | Process Automation (RPA) | `automation-service` |
| `92-INNOVATION-MANAGEMENT.md` | Innovation Management | `innovation-service` |
| `102-FUSION-DATA-INTELLIGENCE.md` | Fusion Data Intelligence (Analytics) | `fdi-service` |
| `103-APPLICATION-COMPOSER.md` | Application Composer | `composer-service` |
| `116-BUSINESS-CONTINUITY.md` | Business Continuity Planning | `continuity-service` |
| `134-AI-AGENTS-SUITE.md` | AI Agents Suite | `aiagents-service` |
| `135-DOCUMENT-IO-AGENT.md` | Document IO Agent | `docio-service` |
| `162-PRODUCT-DEVELOPMENT.md` | Product Development | `productdev-service` |
| `163-FUNCTIONAL-SETUP-MANAGER.md` | Functional Setup Manager | `fsm-service` |
| `164-ENTERPRISE-SEARCH.md` | Enterprise Search | `search-service` |
| `165-NOTIFICATION-CENTER.md` | Notification Center | `notification-service` |
| `167-MAINTENANCE-CLOUD.md` | Maintenance Cloud | `maintenance-service` |
| `172-B2B-GATEWAY.md` | B2B Gateway / EDI | `b2bgateway-service` |
| `173-ENTERPRISE-SCHEDULER.md` | Enterprise Scheduler | `scheduler-service` |
| `174-OTBI.md` | OTBI (Transactional BI) | `otbi-service` |
| `176-SPREADSHEET-DESIGNER.md` | Spreadsheet Designer | `spreadsheet-service` |
| `177-ACCESS-GOVERNOR.md` | Access Governor | `accessgov-service` |
| `178-SANDBOX-MANAGEMENT.md` | Sandbox Management | `sandbox-service` |
| `184-CONTENT-MANAGEMENT.md` | Content Management | `content-service` |
| `186-ORACLE-HEALTH.md` | Oracle Health | `oraclehealth-service` |
| `188-SUPPLIER-QUALIFICATION.md` | Supplier Qualification | `supplierqual-service` |
| `190-ORACLE-ME.md` | Oracle ME Platform | `oracleme-service` |
| `191-DATA-RELATIONSHIP-MGMT.md` | Data Relationship Management | `drm-service` |

## Analytics & Reporting (4 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `114-SCM-ANALYTICS.md` | SCM Analytics | `scmanalytics-service` |
| `149-ERP-ANALYTICS.md` | ERP Analytics | `erpanalytics-service` |
| `150-BI-PUBLISHER.md` | BI Publisher | `bipublisher-service` |
| `157-PROCUREMENT-ANALYTICS.md` | Procurement Analytics | `procurementanalytics-service` |

---

## Industry Verticals (20 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `136-HIGHER-EDUCATION.md` | Higher Education | `highered-service` |
| `137-PUBLIC-SECTOR.md` | Public Sector | `publicsector-service` |
| `138-COMMUNICATIONS.md` | Communications (Telecom) | `comms-service` |
| `192-RETAIL.md` | Retail | `retail-service` |
| `193-AUTOMOTIVE.md` | Automotive | `automotive-service` |
| `194-OIL-GAS.md` | Oil & Gas | `oilgas-service` |
| `195-LIFE-SCIENCES.md` | Life Sciences | `lifescience-service` |
| `196-CONSUMER-GOODS.md` | Consumer Goods | `consumergoods-service` |
| `197-HIGH-TECH.md` | High Tech | `hightech-service` |
| `198-WHOLESALE-DISTRIBUTION.md` | Wholesale Distribution | `wholesale-service` |
| `199-TRAVEL-TRANSPORTATION.md` | Travel & Transportation | `travel-service` |
| `200-INDUSTRIAL-MFG.md` | Industrial Manufacturing | `industrialmfg-service` |
| `201-RESTAURANTS.md` | Restaurants | `restaurants-service` |
| `203-CLINICAL-SUPPLY-CHAIN.md` | Clinically Integrated Supply Chain | `clinicalsc-service` |
| `206-CONSTRUCTION-ENGINEERING.md` | Construction & Engineering | `construction-service` |
| `207-FINANCIAL-SERVICES.md` | Financial Services | `financialservices-service` |
| `208-UTILITIES.md` | Utilities | `utilities-service` |
| `209-PROFESSIONAL-SERVICES.md` | Professional Services | `profservices-service` |
| `210-AEROSPACE-DEFENSE.md` | Aerospace & Defense | `aerospace-service` |
| `211-MEDIA-ENTERTAINMENT.md` | Media & Entertainment | `media-service` |

## Migration Tools (2 specs)

| Spec | Module | Microservice |
|------|--------|-------------|
| `204-TALEO-MIGRATION.md` | Taleo Migration | `taleomigration-service` |
| `205-SIEBEL-MIGRATION.md` | Siebel Migration | `siebelmigration-service` |

---

## Summary Statistics

| Domain | Count |
|--------|-------|
| Foundation & Infrastructure | 11 |
| Core Financials | 11 |
| Financial Extensions | 13 |
| Supply Chain - Core | 6 |
| Supply Chain - Extended | 11 |
| Supply Chain - Advanced | 9 |
| Supply Chain - Logistics | 4 |
| Operations & Projects | 8 |
| HCM - Core | 11 |
| HCM - Workforce | 9 |
| HCM - Talent & Experience | 18 |
| HCM Analytics | 1 |
| CX - Sales & Service | 14 |
| CX - Marketing & Commerce | 12 |
| CX Analytics | 1 |
| EPM | 11 |
| Platform & Cross-Cutting | 34 |
| Analytics & Reporting | 4 |
| Industry Verticals | 20 |
| Migration Tools | 2 |
| **Total** | **211** |
