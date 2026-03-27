# Testing

A pragmatic testing strategy focused on confidence, not coverage percentages. The goal is to test what matters — system boundaries, critical paths, and regression-prone areas — without creating a test suite that's harder to maintain than the code it tests.

---

## The Testing Pyramid

```
        ╱  E2E  ╲          Few — slow, brittle, high confidence
       ╱─────────╲
      ╱Integration╲        Some — moderate speed, tests boundaries
     ╱─────────────╲
    ╱   Unit Tests   ╲     Many — fast, isolated, tests logic
   ╱───────────────────╲
```

**Why this shape:** Unit tests are fast and cheap — run thousands in seconds. Integration tests are slower but catch boundary bugs. E2E tests are slow and brittle but catch what the others miss. Inverting the pyramid (mostly E2E, few unit tests) creates a slow, flaky test suite.

---

## Unit Tests

Test isolated logic — functions that take input and return output without side effects.

### What to Unit Test

- **Pure functions** — data transformations, calculations, formatters, validators
- **Business rules** — scoring logic, classification rules, eligibility checks
- **Parsing/serialization** — JSON parsing, date formatting, content normalization
- **Edge cases** — empty input, boundary values, null/undefined handling

### What NOT to Unit Test

- **Framework internals** — don't test that the router routes or the ORM queries
- **Trivial code** — getters, setters, pass-through functions
- **Implementation details** — don't test private methods; test the public interface
- **Configuration** — don't test that constants have the right values

### Unit Test Principles

- **Fast** — each test should run in milliseconds
- **Isolated** — no database, no network, no filesystem
- **Deterministic** — same input always produces same output (no random, no current time)
- **Readable** — test name describes the scenario; test body shows setup → action → assertion

**Why:** Unit tests are the foundation. They give fast feedback during development and catch regressions instantly. But testing trivial code or framework behavior creates maintenance burden without adding confidence.

> **CyberPulse example:** Good unit test candidates: content hash computation, relevance score bucketing (1-4 → "low", 5-7 → "important", 8-10 → "critical"), URL normalization for dedup, i18n `t()` function with interpolation.

---

## Integration Tests

Test the boundaries between components — where your code meets external systems.

### What to Integration Test

- **API endpoints** — send HTTP requests, verify responses (status, body, headers)
- **Database operations** — insert, query, update, delete with a real (test) database
- **External service clients** — verify request format and response handling (with mocked/stubbed services)
- **Pipeline stages** — verify that data flows correctly through multi-step processes

### Integration Test Principles

- **Use real infrastructure where cheap** — real database (in-memory or temporary), real HTTP framework
- **Mock expensive/slow externals** — LLM APIs, third-party services, email providers
- **Test the contract** — verify request/response shapes, not implementation details
- **Clean up after each test** — reset database state, clear caches

**Why:** Integration tests catch the bugs that unit tests can't — mismatched types between layers, incorrect SQL, wrong HTTP status codes, broken serialization. They're slower than unit tests but worth the cost for critical paths.

### Test Database Strategy

- Use an in-memory or temporary database for tests
- Apply the same schema migration as production
- Seed minimal test data per test (not a shared global fixture)
- Roll back or recreate after each test for isolation

---

## End-to-End (E2E) Tests

Test complete user workflows from the frontend through the backend.

### What to E2E Test

- **Critical happy paths** — the primary workflows users depend on
- **First-run experience** — does the app guide new users correctly?
- **Data round-trips** — create → display → edit → verify persistence

### What NOT to E2E Test

- **Every permutation** — E2E tests are too slow for exhaustive coverage
- **Error handling details** — use unit/integration tests for error scenarios
- **Visual layout** — E2E tests shouldn't assert pixel positions or CSS values

### E2E Test Principles

- **Few and focused** — 5-20 tests for most applications
- **Resilient selectors** — use `data-testid` attributes, not CSS classes or text content
- **Timeout-aware** — async operations need explicit waits, not sleep
- **Independent** — each test should work in any order, on a clean state

