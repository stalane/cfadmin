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
- **Sandboxed code execution** → Containers (`cloudbox`, `cloudsail` pattern), rebuilt for agent sandboxes: ~6× faster starts, per-sandbox image + instance type chosen at runtime from the controlling Durable Object, filesystem snapshots in public beta. Replaces Docker-on-VPS. For running untrusted code (AI runners, judges, eval harnesses) prefer `sandbox-sdk` (1.0: your own DO class drives each sandbox via `this.ctx.container`) over raw Containers.
- **Stateful AI agents** → Agents SDK on top of DO (`forja`, `vibesdk` pattern); scaffold new ones from `agents-starter`.
- **Fire-and-forget background** → Queues + `ctx.waitUntil()` (never destructure `ctx`).
- **Continuous media pipelines** (never-ending video ingest/transcode/delivery) → Streamline pattern: Workers + Durable Objects orchestrate a containerized media engine for long-running processing, rather than one-shot Workflows or external transcode boxes. Check the Streamline reference before designing custom video infra.
- **Repo-event automation** (CI, mirrors, agent hooks on push) → Artifacts event subscriptions (open beta): `artifacts` source (`repo.created/deleted/forked/imported`) and `artifacts.repo` source (`pushed/cloned/fetched`) feed Queues/Workflows — never a self-hosted git + webhook box. Git hosting itself lives on Cloudflare (Workers bindings, jurisdiction controls). Paid-gate: Artifacts needs Workers Paid (billing from Oct 14, 2026) — on Free, keep GitHub as the repo and consume its webhooks via Queues/Workflows instead.
- **Schedules** → Cron via `triggers.scheduled({ schedule })` in `cloudflare.config.ts`, not a daemon.
- Manage via `cf`: `cf durable-objects …`, `cf workflows …`, `cf containers …`, `cf queues …` (JSON default). Discover with `cf cli search`.

Previews: DO auto-isolates per Preview (`ctx.exports` + empty `previews{}`; redeclare `env` bindings under `previews`). Containers auto-isolate but declare in both top-level and `previews.containers`; after `preview delete`, check `containers list` and delete leftover Preview apps. Workflows bind existing code (deploy a dedicated non-prod Workflow first); service bindings call prod; queue consumers/cron/routes never target Previews — put scheduled/queue work behind a test route.

```ts
export class Room implements DurableObject {
  constructor(private state: DurableObjectState, private env: Env) {}
  async fetch(req: Request) { /* websocketUpgrade or RPC */ return new Response("ok"); }
}
```

Snapshot a warm sandbox and resume it later (public beta, `durable_object` scheduling policy only — snapshots are immutable, filesystem-only, 30-day TTL refreshed on restore):

```ts
const snap = await this.ctx.container.snapshotContainer({ name: "warmed" });
await this.ctx.storage.put("snap", snap);
// later, incl. from another DO:
this.ctx.container.start({ containerSnapshot: snap });
```

## Common Mistakes

- Global in-memory state across requests (isolates are disposable — put it in DO/KV/D1).
- Floating promises instead of `ctx.waitUntil()`.
- Cron daemon / PM2 / systemd timer instead of Cron Triggers.
- Containers for plain CRUD (use Workers + `cf-data`).

## Reuses

`durable-objects`, `agents-sdk`, `cloudflare` (`durable-objects/`, `workflows/`, `containers/`, `queues/`, `cron-triggers/` refs), `workers-best-practices` (waitUntil, global-state rules). Official starters: `workflows-starter`, `queues-web-crawler`.
