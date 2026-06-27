<div align="center">

# MilvusCraft

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-D97757?style=flat&logo=anthropic&logoColor=white)](https://claude.ai/code)
[![Milvus](https://img.shields.io/badge/Milvus-2.x-00A3E0?style=flat&logoColor=white)](https://milvus.io)
[![PyMilvus](https://img.shields.io/badge/PyMilvus-SDK-3776AB?style=flat&logo=python&logoColor=white)](https://pymilvus.readthedocs.io)
[![MCP](https://img.shields.io/badge/MCP-enabled-5C4EE5?style=flat&logoColor=white)](https://modelcontextprotocol.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Production Claude Code plugin for senior engineers building with and on Milvus.

</div>

12 specialised skills, 4 lifecycle agents, and the official Milvus MCP server wired in.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Skills](#skills)
- [Agents](#agents)
- [MCP Tools](#mcp-tools)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

---

## Prerequisites

The plugin connects to the Milvus MCP server via `mcp-proxy`. You need both running
before enabling the plugin:

```bash
# Start the Milvus MCP server (adjust host/port to your deployment)
# Then start mcp-proxy forwarding to it:
uvx mcp-proxy --transport streamablehttp http://localhost:8000/mcp
```

If your Milvus MCP server runs on a different URL, set it in your plugin config:

```ini
milvus_mcp_url = http://<your-host>:<port>/mcp
```

---

## Installation

```bash
# Install directly from the repo
claude plugin install https://github.com/josephsearle/milvuscraft

# Or load locally for development
claude --plugin-dir /path/to/milvuscraft
```

Enable the plugin and set your MCP URL when prompted. The plugin is `defaultEnabled: false`
because it requires an external Milvus service.

---

## Skills

All skills are invoked as `/milvuscraft:<skill-name>`.

| Skill | Purpose |
|---|---|
| `/milvuscraft:context` | **Load first.** Shared reference card: deployment modes, cluster limits, index types, consistency levels, auth patterns. |
| `/milvuscraft:connection-auth` | Establish and validate connections. Covers all auth modes including IBM watsonx.data gRPC. |
| `/milvuscraft:schema-design` | Design collection schemas before creation. Schema decisions are largely irreversible. |
| `/milvuscraft:collection-lifecycle` | Create, load, release, rename, drop collections. Alias-based blue-green reindexing. |
| `/milvuscraft:data-ingestion` | Insert, upsert, delete, bulk-insert. Batching guidance and async ingestion (23× speedup). |
| `/milvuscraft:index-management` | Choose and tune indexes. Decision tree across HNSW, IVF variants, DISKANN, scalar indexes. |
| `/milvuscraft:search-optimization` | Vector ANN, scalar filters, hybrid search, BM25, grouping, range search, pagination. |
| `/milvuscraft:lifecycle-compaction-ttl` | TTL expiry, compaction strategy, mmap config, cold data release. |
| `/milvuscraft:multi-tenancy` | Isolation strategies: database/collection/partition-key. RBAC with 56 privileges. |
| `/milvuscraft:observability` | Prometheus + Grafana setup. Key metrics, SLO baselines, alert rules. |
| `/milvuscraft:backup-restore` | Backup/restore with milvus-backup CLI. DR runbooks, cross-environment promotion. |
| `/milvuscraft:diagnostics` | Triage entry point: slow search, wrong results, auth failures, index not built. |

> **Hub skill:** `context` is loaded automatically by the specialist agents and should
> be loaded manually before any other skill when working interactively.

---

## Agents

| Agent | Model | Purpose |
|---|---|---|
| `milvus-architect` | Opus / high | Schema design, index strategy, tenancy modelling, capacity planning. Read-only recommendations. |
| `milvus-engineer` | Sonnet / high | PyMilvus implementation: collections, ingestion, search code. Writes to an isolated worktree. |
| `milvus-operator` | Sonnet / medium | Backup/restore, monitoring dashboards, TTL/compaction, DR runbooks. |
| `milvus-diagnostician` | Sonnet / high | Root cause analysis for any Milvus issue. Uses live MCP tools to inspect collection state. |

---

## MCP Tools

Skills that use the Milvus MCP server call `mcp__milvus__*` tools. Operations not covered
by MCP fall back to PyMilvus SDK (`MilvusClient`). Skills document which path they use.

---

## Development

```bash
# Validate the plugin manifest, skill frontmatter, and agent definitions
claude plugin validate .

# Smoke-test locally
claude --plugin-dir .
# Then inside the session:
# /milvuscraft:context
# /agents
# /plugin list
```

---

## Contributing

Contributions are welcome. To get started:

1. Fork the repository and create a branch from `main`
2. Make your changes and validate the plugin manifest: `claude plugin validate .`
3. Open a pull request with a clear description of the change

Please open an issue before starting work on a significant new skill or agent.

---

## License

[MIT](https://opensource.org/licenses/MIT) © 2026 Joseph Searle
