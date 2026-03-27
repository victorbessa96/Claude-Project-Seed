# Module: AI/LLM Integration

Patterns for integrating large language models into applications. Organized in two layers: conceptual requirements (what to achieve) and recommended implementations (how to achieve it).

**Prerequisite knowledge:** [03-architecture](../03-architecture.md) (layered architecture, API design), [07-error-handling](../07-error-handling.md) (retries, graceful degradation), [06-security](../06-security.md) (secrets management).

---

## Conceptual Layer

### Client Abstraction

All LLM calls should go through a single client module. Business logic should never call LLM APIs directly.

**Why:** A central client is the single place to add logging, retries, rate limiting, cost tracking, and provider switching. Without it, these concerns are scattered (or missing) across the codebase.

**The client should handle:**
- API authentication (read key from settings, never hardcode)
- Request formatting (model selection, system prompt, user prompt, parameters)
- Response parsing (extract content, handle malformed responses)
- Error handling and retries (→ see retry section below)
- Usage tracking (log every call for cost visibility)
- Timeout configuration (LLM calls are slower than typical APIs)

> **CyberPulse example:** All LLM calls go through `openrouter_client.py`. It handles authentication, retries, usage tracking, and timeout configuration. No other module makes HTTP requests to the LLM API.

---

### Prompt Management

Centralize all prompts in a single module. Never scatter prompts across business logic.

**Why:** Prompts are the primary lever for output quality. Scattered prompts are impossible to audit, version, compare, or improve systematically. A central prompt module makes it easy to A/B test, add language variants, and iterate on quality.

**Patterns:**
- **System prompts** — define the AI's role, constraints, and output format
- **User prompts** — provide the specific input and task for each request
- **Language variants** — separate prompts per language (not just translated strings — culturally appropriate instructions)
- **Output format specification** — explicitly define expected format (JSON, markdown, specific structure) in the system prompt

**In practice:**
- Store prompts as string constants or templates in one file
- Use template variables for dynamic content (article text, language, output type)
- Include explicit formatting instructions ("Respond in JSON with keys: title, body, summary")
- For non-English output: include explicit language mandates in the target language

> **CyberPulse example:** `prompts.py` contains all prompts: classification (EN), condensation (EN/PT), generation (LinkedIn/blog/digest in EN/PT). Portuguese prompts include `IMPORTANTE: Você DEVE escrever todo o conteúdo inteiramente em Português do Brasil`.

---

### Bilingual/Multilingual Output

When generating content in non-English languages, explicit mandates outperform simple instructions.

**Why:** LLMs have a strong English default. "Write in Portuguese" often produces English with Portuguese phrases. Explicit mandates in the target language, combined with target-language user prompts, dramatically improve compliance.

**Patterns:**
- Write the system prompt instruction in the target language
- Write the user prompt in the target language
- Include a forceful mandate: "You MUST write entirely in [language]"
- Use a helper function to select the appropriate prompt variant by language code

---

### Cost Control

LLM API calls cost money. Every call should be visible, trackable, and controllable.

**Why:** Without usage visibility, a bug that triggers infinite LLM calls can burn through an entire API budget in minutes. Cost control is both a financial and a reliability concern.

**Patterns:**
- **Usage tracking** — log every API call (model, tokens used, timestamp) in a database table
- **Rate limiting** — cooldown on generation endpoints to prevent accidental double-triggers (→ see [08-performance](../08-performance.md))
- **Model selection per task** — use cheaper/faster models for classification, expensive models for generation
- **Daily usage dashboard** — show today's API call count and estimated cost
- **Budget alerts** — warn when approaching spending thresholds

