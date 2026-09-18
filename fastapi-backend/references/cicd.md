# CI/CD

GitHub Actions pipelines for testing, linting, and deploying FastAPI applications.

## Workflow structure

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --dev
      - run: uv run ruff format --check .
      - run: uv run ruff check .
      - run: uv run bandit -r app -c pyproject.toml
      - run: uv run import-linter

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --dev
      - run: uv run pytest --cov=app --cov-report=xml
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
      - uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
```

## Deployment workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: |
          # Your deploy command here
          # e.g., docker build, push to registry, update service
```

## Branch protection

```yaml
# Required checks in GitHub repo settings
branches:
  main:
    protection:
      required_status_checks:
        - lint
        - test
      required_pull_request_reviews:
        required_approving_review_count: 1
```

## Secrets management

```yaml
# GitHub repo secrets (Settings → Secrets → Actions)
# DATABASE_URL, REDIS_URL, etc.
# Never commit these to the repo
```

## DO NOT

- **Never** commit secrets to CI config — use GitHub Secrets.
- **Never** skip linting in CI — enforce on every PR.
- **Never** skip tests in CI — enforce on every PR.
- **Never** deploy without running tests first.
- **Never** skip branch protection — require reviews and status checks.
- **Never** hardcode environment-specific values in workflows — use variables.
- **Never** skip coverage reporting — track it over time.
- **Never** run migrations automatically in CI — run as explicit deploy step.
