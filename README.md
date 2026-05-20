# Milvus

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)](https://github.com/JosephSearle/milvus-plugin)

Claude Code plugin with 12 skills for operating Milvus vector databases — standard, IBM Cloud, and Zilliz Cloud.

Milvus operations span a broad surface: schema design, index tuning, ingestion strategies, hybrid search, multi-tenancy, observability, and disaster recovery. This plugin loads the right context and guidance into Claude automatically, so you can operate your Milvus cluster through natural language without leaving your terminal.

## Table of Contents

- [Highlights](#highlights)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Deployment Profiles](#deployment-profiles)
- [MCP Server](#mcp-server)
- [Skills](#skills)
- [MCP Tools](#mcp-tools)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## Highlights

- **12 specialised skills** — from schema design and index tuning to TTL management, backup/restore, and Prometheus observability; each skill loads only what Claude needs for the task
- **Three deployment targets** — Standard Milvus (Lite/Standalone/Distributed), IBM watsonx.data, and Zilliz Cloud, each with a tailored constraint profile
- **Bundled MCP server** — ships `mcp-server-milvus` via `uvx`; starts automatically when the plugin is enabled
- **Context-aware loading** — `milvus-context` is always loaded first so Claude knows your exact environment constraints before giving advice
- **Secure credential handling** — Milvus token is marked `sensitive` in `userConfig` and stored in your OS keychain, never written to project files

## Installation

**Prerequisite:** `uv` must be installed — the bundled MCP server runs via `uvx`.

```bash
pip install uv
# or
brew install uv
```

### From marketplace

```bash
/plugin install milvus-plugin
```

### Local development

```bash
claude --plugin-dir /path/to/milvus-plugin
```

On first enable, Claude Code prompts for four configuration values:

| Field | Required | Description |
|-------|----------|-------------|
| **Milvus URI** | Yes | Connection URI — `http://localhost:19530` or `https://host:19530` |
| **Milvus Token** | No | `username:password` for self-hosted; API key for cloud. Leave empty for no-auth dev. |
| **Database name** | No | Milvus database to target (default: `default`) |
| **Deployment profile** | Yes | One of `standard`, `ibm_cloud`, `zilliz_cloud`, or `custom` |

To update credentials after installation:

```bash
/plugin configure milvus
```

## Quick Start

After installing and configuring, Claude loads the appropriate skill automatically based on what you say:

```text
You: Design a schema for a document search collection with dense + sparse vectors
Claude: [loads milvus-context → milvus-schema-design, proposes fields, BM25 analyser, HNSW index]

You: My hybrid search latency jumped to 800ms — what's wrong?
Claude: [loads milvus-context → milvus-diagnostics, walks through nprobe / ef / segment triage]
```

You can also invoke any skill directly:

```bash
/milvus:milvus-search-optimization
```

## Deployment Profiles

Profiles live in `profiles/` and are inlined into the `milvus-context` skill at load time so Claude knows the exact constraints for your environment.

| Profile | File | Use when |
|---------|------|----------|
| `standard` | `profiles/standard.yaml` | Open-source Milvus Lite, Standalone, or Distributed |
| `ibm_cloud` | `profiles/ibm_cloud.yaml` | IBM watsonx.data managed Milvus |
| `zilliz_cloud` | `profiles/zilliz_cloud.yaml` | Zilliz Cloud fully managed |
| `custom` | `profiles/custom.yaml` | Any other managed deployment — edit manually |

## MCP Server

The plugin bundles [`mcp-server-milvus`](https://github.com/zilliztech/mcp-server-milvus) via `uvx`. It starts automatically when the plugin is enabled. Credentials are injected from your `userConfig` — they are never stored in plain text in project files.

## Skills

All skills are namespaced under `/milvus:`. Claude loads them automatically based on context; you can also invoke any skill directly.

| Skill | Invoke | What it does |
|-------|--------|--------------|
| `milvus-context` | `/milvus:milvus-context` | Shared reference card — loaded first before every other skill. Inlines active deployment profile. |
| `milvus-connection-auth` | `/milvus:milvus-connection-auth` | Establish and validate Milvus connection; resolve auth failures. |
| `milvus-schema-design` | `/milvus:milvus-schema-design` | Design immutable collection schema: fields, primary key, vectors, BM25, partition key. |
| `milvus-collection-lifecycle` | `/milvus:milvus-collection-lifecycle` | Create, load, release, inspect, alias, and drop collections. |
| `milvus-index-management` | `/milvus:milvus-index-management` | Choose, build, and tune vector indexes (HNSW, IVF_\*, SCANN, DISKANN, FLAT, GPU). |
| `milvus-data-ingestion` | `/milvus:milvus-data-ingestion` | Insert, upsert, delete, bulk insert; batch sizing; flush strategy. |
| `milvus-search-optimization` | `/milvus:milvus-search-optimization` | Vector ANN, hybrid, BM25, scalar filter, grouping, range search; tune ef/nprobe. |
| `milvus-multi-tenancy` | `/milvus:milvus-multi-tenancy` | Database-, collection-, partition-, or partition-key–level tenant isolation; RBAC. |
| `milvus-lifecycle-compaction-ttl` | `/milvus:milvus-lifecycle-compaction-ttl` | TTL expiry, compaction, cold-data release, segment size tuning. |
| `milvus-backup-restore` | `/milvus:milvus-backup-restore` | Backup and restore with milvus-backup CLI; cross-environment promotion. |
| `milvus-observability` | `/milvus:milvus-observability` | Prometheus scraping, Grafana dashboards, SLO baselines, alert rules. |
| `milvus-diagnostics` | `/milvus:milvus-diagnostics` | Triage: slow search, wrong results, auth errors, ingestion failures, empty results. |

## MCP Tools

The MCP server exposes these tools, pre-approved in the relevant skills:

| Tool | Used by |
|------|---------|
| `milvus_list_collections` | connection-auth |
| `milvus_create_collection` | collection-lifecycle, index-management, multi-tenancy |
| `milvus_load_collection` | collection-lifecycle |
| `milvus_release_collection` | collection-lifecycle, lifecycle-compaction-ttl |
| `milvus_get_collection_info` | collection-lifecycle, diagnostics |
| `milvus_insert_data` | data-ingestion |
| `milvus_delete_entities` | data-ingestion |
| `milvus_vector_search` | search-optimization |
| `milvus_text_search` | search-optimization |
| `milvus_hybrid_search` | search-optimization |
| `milvus_text_similarity_search` | search-optimization (Milvus ≥2.6 only) |
| `milvus_query` | search-optimization, diagnostics |

## Development

```bash
# Test locally
claude --plugin-dir ./milvus-plugin

# Reload after changes (within a session)
/reload-plugins

# Validate the plugin manifest
claude plugin validate ./milvus-plugin
```

## Contributing

Contributions are welcome. To get started:

1. Fork the repository and create a branch from `main`
2. Make your changes — new skills go in `skills/`, profile tweaks in `profiles/`
3. Test locally: `claude --plugin-dir ./milvus-plugin`
4. Validate the manifest: `claude plugin validate ./milvus-plugin`
5. Open a pull request with a clear description of the change

Please open an issue before starting work on a significant new skill so we can align on scope and naming conventions.

**Bug reports and feature requests:** [Open a GitHub Issue](https://github.com/JosephSearle/milvus-plugin/issues)

## License

[MIT](LICENSE) © 2025 Joseph Searle
