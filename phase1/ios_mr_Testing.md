---
name: swift-testing
description: |
  Swift Testing framework (@Test, @Suite). No XCTest, no assertions.
  Pure Swift 6.2 testing con macros, async support, mocking patterns.
trigger: "Add unit tests for ViewModel or Service"
---

# Swift Testing — @Test, @Suite, Mocking

**Purpose:** Unit tests moderni con Swift Testing. Niente XCTest legacy, niente `XCTAssertEqual`, solo puro Swift con #expect e async support nativo.

**When to use:**
- Testare ViewModels (logica senza UI)
- Testare Services (API, database, auth)
- Testare Models (validazione, transformazione)
- Evitare test di UI (non scalano)

**Why this matters:** Swift Testing è il futuro di iOS testing. Syntax più pulita, async support nativo, parametrized tests facili, mocking integrato. XCTest è legacy.

---

## Setup: Aggiungi il Target Test

```bash
# Nel Xcode project, aggiungi target:
# File → New → Target → Unit Testing Bundle
# Product Name: [App]Tests
# Language: Swift

# Nel Package.swift (se SPM):
.testTarget(
    name: "Tests",
    dependencies: ["YourApp"],
    resources: [.copy("Fixtures")]
)
```

---

## Test Structure

```
[App]Tests/
├─ ViewModels/
│  └─ LoginViewModelTests.swift
├─ Services/
│  ├─ APIServiceTests.swift
│  └─ AuthServiceTests.swift
├─ Models/
│  └─ UserTests.swift
├─ Mocks/
│  ├─ MockAuthService.swift
│  ├─ MockAPIService.swift
│  └─ MockURLSession.swift
└─ Fixtures/
   ├─ login-response.json
   └─ user-list.json
```

---

## Basic Test: @Test e #expect

```swift
// Tests/ViewModels/LoginViewModelTests.swift
import Testing

@Suite
final class LoginViewModelTests {
    // MARK: - Setup
    var mockAuthService: MockAuthService!
    var viewModel: LoginViewModel!
    
    init() {
        self.mockAuthService = MockAuthService()
        self.viewModel = LoginViewModel(authService: mockAuthService)
    }
    
    // MARK: - Tests
    
    @Test
    func emailValidation() {
        // Arrange
        viewModel.email = "invalid-email"
        
        // Act & Assert
        #expect(viewModel.isFormValid == false)
        
        // Fix email
        viewModel.email = "user@example.com"
        #expect(viewModel.isFormValid == true)
    }
    
    @Test
    func passwordNotEmpty() {
        viewModel.email = "user@example.com"
        viewModel.password = ""
        
        #expect(viewModel.isFormValid == false)
        
        viewModel.password = "password123"
        #expect(viewModel.isFormValid == true)
    }
    
    @Test("Login success scenario")
    async func loginSuccessful() async {
        // Arrange
        let expectedUser = User(
            id: UUID(),
            email: "user@example.com",
            name: "John Doe",
            createdAt: Date()
        )
        mockAuthService.loginResult = .success(expectedUser)
        
        viewModel.email = "user@example.com"
        viewModel.password = "password123"
        
        // Act
        await viewModel.login()
        
        // Assert
        #expect(viewModel.currentUser?.id == expectedUser.id)
        #expect(viewModel.error == nil)
        #expect(viewModel.isLoading == false)
    }
    
    @Test("Login failure: invalid email")
    async func loginWithInvalidEmail() async {
        // Arrange
        viewModel.email = "not-an-email"
        viewModel.password = "password"
        
        // Act
        await viewModel.login()
        
        // Assert
        #expect(viewModel.error == .invalidEmail)
        #expect(viewModel.currentUser == nil)
    }
    
    @Test("Login network error")
    async func loginNetworkError() async {
        // Arrange
        let networkError = URLError(.networkConnectionLost)
        mockAuthService.loginResult = .failure(.networkFailure(networkError))
        
        viewModel.email = "user@example.com"
        viewModel.password = "password"
        
        // Act
        await viewModel.login()
        
        // Assert
        #expect(viewModel.error != nil)
        guard case .networkFailure = viewModel.error else {
            Issue.record("Expected networkFailure error")
            return
        }
    }
}
```

---

## Parametrized Tests

