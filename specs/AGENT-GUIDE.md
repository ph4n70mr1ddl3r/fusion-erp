# AI Agent Implementation Guide

## 1. Purpose

This guide instructs AI agents on how to consume the specification documents in this directory to build the Fusion ERP system incrementally, following spec-driven development.

---

## 2. How to Read Specs

### 2.1 Spec Reading Order
Read specs in this exact order for each stage:

**Stage 0 — Foundation (Build First):**
1. `00-OVERVIEW.md` — Understand the big picture, module list, build order
2. `02-TECH-STACK.md` — Set up workspace, dependencies, tooling
3. `01-ARCHITECTURE.md` — Understand service decomposition and communication
4. `04-DATABASE.md` — Understand DB conventions, standard columns, migration patterns
5. `05-AUTH-SECURITY.md` — Implement auth service (everything depends on auth)

**Stage 1 — Core Financials:**
6. `06-GENERAL-LEDGER.md` — Build GL first (all financial modules depend on it)
7. `07-ACCOUNTS-PAYABLE.md`
8. `08-ACCOUNTS-RECEIVABLE.md`
9. `09-FIXED-ASSETS.md`
10. `10-CASH-MANAGEMENT.md`

**Stage 2 — Supply Chain:**
11. `11-PROCUREMENT.md`
12. `12-INVENTORY.md`
13. `13-ORDER-MANAGEMENT.md`
14. `14-MANUFACTURING.md`

**Stage 3 — Operations:**
15. `15-PROJECT-MANAGEMENT.md`
16. `16-WORKFLOW.md`

**Stage 4 — Financial Extensions:**
17. `23-TAX-MANAGEMENT.md` — Tax engine (AP/AR depend on tax calculation)
18. `24-INTERCOMPANY.md` — Intercompany transactions
19. `25-EXPENSE-MANAGEMENT.md` — Employee expenses, corporate cards
20. `26-REVENUE-MANAGEMENT.md` — ASC 606 revenue recognition
21. `31-LEASE-ACCOUNTING.md` — IFRS 16/ASC 842 lease accounting

**Stage 5 — Supply Chain Extensions:**
22. `27-ADVANCED-PRICING.md` — Price lists, promotions, discount engine
23. `28-SUPPLY-CHAIN-PLANNING.md` — MRP, demand forecasting
24. `32-COLLECTIONS-CREDIT.md` — Collections, credit scoring, dunning

**Stage 6 — Cross-Cutting Services & Delivery:**
25. `29-DOCUMENT-MANAGEMENT.md` — Universal file attachments
26. `30-DATA-IMPORT-EXPORT.md` — Bulk data loading, templates
27. `17-REPORTING.md`
28. `18-MULTI-TENANCY.md`
29. `19-INTEGRATION.md`
30. `20-FRONTEND.md`
31. `21-DEPLOYMENT.md`
32. `22-TESTING.md`

> **Stages 7–38 continue below.** For a complete categorized view of all 211 specs, see `00-INDEX.md`.

**Stage 7 — Enterprise Extensions:**
33. `33-EPM.md` — Planning, consolidation, reconciliation
34. `34-RISK-COMPLIANCE.md` — SoD, access certifications, audit
35. `40-ENTERPRISE-ASSET.md` — Asset operations, maintenance
36. `42-SUSTAINABILITY.md` — ESG reporting, carbon tracking

**Stage 8 — AI & Intelligence:**
37. `35-AI-ML-PLATFORM.md` — AI agents, ML models, anomaly detection
38. `43-DIGITAL-ASSISTANT.md` — Conversational AI, notifications
39. `44-IOT-INTEGRATION.md` — IoT devices, telemetry, edge computing

**Stage 9 — Collaboration & Delivery:**
40. `45-SUPPLIER-PORTAL.md` — Supplier self-service, sourcing
41. `46-MOBILE.md` — Mobile framework, offline sync
42. `47-BLOCKCHAIN.md` — Blockchain traceability, provenance

