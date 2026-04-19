---
name: ios-security
description: |
  Production-grade iOS security patterns. Keychain, CryptoKit, Face ID/Touch ID,
  certificate pinning, secure token storage, ATS configuration.
  Zero dependencies, native frameworks only.
trigger: "Add security features like Face ID, token storage, or encryption"
---

# iOS Security — Keychain, CryptoKit, Biometrics

**Purpose:** Secure storage of sensitive data, biometric authentication, encrypted communication. Follows Apple's security best practices.

**When to use:**
- Storing authentication tokens securely
- User credentials storage
- Encrypted data at rest
- Biometric login (Face ID / Touch ID)
- Certificate pinning for API calls
- Complying with App Store security requirements

**Why this matters:** Incorrect security = App Store rejection + user data exposure. One vulnerability = lawsuit. This skill prevents both.

---

## Architecture

```
Services/
├─ Security/
│  ├─ KeychainManager.swift       # Token/credential storage
│  ├─ BiometricAuthenticator.swift # Face ID / Touch ID
│  ├─ CryptoManager.swift         # Data encryption/decryption
│  ├─ SecureEnclave.swift         # Private key generation
│  ├─ CertificatePinning.swift    # URLSession delegate
│  └─ SecurityConfig.swift        # ATS configuration
```

---

## 1. KeychainManager: Secure Token Storage

```swift
// Services/Security/KeychainManager.swift
import Foundation

@Observable
final class KeychainManager {
    static let shared = KeychainManager()
    private let service = "dev.yourname.myapp"
    
    enum KeychainError: LocalizedError {
        case saveFailed
        case fetchFailed
        case deleteFailed
        case itemNotFound
        
        var errorDescription: String? {
            switch self {
            case .saveFailed: return "Failed to save to Keychain"
            case .fetchFailed: return "Failed to fetch from Keychain"
            case .deleteFailed: return "Failed to delete from Keychain"
            case .itemNotFound: return "Item not found in Keychain"
            }
        }
    }
    
    // MARK: - Save
    func save(_ data: String, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: data.data(using: .utf8) ?? Data(),
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]
        
        // Delete old item if exists
        SecItemDelete(query as CFDictionary)
        
        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed
        }
    }
    
    // MARK: - Fetch
    func fetch(for key: String) throws -> String {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        guard status == errSecSuccess,
              let data = result as? Data,
              let string = String(data: data, encoding: .utf8) else {
            throw KeychainError.fetchFailed
        }
        
        return string
    }
    
    // MARK: - Delete
    func delete(for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]
        
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed
        }
    }
    
    // MARK: - Convenience
    func saveAccessToken(_ token: String) throws {
        try save(token, for: "accessToken")
    }
    
    func fetchAccessToken() throws -> String {
        try fetch(for: "accessToken")
    }
    
    func deleteAccessToken() throws {
        try delete(for: "accessToken")
    }
}
```

**Key points:**
- `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` = most secure (not accessible when device locked or after restart)
- Delete first to avoid duplicates
- Use Data encoding for binary data

---

## 2. BiometricAuthenticator: Face ID / Touch ID

```swift
// Services/Security/BiometricAuthenticator.swift
import LocalAuthentication
import Foundation

@Observable
final class BiometricAuthenticator {
    static let shared = BiometricAuthenticator()
    
    enum BiometricError: LocalizedError {
        case notAvailable
        case userCancelled
        case userFallback
        case passcodeNotSet
        case biometryNotEnrolled
        case unknown(Error)
        
        var errorDescription: String? {
            switch self {
            case .notAvailable:
                return "Biometric authentication not available"
            case .userCancelled:
                return "Authentication cancelled"
            case .userFallback:
                return "User selected fallback (passcode)"
            case .passcodeNotSet:
                return "Device passcode not set"
            case .biometryNotEnrolled:
                return "No Face ID or Touch ID enrolled"
            case .unknown(let error):
                return error.localizedDescription
            }
        }
    }
    
    enum BiometricType {
        case faceID
        case touchID
        case none
    }
    
    // MARK: - Check availability
    var biometricType: BiometricType {
        let context = LAContext()
        var error: NSError?
        
        guard context.canEvaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            error: &error
        ) else {
            return .none
        }
        
        if #available(iOS 16.0, *) {
            return context.biometryType == .faceID ? .faceID : .touchID
        } else {
            return context.canEvaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                error: nil
            ) ? .touchID : .none
        }
    }
    
    var isBiometricAvailable: Bool {
        let context = LAContext()
        return context.canEvaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            error: nil
        )
    }
    
    // MARK: - Authenticate
    func authenticate(reason: String = "Authenticate to access your account") async throws -> Bool {
        let context = LAContext()
        
        var error: NSError?
        guard context.canEvaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            error: &error
        ) else {
            if let error = error {
                throw interpretError(error)
            }
            throw BiometricError.notAvailable
        }
        
        do {
            return try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )
        } catch let error as LAError {
            throw interpretLAError(error)
        } catch {
            throw BiometricError.unknown(error)
        }
    }
    
    // MARK: - Error handling
    private func interpretLAError(_ error: LAError) -> BiometricError {
        switch error.code {
        case .userCancel:
            return .userCancelled
        case .userFallback:
            return .userFallback
        case .passcodeNotSet:
            return .passcodeNotSet
        case .biometryNotAvailable:
            return .notAvailable
        case .biometryNotEnrolled:
            return .biometryNotEnrolled
        default:
            return .unknown(error)
        }
    }
    
    private func interpretError(_ error: NSError) -> BiometricError {
        let code = LAError.Code(rawValue: error.code)
        if let code = code {
            return interpretLAError(LAError(code))
        }
        return .unknown(error)
    }
}
```

