# Module: Agents & Orchestration

Patterns for building AI agents, multi-step autonomous workflows, human-in-the-loop systems, and automated procedures. This module operates in two layers: conceptual patterns (what to build) and recommended implementations (how to build it).

**Prerequisite knowledge:** [modules/ai-llm-integration](ai-llm-integration.md) (LLM client patterns), [07-error-handling](../07-error-handling.md) (retries, cancellation), [06-security](../06-security.md) (input validation, access control).

---

## Conceptual Layer

### When to Use Agents vs Simple LLM Calls

Not every LLM integration needs an agent. Most tasks are better served by a simple request → response call.

| Approach | Best for | Complexity |
|----------|----------|-----------|
| **Single LLM call** | Classification, summarization, translation | Low |
| **Multi-step pipeline** | Sequential processing (classify → summarize → generate) | Medium |
| **Agent with tools** | Tasks requiring judgment, iteration, and external actions | High |
| **Multi-agent orchestration** | Complex workflows with specialized sub-tasks | Very high |

**Use an agent when:**
- The task requires multiple steps with branching decisions
- The agent needs to use tools (search, code execution, API calls) based on intermediate results
- The number of steps isn't known in advance
- The task benefits from reflection ("did my output meet the criteria?")

**Don't use an agent when:**
- The task is a straightforward input → output transformation
- The steps are fixed and predictable (use a pipeline instead)
- The cost of agent exploration exceeds the value (agents burn more tokens)

---

### The Agent Loop

The fundamental agent pattern is an observe-decide-act-reflect cycle.

```
        ┌─────────────┐
        │   Observe    │  ← Read environment, receive input, check tool results
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │   Decide     │  ← LLM determines next action based on observations
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │    Act        │  ← Execute tool, call API, write output
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │   Reflect     │  ← Evaluate result. Done? Continue? Adjust?
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │  Done / Loop  │  ← If done, return result. If not, loop back.
        └───────────────┘
```

**Why this pattern:** Agents that just act without observing or reflecting tend to go off-track. The observe-reflect steps create natural checkpoints for error detection, course correction, and termination.

**In practice:**
- Set a maximum loop count (prevent infinite loops — 10-25 iterations is typical)
- Log each loop iteration (observation, decision, action, result) for debugging
- Include an explicit "done" decision — agents should know when to stop
- Cost track each iteration — agents can be expensive if they loop excessively

---

### Tool Use

Agents become powerful when they can take actions beyond text generation.

#### Tool Design Principles

- **Focused tools** — each tool does one thing well ("search files," "run query," "send email")
- **Clear descriptions** — the tool description is what the agent reads to decide when to use it
- **Structured input/output** — tools take typed parameters and return structured results
- **Idempotent where possible** — running a read-only tool twice should be safe
- **Scoped permissions** — tools should have the minimum access they need

#### Common Tool Categories

| Category | Examples | Risk level |
|----------|----------|-----------|
| **Read-only** | Search, read file, query database, fetch URL | Low |
| **Write (reversible)** | Create file, insert record, send draft | Medium |
| **Write (irreversible)** | Send email, publish post, delete record, make payment | High |
| **System** | Execute code, run shell command, modify config | Very high |

**Why categorize by risk:** The risk level determines how much human oversight is needed. Read-only tools can run freely. Irreversible tools should require approval.

---

### Human-in-the-Loop (HITL)

Autonomous agents need guardrails. Human-in-the-loop patterns ensure humans maintain control over critical decisions.

#### Approval Gates

Insert a human approval step before high-risk actions.

```
Agent decides action → Present to human → Human approves/rejects → Agent proceeds/adjusts
```

**When to use:**
- Before irreversible actions (sending messages, publishing, deleting)
- Before expensive actions (large API calls, batch operations)
- When the agent's confidence is low
- For the first N executions of a new agent (build trust before increasing autonomy)

#### Review Checkpoints

Pause the agent at natural milestones for human review.

```
Agent completes Phase 1 → Human reviews output → Human says "continue" or "adjust" → Agent proceeds
```

**When to use:**
- Multi-phase workflows where early errors compound
- Creative tasks where output quality is subjective
- Tasks where the agent might go off-track over many steps

#### Override Mechanisms

Humans must always be able to:
- **Cancel** — stop the agent mid-execution, clean up partial work
- **Redirect** — change the agent's goal or constraints mid-task
- **Rollback** — undo the agent's actions (when possible)
- **Escalate** — flag the task for manual handling when the agent can't proceed

**Why HITL matters:** Fully autonomous agents operating without oversight can cause damage at scale — sending wrong emails, deleting wrong records, burning API credits. HITL is not a limitation; it's a safety feature that builds trust incrementally.

---

### Orchestration

When a task is too complex for a single agent, decompose it across multiple specialized agents.

#### Task Decomposition

Break a complex task into sub-tasks, each handled by a specialized agent.

```
Orchestrator
├── Research Agent    → gathers information
├── Analysis Agent    → processes and evaluates
├── Writing Agent     → produces output
└── Review Agent      → validates quality
```