**Stage 10 — Financial Extensions II:**
43. `48-COST-MANAGEMENT.md` — Product costing, cost rollup
44. `49-ACCOUNTING-HUB.md` — Centralized accounting for third-party
45. `54-GRANT-MANAGEMENT.md` — Federal grant lifecycle
46. `55-JOINT-VENTURE.md` — JV cost sharing, partner billing

**Stage 11 — Supply Chain Extensions II:**
47. `50-GLOBAL-ORDER-PROMISING.md` — ATP/CTP, delivery date promising
48. `51-PRODUCT-HUB.md` — Product MDM, catalog publishing
49. `52-SUPPLY-CHAIN-ORCHESTRATION.md` — Saga-based coordination
50. `53-PROCUREMENT-CONTRACTS.md` — Contract lifecycle, compliance

**Stage 12 — Revenue & Commerce:**
51. `56-SUBSCRIPTION-MANAGEMENT.md` — Recurring billing
52. `57-CPQ.md` — Configure-Price-Quote engine
53. `58-CONFIGURATOR.md` — Product configuration rules engine
54. `59-COMMERCE.md` — B2B/B2C e-commerce platform
55. `60-CUSTOMER-DATA-PLATFORM.md` — Unified customer profiles
56. `61-MARKETING.md` — Campaign management, automation

**Stage 13 — Human Capital Management:**
57. `62-CORE-HR.md` — Employee records, org structures
58. `63-PAYROLL.md` — Payroll processing, payslips
59. `64-TIME-LABOR.md` — Time tracking, overtime
60. `65-COMPENSATION.md` — Salary plans, merit cycles
61. `66-BENEFITS.md` — Benefits enrollment
62. `67-RECRUITING.md` — Talent acquisition, hiring
63. `68-PERFORMANCE-MANAGEMENT.md` — Reviews, goals, PIPs
64. `69-LEARNING-DEVELOPMENT.md` — LMS, certifications
65. `70-SUCCESSION-PLANNING.md` — Talent pipeline, readiness
66. `71-CAREER-DEVELOPMENT.md` — Skills, career paths
67. `72-ABSENCE-MANAGEMENT.md` — Leave tracking, accruals
68. `73-WORKFORCE-SCHEDULING.md` — Shift scheduling
69. `74-WORKFORCE-PLANNING.md` — Headcount planning
70. `75-HR-HELP-DESK.md` — HR case management
71. `76-WORKFORCE-SAFETY.md` — Incident reporting, OSHA

**Stage 14 — Customer Experience (CX/CRM):**
72. `77-SALES-AUTOMATION.md` — CRM, leads, opportunities
73. `78-SALES-PLANNING.md` — Territory, quota management
74. `79-SALES-PERFORMANCE.md` — Commissions, incentives
75. `80-FIELD-SERVICE.md` — Technician dispatch
76. `81-CUSTOMER-SERVICE.md` — Support tickets, SLA
77. `82-KNOWLEDGE-MANAGEMENT.md` — Knowledge base
78. `83-SERVICE-LOGISTICS.md` — Service parts, RMA
79. `84-CHANNEL-MANAGEMENT.md` — Partner portal

**Stage 15 — EPM & Governance Extensions:**
80. `85-TAX-REPORTING.md` — Tax provision (ASC 740)
81. `86-NARRATIVE-REPORTING.md` — SEC filings, XBRL
82. `87-ENTERPRISE-DATA-MANAGEMENT.md` — Dimension governance
83. `88-PROFITABILITY-MANAGEMENT.md` — Activity-based costing

**Stage 16 — Platform & Integration:**
84. `89-INTEGRATION-CLOUD.md` — iPaaS, adapters
85. `90-VISUAL-BUILDER.md` — Low-code app builder
86. `91-PROCESS-AUTOMATION.md` — RPA, process mining
87. `92-INNOVATION-MANAGEMENT.md` — Innovation pipeline

**Stage 17 — HCM Talent Extensions:**
88. `93-OPPORTUNITY-MARKETPLACE.md` — Internal talent marketplace
89. `94-DYNAMIC-SKILLS.md` — AI skills ontology
90. `95-EMPLOYEE-EXPERIENCE.md` — Engagement, pulse surveys
91. `96-ONBOARDING.md` — Onboarding journeys

