# Human-in-the-Loop

Allow agents to pause, ask for human approval, and resume. Use this when the agent's decision
has real-world consequences (payments, data deletion, external API calls).

## Flow

```
Agent decides → Publish approval request → Wait for human response → Resume or abort
```

## Implementation with arq + Redis

```python
# app/features/approvals/schemas.py
from pydantic import BaseModel
from datetime import datetime

class ApprovalRequest(BaseModel):
    id: str
    agent_id: str
    action: str
    payload: dict
    status: str = "pending"  # pending|approved|rejected
    requested_at: datetime
    responded_at: datetime | None = None
    reviewer: str | None = None
```

```python
# app/features/approvals/service.py
import uuid
from datetime import datetime
from app.core.redis import get_redis
from app.core.logging import get_logger
from app.features.approvals.schemas import ApprovalRequest

logger = get_logger(__name__)

class ApprovalService:
    async def request_approval(self, *, agent_id: str, action: str, payload: dict) -> str:
        approval_id = str(uuid.uuid4())
        request = ApprovalRequest(
            id=approval_id,
            agent_id=agent_id,
            action=action,
            payload=payload,
            requested_at=datetime.utcnow(),
        )

        redis = await get_redis()
        await redis.setex(f"approval:{approval_id}", 3600, request.model_dump_json())
        logger.info("Approval requested", extra={"approval_id": approval_id, "action": action})
        return approval_id

    async def respond(self, approval_id: str, *, approved: bool, reviewer: str) -> bool:
        redis = await get_redis()
        data = await redis.get(f"approval:{approval_id}")
        if not data:
            return False

        request = ApprovalRequest.model_validate_json(data)
        request.status = "approved" if approved else "rejected"
        request.responded_at = datetime.utcnow()
        request.reviewer = reviewer

        await redis.setex(f"approval:{approval_id}", 3600, request.model_dump_json())
        logger.info("Approval responded", extra={"approval_id": approval_id, "approved": approved})
        return True

    async def wait_for_approval(self, approval_id: str, timeout: int = 300) -> ApprovalRequest | None:
        """Poll Redis for approval response (used by agent)."""
        import asyncio
        redis = await get_redis()

        for _ in range(timeout):
            data = await redis.get(f"approval:{approval_id}")
            if data:
                request = ApprovalRequest.model_validate_json(data)
                if request.status != "pending":
                    return request
            await asyncio.sleep(1)
        return None
```

## Agent integration

```python
# app/features/agents/tools/payment.py
from app.features.approvals.service import ApprovalService

async def process_payment(ctx: dict, *, amount: float, user_id: str) -> dict:
    approval_service = ApprovalService()
    approval_id = await approval_service.request_approval(
        agent_id=ctx["agent_id"],
        action="process_payment",
        payload={"amount": amount, "user_id": user_id},
    )

    # Pause and wait
    result = await approval_service.wait_for_approval(approval_id)
    if not result or result.status != "approved":
        return {"status": "rejected", "reason": "Payment not approved"}

    # Resume execution
    # ... process payment
    return {"status": "completed"}
```

## DO NOT

- **Never** skip human approval for high-risk actions (payments, data deletion, external API calls).
- **Never** let agents approve their own actions — require a different human reviewer.
- **Never** store approval requests in memory — use Redis or Postgres for durability.
- **Never** skip timeout handling — unapproved requests must expire and abort.
- **Never** auto-approve all requests — require explicit human decision.
- **Never** expose internal agent reasoning to the reviewer — show only the action and payload.
- **Never** skip audit logging for approval decisions.
