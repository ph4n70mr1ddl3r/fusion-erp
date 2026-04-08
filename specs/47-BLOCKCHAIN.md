# 47 - Blockchain & Digital Thread Service Specification

## 1. Domain Overview

Blockchain & Digital Thread provides blockchain network configuration and management, smart contract registry and deployment, asset tokenization with NFT support, immutable provenance tracking for chain-of-custody records, blockchain-anchored certification management, supply chain linkage forming directed graphs from raw materials to finished goods, and third-party verification request processing. Integrates with Inventory for product/batch tracking, Procurement for supplier certification verification, Manufacturing for production provenance, Document for certificate storage, and GTM for trade compliance verification.

**Bounded Context:** Blockchain Traceability, Provenance & Digital Thread
**Service Name:** `chain-service`
**Database:** `data/chain.db`
**HTTP Port:** 8074 | **gRPC Port:** 9074

---

## 2. Database Schema

### 2.1 Chain Networks
```sql
CREATE TABLE chain_networks (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    network_name TEXT NOT NULL,
    network_type TEXT NOT NULL
        CHECK(network_type IN ('PRIVATE','HYPERLEDGER','ETHEREUM','MULTICHAIN')),
    consensus_mechanism TEXT NOT NULL,
    node_url TEXT NOT NULL,
    chain_id TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    credentials_encrypted TEXT NOT NULL,          -- Encrypted connection credentials
    config TEXT,                                 -- JSON: network-specific configuration

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, network_name)
);

CREATE INDEX idx_networks_tenant_type ON chain_networks(tenant_id, network_type);
CREATE INDEX idx_networks_tenant_active ON chain_networks(tenant_id, is_active);
```

### 2.2 Chain Smart Contracts
```sql
CREATE TABLE chain_smart_contracts (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    network_id TEXT NOT NULL,
    contract_name TEXT NOT NULL,
    contract_type TEXT NOT NULL
        CHECK(contract_type IN ('TRACEABILITY','CERTIFICATION','PAYMENT','COMPLIANCE')),
    abi TEXT NOT NULL,                           -- Application Binary Interface
    bytecode_hash TEXT NOT NULL,
    deployed_address TEXT,
    deployed_by TEXT,
    deployed_at TEXT,
    version TEXT NOT NULL DEFAULT '1.0.0',
    is_active INTEGER NOT NULL DEFAULT 1,
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (network_id) REFERENCES chain_networks(id),
    UNIQUE(tenant_id, contract_name, version)
);

CREATE INDEX idx_contracts_tenant_network ON chain_smart_contracts(tenant_id, network_id);
CREATE INDEX idx_contracts_tenant_type ON chain_smart_contracts(tenant_id, contract_type);
CREATE INDEX idx_contracts_tenant_active ON chain_smart_contracts(tenant_id, is_active);
```

### 2.3 Chain Assets
```sql
CREATE TABLE chain_assets (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    asset_type TEXT NOT NULL
        CHECK(asset_type IN ('PRODUCT','BATCH','SHIPMENT','CERTIFICATE','DOCUMENT')),
    asset_ref_type TEXT NOT NULL,                -- e.g. "inventory_item", "manufacturing_batch"
    asset_ref_id TEXT NOT NULL,                  -- FK to the originating entity
    token_id TEXT,                               -- Nullable: NFT token ID if minted
    metadata TEXT,                               -- JSON: asset metadata
    is_minted INTEGER NOT NULL DEFAULT 0,
    minted_at TEXT,
    current_owner_did TEXT,                      -- Decentralized Identifier
    provenance_hash TEXT NOT NULL,               -- Hash of provenance data
    status TEXT NOT NULL DEFAULT 'REGISTERED'
        CHECK(status IN ('REGISTERED','MINTED','TRANSFERRED','BURNED')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, asset_ref_type, asset_ref_id)
);

CREATE INDEX idx_assets_tenant_type ON chain_assets(tenant_id, asset_type);
CREATE INDEX idx_assets_tenant_status ON chain_assets(tenant_id, status);
CREATE INDEX idx_assets_tenant_owner ON chain_assets(tenant_id, current_owner_did);
CREATE INDEX idx_assets_tenant_minted ON chain_assets(tenant_id, is_minted);
CREATE INDEX idx_assets_tenant_ref ON chain_assets(tenant_id, asset_ref_type, asset_ref_id);
```

