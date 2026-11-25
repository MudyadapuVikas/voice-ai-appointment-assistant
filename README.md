# Halcyon — Voice AI doctor appointment assistant

A caller talks to **Mira**, a voice receptionist, from the browser. Mira checks doctors' real availability, books appointments and cancels them. Callers are identified **only by the phone number they say**, and Mira reads it back for confirmation before any change. A dashboard shows doctors, slots, appointments and activity, and updates live as Mira works.

Built for the NexovAI SDE Intern Round 1 assignment.

![Overview](docs/overview.png)

| | |
|---|---|
| **Voice pipeline** | Pipecat 1.17 (Python), SmallWebRTC transport, Silero VAD |
| **Speech-to-text** | ElevenLabs Scribe Realtime |
| **LLM** | Google Gemini (`gemini-3.6-flash` by default, set with `GOOGLE_MODEL`), free tier |
| **Text-to-speech** | ElevenLabs Flash v2.5 |
| **API** | FastAPI, Server-Sent Events for live updates |
| **Data** | SQLite (persistent, file in `backend/data/`) |
| **Frontend** | React 19, TypeScript, Vite, Tailwind CSS v4, Lucide icons |

---

## Architecture

```
 Browser (React)                                   Backend (FastAPI + Pipecat)
┌──────────────────────┐   WebRTC audio + data    ┌──────────────────────────────────────────────┐
│ Phone call screen    │ ───────────────────────▶ │ SmallWebRTCTransport                         │
│  mic ─────────────── │   POST/PATCH /api/offer  │   ↓ Silero VAD (turn-taking, barge-in)       │
│  speaker ◀────────── │ ◀─────────────────────── │   ↓ ElevenLabs STT (Scribe Realtime)         │
│  transcript, status  │   RTVI messages          │   ↓ Gemini  ── function calls ──▶ tools      │
│                      │                          │   ↓ ElevenLabs TTS                 │          │
│ Dashboard pages      │   REST /api/*            │                                    ▼          │
│  Overview · Doctors  │ ◀──────────────────────▶ │  services (availability, appointments)       │
│  Appointments ·      │   SSE /api/events        │      ↓                    ↓ publish event     │
│  Activity            │ ◀─────────────────────── │   SQLite               EventBus ─────────────┘
└──────────────────────┘                          └──────────────────────────────────────────────┘
```

* **One code path for every change.** The voice tools and the dashboard's demo actions call the same `services/` functions, so the business rules (confirmation, double booking, validation) are in one place.
* **Live updates.** Every booking or cancellation publishes an event. The dashboard receives it over SSE, refetches, highlights the row and slot, shows a toast, and shows a confirmation card inside the phone if a call is in progress.

## Folder structure

```
.
├── .env.example                 placeholder keys only
├── backend/
│   ├── app/
│   │   ├── main.py              FastAPI app, lifespan, routers
│   │   ├── config.py            settings from env, voice credential check
│   │   ├── agents/
│   │   │   ├── receptionist.py  the Pipecat pipeline for one call
│   │   │   └── prompts.py       Mira's persona and rules
│   │   ├── tools/
│   │   │   └── appointment_tools.py   check_doctor_availability, book_appointment, cancel_appointment
│   │   ├── services/
│   │   │   ├── availability.py  doctors, working days, free slots, date parsing
│   │   │   ├── appointments.py  book / cancel / stats / activity
│   │   │   ├── phone.py         spoken phone number → 10 digits
│   │   │   ├── events.py        in-process pub/sub for SSE
│   │   │   └── errors.py        AppointmentError (business-rule failures)
│   │   ├── database/db.py       SQLite schema, seed data
│   │   └── api/
│   │       ├── routes.py        REST, demo-mode actions, SSE stream
│   │       └── webrtc.py        WebRTC signalling (/api/offer)
│   ├── tests/                   45 pytest tests
│   │   └── e2e/                 end-to-end browser call test + vendor emulator
│   ├── scripts/check_voice_services.py   real-key check for ElevenLabs + Gemini
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── hooks/useVoiceCall.ts    Pipecat client: call lifecycle, transcript, agent state
│       ├── lib/micMediaManager.ts   opens the mic for the WebRTC transport (no external CDN)
│       ├── hooks/useClinicData.ts   data loading + SSE subscription
│       ├── components/call/         phone UI, waveform
│       ├── components/layout/       sidebar, top bar, mobile nav
│       ├── components/ui/           primitives, dialog, toasts
│       └── pages/                   Overview, Doctors, Appointments, Activity
└── docs/                        screenshots, submission notes
```

## Prerequisites

* Python **3.11+**
* Node.js **20+**
* Chrome or Edge (microphone + WebRTC)
* An **ElevenLabs** API key and a **Google AI Studio** (Gemini) API key — needed for voice only

