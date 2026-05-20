# Milvus Search Parameter Tuning Reference

## HNSW — ef sweep (recall@10 vs latency)

`efConstruction=256`, `M=16`, 1 M FLOAT_VECTOR dim=768 vectors, single query node.
Measurements are indicative — re-run on your hardware and dataset.

| ef | recall@10 | Approx p50 latency | Notes |
|----|-----------|-------------------|-------|
| 10 | ~0.85 | very low | Only use when latency is critical and recall ≥85% is acceptable |
| 32 | ~0.93 | low | Good for high-throughput bulk scoring |
| 64 (default) | ~0.97 | moderate | Recommended starting point for RAG |
| 128 | ~0.99 | high | Use when recall ≥99% is required |
| 256 | ~0.995 | very high | Diminishing returns; only for ground-truth benchmarks |

**Rule:** `ef` must be ≥ `topK`. If `topK` is 20, set `ef ≥ 20`; recommended `ef = max(topK × 2, 64)`.

---

## IVF variants — nprobe sweep

`nlist = 1024` (√1M vectors), 1 M FLOAT_VECTOR dim=768 vectors.

| nprobe | recall@10 | Latency multiplier vs nprobe=1 |
|--------|-----------|-------------------------------|
| 1 | ~0.40 | 1× | Not suitable for production |
| 8 | ~0.80 | ~5× | Acceptable for analytics |
| 16 (default) | ~0.90 | ~8× | Recommended starting point |
| 32 | ~0.95 | ~14× | Use when recall >93% required |
| 64 | ~0.98 | ~25× | High accuracy, high cost |
| 128 | ~0.99 | ~45× | Near-exhaustive; prefer HNSW instead |

**Rule:** `nprobe` must be ≤ `nlist`. Setting `nprobe = nlist` = brute-force (same as FLAT).

**nlist selection:** `nlist = √(total_vectors)` is a good starting heuristic.
For 10 M vectors: `nlist ≈ 3162`.

---

## DISKANN — search_list tuning

For datasets that exceed RAM (requires Milvus ≥2.3 + fast NVMe).

| search_list | recall@10 | Notes |
|-------------|-----------|-------|
| 50 | ~0.90 | Minimum for production |
| 100 (default) | ~0.95 | Recommended starting point |
| 200 | ~0.98 | Higher accuracy; more disk I/O |

**Rule:** `search_list` must be ≥ `topK`.

---

## Hybrid search — weight selection guide

Used with `strategy: "weighted"` reranker when combining dense and sparse (BM25) signals.

| Use case | Dense weight | Sparse (BM25) weight | Rationale |
|----------|-------------|----------------------|-----------|
| General RAG | 0.7 | 0.3 | Semantic similarity dominates; BM25 adds keyword precision |
| Keyword-heavy domain (legal, medical) | 0.4 | 0.6 | Exact term matching is critical |
| Code search | 0.5 | 0.5 | Balanced: code identifiers benefit from both |
| Pure semantic | 1.0 | 0.0 | Only meaningful when BM25 adds noise |

**RRF** (`strategy: "rrf"`, `k=60`) is the safe default when you have no signal weighting
data — it combines rankings without requiring calibrated scores.

---

## Recommended search parameter defaults by use case

| Use case | Index | ef / nprobe | Consistency | Notes |
|----------|-------|-------------|-------------|-------|
| RAG retrieval | HNSW | ef=64 | Bounded | Standard starting point |
| Real-time recommendation | HNSW | ef=32 | Session | Lower ef for throughput |
| Batch offline scoring | IVF_FLAT | nprobe=16 | Eventually | Throughput over freshness |
| Compliance audit trail | HNSW | ef=128 | Strong | Read-after-write required |
| Memory-constrained | IVF_PQ | nprobe=32 | Bounded | Accept ~5% accuracy loss |
