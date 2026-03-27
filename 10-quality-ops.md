# Quality & Operations

Patterns for keeping software running reliably after it's built — logging, monitoring, backups, dependency management, CI/CD, and code style.

---

## Logging

### Structured Logging

Log entries should be parseable, not just readable. Include consistent fields: timestamp, level, module, message, and context.

**Why:** Unstructured log lines ("something went wrong") are useless for debugging. Structured logs can be searched, filtered, aggregated, and alerted on.

**In practice:**
- Use log levels appropriately:
  - **DEBUG** — detailed diagnostic info (disabled in production)
  - **INFO** — normal operations (startup, shutdown, scheduled jobs completing)
  - **WARNING** — unexpected but handled situations (retry triggered, deprecated usage)
  - **ERROR** — failures that need attention (external service down, data corruption)
  - **CRITICAL** — system-level failures (database unreachable, out of memory)
- Include context: which operation, which item, what was the input (minus sensitive data)
- Use consistent message formats across modules

### Log Rotation

Logs must not grow unbounded. Configure rotation by size or time.

**Why:** Unbounded log files fill disks, crash applications, and make debugging harder (searching a 10GB file is not fun).

**In practice:**
- Rotate by size (e.g., 5MB per file, 3 backup files)
- Or rotate daily with retention (keep last 7 days)
- Use the framework's built-in rotation (RotatingFileHandler, logrotate, etc.)
- Never use a plain file handler in production

> **CyberPulse example:** `RotatingFileHandler` with 5MB max size and 3 backup files. Third-party loggers (httpx, httpcore) are suppressed to reduce noise.

### What NOT to Log

- API keys, passwords, tokens, or any secrets
- Full request/response bodies (may contain sensitive data)
- Personal information unless required for debugging (and compliant with privacy policy)
- Every individual item in a bulk operation (log counts and summaries instead)

---

## Monitoring

### Health Endpoints

Every application should have a health check that reports:
- **Status** — is the app running and able to serve requests?
- **Dependencies** — is the database reachable? Are external services responding?
- **Metrics** — database size, record counts, last operation timestamps

**Why:** Health endpoints are the simplest form of monitoring. They answer "is it working?" without requiring external monitoring infrastructure.

**In practice:**
- `GET /api/health` or `GET /healthz` — returns JSON with status and metadata
- Include enough detail to diagnose common problems (DB size, last fetch time, error counts)
- Keep the endpoint fast (no expensive queries) — it may be polled frequently
- Return appropriate HTTP status (200 for healthy, 503 for degraded)

> **CyberPulse example:** Health endpoint returns DB size, article count, last fetch time, LAN IP, and app status. Used to populate the dashboard status section.

### Key Metrics to Track

| Category | Metrics | Why |
|----------|---------|-----|
| **Availability** | Uptime, response rate, error rate | Is the app working? |
| **Performance** | Response time (p50, p95, p99), query duration | Is it fast enough? |
| **Usage** | Request count, active users, API call volume | Is it being used? |
| **Resources** | Database size, disk usage, memory usage | Will it keep working? |
| **Business** | Items processed, outputs generated, errors per run | Is it doing its job? |

### Alerting

For production applications, set up alerts for:
- Health endpoint returning non-200 for > 5 minutes
- Error rate exceeding baseline by 3x
- Disk usage above 80%
- Background jobs failing consistently

**When to skip formal monitoring:** For local/personal tools, the health endpoint is sufficient. Add alerting when the app serves others or runs unattended.

---

## Backups

→ See [04-data-modeling](04-data-modeling.md) for backup strategy details.

Key points:
- Automate backups on a schedule (daily minimum for databases with user data)
- Retain multiple copies (7 daily backups is a reasonable default)
- Store backups separately from the primary data
- Test restores periodically — an untested backup is not a backup
- Log backup operations (success/failure, size, duration)

---

## Dependency Management

### Version Pinning

Pin all dependencies to exact versions for reproducible builds.

```
# Good
fastapi==0.104.1
aiosqlite==0.19.0

# Bad
fastapi>=0.100
aiosqlite
```

**Why:** Unpinned dependencies pull new versions on every install. New versions can introduce breaking changes, new bugs, or incompatibilities. Pinning ensures every install produces the same result.

### Update Strategy

- Update dependencies deliberately, not implicitly
- Review changelogs before updating (especially major versions)
- Test after updating — run the full test suite
- Update frequently in small batches (monthly) rather than rarely in large batches
- Use vulnerability scanners to identify security-critical updates

### Dependency Evaluation

Before adding a dependency, evaluate:
- **Maintenance** — Is it actively maintained? When was the last release?
- **Size** — How much does it add to the install? Is that proportional to the value?
- **Alternatives** — Could you write the 20 lines of code this library provides?
- **Transitive dependencies** — What else does it pull in?
- **License** — Is the license compatible with your project?

---

## CI/CD Patterns

### Pipeline Stages

```
Commit → Lint → Test → Build → Deploy
```

- **Lint** — code style, static analysis, type checking
- **Test** — unit tests, then integration tests (fail fast on unit failures)
- **Build** — compile, bundle, package
- **Deploy** — push to staging, then production (with approval gate for production)

### Environment Separation

| Environment | Purpose | Data | Access |
|------------|---------|------|--------|
| **Development** | Local coding and debugging | Synthetic/test data | Developers only |
| **Staging** | Pre-production validation | Copy of production data (anonymized) | Team + CI/CD |
| **Production** | Live users | Real data | Restricted; deploy through pipeline only |

