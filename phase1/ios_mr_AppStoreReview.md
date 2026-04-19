---
name: app-store-review
description: |
  App Store review guidelines checklist, ATT (App Tracking Transparency),
  privacy manifests, PrivacyInfo.xcprivacy, rejection prevention strategies.
trigger: "Prepare app for App Store submission"
---

# App Store Review — Guidelines, ATT, Privacy Manifests

**Purpose:** Navigate App Store review process without rejections. Privacy manifests, ATT compliance, guideline violations prevention.

**When to use:**
- Before first submission
- Updates after major changes
- When adding user tracking
- When adding health/payment features

**Why this matters:** One guideline violation = 2 week rejection delay. Privacy manifest missing = automatic rejection iOS 17+.

---

## 1. Privacy Manifest (PrivacyInfo.xcprivacy)

```swift
// Create in Xcode:
// File → New → Property List
// Name: PrivacyInfo.xcprivacy
// Target: Your main app target
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Declare what data types you collect -->
    <key>NSPrivacyTracking</key>
    <false/>  <!-- Set true if you track users across apps -->
    
    <key>NSPrivacyTrackingDomains</key>
    <array>
        <!-- List domains you use for tracking (if any) -->
    </array>
    
    <!-- List all APIs you use that access sensitive data -->
    <key>NSPrivacyAccessedAPICategoryRequiresMostMinimalizedDataUse</key>
    <array>
        <!-- File timestamp -->
        <dict>
            <key>NSPrivacyAccessedAPICategory</key>
            <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
            <key>NSPrivacyAccessedAPICategoryReasons</key>
            <array>
                <string>CA92.1</string>  <!-- For app functionality -->
            </array>
        </dict>
        
        <!-- User defaults -->
        <dict>
            <key>NSPrivacyAccessedAPICategory</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPICategoryReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
        
        <!-- Pasteboard -->
        <dict>
            <key>NSPrivacyAccessedAPICategory</key>
            <string>NSPrivacyAccessedAPICategoryPasteboard</string>
            <key>NSPrivacyAccessedAPICategoryReasons</key>
            <array>
                <string>C617.1</string>  <!-- User-initiated paste -->
            </array>
        </dict>
        
        <!-- Keychain -->
        <dict>
            <key>NSPrivacyAccessedAPICategory</key>
            <string>NSPrivacyAccessedAPICategorySecureStorage</string>
            <key>NSPrivacyAccessedAPICategoryReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
    </array>
    
    <!-- Declare data collection practices -->
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <!-- Email address -->
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeEmailAddress</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <true/>  <!-- Linked to user identity -->
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/> <!-- Not used for tracking -->
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
                <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
            </array>
        </dict>
        
        <!-- User ID -->
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeUserID</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <true/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

---

## 2. ATT (App Tracking Transparency) Implementation

```swift
// AppDelegate or App.swift
import AppTrackingTransparency
import AdSupport

@main
struct MyApp: App {
    @Environment(\.scenePhase) var scenePhase
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .onChange(of: scenePhase) { _, newPhase in
            if newPhase == .active {
                requestATTPermission()
            }
        }
    }
    
    private func requestATTPermission() {
        // Request ATT permission after app launches
        // User should see native system popup
        ATTrackingManager.requestTrackingAuthorization { status in
            DispatchQueue.main.async {
                switch status {
                case .authorized:
                    // User allowed tracking
                    let idfa = ASIdentifierManager.shared().advertisingIdentifier.uuidString
                    print("IDFA: \(idfa)")
                case .denied:
                    print("User denied tracking")
                case .notDetermined:
                    print("Not determined yet")
                case .restricted:
                    print("Tracking restricted")
                @unknown default:
                    break
                }
            }
        }
    }
}
```

```xml
<!-- Info.plist -->
<key>NSUserTrackingUsageDescription</key>
<string>We use your advertising ID to deliver personalized ads and measure campaign effectiveness.</string>
```

---

## 3. App Store Review Checklist

```swift
// AppStoreSubmissionChecklist.swift
// Use this as reference before submission

