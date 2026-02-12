# Drizzle query cookbook (D1 / SQLite)

Reach-for snippets. Default dialect is SQLite/D1; differences noted. Assume `db: DB` and schema imports as in `SKILL.md`.

## Upsert (insert-or-update)

```ts
import { sql } from "drizzle-orm";

// On conflict, update. SQLite/D1 + Postgres syntax:
await db
  .insert(users)
  .values({ email, name })
  .onConflictDoUpdate({
    target: users.email,                  // the unique column(s)
    set: { name },                        // columns to overwrite
  });

// Reference the rejected row's value with `excluded`:
await db.insert(tags).values({ name }).onConflictDoUpdate({
  target: tags.name,
  set: { name: sql`excluded.name` },
});

// Ignore conflicts entirely:
await db.insert(postTags).values({ postId, tagId }).onConflictDoNothing();
// MySQL differs: use .onDuplicateKeyUpdate({ set: {...} }).
```

## Conditional / dynamic filters

Build a `where` from optional inputs. `and(...)` skips `undefined` conditions.

```ts
import { and, eq, gt, like } from "drizzle-orm";

function searchPosts(db: DB, f: { authorId?: number; minScore?: number; q?: string }) {
  return db
    .select()
    .from(posts)
    .where(
      and(
        f.authorId !== undefined ? eq(posts.authorId, f.authorId) : undefined,
        f.minScore !== undefined ? gt(posts.score, f.minScore) : undefined,
        f.q ? like(posts.title, `%${f.q}%`) : undefined,
      ),
    );
}
```

For incremental composition use `.$dynamic()`:

```ts
let qb = db.select().from(posts).$dynamic();
if (authorId) qb = qb.where(eq(posts.authorId, authorId));
const rows = await qb.limit(20);
```

## Counting

```ts
import { count, eq } from "drizzle-orm";

// Aggregate select
const [{ n }] = await db.select({ n: count() }).from(posts).where(eq(posts.authorId, id));

// Shorthand
const total = await db.$count(posts, eq(posts.authorId, id));
```

## Pagination

```ts
// Offset/limit (simple, slower deep in the table)
const page = await db.select().from(posts)
  .orderBy(desc(posts.createdAt)).limit(20).offset(page0 * 20);

// Keyset / cursor (scales; needs an indexed, ordered column)
import { lt, desc } from "drizzle-orm";
const next = await db.select().from(posts)
  .where(lt(posts.createdAt, cursor))
  .orderBy(desc(posts.createdAt)).limit(20);
```

## Subqueries

```ts
import { eq, inArray } from "drizzle-orm";

const activeAuthorIds = db.select({ id: users.id }).from(users).where(eq(users.active, true));
const fromActive = await db.select().from(posts).where(inArray(posts.authorId, activeAuthorIds));

// Subquery as a derived table
const sq = db.select({ authorId: posts.authorId, n: count() })
  .from(posts).groupBy(posts.authorId).as("sq");
const joined = await db.select({ name: users.name, n: sq.n })
  .from(users).innerJoin(sq, eq(users.id, sq.authorId));
```

## Atomic multi-write with `db.batch` (D1)

D1 has no interactive `db.transaction()`. Group writes that must all-or-nothing into one batch.

```ts
// Move a post to a new author and re-tag it atomically.
const results = await db.batch([
  db.update(posts).set({ authorId: newAuthorId }).where(eq(posts.id, postId)),
  db.delete(postTags).where(eq(postTags.postId, postId)),
  db.insert(postTags).values([
    { postId, tagId: 1 },
    { postId, tagId: 2 },
  ]),
]);
// results is a tuple aligned to the statements above.
```

Rules:
- Statements run in order, in one round-trip, as one implicit transaction.
- They **cannot** read each other's output mid-batch — resolve dependent ids before the batch (run a prior insert, then batch the dependents).
- The array needs at least one statement.

## Prepared statements (hot paths)

```ts
import { sql } from "drizzle-orm";

const postsByAuthor = db
  .select()
  .from(posts)
  .where(eq(posts.authorId, sql.placeholder("authorId")))
  .orderBy(desc(posts.createdAt))
  .prepare();

const mine = await postsByAuthor.execute({ authorId: 42 });
```

## Raw SQL escape hatch

When the builder can't express it. Keep parameterized; never interpolate user input as a string.

```ts
import { sql } from "drizzle-orm";

const rows = await db.all<{ id: number; c: number }>(
  sql`select author_id as id, count(*) as c from posts group by author_id having c > ${threshold}`,
);
// db.run(...) for statements without rows; db.get(...) for a single row.
```
