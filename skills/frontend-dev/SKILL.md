---
name: frontend-dev
description: Build full-stack apps on React Router 7 (framework mode) + Cloudflare Workers + D1 + Drizzle ORM + Tailwind + TypeScript. Use when writing loaders/actions, routes, mutations, SSR, Workers handlers, wrangler config, D1/Drizzle schema & queries, or wiring env bindings into loaders/actions on this stack.
---

# frontend-dev

Engineer applications on this exact stack: **React Router 7 in framework mode**, running on **Cloudflare Workers** with a **D1** database accessed through **Drizzle ORM**, styled with **Tailwind**, in **TypeScript** end to end. This file is the source of truth for how the pieces fit. See `rr7-patterns.md` (same dir) for extended route recipes.

Cross-links: the **data layer** (Drizzle schema, queries, migrations, types) → `drizzle`; the **edge runtime & platform** (Workers model, bindings, wrangler, secrets, deploy) → `cloudflare-workers`; design tokens & Tailwind config → `frontend-design`; loading/empty/error states & flows → `ux`; structure & naming → `clean-code`; test seams → `tdd`; bundle/query/runtime cost → `performance`. Any AWS hosting/infra work hands off to `aws-architect` / `aws-cdk` — do NOT improvise AWS here.

This skill owns how React Router 7 **wires into** that stack — routing, loaders/actions, SSR, the `context` bridge to bindings, and the `.server` boundary. It deliberately does NOT restate Drizzle or Workers depth; reach for `drizzle` and `cloudflare-workers` for that.

## Mental model (read first)

- RR7 framework mode is the evolution of Remix. The runtime concepts (loader, action, `Form`, fetcher, nested routes, error boundaries) are identical, but **imports come from `react-router` and `@react-router/*`, never `@remix-run/*`**. If you reach for a Remix import, you are wrong.
- A request flows: Workers `fetch` → RR7 request handler → matched route's `loader` (GET) or `action` (POST/PUT/PATCH/DELETE) → component renders with `useLoaderData()`. The Cloudflare adapter injects `env` bindings (D1, secrets, vars) as the **`context`** argument to every loader/action. That is the ONLY supported channel for the database and secrets — do not read globals.
- Everything in a loader/action runs **server-side on the Worker**. The component renders on both server (SSR) and client. Drizzle/D1/secrets live only in loader/action. Never import the db module into component code.
- Workers runtime ≠ Node. No Node built-ins unless `nodejs_compat` is on (see wrangler). It is request-scoped: no module-level connection pools, no background timers that outlive the response, and CPU time per request is bounded — keep work small, push set-based work into SQL.

## Project layout

```
app/
  routes.ts              # route config (framework mode)
  root.tsx               # <html> shell, <Links/>, <Meta/>, <Scripts/>, ErrorBoundary
  routes/
    _index.tsx
    posts.$id.tsx
  db/
    schema.ts            # drizzle tables
    client.ts            # drizzle(env.DB) factory
  lib/
    session.server.ts    # cookie/session helpers (.server enforces server-only)
workers/app.ts           # Workers entry (re-exports RR7 handler)
wrangler.jsonc
drizzle.config.ts
```

Suffix files that must never reach the client with `.server.ts`. RR7 strips `*.server` modules from the client bundle and errors if you import one into client code — use it for db, session, and secret-touching helpers.

## Routing & types

Define routes explicitly in `app/routes.ts`:

```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";

export default [
  index("routes/_index.tsx"),
  route("posts/:id", "routes/posts.$id.tsx"),
  route("posts/:id/edit", "routes/posts.edit.tsx"),
] satisfies RouteConfig;
```

RR7 generates per-route types. Import them via the generated `./+types/<routeBasename>` and type your loader/action/component with `Route.*`. This is how params, loader data, and action data stay type-safe — **do not hand-write these types or use `any`.**

```ts
import type { Route } from "./+types/posts.$id";
// Route.LoaderArgs, Route.ActionArgs, Route.ComponentProps, Route.MetaArgs
```