### 2.4 Chain Transactions
```sql
CREATE TABLE chain_transactions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    network_id TEXT NOT NULL,
    contract_id TEXT,
    transaction_hash TEXT NOT NULL,
    block_number INTEGER,
    from_address TEXT NOT NULL,
    to_address TEXT NOT NULL,
    transaction_type TEXT NOT NULL
        CHECK(transaction_type IN ('MINT','TRANSFER','VERIFY','REVOKE')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','CONFIRMED','FAILED')),
    gas_used INTEGER,
    gas_cost_cents INTEGER NOT NULL DEFAULT 0,
    submitted_at TEXT NOT NULL DEFAULT (datetime('now')),
    confirmed_at TEXT,
    payload TEXT,                                -- JSON: transaction data
    error_message TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (network_id) REFERENCES chain_networks(id),
    FOREIGN KEY (contract_id) REFERENCES chain_smart_contracts(id),
    UNIQUE(tenant_id, transaction_hash)
);

CREATE INDEX idx_transactions_tenant_status ON chain_transactions(tenant_id, status);
CREATE INDEX idx_transactions_tenant_type ON chain_transactions(tenant_id, transaction_type);
CREATE INDEX idx_transactions_tenant_dates ON chain_transactions(tenant_id, submitted_at, confirmed_at);
CREATE INDEX idx_transactions_tenant_contract ON chain_transactions(tenant_id, contract_id);
```

### 2.5 Chain Provenance Events
```sql
CREATE TABLE chain_provenance_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    chain_asset_id TEXT NOT NULL,
    event_type TEXT NOT NULL
        CHECK(event_type IN ('CREATED','PROCESSED','TESTED','SHIPPED','RECEIVED','CERTIFIED','STORED')),
    event_data TEXT NOT NULL,                    -- JSON: event-specific data
    location TEXT,
    actor_did TEXT NOT NULL,                     -- Decentralized Identifier of actor
    timestamp_event TEXT NOT NULL,               -- Actual event timestamp
    evidence_hash TEXT,                          -- Hash of supporting evidence
    verification_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(verification_status IN ('PENDING','VERIFIED','REJECTED')),
    verified_by TEXT,
    verified_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (chain_asset_id) REFERENCES chain_assets(id)
);

CREATE INDEX idx_provenance_tenant_asset ON chain_provenance_events(tenant_id, chain_asset_id);
CREATE INDEX idx_provenance_tenant_type ON chain_provenance_events(tenant_id, event_type);
CREATE INDEX idx_provenance_tenant_verification ON chain_provenance_events(tenant_id, verification_status);
CREATE INDEX idx_provenance_tenant_timestamp ON chain_provenance_events(tenant_id, timestamp_event);
CREATE INDEX idx_provenance_tenant_actor ON chain_provenance_events(tenant_id, actor_did);
```

### 2.6 Chain Certifications
```sql
CREATE TABLE chain_certifications (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    chain_asset_id TEXT NOT NULL,
    certification_type TEXT NOT NULL
        CHECK(certification_type IN ('ORGANIC','FAIR_TRADE','ISO','GMP','HALAL','KOSHER','OTHER')),
    certifying_body TEXT NOT NULL,
    certificate_number TEXT NOT NULL,
    issue_date TEXT NOT NULL,
    expiry_date TEXT NOT NULL,
    is_valid INTEGER NOT NULL DEFAULT 1,
    revocation_reason TEXT,
    blockchain_tx_id TEXT,                       -- FK to chain_transactions
    document_hash TEXT,                          -- Hash of the certificate document

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (chain_asset_id) REFERENCES chain_assets(id),
    UNIQUE(tenant_id, certificate_number)
);

CREATE INDEX idx_certifications_tenant_asset ON chain_certifications(tenant_id, chain_asset_id);
CREATE INDEX idx_certifications_tenant_type ON chain_certifications(tenant_id, certification_type);
CREATE INDEX idx_certifications_tenant_valid ON chain_certifications(tenant_id, is_valid);
CREATE INDEX idx_certifications_tenant_expiry ON chain_certifications(tenant_id, expiry_date);
CREATE INDEX idx_certifications_tenant_body ON chain_certifications(tenant_id, certifying_body);
```

### 2.7 Chain Supply Chain Links
```sql
CREATE TABLE chain_supply_chain_links (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    parent_asset_id TEXT NOT NULL,
    child_asset_id TEXT NOT NULL,
    relationship_type TEXT NOT NULL
        CHECK(relationship_type IN ('CONTAINS','DERIVED_FROM','ASSEMBLED_FROM','PACKAGED_WITH')),
    quantity_ratio REAL NOT NULL DEFAULT 1.0,
    linked_at TEXT NOT NULL DEFAULT (datetime('now')),
    linked_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (parent_asset_id) REFERENCES chain_assets(id),
    FOREIGN KEY (child_asset_id) REFERENCES chain_assets(id),
    UNIQUE(tenant_id, parent_asset_id, child_asset_id, relationship_type)
);

CREATE INDEX idx_links_tenant_parent ON chain_supply_chain_links(tenant_id, parent_asset_id);
CREATE INDEX idx_links_tenant_child ON chain_supply_chain_links(tenant_id, child_asset_id);
CREATE INDEX idx_links_tenant_type ON chain_supply_chain_links(tenant_id, relationship_type);
CREATE INDEX idx_links_tenant_active ON chain_supply_chain_links(tenant_id, is_active);
```

