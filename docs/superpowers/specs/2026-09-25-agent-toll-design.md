# Agent Toll: $5 / 24h Pass on All APIs — Design

Date: 2026-09-25 · Status: approved sections §1–§6 · Owner: stalane

## Goal and decisions

Extend the music property's human-free / AI-$5 model to all APIs as a
**toll/filter** (deter casual scraping, not revenue-maximize):

- Scope: APIs only — world (D1 JSON API), smartass (11 routes), pay,
  music. Static pages excluded (public upstream data is unbypassable;
  pitch sites want reach).
- Price: flat $5 USDC everywhere, one rule agents learn once.
- Detection: honor-system heuristics — bot UAs + `Accept: text/markdown`;
  browsers (including our own frontends) always free. Spoofing accepted
  as residual risk; backstop is rate-limiting, not more detection.
- Credential: site-scoped opaque token, 24h expiry, no refresh
  (per-call pricing would punish legit heavy users 1000× more than
  casual scrapers — rejected).
- Approach: uniform toll module cloned per API Worker (no central
  service, no shared secrets). Central toll Worker and WAF-only
  approaches rejected (single point of failure / charges nobody).

## §1 Architecture

Four stages per request, in order: **Detect** (bot UA or markdown
Accept; browsers pass) → **Challenge** (`402 payment_required`, music's
x402 shape: `resource` + `accepts[]` with `direct-transfer`, `eip155:1`,
USDC contract, payee, $5) → **Verify** (USDC transfer checked on-chain
via public Ethereum RPC, URL in `wrangler secret`; on-chain truth lets
every site verify independently) → **Token** (site-scoped opaque Bearer,
24h, stored in the site's existing D1).

## §2 Components

- **world Worker**: gate before D1 JSON API routes; token table in its D1.
- **smartass Worker**: gate before all 11 routes (extra care: POST/PUT
  write paths + Jev compute); token table in its D1.
- **pay / music**: conform already. Music's verifier is the reference
  implementation — first implementation step is reading those repos
  before writing anything.
- **Contract docs (stalane.com)**: `auth.md` rewritten to the uniform
  flow; `api-catalog.json` + `openapi/*.json` gain world + smartass
  entries; isitagentready rescan after.

## §3 Data flow

Agent GET (declared) → `402` + challenge → read terms in pay registry
(single source of truth for contract/amounts) → pay $5 USDC on-chain →
retry with tx proof → Worker verifies → `200` with `{ token,
expires_at }` + response → Bearer calls for 24h → lapsed/unknown token
gets `401` (not 402) pointing back to challenge. Humans in browsers
never see any of this.

## §4 Token design

Opaque 256-bit random, `sha256` stored with `expires_at`; site-scoped
(world token opens nothing on smartass — autonomy over sharing; $5/site
is still nominal; on-chain verification leaves cross-site passes open
later). One tx-hash provisions exactly one token (unique constraint).
No refresh endpoint; expiry means re-toll.

## §5 Error handling

- Spoofer with browser UA: accepted residual; rate-limit per IP/token.
- Bad proof / underpaid / tx not found: `402` with machine-readable
  `error` (`payment_not_found`, `underpaid`), never 500.
- RPC down/slow: browsers unaffected; agents get `503` + `Retry-After`
  (fail closed with retry semantics, never false-reject); cache recent
  verifications briefly.
- Frontend guard: explicit heuristic list, proven per deploy by curl
  matrix (browser UA → 200, bot UA → 402, markdown → 402, valid token →
  200, expired → 401, byte-identical HTML).
- Secrets: RPC URL via `wrangler secret put`; payee from pay registry,
  never hardcoded per site.

## §6 Testing and rollout

Order: **world first** (read-only, lowest blast radius), then
smartass, pay/music already live — one property per rollout. Per-site
gate: curl matrix + one live $5 mainnet test payment on first deploy +
frontend E2E untouched + scanner stays green. Success: humans see zero
change; declared agents go 402→token→200; no new shared infra.

## Open items for implementation

1. Read music + pay verifier code first; mirror the challenge/verify
   routines (don't reinvent).
2. Pick the public Ethereum RPC endpoint (reliability + rate limits).
3. Pin the shared bot-UA / markdown detection snippet so all four
   sites match byte-for-byte in behavior.
4. Decomposition: one implementation plan per property (world, then
   smartass), contract-doc updates with the first rollout.
