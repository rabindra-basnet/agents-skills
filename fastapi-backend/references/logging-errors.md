# Logging & Exception Handling

## Core logging (Frappe-style)

```python
# app/core/logging.py
import logging
import logging.handlers
import json
import sys
import traceback
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

LOG_DIR = Path("logs")
LOG_DIR.mkdir(exist_ok=True)

SENSITIVE_FIELDS = {"password", "token", "secret", "api_key", "authorization", "credit_card"}


class JSONFormatter(logging.Formatter):
    """JSON formatter for structured logging (production)."""

    def format(self, record: logging.LogRecord) -> str:
        log_data: dict[str, Any] = {
            "timestamp": datetime.fromtimestamp(record.created, tz=timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }

        if hasattr(record, "extra_data"):
            log_data.update(record.extra_data)

        if record.exc_info and record.exc_info[1]:
            log_data["exception"] = {
                "type": type(record.exc_info[1]).__name__,
                "message": str(record.exc_info[1]),
                "traceback": traceback.format_exception(*record.exc_info),
            }

        log_data = self._redact_sensitive(log_data)
        return json.dumps(log_data, default=str)

    def _redact_sensitive(self, data: Any) -> Any:
        if isinstance(data, dict):
            return {
                k: "[REDACTED]" if k.lower() in SENSITIVE_FIELDS else self._redact_sensitive(v)
                for k, v in data.items()
            }
        elif isinstance(data, list):
            return [self._redact_sensitive(item) for item in data]
        return data


class TextFormatter(logging.Formatter):
    """Human-readable formatter for development."""

    def format(self, record: logging.LogRecord) -> str:
        timestamp = datetime.fromtimestamp(record.created).strftime("%Y-%m-%d %H:%M:%S")
        return f"{timestamp} | {record.levelname:8} | {record.name} | {record.getMessage()}"


def setup_logging(log_level: str = "INFO", json_output: bool = False) -> None:
    root = logging.getLogger()
    root.setLevel(getattr(logging, log_level.upper()))
    root.handlers.clear()

    formatter = JSONFormatter() if json_output else TextFormatter()

    # Console
    console = logging.StreamHandler(sys.stdout)
    console.setLevel(logging.INFO)
    console.setFormatter(formatter)
    root.addHandler(console)

    # App log — daily rotation, 30 days
    app_handler = logging.handlers.TimedRotatingFileHandler(
        filename=LOG_DIR / "app.log",
        when="midnight",
        interval=1,
        backupCount=30,
        encoding="utf-8",
    )
    app_handler.setLevel(logging.INFO)
    app_handler.setFormatter(formatter)
    root.addHandler(app_handler)

    # Error log — daily rotation, 90 days
    error_handler = logging.handlers.TimedRotatingFileHandler(
        filename=LOG_DIR / "error.log",
        when="midnight",
        interval=1,
        backupCount=90,
        encoding="utf-8",
    )
    error_handler.setLevel(logging.ERROR)
    error_handler.setFormatter(formatter)
    root.addHandler(error_handler)


def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)
```

## Exception hierarchy

```python
# app/shared/errors.py
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from app.core.logging import get_logger

logger = get_logger(__name__)


class AppError(Exception):
    """Base application error."""
    status_code: int = 500
    detail: str = "Internal server error"

    def __init__(self, detail: str | None = None):
        self.detail = detail or self.detail
        super().__init__(self.detail)


class NotFoundError(AppError):
    status_code = 404
    detail = "Resource not found"


class ConflictError(AppError):
    status_code = 409
    detail = "Resource already exists"


class ValidationError(AppError):
    status_code = 422
    detail = "Validation failed"


class AuthenticationError(AppError):
    status_code = 401
    detail = "Authentication required"


class PermissionError(AppError):
    status_code = 403
    detail = "Permission denied"


class RateLimitError(AppError):
    status_code = 429
    detail = "Too many requests"


def register_exception_handlers(app) -> None:
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError):
        logger.error(
            "Application error",
            extra={
                "error_type": type(exc).__name__,
                "detail": exc.detail,
                "path": request.url.path,
                "method": request.method,
            },
        )
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": exc.detail},
        )

    @app.exception_handler(Exception)
    async def unhandled_error_handler(request: Request, exc: Exception):
        logger.critical(
            "Unhandled exception",
            extra={"path": request.url.path, "method": request.method},
            exc_info=True,
        )
        return JSONResponse(
            status_code=500,
            content={"error": "Internal server error"},
        )
```

## Usage across layers

### Service layer — raise domain errors, never HTTPException

