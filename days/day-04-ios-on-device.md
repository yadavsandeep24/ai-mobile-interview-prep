# Day 4: 🍎 iOS On-Device AI (Foundation Models, Private Cloud Compute, Core AI)

Platforms: **🍎 iOS deep dive**, with "equivalent on 🤖 / 🐦 / ⚛️" notes for cross-platform developers.

**Goal for today:** know Apple's AI stack well enough to choose between the on-device model, the **Private Cloud Compute (PCC)** server model and your own model; write Foundation Models code from memory (sessions, `@Generable`, streaming, tools); and explain the production pitfalls (availability, context window, quotas, guardrails).

> **Sources:** Apple's WWDC26 session *"Build with the new Apple Foundation Model on Private Cloud Compute"* (319), Apple's *What's new in iOS 27* page, and the iOS 26 Foundation Models API. iOS 27 APIs were introduced in beta, so check names against the current docs before shipping.

---

## 1. Concept (10 min)

### Apple's AI stack in 2026

| Layer | API | Use it for |
|---|---|---|
| **Built-in perception** | Vision, VisionKit, Natural Language, Speech | OCR, barcodes, classification, language ID (the ML Kit classic equivalent) |
| **On-device LLM** | **Foundation Models**: `SystemLanguageModel` + `LanguageModelSession` | Summarize, extract, classify, rewrite, tool calling. Free, offline, private |
| **Server LLM** (iOS 27) | **`PrivateCloudComputeLanguageModel`** | Longer context, reasoning, harder tasks. Still private, with no API keys |
| **Other providers** (iOS 27) | Any model conforming to the **Language Model protocol** (for example Claude, Gemini) | Frontier capability through the **same session API** |
| **Your own models** | **Core AI** (new in iOS 27, replaces the older Core ML approach) | Custom models: memory-safe Swift, AOT compilation, PyTorch conversion |
| **System assistant** | **App Intents** (intent/entity schemas, View Annotations) | Siri acting on your app (Day 17) |
| **Quality** | **Evaluations framework** | Testing AI behavior beyond unit tests |

### On-device vs PCC: the table Apple showed at WWDC26

| | `SystemLanguageModel` (on-device) | `PrivateCloudComputeLanguageModel` |
|---|---|---|
| Network | **Offline** | Requires internet |
| Limits | **No request limits** | **Daily per-user quota** (counted per iCloud account; iCloud+ raises it) |
| Context window | **4,096** tokens (iOS 26) → **8,192** (iOS 27, newer devices) | **32,768** tokens |
| Reasoning | No | **Yes**: `.light`, `.moderate`, `.deep` |
| Cost to developer | Free | **Free**: no token costs, no API keys (eligibility: apps with under 2M downloads, apply on the developer site) |
| Devices | Apple Intelligence devices only | Apple Intelligence devices only |
| Privacy | Data never leaves the device | E2E privacy design: data is not stored, used only for the request, independently verified |

**The key point:** `@Generable` output and `Tool` calling **work the same on both**, so switching is a **one-line change** (`LanguageModelSession(model: PrivateCloudComputeLanguageModel())`).

### Foundation Models core concepts (memorize these)
- **`SystemLanguageModel.default.availability`** returns `.available` or `.unavailable(reason)`, with reasons `.deviceNotEligible`, `.appleIntelligenceNotEnabled` and `.modelNotReady`.
- **`LanguageModelSession`** is stateful. It keeps a **transcript** (conversation history), so one session equals one conversation. Pass **`instructions`** (the system prompt) when you create it.
- **`respond(to:)`** returns text. **`respond(to:generating: T.self)`** returns a typed `@Generable` value (**guided generation**, which uses constrained decoding, so the output always parses).
- **`streamResponse(to:generating:)`** streams **snapshots of `T.PartiallyGenerated`**, where every property is optional and fills in as tokens arrive. Apple streams *structured snapshots*, not raw tokens, which is ideal for SwiftUI.
- **`@Guide(description:, …)`** adds constraints such as `.count(3)`, `.range(1...5)` and `.anyOf([...])`.
- **`Tool` protocol:** `name`, `description`, `@Generable Arguments`, `call(arguments:)`. The model decides when to call it.
- **`prewarm()`** loads the model before the user's request to cut first-token latency.
- **`GenerationOptions`**: `temperature`, `sampling: .greedy` (deterministic), `maximumResponseTokens`.
- **Errors to handle** (`LanguageModelSession.GenerationError`): `exceededContextWindowSize`, `guardrailViolation`, `unsupportedLanguageOrLocale`, `rateLimited`, `concurrentRequests`.
- **Specialized use case:** `SystemLanguageModel(useCase: .contentTagging)` for tagging and classification.

