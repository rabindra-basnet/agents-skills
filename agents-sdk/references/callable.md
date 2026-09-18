# Callable Methods

## Overview

`@callable()` exposes agent methods to clients via HTTP/WebSocket RPC.

```python
from agents_sdk import Agent, callable

class MyAgent(Agent):
    @callable()
    async def greet(self, name: str) -> str:
        return f"Hello, {name}!"

    @callable()
    async def process_data(self, data: dict) -> dict:
        # Long-running work
        return result
```

## Client Usage

```python
import requests

# Basic call via HTTP
response = requests.post(
    "http://localhost:3000/agents/my-agent/instance-1/rpc/greet",
    json={"args": ["World"]}
)
greeting = response.json()

# With timeout
response = requests.post(
    "http://localhost:3000/agents/my-agent/instance-1/rpc/process_data",
    json={"args": [data]},
    timeout=5
)
```

## Streaming Responses

```python
from agents_sdk import Agent, callable, StreamingResponse

class MyAgent(Agent):
    @callable(streaming=True)
    async def stream_results(self, stream: StreamingResponse, query: str):
        async for item in fetch_results(query):
            await stream.send(json.dumps(item))
        await stream.close()

    @callable(streaming=True)
    async def stream_with_error(self, stream: StreamingResponse):
        try:
            # ... work
            pass
        except Exception as e:
            await stream.error(str(e))  # Signal error to client
            return
        await stream.close()
```

Client with streaming:

```python
import websockets
import json

async def stream_call():
    async with websockets.connect("ws://localhost:3000/agents/my-agent/instance-1") as ws:
        # Send RPC call
        await ws.send(json.dumps({
            "method": "stream_results",
            "args": ["search term"]
        }))
        
        # Receive streamed chunks
        while True:
            message = await ws.recv()
            data = json.loads(message)
            if data.get("type") == "done":
                break
            print("Chunk:", data)
```

## Introspection

```python
# Get list of callable methods on an agent
methods = await agent.call("get_callable_methods", [])
# Returns: ["greet", "process_data", "stream_results", ...]
```

## When to Use

| Scenario | Use |
|----------|-----|
| Browser/mobile calling agent | `@callable()` |
| External service calling agent | `@callable()` |
| Service calling agent (same codebase) | Direct method call |
| Agent calling another agent | `get_agent_by_name()` + method call |
