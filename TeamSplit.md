# SpatialVoiceAI — Team Task Split

> Khush = Pentium N3710, 4GB RAM. Can write, push, review code. Cannot run models, audio pipeline, or backend.
> Teammate = 8GB+ modern CPU. Runs everything. Dev + demo machine.
> Transfer method: Git push/pull. Khush writes code → pushes → teammate pulls → runs.
> Rule: Khush never wastes time trying to run Python AI code locally. That time goes to frontend.

---

## The Core Split Logic

| If the task requires... | Who does it |
|---|---|
| Running Phi-3, Whisper, speechbrain, PyAudio, FAISS | **Teammate only** |
| Writing Python files (no execution needed) | **Khush can draft, teammate verifies** |
| Next.js, React, TypeScript, Zustand, Three.js | **Khush only** |
| Testing AI pipeline latency | **Teammate only** |
| UI/UX design decisions | **Khush only** |
| Git repo management | **Khush owns** |
| Blueprint PDF | **Khush writes, teammate reviews specs** |
| Running the demo | **Teammate's machine, Khush drives UI** |

---

## PHASE 0 — Setup & Verification

### Khush (your shit laptop — do these RIGHT NOW)
- [ ] Create GitHub repo — README + .gitignore for Python + Node + models
- [ ] Create full folder structure: `mkdir -p backend/{audio,asr,diarization,llm,graph,agents,api,db,models,hrtf} frontend docs demo audio_samples`
- [ ] Initialize Next.js 15: `npx create-next-app@latest frontend --typescript --tailwind --app`
  - This runs fine on your laptop. Node/npm work. No models needed.
- [ ] Install frontend deps: `npm install three @react-three/fiber @react-three/drei framer-motion zustand clsx`
- [ ] Install frontend dev deps: `npm install -D @types/three`
- [ ] Create `frontend/.env.local`: `NEXT_PUBLIC_API_URL=http://localhost:8000`
- [ ] Create `lib/store.ts` — full Zustand store with all slice shapes (empty state, typed)
- [ ] Create `lib/api.ts` — typed API client, all functions stubbed with `TODO`
- [ ] Push everything to GitHub
- [ ] **Write the Blueprint** (3-4 hrs, your laptop, just Google Docs + the blueprint prompt file)

### Teammate (good laptop — run these)
- [ ] Pull repo from GitHub
- [ ] Install Python 3.11, ffmpeg, Git
- [ ] Install VB-Cable (Windows) or BlackHole (macOS)
- [ ] `pip install fastapi uvicorn pyaudio scipy numpy torch silero-vad speechbrain sentence-transformers faiss-cpu networkx aiosqlite llama-cpp-python python-dotenv pydantic`
- [ ] Download Phi-3-mini Q4_K_M.gguf (2.3GB) → `backend/models/phi3/`
- [ ] Download ggml-small.en.bin (244MB) → `backend/models/whisper/`
- [ ] Download SADIE II HRTF dataset → `backend/hrtf/sadie2/`
- [ ] Create `backend/.env` with all model paths
- [ ] **VERIFY Phi-3:** `python -c "from llama_cpp import Llama; import time; llm=Llama('backend/models/phi3/phi3.gguf'); t=time.time(); llm('Hello',max_tokens=50); print(time.time()-t)"` — record ms
- [ ] **VERIFY Whisper:** Run on 5s WAV — confirm transcript + latency <400ms
- [ ] **VERIFY speechbrain:** Run ECAPA on 5s two-speaker WAV — confirm speaker IDs
- [ ] **VERIFY HRTF:** Load .sofa/.wav files, print impulse response array shapes
- [ ] Record 2-minute test WAV with two speakers (one decision, one action item, one question)
- [ ] Report latency numbers to Khush

---

## PHASE 1 — Backend Foundation

