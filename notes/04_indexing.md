# Indexing

An index is a logical collection of data backed by shards (primary and replicas).

## Index Operations

### Create

```json
PUT index_name

PUT index_name
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 },
  "mappings": { "properties": { "field": { "type": "keyword" } } },
  "aliases": { "my_alias": {} }
}
```

### Delete

```
DELETE index_name
DELETE index_1,index_2
DELETE index_*
```

### Close / Open

Closed index uses no memory, rejects reads/writes:

```
POST index_name/_close
POST index_name/_open
```

### Check exists

```
HEAD index_name
```

### List indices

Without internal `.` indices:

```
GET _cat/indices?v&index=*,-.*
```

### Index settings

View:

```
GET index_name/_settings
```

Update:

```json
PUT index_name/_settings
{
  "index": {
    "number_of_replicas": 2,
    "blocks.write": true
  }
}
```

## Templates

### Index template

```json
PUT _index_template/my_template
{
  "index_patterns": ["logs-*"],
  "priority": 500,
  "template": {
    "settings": { "number_of_shards": 3 },
    "mappings": { "properties": { "field": { "type": "keyword" } } },
    "aliases": { "my_alias": {} }
  }
}
```

### Component template

```json
PUT _component_template/my_component
{
  "template": {
    "settings": { "number_of_shards": 3 }
  }
}
```

### Compose component templates

```json
PUT _index_template/my_template
{
  "index_patterns": ["logs-*"],
  "priority": 500,
  "composed_of": ["component_1", "component_2"]
}
```

### List / Delete

```
GET _index_template
GET _index_template/my_template*
DELETE _index_template/my_template
GET _cat/templates?v&h=name,index_patterns,order
```

## Index Stats

```
GET index_name/_stats
GET index_name/_stats/docs,store,indexing
GET _stats
```

## Shrink

Reduce shard count — all shards must be on one node first:

```json
// prepare
PUT index_name/_settings
{
  "settings": {
    "index.blocks.write": true,
    "index.routing.allocation.require._name": "node1"
  }
}

// shrink
POST index_name/_shrink/index_name_shrunk
{
  "settings": {
    "number_of_shards": 1,
    "index.routing.allocation.require._name": null
  }
}
```

## Split

Increase shard count — no node constraint:

```json
// prepare
PUT index_name/_settings
{ "settings": { "index.blocks.write": true } }

// split
POST index_name/_split/index_name_split
{
  "settings": { "number_of_shards": 6 }
}
```

New shard count must be a **multiple** of original.

## Rollover

Create a new index when conditions are met:

```json
POST logs-write/_rollover
{
  "conditions": {
    "max_size": "50gb",
    "max_age": "7d",
    "max_docs": 1000000
  }
}
```

Requires an alias with `is_write_index: true` wired up first.

## ILM — Index Lifecycle Management

Automates moving indices through lifecycle phases as they age.

**4 phases:**

| Phase | Purpose |
|---|---|
| `hot` | actively written to and queried |
| `warm` | no longer written to, still queried |
| `cold` | rarely queried, optimise for storage |
| `delete` | remove the index |

You don't have to use all phases.

**`min_age`** — minimum age of the index (from creation) before entering that phase.

**Common actions per phase:**

| Action | Phase |
|---|---|
| `rollover` | hot |
| `set_priority` | hot/warm |
| `forcemerge` | warm |
| `shrink` | warm |
| `freeze` | cold |
| `delete` | delete |

### Wiring it up

```json
// 1. create policy
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_size": "50gb", "max_age": "7d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "14d",
        "actions": {
          "set_priority": { "priority": 50 },
          "forcemerge": { "max_num_segments": 1 },
          "readonly": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}

// 2. create index with policy + rollover alias
PUT logs-000001
{
  "settings": {
    "index.lifecycle.name": "my_policy",
    "index.lifecycle.rollover_alias": "logs-write"
  },
  "aliases": {
    "logs-write": { "is_write_index": true }
  }
}
```

Priority controls the order in which shards are recovered after a cluster restart.

**Check ILM status on an index:**

```
GET logs-000001/_ilm/explain
```
