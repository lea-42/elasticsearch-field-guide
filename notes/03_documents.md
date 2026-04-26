# Documents

## Index document — known ID

```json
PUT index_name/_doc/1
{ "field": "value" }
```

## Index document — auto ID

```json
POST index_name/_doc
{ "field": "value" }
```

## Retrieve by ID

```
GET index_name/_doc/1
```

## Check exists without fetching

```
HEAD index_name/_doc/1
```

## Update — partial (doc)

```json
POST index_name/_update/1
{ "doc": { "field": "new_value" } }
```

## Update — script

```json
POST index_name/_update/1
{
  "script": {
    "source": "ctx._source.count += params.val",
    "params": { "val": 1 }
  }
}
```

## Update — upsert (doc)

```json
POST index_name/_update/1
{
  "doc": { "field": "value" },
  "doc_as_upsert": true
}
```

## Update — upsert (script)

```json
POST index_name/_update/1
{
  "scripted_upsert": true,
  "script": { "source": "ctx._source.field = true" },
  "upsert": {}
}
```

## Update by query

```json
POST index_name/_update_by_query
{
  "query": { "term": { "field": "value" } },
  "script": { "source": "ctx._source.field = true" }
}
```

## Delete

By ID:

```
DELETE index_name/_doc/1
```

By query:

```json
POST index_name/_delete_by_query
{
  "query": { "term": { "field": "value" } }
}
```

## Use script over doc when

- Reading current value: `ctx._source.count += 1`
- Conditional logic: `if(...) { } else { }`
- Removing a field: `ctx._source.remove('field')`

## Bulk

```json
POST _bulk
{ "index": { "_index": "movies", "_id": "1" } }
{ "title": "Inception" }
{ "delete": { "_index": "movies", "_id": "2" } }
{ "update": { "_index": "movies", "_id": "3" } }
{ "doc": { "blockbuster": true } }
```

- Not transactional — partial failure is possible
- Check `errors` field in response, then iterate `items` for individual statuses
- Each action is 2 lines except `delete` (1 line)

## Reindex

```json
POST _reindex
{
  "source": {
    "index": "source_index",
    "_source": ["field1", "field2"],
    "query": { "term": { "field": "value" } }
  },
  "dest": { "index": "dest_index" },
  "script": {
    "source": """
      ctx._source.new_field = ctx._source.old_field;
      ctx._source.remove('old_field')
    """
  }
}
```

- Can filter docs via `query`
- Can pick fields via `_source`
- Can transform via `script`
- Source and dest don't need the same fields
