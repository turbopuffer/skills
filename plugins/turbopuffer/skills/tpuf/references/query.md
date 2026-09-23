# Query

```typescript
// Vector
await ns.query({ rank_by: ["vector", "ANN", v], limit: 10, filters: ["category", "Eq", "news"], include_attributes: ["title"] });

// Native embeddings
await ns.query({ rank_by: ["text", "ANN", ["Embed", "query text"]], limit: 10 });

// Exact kNN — requires filters
await ns.query({ rank_by: ["vector", "kNN", v], filters: ["user_id", "Eq", 42], limit: 10 });

// BM25 (weighted across fields)
await ns.query({ rank_by: ["Sum", [["Product", 2, ["title", "BM25", q]], ["content", "BM25", q]]], limit: 10 });

// Order by attribute / lookup
await ns.query({ rank_by: ["created_at", "desc"], filters: ["status", "Eq", "active"], limit: 20 });
```

`$dist` is returned on every row (distance for vectors, score for BM25).

## Hybrid

```typescript
const res = await ns.multiQuery({
  queries: [
    { rank_by: ["vector", "ANN", v], limit: 20 },
    { rank_by: ["content", "BM25", q], limit: 20 },
  ],
  rerank_by: ["RRF"], // fused list in res.results[0].rows
});
```

## Filters

`["attr", "Op", value]`, combined with `And` / `Or` / `Not`. Ops: `Eq`, `NotEq`, `In`, `NotIn`, `Lt(e)`, `Gt(e)`, `Contains`, `ContainsAny`, `Glob`, `IGlob`, `Regex`, `Fuzzy`, `ContainsAllTokens`, `ContainsTokenSequence`. `["x", "Eq", null]` matches missing values.

## Aggregations

```typescript
const r = await ns.query({ aggregate_by: { n: ["Count"], total: ["Sum", "price"] }, group_by: ["category"] });
// r.aggregation_groups (or r.aggregations without group_by)
```

Aggregations are expensive — keep them off hot paths on large namespaces.

## Pagination

- Scroll: filter `["id", "NotIn", seenIds]`.
- Full scan: `rank_by: ["id", "asc"]` and advance `["id", "Gt", lastId]` (max 10,000 per page).

## Consistency

Strong by default. `consistency: { level: "eventual" }` is faster, but can lag after heavy writes.
