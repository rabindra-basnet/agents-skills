# FastAPI: App Structure & Security

## App factory + lifespan

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.core.config import settings
from app.core.db import engine
from app.core.redis import redis_pool
from app.shared.errors import register_exception_handlers
from app.features.users.router import router as users_router
from app.features.auth.router import router as auth_router

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = await redis_pool()
    yield
    await app.state.redis.aclose()
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        lifespan=lifespan,
        docs_url="/docs" if settings.enable_docs else None,   # disable in prod if desired
        redoc_url=None,
    )
    register_exception_handlers(app)
    register_security_middleware(app)
    app.include_router(auth_router, prefix="/api/v1/auth", tags=["auth"])
    app.include_router(users_router, prefix="/api/v1/users", tags=["users"])
    return app

app = create_app()
```

Config via `pydantic-settings` (`app/core/config.py`), reading from environment/`.env`, never
hardcoded. Never commit `.env`; ship `.env.example` with placeholder values.

## Security middleware

```python
def register_security_middleware(app: FastAPI) -> None:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_allowed_origins,  # explicit list, never "*" with credentials
        allow_credentials=True,
        allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
        allow_headers=["Authorization", "Content-Type"],
    )
    app.add_middleware(TrustedHostMiddleware, allowed_hosts=settings.allowed_hosts)
    app.add_middleware(SecurityHeadersMiddleware)   # HSTS, X-Content-Type-Options, X-Frame-Options, etc.
    app.add_middleware(RequestIdMiddleware)          # for correlating logs/traces per request
```

Behind a reverse proxy/load balancer that terminates TLS (which is the norm on AWS/Azure/Render/
Cloudflare), also set `ProxyHeadersMiddleware`-equivalent handling so `request.url.scheme` and
client IP are correct for rate limiting/logging.

## Auth: JWT with rotating refresh tokens

- **Password hashing**: `passlib` with `argon2` (preferred over bcrypt for new projects).
- **Access token**: short-lived JWT (5–15 min), signed with `python-jose`, `HS256` if
  single-service or `RS256`/`ES256` if other services need to verify tokens without the signing
  secret.
- **Refresh token**: longer-lived, opaque random token (not a JWT), stored hashed in Postgres
  with a `family_id` for rotation-on-use and revocation; rotate on every refresh and reject
  reuse of an already-rotated token (signals theft → revoke the whole family).
- **Session/blacklist**: use Redis for access-token revocation lists (jti-based) and for storing
  short-lived state like OTPs/password-reset tokens, keyed with a TTL matching expiry.
- Auth dependency (`features/auth/deps.py`) is a FastAPI `Depends()` that decodes the JWT,
  checks the Redis blacklist, and loads the current user — inject it into any route that needs
  auth rather than duplicating decode logic.

## Rate limiting

Use `slowapi` (or a small custom Redis-backed token-bucket middleware if you need per-route
limits keyed by user id rather than IP) in front of auth endpoints and any AI/LLM-backed
endpoint specifically (those are the expensive, abusable ones). Return `429` with a
`Retry-After` header.

## Input validation & output shaping

- All request bodies/query params are Pydantic v2 models — never read raw `Request.json()` and
  hand-parse.
- Use `Field(..., max_length=..., pattern=...)` constraints instead of validating manually in
  service code.
- Response models (`response_model=...`) so extra/internal fields (password hashes, internal
  ids) never leak by accident.
- File uploads: enforce content-type and size limits before touching the file; scan/limit
  extensions; never trust the client-provided filename for storage paths (generate your own).

## OWASP-flavored checklist to apply per feature

- SQL injection: only ORM/parameterized queries (SQLAlchemy Core `text()` with bound params if
  raw SQL is ever needed — never f-string interpolation into SQL).
- Broken access control: authorization checks live in `service.py`, checked against the
  authenticated user from the auth dependency — never trust an id in the request body/path
  alone to scope a query.
- Secrets management: `pydantic-settings` + environment variables locally, a real secrets
  manager (AWS Secrets Manager / Azure Key Vault / platform env vars) in prod — never in the
  Docker image layers.
- Logging: never log raw passwords, tokens, or full card/PII payloads; redact known sensitive
  keys in the structured logger.
- Dependency hygiene: this is what `bandit`/`pip-audit` in `tooling-quality.md` are for — wire
  them into CI, don't rely on manual review.
- Error responses: return generic error messages to clients (via `shared/errors.py` handlers);
  log full tracebacks server-side only.

## DO NOT

- **Never** use `allow_origins=["*"]` with `allow_credentials=True` — CORS will reject it.
- **Never** hardcode secrets in code or Docker image layers — use `pydantic-settings` + environment variables.
- **Never** commit `.env` files — ship `.env.example` with placeholders.
- **Never** use `Request.json()` and hand-parse — always use Pydantic models for request bodies.
- **Never** return stack traces or internal error details to clients — generic messages only.
- **Never** log raw passwords, tokens, or full PII — redact sensitive fields.
- **Never** use HS256 for JWT if multiple services need to verify tokens — use RS256/ES256.
- **Never** store refresh tokens as JWTs — use opaque random tokens hashed in Postgres.
- **Never** trust an id in the request body/path alone for authorization — check against authenticated user.
- **Never** use f-string interpolation into SQL — always parameterized queries.
- **Never** disable CORS or security middleware in production.
- **Never** use `HTTPException` directly in service code — raise domain errors and handle in exception handlers.
- **Never** skip rate limiting on auth endpoints and AI/LLM-backed endpoints.
- **Never** trust client-provided filenames for storage paths — generate your own.
- **Never** store API keys in the database — use environment variables or a secrets manager.