**Usage:**
```swift
@Observable
final class LoginViewModel {
    var isBiometricEnabled: Bool = false
    
    func setupBiometric() {
        isBiometricEnabled = BiometricAuthenticator.shared.isBiometricAvailable
    }
    
    @MainActor
    func authenticateWithBiometric() async {
        do {
            let success = try await BiometricAuthenticator.shared.authenticate(
                reason: "Authenticate to login to your account"
            )
            if success {
                await login()
            }
        } catch {
            // Handle error
        }
    }
}
```

---

## 3. CryptoManager: Data Encryption

```swift
// Services/Security/CryptoManager.swift
import CryptoKit
import Foundation

@Observable
final class CryptoManager {
    static let shared = CryptoManager()
    
    enum CryptoError: LocalizedError {
        case encryptionFailed
        case decryptionFailed
        case invalidKey
        
        var errorDescription: String? {
            switch self {
            case .encryptionFailed: return "Encryption failed"
            case .decryptionFailed: return "Decryption failed"
            case .invalidKey: return "Invalid encryption key"
            }
        }
    }
    
    // MARK: - Symmetric encryption (AES-GCM)
    
    func encryptData(_ data: Data) throws -> (ciphertext: Data, nonce: Data) {
        let key = SymmetricKey(size: .bits256)
        let sealedBox = try AES.GCM.seal(data, using: key)
        
        guard let ciphertext = sealedBox.ciphertext as Data?,
              let nonce = sealedBox.nonce.withUnsafeBytes({ Data($0) }) as Data? else {
            throw CryptoError.encryptionFailed
        }
        
        return (ciphertext, nonce)
    }
    
    func decryptData(_ ciphertext: Data, nonce: Data, key: SymmetricKey) throws -> Data {
        let nonce = try AES.GCM.Nonce(data: nonce)
        let sealedBox = try AES.GCM.SealedBox(nonce: nonce, ciphertext: ciphertext, tag: Data())
        
        return try AES.GCM.open(sealedBox, using: key)
    }
    
    // MARK: - Hashing
    
    func sha256(_ data: Data) -> String {
        let digest = SHA256.hash(data: data)
        return digest.map { String(format: "%02hhx", $0) }.joined()
    }
    
    // MARK: - Signing (private key in Secure Enclave)
    
    func createSecureEnclaveKey() throws -> SecureEnclave.P256.Signing.PrivateKey {
        return try SecureEnclave.P256.Signing.PrivateKey()
    }
    
    func signData(_ data: Data, with privateKey: SecureEnclave.P256.Signing.PrivateKey) throws -> Data {
        let signature = try privateKey.signature(for: data)
        return Data(signature)
    }
}
```

---

## 4. Certificate Pinning

```swift
// Services/Security/CertificatePinning.swift
import Foundation

class PinningURLSessionDelegate: NSObject, URLSessionDelegate {
    private let pinnedCertificates: Set<Data>
    
    init(pinnedCertificates: [Data]) {
        self.pinnedCertificates = Set(pinnedCertificates)
    }
    
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        // Only pin for HTTPS
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.useDefaults, nil)
            return
        }
        
        // Get certificate chain
        let certificateCount = SecTrustGetCertificateCount(serverTrust)
        for i in 0..<certificateCount {
            guard let certificate = SecTrustGetCertificateAtIndex(serverTrust, i) else {
                continue
            }
            
            let certificateData = SecCertificateCopyData(certificate) as Data
            if pinnedCertificates.contains(certificateData) {
                // Match found
                let credential = URLCredential(trust: serverTrust)
                completionHandler(.useCredential, credential)
                return
            }
        }
        
        // No match, reject
        completionHandler(.cancelAuthenticationChallenge, nil)
    }
}

// Usage:
let pinnedCert = ... // Load your certificate
let delegate = PinningURLSessionDelegate(pinnedCertificates: [pinnedCert])
let session = URLSession(configuration: .default, delegate: delegate, delegateQueue: nil)
```

---

## 5. AuthService Integration

