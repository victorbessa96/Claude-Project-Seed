# Module: File & Media Handling

Patterns for applications that accept, store, process, and serve files. Covers uploads, storage abstraction, image optimization, serving strategies, exports, and file lifecycle management.

**Prerequisite knowledge:** [06-security](../06-security.md) (input validation, request body limits), [08-performance](../08-performance.md) (memory management, streaming, caching), [03-architecture](../03-architecture.md) (layered architecture, background jobs).

---

## Upload Validation

Every uploaded file is untrusted input. Validate before processing.

### File Type Verification

Don't trust the client-provided filename or Content-Type header. Both can be spoofed.

**In practice:**
- Check the file extension against an allowlist of accepted types
- Verify the file's magic bytes (the first few bytes that identify the file format) against the claimed type. A file named `photo.jpg` with PNG magic bytes is suspicious.
- Reject files with double extensions (`document.pdf.exe`) or no extension
- For images: attempt to parse the file with an image library. If it fails, it's not a valid image regardless of extension.

### Size Limits

- Set per-file size limits appropriate to the file type (e.g., 10MB for images, 100MB for documents, 1GB for video)
- Set per-request limits (total upload size)
- Consider per-user quotas for storage-heavy applications
- Reject oversized files as early as possible (at the web server level, before the file reaches application code)

### Filename Sanitization

User-provided filenames can contain path traversal attacks (`../../etc/passwd`), special characters, or excessively long names.

**In practice:**
- Strip or replace all characters except alphanumeric, hyphens, underscores, and dots
- Remove path components (`../`, `./`, absolute paths)
- Truncate to a reasonable length (255 characters max)
- Better yet: generate your own filename (UUID or content hash) and store the original name as metadata

### Malware Considerations

For internet-facing applications that accept file uploads:
- Consider scanning uploads with an antivirus service before storing
- Store uploads in a quarantine location until scanned
- Never execute uploaded files or include them in server-side template rendering
- Serve user-uploaded files from a separate domain or with restrictive Content-Type headers to prevent browser execution

---

## Storage Abstraction

### Local Filesystem vs Object Storage

| Approach | Pros | Cons | Best for |
|----------|------|------|----------|
| **Local filesystem** | Simple, fast, no dependencies | Doesn't scale to multiple servers, backup is manual | Single-server apps, local tools |
| **Object storage** (S3-compatible) | Scalable, built-in redundancy, CDN integration | External dependency, network latency, cost | Multi-server, cloud-deployed, large file volumes |

### Storage Interface Pattern

Abstract file storage behind a simple interface so switching backends is cheap:

```
Storage interface:
  store(key, data, metadata) → URL
  retrieve(key) → data
  delete(key)
  exists(key) → boolean
  get_url(key) → public URL
```

**Why abstract:** Starting with local storage is simpler. When you outgrow it, switching to object storage should require changing one module, not every file that touches storage.

### Path and Key Generation

Don't store files using user-provided names. Generate your own keys.

**Strategies:**
- **Content hash:** SHA-256 of file contents. Identical files get the same key (natural dedup). But renaming a file doesn't change the hash, which may or may not be desired.
- **UUID:** Random, unique, no collisions. No dedup benefit.
- **Structured path:** `uploads/{year}/{month}/{uuid}.{ext}`. Organizes files chronologically, avoids single-directory file count limits.

**In practice:**
- Use structured paths for local filesystem (prevents thousands of files in one directory)
- Store the original filename, content type, and file size as metadata in the database
- The file record in the database maps the user-visible filename to the storage key

---

## Image Optimization

### When to Optimize

Optimize images that will be served to web browsers. Don't optimize images that are stored for archival or download-only purposes.

### Resize Strategy

| Approach | Pros | Cons |
|----------|------|------|
| **Resize on upload** | Fast serving, predictable storage | Can't generate new sizes later without re-upload |
| **Resize on demand** | Flexible, generates any size needed | Slower first request, needs caching, CPU cost |
| **Hybrid** | Generate common sizes on upload, generate others on demand | More complex, best of both |

