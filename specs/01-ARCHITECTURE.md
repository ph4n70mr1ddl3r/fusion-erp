# 01 - Microservices Architecture Specification

## 1. Architecture Principles

### 1.1 Domain-Driven Design (DDD)
- The system MUST be decomposed into bounded contexts, each mapping to exactly one microservice.
- Each bounded context owns its data, logic, and API surface.
- Ubiquitous language within each context MUST be consistent (e.g., "Invoice" in AP means supplier invoice; in AR it means customer invoice).
- Aggregate roots define transactional boundaries within a service.

### 1.2 Service Autonomy
- Each service MUST be independently deployable, testable, and scalable.
- Services MUST NOT share databases. Data access is only via published APIs (REST or gRPC).
- Services MUST NOT make assumptions about another service's internal implementation.
- Each service owns its SQLite database file: `data/{service_name}.db`.

### 1.3 Eventual Consistency
- Cross-service consistency MUST use eventual consistency, not distributed transactions.
- The Saga pattern MUST be used for business processes spanning multiple services.
- Services MUST be designed to tolerate temporary inconsistency.

### 1.4 Fail-Fast and Resilience
- Services MUST validate input at the boundary (API layer) and fail fast on invalid data.
- Circuit breaker pattern MUST be used for inter-service gRPC calls.
- Services MUST degrade gracefully when dependencies are unavailable.
- Retries with exponential backoff MUST be used for transient failures.

---

## 2. Service Catalog

### 2.1 Service Registry

