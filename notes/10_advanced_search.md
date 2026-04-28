# Advanced Search

## Location Search

Elasticsearch supports geo queries via `geo_point` and `geo_shape` field types.

### Mapping

```json
PUT restaurants
{
  "mappings": {
    "properties": {
      "name":     { "type": "text" },
      "location": { "type": "geo_point" }
    }
  }
}
```

### geo_distance — filter by radius

The most common geo query — returns documents within a given distance of a point:

```json
GET restaurants/_search
{
  "query": {
    "geo_distance": {
      "distance": "5km",
      "location": [-0.0860, 51.5048]
    }
  }
}
```

### geo_bounding_box — filter by rectangle

Returns documents whose geo_point falls inside a bounding box defined by top-left and bottom-right corners:

```json
GET restaurants/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left":     [-0.1500, 51.5300],
        "bottom_right": [-0.0500, 51.4800]
      }
    }
  }
}
```

Useful for map viewport queries — when the user pans or zooms, send the visible bounding box as the filter.

> **Units:** `distance` accepts `km`, `mi`, `m`, `ft` — e.g. `"5km"` or `"3mi"`.

> **Coordinate order:** ES uses **`[lon, lat]`** (GeoJSON order) for arrays, but **`lat, lon`** for string format (`"51.5048, -0.0860"`). The `geo_point` type also accepts a `{ "lat": ..., "lon": ... }` object to make it unambiguous. Getting these backwards is a common source of silent bugs — points end up in the wrong hemisphere.

Fields must be mapped as `geo_point` for point-based queries, or `geo_shape` for polygon/complex shape queries.

---

## Shape Query

The `shape` query matches documents whose shape field intersects, contains, or is within a given geometry. It operates on fields mapped as `geo_shape` or `shape` (for cartesian coordinates).

### Mapping

```json
PUT attractions
{
  "mappings": {
    "properties": {
      "zone": {
        "type": "geo_shape"
      }
    }
  }
}
```

### Indexing a Shape

```json
PUT attractions/_doc/1
{
  "name": "Central Park",
  "zone": {
    "type": "polygon",
    "coordinates": [[
      [-73.9580, 40.8003],
      [-73.9496, 40.7968],
      [-73.9737, 40.7640],
      [-73.9818, 40.7676],
      [-73.9580, 40.8003]
    ]]
  }
}
```

### Querying

```json
GET attractions/_search
{
  "query": {
    "geo_shape": {
      "zone": {
        "shape": {
          "type": "point",
          "coordinates": [-73.9654, 40.7829]
        },
        "relation": "contains"
      }
    }
  }
}
```

The `relation` parameter controls the spatial relationship:

| Relation | Meaning |
|---|---|
| `intersects` | Shape overlaps the query geometry (default) |
| `contains` | Document shape fully contains the query geometry |
| `within` | Document shape is fully inside the query geometry |
| `disjoint` | Document shape has no overlap with the query geometry |

---

## Span Queries

Span queries are low-level positional queries that give precise control over term positions within a field. They are useful when the proximity and order of terms matters — for example, legal or academic text search.

### span_term

The basic building block. Matches a single term at a specific position.

```json
GET quotes/_search
{
  "query": {
    "span_term": {
      "quote": "aristotle"
    }
  }
}
```

### span_near

Matches documents where the specified terms appear within a given number of positions of each other. `slop` defines the maximum allowed positional gap, and `in_order` enforces left-to-right ordering.

```json
GET quotes/_search
{
  "query": {
    "span_near": {
      "clauses": [
        { "span_term": { "quote": "plato" } },
        { "span_term": { "quote": "cave" } }
      ],
      "slop": 5,
      "in_order": true
    }
  }
}
```

### span_or

Matches if any of the span clauses match. On its own it behaves like a `terms` query, but its real value is as a component inside `span_near` — allowing "this term OR that term" at a positional slot.

```json
GET quotes/_search
{
  "query": {
    "span_near": {
      "clauses": [
        {
          "span_or": {
            "clauses": [
              { "span_term": { "quote": "plato" } },
              { "span_term": { "quote": "aristotle" } }
            ]
          }
        },
        { "span_term": { "quote": "friends" } }
      ],
      "slop": 3,
      "in_order": false
    }
  }
}
```

### span_not

Excludes matches where an include span overlaps with an exclude span.

```json
GET quotes/_search
{
  "query": {
    "span_not": {
      "include": { "span_term": { "quote": "plato" } },
      "exclude": { "span_term": { "quote": "aristotle" } }
    }
  }
}
```

---

## distance_feature

`distance_feature` boosts relevance scores based on proximity to an origin point — either geographic or in time. Documents closer to the origin score higher; the score decays continuously with distance.

