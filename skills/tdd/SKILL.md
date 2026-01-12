---
name: tdd
description: Drive code with behavior-first tests in a strict red-green-refactor loop — write the smallest failing test, make it pass, refactor under the test's protection. Use when adding a feature test-first, fixing a bug (reproduce with a failing test first), pinning legacy behavior before a change, deciding what to test, killing brittle/over-mocked tests, or testing this stack (Vitest, React Router 7 loaders/actions, Testing Library, D1/Drizzle, CDK assertions). Tests target observable behavior through public interfaces, not implementation details.
---

# Test-Driven Development

Your tests exist to let you change code without fear. A test that breaks when you refactor *without changing behavior* is not protecting you — it is taxing you. So the whole skill rests on one rule: **test behavior, not implementation.** Test the *what* (observable outputs through public interfaces), never the *how* (private methods, internal state, call sequences). Everything below serves that rule.

## The red-green-refactor loop, run strictly

One behavior at a time. Never skip a phase, never batch phases.

1. **RED.** Write the smallest test for the next slice of behavior. Run it. **Watch it fail, and confirm it fails for the right reason** — the assertion you care about, not an import error or a typo. A test you never saw fail is a test you don't trust.
2. **GREEN.** Write the *minimum* code to pass. Hardcoding a return value is allowed and often correct — it forces the next test to drive out the generality. Resist writing code no current test demands.
3. **REFACTOR.** Now that the bar is green, clean up production *and* test code: dedupe, rename, extract. The test is your safety net — run it after every edit. Add no new behavior here.

Then repeat. Commit on green. If you're stuck for more than a few minutes in GREEN, your last RED step was too big — revert and take a smaller bite.

### Worked example — `priceCart` cycling three times

**Cycle 1 — RED.** Start with the most degenerate case.

```ts
import { describe, it, expect } from "vitest";
import { priceCart } from "./pricing";

it("returns 0 for an empty cart", () => {
  expect(priceCart([])).toBe(0);
});
```

Run it: `priceCart is not defined`. Wrong reason — that's a compile failure, not a behavior failure. Add the stub so the test fails on the *assertion*:

```ts
export function priceCart(items: LineItem[]): number {
  return 0;
}
```

Now it's **GREEN** already, because the minimum code for "empty → 0" is literally `return 0`. Good — don't add more.

**Cycle 2 — RED.** Drive out summation.

```ts
it("sums unit price times quantity across lines", () => {
  const cart = [
    { unitPrice: 250, qty: 2 }, // cents
    { unitPrice: 100, qty: 1 },
  ];
  expect(priceCart(cart)).toBe(600);
});
```

Run it: fails, `expected 600, got 0` — right reason. **GREEN** with the minimum that generalizes:

```ts
export function priceCart(items: LineItem[]): number {
  return items.reduce((total, { unitPrice, qty }) => total + unitPrice * qty, 0);
}
```

Both tests pass. The empty-cart case still holds (`reduce` over `[]` returns the seed `0`) — that earlier test now earns its keep as a regression guard.

**Cycle 3 — RED.** A real rule: percentage discounts.

```ts
it("applies a percentage discount to the subtotal", () => {
  const cart = [{ unitPrice: 1000, qty: 1 }];
  expect(priceCart(cart, { percentOff: 10 })).toBe(900);
});
```

Fails: `priceCart` ignores the second arg. **GREEN**:

```ts
export function priceCart(items: LineItem[], opts: { percentOff?: number } = {}): number {
  const subtotal = items.reduce((t, { unitPrice, qty }) => t + unitPrice * qty, 0);
  const off = opts.percentOff ?? 0;
  return Math.round(subtotal * (1 - off / 100));
}
```

**REFACTOR.** Three green tests now cover empty / sum / discount, so it's safe to extract the subtotal for readability and add an edge test for rounding (`Math.round` on a `.5` cent) — drive that with its own RED first. Notice every test names the *behavior* and asserts the *return value*. None mentions `reduce`, `Math.round`, or the shape of `opts`. Rewrite the body to a `for` loop and they all stay green. That is the payoff.

## Behavior over implementation — the core doctrine

Test through the public interface. Assert on what a caller can observe: return values, thrown errors, rendered output, persisted state read back through the same public API. Do **not** reach into private fields, spy on internal helper calls, or assert "method X was called twice." Those couple the test to a specific implementation, so a behavior-preserving refactor turns the suite red — false alarms that train you to ignore failures.

### Before / after

**Implementation-coupled (brittle):**

```ts
it("processes the order", () => {
  const svc = new OrderService(repo);
  const validateSpy = vi.spyOn(svc as any, "validate");
  const calcSpy = vi.spyOn(svc as any, "calcTax");

  svc.process(order);

  expect(validateSpy).toHaveBeenCalledOnce();   // tests the HOW
  expect(calcSpy).toHaveBeenCalledWith(order);  // tests the HOW
  expect((svc as any).state).toBe("done");      // private field
});
```

This passes only as long as `process` is built from exactly `validate` + `calcTax` and tracks a `state` string. Inline `calcTax`, rename `state`, or merge the steps — all behavior-preserving — and it breaks. It also never checks the order was actually processed correctly.

**Behavioral (durable):**

```ts
it("marks a valid order paid and records the taxed total", () => {
  const svc = new OrderService(repo);

  const result = svc.process(validOrder);

  expect(result.status).toBe("paid");
  expect(result.total).toBe(1078); // 980 + 10% tax, the observable outcome
  expect(repo.find(result.id)?.status).toBe("paid"); // read back through the public API
});

it("rejects an order with no line items", () => {
  expect(() => svc.process({ ...validOrder, items: [] }))
    .toThrow(/at least one item/);
});
```

These assert *outcomes*. Refactor the internals however you like; as long as a valid order ends up paid with the right total and an empty one is rejected, they stay green. They also document the contract better than the spies did.