struct AppStoreSubmissionChecklist {
    static let items = [
        // FUNCTIONALITY
        ChecklistItem(
            category: "Functionality",
            task: "All features work without crashes",
            required: true
        ),
        ChecklistItem(
            category: "Functionality",
            task: "App doesn't use private APIs",
            required: true
        ),
        ChecklistItem(
            category: "Functionality",
            task: "No hard-coded server addresses or credentials",
            required: true
        ),
        ChecklistItem(
            category: "Functionality",
            task: "Network calls use HTTPS (not HTTP)",
            required: true
        ),
        
        // PERFORMANCE
        ChecklistItem(
            category: "Performance",
            task: "App launches in < 20 seconds",
            required: true
        ),
        ChecklistItem(
            category: "Performance",
            task: "No memory leaks (check with Instruments)",
            required: true
        ),
        ChecklistItem(
            category: "Performance",
            task: "Handles offline gracefully",
            required: true
        ),
        
        // PRIVACY & SECURITY
        ChecklistItem(
            category: "Privacy",
            task: "Privacy manifest included",
            required: true
        ),
        ChecklistItem(
            category: "Privacy",
            task: "ATT implemented if tracking users",
            required: true
        ),
        ChecklistItem(
            category: "Privacy",
            task: "User data encrypted at rest",
            required: true
        ),
        ChecklistItem(
            category: "Privacy",
            task: "Privacy policy URL provided",
            required: true
        ),
        ChecklistItem(
            category: "Privacy",
            task: "No selling user data to third parties",
            required: true
        ),
        
        // PAYMENTS
        ChecklistItem(
            category: "Payments",
            task: "In-app purchases use StoreKit 2",
            required: false
        ),
        ChecklistItem(
            category: "Payments",
            task: "No external payment methods (outside IAP)",
            required: true
        ),
        
        // UI/UX
        ChecklistItem(
            category: "UI",
            task: "Supports all screen sizes (iPhone SE to Max)",
            required: true
        ),
        ChecklistItem(
            category: "UI",
            task: "Supports Dark Mode",
            required: false
        ),
        ChecklistItem(
            category: "UI",
            task: "No hardcoded phone numbers or email addresses",
            required: false
        ),
        
        // CONTENT
        ChecklistItem(
            category: "Content",
            task: "App description accurate and complete",
            required: true
        ),
        ChecklistItem(
            category: "Content",
            task: "Keywords match app content",
            required: true
        ),
        ChecklistItem(
            category: "Content",
            task: "No misleading claims",
            required: true
        ),
        ChecklistItem(
            category: "Content",
            task: "Screenshots show real app (not mock-ups)",
            required: true
        ),
        
        // GUIDELINES
        ChecklistItem(
            category: "Guidelines",
            task: "No gambling or betting",
            required: true
        ),
        ChecklistItem(
            category: "Guidelines",
            task: "No adult content (if rated 4+/12+)",
            required: true
        ),
        ChecklistItem(
            category: "Guidelines",
            task: "No medical diagnosis apps (unless FDA approved)",
            required: true
        ),
        ChecklistItem(
            category: "Guidelines",
            task: "No apps that copy other apps exactly",
            required: true
        ),
    ]
}

struct ChecklistItem {
    let category: String
    let task: String
    let required: Bool
}
```

---

## 4. Common Rejection Reasons & Fixes

```swift
// Common rejections and preventions

/*
 REJECTION: "Inadequate privacy practices"
 CAUSE: Privacy manifest missing or incomplete
 FIX: Update PrivacyInfo.xcprivacy with all APIs you use
*/

/*
 REJECTION: "This app only contains promotional or marketing material"
 CAUSE: App does nothing useful
 FIX: Implement core features, not just ads/marketing
*/

