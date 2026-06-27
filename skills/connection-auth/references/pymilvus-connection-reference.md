# PyMilvus MilvusClient — Connection Reference

## MilvusClient constructor parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `uri` | str | required | Connection URI. `"http://localhost:19530"` for local; `"https://<host>:<port>"` for remote |
| `user` | str | `""` | Username for username+password auth |
| `password` | str | `""` | Password for username+password auth |
| `token` | str | `""` | Token string for token-based auth (overrides user/password) |
| `db_name` | str | `"default"` | Target database name |
| `secure` | bool | `False` | Enable TLS. **Must be `True` for any remote deployment** |
| `server_name` | str | `""` | SNI hostname override for TLS certificate validation |
| `server_pem_path` | str | `""` | Path to server CA cert (required for IBM watsonx.data and mutual TLS) |
| `timeout` | float | `None` | Connection timeout in seconds |

---

## Worked example 1 — No authentication (local dev)

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")
collections = client.list_collections()
print(collections)  # [] on a fresh instance
```

---

## Worked example 2 — Username + password (self-hosted)

```python
from pymilvus import MilvusClient

client = MilvusClient(
    uri="https://milvus.internal:19530",
    user="admin",
    password="your-password",
    secure=True,
)
collections = client.list_collections()
```

---

## Worked example 3 — Token (managed cloud)

```python
import os
from pymilvus import MilvusClient

client = MilvusClient(
    uri=os.environ["MILVUS_URI"],
    token=os.environ["MILVUS_TOKEN"],  # format varies by provider; see milvuscraft-context
    secure=True,
)
collections = client.list_collections()
```

---

## Worked example 4 — IBM watsonx.data (API key auth)

```python
import os
from pymilvus import MilvusClient

client = MilvusClient(
    uri=f"https://ibmlhapikey:{os.environ['IBM_API_KEY']}@{os.environ['MILVUS_GRPC_HOST']}:{os.environ['MILVUS_GRPC_PORT']}",
    db_name="default",
    secure=True,
    server_pem_path="certs/watsonx.pem",
    server_name="watsonxdata",
)
```

---

## Worked example 5 — Non-default database

```python
from pymilvus import MilvusClient

client = MilvusClient(
    uri="https://milvus.internal:19530",
    token=os.environ["MILVUS_TOKEN"],
    db_name="production",
    secure=True,
)
```

---

## Notes

- `secure=True` is required for all remote connections. Omitting it causes TLS handshake
  failures that surface as "connection reset" or certificate errors.
- When `token` is set, `user` and `password` are ignored.
- Reuse a single `MilvusClient` per process — it caches schemas locally and is thread-safe.
  Creating a new client per request is a significant performance anti-pattern.
- Token format for managed Milvus cloud is typically `"<username>:<password>"` — consult
  milvuscraft-context → Deployment overrides for the exact format in your environment.
