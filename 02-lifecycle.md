# Project Lifecycle

Every software project moves through a series of phases, from the initial spark of an idea to long-term maintenance. This document defines **phase types** — not a rigid sequence, but a menu of stages that projects select and combine based on their scope and complexity.

---

## Phase Types

### 1. Ideation

Turn a vague feeling into a concrete problem statement.

**Outputs:**
- **Problem statement** — What problem exists? Who experiences it? What happens if it's not solved?
- **Vision** — What does success look like? What does the world look like after this project ships?
- **Stakeholders** — Who benefits? Who decides? Who maintains?
- **Scope signal** — Is this a weekend project, a month-long effort, or a multi-quarter initiative?

**Why this phase matters:** Projects that skip ideation often solve the wrong problem or build more than needed. A clear problem statement is the single most valuable artifact — everything else derives from it.

**When to skip:** Never entirely. Even a one-line problem statement is better than none. For small tools, this can be a single paragraph.

---

### 2. Research

Understand the landscape before building.

**Outputs:**
- **Existing solutions** — What already exists? What do they get right/wrong?
- **Technical feasibility** — Can this be built with available tools and skills? What are the unknowns?
- **Differentiators** — Why build this instead of using what exists?
- **Constraints** — Budget, time, team size, compliance requirements, platform limitations
- **User/market context** — Who will use this and in what environment?

**Why this phase matters:** Building what already exists wastes effort. Building without understanding constraints leads to rework. The best time to discover a blocker is before you've written code.

**When to skip:** For personal tools with no alternatives, or when you already have deep domain expertise. Even then, a quick scan for prior art is valuable.

---

### 3. Concept

Define what the project IS and what it ISN'T.

