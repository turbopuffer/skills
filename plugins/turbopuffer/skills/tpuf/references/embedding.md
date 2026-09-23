# Native Embeddings

turbopuffer embeds a `string` attribute on write and query text at query time. No provider code needed.

```typescript
await ns.write({
  upsert_rows: [{ id: 1, text: "A cat sleeping on a windowsill" }],
  distance_metric: "cosine_distance",
  schema: { text: { type: "string", embed: { model: "nvidia/nemotron-3-embed-8b", dims: 1024 } } },
});

await ns.query({ rank_by: ["text", "ANN", ["Embed", "sleepy pet"]], limit: 10 });
```

- Vectors are stored in `embed_<attr>` by default (set `attribute` to use an existing column).
- If you already have document vectors, embed only the query: `["vector", "ANN", ["Embed", q, { model }]]`, using the same model.
- Rows that include a vector are stored as-is, so you can backfill precomputed vectors.
- Enable on an existing namespace with a schema write and `attribute: "vector"`. Existing rows are not re-embedded.
- `embed` can be added or removed, not changed. Switching models means re-ingesting.

Models and prices: https://turbopuffer.com/docs/embedding.md (e.g. `nvidia/nemotron-3-embed-8b`, `voyage/voyage-4`, `openai/text-embedding-3-small`, `cohere/embed-v4.0`, `qwen/qwen3-embedding-8b`).

Limits: 4 embedded attributes per namespace and 30 rows per upsert. Rate limits are per model; exceeding them returns 429.
