---
name: swiftui-animations
description: SwiftUI animations and transitions for polished UI. Use this skill when adding motion design, creating smooth transitions, building interactive feedback, or improving perceived performance. Essential for app polish and user delight.
compatibility: iOS 17+, SwiftUI
---

# SwiftUI Animations — Motion & Polish

**When to use:** After core functionality works. Animations enhance perceived performance and delight.

## Pattern: Animation Techniques

```swift
import SwiftUI

struct AnimationExamples: View {
    @State private var isAnimating = false
    
    var body: some View {
        VStack(spacing: 30) {
            
            // MARK: - Spring Animation
            VStack {
                Image(systemName: "star.fill")
                    .font(.system(size: 48))
                    .foregroundStyle(.yellow)
                    .scaleEffect(isAnimating ? 1.5 : 1.0)
                    .animation(.spring(response: 0.5, dampingFraction: 0.6), value: isAnimating)
                
                Button("Spring") {
                    isAnimating.toggle()
                }
            }
            
            // MARK: - Phase Animator (iOS 17+)
            VStack {
                HStack(spacing: 8) {
                    ForEach(0..<4, id: \.self) { _ in
                        RoundedRectangle(cornerRadius: 4)
                            .frame(width: 8, height: 32)
                            .foregroundStyle(.blue)
                    }
                }
                .phaseAnimator([0, 1], trigger: isAnimating) { content, phase in
                    content.offset(y: phase == 0 ? 0 : -8)
                }
            }
            
            // MARK: - Keyframe Animator (iOS 17+)
            VStack {
                Image(systemName: "heart.fill")
                    .font(.system(size: 40))
                    .foregroundStyle(.red)
                    .keyframeAnimator(
                        initialValue: 0,
                        trigger: isAnimating
                    ) { content, value in
                        content.scaleEffect(1.0 + value * 0.3)
                    } keyframes: { _ in
                        KeyframeTrack(\.self) {
                            LinearKeyframe(0.0, duration: 0)
                            LinearKeyframe(1.0, duration: 0.1)
                            LinearKeyframe(0.0, duration: 0.1)
                            LinearKeyframe(1.0, duration: 0.1)
                            LinearKeyframe(0.0, duration: 0.1)
                        }
                    }
                
                Button("Heartbeat") {
                    isAnimating.toggle()
                }
            }
            
            // MARK: - Transition Animation
            VStack {
                if isAnimating {
                    Text("Appearing")
                        .padding()
                        .background(Color.green.opacity(0.3))
                        .cornerRadius(8)
                        .transition(.asymmetric(
                            insertion: .scale.combined(with: .opacity),
                            removal: .opacity
                        ))
                }
                
                Button("Toggle") {
                    withAnimation(.easeInOut(duration: 0.3)) {
                        isAnimating.toggle()
                    }
                }
            }
            
            // MARK: - Custom Timing
            VStack {
                Circle()
                    .frame(width: 50)
                    .foregroundStyle(.purple)
                    .offset(x: isAnimating ? 100 : 0)
                    .animation(
                        .easeInOut(duration: 1)
                        .repeatForever(autoreverses: true),
                        value: isAnimating
                    )
                
                Button("Custom Timing") {
                    isAnimating.toggle()
                }
            }
        }
        .padding()
        .onAppear {
            isAnimating = true
        }
    }
}

// MARK: - Reusable Animation Components
struct AnimatedButton: View {
    let title: String
    let action: () -> Void
    
    @State private var isPressed = false
    
    var body: some View {
        Button(action: {
            isPressed = true
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                isPressed = false
            }
            action()
        }) {
            Text(title)
                .font(.headline)
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue)
                .foregroundStyle(.white)
                .cornerRadius(8)
                .scaleEffect(isPressed ? 0.95 : 1.0)
                .animation(.easeInOut(duration: 0.1), value: isPressed)
        }
    }
}

struct LoadingSpinner: View {
    @State private var isRotating = false
    
    var body: some View {
        Image(systemName: "arrowtriangle.right.fill")
            .font(.system(size: 20))
            .foregroundStyle(.blue)
            .rotationEffect(.degrees(isRotating ? 360 : 0))
            .animation(.linear(duration: 1).repeatForever(autoreverses: false), value: isRotating)
            .onAppear {
                isRotating = true
            }
    }
}

struct PulseAnimation: View {
    @State private var isPulsing = false
    
    var body: some View {
        Circle()
            .fill(.blue)
            .frame(width: 16, height: 16)
            .scaleEffect(isPulsing ? 1.5 : 1.0)
            .opacity(isPulsing ? 0 : 1)
            .animation(
                .easeOut(duration: 0.6)
                .repeatForever(autoreverses: false),
                value: isPulsing
            )
            .onAppear {
                isPulsing = true
            }
    }
}
```

## Animation Best Practices

```
Duration Guidelines:
- Quick feedback: 0.1 - 0.2 seconds
- Smooth transitions: 0.3 - 0.5 seconds
- Engaging animations: 0.5 - 1.0 second
- Don't exceed 1 second unless intentional

Timing Functions:
- .linear: Progress bars, spinners
- .easeInOut: Natural motion
- .easeIn: Objects disappearing
- .easeOut: Objects appearing
- .spring: Playful interactions

Performance:
- Use CADisplayLink for complex animations
- Limit simultaneous animations
- Test on older devices
- Use Metal for high-performance needs
```

## Implementation Checklist

- [ ] Add spring animation to interactive buttons
- [ ] Implement phase animator for loading states
- [ ] Create custom transitions for screen changes
- [ ] Add haptic feedback with animations
- [ ] Build loading spinner component
- [ ] Create pulse animation for notifications
- [ ] Implement gesture-driven animations
- [ ] Add motion blur effects
- [ ] Test animations on devices (not simulator)
- [ ] Monitor performance with Instruments
- [ ] Create animation library for reuse
- [ ] Document animation timing decisions
