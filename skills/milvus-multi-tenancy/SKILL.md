---
name: milvus-multi-tenancy
description: >
  Design multi-tenant isolation for a Milvus deployment. Load this skill whenever the user
  mentions: multiple tenants, tenant isolation, separate data by customer, partition key,
  database per tenant, collection per tenant, RBAC for tenants, resource groups, tenant-level
  search, data isolation, SaaS on Milvus, or how to separate tenant data in Milvus. Always
  load milvus-context and milvus-schema-design first.
allowed-tools: mcp__milvus__milvus_create_collection
---

# Milvus Multi-Tenancy Skill

Select and configure the correct isolation strategy based on tenant count and isolation
requirements. Resource groups require Milvus Distributed. RBAC details vary by deployment —
check milvus-context → Deployment overrides for your environment's access control mechanism.

---

## Core Philosophy

There is no single best multi-tenancy strategy. Tenant count and isolation requirements are
the two controlling variables. Choosing the wrong strategy is hard to reverse — apply the
decision table before any schema or collection design.

---

## Step 1 — Select the isolation strategy

| Tenant count | Strategy | Data isolation | RBAC complexity |
|-------------|----------|----------------|-----------------|
| ≤64 with strict isolation | Database-per-tenant | Strong | High |
| ≤65,536 with schema variation per tenant | Collection-per-tenant | Strong | Medium |
| ≤1,024 balanced cost/isolation | Partition-per-tenant | Medium | Low |
| 10,000–10M+, cost priority | Partition-key field | Medium | None at DB level |

**Ceiling awareness:**

1. Database-per-tenant ceiling: **64 databases** per cluster
2. Collection-per-tenant ceiling: **65,536 collections** per cluster
3. Partition-per-tenant ceiling: **4,095 partitions** per collection
4. Partition-key: no hard ceiling on tenant count — Milvus uses 64 virtual partitions internally

---

## Step 2 — Apply the chosen strategy

**Database-per-tenant:**

```python
client.create_database("tenant_acme")
client.using_database("tenant_acme")
# Create collections inside the tenant database
```

**Collection-per-tenant:**

Create one collection per tenant with independent schema. Use aliases for blue-green
promotion. Manage with milvus-collection-lifecycle.

**Partition-per-tenant:**

```python
client.create_partition("shared_collection", partition_name="tenant_acme")
# Insert with partition_name set; search with partition_names=["tenant_acme"]
```

**Partition-key field (most scalable):**

Set `is_partition_key=True` on a VarChar or Int64 field in the schema (see milvus-schema-design).
Only one partition-key field per collection. Milvus automatically routes searches to the
matching virtual partition when `filter_expr` includes the key field.

```json
{
  "name": "milvus_create_collection",
  "arguments": {
    "collection_name": "tenant_docs",
    "metric_type": "COSINE",
    "field_schema": [
      {"field_name": "id",        "data_type": "Int64",       "is_primary": true, "auto_id": true},
      {"field_name": "tenant_id", "data_type": "VarChar",     "max_length": 128, "is_partition_key": true},
      {"field_name": "embedding", "data_type": "FloatVector",  "dim": 768}
    ],
    "index_params": [
      {"field_name": "embedding", "index_type": "HNSW", "metric_type": "COSINE", "params": {"M": 16, "efConstruction": 256}}
    ]
  }
}
```

Always include `"tenant_id == '<value>'"` in `filter_expr` on every search to restrict
results to the calling tenant.

See `references/tenancy-worked-examples.md` for end-to-end examples.

---

## Step 3 — Configure resource groups (Distributed only)

Resource groups physically assign query nodes to tenant tiers (e.g., premium vs standard).
Requires Milvus Distributed on Kubernetes. No MCP tool — use PyMilvus declarative API
(Milvus ≥2.4.1):

```python
from pymilvus import MilvusClient, ResourceGroupConfig

client.create_resource_group(
    "premium_tenants",
    config=ResourceGroupConfig(
        requests={"node_num": 2},
        limits={"node_num": 4},
    ),
)
client.transfer_replica(
    source_group="__default_resource_group",
    target_group="premium_tenants",
    collection_name="tenant_docs",
    num_replicas=1,
)
```

---

## Step 4 — Configure RBAC

Milvus RBAC has three privilege scopes: global, database, and collection level.

```python
# Create role
client.create_role("tenant_acme_reader")

# Grant read privileges on a specific collection
client.grant_privilege(
    role_name="tenant_acme_reader",
    object_type="Collection",
    object_name="tenant_docs",
    privilege="Query",
)

# Assign role to user
client.grant_role(user_name="acme_user", role_name="tenant_acme_reader")
```

Managed deployments may wrap RBAC in a platform policy layer — check milvus-context →
Deployment overrides.

---

## Step 5 — Verify tenant isolation

Insert rows with `tenant_id="a"`, search with `filter_expr="tenant_id == 'a'"`, confirm
no rows from `tenant_id="b"` are returned. Repeat with `tenant_id="b"` and verify no
cross-tenant bleed in either direction.

---

## Reference Files

- `references/tenancy-worked-examples.md` — End-to-end examples for partition-key RAG with
  10K tenants and collection-per-tenant SaaS with 500 customers
