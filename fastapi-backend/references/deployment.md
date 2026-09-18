# Deployment: Docker, Local Dev, Production Targets

## Dockerfile (multi-stage, uv-based)

```dockerfile
FROM python:3.12-slim AS base
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

FROM base AS deps
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

FROM base AS runtime
COPY --from=deps /app/.venv /app/.venv
COPY . .
RUN uv sync --frozen --no-dev
ENV PATH="/app/.venv/bin:$PATH"
RUN useradd --create-home appuser && chown -R appuser /app
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- Build a **separate image/target** (or just override `CMD`) for the arq worker:
  `CMD ["arq", "app.workers.arq_worker.WorkerSettings"]`. Same image, different process — don't
  run uvicorn and the worker in one container.
- `--frozen` ensures the build fails rather than silently re-resolving if `uv.lock` drifts from
  `pyproject.toml`.
- Run as a non-root user; don't bake secrets into the image (see `fastapi-security.md`).

## docker-compose for local dev

```yaml
services:
  postgres:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: dev, POSTGRES_DB: app }
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7
    ports: ["6379:6379"]

  api:
    build: { context: ., target: runtime }
    env_file: .env
    ports: ["8000:8000"]
    depends_on: [postgres, redis]
    command: uvicorn app.main:app --host 0.0.0.0 --reload

  worker:
    build: { context: ., target: runtime }
    env_file: .env
    depends_on: [postgres, redis]
    command: arq app.workers.arq_worker.WorkerSettings

volumes:
  pgdata:
```

Run migrations as a one-off: `docker compose run --rm api alembic upgrade head`.

## Choosing a production target

| Target | Fits this stack when... | Watch out for |
|---|---|---|
| **AWS (ECS Fargate / App Runner)** | You need managed Postgres (RDS) + Redis (ElastiCache) in the same VPC, fine-grained IAM, and are already AWS-committed | More setup than a PaaS; use Secrets Manager for env vars, not plain task-def env |
| **Azure (Container Apps)** | Same shape as AWS, for Azure-committed orgs; pairs with Azure Database for Postgres + Azure Cache for Redis | KEDA-based autoscaling on Container Apps works well for the arq worker (scale on queue depth) |
| **Render** | Simplest Dockerfile-native path: web service (API) + background worker service + managed Postgres + managed Redis, minimal YAML, good default choice for small/medium teams | Cold starts on lower tiers; check region for DB/Redis colocated with the service |
| **Cloudflare (Containers)** | Dockerfile-based containers now run on Cloudflare's platform; a reasonable fit if you're already using Cloudflare for DNS/CDN/R2 | Newer offering than Render/AWS/Azure for this workload shape — validate current feature/region support before committing, and note Postgres/Redis still need to live elsewhere (Cloudflare doesn't provide either natively — pair with e.g. Neon/Supabase for Postgres and Upstash for Redis) |
| **Vercel** | Not a good fit for this stack as a general rule — Vercel's model is serverless functions with short execution limits, which doesn't suit a long-running FastAPI app or an arq worker process. Skip it for the backend; use it only if a separate frontend needs hosting. | — |

For all of these: run the arq worker as its **own** service/task definition, scaled
independently from the API (jobs and requests have very different resource/scaling profiles).

## Environment parity checklist

- Same `DATABASE_URL`/`REDIS_URL` shape (async DSN, e.g. `postgresql+asyncpg://...`) across
  local/staging/prod — only the host/credentials change.
- Migrations run as an explicit deploy step (`alembic upgrade head`) before traffic shifts to
  new revisions, never automatically on container boot.
- Health check endpoint (`/healthz`) that checks DB + Redis connectivity, used by the platform's
  load balancer/orchestrator to gate traffic — not just "process is running."
- Structured (JSON) logs to stdout in every environment so the platform's log aggregation (
  CloudWatch, Azure Monitor, Render logs, etc.) can parse them without extra shipping config.

## DO NOT

- **Never** run uvicorn and arq worker in the same container — they must be separate processes.
- **Never** bake secrets into Docker image layers — use environment variables or a secrets manager.
- **Never** run as root in containers — use a non-root user.
- **Never** skip `--frozen` in `uv sync` — it prevents silent lockfile drift.
- **Never** run migrations automatically on container boot — run as explicit deploy step.
- **Never** use `allow_hosts: ["*"]` in production — set explicit allowed hosts.
- **Never** skip health check endpoints (`/healthz`) — they must check DB + Redis connectivity.
- **Never** use Vercel for this stack — it's serverless with short execution limits, not suited for long-running FastAPI or arq.
- **Never** use plaintext logs in production — structured JSON to stdout.
- **Never** skip connection pooling in production — configure `pool_size` and `pool_pre_ping`.
- **Never** use `--reload` in production — it's for local dev only.
- **Never** skip the non-root user in Docker — it's a security requirement.
- **Never** use `POSTGRES_PASSWORD: dev` in production — use a strong, unique password.
- **Never** skip `depends_on` health checks in docker-compose — services may not be ready.