The `pivot` parameter controls how fast the boost decays with distance — it is **not a cutoff**. Documents beyond the pivot are still returned and still boosted, just less so. Pivot is the distance at which the boost reaches half its maximum value — think of it as the half-life of the decay. A smaller pivot makes the boost fall off aggressively over short distances; a larger pivot keeps documents competitive further out.

The scoring formula is: `score = 1 / (1 + (distance / pivot))`

### Geo Example

```json
GET universities/_search
{
  "query": {
    "distance_feature": {
      "field": "location",
      "origin": [-0.0860, 51.5048],
      "pivot": "10km"
    }
  }
}
```

### Date Example

`distance_feature` also works on `date` fields, boosting documents closer to a given date:

```json
GET articles/_search
{
  "query": {
    "distance_feature": {
      "field": "published_date",
      "origin": "now",
      "pivot": "30d"
    }
  }
}
```

`distance_feature` is typically combined with a `bool` query — the `filter` or `must` clause narrows the result set, while `distance_feature` in `should` boosts by proximity without excluding anything.

```json
GET universities/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { "type": "research" }
      },
      "should": {
        "distance_feature": {
          "field": "location",
          "origin": [-0.0860, 51.5048],
          "pivot": "10km"
        }
      }
    }
  }
}
```

---

## Percolate Query

The percolate query inverts the usual search model. Normally you store documents and query against them. With percolate, you **store queries** and test whether a document matches them.

### Use Case

Alerting systems — store a user's saved search as a query, then when a new document is indexed, run it through the stored queries to find which users should be notified.

### Setup

First, create an index with a `percolator` field to hold the stored queries, plus the fields your documents will use:

```json
PUT job_alerts
{
  "mappings": {
    "properties": {
      "query": {
        "type": "percolator"
      },
      "title": { "type": "text" },
      "location": { "type": "text" },
      "salary": { "type": "integer" }
    }
  }
}
```

### Storing a Query

Each document in the percolator index is a saved query:

```json
PUT job_alerts/_doc/alert_1
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "engineer" } },
        { "match": { "location": "london" } }
      ]
    }
  }
}

PUT job_alerts/_doc/alert_2
{
  "query": {
    "bool": {
      "must": { "match": { "title": "designer" } },
      "filter": { "range": { "salary": { "gte": 50000 } } }
    }
  }
}
```

### Running the Percolate Query

When a new job posting arrives, test it against all stored queries:

```json
GET job_alerts/_search
{
  "query": {
    "percolate": {
      "field": "query",
      "document": {
        "title": "Senior Software Engineer",
        "location": "London",
        "salary": 80000
      }
    }
  }
}
```

The response returns the stored query documents (i.e. `alert_1`) that matched — telling you which users' alerts were triggered.

### Multiple Documents

You can percolate several documents in one request using `documents` instead of `document`:

```json
GET job_alerts/_search
{
  "query": {
    "percolate": {
      "field": "query",
      "documents": [
        { "title": "Senior Software Engineer", "location": "London", "salary": 80000 },
        { "title": "UX Designer", "location": "Manchester", "salary": 55000 }
      ]
    }
  }
}
```

---

## Multilingual Search

### Core challenges

- **Query language ≠ catalogue language** — user searches in French, product indexed in German
- **Ambiguous short queries** — "sport", "USB cable", "Samsung" look the same in every language
- **IDF contamination** — "car" in French is a common conjunction (low IDF), in English it's a product (normal IDF). Shared index = polluted scoring.
- **Compound words** — German "Wanderschuhe" = "Wander" + "Schuhe". Needs decomposition.
- **Score incomparability** — BM25 score of 2.4 in a German index means something different than 2.4 in a French index.

### Architecture choice

#### Option A — single index, per-language fields

```json
{
  "title_de": "Wasserdichte Wanderstiefel",
  "title_fr": "Bottes de randonnée imperméables",
  "title_en": "Waterproof hiking boots"
}
```

- One doc per product, one index
- Search across all language fields in one query — no score incomparability
- **Problem:** IDF contamination across languages

#### Option B — separate index per language ✓ recommended

```
products_de → German docs, German analyzer, German IDF
products_fr → French docs, French analyzer, French IDF
products_en → English docs, English analyzer, English IDF
```

- Clean IDF per language
- **Problem:** score incomparability when merging across indices — solve with RRF

### Per-language analyzers

Each language field needs its own analyzer with correct stopwords and stemming:

