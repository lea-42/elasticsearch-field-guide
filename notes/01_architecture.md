# Architecture

## Overview

**Cluster** — one or more nodes working together, identified by a cluster name

**Node** — a single Elasticsearch instance (JVM process)

**Index** — logical container for documents, split into shards

**Shard** — a single Lucene instance, the unit of distribution

- Primary shard — the original
- Replica shard — copy of a primary, never on the same node as its primary
- Shard count is fixed at index creation — can't change without reindex
- Replica count can be changed at any time

## Cluster health

| Status | Meaning |
|---|---|
| 🟢 Green | all primaries and replicas assigned |
| 🟡 Yellow | all primaries assigned, at least one replica unassigned |
| 🔴 Red | at least one primary unassigned — data missing |

Single node dev setup is always yellow unless `number_of_replicas: 0`

## Shard sizing rules of thumb

- Keep shards under 50gb
- 20–40 shards per gb of heap across the cluster
- Oversharding is a common problem with time-based indices → use ILM

## Heap / JVM

- Set `Xms` = `Xmx` (avoid resize pauses)
- Max 50% of RAM to heap — Lucene needs the rest for OS filesystem cache
- Keep heap under 32gb (compressed oops)
- Sweet spot on 64gb machine: 31gb heap

## Segments

- Immutable — one created on every refresh (default 1s)
- Deletes are flagged not removed until merge
- Background merges combine small segments into larger ones (TieredMergePolicy)
- `forcemerge?only_expunge_deletes=true` — compact segments with > 10% deletes
- `forcemerge?max_num_segments=1` — use on read-only indices only

## Useful health commands

```
GET _cluster/health
GET _cluster/health/index_name
GET _cat/nodes?v&h=name,heap.current,heap.percent,heap.max
GET _cat/indices?v&index=!.*
GET _cat/shards?v
```
