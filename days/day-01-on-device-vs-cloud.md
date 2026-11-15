# Day 1 — On-Device vs Cloud AI

Platforms: 🤖 Android · 🍎 iOS · 🐦 Flutter · ⚛️ React Native

## 1. Concept (10 min)

**The first question in every mobile AI design is where the model runs.**

| | On-device | Cloud |
|---|---|---|
| **Latency** | Low, no network round trip | Network-dependent (often 300ms–several seconds) |
| **Offline** | ✅ Works | ❌ Needs a connection |
| **Privacy** | Data never leaves the phone | Data is sent to a server |
| **Cost** | Free per inference | Pay per token or request |
| **Model size/quality** | Small (MBs to a few GB), less capable | Large frontier models, most capable |
| **Device impact** | Uses battery, RAM, storage; heats the phone | Minimal |
| **Updates** | Ship a new model (app update or download) | Swap it on the server instantly |
| **Device coverage** | Varies by hardware (Gemini Nano and Apple Intelligence run on select devices only) | Every device |

### Tools to name-drop

#### 🤖 Android
- **ML Kit**: ready-made on-device APIs (text recognition, barcode scanning, face detection, translation, plus GenAI APIs backed by Gemini Nano)
- **Gemini Nano via AICore**: the system-level on-device LLM on supported devices. **Gemini Nano 4** (built on Gemma 4, with native function calling) is coming to flagships; prototype it through the AICore Developer Preview
- **LiteRT** (formerly TensorFlow Lite), now shipped in **Google Play services** (about 8 MB smaller APK), plus **LiteRT-LM** for your own small LLMs
- **Firebase AI Logic**: cloud Gemini, with built-in hybrid routing (see below)

#### 🍎 iOS
- **Foundation Models framework**: Apple's on-device model (~3B parameters). Free, works offline, private
- **iOS 27**: the same API also works with cloud models (Claude, Gemini, Private Cloud Compute) through the Language Model protocol, so hybrid routing is built into one API
- **Core AI**: runs your own models on-device (the counterpart to LiteRT)

#### 🐦 Flutter
- **`firebase_ai`** (Firebase AI Logic) for cloud Gemini
- **flutter_gemma** (community) for on-device LLMs that behave the same on Android and iOS
- **Platform channels** to reach Gemini Nano or Apple Foundation Models natively

#### ⚛️ React Native
- **Vercel AI SDK** as one API for every model
- **`@react-native-ai/apple`** for Apple Foundation Models, and **llama.rn** / **React Native ExecuTorch** for open models on-device
- Needs an **Expo development build**, because Expo Go can't load on-device AI native modules

