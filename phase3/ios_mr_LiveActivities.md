---
name: live-activities
description: Live Activities and Dynamic Island with ActivityKit. Use this skill when displaying real-time content, building timer/tracker features, or implementing Dynamic Island experiences. Essential for real-time user engagement and notification persistence.
compatibility: iOS 16.1+, ActivityKit
---

# Live Activities — Dynamic Island & Real-Time Updates

**When to use:** For apps with real-time events (timers, deliveries, sports scores, fitness tracking).

## Pattern: Live Activity Implementation

```swift
import ActivityKit
import SwiftUI

// MARK: - Activity Attributes
struct TimerAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        var timeRemaining: Int
        var isRunning: Bool
    }
    
    var title: String
    var startTime: Date
}

struct DeliveryAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        var deliveryStage: String
        var estimatedArrival: Date
    }
    
    var restaurantName: String
    var orderID: String
}

// MARK: - Live Activity Manager
@Observable
final class LiveActivityManager {
    
    // MARK: - Start Timer Activity (AI-Enhanced)
    func startTimerActivity(
        title: String,
        durationSeconds: Int,
        relevanceScore: Double = 1.0 // High priority for system visibility
    ) {
        let initialState = TimerAttributes.ContentState(
            timeRemaining: durationSeconds,
            isRunning: true
        )
        
        let attributes = TimerAttributes(
            title: title,
            startTime: Date()
        )
        
        do {
            let activity = try Activity.request(
                attributes: attributes,
                contentState: initialState,
                pushType: nil,
                relevanceScore: relevanceScore // New in iOS 18/2026: Priority level
            )
            print("Live activity started: \(activity.id)")
            
            // Update activity periodically
            updateTimerActivity(activity, remaining: durationSeconds)
        } catch {
            print("Error starting live activity: \(error)")
        }
    }

// MARK: - App Intent for Remote Updates (Xcode 26 Pattern)
struct UpdateActivityIntent: AppIntent {
    static var title: LocalizedStringResource = "Update My Activity"
    
    @Parameter(title: "Status")
    var status: String
    
    func perform() async throws -> some IntentResult {
        // This intent can be triggered by Apple Intelligence to update the Live Activity
        // based on background reasoning (e.g., user's location or context).
        let manager = LiveActivityManager()
        manager.updateDeliveryStatus(stage: status, estimatedArrival: Date().addingTimeInterval(300))
        return .result()
    }
}
    
    // MARK: - Update Timer Activity
    private func updateTimerActivity(
        _ activity: Activity<TimerAttributes>,
        remaining: Int
    ) {
        Task {
            for second in stride(from: remaining, through: 0, by: -1) {
                let updatedState = TimerAttributes.ContentState(
                    timeRemaining: second,
                    isRunning: second > 0
                )
                
                await activity.update(
                    using: updatedState,
                    alertConfiguration: nil
                )
                
                if second > 0 {
                    try? await Task.sleep(nanoseconds: 1_000_000_000)
                }
            }
            
            // End activity when timer completes
            await activity.end(
                using: TimerAttributes.ContentState(
                    timeRemaining: 0,
                    isRunning: false
                ),
                dismissalPolicy: .default
            )
        }
    }
    
    // MARK: - Start Delivery Activity
    func startDeliveryActivity(
        restaurantName: String,
        orderID: String,
        estimatedMinutes: Int
    ) {
        let estimatedArrival = Calendar.current.date(
            byAdding: .minute,
            value: estimatedMinutes,
            to: Date()
        ) ?? Date()
        
        let initialState = DeliveryAttributes.ContentState(
            deliveryStage: "Preparing",
            estimatedArrival: estimatedArrival
        )
        
        let attributes = DeliveryAttributes(
            restaurantName: restaurantName,
            orderID: orderID
        )
        
        do {
            let activity = try Activity.request(
                attributes: attributes,
                contentState: initialState,
                pushType: nil
            )
            print("Delivery activity started: \(activity.id)")
        } catch {
            print("Error starting delivery activity: \(error)")
        }
    }
    
    // MARK: - Update Delivery Status
    func updateDeliveryStatus(
        stage: String,
        estimatedArrival: Date
    ) {
        Task {
            for activity in Activity<DeliveryAttributes>.all {
                let updatedState = DeliveryAttributes.ContentState(
                    deliveryStage: stage,
                    estimatedArrival: estimatedArrival
                )
                
                await activity.update(using: updatedState)
            }
        }
    }
    
    // MARK: - End All Activities
    func endAllActivities() {
        Task {
            for activity in Activity<TimerAttributes>.all {
                await activity.end(dismissalPolicy: .immediate)
            }
            
            for activity in Activity<DeliveryAttributes>.all {
                await activity.end(dismissalPolicy: .immediate)
            }
        }
    }
}

// MARK: - Dynamic Island View
struct TimerLiveActivityView: View {
    let context: ActivityViewContext<TimerAttributes>
    
    var timeString: String {
        let minutes = context.state.timeRemaining / 60
        let seconds = context.state.timeRemaining % 60
        return String(format: "%02d:%02d", minutes, seconds)
    }
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(context.attributes.title)
                    .font(.headline)
                Text(timeString)
                    .font(.system(.largeTitle, design: .monospaced))
                    .fontWeight(.bold)
            }
            
            Spacer()
            
            if context.state.isRunning {
                Image(systemName: "play.circle.fill")
                    .font(.title)
                    .foregroundStyle(.blue)
            }
        }
        .padding()
        .background(Color(UIColor.systemBackground))
    }
}

struct DeliveryLiveActivityView: View {
    let context: ActivityViewContext<DeliveryAttributes>
    
    var timeUntilArrival: String {
        let interval = context.state.estimatedArrival.timeIntervalSinceNow
        let minutes = Int(interval / 60)
        return minutes > 0 ? "\(minutes) min" : "Arriving"
    }
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(context.attributes.restaurantName)
                    .font(.headline)
                Text(context.state.deliveryStage)
                    .font(.caption)
                    .foregroundStyle(.gray)
            }
            
            Spacer()
            
            VStack(alignment: .trailing) {
                Text(timeUntilArrival)
                    .font(.headline)
                Text("Order #\(context.attributes.orderID)")
                    .font(.caption)
                    .foregroundStyle(.gray)
            }
        }
        .padding()
        .background(Color(UIColor.systemBackground))
    }
}

// Usage
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

struct TimerView: View {
    @State private var liveActivityManager = LiveActivityManager()
    
    var body: some View {
        VStack {
            Button("Start 5-Min Timer") {
                liveActivityManager.startTimerActivity(
                    title: "Cooking",
                    durationSeconds: 300
                )
            }
            
            Button("Start Delivery Tracking") {
                liveActivityManager.startDeliveryActivity(
                    restaurantName: "Pizza Place",
                    orderID: "12345",
                    estimatedMinutes: 20
                )
            }
            
            Button("Update Delivery Status") {
                liveActivityManager.updateDeliveryStatus(
                    stage: "On the way",
                    estimatedArrival: Date(timeIntervalSinceNow: 600)
                )
            }
            
            Button("End All Activities") {
                liveActivityManager.endAllActivities()
            }
        }
        .padding()
    }
}
```

## ActivityKit Configuration

### Info.plist Additions

```xml
<key>NSSupportsLiveActivities</key>
<true/>
<key>NSSupportsLiveActivitiesFrequentUpdates</key>
<true/>
```

### Capabilities

```
1. Target → Signing & Capabilities
2. Add "Live Activities" capability
3. Enable for main app
4. Configure entitlements
```

## Implementation Checklist

- [ ] Define ActivityAttributes struct
- [ ] Create ContentState codable model
- [ ] Implement LiveActivityManager
- [ ] Create Live Activity view layouts
- [ ] Test on device with Dynamic Island
- [ ] Implement periodic updates
- [ ] Add dismissal policies
- [ ] Configure push notification updates
- [ ] Test on iPhone with Dynamic Island
- [ ] Monitor activity lifecycle
- [ ] Handle app termination
- [ ] Test end-to-end update flow
