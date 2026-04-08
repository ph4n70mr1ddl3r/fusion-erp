# 05 - Authentication & Security Specification

## 1. Authentication

### 1.1 Authentication Methods
- **Primary:** Username + password with JWT token-based authentication
- **Service-to-Service:** API keys with HMAC signature
- No session-based authentication — stateless JWT only

### 1.2 Login Flow
```
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "jdoe",
  "password": "SecureP@ssw0rd!2024",
  "tenant_code": "ACME"
}
```

**Success Response (200):**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJSUzI1NiIs...",
    "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": {
      "id": "0192a3b4-...",
      "username": "jdoe",
      "display_name": "John Doe",
      "email": "jdoe@acme.com",
      "tenant_id": "0192a3b4-...",
      "roles": ["FinanceManager", "Accountant"],
      "permissions": ["gl.journals.create", "gl.journals.post", ...]
    }
  }
}
```

**Failure Response (401):**
```json
{
  "error": {
    "code": "AUTH_LOGIN_001",
    "message": "Invalid credentials",
    "request_id": "req-...",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### 1.3 Token Refresh Flow
```
POST /api/v1/auth/refresh
Content-Type: application/json

{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g..."
}
```

Returns new access_token and new refresh_token (rotation). Old refresh_token is invalidated.

### 1.4 Logout Flow
```
POST /api/v1/auth/logout
Authorization: Bearer {access_token}

{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g..."
}
```

Invalidates the refresh_token family. Access token expires naturally (short-lived).

---

## 2. JWT Token Structure

### 2.1 Access Token
```json
{
  "sub": "0192a3b4-c5d6-7e8f-9a0b-1c2d3e4f5a6b",
  "tid": "0192a3b4-1111-2222-3333-444455556666",
  "username": "jdoe",
  "roles": ["FinanceManager", "Accountant"],
  "permissions": [
    "gl.accounts.read",
    "gl.journals.create",
    "gl.journals.post",
    "ap.invoices.create",
    "ap.invoices.approve",
    "ap.payments.create"
  ],
  "iss": "fusion-erp",
  "aud": "fusion-erp-api",
  "iat": 1705312200,
  "exp": 1705313100
}
```

### 2.2 Token Configuration
| Property | Access Token | Refresh Token |
|----------|-------------|---------------|
| Algorithm | RS256 (RSA) | RS256 (RSA) |
| Expiry | 15 minutes | 24 hours |
| Storage | Client memory | HTTP-only cookie |
| Rotation | No | Yes (each use) |
| Revocation | Natural expiry | DB-based |

### 2.3 Key Management
- RSA key pair generated at setup: `config/jwt-private.pem` and `config/jwt-public.pem`
- Minimum key size: 2048 bits
- Private key: used by auth-service only
- Public key: distributed to all services for token verification

---

## 3. User Management

### 3.1 Users Table
```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    username TEXT NOT NULL,
    email TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    display_name TEXT NOT NULL,
    first_name TEXT,
    last_name TEXT,
    phone TEXT,
    avatar_url TEXT,
    status TEXT NOT NULL DEFAULT 'ACTIVE'
        CHECK(status IN ('ACTIVE','INACTIVE','LOCKED','PENDING')),
    failed_login_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until TEXT,
    last_login_at TEXT,
    password_changed_at TEXT,
    must_change_password INTEGER NOT NULL DEFAULT 0,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, username),
    UNIQUE(tenant_id, email)
);
```

### 3.2 Password Requirements
- Minimum length: 12 characters
- Must contain at least: 1 uppercase, 1 lowercase, 1 digit, 1 special character
- MUST NOT be in common password list (top 10,000)
- Hashed with bcrypt cost factor 12

### 3.3 Account Lockout
- After 5 consecutive failed login attempts, account is locked for 30 minutes
- `failed_login_attempts` counter incremented on each failure
- Counter reset to 0 on successful login
- Lockout enforced by checking `locked_until > datetime('now')`

### 3.4 User CRUD Endpoints
```
POST   /api/v1/auth/users              — Create user (requires AUTH_USER_CREATE permission)
GET    /api/v1/auth/users               — List users (with pagination, filtering)
GET    /api/v1/auth/users/{id}          — Get user details
PUT    /api/v1/auth/users/{id}          — Update user
PATCH  /api/v1/auth/users/{id}/password — Change password
PATCH  /api/v1/auth/users/{id}/status   — Activate/deactivate user
DELETE /api/v1/auth/users/{id}          — Soft-delete user
```

---

## 4. Role-Based Access Control (RBAC)

### 4.1 Roles Table
```sql
CREATE TABLE roles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    role_name TEXT NOT NULL,
    description TEXT,
    is_system INTEGER NOT NULL DEFAULT 0,  -- System roles can't be modified

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, role_name)
);
```

### 4.2 Permissions Table
```sql
CREATE TABLE permissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    resource TEXT NOT NULL,       -- e.g., "gl.journals", "ap.invoices"
    action TEXT NOT NULL,         -- e.g., "create", "read", "update", "delete", "approve", "post"
    description TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    UNIQUE(tenant_id, resource, action)
);
```

### 4.3 Role-Permission Mapping
```sql
CREATE TABLE role_permissions (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    role_id TEXT NOT NULL,
    permission_id TEXT NOT NULL,
    granted_by TEXT NOT NULL,
    granted_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, role_id, permission_id)
);
```

### 4.4 User-Role Mapping
```sql
CREATE TABLE user_roles (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    role_id TEXT NOT NULL,
    granted_by TEXT NOT NULL,
    granted_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    UNIQUE(tenant_id, user_id, role_id)
);
```

### 4.5 System Roles (Pre-seeded per Tenant)

| Role | Description | Permissions Scope |
|------|-------------|-------------------|
| SuperAdmin | Full system access | All resources, all actions |
| TenantAdmin | Tenant administration | User management, all tenant resources |
| FinanceManager | Financial operations | GL, AP, AR, FA, CM full access |
| Accountant | Accounting operations | GL, AP, AR create/read/update |
| APManager | Payable operations | AP full access, GL read |
| ARManager | Receivable operations | AR full access, GL read |
| ProcurementManager | Procurement operations | Procurement full, INV read |
| WarehouseManager | Warehouse operations | INV full access |
| SalesManager | Sales operations | OM full, AR read |
| ProjectManager | Project operations | PM full access |
| ReadOnly | Read-only access | All resources, read action only |
| Approver | Approval capability | Approval actions only |

### 4.6 Permission Naming Convention
Format: `{service}.{resource}.{action}`

**Actions:** `create`, `read`, `update`, `delete`, `approve`, `post`, `close`, `reverse`, `export`, `admin`

**Examples:**
```
gl.accounts.create       gl.accounts.read        gl.accounts.update
gl.accounts.delete       gl.journals.create      gl.journals.read
gl.journals.post         gl.journals.approve     gl.journals.reverse
gl.periods.close         gl.periods.reopen       gl.budgets.create
ap.invoices.create       ap.invoices.approve     ap.payments.create
ap.suppliers.create      ap.reports.view
ar.invoices.create       ar.receipts.create      ar.customers.create
inv.items.create         inv.movements.create    inv.counts.create
om.orders.create         om.orders.fulfill       om.returns.create
```

### 4.7 Permission Check Middleware
```rust
use axum::{extract::State, http::StatusCode, middleware::Next, response::Response};

pub async fn require_permission(
    required: &'static str,
) -> impl Fn(axum::extract::Request, Next) -> Fut<Output = Result<Response, StatusCode>> + Clone {
    move |req: axum::extract::Request, next: Next| {
        let permissions = req.extensions()
            .get::<UserPermissions>()
            .expect("Auth middleware must run first");

        if permissions.has(required) || permissions.has("admin") {
            Ok(next.run(req).await)
        } else {
            Err(StatusCode::FORBIDDEN)
        }
    }
}
```

---

## 5. Refresh Token Management

### 5.1 Refresh Tokens Table
```sql
CREATE TABLE refresh_tokens (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    token_hash TEXT NOT NULL,          -- SHA-256 hash of the token
    token_family TEXT NOT NULL,        -- Family ID for rotation tracking
    expires_at TEXT NOT NULL,
    is_revoked INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_refresh_tokens_tenant_user ON refresh_tokens(tenant_id, user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_refresh_tokens_family ON refresh_tokens(token_family);
```

### 5.2 Token Rotation Rules
1. On login: create new token family, issue refresh token
2. On refresh: invalidate old token, issue new token in same family
3. If a revoked token is used: invalidate entire family (token theft detected)
4. Refresh tokens expire after 24 hours
5. Maximum 5 active refresh tokens per user per family

---

## 6. API Key Management (Service-to-Service)

### 6.1 API Keys Table
```sql
CREATE TABLE api_keys (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    key_hash TEXT NOT NULL,            -- SHA-256 hash
    key_prefix TEXT NOT NULL,          -- First 8 chars for identification
    name TEXT NOT NULL,
    service_name TEXT NOT NULL,        -- Which service uses this key
    permissions TEXT NOT NULL,         -- JSON array of permissions
    expires_at TEXT,
    last_used_at TEXT,

    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    created_by TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,

    UNIQUE(tenant_id, key_prefix)
);
```

### 6.2 API Key Format
`fk_{service}_{random_32_chars}` — example: `fk_gl_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`

### 6.3 API Key Validation
- Key sent via `Authorization: Bearer fk_...` header
- Server hashes key with SHA-256 and looks up in `api_keys` table
- Constant-time comparison to prevent timing attacks
- Service permissions applied same as user permissions

---

## 7. Audit Trail

### 7.1 Audit Log Table
```sql
CREATE TABLE audit_log (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    action TEXT NOT NULL,              -- CREATE, UPDATE, DELETE, LOGIN, LOGOUT, etc.
    resource_type TEXT NOT NULL,       -- gl_account, journal_entry, invoice, etc.
    resource_id TEXT NOT NULL,
    old_values TEXT,                   -- JSON of previous state (null for CREATE)
    new_values TEXT,                   -- JSON of new state (null for DELETE)
    changes TEXT,                      -- JSON array of changed field names
    ip_address TEXT,
    user_agent TEXT,
    request_id TEXT NOT NULL,
    timestamp TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_audit_tenant_resource ON audit_log(tenant_id, resource_type, resource_id);
CREATE INDEX idx_audit_tenant_user ON audit_log(tenant_id, user_id);
CREATE INDEX idx_audit_tenant_timestamp ON audit_log(tenant_id, timestamp);
```

### 7.2 Audit Trail Rules
- All CREATE, UPDATE, DELETE operations MUST log to audit trail
- Login and logout events MUST be logged
- Password changes MUST be logged
- Role/permission changes MUST be logged
- Audit records MUST NOT be modified or deleted (append-only)
- Audit log accessible via admin API with filtering

### 7.3 Audit Log Query Endpoint
```
GET /api/v1/auth/audit-log?filter[resource_type]=journal_entry&filter[user_id]={id}&filter[date_from]=2024-01-01&sort=-timestamp&limit=50
```

---

## 8. Security Middleware Stack

### 8.1 Middleware Order (Outer → Inner)
```rust
use axum::Router;
use tower::ServiceBuilder;
use tower_http::cors::CorsLayer;
use tower_http::trace::TraceLayer;
use tower_http::request_id::{RequestIdLayer, PropagateRequestIdLayer};

let app = Router::new()
    // 1. Rate limiting
    .layer(rate_limit_layer())
    // 2. CORS
    .layer(CorsLayer::permissive())
    // 3. Request ID generation
    .layer(RequestIdLayer::new())
    // 4. Tracing/logging
    .layer(TraceLayer::new_for_http())
    // 5. Auth (JWT validation)
    .layer(middleware::from_fn(auth_middleware))
    // 6. Tenant extraction and validation
    .layer(middleware::from_fn(tenant_middleware))
    // 7. Audit logging (for mutation requests)
    .layer(middleware::from_fn(audit_middleware))
    // 8. Route handlers
    .merge(routes());
```

### 8.2 Security Headers Middleware
```rust
use axum::{http::HeaderValue, response::Response};

fn security_headers(response: &mut Response) {
    let headers = response.headers_mut();
    headers.insert("X-Content-Type-Options", HeaderValue::from_static("nosniff"));
    headers.insert("X-Frame-Options", HeaderValue::from_static("DENY"));
    headers.insert("X-XSS-Protection", HeaderValue::from_static("1; mode=block"));
    headers.insert("Referrer-Policy", HeaderValue::from_static("strict-origin-when-cross-origin"));
    headers.insert(
        "Content-Security-Policy",
        HeaderValue::from_static("default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"),
    );
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=31536000; includeSubDomains"),
    );
}
```

---

## 9. Input Validation & Injection Prevention

### 9.1 SQL Injection Prevention
- ALL queries MUST use parameterized statements via `rusqlite::params![]`
- MUST NOT use string formatting, concatenation, or `format!()` to build SQL
- Dynamic WHERE clauses MUST use parameterized conditions:

```rust
// CORRECT
conn.query("SELECT * FROM accounts WHERE tenant_id = ?1 AND status = ?2",
    params![tenant_id, status])?;

// WRONG — NEVER DO THIS
conn.query(&format!("SELECT * FROM accounts WHERE tenant_id = '{}' AND status = '{}'",
    tenant_id, status))?;
```

### 9.2 Input Validation Rules
- All string inputs validated for max length
- All numeric inputs validated for range
- All enum inputs validated against allowed values
- All date inputs validated for ISO 8601 format
- All email inputs validated for format
- Reject obviously malicious patterns (script tags, SQL keywords in string fields)

### 9.3 Output Encoding
- All API responses JSON-serialized by serde (automatic escaping)
- HTML output (frontend) MUST be auto-escaped by Leptos template engine
- No raw HTML injection

---

## 10. Encryption

### 10.1 At Rest
- SQLite database files encrypted using SQLCipher extension with rusqlite
- Encryption key stored in environment variable (`DATABASE_ENCRYPTION_KEY`)
- Key rotation supported via re-encryption migration

### 10.2 In Transit
- TLS 1.3 enforced for all HTTP connections (terminated at gateway)
- gRPC connections use TLS in production
- Development: plain HTTP allowed on localhost only

### 10.3 Sensitive Data Fields
- Bank account numbers: AES-256-GCM encrypted at application layer
- Tax identification numbers: AES-256-GCM encrypted
- API keys: SHA-256 hashed (never stored plaintext)
- Passwords: bcrypt hashed (never stored plaintext)

### 10.4 Encryption Key Management
```sql
-- Encryption keys table (in auth.db)
CREATE TABLE encryption_keys (
    id TEXT PRIMARY KEY NOT NULL,
    tenant_id TEXT NOT NULL,
    key_type TEXT NOT NULL,            -- DATABASE, FIELD_LEVEL, API
    encrypted_key TEXT NOT NULL,       -- AES key encrypted with master key
    version INTEGER NOT NULL DEFAULT 1,
    active_from TEXT NOT NULL,
    active_to TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## 11. CORS Configuration

```rust
use tower_http::cors::{CorsLayer, AllowOrigin, AllowMethods, AllowHeaders};

let cors = CorsLayer::new()
    .allow_origin(AllowOrigin::exact("https://app.fusion-erp.local".parse()?))
    .allow_methods([
        Method::GET,
        Method::POST,
        Method::PUT,
        Method::PATCH,
        Method::DELETE,
    ])
    .allow_headers([
        HEADER_AUTHORIZATION,
        HEADER_CONTENT_TYPE,
        HeaderName::from_static("x-tenant-id"),
        HeaderName::from_static("x-request-id"),
        HeaderName::from_static("x-idempotency-key"),
    ])
    .allow_credentials(true)
    .max_age(Duration::from_secs(3600));
```

---

## 12. Rate Limiting

### 12.1 Rate Limits

| Scope | Limit | Window |
|-------|-------|--------|
| Per tenant (default) | 100 requests | 1 minute |
| Per tenant (reports) | 20 requests | 1 minute |
| Per user (login) | 5 attempts | 1 minute |
| Per user (password reset) | 3 attempts | 1 hour |
| Per service (internal) | Unlimited | — |

### 12.2 Rate Limit Headers
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1705312200
```

### 12.3 Rate Limit Exceeded Response (429)
```json
{
  "error": {
    "code": "GEN_RATE_LIMIT_001",
    "message": "Rate limit exceeded. Try again in 45 seconds.",
    "details": [
      { "field": "retry_after", "issue": "45 seconds" }
    ]
  }
}
```
