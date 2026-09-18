# Email

Send and receive email using external providers (SendGrid, Resend, Postmark, AWS SES).
Keep email sending in background jobs, never inline in request handlers.

## Provider abstraction

```python
# app/core/email/provider.py
from typing import Protocol

class EmailProvider(Protocol):
    async def send(
        self, *, to: str, subject: str, html: str, from_email: str | None = None
    ) -> dict: ...
```

```python
# app/core/email/sendgrid.py
import httpx
from app.core.logging import get_logger

logger = get_logger(__name__)

class SendGridProvider:
    def __init__(self, api_key: str, from_email: str):
        self.api_key = api_key
        self.from_email = from_email

    async def send(self, *, to, subject, html, from_email=None):
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                "https://api.sendgrid.com/v3/mail/send",
                headers={"Authorization": f"Bearer {self.api_key}"},
                json={
                    "personalizations": [{"to": [{"email": to}]}],
                    "from": {"email": from_email or self.from_email},
                    "subject": subject,
                    "content": [{"type": "text/html", "value": html}],
                },
                timeout=30,
            )
            resp.raise_for_status()
            logger.info("Email sent", extra={"to": to, "provider": "sendgrid"})
            return {"status": "sent", "to": to}
```

## Factory

```python
# app/core/email/factory.py
from app.core.config import settings
from app.core.email.sendgrid import SendGridProvider

def get_email_provider() -> SendGridProvider:
    return SendGridProvider(
        api_key=settings.sendgrid_api_key,
        from_email=settings.email_from,
    )
```

## Background job for sending

```python
# app/workers/jobs/send_email.py
from app.workers.logging import get_worker_logger
from app.core.email.factory import get_email_provider

logger = get_worker_logger(__name__)

async def send_email(ctx: dict, *, to: str, subject: str, html: str) -> dict:
    provider = get_email_provider()
    result = await provider.send(to=to, subject=subject, html=html)
    logger.info("Email sent", extra={"to": to, "subject": subject})
    return result
```

## Enqueue from service

```python
# app/features/auth/service.py
from app.workers.enqueue import enqueue_job

class AuthService:
    async def register(self, data: UserCreate) -> User:
        user = await self._repo.create(data)
        await enqueue_job(
            "app.workers.jobs.send_email",
            queue="short",
            to=user.email,
            subject="Welcome!",
            html="<h1>Welcome to our platform!</h1>",
        )
        return user
```

## Template rendering

```python
# app/core/email/templates.py
from jinja2 import Environment, FileSystemLoader

env = Environment(loader=FileSystemLoader("templates/email"))

def render_template(template_name: str, context: dict) -> str:
    template = env.get_template(template_name)
    return template.render(**context)
```

## DO NOT

- **Never** send email inline in request handlers — always enqueue to background job.
- **Never** hardcode email provider API keys — use `pydantic-settings`.
- **Never** use plain text emails for transactional mail — use HTML templates.
- **Never** skip unsubscribe links in marketing emails (CAN-SPAM compliance).
- **Never** log email content — log only metadata (to, subject, status).
- **Never** send emails without retry logic — use arq's `max_tries`.
- **Never** use your main domain for sending — use a subdomain (mail.example.com).
- **Never** skip SPF/DKIM/DMARC configuration for your sending domain.
