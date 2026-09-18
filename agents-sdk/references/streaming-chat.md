# Streaming Chat with ChatAgent

`ChatAgent` provides streaming chat with automatic message persistence and resumable streams.

## Basic Chat Agent

```python
from agents_sdk import ChatAgent
from openai import OpenAI

class Chat(ChatAgent):
    def __init__(self):
        super().__init__()
        self.client = OpenAI()

    async def on_chat_message(self, on_finish, options=None):
        result = self.client.chat.completions.create(
            model="gpt-4o",
            system="You are a helpful assistant.",
            messages=self.messages,
            stream=True
        )
        
        for chunk in result:
            if chunk.choices[0].delta.content:
                await self.send(chunk.choices[0].delta.content)
        
        on_finish()
```

**Important:** Always pass `abort_signal` and `on_finish` — they enable proper cleanup and message persistence.

## With Tools

```python
from agents_sdk import ChatAgent, tool
from pydantic import BaseModel

class WeatherInput(BaseModel):
    location: str

@tool(description="Get weather for a location")
async def get_weather(input: WeatherInput) -> str:
    return f"Weather in {input.location}: 72°F, sunny"

class Chat(ChatAgent):
    def __init__(self):
        super().__init__()
        self.tools = [get_weather]

    async def on_chat_message(self, on_finish, options=None):
        # Tool handling is automatic
        result = await self.stream_with_tools(
            model="gpt-4o",
            messages=self.messages,
            tools=self.tools
        )
        on_finish()
```

## Resumable Streaming

Streams automatically resume if client disconnects and reconnects:

1. Chunks buffered to storage during streaming
2. On reconnect, buffered chunks sent immediately
3. Live streaming continues from where it left off

**Enabled by default.** To disable:

```python
agent = Chat(resume=False)
```

## HTTP Client

```python
import requests

# Send message
response = requests.post(
    "http://localhost:3000/agents/chat/my-session/message",
    json={"content": "Hello!"}
)

# Stream response
response = requests.post(
    "http://localhost:3000/agents/chat/my-session/message",
    json={"content": "Hello!"},
    stream=True
)

for chunk in response.iter_lines():
    if chunk:
        print(chunk.decode())
```

## WebSocket Client

```python
import websockets
import json

async def chat():
    async with websockets.connect("ws://localhost:3000/agents/chat/my-session") as ws:
        # Send message
        await ws.send(json.dumps({
            "type": "message",
            "content": "Hello!"
        }))
        
        # Receive streamed response
        while True:
            message = await ws.recv()
            data = json.loads(message)
            if data.get("type") == "done":
                break
            print(data.get("content", ""), end="")
```

## Streaming RPC Methods

For non-chat streaming, use `@callable(streaming=True)`:

```python
from agents_sdk import Agent, callable, StreamingResponse

class MyAgent(Agent):
    @callable(streaming=True)
    async def stream_data(self, stream: StreamingResponse, query: str):
        for i in range(10):
            await stream.send(f"Result {i}: {query}")
            await asyncio.sleep(0.1)
        await stream.close()
```

Client receives streamed messages via WebSocket RPC.

## Key Properties

| Property | Purpose |
|----------|---------|
| `self.messages` | All persisted messages |
| `max_persisted_messages` | Limit stored messages (prune oldest) |
| `message_concurrency` | `"queue"` (default), `"latest"`, `"merge"`, `"drop"` |
| `chat_recovery` | `"persist"` (default) or `"continue"` on reconnect |

## Status Values

| Status | Meaning |
|--------|---------|
| `ready` | Idle, ready for input |
| `streaming` | Response streaming |
| `submitted` | Request sent, waiting |
| `error` | Error occurred |
