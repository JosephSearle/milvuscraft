---
name: index-management
description: >
  Choose, build, and tune a Milvus vector index. Load this skill whenever the user asks:
  which index should I use, create an index, change index, HNSW parameters, M efConstruction,
  IVF nlist nprobe, IVF_PQ, SCANN, DISKANN, GPU index, FLAT, AUTOINDEX, index recall, index
  memory usage, search inaccurate, re-index, drop index, scalar index, JSON path index, or
  mmap. Always load milvuscraft-context first — check Deployment overrides for restricted index types.
allowed-tools: mcp__milvus__milvus_create_collection
---

# Milvuscraft Index Management Skill

Select and configure the correct Milvus index type for a given use case. For query-time
parameter tuning (ef, nprobe) see milvuscraft-search-optimization. For post-creation index
changes there are no MCP tools — use the PyMilvus fallback documented in Step 4.

---

## Core Philosophy

Set `index_params` at collection creation time — post-creation re-indexing is expensive
on large collections (requires release → drop_index → create_index → load). Always check
milvuscraft-context → Deployment overrides before recommending an index type; some managed
services restrict the available set.

---

## Step 1 — Select the index type

Consult milvuscraft-context → Step 3 for the full index catalogue. Apply this decision tree:

```
Are GPU nodes with CUDA available?
  └─ YES → GPU_IVF_FLAT or GPU_CAGRA

Does the dataset exceed available RAM?
  └─ YES → DISKANN (Milvus ≥2.3 + fast NVMe; mmap NOT supported for DISKANN)

Is this a prototype or POC?
  └─ YES → AUTOINDEX (resolves to HNSW; not for production tuning)

Dataset size?
  └─ <1 M vectors     → FLAT (exact recall; ground truth benchmarks)
  └─ <50 M vectors    → HNSW (default; recall ≥95%; handles active writes well)
  └─ ≥50 M vectors    → IVF_FLAT, IVF_SQ8, IVF_PQ, or SCANN

topK requirement >2000?
  └─ YES → IVF_FLAT (cluster search handles large topK better than HNSW)

Memory constraint?
  └─ Tight → IVF_SQ8 → IVF_PQ → SCANN (decreasing memory, decreasing recall)
```

**IVF vs HNSW trade-off:** HNSW absorbs writes without recall drift and builds a persistent
graph; IVF recall degrades as data changes and may need periodic rebuilds. For active-write
workloads with <50 M vectors, HNSW is almost always the better choice.

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

| Param | Meaning | Server-enforced range | Guidance |
|-------|---------|----------------------|---------|
| M | Graph connectivity (edges per node) | **[4, 64]** | 16 default; higher = better recall, more RAM |
| efConstruction | Build-time candidate list | **[8, 512]** | 256 default; higher = better graph, slower build |

> **Important:** Milvus 2.4.x server enforces M ∈ [4, 64] and efConstruction ∈ [8, 512].
> Setting M=100 or efConstruction=600 returns a validation error even though the public docs
> list wider theoretical ranges. Always stay within these enforced bounds.

**IVF_FLAT:**

```json
{"field_name": "embedding", "index_type": "IVF_FLAT", "metric_type": "COSINE", "params": {"nlist": 1024}}
```

**IVF_PQ:**

```json
{"field_name": "embedding", "index_type": "IVF_PQ", "metric_type": "COSINE", "params": {"nlist": 1024, "m": 16, "nbits": 8}}
```

`m` must divide `dim` evenly (e.g., dim=768 → m=16 works; m=7 does not).
Rule of thumb: `m = dim / 48`.

**DISKANN:**

```json
{"field_name": "embedding", "index_type": "DISKANN", "metric_type": "COSINE", "params": {"search_list": 100}}
```

See `references/index-param-tuning.md` for recall vs memory trade-off tables.

---

## Step 3 — Apply index at collection creation (preferred path)

Pass `index_params` to `milvus_create_collection`. This is the most efficient path — it
avoids a separate create_index call and indexes data as segments are sealed.

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

**Warning:** Dropping an index on a large collection triggers a full re-index. For
zero-downtime re-indexing, use the alias-swap pattern in milvuscraft-collection-lifecycle.

---

## Step 5 — Add scalar indexes for filter performance

Always index scalar fields that appear in `filter_expr` before running vector searches.
Without scalar indexes, filter evaluation falls back to brute-force scan.

```python
# INVERTED: recommended default for most scalar fields
client.create_index("my_collection", "category", {"index_type": "INVERTED"})

# TRIE: VarChar equality and prefix matching
client.create_index("my_collection", "source_url", {"index_type": "TRIE"})

# STL_SORT: numeric range queries
client.create_index("my_collection", "created_ts", {"index_type": "STL_SORT"})

# BITMAP: low-cardinality fields only (cardinality <500; NOT for JSON, FLOAT, DOUBLE)
client.create_index("my_collection", "status", {"index_type": "BITMAP"})
```

**Known issue (Milvus 2.4.x):** INVERTED scalar index on integer `==` queries has been
reported to return only 20–40% of matching results in some builds (issue #32717). If you see
suspiciously low filter precision, validate against STL_SORT or no-index on your exact version.

---

## Step 6 — JSON path indexes (Milvus 2.4+)

Create an INVERTED index on a specific key path inside a JSON field to accelerate filters
on that path. One index per path.

```python
client.create_index(
    "my_collection",
    "metadata",                    # JSON field name
    {
        "index_type": "INVERTED",
        "json_path": "metadata['region']",   # path to the key
        "json_cast_type": "VARCHAR",         # type of the value at that path
    },
)
```

Without a JSON path index, filtering on nested JSON keys forces a full collection scan.
Index all hot filter paths in JSON fields.

---

## Step 7 — mmap for memory-constrained deployments (Milvus 2.4.10+)

mmap maps index/data files to disk instead of loading them into RAM, trading latency for
capacity. Controlled by four flags since Milvus 2.4.10.

```python
# Enable mmap for a collection (release first)
client.release_collection("my_collection")
client.alter_collection_properties(
    "my_collection",
    {"mmap.enabled": True}
)
client.load_collection("my_collection")
```

**Constraints:**
- Cannot enable mmap on a **loaded** collection — release first
- mmap is **not supported** for DISKANN or GPU indexes
- HNSW + mmap: performance degrades ~30% when data doubles; can expand data size up to ~1.8×
- IVF + mmap: performance degrades significantly; prefer HNSW if using mmap
- Prefers NVMe storage for acceptable latency

---

## Step 8 — Verify index state

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "my_collection" }
}
```

- `index_status = "Indexed"` for all segments with ≥1,024 rows → index built successfully
- Segments with <1,024 rows use brute-force — this is expected, not an error

---

## Reference Files

- `references/index-param-tuning.md` — HNSW M/efConstruction/ef recall-vs-latency table,
  IVF nlist/nprobe sweep guide, IVF_PQ m/nbits accuracy-vs-compression curves, and DISKANN
  search_list tuning notes
