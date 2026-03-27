# Data Modeling

How to design database schemas, manage data over time, and ensure data integrity. These patterns apply to relational databases, document stores, and embedded databases alike.

---

## Schema Design Principles

### Model the Domain, Not the UI

Design tables/collections around business entities, not around screens or API responses. The UI can always reshape data with queries and joins; a schema designed around one UI is painful to adapt when the UI changes.

**Why:** Domain-driven schemas survive UI redesigns. Screen-driven schemas become technical debt the moment a new view needs the same data in a different shape.

### Normalize, Then Denormalize Deliberately

Start with normalized data (no duplication, each fact stored once). Denormalize only when read performance requires it — and document why.

**Why:** Over-normalization makes reads expensive (too many joins). Under-normalization makes writes dangerous (update one copy, forget the other). Normalize by default; denormalize as a measured optimization.

### Use Proper Types for Queryable Fields

Store dates as dates, numbers as numbers, booleans as booleans. Reserve JSON/text blobs for fields that are stored-and-retrieved but not filtered, sorted, or aggregated.

**Why:** JSON fields are flexible but can't be indexed or type-checked by the database. A `published_at` stored as text can't be range-queried efficiently. Use typed columns for anything that appears in WHERE, ORDER BY, or GROUP BY.

> **CyberPulse example:** Article `categories` are stored as a JSON array (queried via LIKE with quoted values), while `published_at`, `relevance_score`, and `source` are typed columns with indexes.

---

## Index Strategy

### Index What You Filter and Sort

Every column that appears in a WHERE clause or ORDER BY should have an index — or be part of a compound index.

**Why:** Missing indexes are invisible during development (small datasets) and catastrophic in production (large datasets). Adding indexes early is cheap; retrofitting them after users report slow queries is firefighting.

**In practice:**
- Index all foreign keys
- Index all columns used in filters (source, category, status, type)
- Index timestamp columns used for date-range queries
- Use compound indexes for common multi-column queries (e.g., `source + published_at`)
- Index unique constraints (content hashes, email addresses)

### Don't Over-Index

Each index slows down writes and consumes storage. Only index columns that are actually queried.

**Why:** An unused index is pure cost — slower inserts, more disk, more memory. Monitor which queries are slow; add indexes to fix them.

---

## Relationships and Referential Integrity

### Foreign Keys and Constraints

Every relationship between tables should be enforced by the database, not just by application code.

**Why:** Application-level relationship enforcement is one bug away from orphaned records. Database-level foreign keys guarantee integrity even when application code has errors, when scripts run directly against the database, or when a crash interrupts a multi-step operation.

**In practice:**
- Define FOREIGN KEY constraints for every relationship
- Enable foreign key enforcement explicitly if your database requires it (e.g., `PRAGMA foreign_keys = ON` for SQLite)
- Choose CASCADE behavior deliberately for each relationship:

| ON DELETE behavior | Use when | Example |
|-------------------|----------|---------|
| **CASCADE** | Child records are meaningless without the parent | Deleting an article deletes its associated fetch log entries |
| **SET NULL** | Child records can exist independently but lose context | Deleting a category sets articles' category to NULL |
| **RESTRICT** | Deletion should be prevented if children exist | Can't delete a source while articles reference it |

- Default to RESTRICT — it's the safest option. Cascading deletes are convenient but can accidentally remove more data than intended.

### Orphan Prevention

Even with foreign keys, orphans can appear in systems with soft-delete, async processing, or external references.

**In practice:**
- After bulk operations, run periodic integrity checks that scan for orphaned records
- When soft-deleting a parent, decide what happens to children — soft-delete them too, or leave them pointing at a soft-deleted parent?
- For async pipelines where records are created in stages, handle the case where a later stage fails: the partial record should be cleaned up or marked as incomplete

> **CyberPulse example:** Articles reference their source. Foreign keys with RESTRICT prevent deleting a source that still has articles. Fetch log entries CASCADE on article deletion since they're meaningless without the parent record.

---

## Concurrency Control

When multiple processes or users can modify the same data, you need a strategy to prevent conflicts.

### Optimistic Locking

Assume conflicts are rare. Read the record with a version number, make changes, then update only if the version hasn't changed.

**Why:** Optimistic locking adds no overhead when conflicts don't happen (the common case). It's ideal for web applications where users edit records independently and simultaneous edits to the same record are infrequent.

