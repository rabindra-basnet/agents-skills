# Client SDK

Generate a typed Python SDK from your OpenAPI spec so frontend/mobile/CLI consumers never call
your API raw — they get autocomplete, type-safety, and automatic retries.

## Generation

```bash
# Install the generator
uv add --dev openapi-python-client

# Generate from your running app or spec file
uv run openapi-python-client generate --path http://localhost:8000/openapi.json

# Or from a saved spec
uv run openapi-python-client generate --path openapi.json
```

```toml
# pyproject.toml
[tool.openapi-python-client]
project_name = "my-app-client"
package_name = "my_app_client"
path = "sdk/"
```

## Usage

```python
from my_app_client import Client
from my_app_client.models import UserCreate, UserResponse

client = Client(base_url="https://api.example.com", token="...")

# Typed, autocomplete, type-checked
user: UserResponse = client.users.create_user(body=UserCreate(email="test@example.com"))
```

## Auto-retry with tenacity

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def request_with_retry(method, url, **kwargs):
    return client.request(method, url, **kwargs)
```

## Versioning the SDK

- Bump SDK version on breaking API changes (use `--version` flag).
- Publish to private PyPI or GitHub Packages.
- Pin SDK version in consuming projects.

## DO NOT

- **Never** hand-write API client code — generate from OpenAPI.
- **Never** expose internal API endpoints in the SDK (use tags/operations to filter).
- **Never** skip authentication in the generated client — always require a token/base_url.
- **Never** publish SDK without running the generated tests first.
- **Never** use `httpx` directly in consuming code — always use the generated client.
- **Never** forget to regenerate the SDK after API changes.
- **Never** use wildcard imports from the SDK — use specific model imports.
