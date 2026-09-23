# Doctor

Start by collecting the schema and index state, then rerun the failing query and check its `performance`: `cache_temperature`, `server_total_ms`, and `exhaustive_search_count`.

## Empty results

- The attribute isn't `filterable`. FTS, regex, glob, and fuzzy attributes default to non-filterable.
- Glob or Regex is used without `glob: true` / `regex: true` in the schema.
- BM25 is run on an attribute without `full_text_search`.
- The filter value's type doesn't match the schema (e.g. uuid, datetime).
- 409 response: an index is still building.

## Wrong results

- Vector dims differ, or queries use a different embedding model than the documents.
- Low recall: check with `ns.recall({ num: 25 })`. Try kNN with filters, or hybrid search.
- Stale results: eventual consistency is on. Use strong consistency.

## Slow queries

- Cold cache: call `ns.hintCacheWarm()`, or pin large, busy namespaces.
- Heavy query shapes: `include_attributes: true`, unanchored `Glob "*x*"`, huge `In` lists, complex `rank_by`, or aggregations on large namespaces.
- High `exhaustive_search_count` means an indexing backlog. See `references/ingestion.md`.

## Cheaper storage

Set `filterable: false` on attributes you never filter. Use `f16` or `i8` vectors and `uuid` types.
