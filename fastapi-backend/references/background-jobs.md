# Background Jobs: arq + Persisted Execution History

**arq** is the right default here: async-native (shares the same event loop style as the rest
of this stack), Redis-backed, low operational overhead, and sufficient for the vast majority of
FastAPI backend job needs (emails, webhooks, scheduled/cron work, LLM calls that shouldn't block
a request). Reach for Celery only if you need multi-language workers, complex routing
topologies, or features arq genuinely lacks — don't default to it.

## Worker setup

### Option 1: Decorator + explicit imports (recommended)

```python
# app/workers/jobs/registry.py
from typing import Callable

_registry: dict[str, Callable] = {}

def register_job(func: Callable) -> Callable:
    """Decorator to auto-register a job function."""
    _registry[func.__name__] = func
    return func

def get_all_jobs() -> list[Callable]:
    return list(_registry.values())
```

```python
# app/workers/jobs/send_email.py
from app.workers.jobs.registry import register_job

@register_job
async def send_email(ctx: dict, *, to: str, template: str) -> dict:
    http: httpx.AsyncClient = ctx["http"]
    resp = await http.post(
        "https://api.emailprovider.com/send",
        json={"to": to, "template": template},
        timeout=30,
    )
    resp.raise_for_status()
    return {"sent": True, "to": to}
```

```python
# app/workers/arq_worker.py
from arq import cron
from arq.connections import RedisSettings
from app.core.config import settings
from app.workers.jobs.registry import get_all_jobs
from app.workers.history import record_job_start, record_job_result

# Explicit imports — IDE autocomplete, type safety, easy to debug
from app.workers.jobs.send_email import send_email
from app.workers.jobs.nightly_report import nightly_report
from app.workers.jobs.process_webhook import process_webhook
from app.workers.jobs.cleanup_expired import cleanup_expired

async def startup(ctx: dict) -> None:
    ctx["db_engine"] = engine
    ctx["http"] = httpx.AsyncClient()

async def shutdown(ctx: dict) -> None:
    await ctx["http"].aclose()
    await ctx["db_engine"].dispose()

async def on_job_start(ctx: dict) -> None:
    await record_job_start(ctx)

async def after_job_end(ctx: dict) -> None:
    await record_job_result(ctx)

class WorkerSettings:
    redis_settings = RedisSettings.from_dsn(settings.redis_url)
    functions = [send_email, nightly_report, process_webhook, cleanup_expired]
    cron_jobs = [
        cron(nightly_report, hour=2, minute=0, run_at_startup=False),
    ]
    on_startup = startup
    on_shutdown = shutdown
    on_job_start = on_job_start
    after_job_end = after_job_end
    max_jobs = 20
    job_timeout = 300
    max_tries = 3
```

### Option 2: Manual registration (simpler projects)

```python
# app/workers/arq_worker.py
from arq import cron
from arq.connections import RedisSettings
from app.core.config import settings
from app.workers.jobs.send_email import send_email
from app.workers.jobs.nightly_report import nightly_report
from app.workers.history import record_job_start, record_job_result

class WorkerSettings:
    redis_settings = RedisSettings.from_dsn(settings.redis_url)
    functions = [send_email, nightly_report]
    cron_jobs = [
        cron(nightly_report, hour=2, minute=0, run_at_startup=False),
    ]
    # ... lifecycle hooks
```

