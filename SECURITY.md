# Security Policy

## Supported Versions

| Version | Security fixes |
|---|---|
| Latest release | ✅ Active |
| Previous minor | ✅ Critical only |
| Older | ❌ Unsupported |

## Reporting a Vulnerability

**Please do NOT open a public GitHub issue for security vulnerabilities.**

Report security issues privately via one of:

1. **GitHub Private Security Advisory** (preferred):
   `https://github.com/zoauth/go-auth-engine/security/advisories/new`

2. **Email**: `security@zoauth.com`
   - Encrypt with our PGP key if sensitive (key available at `https://zoauth.com/security.asc`)

### What to include

- ZoAuth version affected
- Description of the vulnerability and potential impact
- Steps to reproduce (PoC if possible)
- Suggested fix (optional)

### What to expect

| Timeline | Action |
|---|---|
| 24 hours | Acknowledgement |
| 7 days | Initial assessment + severity classification |
| 30 days | Fix + coordinated disclosure (CVE if applicable) |
| 90 days | Public disclosure (sooner if patch is available) |

We follow [responsible disclosure](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html). Reporters who follow this policy will be credited in the release notes (unless they prefer anonymity).

## Security Architecture

ZoAuth's security posture:

- All tokens are signed with rotating Ed25519 / RS256 keys (JWKs)
- JWK private keys are encrypted at rest (AES-256-GCM)
- No secrets stored in the binary — all via environment variables
- Multi-tenant isolation enforced at the database query layer
- PKCE enforced for all public clients (OAuth 2.1)
- Tokens validated on every request (no session state)
- SQL injection prevention via parameterized queries throughout
- Rate limiting on token + authorization endpoints
