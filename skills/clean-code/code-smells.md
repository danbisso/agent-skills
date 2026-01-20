# Code-smell catalog — full before→after refactors

Companion to SKILL.md §4. Each smell: what it looks like, why it hurts, the refactor.

## Long function
A function that scrolls. It mixes orchestration with detail, so you can't grasp it without reading every line.
```ts
// before
function importUsers(rows: string[][]) {
  const out: User[] = [];
  for (const row of rows) {
    if (row.length < 3) continue;
    const email = row[0].trim().toLowerCase();
    if (!email.includes('@')) continue;
    const name = row[1].trim();
    const role = row[2].trim() === 'a' ? 'admin' : 'viewer';
    out.push({ email, name, role });
  }
  return out;
}
// after — orchestration reads as a sentence; details named & testable
const importUsers = (rows: string[][]): User[] =>
  rows.map(parseRow).filter((u): u is User => u !== null);

function parseRow(row: string[]): User | null {
  if (row.length < 3) return null;
  const email = row[0].trim().toLowerCase();
  if (!email.includes('@')) return null;
  return { email, name: row[1].trim(), role: row[2].trim() === 'a' ? 'admin' : 'viewer' };
}
```

## Long parameter list
```ts
// before
function createUser(name: string, email: string, role: string, active: boolean, teamId: string) {}
// after — named, order-independent, extensible
type CreateUser = { name: string; email: string; role: Role; active: boolean; teamId: string };
function createUser(input: CreateUser) {}
```

## Boolean / flag argument
A boolean param means the function does two things. The call site `render(node, true)` is unreadable.
```ts
// before
function render(node: Node, editable: boolean) { /* if (editable) ... */ }
// after
function renderEditable(node: Node) {}
function renderReadonly(node: Node) {}
```

## Primitive obsession
Everything is `string`/`number`, so `getUser(orderId)` type-checks and ships a bug.
```ts
// before
function transfer(from: string, to: string, amount: number) {}
// after — branded types make mismatches a compile error
type UserId = string & { readonly __brand: 'UserId' };
type Cents = number & { readonly __brand: 'Cents' };
function transfer(from: UserId, to: UserId, amount: Cents) {}
```

## Feature envy
A method that reaches into another object's fields more than its own — it belongs over there.
```ts
// before — pricing logic envies Order's internals
class Cart { totalFor(o: Order) { return o.items.reduce((s, i) => s + i.qty * i.unitPrice, 0); } }
// after — Order computes its own total
class Order { total() { return this.items.reduce((s, i) => s + i.qty * i.unitPrice, 0); } }
```

## Shotgun surgery
One conceptual change forces edits in many files (add a currency → touch 9 modules). The knowledge is scattered. Consolidate it into one module that owns the concept, and have the others import it.

## Data clumps
The same group of fields travels together everywhere — they want to be a type.
```ts
// before
function book(startDate: Date, endDate: Date, tz: string) {}
function quote(startDate: Date, endDate: Date, tz: string) {}
// after
type DateRange = { start: Date; end: Date; tz: string };
function book(range: DateRange) {}
function quote(range: DateRange) {}
```

## Dead code
Unreachable branches, unused exports, commented-out blocks "in case we need them". Delete it — version control is the archive. Dead code lies about what matters and rots silently.

## Speculative generality
Abstractions, config hooks, and `options` params added for a future that never came. "We might need to swap the DB" → an interface with one implementation and three unused methods. Remove until a second real caller exists (rule of three).

## Leaky abstraction
The interface forces callers to know the implementation.
```ts
// before — caller must know it's an axios response
function getUser(id: string): Promise<AxiosResponse<User>> {}
// after — abstraction hides transport; swappable without touching callers
function getUser(id: string): Promise<User> {}
```

## Deep nesting
See SKILL.md §4 — guard clauses and early return flatten the happy path. Invert the condition you can exit on, return early, and let the main logic live at the lowest indentation.
