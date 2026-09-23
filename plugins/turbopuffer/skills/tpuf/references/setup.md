# Setup

1. Install: `npm install @turbopuffer/turbopuffer` · `pip install turbopuffer` · `go get github.com/turbopuffer/turbopuffer-go/v2`
2. Add `TURBOPUFFER_API_KEY` and `TURBOPUFFER_REGION` to `.env` (gitignored). Pick the region closest to the backend: https://turbopuffer.com/docs/regions
3. Decide: let turbopuffer embed text (`references/embedding.md`), or send your own vectors.

```typescript
import { Turbopuffer } from "@turbopuffer/turbopuffer";

const tpuf = new Turbopuffer({ region: process.env.TURBOPUFFER_REGION }); // reuse one client
const ns = tpuf.namespace("example");

await ns.write({
  upsert_rows: [
    { id: 1, text: "A cat sleeping on a windowsill" },
    { id: 2, text: "A shiny red sports car" },
  ],
  distance_metric: "cosine_distance",
  schema: { text: { type: "string", embed: "nvidia/nemotron-3-embed-8b", full_text_search: true } },
});

const res = await ns.query({
  rank_by: ["text", "ANN", ["Embed", "a sleepy pet"]],
  limit: 10,
  include_attributes: ["text"],
});
```

With your own vectors, write `vector: [...]` and query `["vector", "ANN", queryVector]`.
