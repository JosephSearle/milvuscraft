---
name: data-ingestion
description: >
  Insert, upsert, delete, or bulk-insert data into a Milvus collection. Load this skill
  whenever the user mentions: insert data, upsert vectors, delete entities, bulk insert,
  ingest embeddings, load dataset into Milvus, batch insert, write performance, async insert,
  high-throughput ingestion, flush, large dataset ingestion, data pipeline, or message too
  large error. Always load milvuscraft-context first — check Deployment overrides for any
  message-size limits.
allowed-tools: >
  mcp__milvus__milvus_insert_data,
  mcp__milvus__milvus_delete_entities
---

# Milvuscraft Data Ingestion Skill

All write operations: insert, upsert, delete, async batch ingestion, and bulk insert.
Provides batching guidance based on Milvus defaults with a clear instruction to check
milvuscraft-context → Deployment overrides for any managed-service message-size caps.

---

## Core Philosophy

Do not flush after every insert. Milvus auto-seals segments when they reach the configured
threshold — manual flush after every batch creates unnecessary compaction pressure and hurts
throughput. Flush explicitly only after the final batch of a large load session or before backup.

Small batches are equally harmful: frequent tiny inserts create many small segments, which
compound into compaction debt and slow searches. Batch to 20–40 MB per call.

---

## Step 1 — Confirm collection is ready

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

`load_state` must be `"Loaded"` before any insert. If not loaded, call
`milvus_load_collection` first (see milvuscraft-collection-lifecycle).

---

## Step 2 — Calculate safe batch size

Default gRPC message limit: **64 MB** (self-hosted). Check milvuscraft-context → Deployment
overrides — some managed deployments enforce a lower cap.

**Per-row size formula:**

```
bytes_per_row = (dim × 4)           # FLOAT_VECTOR: 4 bytes per float
              + scalar_field_bytes   # VARCHAR: actual char count + overhead
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
      {"chunk_text": "...", "embedding": [0.1, 0.2], "category": "finance"},
      {"chunk_text": "...", "embedding": [0.3, 0.4], "category": "tech"}
    ]
  }
}
```

1. Split dataset into batches sized per Step 2
2. Call `milvus_insert_data` for each batch
3. **Do NOT flush between batches** — let Milvus auto-seal segments
4. For large sessions (>500K cumulative rows): flush explicitly every ~500K rows

**Upsert vs insert:**

- `insert()` appends — always creates a new row, even if the primary key already exists
- `upsert()` is delete-then-insert keyed on PK — use for **idempotent re-ingestion** when the
  source may re-emit previously ingested records
- Upsert requires a non-auto PK (VarChar or manually assigned Int64)
- Keep the combined upsert + delete rate ≤ **0.5 MB/s** to avoid compaction overload

---

## Step 4 — High-throughput async ingestion (PyMilvus 2.5.3+)

For ingestion pipelines where throughput matters, `AsyncMilvusClient` with `asyncio.gather`
delivers dramatic gains over sequential sync inserts (benchmarks show ~23× speedup for
batch workloads).

```python
import asyncio
from pymilvus import AsyncMilvusClient

async def ingest(batches: list[list[dict]], uri: str, token: str):
    async with AsyncMilvusClient(uri=uri, token=token, secure=True) as client:
        tasks = [
            client.insert(collection_name="my_collection", data=batch)
            for batch in batches
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

    # Check for errors
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Batch {i} failed: {result}")

asyncio.run(ingest(batches, uri, token))
```

Notes:
- `create_schema()` is synchronous even on the async client
- `search_iterator` is **not** available on AsyncMilvusClient
- Recommended batch size: ~1,000 rows per task when running many concurrent tasks

---

## Step 5 — Delete entities

```json
{
  "name": "milvus_delete_entities",
  "arguments": {
    "collection_name": "my_collection",
    "filter_expr": "id in [101, 102, 103]"
  }
}
```

**How deletes work:** Milvus deletes are **soft (logical)** — a bloom filter marks entities
as deleted immediately, but the actual data is not physically removed until **compaction** runs.
This means:
- Deleted entities may still appear under Bounded consistency until compaction completes
- Storage is not freed immediately after delete
- Use `consistency_level: "Strong"` in searches if immediate deletion visibility is required

**Mass deletion (>30% of collection):** consider recreating the collection — faster than
waiting for compaction to reclaim space (see milvuscraft-lifecycle-compaction-ttl Step 3).

---

## Step 6 — Bulk insert for large datasets (>500K vectors)

No MCP tool exists for bulk insert. Use the PyMilvus SDK with object storage.

**Important constraints:**
- Bulk insert is **not compatible with partition-key collections** — use batched `insert()` instead
- Each file must be <16 GB; supported formats: Parquet, JSON, NumPy

```python
# 1. Prepare data as Parquet files in object storage (MinIO, S3, GCS, etc.)

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
        print(state)
        break
    time.sleep(5)
```

---

## Step 7 — Flush and verify

**Flush** (only after final batch or before backup):

```python
client.flush("my_collection")
```

**Verify insert count:**

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

Check `row_count` reflects inserted data. For row-level verification:

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

---

## Step 8 — Resolve common failures

| Error | Root cause | Fix |
|-------|-----------|-----|
| "message size too large" | Batch exceeds deployment cap | Check milvuscraft-context overrides; halve batch size |
| Rate limit / throttling | Write rate too high | Add sleep between batches; use exponential backoff |
| Collection not loaded | Collection in Released state | Call `milvus_load_collection` first |
| Insert succeeds but search returns empty | Consistency lag | Wait for Bounded window or switch to Strong |
| Bulk insert fails on partition-key collection | Incompatible feature | Switch to batched `milvus_insert_data` |
