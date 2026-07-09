# nexxus-worker-writer

> Executable Writer worker for a Nexxus deployment — the process that turns queued write requests into actual database writes and fans out change events to the rest of the pipeline.

[![License: MPL 2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)](https://opensource.org/licenses/MPL-2.0)
[![Node.js](https://img.shields.io/badge/node-%3E%3D24.0.0-brightgreen.svg)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-6.0.0-blue.svg)](https://www.typescriptlang.org/)

---

## What this is

`nexxus-worker-writer` is the **runnable Writer worker process** for a Nexxus deployment. It's a thin bootstrap around [`@mayhem93/nexxus-worker-lib`](https://www.npmjs.com/package/@mayhem93/nexxus-worker-lib): reads a config file, resolves the pluggable services (logger, database adapter, message-queue adapter), and starts consuming the `writer` queue.

If `nexxus-api` is the front door of a Nexxus deployment, this is one of the back-of-house workers that makes real-time synchronization actually work. Same config shape, same pluggable-service pattern, same graceful-shutdown lifecycle — just a different process at a different point in the pipeline.

---

## What it does

The Writer worker sits between the API and the database. Its job is a straight three-step loop:

1. **Consume** the `writer` queue — every app-model write the API accepts is published here.
2. **Apply** the mutation to the database via the configured DB adapter (Elasticsearch by default).
3. **Publish** a `model_created` / `model_updated` / `model_deleted` event to the `transport-manager` queue so downstream workers can push the change out to subscribed clients.

End-to-end:

```
Client HTTP  →  nexxus-api  →  [writer queue]  →  nexxus-worker-writer  →  DB
                                                            ↓
                                              [transport-manager queue]  →  transport workers  →  subscribed clients
```

---

## Why a Writer worker instead of the API writing directly

Having the API write to the database in-line seems simpler on the surface. It isn't, for a few concrete reasons:

- **Decoupled response time.** The API can accept a write and return `202 Accepted` in milliseconds regardless of how loaded the database is. Clients aren't blocked on a slow index refresh, a cluster rebalance, or a hot write path.
- **Backpressure lives in the queue, not in HTTP.** If the DB gets slow, writes queue up rather than surfacing as timeouts on the API. You get natural back-pressure with observable metrics (queue depth) instead of user-facing failures.
- **Real-time fan-out is a separate concern.** Every write also needs to notify every subscribed client. Doing that in the request-response cycle bloats latency and couples two responsibilities that scale very differently. Splitting write and notify into two workers lets each scale on its own.
- **Horizontal scale.** Multiple `nexxus-worker-writer` instances can consume from the same `writer` queue — write throughput becomes independent of how many API instances you have.
- **Delivery semantics.** Message queues give you at-least-once delivery with retries and dead-letter handling for free. HTTP request handling doesn't.

The trade-off is *consistency semantics*: writes are async from the client's perspective. If a client immediately re-reads a resource it just wrote, it may not be there yet. The client SDK handles this via subscriptions — the "write landed" signal is the `model_created` event on your channel, not the HTTP response.

---

## What goes through the Writer worker

**App-model CRUD.** Every type declared in an application's schema — `task`, `message`, `vehicle`, `poll_vote`, whatever the app defines — routes its writes through this worker.

- `POST /model/:type`         → publishes to `writer` queue → this worker calls `db.createItems(...)` → publishes `model_created`.
- `PATCH /model/:type/:id`    → same path with `db.updateItems(...)` (JSON-patch validated against the app's model schema before applying).
- `DELETE /model/:type/:id`   → same path with `db.deleteItems(...)`.

## What does NOT go through the Writer worker

- **Built-in model writes** — `Application` (creating an app), `User` (registration, profile update), and admin models. These are administrative, low-volume, and the client needs the resulting id / auth token in the response, so the API writes them directly to the DB.
- **Authentication** — login, JWT issuance, OAuth callbacks. All inside the API's request-response cycle.
- **Subscriptions and devices** — those live in Redis, not the DB. The API and transport workers manage them directly. No `writer`-queue involvement.
- **Reads of any kind** — GET / list / search bypass this worker entirely. The API reads from the DB directly.

Rule of thumb: **if it's an app-model write, it comes through here. Everything else doesn't.**

---

## Configuration

Same overall shape as the API's config file, minus the HTTP-server fields. The bootstrap-time keys you care about:

```json
{
  "database": { "host": "localhost", "port": 9200 },
  "message_queue": { "host": "localhost", "port": 5672, "user": "guest", "password": "guest" },
  "redis": { "host": "localhost", "port": 6379, "cluster": false, "password": "1234test" },
  "app": {
    "name": "my-writer",
    "logger": "WinstonNexxusLogger",
    "database": "NexxusElasticsearchDb",
    "message_queue": "NexxusRabbitMq",
    "management": { "port": 5001, "token": "replace-me" },
    "hub": { "endpoint": "http://hub:8080", "token": "replace-me" }
  },
  "logger": { "level": "info", "logType": "json", "transports": [ { "type": "stdout" } ] }
}
```

- `app.logger` / `app.database` / `app.message_queue` — class names of the adapters this deployment uses. Built-ins are recognised by name; any other string is treated as an npm package name and dynamic-imported.
- `app.management` — every node runs a small HTTP server for observability (`/stats`, bearer-auth). See `NexxusManagementServer` in `nexxus-core-lib`.
- `app.hub` — optional. When present, the worker registers itself with the Nexxus Hub on startup and de-registers on shutdown. Omit for standalone dev.

Config file lookup order (first hit wins):

1. Explicit path passed to `NexxusConfigManager`
2. `NXX_CONF_PATH` environment variable
3. `/etc/nexxus/nexxus.conf.json` (the default)

---

## Running

**Prerequisites:**

- Node.js ≥ 24
- The same infrastructure the API talks to: Elasticsearch (or whichever DB adapter is configured), RabbitMQ (or whichever MQ adapter), Redis
- The `writer` and `transport-manager` queues existing on the broker (provisioned out-of-band — the worker doesn't create infrastructure)

**Build and start:**

```bash
npm install
npm run build
npm start
```

`npm start` runs `node --enable-source-maps dist/index.js`. Multiple instances against the same `writer` queue are safe — the broker load-balances between consumers.

---

## Status

🚧 **Pre-alpha.** APIs, config shape, and queue payloads may still shift alongside the underlying library.

---

## Related

- [`nexxus-lib`](https://github.com/Mayhem93/nexxus-lib) — the umbrella framework: config manager, base service, pluggable-service resolvers, worker framework
- [`@mayhem93/nexxus-worker-lib`](https://www.npmjs.com/package/@mayhem93/nexxus-worker-lib) — the actual worker framework this repo bootstraps
- [`@mayhem93/nexxus-core-lib`](https://www.npmjs.com/package/@mayhem93/nexxus-core-lib) — shared types, config manager, logger, Hub client

---

## License

MPL-2.0