Run `react-router typegen` (or rely on the dev server / the `typecheck` script `react-router typegen && tsc`) to refresh these.

## Typing env / bindings

Bindings are typed once and surfaced through `context.cloudflare.env`. With the Cloudflare Vite plugin, generate types from wrangler:

```bash
wrangler types          # writes worker-configuration.d.ts with interface Env
```

Then your loader context is typed automatically by `Route.LoaderArgs`. Access the D1 binding as `context.cloudflare.env.DB`. Keep a single `Env` shape — never redeclare bindings by hand and never widen to `any`.

## A complete route: loader + action + component (D1 via Drizzle through context)

```tsx
// app/routes/posts.$id.tsx
import { Form, redirect, useNavigation } from "react-router";
import { eq } from "drizzle-orm";
import type { Route } from "./+types/posts.$id";
import { getDb } from "~/db/client.server";
import { posts } from "~/db/schema";

export function meta({ data }: Route.MetaArgs) {
  return [{ title: data?.post ? data.post.title : "Post" }];
}

export async function loader({ params, context }: Route.LoaderArgs) {
  const db = getDb(context.cloudflare.env.DB);
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, Number(params.id)),
  });
  if (!post) throw new Response("Not Found", { status: 404 });
  return { post }; // plain object — RR7 serializes it; type flows to useLoaderData
}

export async function action({ request, params, context }: Route.ActionArgs) {
  const db = getDb(context.cloudflare.env.DB);
  const form = await request.formData();
  const intent = form.get("intent");

  if (intent === "delete") {
    await db.delete(posts).where(eq(posts.id, Number(params.id)));
    return redirect("/");
  }

  const title = String(form.get("title") ?? "").trim();
  if (!title) {
    // return validation errors WITHOUT throwing; component reads via actionData
    return { error: "Title is required" };
  }
  await db.update(posts).set({ title }).where(eq(posts.id, Number(params.id)));
  return { ok: true }; // loader auto-revalidates after a successful action
}

export default function PostRoute({ loaderData, actionData }: Route.ComponentProps) {
  const { post } = loaderData;
  const nav = useNavigation();
  const busy = nav.state !== "idle";

  return (
    <article className="mx-auto max-w-2xl space-y-4 p-6">
      <h1 className="text-2xl font-semibold">{post.title}</h1>

      <Form method="post" className="space-y-2">
        <input
          name="title"
          defaultValue={post.title}
          className="w-full rounded border px-3 py-2"
        />
        {actionData?.error ? (
          <p className="text-sm text-red-600">{actionData.error}</p>
        ) : null}
        <div className="flex gap-2">
          <button name="intent" value="save" disabled={busy}
            className="rounded bg-black px-4 py-2 text-white disabled:opacity-50">
            {busy ? "Saving…" : "Save"}
          </button>
          <button name="intent" value="delete"
            className="rounded border px-4 py-2 text-red-600">
            Delete
          </button>
        </div>
      </Form>
    </article>
  );
}

export function ErrorBoundary() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    return <p className="p-6">{error.status} — {error.statusText}</p>;
  }
  return <p className="p-6">Something went wrong.</p>;
}
```

Notes that matter:
- Loaders/actions **return plain objects**; do not wrap in `json()` unless you need custom headers/status (then `return Response.json(data, { headers })`). The return type flows into `loaderData`/`actionData` and `useLoaderData<typeof loader>()`.
- Throw a `Response` for not-found/error to hit the nearest `ErrorBoundary`; **return** a value for recoverable form-validation errors so the form re-renders with messages.
- A successful action automatically revalidates the page's loaders — you rarely refetch manually.
- Import `useRouteError`, `isRouteErrorResponse` from `react-router` at the top of the file.

## Data access from loaders/actions (see `drizzle`)

The **only** integration rule that lives here: build the Drizzle client **per request** from the D1 binding on `context`, never as a module-level singleton (the Worker has no `env` at module load). Keep it in a `.server.ts` factory:

