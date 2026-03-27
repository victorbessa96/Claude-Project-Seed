# AI-Assisted Development

How to structure projects, documentation, and workflows for effective collaboration with AI coding assistants. These patterns are informed by real experience building software with Claude Code, but the principles apply to any AI-assisted workflow.

---

## Project Structure for AI

AI assistants navigate codebases by reading file names, directory structures, and documentation. The clearer your project is organized, the better the AI can help.

### File Organization

- **One responsibility per file** — the AI can read and modify one focused file more accurately than a sprawling multi-purpose file
- **Descriptive names** — `scraper.py` is immediately clear; `utils2.py` is not
- **Consistent conventions** — if all routes are in `routers/`, the AI knows where to look for route code
- **Shallow nesting** — 2-3 levels deep is navigable; 6 levels deep requires extensive exploration

### Module Boundaries

- Keep modules small enough that the AI can hold the entire module in context
- Make dependencies between modules explicit (imports, not implicit global state)
- If a change requires modifying 8 files, the task may be too large for a single prompt — break it down

**Why:** AI assistants have context windows. The less code they need to read to understand the task, the better their output. Clean structure is a force multiplier.

---

## Writing Effective CLAUDE.md

CLAUDE.md is the AI's first read when entering a project. It's the most high-leverage document you can write.

### What to Include

- **Project overview** — one paragraph explaining what the project does and why it exists
- **Tech stack** — languages, frameworks, database, key libraries
- **Project structure** — directory tree with one-line descriptions of key directories
- **How to run** — exact commands to install dependencies and start the application
- **Development conventions** — patterns to follow, anti-patterns to avoid
- **Key design decisions** — the numbered list of "why we chose X" decisions
- **What NOT to do** — explicit anti-patterns save the AI from common mistakes

### What to Omit

- Implementation details that are obvious from reading the code
- Git history or changelog information (`git log` is authoritative)
- Debugging solutions (the fix is in the code)
- Ephemeral state (current bugs, in-progress work)

### Writing Style for CLAUDE.md

- **Imperative and direct** — "All API calls go through `api.js`" not "We decided that API calls should probably go through api.js"
- **Specific over general** — "Never hardcode English strings in view JS files — always use `t()` from i18n.js" not "Follow i18n best practices"
- **Anti-patterns with reason** — "Never use plain FileHandler — use RotatingFileHandler (prevents unbounded log growth)" not just "Don't use FileHandler"
- **Maintain it** — a stale CLAUDE.md is worse than none (the AI trusts it and produces wrong code)

> **CyberPulse example:** CLAUDE.md includes 25 numbered design decisions, explicit conventions for both backend and frontend, and anti-patterns like "never hardcode localhost" and "never log API keys." This level of specificity produces consistent AI output.

---

## Documentation That Helps AI

Beyond CLAUDE.md, certain documentation artifacts dramatically improve AI collaboration.

### Architecture Documents

An architecture doc answers the AI's most common question: "how does this system fit together?"

Include:
- Component diagram (even ASCII) showing how modules communicate
- Data flow from input to output
- Database schema with relationships
- API endpoint summary with request/response shapes

**Why:** Without architecture docs, the AI must infer structure by reading code — which is slow, error-prone, and consumes context window.

### Design Specs

For UI work, a design spec prevents the AI from guessing aesthetic decisions.

Include:
- Color palette and design tokens
- Typography scale
- Component specifications (buttons, cards, tables, modals)
- Layout rules and breakpoints
- Animation and transition specs

**Why:** AI assistants can generate functional UI, but aesthetic judgment is weak. A design spec gives concrete targets instead of vague "make it look good."

### Decision Records

When the AI encounters a choice (which library? which pattern? which approach?), a decision record answers instantly.

Format: `Decision: [what was decided]. Rationale: [why]. Alternatives considered: [what was rejected and why].`

**Why:** Without decision records, the AI may propose alternatives you already rejected — wasting time re-explaining why you chose the current approach.

---

## Planning with AI

### When to Use Plan Mode

- Before any non-trivial implementation (> 1 file, > 30 minutes of work)
- When the approach isn't obvious
- When multiple valid approaches exist and you want to choose
- When you want to review the strategy before committing to it

### Effective Planning Prompts