## Environment variables

Copy `.env.example` to `.env` in the repo root (or in `backend/`) and fill in:

| Variable | Required | Purpose |
|---|---|---|
| `ELEVENLABS_API_KEY` | for voice | Speech-to-text and text-to-speech |
| `GOOGLE_API_KEY` | for voice | Gemini LLM |
| `GOOGLE_MODEL` | no | Default `gemini-3.6-flash`. Change it if your key's free tier lists a different Flash model. |
| `ELEVENLABS_VOICE_ID` | no | Mira's voice |
| `CLINIC_TZ` | no | Default `Asia/Kolkata` |

Without the two keys the backend still starts. The dashboard and **demo mode** work, and **Try call** explains which key is missing.

## Install and run

**Backend** (terminal 1)

```bash
cd backend
python -m venv .venv
# Windows:      .venv\Scripts\activate
# macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt
cd ..
copy .env.example .env          # macOS/Linux: cp .env.example .env   — then add your keys
cd backend
uvicorn app.main:app --port 7860
```

The first call downloads the Silero VAD model. `backend/data/clinic.db` is created and seeded on first start.

**Frontend** (terminal 2)

```bash
cd frontend
npm install
npm run dev                     # http://localhost:5173
```

Open **http://localhost:5173**, click **Try call** and allow the microphone. Use headphones so Mira doesn't hear herself.

**Check your API keys (do this before recording)**

```bash
cd backend
python -m scripts.check_voice_services
```

It makes three real calls: ElevenLabs TTS speaks a sentence, ElevenLabs STT transcribes it back, and Gemini gets Mira's prompt and tools and must answer with a `check_doctor_availability` call. Each step prints PASS or FAIL with the reason (bad key, unknown model, quota).

**Tests**

```bash
cd backend
python -m pytest -q                 # 45 unit/API tests, ~2 s
python -m tests.e2e.call_e2e        # full browser voice call (see below)
```

## How the Pipecat pipeline works

`app/agents/receptionist.py` builds one pipeline per call:

```
transport.input() → ElevenLabs STT → user aggregator (Silero VAD) → Gemini → ElevenLabs TTS → transport.output() → assistant aggregator
```

1. The browser's `SmallWebRTCTransport` POSTs an SDP offer to `/api/offer`. `api/webrtc.py` answers it with Pipecat's `SmallWebRTCRequestHandler` and starts `run_receptionist()` for that peer connection.
2. Mic audio streams to ElevenLabs Scribe Realtime and comes back as text. **Silero VAD** detects speech and lets the caller interrupt Mira mid-sentence. Pipecat's default smart-turn analyzer then decides when the caller has finished their turn.
3. The text is added to the `LLMContext` and sent to **Gemini** with Mira's system prompt and the three tools.
4. When Gemini calls a tool, Pipecat runs the Python function and feeds the result back to Gemini, which then answers.
5. Gemini's text streams sentence by sentence into **ElevenLabs TTS**, and the audio goes back over WebRTC.
6. Pipecat's **RTVI** observer sends transcripts, speaking state and tool names (never arguments) to the browser over the data channel. That drives the live transcript, "Listening / Thinking / Speaking", and "Checking the schedule…".
7. When the caller hangs up, `on_client_disconnected` cancels the pipeline and the connection is closed.

## The three tools

All three are in `app/tools/appointment_tools.py`. They are thin adapters: they run the service call in a worker thread and return a dict to the LLM. Business-rule failures come back as `{"ok": false, "error": "<code>", "message": ...}` so Mira can explain them, not as crashes.

**`check_doctor_availability(doctor, specialty, date)`**
Matches doctors by name, part of a name, or specialty ("cardiologist" → Cardiology). Returns free slots from the database: inside working hours, not booked, not in the past. With no date it returns the next three open days. If the doctor has no free slot on the requested day, the result includes `works_that_day` and `next_available`, so Mira offers a real alternative instead of making one up.

**`book_appointment(patient_name, phone_number, phone_confirmed, doctor_id, date, time)`**
* Normalises the spoken number ("nine eight four…", "double five", "+91 …") to 10 digits, or returns `invalid_phone`.
* Refuses with `phone_not_confirmed` and the number to read back unless `phone_confirmed` is true. The prompt only allows true after the caller says yes, and the tool enforces it either way.
* Validates the name, date (inside the 7-day window) and time, then re-checks that the slot is free.
* Double booking is blocked twice: an availability check, then a partial unique index `(doctor_id, date, time) WHERE status='booked'` that catches two callers racing for the same slot.

