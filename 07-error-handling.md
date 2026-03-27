# Error Handling

How to categorize, handle, and communicate errors. The goal is a system that is resilient to failure, transparent about problems, and helpful to both users and developers.

---

## Error Categories

Not all errors are equal. Different categories require different handling strategies.

### Validation Errors
Input that doesn't meet requirements (wrong type, missing field, out of range).

- **Source:** User input, API request bodies, query parameters, file uploads
- **Handling:** Return immediately with a clear message explaining what's wrong and what's expected
- **HTTP status:** 400 Bad Request
- **Log level:** Debug (these are expected and frequent)

### Business Logic Errors
Operations that are valid input but violate business rules (insufficient credits, duplicate submission, expired resource).

- **Source:** Application rules, state checks, constraint violations
- **Handling:** Return a descriptive error; suggest corrective action if possible
- **HTTP status:** 409 Conflict, 422 Unprocessable Entity, or domain-specific
- **Log level:** Info or Warning

### Infrastructure Errors
Problems with the system itself (database down, disk full, out of memory).

- **Source:** Database, filesystem, network stack, runtime
- **Handling:** Log with full context; return a generic error to the user; alert if persistent
- **HTTP status:** 500 Internal Server Error
- **Log level:** Error or Critical

### External Service Errors
Failures from third-party APIs, remote databases, or external dependencies.

- **Source:** API timeouts, rate limits, invalid responses, service outages
- **Handling:** Retry with backoff for transient failures; degrade gracefully for persistent failures
- **HTTP status:** 502 Bad Gateway or 503 Service Unavailable (if proxying)
- **Log level:** Warning (transient) or Error (persistent)

---

## User-Facing vs Internal Errors

Every error has two audiences: the user who triggered it and the developer who will debug it.

### What the User Sees
- Clear, actionable language: "API key is missing. Go to Settings to add one." not "NoneType has no attribute 'key'"
- No stack traces, no internal paths, no database details
- Suggested next steps when possible
- Consistent format across all endpoints and views

### What Gets Logged
- Full error message and type
- Stack trace for unexpected errors
- Request context (method, path, parameters — not sensitive data)
- Timestamp, correlation ID if applicable
- Duration (how long the operation ran before failing)

**Why:** Users need to understand what went wrong and how to fix it. Developers need to understand why it went wrong and where. These are different messages for different purposes.

---

## Retry Strategies

### Exponential Backoff

For transient failures (network timeouts, rate limits, temporary unavailability).

```
Attempt 1: immediate
Attempt 2: wait 2 seconds
Attempt 3: wait 4 seconds
Attempt 4: wait 8 seconds (or give up)
```

**Why:** Immediate retries often hit the same problem. Exponential backoff gives the external system time to recover while limiting the total wait time.

**In practice:**
- Set a maximum retry count (3 is typical)
- Only retry on transient errors (429, 503, timeout) — don't retry on 400 or 401
- Log each retry attempt with the reason
- Consider adding jitter (random delay) to prevent thundering herd when multiple clients retry simultaneously

> **CyberPulse example:** The OpenRouter LLM client retries up to 3 times with exponential backoff (2^attempt seconds) on 429 (rate limit) responses.

### Circuit Breaker

For persistent failures — stop trying when a service is clearly down.

- Track failure count for each external service
- After N consecutive failures, "open" the circuit — skip the service entirely for a cooldown period
- After the cooldown, try once ("half-open") — if it succeeds, close the circuit; if it fails, reopen
- Log circuit state changes

**Why:** Retrying a dead service wastes time and resources. Circuit breakers fail fast and let the rest of the system continue.

**When to use:** When the application integrates with multiple external services and can function (degraded) without some of them.

---

## Graceful Degradation

When part of the system fails, the rest should continue functioning.

### Per-Source Error Isolation

When processing multiple items or sources, wrap each in its own error handler.

```
results = []
errors = []
for source in sources:
    try:
        results.extend(process(source))
    except Exception as e:
        errors.append({"source": source, "error": str(e)})
```

**Why:** Failing fast on the first error loses all subsequent work. In a batch of 10 sources, one timeout shouldn't discard the 9 that succeeded.

**In practice:**
- Use per-item try/catch, not per-batch
- Collect errors in a list; report all at the end
- Return partial results with an error summary
- Log each error individually with source context

> **CyberPulse example:** During ingestion, each source (RSS, CISA, NVD) is wrapped in its own try/catch. If CISA is down, RSS and NVD articles are still processed. The fetch log records all errors alongside success counts.

### Feature Degradation

When a non-critical feature fails, disable it rather than crashing the whole application.

- Dashboard chart fails to load → show "Chart unavailable" instead of crashing the page
- Export PDF fails → offer "Try DOCX instead" or show the error
- Search index is stale → show results with a "Results may be outdated" warning

---

## Error Collection Pattern

For multi-step operations (pipelines, batch jobs, multi-source fetches), collect all errors and report them together.

### Pattern

```
errors = []
stats = {"processed": 0, "failed": 0}

for item in items:
    try:
        process(item)
        stats["processed"] += 1
    except Exception as e:
        stats["failed"] += 1
        errors.append({
            "item": item.id,
            "error": str(e),
            "timestamp": now()
        })

log_result(stats, errors)
```

### Reporting

- Store errors in the operation's log entry (→ see audit trails in [04-data-modeling](04-data-modeling.md))
- Surface error count to the user: "Processed 47 articles. 3 sources had errors."
- Make full error details accessible (log viewer, API endpoint, downloadable report)

**Why:** Multi-step operations that fail-fast on the first error waste all prior work. Collecting errors gives a complete picture and lets the user address all issues at once.

---

## Async Cancellation

Long-running operations should be cancellable by the user.

### Pattern

- Create a cancellation signal (event, flag, token) at operation start
- Check the signal at each major step of the pipeline
- On cancellation: clean up partial state, log the cancellation, notify the user
- Return partial results if meaningful

**In practice:**
- Use `asyncio.Event` or framework-specific cancellation primitives
- Provide a cancel endpoint or UI button
- Don't check on every iteration — check between pipeline stages
- Log cancellation with what was completed and what was skipped

> **CyberPulse example:** The fetch operation uses an `asyncio.Event` for cancellation. The signal is checked between pipeline stages (ingestion, classification, condensation). A cancel endpoint sets the event, and the pipeline exits gracefully at the next checkpoint.

---

## Error Messages as a Feature

Good error messages are part of the user experience, not an afterthought.

### Principles
- **Specific:** "Feed URL is not reachable (timeout after 10s)" not "An error occurred"
- **Actionable:** "Check your API key in Settings" not "Unauthorized"
- **Honest:** "This feature is not yet implemented" not a silent failure or 500
- **Consistent:** Same format across all views and endpoints
- **Localized:** Error messages go through i18n like any other UI string

---

## Related Documents

- [01-philosophy](01-philosophy.md) — "Fail gracefully, not silently" principle
- [04-data-modeling](04-data-modeling.md) — Audit trails for error logging
- [06-security](06-security.md) — Don't leak internals in error messages
- [08-performance](08-performance.md) — Timeout configuration