- **State the goal clearly** — "Add PDF export for generated outputs" not "make exports work"
- **Provide context** — "The generate view already has a tabbed interface with Preview and Raw tabs. Add an Export tab."
- **Specify constraints** — "Don't add new dependencies. Use the existing WeasyPrint installation."
- **Ask for alternatives** — "Show me two approaches and their trade-offs"

### Scope Control

AI assistants tend to overdeliver. They'll refactor surrounding code, add error handling you didn't ask for, and "improve" things that were fine.

**In practice:**
- Be explicit about scope: "Only modify `generate.js` and `export.py`"
- Review plans before approving: does the plan match what you asked for?
- If the AI suggests improvements, decide whether they're in scope now or for later
- "Don't add features I didn't ask for" is a valid and useful instruction

---

## Code Review with AI

### What to Ask For

- **Security review** — "Check this endpoint for injection vulnerabilities"
- **Bug hunting** — "This function sometimes returns wrong results when X. Find the bug."
- **Consistency check** — "Does this new module follow the patterns in the rest of the codebase?"
- **Simplification** — "Is there a simpler way to do what this function does?"

### How to Frame Reviews

- Point to specific files or functions
- Describe the expected behavior and the actual behavior
- Provide context about what changed and why
- Ask focused questions rather than "review everything"

**Why:** Broad "review my code" prompts produce broad, generic feedback. Focused questions produce actionable findings.

### Reviewing AI-Generated Code

AI output looks correct more often than it is correct. Review it with healthy skepticism.

**What to check:**
- **Does it do what was asked?** — AI tends to add unrequested features, refactor surrounding code, or "improve" things that were fine. Check scope first.
- **Does it handle the unhappy path?** — AI is excellent at happy paths but can be optimistic about error handling. Verify that errors from external calls, missing data, and invalid states are handled.
- **Does it introduce new dependencies?** — AI may import libraries that aren't in the project or use APIs that don't exist in your version.
- **Does it match project conventions?** — AI follows the patterns in CLAUDE.md and existing code, but only if those patterns are clear. Check naming, file organization, and error handling patterns.
- **Are the magic values reasonable?** — AI often uses plausible-looking but arbitrary numbers (timeouts, limits, thresholds). Verify these match your requirements.

### Hallucination Prevention

AI assistants can confidently reference APIs, functions, or patterns that don't exist.

**Common hallucination patterns:**
- Calling methods that don't exist on the libraries you're using
- Referencing configuration options that aren't real
- Using syntax from one language/version in another
- Citing documentation that doesn't exist or says something different

**Mitigation:**
- For unfamiliar APIs: ask the AI to show you where in the documentation the feature is described, then verify
- For code changes: run the code. A function that doesn't exist will fail immediately.
- For configuration: check the actual documentation or source code of the library
- If the AI is confidently wrong about something, point it out directly — it will correct course

---

## Prompt Patterns for Development

### Describing Bugs

```
Bug: [what's wrong]
Expected: [what should happen]
Actual: [what happens instead]
Steps to reproduce: [how to trigger it]
Relevant files: [where to look]
```

### Describing Features

```
Goal: [what the feature does]
Context: [where it fits in the app]
Constraints: [what to avoid, what to reuse]
Acceptance criteria: [how to know it's done]
```

### Describing Refactors

```
Current state: [how it works now]
Problem: [why it needs to change]
Desired state: [how it should work after]
Scope: [what files/modules are in scope]
Constraint: [behavior must not change]
```

**Why:** Structured prompts eliminate ambiguity. The AI doesn't have to guess what you mean, so it produces more accurate results on the first try.

---

## Iterative Refinement

AI rarely produces perfect output on the first attempt. The skill is in giving feedback that converges quickly.

### Effective Feedback Patterns

- **Be specific about what's wrong:** "The error handling catches too broadly — it should catch `TimeoutError` specifically, not all `Exception`s" is better than "fix the error handling"
- **Show, don't tell:** If the AI's approach is wrong, show an example of what you want: "Use this pattern instead: `if not result: return default`"
- **One thing at a time:** Correcting 5 problems in one message leads to some being addressed and others ignored. Fix the most important issue first.
- **Affirm what's right:** If 80% of the output is correct, say so. "The data model is correct. The API routes are correct. But the error handling needs to change:" — this prevents the AI from rethinking the entire solution.

