---
name: diagnostics
description: >
  Diagnose and fix Milvus problems. Load this skill whenever the user says: search is slow,
  wrong results, search returns nothing, deleted data still appearing, permission denied,
  cannot connect, index not built, results look stale, Milvus is down, query timeout, why is
  Milvus slow, my vectors are not being found, or any Milvus error message. This skill
  coordinates all other milvuscraft-* skills for fixes. Always load milvuscraft-context first.
allowed-tools: >
  mcp__milvus__milvus_get_collection_info,
  mcp__milvus__milvus_query
---

# Milvuscraft Diagnostics Skill

Triage entry point for all Milvus problems: slow search, wrong results, auth failures,
ingestion errors, or unexpected empty results. Establishes a baseline, branches to the
relevant symptom tree, and references the owning skill for remediation. Always document
root cause and fix — use `references/incident-log-template.md` as the record.

---

## Core Philosophy

Establish baseline state before diagnosing. A single `milvus_get_collection_info` call tells
you whether the connection works, whether the collection is loaded, and whether the index
was built. Most failures trace back to one of these three states being wrong.

---

## Step 1 — Establish baseline

```json
{
  "name": "milvus_get_collection_info",
  "arguments": { "collection_name": "<collection_name>" }
}
```

```
Call succeeds?
  └─ NO  → Go to Branch C: Auth / Permission denied
  └─ YES → Note: load_state, row_count, index_status. Continue to symptom branch.
```

---

## Step 2 — Branch A: Slow search

Check the Search-Latency-by-Phase Grafana panel (see milvuscraft-observability). Identify
the dominant phase:

```
Queue phase dominant?
  └─ YES → Too many concurrent queries or query nodes overloaded
           Fix: reduce concurrent search rate; scale query nodes

Vector-search phase dominant?
  └─ First search after load?  → Cold index; warm up with 3–5 dummy searches
  └─ No / wrong index size?    → Re-evaluate index type (milvuscraft-index-management)
  └─ Strong consistency in use? → Switch to Bounded; Strong adds 200–400 ms

Reduce phase dominant?
  └─ topK too large?           → Reduce `limit`
  └─ output_fields too broad?  → Specify only needed fields (avoid "*")

[Search slow] log marker present, all phases normal?
  └─ Check milvuscraft-context → Deployment overrides for known delays
     Retry with exponential backoff.
```

---

## Step 3 — Branch B: Wrong or missing results

```
Deleted entities still appearing in search results?
  └─ Bounded consistency + compaction not yet run
     Fix: use Strong consistency on the next search, OR trigger manual compaction
     → milvuscraft-lifecycle-compaction-ttl

Expected entity not found?
  └─ Run milvus_query with the primary key filter:
     { "filter_expr": "id == <expected_id>", "output_fields": ["id"] }

     Entity absent from query?
       └─ Insert was not committed → milvuscraft-data-ingestion Step 7 (verify)

     Entity present in query but missing from vector search?
       └─ Check filter_expr in the search — it may exclude the entity
       └─ Check metric_type mismatch (index vs search must match)
       └─ Check query vector normalisation — for COSINE, un-normalised query vectors
          produce near-zero cosine similarity scores that fall below implicit thresholds

Wrong result order?
  └─ metric_type mismatch between index and search call
     Fix: rebuild index with correct metric → milvuscraft-index-management Step 4
```

---

## Step 4 — Branch C: Auth / Permission denied

```
Check credential format against milvuscraft-context → Step 5 (Authentication patterns)
and Deployment overrides.

Credential format correct?
  └─ NO  → Fix credential format; retry connection (milvuscraft-connection-auth)
  └─ YES → Check RBAC grants at global / database / collection scope
           → milvuscraft-multi-tenancy Step 4 (RBAC)

IBM watsonx.data?
  └─ Confirm username is exactly "ibmlhapikey" (not "ibmhapikey" or an email)
  └─ Confirm port from Infrastructure Manager — do not assume 19530
  └─ Confirm server_pem_path cert matches the gRPC host CommonName
  → milvuscraft-connection-auth Step 3
```

---

