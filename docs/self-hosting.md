# Self-Hosting Guide

ZoAuth is designed to be easy to self-host. This guide covers Docker Compose (recommended for most deployments), hardening for production, TLS setup, and running as a systemd service.

---

## Prerequisites

- Docker + Compose v2 (`docker compose version`)
- PostgreSQL 14+ (or use the bundled one in the compose file)
- A domain name with TLS (for production)

---

## Quick Start (Development)

```bash
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/docker-compose.standalone.yml
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/.env.standalone.example
cp .env.standalone.example .env
# Edit .env — minimum required fields below
docker compose -f docker-compose.standalone.yml --env-file .env up -d
```

**Minimum required env vars:**
```env
JWT_ISSUER=https://auth.yourdomain.com
JWK_ENCRYPTION_PASSWORD=<generate: openssl rand -base64 32>
JWK_ENCRYPTION_SALT=<generate: openssl rand -hex 8>
DATABASE_URL=postgres://zoauth:yourpassword@postgres:5432/zoauth?sslmode=disable
SPRING_ENABLED=false
```

---

## First Boot: Schema + Seed

On first boot, ZoAuth creates all required database tables automatically.

To create a default tenant and admin user, set:
```env
SEED_DEFAULT_TENANT=true
SEED_ADMIN_EMAIL=admin@yourdomain.com
SEED_ADMIN_PASSWORD=change-me-on-first-login
```

> **Important**: After first boot, set `SEED_DEFAULT_TENANT=false` to prevent re-seeding on restart.

---

## Production Hardening

### 1. Use a reverse proxy for TLS

ZoAuth does not terminate TLS itself. Put it behind **Nginx** or **Caddy**:

**Caddy (automatic TLS):**
```
auth.yourdomain.com {
    reverse_proxy localhost:8080
}
```

**Nginx:**
```nginx
server {
    listen 443 ssl;
    server_name auth.yourdomain.com;
    ssl_certificate     /etc/letsencrypt/live/auth.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/auth.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 2. Use an external managed PostgreSQL

Replace the bundled Postgres with a managed service (Cloud SQL, RDS, Supabase):
```env
DATABASE_URL=postgres://user:pass@your-db-host:5432/zoauth?sslmode=require
```
Remove the `postgres:` service from the compose file.

### 3. Secure secrets rotation

Generate strong values:
```bash
# Encryption password (32+ chars)
openssl rand -base64 32

# Salt (exactly 8 bytes hex = 16 hex chars)
openssl rand -hex 8

# Admin password
openssl rand -base64 24
```

### 4. Redis for distributed deployments

If running multiple ZoAuth instances (horizontal scaling), add Redis for shared token state:
```env
REDIS_URL=redis://your-redis-host:6379
```

See [examples/docker-compose/full-stack.yml](../examples/docker-compose/full-stack.yml) for a full example with Redis + Prometheus + Grafana.

---

## Running as a Systemd Service (Binary)

Download the binary from [Releases](https://github.com/zoauth/go-auth-engine/releases/latest):

```bash
# Linux amd64
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/zoauth_linux_x86_64.tar.gz
tar -xzf zoauth_linux_x86_64.tar.gz
sudo mv zoauth /usr/local/bin/
sudo chmod +x /usr/local/bin/zoauth
```

Create `/etc/zoauth/zoauth.env`:
```env
JWT_ISSUER=https://auth.yourdomain.com
JWK_ENCRYPTION_PASSWORD=...
JWK_ENCRYPTION_SALT=...
DATABASE_URL=postgres://...
SPRING_ENABLED=false
LOG_FORMAT=json
```

Create `/etc/systemd/system/zoauth.service`:
```ini
[Unit]
Description=ZoAuth OIDC Server
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=zoauth
EnvironmentFile=/etc/zoauth/zoauth.env
ExecStart=/usr/local/bin/zoauth
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd --system --no-create-home zoauth
sudo systemctl daemon-reload
sudo systemctl enable --now zoauth
sudo journalctl -u zoauth -f
```

---

## Health Checks

ZoAuth exposes two health endpoints:

| Endpoint | Purpose | Returns |
|---|---|---|
| `GET /health/live` | Liveness — is the process alive? | `200 OK` |
| `GET /health/ready` | Readiness — is the DB connected? | `200 OK` or `503` |

Use `/health/ready` for load balancer health checks.  
Use `/health/live` for container restart policies.

---

## Observability

ZoAuth emits Prometheus metrics at `:9090/metrics` (default port, configurable via `METRICS_PORT`).

Key metrics:
```
zoauth_token_issued_total          - tokens issued by grant type
zoauth_token_error_total           - token errors by error code
zoauth_request_duration_seconds    - HTTP request latency
zoauth_mau_current                 - current monthly active users per tenant
zoauth_db_pool_acquired_total      - DB connection pool stats
```

OpenTelemetry tracing is enabled by setting `OTEL_EXPORTER_OTLP_ENDPOINT`.

---

## Upgrading

See [docs/upgrade.md](upgrade.md) for version-specific migration instructions.
