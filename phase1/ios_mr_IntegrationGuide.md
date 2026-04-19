---
name: integration-guide
description: Come integrare le 5 skill essenziali di Phase 1 in un progetto iOS reale
---

# Phase 1 Integration Guide — 5 Core Skills

**Scenario:** Stai creando una prima app iOS con login, lista di item, offline support. Come usi le 5 skill?

**Skills di questa guida:**
1. **swiftui-architecture** — MVVM + @Observable
2. **ios-networking** — API calls con retry
3. **swift-testing** — Unit tests
4. **swiftdata-persistence** — Local storage
5. **app-store-review** (next file) — Submission checklist

---

## Architecture Overview: Come Funziona Insieme

```
┌─ App.swift ──────────────────────────────┐
│ ModelContainer (SwiftData)                │
│ APIService (Networking)                   │
│ AuthService (Networking + SwiftData)      │
│                                            │
│ LoginView                                  │
│  ↓                                         │
│  @State LoginViewModel (MVVM)             │
│  ↓                                         │
│  authService.login() → APIService         │
│  ↓                                         │
│  Salva user in SwiftData                  │
└────────────────────────────────────────────┘

┌─ HomeView ────────────────────────────────┐
│ @Query items: [Item] (SwiftData)          │
│ @State HomeViewModel (MVVM)               │
│ ↓                                          │
│ viewModel.loadItems()                     │
│ ↓                                          │
│ APIService.execute(GetItemsRequest)       │
│ ↓ (cache + retry)                         │
│ Salva items in SwiftData                  │
└────────────────────────────────────────────┘

┌─ Tests ───────────────────────────────────┐
│ LoginViewModelTests (Swift Testing)       │
│ APIServiceTests (Mock URLProtocol)        │
│ HomeViewModelTests (Mock APIService)      │
│ ItemTests (SwiftData in-memory)           │
└────────────────────────────────────────────┘
```

---

## Step 1: Setup Project (30 min)

### 1.1 Create Xcode Project

```bash
File → New → Project → iOS App
- Product Name: MyApp
- Team: Your Apple Developer team
- Bundle ID: dev.yourname.myapp
- Interface: SwiftUI
- Life Cycle: SwiftUI App
- iOS Minimum: 17.0
```

### 1.2 Setup Folder Structure

```
MyApp/
├─ App/
│  ├─ MyApp.swift            # @main entry point
│  ├─ RootView.swift         # Routing (auth vs home)
│  └─ AppState.swift         # Global @Observable
│
├─ Features/
│  ├─ Auth/
│  │  ├─ Models/             # User, LoginRequest, etc.
│  │  ├─ ViewModels/         # LoginViewModel
│  │  └─ Views/              # LoginView
│  │
│  ├─ Home/
│  │  ├─ Models/             # Item, etc.
│  │  ├─ ViewModels/         # HomeViewModel
│  │  └─ Views/              # HomeView, ItemListView
│  │
│  └─ Shared/
│     └─ Views/              # Reusable components
│
├─ Services/
│  ├─ Networking/
│  │  ├─ APIService.swift
│  │  ├─ APIRequest.swift
│  │  ├─ APIError.swift
│  │  ├─ Interceptors/       # AuthInterceptor
│  │  └─ Cache/              # ResponseCache
│  │
│  └─ Auth/
│     └─ AuthService.swift   # Login, token management
│
└─ Tests/
   ├─ ViewModels/
   │  ├─ LoginViewModelTests.swift
   │  └─ HomeViewModelTests.swift
   │
   ├─ Services/
   │  ├─ APIServiceTests.swift
   │  └─ AuthServiceTests.swift
   │
   └─ Mocks/
      ├─ MockAuthService.swift
      └─ MockAPIService.swift
```

### 1.3 Add SPM Dependencies

In Xcode: File → Add Packages

```
// None needed! Usiamo solo native frameworks:
// - SwiftUI (native)
// - SwiftData (native)
// - URLSession (native)
// - Swift Testing (native)
```

