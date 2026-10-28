# Speak-Bridge

A two-way communication app: recognize ASL hand shapes from a webcam and
turn them into text, and turn spoken voice into text — both sides feeding
a shared conversation transcript that can be read aloud with ElevenLabs
TTS. **Either side can also just type**, so a limitation in camera or
microphone recognition never actually blocks the conversation — see
"Reliability: the typed fallback" below.

---

## Recognition architecture (hybrid, as actually implemented)

```
Camera (browser)
  → MediaPipe Hands (landmarks, runs locally in-browser)
  → gestureClassifier.js (rule-based static-pose classifier, local, instant, free)
        │
        ├─ recognized → normalized { gloss, confidence } → transcript
        │
        └─ NOT recognized, held ≥ 700ms
              → POST /api/stream/frame-sse (only if /api/config says NeuralSeek is configured)
              → NeuralSeek maistro agent → normalized { gloss, confidence } → transcript
```

Local recognition runs first because it's zero network latency, free, and
works offline — it covers a fixed, small vocabulary (below). When it can't
name a held pose, and the backend confirms NeuralSeek is actually
configured (checked once via `GET /api/config`, not guessed or assumed),
the frontend uploads that one frame to NeuralSeek as a fallback. Both
paths funnel into the same `emitRecognition(gloss, confidence, source)`
function in `Camera.jsx`, so the transcript treats a NeuralSeek result
identically to a local one — it's tagged with a small ☁️ so you can tell
which path produced it, nothing more.

If `NEURALSEEK_API_KEY` / `NEURALSEEK_STREAM_URL` / an agent ID aren't
set, `/api/config` reports `neuralseek_enabled: false` and the frontend
never attempts the cloud path — no wasted requests, no confusing errors.

### Supported sign vocabulary (local classifier)

**Static, single-hand:** `A, B, C, D, F, I, K, L, O, R, S, U, V, W, X, Y, HELLO, I LOVE YOU, THREE, GOOD (👍), BAD (👎)`

**Static, two-hand (FR-016):** `MORE, SAME` — recognized when exactly two
hands are held still; each is checked against `classifyTwoHandShape` in
`gestureClassifier.js` before falling back to per-hand static recognition.

**Dynamic/motion signs — user-taught, zero out of the box (FR-010):** the
built-in classifier above only recognizes *held* poses. Motion signs (`J`,
`Z`, or anything else that requires tracing a path) are handled by a
separate, real Dynamic-Time-Warping matcher in `src/dynamicSignTrainer.js` —
see "Sign Trainer" below. This ships with **no pretrained motion signs**;
it only knows what you personally teach it in your own browser.

`F`/`O`/`D` are detected via an actual thumb-to-fingertip pinch distance
check (not just finger-extended/curled booleans — a pinched finger doesn't
reliably read as "curled"). `C` is detected as the geometric middle ground
between "extended" and "tightly curled." `GOOD`/`BAD` are distinguished by
comparing the thumb tip's position to the wrist. Letters that need wrist
rotation or an occluded thumb (`G, H, M, N, P, Q, ...`) are deliberately
**not** guessed at — see the comment block at the top of
`gestureClassifier.js` for the reasoning (a wrong guess is worse than
"unrecognized" for an accessibility tool).

