---
name: app-intents
description: App Intents and Siri shortcuts integration. Use this skill when adding Siri voice commands, building Shortcuts app actions, or implementing Apple Intelligence capabilities. Essential for voice-first interactions and automation.
compatibility: iOS 16+, AppIntents framework
---

# App Intents — Assistant & AI Orchestration

**When to use:** To expose your app's capabilities to Apple Intelligence and Siri. Intents are the "tools" the system uses to perform actions on behalf of the user.

## Pattern: Assistant-Integrated Intents (Xcode 26)

```swift
import AppIntents
import Foundation

// MARK: - Semantic Intent (Assistant-Ready)
@AssistantIntent(schema: .system.createTask) // Xcode 26: Map to system semantic domain
struct CreateTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Create a new task"
    static var description = IntentDescription("Creates a task in the app's database")
    
    @Parameter(title: "Task Title", description: "The name of the task")
    var taskTitle: String
    
    @Parameter(title: "Due Date", default: Date())
    var dueDate: Date?
    
    // Xcode 26: Define what the AI should "see" after the intent runs
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // Business logic here
        print("Creating task: \(taskTitle)")
        
        return .result(dialog: "Okay, I've added '\(taskTitle)' to your list.")
    }
}

// MARK: - App Entity with Semantic Properties
struct TaskEntity: AppEntity {
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Task"
    static var defaultQuery = TasksQuery()
    
    var id: String
    
    @Property(title: "Title")
    var title: String
    
    @Property(title: "Is Completed")
    var isCompleted: Bool
    
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(stringLiteral: title)
    }
}

// MARK: - Assistant Queries (Vector Search Support)
struct TasksQuery: EntityQuery {
    func entities(for identifiers: [String]) async throws -> [TaskEntity] {
        // Standard ID fetch
        return []
    }
    
    // Xcode 26: Allow Apple Intelligence to find entities by description (Semantic Search)
    func suggestedEntities() async throws -> [TaskEntity] {
        return []
    }
}
```

## Pattern: System-Wide Reasoning Integration
*How to make your app a "plugin" for Apple Intelligence.*

```swift
// MARK: - App Shortcuts Provider
struct MyAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        // Xcode 26 Tip: Use more natural phrases to help the LLM map the user's intent
        AppShortcut(
            intent: CreateTaskIntent(),
            phrases: [
                "Create a \(.applicationName) task named \(\.$taskTitle)",
                "Add a new item to \(.applicationName)",
                "Remind me to \(\.$taskTitle) in \(.applicationName)"
            ],
            shortTitle: "Create Task",
            systemImageName: "plus.circle"
        )
    }
}
```

## AI Reasoning Checklist (Xcode 26)

| Requirement | Description | Benefit |
| :--- | :--- | :--- |
| **Semantic Mapping** | Use `@AssistantIntent(schema:)` | Maps your code to system-wide AI actions |
| **Entity Properties** | Annotate fields with `@Property` | Allows AI to filter and search your data |
| **Natural Language** | Provide diverse, dynamic phrases | Improves Siri's accuracy in understanding users |
| **Tool Availability** | Mark critical actions as intents | Allows the local LLM to solve complex user requests |

## Implementation Checklist

- [ ] Import AppIntents framework
- [ ] Annotate intents with `@AssistantIntent` where applicable
- [ ] Define `AppEntity` with full `@Property` metadata
- [ ] Implement `EntityQuery` for both ID and semantic search
- [ ] Create natural, variable phrases in `AppShortcutsProvider`
- [ ] Test intents using Siri voice commands
- [ ] Verify intent behavior in the Shortcuts app
- [ ] Add dialog prompts and success confirmations
- [ ] Test "System-Wide Reasoning" (e.g., Siri performing cross-app actions)
- [ ] Document the "Intent-First" architecture for future scalability

// MARK: - Dynamic Shortcuts (in Shortcuts app)
struct ParameterizedShortcut: AppShortcut {
    static var shortcutName: LocalizedStringResource = "Create Weekly Task"
    
    static var intent: CreateTaskIntent {
        CreateTaskIntent(
            taskTitle: "Weekly Review",
            dueDate: Calendar.current.date(byAdding: .weekOfMonth, value: 1, to: Date()),
            priority: .high
        )
    }
    
    static var phrases: [AppShortcut.PhrasePart] {
        [
            "Create my weekly review task",
            "Setup weekly task"
        ]
    }
}

// MARK: - State-based Intents
@Observable
final class IntentStateManager {
    var lastCreatedTask: TaskEntity?
    
    func saveLastTask(_ task: TaskEntity) {
        self.lastCreatedTask = task
        // Also save to UserDefaults for persistence
    }
}

// Intent that uses state
struct CompleteLastTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Complete last task"
    
    func perform() async throws -> some IntentResult {
        let manager = IntentStateManager()
        
        if let lastTask = manager.lastCreatedTask {
            print("Completing last task: \(lastTask.title)")
            return .result()
        }
        
        return .result(dialog: "No recent task found")
    }
}
```

## Info.plist Configuration

```xml
<key>NSAppIntentsSupported</key>
<true/>

<key>NSUserActivityTypes</key>
<array>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER).CreateTaskIntent</string>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER).CompleteTaskIntent</string>
</array>
```

## Common Shortcut Phrases

```
"Siri, [your phrase]"

Examples:
- "Siri, what are my tasks?"
- "Siri, add a task"
- "Siri, mark this done"
- "Siri, quick task reminder"
```

## Shortcuts App Actions

Users can combine your intents in Shortcuts app:
- Receive input from previous action
- Conditional logic (if/then)
- Loops and repetition
- Ask for input
- Show results

## Implementation Checklist

- [ ] Import AppIntents framework
- [ ] Define Intent structs with @Parameter
- [ ] Create Entity and Query types
- [ ] Implement AppShortcutsProvider
- [ ] Define natural language phrases
- [ ] Test intents with Siri
- [ ] Test in Shortcuts app
- [ ] Add dialog prompts for parameters
- [ ] Implement entity suggestions
- [ ] Test voice commands
- [ ] Add App Intents capability
- [ ] Document intent usage
