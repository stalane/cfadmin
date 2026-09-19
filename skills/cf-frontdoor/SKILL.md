---
name: cf-frontdoor
description: Use when exposing, routing, or protecting a Cloudflare project — Zones and DNS, Pages and Static Assets, Tunnels with cloudflared, WAF, Turnstile, Email Routing and Email Workers.
---

# cf-frontdoor (DNS/Pages/Tunnel/WAF/Turnstile/Email, Cloudflare-only)

## Overview

Every public URL terminates on Cloudflare: DNS → (WAF/Turnstile) → Pages/Worker or Tunnel-origin. API-driven, no `cloudflared tunnel login` browser flow.

## When to Use

- Custom domain, CNAME, Pages deploy, local-service exposure, bot protection, contact forms, inbound email.
- When NOT: compute/storage/AI design (→ sibling `cf-*` skills).

## Implementation

Managed tunnel (note: create-body `config` is NOT persisted — always PUT configurations):

1. `POST /accounts/{ACCT}/cfd_tunnel` `{"name":"myapp"}`
2. `PUT /accounts/{ACCT}/cfd_tunnel/{TUNNEL}/configurations` with `ingress: [{hostname, service: http://localhost:PORT}, {service: "http_status:404"}]`
3. `POST /zones/{ZONE}/dns_records` CNAME `app → {TUNNEL}.cfargotunnel.com` (proxied)
4. `GET .../cfd_tunnel/{TUNNEL}/token` → run `cloudflared` with the token; verify with `curl`.

Static sites: Workers Static Assets or `wrangler pages deploy --branch main` for production. Protection: WAF managed rules + Turnstile widget with server-side `siteverify` in the Worker (see `turnstile-spin`; official `turnstile-demo-workers` example). Email: Email Routing + Email Workers (`cloud-mail`, `agentic-inbox` patterns); send via Email Sending binding.

## Common Mistakes

- Skipping step 2 (tunnel serves 503 "no ingress rules were defined").
- Unproxied (grey-cloud) CNAME for a tunneled host.
- Pages preview deploy mistaken for production (missing `--branch main`).
- CAPTCHA via third-party widget instead of Turnstile.

## Reuses

`exposing-local-service-with-cloudflare-tunnel`, `turnstile-spin`, `cloudflare-email-service`, `cloudflare` (`tunnel/`, `pages/`, `waf/`, `turnstile/`, `email-routing/`, `email-workers/` refs).
