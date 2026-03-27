# Project Seed

A universal reference for starting software projects. Extracted and generalized from real-world patterns, this is not a template repo or a checklist. It is a comprehensive knowledge base of principles, patterns, and rationale that applies to any software project.

## Audience

This seed is designed for a developer working with an AI coding assistant (e.g., Claude Code). When starting a new project, read the relevant sections and adapt them to your project's context. Each requirement explains not just *what* to do, but *why*, so you can make informed decisions about when to deviate.

## How to Use

1. **Starting a new project:** Read [01-philosophy](01-philosophy.md) and [02-lifecycle](02-lifecycle.md) first to set the mindset and choose your phases
2. **Designing architecture:** Read [03-architecture](03-architecture.md) and [04-data-modeling](04-data-modeling.md) for structural decisions
3. **Building UI:** Read [05-ui-ux](05-ui-ux.md) for component patterns and interaction design
4. **Hardening:** Read [06-security](06-security.md), [07-error-handling](07-error-handling.md), and [08-performance](08-performance.md)
5. **Quality gates:** Read [09-testing](09-testing.md) and [10-quality-ops](10-quality-ops.md)
6. **Working with AI:** Read [11-ai-collaboration](11-ai-collaboration.md) for structuring projects for effective AI-assisted development
7. **Domain-specific needs:** Check the [modules/](modules/) folder for optional patterns (LLM integration, agents, data ingestion)

## File Map

### Core Documents

| File | Covers |
|------|--------|
| [01-philosophy](01-philosophy.md) | Development mindset, guiding principles, principle conflicts, technical debt |
| [02-lifecycle](02-lifecycle.md) | Full SDLC: idea through maintenance, phase iteration, combining phases |
| [03-architecture](03-architecture.md) | Module separation, layered design, cross-layer concerns, API patterns, versioning, async, SPA |
| [04-data-modeling](04-data-modeling.md) | Schema design, relationships, concurrency control, migrations, data lifecycle, dedup, backups |
| [05-ui-ux](05-ui-ux.md) | Navigation, loading states, notifications, modals, search, tables, bulk actions, forms, a11y, i18n |
| [06-security](06-security.md) | Three-tier security, session management, password storage, TLS, secrets, validation |
| [07-error-handling](07-error-handling.md) | Error taxonomy, idempotency, cascading failures, circuit breakers, telemetry, graceful degradation |
| [08-performance](08-performance.md) | DB maintenance, caching with TTL guidance, pagination, memory management, write optimization, async I/O |
| [09-testing](09-testing.md) | Testing pyramid, isolation techniques, mocking strategy, flakiness debugging, E2E path selection |
| [10-quality-ops](10-quality-ops.md) | Logging, monitoring, incident response, rollback strategies, CI/CD secrets, dependencies, code style |
| [11-ai-collaboration](11-ai-collaboration.md) | CLAUDE.md, AI code review, hallucination prevention, iterative refinement, context management |

### Optional Modules

| Module | Covers |
|--------|--------|
| [modules/ai-llm-integration](modules/ai-llm-integration.md) | LLM client abstraction, prompt management, cost control, caching, token budgeting, streaming, concurrency |
| [modules/agents-orchestration](modules/agents-orchestration.md) | Agent loops, orchestrators, human-in-the-loop, debugging, communication, cost modeling |
| [modules/data-ingestion](modules/data-ingestion.md) | Source abstraction, scheduling, webhooks, file imports, delta fetching, dedup, normalization |

## Conventions

- Every requirement includes a **Why** explaining the rationale
- CyberPulse (the project this seed was extracted from) appears as illustrative examples in `> blockquotes`. These are concrete illustrations, not prescriptions
- Cross-references between documents use relative links: `→ See [architecture](03-architecture.md)`
- All patterns are **tech-agnostic**: they describe what to achieve, not which library to use
