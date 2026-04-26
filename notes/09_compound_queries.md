# Compound queries

Compound queries wrap other queries to combine, modify, or influence scoring. Unlike leaf queries (term, match etc.), they don't search fields directly — they orchestrate other queries.

---

## bool

The workhorse. Combines multiple queries into one using four clauses.

| Clause | Effect | Scores? | Cached? |
|---|---|---|---|
| `must` | Must match. Contributes to score | Yes | No |
| `filter` | Must match. No scoring | No | Yes |
| `should` | Optional — boosts score if matches | Yes | No |
| `must_not` | Must not match | No | Yes |

```json
{
  "bool": {
    "must":     [ { "match":  { "title": "elasticsearch" } } ],
    "filter":   [ { "term":   { "status": "published" } },
                  { "range":  { "price": { "lte": 50 } } } ],
    "should":   [ { "term":   { "tag": "python" } } ],
    "must_not": [ { "exists": { "field": "deleted_at" } } ]
  }
}
```

**`minimum_should_match`**
- `should` + `must` → defaults to **0** (should is optional, purely for boosting)
- `should` alone → defaults to **1** (at least one must match, otherwise all docs return)
- Override explicitly: `"minimum_should_match": 2`

**Rule of thumb:** if a clause doesn't affect ranking, put it in `filter` — faster and cached.

---

## constant_score

Wraps a filter and assigns a flat score to all matching docs. Skips scoring entirely.

- Use when you need docs to match but don't care about ranking
- `boost` sets the fixed score (default 1.0)

```json
{
  "constant_score": {
    "filter": {
      "term": { "status": "published" }
    },
    "boost": 1.5
  }
}
```
The problem it solves: sometimes you want a filter at the top level without wrapping everything in a bool.
The boost part is where it gets more interesting. When you combine it inside a bool should, it lets you assign a fixed score to a whole category of docs:
```json
{
  "bool": {
    "should": [
      { "match": { "title": "elasticsearch" } },
      {
        "constant_score": {
          "filter": { "term": { "category": "featured" } },
          "boost": 2.0
        }
      }
    ]
  }
}
```

Featured docs always get exactly +2.0, regardless of any text matching. Non-featured docs get only their BM25 score. You're mixing a relevance signal with a fixed business signal cleanly.

---

## dis_max

Returns docs matching any of the queries, but scores by the **best single match** rather than summing all matches. Prevents score inflation when terms spread across fields.

```
score = max_query_score + (tie_breaker × sum_of_other_query_scores)
```

Example: query scores 0.9, 0.4, 0.2 with `tie_breaker=0.3`:
```
0.9 + (0.3 × (0.4 + 0.2)) = 0.9 + 0.18 = 1.08
```

```json
{
  "dis_max": {
    "queries": [
      { "match": { "title": "elasticsearch guide" } },
      { "match": { "body":  "elasticsearch guide" } }
    ],
    "tie_breaker": 0.3
  }
}
```

> `multi_match` with `type: best_fields` is syntactic sugar for `dis_max`. Use `dis_max` directly when you need fine-grained control over the individual queries.

| tie_breaker | Effect |
|---|---|
| `0.0` | Pure winner-takes-all |
| `0.3` | Default — light reward for matching in multiple queries |
| `1.0` | Equivalent to summing all scores |

---

## boosting

Demotes docs matching the `negative` clause without excluding them. Useful when you want to downrank rather than filter out.

- `positive` — main query, must match (any query type including `bool`)
- `negative` — applied on top of positive matches, triggers score penalty
- `negative_boost` — multiplier 0.0–1.0 applied to matching docs' scores

```json
{
  "boosting": {
    "positive": {
      "bool": {
        "must":   [ { "match": { "product": "tv" } } ],
        "filter": [ { "term":  { "in_stock": true } } ]
      }
    },
    "negative": {
      "bool": {
        "should": [
          { "range": { "price":  { "gte": 2500 } } },
          { "term":  { "brand":  "cheapbrand" } }
        ]
      }
    },
    "negative_boost": 0.5
  }
}
```

```
normal doc score:  0.84 → stays 0.84
demoted doc score: 0.84 × 0.5 = 0.42
```

> Use `boosting` over `filter` when you want "show but rank lower" rather than "exclude entirely". Both `positive` and `negative` accept any query type including `bool`.

---

## function_score

Modifies scores using custom functions. Lets you blend BM25 relevance with business logic (ratings, recency, popularity).

```json
{
  "function_score": {
    "query": { "term": { "product": "TV" } },
    "functions": [
      {
        "filter": { "term": { "brand": "LG" } },
        "weight": 3
      },
      {
        "filter": { "range": { "user_ratings": { "gte": 4.5 } } },
        "field_value_factor": {
          "field":    "user_ratings",
          "factor":   5,
          "modifier": "square"
        }
      }
    ],
    "score_mode": "avg",
    "boost_mode": "sum"
  }
}
```

**`score_mode`** — how multiple function scores combine with each other

| Value | Effect |
|---|---|
| `multiply` | Multiplied together (default) |
| `sum` | Added together |
| `avg` | Averaged (only over functions that fired) |
| `first` | First matching function wins |
| `max` / `min` | Highest / lowest function score |

**`boost_mode`** — how the combined function score combines with the original `_score`

| Value | Effect |
|---|---|
| `multiply` | `_score × functions_score` (default) |
| `sum` | `_score + functions_score` |
| `replace` | Discard `_score`, use functions_score only |
| `avg` | Average of both |
| `max` / `min` | Highest / lowest of the two |

**Score flow:**
```
term query → _score (BM25)
                │
                ├── function 1 (LG filter)      → 3
                ├── function 2 (ratings factor)  → 115.2
                │        │
                │   score_mode: avg → 59.1
                │        │
                └── boost_mode: sum → 0.84 + 59.1 = 59.94
```

**Script scoring** — full control via Painless script. Uses `doc[]` and `_score`, not `ctx` (that's for update scripts).

```json
"script_score": {
  "script": {
    "source": "_score * doc['user_ratings'].value * params['factor']",
    "params": { "factor": 3 }
  }
}
```

| Context | Variables | Use for |
|---|---|---|
| `script_score` | `_score`, `doc['field']` | Scoring at query time |
| `_update` script | `ctx._source`, `ctx._id` | Modifying documents |

> `doc['field']` reads from doc values (fast, columnar). Prefer it over `ctx._source` for read-only access in scripts.

---

## Quick reference

| Query | Use for |
|---|---|
| `bool` | Combine any queries with must/filter/should/must_not |
| `constant_score` | Match without scoring, flat boost |
| `dis_max` | Best field wins, optional tie_breaker for breadth |
| `boosting` | Demote unwanted docs without excluding them |
| `function_score` | Blend relevance with business logic (ratings, recency) |
