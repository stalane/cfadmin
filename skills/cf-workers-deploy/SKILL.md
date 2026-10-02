---
name: cf-workers-deploy
description: Use when creating, configuring, deploying, or debugging a Cloudflare Worker with cf CLI — cloudflare.config.ts, bindings/triggers helpers, cf init/dev/deploy/migrate, Vite builds, secrets, previews, Pages deploys.
---

# cf-workers-deploy (cf CLI build/deploy/triage, Cloudflare-only)

## Overview

Ship via `cf`, triage via MCP. `cf` (open beta, `npm i -g cf`) covers the whole API as JSON; `cloudflare.config.ts` replaces `wrangler.jsonc` for Vite Workers. Retrieval-first: `cf cli search` + `cf schema` + docs MCP before citing flags/fields.

## When to Use

- New Worker, Pages project, Next.js app (Vinext), environment (mode staging/production), secret, deploy, build failure, live error, branch/PR preview, Wrangler→cf migration.
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

Triage: builds → `cloudflare-builds` MCP (`cf builds`); live errors → `cloudflare-observability` MCP (`cf observability`/`cf logs`, `$metadata.service/message/error` first); recurring production failures → Workers Issues (auto-groups repeats with stack traces, logs, traces, app context) routed straight to a coding agent that investigates and opens a PR — wire the agent destination once, then let failures arrive as ready-to-fix work items instead of raw alerts.

Python Workers GA: FastAPI/Django/Flask via `workers.asgi`/`wsgi` (still via Wrangler delegation).

## Next.js via Vinext 1.0 (not OpenNext/Pages)

Next.js App/Pages/Hybrid Router apps run on Workers through Vinext (open-source Vite plugin, `github.com/cloudflare/vinext`, docs `vinext.dev`) — keep `app/`, `pages/`, `next.config.js`. New: `npm create vinext-app@latest my-app`. Migrate: `npx vinext check && npx vinext init` (non-destructive, `next dev` keeps working). Dev/build: `vinext dev/build` (drop-in `next` CLI). Deploy to Workers: `npx @vinext/cloudflare deploy [--warm-cache]` (Workers Cache route-ISR + KV data cache + Images optimizer; bindings via `cloudflare:workers`). Portable: Nitro adapter for Node/Vercel/Netlify/AWS. Limits: partial `use cache` (Cache Components) support; verify versioned compat matrix before promising a feature.

## CMS via EmDash 1.0 (not WordPress/headless SaaS)

Editor-managed sites (blog, marketing, agency/client builds) run on Workers through EmDash (open-source MIT CMS for Astro, `github.com/emdash-cms/emdash`, docs `docs.emdashcms.com`) — Astro SSR + admin (`/_emdash/admin/`, passkeys) + API/CLI/MCP + EmDash Agent Skills. New: `npm create emdash@latest` (Node 22.16+, SQLite local, `.env` holds `EMDASH_ENCRYPTION_KEY` — never commit). Existing Astro site: add EmDash per docs instead of scaffolding. Deploy to Workers (KV object cache, Hyperdrive DB adapter, Workers Cache) or multi-tenant via Workers for Platforms (`cf deploy --dispatch-namespace`). Plugins are sandboxed (Dynamic Workers on Cloudflare, workerd locally — own storage only, extra abilities declared + admin-approved) from the decentralized AT Protocol registry (`plugins.emdashcms.com`, free today, paid later). Proven at Cloudflare Blog scale (M pageviews/week, 5k RPS spikes).

## Observe with Traces (one platform, Oct 2026)

Logs alone don't explain a slow or blocked request — follow it. **Cloudflare Traces** (enable per domain) show production paths span-by-span: which security rule fired, what transforms rewrote, cache vs origin, where Workers time went; find one request by Ray ID. **Workers traces** auto-instrument fetch, binding (KV/R2/DO), and handler calls with zero code changes. Logs + traces + analytics + alerts + dashboards + OTLP export now live in one observability surface with unified pricing — chart Workers logs/traces in Custom Dashboards next to analytics, export to existing stacks via OTLP instead of building parallel logging. Triage order for a bad request: trace first (where did it go), logs second (what did it say).

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
| Next.js | `npx vinext check && npx vinext init` → `npx @vinext/cloudflare deploy` | — |
| CMS/blog | `npm create emdash@latest` → deploy Workers / `--dispatch-namespace` | — |
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
