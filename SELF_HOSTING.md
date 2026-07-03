# Self-Hosting Guide — Models, Pipeline & Requirements

This repo is the **voice client + a thin FastAPI proxy only**. It contains **no AI models**
and does **no inference**. It captures your mic, streams 16 kHz PCM audio over a WebSocket,
and plays back the audio it receives. Everything intelligent runs on a **separate
speech-to-speech (S2S) server** that you must host.

This document explains: (1) what runs where, (2) the exact models the demo uses and their
Hugging Face repos + custom code, and (3) what you actually need to host.

---

## 1. Architecture — two hops

```
   THIS REPO  (client + proxy)                  YOUR S2S SERVER  (you host this)
 ┌──────────────────────────┐   realtime WS   ┌───────────────────────────────────────────┐
 │ mic → 16 kHz PCM16 → b64  │ ──────────────► │ VAD → STT → VLM (Gemma 4) → TTS            │
 │ noise gate, playback, UI, │                 │        │                                    │
 │ WebSocket client, /api/*  │ ◄────────────── │        └── internal call → your LLM (Q8)    │
 └──────────────────────────┘   24 kHz audio   └───────────────────────────────────────────┘
   http://localhost:7860           ▲                    e.g. ws://5.6.7.8:4567/v1/realtime
                                    │
                    The client only sends `session.update`
                    with { instructions, voice, tools } — it
                    NEVER sends a model name or an LLM API key.
```

### What runs where

| Component | In this repo? | Runs on |
|---|:---:|---|
| Mic capture, 48→16 kHz resample, PCM16 packing | ✅ | Browser ([worklets/mic-capture.js](worklets/mic-capture.js)) |
| Noise gate (RMS volume threshold — **not** VAD) | ✅ | Browser |
| Audio playback (24 kHz), orb UI, WebSocket client | ✅ | Browser |
| Static hosting + `/api/search` + `/api/session` proxy | ✅ | The FastAPI container ([server.py](server.py)) |
| **VAD** · **STT** · **VLM (Gemma 4)** · **TTS** | ❌ | **Your S2S server** |

