# Term-level queries

Term-level queries match **exact values** against indexed data. Unlike full-text queries, they do not analyze the input — what you send is matched directly against the inverted index. Use them for structured data: IDs, statuses, dates, numbers, keywords.

---

## term

Single exact value. Case-sensitive. Use on `keyword` fields — avoid on `text` fields (analyzed tokens won't match what you expect).

```json
"query": {
  "term": {
    "status": { "value": "published" }
  }
}
```

---

## terms

Match any value from a list. SQL `IN (...)` equivalent. Returns docs matching any of the values.

```json
"query": {
  "terms": {
    "tag": ["python", "go", "rust"]
  }
}
```

---

## terms lookup

Fetch the terms list from another document instead of hardcoding it. Useful for dynamic lists (e.g. a user's followed IDs).

- `index` — index containing the source doc
- `id` — ID of the source doc
- `path` — field in the source doc containing the values

```json
"terms": {
  "author_id": {
    "index": "users",
    "id":    "user_42",
    "path":  "following"
  }
}
```

---

## ids

Match documents by `_id` directly.

```json
"query": {
  "ids": {
    "values": ["1", "4", "100"]
  }
}
```

---

## exists

Field has a non-null value. Invert with `must_not` to check for missing fields.

```json
{ "exists": { "field": "published_at" } }

// field is missing
"must_not": [
  { "exists": { "field": "deleted_at" } }
]
```

---

## range

Numeric, date, or string ranges. Supports `gt`, `gte`, `lt`, `lte`. Date math supported.

```json
"range": {
  "created_at": {
    "gte": "now-7d/d",
    "lt":  "now/d"
  }
}

"range": { "price": { "gte": 10, "lte": 50 } }
```

---

## prefix

Field value starts with a string. Operates on unanalyzed terms (`keyword` fields).

- Expensive on high-cardinality fields
- Use `index_prefixes` mapping parameter to pre-index at index time (shifts cost from query to index time)

```json
"prefix": { "username": { "value": "ali" } }

// mapping — pre-index prefixes for fast prefix search
"title": {
  "type": "text",
  "index_prefixes": { "min_chars": 2, "max_chars": 5 }
}
```

---

## wildcard

Glob-style pattern matching. `*` = any characters, `?` = one character.

- Leading `*` requires a full index scan — avoid
- Can be disabled cluster-wide (see below)

```json
"wildcard": {
  "email": { "value": "a*@gmail.com" }
}
```

---

## fuzzy

Matches within edit distance (Levenshtein). Typo-tolerant matching.

- `fuzziness`: 0, 1, 2 or `AUTO` (AUTO: 0 for 1-2 chars, 1 for 3-5, 2 for 6+)
- `prefix_length` — chars that must match exactly (improves performance)
- `max_expansions` — caps the number of term variants generated

```json
"fuzzy": {
  "title": {
    "value":         "elasticsaerch",
    "fuzziness":     "AUTO",
    "prefix_length": 3,
    "max_expansions": 10
  }
}
```

---

## allow_expensive_queries

Cluster-level kill switch for costly query types. Disables `wildcard`, `prefix`, `fuzzy`, `regexp` when set to `false`.

```json
PUT _cluster/settings
{
  "transient": {
    "search.allow_expensive_queries": false
  }
}
```

---

## Query context vs filter context

Term-level queries run in **query context** by default and produce a score (IDF-based — not meaningful for exact matches). Wrap in `filter` to skip scoring and enable caching.

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "elasticsearch" } }
      ],
      "filter": [
        { "term":  { "status": "published" } },
        { "range": { "price": { "lte": 50 } } }
      ]
    }
  }
}
```

| Context | Scores? | Cached? | How to invoke |
|---|---|---|---|
| Query | Yes (IDF-based) | No | `"query": { "term": {} }` |
| Filter | No (always 0) | Yes | `"bool": { "filter": [ { "term": {} } ] }` |

> **Rule of thumb:** if a term/range/exists clause shouldn't influence ranking, put it in `filter`.

---

## Quick reference

| Query | Use for |
|---|---|
| `term` / `terms` | Exact match on keyword/numeric fields |
| `terms lookup` | Dynamic value lists fetched from another doc |
| `ids` | Match by `_id` |
| `exists` | Field presence check |
| `range` | Numeric, date, string ranges |
| `prefix` | Starts-with (use `index_prefixes` to speed up) |
| `wildcard` | Glob-style pattern (use sparingly) |
| `fuzzy` | Typo-tolerant matching |
