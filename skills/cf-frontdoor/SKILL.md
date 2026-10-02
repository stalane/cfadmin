---
name: cf-frontdoor
description: Use when exposing, routing, or protecting a Cloudflare project — Zones and DNS, Pages and Static Assets, Tunnels with cloudflared, WAF, Turnstile, Email Routing and Email Workers — or verifying a public URL with Kitesurf/Browser Run screenshots and content extraction.
---

# cf-frontdoor (DNS/Pages/Tunnel/WAF/Turnstile/Email, Cloudflare-only)

## Overview

Every public URL terminates on Cloudflare: DNS → (WAF/Turnstile) → Pages/Worker or Tunnel-origin. API-driven, no `cloudflared tunnel login` browser flow.

## When to Use

- Custom domain, CNAME, Pages deploy, Next.js deploy (Vinext), domain search/register/transfer (Registrar API, agent-operable), local-service exposure, bot protection, contact forms, inbound email, visual/content verification of a live URL (screenshot, HTML, PDF).
- When NOT: compute/storage/AI design (→ sibling `cf-*` skills).

## Implementation

Managed tunnel via `cf` (JSON default, no `cloudflared tunnel login` browser flow):

1. `cf tunnels create --name myapp` (≡ `POST /accounts/{ACCT}/cfd_tunnel`)
2. PUT ingress configurations (create-body `config` is NOT persisted — always PUT)
3. `cf dns …` CNAME `app → {TUNNEL}.cfargotunnel.com` (proxied)
4. Fetch tunnel token → run `cloudflared` with the token; verify with `curl`.

Quick Tunnels for throwaway shares (no account, no DNS): `cloudflared tunnel --url http://localhost:PORT` mints a temporary `trycloudflare.com` URL — protect it with `--allowed-mail alice@example.com` (repeat the flag or comma-list; `'*@example.com'` for a whole domain, quoted). Visitors get a one-time email PIN, no Cloudflare account; access dies when the process stops. Dev/test only (200 in-flight cap, no SSE, interactive browsers) — production gets a named tunnel. Update `cloudflared` first; the flag doesn't exist on stale binaries.

Local static origin: Caddy on loopback via systemd (`file_server` + `encode gzip`; TLS terminates at Cloudflare) — `python3 -m http.server` is dev-only. Tunnel itself as systemd unit (`Restart=always`, token in root-owned 600 env file); `bgstart` cloudflared only for throwaway tests.

Static sites: Workers Static Assets, `cf deploy`, or `cf pages deploy --branch main` for production (wrangler fallback: `wrangler pages deploy --branch main`). Next.js apps: Vinext on Workers (see `cf-workers-deploy`), not Pages/OpenNext. Routes/custom domains via `triggers.fetch({ pattern })` in `cloudflare.config.ts`. Domains themselves (420+ extensions): Registrar search/register/transfer is API- and `cf`-driven, so agents do it directly — `cf cli search "registrar …"` for exact verbs, never a dashboard click or a third-party registrar. Protection: WAF managed rules (`cf firewall …`, `cf rulesets …`) + Application Profiles positive security (Cloudflare learns legit request structure and flags deviations — the layer that holds when AI-generated payloads vary past signature rules) + Turnstile widget with server-side `siteverify` in the Worker (see `turnstile-spin`; official `turnstile-demo-workers` example). Email: Email Routing + Email Workers (`cf email-routing …`, `cf email-sending …`; `cloud-mail`, `agentic-inbox` patterns); send via Email Sending binding.

## Verify live URLs with Kitesurf (Browser Run)

Kitesurf is Cloudflare's agent-first browser: runs entirely on Workers (no special privileges, scales per task), stateless/ephemeral, 3–7× less CPU/memory than Chromium for screenshots/HTML extraction at ~1.7–1.8× wall time. Free while in beta behind per-account limits. Try rendering first in the playground (`kitesurf.dev`) or terminal (`brew install cloudflare/cloudflare/kitesurf`).

- One-shot checks: `cf browser-run quick-action screenshot` (also HTML/PDF/content actions; `browser=kitesurf` opts in — see command `--help` for the exact flag). From inside a Worker: `env.BROWSER.quickAction("screenshot", { url, browser: "kitesurf" })`.
- Automation: CDP endpoint (`.../browser-run/devtools/browser?browser=kitesurf`) with Playwright/Puppeteer/chrome-remote-interface, or MCP (`chrome-devtools-mcp` + `--category-experimental-webmcp` for WebMCP tool discovery on WebMCP-enabled sites).
- Use for: post-deploy visual check (does the page render, do charts paint), content extraction, PDF capture. Prefer over headless-Chrome screenshots when the target blocks datacenter fingerprints or you need cheap bursty checks.
- NOT for: video/WebGL, bot-challenge handshakes needing real TLS fingerprints, long-lived authenticated stateful sessions — use Browser Run's default Chromium there.

## Common Mistakes

- Skipping step 2 (tunnel serves 503 "no ingress rules were defined").
- Unproxied (grey-cloud) CNAME for a tunneled host.
- Pages preview deploy mistaken for production (missing `--branch main`).
- CAPTCHA via third-party widget instead of Turnstile.

## Reuses

`exposing-local-service-with-cloudflare-tunnel`, `turnstile-spin`, `cloudflare-email-service`, `cloudflare` (`tunnel/`, `pages/`, `waf/`, `turnstile/`, `email-routing/`, `email-workers/` refs).