**Patterns:**
- **Sequential** — agents run in order, each building on the previous output
- **Parallel** — independent sub-tasks run concurrently, results aggregated
- **Hierarchical** — an orchestrator agent delegates to sub-agents and synthesizes results
- **Collaborative** — agents pass work back and forth iteratively

#### Orchestrator Responsibilities

The orchestrator (which may itself be an LLM or simple code) manages:
- Task decomposition and assignment
- Data flow between agents
- Aggregation of results
- Error handling when a sub-agent fails
- Completion criteria (when is the overall task done?)

#### Orchestration Pseudocode

**Sequential orchestration** (most common):
```
results = {}
for step in [research, analysis, writing, review]:
    agent = create_agent(step.type, step.config)
    input = build_input(step, results)  # previous results inform next step
    result = await agent.run(input, max_iterations=step.limit)
    if result.failed:
        handle_failure(step, result)  # retry, skip, or abort
        break
    results[step.name] = result.output
return aggregate(results)
```

**Parallel orchestration** (for independent sub-tasks):
```
tasks = decompose(goal)  # break goal into independent sub-tasks
agents = [create_agent(task) for task in tasks]
results = await gather_with_errors(
    [agent.run(task) for agent, task in zip(agents, tasks)]
)
# Some may have failed — aggregate what succeeded
return synthesize(results)
```

**Why orchestration:** A single agent trying to do everything tends to lose focus. Specialized agents with clear mandates produce higher-quality output. But orchestration adds complexity — only use it when the task genuinely benefits from specialization.

---

### Automated Procedures

Agents that run on schedules or triggers without human initiation.

#### Scheduled Agents

Run an agent on a recurring schedule (daily, hourly, on-commit, etc.).

**Examples:**
- Daily security scan of dependencies
- Weekly summary of project activity
- Hourly monitoring check with automated response
- Post-deploy smoke test

**Requirements:**
- **Idempotent** — running twice should be safe
- **Observable** — log every run with outcome (success, failure, actions taken)
- **Bounded** — set time limits and action limits per run
- **Alerting** — notify humans on failure or anomalous results
- **Graceful failure** — a failed scheduled run shouldn't prevent the next one

#### Event-Triggered Agents

Run an agent in response to events (webhook, file change, new data).

**Examples:**
- New issue filed → agent triages and labels
- PR opened → agent reviews code
- Alert fired → agent investigates and provides summary
- New data ingested → agent classifies and processes

**Requirements:**
- **Debounced** — don't trigger on every event if they arrive in bursts
- **Filtered** — not every event needs an agent; filter first
- **Timeout** — event-triggered agents must complete within a time limit

---

### Agent Memory and State

Agents that run across multiple interactions or sessions need persistent state.

#### Conversation History

- Store recent interactions for context (not unlimited — trim to relevance)
- Summarize old history rather than discarding it entirely
- Separate task state from conversation history

#### Task State

- Track where the agent is in a multi-step process
- Store intermediate results for resumption after interruption
- Persist decisions and their rationale for debugging

#### Persistent Knowledge

- Facts the agent has learned that are relevant to future tasks
- User preferences and feedback on agent behavior
- Domain knowledge accumulated over time

**Why state matters:** Stateless agents repeat mistakes, re-learn preferences, and can't resume interrupted work. Stateful agents improve over time. But state also introduces complexity — manage it deliberately.

---

### Agent Safety

Autonomous agents operating at scale require safety measures proportional to their capabilities.

#### Output Validation

- Validate agent-generated content before publishing or sending
- Check for hallucinations by verifying claims against source data
- Flag low-confidence outputs for human review

#### Action Limits

- Maximum number of actions per run (prevent infinite loops)
- Maximum cost per run (prevent credit burn)
- Maximum scope of changes (prevent accidental mass modifications)
- Rate limits on external actions (prevent spam, API abuse)

#### Rollback Capability

- For reversible actions: maintain an undo log
- For irreversible actions: require human approval (→ see HITL above)
- For batch operations: process in small batches with checkpoints

#### Escalation Paths

When an agent encounters something it can't handle:
1. Log the situation with full context
2. Notify a human with a clear description of the problem
3. Pause or gracefully degrade rather than guessing
4. Provide enough context for the human to take over

---

### Debugging and Observability

Agents fail in ways that are harder to diagnose than traditional software. An agent that produces wrong output may not throw an error — it just makes a bad decision.

#### Logging for Debuggability

- **Log each loop iteration** — observation, decision, action, result. This is the agent's "execution trace."
- **Log the LLM's reasoning** — if the model explains its choice (chain-of-thought), log it. When the agent goes wrong, the reasoning shows where.
- **Log tool inputs and outputs** — "Called search('ransomware') → 15 results" is far more debuggable than "Called search → success."
- **Log timing per step** — if the agent is slow, you need to know whether the bottleneck is LLM calls, tool execution, or processing.

