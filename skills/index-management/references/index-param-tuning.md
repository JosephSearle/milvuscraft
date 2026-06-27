# Milvus Index Parameter Tuning Reference

## HNSW — M and efConstruction trade-offs

Higher M and efConstruction produce better recall and index quality but increase build time
and RAM usage. The table below gives indicative values; re-benchmark on your dataset.

**Server-enforced ranges (Milvus 2.4.x):** M ∈ [4, 64], efConstruction ∈ [8, 512].
Values outside these ranges return a validation error even if the public docs list wider ranges.

| M | efConstruction | Build time | RAM overhead | Recall@10 (approx) | Recommended for |
|---|---------------|-----------|-------------|-------------------|-----------------|
| 8 | 128 | Fast | Low | ~0.93 | High-volume inserts, moderate recall |
| 16 | 256 | Moderate | Moderate | ~0.97 | **Default; most RAG workloads** |
| 24 | 256 | Moderate | High | ~0.98 | Higher recall required |
| 32 | 512 | Slow | Very high | ~0.99 | Near-exhaustive recall required |
| 48 | 512 | Very slow | Extreme | ~0.995 | Only for small, high-value datasets |

**RAM estimate:** each vector stored ~1.1–1.5× its raw byte size in the HNSW graph.
For 1 M × dim=768 FLOAT_VECTOR: raw = 3 GB; HNSW graph ≈ 4–5 GB RAM.

---

## IVF — nlist selection and nprobe sweep

### nlist selection

`nlist` is the number of Voronoi clusters. Rule of thumb: `nlist ≈ √(total_vectors)`.

| Dataset size | Recommended nlist |
|-------------|-------------------|
| <100 K | 256 |
| 100 K–1 M | 1,024 |
| 1 M–10 M | 4,096 |
| 10 M–100 M | 16,384 |

Too few clusters → nprobe must be high for good recall. Too many → build time increases,
small clusters hurt recall.

### nprobe tuning (at query time — set in milvuscraft-search-optimization)

nprobe is the number of clusters searched per query. See `milvuscraft-search-optimization →
references/search-param-tuning.md` for the full recall vs latency sweep table.

---

## IVF_PQ — m and nbits accuracy vs compression

`m` = number of sub-quantisers (must divide `dim` evenly). `nbits` = bits per sub-quantiser
(almost always 8).

| dim | m | Compression ratio | Accuracy loss vs FLAT |
|-----|---|-------------------|----------------------|
| 768 | 8 | ~12× | ~10–15% |
| 768 | 16 | ~6× | ~5–8% |
| 768 | 32 | ~3× | ~2–4% |
| 1536 | 16 | ~12× | ~8–12% |
| 1536 | 32 | ~6× | ~4–6% |

**Rule:** `m` must divide `dim` without remainder. Higher `m` = less compression, better recall.
Typical production choice: `m = dim / 48` (e.g., dim=768 → m=16).

---

## DISKANN — search_list tuning

`search_list` controls the candidate list size during graph traversal. Must be ≥ `topK`.

| search_list | recall@10 | Disk I/O | Notes |
|-------------|-----------|----------|-------|
| 50 | ~0.90 | Low | Minimum for production queries |
| 100 (default) | ~0.95 | Moderate | **Recommended starting point** |
| 200 | ~0.98 | High | Use when recall ≥97% required |
| 400 | ~0.99 | Very high | Diminishing returns; consider HNSW instead |

**Hardware requirement:** NVMe SSD with sequential read ≥2 GB/s. DISKANN is I/O-bound;
SATA SSDs will show significant latency under load. mmap is NOT supported for DISKANN.

---

## Metric type — memory and distance implications

| Metric | Distance computation | Normalisation required | Common models |
|--------|---------------------|----------------------|---------------|
| COSINE | 1 − cosine_similarity | No (Milvus normalises internally) | Most transformer models |
| IP | −dot_product | Vectors must be pre-normalised | Models trained with dot-product loss |
| L2 | Euclidean distance | No | Word2Vec, older CNN embeddings |
| HAMMING | XOR bit count | Binary vectors | Locality-sensitive hashing |
| JACCARD | Jaccard distance | Binary vectors | MinHash sketches |

**COSINE vs IP:** For normalised vectors these produce identical rankings. Use COSINE when
unsure — Milvus handles normalisation internally.
