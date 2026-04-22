---
name: on-device-ai
description: On-device AI using Foundation Models and machine learning. Use this skill when implementing local AI features, text generation, image processing, or machine learning models without cloud dependencies. Essential for privacy, offline functionality, and responsive AI features.
compatibility: iOS 18+, Foundation Models, CoreML
---

# On-Device AI — Foundation Models & Orchestration

**When to use:** For privacy-centric AI features, text generation, and "reasoning" tasks using Apple Intelligence Foundation Models.

## Pattern: Foundation Models Orchestration (Xcode 26)

```swift
import Foundation
import CoreML
import LanguageModels // Theoretical 2026 Framework for System LLMs

@available(iOS 18.2, *)
@Observable
final class AppleIntelligenceManager {
    
    // MARK: - Orchestration: Summarization & Reasoning
    func summarize(text: String) async throws -> String {
        // Xcode 26: Orchestrate system-wide Foundation Model
        let model = try await LanguageModel.load(.small)
        let response = try await model.generate(
            prompt: "Summarize the following text briefly: \(text)",
            maxTokens: 100
        )
        return response.text
    }
    
    // MARK: - High-Performance Buffer Handling (Span Integration)
    // Use Span (from Phase 1) to process large text inputs without copying
    func processMassiveInput(buffer: Span<UInt8>) async throws -> [String] {
        // Zero-copy tokenization for LLM input
        var tokens: [String] = []
        // ... process buffer logic using fast-path Swift types
        return tokens
    }
    
    // MARK: - Tool Use: Triggering App Intents
    func performIntentBasedOnAI(userInput: String) async throws {
        let model = try await LanguageModel.load(.medium)
        let result = try await model.reason(
            input: userInput,
            availableTools: [CreateTaskIntent.self, GetTaskCountIntent.self]
        )
        
        // If the AI decides to call a tool (App Intent)
        if let intent = result.suggestedIntent as? CreateTaskIntent {
            try await intent.perform()
        }
    }
}
```

## Pattern: CoreML + Vision (Hybrid Approach)
*Use CoreML for specialized tasks while using Foundation Models for general reasoning.*

```swift
final class VisionAIManager {
    // Specialized Object Detection with YOLO/CoreML
    func detectObjects(in image: UIImage) async throws -> [VNRecognizedObjectObservation] {
        guard let cgImage = image.cgImage else { throw AIError.invalidImage }
        
        let request = VNCoreMLRequest(model: try MLModel(
            contentsOf: Bundle.main.url(forResource: "YOLOv8", withExtension: "mlmodelc")!
        ))
        
        let handler = VNImageRequestHandler(cgImage: cgImage)
        try handler.perform([request])
        
        return request.results as? [VNRecognizedObjectObservation] ?? []
    }
}
```

## AI Implementation Comparison (Xcode 26)

| Feature | Legacy Approach (CoreML) | Modern Approach (Foundation Models) |
| :--- | :--- | :--- |
| **Model Hosting** | Bundled with App (Heavy) | System-Wide (Shared/Light) |
| **Task Scope** | Fixed (Classification/Detection) | General (Reasoning/Text Gen) |
| **Privacy** | Local but custom | System-Guaranteed (Private Cloud Compute) |
| **Performance** | Blocked on NPU | Orchestrated by System OS |

## Implementation Checklist

- [ ] Use `LanguageModels` for text-based reasoning and summarization.
- [ ] Implement `AppIntents` as "tools" for the Foundation Model to use.
- [ ] Use `Span` and `InlineArray` for low-latency buffer processing.
- [ ] Monitor memory pressure using `MetricKit` (Phase 2).
- [ ] Handle model download states and offline fallbacks.
- [ ] Ensure `Private Cloud Compute` (PCC) compliance for hybrid tasks.
- [ ] Implement "Siri Glow" UI effects for AI states (Phase 3).
- [ ] Verify model inference latency across multiple device tiers.

// MARK: - Vision Framework Integration
final class VisionAIManager {
    
