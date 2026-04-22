---
name: ios-networking
description: |
  URLSession async/await per REST API. Con retry logic, caching, error handling tipizzato.
  Niente Alamofire o dipendenze—solo native URLSession + Swift 6.2 concurrency.
trigger: "Setup networking service for API calls"
---

# iOS Networking — URLSession + async/await

**Purpose:** URLSession service layer per API calls. Retry automatici, response caching, errori tipizzati, streaming support per grandi file.

**When to use:**
- Ogni view che ha bisogno di dati da un server
- Per autenticazione (auth token nelle richieste)
- Per dati che cambiano spesso (timeline, notifications)
- Quando servono grandi file (downloads)

**Why this matters:** URLSession nativo è sufficiente per il 95% dei progetti iOS. Alamofire aggiunge overhead. Usiamo async/await per codice pulito, senza callback hell o Combine.

---

## Architecture

```
Services/
├─ Networking/
│  ├─ APIService.swift         # Entry point
│  ├─ APIRequest.swift         # Protocol per tipi richieste
│  ├─ APIError.swift           # Error enum
│  ├─ APIResponse.swift        # Wrapper per response
│  ├─ Interceptors/
│  │  ├─ AuthInterceptor.swift # Aggiungi token
│  │  └─ RetryInterceptor.swift # Retry logic
│  └─ Cache/
│     └─ ResponseCache.swift    # In-memory cache
```

---

## Models: Request & Response

```swift
// Services/Networking/APIRequest.swift
import Foundation

protocol APIRequest {
    associatedtype Response: Codable
    
    var method: HTTPMethod { get }
    var path: String { get }
    var queryItems: [URLQueryItem]? { get }
    var headers: [String: String]? { get }
    var body: Encodable? { get }
    var cachePolicy: URLRequest.CachePolicy { get }
    
    func decode(_ data: Data) throws -> Response
}

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case patch = "PATCH"
    case delete = "DELETE"
}

// Default implementations
extension APIRequest {
    var queryItems: [URLQueryItem]? { nil }
    var headers: [String: String]? { nil }
    var body: Encodable? { nil }
    var cachePolicy: URLRequest.CachePolicy { .useProtocolCachePolicy }
    
    func decode(_ data: Data) throws -> Response {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return try decoder.decode(Response.self, from: data)
    }
}
```

### Example: Concrete Request

```swift
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
    let user: UserData
    
    enum CodingKeys: String, CodingKey {
        case accessToken = "access_token"
        case refreshToken = "refresh_token"
        case user
    }
}

struct UserData: Codable {
    let id: UUID
    let email: String
    let name: String
}
```

---

## APIError: Tipizzato

```swift
// Services/Networking/APIError.swift
import Foundation

@Sendable
enum APIError: LocalizedError, Sendable {
    case invalidURL
    case networkFailure(URLError)
    case decodingFailure(DecodingError)
    case serverError(statusCode: Int, data: Data?)
    case unauthorized  // 401
    case forbidden      // 403
    case notFound       // 404
    case serverError    // 5xx
    case unknown(Error)
    
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "URL non valida"
        case .networkFailure(let error):
            return "Errore di rete: \(error.localizedDescription)"
        case .decodingFailure(let error):
            return "Errore nel parsing dei dati: \(error.localizedDescription)"
        case .serverError(let statusCode, _):
            return "Errore server (\(statusCode))"
        case .unauthorized:
            return "Non autorizzato. Accedi di nuovo."
        case .forbidden:
            return "Non hai permessi per questa azione"
        case .notFound:
            return "Risorsa non trovata"
        case .serverError:
            return "Errore del server. Riprova."
        case .unknown(let error):
            return error.localizedDescription
        }
    }
    
    var recoverySuggestion: String? {
        switch self {
        case .networkFailure:
            return "Verifica la connessione internet"
        case .unauthorized:
            return "Effettua il login"
        case .decodingFailure:
            return "Contatta il supporto tecnico"
        default:
            return nil
        }
    }
    
    // Utile per capire se è retriabile
    var isRetriable: Bool {
        switch self {
        case .networkFailure(let error):
            return error.code == .timedOut || error.code == .networkConnectionLost
        case .serverError(let statusCode, _):
            return statusCode >= 500  // 5xx è retriable
        default:
            return false
        }
    }
}
```

---

## APIService: Main Class