Run it with `uv run arq app.workers.arq_worker.WorkerSettings` as its own process/container
(never inside the API's `uvicorn` process) — see `deployment.md` for docker-compose wiring.

### Job functions with automatic logging

```python
# app/workers/jobs/send_email.py
from app.workers.jobs.registry import register_job
from app.core.logging import get_logger

logger = get_logger(__name__)

@register_job
async def send_email(ctx: dict, *, to: str, template: str, subject: str, context: dict | None = None) -> dict:
    """Send an email via external provider."""
    logger.info(
        "Job started: send_email",
        extra={"to": to, "template": template, "job_id": ctx.get("job_id")},
    )
    
    try:
        http: httpx.AsyncClient = ctx["http"]
        resp = await http.post(
            "https://api.emailprovider.com/send",
            json={"to": to, "template": template, "subject": subject, "context": context or {}},
            timeout=30,
        )
        resp.raise_for_status()
        
        logger.info(
            "Job completed: send_email",
            extra={"to": to, "template": template, "status": "success"},
        )
        return {"sent": True, "to": to}
    
    except Exception as e:
        logger.error(
            "Job failed: send_email",
            extra={"to": to, "template": template, "error": str(e)},
            exc_info=True,
        )
        raise
```

### Worker with automatic logging hooks

```python
# app/workers/arq_worker.py
from arq import cron
from arq.connections import RedisSettings
from app.core.config import settings
from app.core.logging import get_logger
from app.workers.jobs.send_email import send_email
from app.workers.jobs.nightly_report import nightly_report
from app.workers.history import record_job_start, record_job_result

logger = get_logger(__name__)

async def startup(ctx: dict) -> None:
    ctx["db_engine"] = engine
    ctx["http"] = httpx.AsyncClient()
    logger.info("Worker started")

async def shutdown(ctx: dict) -> None:
    await ctx["http"].aclose()
    await ctx["db_engine"].dispose()
    logger.info("Worker shutdown")

async def on_job_start(ctx: dict) -> None:
    logger.info(
        "Job started",
        extra={
            "job_id": ctx.get("job_id"),
            "job_name": ctx.get("job_name"),
            "job_try": ctx.get("job_try"),
        }
    )
    await record_job_start(ctx)

async def after_job_end(ctx: dict) -> None:
    status = "failed" if ctx.get("job_result_failed") else "success"
    logger.info(
        "Job finished",
        extra={
            "job_id": ctx.get("job_id"),
            "job_name": ctx.get("job_name"),
            "status": status,
        }
    )
    await record_job_result(ctx)

class WorkerSettings:
    redis_settings = RedisSettings.from_dsn(settings.redis_url)
    functions = [send_email, nightly_report]
    cron_jobs = [
        cron(nightly_report, hour=2, minute=0, run_at_startup=False),
    ]
    on_startup = startup
    on_shutdown = shutdown
    on_job_start = on_job_start
    after_job_end = after_job_end
    max_jobs = 20
    job_timeout = 300
    max_tries = 3
```

arq stores results in Redis, but Redis results expire and aren't queryable/joinable the way a
table is — anything you want to audit, alert on, or show in an admin view needs its own
Postgres table, written from the job lifecycle hooks above.

```python
# app/workers/history.py
class JobExecution(Base):
    __tablename__ = "job_executions"
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    job_id: Mapped[str] = mapped_column(index=True)       # arq's ctx["job_id"]
    function_name: Mapped[str] = mapped_column(index=True)
    args: Mapped[dict] = mapped_column(JSONB, default=dict)
    status: Mapped[str] = mapped_column(default="running")  # running|success|failed|retrying
    result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    error: Mapped[str | None] = mapped_column(nullable=True)
    attempt: Mapped[int] = mapped_column(default=1)
    started_at: Mapped[datetime] = mapped_column(server_default=func.now())
    finished_at: Mapped[datetime | None] = mapped_column(nullable=True)

async def record_job_start(ctx: dict) -> None:
    async with AsyncSessionLocal() as session:
        session.add(JobExecution(
            job_id=ctx["job_id"],
            function_name=ctx["job_try"] and ctx.get("job_name", "unknown"),
            args=_safe_args(ctx.get("job_args"), ctx.get("job_kwargs")),
            attempt=ctx.get("job_try", 1),
        ))
        await session.commit()

async def record_job_result(ctx: dict) -> None:
    async with AsyncSessionLocal() as session:
        execution = await session.get(JobExecution, ...)  # look up by job_id + attempt
        execution.status = "failed" if ctx.get("job_result_failed") else "success"
        execution.result = _safe_result(ctx.get("job_result"))
        execution.finished_at = func.now()
        await session.commit()
```

- `_safe_args`/`_safe_result` should redact anything sensitive (tokens, PII) before it's
  persisted — job history is often over-shared internally, treat it like logs.
- Index `function_name` + `started_at` for the common "recent runs of job X" query, and
  `status` for "show me failures" dashboards/alerts.
- For cron jobs specifically, this table doubles as your audit trail that the schedule is
  actually firing — alert if `nightly_report` has no `success` row for >26h, etc.
- Keep the history table pruned/partitioned if job volume is high (a monthly partition or a
  separate cron job that archives/deletes rows older than N days) — it will grow forever
  otherwise.

## Enqueuing from the API

```python
# app/workers/enqueue.py
from typing import Callable
from app.core.redis import get_arq_redis
from app.core.logging import get_logger
from app.workers.jobs import JOB_QUEUES

logger = get_logger(__name__)

async def enqueue_job(
    method: str | Callable,
    *,
    queue: str | None = None,
    timeout: int | None = None,
    on_success: Callable | None = None,
    on_failure: Callable | None = None,
    at_front: bool = False,
    job_id: str | None = None,
    deduplicate: bool = False,
    **kwargs,
) -> str | None:
    """
    Enqueue a background job.

    Args:
        method: Job function or dotted path string
        queue: Queue override (short/default/long). If None, uses JOB_QUEUES mapping.
        timeout: Job timeout in seconds (overrides queue default)
        on_success: Success callback function
        on_failure: Failure callback function
        at_front: Enqueue at front of queue
        job_id: Unique job ID for deduplication
        deduplicate: Don't re-queue if already queued
        **kwargs: Arguments passed to the job function

    Returns:
        Job ID or None
    """
    redis = await get_arq_redis()

    # Resolve function name
    if callable(method):
        job_name = f"{method.__module__}.{method.__qualname__}"
    else:
        job_name = method

    # Use provided queue or fall back to job's default
    queue = queue or JOB_QUEUES.get(method, "default")

    # Deduplication check
    if deduplicate:
        if not job_id:
            raise ValueError("job_id is required for deduplication")
        existing = await redis.get(f"arq:job:{job_id}")
        if existing:
            logger.info(f"Job {job_id} already queued, skipping")
            return job_id

    # Log enqueue
    logger.info(
        "Enqueueing job",
        extra={
            "job_name": job_name,
            "queue": queue,
            "timeout": timeout,
            "kwargs": kwargs,
        }
    )

    # Build enqueue kwargs
    enqueue_kwargs = {"_queue_name": queue}
    if timeout:
        enqueue_kwargs["_job_timeout"] = timeout
    if job_id:
        enqueue_kwargs["_job_id"] = job_id

    # Enqueue to arq
    return await redis.enqueue_job(job_name, **enqueue_kwargs, **kwargs)
```

### Usage

```python
# app/features/auth/service.py
from app.workers.enqueue import enqueue_job

class AuthService:
    async def register(self, data: UserCreate) -> User:
        user = await self._repo.create(data)
        
        await enqueue_job(
            "app.workers.jobs.send_email",
            to=user.email,
            template="welcome",
            subject="Welcome!",
        )
        
        return user
```

```python
# Override queue
await enqueue_job("send_email", queue="long", to="user@example.com")

# Override timeout
await enqueue_job("nightly_report", timeout=7200)

# Deduplication
await enqueue_job("sync_data", job_id="sync-daily", deduplicate=True)
```

Never do the actual work inline in the request/response cycle if it involves external I/O
(email providers, LLM calls, file processing) — enqueue and return immediately; let the worker
own retries/backoff via arq's `max_tries`/`retry_delay`.

## Multiple queues and workers

### Queue configuration

```python
# app/workers/queues.py
from dataclasses import dataclass

@dataclass
class QueueConfig:
    name: str
    timeout: int
    max_jobs: int

QUEUES = {
    "short": QueueConfig(name="short", timeout=60, max_jobs=10),
    "default": QueueConfig(name="default", timeout=300, max_jobs=20),
    "long": QueueConfig(name="long", timeout=3600, max_jobs=5),
}
```

### Job registry with queue assignment

```python
# app/workers/jobs/__init__.py
from app.workers.jobs.send_email import send_email
from app.workers.jobs.nightly_report import nightly_report
from app.workers.jobs.process_webhook import process_webhook
from app.workers.jobs.cleanup_expired import cleanup_expired

# Map jobs to queues
JOB_QUEUES = {
    send_email: "short",
    process_webhook: "short",
    nightly_report: "long",
    cleanup_expired: "long",
}

ALL_JOBS = list(JOB_QUEUES.keys())
```

### Single worker with queue filtering

```python
# app/workers/arq_worker.py
from arq import cron
from arq.connections import RedisSettings
from app.core.config import settings
from app.workers.queues import QUEUES
from app.workers.jobs import JOB_QUEUES
from app.workers.history import record_job_start, record_job_result

def create_worker_settings(queue_name: str):
    """Create worker settings for a specific queue."""
    queue = QUEUES[queue_name]
    jobs = [job for job, q in JOB_QUEUES.items() if q == queue_name]
    
    class WorkerSettings:
        redis_settings = RedisSettings.from_dsn(settings.redis_url)
        functions = jobs
        queue_name = queue_name
        max_jobs = queue.max_jobs
        job_timeout = queue.timeout
        on_job_start = record_job_start
        after_job_end = record_job_result
    
    return WorkerSettings

ShortWorker = create_worker_settings("short")
DefaultWorker = create_worker_settings("default")
LongWorker = create_worker_settings("long")
```

### Enqueue with queue selection

```python
# app/workers/enqueue.py
from typing import Callable
from app.core.redis import get_arq_redis
from app.core.logging import get_logger
from app.workers.jobs import JOB_QUEUES

logger = get_logger(__name__)

async def enqueue_job(
    method: str | Callable,
    *,
    queue: str | None = None,
    timeout: int | None = None,
    on_success: Callable | None = None,
    on_failure: Callable | None = None,
    at_front: bool = False,
    job_id: str | None = None,
    deduplicate: bool = False,
    **kwargs,
) -> str | None:
    """
    Enqueue a background job.

    Args:
        method: Job function or dotted path string
        queue: Queue override (short/default/long). If None, uses JOB_QUEUES mapping.
        timeout: Job timeout in seconds (overrides queue default)
        on_success: Success callback function
        on_failure: Failure callback function
        at_front: Enqueue at front of queue
        job_id: Unique job ID for deduplication
        deduplicate: Don't re-queue if already queued
        **kwargs: Arguments passed to the job function

    Returns:
        Job ID or None
    """
    redis = await get_arq_redis()

    # Resolve function name
    if callable(method):
        job_name = f"{method.__module__}.{method.__qualname__}"
    else:
        job_name = method

    # Use provided queue or fall back to job's default
    queue = queue or JOB_QUEUES.get(method, "default")

    # Deduplication check
    if deduplicate:
        if not job_id:
            raise ValueError("job_id is required for deduplication")
        existing = await redis.get(f"arq:job:{job_id}")
        if existing:
            logger.info(f"Job {job_id} already queued, skipping")
            return job_id

    # Log enqueue
    logger.info(
        "Enqueueing job",
        extra={
            "job_name": job_name,
            "queue": queue,
            "timeout": timeout,
            "kwargs": kwargs,
        }
    )

    # Build enqueue kwargs
    enqueue_kwargs = {"_queue_name": queue}
    if timeout:
        enqueue_kwargs["_job_timeout"] = timeout
    if job_id:
        enqueue_kwargs["_job_id"] = job_id

    # Enqueue to arq
    return await redis.enqueue_job(job_name, **enqueue_kwargs, **kwargs)
```

```python
# API service
from app.workers.enqueue import enqueue_job

# Auto-select queue from JOB_QUEUES mapping
await enqueue_job("send_email", to="user@example.com")  # → "short" queue
await enqueue_job("nightly_report")  # → "long" queue

# Override queue
await enqueue_job("send_email", queue="long", to="user@example.com")  # → "long" queue

# Override timeout
await enqueue_job("nightly_report", timeout=7200)  # 2 hours timeout
await enqueue_job("send_email", timeout=10)  # 10 seconds timeout

# Queue + timeout override
await enqueue_job("nightly_report", queue="short", timeout=60)  # → "short" queue, 60s timeout
```

### Single worker (all queues)

```python
# app/workers/arq_worker.py
from arq import cron
from arq.connections import RedisSettings
from app.core.config import settings
from app.workers.jobs import ALL_JOBS
from app.workers.history import record_job_start, record_job_result

class WorkerSettings:
    redis_settings = RedisSettings.from_dsn(settings.redis_url)
    functions = ALL_JOBS  # Process all queues
    cron_jobs = [
        cron(nightly_report, hour=2, minute=0, run_at_startup=False),
    ]
    on_job_start = record_job_start
    after_job_end = record_job_result
    max_jobs = 20
    job_timeout = 3600  # Max timeout across all queues
    max_tries = 3
```

```bash
# Single worker processes all queues
uv run arq app.workers.arq_worker.WorkerSettings
```

### Multiple workers (per queue)

```python
# app/workers/arq_worker.py
from app.workers.queues import QUEUES
from app.workers.jobs import JOB_QUEUES

def create_worker_settings(queue_name: str):
    queue = QUEUES[queue_name]
    jobs = [job for job, q in JOB_QUEUES.items() if q == queue_name]
    
    class WorkerSettings:
        redis_settings = RedisSettings.from_dsn(settings.redis_url)
        functions = jobs
        queue_name = queue_name
        max_jobs = queue.max_jobs
        job_timeout = queue.timeout
        on_job_start = record_job_start
        after_job_end = record_job_result
    
    return WorkerSettings

ShortWorker = create_worker_settings("short")
LongWorker = create_worker_settings("long")
```

```bash
# Separate workers per queue
uv run arq app.workers.arq_worker.ShortWorker &
uv run arq app.workers.arq_worker.LongWorker &
```

### Docker compose

```yaml
services:
  short-worker:
    build: .
    command: arq app.workers.arq_worker.ShortWorker
    deploy:
      replicas: 3

  default-worker:
    build: .
    command: arq app.workers.arq_worker.DefaultWorker
    deploy:
      replicas: 2

  long-worker:
    build: .
    command: arq app.workers.arq_worker.LongWorker
    deploy:
      replicas: 1
```

## DO NOT

- **Never** run arq worker inside the uvicorn process — it must be a separate process/container.
- **Never** do actual work inline in request/response for external I/O — enqueue and return immediately.
- **Never** rely only on arq's Redis results for audit/history — persist to Postgres `job_executions` table.
- **Never** log raw job args/results with sensitive data — redact before persisting.
- **Never** skip `on_job_start`/`after_job_end` hooks — they're your audit trail.
- **Never** use `max_tries=1` for jobs that call external services — transient failures happen.
- **Never** set `job_timeout` too high — a hung job holds a worker slot forever.
- **Never** create a new `httpx.AsyncClient()` per job — reuse one from `ctx["http"]`.
- **Never** create a new database session per job without proper cleanup — use `AsyncSessionLocal()`.
- **Never** ignore failed jobs — alert on `status="failed"` in `job_executions`.
- **Never** use `run_at_startup=True` for cron jobs in production — they should run on schedule only.
- **Never** store job results in memory — they'll be lost on restart.
- **Never** use Celery unless you genuinely need multi-language workers or complex routing topologies.
- **Never** use importlib for runtime job discovery — use explicit imports for IDE support and type safety.