| Service | Crate | Database | Port | Bounded Context |
|---------|-------|----------|------|-----------------|
| auth-service | `services/auth-service` | `data/auth.db` | 8001 | Identity & Access |
| gateway-service | `services/gateway-service` | none | 8000 | API Gateway |
| gl-service | `services/gl-service` | `data/gl.db` | 8010 | General Ledger |
| ap-service | `services/ap-service` | `data/ap.db` | 8011 | Accounts Payable |
| ar-service | `services/ar-service` | `data/ar.db` | 8012 | Accounts Receivable |
| fa-service | `services/fa-service` | `data/fa.db` | 8013 | Fixed Assets |
| cm-service | `services/cm-service` | `data/cm.db` | 8014 | Cash Management |
| tax-service | `services/tax-service` | `data/tax.db` | 8015 | Tax Management |
| ic-service | `services/ic-service` | `data/ic.db` | 8016 | Intercompany Accounting |
| proc-service | `services/proc-service` | `data/proc.db` | 8020 | Procurement |
| inv-service | `services/inv-service` | `data/inv.db` | 8021 | Inventory |
| om-service | `services/om-service` | `data/om.db` | 8022 | Order Management |
| mfg-service | `services/mfg-service` | `data/mfg.db` | 8023 | Manufacturing |
| expense-service | `services/expense-service` | `data/expense.db` | 8024 | Expense Management |
| rev-service | `services/rev-service` | `data/rev.db` | 8025 | Revenue Management (ASC 606) |
| pricing-service | `services/pricing-service` | `data/pricing.db` | 8026 | Advanced Pricing |
| planning-service | `services/planning-service` | `data/planning.db` | 8027 | MRP & Supply Chain Planning |
| dms-service | `services/dms-service` | `data/dms.db` | 8028 | Document Management |
| etl-service | `services/etl-service` | `data/etl.db` | 8029 | Data Import/Export |
| pm-service | `services/pm-service` | `data/pm.db` | 8030 | Project Management |
| lease-service | `services/lease-service` | `data/lease.db` | 8031 | Lease Accounting |
| collections-service | `services/collections-service` | `data/collections.db` | 8032 | Collections & Credit |
| epm-service | `services/epm-service` | `data/epm.db` | 8060 | Enterprise Performance Management |
| risk-service | `services/risk-service` | `data/risk.db` | 8061 | Risk Management & Compliance |
| ai-service | `services/ai-service` | `data/ai.db` | 8062 | AI/ML Platform & AI Agents |
| wms-service | `services/wms-service` | `data/wms.db` | 8063 | Warehouse Management |
| tms-service | `services/tms-service` | `data/tms.db` | 8064 | Transportation Management |
| plm-service | `services/plm-service` | `data/plm.db` | 8065 | Product Lifecycle Management |
| gtm-service | `services/gtm-service` | `data/gtm.db` | 8066 | Global Trade Management |
| eam-service | `services/eam-service` | `data/eam.db` | 8067 | Enterprise Asset Management |
| quality-service | `services/quality-service` | `data/quality.db` | 8068 | Quality Management |
| sus-service | `services/sus-service` | `data/sus.db` | 8069 | Sustainability / ESG |
| redwood-ux-service | `services/redwood-ux-service` | `data/redwood_ux.db` | 8169 | Redwood UX / Low-Code UI |
| assistant-service | `services/assistant-service` | `data/assistant.db` | 8070 | Digital Assistant / Conversational AI |
| iot-service | `services/iot-service` | `data/iot.db` | 8071 | IoT Integration |
| supplier-service | `services/supplier-service` | `data/supplier.db` | 8072 | Supplier Portal & Sourcing |
| mobile-service | `services/mobile-service` | `data/mobile.db` | 8073 | Mobile Application Framework |
| chain-service | `services/chain-service` | `data/chain.db` | 8074 | Blockchain & Digital Thread |
| costing-service | `services/costing-service` | `data/costing.db` | 8080 | Cost Management |
| accounthub-service | `services/accounthub-service` | `data/accounthub.db` | 8081 | Accounting Hub |
| promising-service | `services/promising-service` | `data/promising.db` | 8082 | Global Order Promising |
| producthub-service | `services/producthub-service` | `data/producthub.db` | 8083 | Product Hub / MDM |
| orchestration-service | `services/orchestration-service` | `data/orchestration.db` | 8084 | Supply Chain Orchestration |
| contracts-service | `services/contracts-service` | `data/contracts.db` | 8085 | Procurement Contracts |
| grant-service | `services/grant-service` | `data/grant.db` | 8086 | Grant Management |
| jv-service | `services/jv-service` | `data/jv.db` | 8087 | Joint Venture Management |
| subscription-service | `services/subscription-service` | `data/subscription.db` | 8088 | Subscription Management |
| cpq-service | `services/cpq-service` | `data/cpq.db` | 8089 | Configure Price Quote |
| configurator-service | `services/configurator-service` | `data/configurator.db` | 8090 | Product Configurator |
| commerce-service | `services/commerce-service` | `data/commerce.db` | 8091 | Commerce (B2B/B2C) |
| cdp-service | `services/cdp-service` | `data/cdp.db` | 8092 | Customer Data Platform |
| marketing-service | `services/marketing-service` | `data/marketing.db` | 8093 | Marketing |
| hr-service | `services/hr-service` | `data/hr.db` | 8094 | Core HR |
| payroll-service | `services/payroll-service` | `data/payroll.db` | 8095 | Payroll |
| timelabor-service | `services/timelabor-service` | `data/timelabor.db` | 8096 | Time & Labor |
| compensation-service | `services/compensation-service` | `data/compensation.db` | 8097 | Compensation |
| benefits-service | `services/benefits-service` | `data/benefits.db` | 8098 | Benefits |
| recruiting-service | `services/recruiting-service` | `data/recruiting.db` | 8099 | Recruiting |
| performance-service | `services/performance-service` | `data/performance.db` | 8100 | Performance Management |
| learning-service | `services/learning-service` | `data/learning.db` | 8101 | Learning & Development |
| succession-service | `services/succession-service` | `data/succession.db` | 8102 | Succession Planning |
| career-service | `services/career-service` | `data/career.db` | 8103 | Career Development |
| absence-service | `services/absence-service` | `data/absence.db` | 8104 | Absence Management |
| scheduling-service | `services/scheduling-service` | `data/scheduling.db` | 8105 | Workforce Scheduling |
| wfplanning-service | `services/wfplanning-service` | `data/wfplanning.db` | 8106 | Workforce Planning |
| hrhelpdesk-service | `services/hrhelpdesk-service` | `data/hrhelpdesk.db` | 8107 | HR Help Desk |
| safety-service | `services/safety-service` | `data/safety.db` | 8108 | Workforce Safety |
| sales-service | `services/sales-service` | `data/sales.db` | 8109 | Sales Force Automation |
| salesplanning-service | `services/salesplanning-service` | `data/salesplanning.db` | 8110 | Sales Planning |
| salesperf-service | `services/salesperf-service` | `data/salesperf.db` | 8111 | Sales Performance |
| fieldservice-service | `services/fieldservice-service` | `data/fieldservice.db` | 8112 | Field Service |
| customerservice-service | `services/customerservice-service` | `data/customerservice.db` | 8113 | Customer Service |
| knowledge-service | `services/knowledge-service` | `data/knowledge.db` | 8114 | Knowledge Management |
| servicelogistics-service | `services/servicelogistics-service` | `data/servicelogistics.db` | 8115 | Service Logistics |
| channel-service | `services/channel-service` | `data/channel.db` | 8116 | Channel Management |
| taxreport-service | `services/taxreport-service` | `data/taxreport.db` | 8117 | Tax Reporting |
| narrative-service | `services/narrative-service` | `data/narrative.db` | 8118 | Narrative Reporting |
| edm-service | `services/edm-service` | `data/edm.db` | 8119 | Enterprise Data Management |
| profitability-service | `services/profitability-service` | `data/profitability.db` | 8120 | Profitability Management |
| integration-service | `services/integration-service` | `data/integration.db` | 8121 | Integration Cloud |
| builder-service | `services/builder-service` | `data/builder.db` | 8122 | Visual Builder |
| automation-service | `services/automation-service` | `data/automation.db` | 8123 | Process Automation |
| innovation-service | `services/innovation-service` | `data/innovation.db` | 8124 | Innovation Management |
| workflow-service | `services/workflow-service` | `data/workflow.db` | 8040 | Workflow & Approvals |
| report-service | `services/report-service` | `data/report.db` | 8050 | Reporting & Analytics |
| consolidation-service | `services/consolidation-service` | `data/consolidation.db` | 8137 | Financial Consolidation |
| reconciliation-service | `services/reconciliation-service` | `data/reconciliation.db` | 8138 | Account Reconciliation |
| fdi-service | `services/fdi-service` | `data/fdi.db` | 8139 | Fusion Data Intelligence |
| composer-service | `services/composer-service` | `data/composer.db` | 8140 | Application Composer |
| federal-service | `services/federal-service` | `data/federal.db` | 8141 | Federal Financials |
| laboropt-service | `services/laboropt-service` | `data/laboropt.db` | 8142 | Workforce Labor Optimization |
| manageredge-service | `services/manageredge-service` | `data/manageredge.db` | 8143 | Manager Edge |
| worklife-service | `services/worklife-service` | `data/worklife.db` | 8144 | Work-Life |
| hcmcontrols-service | `services/hcmcontrols-service` | `data/hcmcontrols.db` | 8145 | Advanced HCM Controls |
| exdesign-service | `services/exdesign-service` | `data/exdesign.db` | 8146 | Experience Design Studio |
| activity-service | `services/activity-service` | `data/activity.db` | 8147 | Activity Centers |
| wfmodel-service | `services/wfmodel-service` | `data/wfmodel.db` | 8148 | Workforce Modeling |
| digitalcs-service | `services/digitalcs-service` | `data/digitalcs.db` | 8149 | Digital Customer Service |
| projectmfg-service | `services/projectmfg-service` | `data/projectmfg.db` | 8150 | Project-Driven Manufacturing |
| scmanalytics-service | `services/scmanalytics-service` | `data/scmanalytics.db` | 8151 | SCM Analytics |
| projectasset-service | `services/projectasset-service` | `data/projectasset.db` | 8152 | Project Asset Management |
| continuity-service | `services/continuity-service` | `data/continuity.db` | 8153 | Business Continuity |
| epmplatform-service | `services/epmplatform-service` | `data/epmplatform.db` | 8154 | EPM Platform |
| freeform-service | `services/freeform-service` | `data/freeform.db` | 8155 | Freeform Planning |
| strategicwf-service | `services/strategicwf-service` | `data/strategicwf.db` | 8156 | Strategic Workforce Planning |
| selfproc-service | `services/selfproc-service` | `data/selfproc.db` | 8200 | Self-Service Procurement |
| processmfg-service | `services/processmfg-service` | `data/processmfg.db` | 8201 | Process Manufacturing |
| prodsched-service | `services/prodsched-service` | `data/prodsched.db` | 8202 | Production Scheduling |
| smartops-service | `services/smartops-service` | `data/smartops.db` | 8203 | Smart Operations |
| sourcing-service | `services/sourcing-service` | `data/sourcing.db` | 8204 | Sourcing |
| scexec-service | `services/scexec-service` | `data/scexec.db` | 8205 | Supply Chain Execution |
| b2cservice-service | `services/b2cservice-service` | `data/b2cservice.db` | 8206 | B2C Service |
| advisor-service | `services/advisor-service` | `data/advisor.db` | 8207 | Intelligent Advisor |
| liveexp-service | `services/liveexp-service` | `data/liveexp.db` | 8208 | Live Experience |
| svccenter-service | `services/svccenter-service` | `data/svccenter.db` | 8209 | Service Center |
| prm-service | `services/prm-service` | `data/prm.db` | 8210 | Partner Relationship Management |
| cxanalytics-service | `services/cxanalytics-service` | `data/cxanalytics.db` | 8211 | CX Analytics |
| deskless-service | `services/deskless-service` | `data/deskless.db` | 8212 | Deskless Workforce |
| payrollint-service | `services/payrollint-service` | `data/payrollint.db` | 8213 | Payroll Interface Connect |
| aiagents-service | `services/aiagents-service` | `data/aiagents.db` | 8214 | AI Agents Suite |
| docio-service | `services/docio-service` | `data/docio.db` | 8215 | Document IO Agent |
| highered-service | `services/highered-service` | `data/highered.db` | 8216 | Higher Education |
| publicsector-service | `services/publicsector-service` | `data/publicsector.db` | 8217 | Public Sector |
| comms-service | `services/comms-service` | `data/comms.db` | 8218 | Communications |
| supplier_mgmt-service | `services/supplier_mgmt-service` | `data/supplier_mgmt.db` | 8219 | Supplier Management |
| demand_mgmt-service | `services/demand_mgmt-service` | `data/demand_mgmt.db` | 8220 | Demand Management |
| sop-service | `services/sop-service` | `data/sop.db` | 8221 | Sales Operations Planning |
| sc_collaboration-service | `services/sc_collaboration-service` | `data/sc_collaboration.db` | 8222 | Supply Chain Collaboration |
| backlog_mgmt-service | `services/backlog_mgmt-service` | `data/backlog_mgmt.db` | 8223 | Backlog Management |
| mixed_mode_mfg-service | `services/mixed_mode_mfg-service` | `data/mixed_mode_mfg.db` | 8224 | Mixed-Mode Manufacturing |
| shipping_exec-service | `services/shipping_exec-service` | `data/shipping_exec.db` | 8225 | Shipping Execution |
| talent_review-service | `services/talent_review-service` | `data/talent_review.db` | 8164 | Talent Review |
| journeys-service | `services/journeys-service` | `data/journeys.db` | 8165 | Oracle Journeys |
| hcm_analytics-service | `services/hcm_analytics-service` | `data/hcm_analytics.db` | 8166 | HCM Analytics |
| erp_analytics-service | `services/erp_analytics-service` | `data/erp_analytics.db` | 8167 | ERP Analytics |
| bi_publisher-service | `services/bi_publisher-service` | `data/bi_publisher.db` | 8168 | BI Publisher |
| redwood-ux2-service | `services/redwood-ux2-service` | `data/redwood_ux2.db` | 8169 | Redwood UX (Spec 151) |
| disclosure_mgmt-service | `services/disclosure_mgmt-service` | `data/disclosure_mgmt.db` | 8170 | Disclosure Management |
| supplemental_data-service | `services/supplemental_data-service` | `data/supplemental_data.db` | 8171 | Supplemental Data |
| financial_reporting-service | `services/financial_reporting-service` | `data/financial_reporting.db` | 8172 | Financial Reporting |
| smartview-service | `services/smartview-service` | `data/smartview.db` | 8173 | Smart View Office |
| epm_automate-service | `services/epm_automate-service` | `data/epm_automate.db` | 8174 | EPM Automate |
| proc_analytics-service | `services/proc_analytics-service` | `data/proc_analytics.db` | 8175 | Procurement Analytics |
| project_costing-service | `services/project_costing-service` | `data/project_costing.db` | 8176 | Project Costing |
| project_billing-service | `services/project_billing-service` | `data/project_billing.db` | 8177 | Project Billing |
| resource_mgmt-service | `services/resource_mgmt-service` | `data/resource_mgmt.db` | 8178 | Resource Management |
| workforce_predictions-service | `services/workforce_predictions-service` | `data/workforce_predictions.db` | 8179 | Workforce Predictions |
| product_dev-service | `services/product_dev-service` | `data/product_dev.db` | 8180 | Product Development |
| fsm-service | `services/fsm-service` | `data/fsm.db` | 8181 | Functional Setup Manager |
| enterprise_search-service | `services/enterprise_search-service` | `data/enterprise_search.db` | 8182 | Enterprise Search |
| notification-service | `services/notification-service` | `data/notification.db` | 8183 | Notification Center |
| connected_worker-service | `services/connected_worker-service` | `data/connected_worker.db` | 8184 | Connected Worker |
| maintenance-service | `services/maintenance-service` | `data/maintenance.db` | 8185 | Maintenance Cloud |
| maxymiser-service | `services/maxymiser-service` | `data/maxymiser.db` | 8186 | Maxymiser |
| goals-service | `services/goals-service` | `data/goals.db` | 8187 | Goals Management |
| touchpoints-service | `services/touchpoints-service` | `data/touchpoints.db` | 8188 | Touchpoints |
| guided_learning-service | `services/guided_learning-service` | `data/guided_learning.db` | 8189 | Guided Learning |
| b2b_gateway-service | `services/b2b_gateway-service` | `data/b2b_gateway.db` | 8190 | B2B Gateway |
| scheduler-service | `services/scheduler-service` | `data/scheduler.db` | 8191 | Enterprise Scheduler |
| otbi-service | `services/otbi-service` | `data/otbi.db` | 8192 | OTBI |
| grow-service | `services/grow-service` | `data/grow.db` | 8193 | Oracle Grow |
| spreadsheet_designer-service | `services/spreadsheet_designer-service` | `data/spreadsheet_designer.db` | 8194 | Spreadsheet Designer |
| access_governor-service | `services/access_governor-service` | `data/access_governor.db` | 8195 | Access Governor |
| sandbox-service | `services/sandbox-service` | `data/sandbox.db` | 8196 | Sandbox Management |
| sales_engagement-service | `services/sales_engagement-service` | `data/sales_engagement.db` | 8197 | Sales Engagement |
| mdf-service | `services/mdf-service` | `data/mdf.db` | 8198 | MDF Management |
| eloqua-service | `services/eloqua-service` | `data/eloqua.db` | 8199 | Eloqua |
| responsys-service | `services/responsys-service` | `data/responsys.db` | 8200 | Responsys |
| cx_advertising-service | `services/cx_advertising-service` | `data/cx_advertising.db` | 8201 | CX Advertising |
| content_management-service | `services/content_management-service` | `data/content_management.db` | 8202 | Content Management |
| planning_budgeting-service | `services/planning_budgeting-service` | `data/planning_budgeting.db` | 8203 | Planning & Budgeting |
| oracle_health-service | `services/oracle_health-service` | `data/oracle_health.db` | 8204 | Oracle Health |
| calculation_manager-service | `services/calculation_manager-service` | `data/calculation_manager.db` | 8205 | Calculation Manager |
| supplier_qualification-service | `services/supplier_qualification-service` | `data/supplier_qualification.db` | 8206 | Supplier Qualification |
| hcm_now-service | `services/hcm_now-service` | `data/hcm_now.db` | 8207 | HCM Now |
| oracle_me-service | `services/oracle_me-service` | `data/oracle_me.db` | 8208 | Oracle Me |
| drm-service | `services/drm-service` | `data/drm.db` | 8209 | Data Relationship Management |
| retail-service | `services/retail-service` | `data/retail.db` | 8210 | Retail |
| automotive-service | `services/automotive-service` | `data/automotive.db` | 8211 | Automotive |
| oil_gas-service | `services/oil_gas-service` | `data/oil_gas.db` | 8212 | Oil & Gas |
| life_sciences-service | `services/life_sciences-service` | `data/life_sciences.db` | 8213 | Life Sciences |
| consumer_goods-service | `services/consumer_goods-service` | `data/consumer_goods.db` | 8214 | Consumer Goods |
| high_tech-service | `services/high_tech-service` | `data/high_tech.db` | 8215 | High Tech |
| wholesale_distribution-service | `services/wholesale_distribution-service` | `data/wholesale_distribution.db` | 8216 | Wholesale Distribution |
| travel_transportation-service | `services/travel_transportation-service` | `data/travel_transportation.db` | 8217 | Travel & Transportation |
| industrial_mfg-service | `services/industrial_mfg-service` | `data/industrial_mfg.db` | 8218 | Industrial Manufacturing |
| restaurants-service | `services/restaurants-service` | `data/restaurants.db` | 8219 | Restaurants |
| fusion_data_intelligence-service | `services/fusion_data_intelligence-service` | `data/fusion_data_intelligence.db` | 8220 | Fusion Data Intelligence (Spec 202) |
| clinical_supply_chain-service | `services/clinical_supply_chain-service` | `data/clinical_supply_chain.db` | 8221 | Clinical Supply Chain |
| taleo_migration-service | `services/taleo_migration-service` | `data/taleo_migration.db` | 8222 | Taleo Migration |
| siebel_migration-service | `services/siebel_migration-service` | `data/siebel_migration.db` | 8223 | Siebel Migration |
| construction_engineering-service | `services/construction_engineering-service` | `data/construction_engineering.db` | 8224 | Construction & Engineering |
| financial_services-service | `services/financial_services-service` | `data/financial_services.db` | 8225 | Financial Services |
| utilities-service | `services/utilities-service` | `data/utilities.db` | 8226 | Utilities |
| professional_services-service | `services/professional_services-service` | `data/professional_services.db` | 8227 | Professional Services |
| aerospace_defense-service | `services/aerospace_defense-service` | `data/aerospace_defense.db` | 8228 | Aerospace & Defense |
| media_entertainment-service | `services/media_entertainment-service` | `data/media_entertainment.db` | 8229 | Media & Entertainment |