### iOS 27 additions
- **Image input** (multimodal prompts), plus built-in vision tools such as OCR and barcode reading that the model can call on-device.
- **Dynamic Profiles:** swap the model, tools and instructions *within one continuous session*. For example, start on-device and escalate to PCC for a hard step.
- **Reasoning levels** on PCC (`ContextOptions(reasoningLevel:)`). Reasoning text uses tokens and counts toward the context window.
- **Tooling:** the `fm` CLI and Python SDK for prompt prototyping, Instruments for agentic apps, and an Xcode scheme option to **simulate availability and quota states**.

### Equivalents on other platforms
| 🍎 iOS | 🤖 Android | 🐦 Flutter | ⚛️ React Native |
|---|---|---|---|
| `SystemLanguageModel` | Gemini Nano (ML Kit Prompt API) | Platform channel | `@react-native-ai/apple` |
| `PrivateCloudComputeLanguageModel` | Firebase AI Logic (cloud Gemini) | `firebase_ai` | Vercel AI SDK cloud provider |
| `@Generable` / `@Guide` | `@Generable` / `@Guide` (ML Kit) | JSON schema | `generateObject` + zod |
| `Tool` protocol | Function calling / ML Kit Agentic API | `firebase_ai` function calling | AI SDK `tool()` |
| Core AI | LiteRT | flutter_gemma / TFLite plugins | ExecuTorch / llama.rn |
| App Intents | AppFunctions (Android MCP) | Native code | Native modules |

---

## 2. Code (10 min)

### A. Availability + prewarm + streaming structured output into SwiftUI

```swift
import SwiftUI
import FoundationModels

@Generable
struct RecipeIdeas {
    @Guide(description: "Short catchy title")
    var title: String
    @Guide(description: "Exactly 3 recipe ideas", .count(3))
    var ideas: [String]
}

@Observable @MainActor
final class RecipeViewModel {
    var partial: RecipeIdeas.PartiallyGenerated?
    var error: String?

    private let session = LanguageModelSession(
        instructions: "You suggest quick weeknight recipes. Be concise."
    )

    var isAvailable: Bool { SystemLanguageModel.default.availability == .available }

    func prepare() { session.prewarm() }                 // call when the screen appears

    func suggest(ingredients: String) async {
        do {
            let stream = session.streamResponse(
                to: "Ingredients: \(ingredients)",
                generating: RecipeIdeas.self
            )
            for try await snapshot in stream {
                partial = snapshot.content                // fields fill in progressively
            }
        } catch let e as LanguageModelSession.GenerationError {
            switch e {
            case .exceededContextWindowSize: error = "That's too long. Try fewer ingredients."
            case .guardrailViolation:        error = "Can't help with that request."
            default:                         error = "Something went wrong."
            }
        } catch { self.error = error.localizedDescription }
    }
}

struct RecipeView: View {
    @State private var vm = RecipeViewModel()
    var body: some View {
        VStack(alignment: .leading) {
            if let title = vm.partial?.title { Text(title).font(.headline) }
            ForEach(vm.partial?.ideas ?? [], id: \.self) { Text("• \($0)") }
        }
        .task { if vm.isAvailable { vm.prepare() } }
    }
}
```

### B. Tool calling: let the model look up app data

```swift
struct FindTasksTool: Tool {
    let name = "findTasks"
    let description = "Finds the user's tasks due on a given day"

    @Generable
    struct Arguments {
        @Guide(description: "Day in ISO-8601 format, e.g. 2026-11-16")
        var day: String
    }

    let store: TaskStore

    func call(arguments: Arguments) async throws -> String {
        let tasks = try await store.tasks(dueOn: arguments.day)
        return tasks.map(\.title).joined(separator: "\n")   // String is PromptRepresentable
    }
}

let session = LanguageModelSession(
    tools: [FindTasksTool(store: store)],
    instructions: "You help users plan their day using their tasks."
)
let answer = try await session.respond(to: "What should I focus on tomorrow?")
```

