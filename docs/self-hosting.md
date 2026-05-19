# Self-Hosting Guide

ZoAuth is designed to be easy to self-host. This guide covers Docker Compose (recommended for most deployments), hardening for production, TLS setup, and running as a systemd service.

**🚀 New to ZoAuth?** Start with our quick setup guides:
- **[Quick Start (30 seconds)](https://github.com/zoauth/go-auth-engine/blob/main/QUICK_START.md)** - Docker Compose fastest path
- **[Platform Setup](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md)** - Windows, macOS, Linux specific instructions
- **[Documentation Hub](https://github.com/zoauth/go-auth-engine/blob/main/docs/README.md)** - Complete documentation index

---

## Prerequisites

- Docker + Compose v2 (`docker compose version`)
- PostgreSQL 15+ (or use the bundled one in the compose file)
- A domain name with TLS (for production)

**Platform-specific prerequisites:**
- **Windows:** Docker Desktop, PowerShell 5.1+
- **macOS:** Docker Desktop, Homebrew (optional for native PostgreSQL)
- **Linux:** Docker Engine + Docker Compose plugin, systemd (optional)

See [Platform Setup Guide](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md) for detailed installation instructions.

---

## Quick Start (30 Seconds)

**Fastest path - Use our optimized quick start configuration:**

```bash
git clone https://github.com/zoauth/go-auth-engine
cd go-auth-engine
docker compose -f docker-compose.quick.yml up
```

This starts:
- ✅ PostgreSQL 15 database
- ✅ ZoAuth server on port 9000
- ✅ Automatic schema migrations
- ✅ Default configuration ready for testing

**Test the deployment:**
```bash
# Health check
curl http://localhost:9000/health/ready

# OpenID Discovery
curl http://localhost:9000/.well-known/openid-configuration

# JWKS endpoint
curl http://localhost:9000/oauth2/jwks
```

**Next steps:**
1. Visit http://localhost:9000/diagnostics for interactive dashboard
2. See [Integration Examples](https://github.com/zoauth/go-auth-engine/tree/main/examples) for React SPA and Node.js API
3. Read [Configuration](#configuration) to customize your deployment

---

## Alternative Setup Methods

### Method 1: Production Standalone Setup

For production deployments with custom configuration:

### Method 2: Binary + Local PostgreSQL

**Best for:** Developers who prefer native binaries without Docker

**Time:** 5 minutes

**See [Platform Setup Guide](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md) for complete instructions per OS.**

Quick version:

**1. Install PostgreSQL:**
```bash
# Windows (via Chocolatey)
choco install postgresql

# macOS (via Homebrew)
brew install postgresql@15
brew services start postgresql@15

# Linux (Ubuntu/Debian)
sudo apt install postgresql-15
sudo systemctl start postgresql
```

**2. Create database:**
```bash
createdb zoauth
psql zoauth -c "CREATE USER zoauth WITH PASSWORD 'localdev123';"
psql zoauth -c "GRANT ALL PRIVILEGES ON DATABASE zoauth TO zoauth;"
```

**3. Download and run ZoAuth:**
```bash
# Download binary for your platform
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/zoauth_linux_x86_64.tar.gz

# Extract
tar -xzf zoauth_linux_x86_64.tar.gz

# Set environment variables
export DATABASE_URL="postgres://zoauth:localdev123@localhost:5432/zoauth?sslmode=disable"
export JWT_ISSUER="http://localhost:9000"
export JWK_ENCRYPTION_PASSWORD="dev-password-min-32-chars-long-12345"
export JWK_ENCRYPTION_SALT="deadbeefcafebabe"

# Run
./zoauth
```

**Platform-specific binary downloads:**
- Windows: `zoauth_windows_x86_64.zip`
- macOS Intel: `zoauth_darwin_x86_64.tar.gz`
- macOS Apple Silicon: `zoauth_darwin_arm64.tar.gz`
- Linux AMD64: `zoauth_linux_x86_64.tar.gz`
- Linux ARM64: `zoauth_linux_arm64.tar.gz`

---

## Configuration

### Required Environment Variables

```env
# Core Configuration
JWT_ISSUER=https://auth.yourdomain.com      # Your OAuth2 issuer URL
DATABASE_URL=postgres://user:pass@host:5432/zoauth?sslmode=require

# Encryption Keys (REQUIRED - generate strong values)
JWK_ENCRYPTION_PASSWORD=<min 32 chars>      # Generate: openssl rand -base64 32
JWK_ENCRYPTION_SALT=<exactly 16 hex chars>  # Generate: openssl rand -hex 8

# Server
SERVER_PORT=9000                            # Default: 9000
LOG_LEVEL=info                              # debug, info, warn, error
LOG_FORMAT=json                             # json or pretty

# Spring Integration (optional)
SPRING_ENABLED=false                        # Set true if using Java Auth backend
SPRING_BASE_URL=http://localhost:8080       # Java Auth service URL
```

### Optional Environment Variables

```env
# Token Configuration
JWT_ACCESS_TOKEN_TTL=1h                     # Default: 1 hour
JWT_REFRESH_TOKEN_TTL=720h                  # Default: 30 days
JWT_ID_TOKEN_TTL=1h                         # Default: 1 hour

# Feature Flags (all default to true)
AUTHCODE_NATIVE=true                        # Authorization code flow
LOGOUT_NATIVE=true                          # Logout endpoint
CONSENT_NATIVE=true                         # Consent screen
DEVICE_NATIVE=true                          # Device flow (RFC 8628)
PAR_NATIVE=true                             # Pushed Authorization Requests
DCR_NATIVE=true                             # Dynamic Client Registration

# Rate Limiting (requires Redis)
REDIS_URL=redis://localhost:6379
RATE_LIMIT_ENABLED=true
RATE_LIMIT_REQUESTS_PER_MINUTE=100

# Observability
METRICS_PORT=9090                           # Prometheus metrics
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318  # OpenTelemetry

# Database Connection Pool
DB_MAX_CONNS=25                             # Default: 25
DB_MIN_CONNS=5                              # Default: 5
DB_MAX_CONN_LIFETIME=1h                     # Default: 1 hour
DB_MAX_CONN_IDLE_TIME=30m                   # Default: 30 minutes
```

**Complete configuration reference:** See [.env.example](https://github.com/zoauth/go-auth-engine/blob/main/.env.example)

---

## Method 3: Custom Docker Compose

For production deployments with custom configuration:

```bash
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/docker-compose.standalone.yml
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/.env.standalone.example
cp .env.standalone.example .env
# Edit .env with your configuration
docker compose -f docker-compose.standalone.yml --env-file .env up -d
```

**Example docker-compose.yml for production:**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: zoauth
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: zoauth
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zoauth"]
      interval: 10s
      timeout: 5s
      retries: 5

  zoauth:
    image: ghcr.io/zoauth/go-auth-engine:latest
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://zoauth:${POSTGRES_PASSWORD}@postgres:5432/zoauth?sslmode=disable
      JWT_ISSUER: ${JWT_ISSUER}
      JWK_ENCRYPTION_PASSWORD: ${JWK_ENCRYPTION_PASSWORD}
      JWK_ENCRYPTION_SALT: ${JWK_ENCRYPTION_SALT}
      SERVER_PORT: 9000
      LOG_LEVEL: info
      LOG_FORMAT: json
    ports:
      - "9000:9000"
    restart: unless-stopped

volumes:
  postgres_data:
```

**Corresponding .env file:**
```env
POSTGRES_PASSWORD=your-strong-password-here
JWT_ISSUER=https://auth.yourdomain.com
JWK_ENCRYPTION_PASSWORD=<generate with: openssl rand -base64 32>
JWK_ENCRYPTION_SALT=<generate with: openssl rand -hex 8>
```

---

## Platform-Specific Troubleshooting

### Windows

**Issue: "Docker daemon not running"**
```powershell
# Ensure Docker Desktop is running
Get-Process | Where-Object {$_.Name -eq "Docker Desktop"}

# Start Docker Desktop if not running
Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"
```

**Issue: "Port 9000 already in use"**
```powershell
# Find process using port 9000
netstat -ano | findstr :9000

# Kill process (replace PID)
taskkill /PID <PID> /F
```

**Issue: Antivirus blocking zoauth.exe**
- Add `zoauth.exe` to Windows Defender exclusions
- See [Platform Setup - Windows Troubleshooting](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md#windows-troubleshooting)

### macOS

**Issue: "zoauth cannot be opened because it is from an unidentified developer"**
```bash
# Remove quarantine attribute
xattr -d com.apple.quarantine ./zoauth

# Or allow in System Preferences
# System Preferences → Security & Privacy → General → "Allow Anyway"
```

**Issue: Homebrew PostgreSQL not starting**
```bash
# Check status
brew services list

# Restart PostgreSQL
brew services restart postgresql@15

# Check logs
tail -f /opt/homebrew/var/log/postgres.log  # Apple Silicon
tail -f /usr/local/var/log/postgres.log     # Intel
```

### Linux

**Issue: Docker permission denied**
```bash
# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Or use rootless Docker
dockerd-rootless-setuptool.sh install
```

**Issue: PostgreSQL authentication failed**
```bash
# Edit pg_hba.conf to allow local connections
sudo nano /etc/postgresql/15/main/pg_hba.conf

# Add this line:
# local   all   zoauth   md5

# Restart PostgreSQL
sudo systemctl restart postgresql
```

**Complete troubleshooting guide:** [Platform Setup - Troubleshooting](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md#troubleshooting)

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

---

## Additional Resources

### 📚 Documentation

| Resource | Description | Target Audience |
|----------|-------------|-----------------|
| **[Quick Start Guide](https://github.com/zoauth/go-auth-engine/blob/main/QUICK_START.md)** | 30-second Docker Compose setup | Everyone |
| **[Platform Setup Guide](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md)** | Windows, macOS, Linux specific instructions | Platform-specific users |
| **[Documentation Hub](https://github.com/zoauth/go-auth-engine/blob/main/docs/README.md)** | Complete documentation index | Developers & Architects |
| **[Enterprise Architecture](https://github.com/zoauth/go-auth-engine/blob/main/docs/ENTERPRISE_ARCHITECTURE_2026.md)** | Production deployment patterns | DevOps & SRE |
| **[API Reference](https://github.com/zoauth/go-auth-engine/blob/main/api/openapi.yaml)** | OpenAPI 3.0 specification | API consumers |

### 💻 Integration Examples

| Example | Technology | Link |
|---------|-----------|------|
| **React SPA** | React 18 + TypeScript + PKCE | [examples/react-spa](https://github.com/zoauth/go-auth-engine/tree/main/examples/react-spa) |
| **Node.js API** | Express + JWT validation | [examples/nodejs-api](https://github.com/zoauth/go-auth-engine/tree/main/examples/nodejs-api) |
| **Python API** | FastAPI + JWT validation | [examples/python-api](https://github.com/zoauth/go-auth-engine/tree/main/examples/python-api) |
| **Mobile App** | React Native + PKCE | [examples/mobile-app](https://github.com/zoauth/go-auth-engine/tree/main/examples/mobile-app) |

### 🚀 Deployment Options

| Platform | Guide | Best For |
|----------|-------|----------|
| **Docker Compose** | [docker-compose.quick.yml](https://github.com/zoauth/go-auth-engine/blob/main/docker-compose.quick.yml) | Local dev, small deployments |
| **Google Cloud Run** | [Enterprise Architecture](https://github.com/zoauth/go-auth-engine/blob/main/docs/ENTERPRISE_ARCHITECTURE_2026.md) | Serverless, auto-scaling |
| **Kubernetes** | [k8s/](https://github.com/zoauth/go-auth-engine/tree/main/k8s) | Enterprise, high availability |
| **AWS ECS** | [aws/](https://github.com/zoauth/go-auth-engine/tree/main/aws) | AWS-native deployments |
| **Systemd** | [See above](#running-as-a-systemd-service-binary) | VPS, bare metal |

### 🛠️ Development Tools

- **[test-docker-setup.ps1](https://github.com/zoauth/go-auth-engine/blob/main/test-docker-setup.ps1)** - Validate Docker Compose configuration (Windows PowerShell)
- **[.env.example](https://github.com/zoauth/go-auth-engine/blob/main/.env.example)** - Complete environment variable reference
- **[Makefile](https://github.com/zoauth/go-auth-engine/blob/main/Makefile)** - Build and test commands
- **[.editorconfig](https://github.com/zoauth/go-auth-engine/blob/main/.editorconfig)** - Code style configuration

### 🎯 Quick Links

**For New Users:**
1. Start with [Quick Start (30 seconds)](https://github.com/zoauth/go-auth-engine/blob/main/QUICK_START.md)
2. Pick your platform guide from [Platform Setup](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md)
3. Test with [Integration Examples](https://github.com/zoauth/go-auth-engine/tree/main/examples)
4. Deploy with this Self-Hosting Guide

**For Production:**
1. Review [Enterprise Architecture](https://github.com/zoauth/go-auth-engine/blob/main/docs/ENTERPRISE_ARCHITECTURE_2026.md)
2. Follow [Production Hardening](#production-hardening) section above
3. Set up monitoring with [Observability](#observability) section above
4. Configure TLS with reverse proxy ([Nginx](#1-use-a-reverse-proxy-for-tls) or Caddy)

**For Troubleshooting:**
- Windows: [Platform Setup - Windows Troubleshooting](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md#windows-troubleshooting)
- macOS: [Platform Setup - macOS Troubleshooting](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md#macos-troubleshooting)
- Linux: [Platform Setup - Linux Troubleshooting](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md#linux-troubleshooting)
- Database: [Platform Setup - Database Issues](https://github.com/zoauth/go-auth-engine/blob/main/PLATFORM_SETUP.md#database-troubleshooting)

### 🆘 Support

- **GitHub Issues:** [Report bugs or request features](https://github.com/zoauth/go-auth-engine/issues)
- **GitHub Discussions:** [Ask questions or share ideas](https://github.com/zoauth/go-auth-engine/discussions)
- **Documentation:** [Full documentation index](https://github.com/zoauth/go-auth-engine/blob/main/docs/README.md)
- **Website:** [https://zoauth.com](https://zoauth.com)

---

## Summary

**ZoAuth Self-Hosting** offers three primary deployment methods:

1. **Docker Compose (Recommended)** - Get started in 30 seconds with `docker-compose.quick.yml`
2. **Binary + PostgreSQL** - Native setup for each platform (Windows/Mac/Linux)
3. **Custom Deployment** - Full control with production hardening

**Key Features:**
- ✅ Multi-tenant OAuth2/OIDC server
- ✅ Automatic schema migrations
- ✅ Health check endpoints
- ✅ Prometheus metrics
- ✅ Platform-specific support (Windows, macOS, Linux)
- ✅ Production-ready with TLS, Redis, managed databases

**Next Steps:**
1. Choose your setup method above
2. Follow platform-specific instructions
3. Test with provided examples
4. Deploy to production with hardening guidelines

For the fastest experience, start with [Quick Start (30 seconds)](https://github.com/zoauth/go-auth-engine/blob/main/QUICK_START.md)!
