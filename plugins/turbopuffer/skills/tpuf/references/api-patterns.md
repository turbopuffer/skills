# Turbopuffer API Patterns

## Endpoints

| Operation | Method | Path |
|-----------|--------|------|
| Write (upsert/delete) | POST | `/v2/namespaces/{namespace}` |
| Query | POST | `/v2/namespaces/{namespace}/query` |
| Get schema | GET | `/v1/namespaces/{namespace}/schema` |
| Update schema | PATCH | `/v1/namespaces/{namespace}/schema` |
| Get metadata | GET | `/v1/namespaces/{namespace}/metadata` |
| List namespaces | GET | `/v1/namespaces` |
| Delete namespace | DELETE | `/v2/namespaces/{namespace}` |
| Export documents | GET | `/v2/namespaces/{namespace}/export` |
| Warm cache | POST | `/v2/namespaces/{namespace}/warm` |

---

## Write Operations

**Endpoint**: `POST /v2/namespaces/{namespace}`

### Upsert (row format)
```json
{
  "upsert_rows": [
    {"id": 1, "vector": [0.1, 0.2, 0.3], "name": "foo", "tags": ["a", "b"]},
    {"id": 2, "vector": [0.4, 0.5, 0.6], "name": "bar", "tags": ["c"]}
  ],
  "distance_metric": "cosine_distance"
}
```

### Upsert (column format — efficient for bulk)
```json
{
  "upsert_columns": {
    "id": [1, 2],
    "vector": [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6]],
    "name": ["foo", "bar"]
  },
  "distance_metric": "cosine_distance"
}
```

### Patch (update specific attributes, no vector changes)
```json
{
  "patch_rows": [
    {"id": 1, "name": "updated_foo"},
    {"id": 2, "name": "updated_bar"}
  ]
}
```

### Delete by ID
```json
{
  "deletes": [1, 2, 3]
}
```

### Delete by filter
```json
{
  "delete_by_filter": ["category", "Eq", "obsolete"]
}
```

### Patch by filter
```json
{
  "patch_by_filter": ["status", "Eq", "draft"],
  "patch_rows": [{"status": "published"}]
}
```

### Write with schema
```json
{
  "upsert_rows": [...],
  "distance_metric": "cosine_distance",
  "schema": {
    "text": {
      "type": "string",
      "full_text_search": true
    },
    "vector": {
      "type": "[768]f16",
      "ann": true
    },
    "metadata": {
      "type": "string",
      "filterable": false
    }
  }
}
```

### Write response
```json
{
  "status": "OK",
  "rows_affected": 100,
  "rows_upserted": 50,
  "rows_patched": 30,
  "rows_deleted": 20,
  "billing": {"billable_logical_bytes_written": 5000},
  "performance": {"server_total_ms": 285}
}
```

---

## Query Operations

**Endpoint**: `POST /v2/namespaces/{namespace}/query`

### rank_by Patterns

**Vector ANN (approximate nearest neighbor)**:
```json
{"rank_by": ["vector", "ANN", [0.1, 0.2, 0.3]]}
```

**Vector kNN (exact, requires filters)**:
```json
{"rank_by": ["vector", "kNN", [0.1, 0.2, 0.3]]}
```

**BM25 full-text search**:
```json
{"rank_by": ["content", "BM25", "search query"]}
```

**Order by attribute**:
```json
{"rank_by": ["timestamp", "desc"]}
```

**Weighted multi-field BM25**:
```json
{
  "rank_by": ["Sum", [
    ["Product", 3, ["title", "BM25", "python"]],
    ["Product", 2, ["tags", "BM25", "python"]],
    ["content", "BM25", "python"]
  ]]
}
```

**Boost by attribute value (Saturate/Decay)**:
```json
{
  "rank_by": ["Sum", [
    ["content", "BM25", "search query"],
    ["Product", 1.5, ["Saturate", ["Attribute", "clicks"], {"midpoint": 100}]],
    ["Decay", ["Dist", ["Attribute", "published_at"], "2026-03-01"], {"midpoint": "7d"}]
  ]]
}
```

### Filter Syntax

Filters use a tuple-based syntax: `["attribute", "operator", value]`

