# 124 - Sourcing Specification

## 1. Domain Overview

Sourcing manages the strategic procurement process of soliciting, evaluating, and awarding supplier bids through Requests for Quotation (RFQs), Requests for Proposal (RFPs), and auctions. It enables competitive bidding, supplier qualification, bid analysis, and optimal award decisions to reduce procurement costs and improve supplier performance.

**Bounded Context:** Strategic Sourcing & Supplier Selection
**Service Name:** `sourcing-service`
**Database:** `data/sourcing.db`
**HTTP Port:** 8204 | **gRPC Port:** 9204

---

## 2. Database Schema

### 2.1 Sourcing Events
```sql
CREATE TABLE src_events (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_number TEXT NOT NULL,
    event_name TEXT NOT NULL,
    description TEXT,
    event_type TEXT NOT NULL CHECK(event_type IN ('RFQ','RFP','RFI','REVERSE_AUCTION','FORWARD_AUCTION','SEALED_BID')),
    status TEXT NOT NULL DEFAULT 'DRAFT'
        CHECK(status IN ('DRAFT','PUBLISHED','BIDDING_OPEN','BIDDING_CLOSED','EVALUATION','AWARDED','CANCELLED')),
    currency_code TEXT NOT NULL DEFAULT 'USD',
    evaluation_method TEXT NOT NULL DEFAULT 'LOWEST_PRICE'
        CHECK(evaluation_method IN ('LOWEST_PRICE','BEST_VALUE','WEIGHTED_SCORE','MANUAL')),
    open_date TEXT NOT NULL,
    close_date TEXT NOT NULL,
    award_date TEXT,
    total_estimated_value_cents INTEGER,
    buyer_id TEXT NOT NULL,
    confidentiality TEXT NOT NULL DEFAULT 'OPEN'
        CHECK(confidentiality IN ('OPEN','SEALED','BLIND')),

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, event_number)
);

CREATE INDEX idx_src_events_tenant_status ON src_events(tenant_id, status);
CREATE INDEX idx_src_events_tenant_dates ON src_events(tenant_id, open_date, close_date);
```

### 2.2 Sourcing Lines
```sql
CREATE TABLE src_lines (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    line_number INTEGER NOT NULL,
    item_id TEXT,
    item_description TEXT NOT NULL,
    category_id TEXT,
    quantity REAL NOT NULL,
    uom TEXT NOT NULL,
    target_price_cents INTEGER,
    delivery_date TEXT,
    delivery_location TEXT,
    specifications TEXT,                 -- JSON: technical requirements
    evaluation_weight REAL NOT NULL DEFAULT 1.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (event_id) REFERENCES src_events(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id, line_number)
);
```

### 2.3 Invited Suppliers
```sql
CREATE TABLE src_suppliers (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    invitation_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(invitation_status IN ('PENDING','ACCEPTED','DECLINED','DISQUALIFIED')),
    invited_date TEXT NOT NULL DEFAULT (datetime('now')),
    responded_date TEXT,
    disqualification_reason TEXT,
    contact_email TEXT,
    contact_name TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,

    FOREIGN KEY (event_id) REFERENCES src_events(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id, supplier_id)
);
```

### 2.4 Supplier Bids
```sql
CREATE TABLE src_bids (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    line_id TEXT NOT NULL,
    bid_price_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    proposed_quantity REAL,
    proposed_delivery_date TEXT,
    lead_time_days INTEGER,
    validity_date TEXT NOT NULL,         -- Bid valid until
    payment_terms TEXT,
    specifications_compliance INTEGER NOT NULL DEFAULT 1,
    technical_score REAL,
    total_score REAL,
    rank INTEGER,
    is_recommended INTEGER NOT NULL DEFAULT 0,
    notes TEXT,
    attachments TEXT,                    -- JSON array of attachment IDs

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (event_id) REFERENCES src_events(id) ON DELETE CASCADE,
    FOREIGN KEY (line_id) REFERENCES src_lines(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id, supplier_id, line_id)
);

CREATE INDEX idx_src_bids_tenant_event ON src_bids(tenant_id, event_id);
CREATE INDEX idx_src_bids_tenant_supplier ON src_bids(tenant_id, supplier_id);
```

