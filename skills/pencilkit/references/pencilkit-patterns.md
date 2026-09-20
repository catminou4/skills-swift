# PencilKit Extended Patterns

Overflow reference for the `pencilkit` skill. Contains advanced patterns
that exceed the main skill file's scope.

## Contents

- [Tool Picker Observer Pattern](#tool-picker-observer-pattern)
- [Custom Tool Picker Items](#custom-tool-picker-items)
- [Canvas View Delegate Lifecycle](#canvas-view-delegate-lifecycle)
- [Undo/Redo Support](#undoredo-support)
- [Thumbnail Generation](#thumbnail-generation)
- [Drawing Comparison and Scoring](#drawing-comparison-and-scoring)
- [Constructing Strokes Programmatically](#constructing-strokes-programmatically)
- [Content Version Management](#content-version-management)
- [Stroke Slicing and Erasing (iOS 27+)](#stroke-slicing-and-erasing-ios-27)
- [Handwriting Recognition (iOS 27+)](#handwriting-recognition-ios-27)
- [Advanced SwiftUI Wrapper](#advanced-swiftui-wrapper)

## Tool Picker Observer Pattern

Observe tool picker changes to update custom UI or track tool usage.

```swift
import PencilKit

class DrawingController: UIViewController, PKToolPickerObserver {
    let canvasView = PKCanvasView()
    let toolPicker = PKToolPicker()

    override func viewDidLoad() {
        super.viewDidLoad()
        toolPicker.addObserver(self)
        toolPicker.addObserver(canvasView)
    }

    func toolPickerSelectedToolItemDidChange(_ toolPicker: PKToolPicker) {
        let item = toolPicker.selectedToolItem
        print("Selected tool: \(item.identifier)")
    }

    func toolPickerVisibilityDidChange(_ toolPicker: PKToolPicker) {
        print("Picker visible: \(toolPicker.isVisible)")
    }

    func toolPickerFramesObscuredDidChange(_ toolPicker: PKToolPicker) {
        let obscured = toolPicker.frameObscured(in: view)
        // Adjust content insets to avoid overlap
        canvasView.contentInset.bottom = obscured.height
    }
}
```

## Custom Tool Picker Items

Create custom tools with unique behaviors and icons. Custom tool picker items
require iOS/iPadOS 18+, Mac Catalyst 18+, or visionOS 2+.

```swift
var customConfig = PKToolPickerCustomItem.Configuration(
    identifier: "com.app.highlighter",
    name: "Highlighter"
)
customConfig.defaultColor = .yellow
customConfig.allowsColorSelection = true
customConfig.defaultWidth = 20
customConfig.widthVariants = [
    10: UIImage(systemName: "line.diagonal")!,
    20: UIImage(systemName: "line.3.horizontal")!,
    40: UIImage(systemName: "rectangle.fill")!
]
customConfig.imageProvider = { item in
    // Return a custom image based on current color/width
    let config = UIImage.SymbolConfiguration(pointSize: 24)
    return UIImage(systemName: "highlighter", withConfiguration: config)!
}

let customItem = PKToolPickerCustomItem(configuration: customConfig)

let toolPicker = PKToolPicker(toolItems: [
    PKToolPickerInkingItem(type: .pen, color: .black, width: 5),
    customItem,
    PKToolPickerEraserItem(type: .vector)
])
```

## Canvas View Delegate Lifecycle

Track the complete drawing lifecycle.

```swift
class DrawingManager: NSObject, PKCanvasViewDelegate {
    var hasUnsavedChanges = false
    var isCurrentlyDrawing = false

    func canvasViewDidBeginUsingTool(_ canvasView: PKCanvasView) {
        isCurrentlyDrawing = true
    }

    func canvasViewDidEndUsingTool(_ canvasView: PKCanvasView) {
        isCurrentlyDrawing = false
    }

    func canvasViewDrawingDidChange(_ canvasView: PKCanvasView) {
        hasUnsavedChanges = true
    }

    func canvasViewDidFinishRendering(_ canvasView: PKCanvasView) {
        // Safe to capture a snapshot for thumbnails
    }
}
```

## Undo/Redo Support

`PKCanvasView` automatically integrates with `UndoManager`.

```swift
class DrawingViewController: UIViewController {
    let canvasView = PKCanvasView()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(canvasView)

        navigationItem.leftBarButtonItem = UIBarButtonItem(
            systemItem: .undo,
            primaryAction: UIAction { [weak self] _ in
                self?.canvasView.undoManager?.undo()
            }
        )
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            systemItem: .redo,
            primaryAction: UIAction { [weak self] _ in
                self?.canvasView.undoManager?.redo()
            }
        )
    }
}
```

## Thumbnail Generation

Generate thumbnails for document browsers or galleries.

```swift
func generateThumbnail(
    for drawing: PKDrawing,
    size: CGSize,
    scale: CGFloat = 2.0
) -> UIImage? {
    let bounds = drawing.bounds
    guard !bounds.isEmpty else { return nil }

    let aspectRatio = bounds.width / bounds.height
    let targetAspect = size.width / size.height
    var renderRect = bounds

    if aspectRatio > targetAspect {
        let scaleFactor = size.width / bounds.width
        renderRect = CGRect(
            x: bounds.minX,
            y: bounds.midY - (size.height / scaleFactor) / 2,
            width: bounds.width,
            height: size.height / scaleFactor
        )
    } else {
        let scaleFactor = size.height / bounds.height
        renderRect = CGRect(
            x: bounds.midX - (size.width / scaleFactor) / 2,
            y: bounds.minY,
            width: size.width / scaleFactor,
            height: bounds.height
        )
    }

    return drawing.image(from: renderRect, scale: scale)
}
```

## Drawing Comparison and Scoring

Compare two drawings by analyzing their strokes and points.

```swift
func strokeSimilarity(
    reference: PKDrawing,
    candidate: PKDrawing,
    tolerance: CGFloat = 20
) -> Double {
    let refPoints = reference.strokes.flatMap { stroke in
        stroke.path.interpolatedPoints(by: .distance(5)).map(\.location)
    }

    let candPoints = candidate.strokes.flatMap { stroke in
        stroke.path.interpolatedPoints(by: .distance(5)).map(\.location)
    }

    guard !refPoints.isEmpty else { return 0 }

    var matchCount = 0
    for refPoint in refPoints {
        let minDist = candPoints.map { point in
            hypot(refPoint.x - point.x, refPoint.y - point.y)
        }.min() ?? .infinity

        if minDist <= tolerance { matchCount += 1 }
    }

    return Double(matchCount) / Double(refPoints.count)
}
```

## Constructing Strokes Programmatically

Use these constructors only when an app generates ink rather than receiving it
from `PKCanvasView`:

```swift
let points = [
    PKStrokePoint(
        location: CGPoint(x: 0, y: 0), timeOffset: 0,
        size: CGSize(width: 5, height: 5), opacity: 1,
        force: 0.5, azimuth: 0, altitude: .pi / 2
    ),
    PKStrokePoint(
        location: CGPoint(x: 100, y: 100), timeOffset: 0.1,
        size: CGSize(width: 5, height: 5), opacity: 1,
        force: 0.5, azimuth: 0, altitude: .pi / 2
    )
]
let path = PKStrokePath(controlPoints: points, creationDate: Date())
let stroke = PKStroke(
    ink: PKInk(.pen, color: .black), path: path,
    transform: .identity, mask: nil
)
let drawing = PKDrawing(strokes: [stroke])
```

## Content Version Management

Handle backward compatibility when sharing drawings across OS versions.

```swift
// Check if a drawing uses features beyond a version
let drawing = canvasView.drawing
let version = drawing.requiredContentVersion

switch version {
case .version1:
    // iPadOS 14-era inks: marker, pen, pencil
    break
case .version2:
    // iPadOS 17 inks: monoline, fountain pen, watercolor, crayon
    break
case .version3:
    // Barrel-roll angle data
    break
case .version4:
    // Reed pen
    break
@unknown default:
    // .version5 (iOS 27+): stable stroke IDs, render state, selection metadata
    break
}

// Limit both canvas and picker to a specific version.
// Use .version1 when saved drawings must load on pre-iPadOS 17 systems.
if #available(iOS 17.0, *) {
    canvasView.maximumSupportedContentVersion = .version1
    toolPicker.maximumSupportedContentVersion = .version1
}
```

When you allow newer inks, branch before CloudKit or cross-device sync and
upload either the original drawing or a verified fallback drawing.

```swift
func drawingForPreiPadOS17Sync(_ drawing: PKDrawing) -> PKDrawing? {
    switch drawing.requiredContentVersion {
    case .version1:
        return drawing
    case .version2, .version3, .version4:
        let fallback = version1Fallback(from: drawing)
        guard fallback.requiredContentVersion == .version1 else {
            // Reusing paths can preserve newer metadata, such as barrel-roll data.
            // Sync a thumbnail/message instead of incompatible drawing data.
            return nil
        }
        return fallback
    @unknown default:
        return nil
    }
}

func version1Fallback(from drawing: PKDrawing) -> PKDrawing {
    let strokes = drawing.strokes.map { stroke -> PKStroke in
        var fallback = stroke
        fallback.ink = PKInkingTool(.pen, color: .black, width: 2).ink
        return fallback
    }
    return PKDrawing(strokes: strokes)
}
```

iOS 27 adds `PKContentVersion.version5` for stroke identity, selection, and
render-state metadata. Name it only behind `if #available(iOS 27.0, *)`; in a
plain `switch`, `.version5` arrives through `@unknown default` on apps built
with the iOS 27 SDK but deployed to earlier versions.

## Stroke Slicing and Erasing (iOS 27+)

`PKStroke` and `PKStrokePath` support `ClosedRange<CGFloat>` parametric-range
operations on iOS 27. The range counts interpolated positions along the path —
`0.0...0.5` covers the first half of the stroke by length, not by index.

```swift
let firstHalf = stroke.substroke(range: 0.0...0.5)      // PKStroke
let tailPath = path[2.0...4.0]                          // PKStrokePath
```

Parametric slicing preserves ink-particle consistency, so sliced strokes render
identically to the original at the same positions. Use `stroke.renderState`
(`PKStroke.RenderState`) to carry grain positioning between a stroke and its
substrokes.

Erasing runs on the whole drawing and splits affected strokes:

```swift
var drawing = canvasView.drawing
drawing.erasePath(erasePath)                 // in place
// or
let erased = drawing.erasingPath(erasePath)  // copy-on-write variant
```

`mask` and `transform` parameters constrain the erase region. Erasure is
expensive on large drawings — batch it rather than calling it per pan event.
Strokes report `renderGroupID` for wet-ink compositing with compatible inks.

`path.bezierRepresentation` converts a path to `CGPath`; rebuild with
`PKStrokePath(bezierPath:creationDate:pointProvider:)`, where the point provider
maps each `ConvertedBezierPoint` to a `PKStrokePoint` (supply size, force,
opacity). Conversion samples multiple `PKStrokePoint`s per Bezier element and
only handles the first subpath.

## Handwriting Recognition (iOS 27+)

`PKStrokeRecognizer` (a Swift actor) recognizes handwriting on-device. Group it
into batches rather than updating on every `canvasViewDrawingDidChange`:

```swift
actor DrawingSearchEngine {
    private let recognizer = PKStrokeRecognizer()

    func update(_ drawing: PKDrawing) async {
        await recognizer.updateDrawing(drawing)
    }

    func find(_ query: String) async -> [CGRect] {
        let results = await recognizer.search(query)
        return results.map(\.bounds)
    }
}
```

- `recognizedText()` returns the best candidate; pass stroke IDs to restrict to
  a lasso selection (`canvasView.selection`, a `Set<UUID>`).
- `indexableContent` concatenates all candidates — suitable for Spotlight
  indexing via `CSSearchableItemAttributeSet`.
- `SearchResult` exposes `bounds` (drawing-space `CGRect` for highlighting) and
  `strokes` (matching `PKStroke` values for re-selection).
- Recheck `recognitionVersion` after OS updates; rebuild indexes when it
  changes.
- Simulator supports Latin-character languages only; test CJK on device.

## Advanced SwiftUI Wrapper

A full-featured SwiftUI wrapper with tool picker, undo, and save support.

```swift
import SwiftUI
import PencilKit

struct DrawingCanvas: UIViewRepresentable {
    @Binding var drawing: PKDrawing
    var drawingPolicy: PKCanvasViewDrawingPolicy = .anyInput
    var showToolPicker: Bool = true

    func makeUIView(context: Context) -> PKCanvasView {
        let canvas = PKCanvasView()
        canvas.delegate = context.coordinator
        canvas.drawingPolicy = drawingPolicy
        canvas.drawing = drawing
        canvas.backgroundColor = .clear
        canvas.isOpaque = false

        let coordinator = context.coordinator
        coordinator.toolPicker.addObserver(canvas)

        return canvas
    }

    func updateUIView(_ canvas: PKCanvasView, context: Context) {
        let coordinator = context.coordinator

        if canvas.drawing != drawing {
            canvas.drawing = drawing
        }

        coordinator.toolPicker.setVisible(showToolPicker, forFirstResponder: canvas)
        if showToolPicker {
            canvas.becomeFirstResponder()
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(parent: self)
    }

    class Coordinator: NSObject, PKCanvasViewDelegate {
        let parent: DrawingCanvas
        let toolPicker = PKToolPicker()

        init(parent: DrawingCanvas) {
            self.parent = parent
        }

        func canvasViewDrawingDidChange(_ canvasView: PKCanvasView) {
            parent.drawing = canvasView.drawing
        }
    }
}
```
