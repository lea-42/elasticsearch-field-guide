# AI in search

---

## Overview — the full pipeline

Each component feeds into the next:

```
user question
      ↓
query understanding (intent, rewrite, clean, parse)
      ↓
hybrid search ES (BM25 + kNN + RRF) → 100 candidates
      ↓
cross-encoder rerank → top 5-10
      ↓
[optional] LTR business-aware rerank
      ↓
[optional] RAG — LLM generates answer from top docs
      ↓
response to user
```

---

## 1. Query understanding and rewriting

Users are bad at writing queries. AI fixes this before the query hits ES.

### Query cleaning and structured extraction

"Cheap", "best", "under £50" are intent signals not search terms. Extract them into structured search parameters:

```python
def parse_query(raw_query: str) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""Parse this ecommerce search query into structured parameters.
Return JSON only:
{{
  "query":    "clean search terms only",
  "sort":     "price_asc | price_desc | rating_desc | relevance",
  "max_price": number or null,
  "intent":   "navigational | transactional | informational | seasonal",
  "ttl_days": 1-30  // how long to cache this parse result
}}

Query: {raw_query}"""
        }]
    )
    return json.loads(response.content[0].text)

parse_query("cheap waterproof jacket under 50")
# → {"query": "waterproof jacket", "sort": "price_asc",
#    "max_price": 50, "intent": "transactional", "ttl_days": 30}

parse_query("newest iphone headphones")
# → {"query": "iphone headphones", "sort": "relevance",
#    "max_price": null, "intent": "navigational", "ttl_days": 1}
```

> **Include `ttl_days` in the structured output** — the LLM can judge whether a query is time-sensitive ("newest", "latest", current year) and set a short TTL accordingly. One call, all metadata.

### Intent types

| Intent | Example | Best search strategy |
|---|---|---|
| Navigational | "Nike Air Max 90" | Exact match, boost brand + model |
| Transactional | "cheap waterproof jacket" | Hybrid + sort by price/rating |
| Informational | "running shoes for flat feet" | Semantic, surface guides + products |
| Seasonal | "Christmas gifts under 50" | Boost gift categories, apply price filter |

```python
def search_by_intent(query: str, parsed: dict) -> dict:
    intent = parsed["intent"]

    if intent == "navigational":
        return es.search(index="products", body={
            "query": {
                "multi_match": {
                    "query":  parsed["query"],
                    "fields": ["brand^3", "model^3", "title"],
                    "type":   "best_fields"
                }
            }
        })
    elif intent == "transactional":
        body = {
            "knn": {
                "field":          "embedding",
                "query_vector":   model.encode(parsed["query"]).tolist(),
                "k":              20,
                "num_candidates": 200
            },
            "sort": [{ "price": "asc" if parsed["sort"] == "price_asc" else "desc" }]
        }
        if parsed["max_price"]:
            body["knn"]["filter"] = { "range": { "price": { "lte": parsed["max_price"] } } }
        return es.search(index="products", body=body)
    else:
        return hybrid_search(parsed["query"])
```

### Intent — implicit meaning vs literal terms

LLMs can infer what users actually mean, not just what they typed:

```
"iphone headphones"     → title: "headphones",  filter: connectivity in [bluetooth, usb-c]
"headphones for gaming" → title: "headphones",  filter: features includes "microphone"
"laptop for video editing" → title: "laptop",   filter: ram >= 16
```

### Query expansion

Add related terms the user didn't type:

```python
def expand_query(query: str) -> list[str]:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""Generate 3 alternative search queries for an ecommerce site.
Original: {query}
Return JSON array of strings only."""
        }]
    )
    return json.loads(response.content[0].text)

# search across all expansions with dis_max
def search_with_expansion(query: str) -> dict:
    expanded   = expand_query(query)
    all_queries = [query] + expanded
    return es.search(index="products", body={
        "query": {
            "dis_max": {
                "queries":      [{"match": {"title": q}} for q in all_queries],
                "tie_breaker":  0.3
            }
        }
    })
```

### Query relaxation — zero results fallback

```python
def search_with_fallback(query: str) -> dict:
    # 1. strict — all terms must match
    results = es.search(index="products", body={
        "query": { "match": { "title": { "query": query, "operator": "and" } } }
    })
    if results["hits"]["total"]["value"] > 0:
        return results

    # 2. relax to OR
    results = es.search(index="products", body={
        "query": { "match": { "title": { "query": query, "operator": "or" } } }
    })
    if results["hits"]["total"]["value"] > 0:
        return results

    # 3. semantic fallback
    return es.search(index="products", body={
        "knn": {
            "field":          "embedding",
            "query_vector":   model.encode(query).tolist(),
            "k":              10,
            "num_candidates": 100
        }
    })
```