The client assumes the server does all four stages — see its own comment in
[ws/s2s-ws-client.js:886](ws/s2s-ws-client.js#L886): *"The s2s server already defaults to
server_vad, whisper-1 transcription, 16 kHz PCM input and 24 kHz PCM output."*

---

## 2. The pipeline models (what the demo uses)

Ground truth is [CONTEXT.md:12-25](CONTEXT.md#L12-L25). The demo's pipeline is:

`you speak → VAD → STT → VLM → TTS → orb replies`

| Stage | Model (exact id) | Hugging Face repo | Framework / custom code | Install |
|---|---|---|---|---|
| **VAD** | `silero-vad` | https://huggingface.co/snakers4/silero-vad | Small helper lib (`get_speech_timestamps`); PyTorch **or** ONNX runtime | `pip install silero-vad` — or `torch.hub.load('snakers4/silero-vad', 'silero_vad')` |
| **STT** | `nvidia/parakeet-tdt-1.1b` | https://huggingface.co/nvidia/parakeet-tdt-1.1b | **NVIDIA NeMo** (not `transformers`) | `pip install -U "nemo_toolkit[asr]"` then `nemo_asr.models.EncDecRNNTBPEModel.from_pretrained("nvidia/parakeet-tdt-1.1b")` |
| **VLM** | `google/gemma-4-31B-it` (~30.7B, multimodal, 256K ctx) | https://huggingface.co/google/gemma-4-31B-it | `transformers` / **vLLM** / **llama.cpp** (for Q8). **Gated** — accept the license + use an HF token | `pip install -U transformers accelerate` — or serve GGUF Q8 via `llama.cpp` / vLLM |
| **TTS** | `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice` | https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice | `qwen-tts` package; streaming, style control | `pip install qwen-tts` |

> These installs are for **the S2S server you host** — **not** for this repo. This repo's
> [requirements.txt](requirements.txt) is only `fastapi`, `uvicorn`, `httpx`, `huggingface_hub`.

### Voices (TTS)

The client's default voice is `Aiden` ([main.js:24](main.js#L24)). Qwen3-TTS speakers exposed
by the demo ([README.md:145](README.md#L145)):
`Aiden` (EN male, default), `Ryan` (EN male), `Dylan`, `Eric`, `Ono_Anna` (JA), `Serena`,
`Sohee` (KO), `Uncle_Fu`, `Vivian`. The client sends only the **speaker name** in
`session.audio.output.voice` — the TTS model must know these names.

---

## 3. ⚠️ Important: the demo's STT is Parakeet, NOT Whisper

You mentioned you've hosted **whisper-large-v3-turbo**. Note:

- **The demo spec uses `nvidia/parakeet-tdt-1.1b` for STT**, not Whisper. The string
  `whisper-1` in the client is just the **OpenAI Realtime protocol's transcription-model
  field label** — a protocol constant, not the model actually loaded.
- Because **your S2S server owns the STT choice** (the client never specifies it), you *can*
  use Whisper instead of Parakeet — your server just has to feed 16 kHz PCM in and emit
  `conversation.item.input_audio_transcription.*` events. Functionally fine; simply a
  different model than the reference demo.

### Which Whisper is "turbo v3"? (clarifying your question)

There is **no separate "turbo v3 large"** — *turbo is derived from large-v3*:

| Model | Repo | Params | Decoder layers | Notes |
|---|---|---|---|---|
| `openai/whisper-large-v3` | https://huggingface.co/openai/whisper-large-v3 | 1,550M | 32 | Full accuracy, slower |
| `openai/whisper-large-v3-turbo` ← **"turbo v3"** | https://huggingface.co/openai/whisper-large-v3-turbo | 809M | **4** | Same model, decoder pruned 32→4 layers → much faster, minor quality loss. Best for realtime. |

So "turbo v3" = `whisper-large-v3-turbo`. "large-v3" is the full one. Pick **turbo** for
low-latency realtime voice.

---

## 4. What you need to run the whole thing

1. **This repo (the client)** — already containerized. Run:
   ```bash
   docker compose up --build      # → http://localhost:7860
   ```
   It ships in **direct mode** with `DEFAULT_S2S_URL` pre-filled (see
   [docker-compose.yml](docker-compose.yml)); point it at your S2S server's
   `/v1/realtime` WebSocket.

2. **Your S2S server (host separately)** — must implement the **OpenAI Realtime GA WebSocket
   protocol** and internally run VAD + STT + VLM + TTS. The contract the client depends on
   ([ws/s2s-ws-client.js:7-18](ws/s2s-ws-client.js#L7-L18)):
   - Accepts a WebSocket at `…/v1/realtime`
   - Reads `input_audio_buffer.append` (base64 PCM16 **16 kHz** mono)
   - Reads `session.update` with `{ instructions, voice, tools }`
   - Emits `response.output_audio.delta` (PCM16 **24 kHz**) + transcript events
   - Emits `response.function_call_arguments.done` when the model calls a tool
   - **Owns the model choice** (Gemma 4 Q8) and any upstream LLM base-URL / key —
     that config lives on **this** server, never in the client.

   A ready-made orchestrator that wires these stages behind `/v1/realtime` is Hugging Face's
   open **`speech-to-speech`** project (https://github.com/huggingface/speech-to-speech) —
   the reference backend this demo was built against.

3. **Your LLM endpoint (Gemma 4 Q8)** — the S2S server calls this as its VLM stage
   (e.g. `POST http://5.6.7.8:4567/v1/chat/completions`, model = your Gemma 4 Q8, key = none).
   The **voice client can't reach this directly** — it's behind the S2S server.

### Verify your S2S host speaks the right protocol

```bash
# 101 Switching Protocols = good (realtime WS). 404/426 = wrong endpoint or a text-only LLM.
curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZQ==" \
  http://5.6.7.8:4567/v1/realtime
```

If that 404s, `5.6.7.8:4567` is only your **Gemma text endpoint** and you still need the
VAD+STT+TTS orchestrator (hop #2) in front of it.
