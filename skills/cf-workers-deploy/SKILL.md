---
name: cf-workers-deploy
description: Use when creating, configuring, deploying, or debugging a Cloudflare Worker with wrangler — wrangler.jsonc, compatibility_date, nodejs_compat, environments, secrets, builds, observability, Pages deploys.
---

# cf-workers-deploy (wrangler build/deploy/triage, Cloudflare-only)

## Overview

Ship Workers via wrangler and triage via MCP. Retrieval-first: check docs MCP + `node_modules/wrangler/config-schema.json` before citing flags or config fields.

## When to Use

- New Worker, Pages project, environment (staging/production), secret, deploy, build failure, live error.
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

Flow: `wrangler dev` → `wrangler types` (generates `Env`, never hand-write) → `wrangler deploy`. Secrets: `wrangler secret put NAME` (never in config/source). Pages production: always `--branch main`. Startup check: `wrangler check startup`. CI: `wrangler-action` with the API token in GitHub Secrets (never in workflow YAML), deploying from `main`. Pre-deploy for auth/frontdoor-adjacent Workers, offer a `security-audit` pass (Cloudflare's audit skill: `npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --global`) — offer, don't mandate: it runs parallel hunter/verifier subagents and is token-heavy by design.

Triage: failed builds → `cloudflare-builds` MCP (list by worker ID, get build + logs by UUID). Live errors → `cloudflare-observability` MCP (confirm keys via keys/values endpoints before filtering; `$metadata.service`, `$metadata.message`, `$metadata.error` first).

## Quick Reference

| Task | Command |
|---|---|
| Local dev | `wrangler dev` |
| Type generation | `wrangler types` |
| Deploy | `wrangler deploy` |
| Tail logs | `wrangler tail` |
| Secret | `wrangler secret put NAME` |

## Common Mistakes

- TOML config for JSON-only features; stale `compatibility_date`; missing `nodejs_compat`.
- `compatibility_date` newer than the bundled workerd's max makes `wrangler dev` fail to start — pin it at or below what the local runtime supports.
- Hand-written `Env` instead of `wrangler types`; secrets in source.
- Trusting first API page (paginate via `result_info.total_pages`); using default-account MCP OAuth for other accounts (wire each account's own auth).

## Reuses

`wrangler`, `workers-best-practices`, `cloudflare` (`workers/`, `pages/`, `observability/` refs), `cloudflare-builds`, `cloudflare-observability`.