---

## 2. Semantic query techniques

Short or vague queries embed poorly — a 5-word query vector is far from rich product description vectors. Two techniques fix this, both by generating a better string to embed before searching.

The first step is always query rewriting — parse intent and clean the query as covered in section 1. The cleaned `query` string is what you embed or pass to HyDE, not the raw user input.

### HyDE — Hypothetical Document Embeddings

Instead of embedding the (rewritten) query string directly, generate a full hypothetical product description and embed that. The resulting vector is document-shaped — much closer to what's actually indexed.

**Query rewriting:** embed a better query string — lightweight, good default
**HyDE:** embed a synthetic document — heavier, bigger gain on vague queries

```python
def hyde_search(query: str, k: int = 10) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""Write a short product description for the ideal product
that satisfies this query. Be specific about features.
Query: {query}
Description (2-3 sentences):"""
        }]
    )
    hypothetical = response.content[0].text.strip()

    # embed the hypothetical doc, not the query
    hyde_vector = model.encode(hypothetical).tolist()

    return es.search(index="products", body={
        "knn": {
            "field":          "embedding",
            "query_vector":   hyde_vector,
            "k":              k,
            "num_candidates": k * 10
        }
    })
```

**Multiple hypotheticals** — average N vectors for robustness:

```python
def hyde_multi(query: str, n: int = 3, k: int = 10) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=400,
        messages=[{
            "role": "user",
            "content": f"""Generate {n} different product descriptions satisfying this query.
Query: {query}
Return JSON array of strings only."""
        }]
    )
    hypotheticals = json.loads(response.content[0].text)
    avg_vector    = np.mean(model.encode(hypotheticals), axis=0).tolist()

    return es.search(index="products", body={
        "knn": {
            "field":          "embedding",
            "query_vector":   avg_vector,
            "k":              k,
            "num_candidates": k * 10
        }
    })
```

### When to use each

| Scenario | Rewriting | HyDE |
|---|---|---|
| Short vague queries | Yes | Yes — bigger gain |
| Long descriptive queries | Marginal | No |
| Navigational queries | No | No |
| Zero-result fallback | Yes | Yes |
| Latency critical path | Cache it | Cache it |

### Caching — applies to both

Both techniques produce a deterministic string + vector from a raw query. Cache the whole pipeline result so LLM calls and embedding generation only happen once per query.

```python
import redis
from datetime import timedelta

cache = redis.Redis()

def cached_pipeline(raw_query: str) -> dict:
    key    = f"pipeline:v1:{raw_query.lower().strip()}"
    cached = cache.get(key)
    if cached:
        return json.loads(cached)

    parsed    = parse_query(raw_query)            # LLM call
    rewritten = rewrite_query(parsed["query"])    # LLM call
    vector    = model.encode(rewritten).tolist()  # embedding

    result = {"parsed": parsed, "rewritten": rewritten, "vector": vector}
    cache.setex(key, timedelta(days=parsed.get("ttl_days", 7)), json.dumps(result))
    return result
```

One cache lookup skips all LLM calls and embedding generation. Include a version prefix (`v1:`) — bump it when you change prompts or switch embedding models to automatically invalidate stale entries.

**Pre-warm from query logs** — top 100-200 queries are often 50-80% of traffic:

```python
from collections import Counter

def prewarm_cache(query_logs: list[str], top_n: int = 500) -> None:
    counts      = Counter(query_logs)
    top_queries = [q for q, _ in counts.most_common(top_n)]
    for query in top_queries:
        cached_pipeline(query)
```

Run as a nightly batch job. LLM calls happen offline, hot path stays fast.

**The offline/online split:**

```
OFFLINE (nightly, no latency constraint)
    query logs → LLM parse + rewrite → Redis cache
    query logs → synonym list updates
    click/purchase data → LTR training data

ONLINE (per request, latency critical)
    query → cache hit (0ms) or local classifier (<10ms)
    → ES search (20-100ms)
    → reranker (50-200ms)
```

---

## 3. Reranking and cross-encoders

### Why first-stage retrieval isn't enough

