# Module: Data Ingestion

Patterns for collecting, processing, and normalizing data from external sources — RSS feeds, REST APIs, web scraping, webhooks, and file imports.

**Prerequisite knowledge:** [03-architecture](../03-architecture.md) (pipeline patterns, background jobs), [04-data-modeling](../04-data-modeling.md) (deduplication, data lifecycle), [07-error-handling](../07-error-handling.md) (graceful degradation, error collection).

---

## Source Abstraction

Each data source should be an independent module with a common interface.

```
Source module interface:
  input:  configuration (URL, API key, time window)
  output: list of normalized items
  errors: collected, not thrown
```

**Why:** Sources fail independently. An API that's down shouldn't prevent RSS feeds from being processed. Independent modules can be tested, retried, and replaced individually.

**In practice:**
- One file per source: `rss.py`, `cisa_kev.py`, `nvd_cve.py`, `scraper.py`
- Each module exports an async function that returns a list of items
- Common item shape (normalized) — regardless of source, the output looks the same
- The orchestrator calls each source, collects results, and merges them

> **CyberPulse example:** Four independent ingestion modules (RSS, CISA, NVD, scraper). Each returns a list of article dicts with normalized fields. The fetch router orchestrates them, collecting results and errors from each.

---

## Scheduling

### Configurable Intervals

Let users (or settings) control how often ingestion runs.

**Why:** Different use cases need different frequencies. News aggregation needs hourly updates; compliance data needs daily checks; social feeds need near-real-time.

**In practice:**
- Store the interval in settings (not hardcoded)
- Use a scheduler library for background execution (APScheduler, cron, Celery Beat)
- Log every scheduled run with its outcome

### Overlap Windows

When fetching time-bounded data, use a window larger than the interval to prevent gaps.

```
Fetch interval: every 6 hours
Fetch window:   last 12 hours (2x interval)
```

**Why:** If a scheduled run is delayed by 30 minutes, a window equal to the interval misses 30 minutes of data. Overlap ensures coverage even with timing variations. Deduplication (see below) handles the duplicate items from the overlap.

> **CyberPulse example:** Scheduled fetch uses 2x the configured interval as the time window. Deduplication via content hash prevents duplicate storage.

### Idempotent Fetches

Running the same fetch twice should produce the same result (no duplicates, no side effects).

**Why:** Scheduled jobs can fire twice (restarts, clock skew, manual trigger during scheduled run). Idempotent fetches combined with deduplication ensure correctness regardless of timing.

---

## Deduplication

→ See [04-data-modeling](../04-data-modeling.md) for full dedup patterns.

For ingestion specifically:

### Content Hashing

- Compute a hash of the fields that define "sameness" (URL + title, not body — body may change)
- Normalize inputs before hashing (lowercase, strip whitespace, remove tracking parameters from URLs)
- Use a UNIQUE constraint on the hash column — let the database enforce uniqueness
- Use INSERT OR IGNORE (SQLite) or ON CONFLICT DO NOTHING (PostgreSQL) for graceful handling

### Dedup Reporting

- Count and log duplicates per fetch run
- Report both new items and duplicates to the user ("47 new, 12 duplicates")
- High duplicate rates may indicate the fetch window is too large (optimization signal)

---

## Rate Limiting and Respectful Fetching

### Per-Domain Rate Limiting

When scraping or fetching from web sources, respect the server's capacity.

**Why:** Aggressive fetching gets your IP blocked, violates terms of service, and harms the source server. Being a good citizen ensures long-term access.

**In practice:**
- Track the last request time per domain
- Enforce a minimum delay between requests to the same domain (2 seconds is typical)
- Respect `robots.txt` directives when scraping
- Set an informative `User-Agent` header identifying your application

> **CyberPulse example:** Per-domain rate limiting with a `defaultdict(float)` tracking last request time. 2-second minimum delay between requests to the same domain. Custom User-Agent header.

### API Quota Management

For APIs with explicit rate limits (NVD API: 5 requests/30 seconds without key, 50/30s with key):

- Read and respect the rate limit headers from API responses
- Implement a request queue with timing control
- Use API keys when available (usually higher limits)
- Cache responses when the data doesn't change frequently

---

## Content Extraction

### HTML Content Processing

When fetching web pages, raw HTML contains navigation, ads, footers, and scripts. Extract the meaningful content.

**Strategy (in priority order):**
1. Remove known noise elements: `<script>`, `<style>`, `<nav>`, `<footer>`, `<form>`, `<iframe>`
2. Look for semantic containers: `<article>`, `<main>`, `div.article-content`, `div.post-body`
3. Fall back to `<body>` if no semantic container found
4. Extract text content, preserving paragraph structure

