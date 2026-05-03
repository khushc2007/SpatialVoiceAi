# SpatialVoiceAI — Master Task Checklist (v2, 9.9/10 Build)

> 104 tasks · 8 phases · Full build window (blueprint = 3-4hrs, build = full deadline)
> Status: `[ ]` Not started · `[~]` In progress · `[x]` Done · `[!]` Blocked
> Rule: If behind by more than 3 tasks in a day, cut polish — never cut core pipeline.
> Two people: Khush owns frontend + integration. Teammate owns backend Python + models.

---

## PHASE 0 — Environment & Verification · Day 1 (both people, 4hrs)

**Milestone:** Every model loads. Phi-3 inference measured. Whisper transcribes test audio. Repo initialized. No surprises on Day 2.

**Khush**
- [ ] Create GitHub repo — initialize with README, .gitignore (Python + Node + model dirs)
- [ ] Create full project folder structure (all directories from CONTEXT.md)
- [ ] Initialize Next.js 15 frontend: `npx create-next-app@latest frontend --typescript --tailwind --app`
- [ ] Install frontend deps: `npm install three @react-three/fiber @react-three/drei framer-motion zustand clsx`
- [ ] Install frontend dev deps: `npm install -D @types/three`
- [ ] Create `.env.local` with: `NEXT_PUBLIC_API_URL=http://localhost:8000`
- [ ] Create `lib/store.ts` — Zustand store skeleton with all slice shapes (empty arrays/nulls)
- [ ] Create `lib/api.ts` — typed API client skeleton (all functions stubbed)

**Teammate**
- [ ] Install Python 3.11, ffmpeg, Git on demo machine
- [ ] Install VB-Cable (Windows) or BlackHole (macOS)
- [ ] Create HuggingFace account — NOT needed for speechbrain (Apache 2.0, no gate)
- [ ] Download Phi-3-mini Q4_K_M.gguf (2.3GB) — leave running
- [ ] Download ggml-small.en.bin (244MB)
- [ ] Download SADIE II HRTF dataset from York acoustics website
- [ ] Install all Python deps: `pip install fastapi uvicorn pyaudio scipy numpy torch silero-vad speechbrain sentence-transformers faiss-cpu networkx aiosqlite llama-cpp-python python-dotenv pydantic`
- [ ] **VERIFICATION:** Run Phi-3-mini in Python shell — `time` a single inference. Record ms. If >800ms, flag immediately.
- [ ] **VERIFICATION:** Run Whisper small on a 5s WAV — confirm transcript output and latency
- [ ] **VERIFICATION:** Run speechbrain ECAPA on a 5s two-speaker WAV — confirm speaker IDs
- [ ] **VERIFICATION:** Load SADIE II HRTF .sofa or .wav files — confirm correct array shapes
- [ ] Create `.env` with: `MODEL_PHI3_PATH`, `MODEL_WHISPER_PATH`, `HRTF_DIR`, `DB_PATH`
- [ ] Record 2-minute test WAV with two clear speakers: one decision, one action item, one question

---

## PHASE 1 — Backend Foundation · Days 2–3 (Teammate leads, Khush reviews)

**Milestone:** FastAPI starts clean. All routes return valid typed JSON. WebSocket echoes. SQLite created with all 5 tables. aiosqlite confirmed async-safe.

- [ ] Create `config.py` — all constants: SAMPLE_RATE=48000, CHUNK_SIZE=960, WHISPER_CHUNK_SECONDS=3, DIARIZATION_WINDOW=5, all model paths, agent config
- [ ] Create `db/schema.py` — SQLite schema: `sessions`, `utterances`, `edges`, `action_items`, `qa_history` tables. Run `init_db()` on startup.
- [ ] Create `db/session_utils.py` — async CRUD: `create_session`, `get_session`, `insert_utterance`, `insert_edge`, `insert_action_item`, `insert_qa`, `get_session_full`
- [ ] Create `main.py` — FastAPI app, lifespan (model pre-warm + DB init), CORS (`http://localhost:3000`), include all routers
- [ ] Create `api/session.py` — `POST /session/start` → `{session_id, created_at}`, `GET /session/{id}` → session + node count, `POST /session/{id}/end` → full graph serialize + action item compile
- [ ] Create `api/graph.py` — `GET /graph/{id}` → all nodes and edges serialized (no numpy arrays — convert to lists)
- [ ] Create `api/qa.py` — `POST /qa` stub → `{"answer": "TODO", "citations": []}`
- [ ] Create `api/ws.py` — `WS /ws/{session_id}` → accept, echo test message, maintain connection registry dict
- [ ] Create `broadcast()` helper in ws.py — sends event envelope JSON to all connected clients for a session
- [ ] Test all routes in Postman — every route returns valid JSON, correct status codes
- [ ] Test WebSocket in Postman WS client — send "ping", receive "pong"
- [ ] Confirm aiosqlite writes don't block FastAPI event loop (test with concurrent requests)

