# Mobile AI Landscape & Hiring Requirements: Android · iOS · Flutter · React Native (researched Nov 15, 2026)

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

## 2. iOS (iOS 27, WWDC June 8, 2026; generally available this fall)

Verified against [Apple's "What's new in iOS 27"](https://developer.apple.com/ios/whats-new/).

| API | What it does |
|---|---|
| **Foundation Models framework** | A Swift API for Apple's on-device model (~3B parameters, introduced in iOS 26). Core types are `SystemLanguageModel` (check `.availability`), `LanguageModelSession`, `@Generable` structs (guided/structured generation) and the `Tool` protocol (tool calling). Free and works offline |
| **iOS 27: any model** | The framework now works with *"any language model, including Apple Foundation Models, cloud models like Claude and Gemini, or any provider that conforms to the Language Model protocol"*. One API covers on-device and cloud |
| **iOS 27: multimodal** | Pass images with text. Built-in vision tools (OCR, barcode) the model can call on-device |
| **iOS 27: Dynamic Profiles** | Swap models, tools and instructions within one session |
| **iOS 27: Private Cloud Compute** | Next-generation Apple Foundation Models (built with Google Gemini) in the cloud, **at no API cost** for Small Business Program apps with under 2M first-time downloads |
| **Core AI** (new) | Load, specialize and run **your own models** on-device. Memory-safe Swift, AOT compilation, zero-copy paths. Converts PyTorch models. It is the iOS counterpart to LiteRT |
| **Evaluations framework** (new) | Tests AI behavior "beyond what unit tests can catch". Supports prompt hill-climbing and agentic app evaluations |
| **App Intents / Siri** | **Intent schemas** (Siri acts on your content with no phrases to define), **entity schemas** (feed the Spotlight semantic index), **View Annotations API** (Siri understands what's on screen), and an **App Intents Testing framework** |
| **Tooling** | `fm` CLI + Python SDK for prototyping prompts. Instruments support for agentic apps |

**Interview angle:** App Intents is Apple's equivalent of Android's AppFunctions. Both let the *system agent* drive your app.

---

## 3. Flutter

| Tool | What it does | Maturity |
|---|---|---|
| **Firebase AI Logic** (`firebase_ai`) | The recommended way to call cloud Gemini from Dart. Proxied, with App Check | Stable. The old `google_generative_ai` Dart SDK is **no longer developed** |
| **Flutter AI Toolkit** | Drop-in chat widgets (`LlmChatView`) with streaming, voice, attachments and function calling, over a swappable `LlmProvider` | 1.0 (uses `firebase_ai` 3.x) |
| **GenUI SDK** (`genui`, `genui_firebase_ai`) | The model composes **real Flutter widgets**. Flutter's counterpart to A2UI | Alpha / experimental |
| **flutter_gemma** (community) | Runs Gemma and other LLMs on-device (Android, iOS, web, desktop). Supports vision, audio, function calling, embeddings and RAG | Community-maintained |
| **Dart & Flutter MCP server** | Lets AI coding agents understand and run your Flutter project | Available |
| **Platform channels** | Reach Gemini Nano (ML Kit) or Apple Foundation Models through native code | DIY |

**Typical 2026 stack:** Firebase AI Logic for cloud, flutter_gemma for cross-platform on-device inference, the AI Toolkit for chat UI, and GenUI only for prototypes. **Watch out:** Firebase ML custom model hosting is deprecated and shuts down **June 15, 2027**.

**Interview angle:** "Firebase's built-in hybrid inference is Android-native. How do you get identical on-device behavior on iOS and Android in Flutter?" Answer: flutter_gemma with the same model on both, *or* platform channels to each OS model, accepting that the two OS models behave differently.

---

## 4. React Native

| Tool | What it does | Notes |
|---|---|---|
| **Vercel AI SDK** (`ai`) | A unified `generateText` / `streamText` / tool API across providers | The standard abstraction in TypeScript |
| **React Native AI** (Callstack, `@react-native-ai/*`) | AI SDK providers for on-device runtimes: `apple()` (Foundation Models, iOS 26+) and llama.rn (GGUF) | Lets you switch on-device and cloud by changing one line |
| **React Native ExecuTorch** (Software Mansion) | Runs PyTorch-exported `.pte` models (Llama 3.2, Qwen 3, SmolLM, Whisper) through a `useLLM` hook | RN 0.83+ / Expo SDK 55+, iOS 17+, Android minSdk 26 |
| **llama.rn** | llama.cpp bindings for GGUF models | Community |
| **Expo** | On-device AI needs a **development build**, because Expo Go can't load the native code. An `expo-ai` package exposing the OS models (Apple FM / Gemini Nano) has been proposed | Check its status |

**Pitfalls reported in late 2026** (from community sources, so verify): React Native AI providers implement AI SDK **v6**, so pin `ai@6` rather than `ai@latest` (7.x).

**Interview angle:** model files are large, so ship them as on-demand downloads rather than in the bundle. Run inference off the JS thread (the native module handles this), and stream tokens to the UI without re-rendering the whole list.

---

## 5. Cross-platform concept map (use this in answers)

| Concept | Android | iOS | Flutter | React Native |
|---|---|---|---|---|
| OS on-device LLM | Gemini Nano (ML Kit Prompt API) | Foundation Models | Platform channel / flutter_gemma | `@react-native-ai/apple`, ExecuTorch |
| Custom model runtime | LiteRT / LiteRT-LM | Core AI | flutter_gemma, TFLite plugins | ExecuTorch, llama.rn |
| Cloud LLM | Firebase AI Logic | Firebase AI Logic / FM with cloud provider / PCC | `firebase_ai` | Vercel AI SDK |
| Hybrid routing | Firebase hybrid modes | Foundation Models Language Model protocol + Dynamic Profiles | Custom router | AI SDK provider swap |
| Structured output | ML Kit Structured Output | `@Generable` | JSON schema | `generateObject` + zod |
| System agent integration | AppFunctions (Android MCP) | App Intents (intent/entity schemas) | Native code per OS | Native modules per OS |
| Generative UI | A2UI Compose renderer | — (SwiftUI from structured output) | GenUI SDK | AI SDK streaming UI |
| Streaming primitive | `Flow` | `AsyncSequence` | `Stream` | async iterators |
| Evaluation | Custom / Firebase | Evaluations framework | Custom | Custom |

---

## 6. What companies are asking for

Common requirements in 2026 Android job postings that mention AI:

1. **Strong Android fundamentals first.** Kotlin, Compose, Coroutines/Flow, Room/DataStore and WorkManager, typically 5+ years of experience. AI skills are added on top of these, not instead of them.
2. **LLM API integration.** Context management, **response validation**, error handling, and **streaming**.
3. **MCP and agents.** MCP tool servers, token management, multi-agent orchestration, Claude/OpenAI/Gemini APIs.
4. **On-device ML.** ML Kit and TFLite/LiteRT. Postings differ in how much they stress on-device versus cloud.
5. **Mobile-specific LLM concerns.** API key security, prompt reliability, **token cost**, battery, offline use and latency.
6. **Daily AI tool use.** Interviewers ask "How do you use AI in your daily workflow?" (for example, at CI&T).

**Flutter and React Native postings:**
- AI-assisted coding tools (Claude, Copilot and similar) are expected almost everywhere. One analysis estimates only about 15% of mobile postings require *building* AI features, but nearly all expect you to *use* AI tools.
- Roles that do require AI ask for **shipped** LLM features: RAG pipelines, OpenAI/Anthropic API integration, local LLMs, agents, and speech/voice APIs. "Real product work, not personal experiments."
- Cross-platform roles often pair React Native or Flutter with React/Next.js on the web, and Jest, Playwright or Flutter Test for testing.

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

**iOS:** [Apple: What's new in iOS 27](https://developer.apple.com/ios/whats-new/) · [iOS 27 Apple Intelligence framework decision guide](https://chatforest.com/builders-log/ios-27-apple-intelligence-framework-decision-guide-wwdc-2026/) · [Core AI vs Foundation Models vs MLX](https://andrew.ooo/answers/apple-core-ai-vs-foundation-models-vs-mlx-ios-27-framework-june-2026/) · [Foundation Models in iOS 27 opens to other models](https://www.o2ods.com/en/blog/foundation-models-in-ios-27-apple-opens-its-framework-to-other-ai-models/)

**Flutter:** [Flutter AI Toolkit docs](https://docs.flutter.dev/ai/ai-toolkit) · [Hybrid AI in Flutter: on-device + cloud routing](https://medium.com/@denisov.shureg/hybrid-ai-in-flutter-routing-between-on-device-and-cloud-models-954da8f25373) · [GenUI + Firebase AI in Flutter](https://flutterfever.com/genui-firebase-ai-flutter/) · [Flutter AI with Gemini and the Dart MCP server](https://ecorpit.com/flutter-ai-features-gemini-dart-mcp-2026/) · [Flutter trends 2026](https://www.monterail.com/blog/flutter-development-trends) · [Flutter Gems: LLM packages](https://fluttergems.dev/chatgpt-llm-genai/)

**React Native:** [Callstack React Native AI (GitHub)](https://github.com/callstackincubator/ai) · [react-native-ai.dev](https://www.react-native-ai.dev/) · [On-device AI in RN, late 2026: one AI SDK call, two engines](https://dev.to/praveensingh/on-device-ai-in-react-native-late-2026-one-ai-sdk-call-two-engines-measured-5455) · [Xybrid vs llama.rn vs RN ExecuTorch](https://www.xybrid.ai/blog/react-native-on-device-llm) · [On-device AI with Expo](https://dev.to/saidkaban/on-device-ai-in-react-native-with-expo-local-llm-speech-to-text-vision-fully-local-agents-57ld) · [WWDC26 and an Expo AI runtime](https://www.stewmore.dev/blog/announcing-expo-ai-runtime)

**Cross-platform hiring:** [AI-assisted dev tops mobile skill demands](https://interviewstack.io/blog/how-ai-is-changing-mobile-developer-2026) · [Akra Tech Flutter/RN posting](https://freehire.me/jobs/flutter-react-native-developer-akra-tech-vuqnzxdv) · [WORKSDELIGHT Android & Flutter posting](https://vibehackers.io/jobs/android-flutter-developer-worksdelight-4) · [Djinni listings](https://djinni.co/jobs/) · [Flutter vs RN 2026](https://www.bidev.dev/blog/flutter-vs-react-native-2026) · [Hiring guide: React Native developers](https://hiredevelopers.com/hire-react-native-developers/)

**Android & general:**
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
