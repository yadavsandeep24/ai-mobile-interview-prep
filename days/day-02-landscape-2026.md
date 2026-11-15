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

<details>
<summary>Sample answer</summary>

**Requirements:** suggestions in under 300 ms, privacy (messages may be end-to-end encrypted), works offline, and low cost at scale (billions of messages).

**This should be on-device first:**
- E2E-encrypted content **should not go to a cloud model**, which rules out the cloud as the default path.
- Tier A: OS model. 🤖 ML Kit Prompt API (Gemini Nano) and 🍎 Foundation Models with a `@Generable struct Replies { var options: [String] }` for structured output.
- Tier B/C: a small custom model (LiteRT / Core AI), or **hide the feature**. Don't silently send E2E-encrypted messages to the cloud.

**Native vs cross-platform:**
- *Native:* use each OS model directly; it's the best quality and has no download.
- *Flutter/RN:* either (a) platform channels or native modules to each OS model (no download, but iOS and Android behave differently), or (b) ship one open model through flutter_gemma / llama.rn / ExecuTorch (the same behavior everywhere, but a model download of a few hundred MB and RAM limits).

**System agent tie-in (2026 bonus):** expose a `sendMessage` tool through **AppFunctions** (Android) and **App Intents** (iOS), so "Text Sam I'm running late" works from Gemini or Siri. Android's guidance prefers on-device Android MCP over remote MCP when E2E encryption is involved.

**Production:** a short context window (last N messages), prefix caching for the instructions, debouncing, discarding stale suggestions when a new message arrives, and tracking which suggestions users accept to measure quality.
</details>

---

## 4. Mock interview questions (5 min)

Answer each out loud in about 60 seconds.

1. What does Google mean by Android becoming an "intelligence system", and what should app developers do about it?
2. Compare **AppFunctions** (Android) and **App Intents** (iOS). What problem do both solve?
   <details><summary>Hint</summary>Both expose your app's actions and data to the system assistant, so agents call your code instead of operating your UI. AppFunctions is MCP-based (Android MCP, in beta). App Intents uses intent and entity schemas plus the Spotlight semantic index. Both reduce agent errors and keep sensitive work on the device.</details>
3. Name the on-device LLM option on each of the four platforms, and give one limitation of each.
4. Which of these would you use in production today, and why: ML Kit Agentic API, Foundation Models, Flutter GenUI, Vercel AI SDK?
   <details><summary>Hint</summary>Foundation Models and the AI SDK are production-ready. The Agentic API and GenUI are alpha (no SLA, and the APIs can change), so prototype with them or put them behind a feature flag.</details>
5. Your company has a React Native app. A PM asks for "Siri and Gemini integration". What is involved?
   <details><summary>Hint</summary>This can't be done in JavaScript alone. You need native code: App Intents in Swift and AppFunctions in Kotlin, exposed through native modules, with shared business logic behind them.</details>
6. What is MCP, and why are mobile platforms adopting it?

---

**Takeaway:** Think in five layers (OS model, custom runtime, cloud, system agent, gen UI), know each platform's API for every layer, and **quote maturity levels accurately.**

**Tomorrow (Day 3):** 🤖 Android on-device in depth: ML Kit, the Gemini Nano 4 Prompt API (system instructions, prefix caching, structured output), and LiteRT.
