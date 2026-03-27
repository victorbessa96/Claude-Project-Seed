# Module: Observability & System Health

Patterns for understanding what your system is doing, detecting problems before users report them, and diagnosing issues quickly when they occur. Observability is the difference between "it's broken" and "the article classifier started timing out at 14:32 after the NVD API began returning 5xx errors."

**Prerequisite knowledge:** [10-quality-ops](../10-quality-ops.md) (logging standards, monitoring basics, alerting), [07-error-handling](../07-error-handling.md) (error telemetry, structured error data), [08-performance](../08-performance.md) (what to measure, timeout configuration).

---

## The Three Pillars

Observability rests on three complementary signal types. Each answers different questions.

| Pillar | What it captures | Question it answers | Example |
|--------|-----------------|---------------------|---------|
| **Logs** | Discrete events with context | What happened? | "Article 4821 classified as malware, confidence 0.92" |
| **Metrics** | Numerical measurements over time | How much? How fast? How often? | "p95 response time: 230ms; error rate: 0.4%" |
| **Traces** | Request path across components | Where did time go? Where did it fail? | "Request spent 12ms in routing, 180ms in LLM call, 8ms in DB write" |

**When each pillar matters:**
- **Logs alone** are sufficient for single-server applications with low traffic. Most local tools and small projects live here.
- **Logs + metrics** cover most production web applications. Metrics tell you something is wrong; logs tell you what.
- **All three** are necessary for distributed systems, microservices, or any architecture where a single request touches multiple services.

**Start simple:** Don't instrument everything on day one. Begin with structured logging, add metrics for the operations you care about most, and introduce tracing only when you can't diagnose cross-component issues without it.

---

## Structured Logging in Depth

The core doc ([10-quality-ops](../10-quality-ops.md)) covers log levels and rotation. This section goes deeper into making logs useful at scale.

### Log Schema Design

Every log entry should follow a consistent schema so logs are searchable and parseable by machines, not just humans.

**Essential fields:**
- `timestamp` -- ISO 8601 with timezone (never local time without zone)
- `level` -- DEBUG, INFO, WARNING, ERROR, CRITICAL
- `module` -- which component produced the log (router, classifier, scheduler)
- `message` -- human-readable description of what happened
- `request_id` -- correlation ID linking all logs from a single request or job run

**Contextual fields (when applicable):**
- `operation` -- what was being attempted (classify, fetch, generate)
- `item_id` -- which entity was being processed
- `duration_ms` -- how long the operation took
- `outcome` -- success, failure, skipped, partial
- `error_code` -- machine-readable error identifier

**Why a schema matters:** Without a schema, each developer logs differently. Searching for "all errors in the classifier that took more than 5 seconds" becomes a regex nightmare instead of a structured query.

### Correlation IDs

A correlation ID (or request ID) is a unique identifier generated at the entry point of an operation and passed through every layer.

**How it works:**
1. Generate a UUID when a request arrives or a background job starts
2. Attach it to every log entry, database query tag, and outbound API call
3. When debugging, filter all logs by that ID to see the complete story

**In practice:**
- Generate the ID in middleware (for HTTP requests) or at job start (for background work)
- Pass it via function parameters, thread-local storage, or context objects
- Include it in error responses so users can report it: "Error reference: req_a1b2c3d4"
- For outbound HTTP calls, send it as a header so downstream services can include it in their logs

> **CyberPulse example:** A request to `/api/fetch` could generate ID `fetch_20240115_143200`. Every log line from source fetching, dedup, classification, and storage includes this ID. If 3 of 50 articles fail classification, filtering by the fetch ID shows exactly which articles failed and why.

### Domain Events vs Infrastructure Events

Not every log belongs in every module.

- **Domain events** belong in business logic: "Article classified as ransomware," "Summary generated, 142 tokens," "Feed returned 0 new articles." These tell you whether the application is doing its job.
- **Infrastructure events** belong in middleware and framework code: "Request received," "Response sent 200 in 45ms," "Database connection opened." These tell you whether the system is healthy.

**Why this distinction matters:** When domain logic logs infrastructure events (HTTP status codes, connection pool stats), it couples to infrastructure. When middleware logs domain events (article IDs, classification results), it knows too much about the business. Keep each layer logging its own concerns.

### Log Sampling

At high volume, logging every event becomes expensive (storage, I/O, processing).

- **Debug logs:** Sample or disable entirely in production. Enable per-module when debugging a specific issue.
- **INFO logs for high-frequency paths:** Sample (log 1 in 100) rather than every occurrence. Log 100% for errors regardless.
- **Always log:** Errors, background job outcomes, security events (auth failures, permission denials), and state changes (settings updated, data deleted).

---

## Metrics Collection

### The RED Method

For request-driven services, track three things per endpoint:

