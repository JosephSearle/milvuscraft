# Milvus Schema — Worked Examples

## Example 1: Simple RAG collection

A single dense embedding field with scalar metadata. Use for standard retrieval-augmented
generation pipelines with a single embedding model.

```json
{
  "collection_name": "rag_docs",
  "metric_type": "COSINE",
  "field_schema": [
    {"field_name": "id",          "data_type": "Int64",       "is_primary": true, "auto_id": true},
    {"field_name": "chunk_text",  "data_type": "VarChar",     "max_length": 2048},
    {"field_name": "source_url",  "data_type": "VarChar",     "max_length": 512},
    {"field_name": "embedding",   "data_type": "FloatVector", "dim": 768},
    {"field_name": "created_ts",  "data_type": "Int64"},
    {"field_name": "doc_id",      "data_type": "VarChar",     "max_length": 128}
  ],
  "index_params": [
    {
      "field_name": "embedding",
      "index_type": "HNSW",
      "metric_type": "COSINE",
      "params": {"M": 16, "efConstruction": 256}
    },
    {"field_name": "source_url", "index_type": "INVERTED"},
    {"field_name": "doc_id",     "index_type": "INVERTED"}
  ]
}
```

**Notes:**
- `doc_id` and `source_url` have INVERTED scalar indexes for efficient filter expressions
- `created_ts` stores Unix epoch milliseconds (Int64)

---

## Example 2: Multi-vector hybrid search (dense + BM25)

Combines dense semantic search with BM25 keyword search for higher recall on keyword-heavy
domains (legal, medical, technical documentation).

```json
{
  "collection_name": "hybrid_docs",
  "field_schema": [
    {"field_name": "id",               "data_type": "Int64",            "is_primary": true, "auto_id": true},
    {"field_name": "chunk_text",       "data_type": "VarChar",          "max_length": 2048, "enable_analyzer": true},
    {"field_name": "embedding",        "data_type": "FloatVector",      "dim": 1536},
    {"field_name": "sparse_embedding", "data_type": "SparseFloatVector"},
    {"field_name": "category",         "data_type": "VarChar",          "max_length": 64},
    {"field_name": "doc_id",           "data_type": "VarChar",          "max_length": 128}
  ],
  "function_schema": [
    {
      "name": "bm25_fn",
      "function_type": "BM25",
      "input_field_names": ["chunk_text"],
      "output_field_names": ["sparse_embedding"]
    }
  ],
  "index_params": [
    {
      "field_name": "embedding",
      "index_type": "HNSW",
      "metric_type": "COSINE",
      "params": {"M": 16, "efConstruction": 256}
    },
    {
      "field_name": "sparse_embedding",
      "index_type": "SPARSE_INVERTED_INDEX",
      "metric_type": "BM25"
    },
    {"field_name": "category", "index_type": "INVERTED"},
    {"field_name": "doc_id",   "index_type": "INVERTED"}
  ]
}
```

**Hybrid search call:**

```json
{
  "name": "milvus_hybrid_search",
  "arguments": {
    "collection_name": "hybrid_docs",
    "query_vectors": {
      "embedding":        [0.1, 0.2, "...1536 floats..."],
      "sparse_embedding": {"indices": [10, 50, 200], "values": [0.7, 0.3, 0.5]}
    },
    "limit": 10,
    "rerank": {"strategy": "rrf", "params": {"k": 60}},
    "output_fields": ["chunk_text", "category"]
  }
}
```

---

## Example 3: Multi-tenant partition-key collection

Partition key routes queries to the correct virtual partition automatically, making it
efficient for up to ~10M tenants. See milvus-multi-tenancy for strategy selection.

```json
{
  "collection_name": "tenant_docs",
  "metric_type": "COSINE",
  "field_schema": [
    {"field_name": "id",         "data_type": "Int64",       "is_primary": true, "auto_id": true},
    {"field_name": "tenant_id",  "data_type": "VarChar",     "max_length": 128,  "is_partition_key": true},
    {"field_name": "chunk_text", "data_type": "VarChar",     "max_length": 2048},
    {"field_name": "embedding",  "data_type": "FloatVector", "dim": 768},
    {"field_name": "doc_id",     "data_type": "VarChar",     "max_length": 128}
  ],
  "index_params": [
    {
      "field_name": "embedding",
      "index_type": "HNSW",
      "metric_type": "COSINE",
      "params": {"M": 16, "efConstruction": 256}
    },
    {"field_name": "doc_id", "index_type": "INVERTED"}
  ]
}
```

**Tenant-scoped search:**

```json
{
  "name": "milvus_vector_search",
  "arguments": {
    "collection_name": "tenant_docs",
    "query_vector": [0.1, 0.2, "..."],
    "limit": 10,
    "filter_expr": "tenant_id == 'acme'",
    "output_fields": ["chunk_text", "doc_id"]
  }
}
```

**Validation:** Insert rows with `tenant_id="a"`, search with `filter_expr="tenant_id == 'a'"`,
confirm no rows from `tenant_id="b"` appear.
