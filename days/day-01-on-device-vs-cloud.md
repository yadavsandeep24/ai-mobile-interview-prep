# Day 1 — On-Device vs Cloud AI

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
| **Updates** | Ship a new model (APK or download) | Swap it on the server instantly |
| **Device coverage** | Varies by hardware (Gemini Nano runs on select devices only) | Every device |

### Android tools to name-drop
- **ML Kit**: ready-made on-device APIs (text recognition, barcode scanning, face detection, translation, plus GenAI APIs backed by Gemini Nano for summarizing and rewriting)
- **LiteRT** (formerly TensorFlow Lite): runs your own custom models on-device
- **MediaPipe**: ready-made pipelines for vision, audio and on-device LLM inference
- **Gemini Nano via AICore**: the system-level on-device LLM on supported devices. **Gemini Nano 4** (built on Gemma 4, with native function calling) is coming to flagships; prototype it through the AICore Developer Preview
- **LiteRT in Google Play services**: the runtime ships with Play services instead of your APK (about 8 MB smaller)
- **Firebase AI Logic** or direct REST/SDK calls: cloud models such as Gemini, Claude and GPT

**The answer that impresses interviewers is hybrid:** do it on-device when possible and fall back to the cloud when needed (for example, the device isn't supported, the task is too complex, or confidence is low).

### 2026 update: hybrid inference is built in
**Firebase AI Logic hybrid inference** (experimental) handles routing for you with four modes:

| Mode | Behavior |
|---|---|
| `PREFER_ON_DEVICE` | Try Gemini Nano first, fall back to the cloud |
| `PREFER_IN_CLOUD` | Use the cloud, fall back to on-device when offline |
| `ONLY_ON_DEVICE` | Throw an error if the on-device model can't handle the request |
| `ONLY_IN_CLOUD` | Always use the cloud |

**Limitations to mention:** on-device mode supports only single-turn text generation (text, or text plus one image), not multi-turn chat. The SDK doesn't download the model automatically, so trigger the download early and call `warmUp()`. App Check is enforced.

**When to build your own routing:** when you need custom rules (Gboard does this for proofreading and rewriting), multi-turn chat, a non-Gemini cloud model, or routing based on confidence. That's the pattern below.

## 2. Code (10 min): a custom hybrid router in Kotlin

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

**Points to mention while you whiteboard this:**
- The ViewModel depends on the `TextSummarizer` interface, so it doesn't know which backend runs. That keeps it testable and lets you swap backends.
- Run inference off the main thread (`Dispatchers.Default` for on-device work).
- Use `Result` so failures are handled explicitly instead of crashing.

## 3. Mini system design (5 min)

**Prompt:** *"Design a feature that auto-tags photos in a gallery app."*

<details>
<summary>Sample answer</summary>

- **On-device** (ML Kit image labeling or a custom LiteRT model). The reasons: photos are private, there are thousands of them (cloud calls would be expensive), and the feature should work offline.
- **Batch processing** in `WorkManager` with constraints (charging, idle) to protect the battery.
- **Store tags in Room** and index them for search.
- **Optional cloud step** for rich captions, opt-in only, with consent UX.
</details>

## 4. Mock interview questions (5 min)

Answer each out loud in about 60 seconds.

1. When would you choose a cloud model over an on-device model? Give a real example.
2. How would you handle a device that doesn't support Gemini Nano?
   <details><summary>Hint</summary>Check feature status at runtime, fall back to the cloud (or use <code>PREFER_ON_DEVICE</code> in Firebase AI Logic), degrade the UI gracefully, and never assume Nano exists.</details>
3. Your app's AI feature costs $40k a month in API calls. What do you do?
   <details><summary>Hint</summary>Caching, on-device for simple tasks, smaller models, prompt trimming, rate limiting.</details>
4. A user complains that the AI feature drains their battery. How do you investigate?
5. *(2026)* When would you choose Firebase AI Logic's built-in hybrid modes over your own router?

---

**Takeaway:** On-device for privacy, latency, offline use and cost; cloud for capability and coverage; hybrid for production.
