---
name: search-optimization
description: >
  Build and tune Milvus searches. Load this skill whenever the user asks: vector search,
  ANN search, similarity search, semantic search, hybrid search, BM25 text search, filter
  query, search parameters, ef nprobe, consistency level, search is slow, wrong search
  results, topK, output_fields, reranker, RRF, weighted reranker, grouping search, range
  search, search iterator, pagination beyond 16384 results, or how to query Milvus.
  Always load milvuscraft-context first.
allowed-tools: >
  mcp__milvus__milvus_vector_search,
  mcp__milvus__milvus_text_search,
  mcp__milvus__milvus_hybrid_search,
  mcp__milvus__milvus_query,
  mcp__milvus__milvus_text_similarity_search
---

# Milvuscraft Search Optimization Skill

All search and query patterns: vector ANN search, scalar filter queries, hybrid search,
BM25 text search, grouping, range search, and large result pagination. Consult
milvuscraft-index-management for index parameter background and milvuscraft-context for
consistency level semantics.

---

## Core Philosophy

Choose the most specific tool for the user's intent — don't default to vector search when
a scalar query or hybrid search is more appropriate. Always specify `output_fields`
explicitly; using `"*"` returns all fields and is expensive at scale.

---

## Step 1 — Select the correct search tool

| User intent | Tool |
|-------------|------|
| Find vectors similar to a query embedding | `milvus_vector_search` |
| Scalar filter only, no vector component | `milvus_query` |
| BM25 full-text keyword search | `milvus_text_search` |
| Dense + sparse / dense + BM25 combined | `milvus_hybrid_search` |
| Semantic similarity using server-side embedding | `milvus_text_similarity_search`* |

*`milvus_text_similarity_search` requires Milvus ≥2.6 **and** a server-side embedding
Function configured in the collection schema. Fall back to `milvus_hybrid_search` with
client-side embeddings on earlier versions.

---

## Step 2 — Set consistency level and search parameters

Consult milvuscraft-context → Step 4 for consistency level semantics.
Default: `Bounded`. Use `Strong` for test assertions or read-after-write guarantees only —
it adds ~200–400 ms latency under write load.

**HNSW search parameters:**

- `ef` ≥ `topK`; recommended: `ef = max(topK × 2, 64)`
- Higher `ef` = better recall, slower query

**IVF variants:**

- `nprobe = 16` default; increase to 32–64 for higher recall at cost of latency

See `references/search-param-tuning.md` for recall vs latency sweep tables.

---

## Step 3 — Execute a vector search

```json
{
  "name": "milvus_vector_search",
  "arguments": {
    "collection_name": "my_collection",
    "query_vector": [0.1, 0.2],
    "limit": 10,
    "output_fields": ["chunk_text", "category"],
    "filter_expr": "category == 'finance'",
    "search_params": {"ef": 64},
    "consistency_level": "Bounded"
  }
}
```

**Limits:** `topK` ceiling is 16,384; `limit + offset` must not exceed 16,384.

**Pre-filter strategy:** `filter_expr` runs on scalar indexes before the ANN phase — always
index scalar fields used in filters (see milvuscraft-index-management Step 5). Filter syntax:

| Pattern | Example |
|---------|---------|
| Equality | `"status == 'active'"` |
| Range | `"score > 0.8 and score < 1.0"` |
| IN list | `"id in [1, 2, 3]"` |
| Compound | `"category == 'a' and score > 0.5"` |

**Query vector normalisation:** for COSINE metric, un-normalised query vectors produce
near-zero cosine similarity scores that fall below implicit thresholds. Always normalise
the query vector when using COSINE.

---

## Step 4 — Execute a hybrid search

Combines multiple vector fields (e.g., dense + sparse BM25). Both vector fields must exist
in the schema (see milvuscraft-schema-design).

