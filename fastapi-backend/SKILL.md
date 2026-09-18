---
name: fastapi-backend
description: Scaffold or extend a production-grade Python backend service — FastAPI + SQLAlchemy/Alembic + Postgres/Redis/DuckDB + arq background jobs + uv/ruff/bandit/import-linter tooling + optional OpenAI/Anthropic AI integration and agentic flows. Use this whenever the user asks to start a new Python backend/API project, add a FastAPI service, set up a database layer, background job queue, AI/LLM integration, agent workflow, CI-quality tooling, git hooks, or Docker/cloud deployment for a Python backend — even if they only name one piece (e.g. "add alembic migrations" or "set up arq workers") rather than the whole stack. This is a generic, opinionated backend template meant to work consistently whether invoked from Claude, Codex, opencode, aider, qwen-code, or any other terminal coding agent.
---

# FastAPI Backend

A generic, opinionated blueprint for production Python backend services. It fixes the stack, the
directory shape, and the quality gates so that any terminal coding agent (Claude Code, Codex,
opencode, aider/"agy", qwen-code, mimo, etc.) produces the same kind of codebase every time,
instead of re-deriving conventions per session.

**Locked-in stack:** Python 3.12+, uv, FastAPI, SQLAlchemy 2.0 (async) + Alembic, Postgres +
Redis + DuckDB, arq, ruff, bandit, import-linter, pre-commit git hooks, Docker.
**Flexible pieces (pick based on the task):** OpenAI-compatible client vs. Anthropic client (or
both, behind one interface), and how much "agentic" orchestration is actually warranted.

## How to use this skill

1. **Figure out what's actually being asked.** Most requests touch one slice of this stack
   ("add a background job", "wire up Claude", "set up migrations") — don't scaffold the whole
   repo unless the user is starting a brand-new project. Read only the reference file(s) that
   match the task; each one is self-contained.
2. **If starting a new project**, work through the references roughly in this order:
   `project-structure.md` → `tooling-quality.md` → `database-layer.md` →
   `fastapi-security.md` → `background-jobs.md` → (if AI is needed) `ai-integration.md` →
   `agentic-flow.md` → `deployment.md`.
3. **If extending an existing repo**, first look at what's already there (`pyproject.toml`,
   directory layout, existing migrations) and match the existing conventions before importing
   a reference file's opinions wholesale — these references are defaults, not overrides.
4. **Never skip the quality gates** when generating new code: ruff format + lint, bandit, and
   import-linter contracts should pass before you consider a task done. See
   `tooling-quality.md`.

## Reference files

| File | Read this when the task involves... |
|---|---|
| `references/project-structure.md` | New project setup, `uv init`, `pyproject.toml`, choosing where new code should live (vertical-slice layout), idiomatic Python 3.12 conventions |
| `references/tooling-quality.md` | ruff, bandit, import-linter contracts, pre-commit / git hooks, CI quality gates |
| `references/fastapi-security.md` | Creating the FastAPI app, routers, auth (JWT/OAuth2), rate limiting, security headers, secrets, request validation |
| `references/database-layer.md` | SQLAlchemy models/sessions, Alembic migrations, choosing Postgres vs. Redis vs. DuckDB, repository pattern |
| `references/background-jobs.md` | arq workers, cron jobs, and persisting job execution history to the database |
| `references/ai-integration.md` | Adding OpenAI-compatible or Anthropic (Claude) LLM calls, provider abstraction, streaming, retries |
| `references/agentic-flow.md` | Multi-step/tool-using agent behavior — decision tree for "do I even need a framework", and how to wire one in when you do |
| `references/deployment.md` | Dockerfile, docker-compose for local dev, and shipping to AWS/Azure/Render/Cloudflare |
| `references/debugging.md` | Local debugging (debugpy), logging-based debugging, remote debugging, common issues and fixes — never add debug code to production |
| `references/testing.md` | Unit tests, integration tests, fixtures, mocking, coverage configuration |
| `references/caching.md` | Redis caching strategies, cache-aside pattern, rate limiting, invalidation |
| `references/cicd.md` | GitHub Actions CI/CD pipelines for testing, linting, deploying |
| `references/documentation.md` | OpenAPI spec, docstrings, API changelog, ReDoc |
| `references/logging-errors.md` | Core logging setup (TimedRotatingFileHandler), exception hierarchy, structured logging across layers |
| `references/email.md` | SendGrid/Resend provider abstraction, background email sending |
| `references/webhooks-push.md` | Webhook receivers with signature verification, FCM push notifications |
| `references/voice.md` | Real-time voice via WebSocket, STT/TTS providers |
| `references/think.md` | Chain-of-thought reasoning for agents |
| `references/codemode.md` | Sandboxed Python code execution for agents |
| `references/browse-the-web.md` | Playwright browser automation for agents |
| `references/mcp.md` | Model Context Protocol server for tool exposure |
| `references/server-driven-messages.md` | Server-controlled UI rendering |
| `references/human-in-the-loop.md` | Agent pause/approve/resume workflow |
| `references/observability.md` | Prometheus metrics, health checks, alerting |
| `references/client-sdk.md` | Typed Python SDK generation from OpenAPI |

## Non-negotiables (apply regardless of which reference you're reading)

- **Python ≥ 3.12** in `pyproject.toml` (`requires-python = ">=3.12"`); use modern syntax
  (`match`, `X | Y` unions, `type` statements, `Self`, generics without `TypeVar` boilerplate
  where 3.12 allows it).
- **uv is the only package/venv manager.** No pip, poetry, or pipenv commands in docs or CI.
- **Vertical slices, not horizontal layers.** Code is organized by feature/domain, not by
  technical role (`models/`, `views/`, `services/` at the top level is the anti-pattern here).
- **Every new dependency** gets added via `uv add` (or `uv add --dev`) so the lockfile stays
  authoritative — never hand-edit `pyproject.toml` dependency lists and forget to sync.
- **Async all the way down** for I/O: FastAPI route handlers, SQLAlchemy sessions, arq jobs, and
  AI provider calls are all `async def` unless there's a specific reason not to be.
- **Secrets never hit the repo.** Configuration goes through `pydantic-settings` reading from
  environment variables / `.env` (gitignored), never hardcoded.
- **Every job/agent run that matters is auditable**: background jobs write their execution
  history to Postgres (see `background-jobs.md`), not just to arq's ephemeral Redis result store.