/*
 REJECTION: "The app doesn't include adequate content"
 CAUSE: App only shows static info
 FIX: Add interactivity, user input, or dynamic content
*/

/*
 REJECTION: "Crash on launch"
 CAUSE: Missing data, bad network, bad Swift version
 FIX: Test on multiple devices, iOS versions. Handle errors gracefully.
*/

/*
 REJECTION: "URLSession certificate validation failing"
 CAUSE: SSL certificate chain issues
 FIX: Use valid SSL cert, update ATS config, test with real server (not localhost)
*/

/*
 REJECTION: "External payment methods detected"
 CAUSE: Using Stripe/PayPal instead of In-App Purchase
 FIX: Route payments through StoreKit or RevenueCat
*/

/*
 REJECTION: "Requires location permission but doesn't use location"
 CAUSE: Unused permission
 FIX: Remove unnecessary permissions from Info.plist
*/
```

---

## 5. Build & Submit Automation

```swift
// Fastlane build and submission (Phase 2 skill)
// For now, manual steps:

/*
 Steps to submit:
 
 1. Bump version in Xcode
    - Project → General → Version (e.g., 1.0.0)
    - Build Number (increment: 1 → 2)
 
 2. Archive for distribution
    - Product → Scheme → Edit Scheme → Release
    - Product → Archive
    - Wait for signing
 
 3. Upload to App Store Connect
    - Window → Organizer
    - Select build
    - Distribute App
    - Follow wizard (manual review, not automatic)
 
 4. Submit for review
    - App Store Connect website
    - Apps → Your App → Version
    - Build → Select your build
    - Add info (whats new, demo account if needed)
    - Submit for Review
 
 5. Monitor status
    - Check email for rejections
    - Fix issues
    - Resubmit
*/
```

---

## 6. Phased Rollout Strategy

```swift
// After approval, don't release to 100% immediately
// Use phased rollout to catch issues early

struct ReleasePlan {
    let day1: Double = 0.05   // 5%
    let day3: Double = 0.25   // 25%
    let day7: Double = 0.50   // 50%
    let day14: Double = 1.0   // 100%
    
    /*
     Monitoring during rollout:
     - Crash rate (should be < 0.1%)
     - User ratings (should stay > 4.0)
     - Sentry alerts (critical errors)
     - Analytics (funnel completion rate)
     
     If issues:
     - Immediately pause rollout
     - Fix bug
     - Increment build number
     - Resubmit
    */
}
```

---

## 7. Version History Template

```swift
// Keep track of submissions
struct AppVersion {
    let version: String = "1.0.0"
    let buildNumber: Int = 1
    let submissionDate: Date
    let status: String = "In Review"  // "Approved", "Rejected", "In Review"
    let rejectionReason: String? = nil
    let notes: String = """
    - Initial release
    - Authentication with Face ID
    - Offline support with SwiftData
    - Sentry crash reporting
    """
}
```

---

## Checklist: Before Submission

- [ ] PrivacyInfo.xcprivacy created and accurate
- [ ] ATT permission request if tracking
- [ ] All required permissions in Info.plist
- [ ] No hardcoded secrets or test credentials
- [ ] No debug logs or test analytics calls
- [ ] HTTPS only (no HTTP)
- [ ] Privacy policy URL working
- [ ] App tested on simulator (multiple iOS versions)
- [ ] App tested on real device
- [ ] Handles offline mode gracefully
- [ ] Crash rate tested (should be near 0%)
- [ ] Screenshots accurate and professional
- [ ] App description complete and honest
- [ ] Version number bumped
- [ ] Build number incremented
- [ ] Signed with correct certificate
- [ ] Phased rollout enabled (start at 5%)

---

## Related Skills

- **ios-security** — Encryption and permissions
- **sentry-setup** (Phase 2) — Crash monitoring
- **fastlane-automation** (Phase 2) — Automated submission

---

**Status:** ✅ Checklist current as of April 2026, iOS 17.4+
