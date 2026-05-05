# SpatialVoiceAI — Team Task Split (v3)

> Khush = Pentium N3710, 4GB RAM. Can write, push, review code. Cannot run models, audio pipeline, or backend. Teammate = 8GB+ modern CPU. Runs everything. Dev + demo machine. Transfer method: Git push/pull. Khush writes code → pushes → teammate pulls → runs. Rule: Khush never wastes time trying to run Python AI code locally. That time goes to frontend.

---

## The Core Split Logic

|If the task requires...|Who does it|
|---|---|
|Running Phi-3, Whisper, speechbrain, PyAudio, FAISS|**Teammate only**|
|Writing Python files (no execution needed)|**Khush can draft, teammate verifies**|
|Next.js, React, TypeScript, Zustand, Three.js|**Khush only**|
|Testing AI pipeline latency|**Teammate only**|
|UI/UX design decisions|**Khush only**|
|Git repo management|**Khush owns**|
|Blueprint PDF|**Khush writes, teammate reviews specs**|
|Running the demo|**Teammate's machine, Khush drives UI**|

---

## PHASE 0 — Setup & Verification

### Khush

- [x] Create GitHub repo — README + .gitignore for Python + Node + models
- [x] Create full folder structure (including new `backend/memory/` dir)
- [x] Initialize Next.js 15 frontend
- [x] Install frontend deps: three, @react-three/fiber, @react-three/drei, framer-motion, zustand, clsx
- [x] Install frontend dev deps: @types/three
- [x] Create `frontend/.env.local`: `NEXT_PUBLIC_API_URL=http://localhost:8000`
- [x] Create `lib/store.ts` — full Zustand store with all slice shapes (now includes affectStates, topicClusters, dominanceStats slices)
- [x] Create `lib/api.ts` — typed API client, all functions stubbed with TODO
- [x] Push everything to GitHub
- [x] **Write the Blueprint**

### Teammate (good laptop — run these)

- [ ] Pull repo from GitHub
- [ ] Install Python 3.11, ffmpeg, Git
- [ ] Install VB-Cable (Windows) or BlackHole (macOS)
- [ ] `pip install fastapi uvicorn pyaudio scipy numpy torch silero-vad speechbrain sentence-transformers faiss-cpu networkx aiosqlite llama-cpp-python python-dotenv pydantic nltk vaderSentiment umap-learn hdbscan`
- [ ] `python -c "import nltk; nltk.download('vader_lexicon')"` — pre-download VADER
- [ ] Download Phi-3-mini Q4_K_M.gguf → `backend/models/phi3/`
- [ ] Download ggml-small.en.bin → `backend/models/whisper/`
- [ ] Download SADIE II HRTF dataset → `backend/hrtf/sadie2/`
- [ ] Create `backend/.env` with all model paths
- [ ] **VERIFY Phi-3:** record inference latency
- [ ] **VERIFY Whisper:** confirm <400ms on 5s WAV
- [ ] **VERIFY speechbrain:** confirm speaker IDs on 2-speaker WAV
- [ ] **VERIFY VADER:** `python -c "from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer; a=SentimentIntensityAnalyzer(); print(a.polarity_scores('This is great!'))"` — should print compound > 0.5
- [ ] **VERIFY UMAP:** `python -c "import umap, numpy as np; u=umap.UMAP(n_components=2); print(u.fit_transform(np.random.rand(20,384)).shape)"` — should print (20, 2)
- [ ] Record 2-minute test WAV with two speakers (include decision, action item, question, disagreement, emotional escalation moment)
- [ ] Report latency numbers to Khush

---

## PHASE 1 — Backend Foundation

### Khush (write code, push, don't run)

