---
name: milvus-connection-auth
description: >
  Establish and validate a connection to any Milvus deployment using PyMilvus MilvusClient
  or the Zilliz MCP server. Load this skill whenever the user mentions: connect to Milvus,
  set up Milvus connection, configure MCP server, authenticate to Milvus, MILVUS_URI,
  MILVUS_TOKEN, connection refused, permission denied, credential error, or any Milvus
  connectivity question. Always load milvus-context first.
allowed-tools: mcp__milvus__milvus_list_collections
---

# Milvus Connection & Auth Skill

Establish and validate a PyMilvus or MCP connection to any Milvus deployment. Covers all
standard auth modes and defers to milvus-context for deployment-specific credential formats.

---

## Core Philosophy

Always validate a connection immediately after establishing it. An empty list from
`milvus_list_collections` is a success — the cluster is reachable and authenticated.
An error means the connection did not work; do not proceed to other operations.

---

## Step 1 — Determine URI format

```
Is Milvus running locally or in a dev environment?
  └─ YES → uri = "http://localhost:19530" (no TLS)
  └─ NO  → uri = "https://<host>:<grpc-port>" (TLS; secure=True required)
```

- Default gRPC port: **19530**
- Managed cloud hosts: check milvus-context → Deployment overrides for exact hostname format

---

## Step 2 — Choose authentication mode

Consult milvus-context → Step 5 for credential format. Consult Deployment overrides for
any environment-specific token structure.

| Mode | When | PyMilvus code |
|------|------|---------------|
| No auth | Local dev only | `MilvusClient(uri=uri)` |
| Username + password | Self-hosted with auth enabled | `MilvusClient(uri=uri, user="u", password="p", secure=True)` |
| Token | Managed cloud or token-based | `MilvusClient(uri=uri, token="<token>", secure=True)` |

See `references/pymilvus-connection-reference.md` for full `MilvusClient` parameter reference.

---

## Step 3 — MCP server is pre-configured

The Milvus MCP server is bundled with this plugin and starts automatically. Credentials
were entered at install time via `userConfig` and are injected into the server's environment:

- `MILVUS_URI` → your configured URI
- `MILVUS_TOKEN` → your configured token (stored in system keychain)
- `MILVUS_DB` → your configured database (default: "default")

To update credentials: run `/plugin configure milvus` and re-enter values.
Run `/reload-plugins` if the MCP server is not responding after a credential change.

---

## Step 4 — Validate the connection

Call `milvus_list_collections`. Any non-error response confirms the connection is live.

```json
{ "name": "milvus_list_collections", "arguments": {} }
```

- **Success**: `{"collections": []}` or a list of names. An empty list is expected on a
  fresh deployment.
- **Any error** → proceed to Step 5.

---

## Step 5 — Resolve common failures

| Error | Root cause | Fix |
|-------|-----------|-----|
| Connection refused | Wrong port or HTTP/HTTPS mismatch | Match `http://` vs `https://` and port to deployment |
| Certificate error | Missing `secure=True` or SNI mismatch | Add `secure=True`; set `server_name` if needed |
| Permission denied | Wrong credential format | Check milvus-context → Deployment overrides |
| Timeout | Host unreachable or service starting | Retry with exponential backoff starting at 2 s |

---

## Reference Files

- `references/pymilvus-connection-reference.md` — Full `MilvusClient` constructor parameter
  reference and worked examples for all three authentication modes
