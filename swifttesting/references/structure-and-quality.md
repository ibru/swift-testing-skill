# Structure And Quality

## Test File Structure

Use a three-part layout:

1. Test suites first
- Organize behavior with nested `@Suite` blocks.

2. Factories in the middle
- Group `makeSUT` and factories under `// MARK: - makeSUT & Factories`.
- Keep them `private` free functions.

3. Test doubles last
- Group stubs/spies under `// MARK: - Test Doubles`.
- Keep implementation detail at the bottom.

## Implementation Patterns

### Nested Suites

```swift
@Suite("NetworkCoverageViewModel")
struct NetworkCoverageViewModelTests {
    @Suite("startMeasurement") struct StartMeasurement {
        @Test func whenNetworkReady_thenBeginsMeasurement() async throws { /* ... */ }
    }
}
```

### Business Helper Extension

```swift
@MainActor extension NetworkCoverageViewModel {
    func simulateSelectFence(id: UUID) {
        selectedFenceItem = fenceItems.first { $0.id == id }
    }
}
```

## Parallel Execution Is Default

Swift Testing runs all tests in parallel by default with randomized order. This is a fundamental difference from XCTest.

Every test must be fully independent:
- No shared mutable state between tests
- No reliance on execution order
- Each `@Test` gets its own fresh suite instance (struct value semantics)

Random order helps expose hidden dependencies — fix them rather than serializing.

## Mirror Production File Structure

Organize test files/folders to match the production code structure for discoverability:

```
Sources/
  Features/
    Fences/
      FenceViewModel.swift
Tests/
  Features/
    Fences/
      FenceViewModelTests.swift
```

## FIRST Principles

Swift Testing is designed to make these principles easy to follow:

- **Fast** — Lean on default parallelism. Avoid unnecessary sleeps.
- **Isolated** — Fresh suite instance per test. No state leaks.
- **Repeatable** — Control all inputs (dates, network, randomness) with deterministic doubles.
- **Self-Validating** — Use `#expect` and `#require`. Never rely on `print()` for validation.
- **Timely** — Write tests alongside production code. Use parameterized tests to cover edge cases early.

## Quality Gates

- Use behavior-driven naming (`when..._then...`) consistent with the suite.
- Create SUT through `makeSUT`.
- Keep business behavior obvious from test body.
- Hide concrete implementation detail through helpers/factories.
- Cover key variants and edge cases.
- Keep setup, fixtures, and assertions immediately understandable.

## Anti-Patterns To Avoid

- Tests named after implementation detail (`test_fetchData`)
- Multiple disjoint `#expect` calls for one business behavior
- Direct manipulation of framework internals instead of business helpers
- Reusing spies across tests without reset (cross-test leakage)
