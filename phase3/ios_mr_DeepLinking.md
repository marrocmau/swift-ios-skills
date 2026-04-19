---
name: deep-linking
description: Deep linking and universal links for app navigation. Use this skill when implementing universal links, handling deep links, routing users from URLs to app screens, or integrating with social media sharing. Essential for user acquisition and seamless cross-platform navigation.
compatibility: iOS 15+, URLScheme and Universal Links
---

# Deep Linking — Universal Links & URL Routing

**When to use:** Before sharing features or integrating with web/social platforms.

## Pattern: Deep Link Handler

```swift
import SwiftUI

@Observable
final class DeepLinkHandler {
    var deepLink: URL?
    
    enum DeepLinkTarget {
        case home
        case item(id: String)
        case user(username: String)
        case settings
        case purchase(productID: String)
        case invalid
    }
    
    // MARK: - Parse URL to Target
    func parseDeepLink(_ url: URL) -> DeepLinkTarget {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true) else {
            return .invalid
        }
        
        // Handle different URL schemes
        if url.scheme == "myapp" || url.scheme == "myapp.io" {
            return parseAppScheme(components)
        } else if components.host == "myapp.io" {
            return parseUniversalLink(components)
        }
        
        return .invalid
    }
    
    // MARK: - App Scheme (myapp://item?id=123)
    private func parseAppScheme(_ components: URLComponents) -> DeepLinkTarget {
        switch components.host {
        case "item":
            if let id = components.queryItems?.first(where: { $0.name == "id" })?.value {
                return .item(id: id)
            }
            
        case "user":
            if let username = components.queryItems?.first(where: { $0.name == "username" })?.value {
                return .user(username: username)
            }
            
        case "settings":
            return .settings
            
        case "purchase":
            if let productID = components.queryItems?.first(where: { $0.name == "product" })?.value {
                return .purchase(productID: productID)
            }
            
        default:
            return .home
        }
        
        return .invalid
    }
    
    // MARK: - Universal Links (https://myapp.io/item/123)
    private func parseUniversalLink(_ components: URLComponents) -> DeepLinkTarget {
        let pathComponents = components.path.split(separator: "/").map(String.init)
        
        guard let first = pathComponents.first else {
            return .home
        }
        
        switch first {
        case "item":
            if pathComponents.count > 1 {
                return .item(id: pathComponents[1])
            }
            
        case "user":
            if pathComponents.count > 1 {
                return .user(username: pathComponents[1])
            }
            
        case "settings":
            return .settings
            
        case "purchase":
            if let productID = components.queryItems?.first(where: { $0.name == "product" })?.value {
                return .purchase(productID: productID)
            }
            
        default:
            return .home
        }
        
        return .invalid
    }
    
    // MARK: - Handle Deep Link in App
    func handleDeepLink(_ url: URL) {
        deepLink = url
    }
}

// MARK: - Scene Delegate Handler
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    func scene(
        _ scene: UIScene,
        openURLContexts URLContexts: Set<UIOpenURLContext>
    ) {
        for context in URLContexts {
            let handler = DeepLinkHandler()
            let target = handler.parseDeepLink(context.url)
            handleDeepLinkTarget(target)
        }
    }
    
    private func handleDeepLinkTarget(_ target: DeepLinkHandler.DeepLinkTarget) {
        switch target {
        case .home:
            print("Navigate to home")
        case .item(let id):
            print("Navigate to item: \(id)")
        case .user(let username):
            print("Navigate to user: \(username)")
        case .settings:
            print("Navigate to settings")
        case .purchase(let productID):
            print("Navigate to purchase: \(productID)")
        case .invalid:
            print("Invalid deep link")
        }
    }
}

// MARK: - SwiftUI Integration
@main
struct MyApp: App {
    @State private var deepLinkHandler = DeepLinkHandler()
    @State private var navPath = NavigationPath()
    
    var body: some Scene {
        WindowGroup {
            NavigationStack(path: $navPath) {
                ContentView()
                    .navigationDestination(for: DeepLinkHandler.DeepLinkTarget.self) { target in
                        handleNavigation(target)
                    }
            }
            .onOpenURL { url in
                deepLinkHandler.handleDeepLink(url)
                let target = deepLinkHandler.parseDeepLink(url)
                navPath.append(target)
            }
        }
    }
    
    @ViewBuilder
    func handleNavigation(_ target: DeepLinkHandler.DeepLinkTarget) -> some View {
        switch target {
        case .item(let id):
            ItemDetailView(id: id)
        case .user(let username):
            UserProfileView(username: username)
        case .settings:
            SettingsView()
        case .purchase(let productID):
            PurchaseView(productID: productID)
        default:
            ContentView()
        }
    }
}
```

## Configuration Files

### Universal Links (apple-app-site-association)

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.mycompany.myapp",
        "paths": [
          "/item/*",
          "/user/*",
          "/settings",
          "/purchase*"
        ]
      }
    ]
  }
}
```

Place this at: `https://myapp.io/.well-known/apple-app-site-association`

### Info.plist URL Schemes

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key>
    <string>com.mycompany.myapp</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>
```

## URL Scheme Examples

```
Deep link to item:
myapp://item?id=12345
https://myapp.io/item/12345

Deep link to user:
myapp://user?username=john_doe
https://myapp.io/user/john_doe

Deep link to purchase:
myapp://purchase?product=premium_monthly
https://myapp.io/purchase?product=premium_monthly

Deep link to settings:
myapp://settings
https://myapp.io/settings
```

## Implementation Checklist

- [ ] Define app URL scheme in Info.plist
- [ ] Create DeepLinkHandler class
- [ ] Implement URL parsing logic
- [ ] Create universal link mapping
- [ ] Set up apple-app-site-association file
- [ ] Configure domain verification
- [ ] Add SceneDelegate URL handling
- [ ] Integrate with SwiftUI navigation
- [ ] Test app scheme URLs
- [ ] Test universal links on device
- [ ] Verify analytics tracking for deep links
- [ ] Share links in social media
