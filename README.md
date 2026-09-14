#  True Tilawah — AI-Powered Quranic Recitation Tutor

True Tilawah is an AI-powered mobile app that listens to a user reciting the Quran in real time and gives instant voice feedback on mispronunciations, omitted words, extra words, and Tajweed violations. When a mistake is detected, the microphone pauses, the correct ayah is played back using Sheikh Alafasy's recitation, and the microphone resumes so the user can continue.

Built as a Final Year Project using a fine-tuned Whisper speech recognition model.

---

## ✨ Features

-  Real-time recitation capture via mobile microphone
-  Speech-to-text using a Whisper model fine-tuned on Quranic Arabic
-  Fuzzy alignment against the official Quran text (114 surahs, 6,236 ayahs)
-  Detects four mistake types: mispronunciation, omitted word, added word, and Tajweed violations (Qalqalah, Ghunna, Madd)
-  Automatic correction playback using Alafasy's recitation audio
-  Progress tracking, accuracy trends, and streaks
-  Memory retention practice mode

---

##  Architecture

True Tilawah is a monorepo with three independent services communicating over HTTP and WebSocket:

| Service | Technology | Responsibility |
|---|---|---|
| **Frontend** | React Native + Expo | Mobile UI, microphone capture, audio playback, feedback |
| **Backend** | Node.js + Express + Prisma | Auth, session management, database, WebSocket proxy |
| **AI Service** | Python + FastAPI | Speech recognition, Arabic alignment, Tajweed detection |
| **Database** | MySQL 8 + Prisma ORM | Users, sessions, feedback, Tajweed rules, Quran text |

```
 Phone (React Native)
      │  HTTP + WebSocket
      ▼
  Backend (Node.js / Express)
      │  WebSocket proxy
      ▼
 AI Service (Python / FastAPI)
      │
      ▼
  Fine-tuned Whisper Model
```

The phone never talks to the AI service directly — the backend acts as a secure proxy between them.

---

##  Tech Stack & Why

| Layer | Choice | Reason |
|---|---|---|
| Frontend | React Native + Expo | One codebase for Android & iOS; Expo AV handles mic + playback |
| Backend | Node.js + Express + Prisma | Efficient for I/O-heavy WebSocket proxying and DB queries |
| AI Service | Python + FastAPI | Best ML ecosystem (PyTorch, Transformers); native async support |
| Speech Model | Fine-tuned Whisper (whisper-small) | General Arabic Whisper mishears Quranic Arabic; fine-tuning on Alafasy's recitation fixes this |
| Alignment | RapidFuzz | Handles Unicode/diacritic mismatches between transcript and stored Quran text |
| Database | MySQL 8 | Relational structure maps cleanly to the data model |
| Deployment | Railway.app + Expo EAS | Docker-based hosting with auto-deploy on push; EAS builds the mobile APK |

---

## ✅ Prerequisites

### Hardware

| Component | Requirement |
|---|---|
| RAM | 8 GB minimum, 16 GB recommended |
| CPU | Any modern x64 CPU (NVIDIA GPU speeds up inference) |
| Disk | 5 GB free minimum |
| OS | Windows 10/11, Ubuntu 20+, or macOS 12+ |
| Network | Internet required on first run to download the Quran dataset (~50 MB) |

### Software

| Tool | Version |
|---|---|
| Node.js | ≥ 18.x |
| npm | ≥ 9.x |
| Python | 3.10 or 3.11 (3.12+ may have PyTorch wheel issues) |
| MySQL | ≥ 8.0 |
| Git | Any recent version |
| Expo Go / EAS | Expo Go app for development, EAS for production builds |
| ffmpeg | Required by the AI service for audio decoding — add to PATH |

---

##  Local Setup

You'll need **three terminals** running simultaneously, in this order: **Backend → AI Service → Frontend**.

