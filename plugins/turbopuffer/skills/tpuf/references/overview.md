# Turbopuffer Overview

Turbopuffer is a serverless vector and full-text search database built from first principles on object storage. It's fast, 10x cheaper than alternatives, and extremely scalable.

## Core Architecture

- **Compute-storage separation**: All durable state lives in object storage (S3/GCS). Compute nodes are stateless and can scale independently.
- **Multi-tier cache**: Object storage → NVMe SSD cache → memory cache. First query to a namespace is slow (~343ms at 1M docs); subsequent queries are fast (~8ms) due to cache locality.
- **SPFresh vector index**: Centroid-based ANN that incrementally updates.
- **Group commit**: Batches writes automatically for high throughput even with slow object storage (~200ms). Write latency: p50=285ms for 500kB.

## Key Concepts

### Namespaces

An isolated set of documents with its own vector index, FTS index, and attribute indexes. Names match `[A-Za-z0-9-_.]{1,128}`.

- **Implicitly created** on first write — there is no "create namespace" endpoint.
- Use one namespace per isolated document space rather than heavy filtering across a single namespace.
- Deleting a namespace removes all its data permanently.

### Document IDs

Every document has a unique ID within its namespace:
- Unsigned 64-bit integer (most efficient, 8 bytes)
- 128-bit UUID
- String up to 64 bytes

### Vectors

Dense embeddings stored in the `vector` attribute:
- All vectors in a namespace must have the same dimensionality.
- Encoded as JSON array of floats or base64 string (little-endian float32).
- Types: `[DIMS]f32` (default) or `[DIMS]f16` (faster, cheaper, ~50% less storage).

### Schema

Defines type and indexing behavior for each attribute. Automatically inferred from first write, or manually specified.

**Attribute types**: `string`, `int`, `uint`, `float`, `bool`, `uuid`, `datetime`
**Array types**: `[]string`, `[]int`, `[]uint`, `[]float`, `[]bool`, `[]uuid`, `[]datetime`
**Vector types**: `[DIMS]f32`, `[DIMS]f16`

By default, attributes are filterable (indexed). Set `filterable: false` for a 50% storage discount on attributes you won't filter on.

### Distance Metrics

Set on first write, **cannot be changed** after:
- `cosine_distance` — most common, measures angle between vectors
- `euclidean_squared` — measures squared L2 distance

### Read Consistency

- **Strong** (default): Sees all writes completed before the query started. Requires a round-trip to object storage.
- **Eventual**: May be up to 60 seconds stale, but sub-10ms latency. Use when freshness isn't critical.

## Authentication

- API key format: `tpuf_...`
- HTTP header: `Authorization: Bearer <API_KEY>`
- Environment variable: `TURBOPUFFER_API_KEY`

## Regions

Turbopuffer runs in multiple cloud regions. Specify via SDK `region` parameter:
- `aws-us-east-1`, `aws-us-west-2`, `aws-eu-west-1`, etc.
- `gcp-us-central1`, `gcp-us-east4`, etc.

## SDKs

- **TypeScript**: `@turbopuffer/turbopuffer`
- **Python**: `turbopuffer`
- **Go**: `github.com/turbopuffer/turbopuffer-go`
