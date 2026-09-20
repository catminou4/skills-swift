# SpeechAnalyzer Patterns

Use this reference when implementing iOS 26+ speech-to-text with
`SpeechAnalyzer`, `SpeechTranscriber`, `DictationTranscriber`, `SpeechDetector`,
or `AssetInventory`.

## Contents

- [Choosing Modules](#choosing-modules)
- [Preparing Assets](#preparing-assets)
- [Transcribing Files](#transcribing-files)
- [Live Audio](#live-audio)
- [Input Providers (iOS 27+)](#input-providers-ios-27)
- [Handling Results](#handling-results)
- [Finishing Sessions](#finishing-sessions)
- [References](#references)

## Choosing Modules

Prefer `SpeechTranscriber` for the newer general-purpose on-device model.
Before building UI around it, check both device and locale support:

```swift
guard SpeechTranscriber.isAvailable,
      let locale = SpeechTranscriber.supportedLocale(equivalentTo: Locale.current)
else {
    // Disable the feature or try DictationTranscriber for compatible devices/locales.
    return
}
```

Use documented presets only:

- `.transcription` for basic accurate transcription.
- `.transcriptionWithAlternatives` for editing suggestions.
- `.timeIndexedTranscriptionWithAlternatives` for audio-time metadata plus alternatives.
- `.progressiveTranscription` for low-latency live UI updates.
- `.timeIndexedProgressiveTranscription` for live UI updates with time ranges.

Use `DictationTranscriber` when `SpeechTranscriber` is unavailable and the app
can accept dictation-model behavior. Add `SpeechDetector` only with a
transcriber module, and only when voice activity detection is worth the risk of
dropping speech-like audio.

## Preparing Assets

`SpeechAnalyzer` modules require model assets. The system installs and shares
them outside the app bundle, but the app must request installation for the
module configuration it plans to use.

```swift
let transcriber = SpeechTranscriber(locale: locale, preset: .transcription)

if let request = try await AssetInventory.assetInstallationRequest(
    supporting: [transcriber]
) {
    try await request.downloadAndInstall()
}
```

For language pickers, use `installedLocales`, `supportedLocales`, and
`AssetInventory.status(forModules:)` to distinguish installed, downloadable,
and unsupported choices. The app has a limited number of locale reservations;
release unused reservations with `AssetInventory.release(reservedLocale:)`.

## Transcribing Files

For files, let the analyzer convert the file to a compatible format and finish
the session after the file is consumed.

```swift
func transcribeFile(at url: URL, locale: Locale) async throws -> AttributedString {
    guard let supportedLocale = SpeechTranscriber.supportedLocale(equivalentTo: locale) else {
        throw SpeechError.unsupportedLocale
    }

    let transcriber = SpeechTranscriber(
        locale: supportedLocale,
        preset: .transcription
    )

    if let request = try await AssetInventory.assetInstallationRequest(
        supporting: [transcriber]
    ) {
        try await request.downloadAndInstall()
    }

    let analyzer = SpeechAnalyzer(modules: [transcriber])
    async let transcript = transcriber.results.reduce(into: AttributedString()) {
        text, result in
        text.append(result.text)
    }

    let file = try AVAudioFile(forReading: url)
    let lastSampleTime = try await analyzer.analyzeSequence(from: file)
    if let lastSampleTime {
        try await analyzer.finalizeAndFinish(through: lastSampleTime)
    } else {
        try analyzer.cancelAndFinishNow()
    }

    return try await transcript
}
```

## Live Audio

For live audio, create an `AsyncStream<AnalyzerInput>`, convert microphone
buffers to the analyzer-compatible format, yield them, and consume results in a
separate task.

```swift
let transcriber = SpeechTranscriber(
    locale: locale,
    preset: .timeIndexedProgressiveTranscription
)
let analyzer = SpeechAnalyzer(modules: [transcriber])
let audioFormat = await SpeechAnalyzer.bestAvailableAudioFormat(
    compatibleWith: [transcriber]
)
let (inputSequence, inputBuilder) = AsyncStream.makeStream(of: AnalyzerInput.self)

// In the audio-engine tap, convert each AVAudioPCMBuffer to audioFormat first.
inputBuilder.yield(AnalyzerInput(buffer: convertedBuffer))
```

Use `AVAudioConverter` or an existing project audio pipeline for the conversion.
Do not feed arbitrary input-node formats directly unless they already match a
compatible analyzer format.

## Input Providers (iOS 27+)

iOS 27 adds providers that hand the analyzer an `AsyncSequence` of
`AnalyzerInput` objects in a compatible format.

```swift
import Speech
import AVFoundation

let transcriber = SpeechTranscriber(
    locale: locale,
    preset: .progressiveTranscription
)
let analyzer = SpeechAnalyzer(modules: [transcriber])

// Microphone or another capture device: configures and starts a new
// AVCaptureSession for the given device.
let provider = try await CaptureInputSequenceProvider.providerWithSession(
    from: microphone,                    // AVCaptureDevice
    compatibleWith: [transcriber]
)
try await analyzer.start(inputSequence: provider.analyzerInputs)
```

```swift
// Recorded file or AVAsset track.
let asset = AVURLAsset(url: fileURL)
let fileProvider = try await AssetInputSequenceProvider.provider(
    from: asset,
    compatibleWith: [transcriber]
)
try await analyzer.start(inputSequence: fileProvider.analyzerInputs)
```

Keep using `AVAudioEngine` when you need tap-level control; convert buffers
yourself with `AnalyzerInputConverter`, which returns `[AnalyzerInput]` batches:

```swift
let converter = try await AnalyzerInputConverter.converter(
    compatibleWith: [transcriber]
)
// In the audio tap:
for input in try converter.convert(buffer, at: sampleTime) {
    inputBuilder.yield(input)
}
// When the source ends, flush pending conversions:
for input in try converter.flush() {
    inputBuilder.yield(input)
}
```

`provider.analyzerInputs` terminates when the capture session and provider are
deallocated. For sessions that must outlive predefined resource limits, create
the analyzer with `SpeechAnalyzer.Options(priority:modelRetention:ignoresResourceLimits:)`
(iOS 27+).

## Handling Results

`SpeechTranscriber.Result.text` is an `AttributedString`. Time-indexed presets
include audio time range attributes that can drive playback highlighting.

When using progressive presets, volatile results may be replaced by later final
results. Keep volatile display state separate so the UI does not duplicate text.

```swift
for try await result in transcriber.results {
    if result.isFinal {
        volatileTranscript = AttributedString()
        finalizedTranscript.append(result.text)
    } else {
        volatileTranscript = result.text
    }
}
```

## Finishing Sessions

The analyzer can only analyze one input sequence at a time. Ending your stream
does not finish the analyzer session; call a finish or cancel method.

Use:

- `finalizeAndFinish(through:)` after `analyzeSequence(_:)` returns a final sample time.
- `finalizeAndFinishThroughEndOfInput()` after autonomous `start(inputSequence:)`.
- `cancelAndFinishNow()` for immediate cancellation.

After the session finishes, result streams terminate and most analyzer methods
no longer accept new work. Create a new analyzer for a new finished session.

## References

- [SpeechAnalyzer](https://sosumi.ai/documentation/speech/speechanalyzer)
- [SpeechTranscriber](https://sosumi.ai/documentation/speech/speechtranscriber)
- [SpeechTranscriber.Preset](https://sosumi.ai/documentation/speech/speechtranscriber/preset)
- [DictationTranscriber](https://sosumi.ai/documentation/speech/dictationtranscriber)
- [SpeechDetector](https://sosumi.ai/documentation/speech/speechdetector)
- [AssetInventory](https://sosumi.ai/documentation/speech/assetinventory)
- [CaptureInputSequenceProvider](https://sosumi.ai/documentation/speech/captureinputsequenceprovider) (iOS 27+)
- [AssetInputSequenceProvider](https://sosumi.ai/documentation/speech/assetinputsequenceprovider) (iOS 27+)
- [AnalyzerInputConverter](https://sosumi.ai/documentation/speech/analyzerinputconverter) (iOS 27+)
- [WWDC25: Bring advanced speech-to-text to your app with SpeechAnalyzer](https://sosumi.ai/videos/play/wwdc2025/277)