#### Common Failure Patterns

- **Infinite loops** — the agent repeats the same action because it doesn't recognize the result hasn't changed. Fix: compare current observation to previous; if identical for N iterations, force stop.
- **Tool misuse** — the agent calls a tool with wrong parameters. Fix: validate tool inputs before execution; return clear error messages that help the LLM self-correct.
- **Goal drift** — the agent starts on-task but gradually shifts focus. Fix: include the original goal in every LLM prompt, not just the first one.
- **Stuck in reflection** — the agent keeps evaluating without acting. Fix: set a maximum consecutive "think" steps before requiring an action.

---

### Agent Communication Patterns

When multiple agents work together, they need a protocol for sharing context.

#### Message-Based Communication

Agents communicate through structured messages, not shared state.

```
Message:
  from: "research_agent"
  to: "analysis_agent"
  type: "handoff"
  payload: { articles: [...], search_context: "...", open_questions: [...] }
```

**Why:** Shared mutable state between agents creates race conditions and makes debugging nearly impossible. Messages create an auditable trail of what each agent knew and when.

**In practice:**
- Define a message schema that all agents understand (sender, receiver, type, payload)
- Include context that the receiving agent needs — don't assume it can read the sending agent's state
- Log all inter-agent messages for debugging
- Keep payloads concise — send summaries and references, not raw data

#### Context Handoff

When one agent passes work to another, the handoff should include:
- **What was accomplished** — summary of completed work
- **What remains** — clear description of the next task
- **Relevant context** — data, decisions made, constraints discovered
- **Open questions** — things the sending agent wasn't sure about

---

### Cost Modeling for Agents

Agents consume more tokens than simple LLM calls because they loop. A 10-iteration agent loop costs 10x a single call (approximately — context grows each iteration).

**Estimating agent costs:**
- **Per-iteration cost** = input tokens (system prompt + history + observation) + output tokens (decision + action)
- **History growth** — each iteration adds to history. By iteration 10, the input is much larger than iteration 1.
- **Tool costs** — some tools trigger additional LLM calls (sub-agents, classification). Include these.
- **Total cost** = sum of all iterations + tool costs

**Controlling costs:**
- Set a maximum iteration count (the primary cost control lever)
- Set a token budget per agent run — stop if exceeded
- Use cheaper models for intermediate reasoning steps; reserve expensive models for final output
- Trim conversation history aggressively — summarize old iterations rather than keeping full transcripts
- Monitor actual costs per agent type and adjust limits based on observed patterns

---

## Recommended Implementations

### Claude Agent SDK

For building custom agents powered by Claude.

- **Tool use** — define tools as functions; Claude decides when to call them
- **Multi-turn conversations** — agent maintains context across interactions
- **Streaming** — stream agent responses and tool results in real-time
- **Built-in safety** — Claude's constitutional AI provides baseline safety

**Best for:** Custom agents with specific tool requirements, embedded in your application.

### Claude Code (CLI/IDE Agent)

For development-focused agents that work with code.

- **File system access** — read, write, and search code
- **Shell execution** — run builds, tests, linters
- **Git integration** — commit, branch, review
- **Scheduled triggers** — cron-based execution for recurring tasks
- **Hooks** — event-driven actions (before/after tool calls)

**Best for:** Development automation, code review, CI/CD integration, scheduled maintenance tasks.

### LangGraph / LangChain

For complex multi-step workflows with state management.

- **Graph-based orchestration** — define workflows as directed graphs
- **State management** — built-in state persistence across steps
- **Branching and loops** — conditional logic, human-in-the-loop, retry
- **Multi-agent** — coordinate multiple agents within a workflow

**Best for:** Complex workflows with branching logic, multiple agents, and persistent state.

### Simple Custom Agents

For straightforward agent needs, a custom loop is often simpler than a framework:

```
while not done and iterations < max_iterations:
    observation = gather_context()
    decision = call_llm(system_prompt, observation, history)
    if decision.action == "done":
        done = True
    else:
        result = execute_tool(decision.action, decision.params)
        history.append(decision, result)
```

**Best for:** Single-purpose agents where framework overhead exceeds the value.

---

## Choosing an Approach

```
Simple task (classification, summarization)
→ Single LLM call (see ai-llm-integration.md)

Fixed multi-step task (ingest → classify → generate)
→ Pipeline (code-driven sequence of LLM calls)

Task requiring judgment and tools
→ Single agent with tool use

Complex task with specialization
→ Multi-agent orchestration

Recurring task
→ Scheduled agent (cron trigger + agent loop)

Event-driven task
→ Event-triggered agent (webhook/hook + agent loop)
```

---

## Related Documents

- [modules/ai-llm-integration](ai-llm-integration.md) — LLM client patterns (the foundation agents build on)
- [07-error-handling](../07-error-handling.md) — Retry, cancellation, graceful degradation
- [06-security](../06-security.md) — Access control for agent actions
- [03-architecture](../03-architecture.md) — Background jobs and pipeline patterns