### Khush (write code, push, don't run)
- [ ] Draft `backend/config.py` — all constants: SAMPLE_RATE, CHUNK_SIZE, model paths from env, agent config. **You can write this without running it.** Push when done.
- [ ] Draft `backend/db/schema.py` — SQLite table definitions (sessions, utterances, edges, action_items, qa_history). Pure Python, no execution needed. Push.
- [ ] Draft `backend/db/session_utils.py` — aiosqlite CRUD stubs with correct function signatures and docstrings. Teammate fills in SQL bodies.
- [ ] Draft `backend/api/session.py` — FastAPI router with correct route signatures, request/response Pydantic models. Stub bodies return hardcoded test data. Push.
- [ ] Draft `backend/api/graph.py` — route signature + Pydantic response model. Stub returns `{"nodes": [], "edges": []}`.
- [ ] Draft `backend/api/qa.py` — route signature. Stub returns `{"answer": "TODO", "citations": []}`.
- [ ] Draft `backend/api/ws.py` — WebSocket route + connection registry dict structure. Teammate implements the async logic.
- [ ] Define all WebSocket event envelope types as Pydantic models in `backend/models/events.py`

### Teammate (implement + run)
- [ ] Pull Khush's stubs
- [ ] Implement `db/schema.py` — `init_db()` with aiosqlite, create all 5 tables
- [ ] Implement `db/session_utils.py` — all CRUD functions with real SQL
- [ ] Implement `main.py` — FastAPI app, lifespan startup (DB init + Phi-3 pre-warm), CORS, router includes
- [ ] Implement `api/ws.py` — async WebSocket accept, connection registry, `broadcast()` helper
- [ ] Wire all routers into main.py. Run server: `uvicorn main:app --reload`
- [ ] Test all routes in Postman — verify valid JSON, correct status codes
- [ ] Test WebSocket echo in Postman WS client
- [ ] Report: "Phase 1 done, all routes green"

---

## PHASE 2 — AI Core

### Khush (write, don't run)
- [ ] Draft `backend/audio/hrtf_renderer.py` — class skeleton with `__init__(hrtf_dir)` and `render(chunk, speaker_id) -> np.ndarray`. Add type hints and docstring explaining the fftconvolve approach. Teammate fills in implementation.
- [ ] Draft `backend/llm/phi3_runner.py` — class skeleton with `run(system_prompt, user_prompt, max_tokens) -> str`. Add the JSON retry logic structure (try parse → retry once → return error dict). Teammate wires llama-cpp.
- [ ] **Write all 4 agent Phi-3 prompts** (this is your most important Phase 2 contribution):
  - `backend/agents/prompts/event_detection.py` — 3-shot prompt for classifying utterances as decision/action_item/question/disagreement/none. JSON output schema. Test on paper with 10 example utterances.
  - `backend/agents/prompts/action_extraction.py` — prompt for extracting owner, task text, deadline_hint from an action_item utterance. JSON output schema.
  - `backend/agents/prompts/qa_with_citations.py` — RAG prompt that takes a question + 5 context utterances → returns answer + citation_node_ids. JSON output schema.
  - Write these carefully. They are the core of the AI quality. Bad prompts = bad demo.
- [ ] Draft `backend/agents/coordinator.py` — AgentCoordinator class skeleton. `process_utterance()` method signature. Tool registry dict structure. Teammate implements the actual dispatch logic.
- [ ] Draft `backend/graph/graph_builder.py` — class skeleton, method signatures for `add_utterance()`, `get_similar()`, `to_dict()`. Teammate implements FAISS + networkx logic.

### Teammate (implement + verify everything)
- [ ] Implement `audio/hrtf_renderer.py` — load SADIE II IRs, fftconvolve stereo render, measure <10ms
- [ ] Implement `audio/vad.py` — Silero VAD wrapper, `is_speech(chunk) -> bool`
- [ ] Implement `asr/whisper_runner.py` — 3s accumulator, temp WAV, Whisper.cpp call, return text+confidence+latency
- [ ] Implement `diarization/speaker_id.py` — speechbrain ECAPA, 5s sliding window, return SPK_0/SPK_1
- [ ] Implement `llm/phi3_runner.py` using Khush's skeleton + prompts
- [ ] Implement `graph/graph_builder.py` — networkx DiGraph, MiniLM embed, FAISS IndexFlatIP, semantic edge creation
- [ ] Implement `graph/graph_serializer.py` — SQLite ↔ networkx round-trip, embeddings as base64
- [ ] Implement all 4 agents using Khush's prompts + skeletons
- [ ] Implement `agents/coordinator.py`
- [ ] Wire real QA logic into `api/qa.py`
- [ ] **TEST each component independently** before wiring
- [ ] **RUN end-to-end pipeline on test WAV** — log all latencies, share with Khush