**In practice:**
- Track usage with a simple INSERT per call (non-critical — wrap in try/catch so tracking failures don't break generation)
- Expose usage data in the UI (dashboard, settings page)
- Let users choose the model per task if cost sensitivity varies

---

### Retry and Resilience

LLM APIs have rate limits, occasional outages, and variable response times.

**Patterns:**
- **Exponential backoff on rate limits (429)** — wait 2^attempt seconds before retrying
- **Maximum retries** — 3 attempts is typical; don't retry indefinitely
- **Timeout configuration** — LLM calls need longer timeouts (60-120s) than typical API calls
- **Fallback models** — if the primary model is unavailable, try a backup model
- **Graceful failure** — if all retries fail, return a clear error; don't crash the application

> **CyberPulse example:** OpenRouter client retries up to 3 times on 429 responses with exponential backoff. Connection timeout is 10s, total timeout is 60s. If all retries fail, the error is logged and surfaced to the user.

---

### Response Parsing and Validation

LLMs don't always follow instructions. Validate responses before storing or displaying.

**Patterns:**
- **Expected format validation** — if you asked for JSON, verify it parses as JSON
- **Required field checking** — verify all expected fields are present
- **Fallback on malformed responses** — retry once, or use a default/error state
- **Content safety** — check for obviously wrong output (empty, truncated, wrong language)
- **Length bounds** — verify output isn't suspiciously short or long

**Why:** A malformed LLM response stored in the database becomes a persistent bug. Validate at the boundary between the LLM client and the storage layer.

---

### Model Selection

Different tasks have different speed/quality/cost trade-offs.

| Task type | Priority | Model tier |
|-----------|----------|-----------|
| Classification / tagging | Speed, cost | Fast/cheap models |
| Summarization | Balance | Mid-tier models |
| Content generation | Quality | Best available model |
| Code generation | Quality + accuracy | Best available model |

**In practice:**
- Let users select the model from the UI (fetch available models from the provider)
- Store the default model in settings; allow per-task override
- Display the model used alongside generated output (for quality assessment)
- Fall back to a hardcoded default if the settings model is unavailable

> **CyberPulse example:** Model selection is UI-configurable. Available models are fetched from OpenRouter and displayed in a dropdown. The selected model is stored in settings. A hardcoded fallback is used if the setting is empty.

---

## Recommended Implementations

### API Providers

| Provider | Strength | Trade-off |
|----------|----------|-----------|
| **OpenRouter** | Model-agnostic; access to Claude, GPT, Llama, etc. via one API | Extra hop; slightly higher latency |
| **Anthropic API** | Direct access to Claude models; best Claude performance | Locked to Claude models |
| **OpenAI API** | Direct access to GPT models; widest ecosystem | Locked to OpenAI models |

**Recommendation:** Start with OpenRouter for flexibility (easy model switching, no vendor lock-in). Switch to a direct API when you've identified your model and need maximum performance.

### SDK Patterns

Most providers offer official SDKs with built-in retry logic, streaming, and type safety:

- **Anthropic SDK** (`anthropic` for Python, `@anthropic-ai/sdk` for JS) — Claude models
- **OpenAI SDK** (`openai` for Python, `openai` for JS) — GPT models, also works with OpenRouter

For simple integrations, a raw HTTP client (httpx, fetch) with your own retry logic is equally valid and avoids an SDK dependency.

### Streaming Responses

For long-generation tasks, stream the LLM response to the user instead of waiting for completion.

- Use Server-Sent Events (SSE) from backend to frontend
- Display text as it arrives (typewriter effect)
- Allow cancellation mid-stream

**When to use:** When generation takes > 5 seconds and the user is watching. For background/batch processing, streaming is unnecessary.

---

## Pipeline Patterns

### Multi-Stage LLM Processing

Complex tasks often benefit from multiple LLM calls in sequence, each with a focused prompt.

```
Raw Input → Classify (fast model) → Summarize (mid model) → Generate (best model)
```

**Why:** A single prompt that classifies, summarizes, and generates is harder to tune and more expensive to re-run. Separate stages allow re-processing individual steps, using different models per step, and caching intermediate results.

> **CyberPulse example:** Three-stage pipeline: Classification (assign categories + relevance score) → Condensation (generate 3-5 sentence summary) → Generation (produce LinkedIn/blog/digest output). Each stage stores its result in the database. Users trigger generation separately from classification.

### Batch Processing

When processing multiple items, batch them for efficiency.

- Group items into batches (e.g., 10 articles per classification call)
- Process batches sequentially (not in parallel, to respect rate limits)
- Track progress per batch for user feedback
- Handle partial batch failures (some items succeed, some fail)

---

## Related Documents

- [03-architecture](../03-architecture.md) — Layered architecture, client abstraction
- [06-security](../06-security.md) — API key management, secrets handling
- [07-error-handling](../07-error-handling.md) — Retry strategies, graceful degradation
- [08-performance](../08-performance.md) — Rate limiting, timeout configuration
- [modules/agents-orchestration](agents-orchestration.md) — When to use agents instead of simple LLM calls
