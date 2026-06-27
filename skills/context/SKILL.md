---
name: context
description: >
  Shared reference card for all Milvus operations. Load this skill FIRST whenever any
  milvuscraft-* skill is triggered. Use whenever the user mentions Milvus, vector database,
  embedding store, collection, or any milvuscraft-* skill is about to run. Contains Milvus
  deployment modes, cluster limits, index types, consistency levels, schema constraints,
  scalar index types, version compatibility gates, and authentication patterns. Operators
  must fill in the Deployment overrides section for environment-specific constraints.
---

# Milvuscraft Context

A shared reference card loaded before every other milvuscraft-* skill. Contains the universal
facts, limits, and options that all other Milvus skills defer to rather than re-state.

> **Plugin context:** These skills run inside the milvuscraft plugin, which connects to the
> official Milvus MCP server (`mcp__milvus__*` tools). Any operation not covered by an MCP
> tool falls back to the PyMilvus SDK (`MilvusClient`). Always use MCP tools first; drop to
> PyMilvus only when documented as a "PyMilvus fallback" in the relevant skill.

---

## Step 1 — Identify deployment mode

| Mode | Description | Notes |
|------|-------------|-------|
| Milvus Lite | In-process, single file | Dev and test only; no persistence across restarts |
| Standalone | Single server node | Small-scale production; no resource groups |
| Distributed | Multi-node on Kubernetes | Full feature set; resource groups require this mode |
| Managed cloud | Provider-hosted | Feature set varies; check Deployment overrides below |

---

## Step 2 — Confirm cluster-wide limits

1. Databases per cluster: **64 maximum**
2. Collections per cluster: **65,536 maximum**
3. Partitions per collection: **4,095 maximum** (partition-key uses 64 virtual partitions internally)
4. Vector fields per collection: **4 maximum**
5. topK ceiling: **16,384**; `limit + offset` must not exceed 16,384
6. Segment indexing threshold: segments with fewer than **1,024 rows** use brute-force
7. gRPC message limit (default self-hosted): **64 MB** per insert call

---

## Step 3 — Confirm available index types

| Index | Best for | Key params | Constraints |
|-------|----------|------------|-------------|
| HNSW | Default; <50 M vectors; recall ≥95% | M=16, efConstruction=256 | Graph in RAM (high memory) |
| IVF_FLAT | >50 M vectors or topK >2000 | nlist=√N, nprobe=16 | Medium memory |
| IVF_SQ8 | Memory-constrained, slight accuracy drop | nlist=√N, nprobe=16 | Low-medium memory |
| IVF_PQ | Most memory-constrained | nlist=√N, m=dim/48, nbits=8 | m must divide dim evenly |
| SCANN | IVF_PQ alternative with better recall | nlist=√N | Low memory |
| DISKANN | Dataset larger than RAM | search_list=100 | Milvus ≥2.3 + fast NVMe; **mmap not supported** |
| FLAT | Ground truth / <1 M vectors | — | Brute-force; exact recall |
| AUTOINDEX | Prototyping only | — | Resolves to HNSW internally; not for production tuning |
| GPU_IVF_FLAT, GPU_CAGRA | GPU workloads | — | CUDA hardware required; **mmap not supported** |

Default for production RAG: **HNSW** (M=16, efConstruction=256).

---

## Step 4 — Select consistency level

| Level | Guarantee | Performance | When to use |
|-------|-----------|-------------|-------------|
| Strong | Read-your-writes; fully linearisable | +200–400 ms latency | Test assertions; financial records |
| Bounded (default) | Reads lag by a bounded delta | Low overhead | Most RAG and search workloads |
| Session | Read-your-own-writes within a session | Low overhead | User-facing apps with immediate feedback |
| Eventually | No freshness guarantee | Lowest overhead | Analytics; batch scoring |

Default: **Bounded**. Strong adds ~200–400 ms because Milvus waits for a timetick message
(emitted every 200 ms by root coordinator) to be consumed before responding.

---

## Step 5 — Verify authentication pattern and schema constraints

**Authentication (universal):**

| Mode | Code |
|------|------|
| No auth (dev only) | `MilvusClient(uri=uri)` |
| Username + password | `MilvusClient(uri=uri, user="u", password="p", secure=True)` |
| Token | `MilvusClient(uri=uri, token="<token>", secure=True)` |

See milvuscraft-connection-auth for the full connection flow, including IBM watsonx.data gRPC.

**Schema constraints:**

- Existing schema fields are **immutable** after creation in all Milvus versions — they cannot
  be changed or removed
- Milvus **2.6+** allows *adding* new **nullable scalar fields** via `add_collection_field`
- VARCHAR `max_length`: up to 65,535 characters; set generously (silent truncation if too low)
- `enable_dynamic_field=True` allows arbitrary extra fields but prevents scalar indexing on them

**Scalar index types for filter performance:**

| Type | Use case |
|------|----------|
| INVERTED | General scalar filtering — recommended default for most field types |
| TRIE | VarChar equality and prefix matching |
| STL_SORT | Numeric range queries |
| BITMAP | Low-cardinality scalars only (cardinality <500; NOT for JSON, FLOAT, DOUBLE) |

---

## Step 6 — Version compatibility quick reference

| Feature | Minimum version |
|---------|----------------|
| Databases + partition key | 2.2.9 |
| MilvusClient / SDK v2 | PyMilvus 2.3.7 |
| DISKANN | 2.3 |
| Nullable fields + default values | 2.4 |
| JSON path indexing | 2.4 |
| Clustering compaction | 2.4 (decoupled from DataNode in 2.4.8) |
| mmap 4-flag split | 2.4.10 |
| BM25 built-in full-text search | 2.5 |
| `grant_privilege_v2` (56 privileges, 9 built-in groups) | 2.5 |
| AsyncMilvusClient | PyMilvus 2.5.3 |
| `add_collection_field` (schema evolution for nullable scalars) | 2.6 |
| Schema cache | 2.6 |

---

## Deployment overrides

<!-- OPERATOR: Fill in this section for your specific deployment.              -->
<!-- Examples:                                                                   -->
<!--   - Milvus version pinned by your provider                                -->
<!--   - Validated/restricted index types                                       -->
<!--   - Auth token format (e.g., IBM watsonx.data uses ibmlhapikey + API key) -->
<!--   - gRPC host and port (for IBM: read from Infrastructure Manager)        -->
<!--   - Message-size caps imposed by middleware                                -->
<!--   - Any features disabled by your managed service                         -->
<!--   - T-shirt sizing limits if on a managed tier                            -->
<!-- Leave blank if running standard open-source Milvus.                        -->