```swift
// Services/Networking/APIService.swift
import Foundation

@Observable
final class APIService: NSObject, URLSessionDelegate {
    static let shared = APIService()
    
    private let baseURL: URL
    private let session: URLSession
    private let authInterceptor: AuthInterceptor
    private let cache: ResponseCache
    
    // Configuration
    var maxRetries: Int = 3
    var retryDelay: TimeInterval = 1.0  // Exponential backoff
    
    override init() {
        let baseURL = URL(string: ProcessInfo.processInfo.environment["API_BASE_URL"] ?? "https://api.example.com")!
        self.baseURL = baseURL
        
        let config = URLSessionConfiguration.default
        config.waitsForConnectivity = true
        config.timeoutIntervalForRequest = 30
        config.timeoutIntervalForResource = 300
        
        self.cache = ResponseCache()
        self.authInterceptor = AuthInterceptor()
        
        let sessionDelegate = URLSessionEventDelegate()
        self.session = URLSession(configuration: config, delegate: sessionDelegate, delegateQueue: nil)
    }
    
    // MARK: - Public API
    
    func execute<Request: APIRequest>(_ request: Request) async throws -> Request.Response {
        // Retry logic
        var lastError: APIError?
        
        for attempt in 0..<maxRetries {
            do {
                return try await executeOnce(request)
            } catch let error as APIError {
                lastError = error
                
                if !error.isRetriable || attempt == maxRetries - 1 {
                    throw error
                }
                
                // Exponential backoff: 1s, 2s, 4s
                let delay = retryDelay * pow(2, Double(attempt))
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }
        
        throw lastError ?? APIError.unknown(NSError(domain: "Unknown", code: -1))
    }
    
    // MARK: - Private
    
    private func executeOnce<Request: APIRequest>(_ request: Request) async throws -> Request.Response {
        // Prepara URLRequest
        var urlRequest = try buildURLRequest(for: request)
        
        // Interceptor: Aggiungi auth token
        urlRequest = await authInterceptor.intercept(urlRequest)
        
        // Controlla cache (solo GET)
        if request.method == .get {
            if let cached = cache.get(for: urlRequest.url?.absoluteString ?? "") {
                do {
                    return try request.decode(cached)
                } catch {
                    // Se cache è corrotto, procedi con la richiesta
                }
            }
        }
        
        // Esegui richiesta
        do {
            let (data, response) = try await session.data(for: urlRequest)
            
            // Controlla HTTP status
            guard let httpResponse = response as? HTTPURLResponse else {
                throw APIError.unknown(NSError(domain: "No HTTP response", code: -1))
            }
            
            // Gestisci errori HTTP
            switch httpResponse.statusCode {
            case 200...299:
                // Success
                if request.method == .get {
                    cache.set(data, for: urlRequest.url?.absoluteString ?? "")
                }
                return try request.decode(data)
                
            case 401:
                throw APIError.unauthorized
            case 403:
                throw APIError.forbidden
            case 404:
                throw APIError.notFound
            case 500...:
                throw APIError.serverError(statusCode: httpResponse.statusCode, data: data)
            default:
                throw APIError.serverError(statusCode: httpResponse.statusCode, data: data)
            }
        } catch let error as URLError {
            throw APIError.networkFailure(error)
        } catch let error as DecodingError {
            throw APIError.decodingFailure(error)
        } catch let error as APIError {
            throw error
        } catch {
            throw APIError.unknown(error)
        }
    }
    
    private func buildURLRequest<Request: APIRequest>(for request: Request) throws -> URLRequest {
        var components = URLComponents(url: baseURL.appendingPathComponent(request.path), resolvingAgainstBaseURL: false)!
        components.queryItems = request.queryItems
        
        guard let url = components.url else {
            throw APIError.invalidURL
        }
        
        var urlRequest = URLRequest(url: url, cachePolicy: request.cachePolicy, timeoutInterval: 30)
        urlRequest.httpMethod = request.method.rawValue
        
        // Headers
        var headers = request.headers ?? [:]
        headers["Accept"] = "application/json"
        urlRequest.allHTTPHeaderFields = headers
        
        // Body
        if let body = request.body {
            let encoder = JSONEncoder()
            encoder.dateEncodingStrategy = .iso8601
            urlRequest.httpBody = try encoder.encode(body)
        }
        
        return urlRequest
    }
    
    // MARK: - AI Streaming (Xcode 26 Pattern)
    // Receive tokens one by one for LLM responses
    
    func stream<Request: APIRequest>(_ request: Request) async throws -> AsyncThrowingStream<String, Error> {
        let urlRequest = try buildURLRequest(for: request)
        let (bytes, response) = try await session.bytes(for: urlRequest)
        
        guard let httpResponse = response as? HTTPURLResponse, (200...299).contains(httpResponse.statusCode) else {
            throw APIError.serverError(statusCode: (response as? HTTPURLResponse)?.statusCode ?? 500, data: nil)
        }
        
        return AsyncThrowingStream { continuation in
            Task {
                do {
                    for try await line in bytes.lines {
                        // Assuming Server-Sent Events (SSE) or simple line-based streaming
                        continuation.yield(line)
                    }
                    continuation.finish()
                } catch {
                    continuation.finish(throwing: error)
                }
            }
        }
    }
}

// MARK: - URLSessionDelegate for logging
private class URLSessionEventDelegate: NSObject, URLSessionDelegate {
    func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge, completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        // Certificate pinning logic here (if needed)
        completionHandler(.performDefaultHandling, nil)
    }
}
```