**In practice:**
- Add a `version` (integer) or `updated_at` (timestamp) column to records that can be concurrently edited
- On update, include `WHERE version = :expected_version` — if no rows are updated, another process changed the record
- Return a conflict error to the caller so they can re-read and retry

### Pessimistic Locking

Assume conflicts are likely. Lock the record before reading, preventing others from modifying it until the lock is released.

**Why:** Pessimistic locking guarantees no conflicts but reduces throughput. Use it for operations where a conflict would be expensive (financial transactions, inventory decrements) or where retrying is impractical.

**In practice:**
- Use database-level locks (SELECT FOR UPDATE) within a transaction
- Keep lock duration as short as possible — never hold a lock while waiting for user input or external API calls
- Always release locks in a finally/cleanup block to prevent deadlocks

### When to Choose Which

- **Read-heavy, low contention** — Optimistic locking (or no locking at all)
- **Write-heavy, high contention on same records** — Pessimistic locking
- **Single-user application** — No locking needed; the user is the only writer
- **Background jobs that don't compete** — Use idempotent operations instead of locks

---

## Migration Strategy

### Schema Versioning

Track the current schema version. When the schema changes, write a migration script that transforms the old schema to the new one.

**Why:** Production databases contain data that can't be thrown away. You can't just drop and recreate tables. Migrations are the bridge between "what the schema was" and "what the schema needs to be."

**Approaches:**
- **Embedded migrations** — Store migration SQL in code, run on startup, track applied versions in a metadata table
- **Migration tools** — Use framework-specific tools (Alembic, Flyway, Knex migrations) for complex schemas
- **Manual scripts** — For simple projects, numbered SQL files (`001_initial.sql`, `002_add_expires_at.sql`)

### Forward-Only Migrations

Design migrations to move forward. Rollback migrations sound good in theory but are rarely safe in practice (data transformations are often lossy).

**Why:** A migration that adds a column is easy to roll back (drop the column). A migration that splits a table, transforms data, and updates references cannot be cleanly reversed. Instead of rollback scripts, rely on backups and tested forward migrations.

**In practice:**
- Test migrations on a copy of production data before applying
- Make migrations idempotent where possible (IF NOT EXISTS, INSERT OR IGNORE)
- Never modify a migration that has already been applied — write a new one

### Fixing a Bad Migration

If a migration is wrong but hasn't been deployed to production yet, you have options:
- **Delete and rewrite** — If only you have run it (local development), reset your local database, fix the migration, and re-apply
- **Write a corrective migration** — If others have applied it (shared dev environment), write a new migration that fixes the problem. Never edit the original.
- **Backup and rebuild** — If the migration corrupted data, restore from backup, fix the migration, and re-apply

The key principle: migrations in shared environments are immutable. The only way to "undo" an applied migration is to write a new one that moves the schema forward to the correct state.

---

## Data Lifecycle

### Retention Policies

Data should not grow unbounded. Define how long each type of data lives and what happens when it expires.

**Why:** Databases that grow without bounds become slow, expensive, and hard to back up. Explicit retention policies prevent the "we have 5 years of logs and the database is 200GB" surprise.

**In practice:**
- Add an `expires_at` column to time-bounded data
- Run a daily pruning job that deletes expired rows
- Make retention configurable (store the policy in settings, not hardcoded)
- Consider soft-delete (mark as deleted) before hard-delete for data that might be needed — but understand the implications (see below)

### Soft-Delete Implications

Soft-delete (adding a `deleted_at` timestamp instead of removing rows) preserves data recoverability but introduces ongoing costs:

- **Every query must filter** — `WHERE deleted_at IS NULL` must appear on every read query, or users see deleted records. Missing this filter once creates a confusing bug.
- **Unique constraints break** — If `email` is unique and a user is soft-deleted, no new user can use that email. Workaround: include `deleted_at` in the unique constraint, or use a partial unique index.
- **Reporting complexity** — Aggregations (counts, sums) must explicitly exclude soft-deleted rows. If they don't, reports silently include stale data.
- **Data accumulation** — Soft-deleted records are still in the database. Without periodic hard-delete of old soft-deleted records, the table grows indefinitely.

**When soft-delete is worth the cost:** Records that are expensive to recreate, have legal retention requirements, or where accidental deletion by users is a realistic risk. **When it isn't:** Records that can be re-fetched from an external source, ephemeral data, or single-user tools where accidental deletion is the user's own problem.