---

## PHASE 3 — Audio Engine

### Khush (write, don't run)
- [ ] Draft `backend/audio/audio_engine.py` — AudioEngine thread class skeleton:
  - `__init__`, `start()`, `stop()` signatures
  - `AudioQueue` and `ASRQueue` with maxsize + drop-oldest policy (use `queue.Queue`)
  - `audio_level` float property
  - PyAudio callback function signature with correct type hints
  - Inline comments explaining the threading model for teammate
- [ ] Draft `backend/nlp_engine.py` — NLPEngine thread class skeleton. `asyncio.run_coroutine_threadsafe` pattern for WS bridge. Teammate implements the loop.

### Teammate (implement + run)
- [ ] Implement AudioEngine — PyAudio callback, queue drain, VAD filter, HRTF render, playback stream
- [ ] Implement NLPEngine — reads ASRQueue, calls coordinator, bridges to asyncio WS broadcast
- [ ] Connect both engines to FastAPI lifespan startup
- [ ] **TEST audio only (no NLP):** 10 minutes of audio, no glitches, stable memory
- [ ] **TEST full pipeline:** Play test audio through VB-Cable → confirm transcript events appear in backend logs in real-time

---

## PHASE 4 — Frontend (Khush solo, teammate is hands-off)

> Teammate: your only job during Phase 4 is to keep the backend running and answer Khush's integration questions.

### Khush owns all of this
- [ ] Build complete `lib/store.ts` — all Zustand slices: session, transcript (utterances[]), actionItems[], qaHistory[], audioLevels ({SPK_0, SPK_1}), graphData (nodes[], edges[])
- [ ] Build complete `lib/api.ts` — `startSession()`, `endSession()`, `submitQuestion()`, `getGraph()` — all typed with fetch + error handling
- [ ] Build `hooks/useWebSocket.ts` — WS connect, JSON parse, Zustand dispatch by event type, exponential backoff reconnect
- [ ] Build `hooks/useAudioLevel.ts` — Web Audio API AnalyserNode, RMS per frame, 30fps updates
- [ ] Build `app/page.tsx` — landing: session name input, start button, dark cinematic style, Orbitron heading, glassmorphic card
- [ ] Build `app/call/[id]/page.tsx` — two-panel layout (60/40 split), SpeakerModal on mount
- [ ] Build `app/call/[id]/ended/page.tsx` — KPI cards, full transcript, export button
- [ ] Build `components/SpatialCanvas.tsx` — Three.js scene, two orbs at az=330/30 positions, glow pulse from audioLevels, particle background, speaker name labels
- [ ] Build `components/TranscriptRibbon.tsx` — utterance cards, speaker color stripe, EventFlag chips, slide-in animation, auto-scroll with user-scroll detection
- [ ] Build `components/EventFlag.tsx` — 4 variants (amber diamond/teal circle/red triangle/blue tag), hover tooltip with confidence + evidence
- [ ] Build `components/ActionItemPanel.tsx` — collapsible section, owner badge, task text, deadline hint, pop-in animation
- [ ] Build `components/QADrawer.tsx` — slide-up panel, text input, loading dots, citation chips, scroll-to-utterance on chip click
- [ ] Build `components/SpeakerModal.tsx` — two name inputs, "Start Listening" button
- [ ] (Stretch) Build `components/GraphView.tsx` — d3-force directed graph, nodes colored by speaker
- [ ] Connect all components to Zustand store
- [ ] **TEST with mock data:** Manually inject 20 WS events → confirm all UI updates, animations smooth
- [ ] **TEST with live backend:** Confirm real events update UI correctly

---

## PHASE 5 — Integration (both, in the same room or call)

### Khush
- [ ] Open browser on teammate's machine — confirm WS connection in DevTools
- [ ] Drive the demo UI during integration testing — you're the "user"
- [ ] Test Q&A: type questions, verify citation chips scroll to correct utterances
- [ ] Test two-tab sync: two browser windows, confirm both update simultaneously
- [ ] Report UI bugs to yourself and fix them

