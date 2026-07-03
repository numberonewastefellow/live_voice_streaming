---
title: HF Realtime Voice
emoji: 🎙️
colorFrom: indigo
colorTo: purple
sdk: docker
app_port: 7860
pinned: false
short_description: Voice chat over WebSocket against a HF speech-to-speech
hf_oauth: true
---

# Minimal Conversation App (S2S backend, **WebSocket** transport)

> **Self-hosting / models / requirements → see [SELF_HOSTING.md](SELF_HOSTING.md).**
> This repo is the **client + proxy only** — it hosts no models. VAD, STT
> (`nvidia/parakeet-tdt-1.1b`), the Gemma 4 VLM, and TTS
> (`Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice`) all run on a separate S2S server you host.

Drop-in alternative to [`amir-tfrere/minimal-conversation-app-s2s-backend`](https://huggingface.co/spaces/amir-tfrere/minimal-conversation-app-s2s-backend)
that uses the **WebSocket** route of the Hugging Face speech-to-speech
backend instead of the WebRTC SDP proxy. Same load balancer, same
`/session` handshake, same UI, same orb. Just a different wire.

---

## 📋 Handoff summary — read this first

**What this repo is:** the **voice client + a thin FastAPI proxy**. It captures the mic,
streams 16 kHz PCM audio over a WebSocket, and plays back the audio it receives.

**What this repo is NOT:** it contains **no AI models** and does **no inference**. VAD, STT,
the Gemma 4 VLM, and TTS all run on a **separate speech-to-speech (S2S) server you host**.

### Two-hop architecture

```
   THIS REPO  (client + proxy)                  YOUR S2S SERVER  (host separately)
 ┌──────────────────────────┐   realtime WS   ┌───────────────────────────────────────────┐
 │ mic → 16 kHz PCM16 → b64  │ ──────────────► │ VAD → STT → VLM (Gemma 4 Q8) → TTS         │
 │ noise gate, playback, UI, │                 │                │                            │
 │ WebSocket client, /api/*  │ ◄────────────── │                └─ internal call → your LLM  │
 └──────────────────────────┘   24 kHz audio   └───────────────────────────────────────────┘
   http://localhost:7860        ▲                   e.g. ws://HOST:PORT/v1/realtime
                                │
              Client sends only `session.update` = { instructions, voice, tools }.
              It NEVER sends a model name or an LLM API key — the server owns those.
```

### What runs where

| Component | In this repo? | Runs on |
|---|:---:|---|
| Mic capture, 48→16 kHz resample, PCM16 packing | ✅ | Browser (`worklets/mic-capture.js`) |
| Noise gate (RMS volume threshold — **not** VAD) | ✅ | Browser |
| Audio playback (24 kHz), orb UI, WebSocket client | ✅ | Browser |
| Static hosting + `/api/search` + `/api/session` proxy | ✅ | FastAPI container (`server.py`) |
| **VAD · STT · VLM (Gemma 4) · TTS** | ❌ | **Your S2S server** |

---

## 🔌 Endpoints to update

Everything a deployer needs to repoint. **Only rows 1–3 live in this repo**; rows 4–6 are on
the servers you host.

| # | Endpoint / setting | Where to change it | Placeholder / example | Set it to |
|---|---|---|---|---|
| 1 | **S2S realtime WebSocket** (direct mode default) | `DEFAULT_S2S_URL` env in [`docker-compose.yml`](docker-compose.yml) — or **Settings → server URL** in the UI | `ws://<s2s-host>:8765/v1/realtime` (`speech-to-speech` default port is **8765**) | Your S2S server's realtime WS. Use `ws://` for plain, `wss://` for TLS. Path is `/v1/realtime`. |
| 2 | **Load balancer** (optional; LB mode instead of direct) | `LOAD_BALANCER_URL` env | _(empty)_ e.g. `https://…endpoints.huggingface.cloud` | Your LB base URL; the app POSTs `<LB>/session`. Leave empty to use direct mode (row 1). |
| 3 | **Web search key** (optional tool) | `SERPER_API_KEY` env | _(empty)_ | A [serper.dev](https://serper.dev) key, or leave empty to disable web search. |
| 4 | **S2S server realtime route** (the contract row 1 dials) | Inside **your S2S server** | `…/v1/realtime` (WebSocket upgrade) | Must accept `input_audio_buffer.append` (16 kHz PCM16) + `session.update`, and emit `response.output_audio.delta` (24 kHz). |
| 5 | **LLM / Gemma 4 Q8 endpoint** (the VLM stage) | Inside **your S2S server's** config — **NOT this repo** | e.g. `http://5.6.7.8:4567/v1/chat/completions`, model = Gemma 4 Q8, key = none | Your OpenAI-compatible LLM endpoint. The voice client cannot reach this directly — it sits behind the S2S server. |
| 6 | **STT / TTS / VAD models** | Inside **your S2S server** | Parakeet / Qwen3-TTS / silero (see below) | Whatever models your S2S server loads. The client is model-agnostic. |

> **Client-served API routes** (usually not changed): `GET /api/config`, `GET /api/me`,
> `POST /api/search`, `POST /api/session`, `GET|DELETE /api/queue/{id}`, `POST /api/queue/end`,
> `POST /api/session/heartbeat`, `POST /api/session/end`, `/*` static. Defined in
> [`server.py`](server.py).

### Verify an S2S host speaks the right protocol

```bash
# 101 Switching Protocols = good (realtime WS). 404/426 = wrong endpoint or a text-only LLM.
curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZQ==" \
  http://HOST:PORT/v1/realtime
```

If that 404s, the host is only a **text LLM** and still needs the VAD+STT+TTS orchestrator
(the S2S server) in front of it — the voice client cannot talk to a `/v1/chat/completions`
endpoint directly.

---

## 🧠 Pipeline models (what the demo uses)

Source of truth: [`CONTEXT.md`](CONTEXT.md). Pipeline order: `you speak → VAD → STT → VLM → TTS
→ orb replies`. All verified live on Hugging Face. **These install on the S2S server you host,
not on this repo** (this repo needs only `fastapi/uvicorn/httpx/huggingface_hub`).

| Stage | Model (exact id) | Hugging Face repo | Framework / custom code | Install |
|---|---|---|---|---|
| **VAD** | `silero-vad` | [snakers4/silero-vad](https://huggingface.co/snakers4/silero-vad) | Small helper lib; PyTorch **or** ONNX | `pip install silero-vad` (or `torch.hub.load('snakers4/silero-vad','silero_vad')`) |
| **STT** | `nvidia/parakeet-tdt-1.1b` | [nvidia/parakeet-tdt-1.1b](https://huggingface.co/nvidia/parakeet-tdt-1.1b) | **NVIDIA NeMo** (not `transformers`) | `pip install -U "nemo_toolkit[asr]"` |
| **VLM** | `google/gemma-4-31B-it` (~30.7B, multimodal, 256K ctx) | [google/gemma-4-31B-it](https://huggingface.co/google/gemma-4-31B-it) | `transformers` / **vLLM** / **llama.cpp** (for Q8). **Gated** — accept license + HF token | `pip install -U transformers accelerate`, or serve GGUF Q8 via llama.cpp/vLLM |
| **TTS** | `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice` | [Qwen3-TTS-12Hz-1.7B-CustomVoice](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice) | `qwen-tts` package; streaming, style control | `pip install qwen-tts` |

**Voices (TTS):** default `Aiden` (`main.js`). Speakers: `Aiden`, `Ryan`, `Dylan`, `Eric`,
`Ono_Anna`, `Serena`, `Sohee`, `Uncle_Fu`, `Vivian`. The client sends only the **speaker name**
in `session.audio.output.voice`.

### ⚠️ STT is Parakeet, NOT Whisper

The demo spec uses **`nvidia/parakeet-tdt-1.1b`** for STT. The `whisper-1` string in the client
([`ws/s2s-ws-client.js`](ws/s2s-ws-client.js)) is just the **OpenAI Realtime protocol's
transcription-field label** — a constant, not the model actually loaded. Because **your S2S
server owns the STT choice** (the client never sends it), you *can* swap in Whisper instead —
functionally fine, just a different model than the reference demo.

**Whisper variants (for reference):** there is no separate "turbo v3 large" — *turbo is derived
from large-v3*:

| Model | Repo | Params | Decoder layers |
|---|---|---|---|
| `whisper-large-v3-turbo` ← **"turbo v3"** | [openai/whisper-large-v3-turbo](https://huggingface.co/openai/whisper-large-v3-turbo) | 809M | **4** (pruned) → fast, best for realtime |
| `whisper-large-v3` (full) | [openai/whisper-large-v3](https://huggingface.co/openai/whisper-large-v3) | 1,550M | 32 → most accurate, slower |

A ready-made orchestrator that wires all four stages behind `/v1/realtime` is Hugging Face's
open [`speech-to-speech`](https://github.com/huggingface/speech-to-speech) project — the
reference backend this demo was built against.

> Deeper self-hosting notes: [SELF_HOSTING.md](SELF_HOSTING.md).

---

## 🚀 Hosting the S2S server — `huggingface/speech-to-speech`

**Yes — you can clone/install and host it as-is.** It is the reference orchestrator this demo
was built against, and its **defaults match this pipeline** (Silero VAD v5, Parakeet TDT STT,
Qwen3-TTS, OpenAI-compatible LLM). It exposes the exact `/v1/realtime` WebSocket this client
needs. Repo: <https://github.com/huggingface/speech-to-speech>.

**Install & run (realtime mode is the default):**

```bash
pip install speech-to-speech          # or: git clone + docker compose up
speech-to-speech --mode realtime       # serves ws://0.0.0.0:8765/v1/realtime
# Docker: git clone the repo, then `docker compose up` from its root
```

**Point its LLM stage at your Gemma 4 Q8** (this is where your hosted LLM plugs in):

```bash
speech-to-speech --mode realtime \
  --responses_api_base_url "http://5.6.7.8:4567/v1" \   # your Gemma 4 Q8 OpenAI-compatible endpoint
  --responses_api_api_key  "none" \                     # your key, or a placeholder if none
  --model_name             "<your-gemma-4-q8-id>" \     # model id your endpoint serves
  --stt parakeet-tdt \                                   # STT runs INSIDE this server (loaded locally)
  --tts qwen3 --qwen3_tts_model_name Qwen/Qwen3-TTS-12Hz-1.7B \
  --ws_port 8765
```

Then set this client's **`DEFAULT_S2S_URL=ws://<s2s-host>:8765/v1/realtime`**
([`docker-compose.yml`](docker-compose.yml)) and it connects.

### ⚠️ Important — which parts are external vs in-process

| Stage | Where it runs under `speech-to-speech` | Your hosted models |
|---|---|---|
| **LLM (Gemma 4 Q8)** | **External** — via `--responses_api_base_url` | ✅ Plugs in directly (this is the one external hop) |
| **STT (Parakeet)** | **In-process** — loaded locally, needs GPU/compute on the S2S box | ⚠️ Your separately-hosted **Whisper is NOT used** unless you configure STT to it; the S2S box loads its own STT |
| **TTS (Qwen3-TTS)** | **In-process** — loaded locally | — |
| **VAD (Silero v5)** | **In-process** — loaded locally | — |

So: hosting `speech-to-speech` as-is gives you a **GPU box** that runs VAD+STT+TTS locally and
calls your Gemma over HTTP. It is **not** "just a proxy" — it needs hardware for the STT/TTS
models. Hardware: Apple Silicon (MLX, `--local_mac_optimal_settings`), CUDA/Linux, or CPU
fallback.

> **Two ports, don't conflate them:** the **S2S realtime WebSocket is `:8765`** (what the
> browser dials); your **Gemma LLM endpoint is a different host/port** (e.g. `:4567/v1`,
> reached only by the S2S server, never the browser).

---

## How it works

1. App POSTs `<lb_url>/session` (empty JSON body).
2. The LB picks a ready compute (round-robin) and returns:
   ```json
   {
     "session_id": "...",
     "websocket_url": "wss://<compute>/v1/realtime",
     "connect_url": "wss://<compute>/v1/realtime?session_token=<JWT>",
     "session_token": "<JWT>",
     "pending_timeout_s": 60
   }
   ```
3. App opens a WebSocket **directly** on `connect_url` (no rewrite to
   `https://`; unlike the WebRTC client which POSTs an SDP offer).
4. Server pushes `session.created` on connect. Client replies with
   `session.update` (OpenAI Realtime **GA** schema: `session.audio.input`,
   `session.audio.output`, `session.output_modalities`).
5. Client streams mic audio as PCM16 16 kHz mono base64 chunks
   (`input_audio_buffer.append`, one frame every ~40 ms).
6. Server pushes `response.output_audio.delta` (PCM16 24 kHz mono base64)
   and transcript deltas.

The backend exposes one concurrent session per compute (same as WebRTC
mode); the LB pins the session via a signed `session_token`.

## Why WebSocket instead of WebRTC

| | WebRTC (original) | WebSocket (this) |
|---|---|---|
| Transport | UDP + Opus 48 kHz + ICE/STUN | TCP + raw PCM16 |
| NAT traversal | needs STUN, can fail on corporate / cellular | none, works everywhere TCP is allowed |
| Audio quality | excellent (Opus, jitter buffer, FEC) | good (raw PCM, simple ring buffer) |
| Latency | lowest (~50-150 ms) | low (~150-300 ms typical) |
| Echo cancellation | browser AEC active on the WebRTC track | browser AEC active via `getUserMedia` constraints |
| Debuggability | needs `chrome://webrtc-internals` | `wscat` / DevTools network tab |
| Mobile data | sometimes blocked (UDP) | always works (HTTPS+WSS) |

## Backend requirement

This app talks to the WebSocket route `@app.websocket("/v1/realtime")`
defined in
[`websocket_router.py`](https://github.com/huggingface/speech-to-speech/blob/feat/webrtc-transport/src/speech_to_speech/api/openai_realtime/websocket_router.py)
on the **`feat/webrtc-transport`** branch. The same compute serves both
the WebRTC POST and the WebSocket upgrade on the same path; no backend
change required.

Smoke-test from the shell:

```bash
LB="https://kaa1l6rplzb1gg3y.us-east-1.aws.endpoints.huggingface.cloud"
curl -X POST "$LB/session" -H "Content-Type: application/json" -d '{}'
# -> { "connect_url": "wss://<compute>/v1/realtime?session_token=..." }
# Feed connect_url into a wscat / websocat and you should get a
# session.created event back immediately.
```

## Tools

The assistant can call two tools mid-conversation (toggle them from the **Tools**
button, top-right):

- **Web search** — Google results via Serper.dev, proxied server-side so the key
  never reaches the browser. Set `SERPER_API_KEY` as a Space secret. Without it,
  the tool is disabled unless the user pastes their own key in the Tools panel.
- **Camera** — while enabled, a live self-view shows bottom-left; when the model
  calls the tool, the current frame is sent to the vision-language model so it can
  see what you're showing it.

## Connecting to a backend

The app connects **directly** to a speech-to-speech server's realtime WebSocket —
no load balancer, no `/session` step. Set the URL in two ways:

- **`LOAD_BALANCER_URL` env** (served via `/api/config`) provides the default URL
  shown in Settings — handy for the deployed Space.
- **Settings → Speech-to-speech server URL** lets you override it: paste a full
  `connect_url` (`wss://host/v1/realtime?...`) or a bare host like `localhost:8080`
  (the app adds `/v1/realtime`).

**Settings → Restart** reconnects with the current voice, instructions and URL.

## Usage limits

Conversation time is metered per UTC day by sign-in tier (see `limiter.py` /
`auth.py`), but **only on the deployed Space** — metering turns on only when BOTH
`LOAD_BALANCER_URL` and `SPACE_ID` (injected automatically by the HF Space
runtime) are present. Running locally — even with `LOAD_BALANCER_URL` exported —
leaves the app unmetered. Tunable via env:

| Env | Default | What |
|-----|---------|------|
| `LIMIT_ANON_SEC` | `300` | Daily seconds for anonymous visitors (5 min) |
| `LIMIT_FREE_SEC` | `600` | Daily seconds for signed-in non-PRO users (10 min) |
| `UNLIMITED_ORGS` | _(adds to defaults)_ | Extra HF org names whose members get **unlimited** usage, like PRO |
| `USAGE_HASH_SECRET` | _(random)_ | HMAC secret for hashing identity keys + signing the anon cookie |

PRO members are always unlimited. Members of `cerebras`, `HuggingFaceM4`,
`smolagents`, and `pollen-robotics` are unlimited out of the box (shown as
"Team", not "PRO"); set `UNLIMITED_ORGS=my-team` to add more. Matched
case-insensitively against the user's organisations from HF OAuth.

## Run locally

The app is now a small FastAPI server (it serves the front-end *and* the search
proxy from one container).

```bash
pip install -r requirements.txt
export SERPER_API_KEY=...        # optional; web search is disabled without it
export LOAD_BALANCER_URL=...     # optional; default s2s server URL (set it in Settings otherwise)
uvicorn server:app --reload --port 7860
# or, matching production: docker build -t s2s . && docker run -p 7860:7860 -e SERPER_API_KEY=... -e LOAD_BALANCER_URL=... s2s
```

Then open <http://localhost:7860/>, click the orb, allow the mic, talk.

> Browsers require **HTTPS or `localhost`** for `getUserMedia()` (mic + camera).
> `127.0.0.1` and `localhost` both work; plain `http://192.168.x.y` does NOT.

## Settings (stored in `localStorage`)

| Key | What |
|-----|------|
| Load balancer URL | Base URL of your S2S deployment. App POSTs `<lb>/session`. |
| Voice | Qwen3-TTS speaker name (Aiden, Ryan, Dylan, Eric, Ono_Anna, Serena, Sohee, Uncle_Fu, Vivian) |
| Instructions | System prompt sent in `session.update` once the WS opens |

LocalStorage keys are namespaced `s2s.ws.*` so this app's settings do
NOT collide with the WebRTC variant.

## Files

| File | Role |
|------|------|
| `index.html` | Single page, orb + settings modal (identical UI to the WebRTC app) |
| `main.js` | State machine, settings, tools, camera, noise-gate UI wiring |
| `ui/chat.js` | `ChatView`: history panel, ephemeral bubbles, transcript/tool streaming |
| `ui/account.js` | `Account`: HF login chip + popover, daily-limit modal |
| `ui/dom.js` | Shared helpers: `$`, `escHtml`, `truncateError`, `DEBUG` |
| `auth.py` | HF OAuth + per-request identity (tier, hashed keys) |
| `limiter.py` | SQLite per-day talk-time budget (chunked server-clock reservation) |
| `ws/s2s-ws-client.js` | WebSocket handshake + OpenAI Realtime GA protocol |
| `ws/codec.js` | base64 <-> PCM helpers + transcript extraction (pure) |
| `ws/orb-visualizer.js` | `OrbVisualiser`: FFT bands -> orb CSS custom properties |
| `worklets/mic-capture.js` | AudioWorklet: 48 kHz Float32 -> 16 kHz Int16 PCM, posts ~40 ms chunks |
| `worklets/audio-playback.js` | AudioWorklet: 24 kHz Float32 ring buffer -> 48 kHz, linear interp, fade in/out |
| `style.css` | Orb animations, layout, dark theme (verbatim from the WebRTC app) |

## Audio pipeline notes

- **Input**: `getUserMedia({ echoCancellation, noiseSuppression, autoGainControl })`
  feeds the `mic-capture` worklet at the `AudioContext` rate. The worklet
  resamples to 16 kHz (boxcar lowpass + decimation on the 48 -> 16 fast
  path, linear interpolation fallback for odd rates) and packs Int16 LE.
- **Output**: `response.output_audio.delta` decodes to Int16 -> Float32
  and is posted to the `audio-playback` worklet. The worklet maintains a
  per-context ring buffer, linearly interpolates 24 -> 48, and applies
  short 32-frame fades on entry/exit to suppress clicks.
- **Barge-in**: when the server VAD detects user speech mid-response
  (`input_audio_buffer.speech_started` while `ai-speaking`), the client
  posts `{ kind: "clear" }` to the playback worklet to wipe the queue
  immediately. The server itself cancels the in-flight response.

## Credits

- Backend: [huggingface/speech-to-speech](https://github.com/huggingface/speech-to-speech) on `feat/webrtc-transport`
- UI verbatim from `amir-tfrere/minimal-conversation-app-s2s-backend` (Pollen Robotics × Hugging Face)
