# MCP (Model Context Protocol)

Expose your tools/resources as MCP servers so AI agents (Claude, Cursor, etc.) can discover
and call them. Keep MCP thin — delegate to your existing service layer.

## Server setup

```python
# app/mcp/server.py
from mcp.server import Server
from mcp.types import Tool, TextContent
from app.features.users.service import UserService
from app.core.logging import get_logger

logger = get_logger(__name__)

server = Server("my-app")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_user",
            description="Get a user by ID",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_id": {"type": "string", "description": "User UUID"},
                },
                "required": ["user_id"],
            },
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    logger.info("MCP tool called", extra={"tool": name, "args": arguments})

    if name == "get_user":
        service = UserService()
        user = await service.get_user(arguments["user_id"])
        return [TextContent(type="text", text=user.model_dump_json())]

    raise ValueError(f"Unknown tool: {name}")
```

## HTTP transport

```python
# app/mcp/router.py
from fastapi import APIRouter
from app.mcp.server import server

router = APIRouter()

@router.post("/mcp")
async def mcp_endpoint(request: Request):
    # Handle MCP JSON-RPC over HTTP
    body = await request.json()
    result = await server.handle_request(body)
    return result
```

## Agent usage

```python
# Client-side (Claude Desktop, Cursor, etc.)
# Add to MCP config:
{
  "mcpServers": {
    "my-app": {
      "url": "https://api.example.com/mcp",
      "headers": {"Authorization": "Bearer <token>"}
    }
  }
}
```

## DO NOT

- **Never** expose raw database queries through MCP — use service layer.
- **Never** skip authentication on MCP endpoints — treat them like any API.
- **Never** expose internal system tools (migrations, cache invalidation) via MCP.
- **Never** log tool arguments with sensitive data — redact before logging.
- **Never** allow MCP tools to modify data without explicit confirmation (human-in-the-loop).
- **Never** skip rate limiting on MCP endpoints — agents can call them in loops.
- **Never** expose MCP tools that aren't useful for external agents.
- **Never** forget to regenerate MCP server after adding new tools.
