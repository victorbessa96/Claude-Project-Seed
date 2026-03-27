# Architecture

Structural patterns for organizing software projects. These are tech-agnostic — they describe how to think about structure, not which framework to use.

---

## Module Separation

Every module should have one clear responsibility. If you can't describe what a module does in one sentence, it's probably doing too much.

**Why:** Single-responsibility modules are testable in isolation, replaceable without cascading changes, and comprehensible without reading the entire codebase. When a bug is in "the settings module," you know where to look.

**In practice:**
- Name files after what they do: `settings.py`, `scraper.py`, `export_pdf.py`
- Group by function, not by type: `routers/settings.py` not `controllers/settings_controller.py`
- If a module imports from too many siblings, it's probably an orchestrator that should be explicit about that role
- When a module grows past ~300 lines, look for a natural split

> **CyberPulse example:** Each ingestion source (RSS, CISA, NVD, scraper) is a separate module. Each API resource (articles, settings, fetch, generate) has its own router file. LLM concerns are split across client, prompts, classifier, condenser, and generator.

---

## Layered Architecture

Organize code in layers where each layer depends only on the layer below it.

```
Handlers / Routes      ← HTTP request/response, input parsing
Business Logic         ← Rules, orchestration, validation
Data Access            ← Database queries, ORM, storage
External Services      ← API clients, file I/O, message queues
```

**Why:** Layers that only depend downward can change independently. You can swap the database layer without touching business logic. You can change the HTTP framework without rewriting business rules.

**In practice:**
- Route handlers should be thin — parse input, call business logic, format output
- Business logic should not know about HTTP status codes or SQL syntax
- Data access should not know about HTTP requests
- External service clients should be behind an abstraction (even a simple function) so they can be swapped

**When to deviate:** For very small projects (< 500 lines), strict layers add ceremony without benefit. A single module that does everything is fine for scripts and prototypes.

---

## Cross-Layer Concerns

Some concerns — authentication, logging, metrics, error handling — span all layers. Handled poorly, they tangle the architecture. Handled well, they're invisible plumbing.

**Why:** If every route handler implements its own auth check, or every service method builds its own log line, you get inconsistency and duplication. Cross-layer concerns need to be wired in once, then applied uniformly.

**Patterns:**

- **Middleware/interceptors** — Auth checks, request logging, rate limiting, and error formatting run as middleware that wraps all route handlers. Business logic never sees HTTP auth headers directly.
- **Context propagation** — A request ID generated at the entry point flows through all layers and appears in every log line. This makes tracing a single request through logs trivial.
- **Centralized error formatting** — Each layer throws or returns errors in its own terms (data layer: "record not found"; business logic: "article does not exist"). A single error handler at the route layer translates these into HTTP responses.
- **Metrics collection** — Instrument at layer boundaries (request received, DB query started/finished, external call made) rather than inside business logic. This keeps metrics code out of domain code.

**In practice:**
- Wire middleware in one place (the app entry point), not scattered across routes
- Logging calls in business logic should only capture domain events ("article classified as malware"), not infrastructure events ("HTTP 200 returned") — let middleware handle the latter
- If a concern appears in more than 3 modules, it belongs in middleware or a shared utility

> **CyberPulse example:** Request logging, CORS, and error formatting are middleware wired in `main.py`. The LLM client logs its own API calls (domain concern), but HTTP-level logging is handled by middleware.

---

## Configuration Management

There are three common approaches to storing settings. Choose based on your deployment model.

| Approach | Best for | Trade-offs |
|----------|----------|------------|
| **Database table** (key-value) | Apps with UI-configurable settings | Changeable at runtime without restart; requires DB access |
| **Environment variables** | Cloud-deployed services, 12-factor apps | Standard convention; can't be changed without restart; no UI |
| **Config files** (YAML, TOML, JSON) | Self-hosted tools, complex configuration | Human-readable; version-controllable; no UI without extra code |

**Why this matters:** Configuration approach affects deployment, debugging, and user experience. Mixing approaches (some settings in env vars, some in DB, some in files) creates confusion.

**In practice:**
- Pick one primary approach and use it consistently
- Define defaults so the app works without any configuration (→ see first-run experience in [05-ui-ux](05-ui-ux.md))
- Never store secrets in code or version control
- Validate settings on load — fail fast on invalid configuration

> **CyberPulse example:** All settings live in a SQLite `settings` table (key-value). Default settings are seeded on first run with INSERT OR IGNORE. This allows the UI to manage settings without env vars or config files.

