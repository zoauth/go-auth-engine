# Configuration Reference

All ZoAuth configuration is done through environment variables. No config files needed.

---

## Required Variables

| Variable | Description | Example |
|---|---|---|
| `JWT_ISSUER` | OIDC issuer URL — must match the public URL of ZoAuth | `https://auth.example.com` |
| `JWK_ENCRYPTION_PASSWORD` | Encrypts JWK private keys at rest. Min 32 chars. **Rotate carefully.** | `openssl rand -base64 32` |
| `JWK_ENCRYPTION_SALT` | 8-byte hex salt for key derivation | `openssl rand -hex 8` |
| `DATABASE_URL` | PostgreSQL connection string | `postgres://u:p@host:5432/zoauth?sslmode=require` |
| `SPRING_ENABLED` | `false` for standalone Go mode | `false` |

---

## Token Settings

| Variable | Default | Description |
|---|---|---|
| `JWT_ACCESS_TOKEN_TTL` | `1h` | Access token lifetime |
| `JWT_REFRESH_TOKEN_TTL` | `720h` | Refresh token lifetime (30 days) |
| `JWT_ID_TOKEN_TTL` | `1h` | ID token lifetime |
| `JWK_ENCRYPTION_MODE` | `GCM` | Encryption mode (`GCM` or `CBC`) |
| `JWK_KEY_ROTATION_INTERVAL` | `24h` | How often to rotate signing keys |

---

## Feature Flags

| Variable | Default | Description |
|---|---|---|
| `AUTHCODE_NATIVE` | `true` | Use Go-native Authorization Code flow |
| `LOGOUT_NATIVE` | `true` | Use Go-native logout (back/front-channel) |
| `CONSENT_NATIVE` | `true` | Use Go-native consent screen |
| `DEVICE_NATIVE` | `true` | Use Go-native Device Authorization flow |
| `PAR_NATIVE` | `true` | Use Go-native Pushed Authorization Requests |
| `DCR_NATIVE` | `true` | Use Go-native Dynamic Client Registration |

---

## Login & Consent URLs

| Variable | Default | Description |
|---|---|---|
| `AUTHCODE_LOGIN_URL` | `http://localhost:8080/login` | Where users are redirected to log in |
| `CONSENT_URL` | *(same host)* | Where users are redirected for consent |

---

## Database

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | — | Full PostgreSQL DSN |
| `DB_MAX_OPEN_CONNS` | `25` | Max open DB connections |
| `DB_MAX_IDLE_CONNS` | `5` | Max idle DB connections |
| `DB_CONN_MAX_LIFETIME` | `5m` | Connection max lifetime |

---

## Redis (optional)

Redis is used for MAU counters, distributed rate limiting, and session sharing across multiple ZoAuth instances. If not set, ZoAuth uses in-process state (single instance only).

| Variable | Default | Description |
|---|---|---|
| `REDIS_URL` | *(empty)* | Redis connection URL. E.g. `redis://host:6379` or `rediss://host:6380` |
| `REDIS_PASSWORD` | *(empty)* | Redis password (alternative to embedding in URL) |
| `REDIS_DB` | `0` | Redis database index |

---

## Server

| Variable | Default | Description |
|---|---|---|
| `PORT` / `SERVER_PORT` | `8080` | HTTP listen port |
| `METRICS_PORT` | `9090` | Prometheus metrics port |
| `READ_TIMEOUT` | `10s` | HTTP read timeout |
| `WRITE_TIMEOUT` | `30s` | HTTP write timeout |
| `IDLE_TIMEOUT` | `120s` | HTTP keep-alive idle timeout |

---

## CORS

| Variable | Default | Description |
|---|---|---|
| `CORS_ALLOWED_ORIGINS` | `*` | Comma-separated allowed origins |
| `CORS_ALLOWED_METHODS` | `GET,POST,OPTIONS` | Allowed HTTP methods |
| `CORS_ALLOWED_HEADERS` | `Authorization,Content-Type` | Allowed request headers |

---

## Observability

| Variable | Default | Description |
|---|---|---|
| `LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `LOG_FORMAT` | `json` | Log format: `json` or `pretty` |
| `METRICS_ENABLED` | `true` | Enable Prometheus metrics at `METRICS_PORT/metrics` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | *(empty)* | OpenTelemetry collector URL (enables tracing) |
| `OTEL_SERVICE_NAME` | `zoauth` | Service name in traces |
| `OTEL_SAMPLER` | `always_on` | Trace sampler: `always_on`, `always_off`, `traceidratio` |
| `OTEL_SAMPLER_ARG` | `1.0` | Sampling ratio when using `traceidratio` |

---

## Seeding (First Boot)

| Variable | Default | Description |
|---|---|---|
| `SEED_DEFAULT_TENANT` | `false` | Create default tenant + admin user on startup |
| `SEED_ADMIN_EMAIL` | `admin@example.com` | Admin user email |
| `SEED_ADMIN_PASSWORD` | `change-me-now` | Admin user password — **change immediately** |

> Set `SEED_DEFAULT_TENANT=false` after first boot.

---

## License

| Variable | Default | Description |
|---|---|---|
| `ZOAUTH_LICENSE_KEY` | *(empty)* | Enterprise license key. Empty = Community tier (10k MAU) |
| `ZOAUTH_TIER` | `community` | Informational tier label shown in logs |

---

## Generating Secrets

```bash
# JWK_ENCRYPTION_PASSWORD (32+ chars, base64)
openssl rand -base64 32

# JWK_ENCRYPTION_SALT (8 bytes, hex)
openssl rand -hex 8

# Strong admin password
openssl rand -base64 24
```