### 2.2 Service Responsibilities

#### auth-service (Port 8001)
- User authentication (login, logout, token refresh)
- User CRUD and credential management
- Role and permission management
- JWT token issuance and validation
- API key management for service-to-service auth
- Audit trail for all security events

#### gateway-service (Port 8000)
- Single external entry point (reverse proxy)
- JWT validation and tenant extraction
- Request routing to backend services
- Rate limiting (per tenant)
- Request/response logging
- CORS handling
- API versioning

#### gl-service (Port 8010)
- Chart of accounts management (multi-segment)
- Journal entry creation, editing, posting
- Period management (open, close, reopen)
- Currency translation and revaluation
- Budget management
- Balance calculation and storage
- Trial balance, balance sheet, income statement data

#### ap-service (Port 8011)
- Supplier management
- Invoice entry (header + lines)
- Invoice approval workflow
- Payment processing and selection
- Payment matching (invoice ↔ payment)
- Aging reports
- GL integration (auto-generate journal entries)

#### ar-service (Port 8012)
- Customer management
- Invoice and memo creation
- Receipt processing and application
- Credit memo management
- Collections and dunning
- Aging reports
- GL integration (auto-generate journal entries)

#### fa-service (Port 8013)
- Asset registration and categorization
- Depreciation calculation (straight-line, declining balance, units of production)
- Asset transfers and reclassifications
- Asset retirement and disposal
- Depreciation posting to GL
- Asset reporting

