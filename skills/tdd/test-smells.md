# Test smells — diagnose and fix

Quick reference. If a test exhibits a smell on the left, it is coupled to implementation or otherwise fragile. Apply the fix.

| Smell | What it looks like | Why it hurts | Fix |
|---|---|---|---|
| **Private-field assertion** | `expect((obj as any)._state).toBe(...)` | Breaks on any internal rename; tests structure not behavior | Assert on a public return value or state read back through the public API |
| **Call-count spying** | `expect(spy).toHaveBeenCalledTimes(2)` on an internal helper | Couples to the exact algorithm; behavior-preserving refactor goes red | Drop the spy; assert the outcome. Keep call-assertions only at true IO boundaries |
| **Over-mocking** | Every collaborator is a `vi.fn()` | You test that mocks were called as configured — a tautology, zero real coverage | Use real collaborators; mock only network/time/random/fs |
| **Mirror test** | Test body restates the production code step by step | No independent check; both can be wrong together | Rewrite from the *contract*: given input, expect observable output |
| **Snapshot-everything** | `toMatchSnapshot()` on large/unstable output | Nobody reads diffs; failures get re-blessed reflexively | Targeted assertions on the fields that matter; snapshot only stable serialized output |
| **Getter/setter test** | `expect(obj.name).toBe("x")` after `obj.name = "x"` | Tests the language, not your logic | Delete it; test code that has behavior |
| **Flaky / order-dependent** | Passes alone, fails in suite or on reorder | Shared mutable state or real clock/RNG/network | Fresh state per test (`beforeEach`/factory); inject clock & random |
| **Slow unit test** | Real HTTP, `setTimeout`, real db file in a "unit" test | Inner loop becomes minutes; people stop running tests | Inject boundaries; use in-memory db; move genuinely-integration cases to the integration band |
| **Multi-behavior test** | One `it` asserting several unrelated outcomes | A failure doesn't pinpoint what broke | Split into one test per behavior |
| **Method-named test** | `it("test process")` | Failure name tells you nothing | Name the behavior: `it("marks a valid order paid")` |
| **Never-saw-it-fail** | Test written after the code, green on first run | May assert nothing meaningful (e.g. wrong matcher) | Temporarily break the production code; confirm the test goes red for the right reason |
| **Conditional logic in test** | `if`/loops deciding what to assert | The test has bugs of its own | Flatten with `it.each`; one explicit case per row |

## The one question

Before keeping any assertion, ask: **"If I refactored the implementation without changing what a caller observes, would this line still pass?"** If no, it tests the *how* — rewrite it to test the *what*, or delete it.
