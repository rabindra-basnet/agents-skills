# Observability

Structured logging, metrics, tracing, and alerting — know what your system is doing at all times.

## Structured logging

```python
# Already covered in logging-errors.md — use get_logger() everywhere
from app.core.logging import get_logger

logger = get_logger(__name__)

logger.info(
    "User action",
    extra={
        "user_id": user.id,
        "action": "login",
        "ip": request.client.host,
        "user_agent": request.headers.get("user-agent"),
    },
)
```

## Metrics with prometheus-client

```python
# app/core/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"],
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["method", "endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0],
)

# Business metrics
ACTIVE_USERS = Gauge("active_users_total", "Active users")
EMAILS_SENT = Counter("emails_sent_total", "Emails sent", ["status"])
```

```python
# app/middleware/metrics.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from app.core.metrics import REQUEST_COUNT, REQUEST_LATENCY

class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start

        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.url.path,
            status=response.status_code,
        ).inc()

        REQUEST_LATENCY.labels(
            method=request.method,
            endpoint=request.url.path,
        ).observe(duration)

        return response
```

## Health checks

```python
# app/features/health/router.py
from fastapi import APIRouter
from app.core.db import engine
from app.core.redis import get_redis

router = APIRouter()

@router.get("/health")
async def health():
    checks = {}

    # Database
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"

    # Redis
    try:
        redis = await get_redis()
        await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"

    status = "healthy" if all(v == "ok" for v in checks.values()) else "degraded"
    return {"status": status, "checks": checks}
```

## Alerting rules (Prometheus)

```yaml
# prometheus/alerts.yml
groups:
  - name: app-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning

      - alert: JobFailures
        expr: rate(scheduler_log_status_total{status="failed"}[1h]) > 0
        for: 1h
        labels:
          severity: warning
```

## DO NOT

- **Never** use `print()` for logging — use `get_logger()` consistently.
- **Never** log without context — always include relevant IDs (user_id, request_id, job_id).
- **Never** skip metrics for critical paths — you need to know when things break.
- **Never** expose `/metrics` endpoint without authentication — it's internal only.
- **Never** skip health checks — load balancers need them.
- **Never** alert on every error — set meaningful thresholds.
- **Never** use logs as the only observability tool — add metrics and traces.
- **Never** skip request ID propagation — it's how you correlate logs across services.
