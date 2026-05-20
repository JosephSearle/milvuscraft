---
name: milvus-lifecycle-compaction-ttl
description: >
  Manage Milvus collection TTL, compaction, and cold-data lifecycle. Load this skill whenever
  the user asks: TTL, data expiry, auto-delete old data, compaction, compact collection,
  deleted data still appearing, segment size, cold collection, release memory, storage keeps
  growing, collection slow after many deletes, or tiered storage. Always load milvus-context first.
allowed-tools: mcp__milvus__milvus_release_collection
---

# Milvus Lifecycle, Compaction & TTL Skill

Collection lifecycle beyond creation: TTL expiry, compaction, cold-data release, and segment
hygiene. The Zilliz MCP server does not expose compaction or TTL — all those operations use
the PyMilvus SDK. `milvus_release_collection` is the only MCP tool used here.

---

## Core Philosophy

TTL and compaction are asynchronous. Setting a TTL does not immediately remove data; deletion
is only physically complete after a compaction cycle runs. Design ingestion patterns around
this: if freshness of deletes is critical, use Strong consistency or trigger manual compaction.

---

## Step 1 — Set TTL (time-to-live)

TTL auto-expires entities after N seconds. Deletion is asynchronous: entity is marked → GC
cycle runs → compaction physically removes it. Expired entities may still appear in search
results until compaction completes.

```python
# Set TTL to 30 days (2,592,000 seconds)
client.alter_collection_properties(
    "my_collection",
    {"collection.ttl.seconds": 2592000},
)

# Disable TTL
client.alter_collection_properties(
    "my_collection",
    {"collection.ttl.seconds": 0},
)
```

**Verify:**

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

Check `properties` for `collection.ttl.seconds` in the response.

---

## Step 2 — Trigger or monitor compaction

**Automatic triggers:**

1. Delete binlog exceeds 20% of segment size
2. Segment delta exceeds 10 MB

These fire without intervention. Check milvus-context → Deployment overrides; some managed
services restrict direct access to compaction triggers.

**Manual compaction** (force cleanup after mass delete):

```python
task_id = client.compact("my_collection")

# Poll until complete
import time
while True:
    state = client.get_compaction_state(task_id)
    if state.state_name in ("Completed", "Failed"):
        print(state)
        break
    time.sleep(10)
```

**Caution:** compaction is CPU and I/O intensive. Schedule during off-peak hours.

---

## Step 3 — Decide: compact vs rebuild

```
What fraction of the collection is deleted?
  └─ <30% deleted → compact (Step 2)
  └─ ≥30% deleted → rebuild is faster than compacting

Rebuild pattern:
  1. Create a new collection with the same schema (milvus-collection-lifecycle)
  2. Re-ingest all non-deleted rows (milvus-data-ingestion)
  3. Swap alias to point at the new collection
  4. Release and drop the old collection
```

---

## Step 4 — Tune segment size

Larger segments improve query throughput by reducing the number of per-segment search tasks.

| Config key | Default | Recommended production range |
|-----------|---------|------------------------------|
| `dataCoord.segment.maxSize` | 512 MB | 1,024–8,192 MB |

Set in `milvus.yaml` or Helm values for open-source deployments. Check milvus-context →
Deployment overrides — managed services may lock this setting.

---

## Step 5 — Manage cold collections

Release collections not needed for active queries to free query-node memory:

```json
{
  "name": "milvus_release_collection",
  "arguments": { "collection_name": "cold_collection" }
}
```

Reload on demand before the next search (`milvus_load_collection`). Suggested pattern:
maintain a registry tracking `last_accessed_at`; release any collection idle beyond a
configurable threshold.

---

## Step 6 — Verify lifecycle operations

**After compact:** call `milvus_get_collection_info` — `row_count` should reflect physically
removed rows (lower than before compaction if deleted entities were present).

**After TTL update:** wait `TTL + one compaction cycle`, then query a known expired primary
key — should return empty:

```json
{
  "name": "milvus_query",
  "arguments": {
    "collection_name": "my_collection",
    "filter_expr": "id == 12345",
    "output_fields": ["id"]
  }
}
```
