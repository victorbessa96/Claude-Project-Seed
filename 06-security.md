# Security

Security requirements tiered by exposure level. Not every application needs authentication and HTTPS — but every application needs input validation and secrets management. Choose the tier that matches your deployment and apply all tiers at and below it.

---

## Tier Overview

| Tier | Deployment | Trust model | Threat surface |
|------|-----------|-------------|----------------|
| **Tier 1: Local** | Single machine | User is the only actor | Malicious input from external data sources |
| **Tier 2: LAN** | Trusted network | Trusted users, untrusted network | Other devices on the network; accidental exposure |
| **Tier 3: Internet** | Public-facing | Adversarial traffic | Everything — bots, scanners, targeted attacks |

Each tier includes all requirements from the tiers below it.

---

## Universal Requirements (All Tiers)

These apply to every project, regardless of deployment.

### Parameterized Queries

Never construct SQL from string concatenation. Always use parameterized queries (placeholders like `?` or `$1`).

**Why:** SQL injection is consistently in the OWASP Top 10. A single un-parameterized query can give an attacker full database access. Parameterized queries are equally easy to write and completely prevent SQL injection.

**In practice:**
- Use the query builder or ORM's parameterization
- For dynamic WHERE clauses, build condition lists and parameter arrays separately, then join
- Never pass user input directly into ORDER BY — use a whitelist of allowed column names

### HTML Escaping (XSS Prevention)

Any user-provided or external data rendered in HTML must be escaped.

**Why:** Cross-site scripting (XSS) lets attackers inject code that runs in other users' browsers. Even in single-user apps, content from external sources (RSS feeds, API responses) can contain malicious scripts.

**In practice:**
- Use the framework's built-in escaping (template engines, `textContent` in DOM)
- For toast/notification messages: set text via `textContent`, not `innerHTML`
- If you must use `innerHTML`, sanitize with a whitelist of allowed tags
- Escape data at the point of rendering, not at the point of storage

> **CyberPulse example:** Toast messages use a `textContent` → `innerHTML` pattern: create a temporary element, set `textContent` (escapes HTML), then read `innerHTML` (safe string).

### Secrets Management

Secrets (API keys, passwords, tokens) must never appear in code, version control, logs, or unmasked API responses.

**Why:** Secrets in code get pushed to repositories. Secrets in logs get shipped to log aggregators. Secrets in API responses get cached in browsers. Every exposure is a credential leak.

**In practice:**
- Store secrets in environment variables, encrypted config, or database (not in source files)
- Add secret-containing files to `.gitignore` (`.env`, `credentials.json`, `*.key`)
- Mask secrets in API responses (show only last 4 characters)
- Never log request/response bodies that might contain secrets
- Rotate secrets that may have been exposed

> **CyberPulse example:** API keys are stored in the SQLite settings table. GET `/api/settings` returns masked keys (last 4 chars visible). Keys are never logged.

### Input Validation at System Boundaries

Validate all input that enters the system — user input, API request bodies, query parameters, file uploads, webhook payloads.

**Why:** Input validation prevents a class of bugs (type errors, injection, buffer overflows) at the cheapest possible point. Validating at boundaries means internal code can trust its inputs.