#### cm-service (Port 8014)
- Bank account management
- Cash receipt and disbursement recording
- Bank statement import and reconciliation
- Cash positioning and forecasting
- GL integration for all cash transactions

#### proc-service (Port 8020)
- Requisition creation and approval
- Purchase order management
- Supplier catalog management
- Goods receipt and inspection
- Three-way match (PO ↔ Receipt ↔ Invoice)
- Blanket agreements and contracts

#### inv-service (Port 8021)
- Item master management
- Warehouse and location management
- On-hand quantity tracking
- Stock movements (receipts, issues, transfers)
- Costing (FIFO, LIFO, weighted average, standard)
- Cycle counting and physical inventory
- Reorder point management

#### om-service (Port 8022)
- Sales order management
- Order fulfillment and allocation
- Shipping and delivery tracking
- Return management (RMA)
- Pricing and discount management
- Credit check integration
- Invoice generation integration with AR

#### mfg-service (Port 8023)
- Bill of Materials (BOM) management
- Work order creation and scheduling
- Routing and operations management
- Production reporting (material consumption, labor, output)
- Quality inspection
- Cost roll-up
- Integration with inventory for material movements

#### pm-service (Port 8030)
- Project definition and structuring (WBS)
- Task management and scheduling
- Resource allocation
- Time and expense entry
- Project costing and billing
- Revenue recognition
- Integration with GL for project accounting

