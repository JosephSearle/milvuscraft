---
name: connection-auth
description: >
  Establish and validate a connection to any Milvus deployment using PyMilvus MilvusClient
  or the Milvus MCP server. Load this skill whenever the user mentions: connect to Milvus,
  set up Milvus connection, configure MCP server, authenticate to Milvus, MILVUS_URI,
  MILVUS_TOKEN, IBM watsonx.data, ibmlhapikey, connection refused, permission denied,
  credential error, or any Milvus connectivity question. Always load milvuscraft-context first.
allowed-tools: mcp__milvus__milvus_list_collections
---

# Milvuscraft Connection & Auth Skill

Establish and validate a PyMilvus or MCP connection to any Milvus deployment. Covers all
standard auth modes plus IBM watsonx.data gRPC. Defers to milvuscraft-context for
deployment-specific credential formats.

---

## Core Philosophy

Always validate a connection immediately after establishing it. An empty list from
`milvus_list_collections` is a success — the cluster is reachable and authenticated.
An error means the connection did not work; do not proceed to other operations.

Reuse a single `MilvusClient` instance per process — SDK v2 caches schemas locally,
cutting per-operation network/CPU. Creating a new client per request is a significant
anti-pattern.

---

## Step 1 — Determine URI format

```
Is Milvus running locally or in a dev environment?
  └─ YES → uri = "http://localhost:19530" (no TLS)
  └─ NO  → uri = "https://<host>:<grpc-port>" (TLS; secure=True required)
```

- Default gRPC port: **19530**
- IBM watsonx.data: port comes from Infrastructure Manager → Milvus service → View connect
  details → GRPC host/port (commonly 30884 or 8443-style; never assume 19530)
- Managed cloud hosts: check milvuscraft-context → Deployment overrides

---

## Step 2 — Choose authentication mode

Consult milvuscraft-context → Step 5 for credential format and Deployment overrides for
any environment-specific token structure.

| Mode | When | PyMilvus code |
|------|------|---------------|
| No auth | Local dev only | `MilvusClient(uri=uri)` |
| Username + password | Self-hosted with auth enabled | `MilvusClient(uri=uri, user="u", password="p", secure=True)` |
| Token | Managed cloud or token-based | `MilvusClient(uri=uri, token="<token>", secure=True)` |

See `references/pymilvus-connection-reference.md` for full `MilvusClient` parameter reference.

---

## Step 3 — IBM watsonx.data connection (gRPC/TLS)

IBM watsonx.data exposes Milvus over gRPC with mandatory TLS and IBM Cloud identity auth.
The auth model is different from standard Milvus.

**Credentials:**
- Username is always **`ibmlhapikey`** (for API-key auth) or **`ibmlhtoken_<username>`** (for IAM-token auth)
- Password is an IBM Cloud user **API key** (or the IAM token itself)
- A **server PEM certificate** is required — download it with:
  ```bash
  openssl s_client -showcerts -connect <host>:<port> </dev/null 2>/dev/null \
    | openssl x509 -outform PEM > certs/watsonx.pem
  ```

**MilvusClient (recommended):**

```python
import os
from pymilvus import MilvusClient

client = MilvusClient(
    uri=f"https://ibmlhapikey:{os.environ['IBM_API_KEY']}@{os.environ['MILVUS_GRPC_HOST']}:{os.environ['MILVUS_GRPC_PORT']}",
    db_name="default",
    secure=True,
    server_pem_path="certs/watsonx.pem",
    server_name="watsonxdata",   # must match the cert CommonName
)
```

**ORM / connections form (when MilvusClient is insufficient):**

```python
from pymilvus import connections

connections.connect(
    alias="default",
    host=os.environ["MILVUS_GRPC_HOST"],
    port=int(os.environ["MILVUS_GRPC_PORT"]),
    secure=True,
    user="ibmlhapikey",
    password=os.environ["IBM_API_KEY"],
    server_pem_path="certs/watsonx.pem",
    server_name="watsonxdata",
)
```

**IBM-specific limits to check milvuscraft-context → Deployment overrides:**
- Max 64 databases, 65,536 collections, 4,095 partitions (same as standard)
- User/Roles v2 APIs are **not supported** on IBM watsonx.data — manage RBAC via IBM IAM
- Recommended embedding dimension for non-Custom instance sizes: 384

---

## Step 4 — Configure the Milvus MCP server

Inject credentials via environment variables — never hardcode secrets in model context.

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "milvus": {
        "command": "uv",
        "args": ["run", "src/mcp_server_milvus/server.py",
                 "--milvus-uri", "https://<host>:<port>"],
        "transport": "stdio",
        "env": {
            "MILVUS_TOKEN": os.environ["MILVUS_TOKEN"],
            "MILVUS_DB": "default",
        },
    }
})
tools = await client.get_tools()
```

For Claude Code / Cowork: set `MILVUS_URI`, `MILVUS_TOKEN`, and `MILVUS_DB` as shell
environment variables before starting the MCP server.

---

## Step 5 — Validate the connection

Call `milvus_list_collections`. Any non-error response confirms the connection is live.

```json
{ "name": "milvus_list_collections", "arguments": {} }
```

- **Success**: `{"collections": []}` or a list of names. An empty list is expected on a fresh deployment.
- **Any error** → proceed to Step 6.

---

## Step 6 — Resolve common failures

| Error | Root cause | Fix |
|-------|-----------|-----|
| Connection refused | Wrong port or HTTP/HTTPS mismatch | Match `http://` vs `https://` and port to deployment |
| Certificate error | Missing `secure=True` or SNI mismatch | Add `secure=True`; set `server_name` to cert CN |
| Permission denied | Wrong credential format | Check milvuscraft-context → Deployment overrides |
| Timeout | Host unreachable or service starting | Retry with exponential backoff starting at 2 s |
| IBM: 401 Unauthorized | API key expired or wrong username | Regenerate IBM Cloud API key; confirm username is `ibmlhapikey` |

**Retry pattern for transient failures:**

```python
import time

def connect_with_retry(uri, token, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            client = MilvusClient(uri=uri, token=token, secure=True)
            client.list_collections()   # validate
            return client
        except Exception as e:
            if attempt == max_attempts - 1:
                raise
            wait = 2 ** attempt
            print(f"Connection attempt {attempt + 1} failed: {e}. Retrying in {wait}s...")
            time.sleep(wait)
```

---

## Reference Files

- `references/pymilvus-connection-reference.md` — Full `MilvusClient` constructor parameter
  reference and worked examples for all authentication modes
