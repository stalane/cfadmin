---
name: cf-realtime
description: Use when building realtime, stateful, scheduled, or long-running work on Cloudflare — Durable Objects, Workflows, Containers, Agents SDK, Queues, Cron Triggers, WebSockets — or replacing socket.io servers and cron daemons.
---

# cf-realtime (DO/Workflows/Containers/Agents, Cloudflare-only)

## Overview

Stateful coordination lives in Durable Objects; multi-step jobs in Workflows; scheduled work in Cron Triggers; agent logic in Agents SDK. Audited baseline: Durable Objects (56 projects) · Cron (56) · Queues (18) · Workflows (8) · Containers (6).

## When to Use

- Chat, presence, multiplayer, per-entity state, WebSockets, background jobs, cron, coding-agent sandboxes.
- When NOT: plain request/response storage (→ `cf-data`), inference (→ `cf-ai`).

## Implementation

- **Per-entity state / WebSockets** → Durable Object (one SQLite-backed stub per room/user/device). Replaces socket.io servers, Redis pub/sub.
- **Long multi-step jobs** (retries, sleeps, human approval) → Workflows. Replaces Celery/RQ workers.
- **Sandboxed code execution** → Containers (`cloudbox`, `cloudsail` pattern). Replaces Docker-on-VPS. For running untrusted code (AI runners, judges, eval harnesses) prefer `sandbox-sdk` over Containers.
- **Stateful AI agents** → Agents SDK on top of DO (`forja`, `vibesdk` pattern); scaffold new ones from `agents-starter`.
- **Fire-and-forget background** → Queues + `ctx.waitUntil()` (never destructure `ctx`).
- **Schedules** → Cron Triggers in `wrangler.jsonc` (`crons: ["*/5 * * * *"]`), not a daemon.

Previews: DO auto-isolates per Preview (`ctx.exports` + empty `previews{}`; redeclare `env` bindings under `previews`). Containers auto-isolate but declare in both top-level and `previews.containers`; after `preview delete`, check `containers list` and delete leftover Preview apps. Workflows bind existing code (deploy a dedicated non-prod Workflow first); service bindings call prod; queue consumers/cron/routes never target Previews — put scheduled/queue work behind a test route.

```ts
export class Room implements DurableObject {
  constructor(private state: DurableObjectState, private env: Env) {}
  async fetch(req: Request) { /* websocketUpgrade or RPC */ return new Response("ok"); }
}
```

## Common Mistakes

- Global in-memory state across requests (isolates are disposable — put it in DO/KV/D1).
- Floating promises instead of `ctx.waitUntil()`.
- Cron daemon / PM2 / systemd timer instead of Cron Triggers.
- Containers for plain CRUD (use Workers + `cf-data`).

## Reuses

`durable-objects`, `agents-sdk`, `cloudflare` (`durable-objects/`, `workflows/`, `containers/`, `queues/`, `cron-triggers/` refs), `workers-best-practices` (waitUntil, global-state rules). Official starters: `workflows-starter`, `queues-web-crawler`.
