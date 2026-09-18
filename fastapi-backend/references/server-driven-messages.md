# Server-Driven Messages

Let the server control what the client renders — UI components, messages, and actions are
described as JSON, not hardcoded in the frontend. The client is a thin renderer.

## Schema

```python
# app/features/messaging/schemas.py
from pydantic import BaseModel
from typing import Literal

class TextBlock(BaseModel):
    type: Literal["text"] = "text"
    content: str

class ImageBlock(BaseModel):
    type: Literal["image"] = "image"
    url: str
    alt: str

class ActionBlock(BaseModel):
    type: Literal["action"] = "action"
    label: str
    action: str  # e.g. "confirm_payment", "open_link"
    payload: dict | None = None

class ServerMessage(BaseModel):
    blocks: list[TextBlock | ImageBlock | ActionBlock]
    metadata: dict | None = None
```

## API endpoint

```python
# app/features/messaging/router.py
from fastapi import APIRouter
from app.features.messaging.schemas import ServerMessage

router = APIRouter()

@router.get("/messages/{conversation_id}", response_model=list[ServerMessage])
async def get_messages(conversation_id: str):
    # Server decides what to render
    return [
        ServerMessage(blocks=[
            TextBlock(content="Welcome!"),
            ActionBlock(label="Get Started", action="start_onboarding"),
        ])
    ]
```

## Client renderer

```typescript
// Client-side (React example)
function ServerMessageRenderer({ message }) {
  return message.blocks.map(block => {
    switch (block.type) {
      case "text": return <p>{block.content}</p>;
      case "image": return <img src={block.url} alt={block.alt} />;
      case "action": return <button onClick={() => handleAction(block)}>{block.label}</button>;
    }
  });
}
```

## DO NOT

- **Never** hardcode UI components in the client for dynamic content — use server-driven messages.
- **Never** return raw HTML from the server — use structured block types.
- **Never** skip validation of server messages with Pydantic on the client side.
- **Never** put business logic in the client renderer — server decides what to show.
- **Never** expose internal system messages to the client — filter server-side.
- **Never** use server-driven messages for static pages — use regular templates/routes.
- **Never** trust client-rendered actions without server-side validation.
