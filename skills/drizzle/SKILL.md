---
name: drizzle
description: Own the Drizzle ORM data layer on Cloudflare D1 (SQLite) — schema with sqliteTable/columns/indexes, the relations + relational query API, the select/insert/update/delete query builder and operators, type inference ($inferSelect/$inferInsert) as the single source of truth, drizzle-kit migrations, the D1 db.batch transaction caveat, and repository-style data functions. Use when defining or changing a schema, writing or debugging Drizzle queries/joins/aggregates, deriving app types from tables, generating or applying migrations, or designing testable data-access functions on the D1 + Drizzle stack (Postgres/MySQL differences noted inline).
---

# Drizzle: the data layer

You write the **data layer** with Drizzle ORM. The library's database is **Cloudflare D1 (SQLite)**, so default to `drizzle-orm/sqlite-core` and `drizzle-orm/d1`. Note Postgres/MySQL differences inline, but write D1-correct code unless told otherwise.

**Scope boundaries — do not duplicate sibling skills:**
- The `db` instance is built **per request** from the D1 binding. The Workers runtime, bindings, and `wrangler.toml` belong to **`cloudflare-workers`** — link there, don't re-explain. Assume you receive `env.DB` and call `drizzle(env.DB, { schema })`.
- Drizzle is **called from inside** React Router loaders/actions — routing, loaders, and actions are **`frontend-dev`**. Don't cover them; just expose data functions they call.
- Type-derivation discipline ties to **`clean-code`** (single source of truth). Query/index perf depth defers to **`performance`**. Testing data functions defers to **`tdd`** (run against a real local D1; never mock Drizzle).

## Golden rules
1. **The schema is the single source of truth for types.** Derive every app type with `$inferSelect`/`$inferInsert`. Never hand-restate a row shape — see `clean-code`.
2. **D1 has no interactive transaction.** Use `db.batch([...])` for atomic multi-statement writes. `db.transaction()` does not work on D1. This is the most common D1 mistake — see below.
3. **Business logic is plain functions of `(db, input)`.** No HTTP, no `env` plumbing inside them — that makes them unit-testable against a real local D1.
4. **Select only the columns you need.** Use partial selects / the relational `columns` option; avoid `SELECT *` on wide tables and avoid N+1.

---

## Schema definition

`drizzle-orm/sqlite-core`. One file (e.g. `schema.ts`) is the source of truth.

```ts
import { sql } from "drizzle-orm";
import {
  sqliteTable, text, integer, real,
  index, uniqueIndex, primaryKey,
} from "drizzle-orm/sqlite-core";

export const users = sqliteTable("users", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  email: text("email").notNull().unique(),
  name: text("name").notNull(),
  role: text("role", { enum: ["admin", "member"] }).notNull().default("member"),
  // SQLite has no UUID type — generate in app with $defaultFn:
  publicId: text("public_id").notNull().$defaultFn(() => crypto.randomUUID()),
  // Timestamps: store epoch seconds, read/write as Date with mode:"timestamp".
  createdAt: integer("created_at", { mode: "timestamp" })
    .notNull()
    .default(sql`(unixepoch())`),      // DB-side default
  // Booleans: SQLite has no bool — integer mode:"boolean" maps 0/1 <-> false/true.
  active: integer("active", { mode: "boolean" }).notNull().default(true),
}, (t) => ({
  emailIdx: uniqueIndex("users_email_idx").on(t.email),
  roleIdx: index("users_role_idx").on(t.role),
}));

export const posts = sqliteTable("posts", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  authorId: integer("author_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  body: text("body").notNull(),
  score: real("score").notNull().default(0),
  createdAt: integer("created_at", { mode: "timestamp" }).notNull().default(sql`(unixepoch())`),
}, (t) => ({
  authorIdx: index("posts_author_idx").on(t.authorId),
}));

// Composite primary key (join table, no surrogate id):
export const postTags = sqliteTable("post_tags", {
  postId: integer("post_id").notNull().references(() => posts.id, { onDelete: "cascade" }),
  tagId:  integer("tag_id").notNull().references(() => tags.id, { onDelete: "cascade" }),
}, (t) => ({
  pk: primaryKey({ columns: [t.postId, t.tagId] }),
}));

export const tags = sqliteTable("tags", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  name: text("name").notNull().unique(),
});
```

