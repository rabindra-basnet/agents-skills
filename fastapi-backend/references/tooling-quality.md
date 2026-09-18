# Tooling & Quality Gates

Four tools, each with one job: **ruff** (format + lint), **bandit** (dependency/code security
scanning), **import-linter** (architectural boundaries), **pre-commit** (git hooks that run all
of the above before a commit lands).

## ruff — formatting and linting

`ruff` replaces black + isort + flake8 + most flake8-plugins in one fast binary.

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"
src = ["app", "tests"]

[tool.ruff.lint]
select = [
    "E", "F", "W",       # pycodestyle/pyflakes
    "I",                  # isort
    "UP",                 # pyupgrade — enforce modern syntax (3.12 unions, etc.)
    "B",                  # bugbear
    "C4",                 # comprehensions
    "SIM",                # simplify
    "ASYNC",              # async-specific lint rules (blocking calls in async def, etc.)
    "S",                  # bandit-equivalent lint rules (belt-and-suspenders with bandit itself)
    "RUF",                # ruff-native rules
]
ignore = ["E501"]  # formatter handles line length

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]  # allow assert in tests

[tool.ruff.format]
quote-style = "double"
```

Commands: `uv run ruff format .` and `uv run ruff check . --fix`.

## bandit — dependency & code security scanning

Bandit scans Python source for common security issues (hardcoded secrets, `eval`/`exec`, weak
hashes, unsafe YAML/pickle loads, shell injection, etc.). Run it against `app/`, not `tests/`.

```toml
# pyproject.toml
[tool.bandit]
targets = ["app"]
exclude_dirs = ["tests", "migrations"]
skips = []  # don't blanket-skip checks; suppress inline with `# nosec: B105` + a reason if a finding is a false positive
```

Command: `uv run bandit -c pyproject.toml -r app`. Treat any new **HIGH** severity finding as a
blocker; MEDIUM findings need a one-line justification if suppressed.

> Note: bandit scans *code*, not the dependency tree. For known-CVE scanning of third-party
> packages, also run `uv run pip-audit` (or `uv audit` once available in your uv version) in CI
> as a companion step — bandit alone does not cover this.

## import-linter — enforcing the vertical-slice boundary

`import-linter` fails the build if code violates the layering rules from
`project-structure.md`. Config lives in `pyproject.toml` or a dedicated `.importlinter` file.

```ini
# .importlinter
[importlinter]
root_package = app

[importlinter:contract:1]
name = Features must not import each other directly
type = independence
modules =
    app.features.users
    app.features.auth
    app.features.billing

[importlinter:contract:2]
name = Core must not depend on features
type = forbidden
source_modules =
    app.core
forbidden_modules =
    app.features

[importlinter:contract:3]
name = Layering inside main
type = layers
layers =
    app.main
    app.features
    app.shared
    app.core
```

- The `independence` contract is the main enforcement of vertical slicing: no feature package
  may import another. Add every feature package to the `modules` list as you create it.
- The `forbidden` contract stops `core/` (low-level, framework-adjacent) from reaching back up
  into business features — dependencies should point inward/downward.
- Command: `uv run lint-imports`. Run it in CI and in the pre-commit hook — this is the
  mechanical guardrail that keeps the architecture from eroding as agents/humans add code fast.

## Git hooks via pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0  # pin to the version matching your ruff in pyproject.toml
    hooks:
      - id: ruff-format
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.10
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
        additional_dependencies: ["bandit[toml]"]

  - repo: local
    hooks:
      - id: import-linter
        name: import-linter
        entry: uv run lint-imports
        language: system
        pass_filenames: false

      - id: pytest-fast
        name: pytest (unit, fast subset)
        entry: uv run pytest -m "not integration" -q
        language: system
        pass_filenames: false

  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.29.0
    hooks:
      - id: commitizen  # enforces Conventional Commits on the commit message itself
        stages: [commit-msg]
```

Install once per clone: `uv run pre-commit install --hook-type pre-commit --hook-type commit-msg`.

### Commit message convention

Pair the `commitizen` hook with **Conventional Commits** (`feat:`, `fix:`, `chore:`, `refactor:`,
`test:`, `docs:`, `ci:`, optionally scoped like `feat(billing): add refund endpoint`). This gives
you free changelog generation and makes `git log` skimmable slice-by-slice. Reject commits that
don't match in the `commit-msg` hook stage rather than relying on discipline alone.

## CI wiring (GitHub Actions example)

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --all-extras --dev
      - run: uv run ruff format --check .
      - run: uv run ruff check .
      - run: uv run bandit -c pyproject.toml -r app
      - run: uv run lint-imports
      - run: uv run pytest --cov=app -q
```

Run these same commands locally before calling any generated code "done" — the pre-commit hooks
catch most of it, but CI is the source of truth (e.g. for the full, slower test suite).

## DO NOT

- **Never** skip pre-commit hooks — they exist to catch issues before they reach CI.
- **Never** use `--no-verify` to bypass git hooks unless you have a documented, temporary reason.
- **Never** suppress bandit findings with blanket `skips = [...]` — suppress inline with `# nosec` + a reason.
- **Never** ignore HIGH severity bandit findings — treat them as build blockers.
- **Never** commit code that fails `ruff check` or `ruff format --check` — fix it first.
- **Never** add a new feature package without adding it to the `import-linter` independence contract.
- **Never** run `pytest` with `-x` (stop on first failure) in CI — run full suite to see all failures.
- **Never** use `# type: ignore` without a specific error code and reason.
- **Never** commit without Conventional Commits format (`feat:`, `fix:`, `chore:`, etc.).
- **Never** run quality tools manually in CI if pre-commit catches them — but CI is the source of truth for the full, slower test suite.
- **Never** treat linting warnings as optional — they are errors in this stack.