### When to Start Over

Sometimes iterative refinement converges slowly or not at all. Start a new approach when:
- The AI is going in circles (fixing one thing, breaking another)
- The conversation has accumulated so much context that responses are degrading
- The fundamental approach is wrong, not just the details

---

## Context Management for Large Codebases

In large projects, the AI can't read everything at once. Effective scoping determines the quality of output.

### Scoping Work for AI

- **Point to specific files:** "Modify `routers/articles.py` and `services/articles.py`" is better than "fix the articles feature"
- **Provide the relevant context:** If the change depends on a utility function in another file, mention it: "The `format_date()` helper in `utils.py` already handles timezone conversion — use it."
- **Break large tasks into steps:** Instead of "implement the export feature," do "Step 1: Add the export endpoint in `routers/export.py`. Step 2: Add the PDF renderer in `services/export_pdf.py`."
- **Use plan mode for complex tasks:** Let the AI explore the codebase and propose a plan before it starts writing code

### Keeping CLAUDE.md Useful at Scale

As the project grows, CLAUDE.md must grow with it — but not unboundedly.

- **Layer your documentation:** CLAUDE.md for project-wide conventions; subdirectory-level docs for subsystem-specific patterns (e.g., `frontend/CLAUDE.md` for frontend conventions)
- **Keep it current:** Review CLAUDE.md during each major feature addition. Remove patterns that no longer apply. Add new conventions as they emerge.
- **Prioritize the non-obvious:** Don't document what the code makes clear. Document the "why" behind surprising choices and the constraints that aren't visible from the code.

---

## AI Strengths and Limitations

### What AI Does Well

- **Boilerplate and patterns** — generating CRUD endpoints, test scaffolds, data models
- **Refactoring** — renaming, extracting functions, restructuring code that follows clear patterns
- **Translating** — converting between languages, frameworks, or paradigms
- **Documentation** — generating docstrings, README sections, API docs from code
- **Exploring** — searching codebases, finding usages, understanding unfamiliar code
- **Applying conventions** — following established patterns consistently across new code

### What AI Does Poorly

- **Novel architecture** — designing entirely new systems without guidance (tends to over-engineer)
- **Aesthetic judgment** — choosing colors, layouts, animations (needs a design spec)
- **Business context** — understanding organizational politics, user preferences, market dynamics
- **Knowing when to stop** — tends to add features, refactor code, and "improve" things beyond scope
- **Debugging non-reproducible issues** — intermittent failures, race conditions, environment-specific bugs

### Working with Limitations

- Provide design specs for UI work (don't rely on AI aesthetic judgment)
- Set explicit scope boundaries for implementation tasks
- Review AI plans before execution — catch scope creep early
- For novel architecture, discuss approaches interactively rather than delegating the full design
- For debugging, provide reproduction steps, logs, and relevant code — don't just describe the symptom

---

## Memory and Continuity

AI coding assistants have various persistence mechanisms. Use the right one for the right purpose.

| Mechanism | Persists across conversations? | Best for |
|-----------|-------------------------------|----------|
| **Conversation context** | No | Current task state, decisions in progress |
| **Plan files** | No (ephemeral) | Implementation approach for current task |
| **Task lists** | No (ephemeral) | Progress tracking for current session |
| **AI memory files** | Yes | User preferences, feedback, project context |
| **CLAUDE.md** | Yes (in repo) | Project conventions and decisions |
| **Code + git** | Yes (in repo) | Implementation, commit messages, history |
| **Architecture docs** | Yes (in repo) | System design, data flow, API contracts |

### What to Save in AI Memory

- User role, expertise, and preferences
- Feedback on AI behavior (what worked, what didn't)
- Ongoing project context not captured in code or docs
- External resource locations (issue trackers, dashboards, docs)

### What NOT to Save in AI Memory

- Code patterns (read the code instead)
- Git history (use `git log`)
- Debugging solutions (the fix is in the commit)
- Anything already in CLAUDE.md

---

## Related Documents

- [01-philosophy](01-philosophy.md) — Principles that guide all development, including AI-assisted
- [02-lifecycle](02-lifecycle.md) — How documentation artifacts feed AI collaboration at each phase
- [10-quality-ops](10-quality-ops.md) — Documentation practices