**In practice:**
- Never test against production data during development
- Staging should mirror production as closely as possible (same versions, same config, same infrastructure)
- Deploy to staging automatically on merge; deploy to production manually or with approval
- Use separate credentials for each environment — never share database passwords, API keys, or service accounts across environments
- Configuration differences between environments should be explicit and minimal — if staging behaves differently from production, the staging validation is less trustworthy

**Environment parity matters:** The most common "works in staging, breaks in production" failures come from: different database versions, different OS/runtime versions, different network topology, or different data volumes. Close the gaps proactively.

### Secrets in CI/CD

CI/CD pipelines need access to credentials (API keys, deploy tokens, database passwords) but should never expose them.

**In practice:**
- Store secrets in your CI platform's secret management (not in the repo, not in environment files checked into version control)
- Never echo or print secrets in build logs — mask them in pipeline output
- Use short-lived credentials when possible (tokens that expire after the build)
- Rotate CI/CD secrets on the same schedule as production secrets
- Limit secret access to the pipeline stages that need them — the lint stage doesn't need the deploy key

### Rollback and Deployment Strategies

When a deployment goes wrong, you need a way back.

**Strategies:**
- **Rollback to previous version** — Keep the previous deployment artifact available. If the new version fails, redeploy the old one. This is the simplest strategy and sufficient for most applications.
- **Blue-green deployment** — Run two identical environments. Route traffic to the new version; if it fails, switch back to the old one. Requires infrastructure to support two environments.
- **Canary deployment** — Route a small percentage of traffic to the new version. Monitor for errors. Gradually increase traffic if healthy. Requires traffic routing capability.
- **Feature flags** — Deploy new code but hide it behind a flag. Enable for specific users or percentages. Roll back by disabling the flag without redeploying.

**Which to use:**
- **Personal/small apps** — Manual rollback (redeploy previous version) is sufficient
- **Team apps with downtime tolerance** — Blue-green gives instant rollback
- **Apps that can't afford downtime** — Canary limits blast radius
- **Apps with complex features** — Feature flags allow incremental rollout independent of deployment

### Incident Response

When something breaks in production, a clear response process prevents panic and reduces resolution time.

**Detection:**
- Health checks and monitoring (→ see Monitoring section above) catch most failures automatically
- User reports catch what monitoring misses — have a clear channel for users to report issues
- Log anomalies (error spikes, unusual patterns) can be early indicators

**Triage:**
- **Severity 1 (outage)** — The application is down or data is being corrupted. All other work stops. Fix or rollback immediately.
- **Severity 2 (degraded)** — The application works but a significant feature is broken. Fix within hours.
- **Severity 3 (minor)** — A non-critical feature has a bug. Fix in normal workflow.

**During the incident:**
- Communicate status to affected users (even "we're aware and investigating" is better than silence)
- Log what you're doing and what you're finding — this becomes the post-mortem timeline
- Prefer rollback over hotfix if the rollback is safe — a known-good state is better than a rushed fix

**After the incident (post-mortem):**
- Write a brief post-mortem: what happened, what was the impact, what was the root cause, what will prevent recurrence
- Focus on systemic fixes, not blame — "we need better monitoring" not "someone pushed bad code"
- Track action items from post-mortems to completion

### When to Skip CI/CD

For personal tools and small projects, a local test + manual deploy is fine. Add CI/CD when:
- Multiple people contribute
- Deployments happen frequently (> weekly)
- Manual deployment has caused errors
- You want to enforce quality gates (lint, test) automatically

---

## Code Style

### Consistency Over Preference

The specific style matters less than consistent application. Pick conventions and enforce them with tools.

**Why:** Style debates waste time. Automated formatting ends them. When all code looks the same, differences in diffs are meaningful changes, not formatting noise.

**In practice:**
- Use a formatter (Prettier, Black, gofmt, rustfmt) — run it on save or pre-commit
- Use a linter (ESLint, flake8, clippy) — catch bugs, not just style
- Configure once, enforce everywhere
- Don't fight the formatter — adapt to its defaults

### Code Organization Conventions

- **One responsibility per file** — name files after what they do
- **Consistent naming** — if routes are in `routers/`, don't put one in `handlers/`
- **Predictable location** — anyone should guess where to find code by convention
- **Minimal comments** — code should be self-explanatory; comment the "why," not the "what"
- **No dead code** — delete unused code; git remembers it if you need it back

---

## Documentation

### What to Document

- **Decisions** — why you chose X over Y (architecture decision records)
- **Setup** — how to install, configure, and run the project
- **Non-obvious behavior** — workarounds, known issues, surprising interactions
- **API contracts** — endpoints, request/response shapes, error codes

### What NOT to Document

- **How the code works** — the code documents itself (if it doesn't, refactor it)
- **Obvious patterns** — don't document that the REST endpoint does REST things
- **Ephemeral state** — current bugs, in-progress features (use issue trackers for these)

### Where to Document

- **README** — setup, running, and high-level overview (in the repo root)
- **CLAUDE.md** — AI-specific conventions and decisions (→ see [11-ai-collaboration](11-ai-collaboration.md))
- **Inline comments** — explain "why" for non-obvious code
- **Architecture docs** — in `docs/` for multi-component systems
- **API docs** — auto-generated from code (OpenAPI/Swagger) when possible

**Why:** Documentation close to code stays current. Documentation in separate systems drifts. The best documentation is the code itself, supplemented by decision records and setup guides.

---

## Related Documents

- [04-data-modeling](04-data-modeling.md) — Backup strategy details
- [06-security](06-security.md) — Security in CI/CD pipelines
- [09-testing](09-testing.md) — Running tests in CI/CD
- [11-ai-collaboration](11-ai-collaboration.md) — Documentation for AI-assisted development
- [modules/observability](modules/observability.md) — Deep dive on logging, metrics, tracing, alerting, and dashboards
