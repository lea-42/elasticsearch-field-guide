# Stuff that doesn't belong anywhere else

## Bloom Filters
A bloom filter is a cheap heap-resident gatekeeper that prevents unnecessary disk reads by quickly ruling out segments that can't possibly contain a term.

## Data streams

A data stream is a named abstraction over a series of time-based backing indices. Looks like one index from the outside, is many indices under the hood.

**Use for:** logs, metrics, events — any append-only time-series data.

```
logs  ← data stream (one name you write/search against)
  ├── .ds-logs-2024-01-000001  (old, read-only)
  ├── .ds-logs-2024-02-000002  (old, read-only)
  └── .ds-logs-2024-03-000003  (current write target)
```

- **Writes** → always go to the current backing index
- **Reads** → fan out across all backing indices transparently
- **Rollover** → automatic when current index hits size/age threshold — new backing index created, old one sealed
- **ILM** → manages hot/warm/cold movement of backing indices automatically

### Key constraint

Append-only — documents cannot be updated or deleted individually. Use ILM delete phase or `delete_by_query` for removal.

### Key Concept

Data streams are a first-class concept alongside regular indices. APIs that historically operated on indices were updated to cover data streams too — hence the paired phrasing.

### Regular index vs data stream

| | Regular index | Data stream |
|---|---|---|
| Write target | Fixed | Always current backing index |
| Updates | Yes | No |
| Rollover | Manual | Automatic |
| Best for | Structured, mutable data | Time-series, append-only |


## Multilingual search strategies



### Core challenges

- **Query language ≠ catalogue language** — user searches in French, product indexed in German
- **Ambiguous short queries** — "sport", "USB cable", "Samsung" look the same in every language
- **IDF contamination** — "car" in French is a common conjunction (low IDF), in English it's a product (normal IDF). Shared index = polluted scoring.
- **Compound words** — German "Wanderschuhe" = "Wander" + "Schuhe". Needs decomposition.
- **Score incomparability** — BM25 score of 2.4 in a German index means something different than 2.4 in a French index.

---

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

---

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

---

### Language detection

Short product queries (1-3 words) are nearly impossible to detect reliably:
```
"sport"    → identical in DE, FR, NL, EN
"polo"     → brand? sport? shirt? any language
"USB cable" → same everywhere
"Samsung"  → brand, no language signal
```

**Best signal is not the query — it's the context:**
- Country domain: `rakuten.fr` → French, `rakuten.de` → German
- Browser `Accept-Language` header
- User account language preference

Use domain routing as primary language signal. Language detection on short queries is unreliable — don't depend on it.

---

### Multilingual embeddings — cross-lingual semantic layer

A multilingual embedding model maps equivalent terms across languages to the same vector space:

```python
model = SentenceTransformer("intfloat/multilingual-e5-base")  # 768 dims, 100+ languages

# these embed to nearby vectors regardless of language
model.encode("waterproof boots")              # EN
model.encode("wasserdichte Stiefel")          # DE
model.encode("bottes imperméables")           # FR
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

---

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

---

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

BM25 handles exact matching with clean per-language IDF.
Multilingual kNN handles semantic and cross-lingual retrieval.
RRF merges without score scale problems.
Cross-encoder reranks with deep query-document interaction.

---

### The IDF problem in one line

> BM25 scores are relative to the corpus they were computed in. Never compare raw BM25 scores across indices — use RRF instead.
