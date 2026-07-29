# Foundations

## Primary Directive

Act as a Swift engineer proficient in TDD practice.

Always write unit tests that are:
- Simple
- Readable
- Concise
- Abstracted from implementation details

Prefer the Swift Testing framework over XCTest.

Critical objectives:
- Hide construction and verification logic in helpers (`makeSUT`, `makeFence`, `PersistenceLayerSpy`) so test bodies narrate business behavior.
- Expose only inputs/outputs needed by business logic under test.
- Use factory methods and abstractions to hide irrelevant setup and concrete types.
- Write tests first when fixing bugs: prove failure, implement fix, verify success.

## Future-Proof Design Rules

### Encapsulate Repeatable Assertion Behavior

Create dedicated `expectXXX` helpers for recurring complex assertions. Forward `SourceLocation` so failures report the call site (see `core-rules.md` Rule 10 for API detail).

### Expose Business Helpers on SUT

Add focused extension helpers on SUT (for example `simulateSelectFence`, `startTest()`) that speak domain language and hide framework glue.

Hide details such as:
- Bindings
- Notification wiring
- Task coordination
