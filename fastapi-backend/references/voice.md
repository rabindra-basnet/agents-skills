# Voice

Voice agents that listen, understand, and respond in real-time. Use WebSockets for streaming
audio and integrate with STT/TTS providers (Deepgram, ElevenLabs, OpenAI).

## Architecture

```
Client (mic) → WebSocket → Server → STT → LLM → TTS → WebSocket → Client (speaker)
```

## WebSocket endpoint

```python
# app/features/voice/router.py
import asyncio
from fastapi import APIRouter, WebSocket, WebSocketDisconnect
from app.features.voice.stt import SpeechToText
from app.features.voice.tts import TextToSpeech
from app.core.ai.factory import get_ai_provider
from app.core.logging import get_logger

logger = get_logger(__name__)
router = APIRouter()

@router.websocket("/ws/voice")
async def voice_ws(websocket: WebSocket):
    await websocket.accept()
    stt = SpeechToText()
    tts = TextToSpeech()
    ai = get_ai_provider()

    try:
        while True:
            audio_chunk = await websocket.receive_bytes()

            # STT: audio → text
            text = await stt.transcribe(audio_chunk)
            if not text:
                continue

            logger.info("Voice input", extra={"text": text[:100]})

            # LLM: text → response
            result = await ai.complete(
                messages=[{"role": "user", "content": text}],
                model="gpt-4o",
            )

            # TTS: response → audio
            audio = await tts.synthesize(result.text)

            await websocket.send_bytes(audio)

    except WebSocketDisconnect:
        logger.info("Voice session ended")
```

## STT provider

```python
# app/features/voice/stt.py
import httpx
from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)

class SpeechToText:
    async def transcribe(self, audio: bytes) -> str:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                "https://api.deepgram.com/v1/listen",
                headers={"Authorization": f"Token {settings.deepgram_api_key}"},
                content=audio,
                params={"model": "nova-2", "language": "en"},
                timeout=30,
            )
            resp.raise_for_status()
            return resp.json()["results"]["channels"][0]["alternatives"][0]["transcript"]
```

## TTS provider

```python
# app/features/voice/tts.py
import httpx
from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)

class TextToSpeech:
    async def synthesize(self, text: str) -> bytes:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                "https://api.elevenlabs.io/v1/text-to-speech/{settings.elevenlabs_voice_id}",
                headers={"xi-api-key": settings.elevenlabs_api_key},
                json={"text": text, "model_id": "eleven_monolingual_v1"},
                timeout=30,
            )
            resp.raise_for_status()
            return resp.content
```

## DO NOT

- **Never** process voice synchronously in HTTP handlers — use WebSockets for streaming.
- **Never** store raw audio logs — they contain PII. Log transcripts only.
- **Never** skip VAD (voice activity detection) — don't process silence.
- **Never** hardcode STT/TTS provider API keys — use `pydantic-settings`.
- **Never** skip rate limiting on voice endpoints — they're expensive.
- **Never** run LLM calls inline without timeout — voice requires low latency.
- **Never** forget to close WebSocket connections on errors.
- **Never** expose raw audio to other services — transcribe first.