## Step 5 — Branch D: Insert / ingestion failure

```
"Message size too large" error?
  └─ Batch exceeds deployment message-size limit
     Check milvuscraft-context → Deployment overrides for the cap
     Fix: halve batch size; recalculate using milvuscraft-data-ingestion Step 2 formula

Rate limit or throttling?
  └─ Reduce concurrent write rate; add exponential backoff between batches

Insert appears to succeed but query returns empty?
  └─ Consistency lag → wait for Bounded window or switch to Strong
  └─ Collection not loaded → confirm load_state = "Loaded" in milvus_get_collection_info

Bulk insert fails on partition-key collection?
  └─ Bulk insert is incompatible with partition-key collections
     Fix: switch to batched milvus_insert_data calls
```

---

## Step 6 — Branch E: Index not built / brute-force search

```
milvus_get_collection_info shows index_status ≠ "Indexed"?

  Segment row_count < 1,024?
    └─ Brute-force is expected below this threshold — not an error; ingest more data

  Segments > 1,024 rows but no index?
    └─ create_index was never called (or failed silently)
       Fix: call PyMilvus create_index → milvuscraft-index-management Step 4
```

---

## Step 7 — Branch F: Empty results (no error)

Work through this checklist in order:

1. **Collection loaded?** `milvus_get_collection_info` → `load_state`. If `"Released"` →
   call `milvus_load_collection` (milvuscraft-collection-lifecycle).

2. **Any data at all?**

   ```json
   {
     "name": "milvus_query",
     "arguments": { "collection_name": "...", "filter_expr": "id > 0", "output_fields": ["id"], "limit": 1 }
   }
   ```

   Empty response → no data ingested. Run milvuscraft-data-ingestion.

3. **Filter too restrictive?** Remove `filter_expr` and re-search — if results appear, the
   filter eliminates all candidates.

4. **Query vector normalised?** For COSINE metric, un-normalised query vectors produce
   near-zero cosine similarity scores. Normalise the vector before searching.

---

## Step 8 — Known bugs and version-specific issues

Before escalating, check whether the symptom matches a known Milvus issue:

| Issue | Affected versions | Symptom | Workaround |
|-------|------------------|---------|-----------|
| INVERTED scalar `==` low precision | 2.4.x (issue #32717) | Integer equality filter returns 20–40% of expected results | Validate with STL_SORT or no-index; upgrade if possible |
| HNSW param enforcement vs docs | 2.4.x (discussion #32751) | M>64 or efConstruction>512 rejected at creation | Stay within M ∈ [4,64], efConstruction ∈ [8,512] |
| mmap on DISKANN/GPU | All versions | mmap config silently ignored or errors | Remove mmap from DISKANN/GPU index configs |
| Mixing ORM + MilvusClient | All versions | `SchemaNotReadyException` on Collection() | Use only one interface per connection alias |
| IBM User/Roles v2 APIs | IBM watsonx.data | `grant_privilege_v2` returns error | Manage RBAC via IBM IAM instead |

---

## Step 9 — Quick-reference error table

| Error / symptom | Most likely cause | Skill for fix |
|----------------|-------------------|---------------|
| Collection not found | Wrong database or collection name | milvuscraft-collection-lifecycle |
| No available query node | All nodes overloaded | milvuscraft-observability (scale) |
| topK exceeds limit | `limit + offset > 16,384` | milvuscraft-search-optimization |
| Slow after many deletes | Compaction debt | milvuscraft-lifecycle-compaction-ttl |
| Wrong result order | metric_type mismatch | milvuscraft-index-management |
| Stale deletes in results | Bounded consistency + no compaction | milvuscraft-lifecycle-compaction-ttl |
| Cannot connect at all | Auth or network | milvuscraft-connection-auth |

---

## Step 10 — Verify fix and document

Re-run the original failing operation with `consistency_level: "Strong"`. Confirm expected
results. Record in an incident log using `references/incident-log-template.md`.

---

## Reference Files

- `references/incident-log-template.md` — Markdown post-incident template with fields for
  timestamp, symptom, triage branch, root cause, fix applied, verification result, and
  prevention action
