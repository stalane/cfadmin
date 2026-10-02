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

## Classification with Clef (open-source, on Workers AI)

For classify-then-route steps (triage, moderation, intent, tool choice) reach for Clef before a general LLM: `@cf/cloudflare/clef` and the faster `@cf/cloudflare/clef-flash` are Cloudflare's first open-source decision models, served on Workers AI for high-speed classification inside agentic workflows — cheaper and lower-latency than asking a chat model to label. Teams can RL-fine-tune decision models on their own data via the new tuning platform. Confirm exact input/output shapes in the Workers AI model docs before wiring (decision models don't take chat `messages`).

## Live web grounding via Web Search API (beta, Oct 2026)

When answers must reflect post-cutoff reality, ground the model with Web Search API instead of letting it guess URLs. Providers at launch: Ceramic.ai, Exa, Linkup (all Zero Data Retention through Cloudflare, verified-bot crawling standards; BYOK supported, otherwise billed to AI Gateway credits at list price). Two integration paths: dedicated search call from a Worker (`env.AI.websearch({ gatewayId, query, provider, limit })`) or REST (`POST /accounts/{id}/ai/websearch/`), both routed through your AI Gateway (logs + billing in one place); or pass-through of providers' native search tools (Anthropic/OpenAI/xAI/Alibaba flags) via the Gateway. Agentic pattern: expose it as a `web_search` tool — model calls the tool, Worker runs the search, results go back into context.

## Spend control: Auto Router + User Insights (free for Gateway users)

Default every gateway to cost-aware routing before hand-picking models. **Auto Router**: send requests with `cloudflare/auto` instead of a provider model — an edge classifier scores task type/difficulty vs cost, attempts the best-fit model, falls back on provider failure. Scope it with `cf-aig-allowed-models` / `cf-aig-allowed-providers` headers; read `cf-aig-routing-reason` to see why a model won. **User Insights**: groups live traffic by task, model, turn, and user; the Potential Savings view flags requests a cheaper model could have served (same signals the router uses); spend + anomaly views catch compromised keys and misbehaving agents. Attribute per-user spend via custom metadata or Access in front of the gateway.

Manage via `cf`: `cf ai …`, `cf ai-gateway …`, `cf ai-search …` (JSON default, discover with `cf cli search`). Wire in config via `bindings.ai()` / `bindings.vectorize({ name })`.

## AI Search (GA Oct 2026) for hosted retrieval

When the project needs search-over-own-content without hand-rolled RAG, prefer managed AI Search over assembling Vectorize + chunking + ranking yourself. GA brings: direct image-pixel embeddings for visual search, OCR over scanned PDFs, files up to 10 MiB, and any-chat-model compatibility. Hand-rolled Vectorize RAG stays for custom embedding/ranking control; AI Search wins for time-to-working-search. Confirm GA pricing in docs before quoting costs.

## Common Mistakes

- Calling OpenAI/Anthropic from a VPS backend instead of from the Worker through AI Gateway.
- Storing embeddings in D1/KV instead of Vectorize.
- Hardcoding model names without checking current availability.
- Skipping Gateway caching and paying per repeated prompt.
- Hand-pinning an expensive flagship model for every call instead of `cloudflare/auto` + User Insights review.

## Reuses

`cloudflare` (`workers-ai/`, `vectorize/`, `ai-gateway/`, `ai-search/` refs), `workers-best-practices` (streaming responses, waitUntil for logging), `ai-utils` dev toolkit.