**Stage 18 — CX & Commerce Extensions:**
92. `97-LOYALTY-MANAGEMENT.md` — Customer loyalty programs

**Stage 19 — Advanced Manufacturing:**
93. `98-CONTRACT-MANUFACTURING.md` — Outsourced manufacturing
94. `99-MANUFACTURING-EXECUTION.md` — MES, shop floor, OEE

**Stage 20 — Financial Close & Consolidation:**
95. `100-FINANCIAL-CONSOLIDATION.md` — Multi-entity consolidation
96. `101-ACCOUNT-RECONCILIATION.md` — Auto-matching, reconciliation
97. `104-FEDERAL-FINANCIALS.md` — U.S. Federal financials

**Stage 21 — Analytics & Platform Extensions:**
98. `102-FUSION-DATA-INTELLIGENCE.md` — Cross-app analytics, KPIs
99. `103-APPLICATION-COMPOSER.md` — Custom objects, fields

**Stage 22 — Workforce Optimization:**
100. `105-WORKFORCE-LABOR-OPTIMIZATION.md` — Labor demand forecasting

**Stage 23 — HCM Experience & Governance:**
101. `106-MANAGER-EDGE.md` — AI manager dashboard
102. `107-WORK-LIFE.md` — Employee wellness
103. `108-ADVANCED-HCM-CONTROLS.md` — SoD enforcement, audit
104. `109-EXPERIENCE-DESIGN-STUDIO.md` — No-code UI customization
105. `110-ACTIVITY-CENTERS.md` — Role-based workspaces

**Stage 24 — Workforce Strategy & Digital CX:**
106. `111-WORKFORCE-MODELING.md` — Scenario headcount modeling
107. `112-DIGITAL-CUSTOMER-SERVICE.md` — AI chatbot, self-service
108. `119-STRATEGIC-WORKFORCE-PLANNING.md` — Multi-year talent planning

**Stage 25 — Advanced Manufacturing & Analytics:**
109. `113-PROJECT-DRIVEN-MANUFACTURING.md` — Project-linked production
110. `114-SCM-ANALYTICS.md` — Supply chain KPIs, dashboards
111. `115-PROJECT-ASSET-MANAGEMENT.md` — Capital project assets
112. `116-BUSINESS-CONTINUITY.md` — Disaster recovery

**Stage 26 — EPM Foundation:**
113. `117-EPM-PLATFORM.md` — OLAP engine, dimensions
114. `118-FREEFORM-PLANNING.md` — Freeform budgeting

**Stage 27 — Supply Chain Advanced:**
115. `120-SELF-SERVICE-PROCUREMENT.md` — Employee requisitions
116. `121-PROCESS-MANUFACTURING.md` — Formula-based production
117. `122-PRODUCTION-SCHEDULING.md` — Shop floor scheduling
118. `123-SMART-OPERATIONS.md` — IoT operational intelligence
119. `124-SOURCING.md` — RFQ/RFP/auction
120. `125-SUPPLY-CHAIN-EXECUTION.md` — Logistics orchestration
121. `139-SUPPLIER-MANAGEMENT.md` — Supplier lifecycle
122. `140-DEMAND-MANAGEMENT.md` — Statistical forecasting
123. `141-SALES-OPERATIONS-PLANNING.md` — Cross-functional S&OP

**Stage 28 — SCM Logistics & Fulfillment:**
124. `142-SUPPLY-CHAIN-COLLABORATION.md` — Trading partner collaboration
125. `143-BACKLOG-MANAGEMENT.md` — Order backlog prioritization
126. `144-MIXED-MODE-MANUFACTURING.md` — Discrete + process hybrid
127. `145-SHIPPING-EXECUTION.md` — Outbound shipment planning

**Stage 29 — HCM Talent & Experience:**
128. `146-TALENT-REVIEW.md` — Calibration sessions, 9-box
129. `147-ORACLE-JOURNEYS.md` — Guided process templates
130. `161-WORKFORCE-PREDICTIONS.md` — AI attrition predictions
131. `169-GOALS-MANAGEMENT.md` — Goal cascading, OKRs
132. `170-TOUCHPOINTS.md` — Continuous check-ins
133. `171-GUIDED-LEARNING.md` — In-app contextual help
134. `175-ORACLE-GROW.md` — AI growth recommendations
135. `189-HCM-NOW.md` — Rapid HCM deployment

