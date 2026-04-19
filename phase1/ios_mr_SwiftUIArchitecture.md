---
name: swiftui-architecture
description: |
  Moderne pattern MVVM con @Observable di Swift 6+. Per View, ViewModel, Models.
  Niente binding manuali, niente @StateObject—solo pure Swift con dependency injection.
  Scalabile per app complesse con 100+ views.
trigger: "Add SwiftUI view with MVVM architecture"
---

# SwiftUI Architecture — MVVM with @Observable

**Purpose:** Struttura scalabile per le View, senza binding manuali o StateObject. Ogni view ha un ViewModel con @Observable, Models sono Sendable, tutto tipicamente safe.

**When to use:**
- Ogni volta che aggiungi una view che ha logica (network, database, user interactions)
- Per feature che hanno stati complessi (loading, error, success)
- Quando devi testare la logica separata dalla UI

**Why this matters:** Il vecchio pattern @StateObject + @ObservedObject crea coupling tra View e ViewModel. @Observable è più puro: il ViewModel è solo una classe, la View lo usa come dipendenza. Testabile, facile da mockare, semplice da ragionare.

---

## Architecture Structure

Ogni feature ha questa struttura:

```
Feature/
├─ Models/
│  ├─ [Entity].swift          # @Sendable struct per dati
│  └─ [RequestBody].swift     # Input per API
├─ ViewModels/
│  └─ [Feature]ViewModel.swift # @Observable class
├─ Views/
│  ├─ [Feature]View.swift      # SwiftUI view principale
│  └─ [Subfeature]View.swift   # Sotto-view (se complessa)
└─ Services/ (shared)
   ├─ APIService.swift
   ├─ DatabaseService.swift
   └─ AuthService.swift
```

---

## Models: Sendable + Codable

Dati immutabili, Codable per persistence, Sendable per concorrenza.

```swift
// Feature/Models/User.swift
import Foundation

@Sendable
struct User: Identifiable, Codable {
    let id: UUID
    let email: String
    let name: String
    let createdAt: Date
    
    enum CodingKeys: String, CodingKey {
        case id, email, name
        case createdAt = "created_at"
    }
}

@Sendable
struct LoginRequest: Codable {
    let email: String
    let password: String
}

// Errori tipizzati per ViewModel
@Sendable
enum AuthError: LocalizedError, Sendable {
    case invalidEmail
    case networkFailure(Error)
    case unauthorized
    
    var errorDescription: String? {
        switch self {
        case .invalidEmail:
            return "Email non valida"
        case .networkFailure:
            return "Errore di rete. Riprova."
        case .unauthorized:
            return "Email o password non corretti"
        }
    }
}
```

---

## ViewModel: @Observable Class

Niente @Published, niente Combine. Solo @Observable e proprietà normali.

```swift
// Feature/ViewModels/LoginViewModel.swift
import Foundation

@Observable
final class LoginViewModel {
    // MARK: - State
    var email: String = ""
    var password: String = ""
    var isLoading: Bool = false
    var error: AuthError? = nil
    var currentUser: User? = nil
    
    // MARK: - Dependencies (injected)
    private let authService: AuthService
    
    init(authService: AuthService = .shared) {
        self.authService = authService
    }
    
    // MARK: - Actions
    func login() async {
        guard isEmailValid() else {
            error = .invalidEmail
            return
        }
        
        isLoading = true
        defer { isLoading = false }
        
        do {
            let request = LoginRequest(email: email, password: password)
            let user = try await authService.login(request)
            
            // Aggiorna lo stato
            self.currentUser = user
            self.error = nil
            
            // Non controllare self.currentUser qui per sapere se login è riuscito
            // La view sa che non c'è errore → login riuscito
        } catch let error as AuthError {
            self.error = error
        } catch {
            self.error = .networkFailure(error)
        }
    }
    
    func logout() {
        authService.logout()
        currentUser = nil
        email = ""
        password = ""
    }
    
    // MARK: - Helpers
    private func isEmailValid() -> Bool {
        email.contains("@") && email.contains(".")
    }
}
```

**Regole:**
- Niente @Published: @Observable gestisce l'invalidazione
- Niente Combine: usa async/await direttamente
- Properties sono var (mutable), non let
- Errori sono @Sendable enum
- init() prende le dipendenze (DI)
- async funzioni, non closures

---

## View: Usa ViewModel come Dependency

