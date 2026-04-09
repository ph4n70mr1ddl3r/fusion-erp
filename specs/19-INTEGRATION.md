# 19 - Inter-Service Integration Specification

## 1. Overview

This spec defines how services communicate with each other, the event bus architecture, service discovery, and the patterns for distributed data consistency.

---

## 2. Communication Matrix

### 2.1 Service Dependencies

| Service | Calls (gRPC) | Called By |
|---------|-------------|-----------|
| gl-service | — | AP, AR, FA, CM, INV, MFG, PM, Report |
| ap-service | GL, INV, Workflow | Report |
| ar-service | GL, OM, Workflow, CM | Report |
| fa-service | GL | AP, Report |
| cm-service | GL | AP, AR, Report |
| proc-service | INV, Workflow | AP, Report |
| inv-service | GL | Proc, OM, MFG, Report |
| om-service | INV, AR, Workflow | Report |
| mfg-service | INV, GL | Report |
| pm-service | GL, AR, AP, Workflow | Report |
| workflow-service | — | All services |
| report-service | All services | — |
| auth-service | — | Gateway |

### 2.2 gRPC Call Patterns

**Synchronous (Request-Response):**
- Used for commands that need immediate response
- Example: AP → GL (create journal entry)
- Timeout: 10 seconds
- Retries: 3 with exponential backoff (1s, 2s, 4s)

**Streaming (Server-Side):**
- Used for bulk data retrieval
- Example: Report → GL (stream trial balance data)
- Batch size: 100 records per stream message

---

## 3. Event Bus Architecture

### 3.1 Event Bus Implementation
Services communicate events across process boundaries using an **outbox-polling pattern** with a central event-relay. Since `tokio::broadcast` only works within a single process, cross-process delivery uses gRPC.

**Architecture:**
```
Publisher Service → outbox_events table (local DB)
                       ↓ (polled by event-relay thread)
                 event-relay (central or per-service)
                       ↓ (delivers via gRPC)
                 Subscriber Service → inbox_events table
```

### 3.2 Event Store Table (per publisher service)
```sql
CREATE TABLE outbox_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    aggregate_type TEXT NOT NULL,
    aggregate_id TEXT NOT NULL,
    payload TEXT NOT NULL,                 -- JSON
    metadata TEXT,                         -- JSON (correlation_id, user_id, etc.)
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    published_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PUBLISHED','FAILED')),
    retry_count INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_tenant_status ON outbox_events(tenant_id, status);
CREATE INDEX idx_outbox_created ON outbox_events(created_at);
```

### 3.3 Event Subscriber Table (per subscriber service)
```sql
CREATE TABLE inbox_events (
    id TEXT PRIMARY KEY NOT NULL,
    event_id TEXT NOT NULL,                -- From publisher
    event_type TEXT NOT NULL,
    source_service TEXT NOT NULL,
    tenant_id TEXT NOT NULL,
    payload TEXT NOT NULL,
    processed_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','PROCESSING','COMPLETED','FAILED')),
    error_message TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(event_id)
);
```

### 3.4 Event Delivery Flow
1. **Publish:** Service writes event to its local `outbox_events` table within the same database transaction as the business data change
2. **Poll:** A central event-relay process (or a relay thread within each publisher service) periodically polls each service's `outbox_events` table for PENDING events
3. **Deliver:** The relay delivers events to subscriber services by calling their gRPC `ProcessEvent` endpoint
4. **Receive:** Subscriber service receives the event via gRPC, writes it to its `inbox_events` table
5. **Process:** Subscriber processes the event idempotently (checks `event_id` uniqueness before applying)
6. **Complete:** Subscriber marks the inbox event as COMPLETED; relay marks the outbox event as PUBLISHED

### 3.5 Event Schema
```json
{
  "event_id": "0192a3b4-event-id",
  "event_type": "ap.invoice.posted",
  "source_service": "ap-service",
  "tenant_id": "0192-tenant-id",
  "aggregate_type": "invoice",
  "aggregate_id": "0192-invoice-id",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlation_id": "req-0192-request-id",
  "payload": {
    "invoice_id": "0192-invoice-id",
    "invoice_number": "AP-INV-2024-00001",
    "supplier_id": "0192-supplier-id",
    "total_cents": 500000,
    "currency_code": "USD",
    "journal_id": "0192-journal-id"
  }
}
```