BM25 and kNN score documents independently without deep query-document comparison. A reranker re-scores candidates with a deeper model.

### Bi-encoder vs cross-encoder

**Bi-encoder** (kNN) — query and doc encoded separately, compared with cosine:
```
query → encoder → vector Q
doc   → encoder → vector D  (pre-computed at index time)
score = cosine(Q, D)
```
Fast. But query and doc never interact during encoding.

**Cross-encoder** — query and doc encoded together:
```
[query + doc] → encoder → relevance score
```
Much more accurate — model attends to interactions between query and doc tokens. But can't pre-compute, must run at query time.

### The two-stage pattern

```
stage 1: hybrid ES search → 100 candidates  (fast)
stage 2: cross-encoder rerank → top 10      (accurate)
```

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, hits: list[dict], top_n: int = 10) -> list[dict]:
    pairs  = [
        (query, hit["_source"]["title"] + " " + hit["_source"]["description"])
        for hit in hits
    ]
    scores = reranker.predict(pairs)

    for hit, score in zip(hits, scores):
        hit["rerank_score"] = float(score)

    return sorted(hits, key=lambda x: x["rerank_score"], reverse=True)[:top_n]

def search_and_rerank(query: str) -> list[dict]:
    results = es.search(index="products", body={
        "query": { "multi_match": { "query": query, "fields": ["title", "description"] } },
        "size":  100
    })
    return rerank(query, results["hits"]["hits"])
```

### Popular models

| Model | Size | Speed | Notes |
|---|---|---|---|
| `cross-encoder/ms-marco-MiniLM-L-6-v2` | Small | Fast | Good default |
| `cross-encoder/ms-marco-MiniLM-L-12-v2` | Medium | Medium | Better accuracy |
| `BAAI/bge-reranker-base` | Medium | Medium | Strong multilingual |
| Cohere `rerank-english-v3.0` | API | ~200ms | No local infra |

### Latency
```
ES hybrid (100 candidates): 50-100ms
cross-encoder rerank:       100-300ms CPU, 20-50ms GPU
total:                      150-400ms
```

---

## 4. RAG — Retrieval Augmented Generation

ES retrieves relevant docs, LLM generates a grounded answer from them.

```
user: "do you have waterproof hiking boots under £100?"
      ↓
ES retrieves: Merrell Moab 3 WP £89.99, Salomon X Ultra 4 GTX £94.99
      ↓
LLM: "Yes! We have two options under £100: the Merrell Moab 3 at £89.99
      and the Salomon X Ultra at £94.99, both waterproof hiking boots."
```

### Basic pipeline

```python
def retrieve(query: str, k: int = 5) -> list[dict]:
    query_vector = model.encode(query).tolist()
    results = es.search(index="products", body={
        "query": { "match": { "description": query } },
        "knn":   { "field": "embedding", "query_vector": query_vector,
                   "k": k, "num_candidates": k * 10 },
        "rank":  { "rrf": {} },
        "size":  k
    })
    return [hit["_source"] for hit in results["hits"]["hits"]]

def rag_answer(question: str) -> str:
    docs    = retrieve(question)
    context = "\n\n".join([
        f"Product: {d['title']}\nPrice: £{d['price']}\n{d['description']}"
        for d in docs
    ])
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": f"""Answer using only the products below.
Say you don't know if the answer isn't there.

Products:
{context}