#### workflow-service (Port 8040)
- Approval workflow definition
- Rule-based routing (amount thresholds, role-based)
- Parallel and sequential approval chains
- Delegation and substitution
- Notification management (email, in-app)
- Workflow history and audit

#### report-service (Port 8050)
- Financial report generation
- Operational dashboard data
- Ad-hoc query builder
- Report scheduling and distribution
- Data export (CSV, PDF, Excel)
- Real-time metrics aggregation

#### epm-service (Port 8060)
- Planning scenarios and budget versions
- Financial consolidation (multi-entity, multi-currency)
- Account reconciliation and close management
- Cost allocation engine
- Budget vs. actual analysis

#### risk-service (Port 8061)
- Separation of Duties (SoD) rules and violation detection
- User access certification campaigns
- Audit policy management and event logging
- Transaction control and fraud detection
- Compliance reporting (SOX, GDPR, etc.)

#### ai-service (Port 8062)
- AI agent framework (document recognition, anomaly detection, forecasting)
- ML model registry, training, and deployment
- Natural language query processing
- Intelligent document recognition (OCR)
- Predictive analytics and anomaly detection

#### wms-service (Port 8063)
- Warehouse, zone, and location management
- Wave planning and execution
- Task management (pick, put, replenish, pack, ship)
- Inbound receiving and putaway
- Outbound shipping and cross-docking

#### tms-service (Port 8064)
- Carrier management and rate shopping
- Shipment planning and booking
- Load planning and optimization
- Freight audit and payment
- Track and trace with real-time visibility

#### plm-service (Port 8065)
- Product lifecycle management (concept to obsolescence)
- Engineering BOM and change management (ECR/ECN)
- Product configurator (rule-based configuration)
- Innovation management and idea scoring
- Supplier design collaboration

#### gtm-service (Port 8066)
- Restricted party screening
- Product classification (HS codes, ECCN)
- Import/export license management
- Customs declarations and filing
- Landed cost simulation

#### eam-service (Port 8067)
- Operating asset lifecycle management
- Preventive and predictive maintenance scheduling
- Maintenance work order management
- Asset failure tracking and root cause analysis
- Safety permits and lockout/tagout

#### quality-service (Port 8068)
- Inspection plan management (incoming, in-process, final)
- Non-conformance reporting (NCR)
- Corrective and preventive action (CAPA)
- Statistical process control (SPC)
- Quality certificate management (CoA, CoC)

#### sus-service (Port 8069)
- Carbon emissions tracking (Scope 1, 2, 3)
- Energy, water, and waste management
- ESG data collection and reporting (GRI, SASB, TCFD, CSRD)
- Sustainability targets and net-zero trajectory
- Supplier ESG assessment

#### assistant-service (Port 8070)
- Conversational AI interface (natural language)
- Proactive notification management
- Quick action shortcuts
- Approval shortcuts (mobile/assistant)
- Multi-channel support (web, mobile, Slack, Teams)

#### iot-service (Port 8071)
- IoT device registration and management
- Telemetry data ingestion and aggregation
- Alert rule engine for threshold violations
- Device command and control
- Edge computing configuration

#### supplier-service (Port 8072)
- Supplier self-registration and qualification
- Sourcing events (RFI, RFQ, RFP, auction)
- Supplier bid management
- Supplier performance scorecards
- Punchout/hosted catalog management

#### mobile-service (Port 8073)
- Device registration and management
- Push notification delivery
- Offline data sync with conflict resolution
- Mobile screen configuration
- Barcode/QR/NFC scanning

#### chain-service (Port 8074)
- Blockchain network and smart contract management
- Asset tokenization and provenance tracking
- Certification anchoring on blockchain
- Supply chain digital thread
- Third-party verification requests

---

## 3. Communication Patterns

### 3.1 External Communication (Client → Gateway → Service)
- Protocol: HTTP/1.1 or HTTP/2 over TLS
- Format: JSON (application/json)
- Entry point: gateway-service on port 8000
- Gateway proxies requests to internal services via HTTP
- All external requests MUST go through the gateway

```
Client → HTTPS → Gateway (:8000) → HTTP → Service (:80XX)
```

### 3.2 Internal Communication (Service ↔ Service)
- Protocol: gRPC over HTTP/2
- Format: Protocol Buffers (proto3)
- Direct service-to-service calls using tonic client
- Service addresses resolved from configuration (no service mesh required)

```
ap-service → gRPC → gl-service
ar-service → GRS → inv-service
```

### 3.3 Asynchronous Communication (Event Bus)
- In-process event bus using tokio broadcast channels
- Events published to a central event dispatcher
- Services subscribe to events they care about
- Events are durable — stored in a local event log table before dispatch

**Event Flow:**
```
Publisher Service → Event Log Table → tokio::broadcast → Subscriber Services
```

**Standard Event Envelope:**
```json
{
  "event_id": "uuid-v7",
  "event_type": "journal.posted",
  "source_service": "gl-service",
  "tenant_id": "tenant-uuid",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlation_id": "request-uuid",
  "payload": { ... }
}
```

### 3.4 Saga Pattern for Distributed Transactions

**Orchestration-style Sagas:**
- The initiating service acts as the saga coordinator.
- Each step invokes the next service via gRPC.
- On failure, the coordinator triggers compensating transactions in reverse order.

