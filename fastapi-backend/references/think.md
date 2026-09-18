# Think (Chain-of-Thought)

Let the agent reason step-by-step before acting. Use this for complex decisions, planning,
and multi-step tasks where the agent needs to "think" before executing.

## Schema

```python
# app/features/agents/think/schemas.py
from pydantic import BaseModel
from typing import Literal

class Thought(BaseModel):
    step: int
    reasoning: str
    conclusion: str | None = None

class ThinkResult(BaseModel):
    thoughts: list[Thought]
    decision: str
    confidence: float  # 0.0 - 1.0
    next_action: str
```

## Tool implementation

```python
# app/features/agents/think/tool.py
from app.core.ai.factory import get_ai_provider
from app.features.agents.think.schemas import ThinkResult
from app.core.logging import get_logger

logger = get_logger(__name__)

THINK_PROMPT = """You are a reasoning engine. Think step-by-step about the user's request.

Return a JSON object with:
- thoughts: list of {step, reasoning, conclusion}
- decision: your final decision
- confidence: 0.0-1.0
- next_action: what to do next

User request: {request}
"""

async def think(ctx: dict, *, request: str) -> ThinkResult:
    ai = get_ai_provider()

    result = await ai.complete(
        messages=[{"role": "user", "content": THINK_PROMPT.format(request=request)}],
        model="gpt-4o",
        temperature=0.3,  # Lower temperature for reasoning
    )

    # Parse with Pydantic (treat parse failure as retryable)
    think_result = ThinkResult.model_validate_json(result.text)

    logger.info(
        "Think completed",
        extra={
            "thought_count": len(think_result.thoughts),
            "decision": think_result.decision,
            "confidence": think_result.confidence,
        },
    )

    return think_result
```

## Agent integration

```python
# app/features/agents/tools.py
from app.features.agents.think.tool import think

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "think",
            "description": "Reason step-by-step before acting",
            "parameters": {
                "type": "object",
                "properties": {
                    "request": {"type": "string", "description": "What to think about"},
                },
                "required": ["request"],
            },
        },
    },
]
```

## Usage pattern

```python
# Agent loop
async def agent_loop(user_message: str):
    # Step 1: Think
    thought = await think(ctx, request=user_message)

    if thought.confidence < 0.5:
        return "I'm not confident enough to proceed. Can you clarify?"

    # Step 2: Act based on thinking
    match thought.next_action:
        case "search_database":
            return await search_database(user_message)
        case "call_api":
            return await call_external_api(user_message)
        case _:
            return thought.decision
```

## DO NOT

- **Never** skip the think step for complex decisions — it reduces errors significantly.
- **Never** use high temperature for reasoning — use 0.2-0.4 for deterministic thinking.
- **Never** skip logging thought steps — you need to debug agent reasoning.
- **Never** trust low-confidence decisions — ask for clarification or retry.
- **Never** use think for simple queries — it adds latency unnecessarily.
- **Never** skip Pydantic validation on think output — LLMs can return malformed JSON.
- **Never** expose raw thought process to users — summarize instead.
- **Never** store full thought history forever — prune after analysis.
