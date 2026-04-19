---
name: storekit2-advanced
description: Advanced StoreKit 2 monetization including family sharing, refund handling, and subscription management. Use this skill when implementing complex in-app purchase flows, subscription groups, family plans, or managing transaction state. Essential for advanced monetization strategies.
compatibility: iOS 15+, StoreKit 2
---

# StoreKit 2 Advanced — Complex Monetization

**When to use:** For advanced monetization needs beyond basic subscriptions.

## Pattern: Advanced StoreKit Management

```swift
import StoreKit

@Observable
final class AdvancedStoreKitManager {
    
    @MainActor var products: [Product] = []
    @MainActor var purchasedProductIDs: Set<String> = []
    @MainActor var subscriptionStatus: String = ""
    
    // MARK: - Fetch Products
    @MainActor
    func fetchProducts() async {
        do {
            let productIDs = [
                "com.myapp.monthly",
                "com.myapp.yearly",
                "com.myapp.premium",
                "com.myapp.family"
            ]
            
            let products = try await Product.products(for: productIDs)
            self.products = products.sorted { lhs, rhs in
                lhs.price < rhs.price
            }
        } catch {
            print("Error fetching products: \(error)")
        }
    }
    
    // MARK: - Handle Purchase with Transaction Listener
    @MainActor
    func purchase(_ product: Product) async {
        do {
            let result = try await product.purchase()
            
            switch result {
            case .success(let verification):
                let transaction = try checkVerified(verification)
                
                // Update subscription status
                await transaction.finish()
                await updateSubscriptionStatus()
                
            case .userCancelled:
                print("User cancelled purchase")
                
            case .pending:
                print("Purchase pending completion")
                
            @unknown default:
                print("Unknown purchase result")
            }
        } catch {
            print("Purchase error: \(error)")
        }
    }
    
    // MARK: - Verify Transaction
    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified:
            throw StoreKitError.failedVerification
        case .verified(let safe):
            return safe
        }
    }
    
    // MARK: - Check Subscription Status
    @MainActor
    func updateSubscriptionStatus() async {
        do {
            for await result in Transaction.currentEntitlements {
                let transaction = try checkVerified(result)
                
                if transaction.productID == "com.myapp.monthly" ||
                   transaction.productID == "com.myapp.yearly" {
                    subscriptionStatus = "Active"
                    purchasedProductIDs.insert(transaction.productID)
                    return
                }
            }
            subscriptionStatus = "Inactive"
        } catch {
            print("Error checking subscription: \(error)")
        }
    }
    
    // MARK: - Restore Purchases
    @MainActor
    func restorePurchases() async {
        do {
            try await AppStore.sync()
            await updateSubscriptionStatus()
            await updatePurchasedProducts()
        } catch {
            print("Restore error: \(error)")
        }
    }
    
    // MARK: - Get Purchased Products
    @MainActor
    private func updatePurchasedProducts() async {
        var purchasedIDs = Set<String>()
        
        for await result in Transaction.currentEntitlements {
            if let transaction = try? checkVerified(result) {
                purchasedIDs.insert(transaction.productID)
            }
        }
        
        self.purchasedProductIDs = purchasedIDs
    }
    
    // MARK: - Request Refund
    @MainActor
    func requestRefund(for transaction: Transaction) async {
        do {
            let status = try await transaction.beginRefundRequest()
            
            switch status {
            case .userCancelled:
                print("Refund cancelled by user")
            case .success:
                print("Refund requested successfully")
            case .failure:
                print("Refund request failed")
            @unknown default:
                print("Unknown refund status")
            }
        } catch {
            print("Refund error: \(error)")
        }
    }
    
    // MARK: - Handle Family Sharing
    @MainActor
    func checkFamilySharingEligibility(product: Product) -> Bool {
        // Family sharing is automatically handled by App Store
        // Just need to offer family plan option
        return product.isFamilyShareable
    }
    
    // MARK: - Transaction Listener
    func listenForTransactions() {
        Task {
            for await result in Transaction.updates {
                do {
                    let transaction = try checkVerified(result)
                    
                    switch transaction.productType {
                    case .autoRenewableSubscription:
                        print("Subscription: \(transaction.productID)")
                    case .nonConsumable:
                        print("Non-consumable: \(transaction.productID)")
                    case .nonRenewableSubscription:
                        print("Non-renewable: \(transaction.productID)")
                    case .consumable:
                        print("Consumable: \(transaction.productID)")
                    @unknown default:
                        break
                    }
                    
                    await transaction.finish()
                } catch {
                    print("Transaction verification failed: \(error)")
                }
            }
        }
    }
}

// MARK: - StoreKit Error Handling
enum StoreKitError: Error {
    case failedVerification
    case invalidProduct
    case networkError
}

// MARK: - SwiftUI Integration
struct StoreView: View {
    @State private var storeManager = AdvancedStoreKitManager()
    
    var body: some View {
        NavigationStack {
            List {
                Section("Subscriptions") {
                    ForEach(storeManager.products.filter { $0.type == .autoRenewableSubscription }, id: \.id) { product in
                        ProductRow(
                            product: product,
                            isPurchased: storeManager.purchasedProductIDs.contains(product.id),
                            action: {
                                Task {
                                    await storeManager.purchase(product)
                                }
                            }
                        )
                    }
                }
                
                Section("Status") {
                    Text("Subscription: \(storeManager.subscriptionStatus)")
                }
                
                Section("Actions") {
                    Button("Restore Purchases") {
                        Task {
                            await storeManager.restorePurchases()
                        }
                    }
                }
            }
            .navigationTitle("Store")
        }
        .task {
            await storeManager.fetchProducts()
            await storeManager.updateSubscriptionStatus()
            storeManager.listenForTransactions()
        }
    }
}

struct ProductRow: View {
    let product: Product
    let isPurchased: Bool
    let action: () -> Void
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(product.displayName)
                    .font(.headline)
                
                if let description = product.description {
                    Text(description)
                        .font(.caption)
                        .foregroundStyle(.gray)
                }
                
                if product.isFamilyShareable {
                    Label("Family Sharing", systemImage: "person.2.fill")
                        .font(.caption)
                        .foregroundStyle(.green)
                }
            }
            
            Spacer()
            
            if isPurchased {
                Image(systemName: "checkmark.circle.fill")
                    .foregroundStyle(.green)
            } else {
                Button(product.displayPrice) {
                    action()
                }
                .buttonStyle(.bordered)
            }
        }
        .padding(.vertical, 8)
    }
}
```

