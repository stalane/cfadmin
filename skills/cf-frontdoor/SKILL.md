---
name: cf-frontdoor
description: Use when exposing, routing, or protecting a Cloudflare project — Zones and DNS, Pages and Static Assets, Tunnels with cloudflared, WAF, Turnstile, Email Routing and Email Workers.
---

# cf-frontdoor (DNS/Pages/Tunnel/WAF/Turnstile/Email, Cloudflare-only)

## Overview

Every public URL terminates on Cloudflare: DNS → (WAF/Turnstile) → Pages/Worker or Tunnel-origin. API-driven, no `cloudflared tunnel login` browser flow.

## When to Use

- Custom domain, CNAME, Pages deploy, Next.js deploy (Vinext), local-service exposure, bot protection, contact forms, inbound email.
- When NOT: compute/storage/AI design (→ sibling `cf-*` skills).

## Implementation

Managed tunnel via `cf` (JSON default, no `cloudflared tunnel login` browser flow):

1. `cf tunnels create --name myapp` (≡ `POST /accounts/{ACCT}/cfd_tunnel`)
2. PUT ingress configurations (create-body `config` is NOT persisted — always PUT)
3. `cf dns …` CNAME `app → {TUNNEL}.cfargotunnel.com` (proxied)
4. Fetch tunnel token → run `cloudflared` with the token; verify with `curl`.

Local static origin: Caddy on loopback via systemd (`file_server` + `encode gzip`; TLS terminates at Cloudflare) — `python3 -m http.server` is dev-only. Tunnel itself as systemd unit (`Restart=always`, token in root-owned 600 env file); `bgstart` cloudflared only for throwaway tests.

Static sites: Workers Static Assets, `cf deploy`, or `cf pages deploy --branch main` for production (wrangler fallback: `wrangler pages deploy --branch main`). Next.js apps: Vinext on Workers (see `cf-workers-deploy`), not Pages/OpenNext. Routes/custom domains via `triggers.fetch({ pattern })` in `cloudflare.config.ts`. Protection: WAF managed rules (`cf firewall …`, `cf rulesets …`) + Turnstile widget with server-side `siteverify` in the Worker (see `turnstile-spin`; official `turnstile-demo-workers` example). Email: Email Routing + Email Workers (`cf email-routing …`, `cf email-sending …`; `cloud-mail`, `agentic-inbox` patterns); send via Email Sending binding.

## Common Mistakes

- Skipping step 2 (tunnel serves 503 "no ingress rules were defined").
- Unproxied (grey-cloud) CNAME for a tunneled host.
- Pages preview deploy mistaken for production (missing `--branch main`).
- CAPTCHA via third-party widget instead of Turnstile.

## Reuses

`exposing-local-service-with-cloudflare-tunnel`, `turnstile-spin`, `cloudflare-email-service`, `cloudflare` (`tunnel/`, `pages/`, `waf/`, `turnstile/`, `email-routing/`, `email-workers/` refs).
