# Database Setup with Drizzle

Four common setups, each with the code and the reasons behind it:

| Setup | Runtime | Use when |
|---|---|---|
| [1. Postgres, direct](#1-node--postgres-direct) | Node (VPS / container) | 1–3 app instances, one Postgres |
| [2. Postgres via PgBouncer](#2-node--postgres-via-pgbouncer) | Node (VPS / container / Lambda) | Many instances, autoscaling, or serverless Node |
| [3. Cloudflare D1](#3-cloudflare-workers--d1) | Cloudflare Workers | Simple relational data, nothing Postgres-specific |
| [4. Postgres via Hyperdrive](#4-cloudflare-workers--postgres-via-hyperdrive) | Cloudflare Workers | You need Postgres from Workers |

**The rule behind all of them:**

- **Long-running Node:** the process stays alive, so create **one pool per process** at startup and reuse it for every request.
- **Serverless (Workers):** nothing stays alive between requests, so let the **platform pool connections** (D1 or Hyperdrive) and create a client **per request**.

---

## Shared package layout (all setups)

Keep the database package free of env and runtime code. It exports the schema and a factory, and each app passes in its own connection.

```ts
// packages/db/src/index.ts
export * from "./schema";
export function createDb(connection) { /* returns drizzle(...) */ }
```

```ts
// apps/server/src/db.ts
const db = createDb(/* connection the app owns */);
```

**Why:**

- Scripts, tests and tools can import the schema without loading the whole app's env.
- The same package works in Node and in Workers.
- Each app picks its own pool size or binding.

---

## Singleton or per request?

The common pattern: the package exports a factory, and the app creates the instance once in `db.ts`:

```ts
// packages/db: a factory, no env, no connection
export const createDb = (connection) => drizzle(...);

// apps/server/src/db.ts: the app creates it once
export const db = createDb(...);

// routes
import { db } from "./db";
```

Whether that module-level `db` is safe depends on **what `createDb` wraps**, because in both runtimes module code runs once and is then reused by many requests.

### Node: one `db` shared by every request

```ts
// apps/server/src/db.ts
import { Pool } from "pg";
import { createDb } from "@app/db";

const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });
export const db = createDb(pool);
```

```
process start ──► db.ts runs once ──► Pool (10 connections)
request 1 ─┐
request 2 ─┼──► same db ──► borrows a pooled connection per query
request N ─┘
```

- A module runs **once per process**. Every `import { db } from "./db"` gets the same object, because Node caches modules.
- The process runs for days, so **every request uses this same `db`**.
- `db` isn't one connection; it wraps a **pool**. Each query borrows a free connection and returns it afterwards. Fifty concurrent requests share the 10 connections, and the rest wait their turn.
- This is the point of the singleton: connections are opened once and reused. A new pool per request would open fresh connections every time and exhaust Postgres.

### Workers: module scope is reused too

A common assumption is that a Worker starts fresh for every request. **It doesn't.** Cloudflare starts an **isolate**, runs your module code once, then sends **many requests** to that isolate while it stays warm. It may run several isolates at once, each with its own copy of the module.

```
isolate A starts ──► db.ts runs once ──► db
   request 1 ─┐
   request 2 ─┼──► same db (while isolate A is alive)
   request 3 ─┘
isolate B (another location, or more traffic) ──► its own db
```

So a module-level `db` in a Worker **is** shared across requests, just like in Node.

**Workers + D1: a singleton is fine.**

```ts
import { env } from "cloudflare:workers";
export const db = createDb(env);      // ✅ OK (createDb reads env.DB)
```

- `drizzle(env.DB)` wraps a **binding**: no socket, no connection, and no network I/O when it's created.
- The D1 binding is designed to be used by any request.
- A singleton and a per-request `createDb(c.env)` behave the same. There's nothing to reuse, so the singleton doesn't make it faster.

**Workers + Postgres (`pg` / Hyperdrive): a singleton breaks.**

```ts
// ❌ don't
const client = new Client({ connectionString: env.HYPERDRIVE.connectionString });
await client.connect();             // ❌ module-level code can't open network connections
export const db = createDb(client); // ❌ request 2 can't use a socket opened in request 1
```

- Here `db` wraps a real **TCP connection**.
- Workers only allow opening network connections while handling a request, and a connection opened in one request **cannot be used by another**. You get an error like "Cannot perform I/O on behalf of a different request".
- Create the client **per request** and close it afterwards. Setup 4 below shows the full middleware.

  ```ts
  app.use("*", async (c, next) => {
    const client = new Client({ connectionString: c.env.HYPERDRIVE.connectionString });
    await client.connect();
    c.set("db", createDb(client));
    try { await next(); } finally { c.executionCtx.waitUntil(client.end()); }
  });
  ```
- This is cheap because **Hyperdrive** keeps the real pool near the database. In Workers, pooling lives in the platform, not in your code.

### Summary

| Runtime + database | What `db` wraps | Module-level singleton? | Why |
|---|---|---|---|
| Node + Postgres | A **pool** | ✅ **Yes, required** | One process serves all requests; reuse open connections |
| Node + PgBouncer | A pool (pointed at PgBouncer) | ✅ **Yes** | Same, with PgBouncer sharing connections between processes |
| Workers + D1 | A **binding** (no connection) | ✅ Fine either way | No connection to share; the wrapper costs nothing |
| Workers + Postgres / Hyperdrive | A **socket** | ❌ **No, create per request** | A connection can't be used by another request |

**Rule of thumb:** keep the factory in the package and `db.ts` in the app. What `db.ts` exports depends on the runtime:

- **Node:** `export const db = createDb(pool)`, one instance per process.
- **Workers + D1:** `export const db = createDb(env)` in `db.ts`, or create it per request. Both work. See [Setup 3](#3-cloudflare-workers--d1).
- **Workers + Postgres:** no exported `db`. Export a middleware or `getDb()` that creates a client for each request.

---

## 1. Node + Postgres (direct)

**Use when** a long-running Node server (VPS, Docker, VM) talks to one Postgres, with only a few app instances.

```ts
// packages/db/src/index.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

export function createPool(connectionString: string) {
  return new Pool({
    connectionString,
    max: 10,                       // connections per process
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 5_000,
  });
}

export const createDb = (pool: Pool) => drizzle({ client: pool, schema });
export * from "./schema";
```

```ts
// apps/server/src/db.ts
import { createDb, createPool } from "@app/db";

export const pool = createPool(process.env.DATABASE_URL!);
export const db = createDb(pool);   // one instance, created once, shared by every request

process.on("SIGTERM", async () => {
  await pool.end();                 // close connections cleanly on deploy or restart
  process.exit(0);
});
```

```ts
// anywhere in the server
import { db } from "./db";
const users = await db.query.users.findMany();
```

```ts
// packages/db/drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/schema",
  out: "./src/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

**Why:**

- **Connections are expensive and processes are long-lived.** Opening a Postgres connection costs a TCP handshake, TLS and authentication. A pool opens a few connections once and reuses them for thousands of requests.
- **One pool per process, at module level.** The process stays alive, so a singleton is correct. A pool created per request would open new connections every time and quickly hit Postgres's limit.
- **Size `max` deliberately.** Total connections = number of processes × `max`. This must stay under Postgres's `max_connections` (100 by default), with some room left over for migrations and admin tools.
- **Call `pool.end()` on shutdown.** Otherwise each deploy leaves old connections open until they time out.
- **Migrations:** `drizzle-kit generate` creates SQL files, and `drizzle-kit migrate` applies them.

---

## 2. Node + Postgres via PgBouncer

**Use when:**
- you run many processes (autoscaling, several services, cron jobs), or
- your app runs on serverless Node (Lambda, Vercel), or
- your provider gives you a "pooled" connection string (Supabase, Neon and others run PgBouncer for you).

```
app processes (many) ──► PgBouncer :6432 (transaction mode) ──► Postgres :5432 (few real connections)
migrations ────────────────────────────────────────────────────► Postgres :5432 (direct)
```

```ts
// apps/server/src/db.ts (node-postgres)
import { createDb, createPool } from "@app/db";

export const pool = createPool(process.env.DATABASE_URL!); // DATABASE_URL → PgBouncer
export const db = createDb(pool);
```

```ts
// apps/server/src/db.ts (postgres.js alternative)
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "@app/db/schema";

const sql = postgres(process.env.DATABASE_URL!, {
  prepare: false, // named prepared statements break in transaction mode
  max: 10,
});
export const db = drizzle({ client: sql, schema });
```

```ts
// packages/db/drizzle.config.ts: migrations bypass the pooler
export default defineConfig({
  schema: "./src/schema",
  out: "./src/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DIRECT_DATABASE_URL! }, // straight to Postgres :5432
});
```

```ini
; pgbouncer.ini (if you run it yourself)
[databases]
app = host=127.0.0.1 port=5432 dbname=app

[pgbouncer]
listen_port = 6432
pool_mode = transaction
default_pool_size = 20          ; real Postgres connections
max_client_conn = 500           ; app connections it accepts
max_prepared_statements = 100   ; PgBouncer >= 1.21: prepared statements work in transaction mode
```

**Why:**

- **Many processes × pool size runs out of connections.** For example, 20 containers × 10 = 200 connections, which is over Postgres's default limit. PgBouncer accepts all 200 app connections but shares just 20 real ones between them.
- **Transaction mode** gives a real connection to a client only for one transaction, then hands it to someone else. That's what makes the sharing work.
- **The consequence: no session state.** Consecutive transactions can run on different real connections, so these don't work:
  - `SET` outside a transaction
  - session-level advisory locks (use `pg_advisory_xact_lock` inside a transaction instead)
  - `LISTEN/NOTIFY`
  - temp tables kept across queries
- **Prepared statements.**
  - `node-postgres` uses unnamed statements for normal queries, so it just works.
  - `postgres.js` uses named ones, so set `prepare: false` (unless PgBouncer ≥ 1.21 with `max_prepared_statements`).
- **Use two URLs.**
  - The app uses `DATABASE_URL`, which points at the pooler.
  - Migrations use `DIRECT_DATABASE_URL`, which goes straight to Postgres. Migrations take locks and run long multi-statement transactions, which shouldn't go through transaction pooling.
- **Keep transactions short.** A transaction holds a real connection for its whole duration.

---

## 3. Cloudflare Workers + D1

**Use when** a Workers app has relational data that fits SQLite, with no Postgres-specific features needed.

```jsonc
// wrangler.jsonc
{
  "name": "api",
  "main": "src/index.ts",
  "compatibility_date": "2026-11-01",
  "d1_databases": [
    { "binding": "DB", "database_name": "app-db", "database_id": "<d1-database-id>" }
  ]
}
```

```ts
// packages/db/src/index.ts
/// <reference types="@cloudflare/workers-types" />
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

// The package only says which bindings it needs. The app passes its Env in.
export type DatabaseEnv = { DB: D1Database };

export const createDb = (env: DatabaseEnv) => drizzle(env.DB, { schema });
export type Database = ReturnType<typeof createDb>;
export * from "./schema";
```

Generate the `Env` type from `wrangler.jsonc` so the runtime `env` is typed:

```bash
npx wrangler types   # writes worker-configuration.d.ts with Env { DB: D1Database; ... }
```

### Option A: `db.ts` singleton (recommended for D1)

```ts
// apps/server/src/db.ts
import { env } from "cloudflare:workers";   // the Worker's Env, available at module level
import { createDb } from "@app/db";

export const db = createDb(env);             // Env has DB: D1Database, so it matches DatabaseEnv
```

```ts
// apps/server/src/index.ts (Hono)
import { users } from "@app/db";
import { Hono } from "hono";
import { db } from "./db";

const app = new Hono();

app.get("/users", async (c) => c.json(await db.query.users.findMany()));

app.post("/users", async (c) => {
  const body = await c.req.json();
  const [created] = await db.insert(users).values(body).returning();
  return c.json(created, 201);
});

export default app;
```

**Why this works:**

- **It's the same pattern as Node:** factory in the package, one instance in `db.ts`, imported everywhere.
- **`env` from `cloudflare:workers` can be read at module level.** It's the same `Env` your handler would receive, so `createDb(env)` can run when the module loads.
- **`db` is created once per isolate** and reused by every request that isolate handles. That's safe because it only wraps the D1 binding: there's no connection, and nothing to share incorrectly.
- **Passing the whole `Env` keeps the package independent of your binding setup.**
  - `createDb` only requires `{ DB: D1Database }`, so any `Env` that has a `DB` binding fits.
  - The package itself never imports `cloudflare:workers`, so it still loads in Node: scripts, tests, drizzle-kit.
- **Keep `db.ts` inside the Worker app.** It imports `cloudflare:workers`, which only exists in the Workers runtime.

### Option B: per request from `c.env`

```ts
// apps/server/src/index.ts (Hono)
import { createDb } from "@app/db";
import { Hono } from "hono";

const app = new Hono<{ Bindings: Env }>();   // Env from `wrangler types`

app.get("/users", async (c) => {
  const db = createDb(c.env);                // per request
  return c.json(await db.query.users.findMany());
});

export default app;
```

**Same result.** Creating the wrapper costs nothing, so use this if you prefer receiving `env` from the request over importing `cloudflare:workers`, e.g. in code that also runs in tests where you pass a fake `env`.

```ts
// packages/db/drizzle.config.ts: migrations over the D1 HTTP API
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/schema",
  out: "./src/migrations",
  dialect: "sqlite",
  driver: "d1-http",
  dbCredentials: {
    accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
    databaseId: process.env.CLOUDFLARE_DATABASE_ID!,
    token: process.env.CLOUDFLARE_D1_TOKEN!,    // API token with D1 edit permission
  },
});
```

**Why:**

- **D1 is a binding, not a connection string.**
  - The Workers runtime hands your code a ready object (`env.DB`), so there's no TCP connection, pool or password in your code.
  - The `D1Database` type comes from `@cloudflare/workers-types`. It's only a type; the real object is provided at runtime.
- **Calling `createDb` per request is fine.** `drizzle(env.DB)` only wraps the binding, with no I/O, so it costs nothing. You could also build it once at module level (`import { env } from "cloudflare:workers"`), but there's nothing to gain.
- **Keep `cloudflare:workers` out of the shared package.** That module only exists inside the Workers runtime, so importing it would make the package crash in plain Node: scripts, tests, drizzle-kit.
- **Atomic writes use `db.batch([...])`.** D1 doesn't support interactive `BEGIN … COMMIT` transactions.
- **Migrations:** `drizzle-kit generate`, then apply with `drizzle-kit migrate` (D1 HTTP API, above). Use **one** migration tool per database. `wrangler d1 migrations apply` and `drizzle-kit migrate` record history in different tables, so mixing them makes migrations run twice.

---

## 4. Cloudflare Workers + Postgres via Hyperdrive

**Use when** a Workers app needs Postgres: extensions, large data, or an existing database.

```
Worker (any location) ──► Hyperdrive (pooled, transaction mode, read cache) ──► Postgres
migrations ─────────────────────────────────────────────────────────────────► Postgres (direct)
```

```jsonc
// wrangler.jsonc
{
  "name": "api",
  "main": "src/index.ts",
  "compatibility_date": "2026-11-01",
  "compatibility_flags": ["nodejs_compat"],            // required by the pg driver
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<hyperdrive-config-id>",                   // created with `wrangler hyperdrive create`
      "localConnectionString": "postgres://postgres:postgres@localhost:5432/app" // for `wrangler dev`
    }
  ]
}
```

```ts
// packages/db/src/index.ts
import { drizzle } from "drizzle-orm/node-postgres";
import type { Client } from "pg";
import * as schema from "./schema";

export const createDb = (client: Client) => drizzle({ client, schema });
export * from "./schema";
```

```ts
// apps/server/src/index.ts (Hono)
import { createDb } from "@app/db";
import { Hono } from "hono";
import { createMiddleware } from "hono/factory";
import { Client } from "pg";

type Env = { HYPERDRIVE: Hyperdrive };
type Vars = { db: ReturnType<typeof createDb> };
const app = new Hono<{ Bindings: Env; Variables: Vars }>();

const withDb = createMiddleware<{ Bindings: Env; Variables: Vars }>(async (c, next) => {
  const client = new Client({ connectionString: c.env.HYPERDRIVE.connectionString });
  await client.connect();
  c.set("db", createDb(client));
  try {
    await next();
  } finally {
    c.executionCtx.waitUntil(client.end());   // close after the response, without delaying it
  }
});

app.use("/users/*", withDb);                     // only routes that need the database
app.get("/users", async (c) => c.json(await c.var.db.query.users.findMany()));

export default app;
```

```ts
// packages/db/drizzle.config.ts: migrations go straight to Postgres, not through Hyperdrive
export default defineConfig({
  schema: "./src/schema",
  out: "./src/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DIRECT_DATABASE_URL! },
});
```

**Why:**

- **Direct TCP from a Worker is slow.** A Worker *can* connect to Postgres directly, but each request would pay the full connect cost (TCP, TLS, authentication), often from a location far from the database. Hyperdrive keeps a warm pool of connections near your database, so connecting from the Worker is fast.
- **A new `Client` per request, never at module level.**
  - Workers can't reuse an open connection from one request in another, so a module-level client or pool breaks.
  - This is cheap because Hyperdrive holds the real pool. It's also what Cloudflare's own guide does.
- **Close the client with `ctx.waitUntil(client.end())`.** The connection is released after the response is sent, without making the user wait.
- **Use `node-postgres` (`pg`).** Cloudflare: *"Pairing Drizzle ORM with the Postgres.js driver over Hyperdrive is not currently supported."* Hyperdrive supports `pg`'s prepared statements, so no special options are needed.
- **Transaction-mode rules apply**, as with PgBouncer: no session state, and keep transactions short.
- **Read queries are cached by default** (about 60s).
  - A read right after a write can return stale data.
  - Where reads must be fresh, disable caching on the Hyperdrive config, or create a second config with caching off and bind it separately.
- **Migrations** use `DIRECT_DATABASE_URL`, never Hyperdrive, for the same reason as with PgBouncer.

### Hyperdrive with PgBouncer behind it?

Usually not needed: Hyperdrive already is a pooler. Add PgBouncer only if **other clients** (a Node API, cron jobs) share the same Postgres and you want one place that limits total connections. In that case, point Hyperdrive at PgBouncer's port.

---

## Side-by-side summary

| | 1. Node + Postgres | 2. Node + PgBouncer | 3. Workers + D1 | 4. Workers + Hyperdrive |
|---|---|---|---|---|
| Client lifetime | One pool per process | One pool per process | Per request (free) | New `Client` per request |
| Who pools | Your app | PgBouncer | Not needed | Hyperdrive |
| Drizzle driver | `node-postgres` | `node-postgres` / `postgres-js` (`prepare: false`) | `d1` | `node-postgres` only |
| App connects with | `DATABASE_URL` | Pooler URL | `env.DB` binding | `env.HYPERDRIVE` binding |
| Migrations connect with | `DATABASE_URL` | Direct URL | D1 HTTP API | Direct URL |
| Session features (SET, locks, LISTEN) | ✅ | ❌ | n/a | ❌ |
| Transactions | Full | Full, keep short | `db.batch()` | Full, keep short |
