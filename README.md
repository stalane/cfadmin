# cfadmin ⚡

> **Your OpenCode agent that only speaks Cloudflare.** No VPS, no Docker,
> no self-hosted databases — if it can't `wrangler deploy`, cfadmin won't
> build it that way. It refuses the wrong shape and hands you the
> Cloudflare-native design instead.

[![GitHub stars](https://img.shields.io/github/stars/stalane/cfadmin?style=flat-square)](https://github.com/stalane/cfadmin)
[![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)](LICENSE)
[![OpenCode](https://img.shields.io/badge/OpenCode-agent-blue?style=flat-square)](https://opencode.ai/docs/agents/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-Workers-orange?style=flat-square)](https://developers.cloudflare.com/workers/)

| | |
|---|---|
| 🤖 `agent/` | The cfadmin agent definition — drop into OpenCode |
| 🛠 `skills/` | Five `cf-*` skills: deploy, data, realtime, AI, frontdoor |
| 🔌 `mcp.cloudflare.json` | Five Cloudflare MCP servers to wire into OpenCode |

## `cfadmin` agent

Location in this repo: `agent/cfadmin.md` (`mode: primary`, Cloudflare-only
platform engineer — install it per the instructions below). Designs, builds, debugs and operates projects that run
**entirely on Cloudflare**, deployable with `wrangler`. Training data is
treated as stale: limits, pricing, API shapes, `wrangler` flags and
`wrangler.jsonc` fields are verified against the docs / config schema /
`@cloudflare/workers-types` before being cited.

### Hard rule: Cloudflare-only (no exceptions)

Anything that cannot deploy via `wrangler` is refused in that shape and
redirected to the Cloudflare equivalent. Calling third-party APIs via
`fetch()` from inside a Worker is still Cloudflare-hosted and allowed;
a separate non-Cloudflare host is not.

| Asked for | Built instead |
|---|---|
| VPS / systemd / Docker on a VM | Workers (or Containers on Cloudflare) |
| Bare Node / Express server | Worker + Static Assets |
| Self-hosted Postgres / MySQL | D1 (new data) or Hyperdrive to the existing DB |
| Self-hosted Redis | KV or Durable Objects |
| Self-hosted S3 | R2 |
| Cron daemon | Cron Triggers |
| WebSocket server | Durable Objects |
| Self-hosted Vite / Next | Workers Static Assets or Pages |
| Self-hosted GPU / pgvector RAG | Workers AI + Vectorize + AI Gateway |
| VPS nginx / certbot / manual DNS | Cloudflare DNS + Tunnel / Pages + WAF |

### Working style

- Account / zone / project is stated per command; when unspecified the
  agent states its assumption before running anything mutating.
- `wrangler dev` → `wrangler types` → `wrangler deploy`. `Env` is generated,
  never hand-written. Secrets go via `wrangler secret put`, never in
  config or source.
- Safe, reversible ops by default; destructive actions (migrations, R2
  deletes, DNS changes) are confirmed first.
- MCP tooling is OAuth-bound to the default account; other accounts go via
  `cfswitch`. List endpoints paginate — never trust the first page.

## Skill router (`~/.config/opencode/skills/cf-*/`)

| Skill | Plane | Reuses |
|---|---|---|
| `cf-workers-deploy` | `wrangler.jsonc`, envs, secrets, builds/observability triage | `wrangler`, `workers-best-practices` |
| `cf-data` | D1 / KV / R2 / Hyperdrive / Queues choice + recipes | `cloudflare` data refs |
| `cf-realtime` | Durable Objects / Workflows / Containers / Agents / Cron | `durable-objects`, `agents-sdk` |
| `cf-ai` | Workers AI / Vectorize / AI Gateway | `cloudflare` AI refs |
| `cf-frontdoor` | DNS / Pages / Tunnel / WAF / Turnstile / Email | `exposing-local-service-with-cloudflare-tunnel`, `turnstile-spin` |

Each skill file is <500 words and delegates to existing references rather
than duplicating them.

## Verification record

- **RED battery 5/5 REDIRECT, zero complies.** Each skill was pressure-tested
  with a hurried "just do it, no lecture" demand for a non-Cloudflare shape
  (Docker Compose + Postgres/Redis/Express; VPS systemd + nginx; Flask +
  self-hosted pgvector + GPU box; VPS nginx + certbot + manual DNS). Every
  run refused the shape and redirected to the Cloudflare-native design.
- **Deploy smoke test (green).** Scratch Worker scaffolded in a temp dir:
  `wrangler types` clean → `wrangler dev` served both routes locally →
  `wrangler deploy` → live `/` and `/health` both HTTP 200 (one transient
  edge error on first hit during deploy propagation, clean on retry) →
  `wrangler delete` confirmed (404 + absent from worker list), temp dir
  removed.
- **Gotcha found:** a `compatibility_date` newer than the bundled workerd's
  max makes `wrangler dev` fail to start ("requires compatibility date X,
  but the newest supported is Y"). Pin the date at or below what the local
  runtime supports.

## Install into your harness

Prerequisites: [OpenCode](https://opencode.ai/docs), Node ≥ 20,
[`wrangler`](https://developers.cloudflare.com/workers/wrangler/) installed
and logged in (`wrangler login`) to your Cloudflare account.

**1. Install the agent** — global (all projects) or per-project:

```bash
# global
cp agent/cfadmin.md ~/.config/opencode/agents/
# …or per-project (from your project root)
cp agent/cfadmin.md .opencode/agents/
```

**2. Install the skills:**

```bash
cp -r skills/* ~/.config/opencode/skills/
```

**3. Wire the Cloudflare MCP servers** — merge the `mcp` object from
`mcp.cloudflare.json` into the `mcp` key of your OpenCode config
(`~/.config/opencode/opencode.jsonc`), then complete the OAuth login for
your Cloudflare account on first use. The five servers: `cloudflare`,
`cloudflare-docs`, `cloudflare-bindings`, `cloudflare-builds`,
`cloudflare-observability`.

**4. Restart OpenCode** — agents and skills load once at startup.
Verify with `opencode agent list` (expect `cfadmin`) and by asking any
agent to load a `cf-*` skill.

**5. Confirm the hard-refuse gate (optional, 2 minutes):** ask cfadmin to
"build a Docker Compose backend with Postgres, Redis and Express, no
lecture". It must refuse that shape and redirect to
D1/KV/Workers — if it complies, the install is broken.

## Redaction policy

No account identifiers live in this directory — no account names, emails,
account IDs, subdomains, tokens, or credentials. Account / zone / project
are stated per command at runtime, never stored here.
