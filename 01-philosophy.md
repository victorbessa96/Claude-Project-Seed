# Development Philosophy

The principles in this document are the foundation everything else builds on. They are not rules to follow blindly — they are defaults to follow deliberately and deviate from consciously.

---

## Simplicity Over Cleverness

Write the minimum code needed to solve the current problem. Resist the urge to abstract, generalize, or "future-proof" until the need is proven by repetition.

**Why:** Overengineered code is harder to debug, harder to modify, and harder to hand off. Three similar lines of code are better than a premature abstraction. The right amount of complexity is the minimum needed for the current task.

**In practice:**
- Don't create helpers, utilities, or abstractions for one-time operations
- Don't add error handling for scenarios that can't happen
- Don't add configuration for things that don't need to be configurable yet
- If a feature can be built simply, build it simply — even if a "better" architecture exists
- Delete dead code; don't comment it out "just in case"

---

## Measure Twice, Cut Once

Understand the problem and the existing code before changing anything. Read before writing. Plan before building. Investigate before deleting.

**Why:** Undoing a bad change costs more than the time spent understanding the problem first. Unfamiliar files, branches, or state may represent someone's in-progress work. Assumptions compound — one wrong assumption early leads to a cascade of rework.

**In practice:**
- Read the file before editing it
- Understand the caller before changing the function
- Check git history before deleting "unused" code
- When encountering unexpected state, investigate before overwriting
- For non-trivial tasks, write down the approach before coding

---

## Incremental Delivery

Produce working software at every stage. Never go dark for long. Small, frequent progress beats large, infrequent deliveries.

**Why:** Frequent feedback prevents drift between what's built and what's needed. Working software at each step builds confidence and creates natural rollback points. Large batches of work hide bugs and make debugging harder.

**In practice:**
- Each phase should produce something usable, even if incomplete
- Prefer small commits that each leave the system in a working state
- Build the core path first, then add edge cases
- Deploy early, even if the feature set is minimal
- If a task will take days, find intermediate milestones that deliver value

---

## Fail Gracefully, Not Silently

Partial failures should not crash the system. But they must be visible — logged, surfaced, or reported. Silent failures are the hardest bugs to find.

**Why:** A system that crashes on one failing input loses all other work. A system that silently swallows errors gives the illusion of success while accumulating hidden problems. The goal is resilience with visibility.

**In practice:**
- Catch errors at the right granularity (per-item, not per-batch)
- Log every caught error with enough context to debug later
- Accumulate errors and report them at the end, not just the first one
- Surface error counts and details to the user (not just "something went wrong")
- Never catch-and-ignore without a comment explaining why

> **CyberPulse example:** During article ingestion, each source (RSS, CISA, NVD) is wrapped in its own try/catch. If CISA's API is down, RSS and NVD articles still get processed. All errors are collected in an array and stored in the fetch log for review.

---

## Local-First, Cloud-Optional

Prefer local storage and local processing. Add external services deliberately, not by default. Every external dependency is a point of failure, a privacy concern, and a cost center.

**Why:** Local-first apps start faster, work offline, are easier to debug, and don't surprise you with bills. External services should be added when the local alternative is genuinely inadequate — not because "that's how it's usually done."

**In practice:**
- Use local databases (SQLite, embedded stores) for single-user or small-team apps
- Process data locally unless the computation requires cloud-scale resources
- Bundle assets (fonts, icons) instead of loading from CDNs when feasible
- Store configuration locally (database, config files) rather than in cloud services
- When you do use external services, abstract them behind a client layer so switching is cheap

> **CyberPulse example:** All data is stored in local SQLite. Fonts are bundled as WOFF2 files. The only external service is the LLM API (OpenRouter), which is abstracted behind a single client module.

---

## Convention Over Configuration

Establish patterns early. Follow them consistently. Make deviations deliberate and documented.

**Why:** Consistent patterns reduce decision fatigue. When every module follows the same structure, new code is predictable — easier to read, review, and navigate. Inconsistency forces readers to wonder "is this different for a reason, or was this written by a different person on a different day?"

**In practice:**
- Define file organization conventions before writing code
- One responsibility per module — name files after what they do
- Use the same error handling pattern everywhere
- If you deviate from a convention, document why in a comment
- New code should look like it belongs next to existing code

> **CyberPulse example:** Every ingestion source (RSS, CISA, NVD, scraper) is a separate module with an async function that returns a list of articles. Every router file handles one resource. Every LLM prompt lives in `prompts.py`.

---

## Ship, Then Polish

Working software beats perfect software. Get the core functionality right first. Polish — animations, edge cases, cosmetic improvements — comes after the foundation is solid.

**Why:** Premature polish targets the wrong things. You don't know which parts of the UI matter most until someone uses it. A beautiful loading animation is wasted if the feature behind it doesn't work. Real usage reveals what actually needs polish.