**Why:** E2E tests are the most expensive to write and maintain. They break easily when UI changes. But they catch integration bugs that nothing else can — "the button calls the API, the API writes to the DB, and the result shows on the next page."

---

## What NOT to Test

Knowing what to skip is as important as knowing what to test.

- **Framework behavior** — the framework's test suite covers this
- **Third-party libraries** — test your usage of them, not their internals
- **Trivial getters/setters** — if it's just `return this.value`, skip it
- **Constants and configuration** — the app will fail obviously if these are wrong
- **UI layout and styling** — unless pixel-perfect rendering is a business requirement
- **Implementation internals** — if you refactor and the tests break but the behavior doesn't change, the tests were testing the wrong thing

---

## Test Data

### Factories and Fixtures

- Create test data programmatically (factory functions), not with shared JSON fixtures
- Each test creates the minimal data it needs
- Use deterministic values (not random, not current timestamp) for reproducibility

### Cleanup

- Reset state between tests (rollback transaction, recreate database, clear caches)
- Don't rely on test execution order
- Don't let one test's data pollute another

**Why:** Flaky tests are worse than no tests. They erode trust in the test suite and waste time investigating false failures. Deterministic data and clean state prevent flakiness.

---

## Test Isolation Techniques

### Database Isolation

Tests that touch the database need isolation so they don't interfere with each other.

**Approaches:**
- **Transaction rollback** — Wrap each test in a transaction, roll back after the test. Fast; no cleanup code needed. Best for most cases.
- **Fresh database per test** — Create and destroy a temporary database for each test (or test class). Slower but guarantees complete isolation. Best for tests that modify schema or test migrations.
- **Shared database with cleanup** — Use a single test database, but truncate/reseed tables between tests. Middle ground; watch for forgotten cleanup.

**In practice:**
- Default to transaction rollback — it's the cheapest isolation
- Use fresh databases for integration tests that test migrations or schema changes
- Never share database state between tests — even if the order happens to work today, it won't tomorrow

### Mocking Strategy

Mocking replaces real dependencies with controlled substitutes. Use it deliberately, not by default.

**When to mock:**
- **External services** — LLM APIs, third-party APIs, email providers (slow, costly, flaky)
- **Time-dependent code** — freeze `now()` to make tests deterministic
- **Non-deterministic code** — random number generators, UUIDs

**When NOT to mock:**
- **Your own code** — if you mock your data layer to test your business logic, you're testing nothing useful. Use a real (test) database instead.
- **Simple dependencies** — if the real thing is fast and deterministic, use it
- **The thing under test** — never mock the function you're testing (this sounds obvious but it happens)

**Mock vs. Stub vs. Spy:**
- **Mock** — replaces the dependency entirely; you control what it returns and assert it was called correctly
- **Stub** — provides canned responses but doesn't assert call patterns; simpler than mocks
- **Spy** — wraps the real implementation and records calls; the real code still runs

**Default to stubs** for simplicity. Use mocks when you need to verify interaction patterns. Use spies when you want the real behavior but need to inspect calls.

---

## Debugging Flaky Tests

Flaky tests — tests that pass sometimes and fail sometimes — erode trust in the test suite faster than missing tests.

### Common Causes

- **Shared state** — tests depend on state from a previous test; reorder breaks them
- **Time sensitivity** — tests that use `now()` or expect operations to complete within a specific time
- **Async race conditions** — asserting before an async operation completes
- **External dependencies** — tests that hit real networks, real APIs, or real file systems
- **Non-deterministic data** — using random values or auto-incrementing IDs in assertions

### Debugging Approach