**Stage 30 — CX Sales & Service Extensions:**
136. `126-B2C-SERVICE.md` — Consumer support
137. `127-INTELLIGENT-ADVISOR.md` — Rule-based policy automation
138. `128-LIVE-EXPERIENCE.md` — Co-browsing, video
139. `129-SERVICE-CENTER.md` — Unified agent workspace
140. `130-PARTNER-RELATIONSHIP-MANAGEMENT.md` — Partner portals
141. `131-CX-ANALYTICS.md` — Journey analytics, churn
142. `179-SALES-ENGAGEMENT.md` — Sales sequences

**Stage 31 — CX Marketing Extensions:**
143. `168-MAXYMISER.md` — A/B testing, personalization
144. `180-MDF-MANAGEMENT.md` — Market development funds
145. `181-ELOQUA.md` — B2B marketing automation
146. `182-RESPONSYS.md` — B2C cross-channel marketing
147. `183-CX-ADVERTISING.md` — Advertising activation

**Stage 32 — EPM & Reporting Extensions:**
148. `152-DISCLOSURE-MANAGEMENT.md` — SEC filings, XBRL
149. `153-SUPPLEMENTAL-DATA.md` — Non-financial data collection
150. `154-FINANCIAL-REPORTING.md` — Financial report generation
151. `155-SMART-VIEW-OFFICE.md` — Office integration
152. `156-EPM-AUTOMATE.md` — CLI automation
153. `185-PLANNING-BUDGETING.md` — Planning, budgeting
154. `187-CALCULATION-MANAGER.md` — Business rule calculations

**Stage 33 — Financial & Project Extensions:**
155. `158-PROJECT-COSTING.md` — Cost tracking, capitalization
156. `159-PROJECT-BILLING.md` — T&M, fixed price billing
157. `160-RESOURCE-MANAGEMENT.md` — Resource planning

**Stage 34 — Platform Services:**
158. `162-PRODUCT-DEVELOPMENT.md` — ECOs, BOM lifecycle
159. `163-FUNCTIONAL-SETUP-MANAGER.md` — Implementation management
160. `164-ENTERPRISE-SEARCH.md` — Full-text search
161. `165-NOTIFICATION-CENTER.md` — Multi-channel notifications
162. `166-CONNECTED-WORKER.md` — IoT workforce safety
163. `167-MAINTENANCE-CLOUD.md` — Asset maintenance
164. `172-B2B-GATEWAY.md` — EDI, B2B messaging
165. `173-ENTERPRISE-SCHEDULER.md` — Job scheduling
166. `174-OTBI.md` — Real-time operational analytics
167. `176-SPREADSHEET-DESIGNER.md` — Web-based reports
168. `177-ACCESS-GOVERNOR.md` — Access certification, SoD
169. `178-SANDBOX-MANAGEMENT.md` — Test environments
170. `184-CONTENT-MANAGEMENT.md` — Enterprise content
171. `188-SUPPLIER-QUALIFICATION.md` — Supplier onboarding
172. `190-ORACLE-ME.md` — Conversational AI platform
173. `191-DATA-RELATIONSHIP-MGMT.md` — Master data governance

**Stage 35 — Advanced Platform:**
174. `132-DESKLESS-WORKFORCE.md` — Mobile-first HR
175. `133-PAYROLL-INTERFACE-CONNECT.md` — Payroll integration
176. `134-AI-AGENTS-SUITE.md` — Cross-app AI agents
177. `135-DOCUMENT-IO-AGENT.md` — AI document processing
178. `186-ORACLE-HEALTH.md` — Clinical data management
179. `202-FUSION-DATA-INTELLIGENCE.md` — Unified data platform

**Stage 36 — Analytics Extensions:**
180. `148-HCM-ANALYTICS.md` — Workforce analytics
181. `149-ERP-ANALYTICS.md` — Financial analytics
182. `150-BI-PUBLISHER.md` — Template-based reports
183. `157-PROCUREMENT-ANALYTICS.md` — Spend analysis

