# Search evaluation and _rank_eval

---

## Why evaluation matters

"I'll know a good result when I see it" doesn't scale:
- Can't manually check 1000 queries after every change
- Biased toward queries you thought of, not what users type
- Not reproducible — two people disagree on quality
- Can't catch regressions — a change that fixes one query silently breaks ten others

You need a number that says "this version is 12% better than last version."

**The evaluation loop:**
```
define queries + relevance judgements
        ↓
run queries against current config
        ↓
compute metric score
        ↓
change something (boost, analyzer, num_candidates...)
        ↓
run again → did the score improve?
```

---

## Relevance judgements

Ground truth — for each query, which documents are relevant?

**Binary relevance:** relevant (1) or not (0).
**Graded relevance:** scale e.g. 0-3:
```
0 = not relevant
1 = slightly relevant
2 = relevant
3 = highly relevant (purchased)
```

**Sources in ecommerce:**

| Signal | Strength | Notes |
|---|---|---|
| Purchase | Very strong | Closest to ground truth |
| Add to cart | Strong | High intent |
| Long dwell time | Medium | Engaged but didn't convert |
| Click | Weak | Interest only |
| No click | Negative | Result wasn't appealing |
| Return / 1-star review | Negative | Looked relevant, wasn't |

---

## The confusion matrix

Every document falls into one of four buckets:

```
                    Retrieved    Not retrieved
                  ┌────────────┬──────────────┐
    Relevant      │     TP      │      FN      │
                  ├────────────┼──────────────┤
    Not relevant  │     FP      │      TN      │
                  └────────────┴──────────────┘
```

- **TP** — relevant and returned ✓
- **FP** — not relevant but returned ✗
- **FN** — relevant but missed ✗
- **TN** — not relevant and not returned ✓ (ignored in search — millions of these)

---

## Set-based metrics

Order doesn't matter — just which docs were returned.

### Precision
Of everything returned, what fraction was relevant?
```
Precision = TP / (TP + FP) = relevant retrieved / total retrieved
```

### Recall
Of all relevant docs, what fraction was found?
```
Recall = TP / (TP + FN) = relevant retrieved / total relevant in corpus
```

### The tradeoff
```
return everything → recall=1.0, precision≈0
return 1 confident result → precision=1.0, recall=low
```

### F1
Harmonic mean — penalises imbalance between precision and recall:
```
F1 = 2 × (Precision × Recall) / (Precision + Recall)
```

**Limitation:** completely ignores order. These score identically:
```
relevant docs: A, B, C
result set 1: [A, B, C, D, E]  ← relevant docs at positions 1,2,3
result set 2: [D, E, A, B, C]  ← relevant docs at positions 3,4,5
```

---

## Rank-aware metrics

Order matters. Relevant results appearing early are worth more.

### Precision@k and Recall@k
Only look at the top k results.
```
P@k = relevant docs in top k / k
R@k = relevant docs in top k / total relevant in corpus
```

Choose k based on your UI — if you show 10 results, P@10 is what matters.

### MRR — Mean Reciprocal Rank
How quickly does the first relevant result appear? Good for Q&A or navigational queries.
```
RR = 1 / rank_of_first_relevant_doc

query 1: first relevant at position 1 → RR = 1.0
query 2: first relevant at position 2 → RR = 0.5
query 3: first relevant at position 4 → RR = 0.25

MRR = (1.0 + 0.5 + 0.25) / 3 = 0.583
```

Limitation: ignores everything after the first relevant result.

### AP and MAP — Average Precision
Rewards finding relevant docs early AND finding all of them.

AP for one query — compute precision each time a relevant doc is hit, then average:
```
relevant: A, B, C
results:  [A, D, E, B, F, C]

precision when A found at pos 1: 1/1 = 1.0
precision when B found at pos 4: 2/4 = 0.5
precision when C found at pos 6: 3/6 = 0.5

AP = (1.0 + 0.5 + 0.5) / 3 = 0.667
```

MAP = average AP across all queries.

Limitation: binary relevance only.

### NDCG — Normalized Discounted Cumulative Gain
The most complete metric. Handles graded relevance and penalises relevant docs appearing late.

```
DCG = Σ relevance_score / log2(position + 1)
```

Example — results with graded scores [3, 0, 2, 1, 0]:
```
pos 1: 3 / log2(2) = 3.0
pos 2: 0 / log2(3) = 0.0
pos 3: 2 / log2(4) = 1.0
pos 4: 1 / log2(5) = 0.43
pos 5: 0 / log2(6) = 0.0
DCG = 4.43
```

Ideal ranking [3, 2, 1, 0, 0]:
```
IDCG = 3.0 + 1.27 + 0.5 + 0 + 0 = 4.77
NDCG = DCG / IDCG = 4.43 / 4.77 = 0.929
```

NDCG is always 0-1. 1.0 = perfect. Comparable across queries. **NDCG@10 is the standard.**

### Which metric to use

| Metric | Use when |
|---|---|
| Precision@k | You show k results, noise is costly |
| Recall@k | Missing relevant results is costly |
| MRR | One good answer is enough (Q&A, navigational) |
| MAP | Multiple relevant docs, binary relevance |
| NDCG | Graded relevance, most complete — default choice |

---

## Practical considerations

### Building a judgement set from ecommerce data

```python
judgements = [
    {"query": "red running shoes", "_id": "product_123", "rating": 3},  # purchased
    {"query": "red running shoes", "_id": "product_456", "rating": 2},  # add to cart
    {"query": "red running shoes", "_id": "product_789", "rating": 1},  # clicked
    {"query": "red running shoes", "_id": "product_999", "rating": 0},  # shown, ignored
]
```