```json
"settings": {
  "analysis": {
    "analyzer": {
      "fr_analyzer": {
        "type":      "custom",
        "tokenizer": "standard",
        "filter":    ["lowercase", "french_stop", "french_stemmer"]
      },
      "de_analyzer": {
        "type":      "custom",
        "tokenizer": "standard",
        "filter":    ["lowercase", "german_stop", "german_stemmer", "german_decompounder"]
      }
    }
  }
}
```

> "car" in French is a conjunction — removed by `french_stop`. "car" in English is a product — kept by `english_stop`. Per-language stopwords are essential.

### Language detection

Short product queries (1-3 words) are nearly impossible to detect reliably:

```
"sport"     → identical in DE, FR, NL, EN
"polo"      → brand? sport? shirt? any language
"USB cable" → same everywhere
"Samsung"   → brand, no language signal
```

**Best signal is not the query — it's the context:**
- Country domain: `rakuten.fr` → French, `rakuten.de` → German
- Browser `Accept-Language` header
- User account language preference

Use domain routing as primary language signal. Language detection on short queries is unreliable — don't depend on it.

### Multilingual embeddings — cross-lingual semantic layer

A multilingual embedding model maps equivalent terms across languages to the same vector space:

```python
model = SentenceTransformer("intfloat/multilingual-e5-base")  # 768 dims, 100+ languages

# these embed to nearby vectors regardless of language
model.encode("waterproof boots")           # EN
model.encode("wasserdichte Stiefel")       # DE
model.encode("bottes imperméables")        # FR
```

**Advantages over BM25 for multilingual:**
- No IDF contamination — embeddings don't use term frequency
- No score incomparability — cosine similarity is always 0-1
- Handles ambiguous queries naturally — model learned cross-lingual relationships
- No language detection needed

**Good multilingual models:**

| Model | Dims | Notes |
|---|---|---|
| `intfloat/multilingual-e5-base` | 768 | Strong default, 100+ languages |
| `intfloat/multilingual-e5-large` | 1024 | Better accuracy, slower |
| `paraphrase-multilingual-MiniLM-L12-v2` | 384 | Fast, lighter |
| `BAAI/bge-m3` | 1024 | State of the art |

### Merging results across language indices

When searching across multiple language indices, raw BM25 scores are not comparable:

```
products_fr: "car" → 50,000 results, top score 0.8  (common word)
products_en: "car" → 200 results,    top score 2.4  (meaningful term)
```

#### Option 1 — RRF (Reciprocal Rank Fusion)

Combines ranks not scores — scale differences disappear:

```python
es.search(
    index="products_de,products_fr,products_en,products_nl",
    body={
        "query": { "match": { "title": query } },
        "rank":  { "rrf": { "window_size": 100 } },
        "size":  10
    }
)
```

A doc ranked 1st in English (score 2.4) and ranked 1st in French (score 0.8) are treated equally — both are rank 1. Score scale is irrelevant.

#### Option 2 — pick best language set by avg top score

Let the index tell you what language the query is in — the index that produces the strongest top results probably understood the query best:

```python
def best_language_results(query: str) -> list[dict]:
    all_results = {}
    for lang in ["de", "fr", "en", "nl", "es"]:
        hits = es.search(index=f"products_{lang}",
                         body={"query": {"match": {"title": query}},
                               "size": 10})["hits"]["hits"]
        if hits:
            avg_score = sum(h["_score"] for h in hits) / len(hits)
            all_results[lang] = {"hits": hits, "avg_score": avg_score}

    if not all_results:
        return []

    best = max(all_results, key=lambda l: all_results[l]["avg_score"])
    return all_results[best]["hits"]
```

> Don't use result **count** as the signal — a common word returns thousands of weak matches. Use **average score of top results**.

| | Pick best set | RRF merge |
|---|---|---|
| Query language known | Good — route directly | Overkill |
| Query ambiguous | Reasonable | Better |
| Same product in multiple languages | Misses cross-lingual confirmation | Rewards it |
| Noisy common words | Risky | RRF handles via rank |

### Recommended architecture for pan-European ecommerce

```
user on rakuten.fr → language = French (from domain)
        ↓
query parsing + cleaning (LLM, cached)
        ↓
hybrid search:
  - BM25 on products_fr  (exact terms, correct French IDF)
  - multilingual kNN     (semantic, cross-lingual safety net)
  - RRF to merge
        ↓
multilingual cross-encoder rerank
  BAAI/bge-reranker-v2-m3 or cross-encoder/msmarco-MiniLM-L6-en-de-v1
        ↓
results in French
```

BM25 handles exact matching with clean per-language IDF. Multilingual kNN handles semantic and cross-lingual retrieval. RRF merges without score scale problems. Cross-encoder reranks with deep query-document interaction.

> BM25 scores are relative to the corpus they were computed in. Never compare raw BM25 scores across indices — use RRF instead.
