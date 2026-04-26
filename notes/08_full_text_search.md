# Full-text search

## match_all

Returns all documents. Score is always 1.0. No body required.

```json
{ "query": { "match_all": {} } }
```

---

## match

Standard full-text query. Analyzes input, scores by BM25.

- Default `operator: OR` — any term matches
- `operator: AND` — all terms must match
- `fuzziness`: 0, 1, 2 or `AUTO` (AUTO: 0 for 1-2 chars, 1 for 3-5, 2 for 6+)

```json
"match": {
  "title": {
    "query":     "elasticsearch guide",
    "operator":  "and",
    "fuzziness": 1
  }
}
```

---

## match_phrase

Terms must appear in exact order. All terms must be present — `slop` allows terms to be separated by N positions or out of order, but does NOT allow terms to be missing.

```json
"match_phrase": {
  "body": {
    "query": "quick brown fox",
    "slop":  1
  }
}
```

---

## match_phrase_prefix

Like `match_phrase` but last term acts as a prefix. Good for search-as-you-type.

- `max_expansions` caps prefix variants (perf)

```json
"match_phrase_prefix": {
  "title": {
    "query": "elastic sear",
    "max_expansions": 10
  }
}
```

---

## query_string

Full Lucene syntax. **Strict — throws a parse error on bad input.** For internal tools / power users only.

- Boolean operators: `AND`, `OR`, `NOT` (or `&&`, `||`, `!`) — **must be uppercase**; lowercase `and`/`or` are treated as search terms
- Field targeting: `title:elastic`
- Boost: `elastic^2`
- Phrase: `"quick brown"`
- Ranges: `date:[2020-01-01 TO 2021-01-01]`

```json
"query_string": {
  "query":         "title:(elastic AND guide) OR body:search^2",
  "default_field": "title"
}
```

---

## simple_query_string

Safer subset of `query_string`. Silently ignores bad syntax. Use for user-facing search boxes.

- `+` = must, `-` = must not, `|` = or
- Never throws — degrades gracefully on bad input

```json
"simple_query_string": {
  "query":  "elasticsearch +guide -java",
  "fields": ["title", "body"]
}
```

---

## multi_match types

| Type | Scoring | Use when |
|---|---|---|
| `best_fields` | dis_max — best field score + tie_breaker × others | Fields independent, best single match wins |
| `most_fields` | bool should — sum ÷ field count | Same content indexed multiple ways (+ stemmed field) |
| `cross_fields` | All fields treated as one combined field | Data split across fields (first_name + last_name) |
| `phrase` | best_fields with match_phrase per field | Exact phrase in any field |
| `phrase_prefix` | best_fields with match_phrase_prefix per field | Autocomplete across fields |

```json
"multi_match": {
  "query":       "elasticsearch guide",
  "fields":      ["title^2", "body"],
  "type":        "best_fields",
  "tie_breaker": 0.3
}
```

---

## dis_max and tie_breaker

`best_fields` is syntactic sugar for `dis_max`. Prevents score inflation when terms spread across fields — rewards docs where all terms land in one field.

```
score = max_field_score + (tie_breaker × sum_of_other_field_scores)
```

Example: scores `0.9`, `0.4`, `0.2` with `tie_breaker=0.3`:
```
0.9 + (0.3 × (0.4 + 0.2)) = 0.9 + 0.18 = 1.08
```

```json
"dis_max": {
  "queries": [
    { "match": { "title": "elasticsearch guide" } },
    { "match": { "body":  "elasticsearch guide" } }
  ],
  "tie_breaker": 0.3
}
```

| tie_breaker | Effect |
|---|---|
| `0.0` | Pure winner-takes-all, other fields ignored |
| `0.3` | Default — light reward for breadth |
| `1.0` | Equivalent to most_fields |

Tune with `_rank_eval` API rather than guessing.