**The answer that impresses interviewers is hybrid:** do it on-device when possible and fall back to the cloud when needed (for example, the device isn't supported, the task is too complex, or confidence is low).

### 2026 update: hybrid inference is becoming built in
- 🤖 **Firebase AI Logic hybrid inference** (experimental, Android) offers four modes:

  | Mode | Behavior |
  |---|---|
  | `PREFER_ON_DEVICE` | Try Gemini Nano first, fall back to the cloud |
  | `PREFER_IN_CLOUD` | Use the cloud, fall back to on-device when offline |
  | `ONLY_ON_DEVICE` | Throw an error if the on-device model can't handle the request |
  | `ONLY_IN_CLOUD` | Always use the cloud |

  **Limitations:** on-device mode supports only single-turn text generation (text, or text plus one image), not chat. The SDK doesn't download the model automatically, so trigger the download early and call `warmUp()`. App Check is enforced.
- 🍎 **iOS 27 Foundation Models** puts on-device and cloud models behind one Language Model protocol, and Dynamic Profiles let one session switch models.
- 🐦⚛️ **Cross-platform** apps usually build their own router.

**When to build your own routing on any platform:** custom rules (Gboard does this for proofreading and rewriting), multi-turn chat, a non-default cloud model, or routing based on confidence.

## 2. Code (10 min): a custom hybrid router

The pattern is the same on every platform: **a common interface, an on-device implementation, a cloud implementation, and a router.** Do your platform's track.

### 🤖 Android (Kotlin)

```kotlin
interface TextSummarizer {
    suspend fun summarize(text: String): Result<String>
}

class OnDeviceSummarizer(/* ML Kit GenAI / Gemini Nano client */) : TextSummarizer {
    suspend fun isAvailable(): Boolean = TODO("check device feature status")
    override suspend fun summarize(text: String) = runCatching { TODO() }
}

class CloudSummarizer(private val api: LlmApi) : TextSummarizer {
    override suspend fun summarize(text: String) = runCatching {
        api.generate(prompt = "Summarize in 3 bullets:\n$text")
    }
}

class HybridSummarizer(
    private val local: OnDeviceSummarizer,
    private val cloud: CloudSummarizer,
    private val network: NetworkMonitor,
) : TextSummarizer {
    override suspend fun summarize(text: String): Result<String> {
        if (local.isAvailable()) {
            local.summarize(text).onSuccess { return Result.success(it) }
        }
        return if (network.isOnline()) cloud.summarize(text)
        else Result.failure(IllegalStateException("Offline and on-device model unavailable"))
    }
}
```

### 🍎 iOS (Swift, Foundation Models)

```swift
import FoundationModels

protocol TextSummarizer {
    func summarize(_ text: String) async throws -> String
}

struct OnDeviceSummarizer: TextSummarizer {
    var isAvailable: Bool { SystemLanguageModel.default.availability == .available }

    func summarize(_ text: String) async throws -> String {
        let session = LanguageModelSession(instructions: "Summarize in 3 short bullets.")
        return try await session.respond(to: text).content
    }
}

struct CloudSummarizer: TextSummarizer {
    let api: LLMClient            // your backend proxy; never ship API keys in the app
    func summarize(_ text: String) async throws -> String {
        try await api.generate(prompt: "Summarize in 3 bullets:\n\(text)")
    }
}

struct HybridSummarizer: TextSummarizer {
    let local = OnDeviceSummarizer()
    let cloud: CloudSummarizer

    func summarize(_ text: String) async throws -> String {
        if local.isAvailable, let result = try? await local.summarize(text) {
            return result
        }
        return try await cloud.summarize(text)
    }
}
```
*Mention:* `availability` can be `.unavailable` with a reason (device not eligible, Apple Intelligence turned off, model not ready), and the UI should handle each one. In iOS 27 you can also point a session at a cloud provider through the Language Model protocol.

### 🐦 Flutter (Dart)

```dart
abstract class TextSummarizer {
  Future<String> summarize(String text);
}

/// On-device: flutter_gemma (same model on Android & iOS), or a platform
/// channel to Gemini Nano / Apple Foundation Models.
class OnDeviceSummarizer implements TextSummarizer {
  Future<bool> isAvailable() async => false; // model downloaded & device capable?
  @override
  Future<String> summarize(String text) async => throw UnimplementedError();
}

class CloudSummarizer implements TextSummarizer {
  final _model = FirebaseAI.googleAI().generativeModel(model: kCloudModel);

  @override
  Future<String> summarize(String text) async {
    final res = await _model.generateContent([Content.text('Summarize in 3 bullets:\n$text')]);
    return res.text ?? '';
  }
}

class HybridSummarizer implements TextSummarizer {
  HybridSummarizer(this.local, this.cloud);
  final OnDeviceSummarizer local;
  final CloudSummarizer cloud;

  @override
  Future<String> summarize(String text) async {
    if (await local.isAvailable()) {
      try {
        return await local.summarize(text);
      } catch (_) {/* fall through to cloud */}
    }
    return cloud.summarize(text);
  }
}
```
*Mention:* Firebase's built-in hybrid modes are in the native Android SDK, so in Flutter you usually write the router yourself. Run heavy on-device work in a native plugin or an isolate so the UI thread doesn't drop frames.

### ⚛️ React Native (TypeScript, Vercel AI SDK)

```ts
import { generateText, type LanguageModel } from 'ai';
import { apple } from '@react-native-ai/apple'; // Apple Foundation Models, iOS 26+

const prompt = (t: string) => `Summarize in 3 bullets:\n${t}`;

async function run(model: LanguageModel, text: string) {
  const { text: out } = await generateText({ model, prompt: prompt(text) });
  return out;
}

export async function summarize(text: string, cloudModel: LanguageModel) {
  try {
    return await run(apple(), text);    // on-device first
  } catch {
    return await run(cloudModel, text); // cloud fallback (through your backend)
  }
}
```
*Mention:* the AI SDK's provider abstraction makes switching between on-device and cloud a one-line change. Pin compatible versions (React Native AI providers target AI SDK v6). On Android, use llama.rn or ExecuTorch for the on-device model, and download models on demand instead of bundling them.

### Points to mention on any platform
- The UI layer (ViewModel, SwiftUI view model, Bloc/Riverpod or a React hook) depends on the `TextSummarizer` interface, so it doesn't know which backend runs. That keeps it testable and lets you swap backends.
- Run inference off the main/UI thread (Kotlin `Dispatchers.Default`, Swift `async` off the MainActor, a Dart isolate or native plugin, a native module instead of the JS thread).
- Handle failures explicitly (`Result`, `throws`, `try/catch`) instead of crashing.
- Never ship cloud API keys in the app. Call through your backend or Firebase with App Check.

## 3. Mini system design (5 min)

**Prompt:** *"Design a feature that auto-tags photos in a gallery app."*

<details>
<summary>Sample answer</summary>

- **On-device**: ML Kit image labeling / LiteRT on Android, the Vision framework / Core AI on iOS, and the same engines through plugins or native modules in Flutter and React Native. The reasons: photos are private, there are thousands of them (cloud calls would be expensive), and the feature should work offline.
- **Batch processing** in the background with constraints (charging, idle) to protect the battery: `WorkManager` on Android, `BGProcessingTask` on iOS, wrapped by a background-task plugin in Flutter or React Native.
- **Store tags locally** (Room / SwiftData / drift / SQLite) and index them for search.
- **Optional cloud step** for rich captions, opt-in only, with consent UX.
</details>

## 4. Mock interview questions (5 min)

Answer each out loud in about 60 seconds **before** opening the model answer. Then compare: did you cover the same structure, the trade-offs and a concrete example?

### Q1. When would you choose a cloud model over an on-device model? Give a real example.

<details>
<summary><b>Model answer</b></summary>

**Structure:** capability → coverage → consistency, then mitigate the downsides.

- **When the task is beyond small models:** long-context reasoning, up-to-date world knowledge, complex multi-step generation, or high-stakes accuracy. On-device models (~3B parameters) are good at summarizing, rewriting and classifying, and weak at knowledge and deep reasoning.
- **When coverage matters:** the feature must work on *every* device, not just flagships with Gemini Nano or Apple Intelligence.
- **When consistency and fast iteration matter:** one model to evaluate, and it can be swapped server-side without an app release.

**Example:** in a travel app, *"plan a 5-day itinerary from this 20-page PDF using live prices"* goes to the cloud (long context, fresh data, reasoning). *"Summarize this notification"* stays on the device.

**Mitigations I'd mention:** call through my backend or Firebase with App Check (no API keys in the app), send the minimum data, get consent when personal data leaves the device, and set timeouts and a fallback for offline users.

</details>

### Q2. How would you handle a device that doesn't support Gemini Nano or Apple Intelligence?

<details>
<summary><b>Model answer</b></summary>

**Detect, tier, degrade gracefully.**

1. **Detect at runtime:** ML Kit `checkStatus()` on Android, `SystemLanguageModel.default.availability` on iOS. Never assume the model exists; users can turn off Apple Intelligence, and models can be evicted.
2. **Tier the device:** OS model ready → downloadable (prefetch on Wi-Fi) → cloud only.
3. **Fall back based on the data:**
   - Non-sensitive: route to the cloud (Firebase `PREFER_ON_DEVICE` does this automatically).
   - Sensitive (for example, E2E-encrypted messages): use a small custom model (LiteRT / Core AI), or **hide the feature** rather than silently sending the data to the cloud.
4. **UX:** never block core flows. On iOS, if the reason is `appleIntelligenceNotEnabled`, you can suggest enabling it.
5. **Measure:** log the tier as an analytics dimension and test on low-end devices in CI or a device farm.

</details>

### Q3. Your app's AI feature costs $40k a month in API calls. What do you do?

<details>
<summary><b>Model answer</b></summary>

**Measure first, then cut the biggest drivers, without letting quality drop.**

1. **Measure:** a cost dashboard per feature, per user and per prompt. Usually 1–2 call sites drive most of the spend.
2. **Avoid calls:** cache identical or semantically similar requests. Debounce triggers (don't call on every keystroke). Generate only when the user asks.
3. **Move work on-device:** simple tasks (classify, short summaries, rewrites) go to Gemini Nano / Foundation Models, which cost nothing per call.
4. **Route by complexity:** a small, cheap model by default, and the large model only for hard requests.
5. **Shrink tokens:** shorter system prompts, prompt/prefix caching, summarize or trim chat history, cap output tokens, ask for structured output instead of prose.
6. **Guardrails:** per-user rate limits, Firebase spend caps and budget alerts. On iOS, Private Cloud Compute is free for qualifying small-business apps.
7. **Validate:** run evals before and after each change, so you know quality didn't regress.

**Close with a number:** "Caching plus moving the two simplest features on-device would likely cut 50–70% of the bill. I'd confirm that with the dashboard."

</details>

### Q4. A user complains that the AI feature drains their battery. How do you investigate?

<details>
<summary><b>Model answer</b></summary>

**Reproduce → profile → find the cause → fix → prevent.**

- **Profile:**
  - 🤖 Android Studio Power Profiler, Perfetto system traces, Battery Historian.
  - 🍎 Instruments (Power Profiler / Energy), MetricKit field data.
  - 🐦⚛️ The same native tools on the host app, plus Flutter DevTools or the React Native profiler for UI-thread work.
- **Usual causes:**
  - Inference on the **CPU** instead of the GPU/NPU (a missing delegate or acceleration).
  - **Reloading the model** on every request instead of keeping it warm.
  - AI running **in the background** too often, wake locks.
  - Network retry storms or long-held streaming connections.
  - A model that is too big (not quantized).
- **Fixes:** hardware acceleration, a quantized model, load once and unload when the app is backgrounded, batch background work under *charging + idle* constraints (WorkManager / BGProcessingTask), throttle triggers, cache results.
- **Thermal:** watch `PowerManager.getCurrentThermalStatus()` / `ProcessInfo.thermalState`, and back off or route to the cloud when the device is hot.
- **Prevent:** add energy metrics to release health dashboards and alert on regressions.

</details>

### Q5. *(2026)* When would you choose built-in hybrid routing (Firebase AI Logic modes, iOS 27 Language Model protocol) over your own router?

<details>
<summary><b>Model answer</b></summary>

**Built-in routing** when:
- The task is simple and **single-turn** (Firebase's on-device mode supports only single-turn text, or text plus one image).
- The vendor's cloud model is fine (Gemini through Firebase; any Language Model provider on iOS 27).
- You want less code to maintain and you get App Check, model download handling and updates for free.

**My own router** when:
- I need **multi-turn chat** on-device.
- I use a **different cloud provider**, or several.
- I need **custom rules**: route by data sensitivity, confidence score, per-user cost budget, or subscription tier.
- It's a **Flutter/React Native** app that needs the same logic on both platforms (Firebase hybrid is a native Android feature).
- I need detailed telemetry or A/B testing of routing strategies.

**Good line:** "Start with the built-in modes behind my own `TextSummarizer` interface. If the requirements outgrow them, I replace the implementation without touching the UI."

</details>

### Q6. *(Cross-platform)* In Flutter or React Native, how do you give users the same on-device AI behavior on iOS and Android?

<details>
<summary><b>Model answer</b></summary>

**Two options. Choose based on what matters more: consistency or footprint.**

| | A. Ship one open model | B. Use each OS model |
|---|---|---|
| How | flutter_gemma / llama.rn / ExecuTorch with the **same model file** (for example, a quantized Gemma) on both platforms | Platform channels / native modules to Gemini Nano and Foundation Models |
| ✅ | Identical outputs; one prompt set and one eval suite; works on more devices | No download; OS-optimized (NPU); better battery use; free |
| ❌ | Download of a few hundred MB to GBs, RAM limits on low-end phones, you handle model updates and licenses | Different quality and behavior per platform; flagship-only coverage; two native integrations |

**What I'd do:** if consistency is a product requirement (for example, extraction that feeds the backend), choose **A**, downloaded on demand over Wi-Fi with a RAM check for tiering. If it's an assistive feature (rewrite, summarize), choose **B** and keep the outputs loosely specified. Either way, hide it behind one Dart or TypeScript interface, so you can switch later and fall back to the cloud.

</details>

---

**Takeaway:** On-device for privacy, latency, offline use and cost; cloud for capability and coverage; hybrid for production, on every platform.
