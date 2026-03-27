# Performance

Patterns for building responsive, efficient applications. Performance optimization should be deliberate — measure first, optimize where it matters, and resist premature optimization.

---

## General Principle: Measure Before Optimizing

Don't optimize code you think is slow. Measure, find the actual bottleneck, and fix that.

**Why:** Developer intuition about performance is often wrong. The code you think is slow may execute in microseconds, while an unnoticed query runs for seconds. Profiling gives facts; intuition gives guesses.

**In practice:**
- Log response times for API endpoints
- Use database query logging to identify slow queries
- Profile before adding caching, concurrency, or architectural complexity
- Set performance budgets (e.g., "API responses under 500ms") and test against them

---

## Database Performance

The database is the bottleneck in most applications.

### Write-Ahead Logging (WAL Mode)

For SQLite and databases that support it: enable WAL mode for significantly better concurrent read performance.

**Why:** In default journal mode, readers block writers and vice versa. WAL mode allows concurrent reads and writes — readers see a consistent snapshot while writers append to a log.

**When to use:** Any application with more reads than writes (which is almost all applications).

### Indexing Strategy

→ See [04-data-modeling](04-data-modeling.md) for full details.

Key points:
- Index every column used in WHERE, ORDER BY, and JOIN conditions
- Use compound indexes for common multi-column queries
- Don't over-index — each index slows writes
- Use EXPLAIN to verify queries use indexes

### Query Optimization

- Select only the columns you need, not `SELECT *`
- Use LIMIT on all list queries — never return unbounded results
- For count queries, use `COUNT(*)` on a covering index when possible
- Avoid N+1 queries: fetch related data in bulk, not per-row
- Use `NULLS LAST` in ORDER BY when sorting by optional columns

> **CyberPulse example:** Article generation uses `ORDER BY relevance_score DESC NULLS LAST` to prioritize scored articles while keeping unscored ones at the end. List queries always include LIMIT/OFFSET.

### Connection Management

