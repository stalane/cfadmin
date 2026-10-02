---
name: cf-ai
description: Use when adding AI to a Cloudflare project — Workers AI inference, Vectorize RAG, AI Gateway routing and caching, or AI Search widgets.
---

# cf-ai (Workers AI/Vectorize/AI Gateway, Cloudflare-only)

## Overview

Run inference at the edge, store embeddings in Vectorize, front every provider call with AI Gateway. Seen in 22 Workers-AI + 6 Vectorize audited projects (`forja`, `resolvehq`, `open-seo` patterns).

## When to Use

- LLM text, embeddings, image models, semantic search/RAG, multi-provider routing, usage caching, live web grounding.
- When NOT: CRUD/realtime (→ `cf-data` / `cf-realtime`).

## Implementation

```ts
const answer = await env.AI.run("@cf/meta/llama-3.1-8b-instruct", {
  messages: [{ role: "user", content: prompt }],
});
const vec = await env.AI.run("@cf/baai/bge-base-en-v1.5", { text: [chunk] });
await env.VECTORIZE.upsert([{ id: docId, values: vec.data[0], metadata: { docId } }]);
const hits = await env.VECTORIZE.query(vec.data[0], { topK: 5 });
```

Route via AI Gateway (caching, rate limits, multi-provider fallback) rather than calling providers directly. Verify model IDs against the docs MCP — they change; never trust memory. Python Workers run `openai`/`langchain`/`mcp` natively; use `langchain-cloudflare` for Workers AI.

## Live web grounding via Web Search API (beta, Oct 2026)

When answers must reflect post-cutoff reality, ground the model with Web Search API instead of letting it guess URLs. Providers at launch: Ceramic.ai, Exa, Linkup (all Zero Data Retention through Cloudflare, verified-bot crawling standards; BYOK supported, otherwise billed to AI Gateway credits at list price). Two integration paths: dedicated search call from a Worker (`env.AI.websearch({ gatewayId, query, provider, limit })`) or REST (`POST /accounts/{id}/ai/websearch/`), both routed through your AI Gateway (logs + billing in one place); or pass-through of providers' native search tools (Anthropic/OpenAI/xAI/Alibaba flags) via the Gateway. Agentic pattern: expose it as a `web_search` tool — model calls the tool, Worker runs the search, results go back into context.

Manage via `cf`: `cf ai …`, `cf ai-gateway …`, `cf ai-search …` (JSON default, discover with `cf cli search`). Wire in config via `bindings.ai()` / `bindings.vectorize({ name })`.

## Common Mistakes

- Calling OpenAI/Anthropic from a VPS backend instead of from the Worker through AI Gateway.
- Storing embeddings in D1/KV instead of Vectorize.
- Hardcoding model names without checking current availability.
- Skipping Gateway caching and paying per repeated prompt.

## Reuses

`cloudflare` (`workers-ai/`, `vectorize/`, `ai-gateway/`, `ai-search/` refs), `workers-best-practices` (streaming responses, waitUntil for logging), `ai-utils` dev toolkit.