Question: {question}"""
        }]
    )
    return response.content[0].text
```

### Chunking in ecommerce

For typical product descriptions (50-300 words) — **don't chunk**, embed whole.

Chunking applies to long-form content alongside products:
- Buying guides and articles
- Size guides
- Technical manuals
- Aggregated reviews

**Parent-child pattern** — index small chunks for retrieval precision, return parent doc to LLM:
```python
def index_with_parent(doc: dict) -> None:
    chunks = chunk_semantic(doc["content"])
    for i, chunk in enumerate(chunks):
        es.index(index="docs", document={
            "chunk_text":   chunk,
            "parent_title": doc["title"],
            "full_content": doc["content"],   # stored but not embedded
            "embedding":    model.encode(chunk).tolist(),
            "chunk_index":  i
        })
```

### Conversational RAG

Rewrite follow-up questions as standalone queries before searching:

```python
def rag_chat(question: str, history: list[dict]) -> tuple[str, list[dict]]:
    # rewrite with context if follow-up
    if history:
        rewrite = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=100,
            messages=[{"role": "user", "content":
                f"History: {json.dumps(history[-3:])}\n"
                f"Rewrite as standalone search query: {question}"}]
        )
        search_query = rewrite.content[0].text.strip()
    else:
        search_query = question

    docs     = retrieve(search_query)
    context  = "\n\n".join([d["title"] for d in docs])
    messages = history + [{"role": "user",
                           "content": f"Context:\n{context}\n\nQuestion: {question}"}]

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        messages=messages
    )
    answer          = response.content[0].text
    updated_history = messages + [{"role": "assistant", "content": answer}]
    return answer, updated_history
```

### RAG pitfalls

- **Retrieval quality gates generation** — wrong docs = wrong answer, confidently
- **Context window limits** — top 3-5 good docs beats 20 mediocre ones
- **Hallucination at boundaries** — prompt explicitly: "answer only from provided context"
- **Chunk size affects recall** — 100-300 tokens with overlap is the sweet spot

---

## 5. Learning to Rank (LTR)

Train a model to learn optimal ranking from user behaviour. Incorporates business signals that cross-encoders can't.

### Features

```python
features = {
    # relevance
    "bm25_title":        2.4,
    "bm25_description":  1.1,
    "knn_similarity":    0.87,
    "exact_title_match": 1,
    # product
    "price":             89.99,
    "avg_rating":        4.3,
    "review_count":      234,
    "in_stock":          1,
    "profit_margin":     0.34,
    # popularity
    "click_rate":        0.12,
    "conversion_rate":   0.04,
    "revenue_per_search":3.20,
}
```

### Algorithms

| Approach | Model | Optimises |
|---|---|---|
| Pointwise | Any regressor | Predict relevance score per doc |
| Pairwise | RankNet, LambdaRank | Doc A should rank above doc B |
| Listwise | **LambdaMART** | Whole ranked list — directly optimises NDCG |

LambdaMART is the production standard.

### XGBoost example

```python
import xgboost as xgb
import numpy as np

def train_ltr(training_data: list[dict]) -> xgb.Booster:
    X       = np.array([list(d["features"].values()) for d in training_data])
    y       = np.array([d["label"] for d in training_data])
    queries = [d["query"] for d in training_data]
    groups  = [queries.count(q) for q in dict.fromkeys(queries)]

    dtrain  = xgb.DMatrix(X, label=y)
    dtrain.set_group(groups)

    return xgb.train(
        {"objective": "rank:ndcg", "eval_metric": "ndcg@10", "eta": 0.1, "max_depth": 6},
        dtrain,
        num_boost_round=100
    )
```

### ES LTR plugin

```json
// define features as ES queries
POST _ltr/_featureset/product_features
{ "featureset": { "features": [
  { "name": "title_bm25", "params": ["query_string"],
    "template": { "match": { "title": "{{query_string}}" } } },
  { "name": "avg_rating",  "params": [],
    "template": { "function_score": { "field_value_factor": { "field": "avg_rating" } } } }
]}}

// rescore with trained model
GET products/_search
{
  "query": { "match": { "title": "hiking boots" } },
  "rescore": {
    "window_size": 100,
    "query": {
      "rescore_query": {
        "sltr": { "params": {"query_string": "hiking boots"},
                  "model": "my_ltr_model", "featureset": "product_features" }
      }
    }
  }
}
```

### Cross-encoder vs LTR

| | Cross-encoder | LTR |
|---|---|---|
| Training data needed | No | Yes |
| Business signals | No | Yes |
| Setup complexity | Low | High |
| Latency | 100-300ms | 10-30ms |
| Best for | Semantic relevance | Business-optimised ranking |

**Stack both in production:**
```
ES retrieval (100)
      ↓
cross-encoder (semantic rerank → 30)
      ↓
LTR (business-aware final rank → 10)
```

---

## Quick reference

| Technique | Problem it solves | Latency cost |
|---|---|---|
| Query parsing / intent | Users write bad queries | LLM call — cache it |
| Query expansion | Vocabulary mismatch | LLM call — cache it |
| Query relaxation | Zero results | Cheap — ES only |
| HyDE | Vague short queries, weak embeddings | LLM call — skip on hot path |
| Hybrid search (BM25 + kNN + RRF) | Keyword + semantic coverage | Low |
| Cross-encoder rerank | First-stage retrieval not accurate enough | 100-300ms CPU |
| RAG | User needs a generated answer not just results | LLM call |
| LTR | Need to incorporate business signals into ranking | Low (fast inference) |