- [x] Draft `backend/config.py` — all constants: SAMPLE_RATE, CHUNK_SIZE, model paths, agent config, **UMAP config (n_neighbors=5, min_dist=0.1, reproject_every_n=5)**
- [x] Draft `backend/db/schema.py` — SQLite table definitions (sessions, utterances, edges, action_items, qa_history). **Add `affect_valence REAL, affect_arousal REAL, affect_label TEXT, umap_x REAL, umap_y REAL, cluster_id INTEGER` columns to utterances table.**
- [x] Draft `backend/db/session_utils.py` — aiosqlite CRUD stubs with correct function signatures
- [x] Draft `backend/api/session.py` — FastAPI router, Pydantic models
- [x] Draft `backend/api/graph.py` — stub returns `{"nodes": [], "edges": []}`
- [x] Draft `backend/api/qa.py` — stub returns `{"answer": "TODO", "citations": [], "past_sessions": []}`
- [x] Draft `backend/api/ws.py` — WebSocket route + connection registry
- [x] Define all WebSocket event envelope types as Pydantic models in `backend/models/events.py` — **include AffectData, GraphReprojectEvent, DominanceUpdateEvent, InterruptionEvent**
- [ ] Draft `backend/api/export.py` — `GET /session/{id}/export/snote` endpoint. Stub returns `{"snote": {}}`. **The .snote format is: `{ "title": str, "sections": [{ "type": "heading|text|list", "content": str|list }], "metadata": { "created_at", "source": "SpatialVoiceAI" } }`.** Teammate implements the real serializer.

### Teammate (implement + run)

- [ ] Pull Khush's stubs
- [ ] Implement `db/schema.py` — `init_db()` with aiosqlite, create all tables including new affect/umap columns
- [ ] Implement `db/session_utils.py` — all CRUD with real SQL
- [ ] Implement `main.py` — FastAPI app, lifespan (DB init + Phi-3 pre-warm + **global memory index load**), CORS, routers
- [ ] Implement `api/ws.py` — async WebSocket, connection registry, `broadcast()` helper
- [ ] Test all routes in Postman
- [ ] Report: "Phase 1 done, all routes green"

---

## PHASE 2 — AI Core

### Khush (write, don't run)

- [x] Draft `backend/audio/hrtf_renderer.py` — class skeleton with fftconvolve docstring
- [x] Draft `backend/llm/phi3_runner.py` — skeleton with JSON retry logic
- [x] **Write all Phi-3 agent prompts:**
    - [x] `backend/agents/prompts/event_detection.py` — 3-shot classification prompt
    - [x] `backend/agents/prompts/action_extraction.py` — owner/task/deadline extraction
    - [x] `backend/agents/prompts/qa_with_citations.py` — **UPDATED: include a `past_context` variable that injects top-3 cross-session utterances above the current-session context. Format: `[Past session {date}] {speaker}: {text}`. Phi-3 must cite these with session_id.**
- [x] Draft `backend/agents/coordinator.py` — AgentCoordinator skeleton
- [ ] Draft `backend/graph/graph_builder.py` — class skeleton, method signatures for `add_utterance()`, `get_similar()`, `to_dict()`, **`reproject_umap()`** (async, runs in threadpool)
- [ ] **Draft `backend/agents/affect_agent.py`** — AffectAgent class:
    
    ```python
    class AffectAgent:    def __init__(self): ...  # load VADER SentimentIntensityAnalyzer    def process(self, text: str, speaker_id: str) -> dict:        # Returns { "valence": float, "arousal": float, "label": str }        # valence: VADER compound score (-1 to +1)        # arousal: heuristic from len(text)/30, count('!','?'), ratio_uppercase        # label: map (valence, arousal) to affect_label enum        ...
    ```
    
    Write the full arousal heuristic logic. Teammate verifies and may tune thresholds.
- [ ] **Draft `backend/audio/interruption_detector.py`** — class skeleton:
    
    ```python
    class InterruptionDetector:    def check(self, frame_probs: dict[str, float]) -> Optional[InterruptionEvent]:        # frame_probs = {"SPK_0": 0.8, "SPK_1": 0.75} — both > threshold means interruption        # Returns InterruptionEvent or None        ...
    ```
    
- [ ] **Draft `backend/memory/global_memory.py`** — GlobalMemory class skeleton:
    
    ```python
    class GlobalMemory:    def __init__(self, index_path, meta_path): ...    def add_utterance(self, embedding, node_id, session_id, speaker_name, text): ...    def search(self, query_embedding, top_k=5) -> list[dict]: ...    def save(self): ...  # persist index + meta to disk    def load(self): ...  # load on startup
    ```
    

### Teammate (implement + verify everything)