---

## Step 2: Models (30 min)

### 2.1 Auth Models

```swift
// Features/Auth/Models/User.swift
import Foundation
import SwiftData

@Model
final class User {
    @Attribute(.unique) var id: UUID
    var email: String
    var name: String
    var accessToken: String
    var refreshToken: String
    var createdAt: Date = Date()
    
    init(id: UUID, email: String, name: String, 
         accessToken: String, refreshToken: String) {
        self.id = id
        self.email = email
        self.name = name
        self.accessToken = accessToken
        self.refreshToken = refreshToken
    }
}

// Features/Auth/Models/LoginRequest.swift
struct LoginRequest: APIRequest {
    typealias Response = LoginResponse
    
    let email: String
    let password: String
    
    var method: HTTPMethod { .post }
    var path: String { "/api/v1/auth/login" }
    var headers: [String: String]? {
        ["Content-Type": "application/json"]
    }
    var body: Encodable? {
        LoginBody(email: email, password: password)
    }
}

struct LoginBody: Codable {
    let email: String
    let password: String
}

struct LoginResponse: Codable {
    let accessToken: String
    let refreshToken: String
    let user: UserDTO
    
    enum CodingKeys: String, CodingKey {
        case accessToken = "access_token"
        case refreshToken = "refresh_token"
        case user
    }
}

struct UserDTO: Codable {
    let id: UUID
    let email: String
    let name: String
}
```

### 2.2 Item Models

```swift
// Features/Home/Models/Item.swift
import Foundation
import SwiftData

@Model
final class Item {
    @Attribute(.unique) var id: UUID
    var title: String
    var description: String?
    var createdAt: Date = Date()
    var updatedAt: Date?
    
    init(id: UUID, title: String, description: String? = nil) {
        self.id = id
        self.title = title
        self.description = description
    }
}

// Features/Home/Models/GetItemsRequest.swift
struct GetItemsRequest: APIRequest {
    typealias Response = GetItemsResponse
    
    let limit: Int
    let offset: Int
    
    var method: HTTPMethod { .get }
    var path: String { "/api/v1/items" }
    
    var queryItems: [URLQueryItem]? {
        [
            URLQueryItem(name: "limit", value: String(limit)),
            URLQueryItem(name: "offset", value: String(offset))
        ]
    }
}

struct GetItemsResponse: Codable {
    let items: [ItemDTO]
    let total: Int
}

struct ItemDTO: Codable {
    let id: UUID
    let title: String
    let description: String?
    let createdAt: String
}
```

---

## Step 3: Services (30 min)

```swift
// Services/Auth/AuthService.swift
import Foundation
import SwiftData

@Observable
final class AuthService {
    static let shared = AuthService()
    
    @MainActor
    var currentUser: User? = nil
    
    private let apiService: APIService
    private let modelContext: ModelContext?  // Injected
    
    init(apiService: APIService = .shared, 
         modelContext: ModelContext? = nil) {
        self.apiService = apiService
        self.modelContext = modelContext
    }
    
    @MainActor
    func login(email: String, password: String) async throws -> User {
        let request = LoginRequest(email: email, password: password)
        let response = try await apiService.execute(request)
        
        // Create user model
        let user = User(
            id: response.user.id,
            email: response.user.email,
            name: response.user.name,
            accessToken: response.accessToken,
            refreshToken: response.refreshToken
        )
        
        // Save to database
        if let context = modelContext {
            context.insert(user)
            try context.save()
        }
        
        self.currentUser = user
        return user
    }
    
    @MainActor
    func logout() {
        currentUser = nil
        // Also clear tokens from keychain (if using)
    }
}
```

---

## Step 4: ViewModels (30 min)