```ts
// app/db/client.server.ts
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

// Per-request — NO module-level singleton; env only exists inside a request.
export function getDb(d1: D1Database) {
  return drizzle(d1, { schema });
}
export type DB = ReturnType<typeof getDb>;
```

Then in a loader/action: `const db = getDb(context.cloudflare.env.DB)` and query (see the route example above). That is the whole RR7↔data contract. **Schema definition, relations, the query/builder API, type inference (`$inferSelect`), `db.batch()` vs the no-interactive-transaction rule, and drizzle-kit migrations all live in the `drizzle` skill — do not restate them here.** Keep business logic in plain functions that take `db` + inputs so loaders/actions stay thin (and testable — see `tdd`).

## Runtime & bindings (see `cloudflare-workers`)

RR7's Cloudflare adapter surfaces the Worker's `env` to every loader/action as **`context.cloudflare.env`** and the execution context as **`context.cloudflare.ctx`**. That bridge is all this skill needs to know:

- Read the D1 binding as `context.cloudflare.env.DB`, secrets/vars as `context.cloudflare.env.X`.
- Use `context.cloudflare.ctx.waitUntil(promise)` for fire-and-forget work (e.g. logging) so it doesn't block the response.
- Respect the Workers model in loader/action code: request-scoped, no long-lived connections, bounded CPU, no Node built-ins without `nodejs_compat`.

**The `wrangler.jsonc` (D1/KV/vars bindings, `compatibility_date`/flags), `wrangler secret put` / `.dev.vars`, `wrangler types` for the `Env` interface, local dev (`wrangler dev` / the Cloudflare Vite plugin), and `wrangler deploy` all live in the `cloudflare-workers` skill** — go there to configure or deploy. Local dev tip: prefer the Cloudflare Vite plugin in framework mode so loaders see the real D1 binding.

## Data-fetching discipline (avoid waterfalls)

- Fire independent queries **in parallel** inside one loader: `const [a, b] = await Promise.all([...])`. Never `await` them sequentially when they don't depend on each other.
- Prefer one set-based SQL query (join / `with`) over N+1 per-row queries — D1 latency makes N+1 brutal.
- For a slow secondary piece of data, **stream it**: return the promise unawaited and render with `<Await>`/`Suspense` so the shell paints immediately (see `rr7-patterns.md`).
- Co-locate each loader with the route that needs it; nested routes load in parallel, so push data to the deepest route that owns it rather than over-fetching in the parent.

## Mutations, pending & optimistic UI

- Use `<Form method="post">` for navigations that change data; use `useFetcher()` for in-place mutations that should NOT navigate (like/favorite, inline edit, type-ahead).
- Read `useNavigation()` for global pending state; read `fetcher.state` / `fetcher.formData` for per-fetcher pending and **optimistic** values (render the submitted value before the server responds).
- Successful actions revalidate loaders automatically; call `useRevalidator()` only for non-mutation refreshes.

## Auth / sessions on Workers

Build sessions with RR7's cookie helpers, keyed by a secret from `context`:

```ts
// app/lib/session.server.ts
import { createCookieSessionStorage } from "react-router";

export function getSessionStorage(env: Env) {
  return createCookieSessionStorage({
    cookie: {
      name: "__session",
      httpOnly: true,
      sameSite: "lax",
      secure: true,
      path: "/",
      secrets: [env.SESSION_SECRET],
    },
  });
}
```

In a protected loader: read the cookie, look up the user (D1), and `throw redirect("/login")` when unauthenticated — throwing short-circuits to the redirect. Cookie session storage is ideal on Workers (no server state); for larger sessions, store a session id in the cookie and the payload in D1 or KV.

## TypeScript rules

- No `any`. Type loaders/actions via `Route.*`; derive db row types from `$inferSelect`/`$inferInsert`; bindings from generated `Env`.
- `useLoaderData<typeof loader>()` / the `loaderData` prop carries the loader's return type — don't re-annotate it.
- Keep server-only types and code behind `.server.ts` so they never leak into the client bundle.