**Outputs:**
- **Core features** — The minimum set of capabilities that make this project useful
- **User stories** — "As a [role], I want to [action] so that [benefit]"
- **Scope boundaries** — Explicit list of what is OUT of scope (often more useful than what's in)
- **Key interactions** — How users will accomplish primary tasks (rough flows, not pixel-perfect designs)
- **Success criteria** — How will you know this project is done and working?

**Why this phase matters:** Scope boundaries prevent feature creep. User stories keep focus on actual needs rather than interesting-but-unnecessary features. Saying "no" early is cheaper than saying "no" after building.

**When to skip:** For very small projects (scripts, utilities), this can merge with Ideation into a single planning paragraph.

> **CyberPulse example:** Concept phase defined 6 core features (fetch, classify, condense, generate, export, manage settings) and explicitly excluded user authentication, multi-tenancy, and internet deployment.

---

### 4. Requirements

Translate the concept into specific, verifiable requirements.

**Outputs:**
- **Functional requirements** — What the system must do (actions, inputs, outputs, data flows)
- **Non-functional requirements** — Performance, security, accessibility, reliability expectations
- **Constraints** — Technology mandates, deployment environment, compliance needs
- **Assumptions** — Things you're taking for granted (document them so they can be challenged)
- **Acceptance criteria** — How each requirement will be verified as "done"

**Why this phase matters:** Ambiguous requirements lead to ambiguous implementations. Writing requirements forces you to confront edge cases, dependencies, and contradictions before they become bugs.

**When to skip:** For tiny projects, requirements can be implicit. But the moment two people are involved (including you and an AI assistant), explicit requirements prevent misunderstandings.

---

### 5. Architecture & Design

Make the structural decisions that are expensive to change later.

**Outputs:**
- **Technology choices** — Language, framework, database, hosting — each with rationale
- **Component design** — Modules, their responsibilities, and how they communicate
- **Data model** — Entities, relationships, storage strategy (→ see [04-data-modeling](04-data-modeling.md))
- **API surface** — Endpoints, request/response shapes, error contract
- **Security posture** — Threat tier, auth approach, secrets handling (→ see [06-security](06-security.md))
- **Visual design** — If there's a UI: design tokens, component specs, layout system (→ see [05-ui-ux](05-ui-ux.md))

**Why this phase matters:** Architecture decisions have the highest blast radius. Changing a database engine after six months of development is orders of magnitude harder than choosing well up front. Write down the rationale so future-you understands why you chose what you chose.

**When to skip:** Never for projects with multiple components. For single-file scripts, architecture is implicit.

> **CyberPulse example:** Architecture phase produced four documents: project plan, architecture spec, phase breakdown, and visual design spec. Key decisions (SQLite, no-build frontend, OpenRouter for LLM) were each documented with rationale.

---

### 6. Foundation

Set up the project skeleton: infrastructure, configuration, and the first working endpoint or screen.

**Outputs:**
- **Project structure** — Folders, entry points, configuration files
- **Database** — Schema creation, connection management, seed data
- **Core infrastructure** — Logging, error handling, settings management
- **First working route/screen** — Proof that the stack works end-to-end
- **Development environment** — How to install, run, and develop locally

**Why this phase matters:** A solid foundation makes everything faster. A broken foundation makes everything painful. Getting the basics right (logging, config, database access patterns) before writing features prevents retrofitting later.

**When to skip:** Never. Even a single-file project needs a working "hello world" before features.

---

### 7. Core Features

Build the primary functionality — the reason the project exists.

**Outputs:**
- **Working features** — Each core capability implemented and manually testable
- **Data flows** — End-to-end paths from input to stored data to output
- **API endpoints** — Complete request/response cycle for each resource
- **Basic UI** — Functional (not polished) interface for each feature

**Why this phase matters:** This is where the project becomes real. Focus on the happy path first. Edge cases, error handling, and polish come later. The goal is a working system that demonstrates value.

**In practice:**
- Build features in dependency order (data model → ingestion → processing → display)
- Test manually as you go — automated tests come in a later phase
- Hardcoded values are acceptable; extract configuration later
- Ugly-but-working is fine; pretty-but-broken is not

---

### 8. Polish & UX

Make the working system pleasant to use.

**Outputs:**
- **Loading states** — Skeleton placeholders, spinners, progress indicators
- **Transitions** — View changes, element animations, micro-interactions
- **Error messages** — User-friendly, actionable, localized
- **Responsive design** — Adapts to different screen sizes
- **Accessibility** — ARIA attributes, keyboard navigation, screen reader support
- **Empty states** — What the user sees when there's no data yet
- **First-run experience** — Guided setup for new users

**Why this phase matters:** A working system that's frustrating to use won't get used. Polish is what turns a tool into a product. But it must come after core features — polishing a broken feature is wasted effort.

**When to skip:** For internal tools with a known audience, minimal polish may be acceptable. But loading states and error messages are never optional.

> **CyberPulse example:** Phase 6 (post-launch) added skeleton loading, toast notifications with progress bars, Chart.js dashboards, responsive sidebar, and view transitions — all after core functionality was stable.

---

### 9. Hardening

Make the system resilient, secure, and operationally sound.

**Outputs:**
- **Security review** — Apply the appropriate tier from [06-security](06-security.md)
- **Error handling audit** — Every external call has a timeout; every catch has a log
- **Rate limiting** — Protect expensive or abusable endpoints
- **Input validation** — Validate at system boundaries
- **Logging** — Structured, rotated, with appropriate levels
- **Backups** — Automated, tested, with retention policy
- **Configuration** — All magic numbers extracted to settings

**Why this phase matters:** Software that works in demo conditions fails in production. Hardening is what makes the difference between "it works on my machine" and "it works reliably."

**When to skip:** For throwaway prototypes only. Anything that persists data or handles user input needs hardening.

---

### 10. Testing & QA

Formalize testing into a suite that runs automatically and catches regressions.

**Important clarification:** This phase is about building the *automated test suite* and running a formal QA pass — it is not the first time the software gets tested. Manual testing and basic unit tests should happen continuously during Core Features (Phase 7) and Hardening (Phase 9). By the time you reach this phase, the software already works; the goal here is to *prove* it works and ensure it keeps working as it evolves.

**Outputs:**
- **Test suite** — Unit, integration, and/or e2e tests for critical paths (→ see [09-testing](09-testing.md))
- **Manual QA** — Systematic walkthrough of all features, including edge cases
- **Performance check** — Verify response times, memory usage, database query performance
- **Cross-environment testing** — Different browsers, screen sizes, OS versions as applicable

**Why this phase matters:** Informal testing during development catches the obvious bugs. A formal test suite catches the subtle ones — regressions, edge cases, interaction bugs — and gives you confidence to refactor, extend, and deploy without fear.

**When to skip:** Tests for throwaway scripts are optional. But if the code will live longer than a week or be touched by someone else, testing pays for itself.

---

### 11. Deployment

Make the software available to its intended users.

**Outputs:**
- **Packaging** — How to build and distribute (Docker, pip install, binary, static files)
- **Hosting** — Where it runs (local, LAN, cloud, edge) and how to set it up
- **CI/CD** — Automated build, test, deploy pipeline (→ see [10-quality-ops](10-quality-ops.md))
- **Monitoring** — Health checks, uptime tracking, error alerting
- **Documentation** — README, setup guide, troubleshooting

**Why this phase matters:** Software that's hard to deploy doesn't get deployed. Software without monitoring fails silently.

**When to skip:** For personal local tools, "deployment" may just be `python main.py`. But even then, a README explaining how to start matters.

---

### 12. Maintenance

Keep the software working, secure, and relevant over time.

**Outputs:**
- **Dependency updates** — Regular, deliberate updates with testing
- **Bug fixes** — Triage, fix, regression test
- **Feature evolution** — New capabilities driven by actual usage, not speculation
- **Data lifecycle** — Pruning, archival, migration as data grows
- **Documentation updates** — Keep docs in sync with reality

**Why this phase matters:** Software without maintenance decays. Dependencies get vulnerabilities. Users find bugs. Requirements change. Maintenance is not "extra work after the project is done" — it's an ongoing phase of the project's life.

**When to skip:** Only for intentionally throwaway projects.

---

## Phase Iteration and Backtracking

Phases are presented in order, but real projects aren't linear. Sometimes Foundation reveals that an architecture decision was wrong. Sometimes Core Features expose a missing requirement. This is normal — the question is how to backtrack without losing progress.

**When to loop back:**
- **Foundation reveals a bad technology choice** — e.g., the database engine can't handle a required query pattern. Loop back to Architecture. The fix is targeted: revise the specific decision, not the entire architecture.
- **Core Features expose a missing requirement** — e.g., users need bulk operations that weren't in the spec. Add the requirement, assess its impact on architecture and data model, then continue building.
- **Hardening reveals a design flaw** — e.g., the error handling pattern doesn't work for async pipelines. Revisit the architectural decision, patch it, and continue hardening.

**How to backtrack well:**
- Fix the specific decision that was wrong — don't restart the entire phase
- Document what changed and why, so the same mistake isn't repeated
- If the backtrack is large (rethinking the data model mid-build), pause and re-plan before coding
- Treat backtracking as information, not failure — early phases are hypotheses; later phases are validation

**When to stop and re-evaluate the project:**
- If you've backtracked to the same phase three times, the problem may be in your understanding of the requirements, not in the implementation
- If a backtrack requires changing more than half of the existing code, consider whether the project concept needs revision

> **CyberPulse example:** During Core Features, it became clear that the original single-pass article processing couldn't handle the condenser step (which needed classified output). This triggered a backtrack to Architecture to redesign the pipeline into explicit stages (fetch → classify → condense → generate). The fix was targeted and improved the design without restarting.

---

## Phase Advancement Criteria

How do you know when a phase is "done"?

| Phase | Done when... |
|-------|-------------|
| Ideation | Problem statement is written and feels accurate |
| Research | You can articulate why this project should exist and what's unique about it |
| Concept | Scope boundaries are defined; someone else could read the concept and understand what to build |
| Requirements | Each requirement has acceptance criteria; no ambiguity remains |
| Architecture | Tech stack chosen with rationale; component boundaries clear; data model designed |
| Foundation | Project runs; database connects; first endpoint returns data; logging works |
| Core Features | All primary features work end-to-end on the happy path |
| Polish & UX | Loading states, error messages, and responsive design are in place |
| Hardening | Security tier applied, errors handled, backups configured, rate limits set |
| Testing & QA | Critical paths have tests; manual QA passed; no known serious bugs |
| Deployment | Software is accessible to intended users; monitoring is active |
| Maintenance | (Ongoing) No unpatched vulnerabilities; docs are current; data lifecycle is managed |

---

## Combining Phases

Not every project needs all twelve phases. Here's guidance on combining:

**Weekend project / personal script:**
- Ideation + Concept (merged) → Foundation → Core Features → Done
- Skip: Research, Requirements, Polish, Testing, Deployment, Maintenance

**Internal tool / small app:**
- Ideation → Concept → Architecture → Foundation → Core Features → Hardening → Done
- Lightweight: Research, Requirements (informal), Polish (minimal), Testing (critical paths only)

**Product / team project:**
- All phases in order, with iteration between Core Features and Polish
- Full: Requirements, Architecture, Testing, Deployment, Maintenance

**AI-assisted project:**
- Ideation → Research → Concept → Requirements → Architecture (documented) → Foundation → Core Features → Polish → Hardening → Testing
- Key: Architecture and Requirements phases produce documents that the AI reads (→ see [11-ai-collaboration](11-ai-collaboration.md))

> **CyberPulse example:** Used 6 phases: Foundation → Ingestion (core) → LLM Pipeline (core) → Frontend (core + polish) → Export & Polish → Post-Launch Enhancements (hardening + polish). Research and concept were done informally. Architecture was fully documented. Testing was manual.

---

## Documentation Artifacts by Phase

| Phase | Artifact | Purpose |
|-------|----------|---------|
| Ideation | Problem statement | Align on what to solve |
| Research | Landscape analysis | Justify building vs. buying |
| Concept | Scope document (IN/OUT) | Prevent feature creep |
| Requirements | Requirements spec | Verifiable acceptance criteria |
| Architecture | Architecture doc + Design spec | Guide implementation; explain decisions |
| Foundation | README + CLAUDE.md | Enable development and AI assistance |
| Core–Hardening | Inline code docs + commit messages | Explain non-obvious decisions |
| Deployment | Setup guide + troubleshooting | Enable others to run the software |
| Maintenance | Changelog + decision log | Track evolution and rationale |

---

## Related Documents

- [01-philosophy](01-philosophy.md) — The principles that guide every phase
- [03-architecture](03-architecture.md) — Deep dive into Phase 5 decisions
- [11-ai-collaboration](11-ai-collaboration.md) — How to structure phase outputs for AI-assisted development
