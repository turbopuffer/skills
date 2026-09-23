# Full-Text Search

Enable on `string` or `[]string` attributes:

```typescript
schema: {
  title: { type: "string", full_text_search: { language: "english", stemming: true } },
  sku: { type: "string", full_text_search: true, filterable: true }, // FTS is not filterable by default
}
```

Options: `tokenizer` (default `word_v4`), `language`, `stemming`, `remove_stopwords`, `case_sensitive`, `ascii_folding`, `k1`, `b`.

## Query patterns

```typescript
rank_by: ["title", "BM25", "lazy wal", { last_as_prefix: true }]         // type-ahead
filters: ["content", "ContainsTokenSequence", "walrus is lazy"]          // phrase
rank_by: ["Sum", [["title", "BM25", q], ["Saturate", ["Attribute", "clicks"], { midpoint: 100 }]]] // boost
compute_attributes: { snippets: ["Highlight", "content"] }               // highlighting
```

`Glob`, `Regex`, and `Fuzzy` filters need `glob: true`, `regex: true`, or `fuzzy: true` in the schema.

Docs: https://turbopuffer.com/docs/fts.md
