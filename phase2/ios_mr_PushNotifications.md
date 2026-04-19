---
name: push-notifications
description: Push notifications and rich notifications with APNs. Use this skill when implementing remote notifications, local notification scheduling, notification handling, or rich notification content. Essential for user engagement and retention.
compatibility: iOS 15+, UserNotifications framework
---

# Push Notifications — APNs & Local Notifications

**When to use:** After Phase 1 launch to drive user engagement and retention.

## Pattern: Notification Management

```swift
import UserNotifications
import SwiftUI

@Observable
final class NotificationManager {
    
    // MARK: - Request Permission
    @MainActor
    func requestNotificationPermission() async {
        do {
            let granted = try await UNUserNotificationCenter.current()
                .requestAuthorization(options: [.alert, .sound, .badge])
            if granted {
                DispatchQueue.main.async {
                    UIApplication.shared.registerForRemoteNotifications()
                }
                print("Notification permission granted")
            }
        } catch {
            print("Notification permission error: \(error)")
        }
    }
    
    // MARK: - Local Notification
    func sendLocalNotification(
        title: String,
        body: String,
        delay: TimeInterval,
        sound: UNNotificationSound = .default,
        badge: NSNumber? = nil
    ) {
        let content = UNMutableNotificationContent()
        content.title = title
        content.body = body
        content.sound = sound
        
        if let badge = badge {
            content.badge = badge
        }
        
        let trigger = UNTimeIntervalNotificationTrigger(
            timeInterval: delay,
            repeats: false
        )
        let request = UNNotificationRequest(
            identifier: UUID().uuidString,
            content: content,
            trigger: trigger
        )
        
        UNUserNotificationCenter.current().add(request) { error in
            if let error = error {
                print("Error scheduling notification: \(error)")
            }
        }
    }
    
    // MARK: - Rich Notification with Image
    func sendRichNotification(
        title: String,
        body: String,
        imageURL: String?,
        delay: TimeInterval
    ) {
        let content = UNMutableNotificationContent()
        content.title = title
        content.body = body
        
        if let imageURL = imageURL, let url = URL(string: imageURL) {
            if let imageData = try? Data(contentsOf: url),
               let attachment = try? UNNotificationAttachment(
                identifier: UUID().uuidString,
                data: imageData,
                options: nil
            ) {
                content.attachments = [attachment]
            }
        }
        
        let trigger = UNTimeIntervalNotificationTrigger(
            timeInterval: delay,
            repeats: false
        )
        let request = UNNotificationRequest(
            identifier: UUID().uuidString,
            content: content,
            trigger: trigger
        )
        
        UNUserNotificationCenter.current().add(request) { error in
            if let error {
                print("Error scheduling rich notification: \(error)")
            }
        }
    }
}

// MARK: - App Delegate for Remote Notifications
class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil
    ) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        return true
    }
    
    func application(
        _ application: UIApplication,
        didReceiveRemoteNotification userInfo: [AnyHashable : Any]
    ) async -> UIBackgroundFetchResult {
        // Handle push notification payload
        print("Received remote notification: \(userInfo)")
        return .newData
    }
    
    func application(
        _ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
    ) {
        let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
        print("Device token: \(token)")
        // Send to backend for push notifications
    }
}

// MARK: - Notification Delegate
extension AppDelegate: UNUserNotificationCenterDelegate {
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification
    ) async -> UNNotificationPresentationOptions {
        let userInfo = notification.request.content.userInfo
        print("Notification received in foreground: \(userInfo)")
        
        return [.banner, .sound, .badge]
    }
    
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let userInfo = response.notification.request.content.userInfo
        
        // Handle notification tap
        if response.actionIdentifier == UNNotificationDefaultActionIdentifier {
            print("User tapped notification: \(userInfo)")
            // Navigate to relevant screen
        }
    }
}

// Usage in SwiftUI
struct ContentView: View {
    @State private var notificationManager = NotificationManager()
    
    var body: some View {
        VStack(spacing: 20) {
            Button("Request Permission") {
                Task {
                    await notificationManager.requestNotificationPermission()
                }
            }
            
            Button("Send Test Notification") {
                notificationManager.sendLocalNotification(
                    title: "Test Notification",
                    body: "This is a test notification",
                    delay: 5
                )
            }
        }
        .padding()
    }
}
```

## Implementation Checklist

- [ ] Implement NotificationManager class
- [ ] Request user permission for notifications
- [ ] Set up AppDelegate notification handling
- [ ] Register for remote notifications
- [ ] Implement UNUserNotificationCenterDelegate
- [ ] Handle notifications in foreground
- [ ] Handle notification taps and actions
- [ ] Test with local notifications
- [ ] Configure APNs certificate in Apple Developer
- [ ] Set up backend to send push notifications
- [ ] Monitor notification delivery and engagement
- [ ] Implement notification preferences in settings
