---
name: analytics-integration
description: Event tracking and analytics with PostHog or Mixpanel. Use this skill when implementing user behavior tracking, funnel analysis, cohort tracking, or analytics dashboards. Essential for understanding user behavior and making data-driven product decisions.
compatibility: iOS 15+, URLSession
---

# Analytics Integration — Event Tracking & User Insights

**When to use:** Alongside Phase 1 launch to track user behavior and growth metrics.

## Pattern: Analytics Service

```swift
import Foundation

@Observable
final class AnalyticsService {
    static let shared = AnalyticsService()
    
    private let apiKey: String = "pk_live_..."  // PostHog API key
    private let baseURL = "https://app.posthog.com"
    
    enum Event: String {
        case appOpened = "app_opened"
        case loginStarted = "login_started"
        case loginSucceeded = "login_succeeded"
        case loginFailed = "login_failed"
        case itemViewed = "item_viewed"
        case itemPurchased = "item_purchased"
        case featureUsed = "feature_used"
        case errorOccurred = "error_occurred"
    }
    
    // MARK: - Track Event
    func track(
        _ event: Event,
        properties: [String: Any] = [:]
    ) {
        var payload: [String: Any] = [
            "api_key": apiKey,
            "event": event.rawValue,
            "timestamp": ISO8601DateFormatter().string(from: Date()),
            "distinct_id": getUserID(),
            "properties": properties
        ]
        
        // Add device info
        var deviceProperties = properties
        deviceProperties["app_version"] = Bundle.main.appVersion
        deviceProperties["os_version"] = UIDevice.current.systemVersion
        deviceProperties["device_model"] = UIDevice.current.model
        payload["properties"] = deviceProperties
        
        sendToAnalytics(payload: payload)
    }
    
    // MARK: - Track Funnel Step
    func trackFunnel(
        name: String,
        step: Int,
        totalSteps: Int,
        properties: [String: Any] = [:]
    ) {
        var props = properties
        props["funnel_name"] = name
        props["funnel_step"] = step
        props["funnel_total"] = totalSteps
        props["completion_percent"] = Int((Double(step) / Double(totalSteps)) * 100)
        
        track(.featureUsed, properties: props)
    }
    
    // MARK: - Track Error
    func trackError(
        _ error: Error,
        context: String = ""
    ) {
        var properties: [String: Any] = [
            "error_description": error.localizedDescription,
            "error_type": String(describing: type(of: error))
        ]
        
        if !context.isEmpty {
            properties["context"] = context
        }
        
        track(.errorOccurred, properties: properties)
    }
    
    // MARK: - Set User Property
    func setUserProperty(
        key: String,
        value: Any
    ) {
        var payload: [String: Any] = [
            "api_key": apiKey,
            "distinct_id": getUserID(),
            "$set": [key: value],
            "timestamp": ISO8601DateFormatter().string(from: Date())
        ]
        
        sendToAnalytics(payload: payload)
    }
    
    // MARK: - Private Helpers
    private func sendToAnalytics(payload: [String: Any]) {
        Task {
            do {
                var request = URLRequest(url: URL(string: "\(baseURL)/capture/")!)
                request.httpMethod = "POST"
                request.setValue("application/json", forHTTPHeaderField: "Content-Type")
                request.httpBody = try JSONSerialization.data(withJSONObject: payload)
                
                let (_, response) = try await URLSession.shared.data(for: request)
                
                if let httpResponse = response as? HTTPURLResponse {
                    if httpResponse.statusCode == 200 {
                        print("Analytics tracked: \(payload["event"] as? String ?? "unknown")")
                    } else {
                        print("Analytics error: \(httpResponse.statusCode)")
                    }
                }
            } catch {
                print("Failed to send analytics: \(error)")
            }
        }
    }
    
    private func getUserID() -> String {
        let key = "analytics_user_id"
        if let userID = UserDefaults.standard.string(forKey: key) {
            return userID
        }
        
        let userID = UUID().uuidString
        UserDefaults.standard.set(userID, forKey: key)
        return userID
    }
}

// MARK: - Usage Examples
struct LoginViewModel {
    private let authService = AuthService.shared
    
    func login(email: String, password: String) async {
        AnalyticsService.shared.track(.loginStarted, properties: ["email_domain": email.split(separator: "@").last ?? ""])
        
        do {
            try await authService.login(email: email, password: password)
            AnalyticsService.shared.track(.loginSucceeded)
            AnalyticsService.shared.setUserProperty(key: "last_login", value: Date())
        } catch {
            AnalyticsService.shared.track(.loginFailed, properties: ["error": error.localizedDescription])
            AnalyticsService.shared.trackError(error, context: "login")
        }
    }
}

struct PurchaseView: View {
    @State private var viewModel = PurchaseViewModel()
    
    var body: some View {
        VStack {
            Button("Buy Premium") {
                AnalyticsService.shared.trackFunnel(
                    name: "purchase_flow",
                    step: 1,
                    totalSteps: 3
                )
            }
        }
    }
}
```

## Dashboard Setup (PostHog)

```
Metrics to Track:
- Daily Active Users (DAU)
- Monthly Active Users (MAU)
- Login funnel conversion rate
- Feature adoption rate
- Purchase conversion rate
- Session duration
- Error rate by type
- User retention cohorts
```

## Implementation Checklist

- [ ] Set up PostHog or Mixpanel account
- [ ] Get API key and configure endpoint
- [ ] Implement AnalyticsService class
- [ ] Add event tracking to critical user flows
- [ ] Track funnel steps (signup, login, purchase)
- [ ] Monitor error tracking
- [ ] Set up user properties
- [ ] Create custom dashboard for key metrics
- [ ] Configure cohort analysis
- [ ] Set retention tracking
- [ ] Monitor daily and weekly active users
- [ ] Review analytics weekly for insights