> **CyberPulse example:** Articles have a configurable `article_retention_days` setting (default 14). A daily scheduler job at 03:00 UTC deletes articles where `expires_at < now`. The setting is stored in the database and changeable from the UI.

### Archival

For data that shouldn't be deleted but shouldn't be in the active database:
- Move to a separate archive table or database
- Export to files (CSV, JSON) on a schedule
- Compress and store in a dedicated location

---

## Deduplication

### Content Hashing

Before inserting data, compute a hash of the identifying content. Use a UNIQUE constraint on the hash column to prevent duplicates at the database level.

**Why:** Duplicate data confuses users, wastes storage, and corrupts aggregations. Content hashing catches duplicates even when metadata (timestamps, source) differs.

**In practice:**
- Normalize input before hashing (lowercase, strip whitespace, remove trailing slashes from URLs)
- Use SHA-256 or similar; hash the fields that define "sameness" (URL + title, not body text)
- Use INSERT OR IGNORE or ON CONFLICT to handle duplicates gracefully
- Log duplicate counts for visibility

> **CyberPulse example:** Articles are deduplicated by SHA-256 of `normalized(url) + normalized(title)`. The `content_hash` column has a UNIQUE constraint. INSERT OR IGNORE skips duplicates silently. Fetch logs report both new and duplicate counts.

---

## Audit Trails

### Operation Logging

For any process that modifies data in bulk (imports, generations, batch updates), maintain a log table that records what happened, when, and the outcome.

**Why:** Without audit trails, debugging is guesswork. "Why are there only 3 articles from yesterday?" is unanswerable without a fetch log showing that the source was down and returned errors.

**Log table pattern:**
- `id`, `started_at`, `completed_at` — timing
- `operation` — what was done (fetch, generate, prune)
- `parameters` — input (sources, date range, options) as JSON
- `result_counts` — new items, duplicates, errors, total processed
- `errors` — array of error messages with context
- `status` — running, completed, failed

> **CyberPulse example:** The `fetch_log` table records every ingestion run: sources attempted, articles found, duplicates skipped, errors encountered, total duration, and final status.

---

## Backup Strategy

### Automated Backups

Back up the database on a schedule. Retain multiple copies. Verify that backups can be restored.

**Why:** Data loss is unrecoverable without backups. "We'll back up later" becomes "we lost everything" when the disk fails.

**In practice:**
- Run backups daily (or more frequently for high-write databases)
- Use the database's native backup mechanism (SQLite `.backup()`, `pg_dump`, `mongodump`)
- Timestamp backup files for easy identification
- Retain N most recent backups; delete older ones automatically
- Store backups in a different location than the database (different disk, different machine)
- Test restores periodically — a backup you've never restored is an assumption, not a guarantee

> **CyberPulse example:** A scheduler job at 02:00 UTC uses SQLite's `.backup()` API to create timestamped copies in `data/backups/`. The 7 most recent are retained; older copies are deleted automatically.

---

## Storage Patterns

### Key-Value Settings Table

For applications that need runtime-configurable settings:

```
settings (
  key   TEXT PRIMARY KEY,
  value TEXT
)
```

**Why:** Simpler than config files for UI-manageable settings. No need to read/write files or manage environment variables. Works with any database.

**In practice:**
- Seed defaults on first run (INSERT OR IGNORE)
- Validate keys against an allowlist — don't accept arbitrary keys
- Mask sensitive values (API keys) when returning to the frontend
- Store complex values as JSON strings; parse on read

### JSON in Relational Columns

Store arrays and nested objects as JSON strings when:
- The field is stored-and-retrieved, not frequently queried
- The structure is variable or evolving
- A junction table would be overkill for the query patterns

**Why:** JSON columns are flexible and avoid schema changes for nested data. But they sacrifice indexing, type checking, and query ability. Use them for supplementary data, not primary fields.

> **CyberPulse example:** RSS feed URLs are stored as a JSON array in the settings table. Article categories are stored as a JSON array in the articles table. Fetch errors are stored as a JSON array in the fetch_log table.

---

## Related Documents

- [03-architecture](03-architecture.md) — How data modeling fits into overall architecture
- [06-security](06-security.md) — Protecting stored data
- [08-performance](08-performance.md) — Query optimization and indexing
- [10-quality-ops](10-quality-ops.md) — Backup operations and monitoring
