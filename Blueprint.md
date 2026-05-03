# SpatialVoiceAI — Blueprint Generation Prompt

Paste this into Claude to generate content for each blueprint slide.

---

You are a senior AI systems architect and technical writer preparing a Phase 1 solution blueprint for the Samsung ennovateX AX Hackathon 2026, Problem Statement 08: "Immersive Spatial Voice Call Experience with AI."

The solution is SpatialVoiceAI. Write the content for each slide below. Be technically precise. Use real model names, real latency numbers, real license names. No vague language. No "AI-powered" without specifics. Write like a senior engineer presenting to a Samsung R&D panel — concise, confident, specific.

## Slide 1: Problem Statement
Write 1 paragraph (4-5 sentences) restating the problem in your own words. Cover: why voice calls are broken today (ephemeral, no spatial presence, no structured memory), what users lose (accountability, context, spatial immersion), and what an ideal solution looks like.

## Slide 2: Solution Overview
Write 3 bullet points, each 1-2 sentences, covering the three core capabilities:
1. Real HRTF spatial rendering
2. Live semantic conversation graph
3. Multi-agent AI companion

## Slide 3: Five Differentiators
For each, write a 2-sentence explanation: what it is, and why it matters to Samsung judges.
1. Real HRTF convolution (not Web Audio API panning)
2. Live queryable conversation graph (mid-call, not post-call)
3. Multi-agent tool-chaining with 4 specialized agents
4. Graph-grounded Q&A with utterance-level citations
5. Samsung ecosystem integration (Galaxy Buds spatial profile, Samsung Notes export)

## Slide 4: Technology Stack
Write a clean table: Component | Model/Library | Size | License
Rows: LLM, ASR, Embeddings, Diarization, VAD, HRTF Dataset, Vector Search, Graph, Backend, Frontend

## Slide 5: Latency Budget
Write a clean table: Component | Target Latency | Notes
Rows: HRTF rendering, Whisper ASR, Speaker diarization, Event detection (Phi-3), Q&A end-to-end, WebSocket broadcast, Total pipeline

## Slide 6: Evaluation Metrics
Write 3 metrics with definitions:
1. Localization accuracy — how measured
2. ASR Word Error Rate — test set description
3. Event detection precision/recall — labeling approach

## Slide 7: Samsung Ecosystem Alignment
Write 4 bullet points covering: Galaxy Buds Pro, Galaxy AI, Samsung Notes, Bixby. For each, 1 sentence on the specific technical connection.

## Slide 8: Open Source Compliance
Write a compliance statement and a list of all components with licenses. End with: "No gated models. No commercial APIs. All code released under Apache-2.0."
