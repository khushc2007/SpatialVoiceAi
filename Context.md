# SpatialVoiceAI — Master Context File (v2, 9.9/10 Build)

> Paste this entire file at the start of every Claude session.
> Last updated: May 2026 | Status: Blueprint ready, build phase open

---

## What We Are Building

SpatialVoiceAI is a production-grade, real-time AI system for the Samsung ennovateX AX Hackathon 2026 (Problem Statement 08: "Immersive Spatial Voice Call Experience with AI").

It intercepts a voice call's audio stream and does three things simultaneously:

1. **Spatial rendering** — each speaker is rendered at a distinct 3D binaural position using real HRTF convolution (SADIE II dataset, not Web Audio API panning). The rendered audio is playable on any headphone; optimized for Galaxy Buds Pro spatial audio profile.

2. **Semantic conversation graph** — a live directed graph where nodes are utterances (speaker, text, timestamp, embedding, event flags) and edges encode semantic similarity, topic continuity, and speaker turn transitions. This graph is the memory of the call.

3. **Multi-agent AI companion** — four specialized agents run against the graph in parallel: Transcription Agent, Event Detection Agent, Action Item Agent, and Q&A Agent. These are orchestrated by a thin Coordinator using tool-chaining (not a monolithic LLM call), producing a live companion UI with event flags, action items, citations, and graph-grounded Q&A.

**The key differentiator:** This is not "record → transcribe → summarize." The graph is live, queryable mid-call, and grounded — every answer cites the exact utterance node it came from. The spatial audio and the AI companion are unified: the orb that represents a speaker in 3D space is the same node in the graph.

---

## Team

| Name | Role |
|---|---|
| Khush | Frontend lead, voice integration, system architecture |
| Teammate | Backend Python, model integration, audio pipeline |

**Machine:** Teammate's laptop — 8GB+ RAM, modern CPU (i5/Ryzen 5, 8th gen+). This is the dev and demo machine.

---

## Stack (Final, Locked)

### Backend
| Component           | Choice                                | License         | Why                                      |
| ------------------- | ------------------------------------- | --------------- | ---------------------------------------- |
| Runtime             | Python 3.11                           | —               | stable, best lib support                 |
| API framework       | FastAPI + uvicorn                     | MIT             | native async WS                          |
| Audio capture       | PyAudio (callback mode)               | MIT             | non-blocking, queue-based                |
| ASR                 | Whisper.cpp ggml-small.en (244MB)     | MIT             | better WER than tiny, still <400ms on i5 |
| LLM                 | Phi-3-mini-4k-instruct Q4_K_M (2.3GB) | MIT             | fits 8GB RAM, fast enough                |
| Embeddings          | all-MiniLM-L6-v2 (80MB)               | Apache 2.0      | 384-dim, fast                            |
| Speaker diarization | speechbrain/spkrec-ecapa-voxceleb     | Apache 2.0      | lighter than pyannote, clean license     |
| VAD                 | Silero VAD                            | MIT             | accurate, low CPU                        |
| HRTF dataset        | SADIE II Subject 002 (~200MB)         | Research (York) | smallest, most compatible                |
| Vector search       | FAISS IndexFlatIP                     | MIT             | no training, fast under 1000 nodes       |
| Graph               | networkx DiGraph                      | BSD-3           | in-memory, sufficient                    |
| Database            | SQLite via aiosqlite                  | Public domain   | async-safe, no Postgres needed           |
| Agent orchestration | Custom tool-chaining (no framework)   | —               | deterministic, no LangChain overhead     |

### Frontend
| Component | Choice |
|---|---|
| Framework | Next.js 15 (App Router) |
| 3D | Three.js + @react-three/fiber + @react-three/drei |
| State | Zustand (slices: session, graph, transcript, actionItems, qaHistory, audioLevels) |
| Animation | Framer Motion + custom CSS |
| Styling | Tailwind CSS + CSS custom properties |
| Fonts | Orbitron (display) + JetBrains Mono (data) + Inter (body) |
| Theme | Cinematic dark — deep navy/black base, teal/cyan accents, glassmorphic panels |

### Audio Routing
- Windows: VB-Cable virtual audio cable
- macOS: BlackHole 2ch

---

## Architecture — Critical Decisions

### Threading model (non-negotiable)
```
PyAudio callback (audio thread)
    → AudioQueue (maxsize=50, drops oldest on overflow)
        → AudioEngine thread
            → VAD filter → HRTF renderer → playback
            → ASRQueue (maxsize=20, drops oldest)
                → NLPEngine thread
                    → Whisper ASR (3s chunks)
                    → Diarization (5s sliding window)
                    → GraphBuilder (add node, embed, FAISS index)
                    → AgentCoordinator (tool-chain dispatch)
                        → EventDetectionAgent
                        → ActionItemAgent
                    → asyncio bridge → WebSocket broadcast
```

**Rule: NLP never touches the PyAudio callback thread. Ever.**

### Multi-agent architecture (the 9.9/10 differentiator)

Instead of one monolithic Phi-3 prompt doing everything, four specialized agents run via tool-chaining:

