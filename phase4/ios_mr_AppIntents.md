---
name: app-intents
description: App Intents and Siri shortcuts integration. Use this skill when adding Siri voice commands, building Shortcuts app actions, or implementing Apple Intelligence capabilities. Essential for voice-first interactions and automation.
compatibility: iOS 16+, AppIntents framework
---

# App Intents — Siri & Shortcuts Integration

**When to use:** After core app is complete. Intents unlock voice and automation possibilities.

## Pattern: App Intents Implementation

```swift
import AppIntents
import Foundation

// MARK: - Simple Intent
struct GetTaskCountIntent: AppIntent {
    static var title: LocalizedStringResource = "Get task count"
    static var description = IntentDescription("Returns the number of tasks")
    static var openAppWhenRun = false
    
    func perform() async throws -> some IntentResult & ReturnsValue<Int> {
        // Query database
        let count = 42
        return .result(value: count)
    }
}

// MARK: - Intent with Parameters
struct CreateTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Create a new task"
    static var description = IntentDescription("Creates a task with a given title and due date")
    static var openAppWhenRun = true
    
    @Parameter(title: "Task Title")
    var taskTitle: String
    
    @Parameter(title: "Due Date", default: Date())
    var dueDate: Date?
    
    @Parameter(title: "Priority")
    var priority: TaskPriority = .normal
    
    func perform() async throws -> some IntentResult {
        // Create task in database
        print("Creating task: \(taskTitle)")
        
        return .result()
    }
}

// MARK: - Enum Parameter
enum TaskPriority: String, AppEnum {
    case low = "Low"
    case normal = "Normal"
    case high = "High"
    case urgent = "Urgent"
    
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Priority"
    
    var displayRepresentation: DisplayRepresentation {
        switch self {
        case .low:
            return DisplayRepresentation(stringLiteral: "Low Priority")
        case .normal:
            return DisplayRepresentation(stringLiteral: "Normal Priority")
        case .high:
            return DisplayRepresentation(stringLiteral: "High Priority")
        case .urgent:
            return DisplayRepresentation(stringLiteral: "Urgent")
        }
    }
}

// MARK: - Queries (for filtering and selection)
struct TasksQuery: EntityQuery {
    func entities(for identifiers: [String]) async throws -> [TaskEntity] {
        // Return specific tasks by ID
        return []
    }
    
    func suggestedEntities() async throws -> [TaskEntity] {
        // Return suggested tasks
        return []
    }
}

struct TaskEntity: AppEntity {
    var id: String
    var title: String
    var isCompleted: Bool
    
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(stringLiteral: title)
    }
    
    static var defaultQuery = TasksQuery()
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Task"
}

// MARK: - Intent with Entity Parameter
struct CompleteTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Complete task"
    static var description = IntentDescription("Mark a task as completed")
    
    @Parameter(title: "Task", requestValueDialog: "Which task?")
    var task: TaskEntity
    
    func perform() async throws -> some IntentResult {
        // Mark task as complete
        print("Completed: \(task.title)")
        return .result()
    }
}

// MARK: - Intent with Dialog Prompts
struct AddQuickTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Quick task"
    
    @Parameter(
        title: "Task",
        requestValueDialog: "What do you want to add?"
    )
    var taskTitle: String
    
    @Parameter(
        title: "Priority",
        requestValueDialog: "How urgent is this?"
    )
    var priority: TaskPriority = .normal
    
    func perform() async throws -> some IntentResult {
        print("Quick task: \(taskTitle)")
        return .result()
    }
}

// MARK: - App Shortcuts Provider
struct MyAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: GetTaskCountIntent(),
            phrases: ["How many tasks", "Task count", "Tell me my task count"]
        )
        
        AppShortcut(
            intent: CreateTaskIntent(),
            phrases: ["Create task", "Add task", "New task"]
        )
        
        AppShortcut(
            intent: CompleteTaskIntent(),
            phrases: ["Complete task", "Finish task", "Mark done"]
        )
        
        AppShortcut(
            intent: AddQuickTaskIntent(),
            phrases: ["Quick task", "Add quickly"]
        )
    }
}

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