### 2.5 Evaluation Criteria
```sql
CREATE TABLE src_evaluation_criteria (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    criterion_name TEXT NOT NULL,
    criterion_type TEXT NOT NULL CHECK(criterion_type IN ('PRICE','QUALITY','DELIVERY','TECHNICAL','FINANCIAL','EXPERIENCE','SUSTAINABILITY')),
    weight_pct REAL NOT NULL,            -- Must total 100% per event
    scoring_method TEXT NOT NULL DEFAULT 'MANUAL'
        CHECK(scoring_method IN ('MANUAL','FORMULA','AUTO_PRICE')),
    scoring_formula TEXT,
    description TEXT,
    max_score REAL NOT NULL DEFAULT 100.0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (event_id) REFERENCES src_events(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id, criterion_name)
);
```

### 2.6 Award Decisions
```sql
CREATE TABLE src_awards (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    line_id TEXT NOT NULL,
    supplier_id TEXT NOT NULL,
    awarded_quantity REAL NOT NULL,
    awarded_price_cents INTEGER NOT NULL,
    currency_code TEXT NOT NULL DEFAULT 'USD',
    award_percentage REAL,               -- For split awards (e.g., 60%/40%)
    status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(status IN ('PENDING','ACCEPTED','REJECTED','CONVERTED')),
    award_justification TEXT,
    purchase_order_id TEXT,              -- Link to PO after conversion

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    FOREIGN KEY (event_id) REFERENCES src_events(id) ON DELETE CASCADE,
    FOREIGN KEY (line_id) REFERENCES src_lines(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id, line_id, supplier_id)
);
```

### 2.7 Auction State
```sql
CREATE TABLE src_auction_state (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    event_id TEXT NOT NULL,
    current_best_price_cents INTEGER,
    current_best_supplier_id TEXT,
    bid_count INTEGER NOT NULL DEFAULT 0,
    minimum_decrement_pct REAL NOT NULL DEFAULT 1.0,
    extension_minutes INTEGER NOT NULL DEFAULT 5,  -- Auto-extend if bid in last N min
    start_price_cents INTEGER,
    reserve_price_cents INTEGER,
    auction_start TEXT,
    auction_end TEXT,
    is_extended INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (event_id) REFERENCES src_events(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, event_id)
);
```

---

## 3. REST API Endpoints

### 3.1 Sourcing Events
```
GET    /api/v1/sourcing/events                           Permission: src.events.read
GET    /api/v1/sourcing/events/{id}                      Permission: src.events.read
POST   /api/v1/sourcing/events                           Permission: src.events.create
PUT    /api/v1/sourcing/events/{id}                      Permission: src.events.update
POST   /api/v1/sourcing/events/{id}/publish              Permission: src.events.publish
POST   /api/v1/sourcing/events/{id}/cancel               Permission: src.events.cancel
GET    /api/v1/sourcing/events/{id}/lines                 Permission: src.events.read
POST   /api/v1/sourcing/events/{id}/lines                Permission: src.events.update
```

### 3.2 Supplier Invitations
```
GET    /api/v1/sourcing/events/{id}/suppliers             Permission: src.suppliers.read
POST   /api/v1/sourcing/events/{id}/suppliers             Permission: src.suppliers.invite
PUT    /api/v1/sourcing/events/{id}/suppliers/{sid}       Permission: src.suppliers.update
POST   /api/v1/sourcing/events/{id}/suppliers/{sid}/disqualify  Permission: src.suppliers.disqualify
```

### 3.3 Bidding
```
GET    /api/v1/sourcing/events/{id}/bids                  Permission: src.bids.read
POST   /api/v1/sourcing/events/{id}/bids                  Permission: src.bids.submit
  Request: { "supplier_id": "...", "line_bids": [{ "line_id": "...", "price_cents": 50000, "delivery_date": "..." }] }
PUT    /api/v1/sourcing/bids/{id}                         Permission: src.bids.update
POST   /api/v1/sourcing/events/{id}/close-bidding         Permission: src.events.close
GET    /api/v1/sourcing/events/{id}/bids/comparison       Permission: src.bids.read
  Response: Bid comparison matrix with rankings
```

### 3.4 Evaluation
```
GET    /api/v1/sourcing/events/{id}/criteria              Permission: src.evaluation.read
POST   /api/v1/sourcing/events/{id}/criteria              Permission: src.evaluation.setup
POST   /api/v1/sourcing/events/{id}/evaluate              Permission: src.evaluation.score
GET    /api/v1/sourcing/events/{id}/scores                Permission: src.evaluation.read
```

### 3.5 Awards
```
POST   /api/v1/sourcing/events/{id}/award                 Permission: src.awards.create
  Request: { "awards": [{ "line_id": "...", "supplier_id": "...", "quantity": 100, "price_cents": 50000 }] }
GET    /api/v1/sourcing/events/{id}/awards                Permission: src.awards.read
PUT    /api/v1/sourcing/awards/{id}/accept                Permission: src.awards.accept
PUT    /api/v1/sourcing/awards/{id}/reject                Permission: src.awards.reject
POST   /api/v1/sourcing/awards/{id}/convert-to-po         Permission: src.awards.convert
```

