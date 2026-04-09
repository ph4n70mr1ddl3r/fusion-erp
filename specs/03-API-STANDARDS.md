# 03 - API Design Standards Specification

## 1. RESTful API Conventions

### 1.1 URL Structure
All external URLs follow this pattern:
```
/api/v{version}/{service-prefix}/{resource}
```

Examples:
```
/api/v1/gl/accounts
/api/v1/gl/journals
/api/v1/ap/invoices
/api/v1/ar/customers
/api/v1/inv/items
/api/v1/proc/purchase-orders
```

### 1.2 URL Rules
- Resource names MUST be plural (`/accounts` not `/account`)
- Multi-word resources MUST use kebab-case (`/purchase-orders`, `/journal-entries`)
- Nested resources for parent-child relationships: `/api/v1/ap/invoices/{id}/lines`
- Maximum nesting depth: 2 levels (`/resource/{id}/sub-resource/{sub-id}/items`)
- No verbs in URLs — use action endpoints for operations

### 1.3 HTTP Method Usage

| Method | Purpose | Idempotent | Request Body | Response Body |
|--------|---------|------------|-------------|---------------|
| GET | Read resource/collection | Yes | No | Yes |
| POST | Create resource or trigger action | No | Yes | Yes |
| PUT | Full replacement of resource | Yes | Yes | Yes |
| PATCH | Partial update of resource | No | Yes | Yes |
| DELETE | Soft-delete (archive) resource | Yes | No | No (204) |

### 1.4 Action Endpoints
Operations that don't map to CRUD use POST with a verb sub-path:
```
POST /api/v1/gl/journals/{id}/post        # Post a journal entry
POST /api/v1/gl/journals/{id}/reverse      # Reverse a journal entry
POST /api/v1/gl/periods/{id}/close         # Close a period
POST /api/v1/ap/payments/run               # Execute payment run
POST /api/v1/inv/items/{id}/adjust         # Adjust inventory
POST /api/v1/om/orders/{id}/fulfill        # Fulfill an order
POST /api/v1/auth/login                    # Login
POST /api/v1/auth/refresh                  # Refresh token
```

---

## 2. Request/Response Standards

### 2.1 Content Type
- Request: `Content-Type: application/json`
- Response: `Content-Type: application/json; charset=utf-8`

### 2.2 Success Response Envelope
```json
{
  "data": {
    "id": "0192a3b4-c5d6-7e8f-9a0b-1c2d3e4f5a6b",
    "account_code": "1000-010-0000",
    "account_name": "Cash - Operating",
    "account_type": "ASSET"
  },
  "meta": {
    "request_id": "req-0192a3b4-0001-7000-0000-000000000001",
    "timestamp": "2024-01-15T10:30:00Z",
    "service": "gl-service"
  }
}
```

### 2.3 Collection Response Envelope
```json
{
  "data": [
    { "id": "...", "account_code": "...", "account_name": "..." }
  ],
  "meta": {
    "request_id": "req-...",
    "timestamp": "2024-01-15T10:30:00Z",
    "service": "gl-service",
    "pagination": {
      "cursor": "eyJpZCI6IjAxOTJhM2I0In0=",
      "has_more": true,
      "total_count": 245
    }
  }
}
```

