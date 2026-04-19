---
name: swiftdata-persistence
description: |
  SwiftData nativo per persistence. @Model, @Query, @Predicate, CloudKit sync.
  No Core Data boilerplate, no third-party ORMs. Pure Swift models.
trigger: "Add local data persistence with SwiftData"
---

# SwiftData Persistence — @Model, @Query, CloudKit

**Purpose:** Salvare dati localmente su device con SwiftData. Automatic CloudKit sync, migrations, indexes. No Core Data XML hell.

**When to use:**
- Salvare user data (preferences, offline cache)
- Salvare downloaded content (feed items, messages)
- Quando serve CloudKit sync (iCloud)
- Quando serve relazioni tra entità

**Why this matters:** SwiftData è il futuro di iOS persistence (iOS 17+). Core Data è legacy. SwiftData è puro Swift, type-safe, zero XML.

---

## Models: @Model

```swift
// Features/Models/CacheItem.swift
import SwiftData
import Foundation

@Model
final class CacheItem {
    @Attribute(.unique) var key: String
    var data: Data
    var createdAt: Date = Date()
    var expiresAt: Date?
    
    init(key: String, data: Data, expiresAt: Date? = nil) {
        self.key = key
        self.data = data
        self.expiresAt = expiresAt
    }
    
    var isExpired: Bool {
        guard let expiresAt else { return false }
        return Date() > expiresAt
    }
}

// Relazione one-to-many
@Model
final class User {
    @Attribute(.unique) var id: UUID
    var email: String
    var name: String
    var createdAt: Date = Date()
    
    @Relationship(deleteRule: .cascade) var posts: [Post] = []
    @Relationship(deleteRule: .cascade) var preferences: UserPreferences?
    
    init(id: UUID, email: String, name: String) {
        self.id = id
        self.email = email
        self.name = name
    }
}

@Model
final class Post {
    @Attribute(.unique) var id: UUID
    var title: String
    var content: String
    var createdAt: Date = Date()
    var updatedAt: Date?
    
    var author: User?  // Backref, not stored
    
    init(id: UUID, title: String, content: String) {
        self.id = id
        self.title = title
        self.content = content
    }
}

@Model
final class UserPreferences {
    var theme: String = "light"
    var language: String = "en"
    var notificationsEnabled: Bool = true
}
```

**Rules:**
- `@Model` = persistente
- `@Attribute(.unique)` = unique constraint
- `@Relationship(deleteRule: .cascade)` = delete relativo user
- Primitive types (String, Int, Date, UUID, Data) sono automatici
- Custom types devono essere @Codable se non sono @Model

---

## Setup: ModelContainer

```swift
// App.swift
import SwiftUI
import SwiftData

@main
struct MyApp: App {
    let container: ModelContainer
    
    init() {
        do {
            // Setup container con schema
            let schema = Schema([
                User.self,
                Post.self,
                CacheItem.self
            ])
            
            let config = ModelConfiguration(
                schema: schema,
                isStoredInMemoryOnly: false,  // true solo per test
                cloudKitDatabase: .private    // Per CloudKit sync
            )
            
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
            ContentView()
        }
        .modelContainer(container)
    }
}
```

---

## Queries: @Query in View

```swift
// Features/Views/UserListView.swift
import SwiftUI
import SwiftData

struct UserListView: View {
    // Automaticamente fetcha users da database
    @Query(sort: \.createdAt, order: .reverse) var users: [User]
    
    @State private var selectedUser: User?
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        NavigationStack {
            List {
                ForEach(users) { user in
                    NavigationLink(value: user) {
                        VStack(alignment: .leading) {
                            Text(user.name)
                                .font(.headline)
                            Text(user.email)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete(perform: deleteUsers)
            }
            .navigationTitle("Users")
            .navigationDestination(for: User.self) { user in
                UserDetailView(user: user)
            }
        }
    }
    
    private func deleteUsers(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(users[index])
        }
        try? modelContext.save()
    }
}
```

---

## Predicates: Type-Safe Queries

```swift
// Filtrare con @Predicate
@Query(
    filter: #Predicate<User> { $0.email.contains("@example.com") },
    sort: \.createdAt
) var exampleUsers: [User]

// Multiple predicates
@Query(
    filter: #Predicate<Post> {
        $0.author?.email == "user@example.com" &&
        $0.createdAt > Date.now.addingTimeInterval(-86400)  // Last 24h
    }
) var recentUserPosts: [Post]

// Relazioni
@Query(
    filter: #Predicate<User> { $0.posts.count > 0 },
    sort: \.createdAt
) var activeUsers: [User]
```

---

## ViewModel: Fetch + Persist

```swift
// Features/ViewModels/UserViewModel.swift
import Foundation
import SwiftData

@Observable
final class UserViewModel {
    @MainActor
    var currentUser: User? = nil
    @MainActor
    var isLoading: Bool = false
    @MainActor
    var error: String? = nil
    
    private let modelContext: ModelContext
    private let apiService: APIService
    
    init(modelContext: ModelContext, apiService: APIService) {
        self.modelContext = modelContext
        self.apiService = apiService
    }
    
    // MARK: - Fetch from API, save locally
    
    @MainActor
    func syncUser(id: UUID) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            // Fetch from API
            let request = GetUserRequest(id: id)
            let userDTO = try await apiService.execute(request)
            
            // Check if exists locally
            let descriptor = FetchDescriptor<User>(
                predicate: #Predicate { $0.id == id }
            )
            
            let existing = try modelContext.fetch(descriptor)
            
            if let existingUser = existing.first {
                // Update existing
                existingUser.email = userDTO.email
                existingUser.name = userDTO.name
            } else {
                // Create new
                let user = User(
                    id: userDTO.id,
                    email: userDTO.email,
                    name: userDTO.name
                )
                modelContext.insert(user)
            }
            
            try modelContext.save()
            
            // Re-fetch to get updated reference
            let updated = try modelContext.fetch(descriptor)
            self.currentUser = updated.first
            self.error = nil
        } catch {
            self.error = "Failed to sync user: \(error.localizedDescription)"
        }
    }
    
    // MARK: - CRUD Operations
    
    @MainActor
    func createUser(email: String, name: String) throws {
        let user = User(id: UUID(), email: email, name: name)
        modelContext.insert(user)
        try modelContext.save()
        self.currentUser = user
    }
    
    @MainActor
    func deleteUser(_ user: User) throws {
        modelContext.delete(user)
        try modelContext.save()
        if self.currentUser?.id == user.id {
            self.currentUser = nil
        }
    }
    
    @MainActor
    func updateUser(_ user: User, name: String) throws {
        user.name = name
        user.updatedAt = Date()
        try modelContext.save()
    }
}
```