This is **not full ASL** — no facial grammar, and the letters above are the
geometrically verified subset, not the full alphabet. The NeuralSeek
fallback can extend real coverage beyond this list *if* you configure a
NeuralSeek agent that actually understands ASL frames — this app sends it
a raw PNG frame and expects back JSON shaped like
`{"gloss": "...", "confidence": 0.0-1.0}` (or plain text, which is used as
the gloss directly); what the agent itself recognizes is entirely up to
how you build/train it. Without NeuralSeek configured, the vocabulary
above (plus whatever you've personally taught the sign trainer) is the
ceiling.

### Explicit "unrecognized" state (FR-012)

When a pose is held for ~900ms and neither the local classifier nor
NeuralSeek (if configured) can name it, the UI now surfaces a distinct
"🤔 Sign not recognized" state — different from "no hands detected" or
mid-recognition — rather than just going quiet. `Camera.jsx` exposes this
via an `onUnrecognized(boolean)` callback.

### Sign Trainer: teaching motion signs (FR-010)

Click **🌀 Sign Trainer** in the header. Type a label, hit **Record**, then
perform the sign 3–5 times in front of the camera (pause between each rep —
each hand-enter→exit cycle is captured as one training example). Recognition
starts working immediately afterward, using DTW (Dynamic Time Warping)
distance between your live performance and your saved examples — see the
comment block at the top of `dynamicSignTrainer.js` for how the matching
actually works and its stated limits.

**This is genuinely local, not a demo:** templates live in this browser's
`localStorage` only. They are **not** synced across devices or shared
between the two people in a conversation — use **Export**/**Import** in the
Sign Trainer panel to move a taught vocabulary between browsers manually.
There is no server-side training, no dataset, and no model file.

### Sign avatar: text/voice → sign (FR-014)

Any voice or typed message in the transcript has a 🤟 button that renders it
through a schematic hand-pose avatar (`SignAvatar.jsx`) — whole-word signs
where the vocabulary has one (e.g. "hello", "i love you"), fingerspelled
letter-by-letter otherwise, and **plain text for anything outside the
verified vocabulary** (never a fabricated pose — see `signAvatarPoses.js`).
It's a simple stepped SVG viewer (◀ ▶ ⏸ controls), not an animated 3D
avatar — an honest "schematic" as the name says.

### Flagging a misrecognition (FR-013)

Every non-typed sign-side message has a ⚑ button. Clicking it POSTs to
`/api/feedback`, which appends a line to `server/feedback/flagged_recognitions.jsonl`
(created on first use, gitignored). There's no dashboard for this yet — it's
a durable log for manual review, not a full feedback pipeline; see G5 in the
project docs for where this is headed.

### Reliability: the typed fallback

Camera and voice recognition both have real, unavoidable limits (small
vocabulary; browser-dependent Web Speech API; camera/mic permissions).
Rather than let a recognition failure block communication, the transcript
panel always has a plain text box — either person can type what they meant
to say at any time, in either mode. Typed messages are marked "⌨️ typed"
in the transcript so it's clear they weren't auto-recognized, but they
work identically for TTS playback. This is the one path in this app that
doesn't depend on a camera, a microphone, a specific browser, or any API
key — treat it as the reliable baseline, and the recognition features as
a convenience on top of it.

---

## ✨ What's inside

- **Frontend:** React 19 + Vite, Tailwind v4, MediaPipe Hands for
  in-browser landmark detection, a rule-based classifier for static signs,
  a NeuralSeek fallback for unrecognized poses, the Web Speech API for
  voice input, and a manual typed-message fallback for both.
- **Backend:** FastAPI with CORS, static file hosting, rate limiting,
  upload validation, a `/api/config` capability check, a NeuralSeek
  streaming endpoint with normalized `{gloss, confidence}` output, and
  `/api/speak` using ElevenLabs TTS.

---

## 🗂 Project layout (key files)

```
speak-bridge/
├── index.html
├── vite.config.js               # includes vitest config
├── package.json
├── .env.example              # frontend env template
├── .github/workflows/ci.yml  # lint + unit tests + build, on every push/PR
├── public/mediapipe/hands/   # self-hosted MediaPipe assets (see below)
├── src/
│   ├── main.jsx
│   ├── App.jsx                # UI shell, transcript, TTS playback, trainer/avatar wiring
│   ├── Camera.jsx             # webcam capture + local/two-hand/dynamic sign recognition
│   ├── gestureClassifier.js   # rule-based static + two-hand ASL pose classifier
│   ├── gestureClassifier.test.js
│   ├── dynamicSignTrainer.js  # DTW motion-sign matcher + localStorage template store
│   ├── dynamicSignTrainer.test.js
│   ├── SignTrainer.jsx        # UI panel for teaching motion signs
│   ├── SignAvatar.jsx         # schematic text/voice → sign avatar
│   ├── avatarTokenizer.js     # text → avatar-step tokenizer
│   ├── signAvatarPoses.js     # canonical hand poses, scoped to verified vocabulary
│   ├── SignAvatar.test.js
│   └── VoiceInput.jsx         # Web Speech API voice input
└── server/
    ├── main.py                 # FastAPI app (CORS, static, SSE, TTS, STT, /api/config, /api/feedback)
    ├── test_main.py            # pytest suite for main.py
    ├── pytest.ini
    ├── requirements.txt
    ├── requirements-dev.txt    # + pytest/httpx, for `pytest`
    ├── neuralseek_client.py    # NeuralSeek streaming, wired in as a fallback (see below)
    ├── eleven_labs_client.py   # ElevenLabs speech generation (TTS) and transcription (STT)
    ├── .env.example            # backend env template
    ├── feedback/                # gitignored — created at runtime by /api/feedback
    └── static/
```

---

## Requirements & known limitations (read before deploying)

- **Voice input now has three tiers, not one.** Chrome/Edge use the
  browser's built-in Web Speech API directly (free, instant, no backend
  call). Firefox/Safari — which don't implement Web Speech — now get a
  real fallback: `VoiceInput.jsx` records audio with `MediaRecorder`
  (broadly supported, including Firefox/Safari) and posts it to
  `/api/transcribe`, which transcribes it via ElevenLabs' Scribe STT. This
  only activates if `/api/config` confirms `ELEVENLABS_API_KEY` is set —
  otherwise, or if transcription fails, there's still the typed-input box
  as a last resort. So "browser-dependent" no longer means "broken outside
  Chrome," it means "a different (slightly slower, record-then-transcribe)
  path outside Chrome."
- **Camera access requires a secure context.** `getUserMedia` (used for the
  webcam) is only available on `https://` origins or `localhost`. Deploying
  the frontend over plain `http://` will make the camera silently
  unavailable — Camera.jsx now detects this and shows an explicit message
  instead of failing silently.
- **ElevenLabs `ELEVENLABS_API_KEY` powers both TTS and STT.** Without it,
  `/api/speak` returns a clear error (not a stack trace) and the 🔈 button
  won't produce audio, and `/api/transcribe` (the Firefox/Safari voice
  fallback above) won't work either — `/api/config` reports both as
  disabled so the frontend doesn't offer features that would just fail.
- **NeuralSeek is an optional fallback for poses the local classifier doesn't recognize.** Set `NEURALSEEK_API_KEY`, an agent ID/name, and `NEURALSEEK_STREAM_URL` to enable it — there's no hard-coded staging tenant fallback, so if any of these are missing the fallback path just stays off (`/api/config` reports `neuralseek_enabled: false`) rather than silently hitting a wrong endpoint.
- **The typed-message box always works** regardless of any of the above — no camera, mic, browser support, or API key required. If you only deploy one thing correctly, make it `ALLOWED_ORIGINS`/`VITE_API_BASE` so typed messages and TTS still work even if recognition doesn't.
- **MediaPipe assets are now self-hosted** under `public/mediapipe/hands/`
  (copied from the `@mediapipe/hands` npm package) instead of being loaded
  from `cdn.jsdelivr.net`. This removes the CDN as a runtime dependency, at
  the cost of a larger repo (~24MB of model/wasm assets). If you'd rather
  use the CDN, see the comment in `Camera.jsx`'s `locateFile`.

---

## 🔑 Environment variables

**Backend** — copy `server/.env.example` to `server/.env` and fill in:

```ini
ALLOWED_ORIGINS=http://localhost:5173   # your real frontend origin(s) in prod
NEURALSEEK_API_KEY=                     # only needed if you wire in NeuralSeek
NEURALSEEK_AGENT_ID=
# NEURALSEEK_STREAM_URL=https://api.neuralseek.com/v1/<tenant>/maistro_stream
ELEVENLABS_API_KEY=                     # required for the 🔈 speak button
# ELEVENLABS_VOICE_ID=                  # optional, defaults to JBFqnCBsd6RMkjVDRZzb
```

If `ALLOWED_ORIGINS` is left unset, the backend falls back to a public,
**non-credentialed** wildcard CORS policy (fine for local demos, not for
production — set it explicitly before deploying).

**Frontend** — copy `.env.example` to `.env`:

```ini
VITE_API_BASE=                          # optional; only for split-origin hosting
```

The frontend now uses **same-origin API calls by default**. During local
development Vite proxies `/api`, `/health`, and `/static` to FastAPI on port
8000. For the recommended production setup, run `npm run build` and deploy
the backend with the generated `dist/` folder beside `server/`; FastAPI
serves the React app and API from one origin. Set `VITE_API_BASE` only when
you intentionally host frontend and backend on different origins.

---

## Running locally

### Backend (FastAPI)

```bash
cd speak-bridge/server
python3 -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # then fill in your keys
python main.py
# → http://localhost:8000
```

### Frontend (Vite + React)

```bash
cd speak-bridge
npm install                     # do NOT reuse a node_modules folder copied
                                 # from a different OS/architecture — native
                                 # deps (e.g. Rollup's platform binary) won't
                                 # match and the build will fail. Always run
                                 # a fresh `npm install` on the target machine.
cp .env.example .env            # optional for local dev
npm run dev
# → http://localhost:5173
```

### Endpoints

| Method | Route | Purpose |
|---|---|---|
| GET | `/health` | health check — includes a `features` block (`tts`, `stt`, `neuralseek_fallback`) so a host's health check can catch a misconfigured-but-running deployment |
| GET | `/api/config` | tells the frontend which optional features (NeuralSeek fallback, TTS, STT) are actually configured |
| POST | `/api/stream/frame` | uploads a frame, updates `/static/latest.png` (rate-limited, size/type-validated) — not called by the current UI, kept for debugging/future use |
| POST | `/api/stream/frame-sse` | streams a held, locally-unrecognized frame to NeuralSeek; called automatically by `Camera.jsx` when `/api/config` says it's enabled (rate-limited) |
| POST | `/api/speak` | text → speech via ElevenLabs (rate-limited to 10 req/min/IP, generic error messages) |
| POST | `/api/transcribe` | audio → text via ElevenLabs Scribe STT; used by `VoiceInput.jsx` on Firefox/Safari (rate-limited, size/type-validated) |
| POST | `/api/feedback` | records a flagged misrecognition to `server/feedback/flagged_recognitions.jsonl` (rate-limited) — FR-013 |

---

## 🧪 Testing

**Frontend** (Vitest — 28 tests covering the classifier, DTW matcher, and avatar tokenizer):

```bash
npm run test        # one-shot
npx vitest           # watch mode
npm run lint          # ESLint
```

**Backend** (pytest — 21 tests covering validation, rate limiting, `/api/feedback`, `/health`, `/api/config`, and the credential sanity check):

```bash
cd server
pip install -r requirements-dev.txt
pytest
```

CI (`.github/workflows/ci.yml`) runs both suites, plus `eslint` and
`vite build`, on every push/PR to `main`.

---

## Deploying

1. **Deploy the backend** (FastAPI) somewhere that can run a long-lived
   Python process (Render, Fly.io, Railway, a VM, etc). Set
   `ALLOWED_ORIGINS`, `ELEVENLABS_API_KEY`, and (if used) the NeuralSeek
   vars as real environment variables on that host — not in a committed
   `.env` file.
2. **Build the frontend** into the backend's `dist/` folder so FastAPI can
   serve both from one origin (the default, safest path — see the SPA
   fallback route in `server/main.py`):
   ```bash
   npm run build   # writes to speak-bridge/dist
   ```
   You don't need to set `VITE_API_BASE` for this path — it defaults to
   same-origin. Only set it if you're deploying frontend and backend to
   **different** origins (e.g. Vercel frontend + Render backend):
   ```bash
   VITE_API_BASE=https://your-backend.example.com npm run build
   ```
3. Serve over **HTTPS** (required for camera access — see limitations
   above).
4. If frontend and backend are on different origins, double-check
   `ALLOWED_ORIGINS` on the backend includes the exact frontend origin
   (scheme + host + port, no trailing slash).

### Quick local tunnel (ngrok)

```bash
ngrok http 8000
# copy the https URL, then:
VITE_API_BASE=https://<random>.ngrok-free.app npm run dev
```

---

## 🔌 NeuralSeek fallback: what it expects from your agent

`neuralseek_client.py` posts a base64 PNG data URL to a `maistro_stream`
agent:

```json
{
  "agent_id": "…",
  "params": [
    { "name": "frame_png_b64", "value": "data:image/png;base64,..." },
    { "name": "prior_hypothesis", "value": "" }
  ]
}
```

with `Authorization: Bearer <NEURALSEEK_API_KEY>` and
`Accept: text/event-stream`. Whatever your agent streams back, the backend
tries to parse each token as JSON and pull out `gloss`/`label`/`text`/
`answer` and `confidence`/`score` fields; if a token isn't JSON, the raw
text is used as the gloss directly. Either way, the backend always emits
a normalized `{"gloss": ..., "confidence": ..., "error": ...}` event to
the frontend — that's the one contract `Camera.jsx` actually parses
(`parseNeuralSeekEvent`), so what your agent internally does to recognize
ASL is entirely up to you as long as it returns something in this shape
(or plain gloss text).

## ElevenLabs TTS

- `/api/speak` uses `ELEVENLABS_VOICE_ID` (default `JBFqnCBsd6RMkjVDRZzb`)
  and model `eleven_multilingual_v2`, returning MP3 (44.1kHz/128kbps).
- Errors (missing key, ElevenLabs API failure, etc.) are logged server-side
  and returned to the client as a generic message — the client no longer
  sees raw exception text.
- Requests are rate-limited per IP (10/min) to protect your ElevenLabs
  quota from being exhausted by repeated requests against a public
  deployment. This is an in-memory, single-process limiter — good enough

## ElevenLabs STT (Firefox/Safari voice fallback)

- `/api/transcribe` calls ElevenLabs' `POST /v1/speech-to-text` REST
  endpoint directly (model `scribe_v1`), not through the `elevenlabs`
  Python SDK — the SDK version pinned here (1.8.0) predates that client
  method, so this talks to the documented REST contract instead.
- **This specific call has not been exercised against a live ElevenLabs
  account** — I don't have a real `ELEVENLABS_API_KEY` or network access to
  `api.elevenlabs.io` in the environment I built this in. The request
  shape matches ElevenLabs' published API docs and the error handling
  follows the same pattern as the (tested) TTS path, but you should
  actually record a clip and confirm you get real text back before relying
  on it. If the response shape has changed since this was written, the
  fix is likely just adjusting the `body.get("text", "")` line in
  `eleven_labs_client.transcribe_audio`.
- Rate-limited per IP (15/min) for the same quota-protection reason as
  `/api/speak`.
  for a small deployment, not a substitute for a real API gateway if you
  scale this up.

---

## Key files explained

| File | Description |
|---|---|
| `src/Camera.jsx` | Captures webcam, runs MediaPipe Hands (2-hand) locally, static/two-hand/dynamic recognition |
| `src/gestureClassifier.js` | Rule-based static + two-hand ASL pose classifier |
| `src/dynamicSignTrainer.js` | DTW motion-sign matcher + localStorage template store |
| `src/SignTrainer.jsx` | UI for teaching motion signs |
| `src/SignAvatar.jsx` / `signAvatarPoses.js` / `avatarTokenizer.js` | Schematic text/voice → sign avatar |
| `src/VoiceInput.jsx` | Web Speech API voice-to-text |
| `src/App.jsx` | UI shell: transcript, mode switch, TTS playback, trainer/avatar/feedback wiring |
| `server/main.py` | FastAPI app: CORS, static files, rate limiting, upload validation, TTS, feedback log |
| `server/neuralseek_client.py` | NeuralSeek streaming logic (available, not wired into the UI) |
| `server/eleven_labs_client.py` | ElevenLabs speech generation |

---

## Troubleshooting

- **Camera shows "Camera requires HTTPS":** deploy behind HTTPS, or test on `localhost`.
- **Camera shows "Camera access denied":** check browser site permissions.
- **Voice mode shows a Chrome message:** the Web Speech API isn't supported in your browser — use Chrome or Edge.
- **`/api/speak` returns 502:** check the backend logs — the real ElevenLabs error is logged server-side even though the client only sees a generic message.
- **`/api/speak` returns 429:** you've hit the per-IP rate limit (10/min); wait a bit.
- **CORS errors:** make sure `ALLOWED_ORIGINS` on the backend exactly matches your frontend's origin.
- **`npm run build` fails with a Rollup/native-binding error:** you're reusing a `node_modules` folder built on a different OS/CPU architecture. Delete it and run `npm install` fresh on the machine that will actually build/run the app.
- **`latest.png` not updating:** that endpoint (`/api/stream/frame`) isn't called by the current UI at all — it's only relevant if you build a feature that posts frames to it.
- **Sign not recognized and nothing happens after ~1s:** if NeuralSeek isn't configured, that's expected — the local classifier only knows its fixed vocabulary. Type the message instead, or configure NeuralSeek. Check the browser console for `NeuralSeek fallback:` warnings if it is configured but still not producing results.
- **Cloud (☁️) results seem wrong or slow:** that's coming from your NeuralSeek agent, not this app's code — the agent's own accuracy and latency are outside what this repo controls. Check `prior_hypothesis` usage and your agent's prompt/logic.

---

## License

MIT (adjust if desired).

## Credits

MediaPipe Hands © Google · Vite & React · NeuralSeek (mAIstro Stream) · ElevenLabs TTS


---

## Compatibility fallback added

Voice recognition still depends on the browser Web Speech API. On browsers
without it, Voice mode now provides a manual text-entry fallback instead of
becoming unusable. Chrome/Edge remain the recommended browsers for live mic
transcription.

## Recommended one-origin production build

```bash
cd speak-bridge
npm install
npm run build
cd server
pip install -r requirements.txt
python main.py
```

Deploy the whole `speak-bridge` directory so `dist/` remains next to
`server/`. FastAPI will serve the built React frontend and the API from the
same public HTTPS origin. This removes the production `localhost:8000`
failure mode and avoids split-origin CORS configuration for the normal
deployment path.

The local static sign vocabulary remains intentionally limited. That is not
claimed as full ASL. Expanding it reliably requires a trained sign-language
model or a fully specified NeuralSeek recognition response contract; adding
more geometric guesses would make the demo less trustworthy, not more.