```swift
// Features/Auth/ViewModels/LoginViewModel.swift
import Foundation
import SwiftData

@Observable
final class LoginViewModel {
    var email: String = ""
    var password: String = ""
    var isLoading: Bool = false
    var error: String? = nil
    
    private let authService: AuthService
    private let modelContext: ModelContext
    
    init(authService: AuthService = .shared, 
         modelContext: ModelContext) {
        self.authService = authService
        self.modelContext = modelContext
    }
    
    @MainActor
    func login() async {
        guard !email.isEmpty, !password.isEmpty else {
            error = "Email e password obbligatori"
            return
        }
        
        isLoading = true
        defer { isLoading = false }
        
        do {
            let user = try await authService.login(
                email: email,
                password: password
            )
            error = nil
        } catch let error as APIError {
            self.error = error.localizedDescription
        } catch {
            self.error = "Errore sconosciuto"
        }
    }
    
    var isFormValid: Bool {
        !email.isEmpty && !password.isEmpty
    }
}

// Features/Home/ViewModels/HomeViewModel.swift
import Foundation
import SwiftData

@Observable
final class HomeViewModel {
    var items: [Item] = []
    var isLoading: Bool = false
    var error: String? = nil
    
    private let apiService: APIService
    private let modelContext: ModelContext
    
    init(apiService: APIService = .shared,
         modelContext: ModelContext) {
        self.apiService = apiService
        self.modelContext = modelContext
    }
    
    @MainActor
    func loadItems(limit: Int = 20) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            let request = GetItemsRequest(limit: limit, offset: 0)
            let response = try await apiService.execute(request)
            
            // Delete old items (sync)
            var descriptor = FetchDescriptor<Item>()
            let oldItems = try modelContext.fetch(descriptor)
            for old in oldItems {
                modelContext.delete(old)
            }
            
            // Insert new items
            for itemDTO in response.items {
                let item = Item(
                    id: itemDTO.id,
                    title: itemDTO.title,
                    description: itemDTO.description
                )
                modelContext.insert(item)
            }
            
            try modelContext.save()
            
            // Reload from database
            descriptor = FetchDescriptor<Item>(
                sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
            )
            self.items = try modelContext.fetch(descriptor)
            self.error = nil
        } catch let error as APIError {
            self.error = error.localizedDescription
        } catch {
            self.error = "Errore sconosciuto"
        }
    }
}
```

---

## Step 5: Views (30 min)

```swift
// Features/Auth/Views/LoginView.swift
import SwiftUI
import SwiftData

struct LoginView: View {
    @State private var viewModel: LoginViewModel
    @Environment(\.modelContext) var modelContext
    
    init(authService: AuthService = .shared,
         modelContext: ModelContext) {
        _viewModel = State(
            initialValue: LoginViewModel(
                authService: authService,
                modelContext: modelContext
            )
        )
    }
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                Text("Accedi")
                    .font(.title)
                    .fontWeight(.bold)
                
                TextField("Email", text: $viewModel.email)
                    .textInputAutocapitalization(.never)
                    .keyboardType(.emailAddress)
                    .textFieldStyle(.roundedBorder)
                
                SecureField("Password", text: $viewModel.password)
                    .textFieldStyle(.roundedBorder)
                
                if let error = viewModel.error {
                    Text(error)
                        .foregroundStyle(.red)
                        .font(.caption)
                }
                
                Button(action: {
                    Task {
                        await viewModel.login()
                    }
                }) {
                    if viewModel.isLoading {
                        ProgressView()
                    } else {
                        Text("Accedi")
                    }
                }
                .frame(maxWidth: .infinity)
                .frame(height: 48)
                .background(viewModel.isFormValid ? .blue : .gray)
                .foregroundStyle(.white)
                .cornerRadius(8)
                .disabled(!viewModel.isFormValid || viewModel.isLoading)
                
                Spacer()
            }
            .padding()
        }
    }
}

// Features/Home/Views/HomeView.swift
import SwiftUI
import SwiftData

struct HomeView: View {
    @Query(sort: \.createdAt, order: .reverse) var items: [Item]
    @State private var viewModel: HomeViewModel
    @Environment(\.modelContext) var modelContext
    
    init(apiService: APIService = .shared,
         modelContext: ModelContext) {
        _viewModel = State(
            initialValue: HomeViewModel(
                apiService: apiService,
                modelContext: modelContext
            )
        )
    }
    
    var body: some View {
        NavigationStack {
            List(items) { item in
                VStack(alignment: .leading) {
                    Text(item.title)
                        .font(.headline)
                    if let desc = item.description {
                        Text(desc)
                            .font(.caption)
                            .foregroundStyle(.secphasery)
                    }
                }
            }
            .navigationTitle("Items")
            .onAppear {
                Task {
                    await viewModel.loadItems()
                }
            }
            .refreshable {
                await viewModel.loadItems()
            }
        }
    }
}
```

