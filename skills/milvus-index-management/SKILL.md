---
name: milvus-index-management
description: >
  Choose, build, and tune a Milvus vector index. Load this skill whenever the user asks:
  which index should I use, create an index, change index, HNSW parameters, M efConstruction,
  IVF nlist nprobe, IVF_PQ, SCANN, DISKANN, GPU index, FLAT, AUTOINDEX, index recall, index
  memory usage, search inaccurate, re-index, drop index, or scalar index. Always load
  milvus-context first — check Deployment overrides for any restricted index types.
allowed-tools: mcp__milvus__milvus_create_collection
---

# Milvus Index Management Skill

Select and configure the correct Milvus index type for a given use case. For query-time
parameter tuning (ef, nprobe) see milvus-search-optimization. For post-creation index
changes there are no MCP tools — use the PyMilvus fallback documented in Step 4.

---

## Core Philosophy

Set `index_params` at collection creation time — post-creation re-indexing is expensive
on large collections (requires release → drop_index → create_index → load). Always check
milvus-context → Deployment overrides before recommending an index type; some managed
services restrict the available set.

---

## Step 1 — Select the index type

Consult milvus-context → Step 3 for the full index catalogue. Apply this decision tree first:

```
Are GPU nodes with CUDA available?
  └─ YES → GPU_IVF_FLAT or GPU_CAGRA
  └─ NO  → continue

Does the dataset exceed available RAM?
  └─ YES → DISKANN (requires Milvus ≥2.3 + fast NVMe)
  └─ NO  → continue

Is this a prototype or POC?
  └─ YES → AUTOINDEX (resolves to HNSW; not for production tuning)
  └─ NO  → continue

Dataset size?
  └─ <1 M vectors     → FLAT (exact recall; ground truth benchmarks)
  └─ <50 M vectors    → HNSW (default; recall ≥95%)
  └─ ≥50 M vectors    → IVF_FLAT, IVF_SQ8, IVF_PQ, or SCANN

topK requirement?
  └─ topK >2000       → IVF_FLAT (cluster search handles large topK better)

Memory constraint?
  └─ Tight            → IVF_SQ8 → IVF_PQ → SCANN (decreasing memory, decreasing recall)
```

---

## Step 2 — Configure index parameters

**HNSW (recommended default):**

```json
{
  "field_name": "embedding",
  "index_type": "HNSW",
  "metric_type": "COSINE",
  "params": {"M": 16, "efConstruction": 256}
}
```

| Param | Meaning | Guidance |
|-------|---------|----------|
| M | Graph connectivity (edges per node) | 8–48; higher = better recall, more RAM |
| efConstruction | Build-time candidate list size | 64–512; higher = better recall, slower build |

**IVF_FLAT:**

```json
{"field_name": "embedding", "index_type": "IVF_FLAT", "metric_type": "COSINE", "params": {"nlist": 1024}}
```

**IVF_PQ:**

```json
{"field_name": "embedding", "index_type": "IVF_PQ",   "metric_type": "COSINE", "params": {"nlist": 1024, "m": 16, "nbits": 8}}
```

`m` must divide `dim` evenly (e.g., dim=768 → m=16 works; m=7 does not).

**DISKANN:**

```json
{"field_name": "embedding", "index_type": "DISKANN",  "metric_type": "COSINE", "params": {"search_list": 100}}
```

See `references/index-param-tuning.md` for recall vs memory trade-off tables.

---

## Step 3 — Apply index at collection creation (preferred path)

Pass `index_params` to `milvus_create_collection`. This avoids a separate create_index call
and is the most efficient path.

```json
{
  "name": "milvus_create_collection",
  "arguments": {
    "collection_name": "my_collection",
    "metric_type": "COSINE",
    "field_schema": "<JSON from milvus-schema-design>",
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

---

## Step 4 — Post-creation index change (PyMilvus fallback)

No MCP tool exists for `create_index` or `drop_index`. For post-creation changes:

```python
# 1. Release (required before dropping index)
client.release_collection("my_collection")

# 2. Drop existing index
client.drop_index("my_collection", "embedding")

# 3. Create new index
client.create_index(
    "my_collection",
    "embedding",
    {
        "index_type": "HNSW",
        "metric_type": "COSINE",
        "params": {"M": 16, "efConstruction": 256},
    },
)

# 4. Reload
client.load_collection("my_collection")
```

**Warning:** Dropping an index on a large collection triggers a full re-index — expect
significant time and I/O. Schedule during off-peak hours.

---

## Step 5 — Add scalar indexes for filter performance

Always index scalar fields that appear in `filter_expr` before running vector searches.
Without scalar indexes, filter evaluation falls back to brute-force scan.

```python
# INVERTED is the recommended default for most scalar fields
client.create_index("my_collection", "category", {"index_type": "INVERTED"})

# TRIE for VarChar equality and prefix matching
client.create_index("my_collection", "source_url", {"index_type": "TRIE"})

# STL_SORT for numeric range queries
client.create_index("my_collection", "created_ts", {"index_type": "STL_SORT"})
```

---

## Step 6 — Verify index state

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

- `index_status = "Indexed"` for all segments with ≥1,024 rows → index built successfully
- Segments with <1,024 rows show brute-force — this is expected, not an error

---

## Reference Files

- `references/index-param-tuning.md` — HNSW M/efConstruction/ef recall-vs-latency table,
  IVF nlist/nprobe sweep guide, IVF_PQ m/nbits accuracy-vs-compression curves, and DISKANN
  search_list tuning notes
