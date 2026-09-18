# Webhooks & Push Notifications

Receive events from external services (Stripe, GitHub, etc.) via webhooks and send push
notifications to clients. Validate signatures, handle retries, and process asynchronously.

## Webhook receiver

```python
# app/features/webhooks/router.py
import hmac
import hashlib
from fastapi import APIRouter, Request, HTTPException
from app.core.config import settings
from app.workers.enqueue import enqueue_job
from app.core.logging import get_logger

logger = get_logger(__name__)
router = APIRouter()

@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    body = await request.body()
    sig = request.headers.get("stripe-signature")

    # Verify signature
    expected = hmac.new(
        settings.stripe_webhook_secret.encode(),
        body,
        hashlib.sha256,
    ).hexdigest()

    if not hmac.compare_digest(sig, expected):
        logger.warning("Invalid webhook signature", extra={"provider": "stripe"})
        raise HTTPException(status_code=401, detail="Invalid signature")

    # Process asynchronously
    await enqueue_job(
        "app.workers.jobs.process_stripe_webhook",
        queue="short",
        payload=body.decode(),
    )

    return {"status": "ok"}
```

## Webhook processing job

```python
# app/workers/jobs/process_stripe_webhook.py
import json
from app.workers.logging import get_worker_logger
from app.core.logging import get_logger

logger = get_worker_logger(__name__)

async def process_stripe_webhook(ctx: dict, *, payload: str) -> dict:
    event = json.loads(payload)
    event_type = event["type"]
    logger.info("Processing webhook", extra={"provider": "stripe", "event_type": event_type})

    match event_type:
        case "payment_intent.succeeded":
            # Handle payment success
            pass
        case "payment_intent.payment_failed":
            # Handle payment failure
            pass
        case _:
            logger.info("Unhandled event type", extra={"event_type": event_type})

    return {"processed": True}
```

## Push notifications (FCM)

```python
# app/core/push/fcm.py
import httpx
from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)

async def send_push(*, token: str, title: str, body: str, data: dict | None = None) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "https://fcm.googleapis.com/v1/projects/{settings.fcm_project_id}/messages:send",
            headers={"Authorization": f"Bearer {settings.fcm_key}"},
            json={
                "message": {
                    "token": token,
                    "notification": {"title": title, "body": body},
                    "data": data or {},
                }
            },
            timeout=30,
        )
        resp.raise_for_status()
        logger.info("Push sent", extra={"token": token[:10] + "..."})
        return resp.json()
```

## Webhook retry handling

```python
# Idempotency with deduplication
async def process_webhook_event(event_id: str, payload: dict):
    # Check if already processed
    existing = await redis.get(f"webhook:processed:{event_id}")
    if existing:
        return {"status": "already_processed"}

    # Process
    result = await _process_event(payload)

    # Mark as processed
    await redis.setex(f"webhook:processed:{event_id}", 86400, "1")
    return result
```

## DO NOT

- **Never** skip webhook signature verification — always validate before processing.
- **Never** process webhooks inline in request handlers — enqueue to background job.
- **Never** assume webhooks arrive in order — handle out-of-order events.
- **Never** skip idempotency — webhooks retry on failure.
- **Never** log full webhook payloads — they may contain sensitive data.
- **Never** expose internal webhook endpoints without rate limiting.
- **Never** store raw webhook payloads forever — process and archive/delete.
- **Never** skip retry logic for push notifications — transient failures happen.
- **Never** send push notifications without checking user notification preferences.
