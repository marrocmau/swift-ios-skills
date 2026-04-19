---
name: revenuecat-subscription
description: In-app subscriptions and monetization with RevenueCat. Use this skill when implementing subscription flows, paywalls, purchase handling, or managing app monetization. Essential for generating recurring revenue and managing entitlements across iOS, Android, and web.
compatibility: iOS 15+, RevenueCat SDK
---

# RevenueCat Subscription — In-App Purchases & Monetization

**When to use:** After Phase 1 launch, when ready to implement monetization. Critical for sustainable app revenue.

## Pattern: RevenueCat Integration

```swift
import RevenueCat

@Observable
final class SubscriptionViewModel {
    var offerings: Offerings?
    var customerInfo: CustomerInfo?
    var isSubscribed: Bool = false
    var selectedPackage: Package?
    
    func loadOfferings() async {
        do {
            let offerings = try await Purchases.shared.offerings()
            self.offerings = offerings
        } catch {
            print("Error loading offerings: \(error)")
        }
    }
    
    func purchaseSubscription(_ package: Package) async throws {
        let result = try await Purchases.shared.purchase(package: package)
        self.customerInfo = result.customerInfo
        self.isSubscribed = result.customerInfo.entitlements.active.isEmpty == false
    }
    
    func restorePurchases() async throws {
        let customerInfo = try await Purchases.shared.restorePurchases()
        self.customerInfo = customerInfo
        self.isSubscribed = customerInfo.entitlements.active.isEmpty == false
    }
    
    func checkSubscriptionStatus() async {
        do {
            let customerInfo = try await Purchases.shared.customerInfo()
            self.customerInfo = customerInfo
            self.isSubscribed = customerInfo.entitlements.active.isEmpty == false
        } catch {
            print("Error checking subscription: \(error)")
        }
    }
}

// Usage in View
struct PaywallView: View {
    @State private var viewModel = SubscriptionViewModel()
    
    var body: some View {
        VStack(spacing: 20) {
            Text("Unlock Premium")
                .font(.title)
            
            if let offerings = viewModel.offerings?.current {
                VStack(spacing: 12) {
                    ForEach(offerings.availablePackages, id: \.identifier) { package in
                        Button(action: {
                            Task {
                                do {
                                    try await viewModel.purchaseSubscription(package)
                                } catch {
                                    print("Purchase failed: \(error)")
                                }
                            }
                        }) {
                            HStack {
                                VStack(alignment: .leading) {
                                    Text(package.localizedPriceString)
                                        .font(.title3)
                                    Text(package.storeProduct.localizedTitle)
                                        .font(.caption)
                                }
                                Spacer()
                                Image(systemName: "chevron.right")
                            }
                            .padding()
                            .background(Color.blue.opacity(0.1))
                            .cornerRadius(8)
                        }
                    }
                }
            }
            
            Button("Restore Purchases") {
                Task {
                    try? await viewModel.restorePurchases()
                }
            }
            .foregroundStyle(.blue)
        }
        .padding()
        .task {
            await viewModel.loadOfferings()
            await viewModel.checkSubscriptionStatus()
        }
    }
}
```

## Implementation Checklist

- [ ] Sign up at RevenueCat.com
- [ ] Create iOS app and configure offerings
- [ ] Install RevenueCat via SPM
- [ ] Configure with API key in App.swift
- [ ] Create SubscriptionViewModel with Purchases integration
- [ ] Build paywall UI with offerings
- [ ] Test purchase flow in Sandbox
- [ ] Configure entitlements and feature gating
- [ ] Handle restore purchases
- [ ] Monitor MRR and churn metrics