```swift
// Feature/Views/LoginView.swift
import SwiftUI

struct LoginView: View {
    @State private var viewModel: LoginViewModel
    
    init(authService: AuthService = .shared) {
        _viewModel = State(initialValue: LoginViewModel(authService: authService))
    }
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                // Header
                VStack(spacing: 8) {
                    Text("Accedi")
                        .font(.title)
                        .fontWeight(.bold)
                    Text("Al tuo account")
                        .foregroundStyle(.secondary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)
                
                // Form
                VStack(spacing: 12) {
                    TextField("Email", text: $viewModel.email)
                        .textInputAutocapitalization(.never)
                        .keyboardType(.emailAddress)
                        .textFieldStyle(.roundedBorder)
                    
                    SecureField("Password", text: $viewModel.password)
                        .textFieldStyle(.roundedBorder)
                }
                
                // Error message
                if let error = viewModel.error {
                    HStack(spacing: 8) {
                        Image(systemName: "exclamationmark.circle.fill")
                        Text(error.localizedDescription)
                        Spacer()
                    }
                    .foregroundStyle(.red)
                    .padding(.vertical, 8)
                    .padding(.horizontal, 12)
                    .background(.red.opacity(0.1))
                    .cornerRadius(8)
                }
                
                // Login button
                Button(action: {
                    Task {
                        await viewModel.login()
                    }
                }) {
                    if viewModel.isLoading {
                        ProgressView()
                            .progressViewStyle(.circular)
                    } else {
                        Text("Accedi")
                            .fontWeight(.semibold)
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

// MARK: - Helpers
extension LoginViewModel {
    var isFormValid: Bool {
        !email.isEmpty && !password.isEmpty
    }
}
```

**Regole:**
- `@State private var viewModel` (non @StateObject)
- Inizializza nel `init()` con DI
- Usa `$viewModel.property` per binding
- `Task { await viewModel.action() }` per async
- Niente `@EnvironmentObject`, usa init()
- Error UI è dichiarativa

---

## Testing ViewModel

```swift
// Feature/ViewModels/LoginViewModelTests.swift
import Testing

@Suite
final class LoginViewModelTests {
    var mockAuthService: MockAuthService!
    var viewModel: LoginViewModel!
    
    init() {
        self.mockAuthService = MockAuthService()
        self.viewModel = LoginViewModel(authService: mockAuthService)
    }
    
    @Test
    func testLoginSuccess() async {
        // Arrange
        viewModel.email = "user@example.com"
        viewModel.password = "correct"
        mockAuthService.loginResult = .success(User(id: UUID(), email: "user@example.com", name: "User", createdAt: Date()))
        
        // Act
        await viewModel.login()
        
        // Assert
        #expect(viewModel.currentUser != nil)
        #expect(viewModel.error == nil)
        #expect(viewModel.isLoading == false)
    }
    
    @Test
    func testLoginInvalidEmail() async {
        // Arrange
        viewModel.email = "notanemail"
        viewModel.password = "password"
        
        // Act
        await viewModel.login()
        
        // Assert
        #expect(viewModel.error == .invalidEmail)
    }
}

// Mock for testing
final class MockAuthService: AuthService {
    var loginResult: Result<User, AuthError> = .failure(.invalidEmail)
    
    override func login(_ request: LoginRequest) async throws -> User {
        try loginResult.get()
    }
}
```

---

## Dependency Injection Pattern

Per evitare GlobalState e @EnvironmentObject:

```swift
// App.swift
import SwiftUI

@main
struct MyApp: App {
    let authService = AuthService.shared
    
    var body: some Scene {
        WindowGroup {
            RootView(authService: authService)
        }
    }
}

// RootView capisce chi è loggato e mostra la view giusta
struct RootView: View {
    let authService: AuthService
    @State private var isAuthenticated: Bool = false
    
    var body: some View {
        if isAuthenticated {
            HomeView(authService: authService)
        } else {
            LoginView(authService: authService)
        }
    }
}
```

---

## Quando Usare @Observable vs @State

| Situazione | Usa |
|---|---|
| Logica semplice (toggle, contador) | @State nel View |
| Logica complessa (network, database) | @Observable ViewModel |
| Cross-feature state (auth, theme) | AppState @Observable globale |
| Transizione tra views | Navigation + init() DI |

---

## Checklist: Architecture Review

- [ ] Models sono @Sendable + Codable
- [ ] ViewModel è @Observable final class
- [ ] @Published → rimosso (usa @Observable)
- [ ] @StateObject → rimosso (usa @State + init())
- [ ] Errori sono typizzati (enum)
- [ ] async/await, non closures
- [ ] DI nel init(), non @EnvironmentObject
- [ ] ViewModel è testabile senza SwiftUI
- [ ] View è dichiarativa, non imperativa

---

## Common Mistakes to Avoid

❌ **Mettere logica nella View**
```swift
// WRONG
struct MyView: View {
    @State var data: [Item] = []
    var body: some View {
        VStack {
            Button("Fetch") {
                Task {
                    let url = URL(string: "...")!
                    let (data, _) = try await URLSession.shared.data(from: url)
                    // ... decode
                }
            }
        }
    }
}
```

✅ **Mettere logica nel ViewModel**
```swift
// CORRECT
@Observable
final class MyViewModel {
    var data: [Item] = []
    
    func fetchData() async {
        // logica qui
    }
}

struct MyView: View {
    @State private var viewModel: MyViewModel
    var body: some View {
        VStack {
            Button("Fetch") {
                Task { await viewModel.fetchData() }
            }
        }
    }
}
```

---

## Related Skills

- **ios-networking** — Come fare chiamate API dal ViewModel
- **swiftdata** — Persistenza dati nel ViewModel
- **swift-testing** — Unit test del ViewModel

---

**Status:** ✅ Production-ready, Swift 6.2+, no external dependencies
