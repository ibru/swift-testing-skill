# Parameterized Tests

## Basic `@Test(arguments:)`

Replace repetitive tests with a single parameterized test. Each argument runs as an independent test case — reports separately, can be re-run individually, and executes in parallel.

```swift
@Test(arguments: [Network.home, .office, .cafe])
func whenConnecting_thenReportsExpectedSSID(_ network: Network) {
    let sut = makeSUT()
    #expect(sut.ssid(for: network) == network.expectedSSID)
}
```

Do not use in-test `for` loops as a substitute — they stop on first failure and provide no per-argument diagnostics.

## Pairing Inputs with Expected Outputs

### Prefer array-of-tuples (recommended)

Co-located pairs are impossible to misalign. Adding a new case forces a matching expected value.

```swift
@Test(arguments: [
    (Network.home, "HomeWiFi"),
    (.office, "CorpNet"),
    (.cafe, "CafeOpen")
])
func whenConnecting_thenReportsExpectedSSID(_ network: Network, expected: String) {
    #expect(makeSUT().ssid(for: network) == expected)
}
```

### `zip` for paired collections

Use `zip` when inputs must remain as separate collections. Pitfall: `zip` silently truncates if arrays differ in length — extra elements are dropped without warning.

```swift
@Test(arguments: zip(
    [Network.home, .office, .cafe],
    ["HomeWiFi", "CorpNet", "CafeOpen"]
))
func whenConnecting_thenReportsExpectedSSID(_ network: Network, expected: String) {
    #expect(makeSUT().ssid(for: network) == expected)
}
```

Prefer array-of-tuples over `zip` unless there is a specific reason to keep inputs separate.

## `CaseIterable.allCases` Guidance

Valid for **property-based tests** that verify a universal invariant holds for every member:

```swift
@Test(arguments: Direction.allCases)
func whenRotatingFourTimes_thenReturnsToOriginal(_ direction: Direction) {
    #expect(direction.rotated().rotated().rotated().rotated() == direction)
}
```

Avoid `allCases` when you need concrete, case-specific expected values — use explicit arrays or tuples instead.

## Anti-Pattern: Derived Expected Values

When the expected value is derived from the same expression as the system under test, both sides shift together and bugs pass silently:

```swift
// BAD — if format(day) has a casing bug, day.rawValue has the same bug
@Test(arguments: Day.allCases)
func dayLabel(day: Day) {
    #expect(format(day) == day.rawValue)
}

// GOOD — concrete literals are independent data points
@Test(arguments: [
    (Day.monday, "Monday"),
    (.friday, "Friday")
])
func whenFormattingDay_thenReturnsExpectedLabel(_ day: Day, expected: String) {
    #expect(format(day) == expected)
}
```

Always use concrete literal expected values in parameterized tests.
