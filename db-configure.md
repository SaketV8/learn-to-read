# Database Configuration Guide

How to set up and use a database with Drizzle in this monorepo style (`packages/db` shared by `apps/server`). It covers two kinds of runtime:

- **Serverless**: Cloudflare Workers (this repo, `my-better-t-app-v2`)
- **Long-running Node**: a VPS, container or VM (`compile-x`)

It also covers four connection setups: Postgres direct, Postgres via PgBouncer, Cloudflare D1, and Postgres via Hyperdrive.

> The Cloudflare, Drizzle and Alchemy details here were checked against the Cloudflare Hyperdrive docs, the Drizzle Postgres/D1 docs, and the Alchemy `2.0.0-beta.84` source installed in this repo (Nov 2026). Hostnames, users and passwords in the examples are placeholders.

---

## Contents

1. [Quick decision guide](#1-quick-decision-guide)
2. [Serverless vs Node: what actually changes](#2-serverless-cloudflare-workers-vs-node-vps-what-actually-changes)
3. [Side-by-side comparison](#3-side-by-side-comparison)
4. [The one rule: shape of the shared `db` package](#4-the-one-rule-shape-of-the-shared-db-package)
5. [Setup A: Node + Postgres (direct)](#5-setup-a-node--postgres-direct)
6. [Setup B: Node + PgBouncer](#6-setup-b-node--pgbouncer)
7. [Setup C: Workers + D1 (this repo)](#7-setup-c-workers--d1-this-repo)
8. [Setup D: Workers + Hyperdrive → Postgres](#8-setup-d-workers--hyperdrive--postgres)
9. [Hyperdrive and PgBouncer together?](#9-hyperdrive-and-pgbouncer-together)
10. [Other options worth knowing](#10-other-options-worth-knowing)
11. [Environment variables: t3-env vs varlock](#11-environment-variables-t3-env-vs-varlock)
12. [Migrations per setup](#12-migrations-per-setup)
13. [Connection math](#13-connection-math)
14. [Local development per setup](#14-local-development-per-setup)
15. [Gotchas checklist](#15-gotchas-checklist)
16. [How `env.DB` reaches the code in this repo](#16-appendix-how-envdb-reaches-the-code-in-this-repo)

---

## 1. Quick decision guide

### Step 1: where does the API run?

| You deploy to… | Runtime type | Go to |
|---|---|---|
| Cloudflare Workers | Serverless (short-lived isolates, at the edge) | Step 2a |
| A VPS, Docker, Kubernetes, ECS, Fly or Railway (a process that stays up) | Long-running Node | Step 2b |
| AWS Lambda or Vercel Functions (Node serverless) | Serverless (many short-lived processes) | Step 2b, but **always** use a pooler (PgBouncer, RDS Proxy, or your provider's pooled URL) |

### Step 2a: Cloudflare Workers

| Need | Use |
|---|---|
| A new app with simple relational data, no Postgres-specific features, little to operate | **D1** (Setup C) |
| Postgres features (extensions such as `pgvector` or PostGIS, `jsonb`, full-text search), large data, or an existing Postgres | **Hyperdrive → Postgres** (Setup D) |
| An existing Postgres that other services also use heavily | **Hyperdrive → PgBouncer → Postgres** (section 9) |

### Step 2b: Node on a VPS or container

| Need | Use |
|---|---|
| 1–3 app instances and one Postgres | **Pool singleton** (Setup A) |
| Many instances, autoscaling, several services, cron jobs, or Node serverless | **PgBouncer in transaction mode** (Setup B) |
| One small box and a small app where you want SQLite | `better-sqlite3`, or libSQL/Turso (section 10) |

### Rule of thumb

> **Serverless → the platform pools for you (Hyperdrive, D1, a provider pooler); create a client per request.**
> **Long-running Node → you pool in-process (one `Pool` per process); add PgBouncer once process count × pool size gets near Postgres's connection limit.**

---

## 2. Serverless (Cloudflare Workers) vs Node (VPS): what actually changes

| | **Cloudflare Workers** | **Node on a VPS** |
|---|---|---|
| Process lifetime | Isolates are created and discarded by Cloudflare. No guaranteed long-lived memory. | One process runs for days or weeks. |
| Where it runs | Cloudflare locations worldwide, close to the user, often far from the database | One region, usually next to the database |
| Concurrency model | Many isolates in many locations, so connection count is unpredictable | Known: number of processes × pool size |
| Raw TCP to Postgres | Possible (`pg` with `nodejs_compat`), but each request pays several round trips for TCP, TLS and authentication | Normal. Connections are opened once and reused. |
| Sharing an open connection between requests | **Not allowed.** A connection opened in one request can't be used by another. | Normal. That's what a pool is for. |
| Configuration and secrets | **Bindings** (`env.DB`, `env.HYPERDRIVE`, `env.SECRET`) from `cloudflare:workers` | `process.env`, loaded from `.env` or the orchestrator |
| Database objects | D1, Hyperdrive and KV are **binding objects**, not URLs | Always a connection **string** |
| What a module-level `db` singleton gives you | Nothing for D1 (a free wrapper); for Postgres it **breaks** (can't share the connection) | Correct: one pool per process |
| Cold start | Very small, but the first DB connection is the expensive part (Hyperdrive fixes that) | Not an issue after boot |
| Max request time and CPU | Limited per request (plan-dependent) | No limit beyond your own timeouts |
| Background work after the response | `ctx.waitUntil(promise)` | Just keep the promise running |
| Scaling cost | Automatic; pay per request | Pay for the box; scale yourself |

**What this means for the database:**

- **Workers:** never keep a Postgres connection at module level. Either use a binding that needs no connection (D1), or connect per request through a pooler that lives near the database (Hyperdrive).
- **Node:** create one pool per process at startup and reuse it for every request. Add an external pooler (PgBouncer) only when the number of processes multiplies connections too far.

---

## 3. Side-by-side comparison

| | **A. Node + Postgres** | **B. Node + PgBouncer** | **C. Workers + D1** | **D. Workers + Hyperdrive → Postgres** |
|---|---|---|---|---|
| Example | `compile-x` | Scaled `compile-x` | `my-better-t-app-v2` (now) | v2 with Postgres |
| Database engine | Postgres | Postgres | SQLite (D1) | Postgres |
| How the app reaches the DB | TCP to Postgres | TCP to PgBouncer, which talks to Postgres | Binding `env.DB` | Binding `env.HYPERDRIVE.connectionString`, then TCP to Hyperdrive, which talks to Postgres |
| Who pools connections | Your app (`pg.Pool`) | PgBouncer, plus a small app pool | Nobody needs to | Hyperdrive (transaction mode) |
| Client lifetime | One for the process | One for the process | Per request (free) | New `Client` per request |
| Drizzle import | `drizzle-orm/node-postgres` | `drizzle-orm/node-postgres` (or `postgres-js` with `prepare: false`) | `drizzle-orm/d1` | `drizzle-orm/node-postgres` **only** |
| Connection config | `DATABASE_URL` | `DATABASE_URL` (pooler) + `DIRECT_DATABASE_URL` | Binding name | Binding name. The real credentials live in the Hyperdrive config. |
| Migrations | `drizzle-kit migrate` | `drizzle-kit migrate` on the **direct** URL | Alchemy applies them on dev/deploy | `drizzle-kit migrate` on the **direct** URL |
| Transactions | Full | Full (but keep them short) | `db.batch([...])` (no interactive `BEGIN…COMMIT`) | Full, but keep them short |
| Session features (SET, advisory locks, LISTEN/NOTIFY, temp tables) | Work | **Break** in transaction mode | n/a | **Restricted** (SET lasts for that transaction only, no session-level advisory locks) |
| Read cache | No | No | No | **Yes, on by default** (60s) |
| Local dev | Local Postgres | Local Postgres (skip PgBouncer) | Alchemy local SQLite file | Alchemy `dev:` origin pointing at a local Postgres |
| Operational effort | Low | Medium (one more service) | Lowest | Low (managed by Cloudflare) |

---

## 4. The one rule: shape of the shared `db` package

In every setup, `packages/db` should:

1. Export the **schema** and **relations**, so they can be imported anywhere.
2. Export a **`createDb(...)` that receives its connection as a parameter**: a pool, a client or a binding.
3. **Not** read env itself, and **not** import runtime-only modules (`cloudflare:workers`, a whole-server env package).

Each **app** decides how to get the connection: a pool at module level in Node, a binding or a per-request client in Workers.

```
packages/db   →  schema, relations, createDb(connection)        // runs in any runtime
apps/server   →  gets the connection, calls createDb, exports db  // knows its runtime
```

Why this matters:

- **Scripts, tests and tools can import the db package without starting the whole app.**
  - compile-x's `@compile-x/db` imports `@compile-x/env/server`, which checks *every* server variable (`BETTER_AUTH_SECRET`, `JUDGE_URL`, the LLM settings, …).
  - So a seed script that only needs the database still fails without all of them.
- **The same package can serve Node and Workers.**
  - Anything that imports `cloudflare:workers` crashes in plain Node:
    ```
    ERR_UNSUPPORTED_ESM_URL_SCHEME ... Received protocol 'cloudflare:'
    ```
- **Each app chooses its own binding name and its own pool size.**

If you really want a ready-made `db` export in the package, put it behind a **subpath** that only that runtime imports (e.g. `@my-better-t-app-v2/db/worker`) and keep the root export free of runtime imports.

---

## 5. Setup A: Node + Postgres (direct)

**Use when:** a long-running Node server with 1–3 instances, and Postgres on the same network or a managed Postgres in the same region.

### Code

```ts
// packages/db/src/index.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool, type PoolConfig } from "pg";
import * as schema from "./schema";

export function createPool(connectionString: string, options: PoolConfig = {}) {
  return new Pool({
    connectionString,
    max: 10,                    // per process
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 5_000,
    ...options,
  });
}

export function createDb(pool: Pool) {
  return drizzle({ client: pool, schema });
}

export type Database = ReturnType<typeof createDb>;
export * from "./schema";
```

```ts
// apps/server/src/db.ts  (the app owns env and the singleton)
import { createDb, createPool } from "@compile-x/db";
import { env } from "@compile-x/env/server";

export const pool = createPool(env.DATABASE_URL);
export const db = createDb(pool);

// close connections cleanly on deploy or restart
for (const signal of ["SIGTERM", "SIGINT"] as const) {
  process.on(signal, async () => {
    await pool.end();
    process.exit(0);
  });
}
```

```ts
// usage anywhere in the server
import { db } from "./db";
const users = await db.query.user.findMany();
```

### Best practice

- **Use one `Pool` per process, created at startup.** Never create a pool per request.
- **Size `max` on purpose:** `processes × max` must stay below Postgres's `max_connections` (100 by default), with room left for migrations and admin tools. See [section 13](#13-connection-math).
- **Use TLS** for anything that crosses a network: `?sslmode=require` in the URL, or `ssl: { rejectUnauthorized: true }`.
- **Set timeouts** so a database outage fails fast instead of hanging requests.
- `pg-native` is optional; Drizzle says it's about 10% faster.
- If the server runs under PM2 cluster mode or as several containers, each process has its own pool. Multiply accordingly.

---

## 6. Setup B: Node + PgBouncer

**Use when:**
- you run many Node processes (autoscaling, several services, cron workers), or
- you use Node serverless (Lambda, Vercel), or
- your provider gives you a "pooled" connection string (Supabase, Neon, DigitalOcean and others run PgBouncer or similar).

PgBouncer keeps a small number of real Postgres connections and shares them among many client connections. In **transaction mode** (the usual setting), a client holds a real connection only for the length of one transaction.

```
app processes (many) ──► PgBouncer :6432 (transaction mode) ──► Postgres :5432 (few real connections)
drizzle-kit / admin ─────────────────────────────────────────► Postgres :5432 (direct)
```

### Code

```ts
// apps/server/src/db.ts
import { createDb, createPool } from "@compile-x/db";
import { env } from "@compile-x/env/server";

// DATABASE_URL points at PgBouncer
export const pool = createPool(env.DATABASE_URL, { max: 10 });
export const db = createDb(pool);
```

With `postgres.js` instead of `pg`:

```ts
import postgres from "postgres";
import { drizzle } from "drizzle-orm/postgres-js";

const sql = postgres(env.DATABASE_URL, {
  prepare: false, // required with transaction pooling (unless PgBouncer >= 1.21 with max_prepared_statements)
  max: 10,
});
export const db = drizzle({ client: sql, schema });
```

```ts
// packages/db/drizzle.config.ts: migrations never go through the pooler
export default defineConfig({
  schema: "./src/schema",
  out: "./src/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DIRECT_DATABASE_URL! },
});
```

### Best practice

- **Use two URLs:**
  - `DATABASE_URL` points at the pooler (e.g. port 6432, or your provider's "pooled" URL). The app uses it.
  - `DIRECT_DATABASE_URL` points straight at Postgres (port 5432). drizzle-kit, migrations and admin tools use it.
- **Prepared statements:**
  - `node-postgres` uses unnamed statements for normal parameterized queries, so it works out of the box.
  - `postgres.js` uses named prepared statements by default, so set `prepare: false`.
  - The exception is PgBouncer ≥ 1.21 with `max_prepared_statements` set, which supports them.
- **Avoid session state.** In transaction mode, each transaction may run on a different real connection:
  - ❌ `SET search_path` / `SET timezone` outside a transaction
  - ❌ session-level advisory locks (`pg_advisory_lock`); use `pg_advisory_xact_lock` inside a transaction instead
  - ❌ `LISTEN/NOTIFY`
  - ❌ temp tables used across queries
  - ❌ `WITH HOLD` cursors
- **Keep transactions short.** A transaction holds a real connection for its whole duration.
- **Keep the app pool small.** PgBouncer does the multiplexing, so each process only needs enough connections for its own concurrent queries.
- **Size PgBouncer** with `default_pool_size` (real connections per database/user pair) and `max_client_conn` (how many app connections it accepts).

### Minimal `pgbouncer.ini` (self-hosted on the VPS)

```ini
[databases]
app = host=127.0.0.1 port=5432 dbname=app

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
default_pool_size = 20
max_client_conn = 500
max_prepared_statements = 100   ; PgBouncer >= 1.21, enables prepared statements in transaction mode
server_tls_sslmode = prefer
```

---

## 7. Setup C: Workers + D1 (this repo)

**Use when:** a Cloudflare Workers app whose data fits SQLite: relational, moderate size, no Postgres-specific features.

### Infra

```ts
// packages/infra/alchemy.run.ts
export const db = Cloudflare.D1.Database("database", {
  migrations: "../../packages/db/src/migrations", // Alchemy applies these on dev and deploy
});

export const server = Cloudflare.Worker("server", {
  main: "../../apps/server/src/index.ts",
  compatibility: { flags: ["nodejs_compat"] },
  env: { DB: db, CORS_ORIGIN: Config.String("CORS_ORIGIN") },
});

export type ServerEnv = Cloudflare.InferEnv<typeof server>; // gives env.DB the type D1Database
```

### Code

```ts
// packages/db/src/config.ts
/// <reference types="@cloudflare/workers-types" />
export type DatabaseConfig = { DB: D1Database }; // D1Database is a global type from @cloudflare/workers-types

// packages/db/src/index.ts
import { drizzle } from "drizzle-orm/d1";
import type { DatabaseConfig } from "./config";
import { relations } from "./relations";

export function createDb(env: DatabaseConfig) {
  return drizzle(env.DB, { relations });
}
export type Database = ReturnType<typeof createDb>;
export * from "./schema";
```

```ts
// apps/server/src/env.server.ts
export { env as ENV } from "cloudflare:workers";

// apps/server/src/services.ts
import { createDb } from "@my-better-t-app-v2/db";
import { ENV } from "./env.server";

export const getDb = () => createDb(ENV);   // free wrapper; per call is fine
// or: export const db = createDb(ENV);     // also fine on Workers; it does no I/O
```

### Best practice

- Keep `packages/db` free of `cloudflare:workers` imports. Only the server reads `env`.
- **Let Alchemy own migrations.**
  - Write migrations with `pnpm db:generate`. They are applied on `pnpm dev` (local) and `pnpm deploy` (remote).
  - **Don't** also run `drizzle-kit migrate` or `push`. Alchemy tracks applied migrations in `__alchemy_migrations` and would try to run them again.
- **Atomic multi-write:** use `db.batch([q1, q2, ...])`. D1 doesn't support interactive `BEGIN … COMMIT`.
- **Look at the data locally** with `pnpm db:studio`, which reads the local D1 SQLite file under `packages/infra/.alchemy/local/d1/`.
- **Add `@cloudflare/workers-types` to `packages/db` devDependencies.** Today `D1Database` resolves only because the root `package.json` has the package.
- **Mind D1's limits.**
  - Each database is a single SQLite instance with size and per-query limits, so check the current D1 limits before committing to it for large data.
  - It suits one database per app or per tenant.

---

## 8. Setup D: Workers + Hyperdrive → Postgres

**Use when:** a Cloudflare Workers app needs a real Postgres.

**Why not connect directly?** A Worker *can* open TCP to Postgres (`pg` with `nodejs_compat`). But each request then pays several network round trips for TCP, TLS and Postgres authentication, from a location that may be far from the database. Hyperdrive keeps a warm pool of connections near your database and caches read queries.

```
Worker (any location) ──► Hyperdrive (Cloudflare, pooled, transaction mode, read cache) ──► Postgres
drizzle-kit / CI ──────────────────────────────────────────────────────────────────────────► Postgres (direct)
```

### Infra (Alchemy has a resource for this)

```ts
// packages/infra/alchemy.run.ts
import * as Cloudflare from "@alchemy.run/cloudflare";
import * as Config from "effect/Config";

export const pg = Cloudflare.Hyperdrive.Connection("postgres", {
  origin: {
    scheme: "postgres",
    host: "db.example.com",        // placeholder
    port: 5432,
    database: "app",
    user: "app",
    password: Config.Redacted("DB_PASSWORD"),   // from .env through varlock; never hard-code it
  },
  caching: { maxAge: 60, staleWhileRevalidate: 15 },  // defaults; read queries are cached
  originConnectionLimit: 20,                           // soft limit on connections to Postgres
  dev: {                                               // used by `alchemy dev`
    scheme: "postgres",
    host: "localhost",
    port: 5432,
    database: "app",
    user: "postgres",
    password: Config.Redacted("LOCAL_DB_PASSWORD"),
    sslmode: "disable",
  },
});

export const server = Cloudflare.Worker("server", {
  main: "../../apps/server/src/index.ts",
  compatibility: { flags: ["nodejs_compat"] },   // required by the pg driver
  env: { HYPERDRIVE: pg, CORS_ORIGIN: Config.String("CORS_ORIGIN") },
});

export type ServerEnv = Cloudflare.InferEnv<typeof server>; // gives env.HYPERDRIVE the type Hyperdrive
```

> In Alchemy's own examples, `Config.Redacted(...)` is written as `yield* Config.Redacted(...)` inside an `Effect.gen` stack. Use whichever style your `alchemy.run.ts` uses.
>
> **Private databases:** Alchemy also supports a database in a private network, e.g. RDS or Aurora in an AWS VPC. It's reached through Workers VPC: a Tunnel plus a TCP VPC Service, then `origin: { serviceId, ... }`. See `Cloudflare.Hyperdrive.Connection`'s doc comment in `node_modules/@alchemy.run/cloudflare/src/Hyperdrive/Connection.ts`.

### Code

```ts
// packages/db/src/index.ts  (runtime-agnostic: receives a pg client)
import { drizzle } from "drizzle-orm/node-postgres";
import type { Client, Pool } from "pg";
import { relations } from "./relations";

export const createDb = (client: Client | Pool) => drizzle({ client, relations });
export type Database = ReturnType<typeof createDb>;
export * from "./schema";
```

```ts
// apps/server/src/db.ts: one connection per request
import { createDb, type Database } from "@my-better-t-app-v2/db";
import { createMiddleware } from "hono/factory";
import { Client } from "pg";
import { ENV } from "./env.server";

export const withDb = createMiddleware<{ Variables: { db: Database } }>(async (c, next) => {
  const client = new Client({ connectionString: ENV.HYPERDRIVE.connectionString });
  await client.connect();
  c.set("db", createDb(client));
  try {
    await next();
  } finally {
    c.executionCtx.waitUntil(client.end()); // close after the response, without delaying it
  }
});
```

```ts
// apps/server/src/index.ts
app.use("/users/*", withDb);   // only routes that need the DB
app.get("/users", async (c) => c.json(await c.var.db.query.user.findMany()));
```

### Best practice

- **Create a new `Client` per request.** Cloudflare's guide says to "create a new client instance for each request". A module-level client or pool breaks because Workers can't share an open connection between requests. Opening one is cheap because Hyperdrive holds the real pool.
- **Use `node-postgres` (`pg`) with Drizzle.** Cloudflare: *"Pairing Drizzle ORM with the Postgres.js driver over Hyperdrive is not currently supported."*
- **Prepared statements:** Hyperdrive supports named prepared statements from `pg` and `postgres.js`, so no `prepare: false` is needed.
- **Watch the read cache. It's on by default.**
  - A read right after a write can return stale data for up to `maxAge` seconds.
  - For pages that must always be fresh, set `caching: { disabled: true }`.
  - Or create a **second** Hyperdrive config with caching off, bind it as `HYPERDRIVE_NOCACHE`, and use it for read-after-write paths.
- **Transaction mode applies here, like PgBouncer:**
  - `SET` lasts only for that query or transaction.
  - Session-level advisory locks aren't supported.
  - Keep transactions short. Cloudflare warns long ones hurt Hyperdrive's scaling.
- **Set `originConnectionLimit`** so all Hyperdrive configs plus every other client stay within Postgres's `max_connections`.
- **Migrations:** run `drizzle-kit migrate` with a **direct** URL (`DIRECT_DATABASE_URL`) from your machine or CI, never through Hyperdrive. Alchemy's automatic migrations are a D1 feature and don't apply here.
- **Region:** put Postgres in one region close to most of your users. Hyperdrive reduces connection cost, but each query still travels to the database's region (unless it's a cache hit).

---

## 9. Hyperdrive and PgBouncer together?

**Usually not needed.** Hyperdrive *is* a transaction-mode pooler, so Workers → Hyperdrive → Postgres is enough.

Add PgBouncer behind Hyperdrive only when **other clients share the same Postgres** (a Node API on a VPS, cron jobs, BI tools) and you want **one place** that limits total connections:

```
Workers ──► Hyperdrive ──┐
Node API (VPS) ──────────┼──► PgBouncer :6432 ──► Postgres :5432
Cron / workers ──────────┘
drizzle-kit (direct) ───────────────────────────► Postgres :5432
```

If you do:

- Point Hyperdrive's `origin.port` at PgBouncer (e.g. 6432).
- Keep PgBouncer in transaction mode with `max_prepared_statements` set (≥ 1.21), so prepared statements keep working through both layers.
- Budget: `Hyperdrive originConnectionLimit` + the Node pools must fit within PgBouncer's `max_client_conn`, and PgBouncer's `default_pool_size` must fit within Postgres's `max_connections`.
- Migrations still go **direct** to Postgres.

If only Workers use the database, point Hyperdrive straight at Postgres. An extra pooler adds one more network hop and one more thing to run.

---

## 10. Other options worth knowing

| Option | Runtime | When it makes sense |
|---|---|---|
| **Provider HTTP/WebSocket drivers** (e.g. Neon's serverless driver, `drizzle-orm/neon-http`) | Workers, Node serverless | Talks to Postgres over HTTP or WebSockets instead of TCP. An alternative to Hyperdrive when the provider supports it. Check whether transactions are supported for the mode you pick. |
| **Provider pooled URLs** (Supabase, Neon, DigitalOcean "pooled" connection strings) | Node, Node serverless | PgBouncer run for you. Follow Setup B: pooled URL for the app, direct URL for migrations. |
| **libSQL / Turso** (`drizzle-orm/libsql`) | Both Workers and Node | SQLite reached over HTTP, so it works the same way in both runtimes. The earlier `my-better-t-app` used this. |
| **`better-sqlite3`** (`drizzle-orm/better-sqlite3`) | Node on one VPS | A SQLite file on local disk: fastest and simplest. Only works for **one** process on **one** machine. Back up the file. |
| **D1 from Node** (D1 HTTP API) | Node | Possible, and drizzle-kit uses it for `studio`/`migrate` with `driver: "d1-http"`. Too slow and rate-limited to serve app traffic from Node, so use it only for tooling. |
| **AWS RDS Proxy** | Node or Lambda on AWS | AWS's managed alternative to PgBouncer. Same transaction-pooling rules apply. |

---

## 11. Environment variables: t3-env vs varlock

| | **t3-env** (`compile-x`) | **varlock** (`my-better-t-app-v2`) |
|---|---|---|
| Schema | zod in TypeScript (`packages/env/src/server.ts`) | `.env.schema` with `# @` comments |
| When values are checked | **App startup**, inside the process, when the env module is imported | **Dev/deploy time**, when `varlock/auto-load` runs (in `alchemy.run.ts` and `drizzle.config.ts`) |
| Converting values | Any zod transform (split lists, numbers, defaults) | Types only (`url`, `enum`, `number`…); no custom logic |
| Typed access | `env.X` | Generated `src/env.ts`. On Workers, code reads `env` from `cloudflare:workers` instead. |
| Sharing between packages | One `@compile-x/env` package | `@import(../../apps/server/, pick=[...])` between schema files |
| Marking secrets | Not built in | `@sensitive` / `@public` |
| Fits | Node (`process.env`) | Workers, where values become **bindings** through Alchemy |

**How each runtime gets its database settings:**

- **Node (t3-env or varlock):** `DATABASE_URL` is a string in `process.env`, checked at startup, and passed to `createPool(...)`.
- **Workers (varlock + Alchemy):** varlock checks `.env` when `alchemy.run.ts` loads. Alchemy turns values into bindings (`Config.String`, `Config.Redacted`), and D1/Hyperdrive become binding objects. The worker reads them from `cloudflare:workers`. **No env library can produce `env.DB` or `env.HYPERDRIVE`**: they only exist inside the Workers runtime.
- **To make the worker itself reject bad values at startup** (like t3-env does), add a small zod `parse` of `env` in `apps/server/src/env.server.ts`.

**Where secrets live:**

| Secret | Node (VPS) | Workers |
|---|---|---|
| App database URL | `DATABASE_URL` in the server env (orchestrator secrets, not committed `.env`) | Not needed (binding) |
| Direct URL for migrations | `DIRECT_DATABASE_URL`, only on the machine or CI that runs migrations | Same |
| Postgres password for Hyperdrive | n/a | `DB_PASSWORD` in `.env` through varlock → `Config.Redacted` → stored in the Hyperdrive config; the worker never sees it |
| Production values | A secrets manager (AWS Secrets Manager, Azure Key Vault) or your orchestrator's secrets | CI secrets that feed `alchemy deploy` |

---

## 12. Migrations per setup

| Setup | Generate | Apply | Tracking table |
|---|---|---|---|
| A. Node + Postgres | `drizzle-kit generate` | `drizzle-kit migrate` (`DATABASE_URL`) | `__drizzle_migrations` |
| B. Node + PgBouncer | `drizzle-kit generate` | `drizzle-kit migrate` with **`DIRECT_DATABASE_URL`** | `__drizzle_migrations` |
| C. Workers + D1 | `pnpm db:generate` | **Alchemy**, automatically on `pnpm dev` / `pnpm deploy` | `__alchemy_migrations` |
| D. Workers + Hyperdrive | `drizzle-kit generate` | `drizzle-kit migrate` with **`DIRECT_DATABASE_URL`**, from CI before `alchemy deploy` | `__drizzle_migrations` |

Rules:

- **Use one tool per database.** Never mix Alchemy's D1 migrations with `drizzle-kit migrate`/`push`, or two tools will each think they own the history.
- **Never migrate through a transaction-mode pooler** (PgBouncer or Hyperdrive). Migrations need session behaviour and may hold long locks.
- **`drizzle-kit push`** is for quick prototyping against a throwaway database only.
- **Deploy order for Postgres:** migrate first, then deploy code that works with both the old and new schema. Avoid dropping or renaming a column in the same release that stops using it.

---

## 13. Connection math

Postgres's `max_connections` defaults to **100**. Each connection costs memory on the database server.

**Setup A (Node direct):**
```
total = app processes × pool max  +  headroom (migrations, admin, monitoring ≈ 5–10)
e.g. 3 containers × max 10 = 30  (+10 headroom) = 40   ✅
     20 containers × max 10 = 200                       ❌ → move to Setup B
```

**Setup B (PgBouncer):**
```
Postgres connections ≈ default_pool_size (per db/user pair) + headroom
App side: processes × pool max must fit within max_client_conn (cheap, thousands are fine)
e.g. 20 containers × max 10 = 200 client conns → PgBouncer → 20 real connections   ✅
```

**Setup D (Hyperdrive):**
```
Postgres connections ≈ sum of originConnectionLimit for every Hyperdrive config on that database
                     + any Node pools / PgBouncer for other clients + headroom
```

**Setup C (D1):** no connection budget to manage.

---

## 14. Local development per setup

| Setup | Local database | How |
|---|---|---|
| A / B | Postgres in Docker | `docker run -e POSTGRES_PASSWORD=… -p 5432:5432 postgres:17`. Skip PgBouncer locally unless you're testing pooling. |
| C (this repo) | SQLite file managed by Alchemy | `pnpm dev`. Browse it with `pnpm db:studio`. |
| D | Postgres in Docker | Alchemy's `dev:` origin on the Hyperdrive resource points `alchemy dev` at `localhost:5432`. |

Keep local passwords in an untracked `.env` (declared in `.env.schema` with `@sensitive`), even for local-only databases.

---

## 15. Gotchas checklist

**All setups**
- [ ] `packages/db` exports schema + `createDb(connection)` and doesn't import env or runtime-only modules
- [ ] Migrations use one tool and a **direct** connection
- [ ] TLS is on for any database reached over a network
- [ ] Secrets are in a secrets manager or untracked `.env`, never in code or committed files

**Node (VPS)**
- [ ] One `Pool` per process, created at startup, never per request
- [ ] `processes × max` + headroom < `max_connections`
- [ ] `pool.end()` runs on SIGTERM
- [ ] With PgBouncer: no session state; `prepare: false` for postgres.js; two URLs

**Cloudflare Workers**
- [ ] No Postgres client or pool at module level; create a `Client` per request and `ctx.waitUntil(client.end())`
- [ ] `nodejs_compat` flag is on (needed by `pg`)
- [ ] Drizzle uses `node-postgres` (not `postgres.js`) over Hyperdrive
- [ ] You know the Hyperdrive read cache is on, and disable it (or use a second config) where reads must be fresh
- [ ] D1: `db.batch()` for atomic writes; Alchemy owns migrations
- [ ] `@cloudflare/workers-types` is in the db package's devDependencies (for `D1Database` / `Hyperdrive` types)

---

## 16. Appendix: how `env.DB` reaches the code in this repo

| | Source |
|---|---|
| **Value** (`ENV.DB` at runtime) | `cloudflare:workers`, filled in by the Workers runtime from Alchemy's `env: { DB: db }` binding |
| **Type** (`ENV.DB: D1Database`) | `apps/server/cloudflare-env.d.ts` → `Cloudflare.Env extends ServerEnv` → `Cloudflare.InferEnv<typeof server>` in `alchemy.run.ts` |
| **`D1Database` itself** | A global type (`declare abstract class D1Database`) in `@cloudflare/workers-types`. It's loaded by `/// <reference types="@cloudflare/workers-types" />` in `packages/db/src/config.ts` and by `"types": ["node", "@cloudflare/workers-types"]` in `packages/config/tsconfig.base.json`. It's compiled away, so nothing ships at runtime. |

```
.env ──varlock/auto-load──▶ alchemy.run.ts ── env: { DB: db } ──▶ Worker binding
                                                                      │
apps/server/src/env.server.ts:  export { env as ENV } from "cloudflare:workers"
                                                                      │
apps/server/src/services.ts:    createDb(ENV)  →  drizzle(ENV.DB, { relations })
```

> The comment in `apps/server/src/env.server.ts` says types live in `env.d.ts`. The file is actually `apps/server/cloudflare-env.d.ts`.

---

> **Before sharing this outside the team:** check that no internal credentials, PII or infrastructure details have been added. The examples use placeholder hosts (`db.example.com`) and read passwords from env.
