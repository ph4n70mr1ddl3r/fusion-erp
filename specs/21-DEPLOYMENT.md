# 21 - Deployment Specification

## 1. Overview

Deployment uses Docker containers orchestrated via Docker Compose (development) with production-ready configurations for each microservice.

---

## 2. Docker Configuration

### 2.1 Base Service Dockerfile
```dockerfile
# deploy/Dockerfile.service
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY crates/ ./crates/
COPY services/ ./services/
COPY migrations/ ./migrations/
RUN cargo build --release -p ${SERVICE_NAME}

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/${SERVICE_NAME} /usr/local/bin/service
COPY config/ ./config/
RUN mkdir -p /app/data
EXPOSE ${HTTP_PORT} ${GRPC_PORT}
CMD ["service"]
```

### 2.2 Gateway Dockerfile
```dockerfile
# deploy/Dockerfile.gateway
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY crates/ ./crates/
COPY services/gateway-service/ ./services/gateway-service/
RUN cargo build --release -p gateway-service

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/gateway-service /usr/local/bin/gateway-service
COPY config/ ./config/
EXPOSE 8000
CMD ["gateway-service"]
```

### 2.3 Frontend Dockerfile
```dockerfile
# deploy/Dockerfile.frontend
FROM rust:1.82-slim AS builder
WORKDIR /app
RUN curl -fsSL https://github.com/trunk-rs/trunk/releases/download/v0.21.12/trunk-x86_64-unknown-linux-musl.tar.gz | tar xzf - -C /usr/local/bin
COPY frontend/ ./frontend/
WORKDIR /app/frontend
RUN trunk build --release

FROM nginx:alpine
COPY --from=builder /app/frontend/dist /usr/share/nginx/html
COPY deploy/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### 2.4 Nginx Configuration
```nginx
# deploy/nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://gateway:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 3. Docker Compose

### 3.1 Development Compose
```yaml
# docker-compose.yml
version: "3.8"

services:
  # Frontend
  frontend:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.frontend
    ports:
      - "80:80"
    depends_on:
      - gateway

  # API Gateway
  gateway:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.service
      args:
        SERVICE_NAME: gateway-service
        HTTP_PORT: 8000
    ports:
      - "8000:8000"
    volumes:
      - ../config:/app/config
    depends_on:
      - auth

  # Auth Service
  auth:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.service
      args:
        SERVICE_NAME: auth-service
        HTTP_PORT: 8001
        GRPC_PORT: 9001
    ports:
      - "8001:8001"
      - "9001:9001"
    volumes:
      - auth-data:/app/data
      - ../config:/app/config

  # GL Service
  gl:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.service
      args:
        SERVICE_NAME: gl-service
        HTTP_PORT: 8010
        GRPC_PORT: 9010
    ports:
      - "8010:8010"
      - "9010:9010"
    volumes:
      - gl-data:/app/data
      - ../config:/app/config

  # AP Service
  ap:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.service
      args:
        SERVICE_NAME: ap-service
        HTTP_PORT: 8011
        GRPC_PORT: 9011
    ports:
      - "8011:8011"
      - "9011:9011"
    volumes:
      - ap-data:/app/data
      - ../config:/app/config

  # AR, FA, CM, Proc, Inv, OM, Mfg, PM, Workflow, Report services...
  # (same pattern as above with respective ports)

volumes:
  auth-data:
  gl-data:
  ap-data:
  ar-data:
  fa-data:
  cm-data:
  proc-data:
  inv-data:
  om-data:
  mfg-data:
  pm-data:
  workflow-data:
  report-data:
```

---

## 4. Environment Configuration

### 4.1 Environment Variables (per service)
```bash
# Service identity
SERVICE_NAME=gl-service
HTTP_PORT=8010
GRPC_PORT=9010
LOG_LEVEL=info

# Database
DATABASE_PATH=/app/data/gl.db
DATABASE_POOL_MAX=8
DATABASE_ENCRYPTION_KEY=changeme-in-production

# Auth
JWT_PUBLIC_KEY_PATH=/app/config/jwt-public.pem
JWT_ISSUER=fusion-erp
JWT_AUDIENCE=fusion-erp-api

# Service URLs
AUTH_SERVICE_URL=http://auth:9001
GL_SERVICE_URL=http://gl:9010
AP_SERVICE_URL=http://ap:9011
# ... etc

# TLS (production)
TLS_ENABLED=false
TLS_CERT_PATH=/app/config/tls/cert.pem
TLS_KEY_PATH=/app/config/tls/key.pem
```

---

## 5. Health Monitoring

### 5.1 Health Check Configuration
```yaml
# In docker-compose.yml, per service
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:8010/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 15s
```

### 5.2 Readiness Check
```yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:8010/ready"]
  interval: 30s
  timeout: 10s
  retries: 3
```

---

## 6. Startup Sequence

Services MUST start in dependency order:
1. auth-service (no dependencies)
2. gl-service (no service dependencies)
3. fa-service, cm-service (depend on GL)
4. inv-service (depends on GL)
5. ap-service, ar-service (depend on GL, Workflow)
6. proc-service (depends on INV, Workflow)
7. om-service (depends on INV, AR, Workflow)
8. mfg-service (depends on INV, GL)
9. pm-service (depends on GL, AR, Workflow)
10. workflow-service (no service dependencies)
11. report-service (depends on all)
12. gateway-service (depends on auth)

### 6.1 Startup Checklist (per service)
1. Load configuration from environment + config files
2. Initialize database connection pool
3. Run database migrations
4. Initialize gRPC server on GRPC_PORT
5. Initialize HTTP server on HTTP_PORT
6. Register health check endpoints
7. Start event subscriber tasks
8. Signal readiness

---

## 7. Resource Requirements

### 7.1 Development
| Service | CPU | Memory | Disk |
|---------|-----|--------|------|
| Frontend (nginx) | 0.1 | 64MB | 50MB |
| Gateway | 0.25 | 128MB | 10MB |
| Each backend service | 0.25 | 128MB | 50MB |
| **Total (14 services)** | **3.5** | **1.8GB** | **750MB** |

### 7.2 Production (per service)
- CPU: 0.5-2 cores (adjust based on load)
- Memory: 256MB-1GB (adjust based on dataset)
- Disk: 1GB+ per service (SQLite database growth)