```python
# app/features/users/service.py
from app.shared.errors import NotFoundError, ConflictError
from app.core.logging import get_logger

logger = get_logger(__name__)

class UserService:
    async def get_user(self, user_id: str) -> User:
        user = await self._repo.get_by_id(user_id)
        if not user:
            raise NotFoundError(f"User {user_id} not found")
        return user

    async def create_user(self, data: UserCreate) -> User:
        existing = await self._repo.get_by_email(data.email)
        if existing:
            raise ConflictError(f"User with email {data.email} already exists")

        user = await self._repo.create(data)
        logger.info("User created", extra={"user_id": str(user.id)})
        return user
```

### Router layer — thin, only HTTP concerns

```python
# app/features/users/router.py
from fastapi import APIRouter, Depends
from app.features.users.service import UserService
from app.features.users.schemas import UserCreate, UserResponse

router = APIRouter()

@router.post("/", response_model=UserResponse)
async def create_user(data: UserCreate, service: UserService = Depends()):
    return await service.create_user(data)
```

### Database layer — log slow queries, catch connection errors

```python
# app/core/db.py
import time
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from app.core.logging import get_logger

logger = get_logger(__name__)

engine = create_async_engine(settings.database_url, pool_size=10, pool_pre_ping=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session():
    async with AsyncSessionLocal() as session:
        yield session


class QueryLogger:
    """Log slow queries (>500ms)."""

    async def __aenter__(self):
        self.start = time.perf_counter()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        duration = time.perf_counter() - self.start
        if duration > 0.5:
            logger.warning("Slow query", extra={"duration_ms": round(duration * 1000)})
        if exc_type:
            logger.error("Database error", extra={"error": str(exc_val)})
        return False
```

### AI integration — log token usage, catch provider errors

```python
# app/core/ai/provider.py
from app.shared.errors import AppError
from app.core.logging import get_logger

logger = get_logger(__name__)

class AICallError(AppError):
    status_code = 502
    detail = "AI provider error"


async def complete_with_logging(provider, messages, *, model, **kwargs):
    logger.info("AI call started", extra={"model": model, "message_count": len(messages)})
    try:
        result = await provider.complete(messages, model=model, **kwargs)
        logger.info(
            "AI call completed",
            extra={
                "model": result.model,
                "input_tokens": result.input_tokens,
                "output_tokens": result.output_tokens,
            },
        )
        return result
    except Exception as e:
        logger.error("AI call failed", extra={"model": model, "error": str(e)})
        raise AICallError() from e
```

### Background jobs — log start/end, catch and record failures

```python
# app/workers/jobs/send_email.py
from app.workers.logging import get_worker_logger
from app.shared.errors import AppError

logger = get_worker_logger(__name__)

class EmailSendError(AppError):
    detail = "Failed to send email"

async def send_email(ctx: dict, *, to: str, template: str) -> dict:
    logger.info("Job started", extra={"job": "send_email", "to": to})
    try:
        # ... send email logic
        logger.info("Job completed", extra={"job": "send_email", "to": to})
        return {"sent": True}
    except Exception as e:
        logger.error("Job failed", extra={"job": "send_email", "to": to, "error": str(e)})
        raise EmailSendError() from e
```

### Middleware — request/response logging

```python
# app/middleware/logging.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from app.core.logging import get_logger

logger = get_logger(__name__)

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start

        logger.info(
            "Request",
            extra={
                "method": request.method,
                "path": request.url.path,
                "status": response.status_code,
                "duration_ms": round(duration * 1000),
                "client": request.client.host if request.client else None,
            },
        )
        return response
```

## Log rotation schedule

| File | Rotation | Retention | Purpose |
|------|----------|-----------|---------|
| `logs/app.log` | Daily | 30 days | API requests, app events |
| `logs/worker.log` | Daily | 30 days | Background job execution |
| `logs/error.log` | Daily | 90 days | Errors only (longer retention) |

## DO NOT

- **Never** scatter `HTTPException(status_code=...)` through service code — raise domain errors from `shared/errors.py`.
- **Never** return stack traces or internal error details to clients — generic messages only.
- **Never** use f-strings for log messages — use structured logging with contextual fields.
- **Never** log raw passwords, tokens, or full PII — redact sensitive fields.
- **Never** swallow exceptions silently — log them and re-raise or return error response.
- **Never** catch `Exception` broadly without logging — you'll lose visibility into failures.
- **Never** use `print()` for logging — use `get_logger()` consistently.
- **Never** skip `exc_info=True` when logging exceptions — you need the traceback.
- **Never** create loggers per request — create them at module level.
- **Never** log inside a loop without context — log once with aggregated data.
- **Never** use `logger.info(f"User {user_id}...")` — use `logger.info("User action", extra={"user_id": user_id})`.
- **Never** ignore `pool_pre_ping=True` — it prevents stale connection errors.