- [ ] Implement `audio/hrtf_renderer.py` — SADIE II IRs, fftconvolve, verify <10ms
- [ ] Implement `audio/vad.py` — Silero VAD wrapper
- [ ] Implement `audio/interruption_detector.py` using Khush's skeleton
- [ ] Implement `asr/whisper_runner.py` — 3s accumulator, Whisper.cpp, return text+confidence+latency
- [ ] Implement `diarization/speaker_id.py` — speechbrain ECAPA, 5s sliding window
- [ ] Implement `llm/phi3_runner.py` using Khush's skeleton + prompts
- [ ] **Implement `agents/affect_agent.py`** — wire VADER, tune arousal thresholds against test WAV
- [ ] Implement `graph/graph_builder.py` — networkx DiGraph, MiniLM embed, FAISS IndexFlatIP, **UMAP reproject in threadpool every 5 utterances**
- [ ] Implement `graph/topic_mesh.py` — UMAP(n_components=2, n_neighbors=5, min_dist=0.1, init="spectral") + HDBSCAN(min_cluster_size=3). Returns `{ node_id: { x, y, cluster_id } }` dict.
- [ ] Implement `graph/graph_serializer.py` — SQLite ↔ networkx round-trip
- [ ] **Implement `memory/global_memory.py`** — FAISS IndexFlatIP 384-dim, disk persistence, merge at session end
- [ ] Implement all 5 agents using Khush's prompts + skeletons
- [ ] Implement `agents/coordinator.py`
- [ ] Wire real QA into `api/qa.py` — include cross-session retrieval
- [ ] **Implement `api/export.py`** — serialize session graph → Samsung Notes .snote JSON → return as file download
- [ ] **TEST each component independently** before wiring
- [ ] **RUN end-to-end on test WAV** — log ALL latencies including AffectAgent (<5ms) and UMAP reproject (<80ms)

---

## PHASE 3 — Audio Engine

### Khush (write, don't run)

- [ ] Draft `backend/audio/audio_engine.py` — AudioEngine thread class skeleton:
    - `__init__`, `start()`, `stop()` signatures
    - `AudioQueue` and `ASRQueue` with maxsize + drop-oldest
    - `audio_level` float property per speaker
    - **`interruption_detector` instance wired into the callback**
    - PyAudio callback signature with type hints
    - Inline threading model comments
- [ ] Draft `backend/nlp_engine.py` — NLPEngine thread skeleton. `asyncio.run_coroutine_threadsafe` WS bridge pattern.

### Teammate (implement + run)

- [ ] Implement AudioEngine — PyAudio callback, queue drain, VAD filter, interruption detection, HRTF render, playback
- [ ] Implement NLPEngine — reads ASRQueue, calls coordinator, bridges to asyncio WS broadcast
- [ ] Connect both to FastAPI lifespan startup
- [ ] **TEST audio only:** 10 minutes, no glitches, stable memory
- [ ] **TEST full pipeline:** VB-Cable → confirm transcript + affect + UMAP events in backend logs in real-time

---

## PHASE 4 — Frontend (Khush solo)

> Teammate: keep backend running and answer integration questions only.

### Khush owns all of this

**Store & Data layer**

- [ ] Build complete `lib/store.ts` — Zustand slices:
    - session, transcript (utterances[]), actionItems[], qaHistory[], audioLevels ({SPK_0, SPK_1})
    - **affectStates: Record<speakerId, AffectState>** — current affect label + history[]
    - **topicClusters: { nodes: NodePosition[], clusters: ClusterMeta[] }** — UMAP coords + cluster colors
    - **dominanceStats: { SPK_0: DominanceStat, SPK_1: DominanceStat }** — talk_pct + interruption_count
    - graphData (nodes[], edges[])
- [ ] Build complete `lib/api.ts` — `startSession()`, `endSession()`, `submitQuestion()`, `getGraph()`, **`exportSnote(sessionId)`**
- [ ] Build `hooks/useWebSocket.ts` — WS connect, parse ALL event types (including graph_reproject, dominance_update, interruption), Zustand dispatch, exponential backoff reconnect
- [ ] Build `hooks/useAudioLevel.ts` — Web Audio API AnalyserNode, RMS per frame, 30fps updates

**Pages**

- [ ] Build `app/page.tsx` — landing: session name input, start button, cinematic dark style
- [ ] Build `app/call/[id]/page.tsx` — two-panel layout (60/40), SpeakerModal on mount
- [ ] Build `app/call/[id]/ended/page.tsx` — KPI cards (talk time split, interruption count, action items, decisions), full transcript, export .snote button