1. **TranscriptionAgent** — wraps Whisper output, adds speaker ID from diarization, writes utterance node to graph. Fires on every 3s chunk.

2. **EventDetectionAgent** — receives utterance text + last 5 graph nodes as context. Phi-3 with a 3-shot prompt classifies: `decision | action_item | question | disagreement | none`. Returns structured JSON. Fires on every utterance.

3. **ActionItemAgent** — fires only when EventDetectionAgent returns `action_item`. Extracts: owner (speaker name), task text, implicit deadline if mentioned. Writes to SQLite `action_items` table.

4. **QAAgent** — fires on demand (user submits question). FAISS retrieves top-5 relevant utterance nodes → Phi-3 generates answer with citation node IDs → frontend highlights cited utterances.

**AgentCoordinator** is a thin Python class that routes events between agents using a tool-call registry pattern. No LangChain. No AutoGen. Pure Python, deterministic, debuggable.

### Graph data model
```python
Node: {
  id: uuid,
  speaker_id: str,          # SPK_0 | SPK_1
  speaker_name: str,        # assigned by user on call start
  text: str,
  timestamp: float,
  embedding: np.ndarray,    # 384-dim MiniLM
  event_flags: list[str],   # ["action_item", "decision"]
  confidence: float,        # Whisper confidence
  audio_segment_path: str   # path to 3s WAV chunk
}

Edge: {
  source: uuid,
  target: uuid,
  weight: float,            # cosine similarity
  type: str                 # "semantic" | "turn" | "reference"
}
```

### Latency budget (verified targets for i5 8th gen, 8GB RAM)
| Component | Target | Notes |
|---|---|---|
| HRTF rendering | < 10ms | 960-sample chunk, scipy.signal.fftconvolve |
| Whisper small | < 400ms | 3s chunk, ggml-small.en |
| Diarization | < 200ms | speechbrain ECAPA, 5s window |
| EventDetection (Phi-3) | < 600ms | 3-shot prompt, JSON output |
| Q&A end-to-end | < 900ms | FAISS retrieval + Phi-3 generation |
| WebSocket broadcast | < 50ms | asyncio, local network |
| **Total pipeline latency** | **< 1.2s per utterance** | perceived as real-time |

---

## Hard Rules (never break)

1. **NLP never runs in the PyAudio audio callback.** Audio glitches are disqualifying in a demo.
2. **Model files never committed to Git.** `/backend/models/` and `/backend/hrtf/` are gitignored.
3. **Real HRTF convolution only.** Web Audio API stereo panning is not spatial audio. Samsung judges know the difference.
4. **Frontend does not start until Phase 2 AI core is verified.** A UI with fake data wastes integration time.
5. **Pre-warm Phi-3 on server startup.** One dummy inference. First real call must not pay the cold-start cost.
6. **Every Phi-3 prompt returns JSON.** Parse with `json.loads()`. If it fails, retry once with a stricter system prompt. Never display raw LLM output to the user.
7. **Every Q&A answer includes citation node IDs.** No uncited answers. The graph is the source of truth.
8. **AudioQueue and ASRQueue have maxsize + drop-oldest policy.** No unbounded queues in production.
9. **Pre-seeded demo session exists in SQLite by Day 15.** Live pipeline failure during demo must be survivable.
10. **speechbrain, not pyannote.** License is Apache 2.0 clean. No HuggingFace gated model agreements to worry about.

---

## Samsung Ecosystem Alignment

| Claim | Specifics |
|---|---|
| Galaxy Buds Pro | HRTF positions match Galaxy Buds Pro spatial audio azimuth range (330°/30°). The demo should be run on Galaxy Buds if available. |
| Galaxy AI | The on-device inference stack (Phi-3 Q4, Whisper small, MiniLM) is the same class of model that Samsung runs on-device via Galaxy AI. Position this as "Galaxy AI for voice calls, running entirely on-device." |
| Samsung Notes / Samsung Link | The conversation graph and action items can be exported directly to Samsung Notes format (JSON → .snote). This is a one-function implementation that judges will remember. |
| Bixby | The Q&A agent is architecturally compatible with a Bixby capsule — mention this in the blueprint without committing to implement it. |

---

## Repository Structure