### 3.6 Auctions
```
GET    /api/v1/sourcing/events/{id}/auction               Permission: src.auctions.read
POST   /api/v1/sourcing/events/{id}/auction/start         Permission: src.auctions.start
POST   /api/v1/sourcing/events/{id}/auction/bid           Permission: src.auctions.bid
GET    /api/v1/sourcing/events/{id}/auction/history       Permission: src.auctions.read
```

---

## 4. Business Rules

### 4.1 Event Lifecycle
- DRAFT → PUBLISHED → BIDDING_OPEN → BIDDING_CLOSED → EVALUATION → AWARDED
- Events can be cancelled at any stage before AWARDED
- Bidding cannot open before open_date or after close_date
- Automatic close when close_date is reached

### 4.2 Bid Rules
- Suppliers MUST be invited before bidding (no unsolicited bids)
- Late bids (after close_date) are rejected automatically
- Sealed bids are hidden until bidding closes
- Blind events hide supplier identity during evaluation
- Bid price MUST be positive
- Suppliers can update bids only while bidding is open

### 4.3 Evaluation Rules
- Evaluation criteria weights MUST total 100%
- WEIGHTED_SCORE method: total_score = SUM(criterion_score × weight)
- LOWEST_PRICE method: automatic ranking by bid price
- BEST_VALUE method: combines price and qualitative scores
- Ties broken by delivery date (earliest wins)

### 4.4 Award Rules
- Award quantity MUST NOT exceed sourcing line quantity
- Split awards allowed (multiple suppliers per line)
- Award price MUST NOT exceed bid price
- Awards generate purchase orders in proc-service via gRPC
- Suppliers MUST accept award before PO is created

### 4.5 Auction Rules
- Reverse auctions: suppliers bid progressively lower prices
- Minimum decrement enforced (bid must be at least X% below current best)
- Auto-extension: auction extended if bid placed in final minutes
- Reserve price: award not made if best bid exceeds reserve
- Real-time leaderboard visible to all participating suppliers

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";
package fusion.sourcing.v1;

service SourcingService {
    rpc CreateSourcingEvent(CreateSourcingEventRequest) returns (CreateSourcingEventResponse);
    rpc SubmitBid(SubmitBidRequest) returns (SubmitBidResponse);
    rpc EvaluateBids(EvaluateBidsRequest) returns (EvaluateBidsResponse);
    rpc CreateAward(CreateAwardRequest) returns (CreateAwardResponse);
    rpc ConvertToPO(ConvertToPORequest) returns (ConvertToPOResponse);
    rpc GetBidComparison(GetBidComparisonRequest) returns (GetBidComparisonResponse);
}

// Entity messages
message SrcEvent {
    string id = 1;
    string tenant_id = 2;
    string event_number = 3;
    string event_name = 4;
    string description = 5;
    string event_type = 6;
    string status = 7;
    string currency_code = 8;
    string evaluation_method = 9;
    string open_date = 10;
    string close_date = 11;
    string award_date = 12;
    int64 total_estimated_value_cents = 13;
    string buyer_id = 14;
    string confidentiality = 15;
    string created_at = 16;
    string updated_at = 17;
}

message SrcLine {
    string id = 1;
    string tenant_id = 2;
    string event_id = 3;
    int32 line_number = 4;
    string item_id = 5;
    string item_description = 6;
    string category_id = 7;
    double quantity = 8;
    string uom = 9;
    int64 target_price_cents = 10;
    string delivery_date = 11;
    string delivery_location = 12;
    string specifications = 13;
    double evaluation_weight = 14;
    string created_at = 15;
    string updated_at = 16;
}

message SrcBid {
    string id = 1;
    string tenant_id = 2;
    string event_id = 3;
    string supplier_id = 4;
    string line_id = 5;
    int64 bid_price_cents = 6;
    string currency_code = 7;
    double proposed_quantity = 8;
    string proposed_delivery_date = 9;
    int32 lead_time_days = 10;
    string validity_date = 11;
    string payment_terms = 12;
    int32 specifications_compliance = 13;
    double technical_score = 14;
    double total_score = 15;
    int32 rank = 16;
    bool is_recommended = 17;
    string notes = 18;
    string created_at = 19;
    string updated_at = 20;
}

