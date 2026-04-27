# Vector search
 
---
 
## Concepts
 
**Embeddings** — fixed-length arrays of floats representing the semantic meaning of text. Similar meaning = similar vectors = small angular distance.
 
**Why vector search** — BM25 matches terms, vectors match meaning:
```
query: "fast cars"
BM25 misses: "high speed vehicles"   ← no term overlap
kNN finds:   "high speed vehicles"   ← semantically close
```
 
**Generating embeddings:**
- **OpenAI** — `text-embedding-3-small` (1536 dims), API call, costs money, high quality
- **HuggingFace** — `all-MiniLM-L6-v2` (384 dims), runs locally, free, good quality for most use cases
- **ELSER** (Elastic) — sparse vectors, built into Elastic Cloud, no external model needed, requires ML node (paid)
- **E5** (Elastic) — dense vectors via inference pipeline, auto-generates embeddings at index time, requires ML node (paid)
---
 
## Mappings
 
```json
PUT /articles
{
  "mappings": {
    "properties": {
      "title":       { "type": "text" },
      "description": { "type": "text" },
      "category":    { "type": "keyword" },
      "title_embedding": {
        "type":       "dense_vector",
        "dims":       384,
        "index":      true,
        "similarity": "cosine"
      },
      "description_embedding": {
        "type":       "dense_vector",
        "dims":       384,
        "index":      true,
        "similarity": "cosine"
      }
    }
  }
}
```
 
**Key parameters:**
 
| Parameter | Notes |
|---|---|
| `dims` | Must match model output exactly. OpenAI ada-002=1536, MiniLM=384, mpnet=768 |
| `index` | `true` to enable kNN search (builds HNSW graph). `false` = store only |
| `similarity` | `cosine` for text (magnitude irrelevant). `dot_product` for normalised vectors. `l2_norm` for spatial/image |
 

**Multiple vector fields** — valid, each independent. Storage cost = `dims × 4 bytes × num_docs` per field.
 
**HNSW index options** (leave at defaults unless tuning):
(HNSW = Heirarchial Navigable Small World)
```json
"index_options": {
  "type":            "hnsw",
  "m":               16,
  "ef_construction": 100
}
```
 
**ef = exploration factor** — how broadly the algorithm explores the graph. Higher = looks harder before deciding it's done.
 
| Parameter | When | Default | Effect |
|---|---|---|---|
| `m` | Index time | 16 | Connections per node per layer. Higher = better recall, more memory, slower indexing. Permanent. |
| `ef_construction` | Index time only | 100 | Candidates evaluated to pick the best `m` connections. Better graph, slower indexing. Zero effect at search time. |
| `num_candidates` | Search time | — | Candidates evaluated per shard during traversal. Main accuracy vs speed lever. (`ef_search` in pgvector) |
 
**HNSW graph structure:**
```
layer 2:  A ————————————— F          (sparse, long-range connections)
layer 1:  A ——— C ——— E — F
layer 0:  A — B — C — D — E — F — G  (all nodes, short connections)
```
Search starts at the top layer and narrows down. Each layer gets closer before dropping to the next.
 
**`m` vs `ef_construction`:** when inserting a node, ES evaluates `ef_construction` candidates and picks the best `m` to connect to. Once inserted only the `m` connections remain — `ef_construction` is irrelevant. `m` is permanent and affects memory forever. Neither can be changed without reindexing.
 
**pgvector equivalent** (same algorithm, slightly different naming):
```sql
CREATE INDEX ON articles USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 100);
 
SET hnsw.ef_search = 100;  -- equivalent to ES num_candidates
```
 
---
 
## Indexing
 
Always generate embeddings outside ES and include as a float array. Batch encode for performance.
 
```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk
from sentence_transformers import SentenceTransformer
from typing import Generator
 
es = Elasticsearch("http://localhost:9200")
model = SentenceTransformer("all-MiniLM-L6-v2")
 
def generate_docs(documents: list[dict], batch_size: int = 32) -> Generator[dict, None, None]:
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        title_embeddings = model.encode([d["title"] for d in batch])
        desc_embeddings  = model.encode([d["description"] for d in batch])
 
        for doc, t_emb, d_emb in zip(batch, title_embeddings, desc_embeddings):
            yield {
                "_index": "articles",
                "_id":    doc["id"],
                "_source": {
                    "title":                 doc["title"],
                    "description":           doc["description"],
                    "category":              doc["category"],
                    "title_embedding":       t_emb.tolist(),
                    "description_embedding": d_emb.tolist()
                }
            }
 
bulk(es, generate_docs(documents))
```
 
> Always `.tolist()` numpy arrays before sending to ES.
> Benchmark batch sizes — 32-64 is a good starting point on CPU.
 
---
 
## kNN search
 
```python
query_vector = model.encode("how to search elasticsearch efficiently").tolist()
 
result = es.search(index="articles", body={
    "knn": {
        "field":          "title_embedding",
        "query_vector":   query_vector,
        "k":              10,
        "num_candidates": 100
    }
})
```
 