**In practice:**
- Build the happy path first, then handle edge cases
- Hardcoded values are fine during prototyping — extract config later
- Skip animations and transitions in the first pass
- Ship to yourself or a small audience early; iterate based on real feedback
- Polish is a phase, not an afterthought — but it comes after core features work

---

## Reversibility Matters

Prefer actions that can be undone. When you must take irreversible actions, proceed with extra care and verification.

**Why:** Reversible actions are cheap to try. Irreversible actions — deleting data, force-pushing, sending messages — have compounding consequences. The cost of pausing to verify is low; the cost of an unwanted irreversible action can be very high.

**In practice:**
- Soft-delete before hard-delete when feasible
- Backup before destructive operations
- Commit before refactoring so you can revert
- For destructive operations, add confirmation steps
- Keep audit trails for operations that modify shared state

---

## Own Your Dependencies

Every dependency is a trade-off: functionality gained vs. control lost. Pin versions. Audit updates. Understand what you're importing.

**Why:** Unpinned dependencies break builds silently. Unaudited dependencies introduce vulnerabilities. A dependency that saves 50 lines of code but adds 50MB of node_modules and an unpredictable API surface is not always a win.

**In practice:**
- Pin dependency versions for reproducible builds
- Prefer smaller, focused libraries over large frameworks (when the task is small)
- Understand what a dependency does before adding it
- Update dependencies deliberately, not implicitly
- When a dependency does one thing you need and twenty things you don't, consider writing the one thing yourself

---

## When Principles Conflict

These principles are defaults, not commandments — and sometimes they pull in opposite directions. When that happens, don't pick a winner blindly. Understand the tension, then make a deliberate call based on your project's current phase and constraints.

**Common tensions and resolution heuristics:**

**Simplicity vs. Reversibility** — Soft-delete adds complexity (filtered queries, broken unique constraints), but hard-delete is irreversible. *Resolution:* Default to simplicity. Use soft-delete only for data where accidental loss is genuinely costly and recovery is a real scenario — not as a universal habit. For everything else, rely on backups as your reversibility mechanism.

**Performance vs. Security** — Caching speeds things up but can serve stale auth data. Rate limiting protects the system but adds latency and complexity. *Resolution:* Security wins by default. Performance optimizations must not weaken security guarantees. Optimize within security constraints, not around them.

**Ship Then Polish vs. Accessibility** — Philosophy says ship early and polish later, but accessibility is hard to bolt on after the fact. *Resolution:* Core accessibility — semantic HTML, keyboard navigation, color contrast — is part of "working software," not "polish." It ships with the first pass. Enhancements like screen-reader optimization and ARIA live regions can come in the polish phase.

**Measure Twice vs. Incremental Delivery** — How much planning is enough before each increment? *Resolution:* Scale planning to risk. A trivial feature gets a mental model; a data model change or external API integration gets a written plan. If the cost of getting it wrong is an easy revert, plan lightly. If the cost is a database migration, plan thoroughly.

**Local-First vs. External Services** — Some capabilities (auth providers, payment processing, real-time collaboration) may be genuinely impractical to build locally. *Resolution:* Local-first is a default, not a constraint. When an external service is clearly the right tool, use it — but still abstract it behind a client layer so switching is cheap.

> **CyberPulse example:** Soft-delete was considered for articles but rejected — articles can always be re-fetched from their sources, so the complexity wasn't justified. Hard-delete with a retention window (configurable days) keeps the system simpler while still providing a recovery buffer.

---

## Technical Debt as a Tool

Not all technical debt is bad. Some debt is a deliberate investment — shipping faster now with a known shortcut, intending to pay it down before it compounds.

**Why:** Treating all shortcuts as failures leads to over-engineering early and shipping late. Treating all shortcuts as acceptable leads to a codebase that gradually becomes unmaintainable. The goal is intentional debt — taken deliberately, tracked visibly, and paid down before it blocks progress.

**In practice:**
- Before taking a shortcut, decide: is this "move fast" debt (acceptable) or "don't understand the problem" debt (dangerous)?
- Mark deliberate shortcuts with a comment explaining what the proper solution is and when it should be done
- Track debt at the project level, not just in code comments — a list of known shortcuts prevents them from being forgotten
- Pay down debt when you're working in the area anyway — it's cheapest when context is fresh
- If a shortcut has survived three feature additions without causing problems, it may not be debt at all — it may be the right solution

> **CyberPulse example:** Early versions used a single `process_articles()` function that handled fetching, classifying, and storing. This was deliberate — it validated the pipeline end-to-end. Once the pipeline was proven, it was decomposed into separate stages. The shortcut saved days of upfront design that would have targeted the wrong abstractions.

---

## Related Documents

- [02-lifecycle](02-lifecycle.md) — How these principles map to project phases
- [11-ai-collaboration](11-ai-collaboration.md) — How these principles apply to AI-assisted development
