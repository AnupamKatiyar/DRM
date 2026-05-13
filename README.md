# Client-Side DRM SDK — High Level + Low Level Design

# Functional Requirements

1. Acquire DRM licenses from remote license server.
2. Support online and offline playback.
3. Securely store licenses and certificates.
4. Support playback session management.
5. Handle key rotation.
6. Retry transient failures.
7. Provide telemetry and observability.
8. Support multiple DRM systems.
9. Integrate with media player.
10. Handle license expiry and renewals.
11. Detect device capability.
12. Handle concurrent playback sessions safely.
13. Support secure playback pipeline.
14. Handle tamper/root/jailbreak detection.

---

# Non-Functional Requirements

1. Thread-safe.
2. Low startup latency.
3. Scalable for millions of playback sessions.
4. Fault tolerant.
5. Highly observable.
6. Secure key handling.
7. Extensible architecture.
8. Minimal memory overhead.
9. Platform portable.
10. Maintainable and testable.

---

# High-Level Design (HLD)

```text
                ┌────────────────────┐
                │    Video Player    │
                └─────────┬──────────┘
                          │
                          ▼
                ┌────────────────────┐
                │ DRM SDK Facade     │
                └─────────┬──────────┘
                          │
      ┌────────────────────────────────────┐
      │                                    │
      │ License Manager                    │
      │ Session Manager                    │
      │ Policy Engine                      │
      │ Secure Storage                     │
      │ Certificate Manager                │
      │ Retry Handler                      │
      │ Telemetry Manager                  │
      │ Device Capability Detector         │
      │ Tamper Detection Module            │
      │ Offline License Manager            │
      │                                    │
      └────────────────────────────────────┘
                          │
                          ▼
                ┌────────────────────┐
                │ License Server     │
                └────────────────────┘
```

---

# Playback Flow

```text
Player Starts Playback
        ↓
Manifest Parsing
        ↓
DRM Type Detection
        ↓
Session Creation
        ↓
License Request Generation
        ↓
License Server Validation
        ↓
License Response
        ↓
Secure Storage
        ↓
Policy Validation
        ↓
Decrypt Segments
        ↓
Secure Decoder Path
        ↓
Playback
```

---

# Low-Level Design (LLD)

# Core Design Patterns Used

| Pattern | Why Used |
|---|---|
| Facade | Unified SDK API |
| Strategy | Multi-DRM support |
| Factory | DRM provider creation |
| Singleton | Telemetry manager |
| Observer | Playback state updates |
| Repository | Secure storage abstraction |
| Builder | License request construction |
| State Machine | Session state management |

---

# 1. DRM Type

```swift
import Foundation

enum DRMType {
    case playReady
    case widevine
    case fairPlay
}
```

---

# 2. Playback Session State Machine

```swift
enum PlaybackSessionState {
    case idle
    case requestingLicense
    case active
    case expired
    case error
    case closed
}
```

---

# 3. Session Model

```swift
struct PlaybackSession {
    let sessionId: UUID
    let contentId: String
    let drmType: DRMType

    var state: PlaybackSessionState
    var createdAt: Date
}
```

---

# 4. License Model

```swift
struct License {
    let keyId: String
    let encryptedKey: Data
    let expiryDate: Date
    let offlineAllowed: Bool
}
```

---

# 5. Strategy Pattern — DRM Provider

```swift
protocol DRMProvider {

    func generateLicenseChallenge(contentId: String) async throws -> Data

    func processLicenseResponse(_ data: Data) async throws -> License

    func decryptSegment(_ encryptedData: Data,
                        using license: License) async throws -> Data
}
```

---

# 6. PlayReady Provider

```swift
final class PlayReadyProvider: DRMProvider {

    func generateLicenseChallenge(contentId: String) async throws -> Data {

        return Data("PLAYREADY_CHALLENGE_\(contentId)".utf8)
    }

    func processLicenseResponse(_ data: Data) async throws -> License {

        return License(
            keyId: UUID().uuidString,
            encryptedKey: data,
            expiryDate: Date().addingTimeInterval(3600),
            offlineAllowed: true
        )
    }

    func decryptSegment(_ encryptedData: Data,
                        using license: License) async throws -> Data {

        // Simulated decryption
        return encryptedData
    }
}
```

---

# 7. Widevine Provider

```swift
final class WidevineProvider: DRMProvider {

    func generateLicenseChallenge(contentId: String) async throws -> Data {

        return Data("WIDEVINE_CHALLENGE_\(contentId)".utf8)
    }

    func processLicenseResponse(_ data: Data) async throws -> License {

        return License(
            keyId: UUID().uuidString,
            encryptedKey: data,
            expiryDate: Date().addingTimeInterval(7200),
            offlineAllowed: false
        )
    }

    func decryptSegment(_ encryptedData: Data,
                        using license: License) async throws -> Data {

        return encryptedData
    }
}
```

---

# 8. Factory Pattern

```swift
final class DRMProviderFactory {

    static func provider(for type: DRMType) -> DRMProvider {

        switch type {
        case .playReady:
            return PlayReadyProvider()

        case .widevine:
            return WidevineProvider()

        case .fairPlay:
            return PlayReadyProvider()
        }
    }
}
```