```swift
@Suite
final class EmailValidationTests {
    @Test(arguments: [
        ("user@example.com", true),
        ("invalid-email", false),
        ("test@domain.co.uk", true),
        ("no-at-sign.com", false),
        ("", false)
    ])
    func emailValidation(email: String, isValid: Bool) {
        let validator = EmailValidator()
        #expect(validator.isValid(email) == isValid)
    }
}

@Suite
final class PaginationTests {
    @Test(arguments: [1, 2, 5, 10, 100])
    func paginationOffset(page: Int) {
        let pageSize = 20
        let expectedOffset = (page - 1) * pageSize
        
        let request = GetItemsRequest(page: page)
        #expect(request.offset == expectedOffset)
    }
}
```

---

## Mocking: Services

```swift
// Tests/Mocks/MockAuthService.swift
import Foundation

// Eredita da AuthService (se inheritance) o conformsa al protocollo
final class MockAuthService: AuthService {
    var loginResult: Result<User, AuthError> = .failure(.invalidEmail)
    var logoutCalled: Bool = false
    
    override func login(_ request: LoginRequest) async throws -> User {
        try loginResult.get()
    }
    
    override func logout() {
        logoutCalled = true
    }
}

// Tests/Mocks/MockAPIService.swift
final class MockAPIService: APIService {
    var executeResult: Any?
    var executeError: Error?
    var executeCalled: Int = 0
    
    override func execute<Request: APIRequest>(_ request: Request) async throws -> Request.Response {
        executeCalled += 1
        
        if let error = executeError {
            throw error
        }
        
        guard let result = executeResult as? Request.Response else {
            throw APIError.decodingFailure(DecodingError.dataCorrupted(.init(
                codingPath: [],
                debugDescription: "Mock result type mismatch"
            )))
        }
        
        return result
    }
}
```

---

## Fixtures: JSON Test Data

```swift
// Tests/Fixtures/user-response.json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "John Doe",
  "created_at": "2024-01-15T10:30:00Z"
}
```

```swift
// Tests/Models/UserTests.swift
import Testing
import Foundation

@Suite
final class UserDecodingTests {
    @Test
    func decodeUserFromFixture() throws {
        // Arrange
        guard let url = Bundle.module.url(forResource: "user-response", withExtension: "json") else {
            Issue.record("Fixture not found")
            return
        }
        let data = try Data(contentsOf: url)
        
        // Act
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        let user = try decoder.decode(User.self, from: data)
        
        // Assert
        #expect(user.email == "user@example.com")
        #expect(user.name == "John Doe")
    }
}
```

---

## APIService Testing

```swift
// Tests/Services/APIServiceTests.swift
import Testing
import Foundation

@Suite
final class APIServiceTests {
    private var mockURLSession: URLSession!
    private var apiService: APIService!
    
    init() {
        // Usa MockURLProtocol per intercettare URLSession
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [MockURLProtocol.self]
        mockURLSession = URLSession(configuration: config)
    }
    
    @Test
    async func executeSuccessfulRequest() async throws {
        // Arrange
        let expectedResponse = GetItemsResponse(items: [], total: 0)
        let data = try JSONEncoder().encode(expectedResponse)
        
        MockURLProtocol.mockResponse = (
            data: data,
            response: HTTPURLResponse(
                url: URL(string: "https://api.example.com/items")!,
                statusCode: 200,
                httpVersion: nil,
                headerFields: nil
            ),
            error: nil
        )
        
        let request = GetItemsRequest(limit: 10, offset: 0)
        
        // Act & Assert
        // (apiService.execute è async, quindi attendiamo risultato)
        // In questo caso usiamo mock APIService per testare
    }
    
    @Test("Handle 401 Unauthorized")
    async func handle401Error() {
        // Arrange
        MockURLProtocol.mockResponse = (
            data: Data(),
            response: HTTPURLResponse(
                url: URL(string: "https://api.example.com/items")!,
                statusCode: 401,
                httpVersion: nil,
                headerFields: nil
            ),
            error: nil
        )
        
        // Act & Assert
        // Expect APIError.unauthorized
    }
}

// URLProtocol per mockare risposte HTTP
class MockURLProtocol: URLProtocol {
    static var mockResponse: (data: Data, response: HTTPURLResponse, error: Error?)?
    
    override class func canInit(with request: URLRequest) -> Bool {
        true
    }
    
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        request
    }
    
    override func startLoading() {
        guard let (data, response, error) = Self.mockResponse else {
            client?.urlProtocol(self, didFailWithError: NSError(domain: "Mock", code: -1))
            return
        }
        
        if let error = error {
            client?.urlProtocol(self, didFailWithError: error)
        } else {
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
        }
        
        client?.urlProtocolDidFinishLoading(self)
    }
    
    override func stopLoading() {}
}
```