> **Important:** Your phone and PC must be on the same WiFi network. Use your PC's IPv4 address (from `ipconfig`), not `localhost`, in the frontend `.env`.

### 1. Create the Database

```bash
mysql -u root -p
```

```sql
CREATE DATABASE true_tilawah CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
EXIT;
```

### 2. Backend Setup (Terminal 1)

```bash
cd backend
npm install
```

Create `backend/.env`:

```env
DATABASE_URL="mysql://root:YOUR_PASSWORD@localhost:3306/true_tilawah"
JWT_SECRET=replace_with_long_random_string_64_chars
JWT_REFRESH_SECRET=replace_with_another_long_random_string
NODE_ENV=development
PORT=5000
CORS_ORIGIN=*
AI_SERVICE_HOST=localhost
AI_SERVICE_PORT=8000
```

Push the schema, seed data, and start:

```bash
npx prisma db push
npm run seed:tajweed
npm run seed:quran   # takes 5–8 minutes, safe to re-run if interrupted
npm run dev
```

You should see:
```
✅ Database connected successfully (MySQL via Prisma)
 True Tilawah API is running
   REST → http://localhost:5000/api
   WS   → ws://localhost:5000/ws/audio
```

### 3. AI Service Setup (Terminal 2)

```bash
cd ai-service
pip install -r requirements.txt
pip install transformers accelerate safetensors pyarabic
```

Copy your fine-tuned model files (`model.safetensors`, `config.json`, tokenizer files) into:
```
ai-service/models/whisper-quran/
```

Create `ai-service/.env`:

```env
TRANSCRIPTION_PROVIDER=local_whisper
WHISPER_MODEL=C:\path\to\ai-service\models\whisper-quran
VAD_SILENCE_THRESHOLD_SEC=1.0
VAD_MIN_SPEECH_SEC=0.5
PORT=8000
TTS_WORD_TIMING_INDEX_PATH=
AI_SERVICE_AUTH_TOKEN=
```

Start the service:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

You should see:
```
[LocalWhisper] whisper-quran loaded successfully ✓
[Startup] OK Ready. 6236 verses indexed.
INFO: Uvicorn running on http://0.0.0.0:8000
```

### 4. Frontend Setup (Terminal 3)

Find your PC's LAN IP with `ipconfig` (e.g. `192.168.1.5`), then:

```bash
cd frontend
npm install
```

Create `frontend/.env`:

```env
EXPO_PUBLIC_API_BASE_URL=http://192.168.1.5:5000/api
EXPO_PUBLIC_WS_AUDIO_URL=ws://192.168.1.5:5000/ws/audio
```

Start Expo and scan the QR code with the Expo Go app (both devices on the same WiFi):

```bash
npx expo start --clear
```

### 5. Serve Correction Audio

Place Alafasy MP3 files (named `001.mp3` … `114.mp3`) in `backend/public/audio/`, then add this to `backend/src/app.js` after `const app = express();`:

```js
const path = require('path');
app.use('/audio', express.static(path.join(__dirname, '../public/audio')));
```

Test in your phone's browser: `http://YOUR_IP:5000/audio/001.mp3`

---

##  Environment Variables

### Backend (`backend/.env`)

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | MySQL connection string |
| `JWT_SECRET` | Signs access tokens (64+ chars recommended) |
| `JWT_REFRESH_SECRET` | Signs refresh tokens |
| `JWT_EXPIRES_IN` | Access token lifetime (default `7d`) |
| `JWT_REFRESH_EXPIRES_IN` | Refresh token lifetime (default `30d`) |
| `PORT` | Default `5000` |
| `AI_SERVICE_HOST` / `AI_SERVICE_PORT` | AI service address |
| `AI_SERVICE_WSS_URL` | Full `wss://` URL, overrides host/port (used in production) |
| `CORS_ORIGIN` | Default `*`; set to your domain in production |
| `BCRYPT_SALT_ROUNDS` | Default `12` |

### AI Service (`ai-service/.env`)

