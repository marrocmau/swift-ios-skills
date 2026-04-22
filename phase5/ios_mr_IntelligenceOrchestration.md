---
name: intelligence-orchestration
description: Agent-centric architecture for Apple Intelligence. Covers multi-turn reasoning, intent delegation, and shared semantic state. Use this skill when building complex apps that function as intelligent plugins for Siri and system-wide automation.
compatibility: iOS 18.2+, AppIntents 3.0, Apple Intelligence
---

# Intelligence Orchestration — Agent-Centric Design

**When to use:** To transform your app into an active "Intelligent Agent" that can perform multi-step tasks across the system.

## Concept: Beyond One-Shot Intents
In 2026, Apple Intelligence doesn't just call one intent at a time. It orchestrates a "conversation" with your app. Your app must maintain a **Semantic Context** so the system can ask follow-up questions or refine actions.

## Pattern: Multi-Turn Intent Delegation (Stateful reasoning)

```swift
import AppIntents
import Foundation

// MARK: - State Manager for Agent Context
@Observable
final class AgentContext {
    static let shared = AgentContext()
    
    // Tracks what the user is currently working on within the app
    var activeEntity: TaskEntity?
    var pendingAction: String?
}

// MARK: - Refinement Intent (Multi-turn)
struct RefineTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Refine my current task"
    
    @Parameter(title: "Specific Detail")
    var detail: String
    
    // Xcode 26: The system uses this intent if the user says "Add more info to that"
    func perform() async throws -> some IntentResult & ProvidesDialog {
        guard let task = AgentContext.shared.activeEntity else {
            return .result(dialog: "I'm not sure which task you're referring to. Should we create a new one?")
        }
        
        // Refine existing entity with new data from Siri
        print("Refining \(task.title) with \(detail)")
        
        return .result(dialog: "Got it, I've updated the details for '\(task.title)'.")
    }
}
```

## Pattern: Shared Semantic State (System Awareness)
*Making your app's state visible to Apple Intelligence.*

```swift
import AppIntents

struct MyAppDataSchema: AppDataSchema {
    // Expose app's current semantic focus to the system
    static var currentFocus: AppEntity? {
        return AgentContext.shared.activeEntity
    }
    
    // Help the system understand "this", "that", "it" in conversations
    static func resolvePronoun(_ pronoun: String) -> AppEntity? {
        if pronoun == "it" || pronoun == "this" {
            return AgentContext.shared.activeEntity
        }
        return nil
    }
}
```

## Agent Implementation Checklist

| Feature | Requirement | Benefit |
| :--- | :--- | :--- |
| **State Tracking** | Maintain an `AgentContext` | Allows Siri to handle follow-up requests |
| **Pronoun Resolution** | Implement `AppDataSchema` | Makes "this" or "it" work in voice commands |
| **Refinement Intents** | Small, modular intents for editing | Enables complex task building via voice |
| **Dialog Response** | Natural, context-aware responses | Humanizes the AI interaction |

## Implementation Checklist

- [ ] Define `AgentContext` to track active entities/actions.
- [ ] Implement `AppDataSchema` to provide system-wide context.
- [ ] Create "Refinement" intents for existing entities.
- [ ] Ensure intents use `ProvidesDialog` for natural feedback.
- [ ] Test multi-turn conversations with Siri ("Do X... now add Y to it").
- [ ] Verify that "this" and "that" resolve correctly to your app's state.
- [ ] Use `MetricKit` (Phase 2) to monitor background reasoning latency.
- [ ] Optimize intent logic to respond in under 500ms.

---

## Related Skills

- **app-intents** — Foundational intent structures
- **on-device-ai** — Local reasoning before calling intents
- **liquid-glass-design** — Visual feedback for agent activity
