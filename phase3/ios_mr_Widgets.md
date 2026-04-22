---
name: widgetkit-homescreen
description: Home screen and lock screen widgets using WidgetKit. Use this skill when adding home screen widgets, lock screen widgets, interactive widgets, or live activities. Essential for user engagement and home screen presence.
compatibility: iOS 16+, WidgetKit
---

# WidgetKit — Home & Lock Screen Widgets

**When to use:** After core app is stable. Widgets drive repeat usage and home screen engagement.

## Pattern: Widget Implementation

```swift
import WidgetKit
import SwiftUI

// MARK: - Widget Definition
struct SimpleWidget: Widget {
    let kind: String = "SimpleWidget"
    
    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: kind,
            provider: SimpleTimelineProvider()
        ) { entry in
            SimpleWidgetEntryView(entry: entry)
        }
        .configurationDisplayName("Simple Widget")
        .description("Shows current time and next task")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}

// MARK: - Timeline Entry
struct SimpleEntry: TimelineEntry {
    let date: Date
    let nextTask: String?
    let taskCount: Int
}

// MARK: - Timeline Provider (AI-Ready)
struct SimpleTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> SimpleEntry {
        SimpleEntry(date: Date(), nextTask: "Sample task", taskCount: 5)
    }
    
    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> ()) {
        let entry = SimpleEntry(date: Date(), nextTask: "Current task", taskCount: 3)
        completion(entry)
    }
    
    func getTimeline(in context: Context, completion: @escaping (Timeline<SimpleEntry>) -> ()) {
        // Xcode 26 Intelligence Pattern:
        // Define when the widget is MOST relevant (e.g., specific location, time, or activity)
        let relevance = TimelineEntryRelevance(score: 1.0, duration: 3600)
        
        let currentDate = Date()
        let entries = (0..<12).map { hour in
            SimpleEntry(
                date: Calendar.current.date(byAdding: .hour, value: hour, to: currentDate)!,
                nextTask: hour == 0 ? "Current task" : "Upcoming task",
                taskCount: Int.random(in: 2...8)
            )
        }
        
        let nextUpdate = Calendar.current.date(byAdding: .hour, value: 12, to: currentDate)!
        // Policy with AI-hinted timeline entries
        let timeline = Timeline(entries: entries, policy: .after(nextUpdate))
        completion(timeline)
    }
}

// MARK: - Widget View
struct SimpleWidgetEntryView: View {
    var entry: SimpleEntry
    
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            HStack {
                Text("Tasks")
                    .font(.headline)
                Spacer()
                Text("\(entry.taskCount)")
                    .font(.title2)
                    .fontWeight(.bold)
            }
            
            if let nextTask = entry.nextTask {
                VStack(alignment: .leading, spacing: 4) {
                    Text("Next:")
                        .font(.caption)
                        .foregroundStyle(.gray)
                    Text(nextTask)
                        .font(.caption)
                        .lineLimit(2)
                }
            }
            
            Spacer()
            
            Text(entry.date, style: .time)
                .font(.caption)
                .foregroundStyle(.gray)
        }
        .padding()
        .background(Color(UIColor.systemBackground))
        .widgetURL(URL(string: "myapp://tasks"))
    }
}

// MARK: - Widget Bundle
@main
struct TaskWidgetBundle: WidgetBundle {
    var body: some Widget {
        SimpleWidget()
    }
}

// MARK: - Lock Screen Widget (iOS 16.1+)
struct LockScreenWidget: Widget {
    let kind: String = "LockScreenWidget"
    
    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: kind,
            provider: LockScreenProvider()
        ) { entry in
            LockScreenWidgetView(entry: entry)
        }
        .configurationDisplayName("Task Count")
        .description("Shows today's task count")
        .supportedFamilies([.accessoryCircular, .accessoryRectangular])
    }
}

struct LockScreenEntry: TimelineEntry {
    let date: Date
    let taskCount: Int
}

struct LockScreenProvider: TimelineProvider {
    func placeholder(in context: Context) -> LockScreenEntry {
        LockScreenEntry(date: Date(), taskCount: 5)
    }
    
    func getSnapshot(in context: Context, completion: @escaping (LockScreenEntry) -> ()) {
        let entry = LockScreenEntry(date: Date(), taskCount: 3)
        completion(entry)
    }
    
    func getTimeline(in context: Context, completion: @escaping (Timeline<LockScreenEntry>) -> ()) {
        let entries = (0..<12).map { hour in
            LockScreenEntry(
                date: Calendar.current.date(byAdding: .hour, value: hour, to: Date())!,
                taskCount: Int.random(in: 1...10)
            )
        }
        
        let timeline = Timeline(
            entries: entries,
            policy: .after(Calendar.current.date(byAdding: .hour, value: 12, to: Date())!)
        )
        completion(timeline)
    }
}

struct LockScreenWidgetView: View {
    var entry: LockScreenEntry
    
    var body: some View {
        ZStack {
            AccessoryWidgetBackground()
            
            VStack {
                Text("\(entry.taskCount)")
                    .font(.system(size: 32, weight: .bold))
                Text("Tasks")
                    .font(.caption)
            }
        }
    }
}
```

## Widget Configuration

### Info.plist for Widget Target

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSExtension</key>
    <dict>
        <key>NSExtensionPointIdentifier</key>
        <string>com.apple.widgetkit-extension</string>
    </dict>
</dict>
</plist>
```

### Xcode Setup Steps

```
1. File → New → Target
2. Select "Widget Extension"
3. Name: "MyAppWidgets"
4. Select main app as container
5. Create Scheme when prompted
6. Configure supported families in Widget definition
7. Add WidgetKit to main app capabilities
```

## Implementation Checklist

- [ ] Create Widget Extension target
- [ ] Define TimelineEntry struct
- [ ] Implement TimelineProvider
- [ ] Create widget view layout
- [ ] Configure supported families
- [ ] Test on home screen
- [ ] Add lock screen widget (iOS 16.1+)
- [ ] Implement widgetURL for deep linking
- [ ] Set up timeline refresh strategy
- [ ] Add background refresh
- [ ] Monitor widget cache usage
- [ ] Test widget on multiple devices
