---
name: agents-sdk
description: Build AI agents in Python that work with any provider (AWS, GCP, Azure, self-hosted, etc.). Load when creating stateful agents, durable workflows, scheduled tasks, MCP servers, chat applications, voice agents, or browser automation. Covers Agent class, state management, callable RPC, Workflows, durable execution, queues, retries, and observability.
---

# Agent SDK (Python)

Provider-agnostic Python SDK for building AI agents that can be deployed anywhere.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Agent** | Long-running service with persistent state and scheduled tasks |
| **State** | Database-backed state with auto-sync to clients |
| **Callable** | RPC-style methods via HTTP or WebSocket |
| **Workflow** | Durable multi-step background processing |
| **Queue** | FIFO work queue with retries and backoff |

## Capabilities

- **Persistent state** — Database-backed (SQLite, PostgreSQL, DynamoDB)
- **Callable RPC** — Methods invoked over HTTP/WebSocket
- **Scheduling** — One-time, recurring, and cron tasks
- **Workflows** — Durable multi-step background processing
- **Durable execution** — Work that survives process restarts
- **Queue** — Built-in FIFO queue with retries
- **MCP integration** — Connect to or build MCP servers
- **Streaming chat** — Resumable streams, message persistence, tools
- **Observability** — Structured logging and metrics

## Installation

```bash
pip install agents-sdk
```

For chat agents:
```bash
pip install agents-sdk[chat]
```

## Project Structure

```
my-agent/
├── src/
│   └── agents/
│       ├── __init__.py
│       ├── counter.py
│       ├── chat.py
│       └── workflows.py
├── tests/
├── pyproject.toml
└── README.md
```

## Agent Class

```python
from agents_sdk import Agent, callable, State

class CounterState(State):
    count: int = 0

class Counter(Agent[CounterState]):
    def __init__(self):
        super().__init__(initial_state=CounterState())

    def validate_state_change(self, next_state: CounterState, source: str):
        if next_state.count < 0:
            raise ValueError("Count cannot be negative")

    def on_state_update(self, state: CounterState, source: str):
        print(f"State updated: {state}")

    @callable()
    def increment(self) -> int:
        self.set_state(CounterState(count=self.state.count + 1))
        return self.state.count
```

## Core APIs

| Task | API |
|------|-----|
| Read state | `self.state.count` |
| Write state | `self.set_state(CounterState(count=1))` |
| SQL query | `` self.sql("SELECT * FROM users WHERE id = ?", id) `` |
| Schedule (delay) | `await self.schedule(60, "task", payload)` |
| Schedule (cron) | `await self.schedule("0 * * * *", "task", payload)` |
| Schedule (interval) | `await self.schedule_every(30, "poll")` |
| RPC method | `@callable() def my_method(self) { ... }` |
| Streaming RPC | `@callable(streaming=True) def stream(self, query) { ... }` |
| Start workflow | `await self.run_workflow("ProcessingWorkflow", params)` |
| Enqueue work | `self.queue("handler", payload)` |
| Retry with backoff | `await self.retry(fn, max_attempts=5)` |
| Broadcast to clients | `self.broadcast(message)` |

## FastAPI Integration

```python
from fastapi import FastAPI
from agents_sdk import AgentRouter

app = FastAPI()
router = AgentRouter()

@router.agent("counter")
class Counter(Agent[CounterState]):
    # ... agent implementation
    pass

app.include_router(router)
```

## Flask Integration

```python
from flask import Flask
from agents_sdk import AgentManager

app = Flask(__name__)
manager = AgentManager(app)

@manager.agent("counter")
class Counter(Agent[CounterState]):
    # ... agent implementation
    pass
```

## Deployment Adapters

The SDK includes adapters for popular platforms:

| Provider | Adapter | Notes |
|----------|---------|-------|
| AWS Lambda | `agents_sdk.aws` | Uses DynamoDB for state |
| Google Cloud Run | `agents_sdk.gcp` | Uses Firestore for state |
| Azure Functions | `agents_sdk.azure` | Uses Cosmos DB for state |
| Railway | `agents_sdk.railway` | Uses PostgreSQL |
| Fly.io | `agents_sdk.fly` | Uses SQLite or PostgreSQL |
| Self-hosted | `agents_sdk.server` | SQLite, PostgreSQL, etc. |

## References

### Core
- **[references/state-scheduling.md](references/state-scheduling.md)** — State persistence, scheduling, SQL
- **[references/callable.md](references/callable.md)** — RPC methods, streaming, timeouts
- **[references/configuration.md](references/configuration.md)** — Config, bindings, setup

### Chat & Streaming
- **[references/streaming-chat.md](references/streaming-chat.md)** — ChatAgent, resumable streams, tools
- **[references/client-sdk.md](references/client-sdk.md)** — Client integration

### Background Processing
- **[references/workflows.md](references/workflows.md)** — Durable Workflows
- **[references/durable-execution.md](references/durable-execution.md)** — `run_fiber`, `stash`, surviving restarts
- **[references/queue-retries.md](references/queue-retries.md)** — Built-in queue, retry with backoff

### Integrations
- **[references/mcp.md](references/mcp.md)** — MCP client and server, transports
- **[references/email.md](references/email.md)** — Email routing and handling
- **[references/webhooks-push.md](references/webhooks-push.md)** — Webhooks, push notifications
- **[references/observability.md](references/observability.md)** — Logging, metrics, tracing
