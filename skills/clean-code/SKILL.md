---
name: clean-code
description: Refactor and clean TypeScript/JavaScript toward DRY, modular, functional code — pure functions, immutability, map/filter/reduce over loops, single source of truth, derived types, smaller modules, and a code-smell catalog. Use when asked to clean up, refactor, modularize, de-duplicate, "make this functional", remove `any`, fix a messy file/module, or improve structure without changing behavior.
---

# Clean Code: TypeScript Architect & Cleaner

You are refactoring TypeScript toward code that is **functional, DRY, modular, and honestly named**. Refactoring means **changing structure without changing behavior**. If behavior changes, it is not a refactor — call it what it is and gate it behind a test.

Be opinionated but balanced. "Clean" advice over-applied makes code worse: premature abstraction, pipeline gymnastics, and a wall of tiny one-line modules are smells too. Optimize for the reader.

## Golden rules
1. **Behavior stays identical.** Lean on tests as a safety net (see the `tdd` skill). No tests covering the code? Add characterization tests first, then refactor.
2. **Small reversible steps.** One transformation per commit. Refactor commits stay separate from feature commits and from formatting-only churn.
3. **Earn every word.** Each identifier carries meaning; delete filler (`data`, `info`, `manager`, `Helper`, `utils2`). A name that needs a comment to explain it is the wrong name.
4. **Rule of three before abstracting.** Duplication is cheaper than the *wrong* abstraction (AHA: Avoid Hasty Abstractions). Inline a bad abstraction back out before reaching for a better one.
5. **Don't sacrifice clarity for cleverness or micro-perf.** Profile first; only the proven hot path earns an imperative loop or mutation (see the `performance` skill).

---

## 1. Functional TypeScript

### Prefer expressions over statements; iterate with `map`/`filter`/`reduce`
```ts
// before — mutable accumulator, hidden control flow
const names: string[] = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].active) names.push(users[i].name.toUpperCase());
}

// after — one expression, no mutation, intent reads top-to-bottom
const names = users
  .filter((u) => u.active)
  .map((u) => u.name.toUpperCase());
```

`reduce` for folding to a single value (sum, group, index):
```ts
const byId = users.reduce<Record<string, User>>((acc, u) => {
  acc[u.id] = u;
  return acc;
}, {});
// or, if available in your target: Object.fromEntries(users.map((u) => [u.id, u]))
```

Use `flatMap` for map-then-flatten, `some`/`every`/`find` for search predicates instead of a loop with a flag.

**When a loop IS clearer — keep it.** Don't force functional style for:
- early termination over a huge collection where `reduce` would still walk all of it,
- genuinely imperative IO sequencing,
- a `reduce` that needs a comment to explain it — that's a loop wearing a costume. Prefer a named helper or a plain `for...of`.

### Immutability
```ts
// before — mutates the caller's array
function addTag(post: Post, tag: string) {
  post.tags.push(tag);
  return post;
}

// after — returns a new value, input untouched
function addTag(post: Post, tag: string): Post {
  return { ...post, tags: [...post.tags, tag] };
}
```
Lock it at the type level so mutation is a compile error:
```ts
type Post = Readonly<{ id: string; tags: readonly string[] }>;
const ROLES = ['admin', 'editor', 'viewer'] as const; // readonly tuple + literal types
type Role = (typeof ROLES)[number]; // 'admin' | 'editor' | 'viewer'
```

### Functional core, imperative shell
Push IO (DB, network, `Date.now()`, randomness, logging) to the edges. Keep the decision logic pure so it's trivially testable.
```ts
// before — pure logic entangled with IO
async function chargeIfOverdue(invoiceId: string) {
  const inv = await db.invoices.get(invoiceId);
  if (inv.dueDate < Date.now() && !inv.paid) await stripe.charge(inv.amount, inv.customer);
}

// after — pure decision, thin IO shell
type ChargeDecision = { kind: 'charge'; amount: number; customer: string } | { kind: 'skip' };

function decideCharge(inv: Invoice, now: number): ChargeDecision {
  return inv.dueDate < now && !inv.paid
    ? { kind: 'charge', amount: inv.amount, customer: inv.customer }
    : { kind: 'skip' };
}

async function chargeIfOverdue(invoiceId: string) {
  const inv = await db.invoices.get(invoiceId);
  const d = decideCharge(inv, Date.now());
  if (d.kind === 'charge') await stripe.charge(d.amount, d.customer);
}
```

### Composition & higher-order functions
```ts
const pipe = <T>(x: T, ...fns: Array<(v: T) => T>): T => fns.reduce((v, f) => f(v), x);
const slug = (s: string) => pipe(s, (t) => t.trim(), (t) => t.toLowerCase(), (t) => t.replace(/\s+/g, '-'));
```
Use **currying / partial application only where it earns its keep** — when you genuinely build specialized functions from a general one (`const isAdmin = hasRole('admin')`). Don't curry everything; `f(a, b)` is clearer than `f(a)(b)` for one-off calls.

### Discriminated unions + exhaustive `switch`
Model "one of N shapes" as a tagged union; let the compiler force you to handle every case.
```ts
type Shape =
  | { kind: 'circle'; r: number }
  | { kind: 'rect'; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.r ** 2;
    case 'rect':   return s.w * s.h;
    default: return assertNever(s); // compile error if a case is added & unhandled
  }
}
const assertNever = (x: never): never => { throw new Error(`Unhandled: ${JSON.stringify(x)}`); };
```

### Result/Option over throwing for expected failures
Exceptions are for *unexpected* faults. Make expected failure a value the type system tracks.
```ts
type Result<T, E = string> = { ok: true; value: T } | { ok: false; error: E };

function parsePort(raw: string): Result<number> {
  const n = Number(raw);
  return Number.isInteger(n) && n > 0 && n < 65536
    ? { ok: true, value: n }
    : { ok: false, error: `invalid port: ${raw}` };
}
// caller can't ignore the failure path
const r = parsePort(env.PORT);
if (!r.ok) return exitWith(r.error);
startServer(r.value);
```
Aim for **total functions** (defined for every input of their type) — narrow the input type or return a `Result`/`Option` instead of throwing for in-band cases.

---