### C. iOS 27: on-device first, escalate to PCC, respect quotas

```swift
enum SummaryModel { case onDevice, pcc }

func pickModel(for text: String) -> SummaryModel {
    let onDevice = SystemLanguageModel.default
    let pcc = PrivateCloudComputeLanguageModel()

    // Rough token estimate (~4 chars per token). Leave room for instructions + output.
    let needsLongContext = text.count / 4 > onDevice.contextSize / 2

    if needsLongContext, pcc.isAvailable, !pcc.quotaUsage.isLimitReached { return .pcc }
    return .onDevice          // otherwise chunk the text and summarize on-device (map-reduce)
}

func summarize(_ text: String) async throws -> String {
    switch pickModel(for: text) {
    case .pcc:
        let session = LanguageModelSession(model: PrivateCloudComputeLanguageModel())
        return try await session.respond(
            to: "Summarize: \(text)",
            contextOptions: ContextOptions(reasoningLevel: .light)
        ).content
    case .onDevice:
        let session = LanguageModelSession(instructions: "Summarize in 3 bullets.")
        return try await session.respond(to: text).content   // chunk first if needed
    }
}
```

### D. Quota UI the way Apple recommends (persistent, no alerts)

```swift
struct QuotaBanner: View {
    let model = PrivateCloudComputeLanguageModel()
    var body: some View {
        if case .belowLimit(let info) = model.quotaUsage.status, info.isApproachingLimit {
            Text("Nearing today's limit").foregroundStyle(.orange)
        }
        if model.quotaUsage.isLimitReached {
            Text("Daily limit reached. Using on-device mode").foregroundStyle(.secondary)
        }
        if let suggestion = model.quotaUsage.limitIncreaseSuggestion {
            Button("Options") { suggestion.show() }
        }
    }
}
```

### Points to mention
- **One session = one conversation.** Create a new session for unrelated tasks, because the transcript uses up the context window.
- **One request at a time per session.** Check `session.isResponding` or you'll get `concurrentRequests`.
- **Test unavailable and quota states** with Xcode's *Simulate Apple Foundation Models Availability* scheme option.
- **Choose models and reasoning levels with data,** using the **Evaluations framework**, not by feel.

---

## 3. Mini system design (5 min)

**Prompt:** *"Design an AI journaling app for iOS with (1) a daily reflection summary and (2) an 'Ask my journal' chat that answers questions across years of entries."*

> **How to use this:** spend 5 minutes sketching your own answer, then read the model answer **out loud**. It follows the structure interviewers expect: **Clarify → Estimate → High-level design → Deep dives → Trade-offs → Failure modes & metrics → Wrap-up**. Then practice the follow-up questions; that's where senior candidates stand out.

<details>
<summary><b>🎤 Model answer (say it to the interviewer)</b></summary>

#### Step 1: Clarify requirements (~2 min)

| I ask | Assume |
|---|---|
| How sensitive is the data? | **Extremely**: mental health, relationships. Privacy is the product |
| Journal size? | ~1 entry/day × 300 words → **~100k words/year**; power users have 5+ years |
| Offline? | Daily summary must work offline; chat can degrade |
| Devices? | iPhone, iOS 27 target with an iOS 26 fallback; not all users have Apple Intelligence |
| Sync? | iCloud (CloudKit), E2E encrypted |

**Functional:** a summary + mood + themes for each day; natural-language Q&A over all entries with citations to dates.
**Non-functional:** nothing readable by us, fast, accurate citations, graceful degradation.

#### Step 2: Key decision and estimates
> "Privacy rules out third-party cloud LLMs by default. Apple gives me two private options: the **on-device model** (offline, unlimited, 4K–8K context) and **PCC** (32K context, reasoning, daily quota, still private, no keys or cost). Five years of journal ≈ **650k tokens**, far beyond any context window, so 'Ask my journal' needs **retrieval** (RAG), not 'stuff everything in the prompt'."

- Daily summary: one entry (~400 tokens) fits easily **on-device**.
- Chat: retrieve the top ~10 relevant entries (~4–6k tokens). That's tight on-device (8K on newer devices) and comfortable on PCC (32K).

