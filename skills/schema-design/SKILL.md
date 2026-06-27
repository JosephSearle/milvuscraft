---
name: schema-design
description: >
  Design a Milvus collection schema. Load this skill whenever the user asks: design a schema,
  what fields should I create, collection schema, primary key strategy, dynamic fields,
  nullable fields, partition key, add a vector field, BM25 full-text setup, analyzer config,
  how to model data in Milvus, schema for embeddings, or add field to existing collection.
  Must run before milvuscraft-collection-lifecycle for any production collection creation.
  Always load milvuscraft-context first.
---

# Milvuscraft Schema Design Skill

Guide through designing a Milvus collection schema before any collection is created. Schema
decisions are largely irreversible — this skill must be completed before calling
milvuscraft-collection-lifecycle for any production collection. Outputs a `field_schema` JSON
ready to pass to `milvus_create_collection`.

---

## Core Philosophy

Most schema decisions cannot be undone. There is no `ALTER COLUMN` — changing a field's type,
dimension, or primary key means dropping and recreating the collection and losing all data.
Design everything upfront; treat the schema as a contract.

**The exception (Milvus 2.6+):** new nullable scalar fields can be added to an existing
collection via `add_collection_field` — but only if `nullable=True`. This is for evolving
optional metadata, not for correcting fundamental design mistakes.

---

## Step 1 — Answer the schema decision checklist

Work through each question before writing any field definitions:

1. **Primary key**: Int64 with `autoID=True` (simplest, best throughput) or VarChar for
   external IDs needed for idempotent upsert?
2. **Vector count**: How many vector fields? Maximum is **4 per collection**.
3. **Vector type**: FLOAT_VECTOR (dense), SPARSE_FLOAT_VECTOR (BM25/SPLADE), or BINARY_VECTOR?
4. **Dimension**: Must match embedding model output exactly. **Cannot change after creation.**
5. **Scalar fields**: Which fields are needed for filtering or output? All must be defined now.
6. **Nullable / defaults**: Which scalar fields are optional? Mark them `nullable=True` and
   set `default_value` if a sensible default exists. Cannot be changed after creation.
7. **Partition key**: Does any scalar field need `is_partition_key=True`? Only **one field**
   per collection. Choose a field with bounded cardinality (not a unique ID).
8. **Dynamic field**: Is `enable_dynamic_field=True` needed? Allows arbitrary extra fields
   but **prevents scalar indexing on those dynamic fields**.
9. **BM25 full-text**: Is keyword search needed? Requires: VarChar source field with
   `enable_analyzer=True` → `Function(BM25)` → SPARSE_FLOAT_VECTOR target.
10. **Metric type**: Set at creation — immutable. See Step 4.

---

## Step 2 — Select field types

| Field type | Use case | Key constraint |
|------------|----------|----------------|
| INT64 | Primary keys, timestamps, counts | Max value: 2⁶³−1 |
| VARCHAR | Text, IDs, categories | `max_length` up to 65,535; set generously — silent truncation if too low |
| FLOAT / DOUBLE | Scores, coordinates | — |
| BOOL | Flags | — |
| JSON | Semi-structured metadata | Not directly indexable; use JSON path indexes for hot keys |
| ARRAY | Lists of scalars | Element type must be uniform |
| FLOAT_VECTOR | Dense embeddings | `dim` must match model output |
| SPARSE_FLOAT_VECTOR | BM25 / SPLADE | No fixed dimension |
| BINARY_VECTOR | Hashes | dim in bits; must be divisible by 8 |

**Nullable fields (Milvus 2.4+):**

```json
{"field_name": "source_url", "data_type": "VarChar", "max_length": 512, "nullable": true}
{"field_name": "score",      "data_type": "Float",   "nullable": true, "default_value": 0.0}
```

Logic: `nullable=True + default_value + field omitted at insert → default`. `nullable=True +
no default + field omitted → null`. Without `nullable`, omitting a field is an error.
The `nullable` attribute **cannot be changed after collection creation**.

---

## Step 3 — Write the field_schema JSON

**Standard RAG schema** (use as starting point):

```json
[
  {"field_name": "id",         "data_type": "Int64",        "is_primary": true, "auto_id": true},
  {"field_name": "chunk_text", "data_type": "VarChar",      "max_length": 2048},
  {"field_name": "embedding",  "data_type": "FloatVector",  "dim": 768},
  {"field_name": "category",   "data_type": "VarChar",      "max_length": 64, "is_partition_key": true},
  {"field_name": "created_ts", "data_type": "Int64"},
  {"field_name": "source_url", "data_type": "VarChar",      "max_length": 512, "nullable": true}
]
```

For BM25 full-text search, add the Function and sparse field:

```json
[
  {"field_name": "id",               "data_type": "Int64",         "is_primary": true, "auto_id": true},
  {"field_name": "chunk_text",       "data_type": "VarChar",       "max_length": 2048, "enable_analyzer": true},
  {"field_name": "embedding",        "data_type": "FloatVector",   "dim": 768},
  {"field_name": "sparse_embedding", "data_type": "SparseFloatVector"}
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

## Step 5 — Plan scalar indexes

Every field that will appear in `filter_expr` needs a scalar index, or Milvus falls back to
brute-force scan. Define the index plan now; apply it alongside vector index creation in
milvuscraft-collection-lifecycle.

| Field type | Recommended index | Notes |
|-----------|------------------|-------|
| VARCHAR (equality/prefix) | TRIE | |
| VARCHAR (general filter) | INVERTED | |
| INT / FLOAT (range) | STL_SORT | |
| Low-cardinality scalar (<500 distinct values) | BITMAP | Not for JSON, FLOAT, DOUBLE |
| JSON key path (hot filter path) | INVERTED on path | Milvus 2.4+; one path per index |

**Known issue (Milvus 2.4.x):** INVERTED scalar index on integer `==` queries has been
reported to return only 20–40% of matching results in some 2.4.x builds (issue #32717).
Validate filter correctness on your exact version; fall back to STL_SORT if precision is critical.

---

## Step 6 — Schema evolution (Milvus 2.6+)

To add a new optional field to an existing collection without recreating it:

```python
from pymilvus import MilvusClient, DataType

client.add_collection_field(
    collection_name="my_collection",
    field_name="language",
    data_type=DataType.VARCHAR,
    max_length=32,
    nullable=True,          # required — new fields must be nullable
    default_value="en",     # optional but recommended
)
```

Constraints: the field must be `nullable=True`; vector fields cannot be added this way;
`enable_dynamic_field` cannot be toggled; adding a field to a loaded collection increases
memory usage on all query nodes.

---

## Step 7 — Validate before handing off to milvuscraft-collection-lifecycle

1. All filter fields are explicitly defined (not buried in `enable_dynamic_field`)
2. Dimension matches the model exactly
3. `max_length` on VARCHAR fields is set generously
4. No more than 4 vector fields total
5. Metric type confirmed and immutability accepted
6. Nullable / default decisions recorded (cannot change later)
7. Partition key field identified if multi-tenancy is needed (see milvuscraft-multi-tenancy)
8. Scalar index plan documented for Step 5 of milvuscraft-collection-lifecycle

Output the validated `field_schema` JSON and `function_schema` (if BM25) to the user.

---

## Reference Files

- `references/schema-worked-examples.md` — Three complete schema examples: simple RAG
  collection, multi-vector hybrid search, and multi-tenant partition-key collection
