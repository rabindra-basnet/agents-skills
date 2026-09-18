# Queue & Retries

## Built-in Queue

FIFO queue persisted in storage. Sequential processing, one item at a time.

```python
class MyAgent(Agent):
    async def on_request(self, request):
        self.queue("process_item", {"id": "abc", "data": "..."})
        self.queue("process_item", {"id": "def", "data": "..."}, retry={"max_attempts": 5})
        return Response("Queued")

    async def process_item(self, payload: dict, queue_item: QueueItem):
        await do_work(payload)
```

### Queue Management

```python
items = self.get_queue()
by_callback = self.get_queues("process_item")
self.dequeue(item_id)
self.dequeue_all()
self.dequeue_all_by_callback("process_item")
```

## Retries

Exponential backoff with full jitter. Defaults: 3 attempts, 100ms base, 3000ms max.

```python
result = await self.retry(
    lambda: fetch_data(),
    max_attempts=5,
    base_delay_ms=200,
    max_delay_ms=5000,
    should_retry=lambda err, attempt: "429" in str(err) or attempt <= 3
)
```

### Retry on Schedules and Queue

```python
await self.schedule(60, "task", payload, retry={"max_attempts": 3})
await self.schedule_every(30, "poll", retry={"max_attempts": 2})
self.queue("handler", payload, retry={"max_attempts": 5})
```

### Class-level Defaults

```python
class MyAgent(Agent):
    options = {
        "retry": {"max_attempts": 5, "base_delay_ms": 200, "max_delay_ms": 10000}
    }
```

## Important

- `should_retry` only works on `self.retry()` — not on schedule/queue (callbacks aren't serializable)
- Queue retries block head-of-line; long delays keep the process awake — use `schedule` for long waits instead
- No dead-letter queue — failed items are removed after retries exhausted
