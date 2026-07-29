# Core Rules

## 1. Use WHEN_THEN Naming

Use:
- `when[ARRANGED SITUATION]_then[EXPECTED BEHAVIOR]`
- Add `And` when chaining conditions/outcomes.

Example:

```swift
func whenLocationDisabled_thenAvailableNetworksDeliversLocationError()
func whenLocationEnabled_thenAvailableNetworksDeliversNetworksUpdates()
func whenLocationEnabledAfterDisabled_thenAvailableNetworksDeliversNetworksUpdates()
```

## 2. Centralize SUT Creation with `makeSUT`

Always create SUT in `makeSUT`:
- Keep **one** `makeSUT` per suite. Express scenario variations as parameters with sensible defaults — never add per-scenario overloads or variants like `makeSUTWithFailingConnection`.
- Arguments are **scenario arrangements** (expected results, states, conditions) — not stubs or spies directly. The caller describes *what should happen*, not *how dependencies are wired*.
- Give every scenario parameter a default representing the happy path; each test passes only the arguments its behavior cares about.
- Create stubs/spies **inside** `makeSUT` — dependency injection is an internal detail hidden from the test body.
- Return SUT and collaborators (stubs/spies) as a tuple so tests can verify interactions.
- Use local `sut` variable in each test.

```swift
func makeSUT(
    connectionResult: Result<Bool, ConnectionError> = .success(true),
    locationAuthorized: Bool = true
) -> (SUT, LocationServicesSpy, WiFiConnectionStub) {
    let locationService = LocationServicesSpy(authorized: locationAuthorized)
    let wifiService = WiFiConnectionStub(connectionResult: connectionResult)
    let sut = LocationAwareWiFiConnectionUseCase(
        locationService: locationService,
        decoratedUseCase: wifiService
    )
    return (sut, locationService, wifiService)
}

// Each test arranges only what its behavior depends on:
let (sut, _, _) = makeSUT(locationAuthorized: false)
```

## 3. Use Fixture Factories

Use `makeXXX` helpers (`makeNetwork`, `makeError`, etc.) to:
- Keep tests behavior-focused
- Hide irrelevant inputs behind defaults
- Avoid setup boilerplate
- Decouple from concrete types

## 4. Keep Focus on Business Logic

Tests should read like business requirements.

Target outcomes:
- Implementation independence
- Reduced duplication
- Easier maintenance
- Clear readability
- Domain-level assertions instead of internal-state checks

## 5. Name Fixtures for Intent

Use semantic names instead of raw literals.

```swift
private let viennaCoordinate = CLLocationCoordinate2D(latitude: 48.2082, longitude: 16.3738)
#expect(sut.region.center == viennaCoordinate)
```

## 6. Wrap Shared Setup and Assertions in Helpers

Encapsulate repeated behavior in helper functions and forward `SourceLocation` for precise failures.

```swift
try await expectFenceItems(expected, in: sut, sourceLocation: sourceLocation) {
    try await action()
}
```

## 7. Prefer Cohesive `#expect`

Keep one business rule in one assertion when possible.

```swift
#expect(sut.visibleFences.map(\.isSelected) == [false, true, false])
```

## 8. Use `#expect(throws:)` Variants for Error Paths

Match the right overload to the assertion need:

```swift
// Specific error value (preferred when Equatable)
#expect(throws: WifiConnectionError.notFound) {
    try sut.connect(to: unknownNetwork)
}

// Specific error type
#expect(throws: WifiConnectionError.self) {
    try sut.connect(to: invalidNetwork)
}

// Any error
#expect(throws: (any Error).self) {
    try dangerousOperation()
}

// Assert NO error is thrown
#expect(throws: Never.self) {
    try sut.connect(to: validNetwork)
}
```

## 9. Use `Issue.record()` for Fine-Grained Throw Testing

When you need to assert a specific error case with associated values, `do/catch` with `Issue.record` gives full control:

```swift
@Test func whenConnectingWithWrongPassword_thenReportsCredentialError() {
    do {
        try sut.connect(password: "wrong")
        Issue.record("Expected an error to be thrown.")
    } catch WifiConnectionError.incorrectCredentials(let network) {
        #expect(network == "SecureNetwork")
    } catch {
        Issue.record("Wrong error thrown: \(error)")
    }
}
```

## 10. Forward `SourceLocation` in Verification Helpers

Use `#_sourceLocation` (underscore required) so failures report the call site, not the helper internals:

```swift
func expectNetworks(
    _ expected: [Network],
    in sut: SUT,
    sourceLocation: SourceLocation = #_sourceLocation
) {
    #expect(sut.networks == expected, sourceLocation: sourceLocation)
}
```

Both `#expect` and `#require` accept `sourceLocation:`. Always forward it in shared assertion helpers.

## 11. Add Custom Test Debug Descriptions

Conform frequently asserted models to:
- `CustomTestStringConvertible`
- `CustomDebugStringConvertible`

Group these extensions under `// MARK: - SwiftTesting Debug Support`.

## 12. Use `// MARK: -` for Structure

Use section markers for major zones:
- Test suites
- `makeSUT` and factories
- Test doubles
- Debug support extensions
