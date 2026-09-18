# Documentation

API documentation, docstrings, and OpenAPI spec management for FastAPI applications.

## OpenAPI configuration

```python
# app/main.py
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        description="Production-grade FastAPI backend",
        version="1.0.0",
        docs_url="/docs" if settings.enable_docs else None,
        redoc_url="/redoc" if settings.enable_docs else None,
    )
    return app

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )
    # Add security scheme
    openapi_schema["components"]["securitySchemes"] = {
        "Bearer": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
        }
    }
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

## Docstrings

```python
# app/features/users/service.py
class UserService:
    """Handle user operations."""

    async def create_user(self, data: UserCreate) -> User:
        """Create a new user account.

        Args:
            data: User registration data including email and password.

        Returns:
            Created user with generated ID and timestamps.

        Raises:
            ConflictError: If email already exists.
            ValidationError: If input data is invalid.
        """
        pass

    async def get_user(self, user_id: str) -> User:
        """Retrieve a user by ID.

        Args:
            user_id: UUID of the user to retrieve.

        Returns:
            User object with all fields.

        Raises:
            NotFoundError: If user with given ID does not exist.
        """
        pass
```

## API description in responses

```python
# app/features/users/router.py
from fastapi import APIRouter, Response

router = APIRouter()

@router.post(
    "/",
    response_model=UserResponse,
    status_code=201,
    summary="Create a new user",
    description="Register a new user account with email and password.",
    responses={
        409: {"description": "Email already exists"},
        422: {"description": "Validation error"},
    },
)
async def create_user(data: UserCreate, response: Response):
    pass
```

## ReDoc customization

```python
# Custom ReDoc page
@app.get("/redoc", include_in_schema=False)
async def redoc():
    return get_redoc_html(
        openapi_url=app.openapi_url,
        title=f"{app.title} - ReDoc",
        redoc_js_url="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js",
    )
```

## Export OpenAPI spec

```bash
# Generate OpenAPI spec file
python -c "import json; from app.main import app; print(json.dumps(app.openapi(), indent=2))" > openapi.json

# Or use httpie
http http://localhost:8000/openapi.json > openapi.json
```

## API changelog

```markdown
# API CHANGELOG

## [1.0.0] - 2026-01-01
### Added
- POST /api/v1/users/ — Create user
- GET /api/v1/users/{id} — Get user

### Changed
- None

### Removed
- None
```

## DO NOT

- **Never** expose docs in production without authentication — disable or protect `/docs`.
- **Never** skip API versioning — use `/api/v1/`, `/api/v2/` prefixes.
- **Never** skip response models — they define the public API contract.
- **Never** skip docstrings on public functions — they're the source of truth.
- **Never** hardcode example values in docs — use `example` in Pydantic models.
- **Never** skip 4xx/5xx response definitions — clients need to know error shapes.
- **Never** forget to export OpenAPI spec before major releases.
- **Never** use vague descriptions — be specific about what each endpoint does.