- Reuse database connections (connection pool or singleton)
- Close connections on application shutdown
- For async applications, use async-compatible drivers (don't block the event loop with sync DB calls)

### Database Maintenance

Databases accumulate overhead that degrades performance over time. Periodic maintenance keeps them healthy.

- **VACUUM** — Reclaims space from deleted rows. Without it, the database file grows but never shrinks. Run periodically (weekly for active databases) or after large bulk deletes.
- **ANALYZE** — Updates query planner statistics so the database can choose optimal query plans. Run after significant data changes (bulk imports, schema changes).
- **Index rebuild** — Indexes can become fragmented with heavy insert/delete patterns. Rebuilding restores optimal structure.
- **WAL checkpointing** — In WAL mode, the write-ahead log grows until checkpointed. Most databases auto-checkpoint, but verify this is happening. Large WAL files indicate stalled checkpoints.

**In practice:**
- Schedule maintenance during low-traffic periods (same window as backups)
- Monitor database file size over time — if it grows much faster than data volume, maintenance is overdue
- For SQLite: `PRAGMA optimize` on connection close handles most statistics updates automatically

### Write Optimization

When inserting or updating large amounts of data, naive approaches can be orders of magnitude slower.

- **Batch inserts** — Insert multiple rows per statement instead of one at a time. A single INSERT with 100 rows is dramatically faster than 100 individual INSERTs.
- **Transaction grouping** — Wrap batch operations in a single transaction. Without explicit transactions, each statement auto-commits, which forces a disk sync per row.
- **Prepared statements** — Reuse compiled statements for repeated operations. The database parses and plans the query once, then executes it many times.
- **Defer index updates** — For very large bulk loads, consider disabling non-essential indexes, loading data, then rebuilding indexes. This avoids updating indexes on every insert.

> **CyberPulse example:** Article ingestion wraps all inserts in a single transaction per source. INSERT OR IGNORE with a content hash handles deduplication within the same transaction.

---

## Pagination

Never return all records in a list endpoint. Always paginate.

### Offset/Limit Pagination

```
GET /api/articles?limit=50&offset=100
```

- Simple to implement
- Return total count alongside results for UI pagination
- Performance degrades at high offsets (database must skip N rows)

**When to use:** Most applications with moderate dataset sizes (< 100K rows).

### Cursor-Based Pagination

```
GET /api/articles?limit=50&after=article_xyz
```

- Consistent performance regardless of page depth
- No "skip to page N" capability (only next/previous)
- More complex to implement

**When to use:** Large datasets, infinite scroll UIs, or when offset performance is measurable.

**In practice:**
- Default to offset/limit — it's simpler and sufficient for most apps
- Switch to cursor-based when offset performance becomes a problem
- Always return enough metadata for the client to render pagination controls

---

## Caching

### What to Cache

- **Expensive computations** — results of complex queries, aggregations, report generation
- **External API responses** — model lists, reference data, rate-limited data
- **Static-ish data** — settings that change rarely, category lists, user preferences

### What NOT to Cache

- Data that changes on every request (current status, live metrics)
- User-specific data in shared caches (security risk)
- Data that's cheap to compute (simple DB lookups with indexes)

### Cache Invalidation

The two hardest problems in CS: cache invalidation, naming things, and off-by-one errors.

- **Time-based (TTL)** — Cache expires after N seconds. Simple; eventual consistency.
- **Event-based** — Invalidate on write/update. Consistent; more complex.
- **Hybrid** — Short TTL + event-based invalidation for critical paths.

**In practice:**
- Start without caching — add it when measurement shows a need
- Cache at the layer closest to the consumer (in-memory > application cache > external cache)
- Log cache hit/miss rates to verify the cache is actually helping
- When in doubt, use short TTLs — stale data is worse than slow data

**TTL guidance by data type:**
- **Volatile data** (real-time status, live metrics): 5-15 seconds, or don't cache at all
- **Semi-stable data** (settings, category lists, model lists): 1-5 minutes
- **Stable reference data** (timezone lists, country codes, static config): 1-24 hours
- **Expensive computations** (aggregations, reports): 5-30 minutes, with event-based invalidation on underlying data changes

---

## Rate Limiting

Protect expensive endpoints from accidental or malicious overuse.

### Endpoint Throttling

Track the last request time for protected endpoints. Reject requests that arrive too soon.

- **Generation/LLM endpoints:** 10-30 second cooldown (prevents credit burn)
- **Bulk import/fetch endpoints:** 30-60 second cooldown (prevents duplicate operations)
- **Authentication endpoints:** 5-10 attempts per minute (prevents brute force)

**Response:** Return HTTP 429 with a `Retry-After` header indicating when the client can retry.

### Per-Client Rate Limiting

For internet-facing applications, limit requests per IP or per authenticated user.

- Use a sliding window or token bucket algorithm
- Return clear error messages: "Rate limit exceeded. Try again in 15 seconds."
- Consider different limits for different endpoints (reads vs writes vs expensive operations)

> **CyberPulse example:** `/api/fetch` has a 30-second cooldown; `/api/generate` has a 10-second cooldown. Both track the last request timestamp and return 429 with remaining wait time.

---

## Async I/O

### When Async Helps

Async is valuable when the application spends most of its time waiting for I/O (network, disk, database) rather than computing.

- **HTTP requests to external APIs** — fetching from multiple sources concurrently
- **Database queries** — especially multiple independent queries
- **File operations** — reading/writing multiple files
- **Long-polling / SSE** — holding connections open without blocking threads

### When Async Doesn't Help

- **CPU-bound work** — data transformation, encryption, image processing (use multiprocessing instead)
- **Sequential operations** — when step 2 depends on step 1's result
- **Simple scripts** — single-threaded, sequential operations with no concurrency

### Practical Patterns

- **Concurrent fetching** — Use `asyncio.gather()` or equivalent to fetch from multiple sources simultaneously
- **Non-blocking database** — Use async database drivers; never call sync DB methods in async handlers
- **Background tasks** — Offload long work to background jobs rather than blocking request handlers
- **Streaming responses** — Use async generators for SSE/streaming without holding threads

---

## Memory Management

### Detecting Memory Issues

Memory leaks are silent performance killers — the application works fine initially and degrades over hours or days.

**Common causes of memory leaks:**
- **Unclosed resources** — database connections, file handles, HTTP clients that aren't cleaned up
- **Growing caches without eviction** — in-memory caches that add entries but never remove them
- **Event listener accumulation** — registering listeners without deregistering on cleanup (especially in SPAs)
- **Circular references** — objects that reference each other and prevent garbage collection (language-dependent)
- **Large objects held in closures** — a callback that captures a reference to a large dataset keeps that dataset alive

**In practice:**
- Monitor process memory usage over time — a steady upward trend indicates a leak
- For long-running processes (servers, schedulers), log memory usage periodically
- Set memory limits on processes so leaks cause a restart rather than exhausting system memory
- When investigating: use language-specific profiling tools to take heap snapshots and compare them over time
- For in-memory caches, always set a maximum size and an eviction policy (LRU or TTL-based)

---

## Frontend Performance

### Lazy Loading

Load resources only when they're needed.

- **View modules** — Load view code on navigation, not on initial page load
- **Images** — Use `loading="lazy"` for below-the-fold images
- **Heavy libraries** — Load Chart.js, PDF renderers, etc. only on views that use them

**Why:** Initial page load speed drives user perception. Loading everything upfront penalizes users who may never visit most views.

### Bundle / Payload Size

- Minimize what's sent to the client
- For no-build frontends: keep individual JS files small and focused
- Compress responses (gzip/brotli) at the server level
- Bundle fonts locally rather than loading from CDNs (eliminates external DNS + connection time)
- Use SVG for icons instead of icon fonts (smaller, scalable, no external dependency)

### Rendering Performance

- Minimize DOM manipulation — batch updates where possible
- Use `textContent` instead of `innerHTML` when inserting plain text
- Avoid layout thrashing — read all DOM measurements first, then write all changes
- Use CSS animations instead of JavaScript animations (GPU-accelerated)
- Debounce event handlers for frequent events (scroll, resize, input)

---

## Network Performance

### Request Optimization

- Combine related API calls when the backend supports it (batch endpoints)
- Use appropriate HTTP methods and cache headers (`Cache-Control`, `ETag`)
- Compress request bodies for large payloads
- Use connection keep-alive for multiple sequential requests

### Timeout Configuration

Set explicit timeouts at every layer:

| Layer | Connect | Total | Use case |
|-------|---------|-------|----------|
| Frontend fetch | — | 30s | UI-triggered API calls |
| Backend → Database | 5s | 30s | Query execution |
| Backend → External API | 10s | 60s | Standard API calls |
| Backend → LLM API | 10s | 120s | Slow AI model responses |

**Why:** Default timeouts are often too high. A 60-second wait for a page load is unacceptable; explicit timeouts let you fail fast and show the user a meaningful error.

---

## Related Documents

- [04-data-modeling](04-data-modeling.md) — Indexing and query optimization
- [06-security](06-security.md) — Rate limiting as security measure
- [07-error-handling](07-error-handling.md) — Timeout handling and retries
- [03-architecture](03-architecture.md) — Async patterns and background jobs
