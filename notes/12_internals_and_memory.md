# Elasticsearch internals — memory, shards, segments

---

## The three storage layers

| Layer | What lives there | Speed | Size |
|---|---|---|---|
| Disk | Index data, shards, segments | Slow | Huge, cheap |
| RAM (off-heap) | OS filesystem cache, doc values, memory-mapped segment data | Fast | Limited by physical RAM |
| JVM Heap | Metadata, caches, aggregation results, fielddata | Fastest | Fixed slice of RAM — precious |

**Golden rule:** give ES half your RAM as heap, leave the other half for the OS filesystem cache.

```
32GB machine → 16GB heap for ES, 16GB free for OS to cache hot index data
```

> Segment data itself never loads into heap. A 50GB segment might cost only a few MB of heap for its metadata. The OS handles caching hot segment data in RAM automatically via memory-mapped files.

---

## Shards

A shard is a chunk of your index stored on **disk**. When Elastic says "keep shards between 10-50GB" they mean physical disk size.

```
index (200GB total)
    ├── shard 0 → 50GB on disk (node 1)
    ├── shard 1 → 50GB on disk (node 2)
    ├── shard 2 → 50GB on disk (node 3)
    └── shard 3 → 50GB on disk (node 4)
```

A **replica** is a copy of a shard on a different node — also on disk. Gives redundancy and read throughput.

```
4 shards × 1 replica = 400GB total disk
```

---

## Shards and heap

Shard data lives on disk, but every shard has **heap overhead** — metadata, Lucene data structures, caches:

```
~30-50MB heap per shard
1000 shards × 50MB = 50GB heap — just overhead
```

**Elastic guidance:** max ~20 shards per GB of heap.
```
16GB heap → ~320 shards max per node
```

### The two problems pull in opposite directions

| Problem | Cause | Symptom |
|---|---|---|
| Too few / too large shards | Slow recovery, can't parallelise | Node failure = long rebuild |
| Too many / too small shards | Heap pressure, search coordination overhead | "Oversharding" — very common |

Logs are the classic culprit: new index per day × 5 shards × months of retention = thousands of shards.

---

## Segments

Each shard is made up of multiple **Lucene segments** — immutable mini-indexes on disk.

```
shard
  ├── segment_1 (1000 docs)
  ├── segment_2 (500 docs)
  └── segment_3 (200 docs, 50 deleted)
```

- Segments are **immutable** — deleted docs are marked, not removed, until a merge
- ES merges segments automatically in the background
- Like shards, each segment has small heap overhead (metadata, term dictionaries, bloom filters)

---

## Force merge on read-only indices

When an index becomes read-only (warm/cold tier) — force merge to 1 segment:

```json
POST /my_index/_forcemerge?max_num_segments=1
```

Benefits:
- Reduces heap overhead (N segments → 1)
- Physically removes deleted documents — reclaims disk space
- Faster searches — one segment to scan
- Better compression

**Don't force merge active (hot) indices** — it's CPU/IO intensive and causes temporary disk spikes (needs ~2× space during the merge).

---

## Hot / warm / cold / frozen tiers

Data moves from fast expensive storage to cheap slow storage as it ages. **Shards physically relocate** between nodes in each tier.

| Tier | Storage | Use |
|---|---|---|
| Hot | NVMe SSD | Active writes + frequent reads |
| Warm | Standard SSD | Read-only, occasional searches |
| Cold | HDD / object storage | Rare searches, compliance |
| Frozen | Object storage (S3 etc.) | Almost never accessed, pulled on demand |

**ILM (Index Lifecycle Management)** automates shard movement and optimisation at each stage:

```json
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover":    { "max_size": "50gb" } } },
      "warm":   { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 },
                                                  "force_merge": { "max_num_segments": 1 } } },
      "cold":   { "min_age": "30d", "actions": { "freeze": {} } },
      "delete": { "min_age": "365d","actions": { "delete": {} } }
    }
  }
}
```

At warm stage ILM also runs `shrink` (reduce shard count) + `force_merge` (reduce to 1 segment) — both reduce heap overhead.

---

## Bloom filters

A bloom filter is a tiny heap-resident data structure that answers one question fast:

> "Is this term **definitely not** in this segment?"

- **"Definitely not here"** → skip segment, no disk I/O ✓
- **"Maybe here"** → go search the segment

Never false negatives. Occasional false positives (unnecessary segment read) — acceptable, results are still correct.

```
query: "elasticsearch"
    ├── bloom filter shard 1 → "definitely not here" → skip ✓
    ├── bloom filter shard 2 → "maybe here" → search → found
    └── bloom filter shard 3 → "definitely not here" → skip ✓
```

Bloom filters don't store actual terms — just a bit array with hash functions. Millions of terms represented in a few MB of heap. Cheap gatekeeper that prevents unnecessary disk reads.

---

## Quick mental model

```
query arrives
    │
    ├── bloom filter (heap) → rules out segments with no disk I/O
    │
    ├── OS filesystem cache (off-heap RAM) → serves hot segment data fast
    │
    └── disk → cold segment data, loaded into OS cache on first access

heap is for:
    metadata, bloom filters, aggregation results,
    fielddata (avoid), query cache, indexing buffer
    
heap is NOT for:
    actual document content, inverted index data, doc values
```