**Example: Purchase Order → Goods Receipt → Invoice → Payment → GL Entries**

```
proc-service: Create PO
  → inv-service: Reserve stock (on PO approval)
  → inv-service: Receive goods (on goods receipt)
  → ap-service: Create invoice (on invoice receipt)
  → ap-service: Create payment (on payment run)
  → gl-service: Post journal entries (on each step)
```

If payment fails:
1. ap-service: Reverse payment
2. ap-service: Reverse invoice (or leave for manual resolution)
3. GL entries remain (they represent what happened)

**Saga State Table (per service):**
```sql
CREATE TABLE saga_instances (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    saga_type TEXT NOT NULL,
    current_step INTEGER NOT NULL,
    status TEXT NOT NULL CHECK(status IN ('RUNNING','COMPLETED','COMPENSATING','FAILED')),
    payload TEXT NOT NULL,  -- JSON
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

---

## 4. Gateway Pattern

### 4.1 Request Routing
- Gateway maintains a routing table mapping URL prefixes to backend services.
- Route configuration loaded from `gateway-routes.toml`.

```toml
[routes.gl]
prefix = "/api/v1/gl"
target = "http://localhost:8010"

[routes.ap]
prefix = "/api/v1/ap"
target = "http://localhost:8011"

[routes.ar]
prefix = "/api/v1/ar"
target = "http://localhost:8012"

[routes.epm]
prefix = "/api/v1/epm"
target = "http://localhost:8060"

[routes.risk]
prefix = "/api/v1/risk"
target = "http://localhost:8061"

[routes.ai]
prefix = "/api/v1/ai"
target = "http://localhost:8062"

[routes.wms]
prefix = "/api/v1/wms"
target = "http://localhost:8063"

[routes.tms]
prefix = "/api/v1/tms"
target = "http://localhost:8064"

[routes.plm]
prefix = "/api/v1/plm"
target = "http://localhost:8065"

[routes.gtm]
prefix = "/api/v1/gtm"
target = "http://localhost:8066"

[routes.eam]
prefix = "/api/v1/eam"
target = "http://localhost:8067"

[routes.quality]
prefix = "/api/v1/quality"
target = "http://localhost:8068"

[routes.sus]
prefix = "/api/v1/sus"
target = "http://localhost:8069"

[routes.assistant]
prefix = "/api/v1/assistant"
target = "http://localhost:8070"

[routes.iot]
prefix = "/api/v1/iot"
target = "http://localhost:8071"

[routes.supplier]
prefix = "/api/v1/supplier"
target = "http://localhost:8072"

[routes.mobile]
prefix = "/api/v1/mobile"
target = "http://localhost:8073"

[routes.chain]
prefix = "/api/v1/chain"
target = "http://localhost:8074"

[routes.redwood-ux]
prefix = "/api/v1/redwood-ux"
target = "http://localhost:8169"

[routes.consolidation]
prefix = "/api/v1/consolidation"
target = "http://localhost:8137"

[routes.reconciliation]
prefix = "/api/v1/reconciliation"
target = "http://localhost:8138"

[routes.fdi]
prefix = "/api/v1/fdi"
target = "http://localhost:8139"

[routes.composer]
prefix = "/api/v1/composer"
target = "http://localhost:8140"

[routes.federal]
prefix = "/api/v1/federal"
target = "http://localhost:8141"

[routes.laboropt]
prefix = "/api/v1/laboropt"
target = "http://localhost:8142"

[routes.manageredge]
prefix = "/api/v1/manageredge"
target = "http://localhost:8143"

[routes.worklife]
prefix = "/api/v1/worklife"
target = "http://localhost:8144"

[routes.hcmcontrols]
prefix = "/api/v1/hcmcontrols"
target = "http://localhost:8145"

[routes.exdesign]
prefix = "/api/v1/exdesign"
target = "http://localhost:8146"

[routes.activity]
prefix = "/api/v1/activity"
target = "http://localhost:8147"

[routes.wfmodel]
prefix = "/api/v1/wfmodel"
target = "http://localhost:8148"

[routes.digitalcs]
prefix = "/api/v1/digitalcs"
target = "http://localhost:8149"

[routes.projectmfg]
prefix = "/api/v1/projectmfg"
target = "http://localhost:8150"

[routes.scmanalytics]
prefix = "/api/v1/scmanalytics"
target = "http://localhost:8151"

[routes.projectasset]
prefix = "/api/v1/projectasset"
target = "http://localhost:8152"

[routes.continuity]
prefix = "/api/v1/continuity"
target = "http://localhost:8153"

[routes.epmplatform]
prefix = "/api/v1/epmplatform"
target = "http://localhost:8154"

[routes.freeform]
prefix = "/api/v1/freeform"
target = "http://localhost:8155"

[routes.strategicwf]
prefix = "/api/v1/strategicwf"
target = "http://localhost:8156"

[routes.selfproc]
prefix = "/api/v1/selfproc"
target = "http://localhost:8200"

[routes.processmfg]
prefix = "/api/v1/processmfg"
target = "http://localhost:8201"

[routes.prodsched]
prefix = "/api/v1/prodsched"
target = "http://localhost:8202"

[routes.smartops]
prefix = "/api/v1/smartops"
target = "http://localhost:8203"

[routes.sourcing]
prefix = "/api/v1/sourcing"
target = "http://localhost:8204"

[routes.scexec]
prefix = "/api/v1/scexec"
target = "http://localhost:8205"

[routes.b2cservice]
prefix = "/api/v1/b2cservice"
target = "http://localhost:8206"

[routes.advisor]
prefix = "/api/v1/advisor"
target = "http://localhost:8207"

[routes.liveexp]
prefix = "/api/v1/liveexp"
target = "http://localhost:8208"

[routes.svccenter]
prefix = "/api/v1/svccenter"
target = "http://localhost:8209"

[routes.prm]
prefix = "/api/v1/prm"
target = "http://localhost:8210"

[routes.cxanalytics]
prefix = "/api/v1/cxanalytics"
target = "http://localhost:8211"

[routes.deskless]
prefix = "/api/v1/deskless"
target = "http://localhost:8212"

[routes.payrollint]
prefix = "/api/v1/payrollint"
target = "http://localhost:8213"

