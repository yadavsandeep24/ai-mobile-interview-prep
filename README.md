# AI in Mobile Dev: 30-Minute Daily Interview Prep

A 30-day curriculum for interviews on building AI features into mobile apps, covering **Android, iOS, Flutter and React Native**.

📚 **Start here:** [Mobile AI landscape & hiring requirements (Nov 2026)](research/2026-11-landscape.md). It includes a cross-platform concept map you can use in interviews.

## Daily format (30 min)

| Block | Time |
|---|---|
| Concept (platform-agnostic) | 10 min |
| Code, in **your** platform's track (🤖 Android · 🍎 iOS · 🐦 Flutter · ⚛️ React Native) | 10 min |
| Mini system design | 5 min |
| Mock interview questions (answer out loud) | 5 min |

Each lesson has code tracks for all four platforms. Do your main platform every day and skim the others: interviewers like candidates who can say "on iOS the equivalent is…".

## Roadmap

Revised on Nov 15, 2026 from [current platform news and job requirements](research/2026-11-landscape.md).

### Week 1: Foundations & the platform stacks
- [x] [Day 1: On-device vs cloud AI (and hybrid inference)](days/day-01-on-device-vs-cloud.md)
- [x] [Day 2: The 2026 landscape across all four platforms (concept map)](days/day-02-landscape-2026.md)
- [ ] Day 3: 🤖 Android on-device: ML Kit, Gemini Nano 4 Prompt API, LiteRT
- [ ] Day 4: 🍎 iOS on-device: Foundation Models (`@Generable`, `Tool`), Core AI, Private Cloud Compute
- [ ] Day 5: 🐦 Flutter AI stack: `firebase_ai`, AI Toolkit, flutter_gemma, platform channels
- [ ] Day 6: ⚛️ React Native AI stack: Vercel AI SDK, React Native AI, ExecuTorch, llama.rn, Expo dev builds
- [ ] Day 7: Review + mock: "native vs cross-platform for an AI-heavy app"

### Week 2: Cloud LLMs & GenAI UX
- [ ] Day 8: Cloud LLM APIs on every platform (Firebase AI Logic, Vercel AI SDK, FM cloud providers)
- [ ] Day 9: Streaming tokens: `Flow` / `AsyncSequence` / `Stream` / async iterators (cancellation, stale responses)
- [ ] Day 10: Prompt engineering for apps & server prompt templates
- [ ] Day 11: Structured output: ML Kit, `@Generable`, JSON schema, `generateObject` + zod
- [ ] Day 12: Function calling / tool use across platforms
- [ ] Day 13: Multimodal & on-device speech (image input, ASR, TTS, translation)
- [ ] Day 14: Review + mock

### Week 3: Agents & system integration
- [ ] Day 15: MCP fundamentals: tools, resources, remote vs local
- [ ] Day 16: 🤖 AppFunctions (Android MCP): expose your app to Gemini
- [ ] Day 17: 🍎 App Intents & Siri: intent/entity schemas, View Annotations
- [ ] Day 18: On-device agents: ML Kit Agentic API, FM Dynamic Profiles, ADK, AI SDK agents
- [ ] Day 19: Generative UI: A2UI Compose, Flutter GenUI, RN streaming UI
- [ ] Day 20: On-device RAG: embeddings + local vector search on each platform
- [ ] Day 21: Review + mock

### Week 4: Production & interviews
- [ ] Day 22: Security: API keys, App Check, prompt injection, spend caps
- [ ] Day 23: Cost, latency, caching, model downloads & app size
- [ ] Day 24: Safety & evaluation: guardrails, hallucinations, Apple's Evaluations framework
- [ ] Day 25: Performance profiling: Perfetto, Instruments, Flutter DevTools, RN profiling
- [ ] Day 26: AI coding tools & the "AI-assisted coding" interview round
- [ ] Day 27: System design: streaming conversational AI app (cross-platform)
- [ ] Day 28: System design: privacy-preserving smart keyboard / photo assistant
- [ ] Day 29: Behavioral: "Tell me about an AI feature you shipped"
- [ ] Day 30: Full mock interview