---

## PHASE 2 — AI Core · Days 4–7 (Teammate owns, Khush assists on integration points)

**Milestone:** Full AI pipeline verified on test audio, end-to-end. Each agent produces correct output independently before wiring together.

### HRTF Renderer
- [ ] Build `audio/hrtf_renderer.py` — load SADIE II impulse responses for az=330° (SPK_0, left-front) and az=30° (SPK_1, right-front). Store as numpy arrays.
- [ ] Implement `render(audio_chunk, speaker_id)` → stereo numpy array using `scipy.signal.fftconvolve`
- [ ] **TEST:** Pass 960-sample 440Hz sine wave for each speaker → play on headphones → confirm distinct spatial positions
- [ ] **MEASURE:** Confirm render latency <10ms on demo machine

### VAD
- [ ] Build `audio/vad.py` — Silero VAD wrapper. `is_speech(chunk) → bool`. Silence frames do not enter ASRQueue.
- [ ] **TEST:** Run on test WAV with silence gaps — confirm silence frames are correctly rejected

### Whisper ASR
- [ ] Build `asr/whisper_runner.py` — accumulates audio frames until 3s, writes temp WAV, calls Whisper.cpp subprocess, returns `{text, confidence, duration_ms}`
- [ ] **TEST:** Run on 30s of test audio — confirm full transcript, measure per-chunk latency
- [ ] **MEASURE:** Must be <400ms on demo machine

### Speaker Diarization
- [ ] Build `diarization/speaker_id.py` — speechbrain ECAPA wrapper. Maintains 5s sliding window buffer. `identify(audio_chunk) → speaker_id`. Returns "SPK_0" or "SPK_1". First speaker seen = SPK_0.
- [ ] **TEST:** Run on 2-minute two-speaker WAV — confirm correct speaker assignment, no frequent flip-flopping
- [ ] **MEASURE:** Must be <200ms per window

### Phi-3 Runner
- [ ] Build `llm/phi3_runner.py` — llama-cpp-python wrapper. `run(system_prompt, user_prompt, max_tokens=256) → str`. Always returns JSON. On parse failure, retry once with stricter prompt. After 2 failures, return `{"error": "parse_failure"}`.
- [ ] **TEST:** Run 10 different prompts across all three agent use cases — confirm JSON output every time
- [ ] **MEASURE:** Must be <600ms on demo machine

### Graph Builder
- [ ] Build `graph/graph_builder.py` — networkx DiGraph. `add_utterance(node_data) → node_id`. Auto-embeds text with MiniLM. Auto-adds FAISS index entry. Auto-creates semantic edges to top-3 most similar existing nodes (cosine sim >0.5 threshold only).
- [ ] Build `graph/graph_serializer.py` — `to_sqlite(session_id)` serializes full graph to DB. `from_sqlite(session_id)` reconstructs graph. Embeddings stored as base64 blobs.
- [ ] **TEST:** Add 20 utterances, run FAISS search, confirm top results are semantically relevant

### Multi-Agent System
- [ ] Build `agents/transcription_agent.py` — receives Whisper output + speaker ID → creates utterance node → writes to graph → writes to SQLite utterances table → returns node_id
- [ ] Build `agents/event_agent.py` — receives utterance text + last 5 graph node texts as context → calls Phi-3 with 3-shot classification prompt → returns `{flag_type, confidence, evidence_text}` or `{flag_type: "none"}`
- [ ] Build `agents/action_agent.py` — fires only when event_agent returns `action_item` → calls Phi-3 with extraction prompt → returns `{owner_speaker, task_text, deadline_hint}` → writes to SQLite action_items
- [ ] Build `agents/qa_agent.py` — receives question string + session_id → FAISS retrieves top-5 nodes → builds context string with node texts → calls Phi-3 with RAG prompt → returns `{answer, citation_node_ids, latency_ms}`
- [ ] Build `agents/coordinator.py` — AgentCoordinator class. `process_utterance(audio_chunk, session_id)` runs the full pipeline: Whisper → Diarization → TranscriptionAgent → EventDetectionAgent → (conditionally) ActionItemAgent → broadcast WS events
- [ ] **TEST:** Run coordinator on full 2-minute test audio. Confirm every utterance produces a graph node, event flags appear, action items are extracted. Log all latencies.