```
spatialvoiceai/
├── backend/
│   ├── main.py                    # FastAPI entry, lifespan, router includes
│   ├── config.py                  # All constants: sample rate, paths, agent config
│   ├── audio/
│   │   ├── audio_engine.py        # AudioEngine thread, PyAudio callback, queues
│   │   ├── hrtf_renderer.py       # SADIE II loader, fftconvolve stereo renderer
│   │   └── vad.py                 # Silero VAD wrapper
│   ├── asr/
│   │   └── whisper_runner.py      # Whisper.cpp wrapper, chunk accumulator
│   ├── diarization/
│   │   └── speaker_id.py          # speechbrain ECAPA wrapper, sliding window
│   ├── llm/
│   │   └── phi3_runner.py         # llama-cpp-python wrapper, JSON parser, retry
│   ├── graph/
│   │   ├── graph_builder.py       # networkx DiGraph, MiniLM embed, FAISS index
│   │   └── graph_serializer.py    # SQLite ↔ networkx round-trip
│   ├── agents/
│   │   ├── coordinator.py         # AgentCoordinator, tool registry, event routing
│   │   ├── transcription_agent.py # Wraps Whisper + diarization, writes graph node
│   │   ├── event_agent.py         # EventDetectionAgent, Phi-3 classification
│   │   ├── action_agent.py        # ActionItemAgent, owner extraction, SQLite write
│   │   └── qa_agent.py            # QAAgent, FAISS retrieval, Phi-3 generation
│   ├── api/
│   │   ├── session.py             # POST /session/start, GET /session/{id}, POST /session/{id}/end
│   │   ├── graph.py               # GET /graph/{id}
│   │   ├── qa.py                  # POST /qa
│   │   └── ws.py                  # WS /ws/{session_id}
│   ├── db/
│   │   ├── schema.py              # SQLite table definitions, aiosqlite init
│   │   └── session_utils.py       # CRUD helpers for sessions, utterances, action_items
│   ├── models/                    # gitignored — downloaded model files
│   └── hrtf/                      # gitignored — SADIE II dataset
├── frontend/
│   ├── app/
│   │   ├── page.tsx               # Landing / session start
│   │   ├── call/[id]/page.tsx     # Live call view
│   │   └── call/[id]/ended/page.tsx # Post-call summary
│   ├── components/
│   │   ├── SpatialCanvas.tsx      # Three.js scene, speaker orbs
│   │   ├── TranscriptRibbon.tsx   # Live scrolling utterance cards
│   │   ├── EventFlag.tsx          # 4 variants: decision/action/question/disagreement
│   │   ├── QADrawer.tsx           # Slide-up Q&A panel with citation highlighting
│   │   ├── ActionItemPanel.tsx    # Live action item list, owner badges
│   │   ├── GraphView.tsx          # Optional: force-directed graph visualization
│   │   └── SpeakerModal.tsx       # Name assignment on call start
│   ├── hooks/
│   │   ├── useWebSocket.ts        # WS connect, parse events, dispatch to Zustand
│   │   └── useAudioLevel.ts       # Web Audio API analyser, feeds orb pulse
│   ├── lib/
│   │   ├── api.ts                 # Typed API client (session, qa, graph endpoints)
│   │   └── store.ts               # Zustand store — all slices
│   └── public/
├── audio_samples/                 # Pre-recorded test WAVs (gitignored if large)
├── demo/
│   └── seed.db                    # Pre-seeded SQLite session for demo fallback
├── docs/
│   ├── architecture.png           # System diagram for blueprint
│   └── README.md
├── .env
├── requirements.txt
└── package.json
```

---

## Ports & Endpoints

| Service | URL |
|---|---|
| FastAPI backend | `http://localhost:8000` |
| FastAPI docs | `http://localhost:8000/docs` |
| WebSocket | `ws://localhost:8000/ws/{session_id}` |
| Next.js frontend | `http://localhost:3000` |

---

## WebSocket Event Schema

All events follow this envelope:

```json
{
  "event": "utterance | event_flag | action_item | qa_response | session_end",
  "session_id": "uuid",
  "timestamp": 1234567890.123,
  "data": { ... }
}
```

Event-specific `data` shapes:
- `utterance`: `{ node_id, speaker_id, speaker_name, text, confidence, event_flags }`
- `event_flag`: `{ node_id, flag_type, confidence, evidence_text }`
- `action_item`: `{ id, owner_speaker, task_text, deadline_hint, source_node_id }`
- `qa_response`: `{ question, answer, citation_node_ids, latency_ms }`
- `session_end`: `{ total_utterances, action_item_count, graph_edge_count }`

---

## Model Download Status

| Model | Size | Path | Downloaded |
|---|---|---|---|
| Phi-3-mini-4k-instruct Q4_K_M.gguf | 2.3 GB | `backend/models/phi3/` | ⬜ |
| ggml-small.en.bin (Whisper) | 244 MB | `backend/models/whisper/` | ⬜ |
| all-MiniLM-L6-v2 | 80 MB | auto (sentence-transformers) | ⬜ |
| speechbrain ECAPA-TDNN | ~90 MB | auto (speechbrain) | ⬜ |
| SADIE II HRTF dataset | ~200 MB | `backend/hrtf/sadie2/` | ⬜ |
| Silero VAD | ~2 MB | auto (torch.hub) | ⬜ |

> Update ⬜ → ✅ as each downloads.

---

## Current Status

**Active Day:** Day [ ] of build  
**Active Phase:** Phase [ ] — [Name]  
**Last Session:** [DATE] by [NAME]

### Working right now
- [ ] Nothing yet

### Blocked right now
- [ ] Nothing blocked

### Deviations from plan
- None yet.

---

## How to Use This File

**Starting a session:**
> Paste this file and say: "You are working on SpatialVoiceAI. Read this context. Today's task: [describe]."

**Ending a session:**
> Update Current Status. Push updated CONTEXT.md to GitHub. Add 5-line entry to SESSION_LOG.md.

**For teammate:**
> Pull latest CONTEXT.md before starting. Push update after.
