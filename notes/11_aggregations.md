# Aggregations

Aggregations analyse data across documents. Three types:

- **Metric** — compute a value (sum, avg, max...)
- **Bucket** — group documents into buckets (by term, range, date...)
- **Pipeline** — operate on the output of other aggregations

Aggregations run alongside queries — the query filters which docs are aggregated.

```json
{
  "query": { "term": { "status": "published" } },
  "aggs": {
    "my_agg_name": {
      "avg": { "field": "price" }
    }
  }
}
```

> Set `"size": 0` to skip returning hits when you only need aggregation results.

---

## Metric aggregations

Compute a single value from a set of documents.

### Single-value metrics

**`avg`** — average value
```json
"aggs": {
  "avg_price": { "avg": { "field": "price" } }
}
```

Also available: `min`, `max`, `sum`, `count`, `value_count` (count of non-null values), `cardinality` (approximate distinct count).

**`cardinality`** — approximate distinct count (HyperLogLog++)
```json
"aggs": {
  "unique_users": { "cardinality": { "field": "user_id" } }
}
```

### Multi-value metrics

**`stats`** — returns count, min, max, avg, sum in one go
```json
"aggs": {
  "price_stats": { "stats": { "field": "price" } }
}
// → { "count": 100, "min": 10, "max": 500, "avg": 120, "sum": 12000 }
```

Also available: `extended_stats` (adds variance, std deviation, bounds).

**`percentiles`** — distribution across percentile bands
```json
"aggs": {
  "price_percentiles": {
    "percentiles": {
      "field":   "price",
      "percents": [25, 50, 75, 95, 99]
    }
  }
}
```

**`top_hits`** — return actual documents from within a bucket (useful nested inside bucket aggs)
```json
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": { "field": "brand.keyword", "size": 5 },
      "aggs": {
        "best_products": {
          "top_hits": {
            "size": 3,
            "_source": ["title", "price", "rating"],
            "sort": [{ "rating": "desc" }]
          }
        }
      }
    }
  }
}
```
results

```json
"buckets": [
  {
    "key": "Samsung",
    "doc_count": 42,
    "best_products": {
      "hits": {
        "hits": [
          { "_source": { "title": "Samsung 4K TV", "price": 899, "rating": 4.9 } },
          { "_source": { "title": "Samsung OLED",  "price": 1200, "rating": 4.7 } },
          { "_source": { "title": "Samsung Frame", "price": 750, "rating": 4.6 } }
        ]
      }
    }
  },
  {
    "key": "LG",
    "doc_count": 38,
    "best_products": { ... }
  }
]
```

If there is no sort then the results are sorted by `_score`based on the query, no query and no sort -> get random results. 

---

## Bucket aggregations

Group documents into buckets. Each bucket can have nested sub-aggregations.

### `terms`

One bucket per unique value. Returns top N by doc count.

```json
"aggs": {
  "by_brand": {
    "terms": {
      "field": "brand.keyword",
      "size":  10
    }
  }
}
```

- `size` — how many buckets to return (default 10)
- `order` — sort by doc count, or by a sub-agg value
- Use `.keyword` field for text, not analyzed `text` field

### `range`

Manual numeric ranges — you define the buckets.

```json
"aggs": {
  "price_ranges": {
    "range": {
      "field":  "price",
      "ranges": [
        { "to": 100 },
        { "from": 100, "to": 500 },
        { "from": 500 }
      ]
    }
  }
}
```

Also available: `date_range` for date-based ranges with date math (`now-7d`).

### `histogram`

Fixed-width numeric intervals — Elasticsearch creates the buckets automatically.

```json
"aggs": {
  "price_histogram": {
    "histogram": {
      "field":    "price",
      "interval": 100
    }
  }
}
```

### `date_histogram`

Buckets by calendar or fixed time interval.

```json
"aggs": {
  "sales_per_month": {
    "date_histogram": {
      "field":             "date",
      "calendar_interval": "month"
    }
  }
}
```

- `calendar_interval`: `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`
- `fixed_interval`: `60m`, `7d` etc. for fixed durations

### `filter` / `filters`

Single filter bucket — aggregate only over a subset of docs.

```json
"aggs": {
  "expensive": {
    "filter": { "range": { "price": { "gte": 500 } } },
    "aggs": {
      "avg_price": { "avg": { "field": "price" } }
    }
  }
}
```

`filters` — multiple named buckets, one per filter:
```json
"aggs": {
  "by_status": {
    "filters": {
      "filters": {
        "published": { "term": { "status": "published" } },
        "draft":     { "term": { "status": "draft" } }
      }
    }
  }
}
```

### `nested` / `reverse_nested`

Required to aggregate on nested object fields (nested objects are indexed as separate hidden documents).

```json
"aggs": {
  "reviews": {
    "nested": { "path": "reviews" },
    "aggs": {
      "avg_rating": { "avg": { "field": "reviews.rating" } }
    }
  }
}
```

`reverse_nested` — go back up to the parent doc from within a nested agg.

