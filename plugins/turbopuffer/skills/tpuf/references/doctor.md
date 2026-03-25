# Doctor — Diagnose Query, Performance, and Schema Issues

## Step 1: Collect baseline data

Always start by checking the namespace and running a diagnostic query:

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const metadata = await ns.metadata();
  console.log("Rows:", metadata.approx_row_count);
  console.log("Size:", metadata.approx_logical_bytes, "bytes");
  console.log("Index:", JSON.stringify(metadata.index, null, 2));
  console.log("Schema:", JSON.stringify(metadata.schema, null, 2));

  const result = await ns.query({ rank_by: ["id", "asc"], limit: 1 });
  console.log("Performance:", JSON.stringify(result.performance, null, 2));
}
```

Key fields to check:
- `cache_temperature` — `hot` (good), `warm`/`cold` (may explain latency)
- `server_total_ms` — query latency server-side
- `exhaustive_search_count` — unindexed docs scanned per query (high = large indexing backlog)
- `index.status` — `up-to-date` or `updating`
- `index.unindexed_bytes` — bytes waiting to be indexed

---

## Empty or missing results

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Attribute not `filterable: true` | Check schema — filter silently matches nothing | Update schema to enable filtering |
| FTS query on non-FTS attribute | Schema shows no `full_text_search` | Enable: `{ type: "string", full_text_search: true }` |
| Filter value doesn't match any docs | Run unfiltered query to see actual values | Adjust filter |
| Tokenizer mismatch | `pre_tokenized_array` but passing string (or vice versa) | Match format to tokenizer |
| Wrong namespace | Check `ns.metadata()` for row count | Use correct namespace |
| BM25 on chunked text | Document length scoring less useful on small chunks | Run BM25 on full document in separate namespace |

## Wrong or unexpected results

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Using `top_k` instead of `limit` | Check query params | Replace with `limit` |
| Vector dimension mismatch | Query vector has different dims than stored | Match dimensions exactly |
| Low ANN recall on dense embeddings | Common with financial/legal text where vectors cluster tightly | Use kNN with selective filters, or hybrid search |
| kNN scanning too much data | No filters bounding the search | Add filters to limit kNN to ~10k docs |
| BM25 scores inconsistent | Approximate statistics during active indexing | Wait for indexing to complete |

## Slow queries

### Expected latency ranges

| Scenario | Expected |
|----------|----------|
| Hot cache, simple query | 5-20ms |
| Hot cache, FTS (BM25) | p50 ~14ms, p90 ~57ms |
| Warm cache | 20-100ms |
| Cold cache (first query) | 200-500ms at 1M docs |
| Heavy write backlog | 500ms+ |

### Expensive patterns

| Pattern | Why it's slow | Fix |
|---------|--------------|-----|
| `Glob "*substring*"` or `IGlob` | Full namespace scan | Use prefix `Glob "prefix*"` or FTS |
| `Regex` filters | Full namespace scan | Use prefix Glob or FTS |
| `include_attributes: true` | Returns all attributes | List only needed fields |
| Large `limit` (>1000) | More ranking and serialization work | Reduce, paginate |
| Large `In` filter (500+ values) | Many index lookups | Split into batches or restructure |
| Complex `rank_by` expressions | Multi-stage scoring | Keep first-stage simple (100-1000 hits), re-rank in second stage |
| `ContainsTokenSequence` | Post-filter pass to verify token order | Works best with selective phrases; `ContainsAllTokens` is faster |
| kNN on large unfiltered set | Exhaustive distance computation | Add filters or use ANN |
| Parallel queries on cold namespace | Competing for cache resources | Warm first, then parallelize |

### Consistency mode impact

Use eventual when: p99 matters, staleness is acceptable, during bulk ingestion.
Use strong when: need to read your own writes, user-facing freshness matters.

### Cache warming

Cold namespace queries are slow because they read from object storage. Warm proactively:

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  await ns.query({ rank_by: ["id", "asc"], limit: 1 });
  const result = await ns.query({ rank_by: ["id", "asc"], limit: 1 });
  console.log("Cache:", result.performance?.cache_temperature);
  console.log("Latency:", result.performance?.server_total_ms, "ms");
}
```

Warm the cache before user-facing queries — e.g., when the user opens a search dialog.

---

## Schema optimization

### Audit filterable attributes

Every attribute defaults to `filterable: true`. Unused filters waste ~50% storage:

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const schema = await ns.schema();
  for (const [name, config] of Object.entries(schema)) {
    if (typeof config === 'object' && config !== null && 'type' in config) {
      const c = config as any;
      console.log(`${name}: type=${c.type}, filterable=${c.filterable !== false}, fts=${!!c.full_text_search}`);
    }
  }
}
```

| Finding | Fix |
|---------|-----|
| Large text attributes with `filterable: true` | Set `filterable: false` — 50% savings |
| `[DIMS]f32` vectors | Switch to `[DIMS]f16` — ~50% savings, minimal quality loss |
| Many filterable attributes rarely used in queries | Ask which ones are actually filtered on |
| Metadata/description fields that are filterable | Usually only need retrieval, not filtering |

### Apply schema fix

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  await ns.updateSchema({
    schema: {
      description: { type: "string", filterable: false },
      raw_content: { type: "string", filterable: false },
    },
  });
  console.log("Schema updated");
}
```

### Other optimizations

- **U64 IDs** instead of string IDs — smaller index, faster lookups
- **Smaller vector dimensions** (512 vs 1536) — proportionally faster
- **Reuse client instance** — SDK manages connection pooling; new client per request wastes TCP+TLS handshake
- **Choose closest region** — can't beat speed of light

---

## Gotchas

- **Filters silently return empty** if the attribute isn't `filterable: true`. No error — just no results.
- **Multi-query consistency must be top-level.** Setting `consistency` per-query inside `queries` array is silently ignored — strong consistency is used instead.
- **Cache eviction** — infrequently queried namespaces get evicted after hours of inactivity.
- **`distance_metric` is immutable** — set on first write, cannot be changed. If wrong, create a new namespace.
- **kNN requires filters** — without them it scans the entire namespace exhaustively.
- **429 errors** — load `references/troubleshoot-429.md` for detailed diagnosis.

## Checklist

- [ ] Collect baseline: schema, metadata, performance object
- [ ] If empty results: check filterable, FTS enablement, tokenizer, filter values
- [ ] If wrong results: check dimensions, ANN vs kNN, consistency
- [ ] If slow (hot cache): check query patterns — Glob, large In, include_attributes, limit
- [ ] If slow (cold cache): warm namespace proactively
- [ ] If p99 matters: try eventual consistency
- [ ] If storage is high: audit filterable attributes, consider f16 vectors
- [ ] If recall issues: consider sharding, kNN with filters, hybrid search
- [ ] If 429: load `references/troubleshoot-429.md`