---

# 9. Thread-Safe Secure Storage

```swift
protocol SecureStorage {

    func saveLicense(_ license: License,
                     for contentId: String) async

    func fetchLicense(for contentId: String) async -> License?

    func deleteLicense(for contentId: String) async
}
```

---

# Actor-Based Storage

```swift
actor InMemorySecureStorage: SecureStorage {

    private var storage: [String: License] = [:]

    func saveLicense(_ license: License,
                     for contentId: String) async {

        storage[contentId] = license
    }

    func fetchLicense(for contentId: String) async -> License? {

        return storage[contentId]
    }

    func deleteLicense(for contentId: String) async {

        storage.removeValue(forKey: contentId)
    }
}
```

---

# Why Actor?

Using Swift Actor:

- prevents race conditions
- isolates mutable shared state
- provides thread safety automatically
- avoids manual locking complexity

---

# 10. Telemetry Manager (Singleton)

```swift
final class TelemetryManager {

    static let shared = TelemetryManager()

    private init() {}

    func log(_ event: String) {
        print("[Telemetry] \(event)")
    }
}
```

---

# 11. Retry Handler

```swift
final class RetryHandler {

    func execute<T>(
        maxRetries: Int = 3,
        operation: @escaping () async throws -> T
    ) async throws -> T {

        var currentRetry = 0

        while true {

            do {
                return try await operation()

            } catch {

                currentRetry += 1

                if currentRetry > maxRetries {
                    throw error
                }

                let delay = UInt64(currentRetry * 1_000_000_000)
                try await Task.sleep(nanoseconds: delay)
            }
        }
    }
}
```

---

# Why Exponential Backoff?

Prevents:

- retry storms
- cascading failures
- server overload

Improves resilience.

---

# 12. Policy Engine

```swift
final class PolicyEngine {

    func validate(_ license: License) -> Bool {

        return license.expiryDate > Date()
    }
}
```

---

# 13. Tamper Detection

```swift
final class TamperDetectionManager {

    func isDeviceCompromised() -> Bool {

        #if targetEnvironment(simulator)
        return false
        #else
        return false
        #endif
    }
}
```

---

# Real Production Checks

Production DRM SDKs typically check:

- debugger attachment
- jailbreak/root indicators
- writable system paths
- dynamic instrumentation tools
- integrity validation
- emulator detection

---

# 14. Device Capability Detector

```swift
struct DeviceCapabilities {
    let supportsHDR: Bool
    let supportsHardwareDRM: Bool
    let supportsOfflinePlayback: Bool
}

final class DeviceCapabilityDetector {

    func detect() -> DeviceCapabilities {

        return DeviceCapabilities(
            supportsHDR: true,
            supportsHardwareDRM: true,
            supportsOfflinePlayback: true
        )
    }
}
```

---

# 15. License Network Layer

```swift
protocol LicenseNetworking {

    func acquireLicense(challenge: Data) async throws -> Data
}
```

---

# URLSession Implementation

```swift
final class LicenseAPIClient: LicenseNetworking {

    func acquireLicense(challenge: Data) async throws -> Data {

        // Simulated network request

        try await Task.sleep(nanoseconds: 1_000_000_000)

        return Data("LICENSE_RESPONSE".utf8)
    }
}
```

---

# 16. Session Manager

```swift
actor SessionManager {

    private var sessions: [UUID: PlaybackSession] = [:]

    func createSession(contentId: String,
                       drmType: DRMType) -> PlaybackSession {

        let session = PlaybackSession(
            sessionId: UUID(),
            contentId: contentId,
            drmType: drmType,
            state: .idle,
            createdAt: Date()
        )

        sessions[session.sessionId] = session

        return session
    }

    func updateState(sessionId: UUID,
                     state: PlaybackSessionState) {

        guard var session = sessions[sessionId] else {
            return
        }

        session.state = state
        sessions[sessionId] = session
    }

    func removeSession(_ sessionId: UUID) {
        sessions.removeValue(forKey: sessionId)
    }
}
```

---

# Why Actor-Based Session Management?

Playback systems are highly concurrent.

Without synchronization:

- duplicate sessions
- race conditions
- stale session cleanup
- inconsistent state transitions

could occur.

---

# 17. Offline License Manager

```swift
final class OfflineLicenseManager {

    private let storage: SecureStorage

    init(storage: SecureStorage) {
        self.storage = storage
    }

    func downloadOfflineLicense(
        license: License,
        contentId: String
    ) async {

        await storage.saveLicense(license,
                                  for: contentId)
    }

    func fetchOfflineLicense(
        contentId: String
    ) async -> License? {

        return await storage.fetchLicense(for: contentId)
    }
}
```

---

# 18. Main DRM SDK Facade

