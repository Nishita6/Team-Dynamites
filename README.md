# Mechatronics Lab Voice Assistant

A voice assistant for a hands-busy mechatronics lab technician: ask a
hardware spec question out loud, get a spoken answer grounded in a real
database - and interrupt it mid-answer without it losing the thread.

The hard voice-engineering problem this project is scoped around:
**interruption & recovery, combined with conversation continuity during tool
work (SQL-RAG lookups)**. See `architecture/architechture.md` for the full
build spec and scope discipline (P0/P1/P2) this repo follows.

## 1. Setup instructions

```bash
cd backend
pip install -r requirements.txt
cp ../.env.example ../.env   # fill in real values, see §3 below
python seed_demo_db.py       # seeds sensors/actuators demo tables
python generate_fallback_audio.py   # pre-caches the offline fallback apology clip

# reproducible, no mic needed:
python ../evidence/stress_test.py -n 10

# once RIME_* / DEEPGRAM_API_KEY / LIVEKIT_* are filled in .env, calibrate
# the truncation constant against the exact voice used in the demo:
python ../evidence/measure_wps.py

# run the live agent (needs a LiveKit room + a client to join it):
python livekit_agent.py dev
```

Dashboard:

```bash
cd dashboard
npm install
npm run dev
```

## 2. Architecture

```
Mic (hands-busy user)
  -> LiveKit Agents: WebRTC transport, VAD, barge-in detection
  -> Deepgram STT
  -> orchestrator.py
       - every user turn gets a monotonic generation_id
       - any SQL-RAG result that resolves under a stale generation_id is
         dropped, never passed to the LLM or TTS
       - on barge-in: elapsed_playback_seconds x measured_words_per_second =
         words actually heard; conversation history is sliced to exactly
         that point and tagged [interrupted]
       - the orchestrator keeps accepting new user turns while Rime is
         speaking and while sql_rag_chain.py is still running - that's the
         part generation-ID fencing exists to make safe
  -> LLM: turn reasoning, SQL generation, answer formatting (never reasons
     over unplayed/dropped content)
  -> Rime TTS (LiveKit's official Rime plugin) -> playback
     (fallback: backend/fallback_tts.py plays a pre-cached local apology
     clip via raw LiveKit audio publish if Rime is unreachable)
  -> Telemetry -> Firebase Firestore -> Next.js dashboard
```

Repository layout:

```
backend/
  livekit_agent.py       LiveKit Agents entrypoint (VAD, turn handling, barge-in)
  orchestrator.py         generation-ID fencing, audio-truncation, turn state machine
  sql_rag_chain.py        NL -> validated SELECT-only SQL -> DB -> answer text
  llm_client.py           shared LLM call wrapper (used by sql_rag_chain + orchestrator)
  telemetry.py            Firestore event logging (fenced_drop, active_speech_provider, turns)
  fallback_tts.py         pre-cached local apology playback when Rime is unreachable
  generate_fallback_audio.py  one-time offline synthesis of the fallback clip
  seed_demo_db.py          seeds sensors/actuators tables
  config.py                env loading, ALLOWED_TABLES, RIME config constants
evidence/
  RIME_EVIDENCE.md         claim, acceptance test, procedure, result, limitations
  measure_wps.py            calibrates words-per-second for truncation math
  stress_test.py             scripted, reproducible interruption stress test
dashboard/                 Next.js live telemetry view backed by Firestore
docs/demo_script.md        4-5 min recording script mapped to the rubric
```

Two small additions beyond the file list in `architecture/architechture.md`:
`backend/llm_client.py` and `backend/telemetry.py` factor out the "call the
LLM" and "write a Firestore event" primitives that both `sql_rag_chain.py`
and `orchestrator.py` need, rather than duplicating them. `backend/fallback_tts.py`
and `backend/generate_fallback_audio.py` implement the fallback speech path
that architecture §3 requires be decided and built, just not named as its
own file there.

## 3. Third-party services

| Service | Role |
|--|--|
| LiveKit Agents | WebRTC transport, VAD, turn handling, barge-in detection |
| Deepgram | Speech-to-text |
| Rime | Text-to-speech (via LiveKit's official Rime plugin) |
| An LLM API (Anthropic by default, via `LLM_API_KEY`/`LLM_MODEL`) | Turn reasoning, SQL generation, answer formatting |
| SQLite | Demo hardware-spec database (`sensors`, `actuators`) |
| Firebase Firestore | Telemetry: `fenced_drop` events, `active_speech_provider`, turn/stress-test events |
| Next.js (`dashboard/`) | Live telemetry view backed by Firestore |

**Exact Rime configuration used in the recorded demo** (architecture §3 -
must match `.env` and pass organizer preflight; fill in after pulling live
values from Rime's catalog and testing end-to-end):

| Field | Value |
|--|--|
| Model ID | *TODO - see `RIME_MODEL_ID` in `.env`* |
| Speaker | *TODO - see `RIME_SPEAKER` in `.env`* |
| Language | *TODO - see `RIME_LANGUAGE` in `.env`* |
| Endpoint / region | *TODO - see `RIME_ENDPOINT` in `.env`* |
| Audio format | *TODO - see `RIME_AUDIO_FORMAT` in `.env`* |
| Transport | LiveKit Agents (WebRTC) via the official Rime LiveKit plugin |

## 4. Known limitations

- Audio-truncation math (`orchestrator.py`, architecture §2.2) assumes a
  roughly constant word-rate within an utterance; it doesn't yet account for
  SSML pauses or punctuation-driven pacing.
- SQL-RAG's allow-list (`config.ALLOWED_TABLES`) covers only `sensors` and
  `actuators`. Extending the schema means updating that list - a real
  scaling constraint, not just a demo shortcut.
- `evidence/stress_test.py`'s default run is non-mic (drives the orchestrator
  directly) for speed and reproducibility; a live-mic confirmation pass is
  recommended before the recorded demo.
- The fallback TTS path has only been reviewed at the code level, not yet
  exercised against a real Rime outage (see `evidence/RIME_EVIDENCE.md`).
- P1 (vector RAG over manuals) and P2 (reservation/checkout) are not part of
  this build unless explicitly added later - see architecture §8-9.

## 5. Failure behavior

- **SQL-RAG failure** (`UnsafeSQLError` or a DB error): the agent says so
  aloud (`FAILURE_SPEECH` in `backend/livekit_agent.py`) - it never stays
  silent and never guesses at a hardware spec.
- **Rime unreachable**: `backend/fallback_tts.py` plays a pre-cached local
  apology clip (generated once by `generate_fallback_audio.py`) by
  publishing raw audio to the room directly, bypassing the TTS plugin that
  just failed. `active_speech_provider` is logged to Firestore as
  `"fallback"` so the dashboard shows which provider is actually speaking.
  Rime remains the default path in the judged flow.
- **Stale background work** (a SQL-RAG or TTS result that resolves after a
  newer turn has already started): fenced and dropped via
  `orchestrator.py`'s generation-ID check, logged as a `fenced_drop` event,
  never spoken.

## 6. Exact Rime model, speaker, language, endpoint, audio format, transport

See the table in §3 above - kept there rather than duplicated so it can't
drift out of sync.