### Wire Q&A Endpoint
- [ ] Replace stub in `api/qa.py` with real QAAgent call. Accept `{question, session_id}`, return `{answer, citation_node_ids, latency_ms}`
- [ ] **TEST:** Send 5 questions about the test audio via Postman. Confirm answers with correct citations.

---

## PHASE 3 — Audio Engine · Days 8–9 (Teammate)

**Milestone:** Full audio pipeline running: capture → VAD → HRTF → playback + parallel ASR queue feeding, no audio glitches over 10 minutes.

- [ ] Build `audio/audio_engine.py` — AudioEngine thread class:
  - PyAudio callback mode, callback pushes to `AudioQueue(maxsize=50)` with drop-oldest on overflow
  - Reads from AudioQueue: VAD filter → if speech, HRTF render (by speaker turn) → output to playback stream
  - If speech frame: also push raw frame to `ASRQueue(maxsize=20)` with drop-oldest on overflow
  - Exposes `audio_level` float (0.0–1.0) updated per frame for frontend orb pulse
- [ ] Integrate HRTF renderer — confirm no blocking inside audio callback (HRTF must run in engine thread, not callback)
- [ ] **TEST:** Run AudioEngine alone (no NLP) for 10 minutes — confirm no audio glitches, no memory growth
- [ ] **TEST:** Confirm audio_level updates at >20Hz for smooth orb animation

### NLP Engine
- [ ] Build `nlp_engine.py` — NLPEngine thread. Reads from ASRQueue. For each drained chunk, calls AgentCoordinator. Uses asyncio.run_coroutine_threadsafe to push WS broadcasts from thread context to FastAPI event loop.
- [ ] Connect NLPEngine startup to FastAPI lifespan
- [ ] **TEST:** Play test audio through VB-Cable with AudioEngine + NLPEngine running. Confirm transcript cards appear in backend logs in real-time.

---

## PHASE 4 — Frontend · Days 8–12 (Khush, parallel to Phase 3)

**Milestone:** All pages built, wired to Zustand. Works with manually injected test data from WebSocket. Visual polish at 80%.

### Foundation
- [ ] Build complete `lib/store.ts` — Zustand slices: `session` (id, name, status), `transcript` (utterances[]), `actionItems[]`, `qaHistory[]`, `audioLevels` ({SPK_0: 0, SPK_1: 0}), `graphData` (nodes[], edges[])
- [ ] Build complete `lib/api.ts` — `startSession()`, `endSession()`, `submitQuestion()`, `getGraph()` — all typed, all use `NEXT_PUBLIC_API_URL`
- [ ] Build `hooks/useWebSocket.ts` — connects to `ws://localhost:8000/ws/{session_id}`. Parses event envelope. Dispatches to correct Zustand slice based on `event` field. Reconnects on disconnect with exponential backoff.
- [ ] Build `hooks/useAudioLevel.ts` — Web Audio API AnalyserNode on mic input. Computes RMS per frame. Returns `{SPK_0: number, SPK_1: number}` updated at 30fps.

### Pages
- [ ] Build `app/page.tsx` — landing page. Session name input. Start button calls `startSession()`. On success, push to `/call/[id]`. Dark, cinematic, Orbitron heading, glassmorphic card.
- [ ] Build `app/call/[id]/page.tsx` — two-panel layout: left 60% = SpatialCanvas, right 40% = CompanionPanel (TranscriptRibbon + EventFlags + ActionItemPanel + QADrawer trigger)
- [ ] Build `app/call/[id]/ended/page.tsx` — post-call summary: KPI cards (utterances, action items, duration), full transcript, export button

### Components
- [ ] Build `SpatialCanvas.tsx` — Three.js scene. Two speaker orbs: SPK_0 at left-front position, SPK_1 at right-front. Orb size/glow pulses with `audioLevels` from store. Subtle particle field background. Smooth camera orbit on load.
- [ ] Build `TranscriptRibbon.tsx` — scrollable list. Each card: speaker color stripe, speaker name badge, utterance text, timestamp, EventFlag chips. New cards appear at bottom with slide-in animation. Auto-scroll to bottom unless user has scrolled up.
- [ ] Build `EventFlag.tsx` — 4 variants: amber diamond (decision), teal circle (action item), red triangle (disagreement), blue tag (question). Shows on utterance card. Hover reveals confidence + evidence text tooltip.
- [ ] Build `ActionItemPanel.tsx` — collapsible right panel section. Each item: owner badge (speaker color), task text, deadline hint if present. Appears with pop-in animation on new item.
- [ ] Build `QADrawer.tsx` — slide-up panel from bottom. Text input. Submit sends to `/qa`. Loading state (streaming dots). Response renders with inline citation chips. Clicking citation chip scrolls TranscriptRibbon to that utterance and highlights for 2s.
- [ ] Build `SpeakerModal.tsx` — modal on call start. Two name inputs for SPK_0 and SPK_1. "Start Listening" button. Names stored in Zustand, sent to backend via session start payload.
- [ ] (Stretch) Build `GraphView.tsx` — force-directed graph using d3-force. Nodes colored by speaker. Edge thickness = semantic similarity weight. Clicking node highlights utterance in TranscriptRibbon.

