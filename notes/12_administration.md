# Elasticsearch admin concepts

---

## Cluster and nodes

- A **cluster** is a group of nodes sharing the same `cluster.name`
- Nodes join automatically when network settings match
- **Horizontal scaling** — add nodes, shards redistribute automatically
- Nodes communicate with each other on **port 9300** (transport)
- Clients communicate via REST on **port 9200** (HTTP)

---

## Shards and replicas

- A node holds multiple indices, each index has multiple shards
- **Shard** — unit of data on disk, keep between 10-50GB
- **Replica** — copy of a shard on a different node

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