    // Image classification
    func classifyImage(_ image: UIImage) async throws -> [String: Double] {
        // Use Vision + CoreML for image classification
        // Returns: [className: confidence]
        
        guard let cgImage = image.cgImage else {
            throw NSError(domain: "Image Error", code: -1)
        }
        
        let request = VNCoreMLRequest(model: try MLModel(
            contentsOf: Bundle.main.url(forResource: "ImageClassifier", withExtension: "mlmodelc")!
        ))
        
        var results: [String: Double] = [:]
        
        // Process image
        let handler = VNImageRequestHandler(cgImage: cgImage)
        try handler.perform([request])
        
        return results
    }
    
    // Text recognition (OCR)
    func recognizeText(in image: UIImage) async throws -> String {
        guard let cgImage = image.cgImage else {
            throw NSError(domain: "Image Error", code: -1)
        }
        
        let request = VNRecognizeTextRequest()
        request.recognitionLevel = .accurate
        
        var recognizedText = ""
        
        let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
        try handler.perform([request])
        
        if let observations = request.results as? [VNRecognizedTextObservation] {
            recognizedText = observations.compactMap { $0.topCandidates(1).first?.string }.joined(separator: "\n")
        }
        
        return recognizedText
    }
}

// MARK: - Natural Language Processing
final class NLPManager {
    
    // Tokenization
    func tokenize(_ text: String) -> [String] {
        let tagger = NSLinguisticTagger(tagSchemes: [.tokenType], options: 0)
        tagger.string = text
        
        var tokens: [String] = []
        let range = NSRange(text.startIndex..., in: text)
        
        tagger.enumerateTags(
            in: range,
            scheme: .tokenType,
            options: [.omitPunctuation, .omitWhitespace]
        ) { _, tokenRange, _ in
            if let range = Range(tokenRange, in: text) {
                tokens.append(String(text[range]))
            }
        }
        
        return tokens
    }
    
    // Part-of-speech tagging
    func analyzeSyntax(_ text: String) -> [(word: String, tag: String)] {
        let tagger = NSLinguisticTagger(tagSchemes: [.nameType], options: 0)
        tagger.string = text
        
        var results: [(String, String)] = []
        let range = NSRange(text.startIndex..., in: text)
        
        tagger.enumerateTags(in: range, scheme: .nameType, options: [.omitPunctuation]) { tag, tokenRange, _ in
            if let range = Range(tokenRange, in: text) {
                let word = String(text[range])
                let tagName = tag?.rawValue ?? "unknown"
                results.append((word, tagName))
            }
        }
        
        return results
    }
}

// Usage in SwiftUI
@available(iOS 18, *)
struct AIFeatureView: View {
    @State private var aiManager = OnDeviceAIManager()
    @State private var result = ""
    @State private var isLoading = false
    
    var body: some View {
        VStack(spacing: 16) {
            Text("On-Device AI Features")
                .font(.title)
            
            Button("Generate Text") {
                Task {
                    isLoading = true
                    do {
                        result = try await aiManager.generateText(prompt: "Hello")
                    } catch {
                        result = "Error: \(error.localizedDescription)"
                    }
                    isLoading = false
                }
            }
            .disabled(isLoading)
            
            if !result.isEmpty {
                Text(result)
                    .padding()
                    .background(Color.gray.opacity(0.1))
                    .cornerRadius(8)
            }
        }
        .padding()
    }
}
```

## Models to Explore

```
Text-Based:
- GPT-2 (text generation)
- BERT (embeddings)
- RoBERTa (sentiment analysis)

Vision-Based:
- MobileNet (image classification)
- YOLO (object detection)
- ResNet (feature extraction)

Specialized:
- MNIST (handwriting recognition)
- Speech-to-Text (audio processing)
```

## Implementation Checklist

- [ ] Check iOS 18+ availability
- [ ] Download ML models from Core ML model zoo
- [ ] Add .mlmodelc files to Xcode
- [ ] Implement CoreML wrapper
- [ ] Test inference performance
- [ ] Add Vision framework integration
- [ ] Implement NLP capabilities
- [ ] Handle offline scenarios
- [ ] Monitor memory usage
- [ ] Optimize model size for distribution
- [ ] Test on various devices
- [ ] Document model sources and licenses
