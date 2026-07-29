---
name: swifttesting
description: Write and review Swift unit tests using the Swift Testing framework with a domain-first mindset. Very opinionated, suited to TDD and BDD workflows. Use when creating or refactoring tests, practicing TDD/BDD, enforcing WHEN_THEN naming, designing test doubles, validating async/error behavior, improving readability with helpers/factories, and guiding production code toward testable architecture that avoids implementation-coupled assertions.
---

# SwiftTesting

## Overview

Use this skill to write and review unit tests that validate business behavior, not implementation details. Prefer Swift Testing primitives and test architecture that keeps production code easy to verify through stable domain seams.

## Workflow

1. Identify business behavior to verify.
2. Define test names with `when..._then...` (and `And` when needed).
3. Build SUT with `makeSUT`, parameterized by test scenario arrangements (results, states, conditions).
4. Hide setup and verification in helpers/factories so tests narrate domain behavior.
5. Apply Swift Testing primitives (`@Suite`, `@Test`, `#expect`, `#require`, `confirmation`, `Issue.record`).
6. Cover success, failure, and edge behavior with deterministic doubles.
7. Refactor production seams only to improve testability.
8. Run quality gates before finishing.

## Agent Behavior Rules

- Swift Testing does **not** support UI tests, performance tests (`XCTMetric`), or Objective-C tests — keep those on XCTest.
- Only import `Testing` in test targets, never in app or library targets.
- Default to parallel-safe guidance. Fix shared state before reaching for `.serialized`.
- Prefer traits (`.disabled`, `.enabled(if:)`, `.bug`, `.timeLimit`) for test metadata over naming conventions or ad-hoc comments.
- Do not rewrite existing XCTest code to Swift Testing unless explicitly requested.

## Reference Map

- `references/foundations.md`: Primary directive and future-proof guidance for testable production design.
- `references/core-rules.md`: Core behavior-first test writing rules (naming, makeSUT, factories, assertions, error testing, source location).
- `references/async-and-errors.md`: Swift Testing async coordination, continuation bridging, and deterministic async.
- `references/test-doubles.md`: Stub/spy strategy, capture patterns, result injection, and deterministic simulation.
- `references/structure-and-quality.md`: Test file structure, parallel execution, FIRST principles, quality gates, and anti-patterns.
- `references/parameterized-tests.md`: `@Test(arguments:)`, zip, tuples, CaseIterable guidance, and parameterization anti-patterns.
- `references/traits-and-known-issues.md`: Conditional traits, `.bug()`, `.timeLimit`, `@available`, `withKnownIssue`, and trait inheritance.
- `references/common-mistakes.md`: LLM-specific pitfalls — negation in `#expect`, unnecessary `@Suite`, struct vs class, `#require` overuse.
- `references/new-features.md`: Swift 6.1/6.2+ features — range confirmations, exit tests, attachments, test scoping traits, raw identifiers.

## Quick Selection

- Start a new test file: `core-rules.md` + `structure-and-quality.md`
- Write async tests: `async-and-errors.md` + `test-doubles.md`
- Refactor brittle tests: `core-rules.md` + `structure-and-quality.md`
- Improve production code testability: `foundations.md` + `structure-and-quality.md`
- Use parameterized tests: `parameterized-tests.md`
- Add traits or handle known issues: `traits-and-known-issues.md`
- Review for common mistakes: `common-mistakes.md`
- Use Swift 6.1/6.2 features: `new-features.md`

## Output Requirements

- Keep test bodies short and domain-readable.
- Assert domain outcomes, not private/internal mechanics.
- Prefer helper/factory abstractions over inline boilerplate.
- Keep async tests deterministic without arbitrary sleeps.
- Ensure naming, fixtures, and failure messages communicate intent clearly.
