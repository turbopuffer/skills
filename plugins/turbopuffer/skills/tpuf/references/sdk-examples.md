# Turbopuffer SDK Examples

## Client Initialization

### TypeScript
```typescript
import { Turbopuffer } from "@turbopuffer/turbopuffer";

const tpuf = new Turbopuffer({
  apiKey: process.env.TURBOPUFFER_API_KEY,
  region: "gcp-us-central1",
});

const ns = tpuf.namespace("my-namespace");
```

### Python
```python
import turbopuffer
import os

tpuf = turbopuffer.Turbopuffer(
    api_key=os.getenv("TURBOPUFFER_API_KEY"),
    region="gcp-us-central1",
)

ns = tpuf.namespace("my-namespace")
```

### Go
```go
import (
    "github.com/turbopuffer/turbopuffer-go"
    "github.com/turbopuffer/turbopuffer-go/option"
)

tpuf := turbopuffer.NewClient(
    option.WithAPIKey(os.Getenv("TURBOPUFFER_API_KEY")),
    option.WithRegion("gcp-us-central1"),
)

ns := tpuf.Namespace("my-namespace")
```

---

## Write Documents

### TypeScript
```typescript
await ns.write({
  upsert_rows: [
    { id: 1, vector: [0.1, 0.2], category: "animal", text: "walrus narwhal" },
    { id: 2, vector: [0.3, 0.4], category: "fish", text: "pufferfish swordfish" },
  ],
  distance_metric: "cosine_distance",
  schema: {
    text: { type: "string", full_text_search: true },
  },
});
```

### Python
```python
ns.write(
    upsert_rows=[
        {"id": 1, "vector": [0.1, 0.2], "category": "animal", "text": "walrus narwhal"},
        {"id": 2, "vector": [0.3, 0.4], "category": "fish", "text": "pufferfish swordfish"},
    ],
    distance_metric="cosine_distance",
    schema={
        "text": {"type": "string", "full_text_search": True},
    },
)
```

### Go
```go
_, err := ns.Write(ctx, turbopuffer.NamespaceWriteParams{
    UpsertRows: []turbopuffer.RowParam{
        {"id": 1, "vector": []float32{0.1, 0.2}, "category": "animal", "text": "walrus narwhal"},
        {"id": 2, "vector": []float32{0.3, 0.4}, "category": "fish", "text": "pufferfish swordfish"},
    },
    DistanceMetric: turbopuffer.DistanceMetricCosineDistance,
})
```

---

## Vector Search (ANN)

### TypeScript
```typescript
const result = await ns.query({
  rank_by: ["vector", "ANN", [0.1, 0.2]],
  limit: 10,
  filters: ["And", [
    ["category", "Eq", "animal"],
    ["public", "Eq", 1],
  ]],
  include_attributes: ["category", "text"],
});

for (const row of result.rows!) {
  console.log(`ID: ${row.id}, Dist: ${row.$dist}`);
}
```

### Python
```python
result = ns.query(
    rank_by=("vector", "ANN", [0.1, 0.2]),
    limit=10,
    filters=("And", (
        ("category", "Eq", "animal"),
        ("public", "Eq", 1),
    )),
    include_attributes=["category", "text"],
)

for row in result.rows:
    print(f"ID: {row.id}, Dist: {row['$dist']}")
```

### Go
```go
res, err := ns.Query(ctx, turbopuffer.NamespaceQueryParams{
    RankBy: turbopuffer.NewRankByVector("vector", []float32{0.1, 0.2}),
    Limit:  turbopuffer.NamespaceQueryParamsLimit{Int: turbopuffer.Int(10)},
})

for _, row := range res.Rows {
    fmt.Printf("ID: %v, Dist: %v\n", row["id"], row["$dist"])
}
```

---

## Full-Text Search (BM25)

### TypeScript
```typescript
const result = await ns.query({
  rank_by: ["content", "BM25", "quick walrus"],
  limit: 10,
  include_attributes: ["content"],
});
```

### Python
```python
result = ns.query(
    rank_by=("content", "BM25", "quick walrus"),
    limit=10,
    include_attributes=["content"],
)
```

### Weighted multi-field BM25 (TypeScript)
```typescript
const result = await ns.query({
  rank_by: ["Sum", [
    ["Product", 3, ["title", "BM25", "python beginner"]],
    ["Product", 2, ["tags", "BM25", "python beginner"]],
    ["content", "BM25", "python beginner"],
  ]],
  limit: 10,
  include_attributes: ["title", "content"],
});
```

---

## Hybrid Search (Vector + BM25)