### Connect everything
- [ ] Connect TranscriptRibbon to Zustand transcript slice — new utterances append in real-time
- [ ] Connect SpatialCanvas orb pulse to Zustand audioLevels slice
- [ ] Connect ActionItemPanel to Zustand actionItems slice
- [ ] Connect QADrawer submit to api.ts `submitQuestion()` — show loading, render response
- [ ] Connect citation chip click to TranscriptRibbon scroll + highlight
- [ ] **TEST:** Inject 20 mock WebSocket events manually — confirm all UI updates correctly, animations smooth, no console errors

---

## PHASE 5 — Integration · Days 13–14 (both)

**Milestone:** Full system running end-to-end for 10 minutes. No crashes. Q&A works from browser. Two-tab sync confirmed.

- [ ] Full round-trip test: `POST /session/start` → AudioEngine starts → NLP pipeline runs → WS events appear in browser → session ends → `GET /session/{id}` returns complete data
- [ ] Test WebSocket from browser DevTools: confirm connection, confirm events appear in Network tab
- [ ] Push live audio through VB-Cable: confirm transcript cards appear in browser within 1.5s of speech
- [ ] Test Q&A from browser: type question, submit, confirm answer with highlighted citations in TranscriptRibbon
- [ ] Two-tab sync test: open two browser tabs on same session → confirm both receive identical WS events simultaneously
- [ ] 10-minute continuous test: no audio glitches, no memory leak (watch Python process RSS in Task Manager), no WS disconnects
- [ ] Measure and log real latencies for all 5 pipeline components. Compare against latency budget. If any miss, optimize before Phase 6.
- [ ] Fix top 5 bugs found during integration

---

## PHASE 6 — Polish & Differentiation · Days 15–16 (Khush leads frontend, teammate optimizes backend)

**Milestone:** Visual polish complete. Samsung ecosystem hooks implemented. Performance verified. Pre-seeded demo session ready.

### Backend polish
- [ ] Add Whisper hallucination deduplication — if output is identical to previous chunk output, discard
- [ ] Add WebSocket rate limiting — max 2 messages/second per session, debounce queue
- [ ] Add `GET /session/{id}/export` — returns Samsung Notes compatible JSON format for action items + transcript
- [ ] Add session analytics: `GET /session/{id}/stats` → `{dominant_speaker, interruption_count, avg_response_latency_ms, topic_clusters[]}`
- [ ] Tune EventDetectionAgent 3-shot prompts based on real test audio results — improve precision on action_item and decision flags

### Frontend polish
- [ ] Add dominant speaker orb drift animation — if one speaker >30s uninterrupted, orb drifts 8% toward center
- [ ] Add interruption pulse — when speaker changes mid-sentence (VAD detects overlap), red flash on interrupter's orb for 500ms
- [ ] Add transcript card text reveal animation — characters appear progressively as Whisper processes (fake streaming using word-by-word reveal at 80ms/word)
- [ ] Add latency indicator on companion panel — small badge showing last pipeline latency in ms. Green <600ms, amber 600-900ms, red >900ms.
- [ ] Add speaker name display on orbs in SpatialCanvas — floating text label above each orb, fades when speaking
- [ ] Add keyboard shortcut: `Cmd/Ctrl + K` opens QADrawer
- [ ] Add export button on ended page — triggers `GET /session/{id}/export`, downloads JSON
- [ ] Mobile responsive check — companion panel stacks below canvas on <768px

### Demo session
- [ ] Record final scripted demo audio (2 minutes):
  - Clear decision: "Let's go with Option A, final answer"
  - Clear action item: "Rahul, can you send the report by Friday?"
  - A question: "What's the current status on the API integration?"
  - A brief disagreement: "I don't think that's the right approach"
