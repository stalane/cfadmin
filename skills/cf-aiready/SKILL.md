---
name: cf-aiready
description: Use when making a Cloudflare-hosted domain agent-discoverable — sitemap, Content-Signal, markdown negotiation, RFC 9727 API catalog, Link headers, auth.md/OAuth discovery — or when an isitagentready.com scan check is failing.
---

# cf-aiready (agent discoverability, Cloudflare-only)

## Overview

Six machine-readable surfaces plus one scan command make any domain agent-ready. Proven on a Pages + Functions site in a Free-plan zone with all six checks green — no paid plan, no origin changes.

## When to Use

- New domain launch checklist; any failing `isitagentready.com` check (`discoverability`, `contentAccessibility`, `discovery`, `botAccessControl`).
- When NOT: compute/storage/AI design (→ sibling `cf-*` skills). Paid-plan edge features are documented only as what to skip.

## The six surfaces

| # | Surface | Serves | Pass condition |
|---|---------|--------|----------------|
| 1 | `sitemap.xml` + `Sitemap:` line in `robots.txt` | `application/xml`, HTTP 200 | `discoverability.sitemap` pass |
| 2 | `Content-Signal: ai-train=…, search=…, ai-input=…` in `robots.txt` | under the `User-agent` block | `botAccessControl.contentSignals` pass |
| 3 | `Accept: text/markdown` → Markdown, `Content-Type: text/markdown`, `x-markdown-tokens` | HTML stays default | `contentAccessibility.markdownNegotiation` pass |
| 4 | `/.well-known/api-catalog` → `application/linkset+json; profile="https://www.rfc-editor.org/info/rfc9727"` | `linkset` array, each entry `anchor` + `service-desc`/`service-doc` (+`status` if a health endpoint exists) | `discovery.apiCatalog` pass |
| 5 | Homepage `Link` headers: `api-catalog`, `service-desc`, `service-doc`, `describedby` (RFC 8288 / RFC 9727 §3) | comma-separated or multiple headers, both valid | `discoverability.linkHeaders` pass |
| 6 | `/auth.md` (Markdown, H1 contains `auth.md`) + `/.well-known/oauth-protected-resource` (RFC 9728) + `/.well-known/oauth-authorization-server` (RFC 8414, matching `issuer`, `agent_auth` block) | anonymous flow if no OAuth: `identity_types_supported: ["anonymous"]`, `anonymous.credential_types_supported`, `claim_uri` | `discovery.authMd` pass |

## Implementation

Static docs at the site root (`sitemap.xml`, `index.md`, `api-catalog.json`, `auth.md`, `openapi/*.json`, `oauth-*.json`) plus one Pages Function middleware that serves them with exact `Content-Type`s via `env.ASSETS` (same pattern as surface 3). Prefer code over dashboard Transform Rules / zone settings: it is versioned, locally testable, and works on Free-plan zones where `content_converter` is `editable: false` (PATCH → `1015 Not allowed`).

```js
// Branch pattern inside functions/_middleware.js — exact media type wins.
if ((request.method === 'GET' || request.method === 'HEAD') &&
    url.pathname === '/.well-known/api-catalog') {
  const assetRes = await env.ASSETS.fetch(new URL('/api-catalog.json', url));
  if (assetRes.ok) {
    const body = await assetRes.arrayBuffer();
    return new Response(request.method === 'HEAD' ? null : body, {
      status: 200,
      headers: {
        'Content-Type':
          'application/linkset+json; profile="https://www.rfc-editor.org/info/rfc9727"',
        Vary: 'Accept',
        'Content-Length': String(body.byteLength),
        Link: '<https://example.com/.well-known/api-catalog>; rel="api-catalog"',
      },
    });
  }
}
return context.next();
```

Homepage `Link` headers: pass through with `context.next()`, then append discovery links to a cloned `Headers` (preserves static `_headers`). Markdown branch: set `Link` explicitly since Function responses bypass `_headers`. Answer HEAD on well-known endpoints (RFC 9727 §2 requires a `Link` header on HEAD).

Verify locally before deploying: `wrangler pages dev` + curl matrix (markdown accept, browser accept, HEAD, non-page passthrough, byte-identical HTML body). Deploy production with `wrangler pages deploy . --project-name=<p> --branch main` (missing `--branch main` silently ships a preview). Then scan:

```
POST https://isitagentready.com/api/scan  {"url": "https://example.com"}
```

Keep it updated: `index.md` ↔ `index.html`, catalog entries on API changes, sitemap `<lastmod>` on publish/remove.

## Validator quirks (banked from live scans)

- PRM requires **non-empty** `authorization_servers` **and** `scopes_supported` — empty arrays fail validation. No OAuth? Advertise the issuer host itself, scope labels for the gated capabilities, and document that no scope negotiation happens.
- Anonymous `agent_auth` requires `identity_types_supported`, `anonymous.credential_types_supported`, **and** `claim_uri`. No ceremony exists? Point `claim_uri` at an anchor documenting exactly that.
- AS metadata must carry no fake endpoints: omit `token_endpoint`/`authorization_endpoint` (both OPTIONAL per RFC 8414) rather than inventing dead URLs agents would call.
- Every `service-desc`/`service-doc` URL should return a real 200 — author minimal OpenAPI specs if none exist; omit `status` when no health endpoint exists; list only verified-live anchors.
- Scanner probes malformed backticked well-known URLs (e.g. `oauth-protected-resource\``) — its bug, informational only.
- Fresh deploys can serve mismatched headers on the apex for ~1–2 min: confirm `cf-cache-status`, wait, re-check before scanning.

## Common Mistakes

- Enabling edge Markdown for Agents on a Free-plan zone instead of the Function fallback.
- `.md`/extensionless docs served as `text/html` soft-404 — force the media type in the Function.
- Inventing OAuth endpoints to satisfy the scanner (agents will call them).
- Preview deploy mistaken for production (missing `--branch main`).

## Reuses

`cf-frontdoor` (Pages deploys, `_headers`), `cf-workers-deploy` (dev → deploy triage). Specs: RFC 9727 (Appendix A.1 shape), RFC 8288, RFC 9728, RFC 8414, contentsignals.org, workos/auth.md `AUTH.md` (real `agent_auth` schema: `skill`, `identity_endpoint`, `identity_types_supported` — `register_uri`/`registration_methods` alone are not recognized).
