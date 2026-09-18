# AI Integration: Provider-Agnostic OpenAI / Anthropic Client

Only pull this in when a feature actually needs an LLM call. When it does, put both providers
behind one small interface so a slice's business logic never imports `openai` or `anthropic`
directly — that keeps provider swaps (or A/B-ing providers) to a one-line config change.

## Common interface

```python
# app/core/ai/provider.py
from typing import Protocol, AsyncIterator

class ChatMessage(TypedDict):
    role: Literal["system", "user", "assistant"]
    content: str

class CompletionResult(BaseModel):
    text: str
    model: str
    input_tokens: int
    output_tokens: int
    raw: dict  # provider's raw response, for debugging — don't rely on this elsewhere

class AIProvider(Protocol):
    async def complete(
        self, messages: list[ChatMessage], *, model: str, max_tokens: int = 1024,
        temperature: float = 0.7, tools: list[dict] | None = None,
    ) -> CompletionResult: ...

    async def stream(
        self, messages: list[ChatMessage], *, model: str, max_tokens: int = 1024,
    ) -> AsyncIterator[str]: ...
```

## OpenAI (and any OpenAI-compatible endpoint)

```python
# app/core/ai/openai_provider.py
from openai import AsyncOpenAI

class OpenAIProvider:
    def __init__(self, api_key: str, base_url: str | None = None):
        # base_url lets this same class hit any OpenAI-compatible API (Azure OpenAI,
        # OpenRouter, a local vLLM/Ollama server, etc.) — don't hardcode api.openai.com
        self._client = AsyncOpenAI(api_key=api_key, base_url=base_url)

    async def complete(self, messages, *, model, max_tokens=1024, temperature=0.7, tools=None):
        resp = await self._client.chat.completions.create(
            model=model, messages=messages, max_tokens=max_tokens,
            temperature=temperature, tools=tools,
        )
        choice = resp.choices[0].message
        return CompletionResult(
            text=choice.content or "", model=resp.model,
            input_tokens=resp.usage.prompt_tokens, output_tokens=resp.usage.completion_tokens,
            raw=resp.model_dump(),
        )
```

## Anthropic (Claude)

```python
# app/core/ai/anthropic_provider.py
from anthropic import AsyncAnthropic

class AnthropicProvider:
    def __init__(self, api_key: str):
        self._client = AsyncAnthropic(api_key=api_key)

    async def complete(self, messages, *, model, max_tokens=1024, temperature=0.7, tools=None):
        system, turns = _split_system_message(messages)  # Anthropic takes system separately
        resp = await self._client.messages.create(
            model=model, system=system, messages=turns, max_tokens=max_tokens,
            temperature=temperature, tools=tools,
        )
        text = "".join(block.text for block in resp.content if block.type == "text")
        return CompletionResult(
            text=text, model=resp.model,
            input_tokens=resp.usage.input_tokens, output_tokens=resp.usage.output_tokens,
            raw=resp.model_dump(),
        )
```

## Wiring & selection

```python
# app/core/ai/factory.py
def get_ai_provider(settings: Settings) -> AIProvider:
    match settings.ai_provider:
        case "openai":
            return OpenAIProvider(api_key=settings.openai_api_key, base_url=settings.openai_base_url)
        case "anthropic":
            return AnthropicProvider(api_key=settings.anthropic_api_key)
        case other:
            raise ValueError(f"Unknown AI provider: {other}")
```

Inject via a FastAPI dependency (`Depends(get_ai_provider)`) so tests can substitute a fake
provider that returns canned `CompletionResult`s — never hit a real API in unit tests.

## Practical rules

- **Retries/timeouts**: both SDKs accept `timeout=` and have built-in retry on transient
  errors; set explicit timeouts (LLM calls are slow and a hung request holds a worker/socket).
  For anything user-facing, run the call inside an arq job (see `background-jobs.md`) rather
  than blocking a request, unless latency requirements demand synchronous streaming.
- **Structured output**: prefer the provider's native structured-output/tool-calling mode
  (OpenAI's `response_format={"type": "json_schema", ...}`, Anthropic's tool-use with a single
  forced tool) over asking the model to "return JSON" in prose and hoping — parse with a
  Pydantic model and treat a parse failure as a retryable error.
- **Usage/cost logging**: every `CompletionResult` carries token counts — log them (with
  request id, feature, model) so spend is attributable per feature/user, not just a lump sum.
- **Secrets**: `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` come from `pydantic-settings`/environment
  only, same as every other secret in this stack.
- **Don't** let `app/features/*/service.py` import `openai`/`anthropic` directly — always go
  through `AIProvider`. This is also enforceable as an import-linter `forbidden` contract if
  you want it mechanically checked.

## DO NOT

- **Never** import `openai` or `anthropic` directly in feature code — always go through `AIProvider`.
- **Never** hit real LLM APIs in unit tests — use a fake provider with canned `CompletionResult`s.
- **Never** hardcode model names in feature code — use `settings` and pass model as parameter.
- **Never** skip explicit timeouts on LLM calls — hung requests hold worker/socket slots.
- **Never** call LLMs inline in request handlers for user-facing features — use arq jobs (unless streaming).
- **Never** ask the model to "return JSON" in prose — use native structured output / tool-calling mode.
- **Never** ignore token usage/cost — log every `CompletionResult` with request id, feature, model.
- **Never** let `base_url` default to `api.openai.com` — make it configurable for Azure OpenAI, OpenRouter, etc.
- **Never** use `temperature=0.0` by default — let the caller decide based on use case.
- **Never** parse LLM output without validation — use Pydantic and treat parse failure as retryable.
- **Never** expose raw provider responses to clients — extract only what's needed.
- **Never** store API keys in Redis — use a dedicated `secrets` table in Postgres with encryption at rest, or a secrets manager.
- **Never** let feature code know which provider is being used — that's `AIProvider`'s job.
