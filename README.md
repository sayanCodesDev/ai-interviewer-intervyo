# Intervyu

> AI mock interviews for software engineers — voice-driven, with a live code editor, system-design whiteboard, and a scored report with a personalised study plan.

<img src="docs/screenshots/landingPage.png" width="820" alt="Intervyu's landing page: 'Do the interview before the interview.'">

---

## Table of Contents

- [Quick start (local development)](#quick-start-local-development)
  - [Prerequisites](#prerequisites)
  - [1 · Clone](#1--clone)
  - [2 · Backend setup](#2--backend-setup)
  - [3 · Frontend setup](#3--frontend-setup)
  - [4 · Open the app](#4--open-the-app)
- [Try it without spending anything (mock mode)](#try-it-without-spending-anything-mock-mode)
- [Run with Docker Compose](#run-with-docker-compose)
- [Environment variables reference](#environment-variables-reference)
- [Running tests](#running-tests)
- [Project layout](#project-layout)
- [Product overview](#product-overview)
- [Architecture](#architecture)
- [Limits you should know about](#limits-you-should-know-about)
- [Security](#security)
- [Performance](#performance)
- [Deploying to production](#deploying-to-production)

---

## Quick start (local development)

### Prerequisites

| Tool | Minimum version | Notes |
|---|---|---|
| **Node.js** | 22 | `node -v` — needed for the backend |
| **npm** | bundled with Node 22 | used for backend install |
| **Bun** | latest | `bun -v` — needed for the frontend ([install](https://bun.sh)) |
| **Docker** | any recent version | required to run the code-execution sandbox |
| **PostgreSQL** | 16 | local install **or** a free [Neon](https://neon.tech) database |

> **Tip — Neon (free cloud Postgres):** Create a project, copy the *pooled* connection string, and paste it as `DATABASE_URL`. No local Postgres install needed.

---

### 1 · Clone

```bash
git clone https://github.com/<your-org>/ai-interviewer.git
cd ai-interviewer
```

---

### 2 · Backend setup

```bash
cd backend

# Copy the example env file
cp .env.example .env
```

Open `backend/.env` in your editor and set **at minimum**:

| Variable | Where to get it |
|---|---|
| `DATABASE_URL` | Your local Postgres connection string or Neon pooled URL |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) — free tier works for dev |
| `DEEPGRAM_API_KEY` | [console.deepgram.com](https://console.deepgram.com) — free $200 credit on sign-up |
| `JWT_SECRET` | Run `openssl rand -hex 32` and paste the output |

Build the **code-execution sandbox image** (only needed once):

```bash
docker build -t ai-interviewer-runner:latest runner
```

Install dependencies and start the dev server (migrations run automatically):

```bash
npm install       # also generates the Prisma client via postinstall
npm run dev       # runs migrations, then starts on http://localhost:2000
```

You should see:

```
Server listening on http://localhost:2000
```

---

### 3 · Frontend setup

Open a **second terminal** from the repo root:

```bash
cd frontend

cp .env.example .env
# VITE_BACKEND_URL defaults to http://localhost:2000 — no change needed for local dev

bun install
bun run dev       # starts on http://localhost:3000
```

---

### 4 · Open the app

Visit **http://localhost:3000** in your browser.

---

## Try it without spending anything (mock mode)

You can run a full interview using a local mock LLM and no speech — **no API keys required**:

```bash
# Terminal 1 — start the mock language model
cd backend
npx tsx scripts/mock-llm.ts

# Terminal 2 — start the backend pointing at the mock
cd backend
LLM_BASE_URL=http://127.0.0.1:2099/v1 \
LLM_API_KEY=mock \
VOICE_MODE=text \
npm run dev

# Terminal 3 — frontend (unchanged)
cd frontend
bun run dev
```

`VOICE_MODE=text` skips speech entirely — the interviewer communicates over a data channel and you type your answers.  
This mode is for development only; the server refuses to start with it in production.

---

## Run with Docker Compose

Docker Compose brings up PostgreSQL, the API, and the frontend in one command.

```bash
# 1. Fill in the backend env file
cp backend/.env.example backend/.env
# Edit backend/.env — set GROQ_API_KEY, DEEPGRAM_API_KEY, JWT_SECRET, etc.
# ALLOWED_ORIGINS must include http://localhost:8080

# 2. Build the code-sandbox image
docker build -t ai-interviewer-runner:latest backend/runner

# 3. Start everything
POSTGRES_PASSWORD=changeme docker compose up -d --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:8080 |
| API | http://localhost:2000 |

> **WebRTC and NAT:** if you're running on a cloud VM you'll need to set `WEBRTC_PUBLIC_IP` and open the UDP port range in your firewall. See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).

Stop everything:

```bash
docker compose down
```

---

## Environment variables reference

All variables are validated at boot. In production the server refuses to start if a required one is missing.

### `backend/.env`

```bash
# ── Core ──────────────────────────────────────────────────────────────────────
NODE_ENV=development          # development | production | test
PORT=2000
LOG_LEVEL=info                # fatal | error | warn | info | debug | trace | silent

# ── Database ──────────────────────────────────────────────────────────────────
DATABASE_URL=postgresql://user:password@host:5432/dbname
DATABASE_POOL_MAX=10

# ── Security ──────────────────────────────────────────────────────────────────
JWT_SECRET=                   # required: openssl rand -hex 32
ALLOWED_ORIGINS=http://localhost:3000   # comma-separated frontend origins

# ── Voice (Deepgram) ──────────────────────────────────────────────────────────
DEEPGRAM_API_KEY=             # required in production
VOICE_MODE=live               # live | text  (text = dev only, no speech)

# ── Language model ────────────────────────────────────────────────────────────
GROQ_API_KEY=                 # required (or set LLM_API_KEY with LLM_BASE_URL)
GROQ_MODEL=qwen/qwen3.8-27b
# Optional: point at any OpenAI-compatible provider instead of Groq
LLM_BASE_URL=
LLM_API_KEY=
# Optional: secondary provider, tried when the primary is exhausted for the day
SECONDARY_LLM_BASE_URL=
SECONDARY_LLM_API_KEY=
SECONDARY_LLM_MODEL=

# ── Code sandbox ──────────────────────────────────────────────────────────────
CODE_RUNNER=auto              # auto | docker | local  (local = dev only)
RUNNER_IMAGE=ai-interviewer-runner:latest

# ── WebRTC (cloud VMs only) ───────────────────────────────────────────────────
WEBRTC_PUBLIC_IP=             # server's public IPv4
WEBRTC_PORT_MIN=10000
WEBRTC_PORT_MAX=10039
```

See [`backend/.env.example`](backend/.env.example) for the complete annotated list.

### `frontend/.env`

```bash
VITE_BACKEND_URL=http://localhost:2000   # API origin (use https:// in production)
VITE_SITE_URL=https://your-domain.example
VITE_CONTACT_EMAIL=                      # optional: shown on the privacy page
```

---

## Running tests

### Backend

Tests require a **local** PostgreSQL instance. They refuse to run against any remote host, so they can never touch your real database. Default port: `5544`.

```bash
# Start a throwaway Postgres for tests
docker run -d \
  --name aii-test-db \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  -p 5544:5432 \
  postgres:16-alpine

# Run all backend tests (creates the test DB and applies migrations automatically)
cd backend && npm test
```

Run specific suites:

```bash
# Sandbox + language harnesses inside Docker (local runner can't compile Java)
CODE_RUNNER=docker npm test

# All 102 problems × 5 languages (slow — takes several minutes)
SLOW_TESTS=1 CODE_RUNNER=docker npx tsx --test src/interview/problems/starters.test.ts
```

### Frontend

```bash
cd frontend && bunx tsc --noEmit   # type-check only
```

### Development scripts

All scripts live in `backend/scripts/`. Run them with `npx tsx scripts/<name>.ts` from the `backend/` directory.

| Script | What it does |
|---|---|
| `mock-llm.ts` | Offline stand-in for the LLM — try everything for free |
| `simulate-interview.ts` | Full interviews through the conductor and scorer without spending LLM tokens |
| `smoke.ts` | Post-deploy sanity check: sign up → create interview → delete account |
| `security-probe.ts` | 57 attack checks against a running server (local addresses only) |
| `voice-check.ts` | Diagnose voice glitches — run this first if audio sounds wrong |
| `load-test.ts` | N simultaneous WebRTC interviews; reports latency/CPU/memory |
| `verify-problems.ts` | Every problem's reference solution against every language harness |

---

## Project layout

```
ai-interviewer/
├── backend/
│   ├── prisma/          schema and migrations
│   ├── src/
│   │   ├── auth/        tokens, sessions, passwords
│   │   ├── http/        Express app, middleware, rate limits, routes
│   │   ├── interview/   plans, conductor, prompts, problem bank, JD analysis
│   │   ├── webrtc/      live-call runtime, peer connection
│   │   ├── voice/       speech recognition, synthesis, turn-taking
│   │   ├── scoring/     rubric, evaluation, report worker
│   │   ├── runner/      Docker sandbox and unsafe local fallback
│   │   └── llm/         model client, router, JSON helper
│   ├── runner/          Dockerfile for the code-execution sandbox image
│   └── scripts/         simulator, mock model, load test, tooling
├── frontend/
│   ├── src/
│   │   ├── pages/       landing, setup, lobby, interview room, report, dashboard
│   │   ├── components/  interview room, editor, report, landing
│   │   └── seo.ts       per-route titles, canonicals, structured data
│   └── prerender.ts     turns public pages into static HTML at build time
├── docs/
│   ├── DEPLOYMENT.md    EC2 / PM2 / Netlify + Docker Compose + scaling guide
│   └── screenshots/
└── docker-compose.yml
```

---

## Product overview

### Interview formats

| Format | Length | Rounds |
|---|---|---|
| Quick screen | 20 min | Introduction · background · 1 coding problem · wrap-up |
| Standard | 45 min | Introduction · background · 2 coding problems · JD-specific technical · behavioural · wrap-up |
| Full loop | 75 min | Standard + a 3rd coding problem + system design on a whiteboard (juniors get concept questions) |
| Coding drill | 30 min | Four DSA problems back to back, complexity question after each |

A state machine — not a giant prompt — decides the round, the question, when to open the editor, when to hint, and when time is up. The model only voices each step.

<img src="docs/screenshots/setup.png" width="820" alt="Setup form: role, level, format, job description, resume, GitHub">

**Job description and GitHub are required.** One analysis pass turns them into weighted skills, role-specific questions, vocabulary for speech recognition, and coding-problem selection. The problem adapts: harder after a clean solve, easier after a struggle.

### An interviewer that actually follows along

- **Never runs ahead of you.** A turn ends when its last word has *played*, not when the model finished writing.
- **Can't switch questions early.** Moving on needs a real answer (25+ words).
- **Sees your code.** The editor opens once the problem is described aloud, closes on the next turn.
- **Behaves like a person.** "I don't know" gets one simpler question; "Okay" isn't scored as an answer.

<img src="docs/screenshots/interview.png" width="820" alt="Live interview room with captions">

### Editor and whiteboard

Monaco (VS Code's editor), self-hosted. JavaScript, TypeScript, Python, C++, Java. **Run** checks examples, **Submit** grades against hidden tests. 102 original problems — not scraped from LeetCode.

The system-design round uses an embedded **Excalidraw** whiteboard. The interviewer reads it as structured text (boxes, connections, labels), not a screenshot — no vision model, no extra latency.

**Every code run is a throwaway Docker container:** no network, capped memory/CPU/processes, read-only filesystem, unprivileged user.

### Report

<img src="docs/screenshots/report-score.png" width="820" alt="Report: score ring, readiness band, and summary">

- Score out of 100 with a readiness band — computed by the server, not the model
- Skill-by-skill breakdown: problem solving, code correctness (from real test results), code quality, complexity analysis, technical depth, communication
- Line-by-line code review on every submitted solution
- Prioritised study plan with **Practice this** follow-up interviews weighted toward what you got wrong
- Full transcript with timestamps; every quote is verified against it server-side

<img src="docs/screenshots/report-code-review.png" width="820" alt="Line-by-line code review">

<img src="docs/screenshots/dashboard.png" width="820" alt="Dashboard listing past interviews">

---

## Architecture

```mermaid
graph LR
    B[Browser: React, Monaco] -- HTTPS --> API
    B -- WebRTC audio + events --> API
    subgraph API [API server, stateless except live calls]
        H[Express: auth, interviews, reports]
        C[Conductor: rounds, questions, hints, time]
        V[Voice: WebRTC peer, turn-taking]
        W[Report worker]
    end
    H --> DB[(PostgreSQL)]
    C --> DB
    W --> DB
    V -- audio --> STT[Deepgram STT]
    V -- text --> TTS[Deepgram TTS]
    C -- prompts --> LLM[Language model: Groq or any OpenAI-compatible API]
    W -- transcript, results --> LLM
    H -- run code --> SB[Docker sandbox]
```

- **Everything durable is in PostgreSQL**: users, interviews, transcripts, submissions, reports. Only a live call's media connection lives in a server process.
- **Reports are a queue in the database.** An interview ending creates a job; any instance claims it with `FOR UPDATE SKIP LOCKED`, retries with backoff, and survives restarts.
- **The scorer never grades its own homework.** Correctness comes from hidden-test results; the model judges conversation and must quote evidence, which the server verifies.

---

## Limits you should know about

- **Free Groq keys aren't enough for real traffic.** 8,000 tokens/min + ~200,000/day per model. One 45-min interview uses ~60–70k tokens. Set `SECONDARY_LLM_*` for a second free-tier account as a fallback.
- **Docker is required** on the API machine for the code sandbox. Production refuses the unsandboxed local runner.
- **Behind NAT (any cloud VM), WebRTC needs setup**: `WEBRTC_PUBLIC_IP`, a UDP port range, and that range open in the firewall. See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).
- **English only.** Accent and voice are selectable.
- **Rate limits are per instance** by default. With multiple instances behind a load balancer, put a shared rate-limit store in front.
- **Live calls are pinned to one instance.** A reconnect within 90 seconds must reach the same instance; otherwise the interview closes and its report generates.

---

## Security

- **Auth:** 15-min access tokens in memory (never `localStorage`); rotating refresh tokens in an httpOnly cookie, stored hashed, with reuse detection that revokes the whole family; argon2id password hashing.
- **API:** every input validated (Zod), Helmet headers, strict CORS + origin check on cookie-authenticated writes, ownership checks on every resource (no IDOR).
- **Untrusted text** (JD, resume, GitHub, speech, code) is wrapped and sanitised before reaching a prompt.
- **Code sandbox:** fork bombs, memory bombs, network access, endless loops — all covered in the test suite.
- **Your data:** delete any interview or your account from the app; interviews auto-remove after `DATA_RETENTION_DAYS` (default 180). Audio is never stored.

---

## Performance

Measured with `backend/scripts/load-test.ts`, 10-core laptop, text mode with the mock model:

| Simultaneous interviews | Server CPU | Event-loop lag p99 | Memory | Completed |
|---|---|---|---|---|
| 10 | 13% of one core | 11 ms | ~190 MB | 10 / 10 |
| 30 | 28% | 11 ms | ~320 MB | 30 / 30 |
| 60 | 45% | 12 ms | ~375 MB | 60 / 60 |
| 100 | 51% | 11 ms | ~480 MB | 100 / 100 |

A real voice round-trip gets a spoken reply ~**3 seconds** after you stop speaking: ~2 s recognition/turn-taking, ~0.75 s synthesis. A real model adds its own first-token time.

---

## Deploying to production

See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md): EC2 + PM2 + Netlify, Docker Compose, WebRTC and cookie settings, and how to scale beyond one instance.
