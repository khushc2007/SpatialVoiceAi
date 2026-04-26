# SpatialVoiceAI — Project Context

> Paste this entire file at the start of every Claude session. Last updated: [DATE] by [YOUR NAME]

---

## 🧠 What We Are Building

SpatialVoiceAI is a real-time AI system for the Samsung ennovateX AX Hackathon 2026 (Problem Statement 08).

It intercepts a voice call's audio stream, renders each speaker in a distinct 3D binaural position using HRTF physics, and runs a parallel NLP pipeline that builds a semantic conversation graph — powering a shared live companion interface with event flags, action-item extraction, live summaries, and graph-grounded Q&A.

**Stack:** Python 3.11 · FastAPI · PyAudio · Whisper.cpp · Phi-3-mini · networkx · FAISS · Next.js 15 · Three.js · Zustand **Constraint:** 100% open-source. No cloud APIs. No OpenAI. Runs on 8GB RAM laptop CPU. **Deadline:** Phase 1 Blueprint due May 13. Full demo: 18-day build window.

---

## 👥 Team

|Name|Role|Contact|
|---|---|---|
|Khush|Frontend · Voice Integration · Lead|[your contact]|
|[Teammate]|[their role]|[their contact]|

---

## 📍 Current Status

**Active Day:** Day [ ] of 18 **Active Phase:** Phase [ ] — [Phase Name] **Last Session Date:** [DATE] **Last Session By:** [NAME]

### What is working right now

- [ ] Nothing yet / [ ] List what actually runs

### What is blocked right now

- [ ] Nothing blocked / [ ] Describe the blocker clearly

### Decisions made that deviate from the roadmap PDF

- None yet. (Add here if you swap a library, skip a feature, change a design decision.)

---

## ✅ Phase Progress Summary

| Phase | Name                     | Status         |
| ----- | ------------------------ | -------------- |
| 0     | Setup & Planning         | 🔄 In Progress |
| 1     | Backend Foundation       | ⬜ Not Started  |
| 2     | AI Core Integration      | ⬜ Not Started  |
| 3     | Feature Development      | ⬜ Not Started  |
| 4     | Frontend Development     | ⬜ Not Started  |
| 5     | Integration              | ⬜ Not Started  |
| 6     | Polish & Differentiation | ⬜ Not Started  |
| 7     | Demo Preparation         | ⬜ Not Started  |

> Update status: ⬜ Not Started · 🔄 In Progress · ✅ Done · 🚫 Blocked

---

## 🗂️ Repository Structure

```
spatialvoiceai/
├── backend/
│   ├── main.py              # FastAPI entry point
│   ├── audio/               # HRTF renderer, VAD, AudioEngine
│   ├── nlp/                 # Whisper runner, Phi-3 runner
│   ├── graph/               # Conversation graph builder
│   ├── api/                 # Route files: /session /graph /qa /ws
│   ├── models/              # Downloaded model files (gitignored)
│   ├── hrtf/                # SADIE II dataset files
│   ├── db/                  # SQLite + session utils
│   └── config.py            # All constants
├── frontend/                # Next.js 15 app
│   ├── app/                 # App router pages
│   ├── components/          # SpatialCanvas, TranscriptRibbon, QADrawer
│   ├── hooks/               # useWebSocket, useAudioLevel
│   └── lib/                 # API client functions
├── audio_samples/           # Pre-recorded test WAV files
├── docs/                    # Architecture diagram, README
├── .env                     # Model paths, ports (gitignored)
├── requirements.txt
└── package.json
```

---

## 🔑 Key Technical Decisions (permanent reference)

|Decision|Choice|Why|
|---|---|---|
|LLM|Phi-3-mini-4k-instruct Q4_K_M (2.3GB)|MIT license, fits 8GB RAM|
|ASR|Whisper.cpp ggml-tiny.en (75MB)|<300ms latency on CPU|
|Embeddings|all-MiniLM-L6-v2 (384-dim)|Apache 2.0, fast, sufficient|
|HRTF Dataset|SADIE II Subject 002|Smallest, most compatible|
|Vector Search|FAISS IndexFlatIP|No training needed under 500 nodes|
|Graph|networkx DiGraph|In-memory, simple, sufficient|
|DB|SQLite|No Postgres needed for hackathon scope|
|Backend|FastAPI + uvicorn|Native async WebSocket, not Flask|
|Frontend|Next.js 15 + Three.js + Zustand|As per roadmap|
|Audio routing|VB-Cable (Win) / BlackHole (Mac)|Virtual cable for demo|

---

## ⚡ Latency Budget (non-negotiable)

|Component|Target|
|---|---|
|HRTF rendering|< 12ms|
|Whisper ASR|< 300ms per chunk|
|Phi-3-mini inference|< 500ms|
|Q&A end-to-end|< 700ms|
|WebSocket broadcast|< 100ms to all clients|

---

## 🚫 Hard Rules (never break these)

1. **Never run NLP inside the PyAudio audio callback** — audio will glitch. NLP runs on a separate thread via `queue.Queue`.
2. **Never commit model files to Git** — Phi-3-mini is 2.3GB. `/backend/models/` and `/backend/hrtf/` must be in `.gitignore`.
3. **Never use Web Audio API stereo panning and call it spatial audio** — Samsung judges will catch it. Must be real HRTF convolution.
4. **Never start frontend before Phase 2 AI core is verified** — frontend has zero value without working events.
5. **Pre-warm Phi-3-mini on server start** — first inference takes 3-4s without this.

---

## 📦 Model Downloads Status

|Model|Size|Location|Downloaded?|
|---|---|---|---|
|Phi-3-mini-4k-instruct Q4_K_M.gguf|2.3 GB|`backend/models/phi3/`|⬜|
|ggml-tiny.en.bin (Whisper)|75 MB|`backend/models/whisper/`|⬜|
|all-MiniLM-L6-v2|80 MB|auto-downloaded|⬜|
|SADIE II HRTF dataset|~200 MB|`backend/hrtf/sadie2/`|⬜|
|pyannote/speaker-diarization-3.1|~300 MB|HF cache|⬜|

> Update ⬜ → ✅ as each model downloads.

---

## 🌐 Ports & Endpoints

|Service|URL|
|---|---|
|FastAPI backend|`http://localhost:8000`|
|WebSocket|`ws://localhost:8000/ws/{session_id}`|
|Next.js frontend|`http://localhost:3000`|
|FastAPI docs|`http://localhost:8000/docs`|

---

## 📋 How to Use This File

**Starting a Claude session:**

> Paste this entire file and say: "You are working on SpatialVoiceAI. Read this context file. My task today is: [describe your task]."

**Ending a Claude session:**

> Update the "Current Status" section above with today's date, what's now working, and any new blockers. Then add a 5-line entry to SESSION_LOG.md.

**For your teammate:**

> Pull the latest `CONTEXT.md` from GitHub before starting any session. Push updated version after.