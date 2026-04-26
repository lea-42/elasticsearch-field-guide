# Intro to Search

## Query vs Filter

|  | Query | Filter |
|---|---|---|
| Scores | yes | no |
| Cached | no | yes |
| Use for | relevance ranking | yes/no matching |

```json
{
  "query": {
    "bool": {
      "must":    [],  // scored, must match
      "should":  [],  // scored, optional (OR)
      "filter":  [],  // not scored, cached
      "must_not": []  // not scored, cached
    }
  }
}
```

Use `filter` for: dates, ranges, keywords, flags — anything where you don't need scoring. Use `must`/`should` for full text search where relevance matters.

## Common queries

```json
{ "match":        { "title": "godfather" }}          // full text
{ "match_phrase": { "title": "godfather returns" }}  // exact phrase
{ "term":         { "certificate": "PG" }}           // exact, no analysis
{ "terms":        { "certificate": ["PG", "PG-13"]}} // multiple exact
{ "range":        { "rating": { "gte": 8, "lte": 10 }}}
{ "exists":       { "field": "rating" }}
{ "match_all":    {} }                               // all docs
{ "match_none":   {} }                               // no docs
```

## Pagination

```json
{
  "from": 0,
  "size": 10
}
```

- `from` is zero-based — `from: 10` skips the first 10, returns from the 11th
- Max `from + size = 10,000` by default
- Deep pagination → use `search_after` instead

```json
GET movies/_search
{
  "size": 10,
  "sort": [{ "rating": "desc" }, { "_id": "asc" }],
  "search_after": [8.5, "abc123"]  // values from last result of previous page
}
```

## Sorting

Can't sort on `text` fields — use the `.keyword` subfield. `_score` sort is default.

```json
{
  "sort": [
    { "rating": "desc" },
    { "release_date": "asc" },
    "_score"
  ]
}
```

## Highlighting

Returns matching fragments with `<em>` tags around matched terms:

```json
{
  "query": { "match": { "synopsis": "crime" }},
  "highlight": {
    "fields": {
      "synopsis": {}
    }
  }
}
```

## Explain

Shows how score was calculated per document — useful for debugging relevance:

```json
GET movies/_search
{
  "explain": true,
  "query": { "match": { "title": "godfather" }}
}
```

Or for a specific doc:

```json
GET movies/_explain/1
{
  "query": { "match": { "title": "godfather" }}
}
```

## Source filtering

```json
"_source": false                        // exclude _source entirely
"_source": ["title", "rating"]          // include only these fields
"_source": {
  "includes": ["title", "rating"],
  "excludes": ["synopsis"]
}
```

**Fields** — returns values from the index rather than `_source`:

```json
{
  "fields": ["title", "release_date"]
}
```

**Scripted fields** — compute values at query time:

```json
{
  "script_fields": {
    "rating_out_of_100": {
      "script": {
        "source": "doc['rating'].value * 10"
      }
    }
  }
}
```
