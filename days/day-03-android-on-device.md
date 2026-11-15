# Day 3: 🤖 Android On-Device AI (ML Kit, Gemini Nano Prompt API, LiteRT)

Platforms: **🤖 Android deep dive**, with "equivalent on 🍎 / 🐦 / ⚛️" notes for cross-platform developers.

**Goal for today:** know the three Android on-device tools well enough to choose between them, write the core code from memory, and explain the production pitfalls (availability, token limits, quotas, threading, GPU fallback).

> **Sources:** code follows Google's official Android agent skills for the ML Kit GenAI Prompt API (updated Sep 2026) and LiteRT CompiledModel (updated Oct 2026). APIs are **beta / experimental**, so check the current docs for version numbers before you ship.

---

## 1. Concept (10 min)

### Three tools, three jobs

| Tool | Use it for | You bring | Example |
|---|---|---|---|
| **ML Kit (classic APIs)** | Common **perception** tasks | Nothing (Google's models) | Text recognition, barcodes, face detection, translation |
| **ML Kit GenAI Prompt API** (Gemini Nano) | **Generative** text and image→text with a free, built-in LLM | A prompt | Summarize, rewrite, extract, classify, smart reply |
| **LiteRT** (`CompiledModel`) | **Your own custom model** (`.tflite`) | A trained model | A custom classifier, embeddings, audio models |
| **LiteRT-LM** | **Your own small LLM** (`.litertlm`) | A fine-tuned SLM | Niche domain assistant |

**Decision rule:** *"Does ML Kit already solve it? → classic API. Is it generative and general? → Prompt API. Is it a domain-specific model? → LiteRT. A domain-specific LLM? → LiteRT-LM."*

### Gemini Nano Prompt API: facts to quote
- **Dependency:** `com.google.mlkit:genai-prompt` (beta), **minSdk 26**.
- **Lifecycle:** `Generation.getClient()` → `checkStatus()` → `download()` if `DOWNLOADABLE` → optional `warmup()` → `generateContent()` / `generateContentStream()` → `close()`.
- **Status values:** `AVAILABLE`, `DOWNLOADABLE`, `DOWNLOADING`, `UNAVAILABLE`.
- **Limits:** input **under ~4,000 tokens** (~3,000 English words), avoid long outputs, and **AICore enforces a per-app inference quota**. Use `countTokens` to check size.
- **Request options:** `temperature`, `topK`, `seed` (deterministic), `candidateCount`, `maxOutputTokens`.
- **Multimodal:** `ImagePart(bitmap)` + `TextPart(...)`.

### Three newer production features (2026)

| Feature | What it does | Gotchas |
|---|---|---|
| **System instructions** (`SystemInstruction`) | Persona, format and constraint rules with higher priority than the user prompt | Gemini Nano **V3+** only. Keep it **under ~150 words**. **Not recommended together with prefix caching** |
| **Prefix caching** (`PromptPrefix`) | Reuses the processed state of a **static prompt prefix**. Google measured on Pixel 9: **2.11 s → 0.5 s** (1,000-token prefix), **0.82 s → 0.45 s** (300-token prefix) | Experimental, subset of devices, **text only** (no images). Caches use app storage (LRU, encrypted). Implicit or explicit (`caches.create/list/delete`) |
| **Structured output** (`@Generable`, `@Guide`) | Returns a **typed Kotlin object** instead of free text | **Kotlin only**, needs KSP plus `genai-schema-compiler`. Check `isStructuredOutputFeatureAvailable()`. Add **ProGuard keep rules** for the classes |

**When to use which:** short behavior rules → system instruction. A long, repeated context (few-shot examples, a product catalog, policies) → prefix caching. The skill's guidance is to **use prefix caching when the prompt is over ~200 words**.

### LiteRT (custom models): facts to quote
- **`CompiledModel` API** (LiteRT 2.x) replaces the deprecated `org.tensorflow.lite.Interpreter`.
- **Two runtimes:**
  - **Google Play services** (`play-services-litert`): the runtime isn't in your APK (**~8 MB smaller**) and Play updates it. This is the default choice.
  - **Bundled** (`litert` + `litert-gpu`): needed for the **NPU**, **custom ops**, or devices **without Play services**. **Never put both in one build variant.**
- **Accelerators:** `CompiledModel.Options(Accelerator.GPU, Accelerator.CPU)` means GPU with CPU fallback.
- **Rules:**
  - One **single-thread** dispatcher owns the model; never use the main thread.
  - **Allocate buffers once** and reuse them.
  - **Warm up** once (the first GPU run compiles shaders).
  - Close the buffers before the model.
- **Large models:** deliver them through **Play for On-Device AI** (AI packs) or download them, rather than bundling them in the base APK.

### Equivalents on other platforms
| Android | 🍎 iOS | 🐦 Flutter | ⚛️ React Native |
|---|---|---|---|
| ML Kit classic | Vision / Natural Language frameworks | `google_mlkit_*` plugins | ML Kit / Vision native modules |
| Prompt API (Gemini Nano) | Foundation Models | Platform channel | Native module (Callstack's React Native AI is adding Gemini Nano support) |
| `@Generable` (ML Kit) | `@Generable` (Foundation Models) | JSON schema | `generateObject` + zod |
| LiteRT | Core AI | `flutter_litert`, TFLite plugins | ExecuTorch, llama.rn |

*Interview tip:* both Google and Apple use the name **`@Generable`** for structured output. Point that out.

---

## 2. Code (10 min)

### A. Gemini Nano repository: availability, download, streaming, system instruction

```kotlin
class OnDeviceLlm : AutoCloseable {
    private val model = Generation.getClient()

    /** Call early (app start, on Wi-Fi). Never wait to download until the user taps. */
    suspend fun ensureReady(onProgress: (Long) -> Unit = {}): Boolean =
        when (model.checkStatus()) {
            FeatureStatus.AVAILABLE -> true.also { model.warmup() }
            FeatureStatus.DOWNLOADABLE -> {
                var ok = false
                model.download().collect { s ->
                    when (s) {
                        is DownloadStatus.DownloadProgress -> onProgress(s.totalBytesDownloaded)
                        DownloadStatus.DownloadCompleted -> ok = true
                        is DownloadStatus.DownloadFailed -> ok = false
                        else -> Unit
                    }
                }
                ok
            }
            else -> false   // DOWNLOADING or UNAVAILABLE → fall back (see Day 1)
        }

    /** Streams tokens to the UI as a Flow. */
    fun rewrite(text: String, tone: String): Flow<String> {
        val request = generateContentRequest(
            SystemInstruction("You rewrite text in a $tone tone. Keep the meaning. Max 3 sentences."),
            TextPart(text),
        ) {
            temperature = 0.3f
            maxOutputTokens = 256
        }
        return model.generateContentStream(request)
            .map { chunk -> chunk.candidates.firstOrNull()?.text.orEmpty() }
    }

    override fun close() = model.close()   // call from ViewModel.onCleared()
}
```

### B. Structured output: extract an event from a message

```kotlin
@Generable("A calendar event found in a message")
data class EventInfo(
    @Guide(description = "Short title of the event") val title: String = "",
    @Guide(description = "Date in ISO-8601 (YYYY-MM-DD), empty if none") val date: String = "",
    @Guide(description = "Location, empty if none") val location: String = "",
)

suspend fun extractEvent(model: GenerativeModel, message: String): EventInfo? {
    if (!model.isStructuredOutputFeatureAvailable()) return null      // fall back to the cloud
    val base = GenerateContentRequest.Builder(TextPart("Extract the event: $message")).build()
    val typed = generateTypedContentRequest(base, EventInfo::class)
    return model.generateContent(typed).candidates.firstOrNull()?.response
}
// ProGuard: -keep class com.example.EventInfo { *; }
```

### C. Prefix caching: a long, static prompt with a short dynamic input

```kotlin
// A ~800-token static block: policies + few-shot examples. Same for every request.
private val SUPPORT_PREFIX = """
    You classify customer messages into: BILLING, DELIVERY, ACCOUNT, OTHER.
    Examples:
    ...
""".trimIndent()

suspend fun classify(model: GenerativeModel, message: String): String =
    model.generateContent(
        generateContentRequest(TextPart("Message: $message\nCategory:")) {
            promptPrefix = PromptPrefix(SUPPORT_PREFIX)   // implicit cache; text-only
            temperature = 0f
        }
    ).candidates.firstOrNull()?.text.orEmpty().trim()
```

### D. LiteRT: a custom model with GPU→CPU fallback (Play services runtime)

```kotlin
// build.gradle.kts: implementation("com.google.android.gms:play-services-litert:17.0.0")

object LiteRtEnv {                                     // one Environment per process
    private val mutex = Mutex(); private var env: Environment? = null
    suspend fun get(ctx: Context): Environment = mutex.withLock {
        env ?: LiteRt.initialize(
            ctx.applicationContext,
            LiteRtInitializationOptions.builder().setEnableGpu(true).build()
        ).await().also { env = it }
    }
}

class Classifier(ctx: Context, env: Environment) : AutoCloseable {
    private val model = CompiledModel.create(
        ctx.assets, "model.tflite",
        CompiledModel.Options(Accelerator.GPU, Accelerator.CPU),   // GPU, CPU fallback
        env,
    )
    private val inputs = model.createInputBuffers()               // allocate once
    private val outputs = model.createOutputBuffers()

    init { infer(FloatArray(224 * 224 * 3)) }                     // warm-up (shader compile)

    fun infer(input: FloatArray): FloatArray {
        inputs[0].writeFloat(input)
        model.run(inputs, outputs)
        return outputs[0].readFloat()
    }

    override fun close() { (inputs + outputs).forEach { it.close() }; model.close() }
}
// Create, run and close on ONE single-thread dispatcher, never on Main.
```

### Points to mention
- **ViewModel owns the lifecycle:** create in `init`, `close()` in `onCleared()`.
- **Stream** generative output for perceived speed. **Don't** stream structured output; wait for the typed object.
- **Count tokens** before sending (4K input limit). Chunk or summarize long inputs.
- **Gate every feature** on `checkStatus()` / `isStructuredOutputFeatureAvailable()`, with a fallback.

---

## 3. Mini system design (5 min)

**Prompt:** *"Design an on-device 'scan a receipt → categorized expense' feature for an Android expense app."*

<details>
<summary><b>Sample answer</b></summary>

**Pipeline:**
1. **CameraX** capture → **ML Kit Text Recognition** (classic, on-device, works on all devices) → raw text.
2. **Gemini Nano + structured output**: a `@Generable data class Expense(merchant, date, total, currency, category)`, with `@Guide` descriptions and an enum-like list of categories. `temperature = 0` for consistency.
3. **Validate** in code: total > 0, a parseable date, a known currency. If invalid → let the user edit, or retry once.
4. Save to **Room**. Sync later with **WorkManager**.

**Why this split:** OCR is a solved perception problem (ML Kit classic). Understanding messy receipt text is a generative task (Nano).

**Fallbacks:**
- Nano `UNAVAILABLE` → cloud extraction (Firebase AI Logic) **with consent**, since receipts can contain card digits. Otherwise use a **rules-based parser** plus manual entry.
- OCR text over the token limit → keep only the lines with prices and the header and footer.

**Performance:**
- Prefetch Nano at onboarding and call `warmup()` when the camera screen opens.
- If the categorization instructions are long, put them in a **prefix cache** (text only, so it applies to the OCR text, not the image).
- Show the OCR result immediately; the structured result fills in afterwards.

**Privacy:** fully on-device on Tier A devices, which is a strong selling point.
</details>

---

## 4. Mock interview questions (5 min)

Answer each out loud in about 60 seconds **before** opening the model answer.

### Q1. When would you use ML Kit classic APIs, the Gemini Nano Prompt API, or LiteRT?

<details>
<summary><b>Model answer</b></summary>

- **ML Kit classic:** for standard **perception** tasks Google already solved well: OCR, barcodes, faces, pose, translation, language ID. It runs on **almost every device**, is small and fast, and is deterministic. *Always check here first.*
- **Prompt API (Gemini Nano):** for **open-ended generative** tasks: summarize, rewrite, extract into a schema, classify with nuance, smart replies, image description. It's free and private, but **device coverage is limited** (supported devices only), input is under ~4K tokens, and there's a per-app quota.
- **LiteRT:** when I have a **domain-specific model**, for example a defect detector, a custom audio classifier or my own embedding model. I control accuracy and size, and it runs on CPU/GPU (NPU with the bundled runtime) on nearly all devices. The cost is training, converting and maintaining the model.
- **LiteRT-LM:** a domain **LLM** I fine-tuned myself.

**Often combined:** ML Kit OCR → Nano extracts structured data. Example: receipt scanning.
</details>

### Q2. Walk me through the lifecycle of using Gemini Nano in an app. What can go wrong?

<details>
<summary><b>Model answer</b></summary>

1. **Create** `Generation.getClient()` in the ViewModel.
2. **`checkStatus()`:** `AVAILABLE` / `DOWNLOADABLE` / `DOWNLOADING` / `UNAVAILABLE`.
3. If downloadable, **`download()`** returns a `Flow<DownloadStatus>`. Do this **early** (onboarding, on Wi-Fi), not when the user taps.
4. **`warmup()`** when the feature's screen opens, to load the model into memory.
5. **Generate:** `generateContent` or `generateContentStream`, with system instructions, options and maybe a prefix cache.
6. **`close()`** in `onCleared()`.

**What goes wrong:**
- The device is `UNAVAILABLE` (unsupported or config not fetched yet), so you **need a fallback**.
- The download fails or is slow, so show progress and retry.
- Input **over ~4K tokens**, so count tokens and chunk or summarize.
- **Quota errors**, because AICore limits inference per app. Debounce and cache.
- **Leaks** if you never `close()`.
- **Inconsistent output**, so lower the temperature, set a `seed`, use structured output, and validate.
</details>

### Q3. Explain prefix caching. When does it help, and when doesn't it?

<details>
<summary><b>Model answer</b></summary>

**What it is:** an LLM processes the prompt (the "prefill") before generating. If many requests share the same **static prefix** (instructions, few-shot examples, reference data), the Prompt API can **store the model's intermediate state** for that prefix and reuse it, so only the **dynamic suffix** is processed.

**Numbers:** Google measured on Pixel 9 that a 1,000-token prefix + 100-token input went from **2.11 s to 0.5 s**.

**It helps when:**
- The prompt is long (the skill suggests **over ~200 words**) and **repeated**: classifiers, extractors, assistants with fixed rules.

**It doesn't help, or doesn't apply, when:**
- The prompt is short or always different.
- The input contains **images** (it's text only).
- The device isn't in the supported subset (it's experimental).
- You're using **system instructions** (Google doesn't recommend combining the two; put the rules in the prefix instead).

**Trade-offs:** caches use **app storage** (LRU, encrypted), and the first call is a cache miss. I'd mention implicit (`promptPrefix`) versus explicit (`caches.create/list/delete`) management and `clearImplicitCaches()`, for example when the user logs out.
</details>

### Q4. How do you get reliable, parseable output from a small on-device model?

<details>
<summary><b>Model answer</b></summary>

1. **Structured output API:** a `@Generable` data class with `@Guide` descriptions and constraints (min/max items, ranges). The SDK returns a **typed object**, so there's no fragile JSON parsing. Check `isStructuredOutputFeatureAvailable()` first and add ProGuard keep rules.
2. **Clear prompts:** short, direct instructions. Few-shot examples, often in a prefix cache.
3. **Low randomness:** `temperature = 0`, and a fixed `seed` for reproducibility.
4. **Validate in code:** never trust the model. Check types, ranges and allowed enum values. If validation fails, **retry once**, then fall back (cloud, rules or manual input).
5. **Keep the task small:** small models do much better on narrow tasks. Split "extract + categorize + summarize" into separate steps.
6. **Measure:** an eval set of real inputs, tracking the parse-success rate and field accuracy across model versions. Nano gets updated by the OS, so **regressions can happen without an app release.**
</details>

### Q5. You ship a LiteRT model and it's slow on some devices. How do you debug and optimize?

<details>
<summary><b>Model answer</b></summary>

**Diagnose:**
- Which **accelerator** actually ran? `Options(GPU, CPU)` silently falls back to the CPU if the GPU accelerator isn't loaded or an op isn't supported. Check Logcat and record the chosen accelerator in analytics.
- **Measure properly:** `model.run` enqueues GPU work asynchronously, so time `run` **plus** the output read together. Exclude the first run (shader compilation).
- Profile with **Perfetto** / Android Studio Profiler: preprocessing (Bitmap → float conversion) is often the real bottleneck.

**Optimize:**
- **Quantize** (int8 / fp16), which makes the model smaller and faster.
- Use the **GPU**. Use the **NPU** via the bundled runtime if needed.
- **Warm up** once at load. **Reuse `TensorBuffer`s** instead of allocating per call.
- Run on a **single-thread dispatcher** and set CPU `numThreads` sensibly.
- **Memory:** load from a file path instead of assets, and use the XNNPack weight cache.
- **Delivery:** Play services runtime (smaller APK), AI packs for large models.
- **Tier devices:** use a smaller model variant on low-end devices.
</details>

### Q6. *(Cross-platform)* Your Flutter app needs on-device OCR and summarization on Android and iOS. What do you use?

<details>
<summary><b>Model answer</b></summary>

- **OCR:** the `google_mlkit_text_recognition` plugin. It wraps ML Kit on both Android and iOS, so you get one Dart API and consistent results. (Alternative: Apple's Vision framework through a channel on iOS.)
- **Summarization, two options:**
  - **OS models through platform channels:** Gemini Nano (Prompt API) on Android and **Foundation Models** on iOS. No download and free, but **different quality per platform** and only on eligible devices.
  - **One open model with flutter_gemma:** the same behavior on both platforms, but a large download and RAM requirements.
- **Architecture:** a Dart `Summarizer` interface with an `OsModelSummarizer`, a `GemmaSummarizer` and a `CloudSummarizer` (`firebase_ai`), chosen by **capability tier** (Day 2).
- **Practical details:**
  - Do the heavy work natively or in an isolate.
  - Stream the summary through an `EventChannel` → Dart `Stream`.
  - Use structured output on each side (`@Generable` exists on both platforms) and map it to one Dart model.
</details>

---

**Takeaway:** ML Kit for perception, Gemini Nano for general generative tasks, LiteRT for your own models. Always check availability, respect the **4K token / quota** limits, use **structured output + validation** for reliability, and **prefix caching** for long repeated prompts.

**Tomorrow (Day 4):** 🍎 iOS on-device: Foundation Models (`LanguageModelSession`, `@Generable`, `Tool`), iOS 27 changes, Core AI and Private Cloud Compute.