1. **Reproduce locally** — run the failing test 50 times in a loop. If it doesn't fail, the flake is environment-specific.
2. **Run in isolation** — does the test pass alone but fail in suite? Shared state issue.
3. **Run in reverse order** — if the test depends on execution order, this surfaces it immediately.
4. **Add logging** — log the actual values that are compared in assertions. Flaky failures with no context are impossible to debug.
5. **Check for time dependence** — does the test fail more around midnight, or near time boundaries? Freeze time in tests.

### Prevention

- Use factories with deterministic values, not auto-generated IDs
- For async operations, use explicit waits/assertions (not `sleep`)
- Isolate every test from every other test — no shared state, no execution order dependency
- Quarantine known flaky tests — don't let them block the pipeline while you investigate, but don't ignore them

---

## Selecting E2E Test Paths

With only 5-20 E2E tests, choosing the right paths is critical.

**Selection criteria:**
- **Revenue/value paths** — the workflows that make the application valuable. If these break, the app is useless.
- **Multi-system paths** — workflows that cross multiple layers (UI → API → DB → external service). These catch integration bugs.
- **High-traffic paths** — the most-used workflows. A bug here affects the most users.
- **Previous failure points** — workflows that have had production bugs. They've proven they're fragile.

**What to deprioritize in E2E:**
- Error handling (use integration tests)
- Edge cases (use unit tests)
- Settings/configuration pages (low risk, easy to test manually)

> **CyberPulse example:** Key E2E paths would be: (1) fetch articles from sources → verify they appear in the list, (2) generate output from articles → verify the output renders, (3) export generated output → verify the file downloads, (4) change settings → verify they take effect on next operation.

---

## When to Write Tests

### Before Shipping

Write tests for the critical path before deploying a feature. You don't need 100% coverage — you need confidence that the primary workflows work.

### After Finding Bugs (Regression Tests)

When a bug is found and fixed, write a test that would have caught it. This prevents the same bug from recurring.

**Why:** Bugs cluster — the code that had one bug is likely to have another. A regression test protects that specific area permanently.

### Before Refactoring

Write tests for the current behavior before changing the implementation. This gives you a safety net to refactor confidently.

**Why:** Refactoring without tests is hoping nothing breaks. Refactoring with tests is verifying nothing breaks.

### During Code Review

If a reviewer asks "how do we know this works?", the answer should be a test, not "I tested it manually."

---

## Coverage Philosophy

### Aim for Critical Paths, Not Percentages

- 100% coverage of trivial code is waste
- 0% coverage of critical code is risk
- Focus coverage on: revenue-affecting paths, data integrity, security boundaries
- Use coverage tools to find untested areas, not as a quality gate

### Coverage Anti-Patterns

- **Mandated percentage targets** — teams write trivial tests to hit numbers
- **Testing to coverage, not to confidence** — covering lines without asserting meaningful behavior
- **Ignoring coverage entirely** — flying blind about which code is tested

**In practice:**
- Run coverage reports periodically to identify blind spots
- Investigate uncovered critical code — should it have tests?
- Don't block PRs on coverage percentage alone
- Track coverage trends (declining coverage on critical modules = warning sign)

---

## Test Organization

### File Structure

```
tests/
├── unit/
│   ├── test_validators.py
│   ├── test_formatters.py
│   └── test_scoring.py
├── integration/
│   ├── test_articles_api.py
│   ├── test_database.py
│   └── test_llm_client.py
└── e2e/
    ├── test_fetch_flow.py
    └── test_generate_flow.py
```

### Naming Convention

- Test files mirror the source file they test: `validators.py` → `test_validators.py`
- Test functions describe the scenario: `test_empty_url_returns_none`, `test_duplicate_article_skipped`
- Group related tests in classes or describe blocks

---

## Related Documents

- [07-error-handling](07-error-handling.md) — Testing error scenarios and graceful degradation
- [10-quality-ops](10-quality-ops.md) — Running tests in CI/CD pipelines
- [02-lifecycle](02-lifecycle.md) — When testing fits in the project lifecycle
