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
# ... etc
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