---

## Nesting aggregations

Any bucket agg can contain sub-aggregations. Sub-aggs operate only on docs in their parent bucket.

```json
"aggs": {
  "by_brand": {
    "terms": { "field": "brand.keyword", "size": 5 },
    "aggs": {
      "avg_price":  { "avg": { "field": "price" } },
      "top_rated":  {
        "top_hits": {
          "size": 1,
          "sort": [{ "rating": "desc" }]
        }
      }
    }
  }
}
```

Result: top 5 brands, each with their average price and highest-rated product.

---

## Pipeline aggregations

Operate on the output of other aggregations, not on documents directly.

### Parent pipeline aggs

Live **inside** a bucket agg. Enrich each bucket with an additional computed value.

**`derivative`** — change from previous bucket
```json
"sales_change": {
  "derivative": { "buckets_path": "total_sales" }
}
// Jan: null, Feb: +500, Mar: -300
```

**`cumulative_sum`** — running total across buckets
```json
"running_total": {
  "cumulative_sum": { "buckets_path": "total_sales" }
}
// Jan: 1000, Feb: 2500, Mar: 3700
```

**`moving_avg`** — rolling average over last N buckets (smooths spikes)
```json
"smoothed": {
  "moving_avg": { "buckets_path": "total_sales", "window": 3 }
}
```

**`bucket_script`** — compute a value from multiple sub-agg values in the same bucket
```json
"margin": {
  "bucket_script": {
    "buckets_path": { "revenue": "total_sales", "cost": "total_cost" },
    "script": "params.revenue - params.cost"
  }
}
```

**`bucket_selector`** — filter out buckets that don't meet a condition
```json
"only_big_months": {
  "bucket_selector": {
    "buckets_path": { "sales": "total_sales" },
    "script": "params.sales > 1000"
  }
}
```

**`serial_diff`** — difference between current bucket and N buckets back
```json
"vs_last_quarter": {
  "serial_diff": { "buckets_path": "total_sales", "lag": 3 }
}
```

### Sibling pipeline aggs

Sit **at the same level** as the bucket agg they reference. Produce a single new value across all buckets.

**`max_bucket`** — which bucket had the highest value
```json
"best_month": {
  "max_bucket": { "buckets_path": "sales_per_month>total_sales" }
}
// → { "value": 1500, "keys": ["2024-02"] }
```

Also available: `min_bucket`, `avg_bucket`, `sum_bucket`, `stats_bucket`, `percentiles_bucket`.

### `buckets_path` syntax

```
"total_sales"                    → sub-agg in same bucket
"sales_per_month>total_sales"    → across agg levels, use >
"agg1>agg2>metric"               → chain multiple levels
```

### Parent vs sibling — mental model

```
date_histogram (Jan, Feb, Mar buckets)
    ├── sum: total_sales
    ├── derivative     ← PARENT: adds a value to each bucket
    │
    └── [same level]
        max_bucket     ← SIBLING: looks across all buckets, one result
```

---

## fielddata vs doc values

Aggregations need to access field values by document — the inverted index can't do this efficiently.

| | `fielddata` | doc values |
|---|---|---|
| Field types | `text` | `keyword`, numeric, date |
| Built | At query time (lazy) | At index time |
| Stored | JVM heap | On disk (off-heap) |
| Cost | Expensive, OOM risk | Fast, efficient |
| Default | Disabled | Enabled |

**Prefer `.keyword` subfields for aggregations** — they use doc values automatically:

```json
"title": {
  "type": "text",
  "fields": {
    "keyword": { "type": "keyword" }
  }
}
// search on: title
// aggregate on: title.keyword
```

Only enable `fielddata: true` on `text` fields if you genuinely need to aggregate on analyzed tokens (e.g. most frequent individual words).

---

## Quick reference

| Aggregation | Type | Use for |
|---|---|---|
| `avg` `min` `max` `sum` | Metric | Basic statistics |
| `stats` `extended_stats` | Metric | All stats in one shot |
| `cardinality` | Metric | Approximate distinct count |
| `percentiles` | Metric | Distribution / outliers |
| `top_hits` | Metric | Sample docs from a bucket |
| `terms` | Bucket | Group by field value |
| `range` `date_range` | Bucket | Manual ranges |
| `histogram` `date_histogram` | Bucket | Auto intervals |
| `filter` `filters` | Bucket | Subset aggregation |
| `nested` | Bucket | Aggregate on nested objects |
| `derivative` | Pipeline (parent) | Change per bucket |
| `cumulative_sum` | Pipeline (parent) | Running total |
| `moving_avg` | Pipeline (parent) | Smooth time series |
| `bucket_script` | Pipeline (parent) | Computed metric per bucket |
| `bucket_selector` | Pipeline (parent) | Filter buckets |
| `max_bucket` `min_bucket` | Pipeline (sibling) | Best/worst bucket |
| `avg_bucket` `sum_bucket` | Pipeline (sibling) | Cross-bucket stats |
