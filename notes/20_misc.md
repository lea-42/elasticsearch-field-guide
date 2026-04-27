# Stuff that doesn't belong anywhere else

## Bloom Filters
A bloom filter is a cheap heap-resident gatekeeper that prevents unnecessary disk reads by quickly ruling out segments that can't possibly contain a term.

## Data streams

A data stream is a named abstraction over a series of time-based backing indices. Looks like one index from the outside, is many indices under the hood.

**Use for:** logs, metrics, events — any append-only time-series data.

```
logs  ← data stream (one name you write/search against)
  ├── .ds-logs-2024-01-000001  (old, read-only)
  ├── .ds-logs-2024-02-000002  (old, read-only)
  └── .ds-logs-2024-03-000003  (current write target)
```

- **Writes** → always go to the current backing index
- **Reads** → fan out across all backing indices transparently
- **Rollover** → automatic when current index hits size/age threshold — new backing index created, old one sealed
- **ILM** → manages hot/warm/cold movement of backing indices automatically

### Key constraint

Append-only — documents cannot be updated or deleted individually. Use ILM delete phase or `delete_by_query` for removal.

### Key Concept

Data streams are a first-class concept alongside regular indices. APIs that historically operated on indices were updated to cover data streams too — hence the paired phrasing.

### Regular index vs data stream

| | Regular index | Data stream |
|---|---|---|
| Write target | Fixed | Always current backing index |
| Updates | Yes | No |
| Rollover | Manual | Automatic |
| Best for | Structured, mutable data | Time-series, append-only |