**`k`** — number of results to return.
**`num_candidates`** — candidates each shard considers before returning top k. The accuracy vs speed lever. Must be `>= k`, typically `k × 5-10`.
 
### How it works — HNSW
 
Multi-layer graph built at index time connecting similar vectors. At search time navigates the graph instead of comparing all vectors.
 
```
exact kNN:  compare against ALL vectors → perfect, O(n)
HNSW:       navigate graph → ~99% recall, O(log n)
```
 
### Filtering within kNN
 
```json
"knn": {
  "field":          "title_embedding",
  "query_vector":   [...],
  "k":              10,
  "num_candidates": 100,
  "filter": { "term": { "category": "tech" } }
}
```
 
Filter runs **before** kNN — only vectors from matching docs are considered.
 
**HNSW + filter caveat:** the graph is built without filter awareness. With selective filters some docs may be unreachable via graph traversal (connectivity problem). ES mitigates by:
- Pre-filtering: skipping non-matching nodes during traversal (works for broad filters)
- Exact fallback: brute-force on filtered subset when filter is very selective
Mitigation: raise `num_candidates` when using selective filters.
 
### Exact kNN (brute force)
 
For small datasets or perfect accuracy requirements:
```json
"query": {
  "script_score": {
    "query": { "match_all": {} },
    "script": {
      "source": "cosineSimilarity(params.query_vector, 'title_embedding') + 1.0",
      "params": { "query_vector": [...] }
    }
  }
}
```
`+1.0` shifts score to non-negative (cosine range is -1 to 1). Practical up to ~50k docs.
 
---
 
## Hybrid search
 
Combines BM25 (exact terms) with kNN (semantic meaning). Use **RRF (Reciprocal Rank Fusion)** to merge — combines ranks not raw scores, avoiding scale mismatch between BM25 and cosine similarity.
 
```python
def hybrid_search(query: str, k: int = 10) -> list[dict]:
    query_vector = model.encode(query).tolist()
 
    response = es.search(index="articles", body={
        "query": {
            "match": { "title": query }
        },
        "knn": {
            "field":          "title_embedding",
            "query_vector":   query_vector,
            "k":              k,
            "num_candidates": k * 10
        },
        "rank": {
            "rrf": { "window_size": 100, "rank_constant": 60 }
        },
        "size": k
    })
 
    return [{"title": h["_source"]["title"], "score": h["_score"]}
            for h in response["hits"]["hits"]]
```
 
**RRF formula:**
```
score = Σ 1 / (rank_constant + rank_in_each_list)
 
doc at BM25 rank 3, kNN rank 5:
score = 1/(60+3) + 1/(60+5) = 0.0159 + 0.0154 = 0.0313
```
 
**`window_size`** — candidates RRF considers from each list. Set `>= size`, typically much higher (100 for size=10). BM25 is cheap so cast a wide net.
**`rank_constant`** — 60 is the default, works well in practice.
 
> RRF available from ES 8.8+. Free on self-hosted.
 
### Multiple vector fields in hybrid search
 
```json
"knn": [
  { "field": "title_embedding",       "query_vector": [...], "k": 10, "num_candidates": 100, "boost": 1.5 },
  { "field": "description_embedding", "query_vector": [...], "k": 10, "num_candidates": 100, "boost": 1.0 }
],
"rank": { "rrf": {} }
```
 
### When to use what
 
| Scenario | Approach |
|---|---|
| Exact terms (product codes, IDs) | BM25 only |
| Semantic / conceptual search | kNN only |
| General purpose search box | Hybrid + RRF |
 
---
 
## Performance tuning
 
### num_candidates tradeoff
 
```python
# minimum
num_candidates = k * 1.5
# good default
num_candidates = k * 10
# high accuracy
num_candidates = k * 50
```
 
Benchmark with real data — the accuracy curve flattens fast after a point.
 
### Memory
 
```
dims × 4 bytes per doc (float32)
 
384 dims  × 1M docs = ~1.5GB
1536 dims × 1M docs = ~6GB
 
HNSW graph: m × 8 bytes × num_docs
m=16, 1M docs = ~128MB
```
 
HNSW graph lives in OS filesystem cache (off-heap) — same as segment data.
 
### int8 quantization — 4× memory reduction
 
```json
"index_options": { "type": "int8_hnsw" }
```
 
Stores vectors as 8-bit integers. ~1-2% recall drop, 4× storage reduction. Worth it for large datasets. Available ES 8.11+.
 
### Tuning checklist
 
| Problem | Fix |
|---|---|
| Slow search | Lower `num_candidates`, use `int8_hnsw` |
| Poor recall | Raise `num_candidates`, raise `m` |
| Too much memory | Lower `m`, `int8_hnsw`, reduce dims |
| Slow indexing | Bulk API, replicas=0 during load, lower `ef_construction` |
| Filter recall problems | Raise `num_candidates` |
 