| Metric | What it measures | Alert when |
|--------|-----------------|------------|
| **Rate** | Requests per second | Unusual drop (service may be unreachable) or spike (possible abuse) |
| **Errors** | Failed requests per second (or error percentage) | Error rate exceeds baseline by 3x or absolute threshold (> 5%) |
| **Duration** | Response time distribution (p50, p95, p99) | p95 exceeds SLO or doubles from baseline |

**Why RED:** It's simple, covers the most common failure modes, and applies to any request/response system. Start here before adding custom metrics.

### The USE Method

For infrastructure resources (CPU, memory, disk, connections), track:

| Metric | What it measures | Alert when |
|--------|-----------------|------------|
| **Utilization** | Percentage of resource in use | Sustained > 80% |
| **Saturation** | Queue depth or backlog | Growing over time (demand exceeds capacity) |
| **Errors** | Resource-related failures | Connection refused, OOM, disk full |

**When to apply USE:** When performance problems aren't explained by RED metrics. If response times are high but error rates are normal, a resource bottleneck (saturated connection pool, high CPU) is likely.

### Metric Types

| Type | What it does | Use for |
|------|-------------|---------|
| **Counter** | Only goes up (resets on restart) | Total requests, total errors, items processed |
| **Gauge** | Goes up and down | Current connections, queue depth, memory usage |
| **Histogram** | Counts values in buckets | Response time distribution, payload sizes |

**In practice:**
- Name metrics consistently: `{component}_{operation}_{unit}` (e.g., `classifier_requests_total`, `db_query_duration_seconds`)
- Always include units in the metric name (bytes, seconds, total)
- Use labels/tags for dimensions (endpoint, status code, source) but keep cardinality low. A label with 10,000 unique values creates 10,000 time series.

### Business Metrics

Beyond system health, track whether the application is achieving its purpose.

| Example metric | What it reveals |
|---------------|----------------|
| Articles ingested per fetch cycle | Is the pipeline finding content? |
| Classification confidence distribution | Is the LLM producing useful results? |
| Time from ingestion to availability | How fresh is the data? |
| Export generation success rate | Are users getting what they asked for? |

**Why business metrics matter:** System metrics can all be green while the application is functionally broken. If the RSS parser silently returns 0 articles because the feed format changed, health checks pass but the product is failing.

> **CyberPulse example:** Tracking "articles ingested per source per cycle" would immediately reveal if a source stops producing results, even though the fetch job completes successfully with no errors.

---

## Health Checks and Readiness

The core doc ([10-quality-ops](../10-quality-ops.md)) covers basic health endpoints. This section distinguishes between health check types and what to expose.

### Liveness vs Readiness

