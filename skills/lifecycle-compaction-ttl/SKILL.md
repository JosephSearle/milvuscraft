---
name: lifecycle-compaction-ttl
description: >
  Manage Milvus collection TTL, compaction, mmap, and cold-data lifecycle. Load this skill
  whenever the user asks: TTL, data expiry, auto-delete old data, compaction, compact
  collection, deleted data still appearing, segment size, clustering compaction, cold
  collection, release memory, storage keeps growing, collection slow after many deletes,
  mmap, or tiered storage. Always load milvuscraft-context first.
allowed-tools: mcp__milvus__milvus_release_collection
---

# Milvuscraft Lifecycle, Compaction & TTL Skill

Collection lifecycle beyond creation: TTL expiry, compaction, clustering compaction, mmap,
cold-data release, and segment hygiene. The Zilliz MCP server does not expose compaction or
TTL — all those operations use PyMilvus. `milvus_release_collection` is the only MCP tool
used here.

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

Check `properties` for `collection.ttl.seconds` in the response. Very short TTL on a
high-ingest collection increases compaction load — size TTL to your compaction capacity.

---

## Step 2 — Trigger or monitor compaction

**Automatic triggers:**

1. Delete binlog exceeds 20% of segment size
2. Segment delta exceeds 10 MB

These fire without intervention. Check milvuscraft-context → Deployment overrides; some
managed services restrict direct access to compaction triggers.

**Manual compaction** (force cleanup after mass delete):

```python
task_id = client.compact("my_collection")

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
  └─ ≥30% deleted → rebuild is faster than waiting for compaction

Rebuild pattern:
  1. Create a new collection with the same schema (milvuscraft-collection-lifecycle)
  2. Re-ingest all non-deleted rows (milvuscraft-data-ingestion)
  3. Swap alias to point at the new collection
  4. Release and drop the old collection
```

---

## Step 4 — Clustering compaction (Milvus 2.4+)

Clustering compaction redistributes data across segments by a **clustering key** (a scalar
field), building a global `PartitionStats`. Subsequent searches with a filter on that field
prune irrelevant segments entirely. Benchmark results show up to **25× search improvement**
for narrow scalar filters on 20 M+ vector collections.

```python
# Set a clustering key (one scalar field per collection)
client.alter_collection_properties(
    "my_collection",
    {"clustering.key.field": "category"}
)

# Trigger clustering compaction
task_id = client.compact("my_collection", is_clustering=True)

import time
while True:
    state = client.get_compaction_state(task_id)
    if state.state_name in ("Completed", "Failed"):
        break
    time.sleep(10)
```

Enable pruning on query nodes:

```yaml
# milvus.yaml
queryNode:
  enableSegmentPrune: true
```

**When to use:** large collections (>5 M vectors) with frequent selective scalar filters
(e.g., `category == 'finance'`, `region == 'us-east'`). Not beneficial for full-collection
scans. Decoupled from DataNode since Milvus 2.4.8 — it runs as a background job.

---

## Step 5 — mmap for memory-constrained deployments (Milvus 2.4.10+)

mmap maps index/data files to disk instead of loading them fully into RAM, trading latency
for capacity. Four independent flags since Milvus 2.4.10:

```python
# Enable mmap for all data types on a collection (release first)
client.release_collection("my_collection")
client.alter_collection_properties(
    "my_collection",
    {
        "mmap.enabled": True,
        # More granular control (set individually as needed):
        # "vectorField.mmap.enabled": True,
        # "vectorIndex.mmap.enabled": True,
        # "scalarField.mmap.enabled": True,
        # "scalarIndex.mmap.enabled": True,
    }
)
client.load_collection("my_collection")
```

**Key constraints:**
- Cannot enable mmap on a **loaded** collection — release first, then set, then reload
- **Not supported** for DISKANN or GPU indexes (error if attempted)
- HNSW + mmap: ~30% performance degradation when data doubles; can expand data size up to ~1.8×
- IVF + mmap: significant performance drop; HNSW degrades more predictably
- Prefers NVMe storage; SATA SSDs will show high latency under load
- If scalar filtering dominates, disable mmap for scalarIndex specifically

---

## Step 6 — Manage cold collections

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

## Step 7 — Tune segment size

Larger segments reduce the number of per-segment search tasks and improve query throughput.

| Config key | Default | Recommended production range |
|-----------|---------|------------------------------|
| `dataCoord.segment.maxSize` | 512 MB | 1,024–8,192 MB |

Set in `milvus.yaml` or Helm values. Check milvuscraft-context → Deployment overrides —
managed services may lock this setting.

---

## Step 8 — Verify lifecycle operations

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