message SrcAward {
    string id = 1;
    string tenant_id = 2;
    string event_id = 3;
    string line_id = 4;
    string supplier_id = 5;
    double awarded_quantity = 6;
    int64 awarded_price_cents = 7;
    string currency_code = 8;
    double award_percentage = 9;
    string status = 10;
    string award_justification = 11;
    string purchase_order_id = 12;
    string created_at = 13;
    string updated_at = 14;
}

message SrcEvaluationCriterion {
    string id = 1;
    string tenant_id = 2;
    string event_id = 3;
    string criterion_name = 4;
    string criterion_type = 5;
    double weight_pct = 6;
    string scoring_method = 7;
    string scoring_formula = 8;
    string description = 9;
    double max_score = 10;
    string created_at = 11;
    string updated_at = 12;
}

// Request/Response messages
message CreateSourcingEventRequest {
    string tenant_id = 1;
    string event_name = 2;
    string description = 3;
    string event_type = 4;
    string currency_code = 5;
    string evaluation_method = 6;
    string open_date = 7;
    string close_date = 8;
    int64 total_estimated_value_cents = 9;
    string buyer_id = 10;
    string confidentiality = 11;
}

message CreateSourcingEventResponse {
    SrcEvent data = 1;
}

message BidLineInput {
    string line_id = 1;
    int64 bid_price_cents = 2;
    double proposed_quantity = 3;
    string proposed_delivery_date = 4;
    int32 lead_time_days = 5;
    string validity_date = 6;
    string payment_terms = 7;
    string notes = 8;
}

message SubmitBidRequest {
    string tenant_id = 1;
    string event_id = 2;
    string supplier_id = 3;
    repeated BidLineInput line_bids = 4;
}

message SubmitBidResponse {
    repeated SrcBid bids = 1;
}

message EvaluateBidsRequest {
    string tenant_id = 1;
    string event_id = 2;
    string evaluation_method = 3;
}

message ScoredBid {
    string bid_id = 1;
    string supplier_id = 2;
    string line_id = 3;
    double total_score = 4;
    int32 rank = 5;
    bool is_recommended = 6;
}

message EvaluateBidsResponse {
    repeated ScoredBid scored_bids = 1;
}

message AwardInput {
    string line_id = 1;
    string supplier_id = 2;
    double awarded_quantity = 3;
    int64 awarded_price_cents = 4;
    double award_percentage = 5;
    string award_justification = 6;
}

message CreateAwardRequest {
    string tenant_id = 1;
    string event_id = 2;
    repeated AwardInput awards = 3;
}

message CreateAwardResponse {
    repeated SrcAward awards = 1;
}

message ConvertToPORequest {
    string tenant_id = 1;
    string award_id = 2;
}

message ConvertToPOResponse {
    SrcAward award = 1;
    string purchase_order_id = 2;
}

message GetBidComparisonRequest {
    string tenant_id = 1;
    string event_id = 2;
}

message BidComparisonRow {
    string line_id = 1;
    string item_description = 2;
    repeated SrcBid bids = 3;
}

message GetBidComparisonResponse {
    repeated BidComparisonRow rows = 1;
    repeated SrcEvaluationCriterion criteria = 2;
}
```

---

## 6. Inter-Service Integration

### 6.1 Dependencies
- **proc-service**: Award conversion to purchase orders
- **supplier-service**: Supplier profiles, qualifications
- **contracts-service**: Long-term contract creation from awards
- **inv-service**: Item master data for sourcing lines
- **dms-service**: Bid documents, specifications, attachments
- **workflow-service**: Approval routing for awards
- **ai-service**: Bid analysis recommendations, anomaly detection

### 6.2 Events Published

| Event | Trigger | Payload |
|-------|---------|---------|
| `src.event.published` | Sourcing event published | event_id, event_type, close_date |
| `src.bid.submitted` | Supplier submits bid | event_id, supplier_id, line_id |
| `src.bidding.closed` | Bidding period ended | event_id, bid_count |
| `src.award.created` | Award decision made | event_id, supplier_id, line_id, amount |
| `src.award.accepted` | Supplier accepts award | award_id, supplier_id |
| `src.award.converted` | Award converted to PO | award_id, purchase_order_id |
| `src.auction.bid.placed` | Auction bid placed | event_id, supplier_id, new_best_price |

---

## 7. Migrations

### Migration Order for sourcing-service:
1. V001: `src_events`
2. V002: `src_lines`
3. V003: `src_suppliers`
4. V004: `src_bids`
5. V005: `src_evaluation_criteria`
6. V006: `src_awards`
7. V007: `src_auction_state`
8. V008: Triggers for `updated_at`
9. V009: Seed data (default evaluation criteria templates)
