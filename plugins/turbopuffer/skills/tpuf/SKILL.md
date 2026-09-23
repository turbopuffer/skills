---
description: >
  Use when the user wants to work with turbopuffer — a serverless vector and
  full-text search database. Covers setup, querying (vector, BM25, hybrid),
  writing, schema, native embeddings, sharding, pinning, branching, and
  troubleshooting. Trigger on any mention of turbopuffer or tpuf.
---

# turbopuffer

## Before you start

- Needs `TURBOPUFFER_API_KEY` (from https://turbopuffer.com/dashboard) and a region (`TURBOPUFFER_REGION`, e.g. `gcp-us-central1`). Never print the key.
- Inspect live data with `curl https://$TURBOPUFFER_REGION.turbopuffer.com/v1/namespaces/{ns}/metadata -H "Authorization: Bearer $TURBOPUFFER_API_KEY"`.
- Write app code with the user's SDK. Examples here are TypeScript; Python is the same API in snake_case.
- When unsure, read the docs: `https://turbopuffer.com/docs/<page>.md` (index: https://turbopuffer.com/llms.txt).

## Routing

| User wants to... | Load |
|---|---|
| Install SDK, first integration | `references/setup.md` |
| Search: vector, BM25, hybrid, filters, aggregations | `references/query.md` |
| Configure full-text search | `references/fts.md` |
| Upsert, patch, delete, schema changes | `references/write.md` |
| Let turbopuffer embed text | `references/embedding.md` |
| Namespace layout, multi-tenancy, permissions, limits | `references/namespace-design.md` |
| One namespace > 1TB / 500M docs | `references/sharding.md` |
| High sustained QPS, always-warm cache | `references/pinning.md` |
| Clone, copy, back up a namespace | `references/branching.md` |
| Bulk load, indexing backlog | `references/ingestion.md` |
| 429 errors | `references/troubleshoot-429.md` |
| Slow, empty, or wrong results | `references/doctor.md` |

## Rules

1. Check the schema (`ns.metadata()`) before querying — don't guess attribute names.
2. Batch writes; never one document per request.
3. Confirm before `deleteAll()`, `delete_by_filter`, `patch_by_filter`, or enabling pinning (billing).
4. Use `limit`, not `top_k`.

## Gotchas

- Namespaces are created on first write. `distance_metric`, vector dims/types, and `sharding` are fixed after that.
- The client needs a region. TS import: `import { Turbopuffer } from "@turbopuffer/turbopuffer"`.
- `full_text_search`, `regex`, `glob`, `fuzzy` default `filterable: false` — set `filterable: true` to also filter on them.
- Multi-query is `ns.multiQuery({ queries, rerank_by: ["RRF"] })`, not `ns.query({ queries })`.
- `aggregate_by` is an object: `{ n: ["Count"] }`.
- `patch_by_filter` is `{ filters, patch }`. Vectors can't be patched.
- Cold queries on big namespaces take ~0.5–1s; call `ns.hintCacheWarm()` before latency-sensitive sessions.
