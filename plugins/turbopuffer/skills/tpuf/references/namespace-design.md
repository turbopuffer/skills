# Namespace Design — Sizing, Sharding, Naming

## Key limits per namespace

| Metric | Limit |
|--------|-------|
| Max documents | 500M |
| Max size | 2 TB |
| Max write throughput | 10k writes/s @ 32 MB/s |
| Max concurrent queries | 16 (more with read replicas) |
| Max attributes | 256 |

## When to use one namespace

Use a single namespace when:
- Data shares the same schema and vector dimensions
- Total size < 500M docs / 2TB
- Query throughput fits within limits (16 concurrent, ~300 QPS at 50ms latency)
- All queries filter within this dataset

## When to shard across namespaces

**By tenant/user** — most common:
- Each user/tenant gets their own namespace: `user-{userId}`
- Natural isolation, simple access control, independent scaling
- Best when users don't need to search across each other's data

**By data type/schema**:
- Different document types with different schemas → separate namespaces
- e.g., `articles`, `users`, `products`

**By size** — when hitting per-namespace limits:
- Shard by ID: `docs-shard-{id % N}`
- Requires application-level scatter-gather for queries

**By performance** — when throughput matters:
- Each namespace gets its own concurrent query slots
- 10 namespaces × 16 concurrent = 160 concurrent queries
- Writes process at 1 batch/s per namespace — more namespaces = more write throughput

## Naming conventions

Names must match `[A-Za-z0-9-_.]{1,128}`.

Recommended patterns:
- `{tenant}-{type}` — `acme-corp-documents`
- `{type}-v{version}` — `articles-v2` (for embedding model rollouts)
- `test-{random}` — for test namespaces (always clean up)

## Namespace lifecycle

- **Created implicitly** on first write — no create endpoint
- **Deleted explicitly** with `ns.deleteAll()` — permanent, cannot be undone
- **Cannot rename** — create new namespace, migrate data, delete old one
- **Schema and distance metric are immutable** after first write — plan ahead

## Concurrent indexing

Multiple namespaces can be indexed concurrently, but there are cluster-level limits. If you're creating many namespaces simultaneously and writing to all of them, indexing may bottleneck. Spread writes over time or prioritize namespaces.
