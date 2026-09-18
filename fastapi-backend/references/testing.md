# Testing

Unit tests, integration tests, fixtures, mocking, and coverage for FastAPI applications.

## Test structure

```
tests/
├── conftest.py                    # Shared fixtures
├── features/
│   ├── users/
│   │   ├── test_service.py        # Unit tests
│   │   └── test_integration.py    # Integration tests
│   └── auth/
│       └── test_service.py
└── e2e/
    └── test_api.py                # End-to-end API tests
```

## Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "integration: marks tests as integration (deselect with '-m \"not integration\"')",
]

[tool.coverage.run]
source = ["app"]
omit = ["tests/*", "migrations/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
```

## Fixtures

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from app.main import create_app
from app.core.db import Base, get_session
from app.core.config import settings

@pytest.fixture
async def db_engine():
    engine = create_async_engine(settings.test_database_url, pool_size=5)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest.fixture
async def db_session(db_engine):
    session_factory = async_sessionmaker(db_engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
        await session.rollback()

@pytest.fixture
async def client(db_engine):
    app = create_app()

    async def override_session():
        async with async_sessionmaker(db_engine, expire_on_commit=False)() as session:
            yield session

    app.dependency_overrides[get_session] = override_session

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac

@pytest.fixture
def sample_user():
    return {"email": "test@example.com", "password": "secret123"}
```

## Unit tests

```python
# tests/features/users/test_service.py
import pytest
from app.features.users.service import UserService
from app.features.users.schemas import UserCreate
from app.shared.errors import NotFoundError, ConflictError

@pytest.fixture
def service(db_session):
    return UserService(session=db_session)

async def test_create_user(service, db_session):
    data = UserCreate(email="new@example.com", password="secret123")
    user = await service.create_user(data)
    assert user.email == "new@example.com"
    assert user.id is not None

async def test_create_user_duplicate_raises(service, db_session):
    data = UserCreate(email="dup@example.com", password="secret123")
    await service.create_user(data)
    with pytest.raises(ConflictError):
        await service.create_user(data)

async def test_get_user_not_found(service, db_session):
    with pytest.raises(NotFoundError):
        await service.get_user("nonexistent-id")
```

## Integration tests

```python
# tests/features/users/test_integration.py
import pytest

@pytest.mark.integration
async def test_create_user_api(client, sample_user):
    response = await client.post("/api/v1/users/", json=sample_user)
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == sample_user["email"]
    assert "id" in data

@pytest.mark.integration
async def test_get_user_api(client, sample_user):
    # Create first
    create_resp = await client.post("/api/v1/users/", json=sample_user)
    user_id = create_resp.json()["id"]

    # Get
    response = await client.get(f"/api/v1/users/{user_id}")
    assert response.status_code == 200
    assert response.json()["email"] == sample_user["email"]
```

## Mocking external services

```python
# tests/features/auth/test_email.py
from unittest.mock import AsyncMock, patch
from app.workers.jobs.send_email import send_email

@pytest.fixture
def mock_email_provider():
    with patch("app.core.email.factory.get_email_provider") as mock:
        provider = AsyncMock()
        provider.send.return_value = {"status": "sent"}
        mock.return_value = provider
        yield provider

async def test_send_email(mock_email_provider):
    result = await send_email({}, to="test@example.com", subject="Hi", html="<p>Hello</p>")
    assert result["status"] == "sent"
    mock_email_provider.send.assert_called_once()

async def test_send_email_failure(mock_email_provider):
    mock_email_provider.send.side_effect = Exception("Provider down")
    with pytest.raises(Exception, match="Provider down"):
        await send_email({}, to="test@example.com", subject="Hi", html="<p>Hello</p>")
```

## Test factories

```python
# tests/factories.py
import factory
from app.features.users.models import User
from app.core.db import AsyncSessionLocal

class UserFactory:
    async def create(self, **kwargs):
        async with AsyncSessionLocal() as session:
            user = User(
                email=kwargs.get("email", factory.Faker("email")),
                hashed_password=kwargs.get("hashed_password", "hashed_secret"),
            )
            session.add(user)
            await session.commit()
            await session.refresh(user)
            return user
```

## Running tests

```bash
# All tests
uv run pytest

# Unit tests only (skip integration)
uv run pytest -m "not integration"

# With coverage
uv run pytest --cov=app --cov-report=html

# Specific file
uv run pytest tests/features/users/test_service.py -v

# Stop on first failure
uv run pytest -x
```

## DO NOT

- **Never** use real database in unit tests — use test database or in-memory SQLite.
- **Never** hit real external APIs in tests — mock them.
- **Never** skip async fixtures — use `@pytest.fixture` with `async def`.
- **Never** rely on test execution order — each test must be independent.
- **Never** commit test database state between tests — use transactions/rollbacks.
- **Never** skip coverage checks — enforce minimum threshold in CI.
- **Never** use `unittest.mock.patch` for database fixtures — use dependency overrides.
- **Never** hardcode test data — use factories or fixtures.