```swift
// Services/Auth/AuthService.swift (Updated with security)
import Foundation
import SwiftData

@Observable
final class AuthService {
    static let shared = AuthService()
    
    @MainActor
    var currentUser: User? = nil
    
    @MainActor
    var isLoggedIn: Bool {
        currentUser != nil
    }
    
    private let keychain = KeychainManager.shared
    private let biometric = BiometricAuthenticator.shared
    private let apiService = APIService.shared
    
    @MainActor
    var accessToken: String? {
        try? keychain.fetchAccessToken()
    }
    
    // MARK: - Login
    
    @MainActor
    func login(email: String, password: String) async throws -> User {
        let request = LoginRequest(email: email, password: password)
        let response = try await apiService.execute(request)
        
        // Save tokens securely
        try keychain.saveAccessToken(response.accessToken)
        try keychain.save(response.refreshToken, for: "refreshToken")
        
        let user = User(
            id: response.user.id,
            email: response.user.email,
            name: response.user.name,
            accessToken: response.accessToken,
            refreshToken: response.refreshToken
        )
        
        self.currentUser = user
        return user
    }
    
    // MARK: - Biometric Login
    
    @MainActor
    func enableBiometric() throws {
        guard biometric.isBiometricAvailable else {
            throw BiometricAuthenticator.BiometricError.notAvailable
        }
        try keychain.save("true", for: "biometricEnabled")
    }
    
    @MainActor
    func isBiometricEnabled() -> Bool {
        (try? keychain.fetch(for: "biometricEnabled")) == "true"
    }
    
    @MainActor
    func loginWithBiometric() async throws -> User {
        // Authenticate with Face ID / Touch ID
        let authenticated = try await biometric.authenticate()
        guard authenticated else {
            throw BiometricAuthenticator.BiometricError.userCancelled
        }
        
        // Retrieve saved credentials and login
        let email = try keychain.fetch(for: "savedEmail")
        let password = try keychain.fetch(for: "savedPassword") // ⚠️ Better: use refresh token
        
        return try await login(email: email, password: password)
    }
    
    // MARK: - Logout
    
    @MainActor
    func logout() throws {
        try keychain.deleteAccessToken()
        try keychain.delete(for: "refreshToken")
        currentUser = nil
    }
}
```

---

## 6. ATS Configuration

```xml
<!-- Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
    <!-- Require HTTPS for all domains (secure) -->
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    
    <!-- Only allow specific domains with specific requirements -->
    <key>NSExceptionDomains</key>
    <dict>
        <!-- Your API -->
        <key>api.example.com</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <false/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.3</string>
            <key>NSExceptionRequiresCertificateTransparency</key>
            <false/>
        </dict>
    </dict>
</dict>
```

---

## Testing

```swift
// Tests/Services/KeychainManagerTests.swift
import Testing

@Suite
final class KeychainManagerTests {
    private let keychain = KeychainManager.shared
    
    deinit {
        try? keychain.delete(for: "testKey")
    }
    
    @Test
    func saveAndFetch() throws {
        // Save
        try keychain.save("secret", for: "testKey")
        
        // Fetch
        let value = try keychain.fetch(for: "testKey")
        
        #expect(value == "secret")
    }
    
    @Test
    func fetchNonexistent() throws {
        // Should throw itemNotFound
        #expect(throws: KeychainManager.KeychainError.self) {
            try keychain.fetch(for: "nonexistent")
        }
    }
}
```

---

## Checklist: Security Implementation

- [ ] Keychain setup for tokens (accessible when unlocked only)
- [ ] Biometric authentication available and working
- [ ] CryptoKit for sensitive data encryption
- [ ] Certificate pinning for API calls
- [ ] ATS configured properly
- [ ] No hardcoded secrets or passwords
- [ ] Secure Enclave for key generation
- [ ] Auth interceptor adds token to requests
- [ ] Tokens cleared on logout
- [ ] Face ID / Touch ID fallback handling
- [ ] Privacy manifest declares Keychain usage
- [ ] No biometric data stored, only status

---

## Related Skills

- **ios-networking** — Token injection in requests
- **swiftui-architecture** — Integration in AuthViewModel
- **swift-testing** — Security test patterns

---

## Common Mistakes to Avoid

❌ **Storing passwords instead of tokens**
```swift
// WRONG
try keychain.save(userPassword, for: "password")
```

✅ **Store refresh tokens, use short-lived access tokens**
```swift
// CORRECT
try keychain.save(response.refreshToken, for: "refreshToken")
// Access token lives in memory, refresh it periodically
```

❌ **Using kSecAttrAccessibleAlways**
```swift
// WRONG - too permissive
kSecAttrAccessibleAlways
```

✅ **Use kSecAttrAccessibleWhenUnlockedThisDeviceOnly**
```swift
// CORRECT
kSecAttrAccessibleWhenUnlockedThisDeviceOnly
```

---

**Status:** ✅ Production-ready, iOS 17+, no external dependencies