**Stage 37 — Industry Verticals (build in parallel):**
184. `136-HIGHER-EDUCATION.md` — Student management
185. `137-PUBLIC-SECTOR.md` — Government finance
186. `138-COMMUNICATIONS.md` — Telecom billing
187. `192-RETAIL.md` — Store management
188. `193-AUTOMOTIVE.md` — Dealer management
189. `194-OIL-GAS.md` — Well lifecycle
190. `195-LIFE-SCIENCES.md` — Clinical trials
191. `196-CONSUMER-GOODS.md` — Trade promotions
192. `197-HIGH-TECH.md` — Configure-to-order
193. `198-WHOLESALE-DISTRIBUTION.md` — Warehouse zones
194. `199-TRAVEL-TRANSPORTATION.md` — Fleet, revenue
195. `200-INDUSTRIAL-MFG.md` — Heavy equipment
196. `201-RESTAURANTS.md` — Multi-location
197. `203-CLINICAL-SUPPLY-CHAIN.md` — Healthcare supply chain
198. `206-CONSTRUCTION-ENGINEERING.md` — Progress billing
199. `207-FINANCIAL-SERVICES.md` — Banking, KYC/AML
200. `208-UTILITIES.md` — Utility billing
201. `209-PROFESSIONAL-SERVICES.md` — Engagement management
202. `210-AEROSPACE-DEFENSE.md` — Gov contracts, EVM
203. `211-MEDIA-ENTERTAINMENT.md` — Content rights

**Stage 38 — Migration Tools:**
204. `204-TALEO-MIGRATION.md` — Taleo to Fusion migration
205. `205-SIEBEL-MIGRATION.md` — Siebel CRM migration

### 2.2 How to Read Each Spec
Every module spec follows this structure:
1. **Domain Overview** — What this service does
2. **Database Schema** — CREATE TABLE statements (copy directly as migrations)
3. **REST API Endpoints** — Method, path, request/response, permissions
4. **Business Rules** — Validation logic, status flows, calculations
5. **gRPC Service** — Proto definitions for inter-service calls
6. **Inter-Service Integration** — What events to publish/consume
7. **Events** — Event names and payloads

---

## 3. Implementation Process per Service

### 3.1 Step-by-Step Checklist

For each service, follow these steps IN ORDER:

```
□ 1. Create Cargo.toml for the service crate
□ 2. Create migration files from the spec's "Database Schema" section
□ 3. Create the database repository layer (SQL queries)
□ 4. Create domain types (structs matching table schemas)
□ 5. Implement business rules and validation
□ 6. Create gRPC service implementation (proto → Rust)
□ 7. Create REST API handlers (axum routes)
□ 8. Wire up middleware (auth, tenant, audit)
□ 9. Implement event publishing (outbox pattern)
□ 10. Implement event subscription (inbox pattern)
□ 11. Write unit tests for domain logic
□ 12. Write repository tests (in-memory SQLite)
□ 13. Write API handler tests
□ 14. Write gRPC contract tests
□ 15. Verify all tests pass
□ 16. Test multi-tenant isolation
```

### 3.2 File Structure per Service
```
services/{service-name}/
├── Cargo.toml
├── src/
│   ├── main.rs                    # Entry point, config, server startup
│   ├── config.rs                  # Configuration loading
│   ├── db/
│   │   ├── mod.rs
│   │   ├── pool.rs                # Connection pool creation
│   │   └── migrations.rs          # Migration runner
│   ├── models/
│   │   ├── mod.rs
│   │   ├── {entity}.rs            # Domain structs per table
│   │   └── ...
│   ├── repository/
│   │   ├── mod.rs
│   │   ├── {entity}_repo.rs       # SQL queries per table
│   │   └── ...
│   ├── service/
│   │   ├── mod.rs
│   │   ├── {entity}_service.rs    # Business logic
│   │   └── ...
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── {entity}_handler.rs    # Axum route handlers
│   │   └── ...
│   ├── grpc/
│   │   ├── mod.rs
│   │   └── {service}_grpc.rs      # Tonic gRPC implementation
│   ├── events/
│   │   ├── mod.rs
│   │   ├── publisher.rs           # Event outbox publisher
│   │   └── subscriber.rs          # Event inbox subscriber
│   ├── middleware/
│   │   ├── mod.rs
│   │   ├── auth.rs                # JWT validation
│   │   ├── tenant.rs              # Tenant extraction
│   │   └── audit.rs               # Audit logging
│   └── routes.rs                  # Route definitions
├── migrations/
│   ├── V001_{name}.up.sql
│   ├── V001_{name}.down.sql
│   └── ...
└── tests/
    ├── api_tests.rs
    ├── repo_tests.rs
    └── grpc_tests.rs
```