---

## Migrations: Schema Changes

```swift
// Setup con migration strategy
do {
    let schema = Schema([User.self, Post.self])
    
    let config = ModelConfiguration(
        schema: schema,
        isStoredInMemoryOnly: false,
        cloudKitDatabase: .private
    )
    
    // Se schema cambia, SwiftData auto-migra
    // Per custom logic:
    let migrations: [Migration] = [
        Migration(from: oldSchema, to: newSchema) { context in
            // Custom migration logic
            let oldUsers = try context.fetch(FetchDescriptor<OldUser>())
            for oldUser in oldUsers {
                let newUser = User(
                    id: oldUser.id,
                    email: oldUser.email,
                    name: oldUser.fullName
                )
                context.insert(newUser)
            }
        }
    ]
    
    container = try ModelContainer(
        for: schema,
        configurations: [config],
        migrations: migrations
    )
} catch {
    fatalError("Could not initialize: \(error)")
}
```

---

## Testing: In-Memory Database

```swift
// Tests/Models/UserTests.swift
import Testing
import SwiftData

@Suite
final class UserModelTests {
    private var modelContainer: ModelContainer!
    private var modelContext: ModelContext!
    
    init() throws {
        // In-memory database per test
        let schema = Schema([User.self, Post.self])
        let config = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: true
        )
        
        modelContainer = try ModelContainer(
            for: schema,
            configurations: [config]
        )
        modelContext = ModelContext(modelContainer)
    }
    
    @Test
    func createAndFetchUser() throws {
        // Arrange
        let user = User(id: UUID(), email: "test@example.com", name: "Test User")
        
        // Act
        modelContext.insert(user)
        try modelContext.save()
        
        // Assert: Fetch
        let descriptor = FetchDescriptor<User>()
        let fetched = try modelContext.fetch(descriptor)
        
        #expect(fetched.count == 1)
        #expect(fetched.first?.email == "test@example.com")
    }
    
    @Test
    func deleteUser() throws {
        // Setup
        let user = User(id: UUID(), email: "test@example.com", name: "Test")
        modelContext.insert(user)
        try modelContext.save()
        
        // Act
        modelContext.delete(user)
        try modelContext.save()
        
        // Assert
        let descriptor = FetchDescriptor<User>()
        let fetched = try modelContext.fetch(descriptor)
        #expect(fetched.isEmpty)
    }
}
```

---

## CloudKit Sync

```swift
// Setup with CloudKit
let schema = Schema([User.self, Post.self])

let config = ModelConfiguration(
    schema: schema,
    cloudKitDatabase: .private  // iCloud private database
)

container = try ModelContainer(for: schema, configurations: [config])

// Automatic sync! No additional code needed.
// Quando user fa login con iCloud → dati syncano automati.
```

**Privacy:** CloudKit sync requires privacy policy + GDPR compliance if EU users.

---

## Cache Strategy

```swift
// Salvare API responses per offline access
@Observable
final class CacheManager {
    private let modelContext: ModelContext
    private let ttl: TimeInterval = 3600  // 1 hour
    
    func cachedResult<T: Codable>(
        for key: String,
        fetch: () async throws -> T
    ) async throws -> T {
        // Check cache
        let descriptor = FetchDescriptor<CacheItem>(
            predicate: #Predicate { $0.key == key }
        )
        
        if let cached = try modelContext.fetch(descriptor).first,
           !cached.isExpired {
            let decoder = JSONDecoder()
            return try decoder.decode(T.self, from: cached.data)
        }
        
        // Fetch fresh
        let fresh = try await fetch()
        
        // Save to cache
        let encoder = JSONEncoder()
        let data = try encoder.encode(fresh)
        let cacheItem = CacheItem(
            key: key,
            data: data,
            expiresAt: Date().addingTimeInterval(ttl)
        )
        
        modelContext.insert(cacheItem)
        try modelContext.save()
        
        return fresh
    }
}
```

---

## Checklist: SwiftData Implementation

- [ ] Models sono @Model con @Attribute(.unique)
- [ ] Relazioni definite con @Relationship
- [ ] ModelContainer setup in App.swift
- [ ] @Query usate nel View (auto-refresh)
- [ ] @Predicate per filtri type-safe
- [ ] ModelContext injected via environment
- [ ] save() dopo insert/delete/update
- [ ] CloudKit sync configured (se needed)
- [ ] In-memory database per test
- [ ] Migrations handled per schema changes

---

## Related Skills

- **swiftui-architecture** — Usare ModelContext nel ViewModel
- **ios-networking** — Syncronizzare dati API con database
- **swift-testing** — Test models con in-memory database

---

**Status:** ✅ SwiftData (iOS 17+), no Core Data
