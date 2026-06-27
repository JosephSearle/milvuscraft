---
name: multi-tenancy
description: >
  Design multi-tenant isolation for a Milvus deployment. Load this skill whenever the user
  mentions: multiple tenants, tenant isolation, separate data by customer, partition key,
  database per tenant, collection per tenant, RBAC for tenants, resource groups, tenant-level
  search, data isolation, SaaS on Milvus, or how to separate tenant data in Milvus. Always
  load milvuscraft-context and milvuscraft-schema-design first.
allowed-tools: mcp__milvus__milvus_create_collection
---

# Milvuscraft Multi-Tenancy Skill

Select and configure the correct isolation strategy based on tenant count and isolation
requirements. Resource groups require Milvus Distributed. RBAC details vary by deployment —
check milvuscraft-context → Deployment overrides for your environment's access control mechanism.

---

## Core Philosophy

There is no single best multi-tenancy strategy. Tenant count and isolation requirements are
the two controlling variables. Choosing the wrong strategy is hard to reverse — apply the
decision table before any schema or collection design.

---

## Step 1 — Select the isolation strategy

| Tenant count | Strategy | Data isolation | RBAC enforcement | Notes |
|-------------|----------|----------------|-----------------|-------|
| ≤64 with strict isolation | Database-per-tenant | Strong | Yes | Wastes resources on idle tenants; flexible per-tenant schema |
| ≤65,536 with schema variation | Collection-per-tenant | Strong | Yes | Per-tenant schema; supports RBAC |
| ≤1,024 balanced cost/isolation | Partition-per-tenant | Medium | No (partition level) | Manual partition management |
| 10,000–10M+, cost priority | Partition-key field | Medium | No (app-layer only) | Most scalable; bulk insert not supported |

**Key ceilings (from milvuscraft-context):**

1. Database-per-tenant: **64 databases** per cluster
2. Collection-per-tenant: **65,536 collections** per cluster
3. Partition-per-tenant: **4,095 partitions** per collection
4. Partition-key: no hard ceiling — Milvus uses 64 virtual partitions internally

**Isolation vs scale trade-off:** Partition-key scales furthest but provides only logical
isolation (filter-based, enforced in application code). Database or collection level provides
physical isolation with RBAC enforcement at the database layer.

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
promotion. Manage with milvuscraft-collection-lifecycle.

**Partition-per-tenant:**

```python
client.create_partition("shared_collection", partition_name="tenant_acme")
# Insert with partition_name set; search with partition_names=["tenant_acme"]
```

**Partition-key field (most scalable):**

Set `is_partition_key=True` on a VarChar or Int64 field in the schema (see milvuscraft-schema-design).
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

Always include `"tenant_id == '<value>'"` in `filter_expr` on every search. Without this
filter, results from all tenants are returned regardless of the partition key.

**Partition Key Isolation (HNSW only, Milvus 2.4+):** builds a separate HNSW index per
partition key group, providing stronger recall isolation between tenants. Enable via
collection property. Works only with HNSW; adds memory overhead proportional to tenant count.

See `references/tenancy-worked-examples.md` for complete end-to-end examples.

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

Milvus 2.5+ uses `grant_privilege_v2` with 56 privilege types and 9 built-in groups
(COLL_RO, COLL_RW, COLL_ADMIN, DB_RO, DB_RW, DB_Admin, Cluster_RO, Cluster_RW, Cluster_Admin).
Grants are non-cascading — setting at instance level does not propagate to DBs/collections.

```python
# Create role scoped to a specific collection
client.create_role("tenant_acme_reader")
client.grant_privilege_v2(
    role_name="tenant_acme_reader",
    privilege="CollectionReadOnly",
    collection_name="tenant_docs",
    db_name="default",
)

# Assign role to user
client.create_user(user_name="acme_svc", password="<password>")
client.grant_role(user_name="acme_svc", role_name="tenant_acme_reader")
```

Best practices: least privilege; scope to specific collections not `*`; revoke roles promptly
on role change; rotate the default `root:Milvus` password immediately.

Check milvuscraft-context → Deployment overrides — IBM watsonx.data does not support User/Roles v2 APIs; use IBM IAM instead.

---

## Step 5 — Verify tenant isolation

Insert rows with `tenant_id="a"`, search with `filter_expr="tenant_id == 'a'"`, confirm
no rows from `tenant_id="b"` are returned. Repeat with `tenant_id="b"` and verify no
cross-tenant bleed in either direction.

---

## Reference Files

- `references/tenancy-worked-examples.md` — End-to-end examples for partition-key RAG with
  10K tenants and collection-per-tenant SaaS with 500 customers
