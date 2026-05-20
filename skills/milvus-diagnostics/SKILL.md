---
name: milvus-diagnostics
description: >
  Diagnose and fix Milvus problems. Load this skill whenever the user says: search is slow,
  wrong results, search returns nothing, deleted data still appearing, permission denied,
  cannot connect, index not built, results look stale, Milvus is down, query timeout, why is
  Milvus slow, my vectors are not being found, or any Milvus error message. This skill
  coordinates all other milvus-* skills for fixes. Always load milvus-context first.
allowed-tools:
  - mcp__milvus__milvus_get_collection_info
  - mcp__milvus__milvus_query
---

# Milvus Diagnostics Skill

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

Check the Search-Latency-by-Phase Grafana panel (see milvus-observability). Identify the
dominant phase:

```
Queue phase dominant?
  └─ YES → Too many concurrent queries or query nodes overloaded
           Fix: reduce concurrent search rate; scale query nodes

Vector-search phase dominant?
  └─ First search after load?  → Cold index; warm up with 3–5 dummy searches
  └─ No / wrong index size?    → Re-evaluate index type (milvus-index-management)

Reduce phase dominant?
  └─ topK too large?           → Reduce `limit`
  └─ output_fields too broad?  → Specify only needed fields (avoid "*")

[Search slow] log marker present, all phases normal?
  └─ Check milvus-context → Deployment overrides for known delays (e.g., upgrade windows)
     Retry with exponential backoff.
```

---

## Step 3 — Branch B: Wrong or missing results

```
Deleted entities still appearing in search results?
  └─ Bounded consistency + compaction not yet run
     Fix: use Strong consistency on the next search, OR trigger manual compaction
     → milvus-lifecycle-compaction-ttl

Expected entity not found?
  └─ Run milvus_query with the primary key filter:
     { "filter_expr": "id == <expected_id>", "output_fields": ["id"] }

     Entity absent from query?
       └─ Check insert was committed → milvus-data-ingestion Step 6 (verify)

     Entity present in query but missing from vector search?
       └─ Check filter_expr in the search — it may exclude the entity
       └─ Check metric_type mismatch (index vs search must match)
       └─ Check query vector normalisation — for COSINE, un-normalised vectors return
          near-zero similarity scores that fall below implicit thresholds

Wrong result order?
  └─ metric_type mismatch between index and search call
     Fix: rebuild index with correct metric → milvus-index-management Step 4
```

---

## Step 4 — Branch C: Auth / Permission denied

```
Check credential format against milvus-context → Step 5 (Authentication patterns)
and Deployment overrides.

Credential format correct?
  └─ NO  → Fix credential format; retry connection (milvus-connection-auth)
  └─ YES → Check RBAC grants at global / database / collection scope
           → milvus-multi-tenancy Step 4 (RBAC)
```

---

## Step 5 — Branch D: Insert / ingestion failure

```
"Message size too large" error?
  └─ Batch exceeds deployment message-size limit
     Check milvus-context → Deployment overrides for the cap
     Fix: halve batch size; recalculate using milvus-data-ingestion Step 2 formula

Rate limit or throttling?
  └─ Reduce concurrent write rate; add exponential backoff between batches

Insert appears to succeed but query returns empty?
  └─ Consistency lag → wait for Bounded window or switch to Strong
  └─ Collection not loaded → confirm load_state = "Loaded" in milvus_get_collection_info
```

---

## Step 6 — Branch E: Index not built / brute-force search

```
milvus_get_collection_info shows index_status ≠ "Indexed"?

  Segment row_count < 1,024?
    └─ Brute-force is expected below this threshold — not an error; ingest more data

  Segments > 1,024 rows but no index?
    └─ create_index was never called (or failed silently)
       Fix: call PyMilvus create_index → milvus-index-management Step 4
```

---

## Step 7 — Branch F: Empty results (no error)

Work through this checklist in order:

1. **Collection loaded?** `milvus_get_collection_info` → `load_state`. If `"Released"` →
   call `milvus_load_collection` (milvus-collection-lifecycle).

2. **Any data at all?**

   ```json
   {
     "name": "milvus_query",
     "arguments": { "collection_name": "...", "filter_expr": "id > 0", "output_fields": ["id"], "limit": 1 }
   }
   ```

   Empty response → no data ingested. Run milvus-data-ingestion.

3. **Filter too restrictive?** Remove `filter_expr` and re-search — if results appear, the
   filter eliminates all candidates.

4. **Query vector normalised?** For COSINE metric, un-normalised query vectors produce
   near-zero cosine similarity scores. Normalise the vector before searching.

---

## Step 8 — Quick-reference error table

| Error / symptom | Most likely cause | Skill for fix |
|----------------|-------------------|---------------|
| Collection not found | Wrong database or collection name | milvus-collection-lifecycle |
| No available query node | All nodes overloaded | milvus-observability (scale) |
| topK exceeds limit | `limit + offset > 16,384` | milvus-search-optimization |
| Slow after many deletes | Compaction debt | milvus-lifecycle-compaction-ttl |
| Wrong result order | metric_type mismatch | milvus-index-management |
| Stale deletes in results | Bounded consistency + no compaction | milvus-lifecycle-compaction-ttl |
| Cannot connect at all | Auth or network | milvus-connection-auth |

---

## Step 9 — Verify fix and document

Re-run the original failing operation with `consistency_level: "Strong"`. Confirm expected
results. Record in an incident log using `references/incident-log-template.md`.

---

## Reference Files

- `references/incident-log-template.md` — Markdown post-incident template with fields for
  timestamp, symptom, triage branch, root cause, fix applied, verification result, and
  prevention action