---

## 4. Implementation Rules

### 4.1 MUST Follow
- Every table MUST have the standard columns (id, tenant_id, created_at, updated_at, created_by, updated_by, version, is_active)
- Every query MUST include tenant_id in WHERE clause
- Every monetary value MUST be stored as INTEGER cents
- Every API endpoint MUST check permissions before executing
- Every mutation MUST log to audit trail
- Every migration MUST have up AND down files
- Every gRPC call MUST have timeout and retry logic
- Every service MUST expose /health and /ready endpoints

### 4.2 MUST NOT
- MUST NOT use FLOAT/REAL for money (use INTEGER cents)
- MUST NOT build SQL via string concatenation (use parameterized queries)
- MUST NOT skip tenant_id in queries
- MUST NOT create cross-service database dependencies
- MUST NOT store passwords in plaintext
- MUST NOT add features not in the specs (YAGNI)

### 4.3 Code Style
- Use `thiserror` for error types, not anyhow
- Use `#[instrument]` for tracing on all handlers and service methods
- Use Rust idioms: `Result<T, E>`, `Option<T>`, pattern matching
- No `unwrap()` in production code — use `?` and proper error handling
- No `clone()` unless necessary — prefer references

---

## 5. Dependencies Between Services

### 5.1 Build Order Dependency Graph
```
auth-service ──────────────────────────────────────────────┐
  │                                                        │
gl-service ────────────────────────────────────────────────┤
  │                                                         │
  ├── ap-service ─── (also depends on: Tax, Workflow) ─────┤
  ├── ar-service ─── (also depends on: Tax, Pricing) ──────┤
  ├── fa-service ───────────────────────────────────────────┤
  ├── cm-service ───────────────────────────────────────────┤
  │                                                         │
inv-service ───────────────────────────────────────────────┤
  │                                                         │
  ├── proc-service ─── (also depends on: AP, Tax, Workflow)┤
  ├── om-service ───── (also depends on: AR, Pricing, INV) │
  ├── mfg-service ──── (also depends on: GL) ──────────────┤
  │                                                         │
workflow-service ──────────────────────────────────────────┤
  │                                                         │
pm-service ──── (depends on: GL, AR, RevMgmt, Workflow) ──┤
                                                           │
tax-service ────── (depends on: GL) ───────────────────────┤
ic-service ─────── (depends on: GL, AP, AR) ───────────────┤
expense-service ── (depends on: AP, GL, PM, Workflow) ─────┤
rev-service ────── (depends on: AR, OM, PM, GL) ──────────┤
pricing-service ── (depends on: OM, AR, Proc) ─────────────┤
planning-service ─ (depends on: INV, Proc, MFG, OM) ──────┤
dms-service ────── (cross-cutting, no service deps) ───────┤
etl-service ────── (depends on: ALL services for data) ────┤
lease-service ──── (depends on: GL, AP, FA, CM) ──────────┤
collections-service (depends on: AR, OM, GL, Workflow) ────┤
                                                           │
report-service ──── (depends on: ALL services) ────────────┘
```

### 5.2 Shared Crate Build Order
1. `fusion-core` (no dependencies) — Error types, domain primitives
2. `fusion-db` (depends on fusion-core) — SQLite pool, migrations
3. `fusion-auth` (depends on fusion-core) — JWT, RBAC types
4. `fusion-proto` (depends on fusion-core) — Protobuf definitions
5. `fusion-client` (depends on fusion-proto, fusion-auth) — gRPC clients
6. `fusion-ui` (depends on fusion-core) — Leptos components

