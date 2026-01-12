# RR7 framework-mode patterns (extended)

Companion to `SKILL.md`. All imports from `react-router` / `@react-router/*`.

## Streaming / deferred data (avoid blocking on slow queries)

Return a promise **unawaited** from the loader; render the fast data immediately and stream the rest.

```tsx
import { Await } from "react-router";
import { Suspense } from "react";
import type { Route } from "./+types/dashboard";
import { getDb } from "~/db/client.server";

export async function loader({ context }: Route.LoaderArgs) {
  const db = getDb(context.cloudflare.env.DB);
  const user = await db.query.users.findFirst({ /* fast, await it */ });
  const stats = db.query.events.findMany({ /* slow — do NOT await */ });
  return { user, stats }; // stats is a Promise; RR7 streams it
}

export default function Dashboard({ loaderData }: Route.ComponentProps) {
  return (
    <>
      <h1>{loaderData.user?.email}</h1>
      <Suspense fallback={<p>Loading stats…</p>}>
        <Await resolve={loaderData.stats}>
          {(stats) => <Stats data={stats} />}
        </Await>
      </Suspense>
    </>
  );
}
```

Stream only genuinely slow, non-critical data; awaited data still blocks the response.

## useFetcher — mutate without navigating

```tsx
import { useFetcher } from "react-router";

function LikeButton({ id, liked }: { id: number; liked: boolean }) {
  const fetcher = useFetcher();
  // optimistic: trust the in-flight submission before the server answers
  const optimistic = fetcher.formData
    ? fetcher.formData.get("liked") === "true"
    : liked;

  return (
    <fetcher.Form method="post" action={`/posts/${id}/like`}>
      <input type="hidden" name="liked" value={String(!optimistic)} />
      <button disabled={fetcher.state !== "idle"}>
        {optimistic ? "♥ Liked" : "♡ Like"}
      </button>
    </fetcher.Form>
  );
}
```

Fetchers don't change the URL; the targeted route's loaders still revalidate on success.

## clientLoader / clientAction

Run logic in the browser (e.g. read a cached value, or hit a same-origin API) while still having a server loader for SSR. `clientLoader` can call the server loader via `serverLoader()`.

```ts
export async function loader({ context }: Route.LoaderArgs) { /* server */ }

export async function clientLoader({ serverLoader }: Route.ClientLoaderArgs) {
  const cached = sessionStorage.getItem("k");
  if (cached) return JSON.parse(cached);
  const data = await serverLoader();          // calls the server loader
  sessionStorage.setItem("k", JSON.stringify(data));
  return data;
}
clientLoader.hydrate = true as const;          // run on initial hydration too
```

Use sparingly — prefer the server loader. `clientAction` mirrors this for client-side mutation logic.

## root.tsx shell, meta & links

```tsx
import { Links, Meta, Outlet, Scripts, ScrollRestoration } from "react-router";
import type { Route } from "./+types/root";
import stylesheet from "./app.css?url";

export const links: Route.LinksFunction = () => [
  { rel: "stylesheet", href: stylesheet }, // Tailwind entry css
];

export function meta(_: Route.MetaArgs) {
  return [{ title: "App" }, { name: "viewport", content: "width=device-width,initial-scale=1" }];
}

export function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head><Meta /><Links /></head>
      <body>
        {children}
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}

export default function Root() {
  return <Outlet />;
}
```

`links` is where Tailwind's compiled CSS is attached. Per-route `meta` merges over root meta. (Token/theme setup lives in `frontend-design`.)

## Error boundaries

```tsx
import { isRouteErrorResponse, useRouteError } from "react-router";

export function ErrorBoundary() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    return <div className="p-6">{error.status} {error.statusText}</div>;
  }
  const message = error instanceof Error ? error.message : "Unknown error";
  return <div className="p-6 text-red-600">{message}</div>;
}
```

A route's `ErrorBoundary` catches thrown `Response`s and errors from its loader/action/render and from descendant routes that lack their own boundary. Always give `root.tsx` one.

## headers / caching

```ts
export function headers(_: Route.HeadersArgs) {
  return { "Cache-Control": "public, max-age=60, s-maxage=300" };
}
```

Set `Cache-Control` via the route `headers` export to leverage Cloudflare's edge cache for cacheable GET loaders. For per-response control, return `Response.json(data, { headers })` from the loader instead of a bare object.

## Redirects & status

`throw redirect("/path", { status: 302 })` from loader/action to short-circuit (auth gating, post-mutation navigation). `throw new Response(null, { status: 404 })` to hit the error boundary. Return values are for rendering; throws are for control flow.
