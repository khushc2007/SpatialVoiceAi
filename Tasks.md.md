# SpatialVoiceAI — Master Task Checklist

> 87 tasks · 7 phases · 18 days Status: `[ ]` Not started · `[~]` In progress · `[x]` Done · `[!]` Blocked Rule: If you are behind by more than 2 tasks in a day, cut polish — never cut core pipeline.

---

## PHASE 0 — Setup & Planning · Day 1

**Milestone:** All tools installed. Models downloading. Repo initialized.

- [ ] Install Python 3.11, Node 20 LTS, Git, VS Code, ffmpeg
- [ ] Install VB-Cable (Windows) or BlackHole (macOS) virtual audio cable
- [ ] Create HuggingFace account — accept pyannote model license agreement
- [ ] Create GitHub repo — initialize with README and .gitignore for Python + Node
- [ ] Create full project folder structure (all 19 directories)
- [ ] Start download: Phi-3-mini Q4_K_M.gguf (2.3GB) — leave running overnight
- [ ] Start download: Whisper.cpp ggml-tiny.en.bin (75MB)
- [ ] Start download: SADIE II HRTF dataset (200MB) from York acoustics website
- [ ] Install Python deps: `pip install fastapi uvicorn pyaudio scipy numpy torch silero-vad`
- [ ] Install Python deps: `pip install pyannote.audio llama-cpp-python sentence-transformers faiss-cpu networkx python-dotenv pydantic`
- [ ] Initialize Next.js frontend: `npx create-next-app@latest frontend --typescript --tailwind --app`
- [ ] Install frontend deps: `npm install three @react-three/fiber @react-three/drei framer-motion zustand clsx`
- [ ] Create `.env` file with: MODEL_PHI3_PATH, MODEL_WHISPER_PATH, HRTF_DIR, DB_PATH, HF_TOKEN
- [ ] Record or source a 2-minute test call WAV with two clear speakers

---

## PHASE 1 — Backend Foundation · Days 2–3

**Milestone:** FastAPI starts. All routes return valid JSON. WebSocket echoes test message. SQLite file created.

- [ ] Create `main.py` — FastAPI app, CORS middleware (`allow_origins=['http://localhost:3000']`), router includes
- [ ] Create `config.py` — all constants: SAMPLE_RATE=48000, CHUNK_SIZE=960, model paths
- [ ] Create SQLite database with 4 tables: sessions, utterances, edges, action_items
- [ ] Create `POST /session/start` route — returns `{session_id, created_at}`
- [ ] Create `GET /session/{id}` route — returns session metadata + node count
- [ ] Create `GET /graph/{id}` route — returns serialized graph nodes and edges
- [ ] Create `POST /qa` route — stub returns "TODO" (real logic in Phase 2)
- [ ] Create `POST /session/{id}/end` route — stub, returns OK
- [ ] Create `WS /ws/{session_id}` WebSocket endpoint — echoes test message
- [ ] Test all routes in Postman — confirm all return valid JSON
- [ ] Verify WebSocket connection in Postman WebSocket client

---

## PHASE 2 — AI Core Integration · Days 4–6

**Milestone:** Full AI pipeline runs end-to-end on test audio. Whisper → Phi-3 → graph → Q&A working.

- [ ] Verify Phi-3-mini loads and responds to test prompt in Python shell (< 5 seconds)
- [ ] Verify Whisper.cpp transcribes a 5-second WAV correctly
- [ ] Verify MiniLM encodes a sentence to a 384-dim vector
- [ ] Verify SADIE II HRTF files load and impulse response arrays are correct shape
- [ ] Build `hrtf_renderer.py` — loads HRTF for az=330 and az=30 on init
- [ ] Test HRTF renderer: pass 960-sample sine wave, get stereo output, play on headphones
- [ ] Build `whisper_runner.py` — accumulates 3s audio, writes temp WAV, calls Whisper
- [ ] Test Whisper runner on 30s of test audio — confirm transcript output
- [ ] Build `phi3_runner.py` — loads Phi-3-mini, 5-shot system prompt, JSON parser
- [ ] Test Phi-3-mini with 5 different conversation windows — confirm JSON output for each
- [ ] Build `graph_builder.py` — networkx DiGraph, MiniLM embeddings, FAISS index, SQLite serializer
- [ ] Test graph builder: add 10 utterance nodes, run FAISS search, confirm relevant results
- [ ] Wire NLP pipeline end-to-end on test audio: Whisper → Phi-3 → graph
- [ ] Implement real `/qa` endpoint with graph-RAG logic — test 5 questions

---

## PHASE 3 — Feature Development · Days 7–11

**Milestone:** Full audio pipeline + NLP engine + WebSocket broadcast all running. Action items saved to DB.

- [ ] Build `AudioEngine` thread class — PyAudio callback mode, AudioQueue and ASRQueue
- [ ] Integrate HRTF renderer into AudioEngine — confirm no audio callback blocking
- [ ] Build VAD integration — Silero VAD labels each frame, silence frames discarded
- [ ] Integrate pyannote diarization on 3s sliding window — speaker IDs assigned to frames
- [ ] Test full audio pipeline: capture → VAD → HRTF → playback (no NLP yet)
- [ ] Build `NLPEngine` thread — reads from ASRQueue, runs Whisper → Phi-3 → graph
- [ ] Connect NLPEngine to FastAPI WebSocket broadcast via asyncio bridge
- [ ] Test: play test audio, confirm transcript cards appear in backend logs with event flags
- [ ] Implement action item extraction — write to SQLite on each action_item event
- [ ] Implement `/session/{id}/end` — serialize full graph, compile action items, save transcript
- [ ] Test action item extraction on test audio — confirm correct owner attribution