| Variable | Purpose |
|---|---|
| `TRANSCRIPTION_PROVIDER` | `local_whisper` or `groq` |
| `WHISPER_MODEL` | Path to fine-tuned model or HF model name |
| `GROQ_API_KEY` / `GROQ_MODEL` | Required only if using `groq` |
| `VAD_SILENCE_THRESHOLD_SEC` | Default `1.0` |
| `VAD_MIN_SPEECH_SEC` | Default `0.5` |
| `PORT` | Default `8000` |
| `TTS_WORD_TIMING_INDEX_PATH` | Optional word-timing JSON path |
| `AI_SERVICE_AUTH_TOKEN` | Shared secret; leave empty for local dev |

### Frontend (`frontend/.env`)

| Variable | Purpose |
|---|---|
| `EXPO_PUBLIC_API_BASE_URL` | REST endpoint (use LAN IP for device testing) |
| `EXPO_PUBLIC_WS_AUDIO_URL` | WebSocket endpoint (`ws://` for local, `wss://` in production) |

---

##  Database Management

| Command | Description |
|---|---|
| `npx prisma db push` | Push schema without migration history (fast initial setup) |
| `npx prisma migrate dev` | Create and apply a versioned migration |
| `npx prisma generate` | Regenerate the Prisma client |
| `npx prisma studio` | Visual DB GUI at `localhost:5555` |
| `npm run seed:tajweed` | Seeds 3 Tajweed rules (Qalqalah, Madd, Ghunna) |
| `npm run seed:quran` | Seeds 114 surahs / 6,236 ayahs from Quran.com API (idempotent, safe to re-run) |

---

##  API Reference

### Auth — `/api/auth`
| Endpoint | Description |
|---|---|
| `POST /register` | `{ fullName, email, password }` |
| `POST /login` | `{ email, password }` → access + refresh tokens |
| `POST /refresh` | `{ refreshToken }` |
| `GET /profile` | Requires Bearer token |

### Sessions — `/api/sessions` (auth required)
| Endpoint | Description |
|---|---|
| `POST /` | Start session — `{ surahId, ayahStart, ayahEnd }` |
| `GET /` | List sessions (paginated) |
| `GET /:id` | Session detail with full feedback |
| `PATCH /:id/complete` | Mark complete with transcript + score |
| `PATCH /:id/abandon` | Mark abandoned |
| `DELETE /:id` | Delete session |

### Progress — `/api/progress` (auth required)
| Endpoint | Description |
|---|---|
| `GET /` | Overall stats: accuracy, sessions, time, streak |
| `GET /trend` | Accuracy trend for charting |
| `GET /errors` | Mistake breakdown by type |
| `GET /tajweed` | Most-violated Tajweed rules |

### Quran Reference — `/api/quran` (public)
| Endpoint | Description |
|---|---|
| `GET /surahs` | List all 114 surahs |
| `GET /surahs/:number` | Surah metadata |
| `GET /surahs/:number/ayahs` | All ayahs in a surah |
| `GET /surahs/:number/range?start=1&end=7` | Ayah range |
| `GET /health` | Health check |

---

## 🔌 WebSocket Audio Protocol

**Connect:** `ws://localhost:5000/ws/audio?token=<JWT>&sessionId=<uuid>`

