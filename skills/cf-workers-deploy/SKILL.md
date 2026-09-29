---
name: cf-workers-deploy
description: Use when creating, configuring, deploying, or debugging a Cloudflare Worker with cf CLI — cloudflare.config.ts, bindings/triggers helpers, cf init/dev/deploy/migrate, Vite builds, secrets, previews, Pages deploys.
---

# cf-workers-deploy (cf CLI build/deploy/triage, Cloudflare-only)

## Overview

Ship via `cf`, triage via MCP. `cf` (open beta, `npm i -g cf`) covers the whole API as JSON; `cloudflare.config.ts` replaces `wrangler.jsonc` for Vite Workers. Retrieval-first: `cf cli search` + `cf schema` + docs MCP before citing flags/fields.

## When to Use

- New Worker, Pages project, environment (mode staging/production), secret, deploy, build failure, live error, branch/PR preview, Wrangler→cf migration.
- When NOT: data-model or realtime-design questions — use `cf-data` / `cf-realtime`.

## Implementation

Programmatic TypeScript config (`cloudflare.config.ts`) — LSP-autocompleted, `mode`-switched envs instead of copied `env` blocks:

```ts
import { bindings, defineConfig, triggers } from "cf/config";
import * as entrypoint from "./index.js" with { type: "cf-worker" };
export default defineConfig(({ mode }) => ({
  worker: {
    name: "my-worker", entrypoint, compatibilityDate: "2026-09-27",
    env: {
      ENVIRONMENT: bindings.text(mode),
      API_TOKEN: bindings.secret(),
      DB: bindings.d1({ name: `mydb-${mode}` }),
      CACHE: bindings.kv({ id: mode === "production" ? "prod-id" : "stage-id" }),
    },
    triggers: [triggers.fetch({ pattern: "example.com/*" }), triggers.scheduled({ schedule: "0 * * * *" })],
  },
}));
```

Flow: `cf init` (new/Vite setup) → `cf dev` → `cf deploy` (static sites: `cf deploy` with no config; Pages: `cf pages deploy`). Migrate: `cf migrate [--dry-run]` (Vite Workers convert to config.ts; esbuild/Rust/Python keep delegating to Wrangler during beta). Secrets via secret bindings / `cf deploy --secrets-file` (never in config). Pages production: `--branch main`. CI: token in GitHub Secrets. Audit gate (first upload): `security-audit` quick pass BEFORE deploy; fix High/Critical first. Abuse gate (every deploy): paid/external-cost route → refuse until Turnstile siteverify verified live (tokenless POST → 403, browser flow → 200).

Triage: builds → `cloudflare-builds` MCP (`cf builds`); live errors → `cloudflare-observability` MCP (`cf observability`/`cf logs`, `$metadata.service/message/error` first).

Python Workers GA: FastAPI/Django/Flask via `workers.asgi`/`wsgi` (still via Wrangler delegation).

## Previews (branch/PR isolation)

`cf previews` manages branch Previews under the same Worker; `cf deploy` stays production. Protect with Access; PR URLs via Workers Builds. Same isolation rules as Wrangler Previews: DO/Containers auto-isolate; KV/D1/R2/Queues/Vectorize/Hyperdrive share unless rebound — see `cf-data`; Workflows/service-bindings/consumers/cron/routes stay on prod — see `cf-realtime`.

## Quick Reference

| Task | cf (default) | Wrangler fallback (beta) |
|---|---|---|
| Discover API op | `cf cli search "<action + resource>"` → `cf schema <cmd>` | docs MCP |
| Scaffold | `cf init [dir]` | `npm create cloudflare@latest` |
| Local dev | `cf dev [--mode production]` | `wrangler dev` |
| Deploy | `cf deploy` / `cf pages deploy` | `wrangler deploy` |
| Migrate | `cf migrate [--dry-run]` | — |
| Builds/logs | `cf builds` / `cf logs` | `cloudflare-builds` MCP |
| Auth | `cf auth login/list` + `--profile` | `wrangler login` |

## Common Mistakes

- Chaining nested `--help` across 3,000 commands instead of `cf cli search` first; putting names/IDs/domains in search queries (keep them anonymous).
- `wrangler.jsonc`/TOML for new Vite features; stale `compatibilityDate`; secrets in source.
- `compatibilityDate` newer than bundled workerd breaks `dev` — pin at/below runtime max.
- Trusting first API page (paginate via `result_info.total_pages`); using default-account MCP OAuth for other accounts (use `cf --profile` / per-account auth).
- Preview pointed at prod D1/KV/R2 (rebind per Preview).

## Reuses

`wrangler`, `workers-best-practices`, `cloudflare` (`workers/`, `pages/`, `observability/` refs), `cloudflare-builds`, `cloudflare-observability`.
