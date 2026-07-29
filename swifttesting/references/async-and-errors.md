# Async And Errors

## Confirmation for Async Verification

Use `confirmation` to verify async events with expected call counts:

```swift
await confirmation("connect sequence", expectedCount: 1) { confirm in
    sut.onConnected = { confirm() }
    await sut.connect()
}
```

## Minimize Timing Sleeps

- Prefer `confirmation` or deterministic signals over `Task.sleep`.
- Avoid explicit delays and avoid sequencing with `Task.yield()` unless unavoidable.
- Never rely on long waits to hide synchronization problems.

```swift
await confirmation("status flips", expectedCount: 2) { confirm in
    spy.onEvent = { confirm() }
    sut.simulateToggle()
}
#expect(spy.events == [.started, .finished])
```

## Bridge Legacy Completion Handlers

For APIs without async overloads, use `withCheckedThrowingContinuation` to wrap callback-based code:

```swift
@Test func whenFetchingLegacy_thenReturnsData() async throws {
    let result = try await withCheckedThrowingContinuation { continuation in
        legacyService.fetchData { result in
            continuation.resume(with: result)
        }
    }
    #expect(result.count > 0)
}
```

Use `withCheckedContinuation` for non-throwing callbacks. Keep continuation wrappers minimal and test-focused.

## Deterministic Async with `withMainSerialExecutor`

Use Point-Free's `swift-concurrency-extras` to serialize async work, making suspension points deterministic:

```swift
import ConcurrencyExtras

@Test func whenLoading_thenIsLoadingFlipsCorrectly() async {
    await withMainSerialExecutor {
        let sut = makeSUT()
        let task = Task { await sut.loadData() }
        await Task.yield()
        #expect(sut.isLoading == true)   // deterministic
        await task.value
        #expect(sut.isLoading == false)  // deterministic
    }
}
```

Prefer this over `Task.sleep` or `Task.yield` loops for testing intermediate state changes.

## Actor-Isolated Event Counting

For thread-safe event counting under strict concurrency, use an actor instead of a mutable var:

```swift
actor EventCounter {
    private(set) var count = 0
    func increment() { count += 1 }
}

@Test func whenProcessing_thenFiresTwice() async {
    let counter = EventCounter()
    await sut.process { await counter.increment() }
    #expect(await counter.count == 2)
}
```

## Error Assertion Patterns

For `#expect(throws:)` variants and `Issue.record()` do/catch patterns, see `core-rules.md` Rules 8–9.