#### Step 3: High-level architecture

```
┌────────────┐  save   ┌──────────────────────────────┐
│ Editor UI  │ ──────▶ │ SwiftData (E2E via CloudKit) │
└────────────┘         └──────┬───────────────────────┘
                              │ on save (background)
                              ▼
          ┌───────────────────────────────────────────┐
          │ Enrichment job (on-device FM)             │
          │  @Generable DaySummary(summary, mood,     │
          │             themes[3], people[])          │
          │  + embedding vector (on-device)           │
          └──────────────┬────────────────────────────┘
                         ▼
               Local index: summaries + vectors + themes
                         │
   "Ask my journal" ─────┤
                         ▼
     Retriever (vector + date/theme filters) → top-k entries
                         │
                         ▼
     Router ── fits on-device & simple? ─▶ SystemLanguageModel
            └─ long/complex & quota OK ─▶ PrivateCloudComputeLanguageModel (.light)
                         │
                         ▼
     @Generable Answer(text, citedDates[])  ─▶ Chat UI with tappable citations
```

#### Step 4: Deep dives
1. **Enrichment on save:**
   - Run guided generation into `@Generable DaySummary` on-device, using `SystemLanguageModel(useCase: .contentTagging)` for themes.
   - Store the results, so the chat uses compact **summaries** (cheap context) and fetches full entries only when needed.
   - Backfill old entries in a `BGProcessingTask` while charging.
2. **Retrieval:**
   - On-device embeddings for each entry plus metadata filters ("last summer" → a date range parsed with a small `@Generable QueryFilter`).
   - Hybrid search: vector plus keyword (names, places). Top-k = 8–12.
3. **Routing (iOS 27):**
   - On-device if the retrieved context fits within ~half of `contextSize`.
   - Otherwise PCC with `.light` reasoning ("How has my sleep changed over 3 years?").
   - Use **Dynamic Profiles** to escalate inside one session, so the chat keeps continuity.
4. **Citations and trust:** the answer is a `@Generable Answer` with `citedDates`. Validate that each cited date exists in the retrieved set and drop any that don't (prevents made-up citations). Tapping a citation opens that entry.
5. **Quota UX:** a persistent indicator, never alerts. When the limit is reached, fall back to on-device with a "shorter answers today" hint and the `limitIncreaseSuggestion` button.
6. **Fallback for non–Apple Intelligence devices:** there's no Apple model, so show plain keyword search and skip the AI summaries. (Optionally an opt-in third-party cloud model through the Language Model protocol, with explicit consent.)

#### Step 5: Trade-offs
| Choice | Why | Cost |
|---|---|---|
| RAG over long context | Years of data never fit; cheaper, faster | Retrieval misses → mitigate with summaries + hybrid search |
| On-device first, PCC escalation | Offline, unlimited, private | Two code paths. Same API keeps it small |
| Precomputed summaries | Fast chat, less context | Background compute and storage |
| No third-party cloud by default | Trust is the product | Lower quality on hard questions for some users |

#### Step 6: Failure modes, safety, metrics
- **Failures:** `exceededContextWindowSize` → reduce k and retry. `guardrailViolation` on sensitive topics → a compassionate fallback message. Model not ready → queue enrichment.
- **Safety:** detect crisis language → show support resources, not an AI answer.
- **Metrics:** % of questions answered with a valid citation, thumbs up/down, on-device vs PCC share, quota-hit rate, enrichment backlog. All **content-free** analytics.

#### Step 7: Wrap-up
> "Everything stays on Apple's private stack: on-device enrichment into structured summaries and embeddings on save, retrieval instead of long context, on-device answers by default with PCC escalation for long or complex questions, validated citations, and quota-aware UX. Privacy, offline use and zero marginal cost come from the platform rather than from our servers."

</details>

### 🔁 Likely follow-up questions

**F1. Why not just use PCC for everything? It's free and has 32K context.**

<details>
<summary>Answer</summary>

It needs a network connection, it has a **daily per-user quota** (heavy users would hit it), and it adds latency. On-device is offline, unlimited and instant for small tasks such as daily summaries. Apple frames them as **complementary**: on-device by default, PCC for tasks that need long context or reasoning. Also, PCC eligibility is limited (apps under 2M downloads), so a fast-growing app needs on-device to scale anyway.

