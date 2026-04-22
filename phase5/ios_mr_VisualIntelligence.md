---
name: visual-intelligence
description: Integration with system Visual Intelligence and real-time vision processing. Covers object recognition, contextual actions, and spatial awareness. Use this skill when building camera-first apps or features that interact with the physical world.
compatibility: iOS 18.2+, Vision framework, Visual Intelligence API
---

# Visual Intelligence — Real-World Integration

**When to use:** For apps that interact with the user's environment (shopping, education, maintenance, productivity).

## Concept: Visual Triggers
Visual Intelligence (VI) is the bridge between the real world and your app's intents. In 2026, when a user points their camera at a known object, Apple Intelligence can suggest your app to perform a specific action.

## Pattern: Contextual Visual Intent (Visual Search)

```swift
import AppIntents
import Vision

// Xcode 26: Intent that receives visual data from the system camera
@AssistantIntent(schema: .system.visualSearch)
struct VisualTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Identify and Create Task"
    
    // The image or context captured by Visual Intelligence
    @Parameter(title: "Visual Input")
    var visualData: VisualEntity
    
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // AI processes the visual data (e.g., identifies a broken chair)
        let model = try await LanguageModel.load(.medium)
        let identification = try await model.reason(on: visualData)
        
        // Execute app logic based on what the camera "saw"
        print("Creating task for: \(identification)")
        
        return .result(dialog: "I identified a \(identification). Should I add it to your tasks?")
    }
}
```

## Pattern: On-Device Vision Processing (Low Latency)
*Using local ML models to identify objects in your app's custom camera view.*

```swift
import Vision

final class RealTimeVisionManager {
    
    func processCameraFrame(pixelBuffer: CVPixelBuffer) async throws {
        // High-performance vision request
        let request = VNRecognizeObjectsRequest { request, error in
            guard let results = request.results as? [VNRecognizedObjectObservation] else { return }
            
            // Map visual detections to App Entities
            for observation in results {
                let label = observation.labels.first?.identifier ?? "Unknown"
                // Trigger an AI "Hint" if it's a known product or task
                print("Found: \(label)")
            }
        }
        
        // Use high-performance Vision handler
        let handler = VNImageRequestHandler(cvPixelBuffer: pixelBuffer)
        try handler.perform([request])
    }
}
```

## Visual Intelligence Standards (Xcode 26)

| Feature | Technical Solution | User Experience |
| :--- | :--- | :--- |
| **Object Identify** | `VNCoreMLRequest` + `LanguageModel` | "What is this?" -> "It's a IKEA chair" |
| **Text Extraction** | `VNRecognizeTextRequest` | "Copy this text into my app" |
| **Spatial Awareness** | `ARKit` + `Visual Intelligence` | "Measure this and add to my inventory" |
| **System Suggestion** | `AppShortcutsProvider` metadata | Siri suggests: "Search with My App" |

## Implementation Checklist

- [ ] Register intents for `.system.visualSearch` domain.
- [ ] Use `VNRecognizeObjectsRequest` for real-time tracking.
- [ ] Implement `LanguageModel` reasoning to interpret visual data.
- [ ] Optimize camera processing loops (no heap allocations, use `Span`).
- [ ] Add `visualEffect` shaders to highlight detected objects (Phase 5: Liquid Glass).
- [ ] Support `Privacy Manifests` for camera usage (Phase 2).
- [ ] Test in various lighting and background conditions.
- [ ] Document all supported object types for system-wide search.

---

## Related Skills

- **on-device-ai** — Reasoning on visual data
- **liquid-glass-design** — Visual highlights and glows
- **app-intents** — Executing actions based on identification