[routes.aiagents]
prefix = "/api/v1/aiagents"
target = "http://localhost:8214"

[routes.docio]
prefix = "/api/v1/docio"
target = "http://localhost:8215"

[routes.highered]
prefix = "/api/v1/highered"
target = "http://localhost:8216"

[routes.publicsector]
prefix = "/api/v1/publicsector"
target = "http://localhost:8217"

[routes.comms]
prefix = "/api/v1/comms"
target = "http://localhost:8218"

[routes.supplier-mgmt]
prefix = "/api/v1/supplier-mgmt"
target = "http://localhost:8219"

[routes.demand-mgmt]
prefix = "/api/v1/demand-mgmt"
target = "http://localhost:8220"

[routes.sop]
prefix = "/api/v1/sop"
target = "http://localhost:8221"

[routes.sc-collaboration]
prefix = "/api/v1/sc-collaboration"
target = "http://localhost:8222"

[routes.backlog-mgmt]
prefix = "/api/v1/backlog-mgmt"
target = "http://localhost:8223"

[routes.mixed-mode-mfg]
prefix = "/api/v1/mixed-mode-mfg"
target = "http://localhost:8224"

[routes.shipping-exec]
prefix = "/api/v1/shipping-exec"
target = "http://localhost:8225"

[routes.talent-review]
prefix = "/api/v1/talent-review"
target = "http://localhost:8164"

[routes.journeys]
prefix = "/api/v1/journeys"
target = "http://localhost:8165"

[routes.hcm-analytics]
prefix = "/api/v1/hcm-analytics"
target = "http://localhost:8166"

[routes.erp-analytics]
prefix = "/api/v1/erp-analytics"
target = "http://localhost:8167"

[routes.bi-publisher]
prefix = "/api/v1/bi-publisher"
target = "http://localhost:8168"

[routes.redwood-ux2]
prefix = "/api/v1/redwood-ux2"
target = "http://localhost:8169"

[routes.disclosure-mgmt]
prefix = "/api/v1/disclosure-mgmt"
target = "http://localhost:8170"

[routes.supplemental-data]
prefix = "/api/v1/supplemental-data"
target = "http://localhost:8171"

[routes.financial-reporting]
prefix = "/api/v1/financial-reporting"
target = "http://localhost:8172"

[routes.smartview]
prefix = "/api/v1/smartview"
target = "http://localhost:8173"

[routes.epm-automate]
prefix = "/api/v1/epm-automate"
target = "http://localhost:8174"

[routes.proc-analytics]
prefix = "/api/v1/proc-analytics"
target = "http://localhost:8175"

[routes.project-costing]
prefix = "/api/v1/project-costing"
target = "http://localhost:8176"

[routes.project-billing]
prefix = "/api/v1/project-billing"
target = "http://localhost:8177"

[routes.resource-mgmt]
prefix = "/api/v1/resource-mgmt"
target = "http://localhost:8178"

[routes.workforce-predictions]
prefix = "/api/v1/workforce-predictions"
target = "http://localhost:8179"

[routes.product-dev]
prefix = "/api/v1/product-dev"
target = "http://localhost:8180"

[routes.fsm]
prefix = "/api/v1/fsm"
target = "http://localhost:8181"

[routes.enterprise-search]
prefix = "/api/v1/enterprise-search"
target = "http://localhost:8182"

[routes.notification]
prefix = "/api/v1/notification"
target = "http://localhost:8183"

[routes.connected-worker]
prefix = "/api/v1/connected-worker"
target = "http://localhost:8184"

[routes.maintenance]
prefix = "/api/v1/maintenance"
target = "http://localhost:8185"

[routes.maxymiser]
prefix = "/api/v1/maxymiser"
target = "http://localhost:8186"

[routes.goals]
prefix = "/api/v1/goals"
target = "http://localhost:8187"

[routes.touchpoints]
prefix = "/api/v1/touchpoints"
target = "http://localhost:8188"

[routes.guided-learning]
prefix = "/api/v1/guided-learning"
target = "http://localhost:8189"

[routes.b2b-gateway]
prefix = "/api/v1/b2b-gateway"
target = "http://localhost:8190"

[routes.scheduler]
prefix = "/api/v1/scheduler"
target = "http://localhost:8191"

[routes.otbi]
prefix = "/api/v1/otbi"
target = "http://localhost:8192"

[routes.grow]
prefix = "/api/v1/grow"
target = "http://localhost:8193"

[routes.spreadsheet-designer]
prefix = "/api/v1/spreadsheet-designer"
target = "http://localhost:8194"

[routes.access-governor]
prefix = "/api/v1/access-governor"
target = "http://localhost:8195"

[routes.sandbox]
prefix = "/api/v1/sandbox"
target = "http://localhost:8196"

[routes.sales-engagement]
prefix = "/api/v1/sales-engagement"
target = "http://localhost:8197"

[routes.mdf]
prefix = "/api/v1/mdf"
target = "http://localhost:8198"

[routes.eloqua]
prefix = "/api/v1/eloqua"
target = "http://localhost:8199"

[routes.responsys]
prefix = "/api/v1/responsys"
target = "http://localhost:8200"

[routes.cx-advertising]
prefix = "/api/v1/cx-advertising"
target = "http://localhost:8201"

[routes.content-management]
prefix = "/api/v1/content-management"
target = "http://localhost:8202"

[routes.planning-budgeting]
prefix = "/api/v1/planning-budgeting"
target = "http://localhost:8203"

[routes.oracle-health]
prefix = "/api/v1/oracle-health"
target = "http://localhost:8204"

[routes.calculation-manager]
prefix = "/api/v1/calculation-manager"
target = "http://localhost:8205"

[routes.supplier-qualification]
prefix = "/api/v1/supplier-qualification"
target = "http://localhost:8206"

[routes.hcm-now]
prefix = "/api/v1/hcm-now"
target = "http://localhost:8207"

[routes.oracle-me]
prefix = "/api/v1/oracle-me"
target = "http://localhost:8208"

[routes.drm]
prefix = "/api/v1/drm"
target = "http://localhost:8209"

[routes.retail]
prefix = "/api/v1/retail"
target = "http://localhost:8210"

