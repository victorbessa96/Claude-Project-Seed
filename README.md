# Project Seed

A universal reference for starting software projects. Extracted and generalized from real-world patterns, this is not a template repo or a checklist — it is a comprehensive knowledge base of principles, patterns, and rationale that applies to any software project.

## Audience

This seed is designed for a developer working with an AI coding assistant (e.g., Claude Code). When starting a new project, read the relevant sections and adapt them to your project's context. Each requirement explains not just *what* to do, but *why* — so you can make informed decisions about when to deviate.

## How to Use

1. **Starting a new project** — Read [01-philosophy](01-philosophy.md) and [02-lifecycle](02-lifecycle.md) first to set the mindset and choose your phases
2. **Designing architecture** — Read [03-architecture](03-architecture.md) and [04-data-modeling](04-data-modeling.md) for structural decisions
3. **Building UI** — Read [05-ui-ux](05-ui-ux.md) for component patterns and interaction design
4. **Hardening** — Read [06-security](06-security.md), [07-error-handling](07-error-handling.md), and [08-performance](08-performance.md)
5. **Quality gates** — Read [09-testing](09-testing.md) and [10-quality-ops](10-quality-ops.md)
6. **Working with AI** — Read [11-ai-collaboration](11-ai-collaboration.md) for structuring projects for effective AI-assisted development
7. **Domain-specific needs** — Check the [modules/](modules/) folder for optional patterns (LLM integration, agents, data ingestion)

## File Map

### Core Documents

| File | Covers |
|------|--------|
| [01-philosophy](01-philosophy.md) | Development mindset, guiding principles, values |
| [02-lifecycle](02-lifecycle.md) | Full SDLC: idea, research, concept, requirements, architecture, build, test, deploy, maintain |
| [03-architecture](03-architecture.md) | Module separation, layered design, API patterns, async, background jobs, SPA patterns |
| [04-data-modeling](04-data-modeling.md) | Schema design, migrations, data lifecycle, dedup, backups |
| [05-ui-ux](05-ui-ux.md) | Navigation, loading states, notifications, tables, forms, a11y, i18n, theming |
| [06-security](06-security.md) | Three-tier security (local, LAN, internet), OWASP, secrets, validation |
| [07-error-handling](07-error-handling.md) | Error taxonomy, user-facing vs internal, retries, graceful degradation |
| [08-performance](08-performance.md) | Caching, pagination, lazy loading, rate limiting, async I/O |
| [09-testing](09-testing.md) | Testing pyramid, what to test, coverage philosophy |
| [10-quality-ops](10-quality-ops.md) | Logging, monitoring, backups, dependencies, CI/CD, code style |
| [11-ai-collaboration](11-ai-collaboration.md) | Writing CLAUDE.md, structuring docs for AI, prompting, AI strengths/limits |

### Optional Modules

| Module | Covers |
|--------|--------|
| [modules/ai-llm-integration](modules/ai-llm-integration.md) | LLM client abstraction, prompt management, cost control, retry patterns |
| [modules/agents-orchestration](modules/agents-orchestration.md) | Agent loops, orchestrators, human-in-the-loop, automated procedures |
| [modules/data-ingestion](modules/data-ingestion.md) | Source abstraction, scheduling, dedup, scraping, normalization |

## Conventions

- Every requirement includes a **Why** explaining the rationale
- CyberPulse (the project this seed was extracted from) appears as illustrative examples in `> blockquotes` — these are concrete illustrations, not prescriptions
- Cross-references between documents use relative links: `→ See [architecture](03-architecture.md)`
- All patterns are **tech-agnostic** — they describe what to achieve, not which library to use