### 3.6 Event Naming Convention
Format: `{service}.{entity}.{action}`

| Event | Publisher | Subscribers |
|-------|-----------|-------------|
| `gl.journal.posted` | GL | Report |
| `gl.period.closed` | GL | AP, AR, FA, Report |
| `ap.invoice.posted` | AP | GL, Report |
| `ap.invoice.paid` | AP | CM, GL, Report |
| `ar.invoice.posted` | AR | GL, Report |
| `ar.receipt.applied` | AR | CM, GL, Report |
| `fa.depreciation.calculated` | FA | GL, Report |
| `cm.transaction.created` | CM | GL, Report |
| `proc.goods.received` | Proc | INV, AP |
| `inv.stock.received` | INV | GL, Report |
| `inv.stock.issued` | INV | GL, Report |
| `om.order.shipped` | OM | INV, AR |
| `mfg.production.completed` | MFG | INV, GL |
| `pm.time_entry.approved` | PM | GL |
| `workflow.approved` | Workflow | Originating service |

---

## 4. Saga Pattern Implementation

### 4.1 Saga Example: Procure-to-Pay
```
Step 1: proc-service → Create PO → Status: ISSUED
Step 2: inv-service  → Reserve stock → On PO approval
Step 3: inv-service  → Receive goods → On goods receipt
Step 4: ap-service   → Create invoice → On supplier invoice
Step 5: ap-service   → Match invoice → 3-way match
Step 6: ap-service   → Create payment → On payment run
Step 7: gl-service   → Post all journal entries → On each step

Compensating actions:
- Step 3 fails → Unreserve stock
- Step 4 fails → Flag for manual invoice entry
- Step 6 fails → Leave invoice open, retry payment
```

### 4.2 Saga State Management
Each saga tracked in `saga_instances` table (see 01-ARCHITECTURE.md). The initiating service:
1. Creates saga instance
2. Executes steps sequentially
3. On failure, triggers compensating transactions
4. Updates saga status on completion

### 4.3 Idempotency in Sagas
All saga steps MUST be idempotent. Each step checks if already completed:
```rust
fn receive_goods(saga: &Saga, receipt: &GoodsReceipt) -> Result<()> {
    // Check if already processed
    if is_step_completed(saga.id, "receive_goods")? {
        return Ok(());
    }
    // Process...
    mark_step_completed(saga.id, "receive_goods")?;
    Ok(())
}
```

---

## 5. Service Discovery

### 5.1 Static Configuration
Services register their addresses in a shared configuration file:
```toml
# config/services.toml
[services.auth]
url = "http://localhost:8001"
grpc_url = "http://localhost:9001"

[services.gl]
url = "http://localhost:8010"
grpc_url = "http://localhost:9010"

[services.ap]
url = "http://localhost:8011"
grpc_url = "http://localhost:9011"
# ... etc
```

### 5.2 Client Factory
```rust
pub struct ServiceRegistry {
    config: HashMap<String, ServiceConfig>,
}

impl ServiceRegistry {
    pub fn get_grpc_client<T: tonic::client::GrpcService<tonic::body::BoxBody>>(
        &self,
        service: &str,
    ) -> Result<T> {
        let config = self.config.get(service)
            .ok_or_else(|| FusionError::NotFound(format!("Service '{}' not found", service)))?;
        // Create and return gRPC client
        todo!()
    }
}
```

---

## 6. Error Handling for Inter-Service Calls

### 6.1 Retry Policy
```rust
use tokio::time::{sleep, Duration};

pub async fn call_with_retry<F, Fut, T>(f: F, max_retries: u32) -> FusionResult<T>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = FusionResult<T>>,
{
    let mut attempts = 0;
    loop {
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if attempts < max_retries && e.is_retryable() => {
                attempts += 1;
                let delay = Duration::from_millis(100 * 2u64.pow(attempts));
                sleep(delay).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

### 6.2 Circuit Breaker
- Threshold: 5 consecutive failures
- Open duration: 30 seconds
- Implementation: tower layer or custom watch channel
- When open: return `FusionError::ServiceUnavailable` immediately

### 6.3 Timeout
- gRPC calls: 10-second timeout
- Report data streaming: 60-second timeout
- Timeout configured via tonic client options:
```rust
let client = tonic::transport::Channel::from_static("http://localhost:9010")
    .timeout(Duration::from_secs(10))
    .connect()
    .await?;
```
