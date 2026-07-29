# Test Doubles

## Stub vs Spy

- Use a **Stub** when only returned data matters.
- Use a **Spy** when you must verify interactions (call, args, count, order).
- Name methods after business actions.

## Spying Approaches

### Spy Skeleton

```swift
final class WiFiConnectionUseCaseSpy: WiFiConnectionUseCase {
    struct ConnectCall: Equatable { let network: TestNetwork; let password: String? }

    var availableNetworksCallCount = 0
    private(set) var connectCalls: [ConnectCall] = []
    var connectResult: Result<Void, WifiConnectionError> = .success(())
    private var continuations: [AsyncStream<AvailableWifiNetworksElement<TestNetwork>>.Continuation] = []

    func availableNetworks() throws -> AsyncStream<AvailableWifiNetworksElement<TestNetwork>> {
        availableNetworksCallCount += 1
        return AsyncStream { continuation in continuations.append(continuation) }
    }

    func connect(to network: TestNetwork, password: String?) async throws {
        connectCalls.append(.init(network: network, password: password))
        try connectResult.get()
    }

    func simulateNetworkUpdate(_ networks: [TestNetwork]) {
        continuations.forEach { $0.yield(.networks(Set(networks))) }
    }
}
```

### 1. Call Count Tracking

Use for methods without arguments.
Pattern: `<methodName>CallCount`

```swift
final class MonitoringSpy: CoverageMonitoring {
    private(set) var startCallCount = 0

    func start() { startCallCount += 1 }
}

#expect(spy.startCallCount == 1)
```

### 2. Parameter Capture

Use for methods with arguments.
Pattern:
- `captured<MethodName>Calls`
- `Equatable` `<MethodName>Parameters`
- Assert full captured array

```swift
final class WiFiConnectionUseCaseSpy: WiFiConnectionUseCase {
    struct ConnectParameters: Equatable {
        let network: TestNetwork
        let password: String?
    }

    private(set) var capturedConnectParameters: [ConnectParameters] = []
    var connectionResult: Result<Void, WifiConnectionError> = .success(())

    func connect(to network: TestNetwork, password: String?) async throws {
        capturedConnectParameters.append(.init(network: network, password: password))
        try connectionResult.get()
    }
}
```

### 3. Message Calls (Cross-Method Ordering)

Use when ordering across multiple methods matters.
Pattern:
- `enum CapturedMessage: Equatable`
- `capturedMessages: [CapturedMessage]`
- Assert full ordered sequence

## Result Injection And Deterministic Simulation

- Store behavior in `Result` properties.
- Use `try result.get()` in spy methods.
- Add `simulate<Action>()` helpers for async paths.
- For `AsyncStream`, store continuations and yield explicitly.

```swift
final class WiFiConnectionUseCaseSpy: WiFiConnectionUseCase {
    private var continuations: [AsyncStream<AvailableWifiNetworksElement<TestNetwork>>.Continuation] = []
    var availableNetworksCallCount = 0

    func availableNetworks() throws -> AsyncStream<AvailableWifiNetworksElement<TestNetwork>> {
        availableNetworksCallCount += 1
        return AsyncStream { continuation in continuations.append(continuation) }
    }

    func simulateNetworkUpdate(with networks: [TestNetwork]) async {
        for continuation in continuations {
            continuation.yield(.networks(Set(networks)))
        }
        await Task.yield()
    }
}
```

## Deterministic Ordering Principle

Capture domain actions in order and assert the full sequence, not partial projections.
