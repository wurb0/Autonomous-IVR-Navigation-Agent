# Autonomous IVR Navigation Agent

> **Demo status:** The live public demo is currently unavailable because paid API/telephony services (Twilio + model APIs) are not active right now.

Autonomous voice agent for navigating real-world IVR phone trees end-to-end. The system places outbound calls, ingests live telephony audio, classifies call state from ASR + VAD signals, chooses safe actions in real time (DTMF / wait / handoff / patch), and bridges the user when a human representative is reached.

## Architecture Overview

### Core modules

| Module | Responsibility |
|---|---|
| `app/main.py` | FastAPI app, HTTP/WS endpoints, session lifecycle, orchestration loop |
| `app/agent.py` | ASR ingestion (Vosk), VAD analysis, planner call, action guardrails |
| `app/state_machine.py` | Explicit call-state transitions + hold-time accounting |
| `app/metrics.py` | Session KPI recording and aggregate metrics |
| `app/telephony.py` | Twilio operations (outbound call, DTMF, hangup, TwiML builders) |
| `app/audio.py` | Twilio mu-law decode, PCM chunking, RMS energy |
| `app/tts.py` | Optional ElevenLabs handoff narration |
| `app/templates/index.html` + `app/static/app.js` | Operator dashboard + live telemetry stream |

### Runtime topology

```text
Dashboard UI
  -> POST /api/start
  -> Twilio outbound call (business leg)
  -> /twiml/outbound response
  -> Twilio Media Stream -> WS /ws/media
  -> decode + ASR + VAD + state machine
  -> planner action
  -> Twilio execution (DTMF / patch / hangup)
  -> live logs -> WS /ws/ui
  -> session finalize -> metrics store
```

## Real-Time Pipeline

1. **Start request**: `POST /api/start` validates payload and required config.
2. **Session init**: shared `SESSION` object is populated under `SESSION_LOCK`.
3. **Business leg dial**: Twilio outbound call is created with TwiML URL `/twiml/outbound`.
4. **Media attach**: TwiML starts media streaming to `WS /ws/media` and joins conference.
5. **Audio processing**:
   - base64 mu-law payload -> PCM16 decode
   - 200 ms chunking (`3200` bytes)
   - 20 ms frames (`320` bytes) for VAD
   - Vosk partial transcript extraction
6. **State classification**: transcript + speech ratio + energy are fed into `CallStateMachine`.
7. **Planning**: observation snapshot is passed to Gemini planner (or deterministic fallback).
8. **Safe action execution**:
   - `PRESS_DTMF` only with transcript evidence
   - repeat-digit suppression
   - handoff-before-patch rule
9. **Finalize**: on stop/disconnect/hangup, state closes, KPIs are recorded, session resets.

## State Machine

States:

- `IDLE`
- `LISTENING`
- `MENU`
- `HOLD`
- `HUMAN_DETECTED`
- `HANDOFF_READY`
- `PATCHING_USER`
- `BRIDGED`
- `FINISHED`
- `ERROR`

Transition drivers:

- audio observation classification (`apply_audio_observation`)
- planner action events (`on_action`)
- conference bridge event (`on_user_bridged`)
- terminal events (`finish`, `fail`)

## Concurrency Model

- Shared live session state is synchronized with `asyncio.Lock` (`SESSION_LOCK`).
- Blocking SDK/API operations are offloaded with `asyncio.to_thread(...)`.
- WebSocket ingestion, planner cadence, Twilio API calls, and UI broadcast run concurrently under real-time telephony timing constraints.

## API Surface

### HTTP routes

- `GET /` dashboard
- `POST /api/start` start autonomous call session
- `POST /api/stop` stop active session
- `GET /api/status` session/state/config snapshot
- `GET /api/metrics` KPI summary
- `GET /health` config readiness
- `GET /audio/handoff.mp3` generated handoff clip
- `POST /twiml/outbound` TwiML for business leg
- `POST /twiml/join_user` TwiML for user leg

### WebSocket routes

- `WS /ws/media` Twilio media ingress
- `WS /ws/ui` live telemetry for dashboard clients

## Project Structure

```text
Caller/
├── .env.example
├── .gitignore
├── .python-version
├── requirements.txt
├── README.md
└── app/
    ├── main.py
    ├── agent.py
    ├── state_machine.py
    ├── metrics.py
    ├── telephony.py
    ├── tts.py
    ├── audio.py
    ├── static/
    │   └── app.js
    ├── templates/
    │   └── index.html
    └── models/
        └── vosk/
```

## Configuration

Required environment variables:

- `PUBLIC_BASE_URL`
- `TWILIO_ACCOUNT_SID`
- `TWILIO_AUTH_TOKEN`
- `TWILIO_FROM_NUMBER`
- `GEMINI_API_KEY`

Optional:

- `ELEVENLABS_API_KEY`
- `ELEVENLABS_VOICE_ID`
- `VOSK_MODEL_PATH`

## Local Run

```bash
cd /Users/mustafa/Desktop/Uni/Caller
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload --port 8000
```

Open `http://127.0.0.1:8000`.
