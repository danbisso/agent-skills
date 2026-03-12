---
name: cloudflare-workers
description: Own the Cloudflare Workers runtime and platform in TypeScript — the fetch/scheduled/queue handlers, the request-scoped isolate model, env bindings (D1 as env.DB, plus KV, R2, Queues, Durable Objects, service bindings, secrets, vars), wrangler.jsonc config, local dev (wrangler dev / Vite plugin / workerd), wrangler types, D1 migrations, secrets, deploy, and runtime limits. Use when writing or debugging a Worker handler, wiring or naming a binding, editing wrangler config, running wrangler d1 migrations apply, generating the Env interface, configuring cron/queues, or hitting Workers runtime limits (no Node built-ins, CPU/size caps, no long-lived connections).
---

# cloudflare-workers — the edge runtime & platform

You own the **Cloudflare Workers runtime and platform**. Everything here is the source of truth for
how a Worker executes, how it reaches resources (D1, KV, R2, Queues, secrets) through `env`, and how
`wrangler` is configured, run, and deployed. TypeScript throughout.

Cross-links — do NOT duplicate these:

- **`drizzle`** — Drizzle ORM schema, queries, and migration *authoring*. This skill provides the
  D1 binding (`env.DB`) and the `wrangler d1 migrations apply` runner only; defer all ORM depth.
- **`frontend-dev`** — React Router 7 routing/loaders/actions. RR7's Cloudflare adapter surfaces
  `env` to loaders/actions as `context.cloudflare.env`; that wiring lives there, not here.
- **`aws-architect`** — the boundary. Workers/D1 is the **edge/app runtime** (request-time logic,
  lightweight KV/SQLite, close to the user). Durable compute, long workflows, large memory/GPU,
  multi-step orchestration hand off to AWS. Do not improvise AWS here.

Extended config matrix and command cheatsheet: see `wrangler-reference.md` (same dir).

## Execution model (read first)

A Worker is a set of **handlers** on a default-exported object. The runtime invokes the matching one
per event. The canonical handler is `fetch`:

```ts
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname === "/health") return new Response("ok");

    const row = await env.DB.prepare("select count(*) as n from items").first<{ n: number }>();
    // fire-and-forget work must be handed to the runtime, not left dangling:
    ctx.waitUntil(env.METRICS.put(`hit:${Date.now()}`, "1"));
    return Response.json({ count: row?.n ?? 0 });
  },
} satisfies ExportedHandler<Env>;
```

Internalize the constraints — they shape every line of Worker code:

- **It is not Node.** No `fs`, `path`, `net`, `http`, `process`, Buffer-by-default, etc. unless you
  enable `nodejs_compat` (see below) — and even then only a subset is polyfilled. Prefer
  **Web-standard APIs**: `fetch`, `URL`, `Request`/`Response`, `crypto.subtle` (Web Crypto),
  `TextEncoder`, `ReadableStream`/`WritableStream`, `structuredClone`.
- **Request-scoped isolates.** Each request runs in a V8 isolate that may be reused or torn down at
  any time. **Never** create module-level connections, singletons holding live state, or
  `setInterval`/background timers expecting to outlive a response. Global scope is for pure
  constants and code only. Initialize per-request inside the handler.
- **Bounded CPU per request.** CPU time (not wall-clock) is capped (default 30s on paid; 10ms on the
  free tier for the burst, with higher sustained allowances — check the plan). Push set-based work
  into SQL, stream large bodies, avoid tight CPU loops. I/O wait does not count against CPU.
- **No long-lived sockets.** No raw TCP/UDP servers. Outbound is `fetch` (and TCP via
  `connect()` from `cloudflare:sockets` for specific cases). Connections do not persist between
  requests.
- **`ctx.waitUntil(promise)`** extends the Worker's life past the returned `Response` for
  fire-and-forget work (logging, cache writes, queue sends). Without it, work started after `return`
  may be killed. `ctx.passThroughOnException()` lets origin requests fall through on error.

## Bindings — the core idea

A Worker reaches **every** external resource and secret through the **`env`** object passed to each
handler. Bindings are the only supported channel — there is no ambient global, no connection string
read from `process.env` at module load. You declare a binding in `wrangler.jsonc`, give it a name,
and that name becomes a typed property on `env`.

| Binding | wrangler key | Accessed as | Use for |
|---|---|---|---|
| **D1** (SQLite) | `d1_databases` | `env.DB` | relational app data (primary store) |
| KV | `kv_namespaces` | `env.CACHE` | edge key/value, config, low-write caches |
| R2 | `r2_buckets` | `env.BUCKET` | object/blob storage (S3-like, no egress fee) |
| Queues | `queues.producers` | `env.QUEUE` | async/decoupled work, batching |
| Durable Objects | `durable_objects` | `env.ROOM` | single-instance coordination, stateful sockets |
| Service binding | `services` | `env.AUTH` | call another Worker in-process (no network) |
| Secret | `wrangler secret put` | `env.API_KEY` | credentials, tokens (never in config/code) |
| Plain var | `vars` | `env.STAGE` | non-secret config (environment name, flags) |