- [ ] Run full pipeline on demo audio — verify all 4 event types are correctly flagged
- [ ] Save resulting SQLite as `demo/seed.db` — this is the demo fallback
- [ ] Verify Q&A works on seed.db for 5 planned demo questions
- [ ] Test on Galaxy Buds (or any good headphones) — spatial positions must be clearly audible

---

## PHASE 7 — Demo Preparation · Days 17–18 (both)

**Milestone:** Demo rehearsed 5+ times. Fallback path tested. Blueprint submitted. GitHub clean.

- [ ] Rehearse full 2:30 demo script 5 times minimum. Time each run.
- [ ] Test fallback path explicitly: kill live pipeline mid-demo → switch to seed.db → Q&A still works
- [ ] Prepare backup pre-rendered binaural WAV — play this if VB-Cable fails
- [ ] Write README.md — setup instructions, architecture diagram, demo video link, model download instructions
- [ ] Push clean code to GitHub — no model files, no .env, no seed.db with personal data
- [ ] Record 60-second demo video for README

---

## PHASE 8 — Blueprint (3-4 hours, do this first before any code)

**Deadline: May 13, 2026. This is separate from the build.**

- [ ] Download blueprint template from ennovatex.io/files/AX_Hackathon_Phase1_Blueprint_Template.pptx
- [ ] Slide 1: Problem restatement — "Voice calls are ephemeral. No spatial presence, no structured memory, no queryable intelligence." 1 paragraph, technically precise.
- [ ] Slide 2: System architecture diagram — show the 5 layers: audio capture → spatial rendering → ASR + diarization → multi-agent pipeline → companion UI
- [ ] Slide 3: Five named differentiators:
  1. Real HRTF convolution (not stereo panning) — azimuth-accurate binaural rendering
  2. Live semantic conversation graph (queryable mid-call, not post-call)
  3. Multi-agent tool-chaining (4 specialized agents, not one monolithic LLM prompt)
  4. Graph-grounded Q&A with utterance-level citations
  5. Samsung ecosystem: Galaxy Buds spatial audio profile + Samsung Notes export
- [ ] Slide 4: Specific model names and sizes — Phi-3-mini-4k Q4_K_M (2.3GB, MIT), Whisper small (244MB, MIT), all-MiniLM-L6-v2 (80MB, Apache 2.0), speechbrain ECAPA (90MB, Apache 2.0), Silero VAD (2MB, MIT)
- [ ] Slide 5: Latency budget table — HRTF <10ms, Whisper <400ms, Diarization <200ms, EventAgent <600ms, Q&A <900ms
- [ ] Slide 6: Metrics — localization accuracy (HRTF azimuth error vs ground truth), ASR WER on test set, EventDetection precision/recall on labeled test audio, Q&A answer relevance (manual eval)
- [ ] Slide 7: Samsung ecosystem alignment — Galaxy Buds Pro HRTF profile, Galaxy AI on-device positioning, Samsung Notes export, Bixby capsule architecture compatibility
- [ ] Slide 8: Open source compliance — list every model and library with license. Statement: "All components are MIT, Apache 2.0, or BSD-3-Clause. No gated models. No commercial APIs. Released under Apache-2.0."
- [ ] Export as PDF. Rename: `teamname-problem08-spatialvoiceai.pdf`
- [ ] Send from institute email to ennovatex.io@samsung.com with subject: "AX Hackathon Phase 1 Submission | Problem 08 | [Team Name]"

---

## What Was Changed From v1 (and Why)

| v1 | v2 | Reason |
|---|---|---|
| Whisper tiny (75MB) | Whisper small (244MB) | Better WER, still <400ms on modern CPU. Demo quality matters. |
| pyannote diarization | speechbrain ECAPA | Apache 2.0 license, no HF gated model, lighter weight |
| Single monolithic Phi-3 prompt | 4-agent tool-chaining | Judges are told to look for agentic workflows. This is the differentiator. |
| SQLite (sync) | aiosqlite (async) | FastAPI is async. Sync SQLite blocks the event loop under load. |
| No agent coordinator | AgentCoordinator class | Clean separation of concerns, testable independently, matches hackathon criteria for "agentic systems" |
| No export feature | Samsung Notes JSON export | Concrete Samsung ecosystem hook judges can verify |
| No session analytics | `GET /session/{id}/stats` | Judges will ask "what insights does this give?" — have an answer |
| No latency indicator in UI | Latency badge on companion panel | Shows judges you're measuring what you claim |
| Demo fallback on Day 17 | Demo fallback on Day 15 | Never build safety nets the day before submission |
