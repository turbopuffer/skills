# Query — Search and Retrieve Data

Always check schema first:
```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  console.log(JSON.stringify(await ns.schema(), null, 2));
}
```

## Vector search (ANN)

```typescript
async function run(client) {
  const ns = client.namespace("articles");
  const result = await ns.query({
    rank_by: ["vector", "ANN", [0.1, 0.2, 0.3]],
    limit: 10,
    include_attributes: ["title", "$dist" "content"],
  });
  for (const row of result.rows!) {
    console.log(`ID: ${row.id}, Dist: ${row.$dist}, Title: ${(row.title as string)?.substring(0, 200)}`);
  }
}
```

## Full-text search (BM25)

Requires attribute with `full_text_search: true`.

```typescript
async function run(client) {
  const ns = client.namespace("articles");
  const result = await ns.query({
    rank_by: ["content", "BM25", "search terms"],
    limit: 10,
    include_attributes: ["title", "content"],
  });
  for (const row of result.rows!) {
    console.log(`ID: ${row.id}, Title: ${(row.title as string)?.substring(0, 200)}`);
  }
}
```

## Weighted multi-field BM25

```typescript
await ns.query({
  rank_by: ["Sum", [
    ["Product", 3, ["title", "BM25", "query"]],
    ["Product", 2, ["tags", "BM25", "query"]],
    ["content", "BM25", "query"],
  ]],
  limit: 10,
})
```

## Order by attribute

```typescript
await ns.query({ rank_by: ["created_at", "desc"], limit: 20, filters: ["status", "Eq", "active"] })
```

## Hybrid search (vector + BM25)

Use multi-query to run vector and BM25 searches in parallel, then fuse results client-side (e.g., Reciprocal Rank Fusion). Use the `search_docs` tool to look up "hybrid search" for the full RRF implementation example.

```typescript
async function run(client) {
  const ns = client.namespace("articles");
  const result = await ns.query({
    queries: [
      { rank_by: ["vector", "ANN", [0.1, 0.2]], limit: 20, include_attributes: ["title"] },
      { rank_by: ["content", "BM25", "search query"], limit: 20, include_attributes: ["title"] },
    ],
  });
  // result.results[0].rows = vector results
  // result.results[1].rows = BM25 results
  // Fuse with RRF — use search_docs for the full example
  console.log(JSON.stringify(result, null, 2).substring(0, 2000));
}
```

## Filter syntax

Only `filterable: true` attributes can be filtered.

| Pattern | Example |
|---------|---------|
| Exact match | `["status", "Eq", "active"]` |
| Combined | `["And", [["category", "Eq", "tech"], ["score", "Gte", 0.5]]]` |
| Array contains | `["tags", "Contains", "python"]` |
| Prefix glob | `["name", "Glob", "tpuf*"]` |
| In set | `["id", "In", [1, 2, 3]]` |
| Null check | `["name", "NotEq", null]` |

## Validation

If results look wrong:
- Empty results? Check: is the attribute filterable? Is FTS enabled on the right field? Are vector dimensions correct?
- Slow? First query to cold namespace is ~300ms — subsequent queries hit cache (~8ms).
- Use `search_docs` tool to look up advanced rank_by patterns (Saturate, Decay, etc.).