**Operators**:
| Operator | Description | Example |
|----------|-------------|---------|
| `Eq` | Equals | `["status", "Eq", "active"]` |
| `NotEq` | Not equals | `["status", "NotEq", "deleted"]` |
| `In` | In set | `["id", "In", [1, 2, 3]]` |
| `NotIn` | Not in set | `["id", "NotIn", [4, 5]]` |
| `Lt`, `Lte` | Less than (or equal) | `["price", "Lt", 100]` |
| `Gt`, `Gte` | Greater than (or equal) | `["score", "Gte", 0.5]` |
| `Contains` | Array contains value | `["tags", "Contains", "python"]` |
| `NotContains` | Array doesn't contain | `["tags", "NotContains", "spam"]` |
| `ContainsAny` | Array contains any of | `["tags", "ContainsAny", ["a", "b"]]` |
| `Glob` | Glob pattern match | `["name", "Glob", "tpuf*"]` |
| `Regex` | Regex match | `["name", "Regex", "^tpuf.*"]` |
| `ContainsAllTokens` | FTS token match | `["text", "ContainsAllTokens", "quick fox"]` |
| `ContainsAnyToken` | FTS any token | `["text", "ContainsAnyToken", "quick fox"]` |

**Combining filters**:
```json
["And", [
  ["status", "Eq", "active"],
  ["Or", [
    ["category", "Eq", "A"],
    ["category", "Eq", "B"]
  ]]
]]
```

**Null checks**:
```json
["name", "Eq", null]
["name", "NotEq", null]
```

### Complete query example
```json
{
  "rank_by": ["vector", "ANN", [0.1, 0.2, 0.3]],
  "limit": 10,
  "filters": ["And", [
    ["category", "Eq", "article"],
    ["published", "Eq", true]
  ]],
  "include_attributes": ["title", "category", "score"]
}
```

### Query response
```json
{
  "rows": [
    {"$dist": 0.12, "id": 42, "title": "Example", "category": "article", "score": 0.95},
    {"$dist": 0.34, "id": 17, "title": "Another", "category": "article", "score": 0.88}
  ],
  "billing": {
    "billable_logical_bytes_queried": 10000,
    "billable_logical_bytes_returned": 500
  },
  "performance": {
    "cache_hit_ratio": 0.95,
    "cache_temperature": "hot",
    "server_total_ms": 8
  }
}
```

### Multi-query (for hybrid search)
```json
{
  "queries": [
    {"rank_by": ["vector", "ANN", [0.1, 0.2]], "limit": 10, "include_attributes": ["text"]},
    {"rank_by": ["content", "BM25", "search query"], "limit": 10, "include_attributes": ["text"]}
  ]
}
```

### Aggregations
```json
{
  "aggregate_by": ["Count"],
  "filters": ["category", "Eq", "article"]
}
```

```json
{
  "aggregate_by": ["Sum", "price"],
  "group_by": ["category"],
  "filters": ["status", "Eq", "active"]
}
```

---

## Schema Operations

**Get schema**: `GET /v1/namespaces/{namespace}/schema`

**Update schema**: `PATCH /v1/namespaces/{namespace}/schema`
```json
{
  "schema": {
    "text": {
      "type": "string",
      "full_text_search": {
        "tokenizer": "word_v3",
        "language": "english",
        "stemming": true,
        "remove_stopwords": true,
        "case_sensitive": false
      }
    }
  }
}
```

### Full-Text Search Config Options
- `tokenizer`: `word_v3` (default), `word_v2`, `word_v1`, `pre_tokenized_array`
- `language`: `english`, `french`, `german`, `spanish`, etc. (16 languages supported)
- `stemming`: `true`/`false` — language-aware word stemming
- `remove_stopwords`: `true`/`false` — remove common words
- `case_sensitive`: `true`/`false`
- `ascii_folding`: `true`/`false` — normalize accented characters
- `k1`: BM25 term frequency saturation (default 1.2)
- `b`: BM25 document length normalization (default 0.75)

---

## Namespace Metadata

**Endpoint**: `GET /v1/namespaces/{namespace}/metadata`

```json
{
  "schema": {"name": {"type": "string", "filterable": true}},
  "approx_row_count": 1000000,
  "approx_logical_bytes": 500000000,
  "created_at": "2024-03-15T10:30:45Z",
  "updated_at": "2024-04-16T09:27:32Z",
  "index": {"status": "up-to-date"},
  "encryption": {"mode": "default"}
}
```

---

## List Namespaces

**Endpoint**: `GET /v1/namespaces`

**Parameters**: `prefix`, `cursor`, `page_size` (1-1000)

```json
{
  "namespaces": [{"id": "ns-1"}, {"id": "ns-2"}],
  "next_cursor": "..."
}
```

---

## Delete Namespace

**Endpoint**: `DELETE /v2/namespaces/{namespace}`

Returns `{"status": "OK"}`. This is permanent and cannot be undone.