*(WebSocket connections can't send Authorization headers, so the JWT is passed as a query parameter.)*

**Flow:**
1. Frontend opens the WebSocket with `token` and `sessionId`.
2. Backend validates the JWT and confirms the session exists.
3. Backend opens a child connection to the Python AI service and sends a config frame.
4. Python replies `{ type: "ready" }`.
5. Frontend streams `{ type: "audio", seq: N, pcm: "<base64 int16>" }` frames.
6. Python emits real-time mistake events.
7. Frontend sends `"STOP"` to finalize; Python sends a `final_report`.

**Event types:** `ready`, `mistake`, `unclear`, `out_of_scope`, `final_report`, `error`

**Close codes:** `4001` auth failure · `4003` session not found/not owned · `4503` AI service unavailable

---

## 📱 App Screens

| Screen | Purpose |
|---|---|
| `ReciteScreen` | Main recitation screen — live mic, mistake display, audio feedback |
| `RetainScreen` | Memory retention exercise using the same feedback pipeline |
| `TrackScreen` | Session history and progress charts |
| `DashboardScreen` | Home screen — progress summary, streaks, quick actions |
| `ProfileScreen` | Account settings and logout |
| `SessionDetailScreen` | Detailed mistake breakdown for a past session |
| `RetainResultsScreen` | Results after a retention exercise |

---

## ☁️ Deployment (Railway + Expo EAS)

1. **Push to GitHub** and create a new Railway project.
2. **Deploy MySQL**: *New Service → Database → MySQL*, then copy `DATABASE_URL`.
3. **Deploy Backend**: add a `Dockerfile` (Node 18 Alpine), deploy from GitHub, set the backend env vars, then run `npm run seed:tajweed` and `npm run seed:quran` from the Railway terminal.
4. **Deploy AI Service**: use `TRANSCRIPTION_PROVIDER=groq` in production (avoids uploading large model files); add a Python 3.10-slim Dockerfile with `ffmpeg` installed.
5. **Build the mobile app**:
   ```bash
   npm install -g eas-cli
   eas login
   cd frontend
   eas build --platform android --profile preview
   ```
   Update `frontend/.env` with your Railway `https://` / `wss://` URLs first.

---

##  Troubleshooting

| Problem | Solution |
|---|---|
| Site unreachable on phone | `netsh advfirewall firewall add rule name="Backend 5000" dir=in action=allow protocol=TCP localport=5000` |
| Nothing on `netstat :5000` | Backend isn't running — run `npm run dev` in `backend/` |
| Database connection failed | Check MySQL service is running; verify `DATABASE_URL` password |
| Mic hardware/permission error | Search `ReciteScreen.js` for `StreamErrorCode`, replace with string checks |
| VAD Double/Float error | Known TorchScript issue — VAD is bypassed by design |
| Model segfault on GPU | CUDA/PyTorch mismatch — set `self._device = "cpu"` in `local_whisper.py` |
| No mistakes detected | Check AI service logs for `[WS] Transcribed:` lines; verify `AI_SERVICE_HOST`/`PORT` |
| Quran endpoints return empty | Seed not run — run `npm run seed:quran` in `backend/` |
| Session 500 error on complete | Ensure `completeSession` doesn't require `status=ACTIVE` |
| Audio file not playing | Confirm `backend/public/audio/001.mp3` etc. exist; test the URL in a phone browser |
| `cd` path errors in Git Bash | Use forward slashes (`cd /c/Users/name/project`) or use Command Prompt instead |
| `pip install` fails on Python 3.12+ | Use Python 3.10/3.11 (`py -3.11` on Windows) |
| `uvicorn: not found` | `pip install uvicorn` or run `python -m uvicorn app.main:app --host 0.0.0.0 --port 8000` |

---

##  Notable Engineering Decisions

- **Audio pipeline**: WAV chunks from Expo AV are base64-encoded, converted from int16 to float32 on the backend, then forwarded to the AI service as binary WebSocket frames.
- **VAD bypass**: Silero VAD's TorchScript internals caused a float32/float64 type conflict with no clean fix, so voice activity detection was dropped in favor of transcribing every 2-second buffer directly — Whisper handles silence well enough that accuracy impact is negligible.
- **Connection timeouts**: CPU inference can take 5–15 seconds, so the backend's AI service timeout was raised from 10s to 30s, with WebSocket sends wrapped in try/except to survive disconnects.
- **Idempotent seeding**: `seed:quran` uses upserts, so it's always safe to re-run if it's interrupted partway through.


