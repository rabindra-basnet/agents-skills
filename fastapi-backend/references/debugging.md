# Debugging

Strategies for finding and fixing bugs in FastAPI applications — from local development to production.

## Local debugging (dev only)

### debugpy attach

```toml
# pyproject.toml
[tool.debugpy]
listen = ["0.0.0.0:5678"]
```

```python
# app/main.py — debugpy ONLY for local dev, never production
import os

if os.getenv("DEBUG"):
    import debugpy
    debugpy.listen(("0.0.0.0", 5678))
    print("Waiting for debugger attach...")
    debugpy.wait_for_client()
```

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to FastAPI",
      "type": "debugpy",
      "request": "attach",
      "connect": { "host": "localhost", "port": 5678 },
      "pathMappings": [
        { "localRoot": "${workspaceFolder}", "remoteRoot": "/app" }
      ]
    }
  ]
}
```

### Docker compose override (local only)

```yaml
# docker-compose.override.yml — NOT committed to repo
services:
  api:
    ports:
      - "5678:5678"
    environment:
      - DEBUG=true
    volumes:
      - .:/app
```

## Production rule

Debug code (debugpy, ipdb, verbose SQL logging, tracemalloc) must **never** exist in
production code. Gate all debug features behind environment checks:

```python
import os
DEBUG = os.getenv("DEBUG", "false").lower() == "true"

if DEBUG:
    import debugpy
    debugpy.listen(("0.0.0.0", 5678))
```

The `DEBUG` flag should be `false` in production Docker images and CI. Debug tooling
(`debugpy`, `ipdb`, `py-spy`, `memray`) should be dev-only dependencies:

```bash
uv add --dev debugpy ipdb py-spy memray
```

## Logging-based debugging

```python
# Add temporary debug logging
from app.core.logging import get_logger

logger = get_logger(__name__)

async def debug_this_function(data: dict):
    logger.debug("Function called", extra={"data_keys": list(data.keys())})
    logger.debug("Input data", extra={"data": str(data)[:500]})
    # ... logic
    logger.debug("Intermediate state", extra={"state": str(state)[:500]})
    # ... more logic
    logger.debug("Final result", extra={"result": str(result)[:500]})
```

## Remote debugging with Docker

```yaml
# docker-compose.override.yml
services:
  api:
    ports:
      - "5678:5678"
    environment:
      - DEBUG=true
    volumes:
      - .:/app  # Mount code for live reload
```

```bash
# Start with debug enabled
docker compose up api
# Then attach VS Code debugger to localhost:5678
```

## SQLAlchemy query debugging

```python
# Enable SQL query logging
import logging
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)

# Or use echo=True on engine
engine = create_async_engine(
    settings.database_url,
    echo=True,  # Log all SQL
    pool_size=10,
    pool_pre_ping=True,
)
```

```python
# Print the actual query SQL
from sqlalchemy import select
from app.features.users.models import User

query = select(User).where(User.email == "test@example.com")
print(query.compile(compile_kwargs={"literal_binds": True}))
```

## Redis debugging

```python
# app/core/redis.py — add debug logging
from app.core.logging import get_logger

logger = get_logger(__name__)

async def debug_redis():
    redis = await get_redis()

    # Check all keys matching a pattern
    keys = await redis.keys("session:*")
    logger.debug("Active sessions", extra={"count": len(keys)})

    # Check a specific key
    value = await redis.get("session:user123")
    logger.debug("Session value", extra={"key": "session:user123", "value": str(value)[:200]})

    # Check TTL
    ttl = await redis.ttl("session:user123")
    logger.debug("Session TTL", extra={"ttl_seconds": ttl})
```

## Async debugging

```python
# Detect blocking calls in async context
import asyncio
from app.core.logging import get_logger

logger = get_logger(__name__)

def detect_blocking():
    """Log if any blocking call takes too long."""
    import time
    start = time.perf_counter()
    # ... your code here
    duration = time.perf_counter() - start
    if duration > 0.1:
        logger.warning("Slow operation detected", extra={"duration_ms": round(duration * 1000)})
```

```python
# Use asyncio debug mode
import asyncio
asyncio.run(main(), debug=True)
```

## Common issues and fixes

### 1. Connection pool exhaustion

```python
# Problem: "QueuePool limit of size X overflow Y reached"
# Fix: Check for missing session.close() or too many concurrent requests

# Add pool debugging
engine = create_async_engine(
    settings.database_url,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600,
    pool_pre_ping=True,
)

# Check pool status
pool = engine.pool
print(f"Pool size: {pool.size()}")
print(f"Checked out: {pool.checkedout()}")
print(f"Overflow: {pool.overflow()}")
```

### 2. Memory leaks

```python
# Track memory usage
import tracemalloc
tracemalloc.start()

# ... your code ...

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

print("[ Top 10 memory consumers ]")
for stat in top_stats[:10]:
    print(stat)
```

### 3. Slow queries

```python
# Add query timing middleware
import time
from sqlalchemy import event

@event.listens_for(engine.sync_engine, "before_cursor_execute")
def log_query_time(conn, cursor, statement, parameters, context, executemany):
    conn.info.setdefault("query_start_time", []).append(time.perf_counter())

@event.listens_for(engine.sync_engine, "after_cursor_execute")
def log_query_duration(conn, cursor, statement, parameters, context, executemany):
    start = conn.info["query_start_time"].pop(-1)
    duration = time.perf_counter() - start
    if duration > 0.5:  # Log slow queries (>500ms)
        logger.warning("Slow query", extra={"duration_ms": round(duration * 1000), "query": statement[:200]})
```

### 4. Race conditions

```python
# Use database locks for critical sections
from sqlalchemy import select, with_for_update

async def atomic_update(session: AsyncSession, user_id: str):
    # Lock the row for update
    result = await session.execute(
        select(User).where(User.id == user_id).with_for_update()
    )
    user = result.scalar_one()
    user.balance -= 100
    await session.commit()
```

### 5. Background job failures

```python
# Debug failed arq jobs
# Check scheduler_log table
SELECT * FROM scheduler_log
WHERE status = 'failed'
ORDER BY started_at DESC
LIMIT 10;

# Add retry debugging
from app.workers.logging import get_worker_logger

logger = get_worker_logger(__name__)

async def my_job(ctx: dict):
    logger.info("Job started", extra={"try": ctx.get("job_try")})
    try:
        # ... job logic
        logger.info("Job completed")
    except Exception as e:
        logger.error(
            "Job failed",
            extra={"error": str(e), "try": ctx.get("job_try")},
            exc_info=True,
        )
        raise  # Re-raise to trigger arq retry
```

## Debugging tools

```bash
# Install debugging tools
uv add --dev debugpy ipdb py-spy memray

# py-spy — sample running process
py-spy top --pid <PID>

# memray — memory profiler
uv run memray run -m app.main
uv run memray flamegraph output.bin
```

## DO NOT

- **Never** leave `debugpy.listen()` in production code — gate on `DEBUG` env var.
- **Never** use `print()` for debugging — use `get_logger(__name__).debug()`.
- **Never** commit debug code (breakpoints, temp logging) to production.
- **Never** disable SQLAlchemy connection pool debugging in production.
- **Never** ignore slow query warnings — investigate and optimize.
- **Never** skip `exc_info=True` when logging exceptions — you need the traceback.
- **Never** use `print()` to inspect request/response in production — use structured logging.
- **Never** run debugpy in production — it opens a network port.
