# Troubleshooting

---

## Slow queries

### Step 0 — enable slow query logging
```json
PUT /my_index/_settings
{
  "index.search.slowlog.threshold.query.warn":  "10s",
  "index.search.slowlog.threshold.query.info":  "5s",
  "index.search.slowlog.threshold.query.debug": "2s",
  "index.search.slowlog.threshold.fetch.warn":  "1s"
}
```
Two phases: **query** (which docs match?) and **fetch** (retrieve their content). Slow fetch usually means large `_source` docs or too many fields returned.

### Step 1 — profile the query
```json
GET my_index/_search
{ "profile": true, "query": { ... } }
```
Breaks down time per shard per clause. Do this before guessing.

### Step 2 — query vs filter context
Anything not affecting score belongs in `filter` — cached and fast.
```json
"filter": [ { "term": { "status": "published" } } ]  // cached ✓
"must":   [ { "term": { "status": "published" } } ]  // scored, not cached ✗
```

### Step 3 — wildcard and leading wildcards
`*foo` = full index scan. Replace with `match_phrase_prefix`, `edge_ngram`, or `index_prefixes`.

### Step 4 — oversharding
Every query hits every shard. 1000 shards = 1000 mini-searches per query.
```json
GET _cat/indices?v&s=pri.store.size:desc
```
Fix: `shrink` old indices, adjust ILM rollover thresholds.

### Step 5 — fielddata on text fields
`fielddata: true` loads tokens into JVM heap — causes GC pressure. Check mappings, switch to `.keyword` subfields for aggregations.

### Step 6 — deep pagination
`from: 9000, size: 10` fetches 9010 docs per shard to return 10. Replace with `search_after` + a Point In Time (PIT):
```json
POST my_index/_pit?keep_alive=1m   // open PIT

GET _search
{
  "pit": { "id": "pit_id", "keep_alive": "1m" },
  "size": 10,
  "sort": [ { "date": "desc" }, { "_id": "asc" } ],
  "search_after": ["2024-03-01", "abc123"]   // sort values from last hit
}
```

### Step 7 — searching too many fields
Use `copy_to` to funnel fields into one `search_field` at index time:
```json
"title": { "type": "text", "copy_to": "search_field" },
"body":  { "type": "text", "copy_to": "search_field" },
"search_field": { "type": "text" }
```
Then search one field instead of four. `search_field` is not stored — no extra disk cost.

---

## Slow indexing

### Check 1 — bulk API
Never index one document at a time. Always use the bulk API:
```json
POST _bulk
{ "index": { "_index": "my_index" } }
{ "title": "doc 1" }
{ "index": { "_index": "my_index" } }
{ "title": "doc 2" }
```
Typical sweet spot: 5-15MB per bulk request, tune from there.

### Check 2 — refresh interval
By default ES refreshes every 1 second — makes new docs searchable but is expensive. For bulk ingestion raise it:
```json
PUT /my_index/_settings
{ "index.refresh_interval": "30s" }
```
Set to `-1` during initial bulk load, then restore afterwards.

### Check 3 — replicas during initial load
Replicas mean every indexed doc is written N+1 times. Set replicas to 0 during bulk load, restore after:
```json
PUT /my_index/_settings
{ "number_of_replicas": 0 }
```

### Check 4 — mapping explosions
Dynamic mapping creates a new field for every new key it sees. Unbounded dynamic mapping on high-cardinality data (e.g. JSON with random keys) causes mapping to grow huge — slows indexing and wastes heap.
```json
PUT /my_index/_mapping
{ "dynamic": "strict" }  // reject unknown fields
// or
{ "dynamic": false }     // ignore unknown fields
```

### Check 5 — merge pressure
Too many segments from heavy indexing → background merges compete with indexing. Signs: high CPU, indexing latency spikes. Usually self-resolving — but `force_merge` on read-only indices cleans it up permanently.

---

## Unstable cluster

### Check 1 — disk watermarks
ES stops allocating shards and eventually makes indices read-only as disk fills:

| Watermark | Default | Action |
|---|---|---|
| `low` | 85% used | Stop allocating new shards to this node |
| `high` | 90% used | Move existing shards off this node |
| `flood_stage` | 95% used | Index goes read-only to prevent data loss |

```json
GET _cluster/health           // red/yellow/green
GET _cat/allocation?v         // disk usage per node
GET _cat/shards?v&h=index,shard,state,node  // unassigned shards
```

### Check 2 — unassigned shards
A `yellow` cluster has unassigned replicas. A `red` cluster has unassigned primaries — some data is unavailable.
```json
GET _cluster/allocation/explain   // tells you exactly why a shard is unassigned
```
Common causes: not enough nodes for replica count, disk watermark hit, node left the cluster.

### Check 3 — split brain
Two parts of the cluster each elect their own master — causes data divergence. Prevented by having **at least 3 master-eligible nodes** (quorum = 2). If you see two masters, check network partitions between nodes.

### Check 4 — heap pressure and GC
Frequent GC pauses cause nodes to appear unresponsive — other nodes may drop them from the cluster. Signs: `OutOfMemoryError` in logs, node dropouts.
```json
GET _nodes/stats/jvm    // heap used, GC stats per node
```
Fix: increase heap (up to 50% of RAM max), reduce fielddata usage, reduce shard count.

### Check 5 — master node overload
Master manages cluster state — if overloaded it can't respond to node heartbeats fast enough, causing false node failures. Dedicated master nodes (not holding data) help on large clusters:
```json
# elasticsearch.yml
node.master: true
node.data: false
```

---

## Circuit breakers

Circuit breakers are ES's self-protection mechanism — they trip **before** an operation would cause an OutOfMemoryError, rejecting the request instead of crashing the node.

Key breakers:

| Breaker | Protects against |
|---|---|
| `request` | Single request using too much heap (e.g. large aggregation) |
| `fielddata` | fielddata cache exceeding its heap limit |
| `in_flight_requests` | Too much data in-flight over the network |
| `parent` | Overall heap usage — last line of defence |

When a breaker trips you get a `429` or `CircuitBreakingException`. It means the operation was too expensive — not a bug, a safety valve.

```json
GET _nodes/stats/breaker    // see current breaker stats and trip counts
```

Fix options:
- Reduce aggregation complexity
- Switch fielddata to doc values (`.keyword`)
- Increase breaker limits (last resort — better to fix the root cause)
- Add more heap / more nodes

---

## Quick reference

| Problem | First things to check |
|---|---|
| Slow queries | Profile API, filter vs must, wildcards, shard count |
| Slow indexing | Bulk API, refresh interval, replica count during load |
| Cluster yellow | Unassigned replicas — disk space, node count |
| Cluster red | Unassigned primaries — node failure, disk flood |
| Node dropping out | Heap/GC pressure, network partition, master overload |
| CircuitBreakingException | Aggregation too large, fielddata, overall heap |
