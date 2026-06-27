---
name: observability
description: >
  Set up monitoring and observability for Milvus. Load this skill whenever the user asks:
  Milvus metrics, Prometheus scraping, Grafana dashboard, search latency metric, insert
  throughput metric, segment count, compaction task queue, SLO for Milvus, alert on Milvus,
  how do I know if Milvus is healthy, or what metrics should I track. Always load
  milvuscraft-context first.
---

# Milvuscraft Observability Skill

Wire Prometheus/Grafana monitoring for Milvus, identify key metrics and dashboards, and
define SLO baselines. For diagnosing a live incident use milvuscraft-diagnostics; this skill
covers instrumentation setup and steady-state monitoring only.

---

## Core Philosophy

The four panels that matter most are: Slow-Query, Search-Latency-by-Phase,
Compaction-Task-Count, and Insert-Throughput. Configure these before anything else. An alert
on `milvus_proxy_sq_latency_bucket p99 > 2s` is cheaper than a production incident.

---

## Step 1 — Check for managed monitoring

Check milvuscraft-context → Deployment overrides for any provider-native observability
platform that exposes Milvus metrics without self-hosting Prometheus. If available, use it
first — it requires no additional configuration.

If no managed monitoring integration exists, proceed to Step 2.

---

## Step 2 — Configure Prometheus scraping

Milvus exposes metrics at `/metrics` on its metrics port (default: **9091**).
Metric namespace: `milvus_*`. Subsystems: `rootcoord`, `proxy`, `querycoord`, `querynode`,
`indexcoord`, `indexnode`, `datacoord`, `datanode`.

**Prometheus Operator (ServiceMonitor):**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: milvus
  namespace: milvus
spec:
  selector:
    matchLabels:
      app: milvus
  endpoints:
    - port: metrics          # must match the service port name
      interval: 30s
      path: /metrics
```

**Raw scrape_config (non-Operator):**

```yaml
scrape_configs:
  - job_name: milvus
    static_configs:
      - targets: ["<milvus-host>:9091"]
    scrape_interval: 30s
```

> Never expose `:9091` publicly — it carries internal cluster state. Restrict to your
> monitoring subnet.

---

## Step 3 — Key metrics to instrument

| Metric | Subsystem | What it measures | Alert threshold |
|--------|-----------|-----------------|-----------------|
| `milvus_proxy_sq_latency_bucket` | proxy | Search/query e2e latency (histogram) | p99 > 2 s |
| `milvus_proxy_mutation_latency_bucket` | proxy | Insert/upsert/delete latency | p99 > 500 ms |
| `milvus_querynode_segment_num` | querynode | Loaded segment count | Sudden drop (unintended release) |
| `milvus_datacoord_compaction_task_num` | datacoord | Pending compaction tasks | >100 sustained |
| `milvus_querynode_sq_queue_latency_bucket` | querynode | Time waiting in search queue | p99 > 500 ms |
| `milvus_datanode_flush_size` | datanode | Data flushed to object storage | Unexpected spikes |

---

## Step 4 — Configure essential Grafana panels

Set up these four panels before anything else:

1. **Service-Quality → Slow-Query**: searches exceeding the slow-query threshold (default 5 s).
   This is the first panel to check on user-reported slow search.

2. **Query-Node → Search-Latency-by-Phase**: breaks latency into queue / vector-search / reduce.
   The dominant phase points to the bottleneck — hand off to milvuscraft-diagnostics for remediation.

3. **Data-Coord → Compaction-Task-Count**: compaction backlog. A rising count after heavy
   deletes signals compaction debt (see milvuscraft-lifecycle-compaction-ttl).

4. **Proxy → Insert-Throughput**: rows/s ingested. Confirms ingestion pipeline health.

See `references/grafana-dashboard-json.md` for official Milvus dashboard templates and
import instructions.

---

## Step 5 — Define SLO baselines

Tune these to your data and SLA; use them as starting points:

| Metric | Starting target | Action if breached |
|--------|----------------|-------------------|
| Search p99 | <500 ms | Run milvuscraft-diagnostics |
| Insert p99 | <200 ms | Check batch sizing (milvuscraft-data-ingestion) |
| Collection load time | <60 s | Check segment count and replica number |
| Compaction backlog | <50 tasks | Reduce delete rate or trigger manual compact |

---

## Step 6 — Monitor log markers

Watch for these structured log markers in Milvus output:

| Marker | Meaning | Alert condition |
|--------|---------|----------------|
| `[Search slow]` | Search exceeded slow-query threshold; contains collection, duration, phase breakdown | >5 occurrences/min in production |
| `[compaction trigger]` | Auto-compaction fired | Informational; watch for high frequency |
| `[segment sealed]` | Segment sealed normally during ingestion | Informational |

---

## Reference Files

- `references/grafana-dashboard-json.md` — Links to official Milvus Grafana dashboard JSON
  templates, import instructions, recommended Prometheus alert rules, and key PromQL queries
