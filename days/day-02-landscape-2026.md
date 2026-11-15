# Day 2: The 2026 Mobile AI Landscape (All Four Platforms)

Platforms: 🤖 Android · 🍎 iOS · 🐦 Flutter · ⚛️ React Native

**Goal for today:** be able to give a 2-minute answer to *"What's changed in mobile AI this year, and how would you build on it?"* that covers every platform and uses correct API names and maturity levels.

Full research and sources: [research/2026-11-landscape.md](../research/2026-11-landscape.md)

---

## 1. Concept (10 min)

### The big idea: the OS is becoming the agent
In 2026 both mobile platforms changed direction in the same way:

- 🤖 **Google:** Android is moving *"from an operating system to an intelligence system"* (I/O 2026). Users state a goal, and Gemini carries out multi-step tasks **across apps**.
- 🍎 **Apple:** iOS 27 lets Siri act on app content through **App Intents** (intent and entity schemas, View Annotations), and Foundation Models now works with **any model**, on-device or in the cloud.

**What this means for app developers:** your app now has two audiences, the **user** (your UI) and the **system agent** (your exposed tools and data). If you don't expose tools, agents fall back to operating your UI, which is slower and less reliable (Android calls this the *Computer Control framework*).

### The five layers of mobile AI
Think of every platform's AI stack in five layers. This framing works in any interview:

| # | Layer | Question it answers |
|---|---|---|
| 1 | **OS model** | Is there a free, built-in LLM on this device? |
| 2 | **Custom model runtime** | Can I run *my own* model on-device? |
| 3 | **Cloud LLM access** | How do I call frontier models safely? |
| 4 | **System agent integration** | How does the OS assistant use my app? |
| 5 | **Generative UI** | Can the model return *UI*, not only text? |

### The map you should memorize

| Layer | 🤖 Android | 🍎 iOS | 🐦 Flutter | ⚛️ React Native |
|---|---|---|---|---|
| **1. OS model** | Gemini Nano (4 coming) through the **ML Kit GenAI Prompt API** | **Foundation Models** (`SystemLanguageModel`) | Platform channels to each OS model | `@react-native-ai/apple`; Gemini Nano through a native module |
| **2. Custom runtime** | **LiteRT** (in Play services), **LiteRT-LM** | **Core AI** | **flutter_gemma** | **ExecuTorch**, **llama.rn** |
| **3. Cloud** | **Firebase AI Logic** (+ hybrid modes) | Firebase AI Logic; FM with a cloud provider; **Private Cloud Compute** | **`firebase_ai`** | **Vercel AI SDK** |
| **4. System agent** | **AppFunctions = Android MCP** (beta) | **App Intents** (intent/entity schemas) | Native code per OS | Native modules per OS |
| **5. Gen UI** | **A2UI Compose renderer** (alpha) | SwiftUI driven by `@Generable` output | **GenUI SDK** (alpha) | AI SDK streaming UI |

### Maturity levels (quoting these correctly makes you sound current)
| Stable / production-ready | Beta | Alpha / experimental |
|---|---|---|
| Foundation Models (iOS 26), ML Kit classic APIs, LiteRT, Firebase AI Logic (cloud), Flutter AI Toolkit 1.0, Vercel AI SDK | **AppFunctions**, ML Kit Prompt API (on specific devices) | **ML Kit Agentic API**, ML Kit Embeddings API, ML Kit speech APIs, **A2UI Compose**, **Flutter GenUI**, ADK for Android, Firebase hybrid inference |

iOS 27 features (multimodal Foundation Models, Dynamic Profiles, Core AI, Evaluations) shipped with iOS 27 this fall. Check Apple's docs for exact availability by device and region.

