# Common Mistakes When Using Turbopuffer

## 1. Not Checking Schema Before Querying

**Mistake**: Querying with `include_attributes` that don't exist in the namespace, or using the wrong attribute names.

**Fix**: Always call `ns.schema()` or `ns.metadata()` first to discover available attributes and their types before constructing queries.

---

## 2. Distance Metric is Immutable

**Mistake**: Trying to change `distance_metric` after the first write to a namespace.

**Fix**: The distance metric (`cosine_distance` or `euclidean_squared`) is set on the first write and cannot be changed. If you need a different metric, create a new namespace.

---

## 3. Namespaces Are Implicitly Created

**Mistake**: Searching for a "create namespace" API endpoint.

**Fix**: Namespaces are created automatically on the first write. Just write to the namespace name you want. There is no explicit create endpoint.

---

## 4. Cold Query Latency

**Mistake**: Expecting sub-10ms latency on the first query to a namespace.

**Fix**: The first query to a namespace reads from object storage (~343ms at 1M docs). Subsequent queries hit cache (~8ms). Use the warm cache endpoint (`POST /v2/namespaces/{ns}/warm`) if you need low latency on the first request.

---

## 5. `limit` vs `top_k`

**Mistake**: Using `top_k` in new code. The parameter was renamed to `limit`.

**Fix**: Use `limit` (integer or object). Both work, but `limit` is the current standard. Example: `"limit": 10` or `"limit": {"per": {"attributes": ["category"], "limit": 5}, "total": 50}`.

---

## 6. Creating New Client Instances Per Request

**Mistake**: Instantiating a new `Turbopuffer` client for every API call.

**Fix**: Create one client instance and reuse it. The SDK handles connection pooling internally.

---

## 7. Not Batching Writes

**Mistake**: Sending one document per write request.

**Fix**: Batch documents together in a single write call (up to 512MB per request). Larger batches get better throughput and a 50% batch discount on write costs. Use column format (`upsert_columns`) for maximum efficiency with bulk data.

---

## 8. Expensive Glob/Regex Filters

**Mistake**: Using `["name", "Glob", "*tpuf*"]` or `IGlob` — these scan every document.

**Fix**: Prefix globs like `["name", "Glob", "tpuf*"]` are optimized. For substring matching, use full-text search instead.

---

## 9. Including Unnecessary Attributes

**Mistake**: Using `include_attributes: true` (returns everything) when you only need a few fields.

**Fix**: Specify only the attributes you need: `include_attributes: ["title", "summary"]`. This reduces response size and latency.

---

## 10. Marking All Attributes as Filterable

**Mistake**: Leaving `filterable: true` (the default) on large text attributes you never filter on.

**Fix**: Set `filterable: false` on attributes you only need for retrieval (not filtering). This gives a 50% storage discount for those attributes.

---

## 11. FTS Tokenizer Mismatch

**Mistake**: Using `pre_tokenized_array` tokenizer but passing a string query, or vice versa.

**Fix**: If schema uses `pre_tokenized_array`, write and query with arrays of tokens: `["quick", "fox"]`. If using default `word_v3`, pass strings normally.

---

## 12. Not Cleaning Up Test Namespaces

**Mistake**: Creating test namespaces without deleting them.

**Fix**: Always use random suffixes for test namespace names and delete them in test teardown. Handle `NotFoundError` in cleanup (namespace may not have been created if test failed early).

---

## 13. Using Eventual Consistency After Writes

**Mistake**: Writing documents then immediately querying with `consistency: {level: "eventual"}` and expecting to see the new data.

**Fix**: Use strong consistency (the default) when you need to read your own writes. Eventual consistency may be up to 60 seconds stale.

---

## 14. Large Attributes with Frequent Patches

**Mistake**: Storing large blobs (>10KB) in attributes that get frequently patched.

**Fix**: Store large attributes in a separate namespace linked by ID. Patches re-write the entire row, so large attributes make patches expensive.

---

## 15. Wrong Vector Dimensions

**Mistake**: Writing vectors with inconsistent dimensions to the same namespace.

**Fix**: All vectors in a namespace must have the same dimensionality. The dimension is set by the first write. Use `[DIMS]f16` instead of `[DIMS]f32` to save ~50% storage with minimal quality loss.
