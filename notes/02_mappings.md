# Mappings

## Create an index with a mapping

```json
PUT index_name
{
  "mappings": {
    "properties": {
      "field": { "type": "keyword" }
    }
  }
}
```

## Add field to existing mapping

Can be PUT or POST:

```json
PUT index_name/_mapping
{
  "properties": {
    "new_field": { "type": "integer" }
  }
}
```

## View mapping

```
GET index_name/_mapping
```

## Field types

| Type | Use for |
|---|---|
| `text` | full text search |
| `keyword` | exact match, aggregations, sorting |
| `integer` `float` `long` | numbers |
| `date` | dates, configure `format` |
| `boolean` | true/false |
| `ip` | IP addresses, supports CIDR queries |
| `object` | nested JSON, relationships lost in arrays |
| `nested` | array of objects, preserves relationships |
| `join` | parent-child in same index, avoid if possible |
| `token_count` | stores number of tokens not the string |
| `wildcard` | wildcard pattern queries, uses BK-grams |
| `completion` | autocomplete, prefix only, in-memory FST |
| `search_as_you_type` | autocomplete, matches mid-string |

**Date format example:**

```json
"joining_date": {
  "type": "date",
  "format": "dd-MM-yyyy||yyyy-MM-dd"
}
```

## Key mapping rules

- Mappings only grow — can't delete or change a field type
- To change a field: reindex
- Dynamic mapping is dangerous for dates — always define explicitly
- `text` + `keyword` multi-field is common:

```json
"title": {
  "type": "text",
  "fields": {
    "keyword": { "type": "keyword" }
  }
}
```

## Normalizer

Like an analyzer but for keyword fields (no tokenization):

```json
"settings": {
  "analysis": {
    "normalizer": {
      "lowercase_normalizer": {
        "type": "custom",
        "filter": ["lowercase"]
      }
    }
  }
}
```

## Nested query

```json
{
  "nested": {
    "path": "attachments",
    "query": { ... }
  }
}
```