### Three trends to mention
1. **Hybrid is becoming built in.** Firebase hybrid modes (`PREFER_ON_DEVICE`…) on Android, and one Language Model protocol for on-device and cloud on iOS.
2. **MCP is the common language for tools.** Android MCP (AppFunctions) on the device, remote MCP servers for cross-platform reach, and Google's **Connected Apps Platform** combines both.
3. **On-device models are getting much better.** Gemini Nano 4 is up to ~3x faster with up to ~4x less memory (Google's internal benchmarks) and adds function calling. Apple's next models are built with Google Gemini.

---

## 2. Code (10 min): detect what AI a device can do

Every 2026 AI feature starts with **capability detection**. The OS model may be available, downloadable, still downloading, or missing. Production apps group devices into **tiers** and gate features by tier.

```
Tier A: OS model ready      → free, private, instant
Tier B: OS model downloadable / capable of running an open model → show "preparing…" or download on Wi-Fi
Tier C: cloud only          → requires network; show consent if data leaves the device
```

> These are interview-style sketches. Check exact class names against the current docs before you ship.

### 🤖 Android (Kotlin, ML Kit GenAI)

```kotlin
enum class AiTier { ON_DEVICE_READY, ON_DEVICE_PENDING, CLOUD_ONLY }

class AiCapabilities(private val model: GenerativeModel /* ML Kit GenAI client */) {

    suspend fun tier(): AiTier = when (model.checkStatus()) {
        FeatureStatus.AVAILABLE    -> AiTier.ON_DEVICE_READY
        FeatureStatus.DOWNLOADABLE,
        FeatureStatus.DOWNLOADING  -> AiTier.ON_DEVICE_PENDING
        else                       -> AiTier.CLOUD_ONLY   // UNAVAILABLE
    }

    /** Start the download early (for example, on app start over Wi-Fi), not when the user taps. */
    fun prefetch(scope: CoroutineScope) = scope.launch {
        if (model.checkStatus() == FeatureStatus.DOWNLOADABLE) {
            model.download().collect { /* report progress to the UI */ }
        }
    }
}
```

### 🍎 iOS (Swift, Foundation Models)

```swift
import FoundationModels

enum AITier { case onDeviceReady, onDevicePending, cloudOnly }

func currentTier() -> AITier {
    switch SystemLanguageModel.default.availability {
    case .available:
        return .onDeviceReady
    case .unavailable(.modelNotReady):
        return .onDevicePending            // downloading or preparing
    case .unavailable(.appleIntelligenceNotEnabled):
        return .cloudOnly                  // could also prompt the user to enable it in Settings
    case .unavailable(.deviceNotEligible):
        return .cloudOnly
    case .unavailable:
        return .cloudOnly
    }
}
```

### 🐦 Flutter (Dart + platform channel)

```dart
enum AiTier { onDeviceReady, onDevicePending, cloudOnly }

class AiCapabilities {
  static const _channel = MethodChannel('app/ai_capabilities');

  /// Native side: Kotlin calls ML Kit checkStatus(); Swift reads
  /// SystemLanguageModel.default.availability. Both return a string.
  Future<AiTier> tier() async {
    try {
      final status = await _channel.invokeMethod<String>('osModelStatus');
      return switch (status) {
        'ready'   => AiTier.onDeviceReady,
        'pending' => AiTier.onDevicePending,
        _         => AiTier.cloudOnly,
      };
    } on PlatformException {
      return AiTier.cloudOnly;
    }
  }
}
// Alternative: if you ship flutter_gemma, Tier B means "the device has enough RAM
// for the open model, and the model is downloaded".
```

### ⚛️ React Native (TypeScript hook)

```ts
import { useEffect, useState } from 'react';

export type AiTier = 'onDeviceReady' | 'onDevicePending' | 'cloudOnly';

/** checkOnDevice wraps your on-device library's availability check
 *  (React Native AI's Apple provider, or a llama.rn / ExecuTorch model-loaded check). */
export function useAiTier(checkOnDevice: () => Promise<'ready' | 'pending' | 'none'>) {
  const [tier, setTier] = useState<AiTier>('cloudOnly');

  useEffect(() => {
    let alive = true;
    checkOnDevice()
      .then(s => alive && setTier(s === 'ready' ? 'onDeviceReady' : s === 'pending' ? 'onDevicePending' : 'cloudOnly'))
      .catch(() => alive && setTier('cloudOnly'));
    return () => { alive = false; };           // avoid setting state after unmount
  }, [checkOnDevice]);

  return tier;
}
```

### Points to mention on any platform
- **Check availability at runtime, every time.** The user can turn off Apple Intelligence, and the OS can evict the model.
- **Prefetch models** in the background, on Wi-Fi, before the user needs the feature.
- **Tier C needs consent UX** if personal data goes to the cloud.
- Log the tier as an **analytics dimension**, so you know what share of users get the on-device experience.

---

## 3. Mini system design (5 min)

**Prompt:** *"Add AI 'smart reply' suggestions to a messaging app that ships on iOS and Android."* Answer it twice: once assuming **native** apps and once assuming **Flutter or React Native**.

> **How to use this:** spend 5 minutes sketching your own answer, then read the model answer **out loud**. It follows the structure interviewers expect: **Clarify → Estimate → High-level design → Deep dives → Trade-offs → Failure modes & metrics → Wrap-up**. Then practice the follow-up questions; that's where senior candidates stand out.

<details>
<summary><b>🎤 Model answer (say it to the interviewer)</b></summary>

#### Step 1: Clarify requirements (~2 min)

| I ask | Assume |
|---|---|
| Are chats end-to-end encrypted? | **Yes**, and this drives everything |
| Where do suggestions appear? | 3 chips above the keyboard after an incoming message |
| Latency target? | Under ~300–500 ms after the message arrives or the chat opens |
| Languages? | Top 10 to start |
| Scale? | 100M DAU, billions of messages per day |

**Functional:** 3 short, relevant replies; tap to send or edit.
**Non-functional:** privacy (E2E), low latency, zero marginal cost, no jank, safe (no offensive suggestions).

#### Step 2: Key decision
> "Because chats are E2E-encrypted, **the server can't read messages**. Sending them to a cloud LLM would break the E2E promise. So suggestions must be generated **on-device**. At billions of messages a day, cloud inference would also cost a fortune."

#### Step 3: High-level architecture

```
Incoming msg ─▶ Chat repository ─▶ Debounce (latest msg wins)
                                        │
                                        ▼
                         ┌────────────────────────────┐
                         │ SmartReplyEngine (iface)   │
                         └───────┬───────────┬────────┘
                    Tier A       │           │    Tier B/C
          ┌──────────────────────▼──┐   ┌────▼────────────────────┐
          │ OS LLM                  │   │ Small custom model      │
          │ 🤖 Gemini Nano (ML Kit) │   │ (LiteRT / Core AI)      │
          │ 🍎 Foundation Models    │   │ or feature hidden       │
          └──────────────┬──────────┘   └────┬────────────────────┘
                         ▼ structured output: Replies(options[3])
                  ┌──────────────────────────┐
                  │ Safety filter + dedupe   │
                  └────────────┬─────────────┘
                               ▼
                      Suggestion chips (UI)
```

#### Step 4: Deep dives
1. **Prompting and context:**
   - Send only the last ~6–10 messages (stays well within Nano's ~4K input limit).
   - A short system instruction ("3 replies, max 6 words each, match the user's language and tone").
   - **Structured output:** `@Generable Replies { options: [String] }` on both 🤖 and 🍎.
2. **Latency:**
   - Prefetch and `warmup()` the model when the chat list opens.
   - Generate as soon as the message arrives, before the user opens the chat.
   - **Cancel stale requests** when a newer message arrives (Flow `collectLatest` / Swift `Task` cancellation).
   - Cache suggestions per message ID.
3. **Device tiers:**
   - Tier A: OS model.
   - Tier B: a small custom model (a classic smart-reply ranker over a fixed reply set, a few MB).
   - Tier C: hide the feature. **Never fall back to the cloud for E2E content.**
4. **Native vs cross-platform:**
   - *Native:* use each OS model directly; best quality, no download.
   - *Flutter/RN, option A:* platform channels / native modules to each OS model. No download, but quality differs per platform.
   - *Flutter/RN, option B:* one open model (flutter_gemma / llama.rn / ExecuTorch) for identical behavior, but a few hundred MB to download and RAM limits.
   - **My choice:** option A for an assistive feature like this, behind one Dart or TS interface.
5. **System agent bonus:** expose `sendMessage` through **AppFunctions (Android MCP)** and **App Intents**, so "Text Sam I'm late" works from Gemini or Siri. Google recommends local Android MCP for E2E journeys.

#### Step 5: Trade-offs
| Choice | Why |
|---|---|
| On-device over cloud | E2E privacy and cost. Lose some quality |
| Structured output over free text | Reliable parsing. Slightly slower |
| Hide on Tier C rather than cloud fallback | Keeps the privacy promise. Lower coverage |

#### Step 6: Failure modes, safety, metrics
- **Safety:** a blocklist + classifier on suggestions, no suggestions in sensitive contexts (grief, conflict detection), never auto-send.
- **Failures:** quota errors → skip silently; model not ready → no chips (never block typing).
- **Metrics:** **suggestion acceptance rate** (the main metric), edit-before-send rate, latency p50/p95, battery impact. All logged **without message content** (privacy-safe analytics).

#### Step 7: Wrap-up
> "On-device generation because of E2E encryption, OS models with structured output on capable devices, a small fallback model or no feature elsewhere, aggressive prefetching and cancellation for latency, safety filtering, and acceptance rate as the success metric. Cross-platform, I'd wrap the OS models behind one interface rather than ship a large model."

</details>

### 🔁 Likely follow-up questions

**F1. How do you improve suggestion quality if you can't see any messages?**

<details>
<summary>Answer</summary>

Use **privacy-preserving signals**: acceptance and edit rates per prompt variant (no content), A/B test prompts, and **on-device personalization** (learn the user's style locally). For model improvement, **federated learning** or opt-in donated data; evaluate offline on public or synthetic chat datasets.

</details>

**F2. The OS model update changed behavior and acceptance dropped 15% overnight. What do you do?**

<details>
<summary>Answer</summary>

Gemini Nano and Foundation Models are updated **by the OS, not by us**. Detect it by segmenting metrics by model version. Respond by adjusting prompts or examples via **remote config** (no app release), and temporarily lower exposure if needed. Prevent it by running an **eval suite against preview models** (AICore Developer Preview / iOS betas) before rollouts.

</details>

**F3. How do you avoid suggestions in the wrong language or tone?**

<details>
<summary>Answer</summary>

Detect the conversation language on-device (ML Kit Language ID / NaturalLanguage framework) and put it in the instruction. Include the user's own recent messages as tone examples. Validate the output language and drop mismatched suggestions.

</details>

**F4. What changes for group chats?**

<details>
<summary>Answer</summary>

Include sender names in the context, cap the context more aggressively, suggest only when the user is directly addressed or the message is a question, and lower the frequency to avoid noise.

</details>

**F5. How would you test this?**

<details>
<summary>Answer</summary>

Unit-test the engine with a fake model. Golden conversation sets for each language with expected properties (length, language, safety) rather than exact strings. Performance tests on low-end devices. A staged rollout with acceptance and latency dashboards.

</details>

---

## 4. Mock interview questions (5 min)

Answer each out loud in about 60 seconds **before** opening the model answer. Then compare: did you cover the same structure, the trade-offs and a concrete example?

### Q1. What does Google mean by Android becoming an "intelligence system", and what should app developers do about it?

<details>
<summary><b>Model answer</b></summary>

**Meaning:** instead of users tapping through apps, they state a goal ("book a ride and text Sam my ETA") and an agent such as **Gemini carries out multi-step tasks across apps**. It covers prompted help (the user asks) and proactive help (suggestion chips on the lock screen, in notifications and on the keyboard).

**What developers should do:**
1. **Expose key actions and data with AppFunctions (Android MCP)**, now in beta. Agents then call your code directly, which is faster and more reliable than operating your UI.
2. **Keep the UI semantic and accessible.** Without integration, agents fall back to the **Computer Control framework** and drive your UI.
3. **Design tools carefully:** clear natural-language descriptions, typed parameters, **user confirmation for sensitive actions** (payments, sending messages), and auth handled natively.
4. **For reach beyond Android,** add a **remote MCP server** and look at the **Connected Apps Platform** (preview) for Gemini and Search.
5. **Test** with Google's AppFunctions test agent.

**One-liner:** "My app now has two clients: the user and the system agent. I design an API for both."

</details>

### Q2. Compare **AppFunctions** (Android) and **App Intents** (iOS). What problem do both solve?

<details>
<summary><b>Model answer</b></summary>

**Shared problem:** let the system assistant (Gemini / Siri) **act on your app's functionality and data** without screen-scraping, with structured inputs and outputs.

| | 🤖 AppFunctions (Android MCP) | 🍎 App Intents |
|---|---|---|
| Model | Your app is an **on-device MCP server**; functions are tools with descriptions | `AppIntent` actions + `AppEntity` data types |
| 2026 additions | Beta; Connected Apps Platform; test agent | **Intent schemas** (no phrases to define), **entity schemas** (Spotlight semantic index), **View Annotations** (on-screen context), App Intents Testing framework |
| Fallback without integration | Computer Control framework drives the UI | Siri has limited abilities |
| Ecosystem | MCP standard, so it lines up with remote MCP | Apple-specific, but also powers Shortcuts, Spotlight and widgets |

**Both:** run locally (good for privacy and E2E-encrypted data), need clear descriptions, and should require confirmation for destructive actions.

**Cross-platform insight:** share the business logic, then write two thin native adapters.

</details>

### Q3. Name the on-device LLM option on each of the four platforms, and give one limitation of each.

<details>
<summary><b>Model answer</b></summary>

| Platform | Option | Key limitation |
|---|---|---|
| 🤖 Android | **Gemini Nano** through the ML Kit GenAI Prompt API (Nano 4 coming) | Only on **supported devices** (mostly flagships). The model may need downloading. Small context; Firebase hybrid on-device mode is single-turn only |
| 🍎 iOS | **Foundation Models** (`SystemLanguageModel`, ~3B) | Needs an **Apple Intelligence–eligible** device, the user must have it **enabled**, and it's region-limited. Weak at world knowledge |
| 🐦 Flutter | **flutter_gemma** (or platform channels to the OS models) | **Community-maintained**. You ship or download the model (hundreds of MB+), and RAM-limited devices can't run it |
| ⚛️ React Native | **`@react-native-ai/apple`**, **llama.rn**, **ExecuTorch** | The Apple provider is **iOS 26+ only**. Open-model runtimes need model downloads and an **Expo dev build**; version compatibility is fragile (pin AI SDK v6) |

**Close with:** "So on every platform I design a cloud or graceful-degradation path, because on-device coverage is never 100%."

</details>

### Q4. Which of these would you use in production today, and why: ML Kit Agentic API, Foundation Models, Flutter GenUI, Vercel AI SDK?

<details>
<summary><b>Model answer</b></summary>

- **Foundation Models: yes.** It's stable since iOS 26, Apple-supported, free and private. I'd still gate it on `availability` and have a fallback.
- **Vercel AI SDK: yes.** It's mature and widely used, and gives one API across providers. I'd pin the major version (React Native AI providers target v6).
- **ML Kit Agentic API: not for critical flows yet.** It's **alpha**: no SLA, the API can change, and it depends on Gemini Nano 4, which is only on flagships. I'd prototype with it behind a feature flag with a fallback.
- **Flutter GenUI: prototype only.** It's **alpha/experimental**. For production generative UI, I'd have the model return **structured output** (a JSON schema) and render it with **my own widget set**, which is more predictable and testable.

**The principle to state:** "Alpha APIs go behind feature flags with a stable fallback, and I track their release notes. I never make a core user journey depend on them."

</details>

### Q5. Your company has a React Native app. A PM asks for "Siri and Gemini integration". What is involved?

<details>
<summary><b>Model answer</b></summary>

**This can't be done in JavaScript alone. It's a native-integration project.**

1. **iOS:** a Swift **App Intents** layer: `AppIntent` structs for the actions, `AppEntity` for the data, adoption of the new **intent/entity schemas**, and an `AppShortcutsProvider`. Expo apps add it with a **config plugin**.
2. **Android:** Kotlin **AppFunctions** (the AppFunctions Jetpack library) that expose the same actions as Android MCP tools.
3. **The hard part is the runtime.** The assistant may invoke an action **while the app isn't open**, so the JS runtime might not be running. Options:
   - Implement the action logic natively.
   - Start a headless JS task (Android Headless JS / a background RN context).
   - Call your backend directly from native code.
4. **Shared state:** auth tokens must be readable from native code (Keychain / Keystore), not only from JS memory.
5. **Safety:** confirmation for sensitive actions; clear, specific descriptions.
6. **Testing:** the App Intents Testing framework and Google's AppFunctions test agent.

**Estimate framing:** "A few well-chosen actions (for example, *create task*, *search tasks*) per platform. Start with the highest-value intents, not the whole app."

</details>

### Q6. What is MCP, and why are mobile platforms adopting it?

<details>
<summary><b>Model answer</b></summary>

**MCP (Model Context Protocol)** is an **open standard** (introduced by Anthropic in late 2024) for how AI applications **discover and call tools, read resources and use prompts** from external servers. Think of it as "USB-C for AI integrations": write a tool once and any MCP-compatible agent can use it.

**Core ideas:** client ↔ server. Servers declare **tools** (name, description, JSON schema for inputs), **resources** (readable data) and **prompts**. The agent decides when to call them.

**Why mobile is adopting it:**
- **One integration for many agents:** Gemini, other system agents and in-app agents all speak the same protocol.
- **Android MCP (AppFunctions)** makes your app a **local MCP server**, which suits **private, on-device, E2E-encrypted or sensor-based** actions.
- **Remote MCP** gives **cross-platform reach** (web, iOS, desktop agents). Google's Connected Apps Platform combines both.
- iOS uses its own App Intents, but the idea is the same.

**Security points:** least-privilege tools, user confirmation, input validation, and treating tool inputs as untrusted (prompt injection).

</details>

---

**Takeaway:** Think in five layers (OS model, custom runtime, cloud, system agent, gen UI), know each platform's API for every layer, and **quote maturity levels accurately.**

**Tomorrow (Day 3):** 🤖 Android on-device in depth: ML Kit, the Gemini Nano 4 Prompt API (system instructions, prefix caching, structured output), and LiteRT.
