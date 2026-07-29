# SwiftTesting Skill

An [agent skill](https://skills.sh) that teaches AI coding agents (Claude Code, Cursor, Codex, and others) to write and review Swift unit tests using the [Swift Testing](https://developer.apple.com/documentation/testing) framework with a **domain-first mindset**.

This is a **very opinionated** skill, built for **TDD and BDD workflows**. It doesn't offer neutral testing advice — it enforces one coherent style: behavior-first tests with `when..._then...` naming, `makeSUT` factories, and deterministic test doubles. If that's how you want your test suite to read, this skill keeps your agent on those rails.

Instead of implementation-coupled assertions, the skill drives agents toward tests that:

- Verify **business behavior**, not private mechanics
- Use `when..._then...` naming that narrates domain intent
- Build SUTs with `makeSUT` factories and deterministic test doubles
- Apply Swift Testing primitives correctly (`@Suite`, `@Test`, `#expect`, `#require`, `confirmation`, traits, parameterized tests)
- Keep async tests deterministic — no arbitrary sleeps
- Nudge production code toward testable architecture through stable domain seams

## What it looks like

A taste of the style this skill enforces. Test names state the business rule, `makeSUT` arguments describe the *scenario* (not the wiring), and factories like `makeOrder` hide everything irrelevant to the behavior under test.

```swift
@Test func whenOrderReachesFreeShippingThreshold_thenShippingIsFree() {
    let sut = makeSUT(freeShippingThreshold: 50)

    #expect(sut.shippingCost(for: makeOrder(total: 50)) == .free)
}

@Test func whenPaymentSucceeds_thenChargesOrderTotalOnceAndDeliversReceipt() async throws {
    let (sut, paymentsSpy) = makeSUT(paymentResult: .success(makeReceipt(id: "R-42")))

    let receipt = try await sut.checkout(order: makeOrder(total: 250))

    #expect(receipt.id == "R-42")
    #expect(paymentsSpy.capturedChargeAmounts == [250])
}

@Test func whenCheckingOut_thenDeliversStatusUpdatesInOrder() async {
    let (sut, _) = makeSUT()
    var capturedStatuses: [CheckoutStatus] = []

    await confirmation("status updates", expectedCount: 2) { received in
        let consumer = Task {
            for await status in sut.statusUpdates {
                capturedStatuses.append(status)
                received()
            }
        }
        _ = try? await sut.checkout(order: makeOrder())
        consumer.cancel()
    }

    #expect(capturedStatuses == [.processingPayment, .completed])
}
```

## Installation

### Option A: skills CLI (recommended)

Works with Claude Code, Cursor, Codex, OpenCode, and 40+ other agents via [skills.sh](https://skills.sh):

```bash
npx skills add ibru/swift-testing-skill
```

Install globally so it's available in every project:

```bash
npx skills add ibru/swift-testing-skill -g
```

### Option B: Claude Code plugin

```
/plugin marketplace add ibru/swift-testing-skill
/plugin install swifttesting@swift-testing-skill
```

Or pin it for your whole team in your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "swift-testing-skill": {
      "source": {
        "source": "github",
        "repo": "ibru/swift-testing-skill"
      }
    }
  },
  "enabledPlugins": {
    "swifttesting@swift-testing-skill": true
  }
}
```

### Option C: Codex / OpenAI-compatible agents

Copy or symlink the `swifttesting/` folder into your agent's skills directory, e.g.:

```bash
git clone https://github.com/ibru/swift-testing-skill.git
ln -s "$(pwd)/swift-testing-skill/swifttesting" ~/.codex/skills/swifttesting
```

### Option D: pi package manager

```bash
pi install https://github.com/ibru/swift-testing-skill
```

### Option E: Manual (any agent)

Clone the repo and copy or symlink the `swifttesting/` folder into your tool's skills location — for Claude Code that's `~/.claude/skills/`:

```bash
git clone https://github.com/ibru/swift-testing-skill.git
cp -R swift-testing-skill/swifttesting ~/.claude/skills/swifttesting
```

## Updating

New releases are published to the `main` branch and tagged — see the [CHANGELOG](CHANGELOG.md) for what changed.

- **skills CLI:** `npx skills update`
- **Claude Code plugin:** `/plugin marketplace update swift-testing-skill`
- **Manual/git installs:** `git pull` and re-copy

## What's inside

```
swifttesting/
  SKILL.md                  Entry point: workflow, agent behavior rules, reference map
  references/
    foundations.md          Testable production design principles
    core-rules.md           Naming, makeSUT, factories, assertions, error testing
    async-and-errors.md     Async coordination, continuation bridging, deterministic async
    test-doubles.md         Stub/spy strategy, capture patterns, result injection
    structure-and-quality.md  File structure, parallel execution, FIRST, quality gates
    parameterized-tests.md  @Test(arguments:), zip, CaseIterable, anti-patterns
    traits-and-known-issues.md  Conditional traits, .bug(), .timeLimit, withKnownIssue
    common-mistakes.md      LLM-specific pitfalls to avoid
    new-features.md         Swift 6.1/6.2+ — exit tests, attachments, test scoping traits
```

The agent loads `SKILL.md` when the task involves Swift unit tests, then pulls in only the reference files relevant to the task at hand.

## Scope

- Swift Testing only — UI tests, performance tests (`XCTMetric`), and Objective-C tests stay on XCTest
- Does not rewrite existing XCTest code unless explicitly asked

## License

[MIT](LICENSE)