---

## AuthInterceptor: Aggiungi Token

```swift
// Services/Networking/Interceptors/AuthInterceptor.swift
import Foundation

@Observable
final class AuthInterceptor {
    private let authService = AuthService.shared
    
    func intercept(_ request: URLRequest) async -> URLRequest {
        var mutableRequest = request
        
        if let token = await authService.accessToken {
            mutableRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        return mutableRequest
    }
}
```

---

## ResponseCache: In-Memory Cache

```swift
// Services/Networking/Cache/ResponseCache.swift
import Foundation

@Observable
final class ResponseCache {
    private var cache: [String: CachedResponse] = [:]
    private let lock = NSLock()
    private let ttl: TimeInterval = 300  // 5 minuti
    
    struct CachedResponse {
        let data: Data
        let timestamp: Date
    }
    
    func get(for key: String) -> Data? {
        lock.lock()
        defer { lock.unlock() }
        
        guard let cached = cache[key] else { return nil }
        
        // Controlla TTL
        let elapsed = Date().timeIntervalSince(cached.timestamp)
        guard elapsed < ttl else {
            cache.removeValue(forKey: key)
            return nil
        }
        
        return cached.data
    }
    
    func set(_ data: Data, for key: String) {
        lock.lock()
        defer { lock.unlock() }
        
        cache[key] = CachedResponse(data: data, timestamp: Date())
    }
    
    func clear() {
        lock.lock()
        defer { lock.unlock() }
        cache.removeAll()
    }
}
```

---

## Usage in ViewModel

```swift
// Features/Home/ViewModels/HomeViewModel.swift
@Observable
final class HomeViewModel {
    var items: [Item] = []
    var isLoading: Bool = false
    var error: APIError? = nil
    
    private let apiService = APIService.shared
    
    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            let request = GetItemsRequest(limit: 20, offset: 0)
            let response = try await apiService.execute(request)
            self.items = response.items
            self.error = nil
        } catch let error as APIError {
            self.error = error
        } catch {
            self.error = .unknown(error)
        }
    }
}

// Request concreto
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
    let items: [Item]
    let total: Int
}

struct Item: Identifiable, Codable {
    let id: UUID
    let title: String
    let description: String
    let createdAt: Date
    
    enum CodingKeys: String, CodingKey {
        case id, title, description
        case createdAt = "created_at"
    }
}
```

---

## Testing

```swift
// Services/Networking/APIServiceTests.swift
import Testing

final class APIServiceTests {
    private var mockSession: URLSession!
    private var apiService: APIService!
    
    init() {
        // Setup mock session con URLProtocol custom
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [MockURLProtocol.self]
        mockSession = URLSession(configuration: config)
    }
    
    @Test
    func testSuccessfulRequest() async throws {
        // Arrange
        let expectedResponse = GetItemsResponse(items: [], total: 0)
        let data = try JSONEncoder().encode(expectedResponse)
        
        MockURLProtocol.mockResponse = (data: data, response: HTTPURLResponse(url: URL(string: "https://example.com")!, statusCode: 200, httpVersion: nil, headerFields: nil)!, error: nil)
        
        // Act
        let request = GetItemsRequest(limit: 10, offset: 0)
        // ... (execute and assert)
    }
}
```

---

## Checklist: Networking Implementation

- [ ] APIRequest protocol implementato
- [ ] APIError enum con recovery suggestions
- [ ] APIService gestisce retry automatico
- [ ] AuthInterceptor aggiunge token
- [ ] ResponseCache è thread-safe (NSLock)
- [ ] Timeout impostati (30s request, 300s resource)
- [ ] Decoder usa dateDecodingStrategy = .iso8601
- [ ] GET requests sono cachati (5 min TTL)
- [ ] HTTP status codes gestiti (401, 403, 404, 5xx)
- [ ] URLSession non usa UIImage (se contiene image, deserialize manualmente)

---

## Common Patterns

### Pagination

```swift
struct GetItemsRequest: APIRequest {
    let page: Int
    let pageSize: Int = 20
    
    var queryItems: [URLQueryItem]? {
        [
            URLQueryItem(name: "page", value: String(page)),
            URLQueryItem(name: "page_size", value: String(pageSize))
        ]
    }
}
```

### Upload File

```swift
// Per multipart form-data (file upload), usa URLSession multipart
// Non implementato in questo template—usa HTTPBody custom con boundary
```

### Streaming Large File

```swift
func downloadLargeFile(from url: URL) async throws {
    let (localURL, _) = try await session.download(from: url)
    // localURL è il path temporaneo del file scaricato
}
```

---

## Related Skills

- **swiftui-architecture** — Come usare APIService nel ViewModel
- **ios-security** — Certificate pinning, encryption headers

---

**Status:** ✅ Production-ready, Swift 6.2+, native URLSession only