---

## PHASE 4 — Frontend Development · Days 9–13

**Milestone:** All pages built and connected to Zustand store. Works with manually injected test data.

- [ ] Create Zustand store with slices: session, transcript (utterances[]), actionItems[], qaHistory[]
- [ ] Create `useWebSocket` hook — connects, parses events, dispatches to store
- [ ] Build `SessionForm` component — session name input, start button, calls `/session/start`
- [ ] Build `/call/[id]` page layout — left: SpatialCanvas, right: CompanionPanel
- [ ] Build `SpatialCanvas` component — Three.js scene, two orbs with color and position
- [ ] Wire `AudioLevel` hook to SpatialCanvas — orbs pulse with audio level
- [ ] Build `TranscriptRibbon` component — scrollable list of utterance cards
- [ ] Build `EventFlag` component — 4 variants: amber diamond, teal circle, red triangle, blue tag
- [ ] Build `QADrawer` component — slide-up panel, text input, response area with citations
- [ ] Build `ActionItemPanel` component — live list, owner badge, task text
- [ ] Connect TranscriptRibbon to Zustand store — new utterances append automatically
- [ ] Connect QADrawer to `/qa` API endpoint — show loading state during inference
- [ ] Implement citation tap → `scrollToUtterance` + 2s highlight animation
- [ ] Build `/call/[id]/ended` page — summary, full transcript, export button
- [ ] Test all pages in browser with manually injected test data

---

## PHASE 5 — Integration · Days 12–14

**Milestone:** Full system running end-to-end for 5 minutes. No crashes. Q&A works from browser.

- [ ] Test FastAPI ↔ SQLite full round trip: start session → add data → retrieve
- [ ] Test WebSocket from browser: open `/call/[id]`, confirm WS connection in DevTools
- [ ] Push test event from Python, confirm it appears in TranscriptRibbon
- [ ] Run HRTF renderer output while UI is open — confirm no audio lag
- [ ] First full end-to-end run: test audio → all pipeline → UI updates → no crashes
- [ ] Test Q&A from UI: type question, submit, confirm answer with citations
- [ ] Two-tab sync test: two browser tabs receive same WS events simultaneously
- [ ] Run 5-minute continuous test: no audio glitches, no memory leak in terminal

---

## PHASE 6 — Polish & Differentiation · Days 15–16

**Milestone:** All performance fixes done. Visual polish complete. 10-minute load test passes.

- [ ] Add dynamic orb drift for dominant speaker (>30s dominance → orb drifts 8% to center)
- [ ] Add interruption pulse animation (red flash on interrupter's orb for 500ms)
- [ ] Add transcript card typing animation (character-by-character reveal at 60fps)
- [ ] Add EventFlag hover tooltips with confidence score + evidence text
- [ ] Add speaker name assignment modal on call start (replaces SPK_0/SPK_1 everywhere)
- [ ] Pre-warm Phi-3-mini on server startup (one dummy inference)
- [ ] Add Whisper hallucination deduplication (discard repeated consecutive output)
- [ ] Add WebSocket message rate limiting (max 1 message/second, debounce queue)
- [ ] Test performance: full pipeline under load for 10 minutes
- [ ] Add real-time translation toggle — EN/HI/KN via Helsinki-NLP (if time permits)

---

## PHASE 7 — Demo Preparation · Days 17–18

**Milestone:** Demo rehearsed 5+ times. Pre-seeded session ready. README complete. Phase 1 blueprint submitted.

- [ ] Record final 2-minute demo call audio with scripted content:
    - One clear decision ("Let's go with Option A")
    - One action item ("Rahul, can you send the report by Friday?")
    - One question
    - One moment of disagreement
- [ ] Run full pipeline on demo audio — save resulting SQLite session
- [ ] Verify Q&A works on pre-seeded session for all 3 planned questions
- [ ] Test headphones on demo laptop — confirm spatial positions are clear
- [ ] Rehearse full 2:30 demo script 5 times
- [ ] Prepare backup pre-rendered binaural WAV for audio fallback
- [ ] Write README.md with setup instructions, architecture diagram, demo video link
- [ ] Push clean final code to GitHub
- [ ] Prepare Phase 1 Blueprint PDF (due May 13)

---

## Phase 1 Blueprint Checklist (due May 13)

- [ ] Problem statement restated in your own words — 1 paragraph, technically accurate
- [ ] System architecture diagram included
- [ ] Five named differentiators with 1-sentence explanations each
- [ ] Specific model names and sizes stated (not "an open-source LLM")
- [ ] Latency budget table: HRTF <12ms, Whisper <300ms, Phi-3-mini <500ms, Q&A <700ms
- [ ] Metrics definition: localization accuracy, ASR WER, Q&A accuracy
- [ ] Samsung ecosystem alignment section: Galaxy Buds, Galaxy AI, Samsung Link
- [ ] Open-source compliance statement: all model licenses listed
- [ ] Sent from institute email to ennovatex.io@samsung.com before May 13