# Traits and Known Issues

## Conditional Execution Traits

### `.disabled("reason")`

Skip a test with a visible reason in reports. Always include reason text — better than commenting out because the test still compiles.

```swift
@Test(.disabled("Backend endpoint not deployed yet"))
func whenSyncing_thenUploadsAllPending() async { /* ... */ }
```

### `.enabled(if:)` / `.disabled(if:)`

Runtime-evaluated conditions for feature flags or CI environments:

```swift
@Test(.enabled(if: FeatureFlags.newSyncEngine))
func whenSyncing_thenUsesNewEngine() async { /* ... */ }

@Test(.disabled(if: ProcessInfo.processInfo.environment["CI"] == "true"))
func whenDebugging_thenLogsVerbosely() { /* ... */ }
```

### `.bug()` — Link to Issue Tracker

Associate a test with a tracked issue. Provides context if the bug resurfaces.

```swift
@Test(.bug("https://github.com/org/repo/issues/42", "Flaky on CI"))
func whenReconnecting_thenRestoresSession() async { /* ... */ }

@Test(.bug(id: 182))
func whenParsingHTML_thenItalicizesHeadings() { /* ... */ }
```

## Time Limits

Use `.timeLimit(.minutes())` as a safety net for hung async tests.

**Important:** Only `.minutes()` is supported — `.seconds()` does not exist. This is a common LLM mistake.

```swift
@Test(.timeLimit(.minutes(1)))
func whenDownloading_thenCompletesWithinLimit() async throws { /* ... */ }
```

When both suite-level and test-level limits exist, the shorter one wins.

## Platform Availability

Use `@available` on individual test functions, **not** on suite types. Suite-level `@available` is not supported and causes compilation issues.

```swift
// CORRECT — on test function
@available(iOS 18, *)
@Test func whenUsingNewAPI_thenReturnsExpected() { /* ... */ }

// WRONG — do not apply to suite
// @available(iOS 18, *)
// @Suite struct NewAPITests { }
```

## `withKnownIssue` for Expected Failures

Better than `.disabled` because the test still compiles and runs. If the underlying issue is fixed, `withKnownIssue` fails — alerting you to remove it.

```swift
@Test func whenExporting_thenGeneratesValidPDF() {
    withKnownIssue("PDF renderer crashes on empty input — tracked in #88") {
        try sut.export(content: "")
    }
}
```

Use `isIntermittent: true` for flaky issues — the test passes if no issue is recorded, but marks an expected failure if one is:

```swift
@Test func whenReconnecting_thenRestoresSession() async {
    withKnownIssue("Flaky on CI due to network timing", isIntermittent: true) {
        try await sut.reconnect()
    }
}
```

Use conditional `when:` to scope known issues to specific environments:

```swift
withKnownIssue("Fails on iOS 17.0") {
    try sut.reproduceEdgeCaseBug()
} when: {
    ProcessInfo().operatingSystemVersion.majorVersion == 17
}
```

## Trait Inheritance

Traits applied to a `@Suite` cascade to all contained tests. Apply at suite level when broadly true:

```swift
@Suite(.timeLimit(.minutes(2)))
struct NetworkTests {
    @Test func whenFetching_thenReturnsData() async { /* ... */ }     // inherits 2-min limit
    @Test func whenUploading_thenConfirmsSuccess() async { /* ... */ } // inherits 2-min limit
}
```

Apply per-test when the trait is specific to individual cases only.