Column gotchas:
- `$defaultFn(() => ...)` runs in **JS at insert time** (UUIDs, computed defaults). `.default(sql\`...\`)` is a **DB-side** default. Use `.default(value)` for literals.
- `text({ mode: "json" })` stores JSON; type it with `.$type<MyShape>()` for a typed column.
- **Foreign keys are off by default in SQLite.** `.references()` generates the constraint, but D1 needs `PRAGMA foreign_keys = ON` to enforce it. D1 enables this for migrations; `onDelete: "cascade"` only fires when FKs are on.
- **Postgres/MySQL differ:** import from `drizzle-orm/pg-core` / `mysql-core`; use `serial`/`uuid`/`timestamp`/`boolean`/`jsonb` native types instead of integer-mode emulation.

## Relations & inferred types

Relations are a **query-time convenience for the relational API** — they do not create DB constraints (`.references()` does that). Declare both.

```ts
import { relations } from "drizzle-orm";

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));
export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
  tags: many(postTags),
}));

// Types derived from the schema — the ONLY place row shapes are defined.
export type User = typeof users.$inferSelect;       // what a SELECT returns
export type NewUser = typeof users.$inferInsert;    // what an INSERT needs (optionals optional)
export type Post = typeof posts.$inferSelect;
```

Build app types **from** these — never restate fields. Example: `type PostSummary = Pick<Post, "id" | "title">;`.

## Wiring `db` (per request)

```ts
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

// env.DB is the D1 binding provided by cloudflare-workers.
export const makeDb = (d1: D1Database) => drizzle(d1, { schema });
export type DB = ReturnType<typeof makeDb>;
```

Pass `{ schema }` so `db.query.*` (relational API) is available.

## Relational query API — `db.query`

Best for reads with nested relations (no manual joins, returns shaped objects).

```ts
const user = await db.query.users.findFirst({
  where: (u, { eq }) => eq(u.id, userId),
  columns: { id: true, name: true },          // partial select
  with: {
    posts: {
      columns: { id: true, title: true, createdAt: true },
      where: (p, { gt }) => gt(p.score, 0),
      orderBy: (p, { desc }) => [desc(p.createdAt)],
      limit: 10,
      with: { tags: true },                    // nested
    },
  },
});
// `user` is fully typed from the schema, including nested `posts[].tags`.

const recent = await db.query.posts.findMany({
  where: (p, { eq }) => eq(p.authorId, userId),
  orderBy: (p, { desc }) => desc(p.createdAt),
  limit: 20,
});
```

## Query builder — `select`/`insert`/`update`/`delete`

Best for joins you control, aggregates, and writes.

```ts
import { eq, and, or, inArray, like, gt, desc, count, sum } from "drizzle-orm";

// Partial select + join
const rows = await db
  .select({ postId: posts.id, title: posts.title, author: users.name })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
  .where(and(eq(users.active, true), gt(posts.score, 0)))
  .orderBy(desc(posts.createdAt))
  .limit(20)
  .offset(0);

// Aggregate + groupBy
const perAuthor = await db
  .select({ authorId: posts.authorId, n: count(), total: sum(posts.score) })
  .from(posts)
  .groupBy(posts.authorId);

// Insert with returning (SQLite/D1 + Postgres support .returning(); MySQL does not)
const [created] = await db
  .insert(users)
  .values({ email, name })            // typed as NewUser; id/createdAt optional
  .returning();

// Update / delete
await db.update(posts).set({ score: 1 }).where(eq(posts.id, id));
await db.delete(posts).where(inArray(posts.id, ids));
```

Common operators: `eq`, `ne`, `and`, `or`, `not`, `inArray`/`notInArray`, `like`/`ilike` (PG), `gt`/`gte`/`lt`/`lte`, `between`, `isNull`/`isNotNull`. See `query-cookbook.md` for upsert, conditional `where`, subqueries, and `$count`.

