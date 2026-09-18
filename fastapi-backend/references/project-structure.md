# Project Structure & Idiomatic Python

## Bootstrapping with uv

```bash
uv init backend --python 3.12
cd backend
uv add fastapi "uvicorn[standard]" sqlalchemy alembic asyncpg redis duckdb arq \
       pydantic-settings pydantic[email] passlib[argon2] python-jose[cryptography] \
       httpx openai anthropic
uv add --dev ruff bandit import-linter pytest pytest-asyncio pytest-cov pre-commit \
       httpx-mocks factory-boy
```

- `requires-python = ">=3.12"` in `pyproject.toml`. uv resolves and pins a lockfile
  (`uv.lock`) — commit it.
- One `uv.lock` per deployable service. If this becomes a monorepo, use a uv **workspace**
  (`[tool.uv.workspace]` with `members = ["services/*"]`) rather than duplicating lockfiles.
- Day-to-day commands: `uv run uvicorn app.main:app --reload`, `uv run pytest`,
  `uv run alembic upgrade head`, `uv run arq app.workers.arq_worker.WorkerSettings`.

## Vertical slice layout

Organize by **feature/domain**, not by technical layer. Each slice owns its own router,
schemas, service logic, and data access — a feature should be understandable (and deletable)
by reading one directory.

```
backend/
├── pyproject.toml
├── uv.lock
├── alembic.ini
├── migrations/                     # Alembic migration scripts
├── app/
│   ├── main.py                     # app factory, mounts routers, lifespan
│   ├── core/                       # cross-cutting, framework-y concerns only
│   │   ├── config.py               # pydantic-settings Settings
│   │   ├── security.py             # jwt/password helpers
│   │   ├── logging.py
│   │   ├── db.py                   # engine/session factories (postgres, duckdb)
│   │   └── redis.py                # redis pool
│   ├── shared/                     # genuinely shared domain types (kept small)
│   │   ├── errors.py               # base exception types + FastAPI exception handlers
│   │   └── pagination.py
│   ├── features/
│   │   ├── users/
│   │   │   ├── router.py           # FastAPI APIRouter, HTTP concerns only
│   │   │   ├── schemas.py          # Pydantic request/response models
│   │   │   ├── models.py           # SQLAlchemy ORM models for this slice
│   │   │   ├── repository.py       # DB access for this slice only
│   │   │   ├── service.py          # business logic, orchestrates repository/other services
│   │   │   └── deps.py             # FastAPI Depends() for this slice
│   │   ├── auth/
│   │   │   └── ...                 # same shape
│   │   └── billing/
│   │       └── ...
│   └── workers/
│       ├── arq_worker.py           # WorkerSettings, cron_jobs
│       └── jobs/
│           ├── send_email.py
│           └── ...
└── tests/
    ├── features/<mirror the slice you're testing>/
    └── conftest.py
```

Rules of thumb:
- A feature slice may import from `core/` and `shared/`, never directly from another feature's
  internals (`features/billing/service.py` importing `features/users/repository.py` is a smell).
  If two slices need to talk, do it through a narrow public interface re-exported from the
  slice's `__init__.py`, or extract the shared concept into `shared/`.
- `import-linter` (see `tooling-quality.md`) enforces this boundary mechanically — don't rely on
  reviewers noticing.
- Keep `router.py` thin: parse/validate input (Pydantic does this), call `service.py`, return.
  No SQL, no business rules in the router.
- `service.py` is where business logic and multi-repository/multi-slice orchestration lives.
  `repository.py` only knows how to talk to the database for its own slice's models.

## Idiomatic Python 3.12 conventions

- **Type everything** that's part of a public surface (function signatures, dataclass/Pydantic
  fields). Use the built-in generics (`list[int]`, `dict[str, Foo]`) and `X | Y` unions —
  never `typing.List`/`typing.Optional` in new code.
- **`match`/`case`** for dispatching on shape (e.g. handling a tagged union of job results,
  parsing webhook payloads) instead of `if isinstance` chains.
- **`type` statement** for reusable aliases: `type UserId = int`.
- **Dataclasses vs. Pydantic**: use Pydantic models for anything crossing a boundary (HTTP
  request/response, LLM structured output, job payloads that get serialized). Use plain
  `@dataclass(slots=True, frozen=True)` for internal-only value objects where you don't need
  validation or serialization.
- **`Protocol` for dependency inversion**, not ABCs, when you just need structural typing (e.g.
  the AI provider interface in `ai-integration.md`, or swappable repository implementations in
  tests).
- **Prefer composition over inheritance**; prefer functions over classes when there's no state
  to hold. A `service.py` full of module-level `async def` functions is fine and often clearer
  than a `UserService` class with no instance state.
- **`pathlib.Path`**, never raw string path concatenation.
- **Structured logging** (see `core/logging.py`) with contextual fields, not f-string log
  messages — makes both local debugging and log aggregation in prod usable.
- **Exceptions**: define a small hierarchy in `shared/errors.py` (e.g. `AppError` →
  `NotFoundError`, `ConflictError`, `ValidationError`) and map them to HTTP responses in one
  place via FastAPI exception handlers — don't scatter `HTTPException(status_code=...)` through
  service code.

## DO NOT

- **Never** use horizontal layer folders (`models/`, `views/`, `services/`, `repositories/` at the top level). Always vertical slices by feature.
- **Never** import from another feature's internals (`features/billing/service.py` importing `features/users/repository.py`). Use `shared/` or a narrow public interface.
- **Never** put business logic in `router.py`. Routers parse input, call service, return output — nothing else.
- **Never** put SQL queries in `router.py`. All DB access goes through `repository.py`.
- **Never** use `typing.List`, `typing.Optional`, `typing.Dict` — use `list`, `X | None`, `dict` (Python 3.12+).
- **Never** use ABCs for dependency inversion — use `Protocol`.
- **Never** use string path concatenation — use `pathlib.Path`.
- **Never** scatter `HTTPException(status_code=...)` through service code — use `shared/errors.py` and exception handlers.
- **Never** use `@dataclass` for anything crossing a boundary (HTTP, LLM, job payloads) — use Pydantic.
- **Never** skip type annotations on public surfaces (function signatures, Pydantic fields).
- **Never** use f-strings for log messages — use structured logging with contextual fields.
- **Never** create a class with no instance state — use module-level functions instead.
- **Never** hand-edit `pyproject.toml` dependency lists — always use `uv add`.
- **Never** use pip, poetry, or pipenv — uv is the only package manager.
- **Never** commit without running quality gates (ruff, bandit, import-linter, tests).
