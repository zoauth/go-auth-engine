# Upgrade Guide

Instructions for upgrading ZoAuth between versions.

---

## General Upgrade Process

1. **Read the release notes** for your target version at [Releases](https://github.com/zoauth/go-auth-engine/releases)
2. **Back up your database** before upgrading
3. **Check for breaking changes** in the section below
4. Pull the new image and restart

```bash
# Docker Compose
ZOAUTH_VERSION=0.2.0 docker compose -f docker-compose.standalone.yml --env-file .env up -d

# Cloud Run
gcloud run deploy zoauth --image ghcr.io/zoauth/go-auth-engine:0.2.0 --region us-central1

# Kubernetes
helm upgrade zoauth oci://ghcr.io/zoauth/charts/zoauth --set image.tag=0.2.0
```

Database migrations run automatically on startup — ZoAuth uses an idempotent migration system (no manual `migrate` commands needed).

---

## Version History

### v0.1.0 (Initial Release)

First public release. Standalone Go-native mode.

**Features:**
- Full OIDC / OAuth 2.1 (Authorization Code + PKCE, Client Credentials, Device, Refresh)
- MFA (TOTP + WebAuthn)
- Dynamic Client Registration (RFC 7591)
- Pushed Authorization Requests (RFC 9126)
- Back-channel + Front-channel logout
- Multi-tenant isolation
- Prometheus metrics + OpenTelemetry tracing
- Community license (10,000 MAU)

**Breaking changes:** None (initial release)

**Migration:** None required

---

## Rollback

Every release binary is available on the [Releases](https://github.com/zoauth/go-auth-engine/releases) page.

ZoAuth migrations are **forward-only** — rolling back to a previous version after a migration that added columns is safe (the extra columns are ignored). Rolling back after a migration that removed or renamed columns requires a database restore.

**Safe rollback (no schema changes):**
```bash
# Just pull the previous image
ZOAUTH_VERSION=0.1.0 docker compose -f docker-compose.standalone.yml --env-file .env up -d
```

**If schema changed (check release notes):**
```bash
# Restore from backup first, then downgrade image
pg_restore -d zoauth backup-before-upgrade.dump
ZOAUTH_VERSION=0.1.0 docker compose -f docker-compose.standalone.yml --env-file .env up -d
```
