---
name: liquid-glass-design
description: Next-generation organic UI using Liquid Glass standards. Covers MeshGradients, Variable Blurs, and Reactive Materials. Use this skill when building immersive, AI-responsive interfaces that feel "alive" and integrated into the system.
compatibility: iOS 18+, SwiftUI 6+, Xcode 26+
---

# Liquid Glass Design — Organic & Reactive UI

**When to use:** For high-end, immersive apps where the interface needs to reflect the fluid nature of Apple Intelligence.

## Concept: Material Intelligence
In 2026, UI is no longer "flat" or "glassy"—it is **Liquid**. It reacts to the user's touch with surface tension and to the AI's "thought process" with light glows and mesh distortions.

## Pattern: Advanced MeshGradient (The "Live" Background)

```swift
import SwiftUI

struct LiquidBackgroundView: View {
    @State private var isAIThinking = false
    
    var body: some View {
        // Xcode 26 Pattern: MeshGradient with 12 points for fluid motion
        MeshGradient(
            width: 3,
            height: 4,
            points: [
                [0, 0], [0.5, 0], [1, 0],
                [0, 0.3], [isAIThinking ? 0.7 : 0.3, 0.3], [1, 0.3],
                [0, 0.7], [0.5, 0.7], [1, 0.7],
                [0, 1], [0.5, 1], [1, 1]
            ],
            colors: [
                .indigo, .blue, .purple,
                .black, isAIThinking ? .cyan : .blue, .black,
                .purple, .black, .indigo,
                .blue, .purple, .black
            ],
            smoothsColors: true
        )
        .ignoresSafeArea()
        .onAppear {
            withAnimation(.easeInOut(duration: 5).repeatForever(autoreverses: true)) {
                isAIThinking.toggle()
            }
        }
    }
}
```

## Pattern: Variable Blur (Depth Focus)
*Use Variable Blur to direct the user's focus during AI interactions.*

```swift
struct ContextualFocusView: View {
    var body: some View {
        ZStack {
            Image("BackgroundContent")
                .resizable()
                .scaledToFill()
                // iOS 18+ Variable Blur: Intense at the bottom, clear at the top
                .visualEffect { content, proxy in
                    content.variableBlur(
                        radius: 20,
                        maxRange: 0.8, // Blur intensity profile
                        anchor: .bottom
                    )
                }
            
            VStack {
                Spacer()
                Text("AI Assistant is ready")
                    .font(.title)
                    .padding()
                    .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 20))
            }
            .padding()
        }
    }
}
```

## Pattern: The "Siri Glow" Effect (Text Rendering)
*Animated text that signals AI activity.*

```swift
struct AIGlowText: View {
    let text: String
    @State private var glowPhase = 0.0
    
    var body: some View {
        Text(text)
            .font(.system(size: 40, weight: .black, design: .rounded))
            // Custom TextRenderer for character-by-character glow
            .textRenderer(ParticleGlowRenderer(phase: glowPhase))
            .onAppear {
                withAnimation(.linear(duration: 2).repeatForever(autoreverses: false)) {
                    glowPhase = .pi * 2
                }
            }
    }
}

struct ParticleGlowRenderer: TextRenderer {
    var phase: Double
    
    func draw(layout: Text.Layout, in context: inout GraphicsContext) {
        for line in layout {
            for run in line {
                for (index, glyph) in run.enumerated() {
                    var copy = context
                    let offset = sin(phase + Double(index) * 0.3) * 3
                    copy.opacity = 0.5 + (sin(phase + Double(index) * 0.5) * 0.5)
                    copy.translateBy(x: 0, y: offset)
                    copy.draw(glyph)
                }
            }
        }
    }
}
```

## Liquid Glass Checklist (Design Standards)

| Principle | Technical Implementation | Goal |
| :--- | :--- | :--- |
| **Organic Motion** | `spring(response:damping:)` + `MeshGradient` | UI feels alive, not mechanical |
| **Dynamic Material** | `ultraThinMaterial` with `Variable Blur` | Focuses attention where it matters |
| **Reactive Light** | `TextRenderer` + `GlowEffects` | Signals AI state without blocking UI |
| **Environmental Sync** | `colorScheme` + `AmbientLightAPI` | App feels physically present in the room |

## Implementation Checklist

- [ ] Replace static backgrounds with animated `MeshGradient`.
- [ ] Implement `Variable Blur` to handle depth of field in complex views.
- [ ] Use `TextRenderer` for critical AI-driven messaging and feedback.
- [ ] Apply `visualEffect` modifiers for scroll-linked or touch-linked distortions.
- [ ] Ensure all materials are adaptive (Light/Dark mode + High Contrast).
- [ ] Test fluid transitions (0.5s - 0.8s) for screen changes.
- [ ] Audit performance: ensure MeshGradients don't drop frame rates on older devices.
- [ ] Integrate haptic feedback (`sensoryFeedback`) with every "liquid" transition.

---

## Related Skills

- **swiftui-animations** — Foundational motion principles
- **on-device-ai** — Triggering UI changes based on AI state
- **accessibility-advanced** — Ensuring sensory UI remains inclusive