```json
{
  "name": "milvus_hybrid_search",
  "arguments": {
    "collection_name": "my_collection",
    "query_vectors": {
      "embedding": [0.1, 0.2],
      "sparse_embedding": {"indices": [10, 50], "values": [0.7, 0.3]}
    },
    "limit": 10,
    "rerank": {
      "strategy": "rrf",
      "params": {"k": 60}
    },
    "output_fields": ["chunk_text"]
  }
}
```

**Reranker options:**

| Strategy | When | Config |
|----------|------|--------|
| RRF (default) | Signals have unknown relative importance | `{"strategy": "rrf", "params": {"k": 60}}` |
| Weighted | Signals have known importance ratios | `{"strategy": "weighted", "params": {"weights": [0.7, 0.3]}}` |

---

## Step 5 — Execute a scalar query or text search

**Scalar query (no vector component):**

```json
{
  "name": "milvus_query",
  "arguments": {
    "collection_name": "my_collection",
    "filter_expr": "tenant_id == 'acme'",
    "output_fields": ["id", "chunk_text"],
    "limit": 100
  }
}
```

**BM25 text search** (requires BM25 Function in schema — see milvuscraft-schema-design):

```json
{
  "name": "milvus_text_search",
  "arguments": {
    "collection_name": "my_collection",
    "query_text": "search terms here",
    "anns_field": "sparse_embedding",
    "limit": 10,
    "output_fields": ["chunk_text"]
  }
}
```

**Grouping search** — return the top-N results deduplicated by a scalar field (e.g., one
chunk per source document):

```json
{
  "name": "milvus_vector_search",
  "arguments": {
    "collection_name": "my_collection",
    "query_vector": [0.1, 0.2],
    "limit": 5,
    "group_by_field": "doc_id",
    "group_size": 1,
    "output_fields": ["chunk_text", "doc_id"]
  }
}
```

Note: in grouping search, `limit` = number of **groups** returned, not total entities.

---

## Step 6 — Range search

Return only results within a distance band from the query vector.

```json
{
  "name": "milvus_vector_search",
  "arguments": {
    "collection_name": "my_collection",
    "query_vector": [0.1, 0.2],
    "limit": 100,
    "search_params": {
      "ef": 128,
      "radius": 0.3,
      "range_filter": 0.9
    },
    "output_fields": ["chunk_text"]
  }
}
```

Distance band conventions:
- **COSINE / IP:** `range_filter > radius` (higher score = more similar)
- **L2:** `range_filter < radius` (lower distance = more similar)

---

## Step 7 — Paginate beyond 16,384 results (search iterator)

A single ANN search returns at most **16,384** entities. For larger result sets use
`client.search_iterator()` via PyMilvus — it guarantees no duplicates across pages and
respects filter expressions.

```python
from pymilvus import MilvusClient

client = MilvusClient(uri=..., token=...)

iterator = client.search_iterator(
    collection_name="my_collection",
    data=[[0.1, 0.2, ...]],      # query vector
    batch_size=1000,              # results per internal page
    limit=50000,                  # total results to retrieve (or -1 for all)
    output_fields=["chunk_text"],
    filter="category == 'finance'",
)

all_results = []
while True:
    batch = iterator.next()
    if not batch:
        iterator.close()
        break
    all_results.extend(batch)
```

**Constraints:** iterators are stateful and **not supported** on `AsyncMilvusClient`.
Use the sync `MilvusClient` for iterator-based pagination.

---

## Step 8 — Verify search results

1. Known-vector test: search with `limit=1`, `consistency_level="Strong"` → confirm the
   expected primary key is returned.
2. Filter sanity: run `milvus_query` with the same `filter_expr` → result set should overlap
   with the search pre-filter population.
3. Empty results: check collection is loaded, data was inserted, and the filter is not
   eliminating all candidates. See milvuscraft-diagnostics Branch F.

---

## Reference Files

- `references/search-param-tuning.md` — HNSW `ef` recall vs latency sweep, IVF `nprobe`
  sweep, and hybrid search weight selection guide
