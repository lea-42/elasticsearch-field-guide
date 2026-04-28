# Introduction

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

---

## Local Setup

How to run Elasticsearch and Kibana locally via Docker.

### Pull the Docker Images

Pull the official images before starting the stack:

```bash
# Replace 8.x.x with the version you want
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.x.x
docker pull docker.elastic.co/kibana/kibana:8.x.x
```

Find available versions at the [Elastic Docker registry](https://www.docker.elastic.co/).

### Remove Security

Here's a summary of what we did to get the local Elastic stack working without security:

1. **Disabled security in Elasticsearch**
   — set `xpack.security.enabled=false` and `xpack.security.http.ssl.enabled=false` in the ES environment vars
2. **Removed the `kibana_settings` service**
   — this service existed solely to set the `kibana_system` password via the security API, which no longer exists with security disabled, causing it to time out and fail
3. **Removed security-related env vars from Kibana**
   — `ELASTICSEARCH_USERNAME`, `ELASTICSEARCH_PASSWORD`, `ELASTICSEARCH_PUBLICBASEURL` were no longer needed
4. **Created `config/kibana.yml`** with the full content including `csp.strict: false` to fix the "upgrade your browser" error (the key is `csp.strict`, not `xpack.security.csp.strict` which is invalid in Kibana 9)
5. **Added the kibana.yml mount** to the kibana volumes section in `docker-compose.yml`
   - Removed `ELASTIC_PASSWORD` / `ES_LOCAL_PASSWORD` from the elasticsearch environment vars — no longer needed with security disabled
6. **Removed `ELASTICSEARCH_USERNAME` and `ELASTICSEARCH_PASSWORD`** from Kibana env vars — same reason
7. **Fixed the Elasticsearch healthcheck** — removed the `-u elastic:${ES_LOCAL_PASSWORD}` auth flag from the curl command since there's no auth anymore:
   ```bash
   curl --output /dev/null --silent --head --fail http://elasticsearch:9200
   ```

### Up and Running

```bash
# Start
cd elastic-start-local
docker compose up -d

# Check status
docker compose ps

# Tail logs
docker compose logs -f elasticsearch

# Stop (keeps data)
docker compose down

# Stop and wipe volumes (fresh start)
docker compose down -v

# Copy synonyms file to container
docker cp config/synonyms.txt es-local-dev:/usr/share/elasticsearch/config/
```

To persist files across restarts, mount a folder in `docker-compose.yml` instead:

```yaml
volumes:
  - dev-elasticsearch:/usr/share/elasticsearch/data
  - ./config/resources:/usr/share/elasticsearch/config/resources
```

> Note: Don't mount the entire config directory — it will replace the container's internal config folder.
