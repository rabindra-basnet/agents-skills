# State & Scheduling

## State Management

State persists to storage and broadcasts to connected clients automatically.

### Define Typed State

```python
from agents_sdk import State

class MyState(State):
    count: int = 0
    items: list[str] = []
```

### Read and Update

```python
class MyAgent(Agent[MyState]):
    def increment(self) -> int:
        # Read
        count = self.state.count
        
        # Write (sync, persists, broadcasts)
        self.set_state(MyState(count=count + 1))
        return self.state.count
```

### Validation Hook

`validate_state_change()` runs synchronously before state persists. Raise to reject the update.

```python
class MyAgent(Agent[MyState]):
    def validate_state_change(self, next_state: MyState, source: str):
        if next_state.count < 0:
            raise ValueError("Count cannot be negative")
```

### Execution Order

1. `validate_state_change(next_state, source)` - sync, gating
2. State persisted to storage
3. State broadcast to connected clients
4. `on_state_update(next_state, source)` - async, non-gating

### Client-Side Sync

```python
import requests

# WebSocket connection for real-time sync
ws = websocket.WebSocket()
ws.connect("ws://localhost:3000/agents/my-agent/instance-1")

# Receive state updates
while True:
    message = ws.recv()
    state = json.loads(message)
    print(f"State: {state}")
```

## SQL API

Direct SQLite access for custom queries (when using SQLite storage):

```python
# Create table
self.sql("""
    CREATE TABLE IF NOT EXISTS items (
        id TEXT PRIMARY KEY,
        name TEXT,
        created_at INTEGER DEFAULT (unixepoch())
    )
""")

# Insert
self.sql("INSERT INTO items (id, name) VALUES (?, ?)", id, name)

# Query with types
items = self.sql(
    "SELECT * FROM items WHERE name LIKE ?",
    f"%{search}%"
)
```

## Scheduling

### Schedule Types

| Mode | Syntax | Use Case |
|------|--------|----------|
| Delay | `self.schedule(60, ...)` | Run in 60 seconds |
| Date | `self.schedule(datetime(...), ...)` | Run at specific time |
| Cron | `self.schedule("0 8 * * *", ...)` | Recurring schedule |
| Interval | `self.schedule_every(30, ...)` | Fixed interval (every 30s) |

### Examples

```python
from datetime import datetime, timedelta

# Delay (seconds)
await self.schedule(60, "check_status", {"id": "abc123"})

# Specific date
await self.schedule(datetime(2025, 12, 25), "send_greeting", {"to": "user"})

# Cron (recurring)
await self.schedule("0 9 * * 1-5", "weekday_report", {})

# Fixed interval (every 30 seconds, overlap prevention built-in)
await self.schedule_every(30, "poll_updates")
await self.schedule_every(300, "sync_data", {"source": "api"})
```

### Handler

```python
async def send_greeting(self, payload: dict, schedule: Schedule):
    print(f"Sending greeting to {payload['to']}")
    # Cron schedules auto-reschedule; one-time schedules are deleted
```

### Manage Schedules

```python
schedules = self.get_schedules()
crons = self.get_schedules(type="cron")
await self.cancel_schedule(schedule.id)
```

### Retry on Schedules

```python
await self.schedule(60, "task", payload, retry={"max_attempts": 3})
await self.schedule_every(30, "poll", retry={"max_attempts": 2})
```

## Lifecycle Callbacks

```python
class MyAgent(Agent[MyState]):
    async def on_start(self):
        # Agent started or woke from hibernation
        pass

    def on_connect(self, conn: Connection, ctx: ConnectionContext):
        # WebSocket connected
        pass

    def on_message(self, conn: Connection, message: str):
        # WebSocket message (non-RPC)
        pass

    def on_state_update(self, state: MyState, source: str):
        # State changed (async, non-blocking)
        pass

    def on_error(self, error: Exception):
        # Error handler
        raise error  # Re-raise to propagate
```
