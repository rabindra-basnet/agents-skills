# Agentic Flow: Do You Need a Framework, and Which One

Research summary (2026 landscape) + a decision tree. The goal is to avoid the default trap of
reaching for a heavyweight agent framework when a plain function call would do, while still
having a real answer for when the backend genuinely needs multi-step, tool-using, stateful
agent behavior.

## What the landscape looks like right now

- **No framework** (a single structured-output LLM call, or one call with tool-use resolved in
  a plain `while` loop you write yourself) handles a large share of what gets labeled "agent"
  work in practice — classification, extraction, single-turn Q&A with a couple of tool calls.
  Try this first; only escalate when it's genuinely insufficient.
- **LangGraph** models an agent as an explicit state machine (nodes + edges you define), giving
  you durable/resumable execution, explicit error-routing, and human-in-the-loop interrupt
  points. It's provider-agnostic (works with any client, including this stack's own
  `AIProvider` abstraction or an OpenAI-compatible base URL), which matters here since this
  stack supports both OpenAI and Anthropic. It's the most commonly cited choice for production
  workflows that need real state control and auditability.
- **OpenAI Agents SDK** and **Claude Agent SDK** are strong, well-supported options but are
  each tied to their own provider's ecosystem/runtime assumptions (sandboxing, provider-native
  tool loop). They fit best when the backend is committed to a single provider — since this
  stack explicitly wants to support both OpenAI-compatible and Anthropic clients behind one
  interface, a provider-locked SDK is a worse structural fit than a provider-agnostic
  orchestrator.
- **CrewAI / AutoGen**-style role-based multi-agent frameworks optimize for fast prototyping of
  multi-agent collaboration, at the cost of less deterministic control over execution flow.
  Good for exploratory/internal tooling, riskier for a production request path where you need
  predictable latency/cost/error handling.
- **Pydantic AI** is a lighter-weight, type-safe option worth considering for a *single* agent
  with tool calls and validated structured output, when you don't need LangGraph's graph/state
  machinery — it fits naturally with this stack since Pydantic is already the schema layer for
  FastAPI and the `AIProvider` result types.

## Decision tree

1. **Is this actually multi-step / does it need tools at all?**
   If the task is "call the model once, maybe with structured output," don't add a framework —
   call `AIProvider.complete()` directly from `service.py`, optionally with one manual
   tool-call-then-respond round trip. This covers most "AI feature" requests.
2. **Single agent, needs tool calls in a loop, but no branching/persistence requirements?**
   Use a thin custom loop (fetch model response → if it requested a tool, run it, feed the
   result back → repeat until a final answer) built on `AIProvider`, or adopt **Pydantic AI**
   if you want its tool-registration ergonomics and validated outputs without hand-rolling the
   loop. Keep it inside the feature's `service.py`.
3. **Multi-step workflow with real state, branching, retries-per-step, or a human-in-the-loop
   approval point (e.g. "draft → review → send"), especially anything that must survive a
   process restart mid-flow?**
   Use **LangGraph**. Model each step as a node, persist state via LangGraph's checkpointer
   (backed by Postgres/Redis, consistent with the rest of this stack), and drive both OpenAI
   and Anthropic calls through this stack's own `AIProvider` inside the graph's nodes — don't
   let LangGraph talk to the providers directly, so provider selection stays centralized.
4. **True multi-agent collaboration (specialist agents handing off work to each other) as an
   internal/ops tool, not a hot request path?**
   CrewAI-style is acceptable there; keep it out of latency-sensitive user-facing endpoints.

## Implementation notes when you do reach for LangGraph

- Run agentic workflows as **arq jobs**, not inline in a request handler — they're
  variable-latency and should follow the same enqueue/poll-or-webhook pattern as any other
  slow background task (see `background-jobs.md`). Persist the job's execution (including
  intermediate LangGraph state snapshots if useful for debugging) to the `job_executions` table
  the same way any other job does.
- Give each graph node explicit, narrow responsibility and a typed input/output (Pydantic
  models) — resist the temptation to let a node's LLM call "figure out" what the next step's
  shape should be.
- Cap iteration/step count and wall-clock time explicitly; a runaway agent loop is a live
  incident (and a live bill), not just a bug.
- Log every tool call and model call with enough context (request id, node name, tokens, cost)
  to reconstruct a run after the fact — this is not optional for anything touching production
  data or spending money.

## When to skip all of this

If someone asks for "an agent" and the real requirement is "summarize this document" or
"classify this ticket," say so and implement the one-call version. Adding orchestration
machinery to a single-call problem is the most common failure mode in this space — it adds
latency, cost, and failure surface for no behavioral benefit.

## DO NOT

- **Never** reach for a heavyweight agent framework for a single LLM call — call `AIProvider.complete()` directly.
- **Never** use provider-locked SDKs (OpenAI Agents SDK, Claude Agent SDK) when the stack supports multiple providers.
- **Never** run agentic workflows inline in request handlers — they're variable-latency, use arq jobs.
- **Never** let agent loops run without caps on iteration count and wall-clock time — runaway agents are live incidents.
- **Never** let a node's LLM call "figure out" the next step's shape — use typed input/output (Pydantic models).
- **Never** skip logging tool calls and model calls — you need to reconstruct runs for debugging.
- **Never** use CrewAI/AutoGen for production request paths — they're for internal/ops tools only.
- **Never** add orchestration to a single-call problem — it adds latency, cost, and failure surface for no benefit.
- **Never** let LangGraph talk to providers directly — route through this stack's `AIProvider`.
- **Never** persist intermediate LangGraph state without a clear debugging purpose — it's overhead.
- **Never** treat "agent" as a feature — treat it as an implementation detail that may or may not be needed.
- **Never** skip the decision tree — most "agent" requests are actually single-call problems.
