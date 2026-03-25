# Setup — Install SDK, Configure API Key, Scaffold Example

## Steps

1. **Detect the project language** from `package.json` (TS/JS), `pyproject.toml`/`requirements.txt` (Python), or `go.mod` (Go).

2. **Install the SDK:**
   - **TypeScript/JS**: `npm install @turbopuffer/turbopuffer`
   - **Python**: `pip install turbopuffer`
   - **Go**: `go get github.com/turbopuffer/turbopuffer-go`

3. **Configure the API key.** Add `TURBOPUFFER_API_KEY=` to `.env`. Add `.env` to `.gitignore`.

4. **Ask which region** (default: none). Options: `gcp-us-central1`, `aws-us-east-1`, `aws-us-west-2`, `aws-eu-west-1`.

5. **Write a basic example file** for the detected language:

### TypeScript

```typescript
import { Turbopuffer } from "@turbopuffer/turbopuffer";

const tpuf = new Turbopuffer({ apiKey: process.env.TURBOPUFFER_API_KEY! });
const ns = tpuf.namespace("example");

await ns.write({
  upsert_rows: [
    { id: 1, vector: [0.1, 0.2, 0.3], title: "First document" },
    { id: 2, vector: [0.4, 0.5, 0.6], title: "Second document" },
  ],
  distance_metric: "cosine_distance",
});

const results = await ns.query({
  rank_by: ["vector", "ANN", [0.1, 0.2, 0.3]],
  limit: 10,
  include_attributes: ["title"],
});
console.log(results.rows);
```

### Python

```python
import os, turbopuffer

tpuf = turbopuffer.Turbopuffer(api_key=os.getenv("TURBOPUFFER_API_KEY"))
ns = tpuf.namespace("example")

ns.write(
    upsert_rows=[
        {"id": 1, "vector": [0.1, 0.2, 0.3], "title": "First document"},
        {"id": 2, "vector": [0.4, 0.5, 0.6], "title": "Second document"},
    ],
    distance_metric="cosine_distance",
)

results = ns.query(rank_by=("vector", "ANN", [0.1, 0.2, 0.3]), limit=10, include_attributes=["title"])
for row in results.rows:
    print(f"ID: {row.id}, Dist: {row['$dist']}, Title: {row['title']}")
```

### Go

```go
tpuf := turbopuffer.NewClient(option.WithAPIKey(os.Getenv("TURBOPUFFER_API_KEY")))
ns := tpuf.Namespace("example")

ns.Write(ctx, turbopuffer.NamespaceWriteParams{
    UpsertRows: []turbopuffer.RowParam{
        {"id": 1, "vector": []float32{0.1, 0.2, 0.3}, "title": "First document"},
    },
    DistanceMetric: turbopuffer.DistanceMetricCosineDistance,
})

res, _ := ns.Query(ctx, turbopuffer.NamespaceQueryParams{
    RankBy: turbopuffer.NewRankByVector("vector", []float32{0.1, 0.2, 0.3}),
    Limit:  turbopuffer.NamespaceQueryParamsLimit{Int: turbopuffer.Int(10)},
})
```

6. **Verify** — run the example and confirm it completes without errors.

## Gotchas

- Create ONE client instance and reuse it — SDK handles connection pooling.
- Don't forget `distance_metric` on the first write — required and immutable.
- If project already has turbopuffer set up, skip to the next step.