### Position bias
Users click top results partly because they're at the top, not purely because they're relevant. Raw click/purchase data encodes your current search's biases.

**Mitigations:**
- Inverse Propensity Scoring (IPS) — weight by inverse probability of being seen
- Randomisation experiments — occasionally shuffle to collect unbiased signal
- Human annotation — expensive but unbiased gold standard

### Pitfalls

| Pitfall | Mitigation |
|---|---|
| Price/promotion driven purchases | Separate organic from promoted, timestamp signals |
| Seasonality | Refresh judgements, decay old signals |
| Vocabulary drift | Regularly review query coverage |
| Survivorship bias | Supplement with human annotation for long-tail |
| Single-session conflation | Keep judgements per query not per product |
| Returns / bad reviews | Factor in post-purchase signals |

> **Survivorship bias** is particularly dangerous — you only have purchase signals for products your current search already surfaces. Products buried at position 50+ never get signals even if they're highly relevant.

### Query set coverage

| Type | Example |
|---|---|
| Navigational | "Nike Air Max 90" |
| Informational | "running shoes for flat feet" |
| Transactional | "cheap waterproof jacket" |
| Seasonal | "Christmas gifts under 50" |
| Long tail | "vintage denim jacket 90s oversized women" |

Minimum 50-100 queries to get a meaningful score. 200-500 for good coverage.

### Dev vs test split
```
all labelled queries (500)
    ├── dev set (400)  — tune against this freely
    └── test set (100) — treat like a final exam, check rarely
```

---

## `_rank_eval` API

### Basic structure

```json
GET /products/_rank_eval
{
  "requests": [
    {
      "id": "running_shoes",
      "request": {
        "query": { "match": { "title": "running shoes" } }
      },
      "ratings": [
        { "_id": "product_123", "rating": 3 },
        { "_id": "product_456", "rating": 2 },
        { "_id": "product_789", "rating": 1 },
        { "_id": "product_999", "rating": 0 }
      ]
    }
  ],
  "metric": {
    "dcg": { "k": 10, "normalize": true }
  }
}
```

### Response

```json
{
  "metric_score": 0.72,
  "details": {
    "running_shoes": {
      "metric_score": 0.85,
      "unrated_docs": [          // returned but not in your ratings
        { "_id": "product_111" }
      ],
      "hits": [
        { "hit": { "_id": "product_123", "_score": 2.4 }, "rating": 3 },
        { "hit": { "_id": "product_111", "_score": 2.1 }, "rating": null },
        { "hit": { "_id": "product_456", "_score": 1.8 }, "rating": 2 }
      ]
    }
  }
}
```

**`unrated_docs`** — docs returned but not rated. Treated as rating=0 by default. Use `ignore_unlabeled: true` if you only labelled known relevant docs.

### Supported metrics

```json
// precision
"metric": { "precision": { "k": 10, "relevant_rating_threshold": 1 } }

// recall
"metric": { "recall": { "k": 10, "relevant_rating_threshold": 1 } }

// MRR
"metric": { "mean_reciprocal_rank": { "k": 10, "relevant_rating_threshold": 2 } }

// NDCG (normalize:true = NDCG, false = raw DCG)
"metric": { "dcg": { "k": 10, "normalize": true } }

// ERR — like MRR but accounts for graded relevance
"metric": { "expected_reciprocal_rank": { "maximum_relevance": 3, "k": 10 } }
```

### Python

```python
from elasticsearch import Elasticsearch
from typing import Any

es = Elasticsearch("http://localhost:9200")

def run_rank_eval(
    index:    str,
    requests: list[dict[str, Any]],
    metric:   dict[str, Any]
) -> dict[str, Any]:
    result = es.rank_eval(index=index, requests=requests, metric=metric)
    print(f"Overall score: {result['metric_score']:.4f}")
    for query_id, detail in result["details"].items():
        print(f"  {query_id}: {detail['metric_score']:.4f}  "
              f"(unrated: {len(detail['unrated_docs'])})")
    return result
```

### Iterating — comparing configurations

```python
from copy import deepcopy

base_request = {
    "id": "running_shoes",
    "request": { "query": { "match": { "title": "running shoes" } } },
    "ratings": [
        { "_id": "product_123", "rating": 3 },
        { "_id": "product_456", "rating": 2 }
    ]
}

metric = {"dcg": {"k": 10, "normalize": True}}

# baseline
v1 = run_rank_eval("products", [base_request], metric)

# try boosting title
v2_request = deepcopy(base_request)
v2_request["request"]["query"] = {
    "multi_match": {
        "query":  "running shoes",
        "fields": ["title^3", "description"],
        "type":   "best_fields"
    }
}
v2 = run_rank_eval("products", [v2_request], metric)

print(f"baseline: {v1['metric_score']:.4f}")
print(f"boosted:  {v2['metric_score']:.4f}")
print(f"delta:    {v2['metric_score'] - v1['metric_score']:+.4f}")
```

### Practical tips

- **NDCG@10** is the right default for ecommerce
- **Watch per-query scores** — a good overall can hide individual broken queries
- **Unrated docs** — if top 10 has many unrated docs your score is meaningless. Rate more or use `ignore_unlabeled`
- **Version judgements in git** — alongside your query configs. Know when the eval set changed.
- **Separate dev and test** — tune against dev, check test rarely
