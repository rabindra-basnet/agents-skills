# Database Layer

## Choosing which store for what

| Store | Use it for |
|---|---|
| **Postgres** | System of record — all transactional/relational data, including background-job execution history (see `background-jobs.md`) |
| **Redis** | arq's broker + result cache, rate-limiting counters, auth/session/blacklist state, short-TTL caches |
| **DuckDB** | Embedded, in-process OLAP: ad-hoc analytics over large read-heavy datasets, exporting/querying Parquet/CSV, reporting queries you don't want competing with Postgres's transactional load. Not a replacement for Postgres — feed it from Postgres exports or object storage, don't write your primary data model into it. |

Don't reach for DuckDB by default — only pull it in when a feature genuinely needs fast
columnar analytics (dashboards, exports, large aggregations) separate from the OLTP path.

## SQLAlchemy 2.0 (async)

```python
# app/core/db.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(settings.database_url, pool_size=10, pool_pre_ping=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

```python
# app/features/users/models.py
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase
from datetime import datetime
import uuid

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    hashed_password: Mapped[str]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

Use the 2.0-style declarative `Mapped[...]`/`mapped_column` annotations (not the legacy
`Column(...)` style) — it's fully type-checked and is what current SQLAlchemy docs lead with.

### Repository pattern, per slice

```python
# app/features/users/repository.py
class UserRepository:
    def __init__(self, session: AsyncSession):
        self._session = session

    async def get_by_email(self, email: str) -> User | None:
        result = await self._session.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def create(self, user: User) -> User:
        self._session.add(user)
        await self._session.flush()
        return user
```

Repositories only compose queries for their own slice's tables. Commit boundaries live in
`service.py` (one transaction per use-case), not inside the repository — makes multi-step
service operations atomic without repositories needing to know about each other.

## Alembic

```bash
uv run alembic init migrations
```

```python
# migrations/env.py — key changes from the default template
from app.core.db import engine
from app.features.users.models import Base as UsersBase
# import every slice's Base/metadata so autogenerate sees all tables
target_metadata = UsersBase.metadata  # or a combined metadata object if you split Base per slice
```

- Use `alembic revision --autogenerate -m "add users table"`, then **always read the generated
  script** before applying — autogenerate misses some constraints/renames.
- One logical change per migration; never edit a migration that's already been applied in a
  shared environment — write a new one.
- Run migrations as an explicit step in deploy (`uv run alembic upgrade head`), not
  automatically on app boot, so a bad migration doesn't take down every replica at once.
- If slices each define their own `DeclarativeBase`, keep a single combined metadata object
  (or a single shared `Base` re-exported from `app/core/db.py`) that Alembic's `env.py` points
  at — don't let autogenerate silently miss a slice's tables.

## Redis usage conventions

- One connection pool per process, created in the app lifespan (`app/core/redis.py`), reused —
  never create a new client per request.
- Namespace keys by concern: `session:{user_id}`, `ratelimit:{route}:{key}`,
  `arq:...` (arq manages its own namespace). Set explicit TTLs on anything cache-like.

## DuckDB usage conventions

- DuckDB is embedded (no separate server) — open it against a file path or `:memory:` inside
  the process/job that needs it, close it when done.
- Typical pattern: an arq job periodically exports relevant Postgres tables (or reads from
  object storage) into Parquet, and an analytics/reporting endpoint queries that Parquet via
  DuckDB (`duckdb.sql("SELECT ... FROM 'reports/2026-01.parquet'")`) rather than hitting
  Postgres directly for heavy aggregations.