[routes.automotive]
prefix = "/api/v1/automotive"
target = "http://localhost:8211"

[routes.oil-gas]
prefix = "/api/v1/oil-gas"
target = "http://localhost:8212"

[routes.life-sciences]
prefix = "/api/v1/life-sciences"
target = "http://localhost:8213"

[routes.consumer-goods]
prefix = "/api/v1/consumer-goods"
target = "http://localhost:8214"

[routes.high-tech]
prefix = "/api/v1/high-tech"
target = "http://localhost:8215"

[routes.wholesale-distribution]
prefix = "/api/v1/wholesale-distribution"
target = "http://localhost:8216"

[routes.travel-transportation]
prefix = "/api/v1/travel-transportation"
target = "http://localhost:8217"

[routes.industrial-mfg]
prefix = "/api/v1/industrial-mfg"
target = "http://localhost:8218"

[routes.restaurants]
prefix = "/api/v1/restaurants"
target = "http://localhost:8219"

[routes.fusion-data-intelligence]
prefix = "/api/v1/fusion-data-intelligence"
target = "http://localhost:8220"

[routes.clinical-supply-chain]
prefix = "/api/v1/clinical-supply-chain"
target = "http://localhost:8221"

[routes.taleo-migration]
prefix = "/api/v1/taleo-migration"
target = "http://localhost:8222"

[routes.siebel-migration]
prefix = "/api/v1/siebel-migration"
target = "http://localhost:8223"

[routes.construction-engineering]
prefix = "/api/v1/construction-engineering"
target = "http://localhost:8224"

[routes.financial-services]
prefix = "/api/v1/financial-services"
target = "http://localhost:8225"

[routes.utilities]
prefix = "/api/v1/utilities"
target = "http://localhost:8226"

[routes.professional-services]
prefix = "/api/v1/professional-services"
target = "http://localhost:8227"

[routes.aerospace-defense]
prefix = "/api/v1/aerospace-defense"
target = "http://localhost:8228"

[routes.media-entertainment]
prefix = "/api/v1/media-entertainment"
target = "http://localhost:8229"
```

### 4.2 Gateway Middleware Stack (ordered)
1. **Rate Limiting** — per-tenant, configurable limits
2. **CORS** — allowed origins from configuration
3. **Request ID** — generate UUID if not present, pass via X-Request-Id
4. **Authentication** — validate JWT, extract user_id, tenant_id, roles
5. **Tenant Resolution** — verify tenant exists and is active
6. **Request Logging** — log method, path, user, tenant
7. **Proxy** — forward to backend service
8. **Response Logging** — log status, duration
9. **Error Handling** — standardize error responses

### 4.3 Rate Limiting
- Default: 100 requests/minute per tenant
- Configurable per tenant and per route
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Returns 429 Too Many Requests when exceeded

---

## 5. Configuration Management

### 5.1 Configuration Structure
Each service loads configuration from:
1. Environment variables (highest precedence)
2. `.env` file in service directory
3. Default config file: `config/{service_name}.toml`

### 5.2 Standard Configuration Schema
```toml
[service]
name = "gl-service"
port = 8010
log_level = "info"

[database]
path = "data/gl.db"
pool_max_size = 8
pool_min_idle = 2

[jwt]
public_key_path = "config/jwt-public.pem"
issuer = "fusion-erp"
audience = "fusion-erp-api"

[events]
enabled = true
bus_address = "127.0.0.1:9090"

[services.auth]
url = "http://localhost:8001"
[services.gl]
url = "http://localhost:8010"
# ... other service URLs
```

---

## 6. Error Handling

### 6.1 Standard Error Codes
Errors use a structured code format: `{SERVICE}_{CATEGORY}_{NUMBER}`

| Service | Prefix | Examples |
|---------|--------|---------|
| Auth | AUTH | AUTH_LOGIN_001 (invalid credentials), AUTH_TOKEN_001 (expired token) |
| GL | GL | GL_PERIOD_001 (period closed), GL_JOURNAL_001 (unbalanced entry) |
| AP | AP | AP_INVOICE_001 (duplicate invoice number), AP_PAYMENT_001 (insufficient funds) |
| AR | AR | AR_INVOICE_001, AR_RECEIPT_001 |
| General | GEN | GEN_VALIDATION_001, GEN_NOT_FOUND_001, GEN_CONFLICT_001 |

### 6.2 Error Response Format
```json
{
  "error": {
    "code": "GL_PERIOD_001",
    "message": "Cannot post to a closed period",
    "details": [
      {
        "field": "period_name",
        "issue": "Period 'Jan-2024' is closed",
        "suggestion": "Use an open period or reopen this period"
      }
    ],
    "request_id": "req-uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### 6.3 Circuit Breaker Configuration
- Threshold: 5 consecutive failures
- Open duration: 30 seconds
- Half-open: allow 1 request, if success → close, if failure → stay open
- Implementation: tower circuit-breaker middleware or custom using `tokio::sync::watch`

---

## 7. Crate Dependencies (per service)

### 7.1 Common Dependencies
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.8"
tonic = "0.12"
prost = "0.13"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v7"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json"] }
thiserror = "2"
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace", "request-id"] }
validator = { version = "0.19", features = ["derive"] }
```

### 7.2 Service-Specific
```toml
# auth-service only
jsonwebtoken = "9"
bcrypt = "0.16"

# Database
rusqlite = { version = "0.32", features = ["bundled"] }
r2d2 = "0.8"
r2d2_sqlite = "0.25"

# Metrics
prometheus = "0.13"
```

---

## 8. Health Check and Readiness

Every service MUST expose:

### 8.1 Liveness Probe
```
GET /health → 200 OK { "status": "alive" }
```

### 8.2 Readiness Probe
```
GET /ready → 200 OK { "status": "ready", "checks": { "database": "ok", "dependencies": "ok" } }
```
Returns 503 if database or critical dependencies are unavailable.

### 8.3 Metrics
```
GET /metrics → Prometheus text format
```
Standard metrics:
- `fusion_http_requests_total{method, path, status}`
- `fusion_http_request_duration_seconds{method, path}`
- `fusion_grpc_requests_total{service, method, status}`
- `fusion_db_pool_size{service}`
- `fusion_db_pool_available{service}`
