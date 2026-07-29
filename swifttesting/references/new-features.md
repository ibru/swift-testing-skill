# New Swift Testing Features

Features from Swift 6.1 and 6.2+ that are absent from most LLM training data. Follow these instructions carefully — do not guess or hallucinate API shapes.

## Range-Based Confirmations (Swift 6.1+)

`confirmation()` accepts a range of expected counts, not just a fixed integer:

```swift
// At least 5 confirmations
await confirmation(expectedCount: 5...) { confirm in
    for await _ in loader { confirm() }
}

// Between 3 and 7 confirmations
await confirmation(expectedCount: 3...7) { confirm in
    for await event in stream { confirm() }
}
```

Partial ranges without lower bound (e.g. `...10`) are explicitly disallowed to avoid ambiguity.

## `#expect(throws:)` Returns the Error (Swift 6.1+)

Both `#expect(throws:)` and `#require(throws:)` now return the caught error for further inspection. The old trailing-closure form with a second `throws:` closure is deprecated.

```swift
// OLD (deprecated) — trailing closure for validation
#expect {
    try game.play(at: 22)
} throws: { error in
    guard let e = error as? GameError else { return false }
    return e == .disallowedTime
}

// NEW — capture and inspect separately
let error = #expect(throws: GameError.self) {
    try game.play(at: 22)
}
#expect(error == .disallowedTime)
```

## Test Scoping Traits (Swift 6.1+)

Provide concurrency-safe shared configuration per test via `TestScoping` protocol. Combine with `@TaskLocal` to avoid shared mutable state.

Define the trait:

```swift
struct DefaultPlayerTrait: TestTrait, TestScoping {
    func provideScope(
        for test: Test,
        testCase: Test.Case?,
        performing function: () async throws -> Void
    ) async throws {
        let player = Player(name: "TestPlayer")
        try await Player.$current.withValue(player) {
            try await function()
        }
    }
}

extension Trait where Self == DefaultPlayerTrait {
    static var defaultPlayer: Self { Self() }
}
```

Apply to tests:

```swift
@Test(.defaultPlayer)
func whenGreeting_thenUsesPlayerName() {
    #expect(Player.current.name == "TestPlayer")
}
```

Multiple scopes can be combined: `@Test(.firstScope, .secondScope)`. Scopes apply in listed order.

## Exit Tests (Swift 6.2+)

Test code that terminates via `precondition()` or `fatalError()` — impossible in XCTest. Runs in a separate process.

```swift
@Test func whenRollingZeroSidedDice_thenCrashes() async {
    await #expect(processExitsWith: .failure) {
        let dice = Dice()
        _ = dice.roll(sides: 0)  // hits precondition
    }
}
```

Must be called with `await` — the test suspends while the child process runs.

## Attachments (Swift 6.2+)

Attach debug data to failing tests. Types conforming to `Attachable` (or `Encodable` + Foundation import) can be recorded.

```swift
@Test func whenGenerating_thenOutputIsValid() {
    let result = generateReport()
    #expect(result.isValid)
    Attachment.record(result, named: "GeneratedReport")
}
```

Supports `String`, `Data`, and any `Encodable` out of the box. Unlike XCTest attachments, no lifetime controls are available.

## Raw Identifiers for Test Names (Swift 6.2+)

Write test function names as natural strings using backtick syntax, removing the need for a separate display name:

```swift
// BEFORE — display name duplicates intent
@Test("Strip HTML tags from string")
func stripHTMLTagsFromString() { /* ... */ }

// AFTER — single source of truth
@Test
func `Strip HTML tags from string`() { /* ... */ }
```

Can be combined with parameterized tests. Suggest but do not adopt by surprise — some teams find this style unfamiliar.

## `ConditionTrait.evaluate()` (Swift 6.2+)

Evaluate condition traits programmatically outside of tests:

```swift
let trait = ConditionTrait.disabled(if: TestConfig.isSmokeTest)
if try await trait.evaluate() {
    // condition is active
}
```

Useful for building test infrastructure that needs to check the same conditions as test attributes.
