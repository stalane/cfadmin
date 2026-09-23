---
name: cf-workers-deploy
description: Use when creating, configuring, deploying, or debugging a Cloudflare Worker with wrangler — wrangler.jsonc, compatibility_date, nodejs_compat, environments, secrets, builds, observability, Pages deploys, branch previews.
---

# cf-workers-deploy (wrangler build/deploy/triage, Cloudflare-only)

## Overview

Ship via wrangler, triage via MCP. Retrieval-first: check docs MCP + config schema before citing flags/fields.

## When to Use

- New Worker, Pages project, environment (staging/production), secret, deploy, build failure, live error, branch/PR preview.
- When NOT: data-model or realtime-design questions — use `cf-data` / `cf-realtime`.

## Implementation

Prefer `wrangler.jsonc` (newer features are JSON-only). Minimal config:

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2026-09-18",
  "compatibility_flags": ["nodejs_compat"],
  "observability": { "enabled": true }
}
```

Flow: `wrangler dev` → `wrangler types` (generates `Env`, never hand-write) → `wrangler deploy`. Secrets via `wrangler secret put` (never in config). Pages production: `--branch main`. CI: `wrangler-action`, token in GitHub Secrets. Audit gate (first upload, no exceptions): `security-audit` quick pass BEFORE deploy; fix High/Critical first. Abuse gate (every deploy): paid/external-cost route → refuse until Turnstile siteverify verified live (tokenless POST → 403, browser flow → 200).

Triage: builds → `cloudflare-builds` MCP; live errors → `cloudflare-observability` MCP (`$metadata.service/message/error` first).

Python Workers GA: FastAPI/Django/Flask via `workers.asgi`/`wsgi`.

## Previews (branch/PR isolation, Wrangler 4.135+)

`npx wrangler preview` creates/updates the branch Preview under the same Worker; `deploy` stays production. Top-level = production, `previews` block = Preview settings (never inherits). Preview URL = latest, Deployment URL = pinned. Protect with Access; PR URLs via Workers Builds; delete via `preview delete --name`.

```jsonc
{ "vars": { "ENVIRONMENT": "production" }, "previews": { "vars": { "ENVIRONMENT": "preview" } } }
```

DO/Containers auto-isolate (`ctx.exports` + empty `previews{}`; redeclare `env` bindings under `previews`). KV/D1/R2/Queues/Vectorize/Hyperdrive share unless rebound — see `cf-data`. Workflows/service-bindings/consumers/cron/routes stay on prod — see `cf-realtime`. Limits: 100 (free)/500 (paid) Previews, 100 deploys each, oldest auto-deleted. Version URLs use prod resources (not for branches); `preview --env staging` nests under `env.staging`.

## Quick Reference

| Task | Command |
|---|---|
| Local dev | `wrangler dev` |
| Type generation | `wrangler types` |
| Deploy | `wrangler deploy` |
| Preview branch | `wrangler preview [--name X] [--env staging]` |
| Delete preview | `wrangler preview delete --name X` |
| Tail logs | `wrangler tail` |
| Secret | `wrangler secret put NAME` |

## Common Mistakes

- TOML config for JSON-only features; stale `compatibility_date`; missing `nodejs_compat`.
- `compatibility_date` newer than bundled workerd breaks `dev` — pin at/below runtime max.
- Hand-written `Env` instead of `wrangler types`; secrets in source.
- Trusting first API page (paginate via `result_info.total_pages`); using default-account MCP OAuth for other accounts (wire each account's own auth).
- Preview without redeclaring `env.*` bindings under `previews` → 1101; Preview pointed at prod D1/KV/R2.

## Reuses

`wrangler`, `workers-best-practices`, `cloudflare` (`workers/`, `pages/`, `observability/` refs), `cloudflare-builds`, `cloudflare-observability`.
