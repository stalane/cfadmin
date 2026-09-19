---
name: cf-ai
description: Use when adding AI to a Cloudflare project — Workers AI inference, Vectorize RAG, AI Gateway routing and caching, or AI Search widgets.
---

# cf-ai (Workers AI/Vectorize/AI Gateway, Cloudflare-only)

## Overview

Run inference at the edge, store embeddings in Vectorize, front every provider call with AI Gateway. Seen in 22 Workers-AI + 6 Vectorize audited projects (`forja`, `resolvehq`, `open-seo` patterns).

## When to Use

- LLM text, embeddings, image models, semantic search/RAG, multi-provider routing, usage caching.
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

Route via AI Gateway (caching, rate limits, multi-provider fallback) rather than calling providers directly. Verify model IDs against the docs MCP — they change; never trust memory.

## Common Mistakes

- Calling OpenAI/Anthropic from a VPS backend instead of from the Worker through AI Gateway.
- Storing embeddings in D1/KV instead of Vectorize.
- Hardcoding model names without checking current availability.
- Skipping Gateway caching and paying per repeated prompt.

## Reuses

`cloudflare` (`workers-ai/`, `vectorize/`, `ai-gateway/`, `ai-search/` refs), `workers-best-practices` (streaming responses, waitUntil for logging), `ai-utils` dev toolkit.
