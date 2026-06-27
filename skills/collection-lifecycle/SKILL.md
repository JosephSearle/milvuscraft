---
name: collection-lifecycle
description: >
  Create, load, release, inspect, rename, or drop a Milvus collection. Load this skill
  whenever the user says: create a collection, list collections, load collection, release
  collection, drop collection, describe collection, collection alias, blue-green deploy,
  zero-downtime reindex, shard count, replica count, how many collections exist, or
  collection does not exist. Requires milvuscraft-schema-design for production creation tasks.
  Always load milvuscraft-context first.
allowed-tools: >
  mcp__milvus__milvus_create_collection,
  mcp__milvus__milvus_list_collections,
  mcp__milvus__milvus_load_collection,
  mcp__milvus__milvus_release_collection,
  mcp__milvus__milvus_get_collection_info
---

# Milvuscraft Collection Lifecycle Skill

All collection CRUD operations: create, list, load, release, describe, drop, and alias
management. Consult milvuscraft-context for cluster limits before creating collections.
For index tuning use milvuscraft-index-management; for inserts use milvuscraft-data-ingestion.

---

## Core Philosophy

A collection must be **loaded** before it can be searched or queried, and **released** before
it can be dropped. Always verify collection state with `milvus_get_collection_info` after
creation and after load — don't assume success without confirming `load_state`.

---

## Step 1 — Check existing collections

Before creating, confirm the collection doesn't already exist and the cluster is within limits
(milvuscraft-context → Step 2).

```json
{ "name": "milvus_list_collections", "arguments": {} }
```

If the target collection name is already listed, skip creation and proceed to load if needed.

---

## Step 2 — Choose a creation path

```
Is this a quick prototype or proof-of-concept?
  └─ YES → Path A: quick-setup (auto-schema, minimal parameters)
  └─ NO  → Path B: full schema (run milvuscraft-schema-design first to produce field_schema JSON)
```

**Path A — Quick setup:**

```json
{
  "name": "milvus_create_collection",
  "arguments": {
    "collection_name": "my_collection",
    "dimension": 768,
    "metric_type": "COSINE"
  }
}
```

Defaults applied: `index_type=HNSW`, `M=16`, `efConstruction=256`. Suitable for prototyping;
run milvuscraft-schema-design before any production collection.

**Path B — Full schema:**

```json
{
  "name": "milvus_create_collection",
  "arguments": {
    "collection_name": "my_collection",
    "metric_type": "COSINE",
    "field_schema": "<JSON from milvuscraft-schema-design>",
    "index_params": [
      {
        "field_name": "embedding",
        "index_type": "HNSW",
        "metric_type": "COSINE",
        "params": {"M": 16, "efConstruction": 256}
      }
    ]
  }
}
```

Always set `index_params` at creation time — post-creation re-indexing is expensive on large
collections (see milvuscraft-index-management Step 4 for the post-creation path).

**Scalar indexes:** create immediately after collection creation using PyMilvus:

```python
client.create_index("my_collection", "category", {"index_type": "INVERTED"})
```

**Shard count:** default is 2. Raise to 4+ for write throughput >10K rows/s on Distributed.

---

## Step 3 — Load the collection

A collection must be loaded into memory before searches or queries. Always load after creation.

```json
{
  "name": "milvus_load_collection",
  "arguments": {
    "collection_name": "my_collection",
    "replica_number": 1
  }
}
```

- `replica_number=1` for dev/test; set `2+` for high-availability production
- Never load an already-loaded collection without releasing first

---

## Step 4 — Inspect collection state

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

Confirm: `load_state = "Loaded"` before running any search. Check `index_status` shows
`Indexed` for all vector field segments after indexing.

---

## Step 5 — Manage collection lifecycle

**Release** (free query-node memory; required before drop):

```json
{ "name": "milvus_release_collection", "arguments": { "collection_name": "my_collection" } }
```

**Drop** (irreversible): first release, then use PyMilvus — no MCP tool for drop:

```python
client.drop_collection("my_collection")
```

**Aliases — blue-green zero-downtime reindex / schema migration:**

The canonical pattern for re-indexing or schema migrations without downtime:

```python
# 1. Build and populate new collection
client.create_collection("my_collection_v2", ...)
# ... ingest data into my_collection_v2 ...

# 2. Load new collection
client.load_collection("my_collection_v2")

# 3. Atomically swap the alias (clients querying "prod" see the new collection instantly)
client.alter_alias(collection_name="my_collection_v2", alias="prod")

# 4. Release and drop old collection when safe
client.release_collection("my_collection_v1")
client.drop_collection("my_collection_v1")
```

Alias operations:

```python
client.create_alias(collection_name="my_collection", alias="prod")
client.alter_alias(collection_name="my_collection_v2", alias="prod")  # atomic swap
client.drop_alias(alias="prod")
```

---

## Step 6 — Resolve common failures

| Error | Root cause | Fix |
|-------|-----------|-----|
| Collection already exists | Name collision | Check with `milvus_list_collections` first |
| Exceeds max collections | Cluster limit reached | Check milvuscraft-context → Step 2; release/drop unused |
| OOM on load | Insufficient query-node memory | Reduce `replica_number` or release other collections |
| Drop fails | Collection still loaded | Release before drop |
