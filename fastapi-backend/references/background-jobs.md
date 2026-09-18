# Background Jobs: arq + Persisted Execution History

**arq** is the right default here: async-native (shares the same event loop style as the rest
of this stack), Redis-backed, low operational overhead, and sufficient for the vast majority of
FastAPI backend job needs (emails, webhooks, scheduled/cron work, LLM calls that shouldn't block
a request). Reach for Celery only if you need multi-language workers, complex routing
topologies, or features arq genuinely lacks — don't default to it.

## Worker setup

```python
# app/workers/arq_worker.py
from arq import cron
from arq.connections import RedisSettings
from app.core.config import settings
from app.workers.jobs.send_email import send_email
from app.workers.jobs.nightly_report import nightly_report
from app.workers.history import record_job_start, record_job_result

async def startup(ctx: dict) -> None:
    ctx["db_engine"] = engine          # reuse the same async engine as the API process
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

Run it with `uv run arq app.workers.arq_worker.WorkerSettings` as its own process/container
(never inside the API's `uvicorn` process) — see `deployment.md` for docker-compose wiring.

## Persisting job execution history to the database

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
# inside a service.py
redis = await get_arq_redis(app.state)
await redis.enqueue_job("send_email", to=user.email, template="welcome")
```

Never do the actual work inline in the request/response cycle if it involves external I/O
(email providers, LLM calls, file processing) — enqueue and return immediately; let the worker
own retries/backoff via arq's `max_tries`/`retry_delay`.

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
