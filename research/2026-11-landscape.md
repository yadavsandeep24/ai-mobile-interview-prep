# Mobile AI Landscape & Hiring Requirements (researched Nov 15, 2026)

This file is the basis for the curriculum. It should be refreshed every couple of months, because these APIs change quickly.

> **Status labels matter in interviews.** Saying "AppFunctions is in beta and the ML Kit Agentic API is in alpha" shows you actually follow the platform.

---

## 1. Android: the big shift

At I/O 2026, Google said Android is moving **"from an operating system to an intelligence system"**. Users state a goal, and agents such as Gemini complete multi-step tasks across apps. Your app either exposes its capabilities to those agents or gets driven through its UI.

### Key APIs and their status (as of Nov 2026)

| API | What it does | Status |
|---|---|---|
| **AppFunctions (Android MCP)** | Your app acts as an **on-device MCP server** and exposes tools and data to Gemini and other agents | **Beta** (announced at Android Dev Summit, Oct 28, 2026) |
| **Computer Control framework** | Agents operate your app's UI when the app has no integration | Platform fallback |
| **Connected Apps Platform** | Reach users through Gemini and Search using Android MCP + remote MCP | Developer preview / waitlist |
| **ML Kit GenAI Prompt API** | Direct access to Gemini Nano. Supports system instructions, thinking mode and **prefix caching** | Production path for Nano 4 |
| **ML Kit Structured Output** | The Prompt API returns typed objects | Upcoming |
| **ML Kit Agentic API** | A full **on-device agent loop**: auto-executes functions, keeps history, includes guardrails | **Alpha** |
| **ML Kit Embeddings API** | On-device embeddings (EmbeddingGemma 2), which enable on-device RAG | Alpha |
| **ML Kit speech** (recognition, translation, TTS) | On-device speech features | Alpha |
| **Gemini Nano 4** | Built on Gemma 4. Native function calling, up to ~3x faster and up to ~4x less memory than the previous Nano (Google's internal benchmarks) | Flagship devices; prototype it through the AICore Developer Preview |
| **LiteRT in Google Play services** | Runs custom models with faster GPU performance. The runtime is removed from your APK (~8 MB smaller) | Available, now also on Wear OS and Automotive |
| **LiteRT-LM** | Bring your own fine-tuned small LLM | Available |
| **Firebase AI Logic hybrid inference** | Built-in routing between on-device and cloud: `PREFER_ON_DEVICE`, `PREFER_IN_CLOUD`, `ONLY_ON_DEVICE`, `ONLY_IN_CLOUD` | Experimental. On-device mode supports single-turn text (+1 image) only |
| **Firebase production guards** | App Check, server prompt templates, spend caps | Available |
| **A2UI Compose renderer** | Agents stream JSON that renders as **native Compose UI** | **Alpha** |
| **ADK for Android** | Multi-agent workflows across on-device and cloud models | Experimental |

**When to use Android MCP and when to use remote MCP** (Google's guidance): use Android MCP for end-to-end encrypted journeys, sensitive local data, device sensors and hardware, local-only execution, or when there's no server. Use remote MCP for cross-platform reach. You can use both.

### Developer tooling
- **Android Studio Rabbit 2 (stable): Bring Your Own Agent.** Claude Agent, Codex, Antigravity, or any of 40+ ACP-compliant agents can run inside Studio.
- **Android CLI** for agents: device streaming, Compose previews without an emulator, Perfetto profiling, and bundle analysis.
- **30+ official Android skills**, plus official agent plugins for Claude Code, Codex and Antigravity.

### Consumer context (useful for "why does this matter" answers)
- Gemini Intelligence expanded to more devices, including mid-range phones, on Nov 10, 2026. It adds multi-step task automation across apps (now in the UK, India and Japan) and "Create My Widget".

---

## 2. iOS: what to know even as an Android developer

- **Foundation Models framework** (iOS 26) gives Swift access to Apple's ~3B on-device model, with guided generation (`@Generable`) and a `Tool` protocol. It's free, works offline and keeps data private.
- **WWDC 2026 / iOS 27** (June 8, 2026; generally available this fall) adds **image input**, **server models via Private Cloud Compute** (reportedly free under 2M users), custom skills, Dynamic Profiles, and next-generation Apple Foundation Models **built in collaboration with Google Gemini**.
- **Core AI framework** runs your own models (replacing the older Core ML approach). The **Evaluations framework** tests AI features.
- **Xcode 27** adds agentic coding.

**How the platforms map:** Foundation Models ≈ ML Kit Prompt API · Tool protocol ≈ function calling / AppFunctions · Private Cloud Compute ≈ Firebase AI Logic cloud fallback · Core AI ≈ LiteRT.

---

## 3. What companies are asking for

Common requirements in 2026 Android job postings that mention AI:

1. **Strong Android fundamentals first.** Kotlin, Compose, Coroutines/Flow, Room/DataStore and WorkManager, typically 5+ years of experience. AI skills are added on top of these, not instead of them.
2. **LLM API integration.** Context management, **response validation**, error handling, and **streaming**.
3. **MCP and agents.** MCP tool servers, token management, multi-agent orchestration, Claude/OpenAI/Gemini APIs.
4. **On-device ML.** ML Kit and TFLite/LiteRT. Postings differ in how much they stress on-device versus cloud.
5. **Mobile-specific LLM concerns.** API key security, prompt reliability, **token cost**, battery, offline use and latency.
6. **Daily AI tool use.** Interviewers ask "How do you use AI in your daily workflow?" (for example, at CI&T).

### What interviews look like
- **AI-assisted coding rounds.** Meta reportedly runs a round where you solve problems *with* AI tools. Practice narrating how you prompt, verify and correct the AI.
- **System design questions with AI in them:**
  - "Design a real-time conversational AI app that streams tokens."
  - "A voice assistant gets interrupted: stop playback and discard stale responses."
  - "Concurrent requests and ordering in a chat UI."
  - "Keyboard next-word prediction that learns without collecting what users type" (on-device / federated learning).
  - "Trade-offs of on-device vs server inference for an assistant."
- **Classic mobile topics still come up:** offline-first sync, secure key storage, and coroutine/lifecycle correctness.

*Note: some hiring and interview sources are prep-vendor sites built from candidate reports, so treat them as signals rather than guarantees.*

---

## Sources
- [Top AI on Android updates from Google I/O '26 (Android Developers Blog)](https://developer.android.com/blog/posts/top-ai-on-android-updates-for-building-intelligent-experiences-from-google-i-o-26)
- [What's new in intelligent apps at the Android Developer Summit (Nov 5, 2026)](https://developer.android.com/blog/posts/android-intelligence-system-in-app-agents-ads)
- [Agentic and Flexible: A next era of Android development (Oct 28, 2026)](https://developer.android.com/blog/posts/build-it-your-way-a-next-era-of-android-development)
- [Hybrid inference (developer.android.com)](https://developer.android.com/ai/hybrid) · [Firebase AI Logic hybrid get-started](https://firebase.google.com/docs/ai-logic/hybrid/android/get-started)
- [Hybrid inference and new AI models are coming to Android (Apr 2026)](https://android-developers.googleblog.com/2026/04/Hybrid-inference-and-new-AI-models-are-coming-to-Android.html)
- [I/O 2026 session: ML Kit GenAI and LiteRT-LM](https://io.google/2026/explore/technical-session-22)
- [On-device speech in Google AI Edge Gallery](https://developers.googleblog.com/on-device-speech-recognition-and-text-to-speech-in-google-ai-edge-gallery/)
- [Gemini Intelligence November update](https://ai-tldr.dev/releases/google-gemini-intelligence-november-2026/) · [Gemini Intelligence expands to more devices](https://emergent.sh/news/gemini-intelligence-expands-more-devices-new)
- [Apple: Foundation Models framework](https://www.apple.com/newsroom/2025/09/apples-foundation-models-framework-unlocks-new-intelligent-app-experiences/) · [Apple ML Research: foundation model updates](https://machinelearning.apple.com/research/apple-foundation-models-2025-updates)
- [WWDC26: Foundation Models gain Gemini support, Xcode 27 agentic coding (FoneArena)](https://www.fonearena.com/blog/484623/apple-wwdc26-foundation-models-gemini-support-xcode-27-agentic-coding-tools.html) · [MacTech WWDC26](https://www.mactech.com/2026/06/08/apple-accelerates-app-development-with-new-intelligence-frameworks-and-advanced-tools/) · [Apple WWDC26 iOS guide](https://developer.apple.com/wwdc26/guides/ios/)
- Job postings: [Synechron Android Engineer](https://synechron.wd1.myworkdayjobs.com/en-US/SynechronCareers/job/Job-Title--Android-Engineer_JR1044604-1) · [GSSTech Sr. Android Engineer – AI/LLM](https://freehire.me/jobs/sr-android-engineer-ai-llm-gsstech-group-5ocxr6tm) · [Indeed: Android ML jobs](https://www.indeed.com/q-Android-Platform-Developer-Machine-Learning-jobs.html) · [Hiring guide: Android devs with AI skills](https://ahex.co/hire-android-app-developer-ai-skills/) · [How to hire an AI developer for mobile (2026)](https://hireflutterdev.com/blog/hire-ai-developer-mobile-app/)
- Interview signals: [Meta mobile engineer guide](https://dataford.io/interview-guides/meta-platforms/mobile-engineer) · [Anthropic mobile engineer guide](https://dataford.io/interview-guides/anthropic/mobile-engineer) · [CI&T guide](https://dataford.io/interview-guides/ciandt/mobile-engineer) · [Telus Digital guide](https://dataford.io/interview-guides/telus-digital-ai-data-solutions/mobile-engineer) · [Android AI Interview Handbook](https://androidengineers.substack.com/p/introducing-the-android-ai-interview) · [MAANG AI engineer questions](https://aiengineeringinsider.substack.com/p/105-maang-ai-engineer-interview-questions)
