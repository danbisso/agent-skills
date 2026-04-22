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

## 2. DRY & single source of truth

DRY is about **single source of knowledge**, not "no two lines look alike". Two snippets that change for *different reasons* aren't duplication — leave them.

### Derive types from one schema — never re-declare a shape
```ts
// before — three copies drift apart
const userSchema = z.object({ id: z.string(), email: z.string() });
interface User { id: string; email: string }      // duplicate
type UserDTO = { id: string; email: string };      // duplicate

// after — schema (or DB table) is the single source; types are inferred
const userSchema = z.object({ id: z.string(), email: z.string() });
type User = z.infer<typeof userSchema>;
// Drizzle: type User = typeof users.$inferSelect;  type NewUser = typeof users.$inferInsert;
```

### Centralize constants & config
No magic literals scattered across files; one canonical definition, imported.
```ts
// constants.ts
export const PAGE_SIZE = 50;
export const RETRYABLE_STATUS = new Set([429, 502, 503, 504]);
```

### The right abstraction vs premature abstraction
- **Rule of three:** extract on the third occurrence, not the first.
- **Wrong-abstraction tell:** a shared function bristling with boolean flags / `if (mode === …)` branches that exist only to serve different callers. Inline it and let the call sites differ.
- One **canonical domain model**; map to DTOs/view models at the boundary rather than letting four near-identical `User`-ish types breed.

---

## 3. Modularization & architecture

- **Cohesion high, coupling low.** A module does one thing; things that change together live together.
- **Feature folders over layer folders** for app code — colocate `feature/x/{api,model,ui}` so a change touches one folder, not five. Keep cross-cutting layers (`db`, `http`) only for genuinely shared infrastructure.
- **Dependency direction points at abstractions.** Core logic depends on interfaces; concrete IO (Stripe, Postgres) depends inward and is injected. Cycles are a smell — break them by extracting the shared type.
```ts
// core defines the port; the adapter implements it. Core never imports the adapter.
interface Clock { now(): number }
const decide = (inv: Invoice, clock: Clock) => /* ... */;
```
- **Barrel files (`index.ts` re-exports):** convenient, but they hurt — they defeat tree-shaking, create import cycles, and slow type-checking. Use sparingly at true public package boundaries; never as an internal "dump everything" file.
- **Small, single-purpose functions.** If you can't name it without "and", split it. But a 30-line linear function with no branching is fine — don't shred it into callback soup.

---

## 4. Code-smell catalog (quick reference)

Each smell → its refactor. Full examples in [code-smells.md](code-smells.md).

| Smell | Refactor |
|---|---|
| Long function | Extract named sub-functions; separate the *what* from the *how* |
| Long parameter list | Pass one options object; or introduce a parameter type |
| Boolean / flag arg | Split into two intent-named functions (`renderEditable` vs `renderReadonly`) |
| Primitive obsession | Wrap in a branded type / small value type (`type Email = string & {__brand}`) |
| Deep nesting | Guard clauses + early return; invert conditions |
| Feature envy | Move the method to the data it keeps reaching into |
| Shotgun surgery | Consolidate the scattered knowledge into one module |
| Data clumps | Group fields that always travel together into a type |
| Dead code | Delete it — git remembers; don't comment it out |
| Speculative generality | Remove unused params/hooks/abstractions added "for later" |
| Leaky abstraction | Hide the implementation detail behind a stable interface |

### Guard clauses (the most common quick win)
```ts
// before — arrow of doom
function price(o: Order) {
  if (o) {
    if (o.items.length) {
      if (o.coupon) { /* ... */ }
    }
  }
}
// after — handle exits first, happy path unindented
function price(o: Order) {
  if (!o) return 0;
  if (!o.items.length) return 0;
  const base = subtotal(o.items);
  return o.coupon ? applyCoupon(base, o.coupon) : base;
}
```

---

## 5. TypeScript-specific cleanliness

- **Eliminate `any`.** It disables checking and silently spreads. Use `unknown` at untrusted boundaries (parsed JSON, `catch`) and narrow before use.
```ts
// before
function parse(json: any) { return json.user.name; }
// after
function parse(json: unknown): string {
  const r = userSchema.safeParse(json);
  if (!r.success) throw new Error('bad payload');
  return r.data.email;
}
try { /* ... */ } catch (e: unknown) { if (e instanceof Error) log(e.message); }
```
- **Lean on inference** for locals/returns; annotate only public APIs and where inference is wrong or too wide. Don't restate what TS already knows.
- **Utility types** instead of hand-maintained variants: `Pick`, `Omit`, `Partial`, `Required`, `Parameters<typeof f>`, `ReturnType<typeof f>`, `Awaited<>`.
- **Unions over enums** in most app code: `type Status = 'open' | 'closed'` is a plain string at runtime, tree-shakes, and pattern-matches in a `switch`. Reach for `enum` only when you need a reverse map or a stable numeric wire value; never mix the two styles in one codebase.
- Prefer `type` for unions/intersections/derived shapes; `interface` for objects you expect others to `extends` or declaration-merge.

---

## 6. Disciplined cleanup workflow (messy existing code)

1. **Characterize.** Read the module; note its public surface and current behavior. Don't refactor what you don't understand.
2. **Get a safety net.** Run the existing tests. If coverage is thin, write characterization tests that pin *current* behavior (warts included) before touching anything — see the `tdd` skill.
3. **Format/lint as its own commit** so the real diff isn't drowned in whitespace.
4. **Refactor in micro-steps**, test green after each: rename → extract function → introduce type → invert condition → de-duplicate. Commit per step.
5. **Keep behavior identical.** Spotting a bug mid-refactor? Note it; fix it in a *separate* commit with its own test so the change is reviewable.
6. **Stop when it's clear enough.** Clean is a means, not an end — don't gold-plate a module nobody touches.

Then apply the result in context: the `frontend-dev` and `aws-cdk` skills cover the surrounding stacks, and `performance` covers when to deliberately break a clean-code rule on a measured hot path.