---

## API Design Principles

For projects that expose an API (REST, GraphQL, RPC, or internal module boundaries).

### Resource Naming
- Use nouns for resources: `/api/articles`, `/api/settings`
- Use verbs for actions that don't map to CRUD: `/api/fetch`, `/api/generate`
- Be consistent: if one resource is plural, all are plural

### Error Responses
- Return a consistent error shape: `{ "detail": "Human-readable message" }`
- Use appropriate status codes: 400 (bad input), 404 (not found), 429 (rate limited), 500 (server error)
- Include enough detail to fix the problem, but not so much that you leak internals

### Pagination
- Support `limit` and `offset` (or `page` and `page_size`) for list endpoints
- Return total count so the client can render pagination controls
- Default to reasonable limits (e.g., 50 items) — never return unbounded results

### Filtering and Sorting
- Accept filter parameters as query strings: `?source=rss&category=vulnerability`
- Accept `sort_by` and `sort_dir` with whitelist validation — never pass user input directly to ORDER BY
- Support multi-value filters where appropriate: `?source=rss,cisa`

**Why:** Consistent API design reduces integration friction. Clients (frontend, scripts, other services) can predict how to interact with new endpoints based on patterns they've already learned.

> **CyberPulse example:** Articles endpoint supports `source`, `category`, `relevance` filters, `sort_by`/`sort_dir` with a whitelist of allowed columns, and `limit`/`offset` pagination with total count.

### API Versioning

When your API has external consumers, changes that break the existing contract need a strategy.

**Approaches:**
- **URL prefix** (`/api/v1/articles`) — Simple, explicit, easy to route. Best for most projects.
- **Header-based** (`Accept: application/vnd.myapp.v2+json`) — Cleaner URLs but harder to test and discover.
- **No versioning** — Acceptable when the only consumer is your own frontend, shipped together.

**When to version:**
- Adding new fields to a response: no version needed (additive, non-breaking)
- Removing or renaming a field: version bump (breaking for consumers)
- Changing the shape of a response: version bump
- Changing validation rules: version bump if it rejects previously valid input

**In practice:**
- Start without versioning if you control all consumers
- When you introduce versioning, keep the old version working for a documented deprecation period
- Don't maintain more than two active versions — the maintenance burden compounds quickly
- Log which version each consumer is using, so you know when it's safe to retire the old one

---

## Data Flow Patterns

### Pipeline Architecture

Many applications process data through a series of stages:

```
Ingest → Validate → Transform → Store → Serve
```

**Why:** Explicit stages make each step testable and replaceable. Adding a new stage (e.g., "enrich" between Transform and Store) doesn't require rewriting the pipeline.

**In practice:**
- Each stage should be callable independently (for re-processing, debugging)
- Pass data forward, not state — each stage receives input and produces output
- Log at stage boundaries (started, completed, error count)
- Partial failures in one stage shouldn't prevent other stages from running

> **CyberPulse example:** The article pipeline has four stages: Ingest (fetch from sources) → Classify (assign categories and scores) → Condense (generate summaries) → Serve (API endpoints). Each stage can be triggered independently.

### Event-Driven vs Request-Driven

| Pattern | Best for | Trade-offs |
|---------|----------|------------|
| **Request-driven** (API call → process → respond) | User-initiated actions, CRUD operations | Simple; response includes result; blocks caller |
| **Event-driven** (action happens → trigger downstream) | Background processing, decoupled systems | Non-blocking; harder to debug; eventual consistency |
| **Hybrid** (request triggers → background process → notify) | Long-running operations | Best of both; more complex to implement |

**In practice:**
- Start with request-driven for simplicity
- Move to hybrid when operations take longer than users will wait (> 2-3 seconds)
- Use server-sent events (SSE) or WebSockets to stream progress for long operations

---

## Async vs Sync

### When Async Matters
- **I/O-bound workloads** — HTTP requests, database queries, file reads where the CPU waits for data
- **Concurrent operations** — fetching from multiple sources simultaneously
- **Long-running requests** — streaming responses, server-sent events