**Components**

- [ ] Build `components/SpatialCanvas.tsx`:
    - Two orbs at az=330/30 HRTF positions
    - **Orb color temperature driven by affectStates: calm=blue, engaged=cyan, excited=amber, tense=orange, conflicted=red** — smooth color lerp using Three.js `lerp()`
    - **Secondary glow ring color = cluster color from UMAP assignment**
    - Audio level pulse (scale + emissive intensity)
    - Interruption: red flash pulse 500ms on interrupter orb
    - Dominant speaker orb drifts 8% toward center after 30s uninterrupted speech
    - Particle background
    - Speaker name labels floating above orbs
- [ ] Build `components/TranscriptRibbon.tsx` — utterance cards, speaker color stripe, EventFlag chips, slide-in animation, auto-scroll, **affect label badge on each card (small pill: color-coded)**
- [ ] Build `components/EventFlag.tsx` — 4 variants (amber diamond/teal circle/red triangle/blue tag)
- [ ] Build `components/ActionItemPanel.tsx` — collapsible, owner badge, task text, deadline hint
- [ ] Build `components/QADrawer.tsx` — slide-up panel, text input, loading dots, citation chips, **past-session citation chips dimmed with clock icon**, scroll-to-utterance
- [ ] Build `components/AffectArc.tsx` ← **NEW**:
    - Per-speaker emotional timeline sparkline
    - X-axis = time, Y-axis = valence (-1 to +1), colored line with area fill
    - Background gradient shifts with arousal level
    - Tiny label markers at inflection points (e.g. "tension spike")
    - Built with SVG + Framer Motion path animation
- [ ] Build `components/DominanceRing.tsx` ← **NEW**:
    - Animated split ring (SVG arc) showing talk time % per speaker
    - Speaker colors match orb colors
    - Live-updating, smooth arc transition
    - Below the ring: interruption count badges per speaker
- [ ] Build `components/GraphView.tsx`:
    - Force-directed graph (d3-force) for turn/semantic edges
    - **UMAP mode toggle: switch from force-directed to UMAP coordinates**
    - In UMAP mode: nodes animate to their projected x/y positions using Framer Motion spring
    - Node color = cluster color, node size = utterance importance (degree centrality)
    - Hover tooltip: speaker name, text snippet, affect label
- [ ] Build `components/SpeakerModal.tsx` — two name inputs, "Start Listening" button
- [ ] Connect all components to Zustand store
- [ ] **TEST with mock data:** inject all WS event types manually → verify UI updates, animations smooth
- [ ] **TEST with live backend:** confirm real events update all UI elements correctly

---

## PHASE 5 — Integration

### Khush

- [ ] Open browser on teammate's machine — confirm WS in DevTools
- [ ] Test full UI flow: start session → speak → watch orbs change color with affect → watch transcript appear → ask Q&A with cross-session citation → export .snote
- [ ] Test two-tab sync
- [ ] Test UMAP mode toggle in GraphView — nodes should animate to clusters
- [ ] Report UI bugs and fix them

### Teammate

- [ ] Run full backend: uvicorn + AudioEngine + NLPEngine
- [ ] Route test audio through VB-Cable
- [ ] Monitor memory 10+ minutes — no leak
- [ ] Confirm ALL WS event types reach frontend: utterance, event_flag, action_item, affect (inline with utterance), graph_reproject, dominance_update, interruption
- [ ] Log real latencies for all pipeline components
- [ ] Fix top backend bugs

---

## PHASE 6 — Polish

### Khush (frontend polish)

- [ ] Affect-to-color lerp tuning — verify transitions look natural, not jarring
- [ ] UMAP cluster color palette — 6 distinct colors that work on dark background
- [ ] Transcript card word-by-word reveal animation (80ms/word fake streaming)
- [ ] Latency badge on companion panel (green/amber/red by ms threshold)
- [ ] Cmd/Ctrl+K shortcut → open QADrawer
- [ ] Export button → `GET /session/{id}/export/snote` → download .snote file
- [ ] Mobile responsive check
- [ ] Final visual pass: spacing, typography, dark theme consistency
- [ ] **Demo script UI polish:** ensure the 5 planned demo Q&A questions all produce clean citation chips

