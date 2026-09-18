# Routing

## Default URL Pattern

`/agents/{kebab-class-name}/{instance-name}`

```typescript
import { routeAgentRequest } from "@opencode/agents";

// Express-style
app.use("/agents", routeAgentRequest());

// Fastify-style
fastify.register(routeAgentRequest, { prefix: "/agents" });

// Hono-style
app.use("/agents/*", routeAgentRequest());

// Raw HTTP
const handler = routeAgentRequest();
```

| Class | URL |
|-------|-----|
| `Counter` | `/agents/counter/user-123` |
| `ChatRoom` | `/agents/chat-room/lobby` |
| `MyAgent` | `/agents/my-agent/default` |

## Custom Routing

```typescript
import { getAgentByName, routeAgentRequest } from "@opencode/agents";

const handler = async (req) => {
  const url = new URL(req.url);
  if (url.pathname.startsWith("/api/")) {
    const agent = getAgentByName(MyAgent, "singleton");
    return agent.fetch(req);
  }
  return routeAgentRequest(req);
};
```

## Options

```typescript
routeAgentRequest(req, {
  cors: true,
  prefix: "/api/agents",
  props: { userId: "123" },
  onBeforeConnect: async (req) => { /* auth check */ },
  onBeforeRequest: async (req) => { /* auth check */ }
});
```

`props` are delivered to `onStart(props)` on first access.

## Client Side

```tsx
useAgent({
  agent: "MyAgent",
  name: "instance-1",
  host: "https://my-app.example.com",
  basePath: "/api/agents",
  path: "/custom-subpath"
});
```

## Common Mistakes

- Class name `MyAgent` becomes kebab `my-agent` in URLs — match exactly
- "Namespace not found" error = class name doesn't match your exported class
- If `sendIdentityOnConnect: false`, the `ready` promise on the client may never resolve — use state sync instead