### 2.8 Chain Verification Requests
```sql
CREATE TABLE chain_verification_requests (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    chain_asset_id TEXT NOT NULL,
    requester_id TEXT NOT NULL,
    request_type TEXT NOT NULL
        CHECK(request_type IN ('PROVENANCE','CERTIFICATION','AUTHENTICITY','COMPLIANCE')),
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','APPROVED','REJECTED','EXPIRED')),
    response_data TEXT,                          -- JSON: verification response
    requested_at TEXT NOT NULL DEFAULT (datetime('now')),
    responded_at TEXT,
    responder_id TEXT,
    notes TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL, updated_by TEXT NOT NULL,

    FOREIGN KEY (chain_asset_id) REFERENCES chain_assets(id)
);

CREATE INDEX idx_verify_requests_tenant_asset ON chain_verification_requests(tenant_id, chain_asset_id);
CREATE INDEX idx_verify_requests_tenant_status ON chain_verification_requests(tenant_id, status);
CREATE INDEX idx_verify_requests_tenant_requester ON chain_verification_requests(tenant_id, requester_id);
CREATE INDEX idx_verify_requests_tenant_type ON chain_verification_requests(tenant_id, request_type);
CREATE INDEX idx_verify_requests_tenant_dates ON chain_verification_requests(tenant_id, requested_at);
```

---

## 3. REST API Endpoints

```
# Network Management
GET/POST      /api/v1/chain/networks                      Permission: chain.networks.manage
GET/PUT       /api/v1/chain/networks/{id}
GET           /api/v1/chain/networks/{id}/status

# Smart Contracts
GET/POST      /api/v1/chain/contracts                     Permission: chain.contracts.manage
GET/PUT       /api/v1/chain/contracts/{id}
POST          /api/v1/chain/contracts/{id}/deploy         Permission: chain.contracts.deploy

# Asset Registration
GET/POST      /api/v1/chain/assets                        Permission: chain.assets.manage
GET/PUT       /api/v1/chain/assets/{id}
POST          /api/v1/chain/assets/{id}/mint              Permission: chain.assets.mint
GET           /api/v1/chain/assets/{id}/provenance
GET           /api/v1/chain/assets/{id}/history

# Provenance Events
GET/POST      /api/v1/chain/provenance                    Permission: chain.provenance.record
GET           /api/v1/chain/provenance/{asset_id}
POST          /api/v1/chain/provenance/{id}/verify        Permission: chain.provenance.verify

# Certifications
GET/POST      /api/v1/chain/certifications                Permission: chain.certifications.manage
GET/PUT       /api/v1/chain/certifications/{id}
POST          /api/v1/chain/certifications/{id}/revoke    Permission: chain.certifications.revoke
POST          /api/v1/chain/certifications/{id}/anchor    Permission: chain.certifications.anchor

# Supply Chain Links
GET/POST      /api/v1/chain/links                         Permission: chain.links.manage
GET           /api/v1/chain/links/{id}
GET           /api/v1/chain/links/trace/{asset_id}

# Verification
POST          /api/v1/chain/verify                        Permission: chain.verify
GET           /api/v1/chain/verify/{id}
POST          /api/v1/chain/verify/{id}/respond           Permission: chain.verify.respond

# Transactions
GET           /api/v1/chain/transactions
GET           /api/v1/chain/transactions/{id}
GET           /api/v1/chain/transactions/by-hash/{hash}

# Reports
GET           /api/v1/chain/reports/traceability
GET           /api/v1/chain/reports/certification-status
GET           /api/v1/chain/reports/supply-chain-map
```

---

## 4. Business Rules

### 4.1 Asset Registration
```
Each product/batch can be registered as a blockchain asset with unique token:
  1. Assets are registered before minting (two-phase lifecycle)
  2. Registration requires: asset_type, asset_ref_type, asset_ref_id, provenance_hash
  3. One blockchain asset per (asset_ref_type, asset_ref_id) combination
  4. Metadata stored as JSON supports flexible asset descriptors
  5. Status transitions: REGISTERED -> MINTED -> TRANSFERRED -> BURNED
  6. Burned assets are permanently invalidated and cannot be reactivated
```

### 4.2 Provenance Tracking
- Provenance events create an immutable chain-of-custody record
- Each event includes actor_did for decentralized identity attribution
- Evidence hash ensures tamper-evident record keeping
- Events can only be verified by authorized parties
- Provenance events are append-only; no updates or deletions permitted
- Location data captured for geographic traceability