### When Sync Is Fine
- **CPU-bound work** — data transformation, computation (async won't help; consider multiprocessing)
- **Simple scripts** — single-threaded, sequential operations
- **Small-scale apps** — if you'll never have more than a few concurrent users, async adds complexity for no gain

**Why:** Async code is harder to read, debug, and reason about. Use it where it provides measurable benefit (I/O concurrency), not as a default.

**The debugging cost of async:** Async introduces challenges that don't exist in synchronous code. Stack traces are often fragmented or missing context. Exceptions can be swallowed silently if a task isn't awaited. Race conditions appear where sequential code had none. Before adopting async, consider whether the concurrency benefit outweighs these costs for your project's scale.

**Structured concurrency:** When using async, prefer patterns that tie concurrent operations to a defined scope — all tasks started in a group are awaited or cancelled when the group exits. This prevents "fire and forget" patterns that lead to orphaned tasks, leaked resources, and errors that nobody handles. The principle: if you start it, you own it until it finishes.

---

## Background Jobs

For operations that run on a schedule or take too long for a request/response cycle.

**Patterns:**
- **Scheduled jobs** — run at intervals (every N hours) or at specific times (daily at 02:00 UTC)
- **Triggered jobs** — initiated by a user action but run in the background
- **Idempotent design** — running the same job twice should produce the same result (handles crashes, retries)

**Why:** Background jobs need different error handling than request/response. There's no user waiting for a response, so errors must be logged and surfaced through other channels (dashboard, logs, alerts).

**In practice:**
- Log the start and end of every background job with duration and outcome
- Implement overlap windows for scheduled fetches (fetch 2x the interval to avoid gaps)
- Store job results in a log table for debugging
- Give users a way to see job status and history

> **CyberPulse example:** Three scheduled jobs — fetch (every N hours), prune (daily at 03:00 UTC), backup (daily at 02:00 UTC). Each logs its outcome. Fetch uses 2x interval windows to prevent missed articles.

---

## SPA Patterns (for web applications)

For single-page applications without build tools or frameworks.

### Hash-Based Routing
- Use URL hashes (`#/dashboard`, `#/articles`) for navigation
- Listen to `hashchange` events to trigger view rendering
- Map routes to render functions in a route dictionary
- Set active nav indicators and ARIA attributes on route change

### View Modules
- Each view is a separate module that renders into a content container
- Views fetch their own data on mount
- Views are stateless — state comes from the server on each navigation

### Centralized API Client
- All HTTP calls go through a single wrapper function
- The wrapper handles errors, JSON parsing, and common headers
- Use relative URLs (`/api/...`) — never hardcode hostnames
- Build query strings from objects, skipping null/empty values

### State Management
- Server is the source of truth — fetch data on each view navigation
- Persist only UI preferences in localStorage (sidebar state, filter selections, sort order)
- Clear persisted state when the user explicitly resets

**Why:** No-build frontends can be well-structured without frameworks. These patterns prevent the common pitfalls (global state spaghetti, duplicated API calls, broken navigation) that make vanilla JS apps hard to maintain.

---

## File and Folder Conventions

A consistent structure across projects reduces the time to find anything.

**Backend:**
```
project/
├── main.py (or index.ts, main.go)    # Entry point, app configuration
├── config.py                          # Constants, defaults
├── database.py                        # Schema, connection management
├── scheduler.py                       # Background job definitions
├── routers/ (or handlers/, routes/)   # One file per API resource
├── services/ (or logic/)              # Business logic modules
├── clients/ (or integrations/)        # External service wrappers
└── export/ (or output/)               # Output formatters
```

**Frontend:**
```
frontend/
├── index.html                         # SPA shell
├── css/style.css                      # Styles with CSS custom properties
├── js/
│   ├── app.js                         # Router, init, global state
│   ├── api.js                         # Centralized API client
│   ├── i18n.js                        # Internationalization strings + helper
│   ├── utils.js                       # Shared utilities
│   └── views/                         # One file per view
└── assets/                            # Fonts, images, icons
```

**Why:** Predictable structure means anyone (including AI assistants) can navigate the codebase by convention. "Where's the settings API?" → `routers/settings.py`. "Where are the translations?" → `js/i18n.js`.

---

## Related Documents

- [04-data-modeling](04-data-modeling.md) — Database schema and storage patterns
- [06-security](06-security.md) — Securing API endpoints and data access
- [07-error-handling](07-error-handling.md) — Error patterns for each architectural layer
- [08-performance](08-performance.md) — Performance considerations per layer
- [modules/real-time](modules/real-time.md) — WebSocket, SSE, and pub/sub patterns for real-time communication
- [modules/authentication](modules/authentication.md) — Auth middleware design and integration with layered architecture
