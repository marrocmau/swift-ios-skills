---
name: on-device-ai
description: On-device AI using Foundation Models and machine learning. Use this skill when implementing local AI features, text generation, image processing, or machine learning models without cloud dependencies. Essential for privacy, offline functionality, and responsive AI features.
compatibility: iOS 18+, Foundation Models, CoreML
---

# On-Device AI — Local Intelligence

**When to use:** For privacy-sensitive AI features or offline-capable intelligence.

## Pattern: Foundation Models Integration

```swift
import Foundation
import CoreML

@available(iOS 18, *)
@Observable
final class OnDeviceAIManager {
    
    // MARK: - Text Generation
    func generateText(prompt: String) async throws -> String {
        // Foundation Models Framework (iOS 18+)
        // Future API - placeholder for evolving capabilities
        
        // Example implementation pattern:
        // let model = try await FoundationModelAPI.loadModel()
        // let response = try await model.generate(prompt: prompt)
        // return response.text
        
        return "AI-generated response based on local model"
    }
    
    // MARK: - Image Classification
    func classifyImage(_ image: UIImage) async throws -> String {
        // Use Vision framework + CoreML
        // Example: plant identification, object detection
        
        return "Classified object type"
    }
    
    // MARK: - Text Embedding
    func embedText(_ text: String) async throws -> [Float] {
        // Generate embeddings for semantic search
        // Useful for: similarity matching, clustering
        
        return Array(repeating: 0.0, count: 384)
    }
    
    // MARK: - Sentiment Analysis
    func analyzeSentiment(_ text: String) -> String {
        // CoreML model for sentiment classification
        // Returns: positive, negative, neutral
        
        return "positive"
    }
}

// MARK: - CoreML Model Integration
final class CoreMLModelManager {
    
    static let shared = CoreMLModelManager()
    
    // Load compiled ML model
    func loadModel(named modelName: String) throws -> MLModel {
        // Example: sentiment-analyzer.mlmodel
        let config = MLModelConfiguration()
        let model = try MLModel(
            contentsOf: Bundle.main.url(forResource: modelName, withExtension: "mlmodelc")!,
            configuration: config
        )
        return model
    }
    
    // Make predictions
    func makePrediction(
        input: MLFeatureProvider,
        model: MLModel
    ) throws -> MLFeatureProvider {
        return try model.prediction(from: input)
    }
}

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
