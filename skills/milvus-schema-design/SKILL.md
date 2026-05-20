---
name: milvus-schema-design
description: >
  Design a Milvus collection schema. Load this skill whenever the user asks: design a schema,
  what fields should I create, collection schema, primary key strategy, dynamic fields,
  partition key, add a vector field, BM25 full-text setup, analyzer config, how to model data
  in Milvus, or schema for embeddings. Must run before milvus-collection-lifecycle for any
  production collection creation. Always load milvus-context first.
---

# Milvus Schema Design Skill

Guide through designing a Milvus collection schema before any collection is created. Schema
decisions are **irreversible in Milvus ≤2.5** — this skill must be completed before calling
milvus-collection-lifecycle for any production collection. Outputs a `field_schema` JSON
ready to pass to `milvus_create_collection`.

---

## Core Philosophy

Schema is immutable in Milvus ≤2.5. There is no `ALTER COLLECTION` — a bad schema decision
means dropping and recreating the collection, losing all data. Design everything upfront;
treat the schema as a contract.

---

## Step 1 — Answer the schema decision checklist

Work through each question before writing any field definitions:

1. **Primary key**: Int64 with `autoID=True` (simplest, recommended) or VarChar for external IDs?
2. **Vector count**: How many vector fields? Maximum is **4 per collection**.
3. **Vector type**: FLOAT_VECTOR (dense), SPARSE_FLOAT_VECTOR (BM25/SPLADE), or BINARY_VECTOR (hashes)?
4. **Dimension**: Must match embedding model output exactly. **Cannot change after creation.**
5. **Scalar fields**: Which fields are needed for filtering or output? All must be defined now.
6. **Partition key**: Does any scalar field need `is_partition_key=True`? Only **one field** per collection can be the partition key.
7. **Dynamic field**: Is `enable_dynamic_field=True` needed? Allows arbitrary extra fields but **prevents scalar indexing on those dynamic fields**.
8. **BM25 full-text**: Is keyword search needed? Requires: VarChar source field → `Function(BM25)` → SPARSE_FLOAT_VECTOR target.

---

## Step 2 — Select field types

| Field type | Use case | Key constraint |
|------------|----------|----------------|
| INT64 | Primary keys, timestamps, counts | Max value: 2⁶³−1 |
| VARCHAR | Text, IDs, categories | `max_length` up to 65,535; set generously |
| FLOAT / DOUBLE | Scores, coordinates | — |
| BOOL | Flags | — |
| JSON | Semi-structured metadata | Not indexable; use for read-only extra fields |
| ARRAY | Lists of scalars | Element type must be uniform |
| FLOAT_VECTOR | Dense embeddings | `dim` must match model output |
| SPARSE_FLOAT_VECTOR | BM25 / SPLADE | No fixed dimension |
| BINARY_VECTOR | Hashes | dim in bits; must be divisible by 8 |

---

## Step 3 — Write the field_schema JSON

**Standard RAG schema** (use as starting point):

```json
[
  {"field_name": "id",         "data_type": "Int64",        "is_primary": true, "auto_id": true},
  {"field_name": "chunk_text", "data_type": "VarChar",      "max_length": 512},
  {"field_name": "embedding",  "data_type": "FloatVector",  "dim": 768},
  {"field_name": "category",   "data_type": "VarChar",      "max_length": 64, "is_partition_key": true},
  {"field_name": "created_ts", "data_type": "Int64"}
]
```

For BM25 full-text search, add the Function and sparse field:

```json
[
  {"field_name": "id",              "data_type": "Int64",         "is_primary": true, "auto_id": true},
  {"field_name": "chunk_text",      "data_type": "VarChar",       "max_length": 512,  "enable_analyzer": true},
  {"field_name": "embedding",       "data_type": "FloatVector",   "dim": 768},
  {"field_name": "sparse_embedding","data_type": "SparseFloatVector"}
]
```

With accompanying function definition:

```json
{
  "name": "bm25_fn",
  "function_type": "BM25",
  "input_field_names": ["chunk_text"],
  "output_field_names": ["sparse_embedding"]
}
```

See `references/schema-worked-examples.md` for multi-vector and multi-tenant schema examples.

---

## Step 4 — Confirm metric type

The `metric_type` set at collection creation is **immutable**.

| Embedding type | Recommended metric |
|----------------|--------------------|
| Normalised transformer embeddings | COSINE |
| Raw dot-product models | IP |
| Euclidean distance required | L2 |
| Binary vectors | HAMMING or JACCARD |

---

## Step 5 — Validate before handing off to milvus-collection-lifecycle

1. All filter fields are explicitly defined (not in `enable_dynamic_field`)
2. Dimension matches the model exactly
3. `max_length` on VARCHAR fields is set generously (silent truncation if too low)
4. No more than 4 vector fields total
5. Metric type confirmed
6. Partition key field identified (if multi-tenancy is needed — check milvus-multi-tenancy)

Output the validated `field_schema` JSON and `function_schema` (if BM25) to the user.
These are passed directly to `milvus_create_collection` in milvus-collection-lifecycle.

---

## Reference Files

- `references/schema-worked-examples.md` — Three complete schema examples: simple RAG
  collection, multi-vector hybrid search, and multi-tenant partition-key collection
