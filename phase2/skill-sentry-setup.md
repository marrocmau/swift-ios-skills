---
name: sentry-setup
description: Crash reporting and performance monitoring with Sentry SDK. Use this skill when implementing error tracking, monitoring app performance, analyzing crash patterns, or setting up production observability. Essential for understanding what goes wrong in production and preventing silent failures.
compatibility: iOS 15+, Sentry SDK
---

# Sentry Setup — Crash Reporting & Performance Monitoring

**When to use:** Right before TestFlight submission or going live. Every production app needs crash reporting.

## Pattern: Sentry Integration

```swift
import Sentry

// App.swift
@main
struct MyApp: App {
    init() {
        SentrySDK.start { options in
            options.dsn = "https://[key]@[project].ingest.sentry.io/[id]"
            options.tracesSampleRate = 1.0
            options.environment = "production"
        }
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// In ViewModel
@Observable
final class MyViewModel {
    func trackError(_ error: Error) {
        SentrySDK.captureException(error)
    }
    
    func trackPerformance(_ operation: String, duration: TimeInterval) {
        let transaction = SentrySDK.startTransaction(name: operation, operation: "custom")
        transaction?.finish()
    }
}
```

## Implementation Checklist

- [ ] Sign up at sentry.io
- [ ] Create iOS project and get DSN
- [ ] Install Sentry via SPM: `https://github.com/getsentry/sentry-swift.git`
- [ ] Initialize in App.swift with your DSN
- [ ] Test crash capture in development
- [ ] Configure sample rates (adjust tracesSampleRate for production)
- [ ] Set environment (development/staging/production)
- [ ] Test error capturing before submission
