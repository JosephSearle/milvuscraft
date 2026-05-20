# Milvus Incident Log Template

Copy this template for each incident. File as `incidents/YYYYMMDD-<short-slug>.md` in your
ops runbook or wiki.

---

## Incident: [short description]

| Field | Value |
|-------|-------|
| **Date / time** | YYYY-MM-DD HH:MM UTC |
| **Duration** | e.g. "23 minutes" |
| **Severity** | P1 / P2 / P3 / P4 |
| **Affected collection(s)** | e.g. `tenant_rag`, `hybrid_docs` |
| **Reported by** | user / alert / monitoring |

---

### Symptom

*What the user or monitor observed. Include exact error messages, search result anomalies,
or latency numbers where available.*

Example: "Search p99 latency spiked to 8 s at 14:32 UTC. Affected collection: `tenant_rag`.
Users reported no results for approximately 20 minutes."

---

### Triage branch taken

*Which branch of milvus-diagnostics Step 2–7 was followed.*

- [ ] Branch A — Slow search
- [ ] Branch B — Wrong or missing results
- [ ] Branch C — Auth / Permission denied
- [ ] Branch D — Insert / ingestion failure
- [ ] Branch E — Index not built / brute-force search
- [ ] Branch F — Empty results (no error)

---

### Baseline state at incident time

*Output of `milvus_get_collection_info` at time of investigation.*

```json
{
  "load_state": "",
  "row_count": 0,
  "index_status": ""
}
```

---

### Root cause

*One or two sentences: what caused the incident. Be specific.*

Example: "Compaction backlog exceeded 200 tasks following a mass-delete of 35% of the
collection, causing deleted entities to remain visible under Bounded consistency."

---

### Fix applied

*Exact steps taken, including MCP tool calls or PyMilvus commands.*

Example:
1. Switched search to `consistency_level: "Strong"` to suppress stale deletes immediately
2. Triggered manual compaction: `client.compact("tenant_rag")`
3. Monitored `milvus_datacoord_compaction_task_num` until it dropped below 10 (18 minutes)
4. Reverted search to `Bounded` consistency

---

### Verification result

*How the fix was confirmed. Include query or search output.*

Example: "Re-ran the failing search with `Strong` consistency. All previously visible deleted
entities are gone. Search p99 returned to 320 ms."

---

### Prevention action

*What change or monitoring will prevent recurrence.*

Example:
- Added `milvus_datacoord_compaction_task_num > 100` Prometheus alert (fires after 5 min)
- Added guidance to runbook: mass deletions >20% of collection should be followed immediately
  by manual `client.compact()` call
- Added `milvus-lifecycle-compaction-ttl` skill to the agent's loaded skill set

---

### Timeline

| Time (UTC) | Event |
|-----------|-------|
| HH:MM | Incident detected |
| HH:MM | Investigation started |
| HH:MM | Root cause identified |
| HH:MM | Fix applied |
| HH:MM | Service restored |
| HH:MM | Incident closed |