---

## Step 6: App Entry Point (15 min)

```swift
// App/MyApp.swift
import SwiftUI
import SwiftData

@main
struct MyApp: App {
    let container: ModelContainer
    @State private var authService = AuthService.shared
    
    init() {
        do {
            let schema = Schema([User.self, Item.self])
            let config = ModelConfiguration(schema: schema)
            container = try ModelContainer(
                for: schema,
                configurations: [config]
            )
        } catch {
            fatalError("Could not initialize ModelContainer: \(error)")
        }
    }
    
    var body: some Scene {
        WindowGroup {
            if authService.currentUser != nil {
                HomeView(modelContext: container.mainContext)
            } else {
                LoginView(modelContext: container.mainContext)
            }
        }
        .modelContainer(container)
    }
}
```

---

## Step 7: Tests (1 hour)

```swift
// Tests/ViewModels/LoginViewModelTests.swift
import Testing
import SwiftData

@Suite
final class LoginViewModelTests {
    private var modelContainer: ModelContainer!
    private var mockAuthService: MockAuthService!
    private var viewModel: LoginViewModel!
    
    init() throws {
        let schema = Schema([User.self])
        let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: true)
        modelContainer = try ModelContainer(for: schema, configurations: [config])
        
        mockAuthService = MockAuthService()
        viewModel = LoginViewModel(
            authService: mockAuthService,
            modelContext: modelContainer.mainContext
        )
    }
    
    @Test
    async func loginSuccess() async {
        // Arrange
        let user = User(
            id: UUID(),
            email: "test@example.com",
            name: "Test",
            accessToken: "token",
            refreshToken: "refresh"
        )
        mockAuthService.loginResult = .success(user)
        
        viewModel.email = "test@example.com"
        viewModel.password = "password"
        
        // Act
        await viewModel.login()
        
        // Assert
        #expect(viewModel.error == nil)
    }
}

// Tests/Mocks/MockAuthService.swift
final class MockAuthService: AuthService {
    var loginResult: Result<User, Error> = .failure(NSError(domain: "Mock", code: -1))
    
    override func login(email: String, password: String) async throws -> User {
        try loginResult.get()
    }
}
```

---

## Step 8: Run & Test

```bash
# Build
Cmd + B

# Run on simulator
Cmd + R

# Run tests
Cmd + U

# Check for warnings
Cmd + Shift + K (clean)
```

---

## Integration Checklist

- [ ] Xcode project creato con iOS 17+ minimum
- [ ] Folder structure organisata per feature
- [ ] Models: User, Item, Requests
- [ ] Services: APIService, AuthService
- [ ] ViewModels: LoginViewModel, HomeViewModel
- [ ] Views: LoginView, HomeView
- [ ] App.swift con ModelContainer
- [ ] Tests creati (ViewModels, Services)
- [ ] Runs on simulator senza errori
- [ ] Tests all pass (Cmd + U)

---

## Related Documentation

- **PROGRESS.md** — Aggiorna quando completi i step
- **CLAUDE.md** — Documenta decisions architetturali
- **session-start.md** — Leggi prima di ogni sessione
- **session-end.md** — Aggiorna dopo ogni sessione

---

**Status:** ✅ Full Phase 1 integration ready for first submission
