# Elasticsearch field guide

A practical reference for working with Elasticsearch — built to get up to speed fast.

Started as notes from reading [Elasticsearch in Action, Second Edition](https://www.manning.com/books/elasticsearch-in-action-second-edition) (Madhusudhan Konda, Manning), but goes beyond the book to cover topics like dense vector search, kNN, hybrid search, and relevance tuning with `_rank_eval`.

Not a tutorial. Not a copy of the docs. A reference you can actually use.

---

## Contents

| Chapter | Topic |
|---|---|
| [Local Setup](./notes/00_local_setup.md) | Docker images, removing security, common commands |
| [Architecture](./notes/01_architecture.md) | Clusters, nodes, shards, segments, heap |
| [Mappings](./notes/02_mappings.md) | Field types, mapping rules, normalizers, nested |
| [Documents](./notes/03_documents.md) | CRUD, update by query, bulk, reindex |
| [Indexing](./notes/04_indexing.md) | Index operations, templates, ILM, shrink/split/rollover |
| [Text Analysis](./notes/05_text_analysis.md) | Analyzers, token filters, synonyms, autocomplete |
| [Intro to Search](./notes/06_intro_to_search.md) | Query vs filter, bool, pagination, sorting, highlighting |
| [Term Level Search](./notes/07_term_level_search.md) | term, terms, range, fuzzy, wildcard, prefix |
| [Full Text Search](./notes/08_full_text_search.md) | match, match_phrase, multi_match, query_string |
| [Compound Queries](./notes/09_compound_queries.md) | bool, boosting, function_score, dis_max |
| [Advanced Search](./notes/10_advanced_search.md) | Percolator, more like this, spans |
| [Aggregations](./notes/11_aggregations.md) | Metric, bucket, pipeline aggregations |
| [Performance and Troubleshooting](./notes/12_performance_and_troubleshooting.md) | Profiling, caching, shard sizing, slow logs |
| [Dense Vector Search](./notes/13_dense_vector_search.md) | kNN, dense_vector, hybrid search |
| [Evaluation](./notes/14_rank_eval.md) | Relevance evaluation, `_rank_eval` |

---

## Philosophy

- Code examples over prose
- Real gotchas called out 
- Performance implications noted where they matter
- Covers both the book content and things the book doesn't cover

---

## Stack

Examples assume:
- Elasticsearch 8.x
- Local dev via Docker (`elastic/start-local`)
- Python client where code examples are needed

