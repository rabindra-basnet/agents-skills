# Durable Execution

Fibers let agent work survive process restarts. Progress is checkpointed to storage; on recovery, you decide what to do.

## `run_fiber`

```python
class MyAgent(Agent):
    async def on_request(self, request):
        await self.run_fiber("process-data", self.process_data)
        return Response("Started")

    async def process_data(self, ctx):
        step1 = await fetch_data()
        ctx.stash = {"step": 1, "data": step1}

        step2 = await transform(step1)
        ctx.stash = {"step": 2, "result": step2}

        self.set_state(MyState(result=step2))

    async def on_fiber_recovered(self, ctx):
        checkpoint = ctx.stash
        if checkpoint["step"] == 1:
            step2 = await transform(checkpoint["data"])
            self.set_state(MyState(result=step2))
```

## Key APIs

| API | Purpose |
|-----|---------|
| `self.run_fiber(name, fn)` | Start a named fiber |
| `ctx.stash` | Read latest checkpoint |
| `ctx.stash = data` | Write checkpoint (JSON-serializable) |
| `on_fiber_recovered(ctx)` | Called on restart if fiber was in-flight |
| `keep_alive()` | Prevent hibernation while fiber runs |
| `keep_alive_while(fn)` | Keep alive for duration of async function |

## Important

- `stash` replaces the entire checkpoint — not a merge
- The function is NOT restored on recovery — only the stash data is. You must re-derive what to do in `on_fiber_recovered`
- No auto-retry on raise — handle errors yourself
- For long-running pipelines with automatic retries, use Workflows instead
- Filter concurrent fibers by `ctx.name` in `on_fiber_recovered`
