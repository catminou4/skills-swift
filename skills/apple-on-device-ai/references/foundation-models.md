# Foundation Models API Reference

Complete reference for Apple's Foundation Models framework (iOS 26+ / macOS 26+;
iOS 27 additions are tagged). On-device language model optimized for Apple
Silicon. No app-managed API key, model hosting, or network round trip for
generation; still handle Apple Intelligence and system model asset availability.

## Contents

- [Framework Overview](#framework-overview)
- [Availability Checking](#availability-checking)
- [Use Cases](#use-cases)
- [Session Management](#session-management)
- [Choosing a Model (iOS 27+)](#choosing-a-model-ios-27)
- [Dynamic Profiles (iOS 27+)](#dynamic-profiles-ios-27)
- [Generating Responses](#generating-responses)
- [Multimodal Prompts (iOS 27+)](#multimodal-prompts-ios-27)
- [Structured Output with `@Generable`](#structured-output-with-generable)
- [Tool Calling](#tool-calling)
- [Error Handling](#error-handling)
- [Generation Options](#generation-options)
- [Safety and Guardrails](#safety-and-guardrails)
- [Custom Adapters](#custom-adapters)
- [Context Management](#context-management)
- [Token Usage (iOS 27+)](#token-usage-ios-27)
- [Serialized Model Access](#serialized-model-access)
- [Prompt Design Best Practices](#prompt-design-best-practices)
- [Feedback](#feedback)

## Framework Overview

- On-device language model optimized for Apple Silicon
- Context window: limited total token budget (input + output combined); check
  `SystemLanguageModel.default.contextSize` for the current limit
- Prefer `SystemLanguageModel.default.supportsLocale(_:)` before generation;
  use `supportedLanguages` only when listing broad language support
- Capabilities: Summarization, entity extraction, text understanding, short
  dialog, creative content, content tagging
- Limitations: Not suited for complex math, code generation, or factual accuracy

iOS 27 changes the framework's shape:

- Rebuilt on-device model with better logic and tool calling, plus refined
  guardrails; prompt for the model version the device ships (see
  `Updating prompts for new model versions` in the docs).
- `LanguageModel` protocol opens sessions to Private Cloud Compute, Core AI,
  MLX, and third-party provider models.
- Multimodal image attachments, dynamic profiles, token-usage reporting, and
  system-provided tools (`OCRTool`, `BarcodeReaderTool` in Vision; a
  Spotlight-backed search tool).
- The framework core and a "Foundation Models framework utilities" package are
  open source; the new Evaluations framework tests model features, and macOS 27
  adds the `fm` CLI and `apple_fm_sdk` Python package.

### SystemLanguageModel Properties

- `contextSize`: Returns the model's maximum context window in tokens
- `supportedLanguages`: `Set<Locale.Language>` values the model supports
- `supportsLocale(_ locale: Locale) -> Bool`: Preferred locale check before generating because it accounts for fallbacks

## Availability Checking

Always check before using. Never crash on unavailability.

```swift
import FoundationModels

// Quick boolean check
if SystemLanguageModel.default.isAvailable {
    // Proceed
}

// Detailed availability
switch SystemLanguageModel.default.availability {
case .available:
    let candidates = [Locale.current] + Locale.preferredLanguages.map(Locale.init(identifier:))
    guard let locale = candidates.first(where: SystemLanguageModel.default.supportsLocale) else {
        // Route to fallback UI before generating
        break
    }
    // Proceed with model usage
case .unavailable(.appleIntelligenceNotEnabled):
    // Guide user to Settings > Apple Intelligence
case .unavailable(.modelNotReady):
    // System model assets are downloading or unavailable for other system reasons
case .unavailable(.deviceNotEligible):
    // Device cannot run Apple Intelligence
default:
    // Graceful fallback for unknown or future unavailable reasons
}
```

## Use Cases

Foundation Models supports specialized use cases:

```swift
// General purpose (default)
let model = SystemLanguageModel(useCase: .general, guardrails: .default)

// Content tagging (optimized for categorization)
let model = SystemLanguageModel(useCase: .contentTagging, guardrails: .default)
```

## Session Management

### Creating Sessions

```swift
// Basic session (uses SystemLanguageModel.default)
let session = LanguageModelSession()

// Session with system instructions
let session = LanguageModelSession {
    "You are a helpful cooking assistant."
    "Focus on quick, healthy recipes."
}

// Session with tools
let session = LanguageModelSession(
    tools: [weatherTool, recipeTool]
) {
    "You are a helpful assistant with access to tools."
}

// Session with specific model
let model = SystemLanguageModel(useCase: .general, guardrails: .default)
let session = LanguageModelSession(model: model, tools: []) {
    "You are a helpful assistant."
}

// Session with the Private Cloud Compute model (iOS 27+)
let session = LanguageModelSession(model: PrivateCloudComputeLanguageModel())
```

### Session Rules

1. Sessions are stateful. Multi-turn conversations maintain context automatically.
2. One request at a time per session. Check `session.isResponding` before new
   requests.
3. Prewarm with `session.prewarm()` before user interaction for faster first
   response.
4. Save and restore transcripts for session continuity:
   `LanguageModelSession(model: model, tools: [], transcript: savedTranscript)`.

### Prewarming

```swift
// Prewarm before user interaction
session.prewarm()

// Prewarm with a prompt prefix for faster specific responses
session.prewarm(promptPrefix: Prompt("Summarize the following text:"))
```

## Choosing a Model (iOS 27+)

`LanguageModelSession(model:)` accepts any type conforming to the new
`LanguageModel` protocol, not just `SystemLanguageModel`.

```swift
// On-device model (default)
let session = LanguageModelSession(model: SystemLanguageModel.default)

// Private Cloud Compute model — the server model behind Apple Intelligence
let pccModel = PrivateCloudComputeLanguageModel()
switch pccModel.availability {
case .available:
    let session = LanguageModelSession(model: pccModel)
    // Deep reasoning costs extra compute; .light is cheaper
    let response = try await session.respond(
        to: prompt,
        contextOptions: ContextOptions(reasoningLevel: .deep)
    )
default:
    break // Fall back to the on-device model
}
```

Private Cloud Compute specifics:

- 32,000-token context window and a `reasoningLevel` (`ContextOptions`)
- Requires the managed entitlement `com.apple.developer.private-cloud-compute`
- No API keys or account setup; check `quotaUsage` for the usage quota
- Makes Foundation Models usable on watchOS 27
- Conforms to `Observable`; `isAvailable` / `availability` gate the session

Open-source `CoreAILanguageModel` and `MLXLanguageModel` adapters let local
models back a session, and third-party providers ship `LanguageModel` packages
over Swift Package Manager. Third-party server models authenticate with OAuth
plus Keychain — never embed private keys in the app binary — and are billed
per token, so monitor `usage`.

## Dynamic Profiles (iOS 27+)

`LanguageModelSession.DynamicProfile` switches a session between complete
configurations as app state changes. Conform to the protocol and return a
`Profile` from `body`; create the session with
`LanguageModelSession(profile:)`:

```swift
struct AssistantProfile: LanguageModelSession.DynamicProfile {
    var body: some DynamicProfile {
        Profile {
            Instructions { "You are a concise assistant." }
            SearchTool()
        }
        .model(PrivateCloudComputeLanguageModel())
        .reasoningLevel(.deep)
        .toolCallingMode(.automatic)
        .historyTransform { entries in trimHistory(entries) }
    }
}

let session = LanguageModelSession(profile: AssistantProfile())
```

Profile modifiers include `model`, `temperature`, `samplingMode`,
`reasoningLevel`, `maximumResponseTokens`, `toolCallingMode`, and
`historyTransform`; hooks such as `onActivate`, `onDeactivate`, `onPrompt`,
`onReasoning`, `onResponse`, `onToolCall`, and `onToolOutput` observe the
lifecycle. A conditional `body` (via `DynamicProfileBuilder`) selects between
profiles at runtime.

## Generating Responses

### Plain Text

```swift
// Simple text response
let response = try await session.respond(to: "Summarize this article: \(text)")
print(response.content) // String

// With generation options
let options = GenerationOptions(
    sampling: .random(top: 40),
    temperature: 0.7,
    maximumResponseTokens: 512
)
let response = try await session.respond(to: prompt, options: options)
```

### Streaming Text

```swift
let stream = session.streamResponse(to: "Tell me a story")
for try await snapshot in stream {
    print(snapshot.content, terminator: "")
}

// Or collect the full response
let response = try await stream.collect()
```

## Multimodal Prompts (iOS 27+)

The on-device model accepts images alongside text. Insert `Attachment` inside a
`Prompt` or `Instructions` builder; attach a `label(_:)` so the model (and
tools) can refer to a specific image.

```swift
let response = try await session.respond {
    "Describe this image:"
    Attachment(image)
        .label("flyer")
}
```

- `Attachment` accepts `UIImage`, `NSImage`, `CGImage`, Core Image types,
  `CVPixelBuffer`, and image file URLs (`init(imageURL:orientation:)`).
- Any size and aspect ratio works — no cropping or padding — but larger images
  consume more context tokens and add latency.
- Give tool arguments image inputs with `ImageReference`: declare
  `var image: ImageReference` in the tool's `@Generable` `Arguments`, then
  resolve it during the call with `image.resolved(in: history)` where
  `history` comes from `@SessionProperty(\.history)`.

## Structured Output with `@Generable`

The `@Generable` macro creates compile-time JSON schemas for type-safe output.

### Basic Usage

```swift
@Generable
struct Recipe {
    @Guide(description: "The name of the recipe")
    var name: String

    @Guide(description: "A brief description of the dish")
    var summary: String

    @Guide(description: "Cooking steps", .count(3))
    var steps: [String]

    @Guide(description: "Prep time in minutes", .range(1...120))
    var prepTime: Int
}

let response = try await session.respond(
    to: "Suggest a quick pasta recipe",
    generating: Recipe.self
)
let recipe = response.content
print(recipe.name)
print(recipe.steps)
```

### Supported Types for `@Generable` Properties

- `String`
- `Int`, `Double`, `Float`
- `Bool`
- `[Element]` where Element is Generable or a supported scalar
- `Optional<T>` where T is Generable or a supported scalar
- Other `@Generable` structs (nested)
- Enums conforming to `@Generable`

### `@Guide` Constraints

```swift
@Generable
struct ProductReview {
    @Guide(description: "Product name")
    var product: String

    @Guide(description: "Rating", .range(1...5))
    var rating: Int

    @Guide(description: "Sentiment", .anyOf(["positive", "neutral", "negative"]))
    var sentiment: String

    @Guide(description: "Key themes", .count(3))
    var themes: [String]

    @Guide(description: "Summary in one sentence", .pattern(/^[A-Z].*\.$/))
    var summary: String

    @Guide(description: "Always English", .constant("en"))
    var language: String
}
```

Complete constraint list:

| Constraint | Type | Purpose |
|---|---|---|
| `description:` | All | Natural language hint for generation |
| `.anyOf([values])` | String | Restrict to enumerated values |
| `.count(n)` | Array | Fixed array length |
| `.minimumCount(n)` | Array | Minimum array length |
| `.maximumCount(n)` | Array | Maximum array length |
| `.range(min...max)` | Numeric | Closed numeric range |
| `.minimum(n)` | Numeric | Lower bound |
| `.maximum(n)` | Numeric | Upper bound |
| `.constant(value)` | String | Always returns this value |
| `.pattern(regex)` | String | Regex format enforcement |
| `.element(guide)` | Array | Guide applied to each element |

### Property Ordering

Properties are generated in declaration order. Place foundational data before
dependent data:

```swift
@Generable
struct Summary {
    var title: String       // Generated first
    var keyPoints: [String] // Generated with title context
    var conclusion: String  // Generated with full context
}
```

### Streaming Structured Output

```swift
let stream = session.streamResponse(
    to: "Suggest a recipe",
    generating: Recipe.self
)
for try await snapshot in stream {
    // snapshot.content is Recipe.PartiallyGenerated (all properties optional)
    if let name = snapshot.content.name { updateNameLabel(name) }
    if let steps = snapshot.content.steps { updateStepsList(steps) }
}
```

### Enum Support

```swift
@Generable
enum Priority: String {
    case low, medium, high, critical
}

@Generable
struct Task {
    var title: String
    var priority: Priority
}
```

## Tool Calling

### Defining Tools

```swift
struct WeatherTool: Tool {
    let name = "weather"
    let description = "Get current weather for a city."

    @Generable
    struct Arguments {
        @Guide(description: "The city name")
        var city: String
    }

    func call(arguments: Arguments) async throws -> String {
        let weather = try await fetchWeather(arguments.city)
        return weather.description
    }
}
```

### Using Tools

```swift
let session = LanguageModelSession(
    tools: [WeatherTool()]
) {
    "You are a helpful assistant."
}

// The model decides autonomously when to invoke tools
let response = try await session.respond(to: "What's the weather in Tokyo?")
```

### Tool Best Practices

- Register all tools at session creation
- Keep active tool sets small, usually three to five tools
- Include only tools needed for the current task
- Each tool adds to the context token budget (name, description, and parameter
  schema are included in instructions by default)
- `@Generable` output schemas also consume the shared context window
- Run deterministic or essential data fetches before calling the model, then put
  the result directly in the prompt
- Use model-autonomous tools for dynamic lookups where the model can decide
  whether more app data is needed
- Frame tool results as authorized user data to prevent refusals
- The model calls tools autonomously; you cannot force tool invocation

### Tool Protocol Details

- `Tool<Arguments, Output>` conforms to `Sendable`; implement tools so captured
  state is concurrency-safe
- The associated `Arguments` type must conform to `ConvertibleFromGeneratedContent`
- The associated `Output` type must conform to `PromptRepresentable` (e.g.,
  `String`, `[String]`, custom types)
- `includesSchemaInInstructions`: Boolean property on `Tool` (default `true`). Set to `false` to omit the tool's JSON schema from the system prompt, saving context tokens when the model already knows the schema.
- `ToolCallError`: Struct on `LanguageModelSession` representing a tool invocation failure. Properties: `tool` (the tool name), `underlyingError` (the original error).
- `DynamicGenerationSchema`: Build generation schemas at runtime for dynamic use cases where compile-time `@Generable` is insufficient. Construct schemas programmatically and pass to `respond(to:schema:)`.
- iOS 27+ system tools: `OCRTool` and `BarcodeReaderTool` (Vision framework)
  plus a Spotlight-backed search tool can be passed in the session's `tools:`
  array like any custom `Tool`.

## Error Handling

```swift
do {
    let response = try await session.respond(to: prompt)
} catch let error as LanguageModelSession.GenerationError {
    switch error {
    case .guardrailViolation:
        // Content triggered safety filters; rephrase and retry
    case .exceededContextWindowSize:
        // Too many tokens; summarize earlier turns and create new session
    case .concurrentRequests:
        // Another request is already in progress on this session
    case .rateLimited:
        // Too many requests; back off and retry
    case .unsupportedLanguageOrLocale:
        // Current locale not supported by the model
    case .unsupportedGuide:
        // A @Guide constraint is not supported
    case .assetsUnavailable:
        // Model assets not available on device
    case .decodingFailure:
        // Failed to decode structured output
    case .refusal(let refusal, _):
        // Model refused the request
        let explanation = try await refusal.explanation.content
        print("Refused: \(explanation)")
    default: break
    }
}
```

## Generation Options

```swift
let options = GenerationOptions(
    sampling: .greedy,              // Deterministic output
    temperature: nil,               // Use default
    maximumResponseTokens: 256      // Limit response length
)

// Random sampling with top-k
let options = GenerationOptions(
    sampling: .random(top: 40),
    temperature: 0.7
)

// Random sampling with probability threshold
let options = GenerationOptions(
    sampling: .random(probabilityThreshold: 0.9)
)

// iOS 27+: control whether the model may call tools
let options = GenerationOptions(toolCallingMode: .automatic)
```

Sampling modes accept an optional `seed` parameter for reproducible output:
`.random(top: 40, seed: 42)`, `.random(probabilityThreshold: 0.9, seed: 42)`.

`respond(to:)` and `streamResponse(to:)` also take a `contextOptions:`
parameter (iOS 27+). `ContextOptions(reasoningLevel:)` sets the model's
reasoning budget (`ContextOptions.ReasoningLevel`), and
`includeSchemaInPrompt` controls schema injection into the prompt.

## Safety and Guardrails

### Guardrail Types

```swift
// Default guardrails (recommended)
let model = SystemLanguageModel(useCase: .general, guardrails: .default)

// Permissive content transformations (for text rewriting tasks)
let model = SystemLanguageModel(
    useCase: .general,
    guardrails: .permissiveContentTransformations
)
```

### Safety Rules

- Guardrails are always enforced and cannot be disabled
- Instructions take precedence over user prompts
- Never include untrusted user content in instructions
- Provide curated selections over free-form input when possible
- Guardrails can produce false positives; handle gracefully
- Frame tool results as authorized user data

## Custom Adapters

Load fine-tuned LoRA adapters for specialized model behavior:

```swift
// Requires com.apple.developer.foundation-model-adapter entitlement
let adapter = try SystemLanguageModel.Adapter(name: "my-adapter")
try await adapter.compile()

let model = SystemLanguageModel(adapter: adapter, guardrails: .default)
let session = LanguageModelSession(model: model)
let response = try await session.respond(to: "Generate styled text")
```

### Adapter Management

```swift
// Check compatible adapters
let ids = SystemLanguageModel.Adapter.compatibleAdapterIdentifiers(name: "my-adapter")

// Remove obsolete adapters
try SystemLanguageModel.Adapter.removeObsoleteAdapters()
```

## Context Management

When conversations grow long:

1. Monitor token usage against `SystemLanguageModel.default.contextSize`
2. Use `SystemLanguageModel.default.tokenCount(for:)` to estimate usage
3. Summarize earlier turns into new session instructions
4. Create fresh sessions with summary context rather than overflowing

## Token Usage (iOS 27+)

`session.usage` accumulates usage across the session and each `response.usage`
reports that turn:

- `usage.input.totalTokenCount` and `usage.input.cachedTokenCount`
- `usage.output.totalTokenCount` and `usage.output.reasoningTokenCount`
- `usage.metadata` for provider-reported extras

Use it to budget PCC quotas and per-token billing on third-party models.

```swift
if transcript.estimatedTokenCount > 3000 {
    let summary = try await summarizeSession(session)
    session = LanguageModelSession {
        "Previous conversation summary: \(summary)"
        "Continue helping the user."
    }
}
```

## Serialized Model Access

When multiple parts of an app need the model:

```swift
actor FoundationModelCoordinator {
    private var session: LanguageModelSession?

    func respond(to prompt: String) async throws -> String {
        if session == nil {
            session = LanguageModelSession()
        }
        guard let activeSession = session else {
            throw FoundationModelError.sessionUnavailable
        }
        let response = try await activeSession.respond(to: prompt)
        return response.content
    }
}
```

Serialize all Foundation Model access through a single coordinator to prevent
Neural Engine contention.

## Prompt Design Best Practices

1. **Be concise.** The context window covers both input and output tokens.
   Check `SystemLanguageModel.default.contextSize` for the current limit.
2. **Use bracketed placeholders** in instructions: `[descriptive example]`.
3. **Use "DO NOT" in all caps** for behavioral prohibitions.
4. **Provide up to 5 few-shot examples** for consistent output.
5. **Use length qualifiers:** "in a few words", "in three sentences".
6. **Estimate token usage** with `SystemLanguageModel.default.tokenCount(for:)`
   to avoid exceeding the context window.

## Feedback

Log feedback for model improvement:

```swift
let data = session.logFeedbackAttachment(
    sentiment: .negative,
    issues: [
        LanguageModelFeedback.Issue(
            category: .didNotFollowInstructions,
            explanation: "Ignored the word count constraint"
        )
    ],
    desiredOutput: nil
)
```

Issue categories: `.didNotFollowInstructions`, `.incorrect`,
`.stereotypeOrBias`, `.suggestiveOrSexual`, `.tooVerbose`,
`.triggeredGuardrailUnexpectedly`, `.unhelpful`, `.vulgarOrOffensive`.