## Subscription Strategy

```
Auto-Renewable Subscriptions:
- Monthly: €9.99 (or local equivalent)
- Yearly: €79.99 (save 33%)
- Family: €14.99/month for up to 6 people

Pricing Strategy:
1. Monthly option (standard baseline)
2. Yearly option (20-30% discount)
3. Family option (premium, multi-user)
4. Trial period (7-14 days free)

Retention:
- Free trial for new users
- Win-back offers for lapsed users
- Price changes grandfathered for existing
- Clear cancellation path
```

## App Store Configuration

### Server-to-Server Notifications

```swift
// Handle App Store notifications about subscription changes
struct NotificationData: Codable {
    let notificationType: String
    let data: NotificationPayload
}

struct NotificationPayload: Codable {
    let signedTransactionInfo: String
    let signedRenewalInfo: String
}

// Decode and verify:
// 1. Decode JWT tokens
// 2. Verify Apple's signature
// 3. Check expiration
// 4. Update subscription status
```

## Implementation Checklist

- [ ] Configure products in App Store Connect
- [ ] Set up subscription groups
- [ ] Implement family sharing eligible products
- [ ] Create AdvancedStoreKitManager
- [ ] Add transaction verification
- [ ] Implement subscription status checking
- [ ] Add restore purchases function
- [ ] Set up transaction listener
- [ ] Handle purchase success/failure
- [ ] Implement refund request flow
- [ ] Test with sandbox credentials
- [ ] Monitor receipt validation
- [ ] Set up server-to-server notifications
- [ ] Plan pricing and trial strategy
