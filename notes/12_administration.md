# Administration

---

## How it works — storage and memory

### The three storage layers

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

### Shards

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

Every shard also has **heap overhead** — metadata, Lucene data structures, caches:

```
~30-50MB heap per shard
1000 shards × 50MB = 50GB heap — just overhead
```

**Elastic guidance:** max ~20 shards per GB of heap.

```
16GB heap → ~320 shards max per node
```

| Problem | Cause | Symptom |
|---|---|---|
| Too few / too large shards | Slow recovery, can't parallelise | Node failure = long rebuild |
| Too many / too small shards | Heap pressure, search coordination overhead | "Oversharding" — very common |

Logs are the classic culprit: new index per day × 5 shards × months of retention = thousands of shards.

### Segments

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

### Bloom filters

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

### Quick mental model

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

---

## Cluster and nodes

- A **cluster** is a group of nodes sharing the same `cluster.name`
- Nodes join automatically when network settings match
- **Horizontal scaling** — add nodes, shards redistribute automatically
- Nodes communicate with each other on **port 9300** (transport)
- Clients communicate via REST on **port 9200** (HTTP)

---

## Shards and replicas

| | Primary shard | Replica |
|---|---|---|
| Handles writes | Yes | No |
| Handles reads | Yes | Yes |
| Improves write perf | More primaries | — |
| Improves read perf | — | More replicas |

- Replicas improve read throughput but cost disk space and I/O
- Test before committing to a shard/replica strategy — watch for spikes

---

## Master node

- One elected master per cluster — manages cluster state, tracks nodes/shards
- **Quorum** — minimum number of master-eligible nodes required to make decisions (elect a new master, commit state changes)
- Always run **at least 3 master-eligible nodes** to avoid split-brain

**Split-brain:** two parts of a cluster each elect their own master and diverge — data corruption risk. Quorum prevents this.

```
3 master-eligible nodes → quorum = 2
if 1 node dies → 2 remaining nodes still form quorum → cluster healthy
if network splits 1|2 → minority of 1 can't reach quorum → won't elect master
```

---

## JVM and heap

- Configured in `jvm.options` — **never edit this file directly**
- Instead create a `*.options` file in `config/jvm.options.d/`
- Set heap with `-Xms` (min) and `-Xmx` (max) — set both to the same value
- **Never exceed 50% of available RAM for heap** — leave the rest for OS filesystem cache

```
# config/jvm.options.d/custom.options
-Xms8g
-Xmx8g
```

---

## Config files

| File | Purpose | Edit? |
|---|---|---|
| `elasticsearch.yml` | Cluster name, paths, network, ports | Yes |
| `jvm.options` | JVM defaults | Never |
| `jvm.options.d/*.options` | Custom JVM overrides | Yes — create new file |
| `log4j2.properties` | Logging config | Rarely |

Common `elasticsearch.yml` settings: `cluster.name`, `node.name`, `path.data`, `path.logs`, `network.host`, `http.port`, `transport.port`.

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

**ILM** automates shard movement and optimisation at each stage:

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

At warm stage ILM also runs `shrink` + `force_merge` — both reduce heap overhead.

---

## Snapshots and restore

A snapshot is a backup of your cluster — incremental after the first full snapshot. Elasticsearch tracks shared segments between snapshots so you can safely delete any snapshot in any order without breaking others.

**A snapshot can include:**
- Indices and data streams
- Cluster state (ILM policies, index templates, persistent settings)

**Repository types:** local filesystem or cloud object storage (AWS S3, GCS, Azure).

```json
// create a snapshot
PUT _snapshot/my_repo/snapshot_1
{ "indices": ["logs-*"], "include_global_state": false }

// restore
POST _snapshot/my_repo/snapshot_1/_restore
{ "indices": ["logs-*"] }
```

**SLM (Snapshot Lifecycle Management)** — automates snapshot scheduling, same idea as ILM but for backups:

```json
PUT _slm/policy/daily_snapshots
{
  "schedule":   "0 30 1 * * ?",
  "name":       "<daily-snap-{now/d}>",
  "repository": "my_repo",
  "config":     { "include_global_state": false }
}
```

> Deleting an old snapshot only reclaims disk space for segments unique to that snapshot — shared segments are kept.

---

## Quick reference

| Concept | Rule of thumb |
|---|---|
| Shard size | 10-50GB |
| Heap size | 50% of RAM, `-Xms` = `-Xmx` |
| Shards per GB heap | ~20 max |
| Master-eligible nodes | Minimum 3 |
| Quorum | (n/2) + 1 nodes must agree |
| Transport port | 9300 (node-to-node) |
| HTTP port | 9200 (client-to-ES) |
| Snapshot incremental | Yes — safe to delete old snapshots |