### D1 (center of gravity)

D1 is Cloudflare's serverless SQLite. Bind it and use the prepared-statement API:

```ts
// single row
const user = await env.DB
  .prepare("select * from users where id = ?")
  .bind(userId)
  .first<{ id: string; email: string }>();

// list
const { results } = await env.DB
  .prepare("select * from items where owner = ? order by created_at desc limit 50")
  .bind(userId)
  .all<Item>();

// atomic multi-statement: db.batch() is the ONLY transaction primitive.
// There are NO interactive transactions (no BEGIN ... app logic ... COMMIT).
// All statements in a batch commit together or not at all.
await env.DB.batch([
  env.DB.prepare("update accounts set bal = bal - ? where id = ?").bind(amt, from),
  env.DB.prepare("update accounts set bal = bal + ? where id = ?").bind(amt, to),
]);
```

For schema, query building, and typed migrations, use **Drizzle** (`drizzle` skill) — point it at
the `env.DB` binding. This skill only owns the binding and the migration *runner* (below).

### Other bindings, briefly

```ts
await env.CACHE.put("k", JSON.stringify(v), { expirationTtl: 60 });   // KV
const obj = await env.BUCKET.get("path/to/file.png");                  // R2
await env.QUEUE.send({ type: "email", to });                          // Queue producer
const stub = env.ROOM.get(env.ROOM.idFromName("game-42"));            // Durable Object
const res = await env.AUTH.fetch(request);                            // Service binding (in-proc)
```

KV is **eventually consistent** and read-optimized — do not use it as a database or for
read-after-write correctness. R2 is for blobs. Service bindings call another Worker with no public
network hop and no extra latency/egress.

## wrangler.jsonc — representative config

Cloudflare's current config format is **`wrangler.jsonc`** (TOML still works but JSONC is preferred
and what `wrangler types` and the Vite plugin assume).

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-app",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",
  "compatibility_flags": ["nodejs_compat"],

  "observability": { "enabled": true },

  "vars": {
    "STAGE": "dev"
  },

  "d1_databases": [
    {
      "binding": "DB",                       // -> env.DB
      "database_name": "my-app-db",
      "database_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "migrations_dir": "drizzle/migrations"
    }
  ],

  "kv_namespaces": [
    { "binding": "CACHE", "id": "abc123..." }
  ],

  "r2_buckets": [
    { "binding": "BUCKET", "bucket_name": "my-app-assets" }
  ],

  "queues": {
    "producers": [{ "binding": "QUEUE", "queue": "jobs" }],
    "consumers": [{ "queue": "jobs", "max_batch_size": 10, "max_batch_timeout": 5 }]
  },

  "triggers": { "crons": ["0 * * * *"] },

  "routes": [{ "pattern": "app.example.com", "custom_domain": true }],

  // Per-environment overrides. Deploy with `wrangler deploy --env production`.
  "env": {
    "production": {
      "vars": { "STAGE": "production" },
      "d1_databases": [
        {
          "binding": "DB",
          "database_name": "my-app-db-prod",
          "database_id": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy",
          "migrations_dir": "drizzle/migrations"
        }
      ],
      "routes": [{ "pattern": "app.example.com", "custom_domain": true }]
    }
  }
}
```

Notes:

- `compatibility_date` pins runtime semantics; bump it deliberately. `compatibility_flags` opt into
  behaviors — most commonly `"nodejs_compat"` for partial Node API support.
- Top-level config is the **default/dev** environment. Named environments under `env.<name>`
  **fully replace** (not deep-merge) array/object sections like `d1_databases` and `vars`, so repeat
  every binding you need in each environment.
- `migrations_dir` points at the SQL migration folder consumed by `wrangler d1 migrations apply`.

## Secrets & config

- **Secrets** (credentials, tokens): `wrangler secret put API_KEY` (prompts; stored encrypted on
  Cloudflare), read as `env.API_KEY`. Never put secrets in `wrangler.jsonc` or source.
- **Local secrets**: a gitignored **`.dev.vars`** file (`KEY=value` lines) supplies secrets/vars to
  `wrangler dev`. Add `.dev.vars` to `.gitignore`.
- **Non-secret config**: `vars` in wrangler config, read as `env.STAGE`.

```
# .dev.vars  (gitignored)
API_KEY=sk_test_local_only
STAGE=dev
```

## D1 operationally (runner only — author migrations via `drizzle`)

```bash
# apply pending migrations to the LOCAL simulated D1 (workerd/Miniflare state)
wrangler d1 migrations apply DB --local

# apply to the REMOTE production D1
wrangler d1 migrations apply DB --remote --env production

# ad-hoc query (debugging only)
wrangler d1 execute DB --local --command "select count(*) from users"
```

`--local` targets the on-disk simulated DB used by `wrangler dev`; `--remote` hits the real
Cloudflare D1. They are separate datasets — local migrations never touch production. Migration
*files* are generated by Drizzle (`drizzle-kit generate`); this command only applies them.

