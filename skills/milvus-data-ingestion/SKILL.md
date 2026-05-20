---
name: milvus-data-ingestion
description: >
  Insert, upsert, delete, or bulk-insert data into a Milvus collection. Load this skill
  whenever the user mentions: insert data, upsert vectors, delete entities, bulk insert,
  ingest embeddings, load dataset into Milvus, batch insert, write performance, flush,
  large dataset ingestion, data pipeline, or message too large error. Always load
  milvus-context first — check Deployment overrides for any message-size limits.
allowed-tools:
  - mcp__milvus__milvus_insert_data
  - mcp__milvus__milvus_delete_entities
---

# Milvus Data Ingestion Skill

All write operations: insert, upsert, delete, and bulk insert. Provides universal batching
guidance based on Milvus defaults with a clear instruction to check milvus-context →
Deployment overrides for any managed-service message-size caps.

---

## Core Philosophy

Do not flush after every insert. Milvus auto-seals segments when they reach the configured
threshold — manual flush after every batch creates unnecessary compaction pressure and hurts
throughput. Flush explicitly only after the final batch of a large load session or before backup.

---

## Step 1 — Confirm collection is ready

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

`load_state` must be `"Loaded"` before any insert. If not loaded, call
`milvus_load_collection` first (see milvus-collection-lifecycle).

---

## Step 2 — Calculate safe batch size

Default gRPC message limit: **64 MB** (self-hosted). Check milvus-context → Deployment
overrides — some managed deployments enforce a lower cap.

**Per-row size formula:**

```
bytes_per_row = (dim × 4)         # FLOAT_VECTOR: 4 bytes per float
              + scalar_field_bytes  # VARCHAR: actual char count + overhead
```

**Example:** dim=768 FLOAT_VECTOR + 1 KB metadata → ~4 KB/row → **10,000 rows/batch** at 40 MB.

Target: **20–40 MB per insert call** for optimal throughput on self-hosted Milvus.

---

## Step 3 — Insert data in batches

```json
{
  "name": "milvus_insert_data",
  "arguments": {
    "collection_name": "my_collection",
    "data": [
      {"chunk_text": "...", "embedding": [0.1, 0.2, "..."], "category": "finance"},
      {"chunk_text": "...", "embedding": [0.3, 0.4, "..."], "category": "tech"}
    ]
  }
}
```

1. Split dataset into batches sized per Step 2
2. Call `milvus_insert_data` for each batch
3. **Do NOT flush between batches** — let Milvus auto-seal segments
4. For large sessions (>500K cumulative rows): flush explicitly every ~500K rows

**Upsert vs insert:** Upsert overwrites on primary key collision — use when the source may
re-emit previously ingested records. Keep the combined upsert + delete rate ≤0.5 MB/s to
avoid compaction overload.

---

## Step 4 — Delete entities

```json
{
  "name": "milvus_delete_entities",
  "arguments": {
    "collection_name": "my_collection",
    "filter_expr": "id in [101, 102, 103]"
  }
}
```

- Deleted entities may still appear under Bounded consistency until compaction runs
- Use `consistency_level: "Strong"` in searches if immediate deletion visibility is required
- **Mass deletion (>30% of collection):** consider recreating the collection — faster than
  waiting for compaction (see milvus-lifecycle-compaction-ttl)

---

## Step 5 — Bulk insert for large datasets (>500K vectors)

No MCP tool exists for bulk insert. Use the PyMilvus SDK with object storage:

```python
# 1. Prepare data as Parquet or JSON files in object storage (MinIO, S3, GCS, etc.)

# 2. Trigger bulk insert
task_id = client.do_bulk_insert(
    collection_name="my_collection",
    files=["s3://bucket/data/chunk1.parquet", "s3://bucket/data/chunk2.parquet"],
)

# 3. Poll until complete
import time
while True:
    state = client.get_bulk_insert_state(task_id=task_id)
    if state.state_name in ("Completed", "Failed"):
        break
    time.sleep(5)
```

---

## Step 6 — Flush and verify

**Flush** (only after final batch or before backup):

```python
client.flush("my_collection")
```

**Verify insert:**

```json
{
  "name": "milvus_query",
  "arguments": {
    "collection_name": "my_collection",
    "filter_expr": "id == 101",
    "output_fields": ["id", "chunk_text"]
  }
}
```

Check `milvus_get_collection_info` to confirm `row_count` reflects the inserted data.

---

## Step 7 — Resolve common failures

| Error | Root cause | Fix |
|-------|-----------|-----|
| "message size too large" | Batch exceeds deployment cap | Check milvus-context overrides; halve batch size |
| Rate limit / throttling | Write rate too high | Add sleep between batches; use exponential backoff |
| Collection not loaded | Collection in Released state | Call `milvus_load_collection` first |
| Insert succeeds but search returns empty | Consistency lag | Wait for Bounded window or switch to Strong |
