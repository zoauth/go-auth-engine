<div align="center">

# ZoAuth

**Enterprise-grade OpenID Connect & OAuth 2.1 server. Self-hosted. Production-ready.**

[![Release](https://img.shields.io/github/v/release/zoauth/go-auth-engine?style=flat-square&color=blue)](https://github.com/zoauth/go-auth-engine/releases/latest)
[![Docker Pulls](https://img.shields.io/badge/docker-ghcr.io%2Fzoauth-blue?style=flat-square&logo=docker)](https://github.com/zoauth/go-auth-engine/pkgs/container/go-auth-engine)
[![OpenID Certified](https://img.shields.io/badge/OpenID-Certified-orange?style=flat-square)](https://openid.net/certification/)
[![License](https://img.shields.io/badge/license-BSL%201.1-lightgrey?style=flat-square)](LICENSE)
[![Community](https://img.shields.io/badge/community-discussions-green?style=flat-square)](https://github.com/zoauth/go-auth-engine/discussions)

[Quick Start](#quick-start) · [Documentation](#documentation) · [Deployment](#deployment-options) · [Pricing](https://zoauth.com/pricing) · [Community](#community)

</div>

---

## What is ZoAuth?

ZoAuth is a **self-hostable OpenID Connect + OAuth 2.1 server** written in Go. It handles authentication and authorization for your applications — SSO, machine-to-machine tokens, PKCE, MFA, Dynamic Client Registration, and more.

- **Deploy anywhere** — Docker Compose, Cloud Run, Kubernetes, bare metal
- **OpenID Certified** — passes the full OpenID Foundation conformance suite
- **Multi-tenant** — manage multiple applications from one instance
- **No vendor lock-in** — you own your data and your infrastructure
- **Production-tested** — runs at [zoauth.com](https://zoauth.com) serving real traffic

---

## Quick Start

### 60-second self-hosted setup (Docker required)

```bash
# 1. Download the compose kit
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/docker-compose.standalone.yml
curl -LO https://github.com/zoauth/go-auth-engine/releases/latest/download/.env.standalone.example

# 2. Configure
cp .env.standalone.example .env
# Edit .env — set JWT_ISSUER, JWK_ENCRYPTION_PASSWORD, JWK_ENCRYPTION_SALT

# 3. Start
docker compose -f docker-compose.standalone.yml --env-file .env up -d

# 4. Verify
curl http://localhost:8080/.well-known/openid-configuration
```

ZoAuth + PostgreSQL start up. Schema is created automatically on first boot.

---

## Features

### OAuth 2.1 / OpenID Connect
| Feature | Community | Enterprise |
|---|:---:|:---:|
| Authorization Code + PKCE | ✅ | ✅ |
| Client Credentials | ✅ | ✅ |
| Device Authorization | ✅ | ✅ |
| Refresh Tokens | ✅ | ✅ |
| Dynamic Client Registration (RFC 7591) | ✅ | ✅ |
| Pushed Authorization Requests (RFC 9126) | ✅ | ✅ |
| Token Introspection & Revocation | ✅ | ✅ |
| OIDC Discovery + JWKS | ✅ | ✅ |
| UserInfo endpoint | ✅ | ✅ |
| Back-channel & Front-channel Logout | ✅ | ✅ |

### Security & Identity
| Feature | Community | Enterprise |
|---|:---:|:---:|
| TOTP / WebAuthn MFA | ✅ | ✅ |
| Multi-tenant isolation | ✅ | ✅ |
| Custom login UI (bring your own) | ✅ | ✅ |
| MAU limit | 10,000 | Unlimited |
| SAML 2.0 | ❌ | ✅ |
| LDAP / Active Directory | ❌ | ✅ |
| SCIM 2.0 provisioning | ❌ | ✅ |
| SSO Bridge (social → enterprise) | ❌ | ✅ |

### Operations
| Feature | Community | Enterprise |
|---|:---:|:---:|
| Prometheus metrics | ✅ | ✅ |
| OpenTelemetry tracing | ✅ | ✅ |
| Structured JSON logs | ✅ | ✅ |
| Health + readiness endpoints | ✅ | ✅ |
| Zero-downtime deploys | ✅ | ✅ |
| SLA + support contract | ❌ | ✅ |

---

## Deployment Options

### Docker Compose (simplest)
```bash
docker compose -f docker-compose.standalone.yml --env-file .env up -d
```
See [docs/self-hosting.md](docs/self-hosting.md) for hardening, TLS, and production tips.

### Google Cloud Run
```bash
gcloud run deploy zoauth \
  --image ghcr.io/zoauth/go-auth-engine:latest \
  --set-env-vars JWT_ISSUER=https://auth.example.com,...
```
Full guide: [docs/cloud-run.md](docs/cloud-run.md)

### Kubernetes / Helm
```bash
helm install zoauth oci://ghcr.io/zoauth/charts/zoauth \
  --set config.jwtIssuer=https://auth.example.com
```
Full guide: [docs/kubernetes.md](docs/kubernetes.md)

### Binary (bare metal / systemd)
Download the binary for your platform from [Releases](https://github.com/zoauth/go-auth-engine/releases/latest), set env vars, run.

---

## Configuration

ZoAuth is configured entirely through environment variables — no config files needed.

```env
# Minimum required
JWT_ISSUER=https://auth.example.com
JWK_ENCRYPTION_PASSWORD=at-least-32-random-characters-here
JWK_ENCRYPTION_SALT=0102030405060708   # 8-byte hex
DATABASE_URL=postgres://user:pass@host:5432/zoauth?sslmode=require
SPRING_ENABLED=false
```

Full reference: [docs/configuration.md](docs/configuration.md)

---

## Documentation

| Guide | Description |
|---|---|
| [Self-hosting](docs/self-hosting.md) | Docker Compose, TLS, reverse proxy, systemd |
| [Configuration](docs/configuration.md) | All environment variables with defaults |
| [Cloud Run](docs/cloud-run.md) | GCP Cloud Run deployment |
| [Kubernetes](docs/kubernetes.md) | Helm chart + kustomize |
| [Upgrade Guide](docs/upgrade.md) | Version migration instructions |

---

## Community

- **Discussions** — [github.com/zoauth/go-auth-engine/discussions](https://github.com/zoauth/go-auth-engine/discussions)
- **Bug reports** — [Open an issue](https://github.com/zoauth/go-auth-engine/issues/new?template=bug_report.yml)
- **Feature requests** — [Start a discussion](https://github.com/zoauth/go-auth-engine/discussions/new?category=ideas)
- **Security** — [SECURITY.md](SECURITY.md) — please do NOT open public issues for vulnerabilities

---

## Enterprise & Cloud

**ZoAuth Cloud** — fully managed, no infra to run: [zoauth.com/pricing](https://zoauth.com/pricing)

**Enterprise license** — unlimited MAU + SAML + LDAP + SCIM + priority support: [zoauth.com/pricing](https://zoauth.com/pricing)

---

## License

ZoAuth Community is released under the [Business Source License 1.1](LICENSE) — free to self-host, source-available. Converts to Apache 2.0 after 4 years.

Enterprise features require a [commercial license](https://zoauth.com/pricing).