**Recommendation:** Resize on upload for common sizes (thumbnail, medium, full). This covers most use cases without the complexity of on-demand resizing.

### Common Image Variants

| Variant | Typical size | Use case |
|---------|-------------|----------|
| **Thumbnail** | 150x150px | Lists, grids, avatars |
| **Medium** | 600px wide | Article images, previews |
| **Large** | 1200px wide | Full-screen viewing |
| **Original** | As uploaded | Download, archival |

### Format Conversion

- Convert uploads to efficient web formats (WebP for photos, SVG for graphics when possible)
- Keep the original format as a fallback for browsers that don't support newer formats
- Strip metadata (EXIF data) from images served to the web. EXIF can contain GPS coordinates, camera serial numbers, and other private information.
- Set quality to 80-85% for lossy formats. The quality difference from 85% to 100% is invisible to most users but the file size difference is significant.

---

## Serving Files

### Static File Serving

For files that don't change (application assets, bundled resources):
- Serve directly from the web server (not through application code)
- Set long cache headers (`Cache-Control: public, max-age=31536000`)
- Use content-based filenames or query strings for cache busting (`style.a1b2c3.css`)

### Dynamic File Serving

For files stored by the application (user uploads, generated exports):
- Serve through application code (to check permissions, track downloads)
- Set appropriate `Content-Type` headers based on the file, not the extension
- Set `Content-Disposition` based on intent:
  - `inline`: browser displays the file (images, PDFs)
  - `attachment; filename="report.pdf"`: browser downloads the file

### Range Requests

For large files (video, audio), support partial content requests:
- Respond to `Range` headers with `206 Partial Content`
- This enables seeking in video/audio players and resumable downloads
- Most web servers handle this automatically for static files. For application-served files, implement or use a library.

### Cache Headers

| File type | Cache strategy |
|-----------|---------------|
| **Static assets** (CSS, JS, fonts) | Long cache (1 year) with fingerprinted filenames |
| **User uploads** | Medium cache (1 hour to 1 day) with ETag validation |
| **Generated content** | Short cache or no-cache, depending on how often it changes |
| **Private files** | `Cache-Control: private, no-store`. Never cache on shared proxies. |

### CDN Patterns

For applications with significant file serving load:
- Put a CDN in front of your file serving endpoint
- The CDN caches files at edge locations close to users
- Use origin-pull (CDN fetches from your server on cache miss) rather than origin-push (you upload to CDN) for simplicity
- Set cache headers on your origin server. The CDN respects them.
- For private files: use signed URLs (time-limited, per-user URLs generated by your application)

---

## Temporary Files

### Why Temp Files Accumulate

Applications create temporary files during processing (image resizing, PDF generation, ZIP creation). Without cleanup, they accumulate indefinitely.

### Cleanup Strategies

- **Immediate cleanup:** Delete temp files in a `finally` block after processing. The simplest approach but misses files if the process crashes.
- **TTL-based cleanup:** Create temp files in a dedicated directory. Run a scheduled job that deletes files older than N hours.
- **On-startup cleanup:** Clear the temp directory when the application starts. Handles crash recovery.
- **Combination:** Use immediate cleanup in normal flow + TTL-based cleanup as a safety net.

**In practice:**
- Use a dedicated temp directory, not the system temp directory (easier to manage, monitor, and clean)
- Name temp files with timestamps or UUIDs for easy age-based cleanup
- Monitor temp directory size. If it's growing over time, cleanup isn't working.
- Set filesystem-level quotas on the temp directory to prevent it from filling the disk

---

## Export Generation

### PDF Generation

