# Milvus Multi-Tenancy — Worked Examples

## Example 1: Partition-key RAG with 10K tenants

Use when you have thousands of tenants and want a single collection with automatic routing.
Suitable up to ~10M tenants. Isolation is logical (filter-based), not physical.

### Schema setup

```python
from pymilvus import MilvusClient

client = MilvusClient(uri=..., token=...)

client.create_collection(
    collection_name="tenant_rag",
    dimension=768,
    metric_type="COSINE",
    # field_schema with partition key
    schema=client.create_schema(
        auto_id=True,
        enable_dynamic_field=False,
    ).add_field("id", "INT64", is_primary=True, auto_id=True)
     .add_field("tenant_id", "VARCHAR", max_length=128, is_partition_key=True)
     .add_field("chunk_text", "VARCHAR", max_length=2048)
     .add_field("doc_id", "VARCHAR", max_length=128)
     .add_field("embedding", "FLOAT_VECTOR", dim=768),
    index_params=client.prepare_index_params().add_index(
        field_name="embedding",
        index_type="HNSW",
        metric_type="COSINE",
        params={"M": 16, "efConstruction": 256},
    ).add_index(
        field_name="doc_id",
        index_type="INVERTED",
    ),
)
```

### Insert for a specific tenant

```python
client.insert(
    collection_name="tenant_rag",
    data=[
        {
            "tenant_id": "acme",
            "chunk_text": "Quarterly revenue exceeded targets...",
            "doc_id": "doc-001",
            "embedding": [0.1, 0.2, ...],  # 768 floats
        }
    ],
)
```

### Tenant-scoped search (always include tenant_id filter)

```python
results = client.search(
    collection_name="tenant_rag",
    data=[[0.1, 0.2, ...]],   # query embedding
    limit=10,
    filter="tenant_id == 'acme'",
    output_fields=["chunk_text", "doc_id"],
    search_params={"ef": 64},
)
```

**Critical:** never run a search without `filter="tenant_id == '<value>'"` — without the
filter, results from all tenants are returned regardless of partition key.

### Validation

```python
# Insert tenant A data, then confirm tenant B search returns nothing
client.insert("tenant_rag", [{"tenant_id": "beta", "embedding": [...], ...}])

results = client.search(
    "tenant_rag", data=[[...]], limit=10,
    filter="tenant_id == 'acme'",
)
assert all(r["entity"]["tenant_id"] == "acme" for r in results[0])
```

---

## Example 2: Collection-per-tenant SaaS with 500 customers

Use when tenants need different schemas, independent backup/restore, or physical data
separation. Manages up to 65,536 collections per cluster.

### Provisioning a new tenant

```python
TENANT_TEMPLATE_SCHEMA = [
    {"field_name": "id",          "data_type": "Int64",       "is_primary": True, "auto_id": True},
    {"field_name": "chunk_text",  "data_type": "VarChar",     "max_length": 2048},
    {"field_name": "embedding",   "data_type": "FloatVector", "dim": 768},
    {"field_name": "doc_id",      "data_type": "VarChar",     "max_length": 128},
]

def provision_tenant(client, tenant_id: str):
    collection_name = f"tenant_{tenant_id}_docs"
    if collection_name in client.list_collections():
        return  # already provisioned

    client.create_collection(
        collection_name=collection_name,
        metric_type="COSINE",
        field_schema=TENANT_TEMPLATE_SCHEMA,
        index_params=[{
            "field_name": "embedding",
            "index_type": "HNSW",
            "metric_type": "COSINE",
            "params": {"M": 16, "efConstruction": 256},
        }],
    )
    client.load_collection(collection_name)
```

### Blue-green alias promotion (zero-downtime schema migration)

```python
OLD = "tenant_acme_docs"
NEW = "tenant_acme_docs_v2"

# 1. Build and populate new collection
provision_tenant(client, "acme_v2")
# ... migrate data to NEW ...

# 2. Swap alias atomically
client.alter_alias(collection_name=NEW, alias="tenant_acme_prod")

# 3. Release and drop old collection
client.release_collection(OLD)
client.drop_collection(OLD)
```

### RBAC per tenant

```python
def grant_tenant_access(client, tenant_id: str, username: str):
    role_name = f"role_{tenant_id}"
    collection_name = f"tenant_{tenant_id}_docs"

    client.create_role(role_name)
    client.grant_privilege(role_name, "Collection", collection_name, "Query")
    client.grant_privilege(role_name, "Collection", collection_name, "Search")
    client.grant_role(user_name=username, role_name=role_name)
```