---

## 6. Testing Strategy

### 6.1 When to Test
- Write unit tests alongside the implementation (TDD preferred)
- Repository tests: test every SQL query with in-memory SQLite
- API tests: test every endpoint with mock auth
- Run `cargo nextest run` after every service completion
- Do NOT proceed to the next service until current service tests pass

### 6.2 Test Priority
1. Multi-tenant isolation (highest — data security)
2. Business rule validation (high — correctness)
3. API contract (high — integration)
4. gRPC contract (medium — inter-service)
5. Edge cases (medium — robustness)

---

## 7. Common Patterns

### 7.1 Creating a New Entity
```rust
// 1. Define model
pub struct NewAccount {
    pub account_code: String,
    pub account_name: String,
    pub account_type: String,
    // ...
}

// 2. Repository function
pub fn create(conn: &Connection, tenant_id: &str, input: &NewAccount, user_id: &str) -> Result<Account> {
    let id = uuid::Uuid::now_v7().to_string();
    conn.execute(
        "INSERT INTO gl_accounts (id, tenant_id, account_code, account_name, account_type, created_by, updated_by)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?6)",
        params![id, tenant_id, input.account_code, input.account_name, input.account_type, user_id],
    )?;
    find_by_id(conn, tenant_id, &id)
}

// 3. Service layer (business rules + validation)
pub async fn create_account(pool: &Pool, tenant_id: &str, input: NewAccount, user_id: &str) -> FusionResult<Account> {
    input.validate()?;
    // Check for duplicate code
    // ...
    let pool = pool.clone();
    tokio::task::spawn_blocking(move || {
        let conn = pool.get()?;
        create(&conn, tenant_id, &input, user_id)
    }).await?
}

// 4. API handler
async fn handle_create(
    State(state): State<AppState>,
    claims: TokenClaims,
    Json(input): Json<NewAccount>,
) -> impl IntoResponse {
    check_permission(&claims, "gl.accounts.create")?;
    let account = create_account(&state.pool, &claims.tid, input, &claims.sub).await?;
    Ok((StatusCode::CREATED, Json(ApiResponse { data: account, meta: default_meta() })))
}
```

### 7.2 Publishing Events
```rust
pub fn publish_event(conn: &Connection, event: OutboxEvent) -> Result<()> {
    conn.execute(
        "INSERT INTO outbox_events (id, tenant_id, event_type, aggregate_type, aggregate_id, payload)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6)",
        params![event.id, event.tenant_id, event.event_type, event.aggregate_type, event.aggregate_id, event.payload_json()],
    )?;
    Ok(())
}
```

---

## 8. Incremental Delivery

### 8.1 Milestone Definitions

**Milestone 0 — Workspace Ready:**
- Cargo workspace builds
- Shared crates compile
- Migrations framework works
- Auth service runs and issues JWTs

**Milestone 1 — GL Operational:**
- GL service: accounts CRUD, journal CRUD, posting, period management
- Tests pass
- Can create chart of accounts and post journal entries

**Milestone 2 — Financial Suite:**
- AP, AR, FA, CM services operational
- Can process full procure-to-pay and order-to-cash flows
- All services post to GL

**Milestone 3 — Supply Chain:**
- Procurement, Inventory, OM, Manufacturing operational
- Full supply chain integration tested

**Milestone 4 — Complete System:**
- All services operational
- Frontend functional
- Integration tests pass
- Multi-tenant isolation verified

### 8.2 Definition of Done per Service
A service is "done" when:
- [ ] All migrations run successfully
- [ ] All CRUD operations implemented
- [ ] All business rules enforced
- [ ] All REST endpoints return correct status codes
- [ ] gRPC endpoints respond correctly
- [ ] Events published for all mutations
- [ ] Multi-tenant isolation tested and verified
- [ ] Unit test coverage >= 80% for domain logic
- [ ] All tests pass: `cargo nextest run -p {service}`