**In practice:**
- Validate type, length, format, and allowed values
- Use allowlists (what IS allowed) over blocklists (what ISN'T allowed)
- For settings/configuration: maintain an `ALLOWED_KEYS` set; reject unknown keys
- Return clear error messages for invalid input (what was wrong, what was expected)

### Timeouts on External Requests

Every HTTP request to an external service must have an explicit timeout.

**Why:** Without timeouts, a single unresponsive external service can hang your entire application. Default timeouts are often too high (or infinite).

**In practice:**
- Set both connection timeout (5-10s) and total request timeout (30-60s)
- For LLM API calls, use longer timeouts (60-120s) since responses are slower
- Handle timeout errors gracefully — log and move on, don't crash

### Dependency Safety

Every dependency is code you didn't write running with your application's permissions.

**Why:** Supply chain attacks (malicious packages, compromised maintainers) are increasing. Outdated dependencies have known vulnerabilities.

**In practice:**
- Pin dependency versions for reproducible builds
- Audit dependencies before adding them (check maintainer reputation, download count, last update)
- Run vulnerability scanners periodically (`npm audit`, `pip-audit`, `cargo audit`)
- Update dependencies deliberately and test after updating

### .gitignore Essentials

At minimum, ignore:
```
# Data
data/
*.db
*.sqlite

# Secrets
.env
*.key
credentials.*

# Build artifacts
__pycache__/
*.pyc
node_modules/
dist/
build/

# OS files
.DS_Store
Thumbs.db
```

---

## Tier 1: Local-Only

For applications that run on a single machine and are not network-accessible.

### Additional Requirements
- **File permissions** — Ensure database and config files aren't world-readable
- **Local data encryption** — Consider encrypting sensitive data at rest if the machine is shared
- **Process isolation** — Don't run the application as root/admin if it doesn't need elevated permissions

### What You Can Skip
- Authentication (the user is already on the machine)
- HTTPS (no network traffic)
- CORS (no cross-origin requests)
- Rate limiting (single user can't DoS themselves meaningfully)

---

## Tier 2: LAN / Internal Network

For applications accessible from a local network (home, office, lab).

### CORS Policy

Configure Cross-Origin Resource Sharing to control which origins can make requests.

**Why:** Other devices on the LAN can reach your app. A malicious page opened in a browser on the same network could make requests to your API.

**In practice:**
- For trusted LANs: `allow_origins=["*"]` is acceptable if the risk is understood
- For mixed-trust networks: restrict to specific origins
- Always set `allow_methods` and `allow_headers` explicitly

### API Key / Secret Masking

Never return full secrets in API responses, even on a trusted network.

**Why:** Browser developer tools, proxy logs, and network captures all expose API responses. Accidental exposure on a LAN is easy — someone checks the network tab while debugging.

**In practice:**
- Mask API keys in GET responses (show last 4 characters)
- On PUT/update: accept the full key but don't echo it back
- Backend reads the key from storage for API calls — the frontend never needs the full key

### Rate Limiting on Expensive Endpoints

Protect endpoints that trigger expensive operations (LLM calls, bulk imports, exports).

**Why:** Even accidental double-clicks can trigger duplicate LLM calls, burning API credits. Rate limiting prevents this without requiring users to think about it.

**In practice:**
- Track the last request timestamp per endpoint
- Return 429 (Too Many Requests) with retry-after time if cooldown hasn't elapsed
- Typical cooldowns: 10-30s for generation endpoints, 30-60s for bulk fetch endpoints

> **CyberPulse example:** `/api/fetch` has a 30-second cooldown; `/api/generate` has a 10-second cooldown. Both return 429 with the remaining wait time if triggered too soon.

### No Sensitive Logging

Don't log request bodies, API keys, or user data in plain text.

**Why:** Log files are often less protected than databases. They get rotated to backups, shipped to log aggregators, or left on disk with broad read permissions.

**In practice:**
- Log request method, path, status, and duration — not bodies
- Redact sensitive fields if you must log structured data
- Suppress noisy third-party loggers that might log request details

---

## Tier 3: Internet-Facing

For applications accessible from the public internet.

### Authentication & Authorization

Verify who the user is (authentication) and what they're allowed to do (authorization).

**Why:** On the internet, anyone can reach your endpoints. Without auth, anyone can read, modify, or delete data.

**Options:**
- **Session-based** — Server stores session; client sends cookie. Simple; stateful.
- **Token-based (JWT)** — Client stores token; server validates signature. Stateless; harder to revoke.
- **OAuth / SSO** — Delegate auth to a provider (Google, GitHub). Less code; external dependency.

**In practice:**
- Protect all state-modifying endpoints (POST, PUT, DELETE)
- Use constant-time comparison for tokens/passwords to prevent timing attacks
- Implement account lockout or rate limiting on login endpoints
- Store passwords with bcrypt/argon2, never plain text or MD5/SHA

### HTTPS / TLS

All traffic must be encrypted in transit.

**Why:** HTTP traffic can be intercepted, modified, and replayed by anyone between the client and server. HTTPS prevents eavesdropping, tampering, and impersonation.

**In practice:**
- Use TLS certificates (Let's Encrypt for free automated certificates)
- Redirect HTTP to HTTPS at the server level
- Set HSTS headers to prevent downgrade attacks (`Strict-Transport-Security: max-age=31536000; includeSubDomains`)
- Don't mix HTTP and HTTPS content (mixed content breaks security guarantees)

**Common misconfiguration gotchas:**
- Expired certificates — automate renewal; monitor expiration dates
- Incomplete certificate chains — test with multiple clients, not just your browser (some browsers fetch missing intermediate certs; others don't)
- Serving HTTPS but allowing HTTP fallback without redirect — users on HTTP get no protection
- Using outdated TLS versions — disable TLS 1.0 and 1.1; require TLS 1.2+

### Session Management

For applications using sessions (cookies, tokens), secure the session mechanism itself.

**Why:** A stolen session is equivalent to a stolen password — the attacker has full access as the user, without needing credentials.

**In practice:**
- **Cookie flags:** Set `HttpOnly` (prevents JavaScript access), `Secure` (HTTPS only), and `SameSite=Lax` or `Strict` (prevents cross-site attachment)
- **Session expiration:** Set reasonable session lifetimes (hours for active sessions, not days or weeks). Require re-authentication for sensitive actions even within a valid session.
- **Session invalidation:** Provide logout that destroys the server-side session, not just the cookie. Invalidate all sessions on password change.
- **Token rotation:** For long-lived tokens, rotate them periodically. A compromised token limits damage to the rotation window.

### Password Storage

Never store passwords in a recoverable format.

**Why:** Database breaches happen. When they do, the question is whether attackers get usable passwords or useless hashes. Proper hashing makes stolen data worthless.

**Properties of a good password hash:**
- **Slow by design** — computational cost should be tunable (work factor / iterations) so that hashing a single password takes 100-500ms. This is imperceptible to users but makes brute-force attacks impractical.
- **Salted** — each password gets a unique random salt, so identical passwords produce different hashes
- **Memory-hard** (ideal) — requires significant memory per hash, making GPU-based cracking expensive
- Never use general-purpose hash functions (MD5, SHA-256) for passwords — they're designed to be fast, which is the opposite of what you want

### Request Body Limits

Set maximum sizes for request bodies and individual fields.

**Why:** Without limits, a single request can exhaust server memory. An attacker (or a buggy client) sending a 10GB POST body shouldn't be able to crash your application.

**In practice:**
- Set a global request body size limit at the web server or framework level (e.g., 10MB for most apps, larger if handling file uploads)
- Validate `Content-Type` headers — reject unexpected content types before parsing
- For file uploads, enforce size limits per file and per request
- For JSON APIs, reject payloads that exceed a reasonable size for the endpoint's purpose

### CSRF Protection

Prevent cross-site request forgery — when a malicious site tricks a user's browser into making requests to your app.

**Why:** If a user is authenticated via cookies, any site they visit can make requests to your API on their behalf.

**In practice:**
- Use CSRF tokens for form submissions
- Check the `Origin` or `Referer` header on state-modifying requests
- Use SameSite cookie attribute (Lax or Strict)

### Content Security Policy (CSP)

Tell browsers which sources of content (scripts, styles, images) are allowed.

**Why:** CSP is a defense-in-depth measure against XSS. Even if an attacker finds an injection point, CSP prevents their script from loading external resources.

**In practice:**
- Start with a restrictive policy and relax as needed
- `script-src 'self'` prevents inline scripts and external script loading
- Report violations to a monitoring endpoint during rollout

### DDoS Considerations

Protect against denial-of-service attacks.

**In practice:**
- Use a reverse proxy or CDN with built-in DDoS protection
- Implement request rate limiting per IP
- Set request body size limits
- Use connection timeouts aggressively
- Consider a web application firewall (WAF) for high-value applications

---

## Security Checklist by Tier

| Requirement | Local | LAN | Internet |
|------------|-------|-----|----------|
| Parameterized queries | Yes | Yes | Yes |
| HTML escaping | Yes | Yes | Yes |
| Secrets not in code/logs | Yes | Yes | Yes |
| Input validation | Yes | Yes | Yes |
| Request timeouts | Yes | Yes | Yes |
| Dependency auditing | Yes | Yes | Yes |
| .gitignore for secrets | Yes | Yes | Yes |
| CORS policy | — | Yes | Yes |
| API key masking | — | Yes | Yes |
| Rate limiting | — | Yes | Yes |
| No sensitive logging | — | Yes | Yes |
| Authentication | — | — | Yes |
| HTTPS | — | — | Yes |
| CSRF protection | — | — | Yes |
| CSP headers | — | — | Yes |
| DDoS protection | — | — | Yes |

---

## Related Documents

- [03-architecture](03-architecture.md) — API design and endpoint security
- [07-error-handling](07-error-handling.md) — Secure error messages (don't leak internals)
- [10-quality-ops](10-quality-ops.md) — Security in CI/CD pipelines