**Why:** LLM processing of raw HTML wastes tokens on boilerplate. Clean content produces better summaries, classifications, and generations.

### Content Enrichment

When the initial fetch returns insufficient content (short RSS descriptions), optionally fetch the full article.

**In practice:**
- Check content length after initial fetch
- If below a threshold (e.g., < 200 characters), fetch the full URL and extract content
- Apply per-domain rate limiting to enrichment fetches
- Make enrichment optional (some sources block scrapers)

> **CyberPulse example:** After RSS fetch, articles with short content are enriched by scraping the source URL. The scraper removes boilerplate HTML and extracts article text from semantic containers.

---

## Normalization

### Why Normalize

Different sources provide data in different formats: date strings vary, field names differ, text encoding is inconsistent. Normalization ensures all data in the database has a predictable shape.

### Normalization Checklist

| Field | Normalization |
|-------|---------------|
| **Dates** | Parse to ISO 8601 / UTC; handle timezone variations |
| **URLs** | Strip tracking parameters, normalize scheme, resolve relative URLs |
| **Text** | Decode HTML entities, normalize Unicode, strip excessive whitespace |
| **Categories/tags** | Map source-specific categories to a common taxonomy |
| **Author names** | Trim, normalize case, handle "Staff" / "Admin" / missing |
| **Content** | Strip HTML, remove boilerplate, extract plain text |

**In practice:**
- Normalize in the source module, before returning items
- Log normalization failures (unparseable dates, invalid URLs) as warnings
- Define a canonical item schema that all sources must conform to

---

## Error Resilience

### Per-Source Error Isolation

Each source is wrapped in its own error handler. One source failing doesn't affect others.

```
all_items = []
all_errors = []

for source in active_sources:
    try:
        items = await fetch_source(source)
        all_items.extend(items)
    except Exception as e:
        all_errors.append({
            "source": source.name,
            "error": str(e),
            "timestamp": utcnow()
        })

# Process all_items even if some sources failed
```

**Why:** External sources are unreliable. APIs go down, feeds break, servers block scrapers. Partial results are better than no results.

### Source Health Tracking

Track the success/failure rate of each source over time.

- Record whether each source succeeded or failed in each fetch run
- Surface source health in the UI ("CISA: last success 2 hours ago" vs "NVD: failing for 3 days")
- Consider disabling chronically failing sources (or reducing their fetch frequency)

---

## Logging and Audit

### Fetch Log Pattern

Every fetch run should produce a log entry with:

| Field | Purpose |
|-------|---------|
| `started_at`, `completed_at` | Duration tracking |
| `sources` | Which sources were attempted |
| `items_new` | Count of new items stored |
| `items_duplicate` | Count of duplicates skipped |
| `items_enriched` | Count of items with full-text fetched |
| `errors` | Array of error objects (source + message) |
| `status` | running / completed / failed / cancelled |

**Why:** Without fetch logs, diagnosing ingestion problems is guesswork. "Why are there no articles today?" becomes answerable: "The RSS source returned a 503 and the NVD API timed out."

**In practice:**
- Create the log entry at fetch start (status: running)
- Update it at fetch end with counts and errors
- Display fetch history in the UI with expandable error details
- Use fetch logs for monitoring: alert if fetch fails N times consecutively

> **CyberPulse example:** `fetch_log` table records every ingestion run with sources attempted, article counts (new/duplicate), errors, duration, and status. Displayed in the Sources view.

---

## Pipeline Integration

### Ingestion as Pipeline Stage

Ingestion is typically the first stage in a data pipeline:

```
Ingest → Deduplicate → Enrich → Classify → Process → Store → Serve
```

Each stage should be independently runnable:
- **Ingest** — fetch raw data from sources
- **Deduplicate** — filter items that already exist
- **Enrich** — fetch full content for items with insufficient data
- **Classify** — tag, score, or categorize items (→ see [ai-llm-integration](ai-llm-integration.md))
- **Process** — transform into the storage schema
- **Store** — persist to database
- **Serve** — make available via API

**Why:** Independent stages allow re-processing (re-classify without re-fetching), debugging (inspect intermediate results), and optimization (skip stages when not needed).

---

## Related Documents

- [03-architecture](../03-architecture.md) — Pipeline architecture, background jobs, scheduling
- [04-data-modeling](../04-data-modeling.md) — Deduplication, data lifecycle, audit trails
- [07-error-handling](../07-error-handling.md) — Per-source error isolation, error collection
- [08-performance](../08-performance.md) — Rate limiting, async I/O for concurrent fetching
- [modules/ai-llm-integration](ai-llm-integration.md) — LLM-powered classification and processing of ingested data