**`cancel_appointment(phone_number, phone_confirmed, appointment_id, cancel_confirmed)`** works in two steps.
1. With `cancel_confirmed=false` it returns the caller's upcoming appointments and changes nothing. Mira says what she found and asks which one to cancel.
2. With the chosen `appointment_id` and `cancel_confirmed=true` it cancels that appointment. An id that doesn't belong to that phone number is rejected (`unknown_appointment`).

## Demo mode (no API keys)

**Appointments → New booking** and the **Cancel** buttons call the same service functions through `POST /api/appointments` and `POST /api/appointments/cancel`. The same validation, double-booking protection and live updates apply. These entries are tagged **Desk** in the dashboard, and voice bookings are tagged **Voice**. **Reset demo data** in the sidebar restores the seed data.

## Testing

`backend/tests` (45 tests, run in about 2 s, no network needed):

| Scenario | Test |
|---|---|
| Ask for cardiologists, give a date, get real slots | `test_full_booking_call_updates_dashboard`, `test_find_cardiologists_by_spoken_specialty` |
| Spoken number, read back, confirmation required | `test_full_booking_call_updates_dashboard`, `test_booking_refused_until_phone_confirmed` |
| Book, then the dashboard receives the live event | `test_full_booking_call_updates_dashboard` |
| Incorrect number corrected before confirmation | `test_wrong_number_then_corrected` |
| No available slots, next day suggested | `test_no_available_slots_suggests_next_day`, `test_no_slots_when_doctor_is_off_and_next_day_is_suggested` |
| Double booking prevented (voice and API) | `test_double_booking_is_prevented`, `test_double_booking_through_voice`, `test_demo_booking_and_cancel_roundtrip` |
| Cancel with confirmed number (two-step) | `test_cancel_with_confirmed_phone`, `test_cancel_is_two_step` |
| Unknown appointment / someone else's booking | `test_cancel_unknown_appointment`, `test_cannot_cancel_someone_elses_appointment` |
| Invalid input (phone, date, time, specialty, missing fields) | `test_invalid_input_is_reported_not_raised`, `test_invalid_requests`, `test_normalize_phone` |
| Voice refused cleanly without credentials | `test_offer_refused_without_credentials` |

The tool tests replay the function calls Gemini makes; they don't call Gemini or ElevenLabs.

### End-to-end voice call test

`python -m tests.e2e.call_e2e` places a complete call from the real dashboard in Chromium and checks 25 things along the way. It needs `pip install -r requirements-dev.txt`, `playwright install chromium`, espeak-ng, and `npm run build` in `frontend/`.

**Real in this test:**
* the React app and the Pipecat browser client
* the microphone and WebRTC audio in both directions
* FastAPI signalling, Silero VAD and Pipecat's turn detection
* Pipecat's ElevenLabs STT/TTS and Gemini client code, and Pipecat function calling
* our three tools, SQLite, and the SSE dashboard updates

**Emulated:** the ElevenLabs and Gemini *servers* (`tests/e2e/vendor_emulator.py`). They speak the same wire protocols. The caller's voice is espeak-ng speech played into a synthetic microphone. The STT emulator only returns a line after real speech audio for it has arrived over WebRTC. The "LLM" is a fixed script that makes the same tool calls a model would, using the real tool results. TTS audio is real speech from espeak-ng.

The scripted call:
1. Ask for a cardiologist on Friday. Mira checks availability and offers real slots.
2. Pick a slot and give a name.
3. Give a **wrong** number. Mira reads it back, the caller says no and corrects it, and Mira reads it back again.
4. Confirm. The booking is made, the confirmation card appears in the phone, and the appointments table updates live.
5. The database holds the booking under the confirmed number and nothing under the wrong one.
6. Cancel. Mira finds the booking by phone number, confirms which one, and cancels it.
7. Hang up. The dashboard shows the cancellation.

Artifacts (a video of the call, screenshots, and backend and emulator logs) go to `backend/tests/e2e/artifacts/`.

## Known limitations

* **Real ElevenLabs and Gemini responses have not been verified from the build environment,** which has no network route to those APIs. Everything between the browser and those APIs is covered by the end-to-end test against protocol emulators. Run `python -m scripts.check_voice_services`, then place one real call, to confirm your keys, voice, model and free-tier quota.
* Speech recognition of spoken phone numbers depends on ElevenLabs. The agent always reads the number back, and the tools refuse to book or cancel until the caller confirms it.
* Mock data: 7 doctors with fixed weekly hours and 30-minute slots, bookings open for 7 days, and seeded sample bookings (tagged *Sample*).
* Gemini model names on the free tier change over time. If you get a model error, set `GOOGLE_MODEL` to a Flash model listed in Google AI Studio.
* Single process with an in-memory event bus: fine for a demo, but several backend instances would need Redis or similar for events.
* No dashboard authentication (by design for this assignment). Phone numbers are masked in the UI.