</details>

**F2. How do you evaluate answer quality without reading users' journals?**

<details>
<summary>Answer</summary>

Build a **synthetic journal dataset** (generated personas across years) with labeled questions and expected cited dates. Run it through Apple's **Evaluations framework** for each model and reasoning level, and on every prompt change ("hill-climbing"). In production, use content-free signals: thumbs up/down, citation-tap rate, rephrase rate.

</details>

**F3. A user deletes an entry. What has to happen?**

<details>
<summary>Answer</summary>

Delete the entry, its summary, its embedding and its theme tags (cascade), and propagate through CloudKit. Any cached chat answers citing that date should be invalidated or marked. The "right to be forgotten" applies to derived AI data too. Mention this explicitly, because interviewers like it.

</details>

**F4. How would you build the same app in Flutter or React Native?**

<details>
<summary>Answer</summary>

The retrieval and storage logic can live in Dart or TS, but the models need native code: a Swift module wrapping Foundation Models and PCC (React Native AI's `apple()` provider covers on-device), and Gemini Nano on Android. Android has no free private cloud model, so the cloud path there is Firebase AI Logic with consent. Model this as a per-platform capability matrix behind one `JournalAI` interface, and accept that features differ by platform.

</details>

**F5. The on-device model refuses a legitimate entry about grief (guardrail false positive). What do you do?**

<details>
<summary>Answer</summary>

Catch `guardrailViolation` and show a gentle message rather than a generic error. Rephrase the instructions to make the summarization context explicit ("summarize the user's own journal entry neutrally"). Track the violation rate as a metric. Don't try to bypass safety systems; instead design the UX so a refusal never loses the user's data (the entry always saves; only the AI enrichment is skipped).

</details>

---

## 4. Mock interview questions (5 min)

Answer each out loud in about 60 seconds **before** opening the model answer.

### Q1. Explain the difference between `SystemLanguageModel`, `PrivateCloudComputeLanguageModel` and Core AI. When do you use each?

<details>
<summary><b>Model answer</b></summary>

- **`SystemLanguageModel`:** Apple's **on-device** LLM. Offline, no request limits, free, private. Context is 4K tokens (8K on iOS 27 with newer devices). Best for **narrow tasks**: summarize, extract, classify, rewrite, tag, simple tool use.
- **`PrivateCloudComputeLanguageModel`** (iOS 27): Apple's **server model** on PCC. **32K context**, **reasoning levels**, still privacy-preserving, no API keys and no token cost for eligible apps, but it needs a network and has a **daily per-user quota**. Use it for long documents and multi-step reasoning.
- **Core AI:** for running **my own model** (converted from PyTorch) when the task is domain-specific (a custom classifier, embeddings, audio) or when I need full control over the model.

**Key point:** the first two share one API (`@Generable` and `Tool` work identically), so I default to on-device and escalate to PCC with a one-line change.
</details>

### Q2. What is guided generation (`@Generable`), and why is it better than asking the model for JSON?

<details>
<summary><b>Model answer</b></summary>

`@Generable` turns a Swift type into a schema, and the framework uses **constrained decoding**: at each step the model can only produce tokens that keep the output valid for that schema. So the result **always parses** into my type. There's no "please return valid JSON", no stripping markdown fences, no `try? JSONDecoder` failures.

**Extras:**
- `@Guide` adds descriptions and constraints (`.count`, `.range`, `.anyOf`), which steer quality.
- **Streaming** gives `PartiallyGenerated` snapshots, so SwiftUI can render fields as they fill in.
- Property order matters: the model generates in declaration order, so put "reasoning" or "summary" fields before fields that depend on them.

**Still validate semantics** (dates exist, values make sense), because valid structure doesn't mean correct content. Android's ML Kit uses the same `@Generable` concept, which is a nice cross-platform point.
</details>

### Q3. How does tool calling work in Foundation Models, and what are the risks?

<details>
<summary><b>Model answer</b></summary>

**How it works:** I define a type conforming to `Tool` with a `name`, a natural-language `description`, `@Generable Arguments`, and `call(arguments:)`. I pass the tools to the `LanguageModelSession`. During a response, the model **decides** to call a tool, the framework generates **typed arguments** (guaranteed to parse), runs my `call`, feeds the output back into the transcript, and the model continues. It can call several tools, even in parallel.

**Risks and mitigations:**
- **Context budget:** tool schemas, calls and results all use tokens in the 4K–8K window. Keep descriptions short and return compact results.
- **Wrong or unintended calls:** never let a tool perform destructive actions (delete, pay, send) without **user confirmation** in the UI.
- **Prompt injection:** tool results containing untrusted text (web pages, emails) can try to redirect the model. Treat them as data, constrain what tools can do, and validate.
- **Errors:** thrown errors surface as `ToolCallError`. Handle them and return a helpful message.
- **Latency:** each tool round-trip adds time, so show progress.
</details>

### Q4. Your Foundation Models feature crashes or fails for some users. What are the likely causes and how do you handle them?

<details>
<summary><b>Model answer</b></summary>

- **Unavailable model:** the device isn't eligible, Apple Intelligence is off, or the model isn't ready. Check `availability` before showing the feature and handle each reason (for example, suggest enabling Apple Intelligence; show "preparing" for `modelNotReady`).
- **`exceededContextWindowSize`:** the transcript grew too long or the input is too big. Start new sessions per task, summarize history, chunk inputs, and check `contextSize`.
- **`concurrentRequests`:** a second request while the session is responding. Disable the button while `session.isResponding`.
- **`guardrailViolation`:** show a graceful message and never lose user data.
- **`unsupportedLanguageOrLocale`:** check supported languages and fall back.
- **PCC:** network failures and **quota limits** (`quotaUsage`). Show persistent UI and fall back to on-device.

**Testing:** Xcode's *Simulate Apple Foundation Models Availability* option covers unavailable and quota states, plus unit tests that use a protocol-wrapped model fake.
</details>

### Q5. How would you minimize latency for a Foundation Models feature?

<details>
<summary><b>Model answer</b></summary>

1. **`prewarm()`** the session when the screen appears or the user starts typing, to load the model early.
2. **Stream** with `streamResponse` so the UI updates immediately; perceived latency matters most.
3. **Keep prompts short:** concise instructions, compact tool descriptions and results, a fresh session per unrelated task (a long transcript means more prefill).
4. **Limit output:** `maximumResponseTokens`, and ask for exactly the fields you need.
5. **Order `@Generable` properties** so the most useful field appears first in the stream.
6. **Choose the model wisely:** on-device has no network latency. PCC reasoning (`.deep`) is slow, so use the lowest reasoning level that passes evals.
7. **Precompute in the background** (summaries, tags) instead of generating on demand.
8. **Measure** with Instruments (which has agentic-app profiling in Xcode 27).
</details>

### Q6. *(Cross-platform)* A React Native team wants to use Apple's on-device model on iOS and something comparable on Android. How do they architect it?

<details>
<summary><b>Model answer</b></summary>

- **iOS:** Callstack's **React Native AI** `apple()` provider exposes Foundation Models through the **Vercel AI SDK** (iOS 26+), so `generateText({ model: apple(), ... })` just works. For PCC or iOS 27 features, write a small **Swift native module** until libraries support them.
- **Android:** Gemini Nano through a Kotlin native module wrapping the ML Kit Prompt API, or an open model through **llama.rn / ExecuTorch** where Nano isn't available.
- **Architecture:**
  - One TypeScript interface (or AI SDK providers) chosen by a **capability check** at startup.
  - Shared prompts and **zod schemas**, mapped to `@Generable` on iOS and ML Kit structured output on Android, so each side returns the same shape.
  - A cloud fallback through the AI SDK, behind the team's backend.
- **Gotchas:** an Expo **development build** is required, AI SDK versions need pinning, and behavior differs between Apple's and Google's models, so run the **same eval set on both**.
</details>

---

**Takeaway:** On-device `SystemLanguageModel` by default (offline, unlimited, private), **PCC** for long context and reasoning (32K, quota, still private, free for eligible apps), **Core AI** for your own models. `@Generable` and `Tool` work identically across them, so design one code path with smart routing, availability checks and quota-aware UX.

**Tomorrow (Day 5):** 🐦 Flutter AI stack: `firebase_ai`, the Flutter AI Toolkit, flutter_gemma, GenUI and platform channels to the OS models.