```swift
final class DRMSDK {

    private let storage: SecureStorage
    private let networking: LicenseNetworking
    private let retryHandler: RetryHandler
    private let policyEngine: PolicyEngine
    private let sessionManager: SessionManager
    private let tamperManager: TamperDetectionManager

    init(
        storage: SecureStorage = InMemorySecureStorage(),
        networking: LicenseNetworking = LicenseAPIClient(),
        retryHandler: RetryHandler = RetryHandler(),
        policyEngine: PolicyEngine = PolicyEngine(),
        sessionManager: SessionManager = SessionManager(),
        tamperManager: TamperDetectionManager = TamperDetectionManager()
    ) {

        self.storage = storage
        self.networking = networking
        self.retryHandler = retryHandler
        self.policyEngine = policyEngine
        self.sessionManager = sessionManager
        self.tamperManager = tamperManager
    }

    func preparePlayback(
        contentId: String,
        drmType: DRMType
    ) async throws {

        if tamperManager.isDeviceCompromised() {
            throw NSError(domain: "Device compromised",
                          code: 403)
        }

        let session = await sessionManager.createSession(
            contentId: contentId,
            drmType: drmType
        )

        await sessionManager.updateState(
            sessionId: session.sessionId,
            state: .requestingLicense
        )

        let provider = DRMProviderFactory.provider(for: drmType)

        let challenge = try await provider
            .generateLicenseChallenge(contentId: contentId)

        let licenseData = try await retryHandler.execute {

            try await self.networking
                .acquireLicense(challenge: challenge)
        }

        let license = try await provider
            .processLicenseResponse(licenseData)

        guard policyEngine.validate(license) else {

            throw NSError(domain: "License expired",
                          code: 401)
        }

        await storage.saveLicense(
            license,
            for: contentId
        )

        await sessionManager.updateState(
            sessionId: session.sessionId,
            state: .active
        )

        TelemetryManager.shared.log(
            "Playback ready for content: \(contentId)"
        )
    }
}
```

---

# 19. Player Integration Layer

```swift
final class VideoPlayer {

    private let drmSDK = DRMSDK()

    func play(contentId: String) async {

        do {

            try await drmSDK.preparePlayback(
                contentId: contentId,
                drmType: .playReady
            )

            print("Playback Started")

        } catch {

            print("Playback Failed: \(error)")
        }
    }
}
```

---

# End-to-End Playback Example

```swift
let player = VideoPlayer()

Task {
    await player.play(contentId: "movie_123")
}
```

---

# Production-Grade Enhancements

# 1. Key Rotation

Support periodic license renewal.

```text
License nearing expiry
        ↓
Background renewal
        ↓
Update active decryption key
```

---

# 2. Hardware DRM Integration

Integrate with:

- Widevine Modular
- PlayReady SL3000
- Apple FairPlay
- TEE
- Secure Enclave

---

# 3. Secure Decoder Path

Decrypted frames should never:

- touch insecure memory
- be accessible by screen capture APIs
- leak into application buffers

---

# 4. Observability

Track:

- playback startup latency
- license acquisition latency
- license failure rate
- retry counts
- decoder failures
- CDN issues
- session crashes

---

# 5. Scalability Considerations

# Problem

Millions of playback sessions.

---

# Optimizations

## Session Cleanup

Use:

```text
TTL cleanup
Background eviction
LRU inactive session cleanup
```

---

## Reduce Lock Contention

Prefer:

- actors
- immutable objects
- sharded caches
- lock-free queues

---

## Network Efficiency

Use:

- HTTP keep alive
- connection pooling
- request batching
- retry throttling

---

# 6. Failure Scenarios

| Failure | Mitigation |
|---|---|
| License server down | Retry + fallback endpoint |
| Expired certificate | Auto rotation |
| Corrupted offline license | Re-download |
| Device compromised | Block playback |
| CDN unavailable | Multi-CDN failover |
| Playback session leak | TTL cleanup |

---

# 7. Unit Testing Strategy

# Test Areas

- license parsing
- retry logic
- policy validation
- concurrent sessions
- offline playback
- expiry handling
- session cleanup
- telemetry correctness

---

# Example Unit Test

```swift
func testExpiredLicenseFailsValidation() {

    let engine = PolicyEngine()

    let license = License(
        keyId: "1",
        encryptedKey: Data(),
        expiryDate: Date().addingTimeInterval(-100),
        offlineAllowed: false
    )

    assert(engine.validate(license) == false)
}
```

---

# Time Complexity Analysis

| Operation | Complexity |
|---|---|
| Session Lookup | O(1) |
| License Cache Lookup | O(1) |
| Session Insert | O(1) |
| Session Cleanup | O(n) |
| Retry Execution | O(retries) |

---

# Space Complexity

| Component | Complexity |
|---|---|
| Session Store | O(n) |
| License Cache | O(n) |
| Telemetry Buffer | O(events) |

---

# Why This Design Is Strong For DRM SDK Interviews

This architecture demonstrates:

- scalable systems thinking
- concurrency awareness
- security-first design
- production observability
- maintainability
- modern Swift concurrency
- modular architecture
- extensibility
- playback pipeline understanding
- DRM lifecycle understanding

---

# Final Summary Statement

"The goal of a DRM SDK is not only secure decryption, but reliable, scalable, observable, and policy-compliant playback across heterogeneous platforms and network conditions while minimizing latency and maintaining strong content protection guarantees."