### Teammate
- [ ] Run full backend stack: uvicorn + AudioEngine + NLPEngine
- [ ] Route test audio through VB-Cable
- [ ] Monitor Python process memory over 10 minutes — confirm no leak
- [ ] Confirm WS broadcast reaches all connected clients
- [ ] Log real latencies for all 5 pipeline components — report to Khush
- [ ] Fix top backend bugs from integration run

---

## PHASE 6 — Polish

### Khush (frontend polish)
- [ ] Dominant speaker orb drift animation (>30s uninterrupted → drift 8% toward center)
- [ ] Interruption pulse (red flash on interrupter's orb, 500ms)
- [ ] Transcript card word-by-word reveal animation (80ms/word fake streaming)
- [ ] Latency badge on companion panel (green/amber/red by ms threshold)
- [ ] Speaker name labels on orbs — float above, fade when speaking
- [ ] Cmd/Ctrl+K shortcut → open QADrawer
- [ ] Export button → `GET /session/{id}/export` → download JSON
- [ ] Mobile responsive check (<768px stacks panels)
- [ ] Final visual pass: spacing, typography, dark theme consistency

### Teammate (backend polish)
- [ ] Whisper hallucination deduplication (identical consecutive output → discard)
- [ ] WebSocket rate limiting (max 2 msg/sec per session)
- [ ] `GET /session/{id}/export` → Samsung Notes compatible JSON
- [ ] `GET /session/{id}/stats` → dominant_speaker, interruption_count, avg_latency, topic_clusters
- [ ] Tune EventDetection Phi-3 prompts based on real test audio results
- [ ] **Build demo/seed.db** — run full pipeline on scripted demo audio, save SQLite
- [ ] Verify Q&A works on seed.db for all 5 planned demo questions

---

## PHASE 7 — Demo Prep (both)

### Khush
- [ ] Write README.md — setup instructions, architecture diagram, demo video link
- [ ] Record 60s demo video for README
- [ ] Final GitHub cleanup — no .env, no model files, no large WAVs
- [ ] Rehearse demo script 5 times as the "user" driving the UI

### Teammate
- [ ] Record scripted demo audio (2 min): decision + action item + question + disagreement
- [ ] Prepare backup pre-rendered binaural WAV (if VB-Cable fails during demo)
- [ ] Verify Galaxy Buds / headphones spatial positions are clearly audible
- [ ] Test demo fallback: kill live pipeline → switch to seed.db → confirm Q&A still works
- [ ] Rehearse the backend explanation for judges (what each agent does, latency numbers)

---

## PHASE 8 — Blueprint (Khush solo, today/tomorrow, 3-4 hrs)

> Teammate: review the tech specs section for accuracy only. Khush writes everything.

- [ ] Download blueprint .pptx template from ennovatex.io
- [ ] Use BLUEPRINT_PROMPT.md — paste into Claude, get all 8 slide texts generated
- [ ] Fill slides with generated content — add architecture diagram (draw.io or Excalidraw)
- [ ] Get teammate to verify: model names, sizes, latency numbers are correct
- [ ] Export as PDF, rename correctly, send from institute email
- [ ] **Deadline: May 13, 2026**

---

## Summary: What Khush Can Always Do On His Laptop

| Task type | Can do on Pentium N3710? | Notes |
|---|---|---|
| Write any Python file | ✅ Yes | VS Code + syntax check, no execution |
| Write TypeScript/React/Next.js | ✅ Yes | Node runs fine |
| Run `npm run dev` for frontend | ✅ Yes | Next.js UI works without backend |
| Design Phi-3 prompts | ✅ Yes | Just text files |
| Git push/pull | ✅ Yes | |
| Write Blueprint (Google Docs/pptx) | ✅ Yes | |
| Excalidraw/draw.io architecture diagram | ✅ Yes | Browser-based |
| Run FastAPI with no AI models | ✅ Yes | Stub routes only |
| Test frontend with mock WS data | ✅ Yes | Use a WS mock in browser |
| Run Phi-3, Whisper, speechbrain | ❌ No | AVX2 missing, 4GB RAM too low |
| Run PyAudio pipeline | ❌ No | Will freeze the machine |
| Run FAISS + sentence-transformers | ❌ No | RAM exhaustion |
| Run llama-cpp-python | ❌ No | Crashes without AVX2 |
