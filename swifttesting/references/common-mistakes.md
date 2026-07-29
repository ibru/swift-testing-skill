# Common Mistakes

Targeted pitfalls that AI agents and developers frequently hit when generating Swift Testing code.

## Don't Negate with `!` in `#expect`

`#expect(!isLoggedIn)` defeats Swift Testing's macro expansion — failure output shows only `false` with no sub-expression values.

```swift
// BAD — unhelpful failure diagnostics
#expect(!sut.isLoggedIn)

// GOOD — shows both sides on failure
#expect(sut.isLoggedIn == false)
```

## Don't Add `@Suite` Unnecessarily

Any type containing `@Test` methods is automatically treated as a test suite. Only add `@Suite` when attaching traits or a display name.

```swift
// UNNECESSARY — @Suite adds nothing here
@Suite struct NetworkTests {
    @Test func whenOffline_thenReturnsError() { /* ... */ }
}

// CORRECT — no @Suite needed
struct NetworkTests {
    @Test func whenOffline_thenReturnsError() { /* ... */ }
}

// CORRECT — @Suite needed to attach traits
@Suite(.timeLimit(.minutes(1)))
struct NetworkTests {
    @Test func whenOffline_thenReturnsError() { /* ... */ }
}
```

## Prefer Struct Over Class for Suites

Structs provide value semantics — each test gets a fresh, isolated instance. Use `class` or `actor` only when you need `deinit` for teardown.

```swift
// PREFERRED — value semantics, no accidental state sharing
struct FenceViewModelTests {
    @Test func whenSelecting_thenUpdatesState() { /* ... */ }
}

// USE ONLY WHEN deinit teardown is needed
@Suite final class DatabaseTests {
    let tempDir: URL
    init() throws { tempDir = /* create temp dir */ }
    deinit { try? FileManager.default.removeItem(at: tempDir) }
}
```

Each `@Test` gets its own suite instance — property values from one test never leak to another.

## Don't Overuse `#require`

`#require` aborts the test on failure. If used for every check, you only see the first failure per run — other broken things stay hidden.

```swift
// BAD — if name check fails, age and email are never validated
try #require(user.name == "Alice")
try #require(user.age == 30)
try #require(user.email == "alice@example.com")

// GOOD — #require for precondition, #expect for assertions
let user = try #require(await fetchUser(id: "123"))  // precondition: must exist
#expect(user.name == "Alice")     // continues on failure
#expect(user.age == 30)           // continues on failure
#expect(user.email == "alice@example.com")
```

Reserve `#require` for true preconditions where subsequent lines would be meaningless.

## Don't Multiply `makeSUT` Overloads

Keep one `makeSUT` per suite. Express scenario variations as defaulted parameters, not separate factory methods — overloads duplicate wiring, drift apart, and obscure which scenario a test arranges.

```swift
// BAD — one factory per scenario, dependency wiring repeated in each
func makeSUT() -> (CheckoutFlow, PaymentGatewaySpy) { /* ... */ }
func makeSUTWithFailingPayment() -> (CheckoutFlow, PaymentGatewaySpy) { /* ... */ }
func makeSUT(freeShippingThreshold: Decimal) -> (CheckoutFlow, PaymentGatewaySpy) { /* ... */ }

// GOOD — single factory, happy-path defaults; tests pass only what they arrange
func makeSUT(
    freeShippingThreshold: Decimal = 50,
    paymentResult: Result<Receipt, PaymentError> = .success(makeReceipt())
) -> (sut: CheckoutFlow, paymentsSpy: PaymentGatewaySpy) {
    let paymentsSpy = PaymentGatewaySpy(result: paymentResult)
    let pricing = OrderPricing(freeShippingThreshold: freeShippingThreshold)
    return (CheckoutFlow(pricing: pricing, payments: paymentsSpy), paymentsSpy)
}

let (sut, paymentsSpy) = makeSUT(paymentResult: .failure(.declined))
```

## `confirmation()` Doesn't Wait for Completion Handlers

Tested code must finish executing before the `confirmation()` closure returns. If you fire-and-forget an async operation, `confirmation` won't know to wait.

```swift
// BAD — test passes immediately, callback never fires before confirmation ends
@Test func whenFetching_thenCallsDelegate() async {
    await confirmation { confirm in
        api.fetch { result in
            #expect(result.isSuccess)
            confirm()  // may never execute in time
        }
    }
}

// GOOD — await the work so confirmation closure doesn't exit early
@Test func whenFetching_thenCallsDelegate() async {
    await confirmation { confirm in
        let result = await withCheckedContinuation { continuation in
            api.fetch { result in
                continuation.resume(returning: result)
            }
        }
        #expect(result.isSuccess)
        confirm()
    }
}
```

If the production code spawns a `Task` internally, return it so the test can `await task.value` before confirming.