---

## Common Assertions

```swift
// Equality
#expect(value == expected)
#expect(value != notExpected)

// Boolean
#expect(condition)
#expect(!condition)

// Optional
#expect(optional != nil)
if let value = optional {
    #expect(value > 0)
}

// Error handling
#expect(throws: APIError.self) {
    try decodeInvalidJSON()
}

// Checking error type
do {
    try something()
    Issue.record("Should have thrown")
} catch let error as APIError {
    #expect(error == .invalidEmail)
}

// Collection
#expect(array.count == 3)
#expect(array.contains(where: { $0.id == expectedID }))
#expect(dictionary["key"] == "value")

// String
#expect(string.contains("substring"))
#expect(string.hasPrefix("prefix"))
```

---

## Async Tests

```swift
@Suite
final class AsyncTests {
    @Test
    async func loadDataAsynchronously() async {
        // Arrange
        let viewModel = HomeViewModel(apiService: MockAPIService())
        
        // Act
        await viewModel.loadItems()
        
        // Assert
        #expect(viewModel.items.isEmpty == false)
        #expect(viewModel.error == nil)
    }
    
    @Test
    async func handleLoadingState() async {
        let viewModel = HomeViewModel()
        
        // isLoading dovrebbe essere true durante il caricamento
        // Non possiamo testarla direttamente (timing issue),
        // ma testiamo lo stato finale
        
        await viewModel.loadItems()
        
        #expect(viewModel.isLoading == false)
    }
}
```

---

## Checklist: Test Coverage

- [ ] ViewModels testati (logica senza UI)
- [ ] Services testati (API, database)
- [ ] Models testati (validazione, decoding)
- [ ] Error cases testati (401, 404, network error)
- [ ] Edge cases (empty list, nil values)
- [ ] Async operations testati (await)
- [ ] Mocks usati per dipendenze esterne
- [ ] Fixtures per test data
- [ ] Niente test di UI (troppo fragili)
- [ ] Test suite gira in <5 secondi (veloce)

---

## Best Practices

✅ **Testa comportamento, non implementazione**
```swift
// GOOD: Testa cosa fa, non come lo fa
@Test
async func loginSetsCurrentUser() async {
    await viewModel.login()
    #expect(viewModel.currentUser != nil)
}

// BAD: Testa i dettagli implementativi
@Test
async func loginCallsAuthService() async {
    await viewModel.login()
    #expect(mockAuthService.loginCalled == true)
}
```

✅ **Testa un comportamento per test**
```swift
// GOOD: Un test = un'asserzione chiara
@Test("Valid email passes validation")
func validEmail() {
    #expect(emailValidator.isValid("user@example.com"))
}

// BAD: Troppe asserzioni in un test
@Test
func emailValidation() {
    #expect(emailValidator.isValid("user@example.com"))
    #expect(!emailValidator.isValid("invalid"))
    #expect(!emailValidator.isValid(""))
    // ... 10 altre asserzioni
}
```

✅ **Arrange-Act-Assert**
```swift
@Test
async func loginError() async {
    // Arrange
    mockAuthService.loginResult = .failure(.invalidEmail)
    viewModel.email = "bad"
    viewModel.password = "pwd"
    
    // Act
    await viewModel.login()
    
    // Assert
    #expect(viewModel.error == .invalidEmail)
}
```

---

## Running Tests

```bash
# Xcode
Cmd + U

# Terminal
swift test

# Specific test suite
swift test LoginViewModelTests

# Verbose output
swift test -v
```

---

## Related Skills

- **swiftui-architecture** — Testare ViewModel
- **ios-networking** — Mockare APIService

---

**Status:** ✅ Swift Testing framework (Swift 6.2+), no XCTest
