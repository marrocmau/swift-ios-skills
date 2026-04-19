---
name: in-app-review-prompt
description: In-app review requests and rating prompts using StoreKit. Use this skill when requesting user ratings, tracking review prompts, timing review requests, or implementing smart review dialogs. Essential for maintaining high app store ratings and visibility.
compatibility: iOS 16+, StoreKit 2
---

# In-App Review Prompt — Rating Requests

**When to use:** After key user achievements (completed task, purchased item, etc.). Timing is critical.

## Pattern: Smart Review Prompt Manager

```swift
import StoreKit

@Observable
final class ReviewPromptManager {
    
    // Configuration
    private let minSessionsBeforePrompt = 3
    private let minDaysBeforeReprompt = 30
    private let maxPromptsPerYear = 3
    
    // Tracking
    @AppStorage("reviewPromptCount") var promptCount = 0
    @AppStorage("lastReviewPromptDate") var lastPromptDate = Date(timeIntervalSince1970: 0)
    @AppStorage("userSessions") var userSessions = 0
    @AppStorage("appInstallDate") var appInstallDate = Date()
    
    // MARK: - Check and Prompt for Review
    func promptForReviewIfNeeded() {
        userSessions += 1
        
        // Don't exceed max prompts per year
        guard promptCount < maxPromptsPerYear else {
            return
        }
        
        // Require minimum sessions
        guard userSessions >= minSessionsBeforePrompt else {
            return
        }
        
        // Check days since last prompt
        let daysSinceLastPrompt = Calendar.current.dateComponents(
            [.day],
            from: lastPromptDate,
            to: Date()
        ).day ?? 0
        
        guard daysSinceLastPrompt >= minDaysBeforeReprompt else {
            return
        }
        
        // Request review
        requestReview()
    }
    
    // MARK: - Request Review After Win Moment
    func promptAfterWinMoment(_ moment: WinMoment) {
        // Win moments: completed task, made purchase, unlocked achievement, etc.
        
        userSessions += 1
        
        guard promptCount < maxPromptsPerYear else {
            return
        }
        
        // More aggressive for win moments
        let daysSinceLastPrompt = Calendar.current.dateComponents(
            [.day],
            from: lastPromptDate,
            to: Date()
        ).day ?? 0
        
        guard daysSinceLastPrompt >= 7 else {  // Only 7 days instead of 30
            return
        }
        
        requestReview()
    }
    
    // MARK: - Private Request
    @MainActor
    private func requestReview() {
        Task {
            guard let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene else {
                return
            }
            
            do {
                try await SKStoreReviewController.requestReview(in: scene)
                
                // Update tracking
                lastPromptDate = Date()
                promptCount += 1
                
                print("Review prompt shown (count: \(promptCount))")
            } catch {
                print("Failed to show review prompt: \(error)")
            }
        }
    }
    
    // MARK: - Reset (for testing or new app version)
    func resetForNewVersion() {
        promptCount = 0
        lastPromptDate = Date(timeIntervalSince1970: 0)
    }
}

// MARK: - Win Moments Enum
enum WinMoment {
    case taskCompleted
    case purchaseMade
    case achievementUnlocked
    case streakReached
    case milestoneHit
}

// Usage in App
@main
struct MyApp: App {
    @State private var reviewManager = ReviewPromptManager()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onAppear {
                    // Check for review on app open
                    reviewManager.promptForReviewIfNeeded()
                }
        }
    }
}

// Usage in Views
struct TaskCompletionView: View {
    @State private var reviewManager = ReviewPromptManager()
    
    var body: some View {
        VStack {
            Text("Task completed!")
                .font(.title)
            
            Button("Continue") {
                // Track the win moment
                reviewManager.promptAfterWinMoment(.taskCompleted)
            }
        }
    }
}

struct PurchaseConfirmationView: View {
    @State private var reviewManager = ReviewPromptManager()
    
    var body: some View {
        VStack {
            Text("Purchase successful!")
            
            Button("Done") {
                reviewManager.promptAfterWinMoment(.purchaseMade)
            }
        }
    }
}
```

## Review Timing Strategy

```
DON'T prompt for review:
- On first launch
- During onboarding
- After negative actions (error, failed purchase)
- More than 3 times per year
- Less than 30 days apart
- If user deleted app before

DO prompt for review:
- After completing a task
- After making a purchase
- After using a feature successfully
- After 3+ app sessions
- At least 30 days after previous prompt
- Max 3 times per year
- After "win moments" (achievements, milestones)
```

## Implementation Checklist

- [ ] Import StoreKit 2
- [ ] Create ReviewPromptManager class
- [ ] Configure min sessions before first prompt
- [ ] Set minimum days between prompts
- [ ] Limit prompts to max 3 per year
- [ ] Identify "win moments" in your app
- [ ] Call prompt after key user actions
- [ ] Test with SKStoreReviewController
- [ ] Monitor review count in UserDefaults
- [ ] Track when prompts are shown
- [ ] Avoid prompting during negative moments
- [ ] Plan quarterly review strategy
