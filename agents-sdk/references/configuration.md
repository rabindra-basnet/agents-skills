# Configuration

## Generic Config

```python
# agent_config.py
from agents_sdk import define_config

config = define_config(
    name="my-agent",
    storage={
        "type": "sqlite",  # or "postgresql", "dynamodb", "redis"
        "database": "./data/agents.db"
    },
    state={
        "sync": True,        # auto-sync state to clients
        "validate": True     # run validate_state_change on updates
    }
)
```

## Provider Adapters

### AWS Lambda

```python
from agents_sdk.aws import create_agent

handler = create_agent(
    storage={
        "type": "dynamodb",
        "table_name": "agents-state"
    }
)
```

### Google Cloud Run

```python
from agents_sdk.gcp import create_agent

app = create_agent(
    storage={
        "type": "firestore",
        "collection": "agents-state"
    }
)
```

### Azure Functions

```python
from agents_sdk.azure import create_agent

handler = create_agent(
    storage={
        "type": "cosmosdb",
        "database": "agents",
        "container": "state"
    }
)
```

### Railway / Fly.io

```python
from agents_sdk import create_agent

app = create_agent(
    storage={
        "type": "postgresql",
        "url": os.environ["DATABASE_URL"]
    }
)
```

### Self-hosted

```python
from agents_sdk.server import create_agent

app = create_agent(
    storage={
        "type": "sqlite",
        "database": "./data/agents.db"
    }
)

if __name__ == "__main__":
    app.run(port=3000)
```

## Environment Variables

```bash
# Storage
DATABASE_URL=postgresql://user:pass@localhost/agents
REDIS_URL=redis://localhost:6379

# AI Providers
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Agent Settings
AGENT_SECRET_KEY=your-secret-key
```

## pyproject.toml

```toml
[project]
name = "my-agent"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "agents-sdk[chat]",
    "fastapi",
    "uvicorn",
]

[project.optional-dependencies]
aws = ["agents-sdk[aws]"]
gcp = ["agents-sdk[gcp]"]
azure = ["agents-sdk[azure]"]
```
