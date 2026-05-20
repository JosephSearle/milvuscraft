# Milvus Grafana Dashboard Reference

## Official dashboard templates

The Milvus project maintains official Grafana dashboard JSON files in the main repository.
Import these into any Grafana instance (self-hosted or managed) for a complete baseline.

| Dashboard | Location in repo | Key panels |
|-----------|-----------------|------------|
| Milvus overview | `deployments/monitor/grafana/milvus-overview.json` | Request rate, error rate, latency p50/p99/p999 |
| Query coord | `deployments/monitor/grafana/query_coord.json` | Collection load/release, replica management |
| Query node | `deployments/monitor/grafana/query_node.json` | Search latency by phase, segment counts, queue depth |
| Data coord | `deployments/monitor/grafana/data_coord.json` | Compaction task counts, flush size, segment state |
| Data node | `deployments/monitor/grafana/data_node.json` | Write throughput, binlog flush, segment sealing |
| Index coord | `deployments/monitor/grafana/index_coord.json` | Index build queue, task state |
| Proxy | `deployments/monitor/grafana/proxy.json` | Insert/search/query latency, mutation rate |

Source: `github.com/milvus-io/milvus` repository, `deployments/monitor/grafana/` directory.
Pin to the tag matching your deployed Milvus version.

---

## Importing into self-hosted Grafana

```bash
# 1. Download the dashboard JSON
curl -sL https://raw.githubusercontent.com/milvus-io/milvus/<version>/deployments/monitor/grafana/query_node.json \
  -o query_node.json

# 2. Import via Grafana API
curl -X POST http://grafana:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <grafana-api-key>" \
  -d "{\"dashboard\": $(cat query_node.json), \"overwrite\": true, \"folderId\": 0}"
```

Or use the Grafana UI: **+ → Import → Upload JSON file**.

---

## Importing into managed Grafana (Grafana Cloud)

1. Log in to your Grafana Cloud instance
2. Go to **Dashboards → New → Import**
3. Upload the dashboard JSON file downloaded from the Milvus repo
4. Set the Prometheus data source to your scrape target
5. Click **Import**

---

## Recommended alert rules (Prometheus alerting)

```yaml
groups:
  - name: milvus
    rules:
      - alert: MilvusSearchLatencyHigh
        expr: |
          histogram_quantile(0.99,
            rate(milvus_proxy_sq_latency_bucket[5m])
          ) > 2
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Milvus search p99 latency > 2s"
          description: "Run milvus-diagnostics → Branch A"

      - alert: MilvusCompactionBacklogHigh
        expr: milvus_datacoord_compaction_task_num > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Milvus compaction backlog > 100 tasks"
          description: "Reduce delete rate or trigger manual compact (milvus-lifecycle-compaction-ttl)"

      - alert: MilvusInsertLatencyHigh
        expr: |
          histogram_quantile(0.99,
            rate(milvus_proxy_mutation_latency_bucket[5m])
          ) > 0.5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Milvus insert p99 latency > 500ms"
          description: "Check batch sizing in milvus-data-ingestion"
```

---

## Key PromQL queries for ad-hoc investigation

```promql
# Search p99 latency (ms)
histogram_quantile(0.99, rate(milvus_proxy_sq_latency_bucket[5m])) * 1000

# Search queue wait p99 (ms)
histogram_quantile(0.99, rate(milvus_querynode_sq_queue_latency_bucket[5m])) * 1000

# Insert throughput (rows/s)
rate(milvus_proxy_mutation_latency_count[1m])

# Loaded segment count per collection
milvus_querynode_segment_num

# Pending compaction tasks
milvus_datacoord_compaction_task_num
```