### 2.4 Error Response Format
```json
{
  "error": {
    "code": "GL_JOURNAL_001",
    "message": "Journal entry is unbalanced",
    "details": [
      {
        "field": "lines",
        "issue": "Total debits (10000) do not equal total credits (9500)",
        "value": null,
        "suggestion": "Adjust line amounts so total debits equal total credits"
      }
    ],
    "request_id": "req-...",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### 2.5 Validation Error Response
```json
{
  "error": {
    "code": "GEN_VALIDATION_001",
    "message": "Request validation failed",
    "details": [
      {
        "field": "account_code",
        "issue": "Account code must match pattern: \\d{4}-\\d{3}-\\d{4}",
        "value": "100",
        "suggestion": "Use format like 1000-010-0000"
      },
      {
        "field": "journal_date",
        "issue": "Date must be in ISO 8601 format (YYYY-MM-DD)",
        "value": "01/15/2024",
        "suggestion": "Use format like 2024-01-15"
      }
    ],
    "request_id": "req-...",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

---

## 3. Pagination

### 3.1 Cursor-Based Pagination
All collection endpoints MUST support cursor-based pagination.

**Request Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `cursor` | string | null | Opaque cursor from previous response |
| `limit` | integer | 50 | Number of items to return (max: 200) |

**Example:**
```
GET /api/v1/gl/accounts?limit=50
GET /api/v1/gl/accounts?cursor=eyJpZCI6IjAxOTIifQ==&limit=50
```

**Response includes pagination metadata:**
```json
{
  "meta": {
    "pagination": {
      "cursor": "eyJpZCI6IjAxOTJhM2I0LWM1ZDYtN2U4Zi05YTBiNTc2M2RlNGYifQ==",
      "has_more": true,
      "total_count": 245
    }
  }
}
```

**Cursor encoding:** Base64-encoded JSON of the last item's sort key:
```json
{"id": "0192a3b4-c5d6-7e8f-9a0b-5763de4f"}
```

---

## 4. Filtering, Sorting, and Field Selection

### 4.1 Filtering
Use bracket notation for filters:
```
?filter[status]=POSTED
?filter[account_type]=ASSET
?filter[date_from]=2024-01-01&filter[date_to]=2024-01-31
?filter[amount_min]=10000&filter[amount_max]=50000
```

Multiple values for the same filter (OR):
```
?filter[status]=DRAFT&filter[status]=APPROVED
```

### 4.2 Sorting
Prefix with `-` for descending, `+` for ascending:
```
?sort=-created_at           # Most recent first
?sort=+account_code         # Ascending by code
?sort=-amount,+created_at   # Sort by amount desc, then date asc
```

Default sort: `-created_at` (most recent first)

### 4.3 Field Selection
Comma-separated list of fields to return:
```
?fields=id,account_code,account_name
```

Returns only the specified fields in the response data objects.

---

## 5. HTTP Status Codes

### 5.1 Success Codes
| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, PATCH, or action POST |
| 201 | Created | Successful POST that creates a resource |
| 204 | No Content | Successful DELETE |

### 5.2 Client Error Codes
| Code | Meaning | Usage |
|------|---------|-------|
| 400 | Bad Request | Malformed request, invalid JSON |
| 401 | Unauthorized | Missing or invalid JWT token |
| 403 | Forbidden | Valid token but insufficient permissions |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Duplicate resource, optimistic lock failure |
| 422 | Unprocessable Entity | Valid JSON but business rule violation |
| 429 | Too Many Requests | Rate limit exceeded |

### 5.3 Server Error Codes
| Code | Meaning | Usage |
|------|---------|-------|
| 500 | Internal Server Error | Unexpected server error |
| 503 | Service Unavailable | Dependency failure, circuit breaker open |

---

## 6. Request Headers

### 6.1 Standard Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | `Bearer {jwt_token}` for all endpoints except login/refresh |
| `Content-Type` | Conditional | `application/json` for POST/PUT/PATCH |
| `Accept` | No | `application/json` (default) |
| `X-Tenant-Id` | Yes | Tenant identifier (validated against JWT) |
| `X-Request-Id` | No | Client-provided request ID; server generates if missing |
| `X-Idempotency-Key` | Conditional | Required for all POST create endpoints |
| `If-Match` | No | ETag for optimistic concurrency control |

### 6.2 Response Headers

| Header | Description |
|--------|-------------|
| `X-Request-Id` | Request ID (echoed or generated) |
| `X-RateLimit-Limit` | Rate limit ceiling |
| `X-RateLimit-Remaining` | Remaining requests in window |
| `X-RateLimit-Reset` | Seconds until rate limit resets |
| `ETag` | Entity tag for optimistic locking |
| `X-Response-Time` | Request processing time in ms |

---

## 7. Idempotency

### 7.1 Idempotency Key
All POST endpoints that create resources MUST support `X-Idempotency-Key`.

**Flow:**
1. Client sends POST with `X-Idempotency-Key: {uuid}`
2. Server checks if key exists in idempotency store
3. If exists: return stored response (200) without re-processing
4. If not exists: process request, store response with key, return (201)
5. Keys expire after 24 hours

**Idempotency Store Table (per service):**
```sql
CREATE TABLE idempotency_keys (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    request_hash TEXT NOT NULL,
    response_status INTEGER NOT NULL,
    response_body TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    expires_at TEXT NOT NULL
);
CREATE INDEX idx_idempotency_tenant_expires ON idempotency_keys(tenant_id, expires_at);
```

---

## 8. Bulk Operations

### 8.1 Bulk Endpoint Pattern
```
POST /api/v1/{service}/{resource}/bulk
```

**Request:**
```json
{
  "operations": [
    { "action": "create", "data": { "account_code": "1100", "account_name": "Petty Cash" } },
    { "action": "update", "id": "0192...", "data": { "account_name": "Updated Name" } },
    { "action": "delete", "id": "0192..." }
  ]
}
```

### 8.2 Bulk Response
```json
{
  "data": {
    "succeeded": 8,
    "failed": 2,
    "results": [
      { "index": 0, "action": "create", "status": 201, "data": { "id": "..." } },
      { "index": 1, "action": "update", "status": 200, "data": { "id": "..." } },
      { "index": 2, "action": "create", "status": 422, "error": { "code": "...", "message": "..." } }
    ]
  }
}
```

### 8.3 Limits
- Maximum 100 operations per bulk request
- Each operation validated independently
- Partial success allowed (some succeed, some fail)
- Transactional for creates only (all-or-nothing within a single create batch)

---

## 9. gRPC Service Standards

### 9.1 Proto Package Naming
```protobuf
package fusion.{service}.v1;
```

Example:
```protobuf
package fusion.gl.v1;
package fusion.ap.v1;
package fusion.inv.v1;
```

### 9.2 Service Naming
PascalCase, suffixed with `Service`:
```protobuf
service GeneralLedgerService { ... }
service AccountsPayableService { ... }
service InventoryService { ... }
```

### 9.3 Message Naming Conventions
- Request: `{Method}Request`
- Response: `{Method}Response`
- Resource: `{Resource}` (singular)
- Collection: `{Resource}List`

### 9.4 Standard Proto Messages

```protobuf
// Common messages shared across all services

message Money {
    int64 cents = 1;
    string currency = 2;  // ISO 4217
}

message PaginationRequest {
    string cursor = 1;
    int32 limit = 2;
}

message PaginationResponse {
    string next_cursor = 1;
    bool has_more = 2;
    int32 total_count = 3;
}

message TenantId {
    string value = 1;
}

message UserId {
    string value = 1;
}

message AuditInfo {
    string created_at = 1;
    string updated_at = 2;
    string created_by = 3;
    string updated_by = 4;
    int64 version = 5;
}

message ErrorResponse {
    string code = 1;
    string message = 2;
    repeated FieldError details = 3;
}

message FieldError {
    string field = 1;
    string issue = 2;
    string value = 3;
}
```

### 9.5 Example Service Proto (GL)

```protobuf
syntax = "proto3";
package fusion.gl.v1;

import "common.proto";

service GeneralLedgerService {
    rpc GetAccount(GetAccountRequest) returns (GetAccountResponse);
    rpc CreateJournal(CreateJournalRequest) returns (CreateJournalResponse);
    rpc PostJournal(PostJournalRequest) returns (PostJournalResponse);
    rpc GetBalances(GetBalancesRequest) returns (GetBalancesResponse);
    rpc ClosePeriod(ClosePeriodRequest) returns (ClosePeriodResponse);
}

message GetAccountRequest {
    fusion.gl.v1.TenantId tenant_id = 1;
    string account_id = 2;
}

message GetAccountResponse {
    Account account = 1;
}

message Account {
    string id = 1;
    string tenant_id = 2;
    string account_code = 3;
    string account_name = 4;
    string account_type = 5;
    bool is_postable = 6;
    AuditInfo audit = 10;
}

message CreateJournalRequest {
    string tenant_id = 1;
    string description = 2;
    string journal_date = 3;
    string currency_code = 4;
    repeated JournalLineInput lines = 5;
    string source = 6;
    string reference_type = 7;
    string reference_id = 8;
}

message JournalLineInput {
    string account_id = 1;
    string description = 2;
    int64 entered_debit_cents = 3;
    int64 entered_credit_cents = 4;
    string currency_code = 5;
}

message CreateJournalResponse {
    string journal_id = 1;
    string journal_number = 2;
}

message PostJournalRequest {
    string tenant_id = 1;
    string journal_id = 2;
}

message PostJournalResponse {
    string journal_id = 1;
    string posted_at = 2;
}

message GetBalancesRequest {
    string tenant_id = 1;
    string account_id = 2;
    string period_name = 3;
    string balance_type = 4;
}

message GetBalancesResponse {
    repeated AccountBalance balances = 1;
}

message AccountBalance {
    string account_id = 1;
    string period_name = 2;
    int64 beginning_balance_cents = 3;
    int64 period_debits_cents = 4;
    int64 period_credits_cents = 5;
    int64 ending_balance_cents = 6;
    string currency_code = 7;
}

message ClosePeriodRequest {
    string tenant_id = 1;
    string period_id = 2;
}

message ClosePeriodResponse {
    string period_id = 1;
    string closed_at = 2;
}
```

### 9.6 gRPC Error Mapping

| Business Error | gRPC Status |
|---------------|-------------|
| Resource not found | NOT_FOUND |
| Validation failure | INVALID_ARGUMENT |
| Permission denied | PERMISSION_DENIED |
| Unauthenticated | UNAUTHENTICATED |
| Conflict (duplicate) | ALREADY_EXISTS |
| Business rule violation | FAILED_PRECONDITION |
| Internal error | INTERNAL |
| Service unavailable | UNAVAILABLE |

---

## 10. Versioning

### 10.1 URL Versioning
- API version in URL path: `/api/v1/...`
- Version incremented only for breaking changes
- Non-breaking changes (new fields, new endpoints) do not require version bump
- Maximum 2 versions supported simultaneously; oldest deprecated for 6 months

### 10.2 Backward Compatibility Rules (within a version)
- MUST add new optional fields (never required)
- MUST NOT remove existing fields
- MUST NOT rename existing fields
- MUST NOT change field types
- MAY add new endpoints
- MAY add new enum values (clients must handle unknown values)