### TypeScript — Multi-query with Reciprocal Rank Fusion
```typescript
const response = await ns.query({
  queries: [
    {
      rank_by: ["vector", "ANN", queryVector],
      limit: 10,
      include_attributes: ["content"],
    },
    {
      rank_by: ["content", "BM25", "search query"],
      limit: 10,
      include_attributes: ["content"],
    },
  ],
});

const [vectorResult, ftsResult] = response.results!;

// Reciprocal Rank Fusion
function rrf(resultLists: any[][], k = 60) {
  const scores: Record<string, number> = {};
  const items: Record<string, any> = {};
  for (const results of resultLists) {
    for (let rank = 0; rank < results.length; rank++) {
      const id = results[rank].id;
      scores[id] = (scores[id] || 0) + 1 / (k + rank + 1);
      items[id] = results[rank];
    }
  }
  return Object.entries(scores)
    .sort(([, a], [, b]) => b - a)
    .map(([id]) => items[id]);
}

const fusedResults = rrf([vectorResult.rows!, ftsResult.rows!]);
```

### Python — Multi-query
```python
response = ns.multi_query(
    queries=[
        {
            "rank_by": ("vector", "ANN", query_vector),
            "limit": 10,
            "include_attributes": ["content"],
        },
        {
            "rank_by": ("content", "BM25", "search query"),
            "limit": 10,
            "include_attributes": ["content"],
        },
    ]
)

vector_result, fts_result = response.results[0].rows, response.results[1].rows
```

---

## Delete Operations

### TypeScript
```typescript
// Delete by ID
await ns.write({ deletes: [1, 2, 3] });

// Delete by filter
await ns.write({
  delete_by_filter: ["category", "Eq", "obsolete"],
});

// Delete entire namespace
await ns.deleteAll();
```

### Python
```python
# Delete by ID
ns.write(deletes=[1, 2, 3])

# Delete by filter
ns.write(delete_by_filter=["category", "Eq", "obsolete"])

# Delete entire namespace
ns.delete_all()
```

---

## Schema & Metadata

### TypeScript
```typescript
// Get schema
const schema = await ns.schema();

// Get metadata (includes schema, row count, size, index status)
const metadata = await ns.metadata();
console.log(`Rows: ${metadata.approx_row_count}`);
console.log(`Size: ${metadata.approx_logical_bytes} bytes`);

// Update schema
await ns.updateSchema({
  schema: {
    text: { type: "string", full_text_search: true },
  },
});
```

### Python
```python
# Get schema
schema = ns.schema()

# Get metadata
metadata = ns.metadata()
print(f"Rows: {metadata.approx_row_count}")

# List namespaces
for namespace in tpuf.namespaces():
    print(namespace.id)
```

---

## Testing Patterns

### TypeScript (Vitest)
```typescript
import { expect, test, beforeEach, afterEach, describe } from "vitest";
import { NotFoundError, Turbopuffer } from "@turbopuffer/turbopuffer";
import * as crypto from "crypto";

const tpuf = new Turbopuffer({ region: "gcp-us-central1" });

describe("turbopuffer tests", () => {
  let ns: any;

  beforeEach(async () => {
    const suffix = crypto.randomBytes(16).toString("hex");
    ns = tpuf.namespace(`test-${suffix}`);
  });

  afterEach(async () => {
    try { await ns.deleteAll(); }
    catch (e: any) { if (!(e instanceof NotFoundError)) throw e; }
  });

  test("vector search", async () => {
    await ns.write({
      upsert_rows: [{ id: 1, vector: [1, 1] }, { id: 2, vector: [2, 2] }],
      distance_metric: "cosine_distance",
    });
    const res = await ns.query({
      rank_by: ["vector", "ANN", [1.1, 1.1]],
      limit: 10,
    });
    expect(res.rows![0].id).toBe(1);
  });
});
```

### Python (pytest)
```python
import pytest, random, string
import turbopuffer

tpuf = turbopuffer.Turbopuffer(region="gcp-us-central1")

@pytest.fixture
def tpuf_ns():
    suffix = "".join(random.choices(string.ascii_letters + string.digits, k=32))
    ns = tpuf.namespace(f"test-{suffix}")
    try:
        yield ns
    finally:
        try:
            ns.delete_all()
        except turbopuffer.NotFoundError:
            pass

def test_vector_search(tpuf_ns):
    tpuf_ns.write(
        upsert_rows=[{"id": 1, "vector": [1, 1]}, {"id": 2, "vector": [2, 2]}],
        distance_metric="cosine_distance",
    )
    res = tpuf_ns.query(rank_by=("vector", "ANN", [1.1, 1.1]), limit=10)
    assert res.rows[0].id == 1
```