### Teammate (backend polish)

- [ ] Whisper hallucination deduplication (identical consecutive output → discard)
- [ ] WebSocket rate limiting (max 2 msg/sec per session)
- [ ] **Tune AffectAgent arousal thresholds** against real test audio — verify emotional arc sparkline looks meaningful
- [ ] **Stress test UMAP reproject:** 100 utterances, confirm <80ms and no main thread blocking
- [ ] **Build demo/seed.db AND demo/seed_global.index** — run full pipeline on scripted demo audio including a PAST SESSION about deployment/budget (so cross-session QA demo works)
- [ ] Verify Q&A works on seed data for all 5 planned demo questions (at least 2 should have cross-session hits)
- [ ] `GET /session/{id}/stats` → dominant_speaker, interruption_count, avg_latency, topic_clusters, affect_distribution
- [ ] Verify .snote export opens correctly in Samsung Notes on a real device

---

## PHASE 7 — Demo Prep

### Khush

- [ ] Write README.md — setup, architecture diagram, demo video link
- [ ] Record 60s demo video
- [ ] Final GitHub cleanup
- [ ] **Rehearse the "wow moments" in order:**
    1. Speak calmly → orbs are blue → raise voice/exclaim → orb shifts to amber in real time
    2. One speaker interrupts → red flash on their orb
    3. Ask Q&A mid-call → get answer with citation chips → click chip → transcript scrolls to utterance
    4. End call → ask "was deployment discussed before?" → cross-session hit appears with clock icon
    5. Hit export → open Samsung Notes on demo phone → structured conversation appears

### Teammate

- [ ] Record scripted demo audio: calm intro → decision → action item → emotional escalation → resolution
- [ ] Prepare backup pre-rendered binaural WAV
- [ ] Verify Galaxy Buds spatial positions clearly audible
- [ ] Test demo fallback: kill live pipeline → seed.db + seed_global.index → Q&A still works
- [ ] Rehearse backend explanation: "five agents, all on-device, no cloud, persistent memory"

---

## PHASE 8 — Blueprint (Khush solo, before May 13)

- [ ] Download blueprint template from ennovatex.io
- [ ] Use BLUEPRINT_PROMPT.md — generate all 8 slide texts
- [ ] Fill slides — **highlight the 5 novel contributions:**
    1. Real HRTF binaural spatial audio (SADIE II, not WebAudio panning)
    2. Affect-driven orb visualization (AffectAgent + color temperature)
    3. Live UMAP Topic Mesh (semantic clustering in real-time)
    4. Cross-session persistent memory (on-device, FAISS global index)
    5. Samsung Notes .snote export (native ecosystem closure)
- [ ] Architecture diagram: show all 5 agents, UMAP pipeline, global memory store
- [ ] Teammate reviews: model names, latency numbers, memory size claims
- [ ] Export PDF, send from institute email
- [ ] **Deadline: May 13, 2026**

---

## Summary: What Khush Can Always Do On His Laptop

|Task type|Can do on Pentium N3710?|Notes|
|---|---|---|
|Write any Python file|✅ Yes|VS Code + syntax check, no execution|
|Write TypeScript/React/Next.js|✅ Yes|Node runs fine|
|Run `npm run dev` for frontend|✅ Yes|Works without backend|
|Design Phi-3 prompts|✅ Yes|Just text files|
|Draft AffectAgent logic|✅ Yes|Pure text/math, no execution needed|
|Draft GlobalMemory skeleton|✅ Yes|Class structure only|
|Draft InterruptionDetector skeleton|✅ Yes|No runtime needed|
|Git push/pull|✅ Yes||
|Write Blueprint|✅ Yes||
|Excalidraw/draw.io architecture diagram|✅ Yes|Browser-based|
|Run FastAPI with stub routes only|✅ Yes||
|Test frontend with mock WS data|✅ Yes|Use a WS mock in browser|
|Run Phi-3, Whisper, speechbrain|❌ No|AVX2 missing, 4GB RAM|
|Run UMAP + HDBSCAN on embeddings|❌ No|RAM exhaustion on large tensors|
|Run FAISS global index search|❌ No|RAM + compute|
|Run PyAudio pipeline|❌ No|Will freeze the machine|