| Check | Question | Failure means | Response |
|-------|----------|---------------|----------|
| **Liveness** | Is the process alive and not deadlocked? | Restart the process | Lightweight: return 200 if the event loop is responsive |
| **Readiness** | Can the process handle requests right now? | Stop sending traffic (but don't restart) | Check dependencies: DB reachable, required services available |

**Why the distinction:** A process can be alive but not ready (still warming up, lost database connection). Restarting it would make things worse. Readiness checks let load balancers route traffic away without killing the process.

**For single-server applications:** A single `/api/health` endpoint that checks dependencies is sufficient. The liveness/readiness split matters most in orchestrated environments (Kubernetes, container platforms).

### Dependency Health

Check each external dependency independently and report status per dependency.

```
GET /api/health

{
  "status": "degraded",
  "checks": {
    "database": { "status": "healthy", "latency_ms": 2 },
    "llm_api": { "status": "unhealthy", "error": "timeout", "last_success": "2024-01-15T14:00:00Z" },
    "rss_feeds": { "status": "healthy", "sources_reachable": 4, "sources_total": 5 }
  },
  "uptime_seconds": 86412
}
```

**Design rules:**
- Return 200 if the core function works (even if non-critical dependencies are down)
- Return 503 only if the application cannot serve its primary purpose
- Include `last_success` timestamps for async dependencies (feeds, LLM) so operators can see recency
- Keep health checks fast (< 500ms). If a dependency check is slow, cache the result and check in the background.
- Never expose sensitive information (connection strings, credentials, internal IPs) in health responses

### What to Expose vs Hide

| Include | Exclude |
|---------|---------|
| Dependency status (up/down) | Connection strings or credentials |
| Latency measurements | Internal IP addresses |
| Record counts and DB size | Full error stack traces |
| Last operation timestamps | API keys (even masked) |
| Application version | Infrastructure topology |

**For internet-facing apps:** Consider a public health endpoint that returns minimal info (`{"status": "ok"}`) and an internal health endpoint (authenticated or network-restricted) with full diagnostics.

---

## Alerting Strategy

### Alert on Symptoms, Not Causes

| Alert on (symptom) | Not on (cause) |
|--------------------|---------------|
| Error rate > 5% for 5 minutes | A single 500 error |
| p95 latency > 2s for 10 minutes | One slow query |
| Health check failing for 3 consecutive checks | Database CPU at 70% |
| Zero articles ingested in 2 fetch cycles | RSS feed returning different HTML |

**Why symptoms:** Causes are noisy and often self-resolving. A single slow query doesn't need a 3am page. But sustained high latency affecting users does. Alerting on symptoms means you only get woken up when something actually impacts the application's function.

### SLIs and SLOs

**Service Level Indicators (SLIs)** are the metrics you measure. **Service Level Objectives (SLOs)** are the targets you set.

| SLI | SLO | Measurement |
|-----|-----|-------------|
| Availability | 99.9% of requests return non-5xx | Count of successful responses / total responses |
| Latency | 95% of requests complete in < 500ms | p95 response time |
| Freshness | Data no older than 2 fetch cycles | Time since last successful ingestion |
| Correctness | < 1% of outputs require manual correction | Manual review sampling |

**In practice:**
- Set SLOs based on what users actually need, not what sounds impressive
- 99.9% availability means ~8.7 hours of downtime per year. For a personal tool, 99% (3.6 days) is fine.
- Measure SLOs over rolling windows (30 days), not calendar months
- When you're within budget (error budget remaining), ship faster. When budget is low, slow down and focus on reliability.

### Avoiding Alert Fatigue

Alert fatigue is when the team stops responding to alerts because too many are false positives or low priority.

**Prevention:**
- Every alert must have a runbook: what to check, what to do, when to escalate
- Review alerts monthly. If an alert fired 10 times and was ignored 10 times, delete it or fix the threshold.
- Use severity levels and route accordingly:
  - **Critical:** Pages someone immediately (system down, data loss risk)
  - **Warning:** Creates a ticket, reviewed next business day (degraded performance, approaching limits)
  - **Info:** Logged for review, no notification (anomalies worth tracking)
- Require that every new alert has fired at least once in staging/testing before enabling in production

---

## Dashboards

### Operational Dashboard

What the on-call engineer or system operator looks at.

**Include:**
- Current error rate and latency (last 15 minutes)
- Health check status for all dependencies
- Background job status: last run time, outcome, duration
- Resource utilization: disk, memory, connection pool usage
- Recent alerts and their resolution status

**Design rules:**
- The dashboard should answer "is anything broken right now?" in under 10 seconds
- Use red/yellow/green indicators for quick scanning
- Show trends (last hour, last day) not just current values
- Put the most critical signals at the top

### Business Dashboard

What the product owner or stakeholder looks at.

**Include:**
- Items processed over time (articles ingested, reports generated)
- Usage patterns (requests per hour, active features)
- Quality indicators (classification accuracy, generation success rate)
- Pipeline throughput (time from source to availability)

**Design rules:**
- Focus on outcomes, not infrastructure
- Use time ranges that match business cycles (daily, weekly)
- Compare against baselines ("20% more articles than last week")

### Avoiding Vanity Metrics

A vanity metric looks impressive but doesn't drive decisions.

| Vanity metric | Actionable alternative |
|---------------|----------------------|
| "Total articles processed: 50,000" | Articles processed per day (trend) |
| "99.99% uptime" (measured since last restart) | Uptime over rolling 30 days |
| "Average response time: 50ms" | p95 and p99 response time (averages hide outliers) |
| "Zero errors today" | Error rate trend over the past week |

---

## Distributed Tracing

### When Tracing Is Necessary

Tracing adds complexity. Only invest in it when you need it.

**You probably need tracing when:**
- A single user action triggers work across 3+ services
- You can't explain latency from logs and metrics alone
- Debugging requires correlating events across multiple processes

**You probably don't need tracing when:**
- Your application is a single process (correlation IDs in logs are sufficient)
- You have fewer than 3 services communicating
- Your team is small and can hold the architecture in their heads

### Trace Structure

A trace represents one end-to-end operation. It consists of spans.

```
Trace: "Classify article 4821"
├── Span: HTTP handler (12ms)
│   ├── Span: Validate input (1ms)
│   ├── Span: LLM API call (180ms)
│   │   ├── Span: Build prompt (2ms)
│   │   └── Span: HTTP request to LLM provider (178ms)
│   └── Span: Store result in DB (8ms)
```

**Each span records:**
- Operation name
- Start time and duration
- Status (success/error)
- Tags (relevant key-value metadata)
- Parent span ID (for nesting)

### Span Design Principles

- Create spans at meaningful boundaries (HTTP handlers, database calls, external service calls), not around every function
- Name spans after the operation, not the implementation: `classify_article` not `post_handler`
- Keep span count reasonable. 5-20 spans per trace is typical. 200 spans per trace means you're over-instrumenting.
- Attach error information to the span where the error occurred, not just the root span

### Sampling

At high traffic, tracing every request is expensive (network, storage, processing).

- **Always trace:** Errors, slow requests (above p95 threshold), manually flagged requests (debug header)
- **Sample the rest:** 1-10% of normal requests is usually sufficient for understanding system behavior
- **Head-based sampling:** Decide at the start of the request whether to trace it. Simple, but you might miss interesting traces.
- **Tail-based sampling:** Collect all spans, then decide after the request completes whether to keep the trace. Catches errors and slow requests but costs more during collection.

---

## Status Pages and Diagnostics

### Internal Diagnostics Endpoint

Beyond health checks, provide a diagnostics endpoint for operators.

**What to include:**
- Application version and build information
- Configuration summary (which features are enabled, not secret values)
- Runtime information (uptime, language version, OS)
- Recent background job history (last 10 runs with outcomes)
- Connection pool status (active, idle, maximum)

**Access control:** This endpoint should be restricted to operators. Never expose it publicly. Use network-level restrictions or authentication.

### System Self-Reporting

For applications that run unattended (background processors, scheduled jobs), build in self-reporting.

- Log a "heartbeat" entry at regular intervals: "Scheduler alive, 3 jobs pending, last run 5 minutes ago"
- If a background job hasn't produced output in N expected cycles, log a warning automatically
- On startup, log the full configuration (with secrets masked) so operators can verify the deployment

> **CyberPulse example:** The scheduler could log at each cycle: "Fetch cycle complete: 42 new articles from 5 sources, 3 classified, 0 errors, next run in 6h." If this log stops appearing, something is wrong even if no errors are logged.

### External Status Pages

For applications with users who depend on availability:

- Show current system status (operational, degraded, major outage)
- Show status per component (API, background processing, external integrations)
- Show recent incidents with timeline and resolution
- Update proactively, before users report the issue

---

## The Cost of Observability

Observability is not free. Every log line, metric point, and trace span has a cost.

### Storage Costs

| Signal | Storage profile | Retention guidance |
|--------|----------------|-------------------|
| **Logs** | High volume, variable size | 7-30 days for DEBUG/INFO, 90 days for ERROR/CRITICAL |
| **Metrics** | Fixed size per series, grows with cardinality | Raw: 15 days. Downsampled (hourly): 1 year. |
| **Traces** | High volume per trace, very high at scale | 7 days for sampled traces, 30 days for error traces |

### Performance Overhead

- **Logging:** Synchronous logging blocks the request. Use async log writers for high-throughput paths. Expect 1-5% overhead for moderate logging.
- **Metrics:** In-process metric collection (incrementing counters) is negligible. Exporting/scraping adds network I/O.
- **Tracing:** Creating spans adds microseconds per span. Exporting traces adds network overhead. Budget 2-5% overhead.

### Controlling Costs

- Use log levels aggressively. DEBUG off in production. INFO only for meaningful events.
- Set metric cardinality budgets. If a label has more than 100 unique values, reconsider whether you need that granularity.
- Sample traces (see Sampling section). 100% tracing is almost never justified outside of debugging sessions.
- Set retention policies and enforce them. Old data that nobody queries is just cost.
- Review observability spend quarterly. Remove metrics nobody dashboards, alerts nobody acts on, and log lines nobody searches.

---

## Putting It Together

### Observability Maturity Levels

| Level | What you have | Appropriate for |
|-------|--------------|----------------|
| **1: Basic** | Structured logs, log rotation, health endpoint | Local tools, personal projects, prototypes |
| **2: Monitored** | Level 1 + key metrics (RED), alerting on symptoms, operational dashboard | Production single-server apps, small team projects |
| **3: Observable** | Level 2 + correlation IDs, business metrics, SLOs, runbooks | Production apps with users depending on reliability |
| **4: Instrumented** | Level 3 + distributed tracing, sampling, status page | Multi-service architectures, large-scale systems |

**Start at Level 1.** Most projects never need Level 4. Move up when you experience pain that the current level can't address, not preemptively.

> **CyberPulse example:** As a single-server application, CyberPulse fits Level 2. Structured logging, health endpoint with dependency checks, scheduled job monitoring, and basic alerting (error rates, job failures) cover its needs. Distributed tracing would be overhead with no benefit.

---

## Related Documents

- [10-quality-ops](../10-quality-ops.md) -- Logging standards, monitoring basics, incident response
- [07-error-handling](../07-error-handling.md) -- Error telemetry, structured error data, alerting thresholds
- [08-performance](../08-performance.md) -- What to measure, timeout configuration, caching metrics
- [03-architecture](../03-architecture.md) -- Cross-layer concerns, middleware for instrumentation
