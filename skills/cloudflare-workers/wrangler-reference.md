# wrangler reference — config matrix & command cheatsheet

Companion to `SKILL.md`. Quick lookup for binding config shapes and the commands you run most.

## Command cheatsheet

| Command | What it does |
|---|---|
| `wrangler dev` | Run locally in workerd (Miniflare) with simulated bindings (`--local`, default) |
| `wrangler dev --remote` | Run on Cloudflare's edge against real remote resources |
| `wrangler deploy` | Build + upload + activate the Worker (default environment) |
| `wrangler deploy --env production` | Deploy a named environment from `env.production` |
| `wrangler versions upload` | Upload a version without activating (for gradual/canary deploys) |
| `wrangler types` | Generate the typed `Env` interface from `wrangler.jsonc` bindings |
| `wrangler tail` | Stream live logs/exceptions from the deployed Worker |
| `wrangler secret put NAME` | Set an encrypted secret (read as `env.NAME`) |
| `wrangler secret list` | List secret names (not values) |
| `wrangler secret delete NAME` | Remove a secret |
| `wrangler d1 migrations apply DB --local` | Apply migrations to local simulated D1 |
| `wrangler d1 migrations apply DB --remote` | Apply migrations to remote D1 |
| `wrangler d1 execute DB --local --command "..."` | Run ad-hoc SQL (debug only) |
| `wrangler kv key put --binding=CACHE k v` | Write a KV key (add `--local` for local) |
| `wrangler r2 object get BUCKET path` | Read an R2 object |

Append `--env <name>` to target a named environment for `deploy`, `secret`, `d1`, etc.

## Binding config shapes (wrangler.jsonc)

```jsonc
// D1
"d1_databases": [
  { "binding": "DB", "database_name": "app-db", "database_id": "uuid", "migrations_dir": "drizzle/migrations" }
],

// KV
"kv_namespaces": [
  { "binding": "CACHE", "id": "namespace-id", "preview_id": "preview-namespace-id" }
],

// R2
"r2_buckets": [
  { "binding": "BUCKET", "bucket_name": "app-assets", "preview_bucket_name": "app-assets-preview" }
],

// Queues (producer + consumer)
"queues": {
  "producers": [{ "binding": "QUEUE", "queue": "jobs" }],
  "consumers": [
    {
      "queue": "jobs",
      "max_batch_size": 10,
      "max_batch_timeout": 5,
      "max_retries": 3,
      "dead_letter_queue": "jobs-dlq"
    }
  ]
},

// Durable Objects (+ migration tag to register the class)
"durable_objects": {
  "bindings": [{ "name": "ROOM", "class_name": "Room" }]
},
"migrations": [
  { "tag": "v1", "new_sqlite_classes": ["Room"] }
],

// Service binding (call another Worker in-process)
"services": [
  { "binding": "AUTH", "service": "auth-worker" }
],

// Plain (non-secret) vars
"vars": { "STAGE": "dev", "FEATURE_X": "on" },

// Static assets (SPA / static files served by the Worker)
"assets": { "directory": "./public", "binding": "ASSETS" }
```

## Compatibility

- `compatibility_date`: pins runtime behavior to a date. Pin it; bump deliberately and test.
- `compatibility_flags`: opt into specific behaviors. Common: `["nodejs_compat"]`.
- `nodejs_compat` polyfills only a subset of Node APIs (some `node:` modules, partial Buffer/stream).
  Native addons and many Node-only libraries still will not run at the edge — verify before adopting.

## Environments

- Top-level config = default (used by `wrangler dev` and a bare `wrangler deploy`).
- `env.<name>` blocks are selected with `--env <name>`.
- Array/object sections (`d1_databases`, `kv_namespaces`, `vars`, `routes`, …) **fully replace** the
  top-level value for that environment — they do NOT deep-merge. Repeat every binding the
  environment needs.

## Secrets locally

- `.dev.vars` (gitignored), `KEY=value` per line, supplies secrets + vars to `wrangler dev`.
- Per-environment: `.dev.vars.<env>` (e.g. `.dev.vars.staging`).
- Production secrets: `wrangler secret put NAME --env production`.

## Generated types

`wrangler types` writes a `worker-configuration.d.ts` with an `Env` interface reflecting all
bindings + vars. Commit it (or regenerate in CI) and use it in handler signatures:

```ts
export default { async fetch(req, env: Env, ctx) { /* ... */ } } satisfies ExportedHandler<Env>;
```

Re-run after any binding/var add, rename, or removal.