### 4.3 Smart Contract Management
- Smart contract interactions are gas-optimized and batched where possible
- Contract deployment requires: ABI, bytecode_hash, target network
- Contract versioning follows semantic versioning (major.minor.patch)
- Only one active version of a contract per name allowed
- Contract upgrades create new records; old versions are deactivated
- Deployment transaction tracked in chain_transactions

### 4.4 Certification Management
- Certifications anchored to blockchain are tamper-evident via hash verification
- Document hash is computed from the original certificate document
- Certification anchoring creates a blockchain transaction for immutability
- Expired certifications (past expiry_date) are automatically marked is_valid = 0
- Revocation requires a reason and creates a blockchain transaction
- Revoked certifications cannot be re-activated (new certificate required)

### 4.5 Supply Chain Linkage
- Supply chain links form a directed graph tracing raw materials to finished goods
- Relationship types define the nature of the link:
  - CONTAINS: parent includes child (e.g. shipment contains products)
  - DERIVED_FROM: child produced from parent (e.g. product from raw material)
  - ASSEMBLED_FROM: child assembled from parent (e.g. finished good from components)
  - PACKAGED_WITH: parent packaged with child (e.g. product with label)
- Circular references are prevented during link creation
- Quantity ratio tracks proportional relationships
- Links are soft-deactivated (is_active = 0) rather than deleted

### 4.6 Transaction Processing
- Failed transactions are retried with exponential backoff up to 3 attempts
- Backoff intervals: 30s, 120s, 300s
- Gas cost tracked in cents for cost accounting
- Transaction status must be CONFIRMED before dependent operations proceed
- Transaction hash uniqueness enforced at database level
- Pending transactions older than 1 hour are flagged for investigation

### 4.7 Verification
- Verification requests can be made by any party with appropriate access
- Request types: PROVENANCE (trace history), CERTIFICATION (validate cert), AUTHENTICITY (verify asset), COMPLIANCE (regulatory check)
- Requests expire after 30 days if not responded to
- Response data includes verification evidence and verdict
- Approved verifications update the provenance event verification_status

### 4.8 Events Published
| Event | Trigger | Consumers |
|-------|---------|-----------|
| `chain.asset.registered` | Asset registered in system | Inventory, Notification |
| `chain.asset.minted` | Asset minted on blockchain | Notification, GL |
| `chain.provenance.recorded` | Provenance event recorded | Inventory, Manufacturing |
| `chain.certification.anchored` | Certification anchored to chain | GTM, Procurement |
| `chain.certification.revoked` | Certification revoked | GTM, Procurement |
| `chain.transaction.confirmed` | Blockchain transaction confirmed | All consumers |
| `chain.verification.requested` | Verification request created | Workflow |

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.chain.v1;

service ChainService {
    rpc RegisterAsset(RegisterAssetRequest) returns (RegisterAssetResponse);
    rpc RecordProvenance(RecordProvenanceRequest) returns (RecordProvenanceResponse);
    rpc GetProvenanceHistory(GetProvenanceHistoryRequest) returns (GetProvenanceHistoryResponse);
    rpc VerifyCertification(VerifyCertificationRequest) returns (VerifyCertificationResponse);
    rpc TraceSupplyChain(TraceSupplyChainRequest) returns (TraceSupplyChainResponse);
}
```

---

## 6. Inter-Service Integration

### 6.1 Data Consumed From
- **Inventory:** Item and batch details for asset registration, lot/serial tracking
- **Procurement:** Supplier information for certification verification, PO details
- **Manufacturing:** Production batch data for provenance events, work order completion
- **Document:** Certificate document storage, evidence document storage
- **GTM:** Trade compliance status for cross-border asset verification
- **Auth:** User identity for decentralized identifier (DID) mapping

### 6.2 Data Published To
- **Inventory:** Asset registration status, provenance updates for lot traceability
- **Procurement:** Supplier certification status, verified supplier credentials
- **Manufacturing:** Provenance recording confirmation, production batch on-chain status
- **GTM:** Certification verification results, blockchain-anchored compliance evidence
- **GL:** Gas cost journal entries, blockchain transaction fee accounting
- **Workflow:** Verification request approval routing, certification expiry workflows
- **Notification:** Asset minting notifications, transaction confirmation alerts

---

## 7. Migrations

1. V001: `chain_networks`
2. V002: `chain_smart_contracts`
3. V003: `chain_assets`
4. V004: `chain_transactions`
5. V005: `chain_provenance_events`
6. V006: `chain_certifications`
7. V007: `chain_supply_chain_links`
8. V008: `chain_verification_requests`
9. V009: Triggers for `updated_at`