- Use a library that converts HTML/CSS to PDF. This lets you reuse your existing UI templates.
- For complex layouts: generate HTML server-side, then convert to PDF. Don't build PDFs programmatically with drawing commands unless you need precise control.
- Include page numbers, headers, and footers for multi-page documents.
- Test with real data. PDF generation often breaks with edge cases (very long text, missing images, special characters).

### CSV/Excel Export

- Stream rows to the output file instead of building the entire file in memory. This handles large datasets.
- Include a header row with column names.
- Escape values properly (CSV fields containing commas or quotes need quoting).
- For Excel: use a streaming library that writes rows incrementally rather than building a workbook in memory.
- Set `Content-Disposition: attachment; filename="export-2024-01-15.csv"` so the browser downloads rather than displays.

### ZIP Archives

- Use streaming ZIP creation for large archives (don't buffer the entire ZIP in memory).
- Include a reasonable directory structure inside the ZIP.
- Set a maximum archive size. A user requesting a ZIP of 10GB of files needs a different approach (background job with notification).

### Progress Tracking for Slow Exports

- For exports that take more than a few seconds: run them as background jobs.
- Return immediately with a job ID. The client polls for status or listens via SSE.
- Show progress: "Generating PDF... 45% (page 23/50)"
- When complete, provide a download link (temporary URL with TTL).

> **CyberPulse example:** PDF export of generated output runs synchronously for single articles but could use background generation for bulk exports. The download uses `Content-Disposition: attachment` with a timestamped filename.

---

## Large File Handling

### Chunked Uploads

For files too large to upload in a single request:

1. Client splits the file into chunks (e.g., 5MB each)
2. Client uploads each chunk with its position (chunk 3 of 20) and an upload session ID
3. Server stores chunks temporarily
4. When all chunks arrive, server assembles the final file
5. If the upload is interrupted, the client resumes from the last successful chunk

**Why chunked:** Single-request uploads fail entirely on network interruption. Chunked uploads are resumable. They also avoid request body size limits and server memory issues.

### Streaming Reads

When processing large files, never load the entire file into memory:
- Read and process in chunks (line-by-line for CSV, block-by-block for binary)
- Use streaming parsers for structured formats (streaming JSON, SAX for XML)
- Monitor memory usage during file processing. If it grows linearly with file size, you're buffering.

### Progress Reporting

- For uploads: track bytes received vs total size. Report percentage to the client.
- For processing: track items/rows processed vs total. Report via SSE or polling endpoint.
- For downloads: set `Content-Length` header so the browser shows a progress bar.

---

## File Lifecycle

### Retention Policies

Not all files need to live forever.

| File type | Typical retention |
|-----------|------------------|
| **User uploads** (avatars, documents) | Until user deletes or account is closed |
| **Generated exports** | 24-72 hours (they can be regenerated) |
| **Temporary processing files** | Hours (delete after processing) |
| **Backups** | N most recent (7 daily, 4 weekly) |
| **Log files** | Days to weeks (see [10-quality-ops](../10-quality-ops.md)) |

### Orphan Detection

Files can become orphaned when the database record referencing them is deleted but the file itself isn't.

**In practice:**
- When deleting a record that references a file, delete the file too (or mark it for cleanup)
- Run a periodic job that scans for files not referenced by any database record
- Don't delete orphans immediately. Move them to a "quarantine" directory first, then delete after a grace period.

### Storage Quota Management

For multi-user applications, track storage usage per user:
- Sum file sizes per user from the database
- Enforce upload limits when quota is reached
- Show usage in the UI ("Using 2.3 GB of 5 GB")
- Send warnings at 80% and 95% usage

---

## Related Documents

- [06-security](../06-security.md) -- Upload validation, request body limits, file permission security
- [08-performance](../08-performance.md) -- Streaming, memory management, caching strategies
- [03-architecture](../03-architecture.md) -- Background jobs for async processing, layered architecture
- [modules/data-ingestion](data-ingestion.md) -- File import patterns (CSV/JSON ingestion)
