---
name: milvus-context
description: >
  Shared reference card for all Milvus operations. Load this skill FIRST whenever any
  milvus-* skill is triggered. Use whenever the user mentions Milvus, vector database,
  embedding store, collection, or any milvus-* skill is about to run. Contains Milvus
  deployment modes, cluster limits, index types, consistency levels, schema constraints,
  scalar index types, and authentication patterns. Deployment-specific constraints are
  loaded automatically from the user's configured deployment profile.
---

# Milvus Context

A shared reference card loaded before every other milvus-* skill. Contains the universal
facts, limits, and options that all other Milvus skills defer to rather than re-state.

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
3. Partitions per collection: **4,095 maximum**
4. Vector fields per collection: **4 maximum**
5. topK ceiling: **16,384**; `limit + offset` must not exceed 16,384
6. Segment indexing threshold: segments with fewer than **1,024 rows** use brute-force

---

## Step 3 — Confirm available index types

| Index | Best for | Key params | Constraints |
|-------|----------|------------|-------------|
| HNSW | Default; <50 M vectors; recall ≥95% | M=16, efConstruction=256 | Graph in RAM (high memory) |
| IVF_FLAT | >50 M vectors or topK >2000 | nlist=√N, nprobe=16 | Medium memory |
| IVF_SQ8 | Memory-constrained, slight accuracy drop | nlist=√N, nprobe=16 | Low-medium memory |
| IVF_PQ | Most memory-constrained | nlist=√N, m=dim/4, nbits=8 | m must divide dim evenly |
| SCANN | IVF_PQ alternative with better recall | nlist=√N | Low memory |
| DISKANN | Dataset larger than RAM | search_list=100 | Milvus ≥2.3 + fast NVMe required |
| FLAT | Ground truth / <1 M vectors | — | Brute-force; exact recall |
| AUTOINDEX | Prototyping only | — | Resolves to HNSW internally |
| GPU_IVF_FLAT, GPU_CAGRA | GPU workloads | — | CUDA hardware required |

Default for production RAG: **HNSW** (M=16, efConstruction=256).

---

## Step 4 — Select consistency level

| Level | Guarantee | Performance | When to use |
|-------|-----------|-------------|-------------|
| Strong | Read-your-writes; fully linearisable | Highest latency | Test assertions; financial records |
| Bounded (default) | Reads lag by a bounded delta | Low overhead | Most RAG and search workloads |
| Session | Read-your-own-writes within a session | Low overhead | User-facing apps with immediate feedback |
| Eventually | No freshness guarantee | Lowest overhead | Analytics; batch scoring |

Default: **Bounded**.

---

## Step 5 — Verify authentication pattern and schema constraints

**Authentication (universal):**

| Mode | Code |
|------|------|
| No auth (dev only) | `MilvusClient(uri=uri)` |
| Username + password | `MilvusClient(uri=uri, user="u", password="p", secure=True)` |
| Token | `MilvusClient(uri=uri, token="<token>", secure=True)` |

**Schema constraints:**

- Schema is **immutable** in Milvus ≤2.5 — all fields must be designed before creation
- VARCHAR `max_length`: up to 65,535 characters
- `enable_dynamic_field=True` allows arbitrary extra fields but prevents scalar indexing on them

**Scalar index types for filter performance:**

| Type | Use case |
|------|----------|
| TRIE | VarChar equality and prefix matching |
| INVERTED | General scalar filtering (recommended default) |
| STL_SORT | Numeric range queries |

---

## Deployment overrides

Active deployment profile: **${user_config.deployment_profile}**

The following constraints are authoritative for this session. All capability checks,
index type validation, message-size limits, and auth format decisions must defer to
these values over the universal defaults above.

!`cat "${CLAUDE_SKILL_DIR}/../../profiles/${user_config.deployment_profile}.yaml"`
